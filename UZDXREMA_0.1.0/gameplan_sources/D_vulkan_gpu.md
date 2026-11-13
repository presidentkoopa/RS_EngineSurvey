# D — Vulkan backend and GPU budget: fork delta for 90 Hz stereo on the modlist

Area: the fork's Vulkan backend (submission, sync, buffers, descriptors, pipelines) and the GPU budget (scene, dynamic
lights, post-process, models, the XR hand-off), plus the lighting mods the owner will ship: RS Radiance, RSQ
GlowInTheDark and a heavier REMA-native lighting mode built from them.

Trees: [F] `/home/claude/uzdxrema` @ e84623f, [FR] `/home/claude/uzdxrema_renderer` @ abb19af, [U] stock 5.0, [M]
`10_MEASUREMENTS.md`. [RAD] = `/home/claude/presidentkoopa/rs_radiance` (mod source), [GITD] =
`/home/claude/mods_x/RSQ_GlowInTheDark`. Every "today" claim was re-read at e84623f; where the survey (written at
f182cbcc) gave a line, it was checked and kept or corrected. **Inference** marks reasoning. Savings show their
arithmetic.

**Pixel assumptions (Inference).** Flat [M] = 3056 × 1689 = 5,161,584 px. VR case Q (Quest 3 over Link, about
2000 × 2000 per eye at 1.0) = 8.0 MP for both eyes = **1.55 × flat**. VR case I (Index at 100%, 2016 × 2240 per eye) =
9.03 MP = **1.75 × flat**. The tables use **S = 1.75** (the worse case). For Q, multiply the "VR today" fragment rows
by 0.89. At 90 Hz the frame is 11.1 ms. The **app GPU target is 9.0 ms for both eyes**, which leaves about 1-2 ms for
the compositor, Link encode and jitter. The fork reads the runtime's own number through `XR_META_performance_metrics`
(`XrAppGpu`, `XrCompGpu` in perf_track, [F] `src/common/rendering/hwrenderer/data/hw_perftrack.cpp:119-121`).

---

## Delta table

