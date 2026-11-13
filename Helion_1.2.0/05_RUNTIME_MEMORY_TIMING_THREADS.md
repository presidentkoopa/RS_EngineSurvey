# H5 — Helion runtime: main loop, frame pacing, allocation discipline, threading, build config, benchmarks

Helion runs everything that matters on one thread. A single OpenTK `RenderFrame` callback polls input, runs at most **one** 35 Hz tic, renders, swaps, optionally calls `glFinish`, and then sleeps in a software frame limiter. The world is drawn by interpolating between the previous and current tic using the fractional tic time. Mouse look skips that interpolation: it is added straight into a per-frame "view angle", so the camera turns at render rate. GC behaviour is managed more than it is avoided: a forced full collection runs after every map load and `GCLatencyMode.SustainedLowLatency` is switched on for gameplay. The hot simulation and render paths reuse `DynamicArray<T>` buffers by setting their length to zero, take objects from a hand-written free-list cache (`DataCache`), and keep geometry in flat struct arrays. Only one background thread does real work on world data during gameplay: `AutomapMarker`, which walks the BSP for automap marking and adaptive-render heuristics. It writes line flags without any synchronisation. (Trivial exceptions: the savegame task reads the map name, and with `Developer.LogGC` on, the finalizer thread posts a message to the world.) Map loading (ZDBSP and world construction), savegame serialisation, music streaming and screenshot encoding run off the main thread. All GL work stays on the main thread. The BenchmarkSuite contains only BenchmarkDotNet micro-benchmarks of AI movement, line of sight, hitscan and entity creation. Nothing in it measures frame time or rendering.

---

## 1. Main loop and frame structure

### 1.1 Entry and window loop

`Client.Main` → `Run` builds the config, `ArchiveCollection`, the OpenAL audio system and the `Client`, then calls `client.Run()` [H] Client/Client.cs:663-680, 742-777. `Run()` starts the async `Initialize()` (file loading) and enters `m_window.Run()`, which is OpenTK's `GameWindow.Run` [H] Client/Client.cs:568-573; Client/Window.cs:138-142. The whole game is driven from one event handler, `m_window.RenderFrame += Window_MainLoop` [H] Client/Client.cs:146. Helion never subscribes to `UpdateFrame`: `grep -rn "UpdateFrame\|OnUpdateFrame\|PollEvents\|ProcessEvents"` over Client/ and Core/ finds only unrelated framebuffer helpers.

The window is created with `UpdateFrequency = 500` [H] Client/Window.cs:191-197. `SetSyncMode` resets it to `UpdateFrequency = 0` every time vsync or max-FPS changes, and it is also called from `UpdateWindow` [H] Client/Window.cs:361, 463-493. OpenTK's pacing is therefore disabled and Helion's own `FrameLimiter` does the pacing.

The per-frame body [H] Client/Client.cs:388-406:

```csharp
private void Window_MainLoop(FrameEventArgs frameEventArgs)
{
    m_window.JoystickAdapter.Poll();
    m_profiler.ResetTimers();
    m_profiler.Global.Start();
    CheckLoadFilesComplete();
    CheckMapLoad();
    CheckLoadMapComplete();
    CheckForErrorsIfDebug();
    RunLogic();
    Render();
    m_soundManager.Update();
    m_profiler.Global.Stop();
    m_profiler.MarkFrameFinished();
}
```

`Render()` [H] Client/Client.cs:355-386 performs the layer render, swap, optional `glFinish` and FPS bookkeeping, then **sleeps in the frame limiter inside `Render()`**. Sound update (`SoundManager.Update`, which ticks the audio manager and frees finished sources [H] Core/Audio/Sounds/SoundManager.cs:643-662) therefore runs after the sleep. Its cost counts toward the next frame's limiter budget.

Strict per-frame order:
1. Controller poll (SDL wrapper) [H] Client/Client.cs:390.
2. Queued map load / load-complete handling [H] Client/Client.cs:394-396.
3. `RunLogic` → `GameLayerManager.RunLogic(m_ticker.GetTickerInfo())` [H] Client/Client.cs:326-331.
4. `Render` → layer render, `SwapBuffers`, optional `GL.Finish`, `FpsTracker.FinishFrame`, `FrameLimiter.Limit(MaxFps)` [H] Client/Client.cs:367-385.
5. `SoundManager.Update` [H] Client/Client.cs:402.

Keyboard and mouse events reach Helion through OpenTK/GLFW callbacks: `KeyDown`, `MouseMove` and so on are wired at [H] Client/Window.cs:95-101. They are queued into `InputManager` (§1.4). GLFW raw mouse motion is enabled when supported [H] Client/Client.cs:301-305. The OS event pump is called inside OpenTK's `GameWindow.Run`, before each `RenderFrame` dispatch. OpenTK is a NuGet package (`OpenTK 4.9.4` [H] Core/Core.csproj:28), and its source is not in the repo, so the pump ordering is not verified here (see Open questions).

### 1.2 Tic accumulation (35 Hz) — `Ticker`

`Constants.TicksPerSecond = 35.0` [H] Core/Util/Constants.cs:25. `Client` owns `new Ticker(Constants.TicksPerSecond)` [H] Client/Client.cs:68. It starts in the constructor [H] Client/Client.cs:152 and restarts whenever a `WorldLayer` is added [H] Client/Client.cs:307-311; Core/Util/Timing/Ticker.cs:64-69.

`GetTickerInfo()` [H] Core/Util/Timing/Ticker.cs:81-104:

```csharp
m_tickAccumulation += m_stopwatch.ElapsedTicks - m_lastTickSeen;
m_lastTickSeen = m_stopwatch.ElapsedTicks;
double tickFractionUnit = (double)m_tickAccumulation / m_stopwatchTicksPerGametick;
int ticks = (int)Math.Floor(tickFractionUnit);
float fraction = (float)(tickFractionUnit - ticks);
RemoveExcessTicks(ticks);
// If we notice that our simulation/rendering loop is overflowing, then we do
// not want to buffer or else we run into a feedback loop that keeps growing.
// In such a case, toss out the extra tick(s), and treat it as a whole new
// timeslice.
if (ticks >= 2)
{
    m_tickAccumulation = 0;
    ticks = 1;
    fraction = 0;
}
return new(ticks, fraction);
```

Facts:
- A frame runs **at most one tic**. Whenever two or more tics are due, the accumulator is zeroed, the fraction is set to 0 and one tic runs [H] Core/Util/Timing/Ticker.cs:96-101. The leftover fractional tic is thrown away too. `GetTickerInfo` is called once per frame, from `Client.RunLogic` only [H] Client/Client.cs:329 (`EndGameLayer` keeps its own separate `Ticker`).
- The cap fires only when one frame's elapsed time plus the carried fraction reaches 2 tics. The carried fraction is always < 1, so this needs a single frame longer than one tic (28.6 ms). Above 35 fps in steady state it never fires.
- `TickerInfo` is a `readonly record struct (int Ticks, float Fraction)`, so it allocates nothing [H] Core/Util/Timing/Ticker.cs:8.
- `WorldLayer.TickWorld` also has an overflow guard (`TickOverflowThreshold = 10 * 35` ticks → clamp to 1) [H] Core/Layer/Worlds/WorldLayer.cs:43; Core/Layer/Worlds/WorldLayer.Logic.cs:127-131. The Ticker already caps at 1, so the guard only matters on the demo fast-forward path, which multiplies `ticksToRun` by the playback speed [H] Core/Layer/Worlds/WorldLayer.Logic.cs:96-102.
- The ticks loop runs `World.Tick()` once per tic. The same `TickCommand` is reused for every tic in the loop [H] Core/Layer/Worlds/WorldLayer.Logic.cs:133-147.

**Inference (time dilation):** Helion never catches up on missed tics, so below 35 fps the simulation runs slower than wall-clock time. Worked example at 25 fps (40 ms frames, 28.57 ms tics): accumulation goes 1.40 → 1.80 → 2.20 tics. The third frame hits `ticks >= 2` and is reset, so 3 frames (120 ms) yield 3 tics (85.7 ms) of game time, i.e. about 71 % speed. A single long hitch (for example 200 ms) produces one tic and throws away the other ~6. For VR at 90–120 Hz the cap never fires in steady state. It only fires on hitches, where dropping time is arguably better than a catch-up burst. Demo sync is unaffected because tics stay deterministic.

