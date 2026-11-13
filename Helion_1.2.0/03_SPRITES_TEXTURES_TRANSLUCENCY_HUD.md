# H3 — Helion entity/sprite rendering, texture management, models, HUD/2D

Helion draws sprites as **GPU-expanded instanced quads**. The CPU writes one 40-byte instance per visible actor, and **only when the game tic changes** (35 Hz). On every render frame in between, the same buffers are drawn again. The vertex shader interpolates `prevPos→pos` with `timeFrac` and builds the billboard corners from `gl_VertexID`. Sprites are pre-packed at map load into five power-of-two `GL_TEXTURE_2D_ARRAY` size buckets (32²–512²). Batching is by (render style × array texture), so opaque sprites usually take a handful of `glDrawArraysInstanced` calls, plus one call for each sprite that has a brightmap. Translucency uses **weighted-blended OIT**, so there is no CPU sort, but each translucent bucket is rasterised twice and each fuzzy bucket up to four times. Palette mode stores indices in RGBA8 textures and looks up colormaps in the shader. In true-colour mode at this revision, no code path applies per-actor translations (see §8). The engine has **no 3D model support** at all: `Core/Models` holds save-game DTOs. The HUD/2D path batches runs of consecutive same-texture quads, but it re-uploads with `glBufferData` once per batch.

---

## 1. Per-tic CPU collection of visible actors

**What runs when.** `PerformRender` computes `renderTickChange`. The entity visibility walk and all instance writes run only when it is true. The flag is true when `world.GameTicker` differs from the last tic rendered (or the render mode / transfer-height view changed).

```csharp
private bool NeedsRenderTickChange(IWorld world, TransferHeightView view) => m_lastTicker != world.GameTicker || m_renderStatic != m_lastRenderStatic || m_lastTransferHeightsView != view;
...
var renderTickChange = !m_config.Developer.Render.Lock.Value && NeedsRenderTickChange(world, renderInfo.TransferHeightView);
...
if (renderTickChange) { ... if (m_renderStatic) IterateBlockmap(world); else TraverseBsp(world, renderInfo); ... }
```
`[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:143, 310-341`

`GameTicker` is incremented once per `WorldBase.Tick()` `[H] Core/World/WorldBase.cs:1042-1049, 1097-1099`. The entity buffers are cleared only on a tic change: `Clear(...)` calls `m_entityRenderer.Clear(world)` only `if (renderTickChange)` `[H] LegacyWorldRenderer.cs:789-799`.

**Two traversal modes feed `RenderEntity`:**
- *Static/blockmap mode*: walks `world.RenderBlockmap` blocks inside a box of `MaxDistance` around the interpolated view position (`Render.MaxDistance = 0`, the default, is mapped to `Constants.DefaultMaxDistance = 6000` units `[H] Core/Render/Renderer.cs:347-349`; `[H] Core/Util/Constants.cs:432`). It skips blocks behind a half-plane (`BlockInView`) and iterates each block's `HeadRenderEntities` linked list `[H] LegacyWorldRenderer.cs:145-178, 198-215`.
- *BSP mode*: walks BSP front-to-back with two 2D view clippers (current and previous position). Entities are emitted per sector when that sector is first reached, de-duplicated through `BlockmapCount == CheckCount` `[H] LegacyWorldRenderer.Bsp.cs:33-71`.

**Per-actor CPU culling** in `LegacyWorldRenderer.RenderEntity` covers: invisible frame or flag, `NoSector`, the viewer itself (unless chase-cam), `RenderStyle.None`, the half-plane occluder test against `OccludePos`, and a squared 2D distance test against `MaxDistanceSquared`. It also writes `entity.RenderDistanceSquared` and `entity.LastRenderGametick` back into the simulation object `[H] LegacyWorldRenderer.cs:271-293`. The occluder sits 32 units behind the camera along its yaw, and it is disabled when |pitch| > 45° `[H] LegacyWorldRenderer.cs:773-787`. Frustum culling exists only in BSP mode (`m_frustumPlanes.BoxInFront`) `[H] LegacyWorldRenderer.Bsp.cs:19, 78-79`.

**Depth-bias index.** Each block or sector hands out `renderIndex` values 0..99 (`EntityRenderIndexMax = 100`) `[H] LegacyWorldRenderer.cs:34, 168-173`; `[H] LegacyWorldRenderer.Bsp.cs:62-70`. The vertex shader multiplies this index by a small depth bias, so coplanar sprites do not z-fight `[H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityProgram.cs:168-171, 184, 217-222`.

**Inference:** CPU sprite work therefore runs at 35 Hz no matter the display rate. At 120 Hz, about 70% of frames do zero CPU work for sprites, and on a stereo frame the second eye costs nothing on the CPU. The flip side is that visibility, the rotation frame and light level are frozen for up to 28.6 ms. The occluder half-plane and the BSP clipper both use the camera yaw from the tic frame. A fast yaw turn within one tic can therefore show blocks or sectors that were rejected at the tic and have no sprite instances. Nothing in the code re-runs the walk on camera rotation (only the conditions in `NeedsRenderTickChange` do). On a desktop this is masked by low turn rates, but in VR (head yaw at 300°/s ≈ 8.6° per tic) it would show as sprite pop-in at the view edges.

## 2. Instance format and GPU quad expansion

