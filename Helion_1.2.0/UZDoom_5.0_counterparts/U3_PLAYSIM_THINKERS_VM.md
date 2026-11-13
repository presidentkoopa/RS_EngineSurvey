# U3 — UZDoom 5.0 playsim and ZScript VM cost

UZDoom 5.0 runs the whole playsim on the main thread at 35 Hz, in the same loop iteration as the frame it renders (`TryRunTics()` then `D_Display()`). Every actor in a thinking statnum is ticked every tic, with no distance, visibility or idle culling. Each tick enters through the ZScript VM calling convention, even when the target is the native `AActor::Tick`. JIT compilation (asmjit) is built only for x86_64 and is on by default there. The per-call VM overhead on the native path is therefore small but not zero, and real ZScript `Tick` overrides (all map pickups, PlayerPawn, modded actors) run as JIT'd code. The structural costs that matter for a 90–120 Hz VR target are these:
1. A full tic's work lands inside one rendered frame.
2. The code makes at least three full passes over all actors per tic (interpolation reset, thinker tick, light/effect pass).
3. A dead "has moved" check makes `UpdateRenderSectorList` run for every sector-linked actor that reaches the end of its tick, every tic.
4. Every state action call does 1–2 heap allocations.
5. Global scratch state (`validcount`, `spechit`, static intercept arrays, free lists) makes the playsim single-threaded by design.

---

## 1. Tic order of operations

Main loop: `D_DoomLoop` calls `TryRunTics()` and then `D_Display()` serially in one iteration [U] src/d_main.cpp:1463-1467. `TryRunTics` first calls `GC::CheckGC()` [U] src/d_net.cpp:2190-2192. It then runs `G_Ticker()` once per available tic, and after that runs `P_RunClientSideLogic()` once per tic [U] src/d_net.cpp:2314-2341. `D_Display` calls `GC::CheckGC()` again at its top [U] src/d_main.cpp:1109-1115.

`P_Ticker` (per level, per tic) [U] src/p_tick.cpp:168-276:

1. `Level->interpolator.UpdateInterpolations()` for every level (sector plane/scroll interpolators) [U] src/p_tick.cpp:172-175.
2. Pause check. If paused, it still iterates **all actors** to service `MF8_RECREATELIGHTS` [U] src/p_tick.cpp:186-204.
3. Freeze toggling and sound resume, then `P_ResetSightCounters` [U] src/p_tick.cpp:208-235.
4. **Full actor pass #1**: `GetThinkerIterator<AActor>()` over all thinking statnums, calling `ClearInterpolation()` and `ClearFOVInterpolation()` on each [U] src/p_tick.cpp:241-252. `ClearInterpolation` copies Pos/Angles/Scale/Alpha into the `Prev*` fields [U] src/playsim/actorinlines.h:46-62.
5. `P_ThinkParticles(Level)` [U] src/p_tick.cpp:254.
6. `P_PlayerThink` for each player [U] src/p_tick.cpp:256-258. It dispatches to the ZScript virtual `PlayerPawn.PlayerThink` through `VMCall` [U] src/playsim/p_user.cpp:1686-1690, after a string-keyed `FloatVar("prevBob")` field lookup [U] src/playsim/p_user.cpp:1684. `FloatVar` takes an `FName` [U] src/common/objects/dobject.h:229, so each call converts the string literal to a name (a name-table lookup) and then does a `FindSymbol` hash lookup up the class chain [U] src/common/objects/dobject.cpp:670-686. This is per player, so negligible in single player.
7. `localEventManager->WorldTick()` (ZScript event handlers), then `Level->Tick()` [U] src/p_tick.cpp:261-262.
8. **Full actor pass #2**: `Level->Thinkers.RunThinkers(Level)` [U] src/p_tick.cpp:263.
9. `P_UpdateSpecials` if the level is not frozen [U] src/p_tick.cpp:266-269.

Then `P_RunClientSideLogic` (per tic) [U] src/p_tick.cpp:54-99:
- **Full pass over client-side actors** for `ClearInterpolation` [U] src/p_tick.cpp:73-82, then `RunClientSideThinkers` [U] src/p_tick.cpp:83.
- Inside `RunClientSideThinkers`, a `recreateLights` lambda does **full actor pass #3**. It iterates all server-side actors and then all client-side actors, calling `RecreateDynamicLights` [U] src/playsim/dthinker.cpp:250-265. That function rebuilds lights when `MF8_RECREATELIGHTS` is set and runs `P_RunEffect` for actors with `effects`/`fountaincolor` [U] src/playsim/dthinker.cpp:211-224. The lambda is invoked once per call [U] src/playsim/dthinker.cpp:308. After that, every `FDynamicLight` in `Level->lights` gets a C++ `Tick()` when `r_dynlights` is on and the world is not paused [U] src/playsim/dthinker.cpp:309-318.

`RunThinkers` [U] src/playsim/dthinker.cpp:93-122 walks statnums `STAT_FIRST_THINKING` (32) through `MAX_STATNUM` (127) [U] src/playsim/statnums.h:42, src/playsim/dthinker.h:42. It ticks `Thinkers[i]` and then repeatedly ticks `FreshThinkers[i]` until no new thinkers appear in a pass. As a result, things spawned during this tic also tick this tic. Newly spawned thinkers with statnum ≥ 32 are linked into `FreshThinkers` [U] src/playsim/dthinker.cpp:70-84.

`FThinkerList::TickThinkers` [U] src/playsim/dthinker.cpp:812-850 is an intrusive doubly linked ring (`NextThinker`/`PrevThinker` in each `DThinker`) [U] src/playsim/dthinker.h:122-126. For each node it does three things:
- Moves the node out of the fresh list and calls `CallPostBeginPlay()` if `OF_JustSpawned` is set.
- Skips nodes with `OF_EuthanizeMe`.
- Otherwise calls `node->CallTick()`.

Linking a thinker issues 4 GC write barriers [U] src/playsim/dthinker.cpp:552-574.

**Inference:** In thinker-list order, actors interleave with sector movers and other thinkers by statnum and are scattered across the heap (each is its own `M_Malloc`, see §7). Each tic walks a pointer chain three times over ~1.6 KB objects. On big maps (5–20k actors) the three passes are cache-miss-bound before any game logic runs: pass #1 touches ~2–3 cache lines per actor plus a dependent load of `Sector->PortalGroup` (`ClearInterpolation` reads it [U] src/playsim/actorinlines.h:52-53), and pass #3 touches flags8/effects. `FThinkerIterator::Next` also performs an `IsKindOf` per thinker [U] src/playsim/dthinker.cpp:1364-1410. That is a parent-chain walk [U] src/common/objects/dobjtype.h:131-145.

## 2. Is AActor::Tick dispatched through the VM? (per-actor per-tic cost)

**Yes, always through the VM calling convention; whether it actually *executes* script code depends on the class.**

The dispatch chain:

```cpp
void DThinker::CallTick()
{
	IFVIRTUAL(DThinker, Tick)
		VMCallVoid<DThinker*>(func, this);
	else
		Tick();
}
```
[U] src/playsim/dthinker.cpp:1265-1271