| ID | Item | Fork status | Where in fork | What is left | Done when (measurable) | Don't touch | Netplay/determinism impact | Est. saving on modlist (Inference, derivation) |
|---|---|---|---|---|---|---|---|---|
| P0.2 | GPU timestamps per pass | Done (flat and VR); breaks once frames are in flight | [F] `vk_commandbuffer.cpp:522-604` (writes at COLOR_ATTACHMENT_OUTPUT; read with `VK_QUERY_RESULT_WAIT_BIT` after the full wait); perf_track records every group ([F] `hw_perftrack.cpp:460`) | Give each frame slot its own query pool, read N-1 without WAIT. Add the groups that are missing: 2D, FinalizeEye, XR blit. | `stat gpu` sums to within 1 ms of `XrAppGpu` | — | none (render only) | 0 ms; it is what locates the unattributed ~12 ms (F-GPU-11) |
| P0-F3 | Vulkan always rebinds the material | Not started | [F] `hw_renderstate.h:1095-1105` sets `mChanged = true` unconditionally; [F] `vk_renderstate.cpp:478-493` binds set 0 and set 2 every time | Compare (material, clamp, translation, override) before setting `mChanged`; drop the redundant set 0 rebind | material binds per frame ≤ 20% of Applies (needs a counter) | — | none | ≈ 0.1-0.2 ms CPU per eye: 10-20% of `drawcalls` 0.93 ms [M] §1 |
| P0-F11 | Pipeline compile hitches | Partial (disk cache only) | [F] `vk_renderpass.cpp:43-60` (pipeline cache file), `:352-363` (lazy compile inside the frame, REASON_PIPELINE) | Precompile the key set behind the loading screen; stop render-scale changes dropping every pipeline (F-GPU-6) | 0 REASON_PIPELINE hitches after warm-up in a 5-minute headset run | — | none | hitches only |
| P1.1 | Frame contexts (frames in flight, N=2) | **Not started** | [F] `vk_renderdevice.cpp:722-751` (`Update` ends in `WaitForCommands`); [F] `vk_commandbuffer.cpp:430-513` waits on every fence of the frame | Per-slot fence, command pool, delete list and query pool; `BeginFrame` waits on slot k only; the end of frame submits and returns | `VRRenderSyncWait` ≈ 0 when GPU-bound; frame ≈ max(CPU, GPU) | lavapipe rig must still run ([FR] `RENDER_LANE_HANDOFF_2026-11-09.md:160-168`) | none if every ring is complete; a missed ring is a visual race, not a sim one | flat modlist: hides the whole 25.9 ms wait behind a 290 ms CPU frame (315 → ~289 ms). VR at target: frame = max(CPU ~5-7, GPU 9) instead of the sum ≈ 14-16 → **saves 5-7 ms/frame**, the difference between missing and making 90 Hz |
| P1.2 | Ring every per-frame buffer | **Not started** | Every one is single-copy: [F] `vk_renderdevice.cpp:607-611` (`pipelineNbr` default 1: `flatvertices.h:115`, `hw_lightbuffer.h:49`, `hw_bonebuffer.h:44`, `hw_viewpointbuffer.h:45`); stream and matrix ([F] `vk_buffer.cpp:38-39`); eight fork buffers ([F] `vk_renderdevice.cpp:615-640`) | Pass `pipelineNbr = 2` (the desktop cap is already 2, [F] `buffers.h:36`), call `NextPipelineBuffer` in `BeginFrame` (only GL/GLES do today: [F] `gl_framebuffer.cpp:273`), give set 1 one descriptor set per slot, and ring the fork buffers and compute host lists (F-GPU-7/8) | Synchronization validation clean in a 10-minute run | — | none | enabling only (it is what makes P1.1 safe) |
| P1.3 | Dedup: StreamData memcmp, push constants, pipeline-key hash | Not started | [F] `vk_renderstate.cpp:376-395` writes a whole StreamData (2000 B, survey-compiled) on every Apply; `:397` push constants every Apply | Compare against the last record and reuse its index. Better: split the 1,024 B `uFlatGlowLines[64]` and 384 B sweep arrays out into an indexed side table, since they change per sector, not per draw | Bytes written per Apply < 300 on average | — | none | `drawcalls` 0.93 ms flat → ×2 eyes 1.9 ms VR; −40-60% ≈ **0.8-1.1 ms/frame CPU** (assumes the memcpy into write-combined memory dominates `drawcalls`; measure) |
| P1.4 | Per-draw data in an SSBO (no 32-record blocks) | Not started | [F] `vk_shader.h:49` `MAX_STREAM_DATA = 65536/sizeof(StreamData)` = 32; [F] `vk_renderstate.cpp:495-512` rebinds set 0+1 every 32 Applies | Bind one std430 SSBO per frame slot, `uDataIndex` frame-wide | HW-buffer binds < 1% of Applies | prolog macro contract ([F] `vk_shader.cpp`, 139 indirections) | none | ≈ 2 binds per 32 Applies removed: small CPU; removes the 9,536-Apply wall (F-GPU-9) |
| P1.5 | Shrink pipeline key and precompile | Not started | [F] `vk_renderpass.h:38-59`; [F] `vk_renderpass.cpp:352-363` | As P0-F11 + dynamic state (cull/depth/stencil) | as P0-F11 | — | none | hitches only |
| P1.6 | Bindless textures | Not started (dead helper) | `OptionalDescriptorIndexing` defined, never called ([F] `libraries/ZVulkan/src/vulkanbuilders.cpp:1834`) | As playbook, with a fallback that keeps per-material batches on lavapipe | 0 set-2 binds per draw | user shaders must compile unchanged | none | CPU: removes about 2 binds per material change; GPU: none |
| P1.7 | Async uploads | Partial | Worker threads exist ([F] `vk_renderdevice.cpp:386-410`) but upload workers are refused on a transfer-only family (`:403-406`), so uploads land in `BeginFrame` | Transfer-queue uploads with ownership barriers | 0 REASON_TEXTURE on the main thread after warm-up | — | none | hitches only |
| P1.8 | PP descriptor caching, dynamic rendering | Not started | [F] `vk_pprenderstate.cpp:55-128` (new set + own render pass per pass); [F] `vk_descriptorset.cpp:484-500` | Cache sets per (pass, inputs); optional dynamic rendering | — | — | none | CPU ≈ 70 PP passes/VR frame × ~5 µs ≈ 0.35 ms |
| P2.8 (budget view) | Multiview single-pass stereo | Partial: complete but off by default, with known gaps | [F] `hw_vrmodes.cpp:643` (`vr_openxr_multiview` false); [F] `vk_openxrdevice.cpp:8608-8616`; scene once [F] `hw_entrypoint.cpp:354-372`; PP still per eye [F] `vk_pprenderstate.cpp:111-112` (`Layers = 1, ViewMask = 0`) | Close the lane's gaps ([FR] `docs/plans/RENDERER_REBUILD.md:93-105`: mirror stencil, SSAO left eye only, IPD culling margin), then layered PP | One scene build per frame; both eyes correct in RenderDoc | model and avatar draw path (render-lane rule) | none (render only) | CPU: scene build 4.35 ms × (2 − ~1.1) ≈ **3.9 ms/frame**. GPU: vertex/setup only; fragment work unchanged (≈0.3-0.6 ms, Inference) |
| P6.A | Lights Tier A: per-tic section/side light lists, GPU loop | Not started | Draw-time `SetupLights` per eye: [F] `hw_walls.cpp:600`, `hw_flats.cpp:157`; three loops [F] `wadsrc/static/shaders/glsl/material_normal.fp:85-146` | As playbook; keep the flood-fill semantics | `iter_dlight` CPU < 0.2 ms/tic; GPU ≤ 1.3 × lights-off | relink stays on the sim thread; positions from the sim, never back | **Render-only data**, but `CollectWithinRadius` runs in the tic: keep thinker order and RNG untouched (`randLight` is a client RNG, [F] `a_dynlight.cpp:42`, `m_random.h:221-225`) | CPU: per-eye `SetupLights` gone (unmeasured; `RenderWall − drawcalls`); GPU see P6.B |
| P6.B | Clustered/froxel light culling | Partial (template exists, effect lights only) | Fork effect lights: CPU-binned world grid ([F] `hw_effectlightcore.h:441` `GridForQuality`, `:526-630` `BinBuilder::Build`), read per pixel by `EffectLightsAt` ([F] `main.fp:882`) | Put `FDynamicLight` (and the REMA lighting mode's lights) through the same bins, or a compute froxel pass in combined-frustum space; one grid for both eyes | 300 visible lights at ≤ 1.8 ms both eyes (the lighting line below) | — | none | GPU: the share of opaque that is light loops (unknown; `r_dynlights 0` A/B) × ~50%. Planning value: −1.0 to −1.5 ms of the 9.3 ms VR opaque |
| P6.9 | Shadow map only when lights/geometry change | Not started | Pass runs once per frame when wanted, lingers 10 s after the last effect light ([F] `hw_entrypoint.cpp:299-316`) | Dirty flag | shadowmap GPU group = 0 on static frames | — | none | small (quality 128 × 1024 R32F) |
| P6.11 | Ray-query shadows off in VR | Done by default | `vk_raytrace` false ([F] `vk_renderdevice.cpp:100`) | Keep off in VR presets | — | — | none | 0 |
| VR8.a | Drop the scene→PP blit / resolve in-pass | Not started | [F] `vk_postprocess.cpp:131-205` (`vkCmdResolveImage`/`blitImage` per eye; layered once under multiview, `:144`) | Render into the PP input when MSAA is off; resolve attachments when on | no `VRSceneTransfer` GPU work | — | none | 9 MP × (8 B read + 8 B write) ≈ 145 MB ≈ **0.2 ms** |
| VR8.b | Fuse tonemap/colormap/present into the XR swapchain write | Not started | per eye: Pass2 ([F] `hw_postprocess.cpp:3583-3593`), then FinalizeEyeImage into an intermediate RGBA8 texture ([F] `vk_openxrdevice.cpp:7196-7240`, textures made at `:3061-3085`), then a separate blit submit ([F] `:6260-6348`) | Acquire the XR image before finalize and draw straight into its layer; drop `xrPresentTextures` and the blit | One full-res write per eye after bloom | — | none | ≈ 0.2-0.3 ms GPU (one 9 MP RGBA8 pass + blit) + the blit fence wait (F-GPU-1/2) |
| VR8.c | Bloom per layer, reduced | Partial (1/4-res chain exists; 22 passes per eye) | [F] `hw_postprocess.cpp:56-80` (level 0 = ¼ × ¼), `:1851-1912` chain, `:1918-1931` full-res additive combine | Layered PP (multiview view mask in PP keys), start at ⅛, 3 levels, fold the final combine into the present pass | bloom ≤ 0.6 ms both eyes | Radiance look/override API ([F] `hw_drawinfo.cpp:1850-1855`) | none | 6.0 → 0.6 ms (below) |
| VR8.d | Lens, FXAA, SSAO off in VR | Done by default | `gl_lens` false, `gl_fxaa` 0, `gl_ssao` 0 ([F] `hw_postprocess_cvars.cpp:472-486`); SSAO is broken under multiview anyway ([FR] `RENDERER_REBUILD.md:96`) | Force them off in the VR preset | — | — | none | 0 today; protects the budget |
| VR8.e | Mod PP shaders: per layer + VR-off switch | Partial | `gl_custompost` all-or-nothing ([F] `hw_postprocess.cpp:2862, 2955-2983`); each custom pass is a full-res draw at `mScreenViewport` ([F] `:3185-3201`); a mod can switch itself off with `PPShader.SetEnabled` ([F] `wadsrc/static/zscript/engine/ppshader.zs:28`) | Owner patches BloomBoost (off in VR) or retires it in favour of Radiance's bloom; engine: per-shader skip list cvar | BloomBoost GPU groups absent in the headset | — | none (UI-side call) | BloomBoost 3.24 ms flat × 1.75 = **5.7 ms VR → 0** |
| VR8.f | B10G11R11 scene colour | Not started | SceneColor RGBA16F ([F] `vk_renderbuffers.cpp:304-317`) | Audit alpha readers first | — | — | none | bandwidth only, ≈ 0.2-0.4 ms (Inference) |
| VR9.a | Hidden-area mesh (`XR_KHR_visibility_mask`) | Not started | No reference anywhere in [F] `src` (grep) | Draw the mask into depth/stencil before the scene, per view | 10-20% fewer shaded px (RenderDoc) | — | none | ×0.85 on every fragment-bound row: opaque 9.3 → 7.9 (**−1.4 ms**) |
| VR9.b | Fixed foveated VRS (`VK_KHR_fragment_shading_rate`) | Not started (planned stage 7) | Not requested (grep: no ShadingRate in [F] src); planned [FR] `RENDERER_REBUILD.md:198, 216, 252` | Rate attachment 1×1 centre, 2×2 periphery; force 1×1 on dither/fuzz pipelines; off on lavapipe | no visible artefacts in the headset | — | none | ×0.7 on the scene: opaque −2.4 ms at S=1.75 |
| VR9.c | Quad views | Not started | — | Only on supporting runtimes | — | — | none | larger, runtime-dependent |
| VR11 | MSAA choice / A2C | Default off | `gl_multisample` 1 ([F] `gl_renderbuffers.cpp:38`), clamped to device ([F] `vk_renderbuffers.cpp:156-184`); resolve is a separate transfer ([F] `vk_postprocess.cpp:177`) | Keep 1 for the 90 Hz preset; try 2× with in-pass resolve once the budget closes | — | — | none | MSAA 4× would add roughly +30-60% opaque (Inference); owner's ini unknown (OQ-3) |
| F-GPU-1 | **Fence waits poll with 1 ms sleeps** | Fork-only, active | [F] `vk_commandbuffer.cpp:181-221` (`sleep_for(1 ms)` at :219), used at :356, :484 and [F] `vk_openxrdevice.cpp:6370`; Windows timer period 1 ms ([F] `i_main.cpp:282-286`) | Keep the bounded deadline, but spin/yield for the first ~2 ms before sleeping, or use `vkWaitForFences` with a 1 ms timeout in a loop | `VRSubmitWait` < 0.2 ms; `VRRenderSyncWait` − GPU tail < 0.2 ms | the hang journal and the 5 s deadline | none | Each failed poll costs 1-2 ms. The XR blit and mirror waits poll right after the submit, so they almost always fail once: **≈ 1-4 ms/frame CPU in VR** |
| F-GPU-2 | XR hand-off: separate submit + CPU wait + intermediate textures | Fork-only (DoomXR) | [F] `vk_openxrdevice.cpp:6260-6379` (own command buffer, `vkQueueSubmit`, polled fence, then `xrReleaseSwapchainImage`) | Record the copy (or the finalize, VR8.b) into the frame's own submit on the same queue and release without a CPU wait. **API fact:** the runtime syncs on work already submitted to the binding's queue | `VRSubmitWait` = only `xrWaitSwapchainImage` time | — | none | 1-2 ms CPU/frame (F-GPU-1) + one submit round trip |
| F-GPU-3 | Three CPU waits per VR frame (sync mode 1) | Fork-only | [F] `vk_renderdevice.cpp:724-750` (main wait, XR blit wait, desktop-mirror submit + wait); `vr_openxr_sync_mode` 1 ([F] `hw_vrmodes.cpp:641`) | With P1.1: one wait on slot N-1 at `BeginFrame`; mirror present without a wait | 1 wait/frame | — | none | part of P1.1 + F-GPU-1 |
| F-GPU-4 | Exposure per eye, tied to `gl_bloom` | Fork/stock mix | [F] `hw_postprocess.cpp:2286-2365` (extract at ½ res, then about 12 mip passes, shared 1×1 camera texture blended per eye); gated by `gl_bloom` (:2288, default **true** [F] `hw_postprocess_cvars.cpp:37`; the playbook's "default false" is stock); exposure impulse needs it too (:3476) | Measure once per frame (eye 0 or layer 0) from bloom level 0 (already ¼ res), one compute reduction, update every 2nd frame (the 0.05 speed smooths it) | exposure ≤ 0.15 ms both eyes | Radiance writes `gl_exposure_*` ([RAD] `rsb_presets.zs:104-111`) | none | 1.38 ms flat → ~2.6 ms VR → 0.15 (**−2.4 ms**) |
| F-GPU-5 | Effects governor (frame-time cruise control) | Fork-only, Done for FX | [F] `hw_effectsgovernor.h:15-60` (rungs: smoke, flash, emissive count, particles, effect lights per bin); budget from `predictedDisplayPeriod` | Add GPU-heavy rungs (bloom levels, BloomBoost/custom PP, dynamic resolution, light-cluster cap). It needs GPU time from timestamps or `XrAppGpu` once P1.1 removes the full wait it reads today (:20-25) | in a stress run the governor holds p95 ≤ 11.1 ms | "never the owner's settings" rule | none ("presentation only", :55-58) | it keeps the tail inside budget; not a mean saving |
| F-GPU-6 | Render-scale change drops every pipeline | Fork/stock | [F] `vk_renderbuffers.cpp:186-196` (size/sample/layer change → `WaitForCommands` + `RenderBuffersReset`); `vr_openxr_render_scale` ([F] `hw_vrmodes.cpp:649`, read at [F] `vk_openxrdevice.cpp:2185`) | Allocate at the max scale and render into a sub-rect (OpenXR `imageRect`) | dynamic resolution step with 0 REASON_PIPELINE | — | none | enables dynamic resolution as a governor rung |
| F-GPU-7 | Fork compute clients rely on the full per-frame sync | Fork-only | [F] `vk_debrispool.cpp:143-145`, [F] `vk_surfacedamage.cpp:153-155` ("every frame's submissions finish before the next frame records"); smoke tile readback [F] `vk_smokevolume.cpp:1385-1450` | Ring their host-visible lists per slot; smoke reads frame N-2 or checks the slot fence | sync validation clean with smoke/debris/damage live | — | none | enabling for P1.1 |
| F-GPU-8 | Eight fork per-frame buffers single-copy | Fork-only | [F] `vk_renderdevice.cpp:615-640` (GPU particles, drawn lines, particle defs, view lights, effect lights, sector planes, mesh particles) + compute lists | Ring them (P1.2) | — | — | none | enabling |
| F-GPU-9 | Stream wall and submit chopping in VR | Fork/stock | 300 × 64,000 B ([F] `vk_buffer.cpp:39`, end test `:131-139`) → about 9,536 Applies/frame, shared by both eyes (reset once per frame, [F] `vk_renderstate.cpp:647-651`); overflow = full drain ([F] `:514-520`); submit every 1000 Applies ([F] `:43, 225-232`) with the recycle wait from the 9th submit ([F] `vk_commandbuffer.cpp:345-366`) | Short term: 1,000 blocks in system (non-BAR) memory; long term P1.4 | 0 `WaitForStreamBuffers` per frame (needs a counter) | — | none | Abandon MAP13 reaches 12-16k walls per view ([FR] `HANDOFF_RENDER_2026-11-11.md:220`) → 24-32k Applies per VR frame = 2-3 forced mid-frame drains, each a full GPU tail on the CPU: several ms (measure) |
| F-GPU-10 | `vk_gpu_checkpoints` on by default | Fork-only | [F] `vk_renderdevice.cpp:114`; a checkpoint per Push/PopGroup ([F] `vk_commandbuffer.cpp:522-560`) | A/B; turn off in the 90 Hz preset unless chasing a device loss | — | the device-loss diagnosis | none | unknown, likely < 0.2 ms (Inference) |
| F-GPU-11 | **~12 ms of the modlist GPU frame unattributed** | — | [M] §1: wait 25.87 ms (= `Finish` on the flat path, [FR] `hw_clock.cpp:165-167`); named groups 6.15 + 1.38 + 3.23 + 1.60 + 1.64 = 14.0 ms | Read the rr342 perf_track CSV, which holds every group ([F] `hw_perftrack.cpp:460, 618`): `scene.translucent`, `scene.psprites`, `scene.worldpsprites`, `fx.gpuparticles`, `fx.compute`, `pp.*`, tonemap/colormap, 2D | the GPU table below has no "unknown" row | — | — | 25.87 − 14.0 = 11.9 ms flat (GPU frame range 23.9-30.2, from ±poll overshoot and overlap with the 4.35 ms recording) → **20.8 ms VR** at S=1.75. The largest single line; nothing can be costed until it is split |
| F-GPU-12 | Fork foveation of marches | Done | `r_effects_foveated` true ([F] `hw_postprocess_cvars.cpp:391`), `r_effects_sharedmarch` | — | — | — | none | already in |
| F-GPU-13 | Render lane's static path evicts lit surfaces | Fork-only ([FR]) | [FR] `hw_levelresidency.cpp:3851, 3870` (`FB_DYNLIGHTS`) | Clustered lights (P6.B) must reach mesh surfaces too, or the static path stays on the old path wherever a light is, which on this modlist is everywhere | eviction for lights = 0% | the model/avatar path | none | precondition for the render lane's GPU/CPU gains under the lighting mods |
| F-LIT-1 | **RS Radiance** (today, reference point) | Engine support mostly Done | Plan [F] `RADIANCE_ENGINE_PLAN.md`: #2 bloom look layer Done ([F] `vmthunks.cpp:7929-7974`, consumed [F] `hw_drawinfo.cpp:1850-1855`); #1 pose smoothing Done ([F] `vmthunks.cpp:8044-8058`); #4 fog lit by dynamic lights Done (`fog_dynlights_max` 8, [F] `hw_drawinfo.cpp:1781-1790`, `uFogLightPos[8]` [F] `vk_shader.cpp:553`); #5 darkness above Done (`heightDir`, [F] `vmthunks.cpp:5120-5140`, [F] `main.fp:2261`); #3 heat outline not done (optional) | GPU: all its features are per-pixel terms in `main.fp` (darkness, sweeps ≤ 8 bands, fog slab with ≤ 8 fog lights, wall/flat glow, beams), uniform-gated in one 5,128-line program. CPU: 8 event handlers ([RAD] `mapinfo`), every one with WorldTick/UiTick | Radiance on/off A/B shows ≤ 1.0 ms both eyes at the VR preset | the Level setter API (mods call it) | the handlers run in play scope; their writes are Level render state; DDZ-style light-level writes in GITD go through net events (F-LIT-2) | today unmeasured (OQ-6). Planning: its per-pixel terms belong to the 1.8 ms lighting line |
| F-LIT-2 | **RSQ GlowInTheDark** (newer, reference point) | Mod only | Sector glow written per tic within a budget (`rsqgd_budget` 128 rooms/tic, [GITD] `zscript/rsqgd/handler.zs:98, 555-600`, `WriteSector` :517-525 → `SetGlowColor/SetGlowHeight`); liquid/item point lights capped ([GITD] `things.zs:163-172, 227`); DarkDoomZ sends a net event **every UiTick** ([GITD] `zscript/rsqgd/ddz.zs:37-39`) and rewrites sector light levels on change; self-timer `rsqgd_debug 2` ([GITD] `handler.zs:606-642`) | GPU: wall glow and GLDEFS glowing flats are cheap analytic terms; the cost is the capped extra point lights and the darkness that makes bloom do more work. CPU: up to 128 sector writes/tic; on the render lane each glow change is a light patch | glow animation done in the shader; CPU writes only on profile change | — | DDZ's `SendNetworkEvent` per tic per player is sim traffic (4 per tic in 4-player co-op): netplay lane should check it | GPU unmeasured (OQ-6); CPU per tic from its own readout |
| F-LIT-3 | **REMA-native lighting mode "at 11"** (design) | Not started | Builds on P6.B + the effect-light bins + Radiance's Level API | See §C, lighting line: clustered lights with a per-cluster cap, glow/emissive in the material, glow animated in the shader, bloom owned by the engine, quality tiers, governor rungs | 300 visible / 1,024 active lights within **1.8 ms both eyes** at the VR preset | sim never reads anything back from it | render only: lights' positions come from the sim; no sim reads | fits the 1.8 ms line by design (derivation in §C) |
| F-LIT-4 | Bloom ownership: Radiance vs BloomBoost 3.01 | Conflict today | Radiance writes `gl_bloom` true and every `gl_bloom_*` / `gl_exposure_*` ([RAD] `rsb_presets.zs:82-123`), pushes the look layer ([RAD] `rsr_looks.zs:406`) and a reactive override ([RAD] `rsb_reactive.zs:146-215`); BloomBoost is a separate GLDEFS full-res pair on top | **Radiance (engine bloom) owns bloom.** Retire BloomBoost; port its look into a Radiance preset (amount/threshold/knee/step/tint) | one bloom chain per eye | — | none | −5.7 ms VR (VR8.e) |

---

## Findings

### 1. Sync model: one frame, three waits, polled with sleeps

- **No frames in flight.** `VulkanRenderDevice::Update` ends every frame in `mCommands->WaitForCommands(...)`
  ([F] `vk_renderdevice.cpp:724-735`). `WaitForCommands` submits, presents (when `finish`), then waits on every submit
  fence of the frame ([F] `vk_commandbuffer.cpp:430-513`; `numWaitFences = min(mNextSubmit, 8)` at :452). The CPU
  never records frame N+1 while the GPU runs frame N. The render lane's audit says the same ([FR]
  `docs/plans/RENDERER_REBUILD.md:56`), but its staged plan has **no frames-in-flight stage** (stages 0-7,
  [FR] `RENDERER_REBUILD.md:259-280`). Its stage 6 (render thread off a sim snapshot) is a different thing. Its backend
  table says Vulkan has a "ring of 4 persistent buffers (`buffers.h:32-36`)" ([FR] `RENDERER_REBUILD.md:121`). That
  is wrong: 4 is the Android cap, desktop is 2 ([F] `buffers.h:30-36`), and Vulkan creates every buffer with the
  default `pipelineNbr = 1` ([F] `vk_renderdevice.cpp:607-611`). Only the GL backends rotate
  ([F] `gl_framebuffer.cpp:273`).
- **VR adds two more waits.** After the main wait comes `AcquireXRSwapchain`: its own command buffer, a blit per eye
  from `xrPresentTextures` into the swapchain layers, `vkQueueSubmit`, then `RS_WaitFencesPolled` on that fence
  before `xrReleaseSwapchainImage` ([F] `vk_openxrdevice.cpp:6260-6379`). With `vr_openxr_sync_mode` 1 (default,
  [F] `hw_vrmodes.cpp:641`) the desktop mirror then gets its own acquire, submit, present and wait
  ([F] `vk_renderdevice.cpp:742-750`).
- **The polling costs time on exactly these two waits.** `RS_WaitFencesPolled` checks `vkGetFenceStatus`, then sleeps
  1 ms between checks ([F] `vk_commandbuffer.cpp:181-221`). It exists to bound a driver hang (comment :69-80). Its
  claim "on a healthy frame the first poll succeeds" ([F] `vk_openxrdevice.cpp:6360-6363`) does not hold for the XR
  blit. The poll comes microseconds after the submit, and the GPU still has to copy 2 × 4-4.5 MP. **Inference:** the
  first check is almost always NOT_READY, so it costs one `sleep_for(1 ms)`, which is 1-2 ms with the 1 ms timer
  period ([F] `i_main.cpp:282-286`). The mirror wait behaves the same. The main wait overshoots the GPU finish by
  0-2 ms. `VRSubmitWait` clocks both the swapchain image wait and the blit poll ([F] `vk_openxrdevice.cpp:6187, 6369`),
  so perf_track's `cpu_syncwait` already shows the sum.
- **What changes it:** (a) now: spin with `std::this_thread::yield()` (or `_mm_pause`) for up to ~2 ms, then fall back
  to 1 ms sleeps until the 5 s deadline, which keeps the hang protection; (b) drop the XR blit wait altogether: record
  the copy into the frame's submit on the same graphics queue, then release the image. **API fact:** with
  `XR_KHR_vulkan_enable` the application must have *submitted* the writes to the binding's queue before release; the
  runtime orders its reads after them. (c) P1.1 frame contexts: one wait at `BeginFrame` on the slot from two frames
  ago.
- **Fork compute depends on the full sync.** Debris and surface damage write host-visible lists in place "because
  every frame's submissions finish before the next frame records" ([F] `vk_debrispool.cpp:143-145`,
  [F] `vk_surfacedamage.cpp:153-155`). The smoke volume reads last frame's tile readback on the same assumption
  ([F] `vk_smokevolume.cpp:1436-1450`). The descriptor pool for set 1 holds 10 sets ([F] `vk_descriptorset.h:132`);
  exhaustion calls `WaitForCommands(false)` ([F] `vk_descriptorset.cpp:83-92`). All of these must be ringed or
  fenced per slot before frames in flight are switched on. Keep them behind a `vk_frames_in_flight` cvar (1 = today)
  so the lavapipe rig can A/B.

