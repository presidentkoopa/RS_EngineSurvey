# U2 — UZDoom 5.0 render backend (Vulkan, with OpenGL contrast) and stereo/VR plumbing

UZDoom 5.0's Vulkan backend is an immediate-mode emulation of the old OpenGL state machine. Every `FRenderState::Draw` call runs a full `Apply()`, which copies a 384-byte `StreamData` record into a persistently mapped UBO, pushes 68 bytes of push constants, and often rebinds descriptor sets. It then issues one `vkCmdDraw` per wall, flat section or sprite. There is no batching, no indirect drawing, no bindless descriptors and no multiview. The frame is fully serialized: `Update()` ends every frame by waiting on every submit fence, so the CPU never records frame N+1 while the GPU runs frame N. Stereo is a CPU-side loop in `RenderViewpoint` that re-runs the whole scene pipeline for each eye: BSP traversal, draw-list build, command recording, SSAO and post-processing. In the Vulkan backend that loop does not even produce a stereo image: `NextEye` is empty, there are no eye buffers and there is no stereo present step, so the second eye overwrites the first. The codebase has no OpenXR or OpenVR code at all. VR in this engine means building a new presentation and submission layer, not turning on an existing one.

---

## 1. Frame skeleton (Vulkan)

**Data flow per displayed frame** (all on the main thread):