`IFVIRTUAL` caches the vtable index in a function-local static. On each call it loads `GetClass()->Virtuals[VIndex]` [U] src/common/scripting/vm/vm.h:1066-1076. `Thinker.Tick` is declared `virtual native void Tick();` in ZScript [U] wadsrc/static/zscript/doombase.zs:259. As a result, **every** ZScript-visible thinker class has a non-null `Virtuals[Tick]`, and the `else Tick();` C++ fallback is effectively dead for script-registered classes. `Actor` does not redeclare `Tick` in ZScript (only `Behavior.Tick` appears in actor.zs) [U] wadsrc/static/zscript/actors/actor.zs:101-110. The slot for plain actors therefore holds the native `DThinker.Tick` function. It is registered with `DEFINE_ACTION_FUNCTION_NATIVE`, so it has both a boxed VM wrapper (`AF_DThinker_Tick`) and a direct `NativeTick`; both just do `self->Tick()` [U] src/playsim/dthinker.cpp:1253-1263. A C++ `VMCall` always uses the boxed wrapper. That call is a C++ virtual, overridden by `AActor::Tick` [U] src/playsim/actor.h:820.

Call path for a plain native actor (e.g. a Doom imp with no ZScript override):
1. `IFVIRTUAL`: class pointer load, `Virtuals.Size()` bounds check, function pointer load.
2. `VMCallVoid`: `VMValidateSignatureSingle` runs **at runtime on every call** [U] src/common/scripting/vm/vm.h:687-714, 609-615. It calls the out-of-line `VMCheckParamCount`, which does two `Proto->...SSize()` compares [U] src/common/scripting/vm/vmframe.cpp:578-584. It calls the out-of-line, empty `VMCheckReturn<void>` [U] src/common/scripting/vm/vmframe.cpp:634-636. It also calls the out-of-line `VMCheckParam<DObject*>`, which makes a virtual call to `ArgumentTypes[0]->isObjectPointer()` [U] src/common/scripting/vm/vmframe.cpp:628-632.
3. `VMCallCheckResult` (out of line) → `VMCall` [U] src/common/scripting/vm/vmframe.cpp:686-691.
4. `VMCall`: tests `VARF_Native` and calls `NativeCall(param, numparam, ret, numret)` through a function pointer [U] src/common/scripting/vm/vmframe.cpp:705-714.
5. `AF_DThinker_Tick`: `PARAM_SELF_PROLOGUE`. In release builds this is a null check (`PARAM_NULLCHECK`), with asserts compiled out [U] src/common/scripting/vm/vm.h:829, 855, 1037-1039, 486-494.
6. `self->Tick()`, a C++ virtual → `AActor::Tick`.

**Inference:** That is roughly 5–7 indirect or out-of-line calls and ~10 dependent loads (class → Virtuals array → VMFunction → Proto → ArgumentTypes → PType vtable) per actor per tic. Release builds use LTO by default (see §3), which may inline the cross-TU check helpers but cannot remove the indirect calls or the dependent loads. Warm, that is on the order of 15–40 ns. With 10k actors the dispatch alone is ~0.2–0.4 ms per tic, before any real work. It is not dominant, but it is pure overhead, and the signature validation is a pure waste after the first call.

Classes whose ZScript `Tick` is a real script override run the compiled function instead. Engine examples: `Inventory` [U] wadsrc/static/zscript/actors/inventory/inventory.zs:115-149, `PlayerPawn`, `ScriptedMarine`, `FastProjectile`, `Debris`, `DynamicLight`, camera classes, powerups and others. `grep -rlE "override\s+void\s+Tick\s*\(" wadsrc/static/zscript` finds 17 files (26 overrides): 13 actor files and 4 status-bar files. (A looser `"override void Tick"` grep gives 24 files because it also matches menu `Ticker()` overrides.) `Inventory.Tick` calls `Super.Tick()` when unowned. **Every pickup lying on the map therefore pays a script frame plus a native call each tic.** Owned inventory items run the script state-advance loop instead [U] wadsrc/static/zscript/actors/inventory/inventory.zs:117-140.

### VM entry cost (script function)
`VMCall` short-circuits empty functions (`RET` NIL) and constant-int returns (`RET` int-konst 0) without building a frame [U] src/common/scripting/vm/vmframe.cpp:717-729. The ZScript compiler emits `return <const>;` as `OP_RET RET_FINAL, REGT_INT|REGT_KONST, k` [U] src/common/scripting/backend/codegen.cpp:12329-12331, 577-579 with `REGT_KONST=4` [U] src/common/scripting/vm/vm.h:63-69. Default virtuals such as `CanCollideWith(){ return true; }` [U] wadsrc/static/zscript/actors/actor.zs:664-667 therefore hit the no-frame path when called from C++. Otherwise it calls `sfunc->ScriptCall` [U] src/common/scripting/vm/vmframe.cpp:730-737:
- `ScriptCall` initially points to `FirstScriptCall` [U] src/common/scripting/vm/vmframe.cpp:135. On the first call it JIT-compiles the function (lazily, or ahead of time if `vm_jit_aot` is set) and swaps itself out [U] src/common/scripting/vm/vmframe.cpp:313-345.
- **Interpreter** (`VMExec`/`Exec`): frames are allocated from a block-pool `VMFrameStack`, `memset` to 0, and S-registers are constructed and destroyed per frame [U] src/common/scripting/vm/vmframe.cpp:435-512, 524-573, src/common/scripting/vm/vmexec.h:2310-2327. Dispatch uses computed goto on GCC/Clang. **Every `NEXTOP` calls `DebugServer::RuntimeEvents::EmitInstructionExecutionEvent`** [U] src/common/scripting/vm/vmexec.cpp:51-60; the switch-dispatch build (MSVC) calls it once per loop iteration too [U] src/common/scripting/vm/vmexec.h:61-64. That call only tests a global flag [U] src/common/scripting/dap/RuntimeEvents.cpp:74-80, but it is defined in a different translation unit. Release and RelWithDebInfo builds turn on LTO by default when the toolchain supports it (`FORCE_NO_LTO` off) [U] CMakeLists.txt:15-27. (**Inference:** with LTO the hook can be inlined to a load and branch per op. Without LTO, e.g. `FORCE_NO_LTO` or an unsupported toolchain, expect a call/ret per bytecode op, which can roughly double interpreter cost for tight scripts.)
- **JIT**: a "simple frame" function (no strings, no special inits, no extra space) gets its VM frame on the native stack [U] src/common/scripting/jit/jit.cpp:312-328. Others push a full frame onto `GlobalVMStack` [U] src/common/scripting/jit/jit.cpp:411-448. JIT'd statically bound calls (`OP_CALL_K`) to natives registered with `DEFINE_ACTION_FUNCTION_NATIVE` go **directly through the platform ABI** (`DirectNativeCall`). Calls to other natives, to script functions, and **all virtual calls** (`OP_VTBL` + `OP_CALL`, even to a direct-native virtual) go through `ScriptCall` with a `VMValue` param array written to memory [U] src/common/scripting/jit/jit_call.cpp:65-128. The native pointer is wired at startup [U] src/common/scripting/core/imports.cpp:209. Count: 829 `DEFINE_ACTION_FUNCTION_NATIVE` vs 626 plain `DEFINE_ACTION_FUNCTION` in src (`grep -rn ... | wc -l`). About 43% of natives therefore still use the boxed `VM_ARGS` convention even from JIT code.