### 2. Per-draw submission (CPU)

- `Apply` ([F] `vk_renderstate.cpp:223-248`) runs, on every draw: the 1000-Apply submit check (`vk_submit_size`,
  :43), `ApplyStreamData` (a full StreamData memcpy into the persistent stream buffer,
  [F] `vk_streambuffer.cpp:34-48`; the writer reads timer and vertex format first, [F] `vk_renderstate.cpp:376-395`),
  matrices, the render pass/pipeline key, dynamic state, unconditional push constants, vertex buffers, HW-buffer set
  (every 32 Applies, [F] `:495-512`) and the material (set 0 + set 2 whenever `mChanged`, [F] `:478-493`).
  `SetMaterial` always sets `mChanged = true` ([F] `hw_renderstate.h:1095-1105`), so every `SetMaterial` call
  rebinds two sets (playbook F3 not applied).
- Measured: `drawcalls` (= `Apply`) is 0.67 ms flat without mods and 0.93 ms with them on Sunder [M] §1, and
  5.0-6.0 ms on Abandon MAP13 with the owner's list on the old path ([FR] `HANDOFF_RENDER_2026-11-11.md:232`). Without
  multiview, VR doubles both: **1.9 ms and 10-12 ms per frame**. Abandon alone eats the whole 7 ms render-thread CPU
  budget.