### 1.3 Render interpolation

`WorldLayer.TickWorld` stores the last `TickerInfo` [H] Core/Layer/Worlds/WorldLayer.Logic.cs:84-90. When `MaxFPS == 35` it forces `Fraction = 1`, which turns interpolation off at the tic rate [H] Core/Layer/Worlds/WorldLayer.Logic.cs:88-90. At render time `SetWorldContextVars` builds the camera from `GetCamera(m_lastTickInfo.Fraction)` and passes the same fraction to the world render context [H] Core/Layer/Worlds/WorldLayer.Render.cs:47-52. That context feeds entity, sector and wall interpolation in the renderer, which is covered by the renderer sections.

Each tic snapshots the previous state before simulating. `TickEntities` sets `entity.PrevPosition = entity.Position` before `PhysicsManager.Move` [H] Core/World/WorldBase.cs:1167-1181. Players call `SetInterpolation()` in `TickPlayers` [H] Core/World/WorldBase.cs:1250.

`Player.GetCamera(t)` interpolates position with `prevPos.Interpolate(currentPos, t)` and then clamps to floor and ceiling [H] Core/World/Entities/Players/Player.cs:682-722. Angle handling has three cases [H] Core/World/Entities/Players/Player.cs:724-760:
- With `Mouse.Interpolate` off (the default), the not-yet-consumed per-frame mouse delta (`ViewAngleRadians`/`ViewPitchRadians`) is **added directly** to the tic angle. Rotation is therefore up to date every render frame. This path also requires no strafe command this tic, chase camera off (`ShouldInterpolateMouse()` is `Mouse.Interpolate || Game.Chase` [H] Core/Util/Configs/Impl/Config.cs:43), and MaxFPS not capped at ≤ 35 [H] Core/World/Entities/Players/Player.cs:727-731.
- If `m_interpolateAngle` is set, yaw and pitch are lerped prev→current with wrap-around handling.
- `ShouldInterpolate()` turns angle interpolation on only for mouse-interpolate (or chase-cam) mode, keyboard turning, death or demo playback. It turns it off when `MaxFPS` is non-zero and ≤ 35 [H] Core/World/Entities/Players/Player.cs:992-1004.

**Inference:** the world is shown at `t ∈ [0,1)` between tic N−1 and tic N, so world motion lags wall-clock time by up to one tic (≤ 28.6 ms) plus the frame. The camera angle has no such lag. This is the classic Doom-source-port split and the right baseline for VR, where the head pose must bypass the tic entirely.

### 1.4 Input sampling relative to render

GLFW callbacks append to `InputManager`:
- Key and mouse-button events go into a `DynamicArray<InputKey> m_events` [H] Core/Window/Input/InputManager.cs:49-64.
- Mouse deltas accumulate into `MouseMove` [H] Core/Window/Input/InputManager.cs:77-85.
- Wheel input accumulates into `m_mouseScroll` [H] Core/Window/Input/InputManager.cs:87-90.