### Inside AActor::Tick (native) — fixed per-tic work for a non-NOINTERACTION actor
[U] src/playsim/p_mobj.cpp:4504-5157:
- If morphed, a `CheckUnmorph` VM virtual [U] src/playsim/p_mobj.cpp:4525-4540. A `freezetics` early-out [U] src/playsim/p_mobj.cpp:4542-4546.
- `TickBehaviors()` (if not frozen). It iterates the `Behaviors` TMap into local `TArray`s and calls the ZScript `Behavior.Tick` only if overridden [U] src/playsim/p_mobj.cpp:564-603. With no behaviors this costs an iteration over a 1-node map and no allocation (empty `TArray` push never happens).
- With `MF9_DECOUPLEDANIMATIONS`, `CalcBones(false)` runs before anything else [U] src/playsim/p_mobj.cpp:4548-4551.
- `MF5_NOINTERACTION` fast path. If frozen it returns. Otherwise, if velocity is non-zero or the actor is still in the blockmap, it does Unlink → move → `CheckPortalTransition` → Link [U] src/playsim/p_mobj.cpp:4568-4602. It does **not** return there: it skips only the movement block and falls through to the common tail below (KILLMONSTERS, special-sector checks, `CheckNoDelay`, `UpdateRenderSectorList`, state advance) [U] src/playsim/p_mobj.cpp:5076-5117.
- Otherwise (the movement block):
  - A walk of **every inventory item, calling its `DoEffect` virtual through `VMCall`** [U] src/playsim/p_mobj.cpp:4611-4622. The default `DoEffect(){}` hits the empty-function shortcut [U] wadsrc/static/zscript/actors/inventory/inventory.zs:998.
  - Freeze check, rocket/grenade trail spawns, VFRICTION, visibility pulse, stealth [U] src/playsim/p_mobj.cpp:4629-4744.
  - The scroller/carry loop over `touching_sectorlist` runs only for players or for actors flagged `MF8_INSCROLLSEC` [U] src/playsim/p_mobj.cpp:4755-4889.
  - The slope-slide check (`P_FindFloorPlane`) runs for solid gravity actors standing on the floor [U] src/playsim/p_mobj.cpp:4891-4928.
  - **`P_XYMovement` is always called** [U] src/playsim/p_mobj.cpp:4948. It returns early when `move.isZero()` [U] src/playsim/p_mobj.cpp:2483-2502. Otherwise it splits the move into `steps = 1 + speed/(radius-1)` sub-steps and calls `P_TryMove` for each [U] src/playsim/p_mobj.cpp:2516-2547, 2567-2578.
  - Z movement runs if `Vel.Z != 0 || BlockingMobj || Z != floorz`, with `P_CheckOnmobj` for PASSMOBJ [U] src/playsim/p_mobj.cpp:4971-5045. `P_ZMovement` always does a **VM call to the `FallAndSink` native virtual** [U] src/playsim/p_mobj.cpp:2960, 3424-3435, so floating monsters pay this every tic.
  - **Always**: `CheckPortalTransition(true)` [U] src/playsim/p_mobj.cpp:5053, 4230-4267 and `UpdateWaterLevel()` [U] src/playsim/p_mobj.cpp:5055, 5384-5403.
  - Poison damage [U] src/playsim/p_mobj.cpp:5063-5073.
- **Always** (even for still actors and for NOINTERACTION actors): the KILLMONSTERS check, `P_ActorOnSpecial3DFloor`, and `checkForSpecialSector`/`P_ActorInSpecialSector` [U] src/playsim/p_mobj.cpp:5076-5094. Then `CheckNoDelay` [U] src/playsim/p_mobj.cpp:5102. Then **`UpdateRenderSectorList()`** [U] src/playsim/p_mobj.cpp:5105.
- State advance: `--tics <= 0` → `SetState(next)` [U] src/playsim/p_mobj.cpp:5107-5117. If a model is present, `CalcBones` [U] src/playsim/p_mobj.cpp:5119-5122. The nightmare-respawn block runs for tics==-1 / CanRaise states [U] src/playsim/p_mobj.cpp:5124-5156.

**Bug-level finding: `UpdateRenderSectorList` never early-outs.** It is guarded by `if (Pos() != OldRenderPos && !(flags & MF_NOSECTOR))` [U] src/playsim/p_secnodes.cpp:375-378. But `OldRenderPos` is only ever written once, at spawn, to `{FLT_MAX,FLT_MAX,FLT_MAX}` [U] src/playsim/p_mobj.cpp:5437. `grep -rni oldrenderpos` over the whole tree finds only the two reads in `UpdateRenderSectorList` (lines 378 and 382), that spawn write, and the declaration [U] src/playsim/actor.h:1188. It is not serialized and has no `DEFINE_FIELD`, so neither savegames nor ZScript touch it. So every tic, for every sector-linked actor that reaches the end of `AActor::Tick`, it:
- frees and rebuilds `touching_lineportallist` (`ClearRenderLineList`);
- if the map has line portals and one is near, linearly scans **all** `Level->linePortals` [U] src/playsim/p_secnodes.cpp:381-400;
- frees and rebuilds `touching_sectorportallist`, walking sector portals up and down with a `PointInSector` per stack level [U] src/playsim/p_secnodes.cpp:401-426.

On portal-free maps this costs two `P_DelSeclist(nullptr)` calls and two `PortalBlocksMovement` tests. On portal-heavy maps it is a free-list churn plus a BSP walk per actor per tic. `LinkToWorld` also calls it after every relink [U] src/playsim/p_maputl.cpp:533, so a moving actor rebuilds the lists once per successful step and again in its tick.

**Inference (corrected):** `OldRenderPos = Pos();` at the end of the function is **not** enough on its own. `UnlinkFromWorld` unconditionally clears both render lists [U] src/playsim/p_maputl.cpp:317-318, so any unlink/relink that leaves `Pos()` unchanged (e.g. `SetOrigin` to the same spot, or a size change that relinks in place) would then leave the lists empty. The fix also needs `UnlinkFromWorld` to invalidate `OldRenderPos` (e.g. back to `FLT_MAX`). The current per-tic rebuild also hides moving sector-portal planes and actor height changes, which a position-only check would miss (see Open questions).

### Per-state action functions (SetState → CallAction)
`AActor::SetState` loops over zero-tic states, calling `newstate->CallAction(this, this, &stp, &returned_state)` [U] src/playsim/p_mobj.cpp:852-957. `FState::CallAction` [U] src/gamedata/info.cpp:148-234:
- `CheckCallerType` is called on every call [U] src/gamedata/info.cpp:172. When `ImplicitArgs >= 1` (always true for actor action functions) it does `auto argtypes = ActionFunc->Proto->ArgumentTypes;` [U] src/gamedata/info.cpp:133-137. `ArgumentTypes` is a `TArray<PType*>` [U] src/common/scripting/core/types.h:704. `auto` copies it: the copy constructor calls `DoCopy`, which does `M_Malloc` whenever the source is non-empty [U] src/common/utility/tarray.h:247-250, 848-858. It is non-empty here because it includes `self`. **One heap alloc and one free per action call.**
- If `DefaultArgs.Size() > 0`, `auto defs = ActionFunc->DefaultArgs;` (a `TArray<TypedVMValue>` [U] src/common/scripting/vm/vm.h:440) makes a **second heap copy** [U] src/gamedata/info.cpp:177-185. Values then go into a global `actionParams` TArray [U] src/gamedata/info.cpp:146, 186-197. `DefaultArgs` is only filled for functions that declare defaults [U] src/common/scripting/frontend/zcc_compile.cpp:2874-2877.
- Parameterless calls such as `A_Chase` map the state directly to the native function, with no script wrapper. This happens when the call has no arguments, is non-virtual and the target has no required explicit parameters [U] src/common/scripting/backend/codegen.cpp:9798-9811, src/common/scripting/backend/vmbuilder.cpp:802-814. `A_Chase` has 3 optional parameters [U] wadsrc/static/zscript/actors/actor.zs:1287, so it takes the DefaultArgs path: **2 mallocs per `A_Chase` invocation**. A parameterless `A_Look` (no defaults) costs 1. Calls with arguments compile into a script wrapper function (VM/JIT frame) that calls the native.

