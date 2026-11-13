# 10 — TIME MANIPULATION

**Engine surveyed:** tag `qzd17.3` of `github.com/emawind84/gzdoom` (QuestZDoom PC edition).
All paths below are repo-relative at that tag. Ground truth: `_BASELINE_FACTS.md`.

**Goal of this survey:** the Matrix lobby scene, built as a mod, efficiently.

> **Revised after the baseline correction of 2026-10-30: both hands are fully tracked.**
> The main hand is readable as `Actor.AttackPos` / `AttackPitch` / `AttackRoll` / `AttackAngle`
> (`wadsrc/static/zscript/actors/actor.zs:268-271`, all `native readonly`; thunks
> `src/scripting/vmthunks_actors.cpp:2187-2190`), filled from the real controller transform at
> `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:586-597` (OpenVR path:
> `src/rendering/gl/stereo3d/gl_openvr.cpp:2525-2537`), symmetric with the offhand.
> `hw_vrmodes.cpp:442-443` is the **non-VR fallback** where `AttackPos` is merely set equal to
> `OffhandPos`; gate on `Actor.OverrideAttackPosDir` (`actor.zs:276`, set at
> `hw_vrmodes.cpp:438`) — false means not in VR.
>
> **§4.6 and §4.8 are rewritten for two hands.** Nothing in §4.1–§4.5 or §5 changed because of
> it — the time machinery never depended on hand count. Still true and unchanged: no velocity
> fields, 35 Hz sampling of a per-frame value, haptics unreachable (§5.5).

---

## 1. Verdict

The Matrix lobby is **buildable today, in pure ZScript, with no engine change**, and the
mechanism is better than the one the brief assumed. GZDoom has a **per-actor skip-this-tic
counter**, `AActor::freezetics` (`src/playsim/actor.h:1284`), which is a writable native field
in ZScript (`wadsrc/static/zscript/actors/actor.zs:265`) and is checked at the very top of
`AActor::Tick` (`src/playsim/p_mobj.cpp:3948-3952`) — before states, before velocity, before
everything. Driving it as a **duty cycle** (freeze N tics, run 1) slows an actor by exactly
1/(N+1) with animation, velocity, AI and sound *all in sync*, which sidesteps the classic
tics-vs-velocity desync trap entirely. On top of that, the engine already ships a true global
freeze — `FLevelLocals::frozenstate` via `Level.SetFrozen()`
(`wadsrc/static/zscript/doombase.zs:511`, `src/scripting/vmthunks.cpp:2612-2615`) — with a
per-player exemption (`player.timefreezer`, writable, `player.zs:3079`) and a per-actor
exemption flag (`MF5_NOTIMEFREEZE`, `src/playsim/actor.h:315`). That triple gives you an
**O(1) world freeze plus a hand-picked cast of actors that keep moving** — which is the lobby
scene, and it costs almost nothing.

There is a real global time scale, `i_timescale` (`src/d_main.cpp:217-234`), but it is
**unusable for this**: it scales the player too, and ZScript is forbidden from setting it
(`src/common/scripting/interface/vmnatives.cpp:981-990`).

**The VR answer, which matters more here than anywhere:** freezing or slowing the playsim
does **not** stutter the VR view. Head look is re-derived from the live OpenXR pose every
rendered frame and written straight into the viewpoint *after* interpolation
(`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:736-751`), so yaw/pitch/roll are
frame-rate smooth regardless of what the playsim clock is doing. What *does* stutter is the
player's **translation** (locomotion is playsim), the world geometry, and the slowed actors
themselves — see §5.2, this is the one genuine quality problem and it has a fix.

**Rewind is a WALL** and the missing piece is named in §5.1.

---

## 2. What already exists

### 2.1 Global freeze — EXISTS, shipped, ZScript-reachable

| Thing | Citation |
|---|---|
| `FLevelLocals::frozenstate` (int, bit 0 = script freeze, bit 1 = `freeze` CCMD) | `src/g_levellocals.h:686`, `:368-371` |
| `Level.isFrozen()` / `Level.SetFrozen(bool)` in ZScript | `wadsrc/static/zscript/doombase.zs:510-511` |
| native thunks | `src/scripting/vmthunks.cpp:2601-2623` |
| The actual gate — `AActor::isFrozen()` | `src/playsim/actorinlines.h:174-195` |
| Where it stops an actor dead (two sites: `MF5_NOINTERACTION` path and normal path) | `src/playsim/p_mobj.cpp:3972-3983` and `:4025-4036` |
| Specials (doors, lifts, scrollers) also stop | `src/p_tick.cpp:176-179` |
| Particles stop unless `SPF_NOTIMEFREEZE` | `src/playsim/p_effect.cpp:292`, flag at `p_effect.h:71` / `constants.zs:721` |
| `VisualThinker`s stop unless `SPF_NOTIMEFREEZE` | `src/playsim/p_effect.cpp:1282-1290` |
| Bots stop | `src/playsim/p_user.cpp:1330-1333`, `src/playsim/bots/b_bot.cpp:143` |

### 2.2 Per-player exemption — EXISTS, writable from script

`player_t::timefreezer` is a **bitmask of which players are exempt**
(`src/playsim/d_player.h:388`), declared in ZScript as a plain writable `native int`
(`wadsrc/static/zscript/actors/player/player.zs:3079`), field-bound at
`src/playsim/p_user.cpp:1886`.

The exemption is applied in `AActor::isFrozen()` (`src/playsim/actorinlines.h:186-190`):

```
if ((state & 1) && player->timefreezer == 0) return true;
```

So: a player actor with **any** bit set in `timefreezer` is **not frozen** by a
`Level.SetFrozen(true)`. **A mod can set this directly — it does not need `PowerTimeFreezer`
at all.** That single line is the whole Matrix lobby.

Note the asymmetry the comment in that function calls out: the script freeze (bit 0) respects
`timefreezer`; the `freeze` CCMD global (bit 1) does not freeze players at all.

### 2.3 Per-actor exemption — EXISTS

`MF5_NOTIMEFREEZE` (`src/playsim/actor.h:315`), registered for DECORATE/ZScript as
`+NOTIMEFREEZE` at `src/scripting/thingdef_data.cpp:254`, so settable at runtime as
`a.bNoTimeFreeze = true`. Checked first thing in `AActor::isFrozen()`
(`src/playsim/actorinlines.h:178`). An actor with this flag ignores the world freeze
completely.

### 2.4 Per-actor skip-tics counter — EXISTS, writable, and it is the good one

```
src/playsim/actor.h:1284
uint32_t freezetics;  // actor has actions completely frozen (including movement)
                      // for this many tics, but they still get Tick() calls
```

```
src/playsim/p_mobj.cpp:3948-3952   (top of AActor::Tick, before everything)
if (freezetics > 0)
{
    freezetics--;
    return;
}
```

- ZScript: `native uint freezetics;` — **not readonly** — `wadsrc/static/zscript/actors/actor.zs:265`
- Field binding: `src/scripting/vmthunks_actors.cpp:2130`
- Serialised into savegames: `src/playsim/p_mobj.cpp:358`
- Also short-circuits `isFrozen()`: `src/playsim/actorinlines.h:176-177`

