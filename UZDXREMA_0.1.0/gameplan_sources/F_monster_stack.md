# F — The monster stack of the owner's rotation (BD22, XMVE, LegenDoom, BD22 VR Reloaded, NiceWallBlood, Universal Entropy, SpawnAdjuster)

Scope: per-monster and per-event actors, handlers and caps of the mods named in the brief. Not covered (another agent):
mk-champions, Legion of Bones' own monster code, crits, IA_Sound, RSQTHP_Bar. Sources: [F] `/home/claude/uzdxrema` @ e84623f,
[FR] `/home/claude/uzdxrema_renderer` @ abb19af, [U] stock 5.0, [MOD] `/home/claude/mods_x/<folder>/<path>` (the unpacked
rotation, third-party code), [BDW] `/home/claude/presidentkoopa/rsq_bd_vr_reloaded` @ 18c7676 (the owner's editable source
of BD22 VR Reloaded, 2.0.4u; the rotation carries 2.0.4t). [M] = `10_MEASUREMENTS.md`. Everything not cited to a line is
labelled **Inference**.

Owner direction (relayed 2026-11-13): engine-side mechanisms that make this stack cheap for *any* mod are the plan of
record; mod patches and cvar caps are listed as stopgaps.

---

## Delta table

| ID | Item | Fork status | Where in fork | What is left | Done when (measurable) | Don't touch | Netplay/determinism impact | Est. saving on modlist (Inference, with derivation) |
|---|---|---|---|---|---|---|---|---|
| P0.5 / F-MON-0 | Make the "unattributed" thinker time visible: spawn events and PostBeginPlay are outside `profilethinkers`' class clocks | Not started | [F] src/playsim/dthinker.cpp:1017-1046 (CallPostBeginPlay before `prof.timer.Clock()`); [F] src/playsim/p_mobj.cpp:6103-6107; [FR] src/p_tick.cpp:646-660 (`ticprofile`) | Add a `spawn events` bucket (time + count of `CallPostBeginPlay`, split native vs `WorldThingSpawned`) to `ticprofile`; per-handler time for spawn/tick events | `ticprofile` prints spawns/tic and their ms; `profilethinkers 8 0` sum + spawn bucket + loop overhead = `thinkers` within 5% | `perf_bench_hash`; read-only counters only | None (clocks only) | 0 ms by itself; it decides where the ~78 ms goes |
| F-MON-1 | Spawn-event filters: engine skips `WorldThingSpawned` VM calls a handler declared it does not want | Not started | [F] src/events.cpp:690-700, 1811-1822 | Native per-handler filter (class roots, missile-only, owner-is-player, skip `+NOINTERACTION`/client-side), set in `OnRegister`; build `FWorldEvent` once per actor, not per handler | With all owner handlers ported, `spawn events` ms/tic falls ≥70% at the same spawns/tic | Unported handlers keep every event (compat) | Deterministic: filter is static per handler, identical on every peer | 43 spawn overrides in the load order; ~0.15-0.3 µs per empty call → ~8-13 µs/spawn today; with ~35 filtered out ≈ -6-10 µs/spawn → at 300 spawns/tic (fight) **-2-3 ms/tic** |
| F-MON-2 | Client-side visual effects for any mod: promote listed classes to `OF_ClientSide`, with RNG and event guards | Partial (5.0 has `+CLIENTSIDE`; no guards) | [F] src/scripting/thingdef_data.cpp:414 (`CLIENTSIDE`), 466 (`CLIENTSIDEONLY` is a dummy); [F] src/playsim/p_mobj.cpp:5914-5922; [F] src/common/scripting/backend/codegen.cpp:8463-8476 (`random()` = sim RNG `pr_exrandom`); [F] src/events.cpp:690-700 (no client-side check) | (a) a lump/GameInfo list to promote classes without editing BD22; (b) in client-side thinker ticks, sim RNG draws redirect to the client RNG; (c) play-scope `WorldThingSpawned`/`Destroyed` not sent for client-side actors | BD22 gore/flare classes run in `csprofilethinkers`, none in `profilethinkers`; netplay suite IN SYNC with promotion on | The avatar/model draw path; anything that damages or is read by the sim | Required guards (b)(c), else desync: client-side ticks also run during prediction ([F] dthinker.cpp:316-410) | Removes gore/flare actors from the sim tic, the interp reset and the netplay hash. Fight model below: ~5-8 ms/tic of gore moves off the sim (it still costs locally, once, cheaper without handlers) |
| F-MON-3 | Engine debris budget: per-group LRU caps + resting debris parked (generalised corpse queue) | Partial: corpse queue + `[SLEEP]` API exist | [F] src/g_cvars.cpp:75-90 (`sv_corpsequeuesize`, fork default **10**, stock 64 [U] src/g_cvars.cpp:73); [F] src/playsim/dthinker.cpp:95-148, 1188-1260 (`Sleep`/`SleepIndefinite`/`Wake`) | Actor property `DebrisGroup "<name>", <cap>`; engine destroys oldest over cap; actors with a `-1` state, zero velocity, on floor, no inventory → `SleepIndefinite`, woken by `SetState`/damage/thrust (native entry points) | Fight replay: live gore actors bounded by the caps; tic flat vs fight length | Corpse queue semantics for `A_QueueCorpse` users | Deterministic (sim state, serialized; sleep fields already saved [F] dthinker.cpp:1082) | Resting gore today ≈ ≤10 actors (queue 10); value is in letting the owner raise caps for looks at ~0.6 µs/actor (0.4 tick + 0.17 reset = 9.02 ms/53,116) |
| F-MON-4 | Native sound-pitch and tic-scale per actor (deletes `Shifter`) | Not started (P10.7) | Shifter: [MOD] 2.4-Enemies_UniversalEntropy/randomise.zs:146-167 | `Actor.SoundPitchScale` used by `StartSound`; `Actor.TicScale` applied in `SetState` (rounded like `A_SetTics(round(tics*size))`); patch UE to set them once | `Shifter` absent from `profilethinkers`; thinkers -7,686 on Sunder MAP15 | Stock `A_SoundPitch` | Deterministic sim fields, serialized | Shifter 2.23 ms [M] §3 + its interp-reset share 7,686 × 0.17 µs = 1.3 ms → **-3.5 ms/tic** |
| F-MON-5 | Per-source sound channel index (makes Shifter-style per-tic sound calls O(own)) | **Done on [FR] only; missing on [F]** | [FR] src/common/audio/sound/s_sound.cpp:1089-1102 vs [F] src/common/audio/sound/s_sound.cpp:1073-1085 | Bring the render lane's sound commits to `main` through the netplay suite | Shifter ≤ 0.35 µs/call on a `main` build (it is 0.29 µs on the renderer build [M] §3) | — | None (same results, indexed walk) | On `main` today Shifter costs like Abandon's 174 ms / 4,752 (SIM_AT_MOD_SCALE §1) ≈ 37 µs each → 7,686 × 37 µs ≈ **280 ms/tic avoided** on main |
| F-MON-6 | User translations: O(N) dedupe, never freed, 65,535 hard cap; per-translation GPU texture copies with linear lookup | Not started | [F] src/common/engine/palettecontainer.cpp:190-206, 273-290; [F] src/r_data/r_translate.cpp:817-829; [F] src/common/textures/hw_texcontainer.h:80-120; [F] src/p_setup.cpp:691 (only `TRANSLATION_LevelScripted` cleared) | Hash dedupe; per-level user-translation slot reset (with savegame remap); `hwTex_Translated` hash; renderer: translation as a palette-row lookup in the sprite shader (render lane P7) | 10 loads of Sunder MAP15 in one session do not `I_Error`; no sprite texture creation after the first second of a map | Render lane owns the shader | Translations are sim-visible ints; reset per level must keep IDs stable inside a level | Load: O(N²) compares at ~7,000 monsters+decor; render: removes thousands of per-monster sprite copies (VR hitches). Not a per-tic sim number |
| F-MON-7 | Native attachments for decoration flames/flares (P10.2) | Not started | — | `AttachTo` + parked statnum; port BD22 torch flames/cores/balls | Torch helpers not in `profilethinkers` | — | Deterministic | 3 helpers per torch × (0.4 + 0.17) µs; minor per tic, large in thinker count |
| F-MON-8 | Decoration spawn streams (TechLamp/Column/BurningBarrel flares) | Not started | [MOD] 1.00_brutal22test6/actors/DECORATIONS/Lamps.dec:45-58, 143-156, 246-252; Torches.dec:555-604 | Engine: F-MON-2 (promote flares) or F-MON-7; stopgap: sight/distance gate in a VR patch | Spawns/tic at map start (standing still) < 20 | — | Promotion needs F-MON-2 guards | 1 spawn/tic per tech lamp; per spawn ≈ 3 µs native + 8-13 µs handlers → **~1.2-1.6 ms/tic per 100 lamps** |
| P9.2 | Dormancy (exact) | Partial: `[SLEEP]` API, no automatic dormancy | [F] src/playsim/dthinker.cpp:95-148 | Engine auto-park of resting `-1`-state debris (F-MON-3) is the first, simplest predicate | — | — | — | see F-MON-3 |
| F-MON-9 | LegenDoom's start-of-map roll: 64 choosers × `A_RadiusGive` over the whole map | Not started | [MOD] 0.99-LegenDoom_3.0/source/LEGENDWRD.acs:25-75; decorate/MonsterGen.dec:9-115 | One pass over monsters (in the owner's RSLD mode), probability `1-(1-p)^k` | Map-start hitch gone (`perf_track` frame at tic 2-5) | LegenDoom pk3 (patch in [BDW]) | Deterministic; RNG sequence changes (fine, new mod version) | One-time: ~8 rolls × ~5,700 monsters ≈ 45k CustomInventory gives (each a spawn + 43 handlers) ≈ **0.5-1 s single frame** at tic 2 |
| F-MON-10 | UE handler reads `players[consoleplayer]` in play scope | Not started | [MOD] 2.4-Enemies_UniversalEntropy/randomise.zs:5-11, 19 | Patch: drop the player-relative test, use `bFriendly` | — | — | **Desync hazard** (low in pure co-op) | 0 |
| F-MON-11 | SpawnAdjuster: density knob; decimate uses `A_Die` | Not started | [MOD] 2.5-Enemies_SpawnAdjuster_SliderBar10.5/zscript/x5_EventHandler.zs:24-110 | Patch decimate to `ClearCounters(); Destroy()`; pick `x5_multiplier` per map | Thinkers fall in proportion | — | `server` cvar, deterministic | At 50: per-monster stack halves (every per-monster ms in [M] §3 × 0.5) |
| F-MON-12 | Fork gore caps changed from stock | Done (defaults) | [F] src/g_cvars.cpp:75 (`sv_corpsequeuesize` 10), 92 (`cl_maxdecals` 20) | Set them explicitly in the fixed netplay config (both are `CVAR_ARCHIVE`, the owner's ini wins) | All four machines report the same values in the netplay preflight | — | `cl_maxdecals` is per machine but decals are sim-list thinkers ([F] src/playsim/a_decals.cpp:664-694): keep it identical | Recommended values in §B.8 |
| F-MON-13 | BD22 per-blood ACS script that does nothing | Not started | [MOD] 1.00_brutal22test6/actors/Gore/Blood.dec:26; src/Options.acs:430-460; [F] src/playsim/p_acs.cpp:3172-3180 | Stopgap patch in [BDW]: `Brutal_BloodVR : Brutal_Blood replaces Blood` without the call | No `BD_CheckBloodIntensity` scripts in a fight | — | Deterministic | ~2-4 µs per blood spawn; at 50 hits/tic ≈ 0.1-0.2 ms |
| F-MON-14 | [BDW] voxel lender queues every TNT1-first spawn and reads `consoleplayer` cvars in play scope | Not started | [BDW] src/zscript/rsq/voxels.zs:44-56, 102-150 | Queue only `Inventory`; mask from a networked/user cvar per player | — | — | `A_ChangeModel` differs per peer: visual only, flag to netplay lane | O(n) `Delete` in a list of hundreds per tic in fights: ~0.1-1 ms |

---

## Findings

### B.1 What actually spawns: Legion of Bones shadows the whole BD22/XMVE monster roster

The load order is files `0.99`, `1.00`, `1.1`, `1.1a`, `1.2`, `1.3`-`1.6`, `2.1`... (owner's rotation, `HANDOFF_RENDER_2026-11-11.md`
"Load orders"). The last `replaces` wins.

- Legion of Bones replaces every vanilla monster through RandomSpawners, e.g. `SkullZombieSpawner : RandomSpawner replaces
  ZombieMan` / `DropItem "SkullZombie"`, `SkeletonImpSpawner ... replaces DoomImp`, `PhantomRevenantSpawner ... replaces
  Revenant` ([MOD] 2.1-Enemies_LegionOfBones_v1.05/DECORATE:22-25, 62-65, 219-222), and all 19 others (Fatso, Arachnotron,
  Baron, Caco, Mastermind, LostSoul, Spectre, PainElemental, Archvile, WolfensteinSS, Demon, ChaingunGuy, ShotgunGuy,
  HellKnight; same file).
- BD22's `Imp Replaces DoomImp`, `BullDemon: Demon Replaces Demon` etc. and XMVE 1.4/1.5's `ImpSpawner : BasicWeaponPickup
  Replaces DoomImp` ([MOD] 1.4-XMVE_MonsterExpansion_Doom2/Decorate.txt, 1.5-XMVE_MonsterExpansion_TUD/Decorate.txt) load
  earlier and lose. LegenDoom's 17 `LD<Monster> replaces <Monster>` (loaded first) lose too.
- The owner's LD mode only rewrites replacements whose result starts with "LD" ([BDW] src/zscript/rsld/handler.zs:297-307),
  so it does not hand monsters back to BD22.
- [M] §3 confirms it: `SkeletonImp` × 1,724 is exactly Sunder MAP15's `DoomImp` count without mods.

**Consequence.** On maps built from vanilla monsters, BD22's monster base classes, their per-monster ACS
(`SlaughtermapCheck`, `BDCHeckClassicMonsters`, `MeatShield_Handler`: [MOD] 1.00_brutal22test6/actors/Enemies/*.dec), their
death gore chains and XMVE 1.3-1.6's 73 monster files are **dead weight**: compile time, memory and class count, no tic cost.
What remains live from my area:

- BD22's **blood/puff/gib replacements**: `Brutal_Blood Replaces Blood`, `SawBlood ... Replaces AxeBlood`,
  `BloodSPlatterReplacer ... Replaces BloodSplatter`, `HitPuff Replaces BulletPuff`, `Gibs2`/`RealGibs2`
  ([MOD] 1.00_brutal22test6/actors/Gore/Blood.dec:3, 99, 139). Every Legion of Bones monster without its own `BloodType`
  bleeds BD22 blood.
- BD22's **decorations** (torches, lamps, columns, trees, barrels: [MOD] 1.00_brutal22test6/actors/DECORATIONS/*.dec) and
  items.
- **XMVE's item replacements** (health, armor, weapon spawners).
- LegenDoom's **legendary roll**, which works on any monster.
- **Universal Entropy** and **SpawnAdjuster**, which work on any actor.
- NiceWallBlood's **decals**.
- The **owner's handlers** in BD22 VR Reloaded.

Verification on the PC is one run (§D): `profilethinkers 8 0` lists every class; no `Imp`/`RifleZombie`/XMVE classes
should appear.

### B.2 Where the "missing" 78 ms can hide: the profiler does not clock spawn events

`ticprofile`'s `thinkers` (130.8 ms [M] §2) is the whole of `RunThinkers`. `profilethinkers` clocks only `CallTick` per
class:

```
if (node->ObjectFlags & OF_JustSpawned) { ... node->CallPostBeginPlay(); }     // before the clock
...
prof.timer.Clock(); node->CallTick(); prof.timer.Unclock();
```
([F] src/playsim/dthinker.cpp:1021-1044). `AActor::CallPostBeginPlay` runs `WorldThingSpawned` in every handler
([F] src/playsim/p_mobj.cpp:6103-6107). So **the cost of every spawned actor's spawn events is in the 130.8 ms but in no
class row**. A spawning actor's own `StaticSpawn` is inside its class time (the parent's tick).

The dispatch walks every handler. Each overriding handler pays `SetupWorldEvent` plus a `VMCall`, even when its body
returns at once ([F] src/events.cpp:698-699, 1811-1822). The loaded set has **43 `WorldThingSpawned` overrides**:

- 12 in BD22 VR Reloaded: `roster`, `rlock`, `recoil`, `flash`, `muzzle`, `flamespout`, `casings`, `ballistics`, `voxels`,
  `axegrab`, `grenade`, `rsld/handler` ([MOD] 1.2-BD22_VR-Reloaded_2.0.4t/zscript/...).
- 1 in BD22 (`Script_DamageMult`, [MOD] 1.00_brutal22test6/zscript/DamageHandler.zc:40-49).
- 1 in Universal Entropy, 1 in champions, 2 in the ambience mods, 1 in GunBonsai.
- 25 across the RSQ_* pk3s (grep count, RSQ_Monsters and RSQ_GunCards excluded).

**Inference, per spawn.**
- Shifter's whole tick (one VM `Tick`, `Super.Tick`, two native calls) is 0.29 µs [M] §3, so an empty handler call is about
  0.15-0.3 µs.
- 43 calls ≈ 6.5-13 µs, plus the non-trivial bodies (UE ≈ 1-2 µs: `IsFriend`/`isHostile`/`frandom`×2/`sqrt`, randomise.zs:17-34;
  RSLD's string `GetClassName().Left(2)` ≈ 0.5 µs, handler.zs:612-640), plus ~3 µs of native spawn.
- Total: **≈ 10-20 µs per spawned actor, of which ~8-13 µs is handler dispatch**.

**Other unattributed parts** of the gap (Inference):
- the profiler's own TMap lookup and two clock reads per thinker (≈ 50-100 ns × 53,116 ≈ 2.7-5 ms in the profiled tic);
- the loop's pointer chase;
- the long tail of classes under the printed cut: [M] §3 lists ~14 rows; `profilethinkers` with a limit prints only the top
  N ([F] dthinker.cpp:249-258).

[M] §3 cannot split these. 53,116 − 35,681 listed calls ≈ 17,400 thinkers sit in unlisted classes; at 0.3-3 µs they are
5-50 ms. So the gap is "long tail + spawn events + profiler overhead", in a proportion that only the §D run can give.
F-MON-0 is the tooling fix.

### B.3 Universal Entropy (2.4): the largest per-actor item in my area

The defaults make every roll certain:
- `ue_resize = 5`, `ue_recolour = 5`, `ue_decor_resize = 5`, `ue_decor_recolour = 5` ([MOD] 2.4-Enemies_UniversalEntropy/CVARINFO:5-14);
- each is tested as `ue_x >= random(1, 5)`, so 5 is 100% (randomise.zs:39, 66, 76, 90; its changelog says "Switch default
  settings for randomisation chances to always", CHANGELOG.txt 3.666).

What it does:
1. **Every monster** whose `A_SetSize(..., true)` succeeds gets a `Shifter` Inventory (randomise.zs:41-61).
2. **Every solid non-monster** gets one **whether or not the resize succeeded**: the `A_GiveInventory('Shifter')` sits outside
   `if (resized)` (randomise.zs:74-87). BD22's torch flames inherit `RedTorch`'s solidity (`TallTorch_RedFlames :
   TallTorch_Red`, [MOD] 1.00_brutal22test6/actors/DECORATIONS/Torches.dec:32-50 sets only `+THRUACTORS`), so one BD22 torch
   carries 2-3 Shifters (torch, flames, core). That is how Sunder reaches 7,686 Shifters [M] §3 on ~5,700 monsters
   (5,666 = 11,332 crit thinkers / 2).
3. **Shifter.Tick** runs every tic for the life of the owner: two `A_SoundPitch` calls on the owner, and an `A_SetTics`
   rescale on each owner state change (randomise.zs:152-166). On `main` each `A_SoundPitch` walks the whole channel list
   ([F] src/common/audio/sound/s_sound.cpp:1073-1085). Only the render branch indexes channels per source
   ([FR] s_sound.cpp:1089-1102). Torches and burning barrels start looping sounds (`A_Playsound("BDFireplace2", 1, 1, 1)`,
   Torches.dec:21; `"BDFireplace"`, Torches.dec:586); these feed the 2,914 evicted channels the render lane counted
   (SIM_AT_MOD_SCALE.md §1). **F-MON-5 is the difference between 2.2 ms and ~280 ms per tic on `main`.**
4. **Recolour** calls `GenerateRandomColorTranslation()` per monster and per solid decoration (randomise.zs:68, 92). Each call:
   - reads the `PLAYPAL` lump into a new String (`Wads.ReadLump`);
   - runs a 256-step loop with 1-4 `random` calls per step;
   - calls `AddTranslation` ([MOD] genct.zs:3-40).

   On the engine side:
   - `StoreTranslation` compares against **every** existing user translation, then `I_Error`s at 65,535
     ([F] src/common/engine/palettecontainer.cpp:273-290).
   - `AddRemap` scans all unique remaps (190-206).
   - Nothing clears `TRANSLATION_User` between maps: only `TRANSLATION_LevelScripted` is cleared ([F] src/p_setup.cpp:691).
   - Consequences: O(N²) work at map load; a cumulative count that will **abort the game after ~10 slaughter-map loads in one
     session** (Inference: ~7,000 per Sunder load, so 65,535/7,000 ≈ 9); and on the GPU side one translated copy of each
     sprite frame per monster, found by a linear `FindEx` over the copies ([F] src/common/textures/hw_texcontainer.h:80-120,
     comment "normally there aren't more than very few different translations"). In VR that means texture creation in the
     frame and VRAM growth.
5. **Missiles**: every missile with a non-player `target` gets a rotated velocity and a `FindInventory('Shifter')` on its
   target (randomise.zs:130-142). BD22 gore is spawned with `A_CustomMissile` (+MISSILE, target = spawner), so **every gore
   piece pays this**.
6. **Netplay**: `self.player = players[consoleplayer].mo` in `WorldLoaded` feeds `IsFriend`/`isHostile` in play scope
   (randomise.zs:5-11, 19). That is a different actor on each peer. In pure co-op the answers coincide, but it is a latent
   desync (F-MON-10).

**Engine plan of record:**
- F-MON-4 (native `SoundPitchScale`/`TicScale`, so the Shifter thinker disappears);
- F-MON-5 to main;
- F-MON-6 (hash, per-level reset, shader-side translation).

**Stopgap (cvars, `server`, identical on all peers):**
- `ue_recolour 0`, `ue_decor_recolour 0`: removes the translation abort and the texture copies;
- `ue_decor_resize 0`: removes ~2,000 decor Shifters, Inference 7,686 − ~5,700;
- keep `ue_resize` at 5 or lower it.

**Patch** ([MOD] randomise.zs:74-87): move the decor `A_GiveInventory` inside `if (resized)`, and give decor Shifters
`shift_speed 0` plus `SleepIndefinite()` after one tick, using the fork's API ([F] wadsrc/static/zscript/doombase.zs:284-292).
That is fork-only API, so the patched pk3 is REMA-only.

### B.4 BD22 blood: what one hit costs today

`Brutal_Blood` (spawned for every bleeding hit, [MOD] 1.00_brutal22test6/actors/Gore/Blood.dec:3-96):

- **At spawn:** `SuperGoreMistTiny` (l.23), an ACS script `BD_CheckBloodIntensity` (l.26), a sound (l.28), then after 2 tics
  `WaterBloodCHecker` (l.30), `Brutal_FlyingBlood` (l.32), `ArterialBlood22Small` (l.35), `BloodMist22Small` (l.36), then
  `LowBlood`: `A_RadiusGive("BloodSplasher_Red", 96, RGF_PLAYERS)` and 10 tics of animation (l.75-84). That is **5 child
  actors + 1 ACS script + 1 radius scan** per hit.
- **`Brutal_FlyingBlood`** flies (16 tics of frames, then a 100-tic frame, l.183-186). On floor contact it spawns
  `Brutal_BloodSpot` and animates 16 tics (l.199-205).
- **`Brutal_BloodSpot`** calls `A_QueueCorpse` and sits in a `-1` frame (l.810-836). `-1` stops the state clock, **not**
  `AActor::Tick`, so each resting spot still ticks until the corpse queue evicts it.
- BD22 has 221 `A_QueueCorpse` calls (`Limbs.dec` 134, `Meat.dec` 18, ...). All of them share `sv_corpsequeuesize`, whose
  fork default is **10** (stock 64) ([F] src/g_cvars.cpp:75 vs [U] src/g_cvars.cpp:73).
- **The ACS call is a no-op on GZDoom-family engines.** `isrunningzandronum` defaults to 1 ([MOD] CVARINFO.txt:45), so the
  script reads `BDCVarInfo[PlayerNumber()].bd_bloodamount` (src/Options.acs:430-445). The activator is the blood actor, so
  `PlayerNumber()` is −1, which is an out-of-range map-array read that returns 0 ([F] src/playsim/p_acs.cpp:3172-3180). The
  script then sets `LowBlood`, where the blood is already going. So **`bd_bloodamount` and `zdoombrutalblood` do not change
  monster blood in this load order**; the BD22 "blood amount" menu is not a lever (F-MON-13).
- NiceWallBlood overrides BD22's `BrutalBloodSplat`/`BrutalBlood` decal groups
  ([MOD] 3.05-Environmental_NiceWallBlood/DECALDEF:930-957). Each splat adds a `LowerDecal` smear whose `Animator` is a
  combiner of a stretcher and a slider (DECALDEF:290-304, 575-590): **2 decals + 2 native decal thinkers for 7-30 tics per
  wall hit**. The decal thinkers are native and cheap. The draw cost is the render lane's. The impact count is capped by
  `cl_maxdecals` (fork default **20**, stock 1024; [F] src/g_cvars.cpp:92-109, [U] :90).

**Fight model (Inference, to be replaced by the §D measurement).**
- Assume 4 players with BD22 shotguns/rifles into crowds, about **50 bleeding hits per tic**. Then 50 × 6 = **300
  spawns/tic**.
- Spawn cost: 300 × 10-20 µs (B.2) = **3-6 ms/tic**, mostly outside the class rows.
- Population: children live ~30 tics on average (flight to floor plus 16-tic anim; mist and arterial similar, Inference), so
  ~50 × 5 × 30 ≈ **7,500 live gore actors**. Each costs ~0.4 µs tick + 0.17 µs reset ≈ **4.3 ms/tic**.
- ACS: 50 × ~3 µs = 0.15 ms.
- **Gore total ≈ 7.5-10.5 ms/tic in a sustained fight**, on top of the standing-start numbers in [M].

### B.5 BD22 decorations: spawn streams that never stop

- **`TechLamp1` / `TechLamp21`** (replacing TechLamp/TechLamp2) spawn 2 `TechLamp1Flare` every 2 tics, forever, seen or not
  ([MOD] actors/DECORATIONS/Lamps.dec:45-58, 143-156). The flare lives 3 tics (l.102-108) and is *not* `+NOBLOCKMAP`
  (l.83-99): each one is linked into the blockmap and sector lists and pays the full spawn plus 43 handlers. That is **1
  spawn per tic per lamp**.
- **`Column1`** spawns one `Column1Flare` per tic while a player can see it (l.246-252, `A_CheckSight("Spawn2")`: 5-tic idle
  when unseen).
- **`BurningBarrel1`** spawns 10 `HumanBBQ` at start, then 2 missiles every 16 tics, forever (Torches.dec:586-604).
- **Torches** keep 3 looping helper actors each (flames 1-tic frames, core, ball; Torches.dec:24-26, 32-81) and a looping
  sound (l.21). GLDEFS gives the lamps and torches dynamic lights (`TechLamp1` → `BIGLAMP`, `TallTorch_Red` →
  `BIGREDTORCH`, [MOD] GLDEFS.txt:362-402).
- **Cost (Inference):** per 100 tech lamps/columns in view, 100 spawns/tic × 12-16 µs ≈ **1.2-1.6 ms/tic standing still**.
  The lamp count of Sunder MAP15 is not known here (§D).
- **Plan of record:** F-MON-2 (promote `TechLamp1Flare`, `TechLamp2Flare`, `Column1Flare`, torch flames/cores/balls to
  client-side) or F-MON-7 (attachments).
- **Stopgap:** `bd_disabledecorations 1` ([MOD] CVARINFO.txt:49; read by `BDCheckDecorations`, src/Options.acs:291-306)
  replaces them with vanilla, losing BD22's look. Alternatively, a [BDW] DECORATE subclass that `replaces TechLamp` and gates
  the flare loop on `A_CheckSight`/distance, the same pattern as `Column1`.

### B.6 LegenDoom 3.0 (0.99)

- **The roll.** `LDWorldInit` (OPEN) spawns a 4×4×4 grid of 64 choosers at 16,384 spacing (LEGENDWRD.acs:61-65). Each does
  `A_RadiusGive(..., 16384, RGF_MONSTERS|RGF_NOSIGHT|RGF_CUBE)` (MonsterGen.dec:9-55), so a monster near the origin is inside
  **8** cubes and gets 8 CustomInventory pickups. Each pickup spawns an item (43 spawn events), runs 3-4 `CallACS`, and rolls
  `A_Jump(PlayerCount*4 + LD_legendarychance*5)` then `A_Jump(240, skip)` (MonsterGen.dec:72-85). The default chance is 0
  (CVARINFO `LD_legendarychance = 0`), so the per-roll probability is 4/256 × 16/256 = 0.098% single player and 0.39% with 4
  players. **Inference:** ~0.8% of monsters single player, **~3% (≈175 on Sunder) in 4-player co-op**.
- **The one-time cost** is ~45,000 gives × 10-20 µs ≈ 0.5-1 s in one tic at map start (F-MON-9). `sim_maxcatchup` does not
  help in a netgame ([FR] src/d_net.cpp:2415, `!netgame`).
- **Per legendary.** A persistent ACS loop with `Delay(1)` reads health each tic (LEGENDMON.acs:37-109). After evolving at
  half health it gives `LDLegendaryMonsterVisualEffectMaker` **every tic** (l.43-46). That spawns an afterimage actor carrying
  a dynamic light and 3 particles per tic (MonsterVisuals.dec:18-170); the afterimage fades in ~5 tics (l.179-200).
  **Inference:** 175 scripts × ~1-2 µs ≈ 0.2-0.35 ms/tic. 20 evolved legendaries in a fight ≈ 20 spawns/tic (0.3 ms) and
  ~100 extra dynamic lights (render lane).
- **Dead parts.** LegenDoom's casing/smoke cvars (`LD_guncasings 2`, ...) belong to LD weapons, which the RSLD mode removes
  ([BDW] src/zscript/rsld/handler.zs `TickPlayer`), so they cost nothing.
- **Plan:**
  - The roll moves into [BDW]'s LD mode: one ZScript pass in `WorldLoaded`, probability `1-(1-p)^k`, which keeps LD's odds.
    This is mod-side by nature.
  - Legendary visuals become an overlay/attachment (P10.1/P10.2) instead of an actor per tic.

### B.7 The owner's BD22 VR Reloaded handlers ([BDW], 2.0.4u)

- **No per-monster thinkers and no per-tic scans over all monsters.** Every per-tic loop is per player:
  - `RSLD_Handler.WorldTick` → `TickPlayer` ([BDW] src/zscript/rsld/handler.zs:880-1000);
  - `counters.zs` uses a 256-unit `BlockThingsIterator` around players (src/zscript/rsq/counters.zs:180).
- **Their cost is the per-spawn dispatch** (B.2): 12 overrides. Most are a class/flag test. `rsld/handler.zs:612-640` builds a
  class-name string for every non-inventory spawn (`an.Left(2) == "LD"`), and `flamespout.zs:35-43` / `casings.zs:95-102`
  compare Names (cheap). All 12 are candidates for F-MON-1's declared filter: "missile whose target is a player", "class is
  `Rocket2`", "class in {PistolCasing, ...}", "not a monster".
- **`voxels.zs` pushes every spawned non-monster whose first frame is TNT1** to `pending` for up to 3 tics
  ([BDW] src/zscript/rsq/voxels.zs:102-113, 133-142). That is every BD22 gore piece and flare, and the list is trimmed with
  `Delete(i)`: O(n) per removal. Its `Mask()` reads `players[consoleplayer]` cvars and the result drives `A_ChangeModel` on
  world actors (44-56, 143-150): a per-peer difference in play state (visual only; F-MON-14). Patch: only `Inventory` goes to
  `pending`; the mask becomes a per-player `user` cvar applied in the renderer, or the lent model becomes a ui-side choice.
- **Packaging.** The rotation's 2.0.4t pk3 contains a nested 40 MB copy of a VR-Reloaded pk3
  ([MOD] 1.2-BD22_VR-Reloaded_2.0.4t/1.2-BD22_VR-Reloaded_2.0.4t.pk3). The engine mounts embedded archives only for `.wad`
  ([F] src/common/filesystem/source/resourcefile.cpp:102-117), so it is dead bytes. 2.0.4u's build does not include it
  ([BDW] build.py:17-22; `pk3/03_*.pk3` lists no `.pk3`).

### B.8 Gore/debris caps: defaults and the setting for 90 Hz

| Cvar | Scope | Stock / fork default | Who uses it here | Recommended (fixed netplay config) | Why (Inference) |
|---|---|---|---|---|---|
| `sv_corpsequeuesize` | server, SERVERINFO ([F] g_cvars.cpp:75) | 64 / **10** | 221 BD22 `A_QueueCorpse` (blood spots, limbs, meat) | **96** | 96 resting pieces × 0.6 µs ≈ 0.06 ms/tic; 10 makes pools vanish within a second of a fight |
| `cl_maxdecals` | per machine, ARCHIVE ([F] :92) | 1024 / **20** | BD22 blood decals, doubled by NiceWallBlood `LowerDecal`s | **128**, same on all 4 | Decals are sim-list thinkers ([F] a_decals.cpp:664-694); keep the count identical; draw cost is render-lane (gate on `scene.translucent` ms) |
| `r_maxparticles` | per machine ([F] :115) | 4000 / 4000 | LegenDoom auras, `A_SpawnParticle` | 4000 | Particles are 0.72 ms/tic all-in [M] §2 |
| `zdoombrutaljanitor` | server ([MOD] CVARINFO.txt:43) | 0 | gives `LowGraphicsMode` → BD22 casings go to `null`, limbs lose collision (Options.acs:249-269; Casing.dec:145-186) | **0** | Casings are the VR set's identity (owner's real-mesh casings, [BDW] rsq/casings.zs) and only 4 players fire |
| `bd_bloodamount` / `zdoombrutalblood` | user / server | 2 / 2 | no effect on monster blood in this order (B.4) | leave | — |
| `bd_disabledecorations` | server | 0 | B.5 streams | 0 (stopgap 1 if F-MON-2/8 slip) | ~1.2-1.6 ms/tic per 100 lamps |
| `ue_recolour`, `ue_decor_recolour` | server | 5, 5 (100%) | B.3 | **0, 0** until F-MON-6 | Removes the translation abort and per-monster sprite copies |
| `ue_decor_resize` | server | 5 | B.3 | **0** | ~2,000 fewer Shifters ≈ 0.9 ms/tic (2,000 × (0.29 + 0.17) µs) |
| `x5_multiplier` | server ([MOD] 2.5.../cvarinfo.txt:1) | 100 (no-op, x5_EventHandler.zs:28-29) | slaughter density | per map, e.g. 50 on Sunder MAP15 (after the `Destroy` patch) | Scales every per-monster row in [M] §3 |
| `LD_legendarychance` | server | 0 | B.6 | 0 | Each +1 adds 5/256 per roll |

### B.9 Smaller items

- **BD22 `Script_DamageMult`** gives every monster a `Damage_Control` Inventory when `Skill >= 6`
  ([MOD] 1.00_brutal22test6/zscript/DamageHandler.zc:40-49; registered in MAPINFO.txt:3). That adds one thinker per
  monster on BD22's top skills; [M]'s skill is unknown. **Inference:** +5,700 × (0.3 + 0.17) µs ≈ 2.7 ms/tic if played at
  skill index ≥6. Fix: engine-free, make it a `ModifyDamage`-only behavior (P10.6) or check skill in `WorldThingDamaged`.
- **XMVE climbing imps** (`LOADACS` `CLIMBINGIMPS`, [MOD] 1.3-XMVE.../LOADACS.txt) apply only to XMVE imps, which are
  shadowed (B.1).
- **SpawnAdjuster's decimate** calls `A_Die` (x5_EventHandler.zs:104-107). That runs every death state (gore, drops, LD drops,
  crit and champion death logic) on tic 0, and the corpse stays as a thinker. Patch: `ClearCounters(); Destroy();`.

---

## 90 Hz contribution

**Budget.** At 90 Hz one frame is 11.1 ms. Stage-6 "thread first" was closed by measurement
(`HANDOFF_RENDER_2026-11-11.md`: ≥13 ms a tic to copy the world), so the sim tic runs inside a frame. A frame that runs a
tic must fit tic + render in 11.1 ms.

**Inference:** with ~5-6 ms of headset render after the render lane's stages, the whole tic gets ~5 ms. My area's share
(monster stack minus champions, LoB code, crits, IA_Sound and HP bars) should be **≤ 1.0 ms/tic standing and ≤ 2.0 ms/tic in
a sustained 4-player fight**, spawn events included.

**Today (Inference from [M] and the models above).** Standing at Sunder MAP15's start:

| Item | ms/tic | Source |
|---|---|---|
| Shifter (renderer build) | 2.23 | measured |
| Shifter's share of the interp reset | 1.3 | 7,686 × 0.17 µs |
| Decoration spawn streams | ~0.6-1.6 | per 50-100 lamps/columns |
| Torch helpers' ticks | ~0.1-0.3 | — |
| LD scripts | ~0.1-0.3 | — |
| **Total standing** | **≈ 4.3-5.7** | |

On a `main` build without F-MON-5, add ~280 ms. In a fight, add the gore model (B.4) of **≈ 7.5-10.5 ms/tic**. Map start
adds a 0.5-1 s LD hitch, and maps add translation load spikes.

**Gap.** About 3.3-4.7 ms/tic standing and ~10-14 ms/tic in a fight over this area's share.

**Closing it (plan of record):**

| Fix | Saving |
|---|---|
| F-MON-4 | -3.5 |
| F-MON-1 | -2-3 in fights |
| F-MON-2 for BD22 gore and flares | sim gore -5-8 in fights, standing flares -0.6-1.6 |
| F-MON-3 | caps at near-zero tick cost |

Total: standing ≈ 0.5-1 ms, fight ≈ 1.5-2.5 ms (Inference). F-MON-5 and F-MON-12 values are prerequisites.
F-MON-6/9/10/14 are hitches and netplay hazards, not ms.

---

## Open questions

1. **Close the 78 ms.** Bridge `boottest -Map MAP15 -Tics 300 -Exec`, with a cfg (new name) that at tic 200 runs
   `profilethinkers 8 0` (all classes, by total) and `profilethinkers 4 0` (by calls), and `ticprofile` at 210. That is
   Sunder, the owner's zdl, `-rngseed 1113`, as rr347. Sum the full class list and compare with `thinkers`. Check that no
   BD22/XMVE monster class appears (B.1). Count `TechLamp1Flare`, `Column1Flare`, `TallTorch_*Flames`, `Brutal_*`.
2. **Spawn events.** Needs F-MON-0 (a build). Then the same run prints spawns/tic and the spawn-event ms. Without a build:
   run (1) once with `-file` of a tiny pk3 that removes the 12 VR-Reloaded spawn overrides. That is not legal for the owner's
   set, so measurement only.
3. **Fight number.** A recorded 4-player-style demo is not available. Use `benchreplay` on the owner's next Sunder MAP15
   recording (Desktop `RENDER TEST` bat), `perf_track` `think_ms`, `profilethinkers 8 0` at the fight's peak. That validates
   B.4's 300 spawns/tic and 7,500 live gore.
4. **The owner's ini values** for `sv_corpsequeuesize`, `cl_maxdecals`, `x5_multiplier`, the `ue_*` cvars and the skill:
   bridge `python` reading `gzdoom-*.ini` (the fork defaults 10/20 apply only if the ini has no entry).
5. **The translation count after N loads.** Boottest Sunder MAP15 five times in one session (`map MAP15` ×5 via `-Exec`). Watch
   for "Too many DECORATE translations"; check log size/VRAM in `perf_track`.
6. **Skill index in [M]'s runs** (B.9): the bridge cfg's `skill` line.
7. **`main` vs renderer Shifter cost** (F-MON-5): the same boottest on `doomxr` main exe, `profilethinkers 8 30`. Expected
   tens of ms for `Shifter` if the evicted channels are there.
