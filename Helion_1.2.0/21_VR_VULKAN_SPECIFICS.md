# P2 — VR and Vulkan specifics: OpenXR, multiview, per-view items, pacing, budgets

This section is the API-level companion to P1 Phase 2 and Phase 8. It says where OpenXR attaches to UZDoom 5.0, what replaces the eye loop, how swapchains and depth are submitted, what multiview changes in UZDoom's shaders and passes, which scene items differ per view, and how to pace frames and spend the 90/120 Hz budget.

UZDoom 5.0 has no HMD runtime: `grep -rniE "openxr|openvr|xrCreate|xrInstance|vr::|IVRSystem"` over the repository returns nothing (U2 §8.5). Its Vulkan stereo path does not even produce two images: `FirstEye`/`NextEye` are empty virtuals that `VulkanRenderDevice` does not override [U] src/common/rendering/v_video.h:258-259, and `VkPostprocess::NextEye` is empty too [U] src/common/rendering/vulkan/renderer/vk_postprocess.cpp:358-360. Against stock 5.0, everything below is new code.

**The owner's fork is different.** His survey of the fork (`UZDXREMA_0.1.0/07_VR_OPENXR_STEREO.md` §1, §5, not re-verified here) describes an existing OpenXR Vulkan device (`vr_mode 15`, an 8.6k-line `vk_openxrdevice.cpp`). It has: a per-eye scene build; multiview that is off by default (`vr_openxr_multiview`) and does not cover post-processing; no depth layer, no visibility mask and no VRS; a fully serialized frame with a separate XR blit and 2–3 polled fence waits; and no late latch. On the fork, read every "add" below as "audit and replace". The section targets are the same.

Conventions:
- `[U]` = UZDoom 5.0.0 @ 292cf420.
- `[H]` = Helion @ 5edd437f.
- `[M] §n` = the owner's flat-screen measurements on his 5.0-based fork, in `rs_enginesurvey/UZDXREMA_0.1.0/10_MEASUREMENTS.md`.
- **API fact** = a standard OpenXR, Vulkan or GLSL fact, not taken from either repository.
- **Inference** = my reasoning.

---

## 1. Integration points: instance, device, session