This is **not** `+NOTIMEFREEZE`'s opposite number and it is **not** the level freeze. It is a
countdown that makes a single actor skip whole tics, and it is the mechanism this whole survey
is built on. See §4.1.

### 2.5 `PowerTimeFreezer` — read in full

`wadsrc/static/zscript/actors/inventory/powerups.zs:1502-1627`.

**What it does:**
- `InitEffect` (`:1516-1557`): pauses music via `S_PauseSound(false, false)` (`:1526`);
  sets `timefreezer` bits for the owner **and every teammate** (`:1529-1542`); forces
  `EffectTics` odd and aligns the start to an even `maptime` (`:1545-1556`); calls
  `Level.SetFrozen(true)`.
- `DoEffect` (`:1569-1586`): **only acts on even `maptime`** (`:1575-1578`) — the comment says
  this is so the Revenant tracer (`A_Tracer`, which keys on `maptime & 3`) is not thrown off.
  Then it re-asserts `Level.SetFrozen(...)` with a blink pattern that strobes the freeze on and
  off faster and faster over the last 4×32 tics as a wearing-off tell (`:1580-1585`).
- `EndEffect` (`:1593-1626`): clears this player's `timefreezer` bit from **all** players,
  checks whether anyone still has bits, and only then `Level.SetFrozen(false)` + `S_ResumeSound`.

**What it freezes:** every `AActor` whose `isFrozen()` returns true — i.e. all monsters,
projectiles, items, corpses; all sector specials (`src/p_tick.cpp:176`); particles;
`VisualThinker`s; bots.

**What it does NOT freeze:**
- The holder and his teammates (`timefreezer` bits).
- Any actor with `+NOTIMEFREEZE`.
- Any actor with `+BOSSCUBE` — it still increments `special2`
  (`src/playsim/p_mobj.cpp:3975-3978`, `:4028-4031`), so Icon-of-Sin cubes do not get
  accelerated by the freeze.
- **Player weapon psprites.** `P_PlayerThink` is called from `src/p_tick.cpp:166-168`,
  *outside* `RunThinkers`, and is not gated on the level freeze (only bots are). It reaches
  `PlayerPawn.PlayerThink` → `player.mo.TickPSprites()`
  (`wadsrc/static/zscript/actors/player/player.zs:1798`). The weapon keeps animating and firing.
- **Decoupled model animations** are advanced by one tic of `startTic` so they visually hold
  still rather than drift (`src/playsim/p_mobj.cpp:3980-3983`).
- The renderer. Nothing about the frame loop changes.

**Can it be made partial?** Yes, three independent ways, all already in the engine:
1. Per-player, via `timefreezer` bits — exempt any set of players.
2. Per-actor, via `+NOTIMEFREEZE` — exempt any set of actors, settable at runtime.
3. Per-tic, via the strobe pattern `PowerTimeFreezer.DoEffect` itself demonstrates
   (`:1580-1585`) — toggling `SetFrozen` on alternating tics is already a shipped 50%
   global slow-mo. **But it must only be toggled on even `maptime`** (`:1575`), which caps the
   duty cycle resolution at 1/2 and is why §4.1 uses per-actor `freezetics` instead.

### 2.6 Global time scale — EXISTS but is a dead end for a mod

`i_timescale`, `src/d_main.cpp:217-234`:

```
CUSTOM_CVAR(Float, i_timescale, 1.0f, CVAR_NOINITCALL | CVAR_VIRTUAL)
```
It sets `TimeScale` (`src/common/utility/i_time.cpp:54`), which scales the clock *at the
source* in `GetClockTimeNS()` (`:67-72`). Everything downstream — `I_GetTime`,
`I_GetTimeFrac`, `I_WaitForTic` — is derived from it (`:120-187`). This is a true, complete
global time scale and it would look perfect.

**Why you cannot use it:**
- **ZScript cannot set it.** `CVar.SetFloat` throws unless the cvar is `CVAR_MOD` or the call
  is inside menu code: `src/common/scripting/interface/vmnatives.cpp:981-990` —
  `"Attempt to change CVAR '%s' outside of menu code"`. `i_timescale` is neither `CVAR_MOD`
  nor reachable from a menu that is open during play (opening a menu pauses, `src/p_tick.cpp:96`).
- **It scales the player too.** It slows the tic *generator*, so the player's own movement,
  weapon, and input sampling slow identically. That is Max Payne, not the lobby.
- **It is refused in netgames** (`src/d_main.cpp:219-223`).
- There is **no ZScript `ConsoleCommand`** in this engine to route around it (searched
  `src/playsim/p_acs.cpp` and all of `src/` — the only `ConsoleCommand` hits are the C++
  `FConsoleCommand` console plumbing in `src/common/console/`).

**How close per-actor scaling gets:** very close, and in one respect closer. `i_timescale` is
continuous (any float ≥ 0.05); the `freezetics` duty cycle is **quantised to 1/(N+1)** — you
get 1/2, 1/3, 1/4, 1/5… and nothing between. For a lobby scene 1/4 and 1/8 are the only values
anyone asks for, so the quantisation is not a real loss. What per-actor scaling gives you that
`i_timescale` cannot: the player stays at full speed, and you choose who is slowed. What
`i_timescale` gives that per-actor cannot: slowed actors move *smoothly* rather than in
stepped jumps (§5.2).

### 2.7 Non-thinking statnums — EXISTS, the zero-cost freeze

`RunThinkers` only iterates statnums `>= STAT_FIRST_THINKING` (= 32):
`src/playsim/dthinker.cpp:147`, `:156`, `src/playsim/statnums.h:41-51`. A thinker parked below
32 is **never ticked at all** — zero per-tic cost, not even a counter decrement.

`Thinker.ChangeStatNum(int)` is exposed to ZScript (`wadsrc/static/zscript/doombase.zs:192`)
and the statnum enum with it (`:151-184`). The engine itself documents this as an intended
technique — see the `VisualThinker` comments at `wadsrc/static/zscript/visualthinker.zs:35-36`
("needs to be called if the thinker is set to a non-ticking statnum").

**The trap, and it is a bad one:** `FThinkerIterator` also starts at `STAT_FIRST_THINKING`
(`src/playsim/dthinker.cpp:938-940`, `:1030-1044`). An actor you park below 32 becomes
**invisible to every `ThinkerIterator` in the game, including your own un-freeze loop.** If
you use this you must keep your own `Array<Actor>` of what you parked. Collision/blockmap
still works (that is link-based, not thinker-based), and so does `BlockThingsIterator`.

### 2.8 Measurement tools — EXISTS

- `profilethinkers [+|-][t|a|#|c] [limit]` CCMD — per-class total time, average time, call
  count: `src/playsim/dthinker.cpp:833-870`, implementation `:189-266`.
- `stat think` — `src/playsim/dthinker.cpp:1054`.