- The stream wall: `NextStreamDataBlock` returns 0xffffffff at the buffer end ([F] `vk_buffer.cpp:131-139`) →
  `WaitForStreamBuffers` = `WaitForCommands(false)` ([F] `vk_renderstate.cpp:514-520`). The writers reset only in
  `EndFrame` ([F] `:647-651`), so both eyes share the ~9,536 slots. **Inference:** an Abandon VR frame (12-16k walls
  reached per view) drains the GPU 2-3 times mid-frame and, past 8,000 Applies, waits on the oldest submit at every
  further 1000-Apply flush ([F] `vk_commandbuffer.cpp:345-366`). Multiview halves the Apply count. The 1,000-block
  stopgap costs 64 MB of host-visible memory, which should not be in the 256 MB BAR heap.
- Descriptor indexing: `OptionalDescriptorIndexing` exists and is never called
  ([F] `libraries/ZVulkan/src/vulkanbuilders.cpp:1834`). `multiDrawIndirect` is required and unused (:1919).
  Fragment shading rate is never requested (no `ShadingRate` symbol in [F] `src`).

### 3. Post-process chain (GPU), per eye

- Order: `Pass1` = exposure, `beforebloom` custom shaders, smoke, volumetric beams, emissive volumes, heat, bloom,
  exposure impulse; `Pass2` = tonemap, colormap, lens, FXAA, `scene` custom shaders
  ([F] `hw_postprocess.cpp:3557-3593`). Each pass is its own render pass with its own descriptor set and one 3-vertex
  draw ([F] `vk_pprenderstate.cpp:55-128, 208-228`). The PP render-pass key is always `Layers = 1, ViewMask = 0`
  ([F] `vk_pprenderstate.cpp:111-112, 158-159`), although the builder supports multiview
  ([F] `vk_renderpass.cpp:640-641`). Under multiview, PP still runs once per layer: eye 1 is a
  "postprocess-only eye" that runs the whole chain again ([F] `hw_entrypoint.cpp:378-401`).