**Vulkan instance and device must come from OpenXR.**
- **API fact:** with `XR_KHR_vulkan_enable2`, the application creates its `VkInstance` through `xrCreateVulkanInstanceKHR` and its `VkDevice` through `xrCreateVulkanDeviceKHR`. It must use the physical device returned by `xrGetVulkanGraphicsDevice2KHR`. The runtime injects the extensions it needs.
- UZDoom creates the instance in the platform layer: [U] src/common/platform/posix/sdl/sdlglvideo.cpp:389-401 and [U] src/common/platform/win32/win32vulkanvideo.h:57-81, through ZVulkan's `VulkanInstanceBuilder`. The actual `vkCreateInstance` call is at [U] libraries/ZVulkan/src/vulkaninstance.cpp:113.
- The device is built in `VulkanRenderDevice`'s constructor [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:112-121, which uses `VulkanDeviceBuilder::SelectDevice(vk_device)` and calls `vkCreateDevice` at [U] libraries/ZVulkan/src/vulkandevice.cpp:127.
- Change: add a `CreateFn` hook to both ZVulkan builders, so the actual create call can be routed through OpenXR. ZVulkan calls `volkLoadDevice` right after `vkCreateDevice` [U] libraries/ZVulkan/src/vulkandevice.cpp:127-130, so the hook must return the handle there. Add a `ForcePhysicalDevice(VkPhysicalDevice)` to the device builder that bypasses the `vk_device` scoring. Keep the desktop path unchanged when `vr_openxr 0`.
- Device features. Adding a feature takes three edits:
  - a struct in `VulkanDeviceFeatures` [U] libraries/ZVulkan/include/zvulkan/vulkaninstance.h:24-32;
  - the query chain [U] libraries/ZVulkan/src/vulkaninstance.cpp:215-240;
  - the copy into the enabled list [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1853-1868, and the `pNext` chain [U] libraries/ZVulkan/src/vulkandevice.cpp:92-125.
- Features to add: `multiview` (`VkPhysicalDeviceMultiviewFeatures`), `shaderDrawParameters` (`VkPhysicalDeviceShaderDrawParametersFeatures`), `drawIndirectCount`, and optionally `fragmentShadingRate` (§9).
  - **API fact:** do not chain `VkPhysicalDeviceVulkan11Features`/`Vulkan12Features` next to the per-feature structs ZVulkan already chains (descriptor indexing, buffer device address). That is invalid usage. Either stay with per-feature structs and get `drawIndirectCount` from the `VK_KHR_draw_indirect_count` extension, or migrate the whole chain to the Vulkan11/12 structs.
- Descriptor indexing (P1 Phase 1) is half there. The features are already copied when present (1864-1868), but the extension is only requested through `OptionalDescriptorIndexing()` [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1792-1796. UZDoom never calls it: the constructor calls only `OptionalRayQuery()` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:116.
- ZVulkan already tries API 1.2 first [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1688. Request 1.3 where available, to get dynamic rendering and extended dynamic state in core. Clamp the version to the range `xrGetVulkanGraphicsRequirements2KHR` reports.

**OpenXR bring-up order (API fact).** The Vulkan instance and device need the `XrSystemId`, so the order is fixed:
1. `xrCreateInstance` with these extensions:
   - `XR_KHR_vulkan_enable2` and `XR_KHR_composition_layer_depth`;
   - optionally `XR_KHR_visibility_mask` (§9);
   - optionally `XR_KHR_convert_timespec_time` or `XR_KHR_win32_convert_performance_counter_time` (clock conversion);
   - optionally `XR_FB_display_refresh_rate` (to select 120 Hz where offered);
   - optionally `XR_EXT_hand_tracking`, `XR_EXT_eye_gaze_interaction` (foveation), and vendor foveation or quad-view extensions (§9).
2. `xrGetSystem` (`XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY`), then check `XR_VIEW_CONFIGURATION_TYPE_PRIMARY_STEREO`.
3. `xrGetVulkanGraphicsRequirements2KHR`. It is mandatory: `xrCreateSession` fails with `XR_ERROR_GRAPHICS_REQUIREMENTS_CALL_MISSING` without it.
4. `xrCreateVulkanInstanceKHR`, then `xrGetVulkanGraphicsDevice2KHR`, then `xrCreateVulkanDeviceKHR` (the ZVulkan hooks above).
5. `xrCreateSession` with `XrGraphicsBindingVulkan2KHR` (instance, physical device, device, queue family, queue index).
6. Reference spaces: `LOCAL` for seated, `STAGE` (or `LOCAL_FLOOR`) for room-scale, plus a `VIEW` space. Then the action sets for the controllers.

Do this in the video-mode creation path, before the `VulkanRenderDevice` exists, because the device depends on it.

The session state machine is pumped from the main loop next to `I_StartTic`/`D_ProcessEvents` [U] src/d_main.cpp:1463-1467. Its transitions:
- `READY` → `xrBeginSession`;
- `STOPPING` → `xrEndSession`;
- `EXITING`/`LOSS_PENDING` → tear down.

**Queue sharing (API fact).** `xrBeginFrame`, `xrEndFrame`, `xrAcquireSwapchainImage` and `xrReleaseSwapchainImage` may use the `VkQueue` given in the graphics binding. Any thread that submits to that queue (P1 Phase 1 upload or submit threads) must take the same mutex as those four calls.

**What replaces the eye loop.**
- Today, `RenderViewpoint` loops over `vrmode->mEyeCount`. Per eye it runs `GetProjection`, `GetViewShift`, `SetupView`, `ProcessScene` and post-processing [U] src/rendering/hwrenderer/hw_entrypoint.cpp:126-188. The eye model is a 3D-TV model: a yaw-only lateral shift of `±½ · vr_ipd · vr_hunits_per_meter` and an asymmetric frustum from `vr_screendist` [U] src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:136-202.
- In the OpenXR mode:
  - `xrLocateViews(predictedDisplayTime, space)` returns two `XrView`s (pose + `XrFovf`).
  - Eye `i`'s world view matrix is `inverse(Body · Head_i)`.
    - `Body` is the tic-interpolated player position and the body yaw from stick or mouse (`G_AddViewAngle`/`G_AddViewPitch`, applied in the event path [U] src/d_main.cpp:3006-3022).
    - `Head_i` is the XR pose, converted from metres to map units.
    - Eye height must be counted once. In `STAGE`/`LOCAL_FLOOR` space, `Body` is at the player's feet (plus crouch and bob offsets) and the pose supplies the height. In `LOCAL` space, `Body` includes `viewheight` and the pose is relative to the recentred origin.
    - **Inference:** at 41 units/m, Doom's 41-unit view height is exactly 1.0 m. A standing user's ≈ 1.6 m eye lands at ≈ 66 units, above the 56-unit player. Offer a height-normalised mode (vertical pose scaled so that the calibrated standing height maps to `viewheight`) next to true scale.
  - Projection `i` is built from the four `XrFovf` tangents (asymmetric), replacing `VREyeInfo::GetProjection`.
- One `RenderViewpoint` call per frame then produces both views with multiview (P1 Phase 2 step 6).

**Units, pixel stretch and pitch.**
- UZDoom's view matrix scales world Z by `Level->info->pixelstretch` (default 1.2 [U] src/gamedata/g_mapinfo.cpp:312; MAPINFO `pixelratio`). The scale is applied before the camera translation [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:405-415. The playsim treats pitch as 1:1, and the renderer re-scales it [U] src/rendering/r_utility.cpp:1237-1244.
- In a headset that vertical stretch is a world distortion the user can measure with their head. Two options:
  - (a) set the stretch to 1.0 in VR, the "true proportions" option;
  - (b) keep 1.2. Apply the head pose in view space, after the stretch, where it stays metric. If the head offset is instead added to the map-space camera position, divide its Z by `pixelstretch`.

  Default to (a), and offer (b) for players who want vanilla proportions (**Inference**). Option (a) also makes a controller aim ray (playsim, 1:1) line up with what is drawn; under (b) a drawn laser pointer and the actual hitscan disagree in pitch.
- Implement the VR stretch as a renderer-side value read at every stretch site: [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:408, the sprite (un)stretch [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:369, 457-558, 1102, 1629, 1705, the skybox [U] src/rendering/hwrenderer/scene/hw_skyportal.cpp:60, models [U] src/r_data/models.cpp:169, 237, camera textures [U] src/rendering/hwrenderer/hw_entrypoint.cpp:382 and the pitch scaling in r_utility.cpp. Do **not** write `Level->pixelstretch`: ZScript reads it (`native readonly float pixelstretch` [U] wadsrc/static/zscript/doombase.zs:461, 580).
- `vr_hunits_per_meter` (default 41 [U] src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:39-45) becomes the world scale for head translation and IPD.
- Head pitch and roll must not feed the playsim's `pitch`. Gameplay aim comes from the controller pose through the tic command (P1 Phase 8 step 3).

**Near and far planes.**
- `GetZNear()` is 5 units and `GetZFar()` is 65,536 [U] src/common/rendering/v_video.h:291-292.
- At 41 units/m, 5 units is about 12 cm. That is too far for hands, weapons and leaning into walls. Use about 1 unit (≈ 2.5 cm).
- Use **reversed-Z with `D32_SFLOAT`** (or `D32_SFLOAT_S8_UINT`, since portals need stencil) and an infinite far plane. That keeps depth precision across that range (**Inference**).
- UZDoom's scene and pipeline depth is D24S8, falling back to D32S8 (defaults [U] src/common/rendering/vulkan/textures/vk_renderbuffers.h:57-58; fallback [U] src/common/rendering/vulkan/textures/vk_renderbuffers.cpp:185-196). Under reversed-Z, these change:
  - the projection builder;
  - every depth compare (`DF_Less`/`DF_LEqual` become `Greater`/`GEqual`);
  - the depth clear value (1 → 0);
  - the portal passes' "write farthest" `SetDepthRange(1, 1)`, which becomes `(0, 0)` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:180-418;
  - the sign of every `SetDepthBias`: walls [U] src/rendering/hwrenderer/scene/hw_walls.cpp:129, decals [U] src/rendering/hwrenderer/scene/hw_decal.cpp:136, 170, sprites [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:163, 284, flats [U] src/rendering/hwrenderer/scene/hw_flats.cpp:264, and [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:559.
- Keep it behind one switch, so that the desktop path can be diffed pixel for pixel.

---

## 2. Swapchains, formats and depth submission

**Colour swapchain (API fact).**
- Use one `XrSwapchain` with `arraySize = 2` and `usage = COLOR_ATTACHMENT | SAMPLED | TRANSFER_DST`. It matches multiview layer-per-view rendering directly. Alternatively use two swapchains, one per eye.
- Pick the format from `xrEnumerateSwapchainFormats`. Prefer an **sRGB** format (`VK_FORMAT_R8G8B8A8_SRGB` or `B8G8R8A8_SRGB`): the runtime then interprets the content correctly, and the final pass writes *linear* values that the hardware encodes on store.
- UZDoom's present shader applies gamma itself (`InvGamma = 1/vid_gamma` [U] src/common/rendering/vulkan/renderer/vk_postprocess.cpp:267, in `DrawPresentTexture`, lines 249-309). In VR, that pass writes into the XR image with gamma, contrast and saturation folded so that the result is linear before sRGB encode. Doing it twice is the classic "washed out" VR bug.
- 10-bit or 16-bit float swapchain formats are optional, if the runtime offers them.

**Per frame (API fact):** `xrAcquireSwapchainImage` → `xrWaitSwapchainImage` (before the GPU writes) → record → `xrReleaseSwapchainImage` (after submit) → `xrEndFrame`.
- The swapchain images are `VkImage`s owned by the runtime. Wrap them in `VkImageView`s (2D array, 2 layers) at swapchain creation.
- Treat them as the final post-process target, in place of the desktop swapchain image that `AcquireImage` provides today. `AcquireImage` also triggers the present pass [U] src/common/rendering/vulkan/textures/vk_framebuffer.cpp:48-77.
- **Layouts (API fact):** an acquired image is in `COLOR_ATTACHMENT_OPTIMAL` (depth images: `DEPTH_STENCIL_ATTACHMENT_OPTIMAL`), and it must be back in that layout when released. The post-process swapchain render pass today goes `UNDEFINED → PRESENT_SRC_KHR` [U] src/common/rendering/vulkan/renderer/vk_renderpass.cpp:416-417. Add an XR variant whose initial and final layout are `COLOR_ATTACHMENT_OPTIMAL`.

**Depth submission with `XR_KHR_composition_layer_depth` (API fact).**
- Chain an `XrCompositionLayerDepthInfoKHR` onto each `XrCompositionLayerProjectionView`. It carries the depth swapchain sub-image, `minDepth`/`maxDepth`, and `nearZ`/`farZ`.
  - `nearZ`/`farZ` are the distances **in metres** of the planes at `minDepth` and `maxDepth`. `nearZ > farZ` marks reversed depth, and either may be `+inf`.
  - For reversed infinite Z: `minDepth = 0`, `maxDepth = 1`, `nearZ = +inf`, `farZ = zNear_units / vr_hunits_per_meter`.
  - Passing map units here is the typical bug.
- The depth swapchain must be a format the runtime lists, typically `D32_SFLOAT` or `D24_UNORM_S8_UINT`.
- Runtimes can use submitted depth for positional reprojection and for compositor features such as passthrough occlusion and space-warp.
- The scene depth is MSAA when `gl_multisample > 1` [U] src/common/rendering/vulkan/textures/vk_renderbuffers.cpp:42-62. Resolve it into the depth swapchain image through `VkSubpassDescriptionDepthStencilResolve` (**API fact:** `VK_KHR_depth_stencil_resolve`, core in 1.2), or through `VkRenderingAttachmentInfo` with dynamic rendering. Constraints (**API fact**):
  - `VK_RESOLVE_MODE_SAMPLE_ZERO_BIT` is the only mode every device must support; `MIN`/`MAX` are optional (`supportedDepthResolveModes`).
  - The resolve target needs the same depth bits and numeric type as the source. `D32_SFLOAT_S8_UINT` → `D32_SFLOAT` works; `D24S8` → `D32_SFLOAT` does not (use a copy shader).
  - The subpass path needs `vkCreateRenderPass2`, but ZVulkan's `RenderPassBuilder` calls `vkCreateRenderPass` [U] libraries/ZVulkan/src/vulkanbuilders.cpp:1430.
- Depth must hold the 3D scene, weapon and hands included (they are scene objects in VR, §4). The HUD and 2D layer is a separate composition layer.

**Desktop mirror.** Keep the existing `VulkanSwapChain` (2 images with vsync, 3 without [U] src/common/rendering/vulkan/textures/vk_framebuffer.cpp:61) as an optional mirror:
- blit the left eye, cropped, from the app-owned post-process image (layer 0), or from the XR image in the same submission *before* `xrReleaseSwapchainImage`. After release the app may not touch the XR image (**API fact**);
- present after `xrEndFrame`, with `IMMEDIATE` (the existing mode table [U] libraries/ZVulkan/src/vulkanswapchain.cpp:108-134; windowed `MAILBOX` is commented out there);
- never present in a way that blocks the VR loop: acquire with a zero timeout, and skip the mirror frame if no image is available.

These are all disabled in VR, because `xrWaitFrame` owns pacing:
- the FPS limiter, `FPSLimit` [U] src/common/rendering/v_framebuffer.cpp:243-278, called only without vsync [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:176-178;
- `r_ticstability` [U] src/d_net.cpp:2102-2122;
- `TryRunTics`' own wait, `I_WaitForTic` when `cl_capfps` is set [U] src/d_net.cpp:2200-2209.

---

## 3. Multiview: images, passes and shader changes

**Images.** Every scene-resolution target becomes a 2-layer image with 2D-array views:
- `SceneColor`, `SceneDepthStencil`, `SceneFog`, `SceneNormal`, the two `PipelineImage`s and `PipelineDepthStencil`. Today all are single-layer [U] src/common/rendering/vulkan/textures/vk_renderbuffers.cpp:87-253.
- Size them from `xrEnumerateViewConfigurationViews` → `recommendedImageRectWidth/Height`, times `vid_scalefactor` [U] src/common/rendering/r_videoscale.cpp:171.

**Passes.** Every scene pass is created with `VkRenderPassMultiviewCreateInfo { viewMask = 0b11, correlationMask = 0b11 }`, or uses `VkRenderingInfo::viewMask` with dynamic rendering (**API fact**). The scene render-pass builder [U] src/common/rendering/vulkan/renderer/vk_renderpass.cpp:176-226 and the post-process render-pass builder (lines 413-437) both gain the view mask. Framebuffers keep `layers = 1` and take 2-layer array views (**API fact**). UZDoom clears through render-pass load ops (`VkRenderState::Clear` ends the pass [U] src/common/rendering/vulkan/renderer/vk_renderstate.cpp:135-139), and a load-op clear covers every view in the mask. One recorded draw rasterises into both layers. Viewport and scissor are shared by all views, so both eye images must have the same size.

**Shader changes in UZDoom's GLSL** (`#extension GL_EXT_multiview : require`, `gl_ViewIndex`, **API fact**). The same `wadsrc/static/shaders/glsl` sources also feed the GL and GLES backends (`gl_shader.cpp`, `gles_shader.cpp`). Put the extension and the indexing behind a Vulkan-multiview define, with index 0 elsewhere. `HWViewpointUniforms` is shared with GL too.

| Item | Where today | Change |
|---|---|---|
| `ProjectionMatrix`, `ViewMatrix`, `NormalViewMatrix` | `ViewpointUBO` [U] src/common/rendering/vulkan/shaders/vk_shader.cpp:191-210; used in [U] wadsrc/static/shaders/glsl/main.vp:72, 114, 119, 128 | Become `[2]` arrays indexed by `gl_ViewIndex` in the VS. `HWViewpointUniforms` [U] src/common/rendering/hwrenderer/data/hw_viewpointuniforms.h:32-56 must match (std140). The record grows by 3 × 64 B plus the new vectors. Slot alignment and the dynamic-UBO range follow `sizeof` automatically ([U] src/common/rendering/hwrenderer/data/hw_viewpointbuffer.cpp:30, [U] src/common/rendering/vulkan/renderer/vk_descriptorset.cpp:86). |
| `uCameraPos` | [U] wadsrc/static/shaders/glsl/main.fp:277, 348, 813, 878; fogboundary.fp:45; material_specular.fp:56; material_pbr.fp:93 | Split by use. Light-band readers (277 `R_WallColormap`, 348 `R_DoomLightingEquation`) use the **cyclops** position (next row). Fog (813, 878, fogboundary.fp:45) uses the cyclops position too: cheaper and consistent. Specular and PBR (`V = normalize(camera − pos)`) **must** be per view: a `[2]` array read through `gl_ViewIndex`, which the FS can use under multiview. |
| Doom light diminishing distance | `R_DoomLightingEquation` uses `pixelpos.w` (eye-space depth) or `distance(pixelpos, uCameraPos)` [U] wadsrc/static/shaders/glsl/main.fp:342-353; `pixelpos.w` is written at main.vp:87; fog also uses `pixelpos.w` (main.fp:809, 874; fogboundary.fp:41) | Compute from the **cyclops** position and forward vector (new `uCyclopsPos`, `uCyclopsFwd`): `z = dot(pos − uCyclopsPos, uCyclopsFwd)`, so both eyes pick the same light band. **Inference:** with parallel eye axes, eye-space depth is already equal in both eyes. It differs on HMDs that report canted views, and the distance mode (348) always differs, by up to half the IPD (≈ 1.3 units). The band edges then sit in different places per eye. The cyclops form is exact and free. |
| Clip planes (`uClipLine`, `uClipHeight`) | main.vp:134-144 | World-space, identical for both views. No change. |
| `gl_ClipDistance[5]` always written | main.vp:157-166 | Unchanged. Keep `shaderClipDistance`. |
| Software fuzz | `gl_FragCoord`, `uViewHeight` [U] wadsrc/static/shaders/glsl/fuzz_software.fp:55-58 | The screen-space pattern differs per eye. Use angular, cyclops-relative noise (P1 Phase 7 step 5). |
| Sky | `HWSkyPortal::DrawContents` → `SetupView(state, 0,0,0, …)` [U] src/rendering/hwrenderer/scene/hw_skyportal.cpp:54 | Both views use rotation-only view matrices (the eye offsets are dropped) and their own projection. |
| Model interpolation, bones | main.vp:61-70 | Unchanged. Pre-skin in compute (P1 Phase 7) so multiview does not skin twice. |
| SSAO and post-process | `hw_postprocess` shaders sample 2D textures; SSAO needs the inverse projection | Sample `sampler2DArray` with `layer = gl_ViewIndex` in multiview post-process passes. Per-view projection constants. |

**Push constants and StreamData** need no per-view data: one draw serves both views.

**Portals and stencil under multiview.**
- `DrawPortalStencil` / `SetupStencil` / `RemoveStencil` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:180-418 draw portal polygons with stencil increment and decrement and depth restore. Under multiview each layer has its own stencil and depth, and the same draw rasterises the polygon from each eye. Per-eye correct masks therefore come for free.
- The portal *scene* is built once (P1 Phase 2 step 9). Its views are `ViewMatrix[i] × M_portal`: the mirror reflection, or the line-portal displacement and rotation. Mirrors flip winding for both views together.
- The "first sky portal without stencil" shortcut [U] src/rendering/hwrenderer/scene/hw_portal.cpp:113-155 still works.
- Portal recursion depth costs GPU time per eye pixel, not CPU per eye.

---

## 4. HUD, 2D, menus and the weapon

- **2D layer.** Today `Draw2D` runs inside `VulkanRenderDevice::Update` into the post-process target [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:201-222. `F2DDrawer` merges commands [U] src/common/2d/v_2ddrawer.cpp:211-224, and the vertices are uploaded once per frame [U] src/common/rendering/hwrenderer/hw_draw2d.cpp:85.
  - In VR, render the 2D drawer into an offscreen RGBA image (for example 1920×1080, alpha-cleared) once per frame, or once per tic when the sim-thread HUD of P1 Phase 8 is used.
  - Submit it as an **`XrCompositionLayerQuad`** (**API fact**), head-locked about 1.5 m ahead for the HUD (in `VIEW` space) and world-locked for menus. Set `XR_COMPOSITION_LAYER_BLEND_TEXTURE_SOURCE_ALPHA_BIT`, plus `UNPREMULTIPLIED_ALPHA_BIT` unless the offscreen image holds premultiplied alpha. Layers are composited in array order, so put the quad after the projection layer.
  - The compositor reprojects it at display rate, so text stays sharp.
  - That covers the status bar, `RenderOverlay`/`RenderUnderlay` output, menus, the console and intermission.
- **Weapon.** The 2D psprite path (`DrawPlayerSprites` [U] src/rendering/hwrenderer/scene/hw_weapon.cpp:114, drawn with an orthographic HUD projection, `GetHUDSpriteProjection` [U] src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:120-129) has no meaning in a headset. Render the weapon as a world-space model, or as a sprite quad parented to the controller pose, inside the scene passes. HUD models currently need a depth clear before drawing [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:918-924. In VR they become ordinary scene objects with the near plane at about 1 unit.
- **Screen blends** (damage flash, pickup tint, invulnerability colormap): apply them in the final pass as a uniform per frame, identical for both eyes. Optionally dim them, because full-field flashes are uncomfortable in VR.

---

## 5. Per-view items: what must agree between the eyes and what must differ

The rule:
- **decisions** (which frame, which variant, which order) are made once, from the **cyclops** position, so both eyes agree;
- **geometry** (projection and parallax) is computed per view in the vertex shader.

Splitting a decision per eye produces binocular rivalry: two different images that the brain cannot fuse (**Inference**, standard VR practice).

Once the scene is built at tic rate (P1 Phase 3), a cyclops decision taken at tic time lags head motion for up to a tic. Head-dependent decisions must then either run per frame, or be made conservative over the tic's head envelope (P1 Phase 3 C). Per frame means in the VS for rotation frames and billboard yaw, and in the per-frame cull for sort order, slope facing and render-hack variants.

| Item | UZDoom 5.0 code | Per-view treatment |
|---|---|---|
| Sprite rotation frame | `(thingpos − vp.Pos).Angle()` [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:969 | Cyclops (CPU or VS). Both eyes show the same frame. |
| Billboard corners | [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:1120-1175; modes `gl_billboard_mode`/`gl_billboard_faces_camera` (82-83) | Orientation from the cyclops (position-facing yaw), so the quad is one planar surface for both eyes. Projection per view. |
| Anamorphic sprite pull (`gl_spriteclip -1`) | [U] src/rendering/hwrenderer/scene/hw_sprites.cpp:1190-1238 | Cyclops. A per-eye pull would put the sprite at different depths per eye. |
| Translucent sort | `ViewDistance` [U] src/rendering/hwrenderer/scene/hw_walls.cpp:592-595; `DoSort`/`Sort` with `SortZ = Viewpoint.Pos.Z` [U] src/rendering/hwrenderer/scene/hw_drawlist.cpp:629-714; the side test in `DrawSorted` (line 882) | One order from the cyclops for both eyes. Additive styles need no order (P1 Phase 7). |
| Missing-texture render hacks | `Planez > Viewpoint.Pos.Z` [U] src/rendering/hwrenderer/scene/hw_renderhacks.cpp:543 | Decision from the cyclops Z, so both eyes agree. Under head roll the eyes' Z values differ. The flood fill's ray-plane intersection is *per view* (correct parallax). |
| Sloped-plane facing | `plane.ZatPoint(vp.Pos)` vs `vp.Pos.Z` [U] src/rendering/hwrenderer/scene/hw_flats.cpp:519, 577 | Cyclops (conservative: draw both faces when the cyclops is within ε of the plane). |
| Heightsec (Boom 242) view zone | `hw_FakeFlat` [U] src/rendering/hwrenderer/scene/hw_fakeflat.cpp:208ff | Cyclops zone. A zone change forces a scene rebuild (P1 Phase 3). |
| Clipper and portal windows | `FrustumAngle` [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:377-397; `ClearClipper` [U] src/rendering/hwrenderer/scene/hw_portal.cpp:427-456 | The **union** of both eyes: dual clipper (P1 Phase 2 step 5). |
| Sky | rotation-only `SetupView(0,0,0)` [U] src/rendering/hwrenderer/scene/hw_skyportal.cpp:54 | Rotation shared, projection per view. Skybox viewpoints go through the cubemap option (P1 Phase 5 step 4). |
| Fog boundaries | extra translucent wall with `EFF_FOGBOUNDARY` [U] src/rendering/hwrenderer/scene/hw_walls.cpp:120-135 | Geometry per view. Fog distance from the cyclops for consistency. |
| Doom light bands | [U] wadsrc/static/shaders/glsl/main.fp:342-353 | Cyclops distance (§3). |
| Coronas | Disabled in 5.0.0. The per-corona `Trace` and `CoronaFade` update are compiled out (`#if 0` [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:865-892), and the `DrawCoronas` call is commented out (lines 912-915). | Nothing to do today. If re-enabled: trace once per frame from the cyclops and advance the fade once per frame. A per-eye run would double the fade speed and write an actor field from the renderer (a Phase 8 hazard). |
| Dither transparency | `P_CheckSight` from the renderer [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:1049-1055 | Once per tic, on the sim thread (P1 Phase 5). |
| Fuzz | [U] wadsrc/static/shaders/glsl/fuzz_software.fp:55-58 | Angular noise, cyclops-relative. |
| SSAO | per scene [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:1034-1038 | Per view (screen-space by nature). Off by default in VR (§8). |
| Specular and PBR view vector | material_specular.fp:56, material_pbr.fp:93 | **Per view.** Highlights must have parallax. |
| HUD and 2D | `Draw2D` | Not in the eye images: a quad layer (§4). |

---

## 6. Frame pacing: `xrWaitFrame`/`xrBeginFrame` against the 35 Hz tic

**Today.** `D_DoomLoop` runs `TryRunTics()` and then `D_Display()` serially [U] src/d_main.cpp:1463-1467. `TryRunTics` runs as many tics as are due, so it catches up [U] src/d_net.cpp:2312-2335, and busy-waits when none are due (`TicStabilityWait`, line 2275). `TicFrac` comes from `I_GetTimeFrac`, using the frame *start* time [U] src/common/utility/i_time.cpp:170-177, sampled at `RenderView` [U] src/rendering/hwrenderer/hw_entrypoint.cpp:343-344. Nothing predicts when photons appear: there is no `present_wait` or present id (U2 §6).

**OpenXR frame loop, single-thread form** (before P1 Phase 8):
```text
loop:
  xrWaitFrame(&frameState)          // blocks to compositor cadence; gives predictedDisplayTime (T), period
  SetFrameTime(T_engine)            // see "Time base" below
  TryRunTics()                      // the tics due by T (cap at 2 per frame; log overruns)
  xrBeginFrame()
  views = xrLocateViews(T)          // poses for the predicted display time
  t = I_GetTimeFrac()               // now on T's grid
  render (one multiview pass) ; xrAcquire/Wait/Release swapchain images
  xrEndFrame(T, layers = [projection(+depth), HUD quad])
```
If `frameState.shouldRender` is false, still call `xrBeginFrame`/`xrEndFrame`, with zero layers.

**With the sim thread** (P1 Phase 8): the render thread runs exactly the loop above without `TryRunTics`. The sim thread ticks on its own clock and publishes snapshots.
- **API fact:** `xrWaitFrame` may be called from a different thread than `xrBeginFrame`/`xrEndFrame`. Concurrent `xrWaitFrame` calls must be externally synchronized. A second `xrWaitFrame` blocks until the previous frame has been begun. An `xrBeginFrame` issued before the previous frame's `xrEndFrame` discards that frame.
- Pipelining therefore works like this: call `xrWaitFrame` for frame N+1 once `xrBeginFrame(N)` has run, typically when frame N's commands are submitted. The CPU work of N+1 then overlaps the GPU work of N, and `xrBeginFrame(N+1)` follows `xrEndFrame(N)`. That is P1 Phase 1's two frames in flight.

**Time base and tic fraction.**
- Convert `XrTime` to the engine clock once per frame (**API fact:** `xrConvertTimeToWin32PerformanceCounterKHR` / `xrConvertTimeToTimespecTimeKHR`).
- In VR, `I_SetFrameTime` [U] src/d_main.cpp:1461 stores the converted, monotonic-clamped T instead of "now". Then `I_GetTime` (the tics due, which `TryRunTics` runs) and `I_GetTimeFrac` [U] src/common/utility/i_time.cpp:165-177 both use T's grid.
- Use one `t` for both eyes.
- **Why both, not only the fraction.** T is typically 1–3 frame periods ahead of "now". If only the fraction used T while tics still ran on "now", T would regularly fall in a tic slot that has not been simulated yet. `t` would then wrap to the start of that slot while the interpolation endpoints are still the previous tic pair, and the world would jump backwards for a frame.
- **With the sim thread:** compute `t` against the newest published snapshot's `ticStartNs` and clamp it to [0, 1]. If T is past the newest snapshot (the sim is late), hold at 1 and log it; do not extrapolate.
- **Inference:** demos and netplay are unaffected. Only the wall-clock schedule of tics moves (a constant offset of the prediction horizon); the tic commands and their order do not change.
- Interpolation still shows the world between tic N−1 and tic N, as today. The world latency of up to one tic does not change, but motion becomes uniform in *display* time, which removes the judder that comes from sampling at frame start.

**Head pose versus tic.** Head pose never waits for a tic. The view is `Body(t) · Head(T)`. Gameplay aim reads the controller pose at tic time through the tic command (P1 Phase 8 step 3). Late latching re-queries `xrLocateViews(T)` just before submit and writes the matrices into the frame's UBO slot (P1 Phase 8 step 6).

**Overrun policy.**
- If the render thread misses `T`, the runtime reprojects. Do not stack frames: always render for the newest `predictedDisplayTime`.
- In the single-thread form, a tic that runs past the frame budget produces a missed frame. That is why the measured with-mods tic (130 ms of thinkers per tic [M] §2) has to come down first (P1 Priorities), and why P1 Phase 8 exists.

**Per-frame `shouldRender = false`** (for example the headset is off): skip rendering but keep the frame calls (above) and keep ticking. Pause the game as a menu would in these cases:
- the session drops from `FOCUSED` to `VISIBLE` (a system overlay owns input);
- `STOPPING`/`IDLE`.

---

## 7. Frame budgets at 90 and 120 Hz

**Inference.** The figures assume a PC VR headset at roughly 2,000 × 2,200 per eye at 1.0× supersampling (about 4.4 MP per eye) and a compositor overhead of about 1 ms GPU (runtime-dependent).

| Budget | 90 Hz | 120 Hz |
|---|---|---|
| Frame period | 11.1 ms | 8.33 ms |
| App GPU, after compositor margin | ≤ 9.5 ms | ≤ 7.0 ms |
| Render thread CPU, per frame | ≤ 7 ms (target 4) | ≤ 5.5 ms (target 3) |
| Sim tic, on the sim thread (35 Hz) | ≤ 28.6 ms hard, ≤ 20 ms target | same |
| Sim tic, single-thread form (tic lands in one frame) | ≤ 3 ms | ≤ 2 ms |
| Pixels shaded per second, both eyes, 1.0× | ≈ 0.8 GP/s | ≈ 1.06 GP/s |

**Reading against the owner's numbers [M] §1, §2:**
- The no-mod flat frame is 13.9 ms: think 2.5, script 3.1, scene build 3.8, GPU wait 3.2 [M] §1. That already exceeds the 90 Hz period before stereo.
  - The think and script clocks overlap: with mods, script (258 ms) exceeds think (250 ms) inside a 315 ms frame. Do not add them.
- Phase 1 removes most of the GPU wait (frames in flight), Phase 2 keeps the scene build at one build, and Phase 3 moves it to tic rate.
- The no-mod tic is ≈ 4.5–5.2 ms: thinkers 3.24–3.79 plus interpolation reset 1.25–1.39 [M] §2. That is already above the ≈ 3 ms single-thread budget at 90 Hz, which is why the sim thread is required.
- With mods, the tic is far beyond even the 28.6 ms hard limit (130.8 ms of thinkers on Sunder MAP15 [M] §2). Nothing renderer-side helps until P1 Phases 9 and 10 bring it down.
- GPU, with mods, flat (3056 × 1689 ≈ 5.2 MP): opaque 6.15 ms [M] §1. **Inference**, scaling linearly in pixels: at 8.8 MP for both eyes that is ≈ 10 ms, three times the opaque budget below. Phases 4–6, MSAA cost control and VRS (§9) have to recover it. Re-measure in the headset ([M] §5 lists it as not measured).

**GPU budget allocation (Inference), 90 Hz at 1.0×, per frame:**

| Pass | Budget |
|---|---|
| Opaque world | 3.0 ms |
| Masked and sprites | 1.0 ms |
| Portals | ≤ 1.5 ms |
| Lights | 1.5 ms |
| Translucency | 0.5 ms |
| Post-process | ≤ 0.8 ms |
| Spare | 1.2 ms |

The owner's with-mods GPU numbers already show 1.4 ms of exposure, 3.2 ms of bloom and 3.2 ms of the BloomBoost mod's two passes on a flat screen [M] §1. Exposure runs only when `gl_bloom` is on [U] src/common/rendering/hwrenderer/postprocessing/hw_postprocess.cpp:432-437. That 7.8 ms is post-processing he would have to drop or reduce in VR (§8).

---

## 8. Post-process trimming at VR resolution

The per-scene chain today. `VulkanRenderDevice::PostProcessScene` [U] src/common/rendering/vulkan/system/vk_renderdevice.cpp:262-266 calls the blit, then `VkPostprocess::PostProcessScene` [U] src/common/rendering/vulkan/renderer/vk_postprocess.cpp:76-108, which runs `Pass1`/`Pass2` [U] src/common/rendering/hwrenderer/postprocessing/hw_postprocess.cpp:1196-1213:
1. A full-resolution blit or MSAA resolve from `SceneColor` to `PipelineImage[0]` (`BlitSceneToPostprocess`, vk_postprocess.cpp:110-165).
2. Exposure and bloom if `gl_bloom` (default false), plus `beforebloom` custom shaders.
3. `DrawEndScene2D` (the weapon).
4. Tonemap (`gl_tonemap` 0, or MAPINFO), colormap, lens (`gl_lens` false), FXAA (`gl_fxaa` 0) and `scene` custom post-process shaders.
5. The present pass (`DrawPresentTexture`, vk_postprocess.cpp:249-309), reached from `AcquireImage` (§2).

SSAO (`gl_ssao` 0) runs inside `DrawScene` (U2 §5.3). Every post-process draw ends the render pass and allocates a descriptor set [U] src/common/rendering/vulkan/renderer/vk_descriptorset.cpp:197-231.

**VR defaults and changes:**
- **Drop the scene→pipeline blit.** With MSAA off, render straight into the post-process input. With MSAA on, resolve inside the render pass with resolve attachments (**API fact**) instead of `vkCmdResolveImage`/blit. **Inference:** at 2 × 4.4 MP × RGBA16F that is about 70 MB copied (≈ 140 MB of read plus write traffic) saved per frame.
- **Fuse tonemap, colormap, gamma folding and the final sRGB write into one pass** writing the XR swapchain image, per layer or with multiview.
- **Bloom off by default.** If enabled, run it per layer (multiview) at quarter resolution.
- **Lens distortion off.** The compositor already applies the HMD lens correction, so UZDoom's lens pass is meaningless here.
- **FXAA off.** It shimmers temporally in VR; use MSAA (§11).
- **SSAO off by default.** If on, use half resolution per layer.
- **Mod post-process shaders** (GLDEFS `HardwareShader PostProcess`, like the owner's BloomBoost): run them per layer, with a per-mod "VR off" switch and a cost readout in the Tracy GPU zones.
- **Use FP16 `SceneColor` only when needed.** Without bloom or tonemap, `B10G11R11_UFLOAT` halves the bandwidth (**Inference**). **Open risk:** the format has no alpha channel. Audit every blend state and shader that reads destination or scene alpha before switching.

---

## 9. Foveated rendering

Pick per GPU and runtime. All of these are **API facts** about availability; the costs and gains are **Inference**.

- **Visibility mask first (`XR_KHR_visibility_mask`).** This is the cheapest saving and needs no special hardware.
  - Get each view's hidden-area mesh, and draw it into depth (at the near-plane value, which is 1 under reversed-Z) or into stencil before the scene. Use the mesh selected by `gl_ViewIndex` under multiview.
  - The lens-occluded corners are then rejected by early-Z.
  - Re-fetch the mesh on `XrEventDataVisibilityMaskChangedKHR`.
  - **Inference:** about 10–20% of the eye pixels on typical PC lenses.
- **`VK_KHR_fragment_shading_rate` (VRS) with a shading-rate attachment.** It needs the `attachmentFragmentShadingRate` feature: NVIDIA Turing and later, AMD RDNA2 and later, Intel Xe. It is the recommended default for PC VR.
  - Build a rate image with 1×1 in the centre, 2×2 in the periphery, and 4×4 at the edges of the lens.
  - A 2-layer rate image (one layer per view under multiview) needs `layeredShadingRateAttachments` (**API fact**). Without it, use one shared, symmetric pattern for both eyes.
  - Use a fixed pattern, centred per eye on the lens centre. With `XR_EXT_eye_gaze_interaction`, centre it on the gaze.
  - **API fact:** coarse fragment size × MSAA samples is limited by `maxFragmentShadingRateCoverageSamples`. With MSAA 4× (§11), 2×2 is usually the largest usable rate.
  - **Inference:** a 25–40% reduction in fragment cost, which matters most for the dynamic-light-heavy and post-process-heavy cases.
  - Doom's low-resolution textures, magnified up close, tolerate coarse peripheral shading well; distant, minified detail shows it more. Keep the HUD and text out of it (they are a separate layer), and render the weapon at 1×1.
- **`VK_EXT_fragment_density_map`.** Mostly found on mobile and Qualcomm (standalone headsets). On PC it is less common than VRS. Support it behind the same abstraction if a standalone port is planned.
- **Quad views.** `XR_VARJO_quad_views` (in OpenXR 1.1 core as `XR_VIEW_CONFIGURATION_TYPE_PRIMARY_STEREO_WITH_FOVEATED_INSET`), with `XR_VARJO_foveated_rendering` for gaze, renders 4 views: 2 wide at low resolution and 2 narrow insets at high resolution.
  - Multiview layers share one image size and one viewport, and the inset and outer views have different recommended sizes. So record **two** multiview passes (outer pair, inset pair, `viewMask = 0b11` each) from one CPU scene build, culled against the outer frusta.
  - This is the largest win on runtimes that support it.
- **Interaction with UZDoom shaders.**
  - The `gl_FragCoord`-based software fuzz is screen-space and changes under coarse rates. Use the angular version (P1 Phase 7), or force 1×1.
  - The ordered dither of `DITHERTRANS` indexes its 8×8 table by `pixelpos.xy` [U] wadsrc/static/shaders/glsl/main.fp:920-938. That is **world** X and height (`pixelpos.xyz = worldcoord`, main.vp:86), so it is already identical in both eyes. Under coarse rates, its 1-unit world pattern is undersampled and aliases. Force 1×1 for that pipeline variant.

---

## 10. Motion vectors and space-warp (optional)

- **Motion vectors are almost free after P1 Phase 3.** Every vertex already carries prev/cur tic state. Add the previous *frame's* `uTicFrac` and view-projection, and output `clipPrev` as a varying. That gives screen-space motion for world geometry, movers, sprites and models.
- **API facts.** Application space-warp is exposed by `XR_FB_space_warp` on Meta runtimes. It takes a motion-vector image and a depth image per view and lets the app render at half rate, with the compositor synthesising the in-between frames. Runtime-generated reprojection needs no app work: Meta's PC ASW uses submitted depth (§2) for its positional component, and SteamVR Motion Smoothing works without it. **Open risk:** whether `XR_FB_space_warp` is exposed on PC over Link depends on the runtime version. Check the extension list at run time.
- **Recommendation (Inference).** Build motion vectors (cheap, and needed for TAA if wanted). Expose app space-warp as an option for weak GPUs. Never rely on it for the 90 Hz target: synthesised frames make artefacts around Doom's sprite edges and alpha-tested textures.

---

## 11. MSAA versus TAA; texture filtering

- **MSAA 4× is the VR default** (**Inference**). Doom geometry is low-polygon with high-contrast edges. MSAA is temporally stable, has no ghosting under head motion, and is cheap at Doom's shading cost. UZDoom already supports MSAA scene buffers (`gl_multisample`, default 1 [U] src/common/rendering/gl/gl_renderbuffers.cpp:38; Vulkan [U] src/common/rendering/vulkan/textures/vk_renderbuffers.cpp:42-62). Resolve in-pass (§8).
- **Alpha-to-coverage** for masked mid-textures and opaque sprites. Today masked geometry uses alpha-test `discard` in non-NAT shader variants [U] wadsrc/static/shaders/glsl/main.fp:858-860, which crawls in VR. A2C gives smooth edges under MSAA and keeps early-Z for opaque geometry, because NAT variants remain separate pipelines.
- **TAA:** only with motion vectors (§10) and per-eye history. It costs blur and ghosting under constant head motion and disocclusion artefacts at sprite edges. Treat it as an option for high-resolution headsets where MSAA is too expensive.
- **Texture filtering.** The 5.0 default is already mode 6: linear minification with linear mips and **nearest magnification**, at 16× anisotropy (`gl_texture_filter` 6, `gl_texture_filter_anisotropic` 16 [U] src/common/rendering/hwrenderer/data/hw_cvars.cpp:127, 132; mode table [U] src/common/rendering/vulkan/textures/vk_samplers.cpp:44-53). So minification is handled.
  - The VR problem is magnification: up close, hard texel edges crawl under sub-pixel head motion.
  - Offer mode 4 (full trilinear), or better a "sharp pixel" shader mode: a nearest look with analytically antialiased texel edges (UV snapping with `fwidth`).
  - Bindless (P1 Phase 1) makes this a per-material sampler index.

---

## 12. Checklist of VR-only correctness traps

1. **Gamma applied twice** (sRGB swapchain plus `InvGamma` in the present shader) → washed out.
2. **Per-eye decisions** (rotation frames, sort order, light bands, render-hack variants) → rivalry. Use the cyclops (§5).
3. **The 46° pitch heuristic in `FrustumAngle`** [U] src/rendering/hwrenderer/scene/hw_drawinfo.cpp:385-388 is wrong for a headset. Use the exact combined-frustum extent (P1 F10).
4. **A near plane of 5 units** clips hands and walls. Use about 1 unit with reversed-Z (§1).
5. **Pixel stretch 1.2** distorts the world and misaligns the aim ray. Default to 1.0 in VR (§1).
6. **HUD drawn into the eye images** → unreadable and uncomfortable. Use a quad layer (§4).
7. **Depth submitted with the wrong near/far, units or convention** (map units instead of metres; `nearZ`/`farZ` not swapped for reversed-Z) → reprojection warps. Validate it with the runtime's debug overlay.
8. **`r_ticstability` busy-wait, `FPSLimit` or `I_WaitForTic` still active** → wasted CPU, fighting `xrWaitFrame` (§2). [M] §4 measured the busy-wait at 2.3 ms per frame on a flat screen.
9. **Tic fraction on T, tics on "now"** → the world jumps backwards for a frame whenever T crosses a tic boundary first (§6).
10. **XR swapchain image in the wrong layout at release, or touched after release** (the desktop mirror) → validation errors or corrupted frames (§2).
11. **Eye height counted twice** (`viewheight` plus the floor-relative head height) → a giant's view (§1).
12. **The camera-texture and skybox cost** paid per eye. Camera textures already render once outside the eye loop [U] src/rendering/hwrenderer/hw_entrypoint.cpp:375-386, so keep it that way. Skyboxes: cubemap option.
13. **Coronas.** Dead code in 5.0.0 (§5). If someone re-enables them, run them once per frame, not per eye.