---

## 3. Building blocks — the exact APIs

Everything in this section is ZScript-reachable with no engine change.

**Freezing and slowing**
| API | Declared |
|---|---|
| `uint Actor.freezetics` (read/write) | `actor.zs:265` |
| `bool Actor.isFrozen()` | `actor.zs:513` |
| `int Level.isFrozen()` / `void Level.setFrozen(bool)` | `doombase.zs:510-511` |
| `int PlayerInfo.timefreezer` (read/write bitmask) | `player.zs:3079` |
| `+NOTIMEFREEZE` / `bNoTimeFreeze` | `src/scripting/thingdef_data.cpp:254` |
| `bool PlayerInfo.IsTotallyFrozen()` | `player.zs:3197-3203` |
| `virtual bool PlayerPawn.CheckFrozen()` — **overridable**, zeroes input when frozen | `player.zs:1103-1134` |
| `void Thinker.ChangeStatNum(int)` + `Thinker.EStatnums` | `doombase.zs:192`, `:151-184` |

**Motion and animation**
| API | Declared |
|---|---|
| `vector3 Actor.Vel` (read/write) | `actor.zs:112` |
| `double Actor.Speed` (read/write) — what walking monsters actually use | `actor.zs:113`; consumed at `src/playsim/p_enemy.cpp:502` |
| `int Actor.tics` (read/write) | `actor.zs:134` |
| `A_SetTics(int)` | `actor.zs:1285` |
| `A_ScaleVelocity(double, ptr)` / `A_ChangeVelocity(...)` | `wadsrc/static/zscript/actors/actions.zs:35`, `:63` |
| `state Actor.FindState(statelabel)` / `SetStateLabel()` | `actor.zs:837-839` |

**Iteration**
| API | Declared |
|---|---|
| `ThinkerIterator.Create(class, statnum)` / `.Next(exact)` | `doombase.zs:201-206` |
| `BlockThingsIterator.Create(origin, radius)` / `.CreateFromPos(...)` | `doombase.zs:218-226` |

**Triggers and hooks**
| API | Declared |
|---|---|
| `EventHandler.WorldTick()` — runs **before** `RunThinkers` each tic | `events.zs:177`; ordering proven at `src/p_tick.cpp:171` vs `:173` |
| `WorldThingDied(WorldEvent)` — on-kill meter | `events.zs:163` |
| `WorldThingDamaged(WorldEvent)` (has `NewDamage`, writable) | `events.zs:166`, fields `:80-95` |
| `WorldHitscanPreFired(WorldEvent) -> bool` — **return true cancels the hitscan** | `events.zs:168`; honoured at `src/playsim/p_map.cpp:4718`; full attack params in `WorldEvent` at `events.zs:98-108` |
| `WorldHitscanFired`, `WorldRailgunPreFired/Fired` | `events.zs:169-171` |
| `NetworkProcess(ConsoleEvent)` / `EventHandler.SendNetworkEvent(name,a,b,c)` | `events.zs:201`, `:225` |
| `NetworkCommandProcess(NetworkCommand)` — typed payloads incl. `ReadVector3`, `ReadQuat` | `events.zs:202`, struct at `:11-34` |
| `RenderOverlay(RenderEvent)` (ui scope) | `events.zs:181` |

**Selling it — shaders**
| API | Declared |
|---|---|
| `PPShader.SetEnabled(name, bool)` | `wadsrc/static/zscript/engine/ppshader.zs:3` |
| `PPShader.SetUniform1f/2f/3f/4f/1i(shader, uniform, value)` | `ppshader.zs:4-8` |
| GLDEFS syntax: `HardwareShader PostProcess <beforebloom\|scene\|screen> { shader "<lump>" <330..450>; name "<id>"; uniform float <name>; enabled; }` | parser at `src/r_data/gldefs.cpp:1514-1763`; targets validated at `:1521-1527`; version range at `:1662-1663`; `name` at `:1565-1569`; `uniform` at `:1570+`; `texture` at `:1743-1751`; `enabled` at `:1753-1756` |
| `cvar_uniform` — bind a uniform straight to a `CVAR_MOD\|CVAR_ARCHIVE\|CVAR_VIRTUAL` cvar, int/float/color | `src/r_data/gldefs.cpp:1645-1740` |

**Selling it — sound**
| API | Declared |
|---|---|
| `A_StartSound(sound, slot, flags, volume, attenuation, **pitch**, startTime)` | `actor.zs:1199` |
| `A_StartSoundIfNotSame(...)` same signature | `actor.zs:1200` |
| `A_SoundPitch(int slot, double pitch)` — **retunes an already-playing channel** | `actor.zs:1202`; engine side `SoundEngine::ChangeSoundPitch` at `src/common/audio/sound/s_sound.cpp:1057-1068`, Doom wrapper `src/sound/s_doomsound.cpp:721-728` |
| `S_StartSound(id, chan, flags, vol, attn, **pitch**, startTime)` (global, non-positional) | `doombase.zs:135` |
| `S_StartSoundAt(vector3 pos, id, ...)` | `doombase.zs:136` |
| `S_PauseSound(notmusic, notsfx)` / `S_ResumeSound(notsfx)` | `doombase.zs:137-138` |
| `S_GetLength(id)` | `doombase.zs:140` |

**Selling it — particles, debris, trails**
| API | Declared |
|---|---|
| `A_SpawnParticle(...)` / `A_SpawnParticleEx(..., texture, style, roll, rollvel, rollacc)` | `actor.zs:1225-1226` |
| `Level.SpawnParticle(FSpawnParticleParams)` | `doombase.zs:560`, struct `:382` |
| `SPF_NOTIMEFREEZE` — particle keeps moving through a world freeze | `constants.zs:721`; honoured at `src/playsim/p_effect.cpp:292` and in the renderer at `src/rendering/hwrenderer/scene/hw_sprites.cpp:1513` |
| `VisualThinker` — a cheap sprite-only thinker with `Pos`, `Prev`, `Vel`, `Roll`, `Scale`, `Alpha`, `Texture`, render style | `wadsrc/static/zscript/visualthinker.zs:1-45`; spawn via `Level.SpawnVisualThinker()` (`doombase.zs:561`) or `VisualThinker.Spawn(...)` (`visualthinker.zs:38`) |
| `VisualThinker.IsFrozen()` | `visualthinker.zs:34`; engine side `src/playsim/p_effect.cpp:1282-1290` |

**VR**
See `_BASELINE_FACTS.md` §1. For this topic the two that matter are
`Actor.HmdPosition` (`actor.zs:277`) and `Actor.OffhandPos` (`actor.zs:272`), both written per
*render frame* (`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:442-445`,
`hw_openxrdevice.cpp:600-611`) and read per *tic*.

---

## 4. Mechanic designs

> **Read this section if you are building it.** Everything above is evidence; this is the build.

### 4.1 Per-actor slow-mo by duty-cycled `freezetics` — **BUILDABLE, cost S**

**What the player physically does:** presses the bullet-time button / hits the meter threshold
(see §4.7). Nothing else — this is the substrate every other mechanic sits on.