**Instance struct, 40 bytes**, `Pack = 1`; every attribute has `divisor: 1`:

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct EntityVertex
{
    [VertexAttribute(divisor: 1)]
    public Vec3F Pos;

    [VertexAttribute(divisor: 1)]
    // X offset written to surface option portions when in health bar mode
    public float SurfaceOptions;

    [VertexAttribute(divisor: 1)]
    public Vec3F PrevPos;

    [VertexAttribute(divisor: 1)]
    public float OffsetXYZ;

    [VertexAttribute(divisor: 1)]
    public float RenderOptions;

    [VertexAttribute(divisor: 1)]
    public float TextureInfo;
}
```
`[H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityVertex.cs:7-28`. Packed layouts: `SurfaceOptions` = alpha 8 | fuzz 1 | flipU 1 | light 8 | colormap 14; `OffsetXYZ` = two sign bits + 14-bit XY offset (<<16) + 14-bit Z offset; `RenderOptions` = light-buffer index 20 | renderIndex 12; `TextureInfo` = array layer 12 | width 10 | height 10 `[H] VertexOptions.cs:37-73`; `[H] EntityProgram.cs:116-137`. Bit packing (an int reinterpreted as a float): `[H] Core/Render/OpenGL/Renderers/Legacy/World/VertexOptions.cs:37-73`. The divisor is applied through `GL.VertexAttribDivisor` `[H] Core/Render/OpenGL/Vertex/Attributes.cs:191`.

**Draw.** Each bucket issues `glDrawArraysInstanced(TriangleStrip, 0, 4, instanceCount)` `[H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderData.cs:61-63`; `[H] Core/Render/OpenGL/Buffer/VertexBufferObject.cs:32-38`. The vertex shader builds the corner from `gl_VertexID`:

```glsl
vec3 posMoveDir = vec3(viewRightNormal, 0);
vec3 offsetXY = vec3(posMoveDir.xy * offsetXYOption, 0);
vec3 interpolatedPos = mix(prevPos, pos, timeFrac);
...
float xSelect = float(gl_VertexID & 1);     // 0,1,0,1
float ySelect = float(gl_VertexID >> 1);    // 0,0,1,1
float x = mix(minPos.x, maxPos.x, xSelect);
float y = mix(minPos.y, maxPos.y, xSelect);
float z = mix(minPos.z, maxPos.z, ySelect);
```
`[H] EntityProgram.cs:142-159`. `minPos`/`maxPos` come from `interpolatedPos`, the X/Z sprite offsets, and `textureDim / textureSize(boundTexture,0)`. The texture size is the bucket's layer size, so UVs cover only the sprite's sub-rectangle of the padded layer `[H] EntityProgram.cs:204-214`.

The billboard is **view-plane-aligned around world Z**: the quad spans `viewRightNormal`, a uniform set from `Camera.Direction.XY.RotateRight90()` on every program bind `[H] Core/Render/OpenGL/Renderers/Legacy/World/Entities/EntityRenderer.cs:368-369`. There is no pitch or roll billboarding, and no Y-billboarding toward the camera position.

The vertex shader also fetches the sector colormap and fog from buffer textures, using the light index packed in `RenderOptions` `[H] EntityProgram.cs:123-125, 140, 182`; `[H] Core/Render/OpenGL/Renderers/Legacy/World/Shader/SectorColorMap.cs:15-29`.

**Inference:**
- This is effectively the "GPU-generated quad via instancing" design. There is no geometry shader: `grep -rn "override string? GeometryShader" Core` returns nothing, and `SkySphereGeometryShader` is only a class name.
- CPU→GPU traffic is 40 B per sprite per tic, against UZDoom's 4 × 32 B = 128 B per sprite per frame (§Transfer notes). That is ~31% per upload; at 120 Hz it is ~9% of UZDoom's per-second vertex traffic (40 × 35 vs 128 × 120 B/s), and ~4.5% if UZDoom rebuilds per eye in stereo. This is before counting UZDoom's per-sprite state.
- Vertex shader cost is negligible: 4 invocations per sprite with a handful of texel fetches.

## 3. Batching, buckets and draw calls

**Bucket key = (render style, array texture).** `RenderEntity` maps the sprite's texture to its parent array (`texture.ParentArrayTexture ?? texture`). It then calls `m_dataManager.GetByRenderStyle(renderStyle, arrayTexture, brightmapTexture)` `[H] EntityRenderer.cs:278-280`. `RenderDataManager` keeps two families of `RenderDataCollection`, one per `RenderDataStyle` (Normal, Fuzzy, Translucent, Add, ColorAdd, FogBarrier):
- index 1 is **Pinned**, for array textures;
- index 0 is **Recycle**, for standalone textures.

The decorate `RenderStyle` enum folds into those buckets through a static lookup table `[H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderDataManager.cs:12-22, 28-40, 74-79`.

```csharp
private RenderData<TVertex> GetStatic(GLLegacyTexture texture, GLLegacyTexture? brightmapTexture)
{
    if (!m_allRenderData.TryGetValue(texture.TextureId, out var data)) { data = m_renderDataPool.Get(...); ... }
    if (data.RenderCount != m_renderCount) { m_dataToRender.Add(data); data.RenderCount = m_renderCount; }
    return data;
}
```
`[H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderDataCollection.cs:89-105`. The lookup is a `LookupArray` indexed by the GL texture name, so there is no hashing. "Recycle" collections return their `RenderData` to the pool on `Clear` `[H] RenderDataCollection.cs:45-60`.

**Each bucket owns its own VAO+VBO** (`RenderData` builds a `VertexPipeline` with a `DynamicVertexBuffer`) `[H] RenderData.cs:25-30`. `Draw()` runs, in order: bind the colour texture on unit 0, bind the brightmap (or 0) on unit 12, bind the VAO and VBO, `Upload()`, `DrawArraysInstanced`, unbind `[H] RenderData.cs:48-67`. `Upload()` is a no-op once `Uploaded` is set `[H] Core/Render/OpenGL/Buffer/BufferObject.cs:94-101`. The flag is reset by `Clear()`, which happens once per tic `[H] BufferObject.cs:141-145` (and by `Add()` `[H] BufferObject.cs:90-91`, which `RenderEntity` bypasses by writing `ArrayData.Data[length]` directly `[H] EntityRenderer.cs:282-314`). The actual upload is a full `glBufferData(size = count × 40)` with `DynamicDraw` (orphaning) `[H] Core/Render/OpenGL/Buffer/ArrayBufferObject.cs:24-29`.

**Pool sizes (pre-allocated):**
- array pool: 64 `RenderData` × capacity 8192;
- overflow pool: 512 × capacity 128;
- health bars: one more 8192-capacity buffer.

`[H] EntityRenderer.cs:30, 61-64`; `[H] RenderDataManager.cs:39`. When a pool runs dry, `Get` constructs a new `RenderData` (a new VAO+VBO) in the middle of the frame `[H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderDataPool.cs:45-58`. `UpdateTo` refills the overflow pool to 128 entries at map load `[H] EntityRenderer.cs:97-100`.

**Draw-call count per frame.** Opaque pass: one draw per non-empty (Normal × array bucket), plus one per distinct standalone sprite texture visible (sprites with brightmaps, §9) `[H] RenderDataManager.cs:81-85`; `[H] RenderDataCollection.cs:107-117`. The opaque path in `RenderOpaque` binds `EntityProgram`, sets ~35 uniforms, and draws `RenderDataStyle.Normal` (plus an optional health-bar program) `[H] EntityRenderer.cs:353-424`. The translucent passes are in §6.

**Inference:** For vanilla Doom 2 content, all sprites without brightmaps fall into at most five array buckets, or more if a bucket exceeds `GL_MAX_ARRAY_TEXTURE_LAYERS`, which is 2048 on many GPUs. The opaque sprite pass is then ≤5 draws regardless of monster count. Per-sprite CPU cost is a few dozen loads and stores plus bit packing, with no virtual calls inside `RenderEntity` and no per-draw texture handle hashing. The pre-allocated CPU arrays are large: 64 × 8192 × 40 B ≈ 21 MB for the array pool alone (`DynamicArray` allocates `new T[capacity]` eagerly `[H] Core/Util/Container/DynamicArray.cs:60-69`). GPU buffers start at zero size because they are sized on upload.

## 4. Interpolation between tics

- **Position.** `TickEntities` copies `PrevPosition = Position` before moving each entity `[H] Core/World/WorldBase.cs:1167-1182`. `RenderEntity` writes both positions into the instance `[H] EntityRenderer.cs:292-297`. The vertex shader computes `mix(prevPos, pos, timeFrac)`; `timeFrac` is set from `renderInfo.TickFraction` once per program bind `[H] EntityProgram.cs:144`; `[H] EntityRenderer.cs:367`. Interpolation is therefore entirely on the GPU and costs nothing on the CPU per frame.
- **Ceiling-spawned sprites** in vanilla-render mode get `PrevPos.Z` from `Ceiling.PrevZ` `[H] EntityRenderer.cs:303-311`.
- **Not interpolated:** angle (so the rotation frame), light level, alpha, offset, texture. All of them are frozen at the tic, because the instance is rebuilt only then (§1).
- **Weapon (HUD) sprite** is interpolated on the CPU every frame. `GetInterpolatePlayerWeaponBox` lerps the previous and current weapon/bob offsets by `m_lastTickInfo.Fraction` `[H] Core/Layer/Worlds/WorldLayer.Render.Hud.cs:600-626`.

**Inference:** For VR this is the correct split. Positions stay smooth at any refresh rate, and both eyes read identical instance data.

## 5. Rotation frame choice, mirroring, offsets

Rotation is chosen **on the CPU, once per tic**, with BAM-style integer math:

```csharp
const uint SpriteFrameRotationAngle = 9 * (uint.MaxValue / 16);
return unchecked((viewAngle - entityAngle + SpriteFrameRotationAngle) >> 29);
```
`[H] EntityRenderer.cs:115-137`.

- `viewAngle` is `ViewClipper.ToDiamondAngle(viewPos → entity)`, using the view position interpolated at the tic frame `[H] EntityRenderer.cs:200-206`; `[H] LegacyWorldRenderer.cs:292`.
- `+MIRROR` flips both the rotation index and `flipU` `[H] EntityRenderer.cs:224-243`.
- The texture comes from a cached `SpriteDefinition` (`m_spriteDefs` dense array keyed by sprite index), then `Rotations[frame, rotation]` `[H] EntityRenderer.cs:182-198, 166-176`.

`SpriteDefinition` holds a fixed `SpriteRotation?[29, 8]` table. It parses frame/rotation from lump name characters 5–8, and rotation 0 fills all 8 slots `[H] Core/Resources/SpriteDefinition.cs:12-15, 22-45, 74-89`. Mirrored pairs set `FlipU`, which the shader uses to swap U `[H] Core/Resources/SpriteRotation.cs:7-16`; `[H] EntityProgram.cs:160-163`.

`GetOffsetZ` applies floor-clip rules for corpses on the CPU: `SpriteClip`, `SpriteClipMin`, `SpriteClipFactorMax`, blank-row trimming and `VanillaRender` bypass `[H] EntityRenderer.cs:139-164`.

The rotation index has 3 bits (`>> 29`), and `MaxRotations = 8`. **16-rotation sprites are not supported**: the parser takes `Name[5] - '0'` and returns on `rotation >= 8` `[H] SpriteDefinition.cs:13, 34, 84-86`. There is no `SpriteAngle`/`SpriteRotation` actor property in the renderer: `grep -n "SpriteAngle\|SpriteRotation\b" EntityRenderer.cs` finds only the type name.

## 6. Render styles: opaque, translucent (weighted-blended OIT), additive, fuzz

**Style resolution** happens on the CPU in `RenderEntity` `[H] EntityRenderer.cs:249-280`:
- shadow/fuzzy → `Fuzzy`;
- with `SpriteTransparency` off, everything else → `Normal`;
- `ColorAddFullBright` → `ColorAdd` when fullbright, otherwise `Translucent`;
- `ColorAddExplosion` → `Normal` for missiles, otherwise `ColorAdd`;
- `Translucent` with alpha ≥ 1 → `Normal`.

The `RenderStyle` enum has only None/Normal/Fuzzy/Translucent/Add/ColorAdd plus the two pseudo styles `[H] Core/Resources/Definitions/Decorate/Properties/Enums/RenderStyle.cs:3-16`.

**Opaque pass** (`EntityProgram`, `OitOptions.None`): binary alpha test, `a = a > 0.5 ? 1 : 0`, discard ≤ 0. It writes depth, and the distance fade is applied after `[H] EntityProgram.cs:405-421`.

**Translucent pass = weighted-blended OIT.** `RenderTransparent` binds the OIT FBO and clears it. The FBO has three attachments:
- `accum`: RGBA16F;
- `accumCount`: RG16F;
- `fuzz` mask: R8.

It shares the opaque depth texture, blends with `One, One`, and turns depth writes off `[H] LegacyWorldRenderer.cs:586-601`; `[H] Core/Render/OpenGL/Framebuffer/OitFrameBuffer.cs:40, 48, 56, 61-64, 72-85`. The accumulate shader writes a distance-weighted premultiplied colour:

```glsl
float weight = max(exp(-dist3D * 0.0007), 0.16);
...
accum = vec4(min(fragColor.rgb, vec3(colorClamp)) * fragColor.a, fragColor.a) * weight * weightClear;
accumCount = vec2(fragColor.a * weightClear, 1 * weightClear);
```
`[H] Core/Render/OpenGL/Renderers/Legacy/World/Shader/FragFunction.cs:345-355`.

**Composite is not a full-screen quad.** `StartRenderOitCompositePass` + `RenderCompositeStyles` **re-draws the sprite instance buckets** (Translucent, then Add with `SrcAlpha,One`, then ColorAdd with `SrcColor,One`). The composite shader reads `accum`/`accumCount` at `gl_FragCoord` and writes the averaged colour `[H] LegacyWorldRenderer.cs:662-666, 697-736`; `[H] FragFunction.cs:360-378`.

**Fuzz (spectre/partial invisibility)** uses up to four passes over the Fuzzy bucket:
1. In the OIT transparent pass, fuzzy fragments write `outFuzz = 1` but contribute zero colour (`weightClear = mix(1,0,fuzzFrag - renderFuzz)`) `[H] EntityRenderer.cs:426-438`; `[H] FragFunction.cs:348-352`.
2. If `PostProcessingEffects` is on, the **refraction pass** draws into the main framebuffer. It samples `opaqueTexture`, which is that same framebuffer's `ColorAttachment0`, at noise-offset pixel blocks `[H] LegacyWorldRenderer.cs:603-616`; `[H] OitFrameBuffer.cs:92-102`; `[H] FragFunction.cs:252-283`. The block size is `fuzzDiv / max(1, dist/96)`.
3. `RenderOitTransparentFuzzPass` accumulates the fuzzy sprites into OIT with `renderFuzz = 1` `[H] LegacyWorldRenderer.cs:618-622`; `[H] EntityRenderer.cs:440-448`.
4. The final `RenderOitFuzzRefractionPass(renderColor: true)` blends a darkened, noise-masked colour, re-reading accum/count itself `[H] LegacyWorldRenderer.cs:690-691`; `[H] FragFunction.cs:284-315`.

Noise is the hash function `rand/noise` from a gist, animated by `fuzzFrac`. That value is `((GameTicker/4) % 8) + 1`, so it steps per tic `[H] FragFunction.cs:76-93`; `[H] Core/Render/Renderer.cs:285-294`.

**Inference:**
- WBOIT removes all CPU sorting. That is a real CPU win over UZDoom's BSP-split sprite sort (§Transfer notes).
- On the GPU it costs ~2× rasterisation for translucent sprites and ≤4× for fuzzy ones. It also needs ~13 bytes/pixel of extra render-target storage (8 + 4 + 1) at native resolution (RGBA16F + RG16F + R8), which doubles in stereo.
- The refraction pass samples a texture that is attached to the bound draw framebuffer. The only guard is `glMemoryBarrier(FRAMEBUFFER_BARRIER_BIT)` `[H] LegacyWorldRenderer.cs:609-610`. That is a GL feedback loop with undefined results; it would need `glTextureBarrier` or a copy. In Vulkan it must become an explicit copy or input attachment.
- WBOIT is order-independent but only approximate. Stacked additive explosions and heavy overlap will not match the software renderer.

## 7. Fullbright and brightmaps

- **Per-actor fullbright** (`+BRIGHT` flag or frame `Bright`, unless the brightmap says `disablefullbright`) forces `lightLevel = 255` into the instance on the CPU `[H] EntityRenderer.cs:245-246, 287`.
- **Per-texel fullbright (colormap fullbright indices).** `PaletteToArgb` stamps alpha = 254 (`Image.AlphaFlag`) on texels whose palette index is marked fullbright `[H] Core/Graphics/Image.cs:24, 244-270`. In true-colour mode, the shader checks for `a == 0.99607…` to force light to 1 `[H] FragFunction.cs:95-106`. In palette mode the same flag means "colormap-fetch this texel" `[H] FragFunction.cs:152-161`.
- **Brightmaps (GLDEFS `brightmap` only).** In true colour: `rgb *= min(1, brightmap.rgb + lightLevel)` `[H] FragFunction.cs:225-238`. In palette mode, a grey brightmap lowers `lightColorIndex` `[H] FragFunction.cs:166-171`. Sprites with brightmaps are **excluded from texture arrays**, so each becomes a standalone texture and a separate draw `[H] Core/Render/OpenGL/Texture/Legacy/GLTextureArrayBuilder.cs:201-205, 103-109`.
- GLDEFS parsing recognises only `#include`, `brightmap`, `skybox` `[H] Core/Resources/Definitions/ZDoom/GldefsDefinition.cs:80-84`.

**Inference:** In brightmap-heavy mods (for example retro-brightmap packs) every brightmapped sprite frame becomes a separate draw call and texture bind. That defeats the bucket batching.

## 8. Palette/colormap emulation and translations

**Two global modes, switched by `ShaderVars.PaletteColorMode`.** The value is set from `Window.ColorMode`, whose default is `TrueColor` `[H] Core/Render/OpenGL/Renderers/Legacy/World/Shader/ShaderVars.cs:3-10`; `[H] Core/Render/Renderer.cs:257-264`; `[H] Core/Util/Configs/Components/ConfigWindow.cs:103-105`.

**Palette mode (indexed textures + shader lookup).**
- *Upload format.* Textures are uploaded as RGBA8 with R = palette index and A = 254/0. `GetGlTexturePixels(true)` builds a temporary `uint[]` per image on every upload `[H] Image.cs:520-534`; `[H] Core/Render/OpenGL/Texture/Legacy/LegacyGLTextureManager.cs:125, 156`.
- *Colormap table.* The fragment shader fetches from one big `samplerBuffer` holding RGBA32F per (colormap × 14 palettes × 34 light levels × 256 indices):
  ```glsl
  int texIndex = lightLevelOffset + (useColormap * colormapSize) + (usePalette * paletteSize + int(fragColor.r * 255.0));
  vec3 fetchColor = texelFetch(colormapTexture, texIndex).rgb;
  ```
  `[H] FragFunction.cs:153-161`; buffer build `[H] Core/Render/ColorMapBuffer.cs:9-23`; upload `[H] Renderer.cs:237-254`.
- *Which colormap.* For entities, the per-instance 14-bit colormap index (translation / Boom colormap / blood colour) replaces the default when the sector colormap is 0 `[H] FragFunction.cs:119-127`; `[H] VertexOptions.cs:38-43`.
- *Filtering.* Mag/min filters are forced to `Nearest` and anisotropy to 1 `[H] LegacyGLTextureManager.cs:280-310`.

**Inference:** Each colormap costs 256 × 34 × 14 × 16 B ≈ 1.86 MiB of TBO. Every textured fragment does 1–2 dependent fetches: texture → index → TBO.

**True-colour mode.**
- Textures are ARGB8 (`m_pixels`).
- Filtering follows `Render.Filter.Texture` (default `Nearest`) plus `Anisotropy` (default 8) `[H] Core/Util/Configs/Components/ConfigRender.cs:71, 119`.
- `ColorMapFetch` emits an empty string, and `colorMapTranslationFrag` is read only inside it `[H] FragFunction.cs:110-111, 126`.
- The CPU path that once regenerated translated sprite textures, `GLTextureManager.GetSpriteRotation`, sits behind `!ArchiveCollection.StoreImageIndices && colorMapIndex > 0` `[H] Core/Render/OpenGL/Texture/GLTextureManager.cs:361-391`. `StoreImageIndices` is now hard-coded `=> true` `[H] Core/Resources/Archives/Collection/ArchiveCollection.cs:88`. The previous value was `ShaderVars.PaletteColorMode`. It was changed in commit f543542d9 ("Support render recreation for config values that previously required restart (#1865)", 2026-11-07), two commits before the pinned HEAD; the two later commits do not touch rendering. Verified with `git show f543542d9 -- Core/Resources/Archives/Collection/ArchiveCollection.cs` (`-    public static bool StoreImageIndices => ShaderVars.PaletteColorMode;` → `+    public static bool StoreImageIndices => true;`). The same commit dropped "Application restart required" from `Window.ColorMode` and added the release note "Changing video color mode … no longer requires application restart". It did not change `GetSpriteRotation`'s branch or `FragFunction.cs`. **Inference:** keeping indices always lets the palette mode be switched on at runtime, and the dead true-colour translation branch is a side effect.

So at HEAD, **no code path applies per-actor translations in true-colour mode** (player colours, blood colormaps, dehacked colormaps). The comment that says the overflow pool exists for "sprites using colormaps in true color mode" `[H] EntityRenderer.cs:61-62` describes code that can no longer run. See Open questions.

**Inference:** The intended true-colour design, visible in the dead branch, makes a **new GL texture per (sprite rotation × colormap)**, lazily, named `"<sprite>-ColorMap-<n>"`, and cached per rotation in a `LookupArray` `[H] GLTextureManager.cs:364-389`; `[H] Core/Resources/SpriteRotation.cs:17-34`. These textures do not go into arrays, so each translated sprite would be one draw, and the first sight of one would trigger a mid-frame image decode and upload.

## 9. Texture management: arrays, buckets, upload timing, filtering, lookup cost

**Every texture is a 2D array.** The `GLLegacyTexture` constructor passes `TextureTarget.Texture2DArray` to its base whatever `target` the caller gives `[H] Core/Render/OpenGL/Texture/Legacy/GLLegacyTexture.cs:16-20`. Standalone textures are therefore 1-layer arrays (`TexImage3D … depth 1`) `[H] LegacyGLTextureManager.cs:159-166`, and every shader samples `sampler2DArray` with `boundTextureIndex` `[H] FragFunction.cs:187-189`.

**Sprite arrays (load time).** `LegacyWorldRenderer.UpdateTo` calls `GLTextureArrayBuilder.BuildSprites` exactly once per renderer lifetime (`m_spritesCached`) `[H] LegacyWorldRenderer.cs:114-118`. `BuildSprites`:
1. Collects every unique rotation texture across all `SpriteDefinitions` `[H] GLTextureArrayBuilder.cs:50-75`.
2. Buckets each by `max(w,h)` into 32/64/128/256/512 squares. Brightmapped sprites go into an overflow bucket `[H] GLTextureArrayBuilder.cs:77-79, 248-274, 307-324`.
3. Splits buckets at `GL_MAX_ARRAY_TEXTURE_LAYERS` `[H] GLTextureArrayBuilder.cs:276-305`; `[H] Core/Render/OpenGL/Context/GLInfo.cs:44-48`.
4. Uploads each bucket with one `TexImage3D(RGBA8)` and then one `TexSubImage3D` per layer. Smaller images are first copied into a reused padded `fitImage`. Then `GenerateMipmap` `[H] GLTextureManager.cs:250-307`; `[H] LegacyGLTextureManager.cs:109-139`.

With `Render.CacheSprites = true` (the default), `TextureManager` creates every sprite definition, and therefore decodes every sprite image, at archive load rather than per map `[H] Core/Resources/TextureManager.cs:108-109, 155-165`; `[H] ConfigRender.cs:218`. With it off, sprite definitions are created per map for the definitions present, or lazily via `GetSpriteDefinition` `[H] TextureManager.cs:129-150, 178-187`.

Wall and flat arrays are built per level and exclude animated textures (`BuildLevel`) `[H] GLTextureArrayBuilder.cs:21-48`.

**Padding cost (Inference).** A 33×57 sprite occupies a 64×64 layer, and a 260×200 sprite a 512×512 layer. At RGBA8 with mips, one 512² layer ≈ 1.33 MiB, so 300 large sprites ≈ 400 MiB of VRAM before any translation copies. Array layers are allocated for the whole sprite set, not for what the map uses.

**Edge cases (Inference, from code):**
- `GetBucketIndex` caps at the 512 bucket `[H] GLTextureArrayBuilder.cs:317-323`, and `CopyPixelsFrom` copies only `min(srcW,dstW) × min(srcH,dstH)` `[H] Image.cs:442-458`. Sprites larger than 512 would be cropped, while `TextureInfo` still carries the original width. `TextureInfo` holds width and height in 10 bits each (≤1023) and the layer index in 12 bits (≤4095) `[H] VertexOptions.cs:67-73`.
- The 14-bit `OffsetXYZ` fields limit sprite offsets to ±16383 `[H] VertexOptions.cs:52-65`.
- `BuildSprites` builds every bucket except the last one (`i < buckets.Length - 1`), assuming the last is the brightmap overflow bucket `[H] GLTextureArrayBuilder.cs:81-85`. If `FinalizeTextureBuckets` splits the overflow bucket (more than `GL_MAX_ARRAY_TEXTURE_LAYERS` brightmapped sprite frames), all but its last chunk would be built as 0×0 arrays `[H] GLTextureArrayBuilder.cs:263-264, 276-305`. (Added by verifier; latent and unlikely in practice.)
- With `Render.CacheSprites = false`, the arrays are still built only once (`m_spritesCached`) from the definitions loaded for the first map. Sprites first loaded on later maps go through the standalone `CreateTexture` path in `GLTextureManager.GetSpriteRotation` and become one draw each `[H] LegacyWorldRenderer.cs:114-118`; `[H] GLTextureManager.cs:392-396`. (Added by verifier.)

**Mipmaps and filtering.** `GenerateMipmap` runs on every texture and array, even in palette mode where the filter is forced to `Nearest` `[H] LegacyGLTextureManager.cs:134, 176, 280-294`. Sprites and brightmaps are always `ClampToEdge` on both axes `[H] LegacyGLTextureManager.cs:141-146`. Changing filter or anisotropy loops over every tracked texture and rebinds it `[H] LegacyGLTextureManager.cs:53-101`.

**Lookup cost per draw (world sprites):** `m_spriteDefs.Data[spriteIndex]` → `Rotations[frame, rot]` → `RenderStore as GLLegacyTexture` → `ParentArrayTexture` → `LookupArray[TextureId]`. These are all array indexing and pointer loads, with no string hashing `[H] EntityRenderer.cs:182-198, 228-229, 278-279`; `[H] RenderDataCollection.cs:89-105`.

**HUD lookups are by string name.** `ResourceTracker` lookups use a case-insensitive `HashTable` or `Dictionary` per image draw `[H] Core/Render/OpenGL/Renderers/Legacy/Hud/LegacyHudRenderer.cs:54-64`; `[H] GLTextureManager.cs:202-231`; `[H] Core/Resources/ResourceTracker.cs:17-29`.

**Tracker mismatch for sprite-namespace HUD lookups (Inference).** `TryGet` checks `TextureTracker` first `[H] GLTextureManager.cs:208`. But `CreateTexture` forces `repeatY = false` for the Sprites namespace and inserts into `TextureTrackerClamp` `[H] GLTextureManager.cs:485-509, 511-512`. A HUD weapon sprite lookup (namespace Sprites `[H] WorldLayer.Render.Hud.cs:52, 558`) therefore misses `TextureTracker` on every call. It then goes through `ImageRetriever` (cached `m_compiledImages`) and finds the texture in the clamp tracker. That is about 3 hash lookups per weapon layer per call, and the HUD path calls it twice per weapon layer per frame: once in `DrawHudWeapon` and once in `LegacyHudRenderer.DrawImage`.

The weapon sprite also becomes a **second, standalone copy** of the sprite in VRAM, because sprite array sub-textures are not registered in the tracker (`addToTextureTracker: false`) `[H] GLTextureArrayBuilder.cs:84`. The copy is created lazily on the first draw.

**Unused alternative stack.** `Core/Render/OpenGL/Textures/GLTextureManager` (a 2048² × 8-layer `GLTexture2DArrayAtlas` built on `Atlas2D`, plus translation and handle buffer textures) is never instantiated. `grep -rn "new GLTextureManager\|GLTexture2DArrayAtlas(" Core Client` matches only `GLTexture2DArrayAtlas`'s own constructor, and `Atlas2D` is referenced only from `GLTexture2DArrayAtlas` and the equally unreferenced `Core/Util/Atlas/Atlas3D` `[H] Core/Render/OpenGL/Textures/GLTextureManager.cs:12-35`. Other classes in that namespace *are* live: `GLBufferTextureStorage`/`GLBufferTexture` back the colormap buffer and the framebuffers (`ColorMapBuffer.cs`, `OitFrameBuffer.cs`, `GLFramebuffer.cs` import it). Likewise, `Core/Resources/Textures/ResourceTextureManager` is constructed as `ArchiveCollection.Textures` `[H] Core/Resources/Archives/Collection/ArchiveCollection.cs:109`, but a case-sensitive `grep -rnE "[Cc]ollection\.Textures\b" Core Client` finds no readers (the case-insensitive variant only hits `Helion.Resources.Textures` namespace lines).

## 10. Software sprite clipping emulation (`VanillaRender`)

With `Render.VanillaRender` on, two extra passes render wall and plane "clip" buffers. The sprite fragment shader then does `texelFetch` on those buffers and on line and height buffer textures. It discards sprite pixels that the software renderer would have clipped, using line-segment intersection tests per fragment `[H] EntityProgram.cs:306-403`; `[H] LegacyWorldRenderer.cs:410-411`. `DownScaleVanillaRenderSampleBuffer` lowers the clip buffer's resolution `[H] ConfigRender.cs:184`.

**Inference:** This adds per-fragment cost on every sprite pixel: two texel fetches plus, for wall hits, two buffer fetches and two segment intersections. It is a correctness feature to drop for VR.

## 11. Models

**None.** `Core/Models/*` are save-game serialization DTOs (`EntityModel` holds velocity, health, armor…) used by `WorldBase.Serialize`, `EntityManager.PopulateFrom` and `SaveGameManager` `[H] Core/Models/EntityModel.cs:6-25`; `[H] Core/World/WorldBase.Serialize.cs:15, 181-199`; `[H] Core/World/Save/SaveGameManager.cs:16`. Searches that found nothing:
- `grep -rliE "modeldef|voxeldef|\bmd3\b|\bmd2\b|kvx" Core Assets`;
- `grep -rniE "md2|md3|modeldef|voxel|\.obj\b|iqm" Core --include=*.cs` outside `Models/`.

The entity `Scale` property exists `[H] Core/World/Entities/Definition/Properties/EntityProperties.cs:53`, but the entity renderer never reads it: `grep -rn "Scale" …/Entities/` finds only health-bar and downscale uses. Sprite scaling is therefore also absent.

## 12. HUD / 2D

**Command list.**
- `GLHudRenderContext` translates alignment and virtual resolution, then appends structs (`DrawImageCommand` carries the texture *name* as a string) to `RenderCommands` `List<>`s, which are cleared each frame `[H] Core/Render/OpenGL/GLHudRenderContext.cs:142-181`; `[H] Core/Render/Common/Commands/RenderCommands.cs:60-69, 93-107, 123-147`.
- `Renderer.Render` replays the commands in order into `LegacyHudRenderer` `[H] Renderer.cs:546-614, 822-867`.
- Queued HUD quads are flushed before each world draw and again at the end of the frame `[H] Renderer.cs:612, 900, 931-941`.

**Batching = run-length merge on the parent array texture.** `HudDrawBuffer.GetOrCreate` extends the last batch when `front.Texture == texture.ParentArrayTexture ?? texture`; otherwise it starts a new batch, reusing pooled `HudDrawBufferData` `[H] Core/Render/OpenGL/Renderers/Legacy/Hud/HudDrawBuffer.cs:51-77`.
- Each quad is 6 non-indexed `HudVertex` of 15 floats (60 B), so 360 B per quad `[H] Core/Render/OpenGL/Renderers/Legacy/Hud/HudVertex.cs:5-62`; `[H] HudDrawBuffer.cs:25-38`.
- The merge key ignores the brightmap, so a later quad with a different brightmap inherits the first one's brightmap `[H] HudDrawBuffer.cs:57-58`.
- Status-bar graphics are packed into same-dimension texture arrays the first time each layout is seen (`BuildHud`). For example, all STTNUM digits share one array, so the status bar collapses into a few batches `[H] Core/Layer/Worlds/StatusBar/StatusBarRenderer.cs:288-296`; `[H] GLTextureArrayBuilder.cs:123-157, 165-192`.
- **Text**: glyph quads go into the batch of the font's texture (one atlas image per font) `[H] LegacyHudRenderer.cs:89-136`; `[H] Core/Render/OpenGL/Texture/Legacy/LegacyGLTextureManager.cs:254-260`. `RenderableString` objects and their glyph and sentence arrays are pooled, using `lock`-protected free lists `[H] Core/Render/OpenGL/Texture/Fonts/RenderableString.Cache.cs:11-109`.

**Submission.** For each batch, `Render` does `m_pipeline.Clear()`, a CPU memcpy into the single stream VBO, `glBufferData` (orphan), texture binds, then `glDrawArrays` `[H] LegacyHudRenderer.cs:144-192, 226-237`. Depth is a painter's counter (`DrawDepth++` per quad) fed into an ortho matrix `[H] LegacyHudRenderer.cs:296, 299-310`. The HUD shader does colormap/palette fetch, true-colour invulnerability, brightmaps and screen-space fuzz refraction against the virtual framebuffer `[H] Core/Render/OpenGL/Renderers/Legacy/Hud/LegacyHudShader.cs:71-186`; `[H] Renderer.cs:931-941`.

**Weapon psprites** are ordinary HUD images drawn in 2D screen space. Light level is computed on the CPU from the player's sector and folded into the multiply colour. Colormap index and brightmap are passed per quad `[H] WorldLayer.Render.Hud.cs:514-598`.

**Inference:**
- Uploading per batch means N `glBufferData` calls per frame, where N is the batch count (dozens for a full HUD, more for menus with mixed fonts and graphics). On GL drivers each orphan is cheap but not free; UZDoom uploads once per frame (below).
- `Line`/`Points`/`DrawBox` are stubs ("Not implemented in the legacy renderer") `[H] GLHudRenderContext.cs:78-106`.

## 13. Per-frame allocations in these paths

- **World sprites, steady state:** none. `RenderEntity`, `RenderData.Draw`, `SetUniforms` and `ProgramUniforms.Set(mat4)` (which uses a static `MvpBuffer`) allocate nothing `[H] EntityRenderer.cs:178-318, 353-409`; `[H] Core/Render/OpenGL/Util/GLHelper.cs:122-140`; `[H] Core/Render/OpenGL/Shader/ProgramUniforms.cs:44`.
- **Conditional allocations:**
  - pool exhaustion constructs `RenderData`, i.e. VBO+VAO creation `[H] RenderDataPool.cs:55-57`;
  - `DynamicArray` growth on capacity overflow `[H] EntityRenderer.cs:284-285, 335`;
  - the translated-sprite path (unreachable at HEAD) concatenates a string and builds `Texture`/`Image` objects plus GL textures `[H] GLTextureManager.cs:370-389`;
  - first-time HUD weapon textures are created lazily (§9).
- **HUD:** pooled `HudDrawBufferData` uses `ArrayPool`-backed `DynamicArray(…, arrayPool: true)` `[H] Core/Render/OpenGL/Renderers/Legacy/Hud/HudDrawBufferData.cs:5-15`. Pooled `RenderableString` objects are freed back after `HandleDrawText` `[H] Renderer.cs:864-867`. Command `List<T>`s are cleared, not reallocated `[H] RenderCommands.cs:93-107`.
- **Load-time only:** `new Image[textures.Length]` per array build, the temporary `uint[]` per uploaded image in palette mode, and the `fitImage` scratch `[H] GLTextureManager.cs:260-285`; `[H] Image.cs:525`.

---

## Absent / not done (proven by search)

- **3D models and voxels** (MD2/MD3/OBJ/IQM/UE1/KVX, MODELDEF/VOXELDEF): no hits; see §11.
- **Geometry shaders, bindless textures, indirect or multi-draw**: `grep -rniE "GeometryShader|bindless|MultiDraw|DrawElementsIndirect|DrawArraysIndirect" Core` matches only the unused `RenderProgram.GeometryShader()` hook and a class name; `grep -rn "override string? GeometryShader" Core` is empty.
- **Compressed textures**: `grep -rniE "CompressedTex|Dxt|BC7|S3tc" Core` finds only false positives (`m_textureIdxToPosition` in the unused `Textures/GLTextureManager`); everything is RGBA8 `[H] LegacyGLTextureManager.cs:115, 161, 171`.
- **Dynamic lights on sprites, and GLDEFS lights**: only brightmap/skybox/#include are parsed `[H] GldefsDefinition.cs:80-84`; `grep -rniE "dynamiclight|pointlight|shadowmap" Core` is empty.
- **Render styles** Subtract, ReverseSubtract, Stencil, AddStencil, Shaded, Shadow-as-blend: not in the enum `[H] RenderStyle.cs:3-16`.
- **Flat, wall, roll or Y-billboard sprites** (RF_FLATSPRITE/WALLSPRITE/ROLLSPRITE equivalents): `grep -rn "FlatSprite\|WallSprite\|ROLLSPRITE" Core/Render` is empty; the billboard is always view-plane yaw (§2). The DECORATE flags `FLATSPRITE`/`WALLSPRITE`/`ROLLSPRITE` *are* parsed into `ActorFlags`/`EntityFlag` `[H] Core/Resources/Definitions/Decorate/Parser/DecorateFlagParser.cs:494, 848, 971`, but nothing in the renderer reads them.
- **16-rotation sprites, sprite scale, sprite angle overrides**: see §5 and §11.
- **CPU translucency sorting**: none, by design (WBOIT, §6).
- **Per-frame (sub-tic) rotation, light or visibility updates**: by design (§1).
- **Texture streaming or eviction**: all sprites are built once (`m_spritesCached`); no eviction code in `LegacyGLTextureManager` (`grep -n "Evict\|Release\|Lru" …` is empty).
- **Atlas usage in the live path**: `Atlas2D`/`Atlas3D` (in `Core/Util/Atlas`) are referenced only by the unused `GLTexture2DArrayAtlas`/`Textures.GLTextureManager` path and by each other (§9).

## Perf-relevant settings

| Setting | Default | Effect | Citation |
|---|---|---|---|
| `Window.ColorMode` | `TrueColor` | Palette mode: indexed RGBA8 + TBO colormap fetch per fragment, nearest filtering forced | `[H] ConfigWindow.cs:105`; `[H] Renderer.cs:261` |
| `Render.Filter.Texture` | `Nearest` | True-colour min/mag filter (Trilinear uses mips) | `[H] ConfigRender.cs:71`; `[H] LegacyGLTextureManager.cs:280-294` |
| `Render.Anisotropy` | 8 | Anisotropic filtering (true colour only) | `[H] ConfigRender.cs:119` |
| `Render.CacheSprites` | true | Decode all sprites at archive load (no runtime hitch, more RAM) | `[H] ConfigRender.cs:218`; `[H] TextureManager.cs:108-109` |
| `Render.SpriteTransparency` | true | Off forces all non-fuzzy sprites into the opaque bucket (skips OIT) | `[H] ConfigRender.cs:176`; `[H] EntityRenderer.cs:258` |
| `Render.PostProcessingEffects` | true | Enables the fuzz refraction pass (and reverse-Z) | `[H] ConfigRender.cs:237`; `[H] LegacyWorldRenderer.cs:607-616`; `[H] Renderer.cs:266-277` |
| `Render.FuzzAmount` | 1 | Fuzz block size (`fuzzDiv`) | `[H] ConfigRender.cs:164`; `[H] Renderer.cs:296-299` |
| `Render.VanillaRender` | false | Software sprite clipping: clip-buffer passes + heavy per-fragment discard logic | `[H] ConfigRender.cs:180`; `[H] EntityProgram.cs:306-403` |
| `Render.DownScaleVanillaRenderSampleBuffer` | 1 | Clip-buffer downscale | `[H] ConfigRender.cs:184` |
| `Render.MaxDistance` | 0 (→ 6000 units) | CPU distance cull + shader fade (fade starts at √½ × max ≈ 4243 units) | `[H] ConfigRender.cs:136`; `[H] Renderer.cs:347-349`; `[H] LegacyWorldRenderer.cs:285-289`; `[H] EntityProgram.cs:418-419` |
| `Render.Mode` | `Static` | Static (blockmap) / BSP / Adaptive (heuristic) entity collection; a non-middle transfer-heights view forces BSP | `[H] ConfigRender.cs:52-59, 93`; `[H] LegacyWorldRenderer.cs:300-308, 336-339`; `[H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.BspHeuristics.cs:19-31` |
| `Render.HealthBar.Enable` | false | Extra instanced pass + program | `[H] ConfigRender.cs:78`; `[H] EntityRenderer.cs:316-317, 418-423` |
| `Render.SpriteClip` / `SpriteClipMin` / `SpriteClipFactorMax` | true / 16 / 0.02 | CPU corpse floor-clip math | `[H] ConfigRender.cs:224, 230, 227` |
| `Render.EmulateInvulnerabilityColorMap` | false | Creates the colormap TBO in true colour | `[H] ConfigRender.cs:188`; `[H] Renderer.cs:239-240` |
| `Render.NullTexture` | false | Checkerboard null texture | `[H] ConfigRender.cs:125` |

## Transfer notes for UZDoom VR

**1. Sprite submission (per-tic instancing vs per-frame CPU quads).**
- *UZDoom today.* `HWSprite::Process` runs per frame per eye, because `ProcessScene` is called inside the eye loop `[U] src/rendering/hwrenderer/hw_entrypoint.cpp:129-166`. It interpolates the actor position per frame `[U] src/rendering/hwrenderer/scene/hw_sprites.cpp:847`. It builds 4 CPU-billboarded `FFlatVertex` (8 floats = 32 B each) `[U] hw_sprites.cpp:624-640`; `[U] src/common/rendering/hwrenderer/data/flatvertices.h:35-41`. It issues one `SetMaterial` and one `state.Draw(DT_TriangleStrip, …, 4)` per sprite, plus a second draw for coloured-fog layers and more per light-list slice `[U] hw_sprites.cpp:228, 287-297`.
- *Grade:* **Rework.** Keep `HWSprite::Process` for game-side logic (translations, styles, clipping decisions), but emit a compact instance record into a per-frame SSBO or instance buffer instead of 4 vertices. Expand corners in the vertex shader as Helion does `[H] EntityProgram.cs:142-166`.
- *Stereo/Vulkan.* Run `Process` once per frame, not per eye, using the centre-eye pose. Draw instance buckets with `VK_KHR_multiview` so one `vkCmdDraw*Indirect` covers both eyes. Sort instances by (pipeline, material) into indirect draws. Under a bindless descriptor array, the material can be an index inside the instance, so whole style classes collapse into one draw.
- *Do not copy Helion's tic-locked visibility* (§1): in VR, visibility must be re-evaluated per frame, or conservatively widened, for head yaw.

**2. Interpolation in the shader.**
- *UZDoom today:* CPU `InterpolatedPosition(vp.TicFrac)` `[U] hw_sprites.cpp:847`.
- *Grade:* **Port** (inside the Rework above): store prev/cur position in the instance and lerp in the vertex shader. This lets a later late-latch or reprojection step change `TicFrac` without touching the CPU.

**3. Rotation-frame selection.**
- *UZDoom today:* per frame with 16-rotation support (`Texture[16]`) and `GetSpriteAngle(ang, TicFrac)` `[U] src/r_data/sprites.h:40`; `[U] hw_sprites.cpp:992-1012`. It also honours `MF7_SPRITEANGLE` and the actor's `SpriteRotation` offset there, which Helion lacks (§5).
- *Grade:* **Port.** Keep it on the CPU, but compute it **once per frame from the centre eye** and share it across eyes. Selecting per eye can give different frames to the two eyes at a boundary, which causes binocular rivalry.

**4. Billboard orientation.**
- *UZDoom today:* `gl_billboard_mode` and `gl_billboard_faces_camera` choose between yaw-plane and camera-facing billboards `[U] hw_sprites.cpp:82-84, 412-416`. Helion supports only view-plane yaw `[H] EntityRenderer.cs:369`.
- *Grade:* **Port.** For VR, use *position-facing* yaw billboards. Compute the normal from `entityPos - headCentrePos` in the vertex shader. View-plane billboards visibly swing when the head rotates. Use the centre eye for both eyes so the quad is the same planar surface in each eye; otherwise stereo depth breaks.

**5. Translucency ordering.**
- *UZDoom today:* BSP-split sorting with `std::stable_sort` on sprite lists `[U] src/rendering/hwrenderer/scene/hw_drawlist.cpp:598-622, 629-712`.
- *Helion:* WBOIT with no CPU sort but 2–4× overdraw (§6).
- *Grade:* **Rework**, optional. A per-frame CPU sort done once (centre eye) and reused for both eyes is accurate and cheap at Doom sprite counts. WBOIT costs RGBA16F + RG16F targets per eye at VR resolution, plus a re-raster composite. **Inference:** keep UZDoom's sort but move it off the per-eye path. Consider WBOIT only for particle-heavy mods.

**6. Fuzz/spectre.**
- *UZDoom today:* blend-op `STYLEOP_Shadow`/`Fuzz` handled in `DrawSprite` `[U] hw_sprites.cpp:132-150`.
- *Helion:* a screen-space refraction that reads the current colour attachment (§6).
- *Grade:* **Port** a screen-space fuzz only if it reads a *copied* scene colour (Vulkan: copy or input attachment). Fuzz block size must be in angular units, not pixels; otherwise it shimmers differently in each eye at VR resolution.

**7. Texture strategy (arrays/buckets vs per-texture objects).**
- *UZDoom today:* one `FHardwareTexture` per material and translation, with translations looked up linearly in `TranslatedTexture` lists `[U] src/common/textures/hw_texcontainer.h:79-141`; `[U] src/common/textures/hw_material.cpp:168-186`. Precached at level start `[U] src/rendering/hwrenderer/hw_precache.cpp:43-89`.
- *Grade:* **Rebuild** around Vulkan descriptor indexing (a bindless sampled-image array). This beats Helion's padded power-of-two array buckets: no 2–4× VRAM padding waste, no layer cap, no bucket-count draw splits, and no exclusion of brightmapped sprites. Helion's lesson is to *resolve the texture handle on the CPU once and put the index in the instance* `[H] VertexOptions.cs:67-73`. Keep translations as GPU palette remaps (indexed texture + translation table index in the instance), not per-translation texture copies. That eliminates both UZDoom's per-translation `FHardwareTexture` and Helion's (dead) per-colormap regeneration.

**8. Palette mode.**
- *UZDoom:* `CTF_Indexed` forces `translation = -1` (indexed sampling path) `[U] hw_material.cpp:184`.
- *Helion:* RGBA8 with R = index, plus an RGBA32F TBO of all colormaps (1.86 MiB each) `[H] ColorMapBuffer.cs:9-23`.
- *Grade:* **Port**, with changes: store indices as R8 (4× less bandwidth than Helion's RGBA8), keep the colormap table as an RGBA8 or R10G10B10 2D texture rather than RGBA32F, and select the light row per fragment.

**9. Fullbright and brightmaps.**
- *Helion:* alpha = 254 flag texels plus a separate brightmap texture (§7). *UZDoom:* material layers.
- *Grade:* **Port.** Keep brightmaps as a second bindless index in the instance so they do not break batching. Helion's array exclusion is the anti-pattern `[H] GLTextureArrayBuilder.cs:201-205`.

**10. Models.**
- *UZDoom:* MD2, MD3, OBJ, UE1, IQM (with bones) and KVX voxels `[U] src/common/models/` (models_md2.cpp, models_md3.cpp, models_obj.cpp, models_ue1.cpp, models_iqm.cpp, models_voxel.cpp). Rendered per model with its own vertex/index buffer, frame-pair interpolation via `SetVertexBuffer(buf, frame1, frame2)`, and a bone base index `[U] src/rendering/hwrenderer/hw_models.cpp:108-140`.
- *Helion:* nothing to borrow.
- *Grade:* **Rework** for VR: per-eye cost is a draw per model per eye today. Use multiview, and convert the frame-pair vertex streams to a compute pre-skin or vertex pulling from SSBOs so models can join indirect draws.

**11. Weapon psprites.**
- *UZDoom:* `HWDrawInfo::DrawPlayerSprites` (2D overlay, HUD-model step) `[U] src/rendering/hwrenderer/scene/hw_weapon.cpp:114`.
- *Helion:* 2D HUD quads with CPU interpolation (§12).
- *Grade:* **Rebuild** for VR. A screen-space weapon is unusable in a headset. It must become a world-space quad (or model) attached to the hand or head pose, rendered in the opaque and translucent passes with the same instance path as world sprites.

**12. HUD/2D.**
- *UZDoom today:* `F2DDrawer::AddCommand` merges compatible consecutive commands (texture, translation, style, scissor, colour…) `[U] src/common/2d/v_2ddrawer.cpp:211-224`; `[U] src/common/2d/v_2ddrawer.h:182-202`. It uploads all 2D vertices and indices **once per frame** `[U] src/common/rendering/hwrenderer/hw_draw2d.cpp:85` and draws each command with `DrawIndexed` `[U] hw_draw2d.cpp:219`. That is already better than Helion's upload per batch.
- *Grade:* **Rework.** Render the 2D drawer into an offscreen HUD layer (one per frame, not per eye). Composite it as a world-locked or head-locked quad, or submit it as an OpenXR quad layer (`XrCompositionLayerQuad`) so the runtime reprojects it at display rate. Bindless indices would let the 2D path merge across textures too.

**13. Tic-locked CPU work (Helion's main win).**
- *UZDoom:* everything is per frame and per eye `[U] hw_entrypoint.cpp:129-166`.
- *Grade:* **Rework.** Split the scene into "per-tic game snapshot → instance build" and "per-frame per-view culling + draw". Instance build happens per tic. Culling uses per-frame head pose against those instances, on the GPU (compute frustum cull into indirect args) or with a cheap per-frame CPU pass. That keeps Helion's CPU savings without its stale visibility.

## Open questions

1. **True-colour translations at HEAD.** By code reading, f543542d9 (`StoreImageIndices => true`) makes `GetSpriteRotation`'s regeneration branch unreachable, and the true-colour shader never reads `colorMapTranslationFrag` (§8). In true colour, `GetGlTexturePixels(ShaderVars.PaletteColorMode)` also uploads plain ARGB, so the stored indices are not used either `[H] LegacyGLTextureManager.cs:125, 156`; `[H] Image.cs:520-523`. That means player colours, coloured blood and dehacked colormaps would render untranslated in the default colour mode. The verifier confirmed the diff and the call chain (`EntityRenderer.GetSpriteRotation` → `GLTextureManager.GetSpriteRotation` → `else` branch) but did not run the engine. Nothing in the commit or its release note mentions translations, so a regression seems more likely than an unfinished move toward shader translations.
2. **Visibility pop-in on fast yaw.** The tic-locked visibility walk (§1) suggests sprites can be missing for the rest of a tic after a fast turn. I did not check whether the camera yaw used in `SetOccludePosition` and `Frustum.SetFrustumPlanes` is itself tic-quantised or per-frame mouse-interpolated (`renderInfo.Camera` origin not traced). If it is tic-quantised, the issue does not appear on a desktop. Verifier note: `SetOccludePosition` runs every frame `[H] LegacyWorldRenderer.cs:328-329`, but its result is copied into `m_renderData.OccludePos` only in `SetupRenderData` on a tic change `[H] LegacyWorldRenderer.cs:180-196, 331-341`. The frustum planes are set only inside `TraverseBsp` `[H] LegacyWorldRenderer.Bsp.cs:17-19`. Both culls therefore use the camera state of the first frame of the tic.
3. **Sprites larger than 512 px** (`GetBucketIndex` cap plus `CopyPixelsFrom` crop, §9): this looks like a crop or UV-overrun bug. I did not verify against an actual large-sprite WAD.
4. **Feedback-loop correctness of the fuzz refraction pass** on drivers without implicit texture barriers. I did not test it.
5. **Real VRAM footprint** of the sprite buckets for typical mod sets: not measured, only estimated from the bucket arithmetic.
6. **Whether `HudDrawBuffer`'s brightmap-insensitive merge** produces visible errors (HUD weapon brightmaps merged with adjacent same-array HUD images): not tested.
