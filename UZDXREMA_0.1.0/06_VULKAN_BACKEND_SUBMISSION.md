# 06 — Vulkan backend and draw submission (teardown)

Tree of record: **main** `/home/claude/uzdx_main` (f182cbcc). All `path:line` citations are in main unless prefixed
`renderer:`, `netplay:` or `selaco:`. **V** = VERIFIED (code read), **I** = INFERRED (deduction; basis stated).
Struct sizes marked "(compiled)" were obtained by compiling the real headers with g++ (x86-64) in a scratch dir and
reading `sizeof` from a template error; they are layout facts of the C++ structs, not guesses.

---

## 1. Summary

1. The backend is single-queue, single-frame: every displayed frame ends in `VkCommandBufferManager::WaitForCommands`, which submits and then **blocks the CPU until every submit of the frame has finished** (`vk_commandbuffer.cpp:430-513`, called from `vk_renderdevice.cpp:735/739`). CPU frame N+1 never overlaps GPU frame N. **V**
2. All per-frame CPU→GPU buffers (stream data, matrices, viewpoints, lights, bones, flat vertices, compute host lists) are **single-buffered persistent mappings** (`pipelineNbr = 1` on Vulkan, `vk_renderdevice.cpp:607-611`; `hw_viewpointbuffer.h:45`, `flatvertices.h:115`, `hw_lightbuffer.h:49`, `hw_bonebuffer.h:44`). The full per-frame sync in (1) is what makes this correct; removing it requires ring-buffering all of them. **V**
3. In VR the frame has **two or three full CPU waits**: the main wait, then a separate blit-to-XR-swapchain submit with its own fence wait (`vk_openxrdevice.cpp:6260-6379`), then (sync mode 1, default) a desktop-mirror submit + present + wait (`vk_renderdevice.cpp:743-750`). **V**
4. Per draw, `VkRenderState::Apply` (`vk_renderstate.cpp:223-248`) memcpy's a **2000-byte `StreamData`** (compiled) into a 64 KB UBO block that holds only **32 draws** (`MAX_STREAM_DATA = 32`, compiled), always issues `vkCmdPushConstants` (64 B), and rebinds descriptor sets 0+1 every 32 draws and sets 0+2 on **every `SetMaterial` call** (no same-material check, `hw_renderstate.h:1095-1105`). Minimum 2 vkCmd calls per draw, typically 4, up to 13 (+1 vkCmdBeginRenderPass) [xc: max is 12 from Apply + the draw = 13, not ~12]. **V**
5. Hard per-frame capacity walls cause **forced mid-frame full GPU syncs**: 298 stream blocks × 32 = ~9,536 Applies (`vk_buffer.cpp:39,131`, `vk_renderstate.cpp:390-394,514-520`; the matrix buffer shares the same `WaitForStreamBuffers` drain at ~49,998 matrix changes). Below that wall: every 1000 Applies a submit with no CPU wait (`vk_submit_size`, `vk_renderstate.cpp:43,228-232`); the 9th submit of a frame (≥~8,000 Applies) waits on the fence of slot `mNextSubmit % 8`, i.e. the frame's first submit, not a full drain (`vk_commandbuffer.cpp:343-372`) [xc: the 1000-Apply submit is not a sync and the recycle wait is a one-fence wait on the oldest submit, not a full GPU sync]. **V** (the trigger counts are arithmetic from the code; whether a real frame reaches them is **I**, measure).
6. Device features: `multiDrawIndirect` is **required** (`vulkanbuilders.cpp:1917-1925`) but **never used**; descriptor indexing, drawIndirectCount, shaderDrawParameters, timeline semaphores, fragment shading rate are **not enabled** (descriptor indexing: unless the OpenXR runtime's required device-extension list names `VK_EXT_descriptor_indexing`, `vk_renderdevice.cpp:310-331`, in which case the copied feature bits are chained, `vulkandevice.cpp:167-171` [xc: added path; I, runtime-dependent]); multiview is enabled but the multiview render path is **off by default** (`vr_openxr_multiview` = false, `hw_vrmodes.cpp:643`). **V**
7. Pipelines: lazy, compiled on first use inside the frame (`vk_renderpass.cpp:352-363`), one `std::map` per render-pass key; any change of scene size/samples/layers **drops every pipeline and render pass** (`vk_renderbuffers.cpp:192-196`, `vk_renderpass.cpp:120-124`). Disk `VkPipelineCache` exists (`vk_renderpass.cpp:43-118`). **V**
8. Shaders: all scene shaders compiled from GLSL by glslang **serially on the main thread at startup**, no SPIR-V cache (`d_main.cpp:4694`, `vulkanbuilders.cpp:251-311`): per pass 23 material + 6 NAT + U user + 10 effect programs (+4 scene-depth frags) × 2 pass types; each `main.fp` compile is ~260 KB of GLSL (main.fp 207 KB). PP and compute shaders compile lazily mid-frame. **V**
9. Descriptor model is GZDoom's: set 0 "fixed" (bindings 0,1,[2],3-10), set 1 "HW buffers" (0-15 on main; 16-17 added on renderer), set 2 per-material texture set (≥11 combined samplers). No bindless, no indirect; ZVulkan already contains unused bindless allocation and per-binding-flag support. **V**
10. Fork additions in the backend: 5 lazily created compute clients (smoke, level field, debris, surface damage, emissive volumes) recorded on the graphics queue into the draw command buffer before the eye loop; they also **depend on the per-frame full sync** for their host-visible lists and the smoke tile readback. **V**

---

## 2. Entry points and call graph

### 2.1 Startup
- Instance: `Win32VulkanVideo::Win32VulkanVideo` (`win32vulkanvideo.h:52-95`) / SDL `sdlglvideo.cpp:398-436`. Tries API 1.2→1.1→1.0, filtered by the XR runtime's min/max when `vr_mode == VR_OPENXR_MOBILE` (15). Instance optional exts: surface caps2, physdev props2, swapchain colorspace (`vulkanbuilders.cpp:1728-1765`). **V**
- Device: `VulkanRenderDevice::VulkanRenderDevice` (`vk_renderdevice.cpp:299-435`) → `VulkanDeviceBuilder` (`vulkanbuilders.cpp:1807-2080`) → `VulkanDevice::CreateDevice` (`vulkandevice.cpp:96-203`). Details §6.
- `VulkanRenderDevice::InitializeState` (`vk_renderdevice.cpp:492-648`): creates `VkCommandBufferManager` (graphics queue), samplers, texture/framebuffer/buffer managers, `VkRenderBuffers` ×2 (screen, savepic), light/scene mask capability probes (`534-600`), `VkPostprocess`, `VkDescriptorSetManager`, `VkRenderPassManager` (loads pipeline cache), `VkRaytrace`, the shared HW buffers (vertex, sky, viewpoints, lights, bones, GPU particles, drawn lines, particle defs, view lights, effect lights, sector planes, mesh particles), `VkShaderManager`, `VkDescriptorSetManager::Init`, `VkRenderState`, `VkComputeManager`. **V**
- Shader compile: `d_main.cpp:4629` (first call) and `d_main.cpp:4694-4700` loop `screen->CompileNextShader()` → `VkShaderManager::CompileNextShader` (`vk_shader.cpp:75-296`), one program per call, main thread. **V**
- Background loaders (if `gl_texture_thread`, default true, `hw_cvars.cpp:74`): `vk_renderdevice.cpp:386-434`. Upload-capable workers are created only if the upload queue family **supports graphics** (`403-406`); otherwise load-only workers whose results are uploaded **on the main thread** in `UploadLoadedTextures` (`1071-1121`). **V**. On NVIDIA the first transfer-only family is chosen as UploadFamily (`vulkanbuilders.cpp:2003-2022`), so upload workers would be refused and uploads land on the main thread — **I** (depends on the GPU's queue families; the log line `Upload Queue Family: #N (graphics-capable|transfer-only)` at `vk_renderdevice.cpp:1585` settles it).

### 2.2 Per displayed frame (flat and VR)
```
D_Display (d_main.cpp:1685)  screen->BeginFrame()
  VulkanRenderDevice::BeginFrame                 vk_renderdevice.cpp:1449-1507
    VR: vrmode->SetUp(); BeginXRFrame()          vk_openxrdevice.cpp:6006-6131 (xrWaitFrame, xrBeginFrame, xrLocateViews)
    SetViewportRects
    mViewpoints->Clear()
    UpdateBackgroundCache(false) -> UploadLoadedTextures(budget gl_background_flush_count=100)   :1123-1249
    mCommands->BeginFrame()   (reset timestamp queries)                                    vk_commandbuffer.cpp:332-339
    mTextureManager->BeginFrame() (shadow map size, particle atlases)                      vk_texture.cpp:68-118
    mScreenBuffers->BeginFrame(...)  size/samples/layers change => WaitForCommands + RenderBuffersReset  vk_renderbuffers.cpp:186-211
    mSaveBuffers->BeginFrame(...)
    UpdateLightMask(); UpdateSceneMask()                                                   :1719-1789
    mRenderState->BeginFrame()                                                             vk_renderstate.cpp:531-536
    mDescriptorSetManager->BeginFrame() -> UpdateFixedSet + UpdateHWBufferSet              vk_descriptorset.cpp:77-276
... hw_entrypoint.cpp RenderViewpoint:
    screen->RunFrameCompute(input)  (main view only, before eye loop)  hw_entrypoint.cpp:335-338 -> VkComputeManager::RunFrame vk_compute.cpp:54-110
    for each eye (hw_entrypoint.cpp:358-...):
        SetSceneRenderTarget  vk_renderdevice.cpp:1620-1662  (-> VkRenderState::SetRenderTarget, ends pass)
        scene draws -> VkRenderState::Draw/DrawIndexed/DrawInstanced -> Apply
        PostProcessScene -> VkPostprocess::BlitSceneToPostprocess (resolve/blit) + PP passes (VkPPRenderState::Draw)
End2DAndUpdate (d_main.cpp:1533) -> VulkanRenderDevice::Update  vk_renderdevice.cpp:657-755
    per eye: Draw2D(true/false), FinalizeEyeImage
    EndRenderPass; EndFrame (reset stream/matrix writers)          vk_renderstate.cpp:647-651
    WaitForCommands(...)  == FlushCommands(final submit) + [present] + poll ALL fences   vk_commandbuffer.cpp:430-513
    UpdateGpuStats (vkGetQueryPoolResults WAIT)                    vk_commandbuffer.cpp:573-604
    VR: AcquireXRSwapchain: xrAcquire/xrWaitSwapchainImage, own cmd buffer blit per eye,
        vkQueueSubmit, poll fence, xrReleaseSwapchainImage, xrEndFrame   vk_openxrdevice.cpp:6133-6479
    VR+sync_mode 1: AcquireImage (desktop mirror recording) + WaitForCommands(true,false,false) (present+wait)
```
All **V** at the cited lines.

### 2.3 Per draw
```
FRenderState::Draw* (callers in rendering/hwrenderer/*, 55 call sites)
 -> VkRenderState::Draw / DrawIndexed / DrawInstanced / DrawIndexedInstanced   vk_renderstate.cpp:59-100
    if (apply || mNeedApply) Apply(dt)                                          :223-248
       mApplyCount >= vk_submit_size(1000) -> FlushCommands(false)              :227-232
       ApplyStreamData   :379-395  (StreamData 2000 B memcpy; full => WaitForStreamBuffers)
       ApplyMatrices     :448-455  (3x mat4 compare; changed => new 192 B block)
       ApplyRenderPass   :259-313  (VkPipelineKey build+memcmp; maybe BeginRenderPass; maybe map lookup/compile + bindPipeline)
       ApplyScissor/Viewport/StencilRef/DepthBias  (dirty-flag gated)
       ApplyPushConstants :397-446 (always: 64 B vkCmdPushConstants)
       ApplyVertexBuffers :457-476 (on change)
       ApplyHWBufferSet   :495-512 (on viewpoint/matrix/stream offset change: bind set0 + set1 w/ 3 dynamic offsets)
       ApplyMaterial      :478-493 (on mMaterial.mChanged: VkMaterial::GetDescriptorSet + bind set0 + set2)
    vkCmdDraw / vkCmdDrawIndexed
```
**V**.

---

## 3. Per-function table (hot path)

| Function | file:line | Called from | Per | Cost class | Notes |
|---|---|---|---|---|---|
| `VulkanRenderDevice::BeginFrame` | vk_renderdevice.cpp:1449 | D_Display (d_main.cpp:1685) | frame | O(1) + O(textures uploaded) | VR xrWaitFrame blocks here (pacing). V |
| `UpdateBackgroundCache(false)` | vk_renderdevice.cpp:1123 | BeginFrame | frame | O(queued patches+models) + ≤100 texture creations | Main-thread `BackgroundCreateTexture` when load-only workers (staging alloc, memcpy, vmaCreateImage, copy, mip blits). V |
| `VkCommandBufferManager::BeginFrame` | vk_commandbuffer.cpp:332 | BeginFrame | frame | O(1) | resets timestamp pool only if used. V |
| `VkTextureManager::BeginFrame` | vk_texture.cpp:68 | BeginFrame | frame | O(1); rebuild on generation change | Shadowmap re-create on `gl_shadowmap_quality` change; particle atlases rebuild with ≥64 MB staging → `WaitForCommands(false,true)` (vk_texture.cpp:693-695). V |
| `VkRenderBuffers::BeginFrame` | vk_renderbuffers.cpp:186 | BeginFrame | frame | O(1); on change: full GPU wait + drop all pipelines | V |
| `VkDescriptorSetManager::BeginFrame` | vk_descriptorset.cpp:77 | BeginFrame | frame | 2 vkAllocateDescriptorSets, ~26 descriptor writes, 2 vkUpdateDescriptorSets | Old sets go to DrawDeleteList (freed after the frame wait). Pool `maxSets=10` (vk_descriptorset.h:132); exhaustion → `WaitForCommands(false)` (:88-91, :185-190). V |
| `VkComputeManager::RunFrame` | vk_compute.cpp:54 | hw_entrypoint.cpp:338 | frame | 0 when idle; O(dispatches) | §8. V |
| `VulkanRenderDevice::SetSceneRenderTarget` | vk_renderdevice.cpp:1620 | hw_entrypoint eye loop | eye | O(1), ends render pass | V |
| `VkRenderState::Draw/DrawIndexed` | vk_renderstate.cpp:59/67 | scene/2D code | draw | Apply + 1 vkCmdDraw* | V |
| `VkRenderState::DrawInstanced/DrawIndexedInstanced` | vk_renderstate.cpp:80/91 | mesh particles | draw | same; uses firstInstance | Only instanced path in the engine. V |
| `VkRenderState::Apply` | vk_renderstate.cpp:223 | Draw* | draw (apply=true) | O(1) but ~2.3 KB written + 1–12 vkCmd | Clocked by `drawcalls` (excludes the vkCmdDraw itself). V |
| `ApplyStreamData` | vk_renderstate.cpp:379 | Apply | draw | memcpy 2000 B to host-visible (likely BAR/WC) memory; `timer` double mul; vertex-format vector index | No dedupe: every Apply writes a new slot. V |
| `VkStreamBufferWriter::Write` | vk_streambuffer.cpp:34 | ApplyStreamData | draw | O(1); every 32nd call a new 64 KB block | Returns false at end of buffer → forced sync. V |
| `ApplyMatrices` / `VkMatrixBufferWriter::Write` | vk_renderstate.cpp:448 / vk_streambuffer.cpp:81 | Apply | draw | 3×64 B memcmp; on change computeNormalMatrix (:89) + 192 B memcpy (:113) | Identity substituted when disabled. V |
| `ApplyRenderPass` | vk_renderstate.cpp:259 | Apply | draw | build 64 B key [xc: 64 B, not 68: FRenderStyle 4 B + 15 ints, vk_renderpass.h:38-59]; memcmp vs last; on change `std::map` lookup (memcmp compares) + vkCmdBindPipeline; on miss pipeline compile | `r_skipmats` read per draw (:288). V |
| `VkRenderState::BeginRenderPass` | vk_renderstate.cpp:680 | ApplyRenderPass, SetSceneDepthReadable | render pass | map lookups (render pass key, framebuffer key), maybe framebuffer create, vkCmdBeginRenderPass | Forces material rebind (:763). V |
| `VkRenderPassSetup::GetPipeline` | vk_renderpass.cpp:352 | ApplyRenderPass | pipeline change | `std::map<VkPipelineKey>` O(log n); miss = vkCreateGraphicsPipelines (+perf_track REASON_PIPELINE) | V |
| `ApplyScissor/Viewport/StencilRef/DepthBias` | vk_renderstate.cpp:324/353/315/250 | Apply | draw | dirty-flag; each 1 vkCmd when dirty | Forced dirty at each new render pass. V |
| `ApplyPushConstants` | vk_renderstate.cpp:397 | Apply | draw | 64 B, **unconditional** vkCmdPushConstants | Also derefs material source for canvas/gloss each draw. V |
| `ApplyVertexBuffers` | vk_renderstate.cpp:457 | Apply | draw | 0–2 vkCmd on change (2 bindings, same buffer) | V |
| `ApplyHWBufferSet` | vk_renderstate.cpp:495 | Apply | draw | 2 vkCmdBindDescriptorSets when viewpoint/matrix/stream offset changed | Stream offset changes every 32 Applies; matrix offset on every matrix change. Set 0 is re-bound redundantly. V |
| `ApplyMaterial` | vk_renderstate.cpp:478 | Apply | draw (on SetMaterial) | `VkMaterial::GetDescriptorSet` (linear scan + `GPalette.GetTranslation`) + 2 vkCmdBindDescriptorSets; canvas NeedUpdate | Set 0 re-bound redundantly (layouts compatible). V |
| `VkMaterial::GetDescriptorSet` | vk_hwtexture.cpp:643 | ApplyMaterial | material change | O(#sets of material); miss: allocate + ≥11 writes + possible **synchronous texture create** via `GetImage` | V |
| `VkHardwareTexture::GetImage/CreateImage` | vk_hwtexture.cpp:93/139 | GetDescriptorSet, precache | first use | CPU decode + staging + vmaCreateImage + copy + mips; >64 MB queued → `WaitForCommands(false,true)` (:275-278) | perf_track REASON_TEXTURE. V |
| `VkRenderState::EndRenderPass` | vk_renderstate.cpp:538 | many | pass | vkCmdEndRenderPass; invalidates cached bindings | V |
| `VkPPRenderState::Draw` | vk_pprenderstate.cpp:55 | PP passes | PP pass | own render pass, new descriptor set + vkUpdateDescriptorSets per pass, image barriers, 1 fullscreen draw | Lazy PP shader + pipeline create on first use. V |
| `VkPostprocess::BlitSceneToPostprocess` | vk_postprocess.cpp:131 | PostProcessScene | eye | vkCmdResolveImage (MSAA) or vkCmdBlitImage | Resolve is a separate transfer, not a render-pass resolve. V |
| `VulkanRenderDevice::Update` | vk_renderdevice.cpp:657 | End2DAndUpdate | frame | 2D per eye + final sync(s) | Clocks `Flush3D`, `VREyeComposite`, `VRRenderSyncWait`. V |
| `VkCommandBufferManager::FlushCommands` | vk_commandbuffer.cpp:341/400 | Apply (every 1000), WaitForCommands, others | submit | vkEndCommandBuffer, vkQueueSubmit, semaphore chain; slot ≥8 → poll fence | Command buffers allocated per frame, freed via delete list. V |
| `VkCommandBufferManager::WaitForCommands` | vk_commandbuffer.cpp:430 | Update, many sync points | frame + forced syncs | full GPU drain (polled `vkGetFenceStatus` + 1 ms sleeps, `RS_WaitFencesPolled` :181-221) | Clocked by `Finish` when `finish`. V |
| `PushGroup/PopGroup` | vk_commandbuffer.cpp:522/549 | PP passes, perflog groups, compute | group | std::string concat + std::set lookup + vkCmdSetCheckpointNV (when `vk_gpu_checkpoints`, default on) | Runs even with `stat gpu` off. V |
| `UpdateGpuStats` | vk_commandbuffer.cpp:573 | Update | frame | vkGetQueryPoolResults(WAIT) only if timestamps written | V |
| `VKOpenXRDeviceMode::AcquireXRSwapchain` | vk_openxrdevice.cpp:6133 | Update | VR frame | acquire+wait image (≤20 ms timeout), N blits, 1 submit, **CPU fence poll**, release, xrEndFrame | Clocks `VRSubmit`, `VRSubmitCopy`, `VRSubmitWait` (shared with image wait). V |
| `VkFramebufferManager::AcquireImage/QueuePresent` | vk_framebuffer.cpp:56/139 | WaitForCommands(finish) | frame | swapchain acquire (1 s timeout), desktop present draw or mirror composition, present | V |

---

## 4. Data structures that matter for speed

### 4.1 Per-draw blocks (compiled sizes; identical on main, renderer, netplay)
| Struct | Size | Where | Notes |
|---|---|---|---|
| `StreamData` | **2000 B** | hw_renderstate.h:213-385 | Mirrors GLSL `StreamData` in prolog (vk_shader.cpp:767-846). Contains 64×vec4 flat-glow lines, 3×8 sweep vec4 arrays, glow/fade/outline. Written whole on every Apply. |
| `StreamUBO` | 64,000 B (32 × 2000) | vk_shader.h:49-54 | `MAX_STREAM_DATA = 65536/2000 = 32`. Bound as dynamic UBO, range 64,000 (vk_descriptorset.cpp:164). Shader indexes `data[uDataIndex]` with a push constant (vk_shader.cpp:984-1040). |
| `MatricesUBO` | 192 B | vk_shader.h:42-47 | Model, normal-model, texture matrix. One block per *change*. |
| `PushConstants` | 64 B | vk_shader.h:56-80 | Vertex+fragment range (vk_renderpass.cpp:193). |
| `HWViewpointUniforms` | **26,944 B** | hw_viewpointuniforms.h | Bound with range 2× block (both eyes, `vk_descriptorset.cpp:94`) = 53,888 B when `minUniformBufferOffsetAlignment` ≤ 64 (block = sizeof rounded up, `hw_viewpointbuffer.cpp:31`; 54,272 B at alignment 256) [xc: range depends on device alignment]; one memcpy of 26,944 B per SetViewpoint/Set2D, two (53,888 B) per SetViewpoint when multiview is active (`hw_viewpointbuffer.cpp:109-118,140`) [xc: multiview case added]. Holds 128-entry beam/shape arrays etc. |
| selaco | StreamData 1984 B, MAX_STREAM_DATA 33, viewpoint 24,624 B (compiled) | | |

### 4.2 Ring/stream buffers (all Persistent = HOST_VISIBLE|COHERENT, preferring DEVICE_LOCAL, dedicated, mapped: vk_hwbuffer.cpp:100-115)
- `StreamBuffer`: 300 blocks × 64,000 B = 19.2 MB (`vk_buffer.cpp:39`, `118-139`). Usable blocks per frame: 298 (offset 0 is skipped by the first `Write`, `vk_streambuffer.cpp:34-48` + `Reset` at :49; end test `offset+block >= size` at `vk_buffer.cpp:131-139`) → **~9,536 Applies/frame** before `WaitForStreamBuffers` (vk_renderstate.cpp:514-520: `WaitForCommands(false)` = full drain, then reuse from 0). **V** (arithmetic I for alignment 64; with alignment 256 the block is still 64,000 B because 64,000 = 250×256).
- `MatrixBuffer`: 50,000 × 192 B (alignment 64) = 9.6 MB (`vk_buffer.cpp:38`); ~49,998 matrix changes/frame before the same forced drain. **V/I** (alignment from device).
- Viewpoints: 100 × block (26,944 B at alignment ≤64), single buffer; growth once `mUploadIndex + count + 1 > 100`, i.e. from the 99th upload of a frame (`hw_viewpointbuffer.cpp:53-70`) [xc: threshold is 99 uploads, not >100] → `VkHardwareBuffer::Resize` → `WaitForCommands(false)` + `UpdateHWBufferSet` mid-frame (`hw_viewpointbuffer.cpp:CheckSize` → `vk_hwbuffer.cpp:161-191`). **V**
- Flat vertex buffer: 2,000,000 vertices persistent, pipelineNbr 1 (`flatvertices.h:100,115`; `flatvertices.cpp:82`). **V** Its wall is not a sync: `AllocVertices` past 1,999,500 vertices in a frame is `I_FatalError` (`flatvertices.cpp:139-149`). [xc: added]
- Light / bone buffers: fixed size persistent, pipelineNbr 1 (`hw_lightbuffer.cpp:57`, `hw_bonebuffer.cpp:50`). **V**
- Static/Stream `SetData`: new GPU buffer + new CPU staging each call, old ones to delete list; comment "we could recycle buffers here for the stream usage type" (`vk_hwbuffer.cpp:59-98`). Mid-frame `SetData` uploads go to the transfer command buffer, which is submitted *before* the draw buffer in the same `vkQueueSubmit` (`vk_commandbuffer.cpp:405-424`) — so a `SetData` recorded after draws that used the old contents is safe only because a new buffer is created each time. **V**

### 4.3 Pipeline key / render pass key
- `VkPipelineKey` 64 B: RenderStyle (4-byte union, `renderstyle.h:138-147`) + 15 ints (`vk_renderpass.h:38-59`) [xc: was "68 B, 16 ints"], memcmp ordering. Per render-pass key a separate `std::map<VkPipelineKey, pipeline>` (`vk_renderpass.h:115`). Dynamic state: viewport, scissor, depth bias, stencil ref only (`vk_renderpass.cpp:439-442`); cull, depth test/write/func, stencil op, blend, color mask, topology are **baked** → each combination is a pipeline. **V**
- `VkRenderPassKey` 9 ints (`vk_renderpass.h:61-100`, static_assert). Pass variants: samples, draw buffers (1 or 3), layers, view mask, depth read-only, light mask, post mask. Each variant owns its own pipeline map (duplication). `RenderPasses[8]` per key by clear mask (`vk_renderpass.cpp:345-350`). **V**
- Pipeline layouts: one per `NumTextureLayers` (`vk_renderpass.cpp:177-197`), forced ≥ 11 (`vk_renderstate.cpp:276`, `SHADER_MIN_REQUIRED_TEXTURE_LAYERS 11` vk_shader.h:34). **V**

### 4.4 Descriptor sets (main)
**Set 0 — FixedSet** (`vk_descriptorset.cpp:614-662`, written each frame `181-276`):

| Binding | Type | Stage | Content | Declared in GLSL by |
|---|---|---|---|---|
| 0 | CIS | frag | shadowmap 1D-as-2D | prolog (vk_shader.cpp:491) |
| 1 | CIS | frag | lightmap array | prolog (:492) |
| 2 | accel struct | frag | TLAS, only if `RaytracingEnabled()` | prolog under SUPPORTS_RAYTRACING (:493-495) |
| 3 | CIS | frag | scene depth (read-only depth pass) | only scene-depth variants (vk_shader.cpp:1079-1100) |
| 4 | CIS | frag | particle atlas | gpuparticles.fp |
| 5,6 | CIS | vert | level field fine/coarse (GENERAL) | gpuparticles.vp |
| 7,8 | CIS | frag | surface damage pages/detail | main.fp (SURFACE_DAMAGE) |
| 9 | CIS | vert | level field header | gpuparticles.vp |
| 10 | CIS | frag | BC7 particle atlas | gpuparticles.fp |

**Set 1 — HWBufferSet** (`vk_descriptorset.cpp:543-612`, written each frame `83-179`):

| Binding | Type | Stage | Buffer | Declared by |
|---|---|---|---|---|
| 0 | UBO dyn | v+f | viewpoints (range 2 blocks) | prolog |
| 1 | UBO dyn | v+f | matrices (192 B) | prolog |
| 2 | UBO dyn | v+f | stream data (64,000 B) | prolog |
| 3 | SSBO | f | lights | prolog |
| 4 | SSBO | v | bones | prolog |
| 5 | SSBO | v | GPU particle ring | prolog |
| 6 | SSBO | v | drawn lines | prolog |
| 7 | SSBO | v+f | particle definitions | prolog |
| 8 | SSBO | v | view lights | gpuparticles.vp, meshparticles.vp |
| 9 | SSBO | v | mesh particles | meshparticles.vp |
| 10,11 | SSBO | v | debris pieces / definitions | gpuparticles.vp, meshparticles.vp |
| 12 | SSBO | v+f | sector planes | **main: no GLSL lump declares it** (grep §7); **renderer: `levelmesh.vp:109`**, read at :229-230 (flat plane per sector) [xc: renderer's levelmesh.vp declares and reads it] |
| 13 | SSBO | f | surface damage data | main.fp |
| 14,15 | SSBO | v+f | effect light records / bins | main.fp, gpuparticles.vp, meshparticles.vp |
| 16,17 | SSBO | v / v+f | **renderer only**: level mesh surfaces / lights | renderer: levelmesh.vp:66,96; main.fp:80 |

Absent buffers are stood in by the bone buffer so every binding is always valid (`vk_descriptorset.cpp:101-159`). **V**

**Set 2 — material textures**: layout per layer count, all CIS frag (`vk_descriptorset.cpp:453-470`); allocated from pools of 1000 sets / 2000 descriptors (`434-451`) → ~181 materials per pool at 11 descriptors each (I, arithmetic). Each material caches one set per (clamp, translation) (`vk_hwtexture.cpp:643-699`). `ResetHWTextureSets` (filter change, precache start) drops all (`vk_descriptorset.cpp:400-415`). **V**

**PP sets**: one new set per PP pass per frame from a 100-set pool, replaced (old to delete list) when full (`vk_descriptorset.cpp:484-541`). **V**

### 4.5 Render target chain (`vk_renderbuffers.cpp`)
- Scene (per `CreateScene`, :274-302): SceneColor RGBA16F MSAA, SceneDepthStencil D24S8 (fallback D32S8), SceneNormal A2R10G10B10, SceneFog RGBA8 — **all four always created**, even when `gl_ssao` = 0 (default, `hw_postprocess_cvars.cpp:486`) and the pass has DrawBuffers = 1 (VRAM waste at MSAA; **V** creation unconditional). Optional SceneLightMask / ScenePostMask. Layers = eye count when multiview.
- Pipeline images: 4 × RGBA16F + pipeline depth (:237-272); eyes use pairs (0/1, 2/3) (`vk_renderdevice.cpp:684-687`, `1418-1420`).
- MSAA count from `gl_multisample` clamped to device support (:156-184). Resolve by `vkCmdResolveImage` per eye (vk_postprocess.cpp:162-181).
- Recreate on any change of pipeline size, samples or layers, which also calls `RenderBuffersReset` (all scene + PP pipelines and render passes discarded) after a full `WaitForCommands(false)` (:192-196). In VR the size flips between recommended eye size and window size with `FrameRenderMode` (`vk_openxrdevice.cpp:2191-2196`, used in `BeginFrame` :1467-1478) — **I**: entering/leaving a non-MenuOverWorld menu recreates buffers and re-creates every pipeline lazily (pipeline cache helps; measure with perf_track REASON_PIPELINE).

---

## 5. What slows it down (ranked)

| # | Issue | Evidence | Confidence | Effect |
|---|---|---|---|---|
| 1 | **No frames in flight**: CPU waits for the whole GPU frame at end of every frame; VR adds a second (XR blit) and third (desktop mirror, sync mode 1) wait. CPU and GPU time add instead of overlap. | vk_commandbuffer.cpp:452-502; vk_renderdevice.cpp:730-751; vk_openxrdevice.cpp:6342-6379 | V (mechanism) | Frame time ≈ CPU + GPU + 2 extra submit round trips. Previously measured: "of 14.5 ms, scene build is 4.7, GPU wait 3.6" (HANDOFF_RENDER_2026-11-11.md, 1112r build, flat). |
| 2 | **Single-buffered persistent buffers** everywhere (stream, matrix, viewpoint, light, bone, flat vertices, level frame indices on renderer, compute host lists, smoke tile readback). Blocks fixing #1. | §4.2; vk_debrispool.cpp:143-145, vk_surfacedamage.cpp:153-155 (they state the reliance); vk_smokevolume.cpp:1436-1450 | V | Any frames-in-flight change needs N-copies of each, keyed by frame slot. |
| 3 | **Per-draw CPU work in Apply**: 2000 B StreamData memcpy into likely write-combined memory on every draw, 64 B push constant every draw, pipeline key build + memcmp, 2 descriptor binds per SetMaterial (no same-material check), 2 binds every 32 draws, matrix compare. | vk_renderstate.cpp:223-512; hw_renderstate.h:1095-1105; vk_streambuffer.cpp:34-48 | V (work); I (WC memory: allocation prefers DEVICE_LOCAL+HOST_VISIBLE, vk_hwbuffer.cpp:104-108) | Previously measured `drawcalls`: 0.74–0.76 ms (Sunder MAP15) and 5.0–6.0 ms (Abandon MAP13 owner list) per frame on the old path (HANDOFF_RENDER_2026-11-11.md, builds 1112q/1112r and 1112j). |
| 4 | **Forced mid-frame full drains** when capacity walls are hit: stream buffer (~9,536 Applies), matrix buffer (~50k changes), viewpoint growth (99th upload in a frame), descriptor pool exhaustion, >64 MB of queued staging (texture creates, particle atlas). The 64 MB case is `WaitForCommands(false, true)` (uploadOnly): it submits only the transfer buffer and waits on the fences of the submits made so far this frame; the draw buffer being recorded is not submitted, so it is not a full drain of the frame's work, and when `bufManager` is a worker's upload manager the wait is on that worker thread [xc: 64 MB case is not a full drain; viewpoint threshold is 99]. | vk_renderstate.cpp:390-394,450-454,514-520; vk_hwbuffer.cpp:185; vk_descriptorset.cpp:88-91,185-190; vk_hwtexture.cpp:275-278,510-512; vk_texture.cpp:693-695 | V (code); I (whether hit; Abandon-class frames with 12–16k walls reached per view ×2 eyes could reach 9.5k Applies) | Each is a CPU stall of the remaining GPU work, mid-frame. Not separately counted anywhere. |
| 5 | **Submit chopping**: a submit every 1000 Applies (`vk_submit_size`, not archived), each ending the render pass (re-begin with LOAD); after 8 submits a CPU wait on slot recycling. | vk_renderstate.cpp:43,227-232; vk_commandbuffer.cpp:343-372,400-428 | V | Extra render-pass restarts every 1000 draws (LOAD/STORE of the pass attachments: SceneColor + depth/stencil by default, 4 only with `gl_ssao` on, since `GetPassDrawBufferCount` is 1 for NORMAL_PASS, `hw_renderstate.h:1183-1186`) [xc: was "4 MSAA attachments"]; CPU fence wait at the 9th submit of a frame (≥~8,000 Applies), on the frame's oldest submit. `rendered_commandbuffers` counts them (vk_commandbuffer.cpp:426,510). |
| 6 | **Two full scene submissions per VR frame** (multiview off by default): every draw, Apply, StreamData and bind is done twice. | hw_vrmodes.cpp:643; vk_openxrdevice.cpp:8608-8616; hw_entrypoint.cpp:354-372 | V (default); owner's ini may override (I) | ~2× #3 and #4 pressure in VR. |
| 7 | **Lazy pipeline creation inside the frame** and wholesale pipeline drop on target size/sample/layer change. | vk_renderpass.cpp:352-363,120-124; vk_renderbuffers.cpp:192-196 | V | Hitches; perf_track REASON_PIPELINE marks them. |
| 8 | **Synchronous texture creation on first draw** for walls/flats (sprites/decals/weapons go through the background loader and skip until ready); main-thread upload of loader output when no graphics-capable upload queue. | vk_hwtexture.cpp:93-107,139-175,643-699; vk_renderdevice.cpp:403-406,1071-1121; hw_sprites.cpp:1264-1321 | V; I (NVIDIA queue layout) | Hitches; perf_track REASON_TEXTURE. |
| 9 | **Fat UBOs**: 53.9 KB viewpoint range and 64 KB stream range bound per draw; fragment shader reads per-draw data through a dynamically indexed UBO; main.fp is 207 KB with many uniform-count loops (128-iteration beam/shape loops, 64 flat-glow lines, 8 sweep bands). | vk_shader.cpp:663-665,848-850; main.fp:1593,1876,2303,2407,4262 | V (sizes/loops); GPU cost I | GPU-side register/cache pressure; the GPU groups `gpu.opaque.*` measure it. |
| 10 | **MSAA resolve as a separate transfer per eye** and 4 MSAA scene attachments always allocated. | vk_postprocess.cpp:162-181; vk_renderbuffers.cpp:274-302 | V | Bandwidth/VRAM; small on desktop (I). |
| 11 | **PP passes**: one render pass, one descriptor set allocate+update and barriers per pass; PP shaders and compute programs compiled on first use mid-frame. | vk_pprenderstate.cpp:54-121; vk_ppshader.cpp:31-63; vk_compute.cpp:125-199 | V | Small per pass; first-use hitches. |
| 12 | **Diagnostics always on**: `vk_gpu_checkpoints` default true → vkCmdSetCheckpointNV per Push/PopGroup; string building + std::set lookup per group; fence wait polls with `sleep_for(1 ms)` between polls (`vk_commandbuffer.cpp:219`); with the 1 ms Windows timer period (`i_main.cpp:286`) each failed poll costs ≥1 ms and typically 1–2 ms, so the wait overshoots the GPU finish by up to ~1–2 ms [xc: "up to ~1 ms" understated; Sleep granularity I]. | vk_renderdevice.cpp:114; vk_commandbuffer.cpp:181-221,237-269,522-530 | V (code); I (sleep granularity cost; NV checkpoint overhead) | Measure with checkpoints off and with a blocking vkWaitForFences build. |
| 13 | Startup: serial glslang compile of ~2×(39+U) programs, no SPIR-V cache. | vk_shader.cpp:75-296; d_main.cpp:4694; vulkanbuilders.cpp:251-311 | V | Load time only. |

---

## 6. Device, features, extensions, queues, swapchain

### 6.1 Requested vs enabled (main)
| Feature/ext | Request site | Enabled? |
|---|---|---|
| samplerAnisotropy, fragmentStoresAndAtomics, **multiDrawIndirect**, independentBlend | required, `vulkanbuilders.cpp:1917-1925` | yes (device rejected otherwise) |
| depthClamp, shaderClipDistance, textureCompressionBC | copied if present, :1945-1951 | yes if supported |
| multiview (+GS/TS) | :1952-1954; chained if API≥1.1 (`vulkandevice.cpp:147-151`) | yes if supported |
| bufferDeviceAddress, accelerationStructure, rayQuery (+deferred host ops) | `OptionalRayQuery` (vk_renderdevice.cpp:303; vulkanbuilders.cpp:1825-1832) | yes if supported (always enabled on RT-capable GPUs even with `vk_raytrace` 0) |
| descriptor indexing (runtimeDescriptorArray, partiallyBound, updateAfterBind, variableCount, nonUniform) | feature bits copied (:1958-1962) but extension only added by `OptionalDescriptorIndexing`, **never called** (grep: only its definition at :1834); struct chained only if the extension is enabled (`vulkandevice.cpp:167-171`) | **no** (yes only if the OpenXR runtime's required device extensions include `VK_EXT_descriptor_indexing`, `vk_renderdevice.cpp:313-330`; I) [xc: added the XR path] |
| drawIndirectCount, shaderDrawParameters, timelineSemaphore, VK 1.2 feature structs | not referenced in any feature chain (grep §7) | **no** |
| fragment shading rate / VRS | not requested (only volk loader symbols) | **no** |
| VK_EXT_device_fault, VK_NV_device_diagnostic_checkpoints | :305-306 | yes if supported (checkpoints gated by `vk_gpu_checkpoints`) |
| robustBufferAccess | only with `vk_robust_buffer_access` (default 0) | no by default |
| dedicated alloc / get mem req 2 | builder ctor :1807-1811 | yes |
| full screen exclusive (Win32) | :1876-1878 | optional |
| XR-required device exts | vk_renderdevice.cpp:310-331 | yes |

GLSL target: `#version 460` + `GL_EXT_ray_query` when instance API is exactly 1.2, else 450; `GL_EXT_multiview` + `SUPPORTS_MULTIVIEW` whenever the multiview feature is enabled (`vk_shader.cpp:1271-1289`). SPIR-V 1.4 for 1.2 (`vulkanbuilders.cpp:269-278`). **V**

### 6.2 Queues
- Graphics = first graphics family; present = first present-capable family (separate queue slot if the family has ≥2 queues, `vulkandevice.cpp:101-110`); upload family = first transfer-without-graphics family, else graphics (`vulkanbuilders.cpp:2003-2022`); up to `vk_max_transfer_threads` (default 2) upload queues (`vk_renderdevice.cpp:340`). **V**
- All rendering, compute and the XR blit go to the single graphics queue. No async compute queue use. **V**

### 6.3 Swapchain / present (desktop)
- `VkFramebufferManager::AcquireImage` recreates the swapchain on size/vsync/HDR/exclusive change; image count 2 (vsync) / 3 (no vsync) (`vk_framebuffer.cpp:55-77`). Present mode: vsync → FIFO_RELAXED if available else FIFO; no vsync → IMMEDIATE (MAILBOX deliberately commented out for windowed) (`vulkanswapchain.cpp:108-134`). `vid_vsync` default false (`v_video.cpp:203`). **V**
- `vkAcquireNextImageKHR` timeout 1 s (`vulkanswapchain.cpp:230`). **V**
- FPS limiter `FPSLimit` runs inside `WaitForCommands` before present when vsync off (`vk_commandbuffer.cpp:446-447`, `v_framebuffer.cpp:244`). **V**

### 6.4 Submission structure
- `FlushCommands`: submits [transfer cmdbuf, draw cmdbuf] in one `vkQueueSubmit`, waits on previous submit's semaphore at TOP_OF_PIPE, signals own semaphore unless last; the final presenting submit waits on image-available and signals render-finished (`vk_commandbuffer.cpp:374-397`). 8 fence slots (`vk_commandbuffer.h:128`). Command buffers are allocated fresh from one pool per frame and freed via the delete list (`vk_commandbuffer.cpp:302-322,515-520`). **V**
- Submits per frame (flat): 1 + floor(Applies/1000) + forced syncs. VR (sync mode 1): the above + 1 XR blit submit (own pool/cmdbuf/fence, `vk_openxrdevice.cpp:6261,6342-6348`) + 1 desktop-mirror submit. **V**

---

## 7. Hooks, cvars and dead code (this area)

| Item | Status | Evidence |
|---|---|---|
| `VulkanDeviceBuilder::OptionalDescriptorIndexing` | **DEAD** (defined, never called) | grep `OptionalDescriptorIndexing` → only `vulkanbuilders.cpp:1834` |
| Descriptor-indexing feature bits copied in `FindDevices` | **DEAD effect** (struct never chained without the ext) | vulkanbuilders.cpp:1958-1962; vulkandevice.cpp:167-171 |
| `VulkanCommandBuffer::drawIndirect / drawIndexedIndirect` | **DEAD** (no caller in src/) | vulkanobjects.h:391-392,780-788; grep `drawIndirect\|drawIndexedIndirect` in src → none |
| `multiDrawIndirect` feature | required, **unused** | vulkanbuilders.cpp:1919; no indirect draw call anywhere |
| ZVulkan bindless allocate (`allocate(layout, bindlessCount)`) + the non-zero `flags` path of `AddBinding` | **DEAD** (no caller of the bindless overloads; `AddBinding` itself is called everywhere, always with the default `flags = 0`, so `VkDescriptorSetLayoutBindingFlagsCreateInfo` is never chained) [xc: AddBinding is called; only its flags path is dead] | vulkanobjects.h:1049-1087; vulkanbuilders.cpp:780-800; vulkanbuilders.h:299; grep `bindlessCount` and 5-argument `AddBinding(` in src → none |
| `VkRenderState::EnableClipDistance / EnableMultisampling / EnableLineSmooth` | **DEAD on Vulkan** (empty bodies) | vk_renderstate.cpp:159-161, 215-221 |
| `VkRenderStateMolten` + `FanToTrisIndexBuffer` | platform-only (`__APPLE__`); the index buffer is still created on every platform (`vk_buffer.cpp:41,102-114`) | vk_renderdevice.cpp:640-644; vk_renderstate.cpp:769-792 |
| Set 1 binding 12 (SectorPlaneSSO) | **main and netplay: DEAD on the GPU side**: in every layout, written every frame, no GLSL (scene, PP or compute) declares it; CPU still fills it (hw_sectorplanes.cpp:111,228). **renderer: LIVE** — `levelmesh.vp:109` declares it and reads `sectorPlanes[]` at :229-230 [xc: not dead on renderer] | grep `binding *= *12` in wadsrc and src → main/netplay: only the comment in hw_sectorplanebuffer.h:31; renderer: also wadsrc/static/shaders/glsl/levelmesh.vp:109 |
| Set 0 binding 2 (TLAS), `VkRaytrace`, `vk_raytrace` (default false) | **DORMANT** (only with `vk_raytrace` or `gl_light_castshadows 2` on RT GPU, decided at device creation) | vk_renderdevice.cpp:100,365-368; vk_descriptorset.cpp:195-196,619-620 |
| Set 0 binding 3 scene depth | **ACTIVE by default** whenever the GPU particle pass runs: `r_gpuparticles_soft` defaults to 8.0 (`hw_cvars.cpp:311`), which turns on the read-only depth pass (`hw_drawinfo.cpp:3928-3932`; the comment there saying "At 0, the default" is stale) — one extra render-pass switch per main view [xc: was DORMANT; cvar default is 8, not 0] | vk_renderstate.cpp:598-645 |
| Light mask / scene mask attachments and SCENE_EXTRA program sets | DORMANT (light mask by cvars; scene mask only if a PP shader names `SceneMask`) | vk_renderdevice.cpp:1719-1789 |
| `vr_openxr_multiview` (default false) and the whole layered/multiview scene path | **DORMANT by default** | hw_vrmodes.cpp:643; vk_openxrdevice.cpp:8608-8616 |
| `vr_openxr_multiview_postprocess` (default true) | only matters when multiview on | hw_vrmodes.cpp:655 |
| `vr_openxr_sync_mode` (default 1: defer desktop present after XR) | ACTIVE | hw_vrmodes.cpp:641; vk_renderdevice.cpp:731 |
| `vk_submit_size` (1000, not archived) | ACTIVE | vk_renderstate.cpp:43 |
| `vk_gpu_checkpoints` (default true) | ACTIVE on NVIDIA | vk_renderdevice.cpp:114; vk_commandbuffer.cpp:256-269 |
| `vk_robust_buffer_access` (0) | DORMANT diagnostic | vk_renderdevice.cpp:115 |
| `vk_debug`, `vk_debug_callstack`, `vk_device`, `vk_hdr`, `vk_exclusivefullscreen`, `vk_listdevices` | ACTIVE | vk_renderdevice.cpp:121-139; vk_framebuffer.cpp:36-37 |
| `r_skipmats` | ACTIVE, read per draw | vk_renderstate.cpp:288 |
| `gl_texture_thread*`, `vk_max_transfer_threads`, `gl_background_flush_count` | ACTIVE | hw_cvars.cpp:74-95 |
| `REMA_RIG_FENCE_SECONDS`, `REMA_RIG_RELAX_XR_EXT` env | rig-only | vk_commandbuffer.cpp:58-64; vk_renderdevice.cpp:318 |
| RS submit journal (`gpu_stall.txt`) | ACTIVE, O(1) per submit, dump on timeout | vk_commandbuffer.cpp:103-176 |

---

## 8. Compute (fork-added, `vk_compute.cpp` and clients)

All created lazily on first demand, recorded into the **draw** command buffer on the graphics queue, between `BeginWork` (ends any render pass, opens group `fx.compute`) and the end of `RunFrame`, once per displayed frame before the eye loop (`vk_compute.cpp:54-123`; `hw_entrypoint.cpp:335-338`). Programs: glslang from `shaders/compute/*.comp` at first use (`vk_compute.cpp:125-199`). Descriptor pools of 32 sets, grown on demand (`201-225`). **V**

| Client | Programs | When dispatched | Notes |
|---|---|---|---|
| `VkSmokeVolume` (vk_smokevolume.cpp) | smoke_inject, smoke_advect, smoke_tiles, smoke_shift (3 variants), smoke_light (light / effect-light / surface-ambient variants) (:573-588,1473,1587) | `Run` while `frame.Active`; `RunStep` per sim step (≤ MAX_STEPS_PER_FRAME); `RunLight` when smoke present (:156-276, 838-950, 1150) | Tile-map GPU→CPU readback each frame consumed next frame **assuming the previous frame finished** (:1385-1450). |
| `VkLevelField` (vk_levelfield.cpp) | field_bake (:448) | `Run` while collision asked; bakes boxes/tiles (:507, :652) | storage images in GENERAL, sampled by gpuparticles.vp (set 0 b5,6,9). |
| `VkDebrisPool` (vk_debrispool.cpp) | debris_step (:352) MODE_CLEAR/MODE_STEP | once per new serial (:86-90), steps ×N (:207-217) | host-visible definitions written in place, relying on full per-frame sync (:143-145). |
| `VkSurfaceDamage` (vk_surfacedamage.cpp) | damage_stamp (stamp, mip), damage_cool (:520-535) | once per new serial, if stamp/cool tiles (:70-196) | host-visible lists written in place (:153-155). |
| `VkEmissiveVolumes` (vk_emissivevolumes.cpp) | none (uploads only: copyBufferToImage :87, :212) | when the list changed | absent on selaco. |

Cost: zero when idle (V: early returns); otherwise proportional to grid/tile counts; CPU clocked as `fx.compute` in r_perflog (`vk_compute.cpp:58-59,108-109`); GPU groups `fx.compute`, `fx.smokesim`, etc. via PushGroup.

---

## 9. Shader system

- Prolog (`shaderBindings`, vk_shader.cpp:489-1057): declares set 0 b0-2, set 1 b0-7, set 2 b0-11, push constants, 139 `#define` indirections (83 into `viewpoints[HW_VIEWPOINT_INDEX]`, 56 into `data[uDataIndex]`) [xc: was ~60], `noise1-4` stubs. Every scene fragment shader additionally gets `func_surfacestamps.fp` (25 KB) + `func_spriteoutline.fp` (6.8 KB) prepended, `SURFACE_DAMAGE` + `EFFECT_LIGHTS` defines for `main.fp` (:1171-1176, :1188-1192). **V**
- Program counts per pass type (2 pass types, NORMAL/GBUFFER): 23 default material programs (`hw_shaderpatcher.cpp:281-327`), 6 NAT (non-alpha-test, `SHADER_NoTexture`=6), U user shaders, 10 effects (`MAX_EFFECTS`, hw_renderstate.h:40-69), +4 gpuparticles scene-depth fragment variants. Each program = 1 vert + 1 frag compile. Total at startup = 2 × (2×(39+U) + 4) = 164 + 4U glslang compiles (I, arithmetic from the loop at vk_shader.cpp:75-296). Light/scene-mask program sets add another full fragment set per pass/combination on first enable (vk_shader.cpp:367-452).
- Sizes: main.fp 207,350 B / 5,128 lines with `#include beamglow.glsl` (22 KB); main.vp 235 lines; gpuparticles.vp 61 KB; meshparticles.vp 31 KB (wc on wadsrc/static/shaders/glsl). **V**
- Compile: `ShaderBuilder::Create` = glslang parse+link+GlslangToSpv with optimizer (optimizeSize) per module, then vkCreateShaderModule (`vulkanbuilders.cpp:251-311`). No SPIR-V cache; the pipeline cache file `M_GetCachePath()/vulkanpipelinecache` is the only persistent cache (`vk_renderpass.cpp:45-47,106-118`). **V**

---

## 10. Per-draw API call count (Vulkan, scene draw)

| Call | When | Count |
|---|---|---|
| vkCmdPushConstants | every Apply | 1 |
| vkCmdDraw / vkCmdDrawIndexed | every draw | 1 |
| vkCmdBindDescriptorSets (set0, set2) | every `SetMaterial` before the draw (always flags change) | 2 |
| vkCmdBindDescriptorSets (set0, set1 w/ 3 dyn offsets) | every 32 Applies, every matrix change, every viewpoint change | 2 |
| vkCmdBindPipeline | pipeline key differs from last | 0–1 |
| vkCmdBindVertexBuffers / vkCmdBindIndexBuffer | buffer or offset change | 0–2 |
| vkCmdSetScissor / SetViewport / SetStencilReference / SetDepthBias | dirty (forced at each new render pass) | 0–4 |
| vkCmdEndRenderPass + vkQueueSubmit + vkCmdBeginRenderPass | every 1000 Applies | amortized |

Typical wall draw with a material set: 4 calls (push, 2 binds, draw); worst 13 (12 from Apply + the draw; +1 vkCmdBeginRenderPass after a pass break) [xc: was ~12]. CPU bytes written per Apply: 2000 (stream) + 64 (push) + 0/192 (matrix). **V** from code paths above; real distribution **I** (no counter exists for binds/pipelines; see §12).

---

## 11. GPU-driven rendering: blockers and what exists

### Blockers (V unless noted)
1. **Per-draw data via dynamic-offset UBO + push-constant index** (`uDataIndex`, 32 slots per binding offset). An indirect draw cannot change dynamic offsets or push constants per draw; needs per-draw data in an SSBO indexed by `gl_DrawID` (shaderDrawParameters not enabled) or by `gl_InstanceIndex`/firstInstance (available: `DrawInstanced` already passes firstInstance, vk_renderstate.cpp:80-100).
2. **Model/texture matrices in a separate dynamic UBO** with per-change offset (vk_renderstate.cpp:448-455, 495-512).
3. **Per-material descriptor set (set 2)**; descriptor indexing not enabled; no texture array/bindless table. ZVulkan already supports variable-count allocation and binding flags (dead today).
4. **Pipeline state per draw** (blend style, depth/stencil ops, cull, effect/material shader index baked into the key): indirect batches must be partitioned per pipeline.
5. **drawIndirectCount not enabled** (needs VK 1.2 Vulkan12Features or the KHR ext); vkCmdDrawIndexedIndirectCount only exists as a volk symbol.
6. **No frames in flight + single-buffered persistent buffers** (§5 #1–2): a GPU-culled indirect buffer per frame plus CPU-written instance data needs per-frame ring copies.
7. **Shader input contract**: `main.vp`/`main.fp` read everything through the macro indirections into `data[uDataIndex]` and `viewpoints[HW_VIEWPOINT_INDEX]`; changing the per-draw source means a new prolog, i.e. every program (vk_shader.cpp:984-1040).
8. **Vertex input**: `FFlatVertexBuffer` streaming via per-draw vertex offsets bound with `vkCmdBindVertexBuffers` (vk_renderstate.cpp:457-469); indirect can carry vertexOffset/firstIndex, so this is not a hard blocker (I).

### What already exists (V)
- `multiDrawIndirect` enabled on every device that runs the engine (required).
- ZVulkan `drawIndirect`/`drawIndexedIndirect` wrappers; bindless set allocation; per-binding flags.
- Instanced draws with firstInstance (mesh particles).
- A compute manager that records dispatches into the frame's draw command buffer with barriers handled by clients (vk_compute.cpp:262-275) — the place a cull pass would go.
- SSBO-heavy set 1 (16 bindings) and, on **renderer**, a static level mesh: one static vertex buffer, static index buffer, per-surface and per-light SSBOs (set 1 b16/b17) read by `levelmesh.vp` through a surface id carried in the vertex, and one `DrawIndexed` per material group (`renderer:src/rendering/hwrenderer/hw_levelresidency.cpp:4629-4688`). Still goes through `Apply` (2000 B StreamData per group) and per-group `SetMaterial`; per-frame indices are CPU-written into a persistent `FrameIndexBuffer` (`renderer:src/common/rendering/hwrenderer/data/hw_levelbuffers.cpp:145`).
- Multiview render passes and layered targets (dormant by default) — the stereo half of the target design.
- `bufferDeviceAddress` enabled on RT-capable GPUs (VMA flag set, `vulkandevice.cpp:86-87`).

---

## 12. Clocks and counters for this area (what they cover / do not cover)

| Counter | Defined | Covers | Does NOT cover |
|---|---|---|---|
| `drawcalls` (glcycle_t) | hw_clock.cpp:59; vk_renderstate.cpp:225,247 | `VkRenderState::Apply` only: stream/matrix writes, pipeline lookup/compile, binds, push constants, mid-frame `FlushCommands` and `WaitForStreamBuffers` | the `vkCmdDraw*` call itself; draws issued with `apply=false && !mNeedApply`; PP draws; ticks only when `glcycle_t::active` (stat rendertimes / printstats / r_perflog, hw_clock.cpp:440-445) |
| `Finish` | hw_clock.cpp:53; vk_commandbuffer.cpp:434-435,509 | `WaitForCommands(finish=true)`: desktop acquire + present-pass recording, final submit, FPSLimit, present, fence poll | VR main wait when sync mode 1 (`finish=false` there). In VR sync mode 1 `Finish` holds only the third wait, the desktop-mirror submit + present + wait (`vk_renderdevice.cpp:747-749`) [xc: added] |
| `VRRenderSyncWait` | hw_clock.cpp:56; vk_renderdevice.cpp:734 | the main end-of-frame wait in VR | the XR blit wait; the sync-mode-1 desktop-mirror wait (in `Finish`) [xc: added] |
| `VRSubmitWait` | vk_openxrdevice.cpp:6187,6369 | xrWaitSwapchainImage **and** the XR blit fence poll (same counter) | |
| `VRSubmit`, `VRSubmitCopy` | vk_openxrdevice.cpp:6260,6282 | XR blit recording/submit/wait / the blit loop | |
| `Flush3D`, `twoD`, `VREyeComposite` | vk_renderdevice.cpp:659-728 | `Update` 2D + composite before the wait | |
| `VRSceneTransfer` (+`VRSceneTransferOps`) | vk_postprocess.cpp:149-150 | scene resolve/blit recording | GPU time |
| perf_track `cpu_syncwait` | hw_clock.cpp:162 | VRSubmitWait + VRRenderSyncWait | flat-mode wait (that is in `Finish`); the VR sync-mode-1 desktop-mirror wait (also in `Finish`) [xc: added] |
| perf_track REASON_PIPELINE / REASON_TEXTURE | vk_renderpass.cpp:359; vk_hwtexture.cpp:227 | pipeline compile, texture create (main and worker threads) | descriptor set writes, PP shader compile |
| GPU timestamp groups (`stat gpu`, r_perflog) | vk_commandbuffer.cpp:522-604 | per PushGroup interval, max 256 queries | per-draw; stage TOP vs COLOR_ATTACHMENT_OUTPUT both written at COLOR_ATTACHMENT_OUTPUT |
| `rendered_commandbuffers` | vk_commandbuffer.cpp:426,510 | command buffers submitted per frame | XR blit cmdbuf |
| Not counted anywhere | — | number of Applies, descriptor binds, pipeline binds, pipelines alive, forced drains (`WaitForStreamBuffers`, Resize, pool exhaustion, 64 MB), recycle waits | — |

---

## 13. Branch differences

| Branch | Backend difference vs main | Evidence |
|---|---|---|
| **renderer** (175063a) | Set 1 bindings 16/17 (LevelMeshSurfaceSSO, LevelMeshLightSSO), storage pool 13→15 per set, vertex-stage SSBO count check 11→13; `EFF_LEVELMESH` guarded like the particle effects + ShaderReady gate; `LevelMeshBuffers` created/destroyed with the device; Tracy zones `VkUpdate`/`GpuWait` around `Update` and the wait (no behaviour change). StreamData/viewpoint sizes unchanged (compiled). Command buffer, render state, render pass, buffers: identical. [xc: added:] set 1 binding 12 (sector planes) is read on renderer by `levelmesh.vp:109,229-230`, so it is not dead there; `EFF_LEVELMESH` makes `MAX_EFFECTS` 11 (renderer `hw_renderstate.h:68-70`), i.e. +4 startup glslang compiles (one more effect program per pass); `vk_openxrdevice.cpp` also differs: renderer's `IsGameplaySceneActive` adds `gameaction != ga_fullconsole` (renderer :669-670) and renderer lacks main's `[STANDEYE]` `VRRig_StandEyeTic` call (main :219-223, :8355-8360) — not on the submission path. | `diff -u` of vk_descriptorset.cpp, vk_buffer.{h,cpp}, vk_shader.cpp, vk_renderdevice.cpp, vk_openxrdevice.cpp; `diff -rq src/common/rendering/vulkan` |
| **netplay** (1c1805f8) | Backend identical except `vk_openxrdevice.{h,cpp}` (input-path changes: `multiplayer` vs `VR_InputViaCommand`, controller edge copies, STANDEYE helper) — nothing on the submission path. | `diff -rq` src/common/rendering/vulkan; diff of openxrdevice (75 changed lines) |
| **selaco** (7e111a3d) | Older backend: **unbounded `vkWaitForFences(UINT64_MAX)`** (selaco vk_commandbuffer.cpp:170,253; no RS_WaitFencesPolled/journal); no scene mask; light mask keyed differently; set 1 bindings 0-13 only (no effect lights 14/15); fixed set without binding 10 (BC7 atlas); no emissive volumes; StreamData 1984 B → MAX_STREAM_DATA 33; viewpoint 24,624 B (compiled); ZVulkan without rig changes (separate present queue always requested). Same `vk_submit_size`, same single-frame model. | `diff -rq`, grep in selaco tree |

---

## 14. Open questions and the measurements that settle them

1. **How often are the capacity walls hit?** Add/read counters for: Applies per frame, `WaitForStreamBuffers` calls, `VkHardwareBuffer::Resize`, descriptor-pool-exhaustion waits, 64 MB transfer waits, recycle waits in `FlushCommands`. Run Abandon MAP13 owner list in VR (two eyes, multiview off). Settles #4/#5.
2. **What fraction of a frame is the end-of-frame wait?** Read `Finish` (flat) and `VRRenderSyncWait` + `VRSubmitWait` (VR) in r_perflog alongside GPU group totals. If wait ≈ GPU time, overlap (frames in flight) would remove roughly min(CPU, GPU) per frame (I).
3. **Is `drawcalls` dominated by the 2000 B memcpy, binds, or pipeline misses?** Profile `Apply` sub-functions (Tracy zones on renderer exist only around Update); compare with `r_skipmats`/fewer materials. Count descriptor binds per frame.
4. **Pipeline churn in VR menu transitions**: perf_track REASON_PIPELINE count when entering/leaving menus (RenderBuffersReset path, I).
5. **Upload queue family on the owner's RTX 3080 Ti**: the startup log line `Upload Queue Family: #N (...)` and whether "Asset streaming upload queues requested, but ... do not support graphics" is printed; if so, all streamed textures are created on the main thread inside `BeginFrame` (up to 100/frame).
6. **Cost of `vk_gpu_checkpoints` and the 1 ms poll granularity**: A/B with `vk_gpu_checkpoints 0`; compare `Finish`/`VRRenderSyncWait` against a build using a blocking `vkWaitForFences` with timeout.
7. **Number of live pipelines per render-pass key** (no counter exists): count `VkRenderPassSetup::Pipelines.size()` at level end.
8. **Whether the owner runs `vr_openxr_multiview 1`**: read the ini; it decides whether VR submits the scene once or twice.
9. **GPU cost of the fat UBO/fragment path**: `gpu.opaque.*` groups with r_perflog; previously measured GPU opaque 13.9 ms at Abandon spots on the old path (HANDOFF_RENDER_2026-11-11.md, build 1112i) — attribute between overdraw and shader cost with a minimal `main.fp` variant.

## Cross-check (date 2026-11-13)

Method: every cited line opened in the cited tree (main f182cbcc unless noted; renderer at abb19af1 = 175063a + docs; netplay 1c1805f8; selaco 7e111a3d). Struct sizes re-derived member by member from the headers (StreamData 2000 B, selaco 1984 B; HWViewpointUniforms 26,944 B, selaco 24,624 B; PushConstants 64 B; MatricesUBO 192 B; VkPipelineKey 64 B). Dead/dormant claims re-grepped across all of src/ and wadsrc/ (scene GLSL, PP, compute .comp, ZScript, MENUDEF) in main, renderer and netplay. Device features read from `VulkanDeviceBuilder::FindDevices` and `VulkanDevice::CreateDevice` (what is actually chained into vkCreateDevice).

| Category | Checked | Right | Line-fixed | Corrected | Unconfirmed | Added |
|---|---|---|---|---|---|---|
| a. Summary claims | 10 | 7 | 0 | 2 | 0 | 1 |
| b. "What slows it down" items (rank + evidence) | 13 | 10 | 0 | 3 | 0 | 0 |
| c. DEAD / DORMANT / never-called claims | 14 | 10 | 1 | 3 | 0 | 0 |
| d. Cvar defaults | 13 | 12 | 0 | 1 | 0 | 0 |
| e. Branch-difference claims | 5 | 4 | 0 | 1 | 0 | 0 |
| f. Table rows (§3 28 of 34; §4.1 all 6; §4.4 all 27; §6.1 all 13; §8 5; §10 8; §12 6) | 93 | 83 | 5 | 4 | 0 | 2 |
| **Total** | **148** | **126** | **6** | **14** | **0** | **3** |

Ranking in §5 left as is: the corrections change the wording of #4, #5 and #12, not their order.

Corrections (one line each):
1. Summary 4 / §10: worst case per draw is 13 vkCmd (12 from Apply + the draw), not ~12.
2. Summary 5: the 1000-Apply submit has no CPU wait, and the 9th-submit recycle wait is a one-fence wait on the frame's oldest submit, not a full GPU sync.
3. §3 / §4.3: `VkPipelineKey` is 64 B (4-byte FRenderStyle + 15 ints), not 68 B / 16 ints; `std::map` is at vk_renderpass.h:115, not :106.
4. §4.4 / §7: set 1 binding 12 is dead on main and netplay only; on renderer `levelmesh.vp:109` declares it and :229-230 read it.
5. §7: set 0 binding 3 (scene depth) is ACTIVE by default, not DORMANT: `r_gpuparticles_soft` defaults to 8.0 (`hw_cvars.cpp:311`); the "At 0, the default" comment in hw_drawinfo.cpp is stale.
6. §5 #4: the >64 MB staging wait is `WaitForCommands(false, true)` (uploadOnly): it submits only the transfer buffer and is not a full drain of the frame's recorded draws; on a worker's manager it blocks the worker.
7. §5 #5: a submit restart reloads 2 attachments (SceneColor + depth) at the default NORMAL_PASS, 4 only with `gl_ssao`; not "4 MSAA attachments".
8. §5 #12: the poll sleeps `sleep_for(1 ms)`; with the 1 ms Windows timer period each failed poll costs 1–2 ms, so "up to ~1 ms" understated (granularity I).
9. §4.1: viewpoint bound range is 53,888 B only at uniform alignment ≤64 (54,272 B at 256); SetViewpoint copies two blocks under multiview.
10. §4.2 / §5 #4: viewpoint buffer grows at the 99th upload of a frame (`mUploadIndex + count + 1 > 100`), not ">100".
11. §7: `AddBinding` is called everywhere; only its non-zero `flags` path (binding-flags pNext) is dead, alongside the bindless allocate overloads.
12. §9: the prolog has 139 `#define` indirections (83 viewpoint, 56 stream), not ~60.
13. §13 renderer row: added binding 12 being live, `MAX_EFFECTS` 11 (`EFF_LEVELMESH`, +4 startup compiles) and the `vk_openxrdevice.cpp` difference (gameaction check; no STANDEYE call), which the row's evidence list had missed.
14. Line fixes: BeginXRFrame 6006-6131 (not 6047-6130); `maxConcurrentSubmitCount` vk_commandbuffer.h:128 (not :121); VkPPRenderState::Draw :55; AcquireImage/QueuePresent vk_framebuffer.cpp:56/139; FanToTrisIndexBuffer vk_buffer.cpp:102-114; viewpoint SetViewpoint hw_viewpointbuffer.cpp:109-118.

Additions:
- Summary 6 / §6.1: descriptor indexing becomes enabled if the OpenXR runtime's required device-extension list names `VK_EXT_descriptor_indexing` (`vk_renderdevice.cpp:313-330` → chained at `vulkandevice.cpp:167-171`); runtime-dependent, I.
- §4.2: the flat vertex buffer wall (1,999,500 vertices in a frame) is `I_FatalError`, not a sync (`flatvertices.cpp:139-149`).
- §12: in VR sync mode 1 the third (desktop-mirror) wait is clocked only by `Finish`; it is in neither `VRRenderSyncWait` nor perf_track `cpu_syncwait`.

Confirmed and left unchanged, among others: StreamData 2000 B / MAX_STREAM_DATA 32 / 64,000 B blocks; 298 usable stream blocks → 9,536 Applies; matrix wall 49,998; `vk_submit_size` 1000 with flags 0; 8 submit slots; multiDrawIndirect required in `FindDevices` and no indirect draw anywhere in src/; `OptionalDescriptorIndexing` never called; drawIndirectCount / shaderDrawParameters / timelineSemaphore / shading rate never chained; multiview feature copied and chained when API ≥1.1, `vr_openxr_multiview` default false; 23 + 6 + 10 programs per pass + 4 scene-depth frags = 164 + 4U compiles; selaco unbounded `vkWaitForFences` at vk_commandbuffer.cpp:170,253.