**How script detects it:** it doesn't; this is the resolver.

**How it resolves.** In an `EventHandler.WorldTick()` — which runs at `src/p_tick.cpp:171`,
guaranteed *before* `Thinkers.RunThinkers` at `:173`, so what you write this tic takes effect
this tic:

```zscript
// factor 4 => quarter speed. freezetics = factor-1 makes the actor skip
// (factor-1) tics and run 1, because p_mobj.cpp:3948 decrements and returns.
ThinkerIterator it = ThinkerIterator.Create("Actor");
Actor a;
while (a = Actor(it.Next()))
{
    if (!ShouldSlow(a)) continue;        // your filter: monsters + missiles, not the player
    if (a.freezetics == 0) a.freezetics = factor - 1;
}
```

**A frozen actor stays in the world — this is the property that makes the whole thing work.**
`AActor::Tick` returns at `src/playsim/p_mobj.cpp:3948-3952` *before* any
`UnlinkFromWorld`/`LinkToWorld` call and without touching `MF_NOBLOCKMAP`. The actor keeps its
blockmap link, so it is **still shootable, still blocks movement, still found by
`BlockThingsIterator`, still a valid `LineAttack`/`LineTrace` target, and can still be grabbed
and damaged** (§4.8). It simply does not act. That is exactly what a lobby scene needs and it
is why this beats every state-machine-based "freeze".

**Do not reach for `+NOINTERACTION` instead.** It looks like a freeze and is a trap: the
`MF5_NOINTERACTION` branch forces `MF_NOBLOCKMAP` and relinks the actor out of the blockmap
(`src/playsim/p_mobj.cpp:3988-3994`). The actor leaves the world — unshootable, non-colliding,
invisible to iteration. It is a performance shortcut for gibs, not a freeze.

**Why this is the right mechanism and not the trap the brief warns about.** The classic trap is
real: scaling `tics` slows state advancement but leaves `Vel` untouched, so a projectile keeps
its speed while its animation crawls; scaling `Vel` without `tics` desynchronises the other
way. `freezetics` has neither failure mode, because the check sits at the **top** of
`AActor::Tick` (`src/playsim/p_mobj.cpp:3948`) — above the state machine
(`:4498-4508`), above `P_XYMovement`/`P_ZMovement`, above the inventory `DoEffect` loop
(`:4006-4019`), above the rocket/grenade trail spawners (`:4038-4075`). Nothing in that actor's
tic happens. Animation, movement, AI, sound and inventory all advance at the same reduced rate
by construction. **You never touch `tics` or `Vel` at all.**

**Actors that do their work in `Tick()` overrides.** This is where you must be careful.
`AActor::Tick` is virtual and ZScript classes override it. The `freezetics` check lives in the
C++ `Super`, so:
- A ZScript `override void Tick()` that calls `Super.Tick()` **first** is fully slowed. This is
  the overwhelming majority.
- A ZScript `override void Tick()` that does work **before** `Super.Tick()` — timers,
  homing, particle emission, custom physics — **that work still runs every tic and is not
  slowed.** You cannot reach it from outside.