- **Exposure:** extract at ½ × ½ (R32F), then halve to 1 × 1 (about 12 passes at 3056 × 1689), then blend into a
  shared camera texture ([F] `hw_postprocess.cpp:2286-2365, 2397-2425`). It is skipped only when `gl_bloom` is off
  (:2288). **Inference:** most of 1.38 ms is per-pass fixed cost (a barrier and a render pass for each tiny level),
  so it scales with the number of eyes more than with pixels.
- **Bloom:** level 0 is already ¼ × ¼ ([F] `hw_postprocess.cpp:56-80`, `NumBloomLevels = 4` [F]
  `hw_postprocess.h:2222`): 1 extract + 7 blurred levels × 2 + 3 down + 3 up + a **full-res additive combine**
  ([F] `:1918-1931`) = 22 draws. The no-mod benchmark measured "bloom 0.5" ms ([FR] `HANDOFF_RENDER_2026-11-11.md:83`);
  the modlist 3.23 ms [M] §1. **Inference:** the 6× rise is mod-driven. Candidates: Radiance's presets set
  `gl_bloom_step` (the wide stepped blur) and anamorphic/chromatic ([RAD] `rsb_presets.zs:90-123`); pinned beam
  bloom (`gl_bloom_pin_beams`, [F] `hw_postprocess_cvars.cpp:139`) runs **two chains**
  ([F] `hw_postprocess.cpp:1800-1816`). OQ-2 settles which.
- **Custom PP (BloomBoost):** each GLDEFS PP shader is a full-res pass over `mScreenViewport`, writing the next
  pipeline image with no blend ([F] `hw_postprocess.cpp:3185-3201`). At 1.6 ms each on 5.2 MP that is about 0.31 ns
  per pixel, many taps per pixel (Inference). The only engine switch is all-or-nothing `gl_custompost`
  ([F] `:2862`). The mod can turn itself off with `PPShader.SetEnabled(name, false)` (clearscope,
  [F] `wadsrc/static/zscript/engine/ppshader.zs:28`; native [F] `hw_postprocessshader.cpp:115`), for example from
  a UiTick gated on `vr_mode`. That has no sim effect.
- **After PP, per eye:** `FinalizeEyeImage` draws the present pass into an intermediate swapchain-format texture
  ([F] `vk_openxrdevice.cpp:7196-7240`; textures [F] `:3061-3085`) plus, by default (`vr_desktop_view` 2,
  [F] `hw_vrmodes.cpp:628`), an unbiased mirror texture for one eye ([F] `vk_openxrdevice.cpp:1133-1146, 7220-7235`).
  Then the XR blit. That is three full-res writes per eye where one would do (VR8.b).