`GameLayerManager.HandleInput` calls `inputManager.Poll(tickerInfo.Ticks > 0)` [H] Core/Layer/GameLayerManager.cs:419-433:
- Key state (`CreateCurrentInputKeys`, analog `AnalogAdapter.Poll`) is consolidated **only on frames that run a tic** [H] Core/Window/Input/InputManager.cs:217-266. On those frames `ProcessedKeys()` clears the key edge state [H] Core/Window/Input/InputManager.cs:179-195; Core/Layer/GameLayerManager.cs:429-430.
- The controller is still polled every frame. `AnalogAdapter` is the same `ControllerAdapter` instance as `Window.JoystickAdapter` [H] Client/Input/Controller/ControllerAdapter.cs:125, which `Window_MainLoop` polls at the top of every frame [H] Client/Client.cs:390. That poll turns button and axis edges into `SetKeyDown`/`SetKeyUp` events [H] Client/Input/Controller/ControllerAdapter.cs:156-222. On tic frames it is polled a second time. The events are still consolidated only at tic time.
- Mouse movement is consumed **every frame**, and `ProcessedMouseMovement()` zeroes it every frame [H] Core/Layer/GameLayerManager.cs:428-432; Core/Window/Input/InputManager.cs:211-214.
- `WorldLayer.HandleInput` calls `World.HandleMouseMovement(input)` on every frame and handles key and command input only when `input.HandleKeyInput` is set [H] Core/Layer/Worlds/WorldLayer.Input.cs:90-102.
- `HandleMouseLook` converts pixels to radians (`PixelDivisor`, sensitivity, yaw/pitch scale) and calls `player.AddToYaw(delta, isMouse: true)`, which only accumulates into `ViewAngleRadians` [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:564-585; Core/World/Entities/Players/Player.cs:549-569. Gyro uses the same path [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:595-621.
- At tic time, `SetTickCommand` moves the accumulated view angle and pitch into `tickCommand.MouseAngle/MousePitch` and zeroes the view accumulators [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:396-407. Movement keys are turned into speeds in the same function [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:409-445.

Latency facts:
- Keyboard and controller button changes take effect on the next tic, which is ≤ 28.6 ms later.
- Mouse and gyro look takes effect on the next rendered frame.
- `Client.HandleWinMouseMove` (a Windows raw-input path) has no caller. `grep -rn HandleWinMouseMove` finds only its definition [H] Client/Client.cs:575-581, so it is dead code.

**Data race (fact plus inference):** `Client.LoadMap` runs on a thread-pool thread (§4) and calls `m_window.InputManager.Clear()` [H] Client/Client.Commands.cs:914-916, 977. GLFW callbacks may append to the same `DynamicArray`s on the main thread at the same moment. `InputManager` has no lock (no `lock` in the file). During loading `LockInput` is true [H] Client/Client.cs:434, but callbacks still fire.

### 1.5 VSync, uncapped mode and the frame limiter

Config [H] Core/Util/Configs/Components/ConfigRender.cs:98-112:
- `Render.VSync` = `On` (enum Off/On/Adaptive).
- `Render.MaxFPS` = `0`, clamped so that ≤0 → 0 and 1–34 → 35.

`SetSyncMode` [H] Client/Window.cs:463-493:

```csharp
// (switch mapping RenderVsyncMode → OpenTK VSync elided, lines 465-478)
UpdateFrequency = 0;

if (maxFps == 0)
{
    _ = GetMonitors(out MonitorData? current);
    MaxFps = current?.RefreshRate > 0 && vsync != RenderVsyncMode.Off
        ? current.RefreshRate
        : 0;
    return;
}

MaxFps = maxFps;
```

So:
- **Uncapped** requires `MaxFPS = 0` **and** `VSync = Off`.
- With the defaults (VSync On, MaxFPS 0) the software limiter is **also** armed, at the monitor refresh rate. `GetMonitors` allocates a `List<MonitorData>` [H] Client/Window.cs:164-183, but only when the sync mode changes, not per frame.

`FrameLimiter.Limit` [H] Client/FrameLimiter.cs:12-40 is a hybrid sleep/yield wait with drift carry-over:

```csharp
if (targetFps > 0)
{
    var target = 1000.0 / targetFps;
    var driftTarget = Math.Max(target - m_drift, 0);
    m_drift = 0;
    while (true)
    {
        var elapsed = m_stopwatch.Elapsed.TotalMilliseconds;
        if (elapsed >= driftTarget)
        {
            m_drift = Math.Clamp(elapsed - driftTarget, 0, target);
            break;
        }

        var sleepTime = driftTarget - elapsed;
        if (sleepTime <= 0)
            break;

        if (sleepTime < 2)
            Thread.Sleep(0);
        else
            Thread.Sleep((int)(sleepTime - 1));
    }
}

m_stopwatch.Restart();
```

On Windows, `timeBeginPeriod(wPeriodMin)` raises the scheduler tick for the whole process lifetime [H] Client/Client.cs:187-197; Client/WinNative.cs:16-40. It is undone in `PerformDispose` [H] Client/Client.cs:595-596.

`Render.ForcePipelineFlush` (default false) calls `GL.Finish()` after `SwapBuffers` [H] Client/Client.cs:376-379; Core/Render/Renderer.cs:674-677; ConfigRender.cs:220-221. This is the only CPU/GPU throttle. `grep -rn "FenceSync\|ClientWaitSync"` in Core finds nothing, so Helion has no frame-in-flight fences. That matters for its persistent-mapped buffers (renderer sections).

**Inference:** with vsync on, the limiter adds a second wait on top of the swap block. The limiter measures from its own previous return, so after a swap that already blocked until vblank, elapsed is normally ≥ target and the limiter returns at once. Its drift term exists to absorb jitter. UZDoom's Vulkan path calls its limiter only when vsync is off. Its OpenGL path calls it on every swap (§Transfer). For VR the limiter has to go: the compositor (`xrWaitFrame`) owns pacing.

### 1.6 Layer manager: how logic and render interleave

`GameLayerManager.RunLogic` handles input, then calls `RunLogic(tickerInfo)` on each optional layer (console, options, menus, title, intermission, world, ENDOOM). These are fixed nullable properties, not a list [H] Core/Layer/GameLayerManager.cs:733-753. `Render` hands a **cached delegate** to the renderer (`m_renderDefaultAction = RenderDefault`, `m_renderHudAction = RenderHud`, created once in the constructor [H] Core/Layer/GameLayerManager.cs:79-80, 111-112, 755-759). There is no per-frame closure allocation. `RenderDefault` draws the world if `WorldLayer.ShouldRender`, then the HUD pass, which composites the virtual framebuffer, HUD, automap, transition, menus and console [H] Core/Layer/GameLayerManager.cs:761-903.

`WorldLayer` uses the same delegate caching (`m_renderWorldAction`, `m_renderAutomapAction`, `m_drawHudAction` [H] Core/Layer/Worlds/WorldLayer.Render.cs:18-20).

The `Layers` enumerable *does* allocate a `List` plus a LINQ `WhereNotNull` on each call [H] Core/Layer/GameLayerManager.cs:92-95. It is used only by `ClearAllExcept` and `ExpireAnimations` [H] Core/Layer/GameLayerManager.cs:294, 572, which are not per-frame.

---

## 2. Allocation discipline and GC control

### 2.1 Explicit GC control

| Where | What | Citation |
|---|---|---|
| Map load queued | `GCSettings.LatencyMode = Interactive` | [H] Client/Client.cs:430; Core/Util/GCUtil.cs:8 |
| Map load finished | `GC.Collect(GC.MaxGeneration, GCCollectionMode.Forced, blocking: false)` then `LatencyMode = SustainedLowLatency` | [H] Client/Client.cs:528-529; Core/Util/GCUtil.cs:10 |
| Menu opened | `LatencyMode = Interactive` | [H] Core/Layer/GameLayerManager.cs:185-188 |
| Menu closed with a world loaded | `LatencyMode = SustainedLowLatency` | [H] Core/Layer/GameLayerManager.cs:316-317 |
| Console `gc` | `GCUtil.ForceGarbageCollection()` → forced blocking gen2 + `WaitForPendingFinalizers` | [H] Client/Client.Commands.cs:656-658; Core/Util/GCUtil.cs:12-19 |
| Debug shutdown | double `GC.Collect`/`WaitForPendingFinalizers` to trigger finalizer-based leak asserts | [H] Client/Client.cs:623-644 |
| Before load | `DataCache.FlushReferences()` nulls sector refs in pooled entities so the previous world can be collected | [H] Client/Client.cs:462; Core/Util/DataCache.cs:76-94 |
| Load (ZDBSP) | `map.ClearAll(); map.Reject = null;` "Large UDMF maps don't seem to free the memory without this" | [H] Client/Client.Commands.cs:954-956 |

Absent (searched `grep -rn "NoGCRegion\|GCSettings\|LatencyMode\|ServerGarbage\|ConcurrentGarbage\|TieredPGO\|GCHeap"` over *.cs, *.csproj, *.props, *.targets, *.json): no `GC.TryStartNoGCRegion`, no `GCSettings.LargeObjectHeapCompactionMode`, no server-GC or concurrent-GC overrides, no GC heap-count or conserve settings, and no `runtimeconfig.template.json` (`find` found none). The runtime defaults therefore apply: workstation GC, concurrent/background enabled (**Inference:** .NET default).

**Inference:** `SustainedLowLatency` suppresses blocking gen2 collections while the game runs and still allows background gen2. The design assumes gameplay allocation is low enough that gen0 and gen1 collections stay short. The design does not assume zero allocation.

### 2.2 GC visibility

- `Developer.LogGC` (default false) [H] Core/Util/Configs/Components/ConfigDeveloper.cs:38. When enabled, `GCNotifier` creates a finalizer `Sentinel` that re-registers itself on every GC and prints "GC Event <time>" to the in-game message line [H] Core/Util/GCNotifier.cs:5-20; Client/Client.Config.cs:56-85. The callback runs on the .NET finalizer thread and calls `world.DisplayMessage` from there [H] Core/Util/GCNotifier.cs:11-16; Client/Client.Config.cs:74-85. This debug option is the one other off-thread writer into world-side state.
- `GCTracker` (per-generation `GC.CollectionCount` diffing) exists, but `grep -rn "GCTracker"` finds only its own file [H] Core/Util/GCTracker.cs:11-45, so it is unused.
- `Profiler.GarbageCollection` is an empty `GCProfiler` [H] Core/Util/Profiling/GCProfiler.cs:5-8.

### 2.3 `DynamicArray<T>` — the core container

[H] Core/Util/Container/DynamicArray.cs:21-354. It is a public-field growable array:
- Fields: `public int Length; public T[] Data; public int Capacity;` [H] Core/Util/Container/DynamicArray.cs:26-38.
- `Clear()` only sets `Length = 0` and never zeroes or frees the array [H] Core/Util/Container/DynamicArray.cs:100-103. The comment at :94-99 says references are retained.
- `Add` doubles capacity on growth [H] Core/Util/Container/DynamicArray.cs:105-111. `AddUnsafe` (`[AggressiveInlining]`, no bounds or capacity check) [H] Core/Util/Container/DynamicArray.cs:113-117. `AddMemoryCopy` uses `Buffer.MemoryCopy` for unmanaged `T` [H] Core/Util/Container/DynamicArray.cs:119-135.
- Optional `ArrayPool<T>.Shared` backing (`arrayPool: true`) [H] Core/Util/Container/DynamicArray.cs:60-69, 284-301. It is used for every entity's `IntersectSectors` (`new(16, arrayPool: true)`) [H] Core/World/Entities/Entity.cs:113.
- `capacityAlloc` pre-fills the array with objects, which is used to pre-warm pools [H] Core/Util/Container/DynamicArray.cs:71-76.
- `GetEnumerator()` is `Data.Take(Length).GetEnumerator()` [H] Core/Util/Container/DynamicArray.cs:303-311. **A `foreach` over a `DynamicArray` allocates**, so hot code uses `for (i < Length) ... Data[i]`, for example `TickScrollers` [H] Core/World/WorldBase.cs:1104-1109 and the render collection `Clear` [H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderDataCollection.cs:45-58.
- `Precondition` asserts are `[Conditional("DEBUG")]` [H] Core/Util/Assertion/Assert.cs:45-46, so they compile out of Release.

Examples of "member buffer, clear, refill" in hot paths:
- `PhysicsManager` holds eight scratch `DynamicArray`s (`m_crushEntities`, `m_sectorMoveEntities`, `m_onEntities`, `m_stackCrush`, …) that are cleared rather than reallocated [H] Core/World/Physics/PhysicsManager.cs:68-76, 205-206, 407-410, 524, 1062, 735.
- `WorldBase.TickEntities` collects fall-check entities into `m_fallCheckEntities` and clears it each tic [H] Core/World/WorldBase.cs:193, 1196, 1210-1211.
- `InputManager` keeps all key-state lists as `DynamicArray<Key>` and refills them per tic [H] Core/Window/Input/InputManager.cs:14-25, 229-264.
- Renderer: `RenderDataCollection` keeps a `LookupArray` of per-texture vertex buckets plus a `DynamicArray` of buckets to draw, and returns them to a `RenderDataPool` in `Recycle` mode [H] Core/Render/OpenGL/Renderers/Legacy/World/Data/RenderDataCollection.cs:22-58.
- `SpanString` (a DynamicArray<char> with `Buffer.MemoryCopy` append) is used for HUD text [H] Core/Strings/SpanString.cs:8-51.

### 2.4 `DataCache` — free-list object pools

`ArchiveCollection.StaticDataCache` is a **process-lifetime static** [H] Core/Resources/Archives/Collection/ArchiveCollection.cs:58; Client/Client.cs:751. It pools [H] Core/Util/DataCache.cs:35-51:
- **Entities.** A pre-allocated `Entity[1024]`, with index 0 reserved and every slot constructed up front [H] Core/Util/DataCache.cs:37, 55-73. `GetEntity` pops a free index and calls `entity.Set(...)` on the recycled object [H] Core/Util/DataCache.cs:96-123. `FreeEntity` pushes the index back [H] Core/Util/DataCache.cs:125-132. `SetEntitiesForMapLoad(count)` grows to `count*1.2` and resets the free list [H] Core/Util/DataCache.cs:134-148.
- **Audio sources.** `OpenALAudioSource` objects are recycled via `Set(...)` [H] Core/Util/DataCache.cs:181-206.
- **Entity lists, `LinkedListNode`s for specials, waiting sounds and console messages, and special objects** (`LightChangeSpecial`, `SectorMoveSpecial`, `SwitchChangeSpecial`, `StairSpecial`) [H] Core/Util/DataCache.cs:211-416. The light, sector-move and switch special pools check `GetType()` on free so that subclasses are never pooled [H] Core/Util/DataCache.cs:287, 329, 368.
- **Per-wall vertex arrays** `DynamicVertex[6]` / `SkyGeometryVertex[6]` [H] Core/Util/DataCache.cs:418-440, used by the dynamic geometry renderer [H] Core/Render/OpenGL/Renderers/Legacy/World/Geometry/GeometryRenderer.cs:260, 283, 2027, 2213.

`LinkableList<T>` adds a per-closed-generic **static** node pool pre-filled with 1024 nodes [H] Core/Util/Container/LinkableList.cs:42-54, 135. It backs sector↔entity links (`centerSector.Link(entity)` [H] Core/World/Physics/PhysicsManager.cs:1468).

**Inference:** these are free lists (LIFO stacks) of class instances, not arenas. They remove allocation and GC pressure. They do nothing for memory locality: recycled entities sit wherever the original `new Entity()` put them, and ticking walks an intrusive linked list `EntityManager.Head → entity.Next` [H] Core/World/WorldBase.cs:1169-1206. The static pools are not thread-safe. Today only the main thread uses them (the load thread builds a world through the same static `DataCache` while the main thread renders the loading screen, see §4, and that has not been audited).

### 2.5 Avoiding delegate and closure allocations

- `PhysicsManager` caches `Func<Entity, GridIterationStatus>` delegates for blockmap traversal in its constructor [H] Core/World/Physics/PhysicsManager.cs:82-84, 95-97. It also caches `Comparison<Entity>` for sector-move sort [H] Core/World/Physics/PhysicsManager.cs:73.
- Layer and world render actions are cached (see §1.6).

### 2.6 Cross-map reuse via statics

- `WorldBase.LastStructLines` is a static `DynamicArray<StructLine>` that is cleared and resized per map instead of reallocated [H] Core/World/WorldBase.cs:91, 135, 459-462.
- `WorldLayer.LastMapGeometry` and `LastIslandGeometry` are statics [H] Core/Layer/Worlds/WorldLayer.cs:45-46.
- `WorldStatic` holds 67 public static fields (plus one helper method): the current `World`, `Random`, `DataCache`, a shared `DynamicArray<BlockmapIntersect> Intersections`, and config flags flattened into plain `bool`/`short` (e.g. `SlowTickEnabled`, `InfinitelyTallThings`, `MissileClip`) [H] Core/World/WorldStatic.cs:14-39.

**Inference:** this turns hot config checks into static loads instead of `ConfigValue<T>.Value` property calls (which are cheap auto-properties anyway [H] Core/Util/Configs/Values/ConfigValue.cs:33, 90). It hard-wires "one world per process".

### 2.7 Remaining per-frame allocations found

- HUD FPS text: `str.Append($"{Math.Min((int)Math.Round(fps), 9999),4:####}")` builds a `string` and passes it as `ReadOnlySpan<char>` [H] Core/Layer/Worlds/WorldLayer.Render.Hud.cs:440-451. `SpanString` has no interpolated-string-handler overload (`grep -rn InterpolatedStringHandler Core` finds nothing; its Append overloads are at [H] Core/Strings/SpanString.cs:30, 53, 58). This costs one string per FPS line per frame when `Hud.ShowFPS` is on.
- `Task.Run` for a screenshot is per event, not per frame [H] Client/Client.cs:339-353.
- Savegame `world.ToWorldModel()` builds a complete object graph on the main thread before the serialisation task starts [H] Core/World/Save/SaveGameManager.cs:117-129. Quick-save timer: [H] Core/Layer/GameLayerManager.cs:247-265.

**Inference:** a full per-frame allocation audit needs a profiler run (`dotnet-counters`/`dotnet-trace` alloc ticks), which is not possible here. The static review found mostly clean hot paths. LINQ appears in cold paths: the only hits in physics, entities, WorldBase and renderers are at load or cheat time [H] Core/World/WorldBase.cs:1491, 4410-4411, 4863; Core/World/Entities/EntityManager.cs:369; Core/Render/OpenGL/Renderers/TransitionPrograms.cs:86.

---

## 3. Data-oriented patterns

Counts come from `grep -rn -e PATTERN --include='*.cs' Core Client`:

| Pattern | Files / lines |
|---|---|
| `[MethodImpl(AggressiveInlining)]` | 39 files / 482 lines (358 in `EntityFlags.cs`, 28 in `Entity.cs`, 17 in `MathHelper.cs`, 7 in `StructLine.cs`, 6 in `PhysicsManager.cs`) |
| `AggressiveOptimization`, `SkipLocalsInit`, `InlineArray`, `ref readonly` | 0 |
| `[StructLayout]` | 46 (vectors/boxes/segments `Pack=4/8`, GPU vertex structs `Pack=1`, `BspNodeCompact`, `SubsectorSegment`, render commands) |
| `unsafe ` | 35 files / 73 lines; `fixed (` 11 files / 27 lines |
| `stackalloc` | 3 sites (physics corners, a clear-colour array, a language lower-case buffer) |
| `Span<` | 74 files / 318 lines |
| `Unsafe.*` / `MemoryMarshal` / `CollectionsMarshal` | 5 / 2 / 1 files |
| SIMD (`Vector128/256`, `System.Numerics.Vector`, `Intrinsics`) | 0 |
| `ObjectPool` (MS) | 0; `ArrayPool` 3 files |

Concrete patterns:

1. **Bit-packed flags as value types.** `EntityFlags` is a `struct` of three `int`s with hundreds of `[AggressiveInlining]` get/set/clear accessors [H] Core/World/Entities/Definition/Flags/EntityFlags.cs:8-80. `StructLineFlags` is packed into `StructLine.Flags` [H] Core/World/Geometry/Lines/StructLine.cs:10-51.
2. **Struct arrays for hot geometry (array-of-structs, not struct-of-arrays).** `StructLine` is a `record struct` copy of the per-line data needed in hot loops (segment, front/back sector and planes, flags, lock key) [H] Core/World/Geometry/Lines/StructLine.cs:23-88. It is stored in `DynamicArray<StructLine>` and accessed via `ref var line = ref world.StructLines.Data[i]` [H] Core/Render/Renderer.World.cs:135-149; Core/World/Impl/SinglePlayer/AutomapMarker.cs:253. It still holds managed references (`Sector`, `SectorPlane`, `Line`), so it is not blittable.
3. **Compact BSP.** `BspNodeCompact` is `[StructLayout(Sequential)] unsafe struct` with `Vec2D SplitStart, SplitDelta; Box2D BoundingBox; fixed uint Children[2]`, and the high bit tags subsector children [H] Core/World/Bsp/BspNodeCompact.cs:11-56. Traversal is branchless on side choice: `int front = *(byte*)&onRight; ... node.Children[front ^ 1]` [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:201-218.
4. **`ref` locals** everywhere in hot loops (`ref var` 164 lines across 40 files). Example: `ref var node = ref world.BspTree.Nodes[nodeIndex]` [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:205. `CollectionsMarshal.GetValueRefOrAddDefault` is used for scroll accumulators to avoid a double dictionary lookup [H] Core/World/WorldBase.cs:1109, 4831.
5. **`stackalloc` scratch**: `Span<Vec2D> corners = stackalloc Vec2D[3]` in the XY move tracer [H] Core/World/Physics/PhysicsManager.cs:1927.
6. **Raw memory clears**: `Unsafe.InitBlockUnaligned` to zero vertex arrays [H] Core/Util/Container/ArrayExtensions.cs:9-10; Core/Render/OpenGL/Renderers/Legacy/World/Geometry/Static/StaticCacheGeometryRenderer.cs:1590.
7. **Entities stay classes** (`public partial class Entity`, 1418 lines, intrusive `Next`/`Previous` [H] Core/World/Entities/Entity.cs:40, 52-53). There is no struct-of-arrays entity storage. `grep -rn "SparseSet<"` outside the container folder finds no users, so `SparseSet<T>` [H] Core/Util/Container/SparseSet.cs:5 is unused. `LookupArray<T>` (id-indexed arrays) is used for entity lookup, players and inventory [H] Core/World/Entities/EntityManager.cs:47-48; Core/World/Entities/Inventories/Inventory.cs:55; Core/World/Entities/Inventories/Weapons.cs:21.

**Inference:** the data-oriented work is local: compact BSP nodes, flat line structs, bit flags, no-alloc containers. The entity/actor model is still object-graph-and-pointer-chasing, and nothing is vectorised. For a C++ VR engine the transferable ideas are the *flattened per-line/per-node hot data* and *free-list recycling*, not the C#-specific mechanics.

---

## 4. Threading

`grep -rn "new Thread(\|Task\.Run\|Task\.Factory\|Parallel\.\|ThreadPool\|lock (\|Interlocked\|volatile \|ConcurrentQueue"` over Core and Client gives the complete inventory:

| Work | Mechanism | Citation |
|---|---|---|
| IWAD discovery, archive loading | `await Task.Run(FindInstalledIWads)`, `await Task.Run(LoadFiles)` | [H] Client/Client.Initialize.cs:40, 58 |
| **Map load**: find map, run **ZDBSP**, build `WorldLayer`/world | `Task.Run(() => LoadMap(...))`; sets `m_loadComplete = true`; main thread polls it in `CheckLoadMapComplete` and does GL upload (`Renderer.UpdateToNewWorld`) | [H] Client/Client.Commands.cs:914-920, 921-1031; Client/Client.cs:468-532 |
| Savegame serialise + write | `await Task.Run(m_saveFunc)`; the world snapshot `ToWorldModel()` is taken on the main thread first | [H] Core/World/Save/SaveGameManager.cs:117-135 |
| Screenshot PNG encode | `Task.Run(() => image.SavePng(path))` after `GetMainFramebufferData` (which does `GL.Finish`) | [H] Client/Client.cs:351-352; Core/Render/Renderer.cs:784-786 |
| Music stream buffer refill | `Task.Run(PollingLoop)` that sleeps `m_bufferRefreshInterval` | [H] Client/Music/AudioStream.cs:156, 192-215 |
| Music play queue | `LongRunning` task, `ConcurrentQueue`, `Thread.Sleep(10)` poll | [H] Client/Music/MusicPlayer.cs:29, 48-49, 204-223 |
| Audio device-change watch | `LongRunning` task, 1 s sleep | [H] Core/Audio/Impl/OpenALAudioSystem.cs:62-63, 221-235 |
| Discord presence | `Task.Run` | [H] Client/Discord/DiscordHandler.cs:28 |
| **AutomapMarker** (gameplay) | dedicated background `Thread` | [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:56-73 |
| Gyro calibration dialog | `System.Timers.Timer(1000)` callback (options UI only) | [H] Core/Layer/Options/Dialogs/GyroCalibrationDialog.cs:33-35 |
| SFX mixing | inside OpenAL Soft (native), not Helion code | — |

**Main-thread-only:** all simulation (`World.Tick`), all GL, input, sound-source management. The comment "OpenGL can't do things outside of the main thread" [H] Client/Client.Commands.cs:918 matches the code: load completion is marshalled by a polled bool. The `await` continuation that sets `m_loadComplete` runs on a pool thread, because no `SynchronizationContext` is installed (`grep -rn SynchronizationContext` finds nothing), and the field is not `volatile` [H] Client/Client.cs:76.

**AutomapMarker in detail.**
- `SinglePlayerWorld.Tick` enqueues the camera position every tic [H] Core/World/Impl/SinglePlayer/SinglePlayerWorld.cs:230-236. In `Adaptive` render mode it enqueues only when the queue is empty [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:108-112.
- The thread dequeues positions, builds a frustum, and does a front-to-back BSP walk with an angular `ViewClipper` to mark visible lines as seen for the automap. It also records timing, subsector, seg and line counts as `IBspHeuristics` [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:114-162, 201-280.
- Those heuristics drive `Render.Mode = Adaptive`, which switches between the static-geometry renderer and BSP-culled rendering by median BSP time [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.BspHeuristics.cs:19-94.
- Idle behaviour: `Thread.Sleep(remaining of 28 ms)` in Static/BSP mode, but `Thread.Yield()` in Adaptive mode [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:154-161.
- **Synchronisation hazard (fact):** the thread performs `line.Flags |= StructLineFlags.SeenForAutomap; line.Line.DataChanges |= LineDataTypes.Automap;` [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:262-266 with no lock. `line` is a `ref` into the shared `world.StructLines.Data` array [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:222, 253. The main thread does read-modify-write on both of the same fields during gameplay:
  - `Line.DataChanges`: `|= Texture` [H] Core/World/Geometry/Sides/Side.cs:171; `Activated`/`EverActivated` [H] Core/World/Geometry/Lines/Line.cs:255-262; `BlockFlags`/`Special`/`Args`/`BlockSound` [H] Core/World/WorldBase.cs:4278, 4298-4299, 4341; `Args` [H] Core/World/Special/SpecialManager.cs:2406.
  - `StructLine.Flags`: `|=`/`&= ~StructLineFlags.MarkAutomap` from the mark-specials feature [H] Core/World/Impl/SinglePlayer/MarkSpecials.cs:436-442.
  - The thread also reads sector heights via `RenderBlock.IsBlocked` while the main thread moves sectors [H] Core/World/Impl/SinglePlayer/AutomapMarker.cs:242-244.
- **Inference:** non-atomic `|=` on both threads can lose a flag bit. A lost `Line.DataChanges` bit (for example `Texture` or `Activated`) means a missing savegame delta. A lost `StructLine.Flags` bit means wrong automap state (`SeenForAutomap`/`MarkAutomap`). Racy height reads only mis-mark automap lines. In Adaptive mode the `Thread.Yield` loop keeps a core busy.

Locks exist only around console message lists [H] Core/Util/Consoles/HelionConsole.cs:187, 197; Core/Layer/Worlds/WorldLayer.Render.Hud.cs:140, 829, 903; Core/Layer/Consoles/ConsoleLayer.Render.cs:115, the text-render string caches [H] Core/Render/OpenGL/Texture/Fonts/RenderableString.Cache.cs:19-105, and temp files.

**Absent:** no parallel simulation, no parallel culling, no render worker or command-recording thread, and no `Parallel.For` (searched as above).

---

## 5. Built-in profiling and instrumentation

- **CPU stopwatches.** `Profiler` contains `Global`, `Input`, `Logic`, `Render` (a `RenderProfiler`) and `World` (a `WorldProfiler`) [H] Core/Util/Profiling/Profiler.cs:17-22. Each `ProfilerStopwatch` wraps `Stopwatch` with per-frame and accumulated ticks [H] Core/Util/Profiling/ProfilerStopwatch.cs:6-38.
- **Timed regions.**
  - Render: `Total`, `SwapBuffers` and `FlushPipeline` in Client [H] Client/Client.cs:367-383; `World`, `Automap` and `Hud` in WorldLayer [H] Core/Layer/Worlds/WorldLayer.Render.cs:26-68; `MiscLayers` in the layer manager [H] Core/Layer/GameLayerManager.cs:787-789; `WorldTraversal`, `WorldGeometry`, `WorldFloodFill` and `WorldTransparent` in the legacy world renderer [H] Core/Render/OpenGL/Renderers/Legacy/World/LegacyWorldRenderer.cs:333-340, 345-426, 434-439, 595-694.
  - Simulation: `World.Total`, `TickPlayer` and `TickEntity` [H] Core/World/WorldBase.cs:1051-1101, 1169-1212, 1238-1268.
  - `Logic` and `Global` in Client [H] Client/Client.cs:328-330, 392-404. `Input` is never started: `grep -rn "\.Input\.Start"` finds nothing.
- **Draw counts.** `DrawCounts` struct counters (static/dynamic geometry, flood fill, skies, sprites, HUD) are incremented at draw sites [H] Core/Util/Profiling/Timers/RenderProfiler.cs:5-17, snapshotted into `Renderer.LastDrawCounts` [H] Core/Render/Renderer.cs:55, 617, and printed by console command `drawcounts` [H] Client/Client.Commands.cs:660-673.
- **Spike logging.** `Developer.ProfilerTimeTrigger` (ms, default 0 = off) [H] Core/Util/Configs/Components/ConfigDeveloper.cs:41. When set, `ResetTimers` raises `TimeThresholdTriggered` for any registered stopwatch over the threshold, and Client logs it with its siblings [H] Core/Util/Profiling/Profiler.cs:45-57; Client/Client.cs:170-185. Only `Render.*` and `World.*` stopwatches are registered as `ProfilerPath`s [H] Core/Util/Profiling/Timers/RenderProfiler.cs:38-51; WorldProfiler.cs:13-18. `ProfilerStopwatch.Profilers` is an empty list [H] Core/Util/Profiling/ProfilerStopwatch.cs:18, so `Global`, `Logic` and `Input` never trigger.
- **Exit summary.** `m_profiler.LogStats()` writes accumulated totals via reflection over public fields [H] Client/Client.cs:572; Core/Util/Profiling/ProfileComponent.cs:21-52.
- **FPS.** `FpsTracker` reports average, min and max over 600 ms windows [H] Core/Util/Timing/FpsTracker.cs:14, 30-74.
- **GPU timing.** A `GLElapsedTimeQuery` class exists (`GL.BeginQuery(QueryTarget.TimeElapsed)`) [H] Core/Render/OpenGL/Queries/GLElapsedTimeQuery.cs:7-37, but `grep -rn GLElapsedTimeQuery` finds no user. **There is no GPU timing in use.**
- **Absent:** no frame-time histogram or percentiles, no Tracy/ETW/EventSource/ActivitySource markers (`grep -rn "EventSource\|Tracy\|ActivitySource" --include='*.cs' Core Client` returns nothing; the profiler classes above are the whole system), no per-frame allocation counter (`grep -rn GetAllocatedBytes` finds nothing).

---

## 6. Build and runtime configuration

- Target `net10.0`, nullable on [H] Directory.Build.props:3, 7. SDK pinned `10.0.0` with `rollForward: latestMinor` [H] global.json:2-5.
- All non-test projects set `IsTrimmable` and `IsAotCompatible` [H] Directory.Build.targets:6-9. `AllowUnsafeBlocks` is on in Core and Client [H] Core/Core.csproj:10; Client/Client.csproj:11.
- **Two release flavours**, both built by CI [H] .github/workflows/Release.yml:48, 52; Nightly.yml:40, 44:
  - `SelfContainedRelease=true`: SelfContained, PublishSingleFile, **PublishReadyToRun**, compressed single file, PublishTrimmed [H] Client/Client.csproj:14-20.
  - `AOT=true`: **PublishAot**, `OptimizationPreference=Speed`, `DefineConstants=AOT`. `IlcInstructionSet` is **commented out** (the line reads `<!--<IlcInstructionSet>x86-x64-v3</IlcInstructionSet>-->`) [H] Client/Client.csproj:22-35. Native libs (GLFW, SDL2, OpenAL, ZMusic, HelionACS) are statically linked with `DirectPInvoke` [H] Client/Client.csproj:43-49, 126-144. `Justfile` and the AppImage script publish AOT [H] Justfile:1-2; Scripts/buildAndPrepareAppDir.sh:15.
- **Not set anywhere** (grep over csproj/props/targets/json): `TieredPGO`, `TieredCompilation*`, `ServerGarbageCollector`, `ConcurrentGarbageCollection`, `InvariantGlobalization`, `IlcInstructionSet` (only the comment). Code analysis uses `AnalysisModePerformance=Recommended` [H] Directory.Build.targets:27-31. That is analysis only and has no codegen effect.
- `ProfileComponent` uses `[DynamicallyAccessedMembers(PublicFields)]` so that its reflection survives trimming and AOT [H] Core/Util/Profiling/ProfileComponent.cs:17.
- Windows: `timeBeginPeriod` [H] Client/Client.cs:187-197. A laptop dGPU preference is written to `HKCU\Software\Microsoft\DirectX\UserGpuPreferences`, followed by a self-restart on first run [H] Client/Client.cs:199-219; Client/LaptopGpuSettings.cs:80, 107-112.
- GL context: tries 4.5 → 4.4 → 3.3 core and uses `ClipControl`/`MapPersistentBit`/`MemoryBarrier` when they are available [H] Client/Client.cs:86-91, 249-288.

**Inference:**
- The AOT build has no JIT, so no tiered or dynamic PGO. Codegen targets the default baseline ISA because `IlcInstructionSet` is commented out, which means no AVX2/FMA, though the code has no explicit SIMD anyway.
- The R2R single-file build starts from precompiled code and then re-JITs hot methods at tier 1 with .NET's default dynamic PGO.
- This is the only place in Helion where "which build" changes steady-state CPU speed. A C++ port gets the equivalent through compiler flags (`-O2`, `-march`, LTO/PGO).

---

## 7. Config options that affect performance

Covered in detail above; consolidated in "Perf-relevant settings" below. Render-quality options such as vanilla sprite clipping, anisotropy and max distance belong to the renderer sections and are listed only for completeness.

---

## 8. BenchmarkSuite

- `Program.Main` → `BenchmarkSwitcher.FromAssembly(...).Run(args)` [H] BenchmarkSuite/Program.cs:5-10. It references BenchmarkDotNet 0.15.2 and the VS DiagnosticsHub diagnosers package [H] BenchmarkSuite/BenchmarkSuite.csproj:8-9, plus the `Tests` project, which it uses for `WorldAllocator.LoadMap` [H] BenchmarkSuite/BenchmarkSuite.csproj:12-13.
- Worlds load headless through `WorldAllocator.LoadMap` with `MockAudioSystem`, a fresh `Profiler` and `DataCache`, and ZDBSP [H] Tests/Unit/GameAction/WorldAllocator.cs:40-100. `WorldAllocator` calls `world.DisableAutomapMarker()` before `world.Start` [H] Tests/Unit/GameAction/WorldAllocator.cs:112-113, so the background AutomapMarker thread never runs in the benchmarks. Setup runs in **constructors**: there is no `[GlobalSetup]`, `[IterationSetup]`, `[MemoryDiagnoser]`, `[Params]` or custom config (`grep -rn "Diagnoser\|GlobalSetup\|IterationSetup\|Params\|SimpleJob\|Config("` over BenchmarkSuite finds nothing). The diagnoser package is therefore inert unless enabled on the command line, and **allocation is not reported by default**. Reports are BenchmarkDotNet's default mean, error and stddev.
- Maps ship as zips in BenchmarkSuite/Resources, each holding one WAD and loaded as MAP01 (directory listing; e.g. [H] BenchmarkSuite/EntityCreate.cs:15): `idumea.zip` (765 KB), `helion-castle.zip` (1.9 MB, 3D floors), `1024enemies.zip`, `1024enemieswithpickups.zip`, `floatmove.zip`.

| Benchmark | Map | Work per invocation | Citation |
|---|---|---|---|
| `EntityCreate.CreateEntities` | idumea, `SetEntitiesForMapLoad(16384)` | spawn 16,000 `DoomImpBall` | [H] BenchmarkSuite/EntityCreate.cs:13-37 |
| `EntityFireHitScan.FireHitScan` | idumea | 10 × hitscan from every non-closet monster toward the player | [H] BenchmarkSuite/EntityFireHitScan.cs:17-35 |
| `EntityLineOfSight2D.LineOfSight` | idumea | 100 × `CheckLineOfSight(monster, player)` | [H] BenchmarkSuite/EntityLineOfSight2D.cs:15-32 |
| `EntityLineOfSight3D.LineOfSight` | helion-castle | 160 × LOS | [H] BenchmarkSuite/EntityLineOfSight3D.cs:15-32 |
| `EntityMoveGeneral2D/3D.SetNewChaseDirection` | idumea / helion-castle | 50 / 26 × `SetNewChaseDirection` | [H] BenchmarkSuite/EntityMoveGeneral2D.cs:28-36; EntityMoveGeneral3D.cs:28-36 |
| `EntityOnEntityCollisionOnly` | 1024enemies | 100 × chase-dir | [H] BenchmarkSuite/EntityOnEntityCollisionOnly.cs:28-36 |
| `EntityOnEntityCollisionWithMissile` | 1024enemies | 100 × (`A_TroopAttack` + chase-dir) | [H] BenchmarkSuite/EntityOnEntityCollisionWithMissile.cs:29-40 |
| `EntityOnEntityCollisionWithPickups` | 1024enemieswithpickups | 100 × chase-dir | [H] BenchmarkSuite/EntityOnEntityCollisionWithPickups.cs:28-36 |
| `EntityOnEntityFloatCollision` | floatmove | 50 × (raise player Z, chase-dir) | [H] BenchmarkSuite/EntityOnEntityFloatCollision.cs:28-38 |

**Inference on validity:** world state is never reset between iterations. `EntityCreate` keeps adding 16k entities each invocation, and the chase benchmarks keep moving monsters. Later iterations therefore measure a different world from earlier ones, so treat the numbers as relative regression signals only. Nothing benchmarks `World.Tick()` as a whole, rendering, frame time or GC pauses.

---

## Absent / not done (proven by search)

- **No multi-tic catch-up.** Ticker caps at 1 tic per frame (code above). `TickOverflowThreshold` is effectively unreachable outside demo fast-forward.
- **No GC no-region or GC tuning knobs.** `grep -rn "NoGCRegion\|ServerGarbage\|ConcurrentGarbage\|TieredPGO\|InvariantGlobalization\|GCHeap"` over *.cs/*.csproj/*.props/*.targets/*.json finds nothing. No runtimeconfig template (`find -name "runtimeconfig*.json"` returns nothing).
- **No SIMD.** `grep -rn "Vector128\|Vector256\|Numerics.Vector\|Vector<\|Intrinsics" --include='*.cs' Core Client` returns 0 hits.
- **No parallelism in tick or render.** No `Parallel.`, no render or culling worker. The only gameplay thread is AutomapMarker.
- **No GPU fences or frame-in-flight control.** `grep -rn "FenceSync\|ClientWaitSync" Core` returns nothing. The only throttle is `GL.Finish` when `ForcePipelineFlush` is set.
- **No GPU timing in use.** `GLElapsedTimeQuery` has no callers.
- **No allocation telemetry.** `GetAllocatedBytes`/`GetTotalAllocatedBytes` are not used. `GCTracker` is unused. Benchmarks lack `[MemoryDiagnoser]`.
- **No frame-time percentile reporting.** Only 600 ms avg/min/max FPS.
- **No late input latching.** Key state is consolidated once per tic. There is no re-sample just before render. Mouse delta is consumed in `RunLogic`, before `Render`, so the next callbacks wait a frame.
- **No pose prediction or late-latch hook.** The camera comes from `GetCamera(fraction)` before the draw is recorded, and nothing re-samples it afterwards.
- **No `SynchronizationContext`** for async continuations. Completion is signalled by a polled non-volatile bool.
- **Dead code:** `Client.HandleWinMouseMove`, `GCTracker`, `SparseSet<T>`, `GLElapsedTimeQuery`, `Profiler.Input`.

## Perf-relevant settings

| Setting | Default | Effect | Citation |
|---|---|---|---|
| `render.vsync` | `On` | swap interval. With `maxfps=0` it also arms the software limiter at the monitor refresh rate | [H] Core/Util/Configs/Components/ConfigRender.cs:98-100; Client/Window.cs:463-493 |
| `render.maxfps` | `0` (1–34 → 35) | software limiter target. `35` disables interpolation (fraction = 1, no angle lerp) | [H] ConfigRender.cs:102-112; WorldLayer.Logic.cs:88-90; Player.cs:992-998 |
| `render.forcepipelineflush` | `false` | `glFinish` after swap | [H] ConfigRender.cs:220-221; Client/Client.cs:376-379 |
| `render.mode` | `Static` (Static/BSP/Adaptive) | geometry path. Adaptive uses AutomapMarker heuristics and a `Thread.Yield` loop | [H] ConfigRender.cs:91-94; AutomapMarker.cs:154-161 |
| `render.adaptive.timethreshold / hysteresispercent / segthreshold / timewindow / switchcount` | 2200 µs / 0.05 / 8000 / 10 / 3 | adaptive switching | [H] ConfigRender.cs:240-256 |
| `render.cachesprites` | `true` | preload sprites to avoid hitches (restart) | [H] ConfigRender.cs:217-218 |
| `render.postprocessingeffects` | `true` | off = skip FBO (restart) | [H] ConfigRender.cs:235-237 |
| `render.vanillarender`, `render.downscalevanillarendersamplebuffer` | `false`, `1` | software sprite-clip emulation cost | [H] ConfigRender.cs:178-184 |
| `render.anisotropy`, `render.maxdistance` | `8`, `0` | GPU cost / distance cull | [H] ConfigRender.cs:117-119, 134-136 |
| `window.virtual.enable` / `.dimension` | `false` / 800×600 | internal render resolution | [H] Core/Util/Configs/Components/ConfigWindow.cs:58-64 |
| `window.state`, `window.dimension`, `window.colormode`, `window.clearscreen` | Fullscreen, 1024×768, TrueColor, false | | [H] ConfigWindow.cs:83, 91, 105, 113 |
| `window.laptopgpu` | `HighPerformance` (Windows) | dGPU preference registry | [H] ConfigWindow.cs:117; Client/LaptopGpuSettings.cs:80 |
| `mouse.interpolate` | `false` | true = lerp mouse yaw instead of applying it immediately | [H] Core/Util/Configs/Components/ConfigMouse.cs:34-36; Player.cs:727-731 |
| `slowtick.enabled` (+ distance 2000, multipliers 4/2/4, chase-fail skip 4) | `false` | reduces AI cost for distant actors | [H] Core/Util/Configs/Components/ConfigSlowTick.cs:12-34 |
| `developer.loggc` | `false` | GC event messages | [H] ConfigDeveloper.cs:38 |
| `developer.profilertimetrigger` | `0` | log Render/World stopwatch spikes over N ms | [H] ConfigDeveloper.cs:41 |
| `hud.showfps` / `hud.showminmaxfps` | `false` / `false` | per-frame string allocation per FPS line when on | [H] Core/Util/Configs/Components/ConfigHud.cs:242, 246; WorldLayer.Render.Hud.cs:423-451 |
| Build: `-p:AOT=true` vs `-p:SelfContainedRelease=true` | — | NativeAOT (Speed) vs R2R+JIT | [H] Client/Client.csproj:14-35 |

## Transfer notes for UZDoom VR

**Main loop and tic pacing.** UZDoom's `D_DoomLoop` runs `TryRunTics()`, then `I_StartTic`, `D_ProcessEvents`, `D_Display` [U] src/d_main.cpp:1437-1467. `TryRunTics` starts with `GC::CheckGC()`, optionally waits for a tic, and computes `totalTics = EnterTic - LastEnterTic`, so it **runs multiple tics to catch up** [U] src/d_net.cpp:2190-2216, 2312-2335. `singletics` caps it to 1 [U] src/d_net.cpp:2214-2215, and the netcode may add one extra tic to catch up [U] src/d_net.cpp:2259-2265. Fraction comes from `I_GetTimeFrac` against a fixed frame start time [U] src/common/utility/i_time.cpp:170-177.
- Helion's "never more than one tic" is the opposite policy.
- **Grade: Port.** Keep UZDoom's catch-up (needed for netplay and timing) but cap the burst per frame, as Helion does in spirit, so a hitch in VR does not become a multi-tic stall on the next frame.
- For stereo, compute the tic fraction **once per frame from the predicted display time** (`XrFrameState.predictedDisplayTime`), not from `CurrentFrameStartTime`, and use the same fraction for both eyes.

**Frame limiter and vsync.** UZDoom: `DFrameBuffer::FPSLimit` uses a sleep that wakes 2 ms early plus `sleep_for(0)` spinning [U] src/common/rendering/v_framebuffer.cpp:243-278. `vid_maxfps` defaults to 500 [U] src/common/rendering/v_video.cpp:71-81. The Vulkan backend calls it only when vsync is off [U] src/common/rendering/vulkan/system/vk_commandbuffer.cpp:176-178. The OpenGL backend calls it before every `SwapBuffers` whatever the vsync setting, and also uses buffer sync objects (`DropSync`/`WaitSync`) when `gl_pipeline_depth ≥ 1` [U] src/common/rendering/gl/gl_framebuffer.cpp:248-268. Helion's limiter has the same sleep-then-yield structure plus drift carry-over and also runs with vsync on.
- **Grade: Rework.** In VR both limiters and desktop vsync must be bypassed. `xrWaitFrame` blocks to the compositor cadence, and `xrBeginFrame` and `xrEndFrame` bracket submission.
- Helion's drift-carry trick is irrelevant there. It is useful only for the desktop mirror window, which should never pace the VR loop.

**Interpolation and input split.** Both engines apply mouse look per event or frame outside the tic. UZDoom does it via `G_AddViewAngle`/`G_AddViewPitch` in the event dispatcher [U] src/d_main.cpp:3006-3022. Helion does it via `ViewAngleRadians`, folded into the tic command later [H] SinglePlayerWorld.cs:396-407.
- **Grade: Rework.** VR adds a third input class: head pose. It must bypass both the tic and the mouse accumulator.
- The render camera becomes `tic-interpolated body position + body yaw (mouse/stick) + HMD pose`, re-sampled as late as possible (ideally after culling, before command submission).
- Gameplay (aiming and hitscan direction) should take the HMD or controller pose at tic time through the tic command, the same way Helion folds `ViewAngleRadians` into `MouseAngle`.
- Helion's "key state once per tic, mouse every frame" model is a clean template for this split.

**Allocation discipline.** UZDoom is C++ and has no CLR GC. Its equivalent hazard is the **ZScript/DObject incremental GC**, stepped once per `TryRunTics` [U] src/d_net.cpp:2192; src/common/objects/dobjgc.cpp:155-163.
- Helion-specific mechanics (`SustainedLowLatency`, forced gen2 after load) do not transfer.
- Two ideas do: **(a) do a full collection at map load and keep steady-state steps small and bounded**, the `FullGC` analogue [U] src/common/objects/dobjgc.cpp:532; **(b) reuse per-frame buffers by length reset**, as in Helion's `DynamicArray.Clear` [H] Core/Util/Container/DynamicArray.cs:100-103 and `TArray::Clear` semantics.
- **Grade: Port.** For 90–120 Hz, make sure GC steps never run inside the render-critical window. Run them right after `xrEndFrame`, or with a time budget.
- Helion's `DataCache` free lists map to slab or free-list allocators for actors, specials and sound channels in C++, which UZDoom largely already has. **Grade: Port** where profiling shows `new`/`delete` in tic hot paths.

**Data-oriented layout.** Helion's `BspNodeCompact` (32-bit tagged children, inline bbox) and `StructLine` (hot line data in one flat struct) [H] Core/World/Bsp/BspNodeCompact.cs:11-56; Core/World/Geometry/Lines/StructLine.cs:23-88 are directly portable ideas.
- **Grade: Port** for flat hot-data mirrors used by culling and physics.
- **Rebuild** if the goal is a real struct-of-arrays actor model. Neither engine has one.
- Stereo-specific: one BSP or cull traversal with a **combined-eye frustum**, so per-node data is touched once for both eyes. Compact nodes make that traversal cheaper.

**Threading.**
- UZDoom already has a hardware-renderer BSP worker: `gl_multithread` defaults to true and runs `HWDrawInfo::WorkerThread` via a one-thread `ctpl::thread_pool` [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:43, 49-50, 111-117, 1024-1030. It also has software-renderer drawer threads [U] src/common/rendering/r_thread.h:40-43; src/rendering/swrenderer/scene/r_scene.cpp:41.
- Helion has none on the render path. Its only gameplay thread (AutomapMarker) is an unsynchronised side channel, and its async map load requires main-thread GL finalisation.
- **Grade: Rework.** For VR, keep the simulation single-threaded (determinism and demos). Move culling and command recording to workers: Vulkan secondary command buffers or per-thread pools, one per eye or per pass. Keep UZDoom's worker-may-not-call-API rule (`isWorkerThread` assert, hw_bsp.cpp:117).
- Helion's AutomapMarker is the cautionary example: any background reader of world state needs a snapshot or double-buffered tic state, not racy reads.

**Profiling.**
- Helion: stopwatch trees, draw counts, spike trigger, no GPU timing.
- UZDoom has its own `cycle_t` clocks (e.g. `WTTotal.Clock()` in the BSP worker [U] src/rendering/hwrenderer/scene/hw_bsp.cpp:116). Its stat system was not inventoried in this section.
- **Grade: Port.** Add Vulkan timestamp queries per pass and per eye, CPU frame-phase markers aligned with `xrWaitFrame`/`xrBeginFrame`/`xrEndFrame`, and percentile frame-time reporting (p50/p99 at 90 Hz). Helion's "log when a stage exceeds N ms" trigger is worth copying as a cheap spike detector.

**Build configuration.** Helion's only perf-relevant build switches are AOT vs R2R. The C++ counterparts are `-O3`/`-march=x86-64-v3`/LTO/PGO in CMake. **Grade: Port** (build-system only). Note Helion leaves `IlcInstructionSet` commented out, so it gets no AVX2 baseline. A VR-only build can safely target v3, because VR-capable PCs are all Haswell or newer (Inference).

**Benchmarks.** Helion's BenchmarkDotNet suite (AI, LOS, hitscan micro-benchmarks on stress maps) is a useful *list of stress cases* to replicate: 1024-monster maps, a 3D-floor map, float collisions. It is not a frame-time harness. **Grade: Rebuild** for VR: a deterministic demo-playback benchmark that records per-frame CPU tic, cull, record and GPU per-eye times plus missed-frame counts at 90 and 120 Hz, using `timedemo`-style playback.

## Open questions

- OpenTK 4.9.4 `GameWindow.Run` internals (exact `GLFW.PollEvents`/`NewInputFrame` order relative to `RenderFrame`, and whether `UpdateFrequency = 0` also fires an unused `UpdateFrame` each loop) could not be verified because the package source is not in the repo or the NuGet cache.
- Whether the map-load thread mutating the static `ArchiveCollection.StaticDataCache` (entity pre-allocation via `SetEntitiesForMapLoad`, called from world construction) can race with main-thread users during the loading screen. The loading layer renders but does not tick a world, so probably not, but this was not fully traced.
- Actual gen0/gen1 collection frequency and pause lengths during gameplay. That needs a runtime trace (`dotnet-counters`, `DOTNET_gcServer=0` default), which was not run here.
- How severe the `Line.DataChanges` lost-update race is in practice: how often the AutomapMarker write coincides with main-thread line changes, and whether a lost `Automap` or `Texture` bit can corrupt a savegame delta.
- UZDoom's stat or profiling infrastructure and its full input path (`I_GetInputFrac` [U] src/common/utility/i_time.cpp:203-209 and its callers) were only sampled, not fully traced.