**Inference:** With ~1000 active monsters in chase states of 3–4 tics, that is ~500–700 malloc/free pairs per tic from this alone, plus allocator lock and cache traffic. A trivial fix is `const auto&`.

## 3. JIT: built and enabled?

- Built only when `TARGET_ARCHITECTURE` matches `x86_64` [U] CMakeLists.txt:274-276. On UNIX it is disabled if `backtrace` is unavailable [U] CMakeLists.txt:407-419. asmjit is added only when enabled [U] CMakeLists.txt:421-426, and `-DHAVE_VM_JIT` is passed to src [U] src/CMakeLists.txt:351-352.
- `vm_jit` defaults to `true` (false on DragonFly). It has `CVAR_NOINITCALL`, prints "You must restart" and is **not saved** (command line only) [U] src/common/scripting/vm/vmframe.cpp:39-48. `vm_jit_aot` defaults to `true` [U] src/common/scripting/vm/vmframe.cpp:49-53. Without `HAVE_VM_JIT`, both are forced false and unsettable [U] src/common/scripting/vm/vmframe.cpp:54-59.
- With both on, all script functions are compiled at script-compile time [U] src/common/scripting/backend/vmbuilder.cpp:954-959. Otherwise compilation happens lazily on first call [U] src/common/scripting/vm/vmframe.cpp:332-345.
- Debugging (`-debug` or `vm_debug`) forces both off [U] src/d_main.cpp:3411-3418.
- Functions are interpreted instead if they use ≥200 registers [U] src/common/scripting/vm/vmframe.cpp:297-311 or contain a function-pointer call (`blockJit`) [U] src/common/scripting/backend/codegen.cpp:9832.
- `vm_jit` also changes bytecode emission for calls (return count encoding, extra info for the JIT) [U] src/common/scripting/backend/vmbuilder.cpp:1157-1183.

**Consequence for VR:** PC VR on x64 gets the JIT. Any aarch64 build (standalone Quest, Apple Silicon, Snapdragon PCs) gets the interpreter plus the per-op debug hook.

## 4. Blockmap iteration, duplicate checks, CheckPosition, portals

### Data layout
- Blockmap cells are 128 units [U] src/gamedata/p_blockmap.h:69. Lines come from the (possibly rebuilt) blockmap lump as `int` lists terminated by -1. `GetLines` skips the leading 0 [U] src/gamedata/p_blockmap.h:87-93.
- Things are tracked per cell through `FBlockNode` doubly linked lists (`blocklinks[]`). An actor touching N cells owns N nodes linked through `NextBlock` [U] src/gamedata/p_blockmap.h:32-46. `LinkToWorld` creates one node per touched cell, for every portal group the actor overlaps [U] src/playsim/p_maputl.cpp:481-531. Nodes come from a global free list and fall back to an arena allocator [U] src/playsim/p_secnodes.cpp:447-475.

### Duplicate-avoidance scheme
- **Lines:** a global `int validcount` (defined in the renderer [U] src/rendering/r_utility.cpp:176) is incremented per query, and `line_t::validcount` / `polyobj->validcount` are stamped as they are visited [U] src/playsim/p_maputl.cpp:563-683. The scheme writes to shared map data on every query (**Inference:** it dirties line cache lines, prevents concurrent queries, and couples renderer and playsim, since both use the same counter).
- **Things:** `FBlockThingsIterator::Next` returns an actor immediately if it occupies only one block (`NextBlock==NULL && PrevBlock==&me->BlockNode`). Otherwise it dedupes with a 32-bucket hash over a fixed 10-entry array that spills into a `TArray DynHash` (which `Grow(50)`s on first spill — **a heap allocation inside the query**). The iterator `memset`s 128 bytes of buckets per init [U] src/playsim/p_maputl.cpp:886-1026, src/playsim/p_maputl.h:313-357. It calls `PrefetchL3(block->NextActor)` [U] src/playsim/p_maputl.cpp:955.

### P_CheckPosition
[U] src/playsim/p_map.cpp:1770-1987:
1. `PointInSector` (BSP walk) for the destination. Base floor/ceiling come from that sector's planes, plus a loop over its 3D floors [U] src/playsim/p_map.cpp:1782-1832.
2. `validcount++`; clear the global `spechit`/`portalhit` arrays [U] src/playsim/p_map.cpp:1845-1849, 81-82.
3. Thing pass: a `FMultiBlockThingsIterator` over the radius box → `PIT_CheckThing` per candidate [U] src/playsim/p_map.cpp:1865-1917. The cheap rejects come first: self, THRUACTORS, ThruBits, not solid/special/shootable, AABB distance [U] src/playsim/p_map.cpp:1366-1381. Only for pairs that could act does it call `P_CanCollideWith`, which makes **two VM virtual calls**, one per side [U] src/playsim/p_map.cpp:1469-1476, 117-147. The code comment explicitly avoids calling it for "all the monster corpses lying around".
4. `validcount++` again, then a line pass with `FMultiBlockLinesIterator` → `PIT_CheckLine` per line. It runs an `inRange`/`BoxOnLineSide` reject [U] src/playsim/p_map.cpp:824-829, then `P_LineOpening` (front/back plane `ZatPoint`s, 3D floors, 3D midtex) [U] src/playsim/p_map.cpp:968-970, src/playsim/p_maputl.cpp:110-230. `CanCrossLine` is a VM call, gated by `MF8_CROSSLINECHECK` [U] src/playsim/p_map.cpp:1003. Passable non-linked line portals get an extra `PIT_CheckPortal` [U] src/playsim/p_map.cpp:1950-1968.

`P_TryMove` calls `P_CheckPosition` [U] src/playsim/p_map.cpp:2338. On success it does `UnlinkFromWorld`/`LinkToWorld` [U] src/playsim/p_map.cpp:2679-2693 and processes crossed specials from `spechit` [U] src/playsim/p_map.cpp:2704-2739. Each relink:
- rebuilds `touching_sectorlist` with a fresh `FBlockLinesIterator` over the radius box (`P_CreateSecNodeList`), and rebuilds `touching_rendersectors` the same way when `renderradius >= 0` [U] src/playsim/p_maputl.cpp:441-477, src/playsim/p_secnodes.cpp:216-283;
- calls `PointInSector` **and** `PointInRenderSubsector` (two BSP walks) [U] src/playsim/p_maputl.cpp:428, 437;
- calls `CollectConnectedGroups` [U] src/playsim/p_maputl.cpp:483-485;
- calls `UpdateRenderSectorList` again [U] src/playsim/p_maputl.cpp:533.

**Inference:** One successful monster step therefore costs roughly 3 BSP point lookups, 3 blockmap line iterations (CheckPosition, sector-list rebuild, render-sector-list rebuild), 1 blockmap thing iteration and 1 portal-group collection. A moving actor typically spends 1–4 µs per step on medium maps.