### 4. Multiview

Complete, off by default ([F] `hw_vrmodes.cpp:643`), gated per frame ([F] `vk_openxrdevice.cpp:8608-8616`). The
scene is built and drawn once with two viewpoint blocks ([F] `hw_entrypoint.cpp:453-478`), and the resolve is layered
once ([F] `vk_postprocess.cpp:131-150`). Gaps (the render lane's own audit, [FR] `RENDERER_REBUILD.md:93-105`):
mirror stencil, SSAO left eye only, culling from the left eye with no IPD margin, PP per layer. **GPU effect
(Inference):** it saves vertex and raster setup and the second set of draws. Fragment cost is unchanged, so it does
not fix the GPU budget. It fixes the CPU budget (−3.9 ms/frame on Sunder) and halves the stream wall.

### 5. Lights on the GPU

- Per-surface lists: walls ≤ 8, flats ≤ 16 candidates (`gl_light_*_candidate_budget`), uploaded per eye into one
  80,000-light SSBO. The fragment shader loops three ranges, capped by `uLightRangeLimit`
  ([F] `material_normal.fp:96-138`). A light in a surface's list is evaluated over the whole surface. PBR/specular
  materials have no range cap (survey 05 §5 GPU 1). Shadow taps: 1 at filter 0, `2·filter+1` PCF, 10 ray queries in RT
  mode ([F] `main.fp:581-760`). RT is off by default ([F] `vk_renderdevice.cpp:100`).
- The fork already has a binned system for effect lights: a CPU-built eye-centred world grid, uploaded once per frame,
  read per pixel with ≤ 32 lights per bin ([F] `hw_effectlightcore.h:441, 526-630`; [F] `main.fp:882`). This is the
  template for clustering `FDynamicLight` (P6.B) and the REMA lighting mode.
- [M] has 122 dynamic lights with mods against 56 without. The wall GPU cost went from 0.58 ms
  (`r_dynlights 0`, no mods, [FR] `HANDOFF_RENDER_2026-11-11.md:92`) to 3.09 ms. Lights versus other mod content is
  not separable from what was measured (OQ-5).

### 6. Radiance and GlowInTheDark today

- **Radiance's engine plan is mostly in the fork.** Of `RADIANCE_ENGINE_PLAN.md`'s five items, #1 (pose smoothing),
  #2 (bloom look layer), #4 (fog lit by dynamic lights) and #5 (darkness above) exist (rows F-LIT-1). Only #3's
  optional heat outline mode is missing: `func_spriteoutline.fp` has OFF/EDGE/WIRE…
  ([F] `wadsrc/static/shaders/glsl/func_spriteoutline.fp:33-35`), no `OUTLINE_HEAT`. So Radiance already is an engine
  feature set driven by a ZScript controller: 25 files, 9,879 lines ([RAD] `zscript/`), 8 event handlers
  ([RAD] `mapinfo`).
- **Its GPU cost lives inside `main.fp`, not in its own pass.** Darkness, sweep bands, fog slab with up to 8 fog
  lights (`FillFogDynLights` [F] `hw_drawinfo.cpp:1783-1790`, a per-draw-info O(lights) walk plus `nth_element`),
  wall/flat glow (64-segment flat-edge loop) and beam glow are uniform-gated terms of one 5,128-line program. With
  them off, the program still carries the register allocation of the biggest path (Inference; OQ-7). Radiance also
  forces `gl_bloom` on and rewrites exposure ([RAD] `rsb_presets.zs:82-123`), so with Radiance loaded the "`gl_bloom 0`"
  quick win is not available: bloom/exposure have to be made cheap, not removed.
- **GITD** writes sector glow colour and height within a per-tic room budget ([GITD] `handler.zs:517-525, 555-600`).
  It adds capped liquid/item point lights ([GITD] `things.zs:163-172, 227`). Its DarkDoomZ part sends a network event
  every UiTick ([GITD] `ddz.zs:37-39`) and rewrites `Sector.LightLevel` on change. On the render lane every glow
  write is a light-record patch, and the lane has already measured the per-tic light patch as a cost (2,339 sides a tic,
  [FR] `HANDOFF_RENDER_2026-11-11.md:113`). An "at 11" GITD that animates every room's glow per tic multiplies that.
  The design fix is to animate in the shader: write per-sector waveform parameters (base colour, amplitude, phase,
  period, shape) once per profile change and evaluate `glow(t)` per pixel. The cost per pixel is a few ALU ops.

### 7. Render size and dynamic resolution

The VR eye size is the runtime's recommended size × `vr_openxr_render_scale` clamped 0.25-4
([F] `vk_openxrdevice.cpp:2163-2189`), applied in `BeginFrame` ([F] `vk_renderdevice.cpp:1467-1477`). Any change of
size, samples or layers calls `WaitForCommands` and `RenderBuffersReset`, which drops every scene and PP pipeline
([F] `vk_renderbuffers.cpp:186-196`). So the scale cannot be a governor rung until targets are allocated at the max
and rendered into a sub-rect. Refresh rate can be chosen through `XR_FB_display_refresh_rate`
([F] `vk_openxrdevice.cpp:2366-2370`).

---

## C. 90 Hz contribution

### GPU budget per pass (both eyes, 90 Hz, app GPU ≤ 9.0 ms)

"Flat today" is [M] §1 (Sunder MAP15, modlist, 3056 × 1689). "VR today" scales by **S = 1.75** for fragment-bound
rows and ×2 (two eyes) for pass-count-bound rows. Multiview is off, as it is by default. "Target" assumes every step in
the last column has landed.

| Pass | Flat today (ms) | VR today (ms, derivation) | Target (ms) | Steps that get there (derivation) |
|---|---|---|---|---|
| Opaque walls + flats + masked + decals (no light loops) | 5.31 incl. lights (6.15 − 0.84 models) | 5.31 × 1.75 = **9.3** incl. lights | **2.2** | visibility mask ×0.85, VRS ×0.7, render scale 0.9 (×0.81 px), move light loops to the lighting line (see below): 9.3 × 0.85 × 0.7 × 0.81 = 4.5 incl. lights. Then shader specialisation (Radiance terms compiled out where unused) and taking the light share out (assumed ~45% of 4.5) → 2.2-2.5 |
| Models (monsters, avatar, world weapons) | 0.84 | 0.84 × 1.75 = **1.5** | **0.9** | vis mask + VRS + scale: 1.5 × 0.85 × 0.7 × 0.81 = 0.72; +0.2 headroom. The model draw path itself is not touched by the render lane; compute pre-skinning (playbook P7.8b) is the avatar lane's call |
| **REMA lighting line** (dynamic lights + glow/emissive + darkness/fog/sweep terms) | inside opaque, unknown split | inside the 9.3 | **1.8** | clustered lights (P6.B) with a per-cluster cap, glow animated in the shader, emissive in the material. Budget derivation below |
| Translucent, sprites, particles, psprites, world psprites, FX compute, 2D, tonemap/colormap (**unattributed**) | 25.87 − 14.0 = **11.9** | 11.9 × 1.75 = **20.8** (assumes fragment-bound) | **1.8** | cannot be planned until split (F-GPU-11, OQ-1); vis mask, VRS and scale apply (×0.48 → ~10); the rest has to come from the item it turns out to be (e.g. instanced/opaque-routed sprites, playbook P7.3) |
| Exposure | 1.38 | ×2 eyes ≈ **2.6-2.8** | **0.15** | once per frame from bloom level 0 (¼ res), one compute reduction, every 2nd frame |
| Engine bloom (Radiance owns it) | 3.23 | ×2 eyes ≈ **6.0-6.5** | **0.6** | one chain (pinned two-chain off in VR), start at ⅛ res, 3 levels, final combine folded into the present pass, layered PP (one pass for both layers): 6.0 × 0.5 (layers) × 0.4 (fewer, smaller levels) ≈ 1.2 → fold the combine and use the VR preset's step → 0.6 |
| BloomBoost 3.01 (2 passes) | 1.60 + 1.64 = 3.24 | 3.24 × 1.75 = **5.7** | **0** | owner retires it; its look goes into a Radiance bloom preset (F-LIT-4) |
| Finalize + mirror + XR blit + scene→PP blit | not in [M] | ≈ **0.6** (Inference: 4 full-res writes of 9 MP) | **0.35** | VR8.a, VR8.b, mirror reuse (`vr_desktop_view_openxr_render 1`) or `vr_desktop_view -1` |
| Spare (compositor jitter, spikes) | — | — | **1.2** | the governor (F-GPU-5) defends it |
| **Total** | **25.87** (GPU wait) | **≈ 46-48** (≈ 25.4 without the unattributed row) | **9.0** | |

**Reading (Inference).** On the modlist the GPU is **5× over** the 90 Hz budget in the headset, before any CPU
question. Three rows explain 37 of the ~47 ms: the unattributed 20.8, opaque 10.8 and post (14.3, of which BloomBoost
5.7). The case-Q headset (×1.55) moves the total to about 42 ms. Nothing here gets close by cvars alone.

**Stopgap only: the cvar-and-mod-patch floor (no engine code, days; for playtests until the structural plan
lands):** `vr_openxr_render_scale 0.8` (×0.64 px → S_eff = 1.12),
BloomBoost off, Radiance VR preset (bloom step 1, no anamorphic, no pinned beams), `gl_multisample 1`, `gl_ssao 0`,
`vk_gpu_checkpoints 0`, `vr_desktop_view -1`. Then: opaque 6.15 × 1.12 = 6.9; exposure + bloom at the no-mod
level × 2 eyes ≈ 2 × (0.5 + ~0.5) = 2.0 (from [FR] `HANDOFF_RENDER_2026-11-11.md:83`); unattributed 11.9 × 1.12 = 13.3
→ **≈ 22 ms**. So 45 Hz with reprojection is the cvar-only ceiling (Inference). 90 Hz needs the engine steps.

### The lighting line: what fits in 1.8 ms (design for a heavier REMA-native mode)

Derivation (Inference). At S = 1.75, scale 0.9, visibility mask and 2 × 2 peripheral VRS, the scene shades about
9.03 MP × 0.81 × 0.85 × ~0.65 ≈ 4.0 M fragment invocations, × ~1.3 opaque overdraw ≈ **5.2 M**. 1.8 ms / 5.2 M ≈
**0.35 ns per fragment** for lighting. Today's whole opaque pass costs 6.15 ms / (5.16 M × 1.3) ≈ 0.92 ns per
fragment, lights included. So the lighting line is about a third of today's per-fragment opaque cost. That buys:

- **Dynamic lights, clustered.** Average ≤ 8 lights evaluated per fragment, cap 16 per cluster at tier "11" (24 hard,
  the rest folded into one residual term, as the effect-light `BinBuilder` already does for crowded bins,
  [F] `hw_effectlightcore.h:530-630`). That supports 1,024 active / ~300 visible lights: muzzle flashes, projectiles,
  GITD liquids, lamps. Today the lists cap at 8/16 per *surface* but evaluate each light over the whole surface; a
  cluster evaluates a light only where it reaches. The side test that today rejects lights behind a wall
  ([F] `hw_walls.cpp:678`, flats [F] `hw_flats.cpp:205-209`) becomes an N·L plus a section mask (playbook Tier B step
  7), so light does not leak through walls.
- **Shadows:** at most 8 shadowed lights per frame, 1 tap (filter 0) at tier 11, 0 at low tiers. No ray queries in VR.
- **Glow and emissive in the material, not as lights.** GITD's room glow and Radiance's wall/flat glow stay analytic
  (two planes per sector, a few ALU ops). Animation moves into the shader as per-sector waveform parameters, so the CPU
  writes on profile change only (§6). Glowing flats, liquids, screens and items get an emissive term (brightmap/glow
  colour). Only a capped few become real point lights, as GITD's `ItemLightsSecond` already caps them
  ([GITD] `things.zs`, "capped lights").
- **The halo is the bloom's job, at reduced resolution.** Emissive output > threshold feeds the engine bloom chain at
  ⅛-¼ res (0.6 ms line). No separate glow pass at full res.
- **Darkness, fog slab, sweeps:** keep them as uniform-gated terms, but compile them out per variant
  (specialisation constants) so a scene without sweeps does not pay their registers. Fog lights stay ≤ 8, or ≤ 4 at
  lower tiers (`fog_dynlights_max`, [F] `hw_drawinfo.cpp:1781`).
- **Quality tiers** (Radiance menu + governor rungs, never written into the owner's cvars, per [F]
  `hw_effectsgovernor.h:50-54`):

  | Tier | cluster cap | shadowed lights | fog lights | bloom start res | fits (both eyes) |
  |---|---|---|---|---|---|
  | Low | 4 | 0 | 0 | ⅛ | ≈ 0.6 ms |
  | Medium | 8 | 2 | 4 | ⅛ | ≈ 1.0 ms |
  | High | 12 | 4 | 8 | ¼ | ≈ 1.4 ms |
  | 11 | 16 (+residual) | 8 | 8 | ¼ | ≈ 1.8 ms |

  These numbers are proportional estimates from the 0.35 ns/fragment line, not measurements.
- **Per-light cost caps:** clamp the radius by screen-space size (a light under ~2 % of the eye's width at its
  distance becomes residual-only). Keep `gl_light_max_intensity` 1000 ([F] `a_dynlight.cpp:44`) and add a VR clamp.
  Distance cull from the head centre, not the left eye.
- **CPU side of the lighting mode:** today's ZScript controllers (8 Radiance handlers, the GITD tic loop with its
  `rsqgd_budget` 128 room writes) are part of [M]'s 4.1 ms/tic of WorldTick handlers; their share is unmeasured
  (OQ-6). Target for the REMA-native version: ≤ 0.3 ms/tic of ZScript. That means a profile is set by a few Level calls
  and the per-tic animation runs in the shader. Sector/glow writes go through the render lane's change contract, never
  per tic per room.

