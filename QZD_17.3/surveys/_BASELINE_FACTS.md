# Baseline facts — verified by hand before the topic surveys

**Engine surveyed:** tag `qzd17.3` of `github.com/emawind84/gzdoom` (emawind84 QuestZDoom PC edition).
All citations in every report are **repo-relative** paths at that tag (`src/...`, `wadsrc/...`).

This file is ground truth. Every topic report may rely on it without re-deriving it.
If a report contradicts something here, the report must say so explicitly and show its citation.

---

## 1. The entire VR surface exposed to ZScript

This is the whole list. There is nothing else.

| ZScript | Declared | Native field | Thunk |
|---|---|---|---|
| `Actor.HmdPosition` (vector3, readonly) | `wadsrc/static/zscript/actors/actor.zs:277` | `src/playsim/actor.h:1668` | `src/scripting/vmthunks_actors.cpp:2195` |
| `Actor.OffhandPos` (vector3, readonly) | `actor.zs:272` | `actor.h:1661` | `vmthunks_actors.cpp:2191` |
| `Actor.OffhandPitch` (double, readonly) | `actor.zs:273` | `actor.h:1662` | `vmthunks_actors.cpp:2192` |
| `Actor.OffhandRoll` (double, readonly) | `actor.zs:274` | `actor.h:1664` | `vmthunks_actors.cpp:2193` |
| `Actor.OffhandAngle` (double, readonly) | `actor.zs:275` | `actor.h:1663` | `vmthunks_actors.cpp:2194` |
| `Actor.OffhandDir(actor, angle, pitch)` | `actor.zs:859` | `actor.h:1666` (fn ptr) | — |
| `Actor.AttackPos` (vector3, readonly) | `actor.zs:268` | `actor.h:1654` | `vmthunks_actors.cpp:2187` |
| `Actor.AttackPitch` (double, readonly) | `actor.zs:269` | `actor.h:1655` | `vmthunks_actors.cpp:2188` |
| `Actor.AttackRoll` (double, readonly) | `actor.zs:270` | `actor.h:1657` | `vmthunks_actors.cpp:2189` |
| `Actor.AttackAngle` (double, readonly) | `actor.zs:271` | `actor.h:1656` | `vmthunks_actors.cpp:2190` |
| `Actor.OverrideAttackPosDir` (bool, readonly) | `actor.zs:276` | `actor.h:1652` | `vmthunks_actors.cpp:2186` |
| `PlayerInfo.PlayInVR` (bool, readonly) | `wadsrc/static/zscript/actors/player/player.zs:3136` | `src/playsim/d_player.h:502` | — |

### 1a. BOTH HANDS ARE FULLY TRACKED. (Corrected 2026-10-30.)

**An earlier draft of this file claimed there was no main-hand pose. That was wrong.**
It was caught by the grapple survey and verified by hand. The error came from reading
`hw_vrmodes.cpp:442-444` — which is the **non-VR fallback path**, where `AttackPos` is simply
set equal to `OffhandPos` and the player position — and mistaking it for the live VR path.

The live VR paths write a real, independent weapon-hand controller transform, exactly
symmetric with the offhand:

```
src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:586-597   AttackPitch/Angle/Roll/Pos from weaponangles[] and weaponoffset[]
src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:600-611   OffhandPitch/Angle/Roll/Pos from offhandangles[] and offhandoffset[]
src/rendering/gl/stereo3d/gl_openvr.cpp:2525-2537                  the OpenVR path, same shape
```

So the real picture is:

- **Main hand** — `AttackPos`, `AttackAngle`, `AttackPitch`, `AttackRoll`. Full 6DOF.
- **Offhand** — `OffhandPos`, `OffhandAngle`, `OffhandPitch`, `OffhandRoll`. Full 6DOF.
- **Head** — `HmdPosition`.
- Direction helpers for each: `AttackDir` (`actor.h:1659`) and `OffhandDir` (`actor.zs:859`).

**Two independently tracked hands plus a tracked head.** Any mechanic needing two hands doing
different things in free space is viable. Boxing with both fists, two-hand handlebars, a bow
drawn between two hands, hand-over-hand climbing — none of these are blocked.

