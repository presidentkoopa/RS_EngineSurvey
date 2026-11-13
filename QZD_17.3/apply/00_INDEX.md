# apply/ — the teardown used on the mods

The audit in [`../surveys/`](../surveys/) is what the engine can do. This folder is that audit
applied to **RSQ_UBQSeries**, read at `github.com/presidentkoopa/RSQ_UBQSeries` HEAD `6a5c98c`
(2026-10-30).

Every report cites both sides: `path:line` in the engine at tag `qzd17.3`, and `path:line` in
the mod. Nothing here was written into any mod's tree — these are recommendations.

| Report | Covers |
|---|---|
| [01_GRAPPLE_TETHER.md](01_GRAPPLE_TETHER.md) | Hook / whip / lasso / rope as one tether; the sag diagnosed |
| [08_WHIP_DESIGN.md](08_WHIP_DESIGN.md) | The whip proper: wind-up, loop-arrival crack, three aim modes, four game presets |
| [02_MELEE_BUILD.md](02_MELEE_BUILD.md) | The melee-only build on the existing punch detector |
| [03_SABER_AND_SAW.md](03_SABER_AND_SAW.md) | QuestSaber and ShieldSaw refinement |
| [04_DISTANCEGRAB.md](04_DISTANCEGRAB.md) | Grabbing, carrying, throwing, doors, the Headlock |
| [05_SLOWMO.md](05_SLOWMO.md) | The clock checked against `10_TIME` |
| [06_RIG_AND_LASER.md](06_RIG_AND_LASER.md) | TacticalRig holsters and LaserSight |
| [07_WHEEL.md](07_WHEEL.md) | The weapon wheel |

---

## The finding that spans the series

**One engine behaviour has produced three separate shipped bugs, in three different mods, and
all three are live on SteamVR and dormant on Quest.**

The behaviour is the two-hand stabilisation latch, documented in
[`../surveys/00_CAPABILITIES.md`](../surveys/00_CAPABILITIES.md) gotcha 10a:
`vr_two_handed_weapons` defaults **true** (`hw_vrmodes.cpp:119`), and pressing the offhand
**grip** with the hands inside **0.50 m** sets `weaponStabilised` (`gl_openvr.cpp:1716-1727`).
The offhand weapon then fires nothing (`player.zs:393`, `:426`), checks no buttons
(`p_pspr.cpp:755-757`) and renders nothing (`hw_weapon.cpp:832-835`, `:917-920`), and the flag
is re-asserted on **every** state set (`p_pspr.cpp:514`), so script cannot clear it.

It writes only in `gl_openvr.cpp:1723` / `:1726`, so this is an **OpenVR/SteamVR** problem.
On Quest/OpenXR nothing sets it today — which is precisely why it has survived in three mods.

| Mod | What breaks | Report |
|---|---|---|
| **QuestSaber** | The 10-29 flip of `ubqs_world_saber` to false made the off blade psprites-only. Its beam is `A_Overlay` layer 2000052, and `A_Overlay` sets `Caller = stateowner` (`p_pspr.cpp:1241`) — the offhand weapon. **Hilt and blade both vanish under a two-hand grip while the blade keeps cutting.** | [03](03_SABER_AND_SAW.md) |
| **TacticalRig** | The 0.50 m guard was deleted on 10-29 (`STEADY_RANGE` now dead code). The two shipped hips are 0.36 m apart — inside the window — so an **ordinary cross-draw** latches the stabiliser and hands the player a weapon that renders, fires and answers nothing. | [06](06_RIG_AND_LASER.md) |
| **Wheel** | Delivers weapons into the off hand (`SetPsprite(PSP_OFFHANDWEAPON, …)`, `zscript.zs:9003`) with **zero** occurrences of `WF_TWOHANDSTABILIZED` in 16,687 lines. Brace a rifle, open the ring, pick: all four confirmations fire and the hand is empty-looking and inert. | [07](07_WHEEL.md) |
| **LaserSight** | The off-hand beam is a psprite with the gun as caller, so it is skipped — while the dot, glow, light and ring are world actors and **keep drawing from an invisible gun**. The service then reports the hand as live. | [06](06_RIG_AND_LASER.md) |

**The shared fix is one bitwise test** — `player.WeaponState & WF_TWOHANDSTABILIZED` — and then
each mod's own choice: refuse, reroute to the main hand, or force a world prop on. ShieldSaw
already does the last of these (`state.zs:860-862`) and is the in-family example.

**Do not try to clear the flag**, and do not restore a geometric guard — grip release clears the
latch with no distance test (`gl_openvr.cpp:1724-1726`), so a geometric guard releases before the
engine does.

### The second cross-cutting one: the vertical unit term

`vr_vunits_per_meter` is 34 horizontally, but a vertical distance **also** needs
`level.pixelstretch` (default 1.2), because the hand transform divides the up axis by it
(`gl_openvr.cpp:1282-1292`). Forget it and every height reads ~17% short. Across the series:

- **TacticalRig** — correct in all six conversions. The audit nominated this as its likely top
  finding and it did not land.
- **DistanceGrab** — correct, and documents it better than the survey did.
- **GunBonsai gloves** — correct (`gloves.zs:204`).
- **LaserSight** — reads `vr_vunits_per_meter` live but hardcodes 1.2.
- **Wheel** — wrong in two places: `wr_q_watch.zs:38-48` and `zscript.zs:8044-8056`, both on
  vertical-dominated gestures. The pickup-voxel sizing is clean (map units on both sides).

### Ranked, across the whole series

1. **Gate the off-hand paths on `WF_TWOHANDSTABILIZED`** — Wheel, TacticalRig, LaserSight,
   QuestSaber. Four mods, one test, all cost S.
2. **`player.keepmomentum = true`** — **zero** occurrences in all 152 ZScript files in the repo,
   so `player.zs:1373-1378` wipes `vel.XY` every grounded tic. Blocks the Grapple's swings and
   makes a Bionic Commando preset impossible. Cost S.
3. **Occlusion on the punch test** — `gloves.zs:250-254` uses `BlockThingsIterator` only, so
   punches land through walls today. Cost S.
4. **The Headlock's `bTHRUACTORS`** — `near.zs:324` makes the human shield transparent to every
   projectile (`p_map.cpp:1409-1410` ORs both actors' flags before any other gate). Cost S.
5. **The Wheel's absolute `rsm_off`** (`zscript.zs:4052-4053`) cancels slow motion the player was
   already holding. Ask the service first. Cost S.