### CPU submission budget (render thread, per VR frame; target ≤ 7 ms, aim 4)

| Item | Today VR (Inference from [M]/[FR]) | Fix | After |
|---|---|---|---|
| Scene build ×2 eyes | 4.35 × 2 = 8.7 (Sunder); Abandon 20-30 | multiview (−3.9 on Sunder); render lane's static path and culling for Abandon | ~4.8 |
| `drawcalls` (Apply) | 0.93 × 2 = 1.9 (Sunder); 10-12 (Abandon) | multiview halves it; P1.3 dedup, P0-F3, P1.4 | ~0.5 |
| Polled waits overshoot | ~1-4 | F-GPU-1/2 | ~0.1 |
| Waiting for the GPU at all | GPU tail serialised with CPU | P1.1 + P1.2 + F-GPU-7/8 | 0 when GPU-bound |
| Stream wall drains (Abandon) | 2-3 full drains/frame | multiview + 1,000 blocks, then P1.4 | 0 |
| PP descriptor/render-pass CPU | ~70 passes × ~5 µs ≈ 0.35 | layered PP + P1.8 | ~0.1 |

**Gap:** on Sunder the render thread needs ~4.5 ms out (≈ 11-13 → ≈ 5); on Abandon the render lane's static path must
land as well. The sim (250 ms/frame) is other areas' work; nothing here helps it, except that frames in flight stop the
GPU adding to it.