`OverrideAttackPosDir` (`actor.zs:276`) is the flag saying the VR override is live; it is set
to `!puristmode && (IsVR() || vr_override_weap_pos)` at `hw_vrmodes.cpp:438`, and the engine
branches on it all through `p_map.cpp` (`:4597`, `:4647`, `:4784`, `:4888`, `:5185`, `:5219`,
`:5649`) and `p_actionfunctions.cpp:1302`. **Gate your hand-reading code on it** — when it is
false you are not in VR and the Attack/Offhand values are degenerate.

### 1b. There are no hand or head velocity fields.

Nothing in the table above is a velocity. Any speed, swing, flick, draw or throw
must be computed by storing the previous tic's `OffhandPos` / `HmdPosition` and
differencing. Budget for that in every design, and see §3 for the rate you get.

---

## 2. The offhand weapon system (this is real and substantial)

- `PlayerInfo.OffhandWeapon` — `src/playsim/d_player.h:384`
- `PSP_OFFHANDWEAPON` psprite layer, used throughout `player.zs`
  (`player.zs:411`, `:563`, `:1947`, `:1953`, `:1998`, `:2095`)
- Weapon flag `bOffhandWeapon` routes a weapon to that hand — `player.zs:1977-1998`
- Weapon flag `bTwoHanded` forcibly clears the offhand weapon — `player.zs:1989-1992`
- Per-hand button checking: `CheckWeaponButtons(hand)` — `player.zs:588`, `:2031`
- Per-hand ready/fire/down state selection keyed on `hand` — `player.zs:398`, `:431`,
  `:466`, `:2092`, `:2429`
- Morph preserves it: `PremorphWeaponOffhand` — `d_player.h:410`

So: **two independently-driven weapons, one per hand, each with its own psprite layer and
its own state machine, is a shipped engine feature, not a hack.**

---

## 3. Update rate: hand pose is written per *frame*, read per *tic*

The hand and head poses are written from the **renderer**, not the playsim:

- `src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:442-445`
- `src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:600-603`
- `src/rendering/gl/stereo3d/gl_openvr.cpp:2549-2552`

Two consequences, and they pull in opposite directions:

1. **The value script reads is fresh** — it is the latest rendered frame's pose, not a
   35 Hz-stale one. Good for thresholds.
2. **Script still only samples at 35 Hz**, so any motion detail between tics is gone
   forever. A fast swing is 2-4 samples. Every gesture design in this survey must survive
   on that budget, and "trace the path between the last two samples" is the standard
   remedy — do not assume you can see the arc.

---

## 4. Haptics: present in C++, **not reachable from ZScript**

- `VRMode::Vibrate(duration, channel, intensity)` — `src/common/rendering/hwrenderer/data/hw_vrmodes.h:94`
- `VR_HapticEvent(event, position, intensity, angle, yHeight)` — `hw_vrmodes.h:97`
- Intensity cvars `ext_haptic_level_*` — `src/playsim/p_actionfunctions.cpp:92-101`
- **Zero hits for haptic/rumble/vibrate anywhere in `wadsrc/static/zscript`.**

Treat "fire a custom rumble from script" as **unproven** until someone cites a hook.
If your topic needs haptics, find the caller of `VR_HapticEvent` and report whether a mod
can reach it (a hardcoded damage/pickup/fire event you could provoke is a HACK, not a BUILDABLE).

---

## 5. VR cvars a mod can read or set

From `src/QzDoom/VrCommon.h:16-27`:
`vr_control_scheme`, `vr_move_use_offhand`, `vr_weaponRotate`, `vr_snapTurn`, `vr_ipd`,
`vr_weaponScale`, `vr_teleport`, `vr_switch_sticks`, `vr_secondary_button_mappings`,
`vr_two_handed_weapons`, `vr_crouch_use_button`.

`vr_two_handed_weapons` in particular is an engine-level toggle, not something a mod invents.

---

## 6. Standing rules for every report

- **Cite or say you couldn't.** `path:line` at the `qzd17.3` tag. "I could not find this"
  is an acceptable and useful answer; a confident guess is not.
- **Label every mechanic** EXISTS / BUILDABLE / HACK / WALL, and cost S / M / L.
- **A WALL must name the missing piece**, not just say it's impossible.
- **Do not cite REMA.** `E:\DOOMWork\UZDXREMA` is a different, heavily modified engine.
  Anything found only there is a WALL here.