### Portal-aware multi-block iteration
`FPortalGroupArray` stores 4 entries inline and spills into a `TArray` [U] src/playsim/p_maputl.h:180-239. `CollectConnectedGroups` [U] src/playsim/portal.cpp:1049-1139:
- Returns immediately if the map has one portal group (`Displacements.size == 1`) [U] src/playsim/portal.cpp:1054-1059. This is the common Doom case, and there `FMulti*Iterator` degenerates to a single `FBlock*Iterator`.
- Otherwise, if any linked line portals exist, it **linearly scans every linked portal in the map** with a bbox and `BoxOnLineSide` test, then flood-fills groups [U] src/playsim/portal.cpp:1061-1099. It also walks sector portals up and down with `PointInSector` per level [U] src/playsim/portal.cpp:1100-1124. `PGA_Full3d` additionally runs a blockmap line iteration per group [U] src/playsim/portal.cpp:1125-1139.
- `FMultiBlockLinesIterator::Next` recurses into `GoUp`/`GoDown` at the end of each group to find the real floor and ceiling through sector portals [U] src/playsim/p_maputl.cpp:768-821. Group switches call `PointInSector` [U] src/playsim/p_maputl.cpp:829-842.

**Inference:** On maps with many linked portals, every CheckPosition and every LinkToWorld is O(#linked portals). The portal machinery is a cost multiplier on movement, not a constant.

## 5. Sight checks and A_Look / A_Chase

`P_CheckSight` [U] src/playsim/p_sight.cpp:840-951:
1. Reject matrix test (`CheckReject`: a bit lookup in `sectors×sectors`, or always pass if there is no REJECT lump) [U] src/g_levellocals.h:443-451, src/playsim/p_sight.cpp:861-866.
2. Invisibility / stealth random reject; fake-floor (heightsec) blocking [U] src/playsim/p_sight.cpp:873-912.
3. `validcount++`, then a `SightCheck::P_SightPathTraverse`. It is a DDA blockmap walk (≤1000 cells) [U] src/playsim/p_sight.cpp:619-760. Per cell it tests polyobject lines and blockmap lines with `P_SightCheckLine`: a `validcount` dedupe, two side tests, an **early-out on a sight-blocking line** when no portal has been seen, otherwise a push into a static `intercepts` array [U] src/playsim/p_sight.cpp:432-466, 476-523, 89-90.
4. `P_SightTraverseIntercepts` computes fractions and then processes intercepts in order with an **O(n²) selection loop** (it rescans all intercepts for each step) [U] src/playsim/p_sight.cpp:534-583.
5. If the first trace fails, it re-traces through each collected portal [U] src/playsim/p_sight.cpp:931-945.

There is no sight result caching between tics (`grep -rniE "sightcache|sight_cache|lastsighttic|cachedsight" src` returns nothing).

`A_Look` (plain `DEFINE_ACTION_FUNCTION`, no direct-native path) [U] src/playsim/p_enemy.cpp:1912-2022 picks a sound target and otherwise calls `P_LookForPlayers` [U] src/playsim/p_enemy.cpp:1757-1901. That loops over MAXPLAYERS slots, with `isTargetablePlayer` → `P_IsVisible`. `P_IsVisible` checks distance and FOV first and calls `P_CheckSight` last [U] src/playsim/p_enemy.cpp:1245-1291. Friendly monsters and `MF8_SEEFRIENDLYMONSTERS` use `P_LookForEnemies`, a blockmap thing search [U] src/playsim/p_enemy.cpp:1775-1821, 1441.

`A_DoChase` [U] src/playsim/p_enemy.cpp:2365-2717 (reached via `A_Chase`, which is `DEFINE_ACTION_FUNCTION_NATIVE` [U] src/playsim/p_enemy.cpp:2953) may run `P_LookForPlayers` (when the target is lost or for friends) [U] src/playsim/p_enemy.cpp:2485, 2659. It may run `P_CheckMeleeRange`, which includes `P_CheckSight` [U] src/playsim/p_enemy.cpp:251-289, 2616. It may run `P_CheckMissileRange`, also with `P_CheckSight` [U] src/playsim/p_enemy.cpp:305-312, 2634. In multiplayer or with `TIDtoHate` it runs another `P_CheckSight` when threshold is 0 [U] src/playsim/p_enemy.cpp:2645-2659. Then `P_SmartMove`/`P_NewChaseDir` → `P_TryMove` [U] src/playsim/p_enemy.cpp:2687-2689.

**Inference:** A_Look runs once per look-state duration (Doom idle states are typically 10 tics), so idle monster cost is about 1 sight trace per 10 tics per monster, and the reject matrix kills most of them on vanilla maps. Chasing monsters call A_Chase every 2–4 tics: up to 2 sight traces plus 1–3 `P_TryMove` (each with the full relink above), plus the 2 mallocs from §2. On slaughter maps (2–5k awake monsters) this is the dominant playsim cost, typically 3–15 ms per tic. In VR that lands entirely inside one 8–11 ms frame.

## 6. Dormancy / sleep / tick culling

**There is none.** Proof:
- `RunThinkers` ticks every node in statnums 32–127 unconditionally except `OF_EuthanizeMe` [U] src/playsim/dthinker.cpp:105-121, 841-846.
- `AActor::Tick` early-outs are limited to the following:
  - unmorphed originals [U] src/playsim/p_mobj.cpp:4525-4540, 4624-4627;
  - `freezetics` [U] src/playsim/p_mobj.cpp:4542-4546;
  - freeze or time-freeze via `isFrozen()` (the level freeze state, with `MF5_NOTIMEFREEZE` exempting the actor) [U] src/playsim/actorinlines.h:193-214, src/playsim/p_mobj.cpp:4576-4590, 4629-4643;
  - zero-move inside `P_XYMovement` [U] src/playsim/p_mobj.cpp:2483-2502.

  Even a static decoration with `tics == -1` still runs TickBehaviors, the inventory loop, P_XYMovement, CheckPortalTransition, UpdateWaterLevel, the special-sector checks and UpdateRenderSectorList, as listed in §2. `MF5_NOINTERACTION` skips only the movement block, not the tail.
- `MF2_DORMANT` only affects respawn and skullfly handling inside Tick [U] src/playsim/p_mobj.cpp:5131, 2490.
- `grep -rniE "sleep|tickdist|ticking ?dist|notick|lod_tick|cull.*tick|tick.*cull" src/playsim src/p_tick.cpp` returns no relevant hit.
- The only way to stop ticking is to move a thinker into a statnum below `STAT_FIRST_THINKING` (e.g. `STAT_INFO`, `STAT_DECAL`) [U] src/playsim/statnums.h:34-42. `ChangeStatNum` from ZScript only forbids `STAT_STATIC` (for actors) and `STAT_TRAVELLING` [U] src/playsim/dthinker.cpp:1054-1076, so modders *can* park actors. **Inference:** a parked actor stays in sector and blockmap links, so it still renders and collides; it just never runs `Tick` or advances states.

## 7. Per-tic allocations and the GC

### Allocations per tic (found in code)
| Site | When | Evidence |
|---|---|---|
| `TArray` copy of `Proto->ArgumentTypes` | every state action call | [U] src/gamedata/info.cpp:133-137; src/common/utility/tarray.h:247-250, 848-858 |
| `TArray` copy of `DefaultArgs` | action funcs with default args (A_Chase, A_LookEx…) | [U] src/gamedata/info.cpp:177-185 |
| Object allocation on spawn: `M_Malloc(Size)` + `memcpy(Defaults)` | every Spawn (puffs, blood, smoke trails, projectiles) | [U] src/common/objects/dobjtype.cpp:448-474 |
| `TMap` default ctor allocates a 1-node vector | every actor construction (`Behaviors`) | [U] src/common/utility/tarray.h:1186; src/playsim/actor.h:1417 |
| `DynHash.Grow(50)` | thing iteration once >10 multi-block actors are seen | [U] src/playsim/p_maputl.cpp:1004-1011 |
| `FPortalGroupArray` spill `TArray` | >4 portal groups in a query | [U] src/playsim/p_maputl.h:216-220 |
| VM frame blocks | only when `GlobalVMStack` needs a new block (pooled thereafter) | [U] src/common/scripting/vm/vmframe.cpp:477-505 |

Non-allocating pools: sector nodes and block nodes come from free lists backed by `secnodearena` [U] src/playsim/p_secnodes.cpp:42-70, 447-475. Particles use a fixed array with an index free list [U] src/playsim/p_effect.cpp:131-158. `spechit`, `portalhit`, `actionParams` and the sight/path intercept arrays are global or static `TArray`s that keep their capacity [U] src/playsim/p_map.cpp:81-82, src/gamedata/info.cpp:146, src/playsim/p_sight.cpp:89-90, src/playsim/p_maputl.cpp:1139.

### GC (DObject incremental tri-colour mark & sweep)
- `CheckGC()` is **called** twice per main-loop iteration (once per rendered frame): first statement of `TryRunTics` and near the top of `D_Display`, before its `nodrawers` early return (see §1). `grep -rn CheckGC src` finds no other callers. Each call pushes the bytes allocated since the last call into a 512-entry history, and **runs `Step()` only** if a collection is in progress (`State > GCS_Pause`) or `AllocBytes >= Threshold` [U] src/common/objects/dobjgc.cpp:155-163, 80-93, 704-727. So "GC runs twice per frame" is true of the check, not of collection work: between cycles, both calls are a few loads and stores.
- `Step()` runs `SingleStep()` until it has covered `max(16*sizeof(DObject), avgAlloc * StepMul/100)` bytes, where `avgAlloc` is the mean bytes allocated **per `CheckGC` call** over the last 512 calls [U] src/common/objects/dobjgc.cpp:331-335, 482-522. The states are: MarkRoot (all registered marker functions plus soft roots) → Propagate (one gray object per single step) → Atomic → Sweep (40 objects per step) → Destroy → Done/SetThreshold [U] src/common/objects/dobjgc.cpp:352-377, 420-470. After a cycle, `Threshold = min(Estimate, AllocBytes)/100 * Pause` [U] src/common/objects/dobjgc.cpp:173-176.
- Defaults: `Pause` is 150 (the next cycle starts when memory reaches 150% of the post-collection estimate). `StepMul` is 600 in release builds (800 in debug). The minimum step is 16 objects' worth. `GCDELETECOST` is 75 in release (230 in debug) and `GCDESTROYCOST` is 15 [U] src/common/objects/dobjgc.cpp:44-76. They can be changed at runtime with the `gc pause N` / `gc stepmul N` console command [U] src/common/objects/dobjgc.cpp:811-851.
- Mark cost per object: `DObject::PropagateMark` walks the class's flattened pointer offsets, pointer arrays and pointer maps [U] src/common/objects/dobject.cpp:357-397. For actors that is every `TObjPtr` (target, tracer, master, lastenemy, goal, Inventory, Poisoner, …).
- Barriers: `TObjPtr::operator=` is a plain store [U] src/common/objects/dobjgc.h:251-255. **Every TObjPtr read** (`->`, implicit conversion, `Get()`) runs a read barrier that loads `obj->ObjectFlags` and nulls the pointer if the object is `OF_EuthanizeMe` [U] src/common/objects/dobjgc.h:137-144, 279-319. Explicit `GC::WriteBarrier` calls exist on thinker list links [U] src/playsim/dthinker.cpp:563-574. `Barrier()` grays the pointed object only during Propagate [U] src/common/objects/dobjgc.cpp:573-593.
- Thinker lists are roots through their sentinels [U] src/playsim/dthinker.cpp:609-617, so every live actor is reached each cycle by walking the lists.

**Inference (corrected):** While a cycle is active, each step covers 6× the *average bytes allocated per `CheckGC` call*. Calling `CheckGC` more often therefore shrinks each step proportionally. Total GC work per second follows the allocation rate (puffs, blood and projectiles of ~1.6 KB+ each), not the frame rate or the "twice per frame" call count. The exceptions are the fixed per-call overhead and the `GCMINSTEPSIZE` floor, which takes over when per-call allocation is small (e.g. at high VR frame rates). In combat-heavy scenes a step can cover tens to hundreds of KB: sweeping and freeing a few dozen dead actors and marking a few hundred live ones, roughly 0.05–0.5 ms per active step. Mark phase cost grows with total live actors and their `TObjPtr` fan-out. The read barrier means every `actor->target->...` dereference touches the target's header cache line even when the code only needs a field further in. The GC is not a stop-the-world risk under defaults, but `FullGC` [U] src/common/objects/dobjgc.cpp:532-562 runs when all thinkers are destroyed on level change [U] src/playsim/dthinker.cpp:428 and during shutdown/restart cleanup [U] src/d_main.cpp:4269, 4347, 4365.

## 8. AActor size and layout

- `class AActor final : public DThinker` [U] src/playsim/actor.h:805. The base `DObject` holds a vptr, `Class`, `ObjNext`, `GCNext` and `ObjectFlags` [U] src/common/objects/dobject.h:189-194. `DThinker` adds `_statNum`, `NextThinker`/`PrevThinker` and `Level` [U] src/playsim/dthinker.h:122-126.
- The data members of AActor span src/playsim/actor.h:1141-1417, plus `hasmodel` at :1715. A scripted tally of that range (Python, LP64 sizes: DVector3=24, DAngle=8, TObjPtr/pointers=8, TFlags=4, FName/FSoundID/FTextureID=4, TArray=16, TMap=24) counts **~224 data members and ~1,600 bytes** (**Inference**; `sizeof` was not compiled). The script-side `PClass::Size` (what `CreateNew` allocates) is at least that [U] src/common/objects/dobjtype.cpp:450.
- Hot fields are spread across the object:
  - `__Pos` sits at the top [U] src/playsim/actor.h:1142;
  - `flags`…`flags9` are in the middle [U] src/playsim/actor.h:1171-1179;
  - `Vel` is at :1189 and `Sector`/`floorz` at :1198-1201;
  - `tics`/`state` are at :1222-1223 and `target` at :1252;
  - `Inventory` is at :1337;
  - `Prev`/`PrevAngles` (touched by the interpolation pass) are at :1395-1401, near the end;
  - `Behaviors` is last, at :1417.

  **Inference:** A typical non-moving actor tick touches ~10–14 distinct 64-byte lines. The interpolation pass touches the beginning (Pos, Angles, Scale, Alpha) and the end (Prev*) of each object. Combined with pointer-chased thinker lists, the playsim is memory-latency-bound.

## 9. Interpolation and render-side coupling

- Actor interpolation: the renderer lerps `Prev`→`Pos` with the frame fraction. The tic resets `Prev` through the full-actor pass in §1 [U] src/p_tick.cpp:241-252. `nomonsterinterpolation` (default false) clears monster interpolation after moves [U] src/playsim/p_enemy.cpp:74, 599-604.
- Sector plane interpolation: `D_Render` calls `interpolator.DoInterpolations(I_GetTimeFrac())` before rendering and `RestoreInterpolations()` after [U] src/d_main.cpp:600-620. `DSectorPlaneInterpolation::Interpolate` **writes the interpolated height into the live sector plane**, and also calls `P_RecalculateAttached3DFloors` and `CheckPortalPlane` [U] src/r_data/r_interpolate.cpp:470-500. The restore pass writes the original values back [U] src/r_data/r_interpolate.cpp:261-271. In other words, rendering mutates playsim state per frame. **Inference:** this forbids running the playsim concurrently with the renderer without a snapshot.
- Also per frame: `P_FindParticleSubsectors` clears `subsectors[i].sprites` for **every subsector**, re-buckets all VisualThinkers and particles, and calls `PointInRenderSubsector` for particles without a cached subsector [U] src/playsim/p_effect.cpp:204-239, called from `D_Render` [U] src/d_main.cpp:611.

## 10. Particles

- A fixed pool of `r_maxparticles` entries (default 4000, clamped 100–65535) [U] src/g_cvars.cpp:113-120, src/playsim/p_effect.cpp:165-181. Each `particle_t` is 136 bytes, AoS, linked through `uint16` indices [U] src/playsim/p_effect.h:78-99. When the pool is full, new particles are dropped unless the spawn asks to replace the oldest [U] src/playsim/p_effect.cpp:131-158.
- `P_ThinkParticles` runs per tic per active particle [U] src/playsim/p_effect.cpp:278-351:
  - alpha/size/fade update;
  - free on expiry;
  - `GetPortalOffsetPosition` (line-portal crossing);
  - Euler integration;
  - **`PointInRenderSubsector` (a BSP descent) for every particle every tic**;
  - a sector-portal plane check.

  There is no collision with geometry.
- `DVisualThinker` (ZScript-spawnable sprite particles) are full `DThinker`s at `STAT_VISUALTHINKER` [U] src/playsim/statnums.h:63. Their native `Tick` does the same portal offset and `PointInRenderSubsector` work [U] src/playsim/p_effect.cpp:1174-1214, and each one goes through the `CallTick` VM dispatch from §2.
- Actor-based effects: `FX_ROCKET`/`FX_GRENADE` spawn `RocketSmokeTrail`/`GrenadeSmokeTrail` actors every 4 or 8 tics (full actors, each a malloc) [U] src/playsim/p_mobj.cpp:4645-4677. Whether those are visible is controlled by `cl_rockettrails` (default 0) [U] src/playsim/p_effect.cpp:51.

**Inference:** 4000 particles × (one BSP descent of ~10–20 nodes plus integration) ≈ 0.2–0.6 ms per tic, plus a per-frame O(subsectors + particles) re-bucket. That is cheap relative to actors, but the per-frame part scales with VR frame rate.

## Absent / not done

- **Tick culling / sleeping / LOD ticking:** absent; see §6 for the grep and the code paths.
- **Multithreaded playsim:** `grep -rnE "std::thread|TaskGroup|parallel_for|#pragma omp|std::async" src/playsim src/p_tick.cpp src/common/scripting` finds only the DAP debug server's restart thread [U] src/common/scripting/dap/DebugServer.cpp:91. The design actively depends on single-threading: global `validcount` [U] src/rendering/r_utility.cpp:176, global `spechit`/`portalhit` [U] src/playsim/p_map.cpp:81-82, static sight intercepts [U] src/playsim/p_sight.cpp:89-90, static `FPathTraverse::intercepts` [U] src/playsim/p_maputl.cpp:1139, global free lists [U] src/playsim/p_secnodes.cpp:30, 447, `NextToThink` [U] src/playsim/dthinker.cpp:64, `actionParams` [U] src/gamedata/info.cpp:146, the per-level `processMask`/`foundPortals` scratch [U] src/g_levellocals.h:282-284, and render-time sector mutation [U] src/r_data/r_interpolate.cpp:496-499.
- **Sight caching:** absent (§5 grep).
- **JIT on non-x86_64:** absent [U] CMakeLists.txt:274-276.
- **Data-oriented actor storage (SoA, pools, arenas for DObjects):** `CreateNew` uses `M_Malloc` per object [U] src/common/objects/dobjtype.cpp:450. `grep -n "alignas" src/playsim/actor.h` returns nothing.
- **Inline caching of VM signature checks:** absent. `VMValidateSignatureSingle` runs on every templated call [U] src/common/scripting/vm/vm.h:690.
- **Moved-check for render sector lists:** effectively absent because `OldRenderPos` is never updated (§2).

## Perf-relevant settings

| Setting | Default | Effect | Citation |
|---|---|---|---|
| `vm_jit` | true on x86_64 builds (false on DragonFly; forced false without JIT) | JIT vs interpreter; not saved, command line only | [U] src/common/scripting/vm/vmframe.cpp:39-59 |
| `vm_jit_aot` | true | compile all functions at load instead of on first call | [U] src/common/scripting/vm/vmframe.cpp:49-53; src/common/scripting/backend/vmbuilder.cpp:954-959 |
| `-debug` / `vm_debug` | off | forces both JIT cvars off | [U] src/d_main.cpp:3411-3418 |
| `gc pause N` (ccmd) | 150 | GC cycle trigger threshold (% of live estimate) | [U] src/common/objects/dobjgc.cpp:44, 811-851 |
| `gc stepmul N` (ccmd) | 600 (release) | GC work per step relative to allocation rate | [U] src/common/objects/dobjgc.cpp:54-59 |
| `r_maxparticles` | 4000 (100–65535) | particle pool size; `-numparticles` overrides | [U] src/g_cvars.cpp:113-120; src/playsim/p_effect.cpp:58-60, 165-177 |
| `r_particles` | true | per-frame particle bucketing skipped when false | [U] src/playsim/p_effect.cpp:55, 227-230 |
| `cl_rockettrails` | 0 | visibility of actor smoke trails (actors are spawned regardless) | [U] src/playsim/p_effect.cpp:51; src/playsim/p_mobj.cpp:4651-4656 |
| `cl_pufftype`, `cl_bloodtype` | 0, 0 | sprite actors vs particles for puffs/blood | [U] src/playsim/p_mobj.cpp:135-136 |
| `cl_bloodsplats` | true | blood decals on walls | [U] src/playsim/p_map.cpp:65 |
| `cl_maxdecals` | 1024 | decal thinker cap | [U] src/g_cvars.cpp:90 |
| `sv_corpsequeuesize` | 64 | corpse cap (Hexen-style queue) | [U] src/g_cvars.cpp:73 |
| `r_dynlights` | true | per-tic light Tick loop and recreate-lights work | [U] src/g_cvars.cpp:64; src/playsim/dthinker.cpp:236-243, 309-318 |
| `nomonsterinterpolation` | false | clears monster interpolation after moves | [U] src/playsim/p_enemy.cpp:74, 601-604 |
| `cl_capfps` / `vid_maxfps` | false / 500 | frame pacing (renderer side; determines frames per tic) | [U] src/common/rendering/v_framebuffer.cpp:51; src/common/rendering/v_video.cpp:71 |
| `profilethinkers N` / `profilecsthinkers` (ccmd) | — | per-class tick timing for one tic | [U] src/playsim/dthinker.cpp:123-198, 1143, 1193 |
| `stat think`, `stat sight`, `stat gc` | — | timing HUD | [U] src/playsim/dthinker.cpp:1418; src/playsim/p_sight.cpp:953; src/common/objects/dobjgc.cpp:737 |

## Transfer notes for UZDoom VR

All items are UZDoom-internal. Grades describe the change needed to the 5.0 code for a stereo 90–120 Hz Vulkan target.

1. **Sim/render frame coupling (§1).** Both run serially in one loop iteration [U] src/d_main.cpp:1463-1467. A 5–15 ms tic on a slaughter map makes one frame miss in every ~3 frames at 90 Hz.
   - **Rework (cheap form):** keep the single thread but cap the tic budget and spread work across frames. Not possible without changing tic semantics.
   - **Rebuild (real fix):** run the playsim on its own thread and have it publish an immutable render snapshot (positions, Prev*, sprites, sector plane heights) per tic. This requires removing render-time sector mutation [U] src/r_data/r_interpolate.cpp:470-500 (interpolate in the renderer's snapshot instead) and moving `validcount` out of the shared renderer global [U] src/rendering/r_utility.cpp:176.
   - **Stereo impact:** none in the sim; the snapshot is consumed once and both eyes share it.

2. **CallTick VM dispatch (§2): Port.**
   - Add a `VARF_Native` fast path in `DThinker::CallTick`: if `func` is the stock `DThinker.Tick` native, call `Tick()` directly.
   - Hoist `VMValidateSignatureSingle` to registration or debug builds.

   Estimated saving: ~0.2–0.4 ms per tic at 10k actors (Inference). Real ZScript overrides keep VM cost; the Inventory `Tick` override [U] wadsrc/static/zscript/actors/inventory/inventory.zs:115-149 could be made native.

3. **Three full actor passes per tic (§1): Rework.**
   - Fold `ClearInterpolation` into `TickThinkers`, i.e. clear right before `CallTick`. Caveat: actors moved by others before their own tick would lose interpolation, which is why the comment wants indirect movement interpolated [U] src/p_tick.cpp:241. That needs a "dirty Prev" scheme instead.
   - Fold recreate-lights/effects into a list of only the flagged actors.

   Either way, iterate a dense actor array rather than `FThinkerIterator<AActor>`, which does an `IsKindOf` per node.

4. **`UpdateRenderSectorList` dead check (§2): Port.** Set `OldRenderPos = Pos()` after the rebuild [U] src/playsim/p_secnodes.cpp:375-427, **and** reset `OldRenderPos` to `FLT_MAX` in `UnlinkFromWorld`, which clears the lists unconditionally [U] src/playsim/p_maputl.cpp:317-318. Without the second part, in-place relinks drop the portal render lists. Verify that portal lists are still rebuilt when *portals* move (polyportal, moving sector-portal planes) and when actor height changes, not only when actors move.

5. **Action-call allocations (§2): Port.** Change `auto` to `const auto&` at [U] src/gamedata/info.cpp:135 and :179. This removes 1–2 mallocs per action call. Trivial.

6. **No tick culling (§6): Rework.**
   - Add an opt-in or heuristic "dormant" fast path in `AActor::Tick` for actors with `tics==-1`, zero velocity, on the floor, no inventory, no behaviors and no special sector. Skip XY/Z, portal, water and special-sector work.
   - Distance- or PVS-based reduced-rate ticking for idle `A_Look` monsters is possible, but it changes demo and netgame determinism, so gate it behind a compat flag.

   VR benefit: large maps with thousands of decorations become close to free.

7. **Sight (§5): Port** (local algorithmic fixes) or **Rework** (cache).
   - Replace the O(n²) intercept selection with a sort [U] src/playsim/p_sight.cpp:564-583.
   - Optionally cache (t1, t2, tic) → result for multi-check monsters.
   - The REJECT test stays first.

   Determinism must hold, so caches must be within-tic only.

8. **CheckPosition / relink (§4): Rework.**
   - Merge the three blockmap line iterations of a successful move: CheckPosition already enumerates the touched lines, so it could reuse that list for `P_CreateSecNodeList`.
   - Replace `validcount` stamping with a per-query bitset or generation counter in a thread-local scratch, which is a prerequisite for any parallel sight or movement queries.
   - Precompute per-blockmap-cell lists of linked portals so `CollectConnectedGroups` stops scanning every linked portal [U] src/playsim/portal.cpp:1070-1081.

9. **Actor layout (§8): Rebuild.** Getting cache-dense iteration (hot transform/velocity/flags/state in arrays, cold data out of line) means changing `AActor` storage, which ZScript field offsets (`DEFINE_FIELD`) and serialization bind to. This is a multi-month change touching the script ABI. The cheaper alternative (**Rework**) is reordering fields so the tick-hot ones share 2–3 cache lines. Field order is free in C++, but the native-field exports must be checked.

10. **GC (§7): Port.**
    - Tune `stepmul`/`pause`.
    - Optionally move `CheckGC` to once per tic instead of twice per frame. At 120 Hz there are ~7 calls per tic, but because the step budget scales with bytes allocated per call (§7), this saves mainly the per-call overhead and the `GCMINSTEPSIZE` floor work, not most of the GC work. (Corrected: an earlier draft implied that total GC work scales with call count.)
    - Add a slab allocator for common actor sizes to avoid `M_Malloc` per spawn [U] src/common/objects/dobjtype.cpp:450.

    The read barrier on `TObjPtr` stays; it is cheap if the header is hot.

11. **Particles (§10): Port.**
    - Cache the subsector across tics and redo the BSP descent only when the particle crosses its subsector bbox (bounds test).
    - Move per-frame `P_FindParticleSubsectors` to per-tic. Positions only change per tic; the renderer interpolates.

    For VR, GPU-driven particles are a renderer-side Rebuild, out of scope here.

12. **JIT (§3): Port** for x64 PC VR. It is already default-on. Make `vm_jit` archivable or at least documented. For aarch64 standalone VR the options are a Rebuild (asmjit aarch64 backend or a different JIT) or making `EmitInstructionExecutionEvent` an inline flag check in a header [U] src/common/scripting/vm/vmexec.cpp:57, src/common/scripting/dap/RuntimeEvents.cpp:74-80 (**Port**). The inline check only matters for builds without LTO, since release LTO is on by default [U] CMakeLists.txt:15-27.

## Open questions

- Exact `sizeof(AActor)` and `PClass::Size` for `Actor` were estimated, not compiled. A `static_assert`/`printf` build would settle them.
- LTO (resolved by the verifier): the top-level CMake turns on `CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE`/`_RELWITHDEBINFO` when `check_ipo_supported` passes, unless `FORCE_NO_LTO` is set [U] CMakeLists.txt:15-27. Whether the compiler actually inlines `EmitInstructionExecutionEvent` into the interpreter loop under LTO was not checked in a binary.
- Real-world per-tic cost split (dispatch vs movement vs sight vs GC) was not measured. `profilethinkers 8` and `stat think`/`stat sight`/`stat gc` on a slaughter map (e.g. Sunder MAP15, Nuts) would quantify the inferences above.
- Whether fixing the `OldRenderPos` dead check is safe for actors moved by polyobject portals or by sector portal plane movement was not traced. (Verifier: the `UnlinkFromWorld` interaction is now covered in §2 and transfer note 4.)
- Determinism impact of folding `ClearInterpolation` into per-thinker ticking (interpolation only, not sim state) was not verified against prediction (`P_PredictClient`) code paths.