1. `D_Display` calls `screen->BeginFrame()` [U] src/d_main.cpp:1219. `VulkanRenderDevice::BeginFrame` resets the viewpoint buffer, resets the timestamp queries, (re)creates the scene and pipeline images if the size or MSAA count changed, resets the render state and re-allocates and re-writes the two global descriptor sets [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:451-461, [U] src/common/rendering/vulkan/renderer/vk_descriptorset.cpp:68-111.
2. `RenderView` [U] src/rendering/hwrenderer/hw_entrypoint.cpp:311-407:
   - resets the flat vertex buffer and clears the light, bone and viewpoint buffers once per frame (lines 314-315 and 346-348);
   - renders any changed canvas textures and camera textures, each through its own `RenderViewpoint` call (lines 358-386);
   - calls `RenderViewpoint(..., mainview=true, toscreen=true)` (line 403).
3. `RenderViewpoint` [U] src/rendering/hwrenderer/hw_entrypoint.cpp:97-191 updates the 1D shadow map once (lines 103-110), then runs the **eye loop** (lines 126-188); see §8.
4. `VulkanRenderDevice::Update` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:201-222 binds the post-process image as the target, records all 2D (`Draw2D`), ends the render pass, resets the stream writers, then calls `mCommands->WaitForCommands(true)` (line 218).
5. `WaitForCommands(true)` [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:163-199 does the following, in order:
   - **acquires** the swapchain image (`AcquireImage`, which also records the final "present" post-process pass into the swapchain image) [U] src/common/rendering/vulkan/textures/vk_framebuffer.cpp:48-77;
   - submits;
   - runs the CPU FPS limiter if vsync is off;
   - calls `vkQueuePresentKHR`;
   - **blocks on all outstanding submit fences** (lines 182-188);
   - frees the frame's delete lists.

```cpp
// [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:173-188
	FlushCommands(finish, true, uploadOnly);
	if (finish)
	{
		if (!fb->GetVSync())
			fb->FPSLimit();
		fb->GetFramebufferManager()->QueuePresent();
	}
	int numWaitFences = min(mNextSubmit, (int)maxConcurrentSubmitCount);
	if (numWaitFences > 0)
	{
		vkWaitForFences(fb->device->device, numWaitFences, mSubmitWaitFences, VK_TRUE, std::numeric_limits<uint64_t>::max());
		vkResetFences(fb->device->device, numWaitFences, mSubmitWaitFences);
	}
```

**Inference:** this is a "0 frames in flight" design. The CPU and GPU overlap only inside a frame, through the mid-frame flushes described in §6. Once the last submit of frame N goes out, the CPU sits idle until the GPU drains it. Wall-clock frame time is roughly the CPU recording time plus the GPU tail after the last flush. At 90 Hz (11.1 ms) that serialization is the single biggest structural obstacle. It is also what makes the single, non-ring-buffered stream, matrix, light and vertex buffers safe (§3). Any move to N frames in flight has to multi-buffer all of them at the same time.

---

## 2. Draw submission: `FRenderState` → `VkRenderState::Apply` → `vkCmdDraw`

### 2.1 Call chain and per-draw work

The scene code sets state on the API-neutral `FRenderState`: `SetMaterial`, `SetColor`, `SetFog`, `SetLightIndex` and others [U] src/common/rendering/hwrenderer/data/hw_renderstate.h:331-701. Most of these setters write straight into the `mStreamData` struct (for example `SetColor`, lines 341-345, and `SetGlowParams`, lines 462-466). It then calls the virtual `Draw` or `DrawIndexed` (lines 742-743). One wall equals one draw:

```cpp
// [U] src/rendering/hwrenderer/scene/hw_walls.cpp:93-95
	assert(vertcount > 0);
	state.SetLightIndex(dynlightindex);
	state.Draw(DT_TriangleFan, vertindex, vertcount);
```

Flats issue one `DrawIndexed` per section [U] src/rendering/hwrenderer/scene/hw_flats.cpp:192-204. `VkRenderState::Draw` always applies when `apply` is true, which is the default [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:58-72. Only portal code ever skips the apply: the stencil fan primitives and the plane-mirror/horizon flats apply only on the first primitive (`i == 0`), and one horizon strip passes `false` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:222,877,970,1136. The stencil caps pass `screen->IsVulkan()`, so under Vulkan they do apply [U] src/rendering/hwrenderer/scene/hw_portal.cpp:231,235. Any pending state change (`mNeedApply`) forces an apply regardless.

`Apply` runs these steps for every draw:

```cpp
// [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:185-203
	mApplyCount++;
	if (mApplyCount >= vk_submit_size)
	{
		fb->GetCommands()->FlushCommands(false);
		mApplyCount = 0;
	}
	ApplyStreamData();
	ApplyMatrices();
	ApplyRenderPass(dt);
	ApplyScissor();
	ApplyViewport();
	ApplyStencilRef();
	ApplyDepthBias();
	ApplyPushConstants();
	ApplyVertexBuffers();
	ApplyHWBufferSet();
	ApplyMaterial();
```

| Step | What it does per draw | Unconditional? | Citation |
|---|---|---|---|
| `ApplyStreamData` | Looks up the vertex format (vector index), computes the shader timer in double precision, then `memcpy`s the whole `StreamData` (384 B, see §3.1) into the next slot of the mapped stream UBO | **Yes**, every draw, with no change detection | vk_renderstate.cpp:337-353, vk_streambuffer.cpp:34-47 |
| `ApplyMatrices` | 2–3 × 64-byte `memcmp` against the cached model (and, when the model matrix is disabled, normal) and texture matrices. On change it computes the normal matrix and copies 192 B into a new aligned block | Compare always; write on change | vk_streambuffer.cpp:81-117 |
| `ApplyRenderPass` | Builds a 16-int `VkPipelineKey` and `memcmp`s it with the current one. On change: `std::map` lookup (`memcmp` ordering) and `vkCmdBindPipeline`. Begins a render pass if none is open | Key build and compare always | vk_renderstate.cpp:217-271, vk_renderpass.h:38-61, vk_renderpass.cpp:235-241 |
| Scissor, viewport, stencil ref, depth bias | Dynamic state, dirty-flagged | No | vk_renderstate.cpp:208-215, 273-335 |
| `ApplyPushConstants` | Fills `PushConstants` (fog mode, texture mode, light params, alpha threshold, clip split, light index, bone base, `uDataIndex`) and calls `vkCmdPushConstants` | **Yes** | vk_renderstate.cpp:355-399 |
| `ApplyVertexBuffers` | Binds vertex buffers (2 bindings) and the index buffer when they change | No | vk_renderstate.cpp:410-429 |
| `ApplyHWBufferSet` | Rebinds set 0 (fixed) and set 1 (HW buffers, **3 dynamic UBO offsets**) when the viewpoint, matrix or stream block offset changes | When the stream block rolls over (every 170 draws), a matrix changes, or the viewpoint changes | vk_renderstate.cpp:448-465 |
| `ApplyMaterial` | When `mMaterial.mChanged`: finds or creates the material's descriptor set, then binds set 0 again plus set 2 (textures) | Whenever `SetMaterial` was called, even with the same material | vk_renderstate.cpp:431-446 |

`SetMaterial` sets `mChanged = true` with no same-material check [U] src/common/rendering/hwrenderer/data/hw_renderstate.h:629-639. Every textured wall therefore pays two `vkCmdBindDescriptorSets` plus a linear search over the material's descriptor-set list [U] src/common/rendering/vulkan/textures/vk_hwtexture.cpp:338-341. The GL backend does filter redundant binds (`lastMaterial`) [U] src/common/rendering/gl/gl_renderstate.cpp:322-326. Vulkan does not.

**Inference (CPU cost):** a typical opaque wall draw records roughly 2–8 `vkCmd*` calls:
- 1 push-constant call (68 B);
- 0–4 descriptor-set binds (set 0 + set 2 from `ApplyMaterial`, set 0 + set 1 from `ApplyHWBufferSet`; a textured wall always pays at least the two material binds);
- 0–1 pipeline bind;
- 0–4 dynamic-state or vertex/index-buffer binds, only when dirty;
- 1 draw;
- one 384 B write to write-combined or ReBAR memory;
- a 64-byte key build plus several `memcmp`s.

That is roughly 0.3–1 µs of main-thread time per draw. With the eye loop (§8) doubling the draw count, a 5k-draw scene becomes 10k recorded draws per frame. That is still affordable in µs terms, but it all happens on one thread, after BSP traversal, also on that thread. The `drawcalls` clock wraps every `Apply` (lines 183 and 205), so `stat` output can measure this directly.

### 2.2 Pipeline lookup and caching

- **Key fields:** render style, special effect, effect state (material shader index), alpha test, depth write/test/func/clamp/bias, stencil test and op, color mask, cull mode, vertex format, draw type (topology) and number of texture layers [U] src/common/rendering/vulkan/renderer/vk_renderpass.h:38-61. `NumTextureLayers` is forced to at least `SHADER_MIN_REQUIRED_TEXTURE_LAYERS` = 11 [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:233-234, [U] src/common/rendering/vulkan/shaders/vk_shader.h:34.
- **Storage:** one `std::map<VkPipelineKey, unique_ptr<VulkanPipeline>>` per render-pass setup. A render-pass setup is keyed by color format, sample count, draw-buffer count (1 normally, 3 in the SSAO G-buffer pass) and depth-stencil presence [U] vk_renderpass.h:63-93, [U] vk_renderpass.cpp:89-95.
- **Creation is lazy, on first use** (`GetPipeline` → `CreatePipeline`) [U] vk_renderpass.cpp:235-338. It uses a `VkPipelineCache` that is loaded from and saved to `<cache>/vulkanpipelinecache` [U] vk_renderpass.cpp:40-81. **Inference:** the first sighting of a new state combination (a new render style, fuzz shader or SSAO toggle) stalls the CPU for a driver compile. The disk cache hides this only on later runs. In VR a hitch like that becomes a dropped or reprojected frame.
- **Topology is baked in:** walls are `TRIANGLE_FAN`, flats are `TRIANGLE_LIST`, decals and sprites are `TRIANGLE_STRIP` [U] vk_renderpass.cpp:300-311. That splits otherwise identical pipelines three ways. Only MoltenVK converts fans to lists [U] vk_renderstate.cpp:583-606.
- **Dynamic state:** viewport, scissor, depth bias and stencil reference only [U] vk_renderpass.cpp:291-294. There is no extended dynamic state, so depth, cull, blend and color-mask changes all mean a pipeline switch.

### 2.3 Descriptor-set layout

| Set | Contents | Binding frequency | Citation |
|---|---|---|---|
| 0 "Fixed" | `ShadowMap` (sampler2D), `LightMap` (sampler2DArray), optional TLAS | Re-bound with every set-1 or set-2 bind | vk_descriptorset.cpp:94-111, 267-276 |
| 1 "HWBuffer" | b0 Viewpoint UBO (dynamic), b1 Matrices UBO (dynamic), b2 StreamUBO (dynamic), b3 light SSBO, b4 bone SSBO | Dynamic offsets per draw when they change | vk_descriptorset.cpp:74-92, 255-265 |
| 2 "Texture" | 11+ `COMBINED_IMAGE_SAMPLER`, one set per (material, clamp, translation) | On every `SetMaterial` | vk_descriptorset.cpp:147-183, vk_hwtexture.cpp:329-386 |

The shader side declares the same layout [U] src/common/rendering/vulkan/shaders/vk_shader.cpp:182-306. Per-draw fields are reached through `data[uDataIndex].field` macros (lines 329-352), where `uDataIndex` is a push constant. This "dynamic UBO block + push-constant index" design is the closest thing the backend has to batching. It amortizes descriptor rebinding over 170 draws, but it still issues one draw per record.

### 2.4 Batching, indirect, bindless

- **No draw merging.** Every draw-list item turns into its own `Draw` or `DrawIndexed` ([U] src/rendering/hwrenderer/scene/hw_drawlist.cpp:840-864 iterates items and calls `DrawWall`/`DrawFlat` one at a time). Texture sorting exists behind `gl_sort_textures`, default **false** [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:45,531-538. It sorts by texture pointer (walls additionally by clamp flags) ([U] src/rendering/hwrenderer/scene/hw_drawlist.cpp:716-743) and does not merge anything.
- **No indirect draws.** ZVulkan wraps `vkCmdDrawIndirect` and `vkCmdDrawIndexedIndirect` [U] libraries/ZVulkan/include/zvulkan/vulkanobjects.h:391-392,780-788. A `grep -rn "drawIndirect\|DrawIndexedIndirect" src` finds no engine caller. ZVulkan even *requires* the `multiDrawIndirect` feature during device selection [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1834-1837, but nothing uses it.
- **No bindless.** ZVulkan has `OptionalDescriptorIndexing()` [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1792-1796, but the engine never calls it (`grep -rn OptionalDescriptorIndexing src` finds nothing). The device builder calls only `OptionalRayQuery()` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:112-121. ZVulkan does copy the descriptor-indexing feature bits into the enabled features when the device reports them [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1864-1868, but no engine shader or layout uses runtime arrays or non-uniform indexing. Textures are bound per material through set 2.

---

## 3. Buffers: vertex data and per-draw data

### 3.1 Sizes and mapping (Vulkan)

| Buffer | Size / capacity | Memory | Lifetime | Citation |
|---|---|---|---|---|
| `StreamData` record | 24 × vec4 = 384 B (see the struct) | — | per draw | hw_renderstate.h:190-221 |
| StreamUBO block | `MAX_STREAM_DATA` = 65536/384 = **170 records** (65,280 B) | — | per 170 draws | vk_shader.h:49-54 |
| `StreamBuffer` | 300 blocks × align(65,280) ≈ 19.6 MB. Block 0 and the last block are never handed out, so 298 usable blocks × 170 → about **50,660 draws per frame** before overflow | Persistent, host-visible + coherent, device-local preferred | per frame | vk_buffer.cpp:36-40,112-118; vk_hwbuffer.cpp:100-116 |
| `MatrixBuffer` | 50,000 blocks × align(192 B) | Persistent, same as above | per frame | vk_buffer.cpp:38; vk_shader.h:42-47 |
| Viewpoint UBO | 100 × align(sizeof `HWViewpointUniforms`); doubles on overflow (`Resize`, which does a **full GPU wait**) | Persistent | per frame | hw_viewpointbuffer.cpp:24-62; vk_hwbuffer.cpp:161-190 |
| Light SSBO | 80,000 lights × 4 vec4 = 5.12 MB, append-only `fetch_add` | Persistent | per frame | hw_lightbuffer.cpp:26-58,75-123 |
| Bone SSBO | 80,000 × mat4 = 5.12 MB | Persistent | per frame | hw_bonebuffer.cpp:24-53 |
| Flat vertex buffer | 2,000,000 × 32 B `FFlatVertex` = 64 MB. Static flat geometry sits in the low region; per-frame walls, sprites and other geometry are appended through atomic `AllocVertices`; fatal error on overflow | Persistent | static part + per-frame tail | flatvertices.h:35-41,100-101; flatvertices.cpp:37-97,138-148 |
| 2D vertex/index buffers | Sized per frame | **New GPU-only buffer + staging buffer per `SetData(Stream)`, every frame** | per frame | v_2ddrawer.h:372-376; hw_draw2d.cpp:84-86; vk_hwbuffer.cpp:73-99 |

Vulkan creates each of these with `pipelineNbr = 1` (`new FFlatVertexBuffer(GetWidth(), GetHeight())`, `new FLightBuffer()`, and so on) [U] vk_renderdevice.cpp:186-190. There is exactly one copy of each. That is safe only because of the end-of-frame full wait (§1).

**Overflow handling:** when the stream or matrix ring runs out, `NextStreamDataBlock` returns `0xffffffff` [U] vk_buffer.cpp:125-134. `Apply` then calls `WaitForStreamBuffers()`, which runs `fb->WaitForCommands(false)`: flush plus a wait on all fences mid-frame. It then rewinds both writers [U] vk_renderstate.cpp:348-352,403-407,467-473. **Inference:** in stereo with heavy mods (50k+ draws across both eyes), this turns into a hidden mid-frame GPU drain.

**Inference:** the stream buffer's first block (offset 0) is never used. The writer starts at index MAX−1 with offset 0, and its first `Write` advances to `NextStreamDataBlock()`, which returns `mBlockSize`; the matrix writer behaves the same way [U] vk_streambuffer.cpp:34-54,81-123, vk_buffer.cpp:125-133. This costs only one 64 KB block.

### 3.2 Upload strategy

- **Persistent buffers** (`BufferUsageType::Persistent`) use a dedicated VMA allocation that is host-visible and coherent, preferring device-local, mapped once [U] vk_hwbuffer.cpp:100-116. `Map()` and `Unmap()` are no-ops for these (lines 192-205). All per-frame data is written by direct `memcpy` into this memory: walls, lights, stream records, matrices and viewpoints. **Inference:** this is write-combined system RAM unless ReBAR or SAM exposes device-local host-visible memory. GPU reads of `StreamData` then cross PCIe, though they are small and cached.
- **Static and Stream buffers** get a GPU-only buffer plus a CPU staging buffer and a `copyBuffer` on the transfer command buffer [U] vk_hwbuffer.cpp:73-99. The comment notes "we could recycle buffers here for the stream usage type", and the code never does. Model meshes and the fan-to-triangles index buffer use this path.
- **Mid-frame light uploads.** Flats call `SetupLights` → `UploadLights` while *drawing* (`HWFlat::DrawSubsectors`) [U] src/rendering/hwrenderer/scene/hw_flats.cpp:192-201. Walls do it during `CreateScene`, between `mLights->Map()` and `Unmap()` [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:491-507. **Inference:** in a two-eye frame every light list is uploaded twice, because the eye loop re-runs both phases.

---

## 4. Textures and samplers

- **Creation and upload:** `VkHardwareTexture::CreateTexture` creates a CPU staging buffer, copies the pixels into it, creates the image (BGRA8 or R8 with a full mip chain), records a transition and `copyBufferToImage`, and generates mips with blits on the **transfer command buffer**. This all happens synchronously on the main thread at the first `GetImage` [U] src/common/rendering/vulkan/textures/vk_hwtexture.cpp:67-74,138-187. When more than 64 MB of staging is queued, it forces `WaitForCommands(false, true)`, an upload-only full wait (lines 183-186). The transfer command buffer is submitted ahead of the draw command buffer in the same `vkQueueSubmit` [U] vk_commandbuffer.cpp:138-157.
  **Inference:** a texture that is first seen mid-game causes a hitch, because there is no streaming thread and no async transfer queue. `PrecacheMaterial` [U] vk_renderdevice.cpp:278-293 exists, but `gl_precache` defaults to false [U] src/common/rendering/hwrenderer/data/hw_cvars.cpp:138.
- **Descriptors per material:** `VkMaterial::GetDescriptorSet` keeps a list of `(clampmode, translation) → descriptor set`. On a miss it allocates a set with at least 11 combined image samplers and writes every layer (unused slots get a null texture) [U] vk_hwtexture.cpp:329-386. Sets come from pools of 1,000 sets and 2,000 descriptors that are created on demand [U] vk_descriptorset.cpp:147-164. Changing the filter mode or starting precache throws away all material sets [U] vk_descriptorset.cpp:113-128, vk_renderdevice.cpp:320-333.
- **Samplers:** a fixed table of about 10 samplers indexed by clamp mode. Filter and anisotropy come from `gl_texture_filter` (default 6) and `gl_texture_filter_anisotropic` (default 16) [U] src/common/rendering/vulkan/textures/vk_samplers.cpp:86-141, [U] hw_cvars.cpp:127,132. Combined image samplers mean the sampler is baked into each material's set.
- **Canvas and camera textures** are rendered through `RenderTextureView`. This ends the render pass, transitions the image, switches the render target, records a full scene pass, then transitions it back [U] vk_renderdevice.cpp:229-260. Camera textures go through `RenderViewpoint(..., mainview=false)` and therefore render once, outside the eye loop [U] hw_entrypoint.cpp:375-386.

---

## 5. Render passes, render targets and the post-process chain

### 5.1 Targets

`VkRenderBuffers` [U] src/common/rendering/vulkan/textures/vk_renderbuffers.cpp:64-200 allocates:
- `SceneColor` RGBA16F, MSAA by `gl_multisample` (default 1) [U] vk_renderbuffers.cpp:42-62,169-183, [U] src/common/rendering/gl/gl_renderbuffers.cpp:38;
- `SceneDepthStencil` D24S8, falling back to D32S8 [U] vk_renderbuffers.h:58, vk_renderbuffers.cpp:185-196;
- `SceneFog` RGBA8 and `SceneNormal` A2R10G10B10, used only in the SSAO G-buffer pass [U] vk_renderbuffers.h:59, vk_renderpass.cpp:180;
- two `PipelineImage` RGBA16F ping-pong images plus a pipeline depth-stencil [U] vk_renderbuffers.cpp:118-147, vk_renderbuffers.h:61-63.

Every one of these is **a single 2D image at screen size**. There are no array layers and no per-eye images (layerCount is always 1; see the blits in [U] vk_postprocess.cpp:127-163).

### 5.2 Scene render pass

- `BeginRenderPass` picks a `VkRenderPassSetup` by key, lazily builds and caches a `VkFramebuffer` on the target image, and begins one of 8 render-pass variants indexed by clear mask [U] vk_renderstate.cpp:533-579, vk_renderpass.cpp:176-233.
- Load and store ops are LOAD/STORE unless a clear is requested. `Set3DViewport` clears color, depth and stencil at the start of every scene, and so at the start of **every eye** [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:965-980.
- The render pass is broken (ended and restarted with LOAD) by each of these:
  - `SetRenderTarget`;
  - `Clear`;
  - `EnableDrawBuffers` changes;
  - every post-process `Draw`;
  - every mid-frame flush.

  Sources: [U] vk_renderstate.cpp:135-139,512-531, vk_pprenderstate.cpp:53-55, vk_commandbuffer.cpp:133-136.

  **Inference:** on desktop GPUs a restart is cheap. It does block tile-friendly or MSAA-resolve-in-pass designs, which matters for standalone headsets and much less for PC VR.

### 5.3 Post-process chain (per eye; see §8)

`RenderViewpoint` calls `screen->PostProcessScene(...)` inside the eye loop [U] hw_entrypoint.cpp:168-181. In Vulkan this does the following:

1. `BlitSceneToPostprocess`: a full-resolution `vkCmdBlitImage` from SceneColor to PipelineImage[0], or an MSAA resolve [U] vk_postprocess.cpp:110-165.
2. `Pass1`: exposure and bloom, only if `gl_bloom` (default **false**) [U] hw_postprocess.cpp:1196-1201,432-437,76-79, [U] hw_postprocess_cvars.cpp:31.
3. `DrawEndScene2D` (the weapon sprite) into the pipeline image.
4. `Pass2`: tonemap (only if `gl_tonemap` ≠ 0, default 0, or the level's MAPINFO sets a tonemap), colormap (only for special colormaps or flash ≠ 1), lens (`gl_lens`, default false), FXAA (`gl_fxaa`, default 0), then custom "scene" shaders [U] hw_postprocess.cpp:1203-1213,601-608,535-542,303-305,357-362.

SSAO (`gl_ssao`, default 0) runs *inside* `DrawScene` after the opaque pass, and so per eye [U] hw_drawinfo.cpp:1034-1038, hw_postprocess.cpp:721-726.

Every PP `Draw` does all of the following [U] src/common/rendering/vulkan/renderer/vk_pprenderstate.cpp:53-98,110-149, vk_descriptorset.cpp:197-231:
- ends the scene render pass;
- allocates a **fresh descriptor set** from a 100-set pool through `GetInput`, and frees it after the frame;
- emits image barriers;
- begins a dedicated render pass;
- binds pipeline, set, vertex buffer, viewport and scissor;
- pushes the uniforms as push constants;
- draws a 3-vertex full-screen triangle.

At the end of the frame, `AcquireImage` → `DrawPresentTexture` runs the "present" shader into the swapchain image. That shader handles gamma, contrast, saturation, dithering and the HDR flag [U] vk_postprocess.cpp:249-309.

**Inference (VR resolution):** take about 2016×2240 per eye (4.5 MP) at RGBA16F (8 B/px). With default cvars that costs, per eye:
- one 36 MB blit (read and write);
- a full-screen clear of color, depth and stencil;
- the scene itself.

Then there is one present pass. Bloom adds a few more full-resolution and downsampled passes, and SSAO adds linear depth, AO, two blurs and a combine at full or half resolution. With defaults the PP overhead is small, around 0.2–0.4 ms per eye on a modern GPU. Enabling bloom, SSAO and FXAA in VR can cost several ms, doubled by the eye loop. The final present runs at swapchain size, which for an OpenXR swapchain would be per eye.

---

## 6. Command buffers, synchronization, swapchain and pacing

- **One graphics queue, one command pool, one draw command buffer and one transfer command buffer**, created lazily [U] vk_commandbuffer.cpp:40-90. Command buffers are never reused; each flush hands them to a delete list [U] vk_commandbuffer.cpp:143-155.
- **Mid-frame flushes:** every `vk_submit_size` applies (default **1000**) [U] vk_renderstate.cpp:42,186-190, `FlushCommands(false)` ends the render pass and submits. Each submit waits on the previous submit's semaphore at `TOP_OF_PIPE` and signals its own [U] vk_commandbuffer.cpp:111-127. The ring of 8 fences and semaphores (`maxConcurrentSubmitCount = 8`) [U] vk_commandbuffer.h:93-96 makes the 9th submit in a frame wait on the oldest fence (vk_commandbuffer.cpp:105-109). **Inference:** this lets the GPU start the early part of a frame while the CPU records the rest. It is the only CPU/GPU overlap in the design. Each flush costs a render-pass restart and a queue submit.
- **End of frame** (§1): acquire with a 1 s timeout [U] libraries/ZVulkan/src/vulkanswapchain.cpp:224-249, a final submit that waits on `SwapChainImageAvailableSemaphore` at `COLOR_ATTACHMENT_OUTPUT` and signals `RenderFinishedSemaphores[i]`, present, then **wait on all fences**.
- **Swapchain:** 2 images with vsync, 3 without [U] vk_framebuffer.cpp:61. Present mode is `FIFO_RELAXED` (or FIFO) with vsync. Without vsync it is `IMMEDIATE`; mailbox is commented out except in the exclusive-fullscreen fallback [U] libraries/ZVulkan/src/vulkanswapchain.cpp:108-134. `vid_vsync` defaults to false [U] src/common/rendering/v_video.cpp:185. Optional HDR (`vk_hdr`) and exclusive fullscreen (`vk_exclusivefullscreen`) are both off by default [U] vk_framebuffer.cpp:31-32.
- **Frame pacing:** without vsync, a sleep-then-yield limiter to `vid_maxfps` (default **500**) runs right before present; `cl_capfps` (default false) disables it [U] src/common/rendering/v_framebuffer.cpp:243-275,51, [U] v_video.cpp:71-82. There is no `VK_KHR_present_wait`/`present_id`, no latency markers and no predicted display time (`grep -rn "present_wait\|presentId\|PresentId" src libraries/ZVulkan/src` finds nothing in the engine). **Inference:** the game tic interpolation (`TicFrac`, hw_entrypoint.cpp:343-344) samples time when RenderView starts. Nothing predicts when photons will actually appear, which is exactly what VR pose prediction needs.
- **GPU timing:** timestamp queries per `PushGroup`/`PopGroup` when gpu stats are on. Results are read back with `VK_QUERY_RESULT_WAIT_BIT` right after the full wait [U] vk_commandbuffer.cpp:208-266.

---

## 7. Shaders: the uber-shader and its variants

### 7.1 Variant handling

- **Variants compiled:** one program per `defaultshaders` entry (15 entries: Default, Warp1/2, Specular, PBR, Paletted, NoTexture and 8 fuzz styles) [U] src/common/rendering/hwrenderer/data/hw_shaderpatcher.cpp:281-299. On top of that:
  - "NAT" (no-alpha-test) versions for indices 0–5;
  - user shaders;
  - 5 effect shaders (fogboundary, spheremap, burn, stencil, dithertrans) (lines 301-308).

  Each of these exists for both the NORMAL and GBUFFER passes [U] src/common/rendering/vulkan/shaders/vk_shader.cpp:59-140. They are GLSL compiled to SPIR-V by the bundled glslang at startup, in a blocking loop [U] src/d_main.cpp:3859-3862.
- **Per-draw selection:** `pipelineKey.EffectState` is the material's shader index, and `AlphaTest` picks NAT or alpha-test [U] vk_renderstate.cpp:235-249, vk_shader.cpp:166-180.
- **Runtime branches instead of variants:** `#define` flags are only compile-time for material type, alpha test, G-buffer and ray tracing [U] vk_shader.cpp:393-411. Everything else is a branch on push constants or `StreamData`:
  - fog mode (`uFogEnabled`: 0, ±1, ±2, −3);
  - light-mode equation (`uLightLevel` ≥ 0 selects the software-light path);
  - texture mode (`switch (uTextureMode & 0xffff)`);
  - brightmap, detail and glow layers (`uTextureMode & TEXF_*`);
  - glow and gradient planes;
  - texture manipulation (`uTextureAddColor.a`);
  - dynamic lights (`uLightIndex`).

  Sources: [U] wadsrc/static/shaders/glsl/main.fp:196-262,671-697,714-780,833-944.

  **Inference:** these branches are uniform within a draw, so divergence is low. Every fragment of every surface still carries the full code size and register pressure of the uber-shader, including the dynamic-light loops.

### 7.2 Per-fragment work in `main.fp` (Default material)

1. A `getTexel` texture fetch, the texture-mode switch, then ClampY, object color, add color and desaturate handling (main.fp:196-262).
2. Optional brightmap, detail and glow fetches (main.fp:685-697).
3. Alpha-test `discard` in non-NAT variants (main.fp:858-860). **Inference:** this disables early-Z for masked geometry.
4. Fog distance (`pixelpos.w` or a `distance()`), thick-fog adjustment and `exp2` (main.fp:862-888).
5. `getLightColor`: the Doom light equation or fog-based diminishing, glow planes, glowmap and brightmap, then `ProcessLight`, a lightmap fetch, then `ProcessMaterialLight` (main.fp:714-780).
6. **Dynamic lights** (`material_normal.fp`): three loops over this surface's light list (modulated, subtractive and additive), 4 vec4 SSBO reads per light, distance, normalize and dot, an optional spot-cone term and an optional shadow test [U] wadsrc/static/shaders/glsl/material_normal.fp:18-112. The lists are per surface and built on the CPU, so this is not clustered lighting.
7. **Shadow test per light, when shadow-mapped:** a 1D shadow-map lookup with PCF of `2*gl_shadowmap_filter+1` taps [U] main.fp:526-590. With `vk_raytrace` on and supported, it is instead `1 + 9*filter` ray queries per light per fragment [U] main.fp:380-453.
8. With `DITHERTRANS`: rgb2hsv and an ordered-dither discard (main.fp:920-938).
9. G-buffer pass: two extra MRT outputs, including a second fog-distance evaluation (main.fp:799-825,940-943).

The vertex shader always writes 5 clip distances (main.vp:157-166). It uses a single `ProjectionMatrix * ViewMatrix` [U] wadsrc/static/shaders/glsl/main.vp:72,128. There is no `gl_ViewIndex`.

**Inference (VR GPU cost):** with no dynamic lights the Default path is cheap: one or two texture fetches plus ALU, fine at 2× 4.5 MP at 90 Hz on modern GPUs. Dynamic-light-heavy mods are the risk. Every lit fragment loops over every light touching its surface, all at full per-eye resolution, with no early-Z for alpha-tested sprites. Shadowmaps (`gl_light_shadowmap`, default off) multiply that by the PCF taps. Ray-query shadows multiply it by up to 10 rays per light.

---

## 8. Stereo: `hw_vrmodes` and the per-eye loop

### 8.1 Modes

`vr_mode` (default 0) [U] src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:33 maps to eight `VRMode` tables (mono, stereo, sbsfull, sbssquished, lefteye, righteye, topbottom, checker) [U] hw_vrmodes.cpp:49-56,68-105, [U] hw_vrmodes.h:31-47:

| vr_mode | Name | Table | Eyes | Viewport scale H×V | Composition (GL only) |
|---|---|---|---|---|---|
| 0 | Mono | mono | 1 | 1×1 | — |
| 1 / 2 / 9 | Green-magenta / red-cyan / amber-blue anaglyph | stereo | 2 | 1×1 | `PresentAnaglyph` |
| 3 | Side-by-side full | sbsfull | 2 | 0.5×1 (weapon proj ×2) | `PresentSideBySide` |
| 4 | Side-by-side squished | sbssquished | 2 | 0.5×1 | `PresentSideBySide` |
| 5 / 6 | Left / right eye only | lefteye / righteye | 1 | 1×1 | — |
| 7 | Quad-buffered | stereo | 2 | 1×1 | `PresentQuadStereo` |
| 8 | SBS letterbox | stereo | 2 | 1×1 | `PresentSideBySide` |
| 11 | Top-bottom | topbottom | 2 | 1×0.5 | `PresentTopBottom` |
| 12 / 13 / 14 | Row / column / checker interleaved | topbottom / sbssquished / checker | 2 | various | interleave shaders |

Per-eye math: `GetProjection` builds an **asymmetric frustum** shifted by `zNear * (±0.5*vr_ipd) / vr_screendist` [U] hw_vrmodes.cpp:137-184. `GetViewShift` offsets the camera sideways by `vr_hunits_per_meter * shift` along the yaw-perpendicular axis, ignoring pitch and roll [U] hw_vrmodes.cpp:189-202. Defaults are `vr_ipd` 0.062 m, `vr_screendist` 0.80 m and `vr_hunits_per_meter` 41 (lines 36-45). This is a desktop 3D-TV model: no HMD poses, no per-eye FOV tangents and no head orientation.

### 8.2 What runs per eye

```cpp
// [U] src/rendering/hwrenderer/hw_entrypoint.cpp:129-166 (abridged)
	for (int eye_ix = 0; eye_ix < eyeCount; ++eye_ix)
	{
		const auto& eye = vrmode->mEyes[eye_ix];
		screen->SetViewportRects(bounds);
		if (mainview) { screen->SetSceneRenderTarget(useSSAO); ... }
		auto di = HWDrawInfo::StartDrawInfo(mainvp.ViewLevel, nullptr, mainvp, nullptr);
		di->Set3DViewport(RenderState);
		...
		di->VPUniforms.mProjectionMatrix = eye.GetProjection(fov, ratio, fovratio * inv_iso_dist, iso_ortho);
		vp.Pos += eye.GetViewShift(vp.HWAngles.Yaw.Degrees());
		di->SetupView(RenderState, vp.Pos.X, vp.Pos.Y, vp.Pos.Z, false, false);
		di->ProcessScene(toscreen);
```

For each eye this sequence runs:
- a new `HWDrawInfo`;
- clipper setup;
- **full BSP traversal and draw-list build** (`CreateScene` → `RenderBSP`), which generates wall and sprite vertices and light lists [U] hw_drawinfo.cpp:470-511;
- the sky portal;
- **command recording** for opaque geometry, portals (recursively, each portal re-traversing its own sub-scene) and translucent geometry [U] hw_drawinfo.cpp:991-1046;
- SSAO;
- `EndDrawScene` and the HUD weapon;
- **post-processing** (hw_entrypoint.cpp:168-181).

Only the shadow-map update, light-buffer clear, vertex reset and camera textures are hoisted out of the loop (lines 103-118 and 314-386). Each eye uploads its own viewpoint UBO entry (`SetupView` → `mViewpoints->SetViewpoint`) [U] hw_drawinfo.cpp:425-432.

**Inference:** stereo CPU cost is close to 2.0× mono for the whole renderer, and GPU cost is 2× geometry plus 2× PP. Only the eye position differs, by about 2.5 map units, so the visible set is nearly identical and the second traversal is almost entirely redundant work.

### 8.3 Vulkan backend: stereo is effectively non-functional

- `DFrameBuffer::FirstEye()` and `NextEye()` are empty virtuals [U] src/common/rendering/v_video.h:258-259. `VulkanRenderDevice` does not override them (the only overrides are in the GL and GLES framebuffers), so `RenderViewpoint`'s `screen->NextEye(eyeCount)` call is a no-op [U] src/rendering/hwrenderer/hw_entrypoint.cpp:186-187. `VkPostprocess::NextEye` exists but is empty [U] vk_postprocess.cpp:358-360.
- Only GL and GLES call `VRMode::AdjustViewport` in `SetViewportRects` [U] src/common/rendering/gl/gl_framebuffer.cpp:377-385. The only callers are found by `grep -rn AdjustViewport src`: gl_framebuffer.cpp:383 and gles_framebuffer.cpp:316.
- No stereo presentation exists in `src/common/rendering/vulkan`. `grep -rn "PresentStereo\|PresentSideBySide\|Row3D" src/common/rendering/vulkan` finds nothing. `PPPresent` declares `Checker3D`/`Column3D`/`Row3D` PP shaders [U] src/common/rendering/hwrenderer/postprocessing/hw_postprocess.h:791-793, but nothing references them. GL loads the same `present_*3d.fp` files through its own shader objects [U] src/common/rendering/gl/gl_shaderprogram.cpp:325-343 and uses them only in gl_stereo3d.cpp.
- Each eye clears and redraws the same `SceneColor` (hw_drawinfo.cpp:970-971) and re-blits to `PipelineImage[0]` (vk_postprocess.cpp:117).

**Inference:** in the Vulkan backend, any `vr_mode` with 2 eyes costs about 2× and shows only the last eye, at full viewport, with an off-axis projection. (Unrelated to the backend, `d_main` suppresses screen wipes whenever `vr_mode` ≠ 0 and the hardware renderer is active, `vid_rendermode` 4 [U] src/d_main.cpp:1181-1184.)

### 8.4 OpenGL contrast for stereo

- GL keeps `mEyeFBs` and `mEyeTextures`. `NextEye` does **two full-resolution `glBlitFramebuffer`s** (pipeline → eye N, eye N+1 → pipeline) [U] src/common/rendering/gl/gl_renderbuffers.cpp:534-568,1016-1024.
- `Flush` draws the 2D layer *once per eye* and then calls `PresentStereo`, which composes into the back buffer with anaglyph, side-by-side, top-bottom, interleaved or quad-buffer output [U] src/common/rendering/gl/gl_postprocess.cpp:118-144, [U] src/common/rendering/gl/gl_stereo3d.cpp:398-449.
- `vr_enable_quadbuffered` (Windows GL) enables `GL_BACK_LEFT/RIGHT` [U] src/common/platform/win32/win32glvideo.cpp:67-68.

- GLES overrides `FirstEye`/`NextEye`, but both bodies are commented out [U] src/common/rendering/gles/gles_framebuffer.cpp:327-335.

So the only working stereo path in 5.0 is desktop GL, and it is still a 2× serial re-render with extra blits.

### 8.5 Multiview, OpenXR and OpenVR: proven absent

- `grep -rni "multiview" src wadsrc` → only `WGL_GENLOCK_SOURCE_MULTIVIEW_I3D` in src/common/platform/win32/wglext.h:537. ZVulkan's device builder enables no multiview feature: the enabled-feature list in [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1853-1868 has no `multiview` entry. Render passes have a single subpass with no view mask [U] vk_renderpass.cpp:176-226. The vertex shader has no `gl_ViewIndex`.
- `grep -rniE "openxr|openvr|xrCreate|xrInstance|vr::|IVRSystem" .` (whole repo, excluding .git) → **no matches**. There is no headset runtime, no pose input, no XR swapchain and no compositor submission.

---

## 9. OpenGL backend: points of contrast

- **Per-draw uniforms:** GL uses classic `glUniform*` calls with CPU-side redundancy filtering (`FBufferedUniform4f` `memcmp`s before `glUniform4fv`) [U] src/common/rendering/gl/gl_shader.h:152-172. `ApplyShader` sets about 25 uniforms per draw [U] src/common/rendering/gl/gl_renderstate.cpp:81-160. Vulkan replaced this with one 384 B UBO record and a 68 B push constant, which is fewer API calls but **always copies the full record** with no filtering.
- **Material binding:** GL skips the rebind for the same material, clamp and translation (gl_renderstate.cpp:322-326). Vulkan always rebinds (§2.1).
- **Sync:** GL with `gl_pipeline_depth` 0 (default) calls `glFinish()` after `SwapBuffers` every frame [U] src/common/rendering/gl/gl_framebuffer.cpp:246-273, [U] v_video.cpp:56-69. With `gl_pipeline_depth` ≥ 1, GL uses N-buffered vertex, light and viewpoint buffers with `glFenceSync`/`glClientWaitSync` [U] gl_framebuffer.cpp:262-268, gl_buffers.cpp:171-190. That is real frames-in-flight. The Vulkan backend has no equivalent; it passes `pipelineNbr = 1` (vk_renderdevice.cpp:186-190).
- **Default backend** is Vulkan on non-Apple builds with Vulkan [U] src/common/rendering/v_video.h:68-80.

---

## Absent / not done (each proven by search)

| Feature | Search | Result |
|---|---|---|
| OpenXR / OpenVR / any HMD runtime | `grep -rniE "openxr\|openvr\|xrCreate\|xrInstance\|vr::\|IVRSystem" .` | no matches |
| Vulkan multiview / `gl_ViewIndex` | `grep -rni multiview src wadsrc`; `grep -n gl_ViewIndex wadsrc/static/shaders/glsl/main.vp` | only a WGL constant; no `gl_ViewIndex` |
| Indirect / multi-draw | `grep -rn "drawIndirect\|DrawIndexedIndirect" src` | none in the engine (ZVulkan wrapper only) |
| Bindless / descriptor indexing | `grep -rn OptionalDescriptorIndexing src` | none |
| Draw merging / instancing | `vkCmdDraw` calls: `draw(count, 1, ...)` with instanceCount = 1 everywhere (vk_renderstate.cpp:63,71) | none |
| Frames in flight (Vulkan) | `WaitForCommands(true)` waits on all fences each frame (vk_commandbuffer.cpp:182-188); buffers created with `pipelineNbr` 1 | none |
| Async transfer / texture streaming thread | all uploads use `GetTransferCommands()` on the graphics queue (vk_hwtexture.cpp:167; vk_commandbuffer.cpp:70-79) | none |
| Present timing / latency APIs | `grep -rn "present_wait\|presentId\|PresentId\|LowLatency" src` | none |
| Stereo composition in Vulkan | `grep -rn "PresentStereo\|SideBySide" src/common/rendering/vulkan` | none; `NextEye` is empty |
| Extended dynamic state / dynamic rendering | `grep -rn "dynamicRendering\|EXTENDED_DYNAMIC\|vkCmdBeginRendering" src` | none |
| Foveated / variable-rate shading | `grep -rni "fragment_shading_rate\|ShadingRate\|foveat" src libraries/ZVulkan/src/vulkanbuilders.cpp` | none |

---

## Perf-relevant settings

| CVar | Default | Effect | Citation |
|---|---|---|---|
| `vid_preferbackend` | Vulkan (non-Apple, if built) | backend | v_video.cpp:85; v_video.h:68-80 |
| `vk_submit_size` | 1000 | applies between mid-frame submits | vk_renderstate.cpp:42 |
| `vk_raytrace` | false | ray-query shadows (rays per light per fragment) | vk_renderdevice.cpp:63 |
| `vk_hdr` | false | scRGB swapchain | vk_framebuffer.cpp:31 |
| `vk_exclusivefullscreen` | false | exclusive fullscreen, immediate present | vk_framebuffer.cpp:32 |
| `vk_device` / `vk_debug` | 0 / false | device select / validation layer | vk_renderdevice.cpp:69-79 |
| `vid_vsync` | false | FIFO_RELAXED vs IMMEDIATE, 2 vs 3 images | v_video.cpp:185; vk_framebuffer.cpp:61 |
| `vid_maxfps` | 500 | CPU frame limiter (no vsync) | v_video.cpp:71 |
| `cl_capfps` | false | disables limiter and interpolation | v_framebuffer.cpp:51 |
| `vid_scalefactor` / `vid_scalemode` | 1.0 / 0 | internal render resolution | r_videoscale.cpp:171,180 |
| `gl_multisample` | 1 | MSAA on scene buffers, resolve per eye | gl_renderbuffers.cpp:38; vk_renderbuffers.cpp:42-62 |
| `gl_ssao` / `gl_ssao_portals` | 0 / 1 | G-buffer pass (3 MRTs) + AO chain per eye and per portal | hw_postprocess_cvars.cpp:62,68 |
| `gl_bloom` | false | exposure + bloom chain | hw_postprocess_cvars.cpp:31 |
| `gl_tonemap` | 0 | tonemap pass | hw_postprocess_cvars.cpp:42 |
| `gl_fxaa` | 0 | FXAA pass | hw_postprocess_cvars.cpp:54 |
| `gl_lens` | false | lens distortion pass | hw_postprocess_cvars.cpp:48 |
| `gl_light_shadowmap` | false | 1D shadow map + per-light PCF | hw_shadowmap.cpp:57 |
| `gl_shadowmap_quality` | 1024 | shadow map width (×1024 R32F) | hw_shadowmap.cpp:66; vk_texture.cpp:166-167 |
| `gl_shadowmap_filter` | 1 | PCF taps = 2n+1 (or 9n+1 rays) | hw_cvars.cpp:141 |
| `r_dynlights` | true | dynamic-light loops in shader | g_cvars.cpp:64 |
| `gl_light_sprites` / `gl_light_particles` / `gl_light_models` | true / true / true | more lit draws | hw_dynlightdata.cpp:35-36; hw_models.cpp:36 |
| `gl_texture_filter` / `gl_texture_filter_anisotropic` | 6 / 16 | sampler state | hw_cvars.cpp:127,132 |
| `gl_texture_hqresizemode` / `_mult` | 0 / 1 | CPU upscaling at upload | hqresize.cpp:42,52 |
| `gl_precache` | false | upfront texture upload | hw_cvars.cpp:138 |
| `gl_sort_textures` | false | sort draw lists by texture | hw_drawinfo.cpp:45 |
| `gl_multithread` | true | threaded BSP traversal (scene side) | hw_bsp.cpp:43 |
| `gl_fogmode` / `gl_lightmode` | 2 / 1 | shader fog/light branch | hw_cvars.cpp:32; g_level.cpp:146 |
| `r_skipmats` | false | demote spec/PBR materials to default | v_video.cpp:54; vk_renderstate.cpp:246-247 |
| `gl_pipeline_depth` | 0 | GL-only buffer pipelining | v_video.cpp:57; gl_framebuffer.cpp:125-126 |
| `gl_finishbeforeswap` | false | GL-only | gl_framebuffer.cpp:246 |
| `vr_mode`, `vr_ipd`, `vr_screendist`, `vr_hunits_per_meter`, `vr_swap_eyes` | 0, 0.062, 0.80, 41, false | stereo | hw_vrmodes.cpp:33-45 |

---

## Transfer notes for UZDoom VR

Each row names the mechanism, its UZDoom 5.0 location, a grade, and what VR or Vulkan changes.

1. **Frame sync (0 frames in flight)**: `VkCommandBufferManager::WaitForCommands(true)` (vk_commandbuffer.cpp:163-199). **Rework.** Introduce N=2–3 frame contexts, each with its own command pool, fence, stream/matrix/viewpoint/light/bone/vertex buffer region and delete list. The `FRenderState` interface survives. All persistent per-frame buffers in §3.1 must be ring-buffered, as GL already does with `gl_pipeline_depth`. Under OpenXR the end-of-frame wait is replaced by `xrWaitFrame`/`xrBeginFrame`/`xrEndFrame`, and the fence wait moves to "before reusing frame context k".
2. **Presentation**: `VkFramebufferManager` / `VulkanSwapChain` (vk_framebuffer.cpp:48-83). **Rebuild for VR.** Add an `XrSwapchain` path: either one 2-layer array swapchain or two swapchains, with acquire, wait and release around the per-eye render, and `xrEndFrame` with a projection layer that carries the poses and FOVs used to render. Keep the desktop swapchain only for an optional mirror window. Feed the present pass (`DrawPresentTexture`, vk_postprocess.cpp:249-309) per eye into the XR image (gamma; dithering optional).
3. **Eye loop**: `RenderViewpoint` (hw_entrypoint.cpp:126-188). **Rework, then Rebuild.**
   - Step 1 (Port, about days): give Vulkan real per-eye targets. Either SceneColor/Pipeline as 2-layer arrays or two sets, plus a non-empty `NextEye`. This gets correct stereo at 2× cost.
   - Step 2 (Rework): run `CreateScene` once with a frustum that is the union of both eyes (or the head-centered view widened by the IPD), then record once for both eyes. Traversal and draw lists are view-independent apart from the clip angles.
   - Step 3: multiview (`VK_KHR_multiview`, core in 1.1, and the engine already requests 1.2, vulkanbuilders.cpp:1688). Render passes get viewMask 0b11, the viewpoint UBO becomes `ViewMatrix[2]`/`ProjectionMatrix[2]` indexed by `gl_ViewIndex` in main.vp:72,128, and every per-draw command is recorded once. That halves CPU submission and vertex work.

   Portals use stencil and depth-range tricks per view (hw_portal.cpp). They need checking under multiview because the stencil is shared per layer, which works, but the portal clip planes are per view.
4. **Per-eye projection**: `VREyeInfo::GetProjection` / `GetViewShift` (hw_vrmodes.cpp:137-202). **Port.** Replace the screen-distance model with the asymmetric FOV tangents from `xrLocateViews` and a full 6-DoF eye pose: yaw, pitch and roll plus head position mapped to map units. The weapon and HUD projection (`GetHUDSpriteProjection`, hw_vrmodes.cpp:120-129) has to become a world-anchored quad or layer.
5. **Per-draw Apply**: `VkRenderState::Apply` (vk_renderstate.cpp:181-206). **Rework.**
   - Cheap wins (Port): change-detect `StreamData` (hash or dirty bits from the setters), skip redundant `SetMaterial` rebinds as GL does, and avoid re-binding set 0 on every set change.
   - Bigger win (Rework): record draws into a per-frame packet array keyed by pipeline, then emit them sorted, and `vkCmdDrawIndexedIndirect` runs per pipeline. That needs walls converted from fans to an index list (the Molten path already does this with `FanToTrisIndexBuffer`, vk_buffer.cpp:96-108) and textures moved to a bindless array indexed from `StreamData`.

   The `StreamData` + `uDataIndex` layout is already halfway to an SSBO-indexed draw-data model.
6. **Descriptor model**: `VkDescriptorSetManager` and `VkMaterial::GetDescriptorSet` (vk_descriptorset.cpp; vk_hwtexture.cpp:329-386). **Rework.** Use descriptor indexing (ZVulkan already has `OptionalDescriptorIndexing`) with one global texture array plus a sampler array, and a material ID in `StreamData`. That removes set-2 binds and the 11-slot null padding.
7. **Stream/matrix buffers**: `VkStreamBuffer` (vk_buffer.cpp:112-134). **Port.** Multiply by frames in flight. Stereo doubles consumption unless multiview is used, so check the 50k-draw ceiling against the mid-frame stall (vk_renderstate.cpp:467-473). Prefer a device-local + host-visible (ReBAR) heap; that is already the preferred memory type (vk_hwbuffer.cpp:106-108).
8. **Pipelines**: `VkRenderPassSetup::GetPipeline` (vk_renderpass.cpp:235-338). **Port.** Pre-warm all combinations at load time, from a recorded key list or by enumerating material × style × pass. That avoids first-use compile hitches, which are fatal in VR. Multiview needs new render passes with a view mask; the keys are otherwise unchanged.
9. **Texture upload**: `VkHardwareTexture::CreateTexture` (vk_hwtexture.cpp:138-187). **Rework.** Move CPU decode, hq-resize and staging onto a worker thread plus a transfer queue. Force `gl_precache`-style preloading at map start. Keep mip generation on the GPU.
10. **Post-process**: `VkPostprocess` / `VkPPRenderState` (vk_postprocess.cpp, vk_pprenderstate.cpp). **Rework.**
    - Make every PP pass operate on 2-layer arrays, or run each pass once per layer.
    - Cache PP descriptor sets instead of allocating one per draw (vk_descriptorset.cpp:197-231).
    - Drop the scene→pipeline blit by rendering straight into the pipeline image when MSAA is off.
    - Default VR config should keep bloom, SSAO, lens and FXAA off, or run them at reduced resolution. Lens distortion is irrelevant, since the compositor does it.
11. **Shader**: `main.fp` / `material_*.fp`. **Port for multiview** (`gl_ViewIndex` in the vertex shader only). For cost, consider specialization constants per fog mode and per dynamic-lights on/off, to cut uber-shader register pressure. Keep alpha-test as separate pipelines (already the case) so opaque walls keep early-Z.
12. **Render buffers**: `VkRenderBuffers` (vk_renderbuffers.cpp). **Port.** Size buffers to the XR recommended per-eye resolution × `vid_scalefactor`. Use array layers for multiview. MSAA can resolve inside the render pass (a resolve attachment) instead of `vkCmdResolveImage` (vk_postprocess.cpp:124-143).
13. **GL stereo path** (gl_stereo3d.cpp, gl_renderbuffers.cpp:534-568). **Not worth porting.** It is useful only as a reference for the composition modes on a desktop mirror.

---

## Open questions

- **Measured draw counts per frame** on representative maps, with `stat` / `drawcalls` and `rendered_commandbuffers`, were not measured. The CPU per-draw estimates above are inference.
- **Actual `minUniformBufferOffsetAlignment` and the ReBAR availability** on target GPUs, which determine the stream and matrix block footprint and whether persistent buffers are VRAM or sysmem. Code reads the value at init (vk_renderdevice.cpp:166-167) but behaviour depends on hardware.
- **Whether `gl_multithread` BSP traversal** (hw_bsp.cpp:1008-1045) overlaps meaningfully with command recording or only with itself. That belongs to the scene section and was not traced here.
- **Exact `sizeof(PushConstants)`.** The C++ struct has `padding1..3`, giving 68 bytes by count (vk_shader.h:56-80). The GLSL block declares only `padding2, padding3`, 64 bytes (vk_shader.cpp:282-306). The push uses `sizeof(PushConstants)` (vk_renderstate.cpp:398). Verifier check: every C++ member is 4-byte aligned (`FVector2` is two floats), so there is no implicit padding and the size is 68 B; the C++ and GLSL member offsets agree through `uDataIndex` (offset 52), so the 4-byte mismatch is only trailing padding and harmless (≤128 B).
- **Whether Vulkan `vr_mode` actually shows the right eye.** This was not run. The conclusion that the last eye overwrites the first comes from code reading (clear per eye at hw_drawinfo.cpp:970-971 and the empty `NextEye`).
- **Portal stencil behaviour under multiview** (hw_portal.cpp) was not analysed.