### Plan of record: structural fixes (setting cuts are stopgaps only)

The owner wants permanent engine solutions, so the plan of record is the structural work below. Features stay on and
get cheaper. The cvar floor above is a **stopgap** for playtesting until each structural item lands. Every cut is
undone when the item that replaces it lands.

1. **Measure first (days):** OQ-1 (split the unattributed 12 ms), OQ-2, OQ-5, OQ-6. Without them the largest row
   cannot be costed.
2. **Backend sync and submission (on `main`, behind `vk_frames_in_flight`, default 1 until the rig passes):**
   - fence polls that spin, then sleep;
   - the XR copy/finalize recorded into the frame's own submit on the same queue, released with no CPU wait;
   - **frames in flight N=2**: per-slot fences, command pools, delete lists and query pools; P0.2 timestamps read from
     N-1; every per-frame buffer ringed (P1.2), plus the fork's eight buffers and the debris/damage/smoke host lists
     (F-GPU-7/8);
   - material dedup (P0-F3);
   - StreamData split and dedup, then per-draw data in an SSBO (P1.3/P1.4), which removes the stream wall.

   This is the base everything else stands on. Add it to the render lane's staged plan, which lacks it.
3. **Stereo (render lane):**
   - make `vr_openxr_multiview` the VR default once its four gaps are closed;
   - **layered post-process**: give the PP render-pass key a view mask ([F] `vk_pprenderstate.cpp:111-112`), so each PP
     pass draws both layers in one pass;
   - the visibility mask, drawn per view.
4. **A post chain built for VR resolution (engine, owned by Radiance's bloom API):**
   - exposure measured once per frame by one compute reduction from bloom level 0;
   - one bloom chain starting at ⅛-¼ res;
   - the final combine fused with tonemap, colormap and present, written straight into the XR swapchain layer;
   - a resolve inside the render pass when MSAA is on;
   - custom PP shaders run per layer with a cost readout, and a per-shader VR switch.

   BloomBoost's look is ported into a Radiance bloom preset rather than run as two full-res passes.
5. **Engine-native lighting (render lane stage 5, shaped by the lighting line):**
   - `FDynamicLight` and the REMA lighting mode's lights clustered through the effect-light bin pipeline, or a compute
     froxel pass, with section masks so lights do not leak;
   - glow animated in the shader from per-sector waveform parameters;
   - emissive in the material;
   - shadowed lights capped per tier;
   - shader variants specialised so Radiance's terms cost nothing where unused;
   - the static mesh lit by clusters, so lights stop evicting it (F-GPU-13).
6. **Foveation:** VRS fixed, then gaze-driven (stage 7). **Dynamic resolution:** allocate at the max scale and render
   into a sub-rect (F-GPU-6), then add it as a governor rung, together with bloom levels and the light-cluster cap
   (F-GPU-5).
7. **Pipeline precompile + bindless + async uploads** (P1.5-P1.7): these remove hitches, and they come last because
   they do not move the mean.

Estimated effect once 2-6 land: the target column of the GPU table (9.0 ms) and the CPU table (~5 ms render thread),
overlapped by frames in flight. Each item's derivation is in its row.

All of it is render-only. Every step must keep `tools/netplay/run.sh <name> 2 600` IN SYNC on lavapipe with the new
paths on and off. VRS, visibility mask and update-after-bind bindless must switch off cleanly there
([FR] `RENDER_LANE_HANDOFF_2026-11-09.md:160-168`).

---

## D. Open questions (exact runs on the owner's PC)

- **OQ-1 What is the unattributed ~12 ms?** Bridge `readfile` on the rr342 perf_track CSV and summary under
  `E:\DOOMWork\_reports\render_bench\teardown_*`. Every GPU group is a column ([F] `hw_perftrack.cpp:321-323`). If the
  CSV is gone: `boottest` with `-Exe doomxr_rend1113a.exe -Zdl <57-file zdl> -Map MAP15 -Tics 700 -ExtraArgs
  "+vid_vsync 0 +vid_maxfps 0 +cl_capfps 0 +perf_track 1 +r_perflog 5"`.
- **OQ-2 Why bloom is 3.23 ms with mods against 0.5 without:** read the owner's ini (`readfile
  %USERPROFILE%\Documents\My Games\DoomXR\doomxr.ini`) for `gl_bloom_step`, `gl_bloom_pin_beams`,
  `gl_bloom_anamorphic`, `gl_bloom_amount`. Then rerun OQ-1's boottest with `+gl_bloom_pin_beams 0 +gl_bloom_step 1`.
- **OQ-3 The owner's VR settings:** the same `readfile` for `vr_openxr_multiview`, `vr_openxr_render_scale`,
  `gl_multisample`, `gl_ssao`, `vr_desktop_view`, `vr_openxr_sync_mode`, `vk_gpu_checkpoints`, `gl_custompost`.
- **OQ-4 BloomBoost's passes:** `python scripts\grep_mods.py "HardwareShader|PostProcess|Shader " E:\DOOMWork\<BloomBoost folder>`
  to see GLDEFS target (`beforebloom` vs `scene`), texture inputs and tap count.
- **OQ-5 The share of light loops in opaque:** OQ-1's boottest twice, `+r_dynlights 0` against `1`; compare
  `opaque.walls/flats/models`.
- **OQ-6 Radiance and GITD cost:** boottest with the 57-file zdl with and without RS_Radiance.pk3 (GPU groups +
  `ticprofile` WorldTick handlers); GITD with `+rsqgd_debug 2` (prints ms/tic, sector writes and live lights,
  [GITD] `handler.zs:640-642`). Also: is RS_Radiance in the 57-file list? The bridge's baseline zdl has it
  (`tools/claude_bridge/test_avatar1101.zdl` file5); the 57-file list was not available here.
- **OQ-7 `main.fp` register pressure:** an engine build that dumps `VK_KHR_pipeline_executable_properties`
  statistics for the default material program (needs a build: `build_named.py`).
- **OQ-8 Fence poll overshoot:** perf_track `cpu_syncwait` and `VRSubmitWait` against `XrAppGpu`; the owner's headset
  run only (boottest forces `vr_mode 0`, [F] `tools/claude_bridge/main_boottest/main_boottest.ps1:59`). Then the
  spin-then-sleep build A/B.
- **OQ-9 Upload queue family on the 3080 Ti:** the startup log line `Upload Queue Family` from any boottest log.
- **OQ-10 In the headset (owner, manual):** `vr_openxr_multiview 0/1` × `vr_openxr_render_scale 1.0/0.8` on Sunder
  MAP15 with `perf_track 1`. Read `XrAppGpu`, `XrCompGpu`, `cpu_syncwait`, `cpu_all` and the GPU groups per eye.