- Non-`Actor` thinkers (a mod's own `Thinker` subclasses, `VisualThinker`s) have **no**
  `freezetics` at all — that field is on `AActor` only (`src/playsim/actor.h:1284`). They need
  §4.2 or their own cooperation.
- `FastProjectile` already checks `isFrozen()` in its own ZScript tick
  (`wadsrc/static/zscript/actors/shared/fastprojectile.zs:53`), so it respects the *level*
  freeze — and `freezetics` also short-circuits `isFrozen()`
  (`src/playsim/actorinlines.h:176`), so it respects this too. Good.

For the uncooperative minority, fall back to §4.2 (level freeze + `+NOTIMEFREEZE`), which gates
inside `isFrozen()` and is therefore honoured by anything that asks the engine whether it is
frozen.

**Feel tuning knobs:** `factor` (2/3/4/8 — note it is quantised, §2.6); which classes are in
`ShouldSlow`; whether the factor ramps (set `factor` from a float and re-evaluate per actor each
tic for an ease-in rather than a snap); whether corpses and dropped items are included (they
should be — a corpse falling in slow motion sells it more than the monster that dropped).

**Restoring cleanly.** `freezetics` is a countdown, so **doing nothing restores it** within
`factor-1` tics. There is no cleanup state to leak, no saved original value, and it survives a
savegame correctly (`src/playsim/p_mobj.cpp:358`). This is the single biggest reason to prefer
it over scaling `tics`/`Vel`/`Speed`, where you must store and restore per-actor originals and
will leak them when an actor dies mid-effect.

### 4.2 Full world freeze with a cast of exceptions — **BUILDABLE, cost S**

**What the player physically does:** triggers the lobby moment.

**How it resolves:**

```zscript
// Freeze the world. O(1) — one int, no iteration.
players[consoleplayer].timefreezer |= 1 << consoleplayer;   // player.zs:3079
Level.SetFrozen(true);                                      // doombase.zs:511

// Hand-pick who keeps moving.
a.bNoTimeFreeze = true;                                     // thingdef_data.cpp:254
// ...and then slow *those* with §4.1, because freezetics is checked
// before the NOTIMEFREEZE test (actorinlines.h:176 vs :178).
```

To end it: clear the `timefreezer` bit for every player, then `Level.SetFrozen(false)`, then
clear `bNoTimeFreeze` on your cast. Copy `PowerTimeFreezer.EndEffect`
(`powerups.zs:1593-1626`) — it already handles the multi-player bookkeeping correctly.

**This is the efficient lobby.** The world costs nothing to hold still; you only pay per-tic for
the handful of actors you deliberately kept alive. Compare §4.1 alone, which pays an iteration
over every actor every tic.

**Hazards, all of them real:**
- `S_PauseSound` is **not** automatic here. `PowerTimeFreezer` calls it explicitly
  (`powerups.zs:1526`). If you want the music to drop out, call it yourself.
- Sector specials stop (`src/p_tick.cpp:176`). Doors and lifts in motion will hang. That is
  usually what you want; it will also strand a player riding a lift.
- `P_PlayerThink` runs regardless (`src/p_tick.cpp:166-168`), so psprites keep ticking
  (`player.zs:1798`) — the weapon animates even for a player you did *not* exempt. But that
  player's `AActor::Tick` still early-returns, so he cannot move and his `Vel` is dead. Set the
  `timefreezer` bit; do not rely on the psprite path alone.
- `PlayerPawn.CheckFrozen()` (`player.zs:1103-1134`) zeroes `forwardmove`/`sidemove`/`upmove`
  and masks `buttons` down to `BT_USE` for a totally-frozen player. It is `virtual`, so you can
  override it — but prefer the `timefreezer` bit, which fixes the cause rather than the symptom.
- Toggling `SetFrozen` per tic to fake a 50% slow is possible (it is literally what
  `PowerTimeFreezer.DoEffect` does, `:1580-1585`) but **only on even `maptime`**
  (`:1575-1578`) or you break `A_Tracer` homing, which keys on `maptime & 3`
  (`src/p_tick.cpp:118-120`). Use §4.1 instead.

### 4.3 Zero-cost parking for background actors — **HACK, cost S**

**How it resolves:** `a.ChangeStatNum(Thinker.STAT_STATIC)` (`doombase.zs:192`, enum `:159`).
The actor stops being ticked entirely (`src/playsim/dthinker.cpp:147`) at literally zero cost —
not even the `freezetics--` of §4.1.

**Use it for:** a large crowd that must be *stopped*, not slowed, and that the player will never
interact with during the moment — the far side of the lobby.

**Do not use it for** anything you will slow, damage, or grab.

**Named hazards:**
- Parked actors are invisible to `ThinkerIterator` (`src/playsim/dthinker.cpp:938-940`,
  `:1030-1044`). **Keep your own `Array<Actor>`** or you will never find them again.
- `STAT_STATIC` means "persistent across maps" (`src/playsim/statnums.h:50`). Un-park before a
  level change.
- `p_tick.cpp:155-162` iterates actors to `ClearInterpolation()`; a parked actor is skipped, so
  its interpolation is stale. It is not moving, so this is harmless — but it becomes a visible
  one-frame snap on the tic you un-park it.

**Label:** HACK, because the engine does not intend statnum as a freeze mechanism for actors
(though `visualthinker.zs:35-36` shows it does intend it for `VisualThinker`s).

### 4.4 Visible, dodgeable bullets — **BUILDABLE, cost M**

**What the player physically does:** sees a tracer leave a muzzle and moves his actual body out
of its line.

**How script detects the shot:** `WorldHitscanPreFired(WorldEvent e)` (`events.zs:168`). The
event carries the full attack: `e.Thing` (shooter), `e.AttackAngle`, `e.AttackPitch`,
`e.AttackDistance`, `e.AttackPos`, `e.AttackZ`, `e.AttackLineFlags`, `e.AttackPuffType`,
`e.Damage`, `e.DamageType` (`events.zs:98-108`).

**How it resolves:** return `true` and the engine abandons the hitscan — proven at
`src/playsim/p_map.cpp:4718`, where a true return makes `P_LineAttack` bail before tracing. In
its place, `Spawn` a slow projectile actor along `e.AttackAngle`/`e.AttackPitch` carrying
`e.Damage` and `e.DamageType`. Give it `+NOTIMEFREEZE` so it keeps flying through §4.2's world
freeze, then duty-cycle its `freezetics` (§4.1) to pick its speed.

**This is the mechanic the lobby is actually about.** Without it, slow-motion changes nothing
about incoming fire — hitscans resolve instantly and there is nothing to dodge.

**Feel tuning knobs:** projectile `Speed` × slow factor (aim for roughly 2–4 metres per second
apparent, which at 35 Hz and ~8 samples per second of player motion is dodgeable); trail
density; whether the projectile is a real `Actor` (collides, can be shot down, costs a full tic)
or a `VisualThinker` plus a scripted `LineTrace` at the end (cheap, cannot be interacted with).

**Hazard:** returning `true` also suppresses the puff, the decal and the sound. Spawn them
yourself at impact.

### 4.5 Selling it — **BUILDABLE, cost M overall**

**Post-process grade.** Ship a GLDEFS lump:

```
HardwareShader PostProcess scene
{
    shader "shaders/bullettime.fp" 330
    name "bullettime"
    uniform float amount
}
```
Parser: `src/r_data/gldefs.cpp:1514-1763`. Valid targets are `beforebloom`, `scene`, `screen`
(`:1521-1527`). Shader version must be 330–450 (`:1662-1663`). Then drive it from `WorldTick`:
`PPShader.SetEnabled("bullettime", true)` and `PPShader.SetUniform1f("bullettime", "amount", t)`
(`ppshader.zs:3-4`). You can also bind a uniform directly to a mod cvar with `cvar_uniform`
(`src/r_data/gldefs.cpp:1645-1740`) if you want a user-facing intensity slider for free.

**VR caution on the grade:** this runs per eye, in the scene buffer. Heavy chromatic
aberration, strong vignettes and any screen-space radial warp are nauseating in a headset and
break stereo fusion at the edges. Keep it to desaturation + a green push + a mild contrast
curve — which is the Matrix look anyway — and put the *motion* information into the world
(trails, debris), not into the post-process.

**Sound.** There is **no global pitch bus** in this engine. What you have:
- Spawn every sound at a scaled pitch: the `pitch` argument on `A_StartSound`
  (`actor.zs:1199`), `S_StartSound` (`doombase.zs:135`), `S_StartSoundAt` (`doombase.zs:136`).
- Retune a sound **already playing** with `A_SoundPitch(slot, pitch)` (`actor.zs:1202` →
  `SoundEngine::ChangeSoundPitch`, `src/common/audio/sound/s_sound.cpp:1057-1068`). This is the
  one that sells the *transition* — sweep every active combat channel down as the effect
  engages.
- Music: `S_PauseSound(false, false)` / `S_ResumeSound(false)` (`doombase.zs:137-138`) is
  all you get. There is no music pitch control. `PowerTimeFreezer` just kills the music
  (`powerups.zs:1526`); that is a legitimate choice and it reads as deliberate.

**Projectile trails, casings, debris.** Two tiers:
- `VisualThinker` (`visualthinker.zs:1-45`) for anything that is purely visual: casings,
  sparks, concrete chips, the smoke ribbon behind a round. It has `Pos`/`Prev` (so it
  interpolates), `Vel`, `Roll`, `Scale`, `Alpha`, `Texture`, render style — and it is far
  cheaper than an `Actor`. It respects the level freeze via `IsFrozen()`
  (`visualthinker.zs:34`, `src/playsim/p_effect.cpp:1282-1290`), so under §4.2 your casings
  hang in the air for free, with no per-tic cost at all.
- Particles via `A_SpawnParticleEx` (`actor.zs:1226`) or `Level.SpawnParticle`
  (`doombase.zs:560`) for volume. Pass `SPF_NOTIMEFREEZE` (`constants.zs:721`) on the ones that
  must keep drifting through a freeze — honoured in the playsim at
  `src/playsim/p_effect.cpp:292` **and** in the renderer at
  `src/rendering/hwrenderer/scene/hw_sprites.cpp:1513`.

**Shell casings hanging in the air specifically:** spawn them as `VisualThinker`s **without**
`SPF_NOTIMEFREEZE` and run §4.2. They stop dead and stay stopped, at zero cost, and resume
correctly when you unfreeze. This is the cheapest high-value shot in the whole scene.

### 4.6 The player in slow-mo — **BUILDABLE, cost M**

The player is never slowed. He runs at 35 Hz while the world runs at 35/factor.

**Both hands are available here.** Main hand: `AttackPos` / `AttackPitch` / `AttackRoll` /
`AttackAngle` (`actor.zs:268-271`). Offhand: `OffhandPos` / `OffhandPitch` / `OffhandRoll` /
`OffhandAngle` (`actor.zs:272-275`). Head: `HmdPosition` (`actor.zs:277`). All seven are
`native readonly` and all are written per *render frame*, read per *tic*. **Gate every one of
them on `Actor.OverrideAttackPosDir` (`actor.zs:276`)** — when it is false you are not in VR
and `AttackPos` is just a copy of `OffhandPos` (`hw_vrmodes.cpp:442-443`), so a two-hand
mechanic would silently collapse into a one-hand one.

**Leaning / body dodge.** `HmdPosition` (`actor.zs:277`) is a vector3 in metres, written per
render frame (`hw_vrmodes.cpp:442-445`). Store it each `WorldTick` and difference against the
previous tic to get a lateral velocity. A lean is a sustained lateral offset from a running
baseline; a *dodge* is a lateral speed spike. At 35 Hz a real dodge is 3–6 samples, which is
enough for a speed threshold but **not** enough to reconstruct the arc — do not try to.
See `_BASELINE_FACTS.md` §3.

**Resolution:** a successful lean is a *gate*, not a hit test. Compute the player's lateral
offset from his pawn centre, and use it to widen the miss window on incoming §4.4 projectiles
— i.e. shift the projectile's effective hit check by the HMD offset rather than trying to
detect a dodge event. This is both more robust at 35 Hz and more forgiving to play.

**Catching a bullet — BUILDABLE, cost M.** This is the shot the scene is named for and it is
now available. You own the §4.4 projectiles. Each tic, for each slowed projectile, test its
position against **both** hand positions (`AttackPos` and `OffhandPos`) with a generous radius.
On a hit: destroy the projectile, spawn a held `VisualThinker` parented to that hand, and keep
it there until the player's hand speed (differenced across tics) exceeds a throw threshold.
Because the projectile is slowed to a few metres per second (§4.4) and you are sampling two
hands rather than one, a 35 Hz sample rate is sufficient — the brief's sampling constraint bites
on *fast* gestures, and a catch is a slow one. Use a capsule between the last two samples of
each hand rather than a point test, per `_BASELINE_FACTS.md` §3.

**Two-handed dodge reading.** With both hands plus the head you have three tracked points, so
you can distinguish a *lean* (head displaces, hands follow rigidly) from a *reach* (hand
displaces, head does not) from a *duck* (head drops). That removes the main ambiguity that made
one-hand gesture work unreliable. Keep it to thresholds on offsets; do not attempt arc
reconstruction at 35 Hz.

**Walls, named:**
- **Height is a cvar-driven crouch, not a pose.** The OpenXR layer overwrites
  `player->crouchfactor` every frame from the HMD height
  (`hw_openxrdevice.cpp:574-580`), unless `vr_crouch_use_button` is set. Ducking *does* work,
  physically, and the engine already does it for you — but you do not own that channel and
  must not fight it.
- **Nothing in script can move the VR camera** and nothing should try.
- **No hand or head velocity fields exist** (`_BASELINE_FACTS.md`). Every speed is yours to
  difference, for all three tracked points.
- **Wall-running is a WALL.** See §5.3.

**Feel tuning knobs:** lean threshold in metres; how much of the HMD lateral offset transfers
to the hit window (start at 1.0 — it is the honest mapping); whether the slow factor deepens
while the player is leaning (it should — it is the reward).

### 4.7 Meter and triggers — **BUILDABLE, cost S**

**On kill:** `WorldThingDied(WorldEvent e)` (`events.zs:163`); `e.Thing` is the victim,
`e.Inflictor` the cause. Credit the meter when `e.Inflictor` traces back to a player.

**On near-miss / dodge:** you own the §4.4 projectiles, so you know when one passed within
*r* of `HmdPosition` without hitting. That is the dodge trigger, and it is exact — no gesture
recognition required.

**On damage avoided:** `WorldThingDamaged` (`events.zs:166`) has a writable `NewDamage`
(`events.zs:97`) if you want a graze to cost less instead of nothing.

**On button:** `PlayerInfo.cmd.buttons` read in `WorldTick`, or a bound CCMD →
`EventHandler.SendNetworkEvent(name, a, b, c)` (`events.zs:225`) → `NetworkProcess`
(`events.zs:201`). The network route is the correct one — it is playsim-scope and
multiplayer-safe.

**Display:** `RenderOverlay(RenderEvent)` (`events.zs:181`, ui scope). In VR a screen-space
overlay is uncomfortable; prefer a wrist/world indicator drawn off the offhand (`OffhandPos`,
`actor.zs:272`).

**Cost note:** the meter itself is free. Do not iterate actors to maintain it.

### 4.8 Interaction with melee and grabbing — **BUILDABLE, cost S**

**Melee gets easier, not harder, and it is now two-handed.** A hand-swept melee check
differences a hand position across tics and sweeps that segment against targets. Run it for
**both** hands — `AttackPos` (`actor.zs:268`) and `OffhandPos` (`actor.zs:272`) — so a shove
with one hand and a strike with the other are distinguishable. When targets are frozen (§4.2)
or duty-cycled (§4.1), their position is **constant across your sample window**, which removes
the sample-rate problem entirely: you are intersecting a moving segment against a static point
instead of two moving things. **Slow-mo is the best case for 35 Hz melee, not the worst.**

**Grabbing works on frozen actors, and now with either hand.** `freezetics` and `isFrozen()`
only stop the actor's *own* `Tick`. They do not stop **you** from writing its position,
velocity or state from outside. `SetOrigin`, direct `Vel` writes and `SetStateLabel` all still
apply, and critically the actor **stays in the blockmap** (§4.1), so it remains findable with
`BlockThingsIterator` and remains a valid trace target while held. So you can grab a frozen
enemy with either hand, reposition him, and release him — the classic lobby beat. With both
hands readable, a two-handed grab (both hand positions inside the target's radius) is a
distinct, reliable gesture worth using as the trigger for a throw.

**But know what does not happen while he is frozen:**
- He will not enter his **pain state** — state advancement is below the early return
  (`src/playsim/p_mobj.cpp:3948` vs `:4498`). `DamageMobj` still applies damage and still kills
  him; he just will not *flinch* until he ticks. Call `SetStateLabel("Pain")` yourself if you
  want the reaction.
- His **inventory `DoEffect` loop** does not run (`src/playsim/p_mobj.cpp:4006-4019`) — powerup
  timers on him are paused.
- His **blockmap link is not refreshed** by his own tic. If you move him, move him with
  `SetOrigin` (which relinks) rather than by writing `Pos` directly.
- If you parked him via §4.3 you must un-park him **before** releasing him, or he will hang
  mid-air forever and you will not be able to find him with a `ThinkerIterator`.

---

## 5. Walls

### 5.1 Rewind — **WALL**

**It is a wall, and here is exactly why.** Rewind requires snapshotting and restoring actor
state. The engine has exactly one mechanism that does this — the savegame serialiser
(`AActor::Serialize`, `src/playsim/p_mobj.cpp:251-360`) — and **it is not exposed to ZScript in
any form.** There is no `Actor.Clone()`, no snapshot object, no `Thinker` copy constructor, no
script-reachable serialisation. Searched: all of `wadsrc/static/zscript`, and the VM thunks in
`src/scripting/vmthunks_actors.cpp`.

**The missing piece, named:** a script-reachable per-actor state snapshot/restore pair —
concretely, a ZScript-visible `Actor.SaveSnapshot() -> Object` / `Actor.RestoreSnapshot(Object)`
built on the existing `FSerializer` path, or at minimum a `Thinker`-level deep copy.

**What you *can* do instead — "replay", not rewind — BUILDABLE, cost L.** Per tic, for a
**bounded** set of actors (≤ ~32), record a struct of `Pos`, `Angles`, `Vel`, `health`,
`tics`, and the *name* of the current state. To rewind, walk the ring backwards applying
`SetOrigin` + angle + `SetStateLabel`. This reproduces the *look* of a rewind for the things
the camera is on.

**What it will not reproduce**, and say so to the team up front: inventory, AI target pointers
and `movecount`/`movedir`, RNG, particles and `VisualThinker`s, sector state, anything spawned
or destroyed in the window (a rewound death is a **respawn**, which is a different actor with a
different pointer), and anything a mod stored in its own fields. Budget for it being a
set-piece effect on a scripted cast, never a general mechanic.

### 5.2 Smooth motion for slowed actors — **WALL (quality), workaround exists**

This is the one genuine quality problem with §4.1 and the team must know it before they build.

**The mechanism, exactly.** At the top of every tic, *before any thinking*,
`src/p_tick.cpp:152-162` calls `ClearInterpolation()` on every actor, which does
`Prev = Pos()` (`src/playsim/actorinlines.h:29-35`). The renderer then lerps `Prev → Pos` across
that one tic of wall-clock time. **This is good news in general** — any position the playsim
writes during a tic, including one your own controller writes, interpolates smoothly to the
headset. It is the reason the §5.2 workaround below actually looks right.

**But it is precisely why the duty cycle strobes.** An actor under `freezetics` moves its
**full one-tic distance on its one live tic, then holds still for `factor-1` tics**. `Prev` is
re-snapshotted each of those tics, so the renderer draws it perfectly still for the frozen tics
and then lerps the whole step across a single tic of frame time — not across `factor` tics.
Timing is correct; motion is **stepped and strobing** — step, hold, hold, hold, step. At factor
4 that is a visible 8.75 Hz judder. In a VR headset at 72–90 Hz, with the player free to lean
in and stare at it, that reads as broken.

**The missing piece, named:** a per-actor interpolation stretch — the renderer would need to
lerp a frozen actor's `Prev`→`Pos` over `freezetics+1` tics instead of one.

**Workaround — and it is the place, the only place, where the tics-vs-velocity split is the
right tool:** for the **few** actors the player is looking at (the bullets of §4.4, the one
hero enemy), do **not** use `freezetics`. Instead let them tick every tic and halve both halves
in lockstep:
- Motion: scale `Vel` by `1/factor` (`actor.zs:112`) — for projectiles and anything airborne.
- Walking monsters do **not** move by `Vel`. `P_Move` reads `actor->Speed`
  (`src/playsim/p_enemy.cpp:502`). Scale `Speed` (`actor.zs:113`) for those.
- Animation: multiply `tics` by `factor` on each state entry (`actor.zs:134`, or
  `A_SetTics`, `actor.zs:1285`).
- Both, or you get exactly the desync the brief warns about: `Vel` alone gives a sprinting
  animation drifting slowly; `tics` alone gives a crawling animation flying at full speed.

This is per-tic smooth — the actor moves a small distance *every* tic, and the `Prev` snapshot
at `p_tick.cpp:152-162` makes the renderer lerp each small step across a full tic of frame
time. It costs a full `Tick` per actor, which is why it is reserved for the handful in close-up.

**Variant — the puppeted actor (BUILDABLE, cost M).** Because `Prev` is snapshotted *before*
thinking, a position your controller writes during the tic also interpolates smoothly. So you
can hold an actor permanently frozen (`freezetics = 1`, refreshed every tic) and move it
yourself with `SetOrigin` by `Vel/factor` each tic, advancing its state machine yourself every
`factor` tics with `SetStateLabel`. You get smooth motion plus total authority over the actor —
worth it for a single scripted hero bullet or the one enemy the camera is on, not for a crowd.
If you build a controller that must run **after** everything it manipulates has ticked, put it
in a thinker at a high statnum: `RunThinkers` walks statnums in ascending order
(`src/playsim/dthinker.cpp:145-149`), so `STAT_BOT` (104) ticks after the actors at
`STAT_DEFAULT` (100) — `wadsrc/static/zscript/doombase.zs:178-182`. **Use `freezetics` for the crowd, `Vel`+`Speed`+`tics` for the hero
actors, `ChangeStatNum` for the far background.** That three-tier split is the efficient lobby.

**Cleanup burden, which `freezetics` did not have:** you must store each hero actor's original
`Speed` and restore it, and you must stop multiplying `tics` on state entry. Track them in an
explicit array and clear it on `WorldThingDestroyed` (`events.zs:167`).

### 5.3 Wall-running — **WALL**

Wall-running needs the player pawn to be stuck to and driven along a wall plane against
gravity, with the view following. In this engine:
- Nothing in script can move the VR camera (`_BASELINE_FACTS.md`; the viewpoint is
  re-derived per frame at `hw_openxrdevice.cpp:736-751` and overwrites what the playsim set).
- The player's `crouchfactor` — the only vertical channel — is already owned and overwritten
  per frame by the OpenXR layer (`hw_openxrdevice.cpp:574-580`).
- Roll on the player view is likewise set per frame from the HMD (`:740`), so you cannot bank
  the camera into a wall-run.

**The missing piece, named:** a script-settable camera/body offset that survives the per-frame
VR viewpoint overwrite — i.e. an additive offset applied *after* `OpenXRDeviceMode::SetUp`
rather than before it.

Note this is an engine wall *here*. UZDXREMA is a different engine and is out of scope for this
survey (`_BASELINE_FACTS.md` §6).

### 5.4 Global time scale from script — **WALL**

`i_timescale` exists (`src/d_main.cpp:217-234`) and works, but ZScript cannot set it
(`src/common/scripting/interface/vmnatives.cpp:981-990`), there is no script `ConsoleCommand`
in this engine, and it would slow the player anyway. **The missing piece, named:** either
`CVAR_MOD` on `i_timescale`, or a player-exempting variant of `TimeScale` applied to the
thinker loop rather than the clock. Do not build around it.

### 5.5 Haptics on the time-stop — **WALL**

A rumble on the freeze transition would sell it enormously. `VRMode::Vibrate` exists
(`src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`) and is wired through
(`hw_openxrdevice.cpp:754-757`) but has **no ZScript binding**. Worse, the event-driven entry
point is a **do-nothing stub**: `VR_HapticEvent` at `src/QzDoom/qzdoom_common.cpp:65` has an
empty body. Confirmed independently by a second survey. Treat as unavailable; do not design a
transition that needs it.

---

## 6. Cost

**The honest answer: I am not going to invent a number, and neither should the design doc.**
Here is the real per-actor work and exactly what to measure.

**What §4.1 costs per slowed actor, per tic:** one `ThinkerIterator.Next()` VM call, one field
read, one conditional, one field write — plus, in the engine, one `freezetics--` and an
immediate return from `AActor::Tick` (`src/playsim/p_mobj.cpp:3948-3952`). This is **far
cheaper than ticking the actor**, which would otherwise run the inventory `DoEffect` virtual
call loop (`:4006-4019`), `P_XYMovement`/`P_ZMovement`, blockmap unlink/relink, portal
transition checks, sector-special checks (`:4490-4496`), the state machine (`:4498-4508`), and
for monsters `A_Chase` with its `P_Move` (`src/playsim/p_enemy.cpp:498+`) and line-of-sight
checks. **Slowing actors makes the tic cheaper, not more expensive.** The only new cost is the
iteration.

**So the thing that can actually bite is the `ThinkerIterator` sweep in `WorldTick`,** which is
O(all actors) — every corpse, every dropped item, every decoration — once per tic, with VM
dispatch per `Next()`.

**Three cost tiers, cheapest first:**
1. §4.2 world freeze + `+NOTIMEFREEZE` cast — **O(1) per tic.** One int. This is why the brief's
   "efficiently" points here.
2. §4.3 statnum parking — **zero per tic**, one-time cost to park and un-park.
3. §4.1 duty cycle — **O(all actors) per tic.** Mitigate by using `BlockThingsIterator`
   (`doombase.zs:224`) around the player instead of a global `ThinkerIterator`, or by sweeping
   1-in-N tics (set `freezetics = k*factor - 1` on a less frequent pass).

**What to measure, concretely:**
- `profilethinkers -t 20` (`src/playsim/dthinker.cpp:833-870`) — total tick time per thinker
  class, descending, top 20. Run it on the lobby map with the effect off, then on. The number
  that matters is whether your `EventHandler` has climbed into the top five.
- `stat think` (`src/playsim/dthinker.cpp:1054`) — total think time per tic, as a single
  headline figure. Watch it across the on/off transition.
- `profilethinkers -# ` — call counts, to confirm your slowed actors really are skipping tics.
  **If a slowed actor's call count does not drop, it has a `Tick()` override doing work before
  `Super.Tick()`** (§4.1) and you have found it.
- In VR, the figure that actually decides it is **dropped frames at the headset's refresh**,
  not playsim ms. The playsim is one 35 Hz thread; a lobby's worth of actors is not what will
  miss a 90 Hz frame — the §4.5 post-process and the trail/particle count are. Budget the
  measurement effort accordingly: measure `stat think` once to confirm it is not the problem,
  then spend the rest of your time on frame timing.

---

## 7. Smallest prototype

One `EventHandler`, one GLDEFS lump, roughly 120 lines. Build in this order; each step is
independently verifiable.

1. **Prove the freeze.** `EventHandler` with a `NetworkProcess` toggle. On: set
   `players[e.Player].timefreezer |= 1 << e.Player` and `Level.SetFrozen(true)`. Off: reverse,
   copying `PowerTimeFreezer.EndEffect` (`powerups.zs:1593-1626`). **Verify:** monsters stop,
   you still walk and fire. *This alone is 60% of the lobby and it is an afternoon.*

2. **Prove the VR view is unaffected.** Hold the freeze and look around. Head motion must stay
   perfectly smooth (it will — `hw_openxrdevice.cpp:736-751`). This is the check that decides
   whether the whole topic is viable in VR; do it second, not last.

3. **Prove the duty cycle.** Pick one monster, set `bNoTimeFreeze = true`, and in `WorldTick`
   set `freezetics = 3` when it reaches 0. **Verify:** it walks at quarter speed with its
   animation matching, while everything else is stopped. Note the judder (§5.2) and decide then
   whether this one needs the hero treatment.

4. **Prove the dodge.** `WorldHitscanPreFired` → return true → spawn a slow `+NOTIMEFREEZE`
   projectile along `e.AttackAngle`/`e.AttackPitch` carrying `e.Damage`. **Verify:** you can
   physically lean out of the way.

5. **Sell it.** GLDEFS post-process + `PPShader.SetUniform1f` ramp on the transition;
   `A_SoundPitch` sweep on active channels; `VisualThinker` casings with no `SPF_NOTIMEFREEZE`
   so they hang.

Steps 1–2 are the go/no-go. Steps 3–5 are polish and can be cut.

---

## 8. Open questions

1. **Does a custom post-process shader run correctly per-eye under OpenXR?** I confirmed the
   GLDEFS parser and the `PPShader` script API, and that the scene target exists. I did **not**
   trace a custom shader through the stereo submit path (`OpenXRDeviceEyePose::submitFrame`,
   `hw_openxrdevice.cpp:238-259`) to confirm it is applied to both eye buffers rather than once
   to a shared one. **Test it before designing around it.**

2. **Exact VR judder threshold for §5.2.** I can show structurally that a `freezetics` actor
   moves in discrete steps interpolated over one tic. I cannot tell you at what slow factor it
   becomes objectionable in a headset — that is a measurement with a human in it. Run step 3 of
   §7 at factors 2, 4 and 8 and look.

3. **`A_SoundPitch` on an arbitrary channel of an arbitrary actor.** It is an `action` function
   (`actor.zs:1202`), so calling it on a non-self actor from an `EventHandler` needs checking.
   The underlying `SoundEngine::ChangeSoundPitch` takes an explicit source
   (`src/common/audio/sound/s_sound.cpp:1057`), so the capability is there; the script-side
   call convention is what I did not verify.

4. **Interaction with `A_Tracer` homing under per-actor `freezetics`.** `A_Tracer` keys on
   `maptime & 3` (noted at `src/p_tick.cpp:118-120`), which is **world** time, not the actor's.
   A Revenant missile under a duty cycle will sample its homing phase on whichever tics it
   happens to be awake for. It will probably look fine; it is not something I could prove by
   reading.

5. **Whether `ChangeStatNum` to a sub-32 statnum is safe across a savegame.** Thinkers
   serialise by statnum list; a parked actor should restore parked. Not verified.

6. **Catch radius and throw threshold for §4.6.** The capability is confirmed; the numbers are
   a tuning exercise with a human in a headset, not something readable from source. Start at a
   0.15 m catch radius and tune from there.

**Closed since the first draft** (baseline correction of 2026-10-30): whether the main hand is
readable — it is (`actor.zs:268-271`, gated on `OverrideAttackPosDir`). That upgraded
bullet-catching and two-handed melee/grab from speculative to BUILDABLE/M. **No label in this
report changed**, because nothing here was marked WALL or HACK solely for want of a second
hand — the time machinery never depended on hand count.
