# 99 — Overlooked

Things found while reading the source that suggest a mechanic nobody asked for.
One entry each: the citation, then a sentence on what it could become.

This file grows as the topic reports come in. Entries marked **(mine)** were found during the
capability pass; the rest come from topic surveys and name their source report.

---

## 1. There is a persistent, separately-rendered psprite band above `PSP_TARGETCENTER` **(mine)**

`PSP_TARGETCENTER = INT_MAX - 2` (`src/playsim/p_pspr.h:59`) is not just the crosshair's layer —
it is the boundary of a **second render pass with different clearing rules**:

- The weapon pass iterates only layers *below* it — `src/rendering/hwrenderer/scene/hw_weapon.cpp:830`, `:915`
- Layers at or above it are drawn in their own pass — `hw_weapon.cpp:1068`
- `A_ClearOverlays(start, stop, safety)` with `safety` set **breaks out before touching them** —
  `src/playsim/p_pspr.cpp:1273-1274` — and also skips `PSP_STRIFEHANDS`, `PSP_WEAPON`,
  `PSP_OFFHANDWEAPON` and `PSP_FLASH` (`p_pspr.cpp:1275-1276`)
- They are given `PSPF_CVARFAST|PSPF_POWDOUBLE` automatically — `p_pspr.cpp:360-362`

**What it could become:** a persistent **sprite** overlay that survives `A_ClearOverlays(…, safety)`
— and that is now the band's *only* real edge, see the corrections below.

> **Correction (11_HUD_MODELS, verified here).** This entry originally said the band was the way
> to get "a hand model that survives weapon switches". Both halves of that were wrong.
> **No layer at or above `PSP_TARGETCENTER` can draw a model at all** —
> `PrepareTargeterSprites` hardcodes `hudsprite.mframe = nullptr` (`hw_weapon.cpp:1058`) and the
> model call site is gated on `mframe` (`:101-108`). The band is sprites only.
> **And the band is not what grants survival.** The cull at `player.zs:559-563` gates its
> "weapon not currently selected" clause on `pspr.Caller is "Weapon"`, so a layer owned by a
> plain `Inventory` item persists across weapon switches **at any layer id**. Persistence comes
> from the **caller**, not the band.
> The wrist-panel idea still works — just below the band, owned by an item. See
> `surveys/11_HUD_MODELS.md` for the recipe.

**Correction (03_MECHS).** My first draft of this entry called it a cockpit frame. It is not —
**the band is not view-locked.** `PrepareTargeterSprites` pushes into the same `hudsprites`
array (`hw_weapon.cpp:1072`, `:1076`) and gets the same hand matrix as any other psprite,
landing on the **main hand**. In VR every non-model psprite layer is transformed by a hand
matrix, with the hand chosen by the layer's caller (`hw_weapon.cpp:252-258`; HUD models
likewise at `src/r_data/models.cpp:219-226`), and `psp.x` / `psp.y` become **rotations**, not
translations (`hw_weapon.cpp:108`). A head-locked cockpit frame is a **WALL**; see
`03_MECHS.md`, which builds the cockpit from spawned actors instead.

**The traps — both corrected by 11_HUD_MODELS, verified here:**

1. A layer created at or above `PSP_TARGETCENTER` defaults its `Caller` to the player's
   `PowerTargeter` item (`p_pspr.cpp:310-318`). Without that powerup the layer is **not
   created at all** — `GetPSprite` returns `nullptr` (`p_pspr.cpp:334`). It is not "culled
   later", as this entry first said; nothing ever exists. Pass `newcaller` explicitly.
2. The targeter render pass starts at `FindPSprite(PSP_TARGETCENTER)` (`hw_weapon.cpp:1068`),
   which matches that id **exactly** (`p_pspr.cpp:241-254`). So a layer at `PSP_TARGETCENTER + 1`
   or `+ 2` draws **only while something else holds `PSP_TARGETCENTER` itself** — in practice
   the crosshair, which that same loop then skips (`:1070`). Depending on an unrelated layer's
   existence is not a foundation; put the panel below the band and own it with an item.

---

## 1a. `A_QuakeEx` is a back door to per-hand haptics — and the camera shake is off by default **(03_MECHS, verified here)**

This is the most valuable thing in this file. Everywhere else the survey says haptics are
unreachable from ZScript; this is the exception, and it is controllable.

`R_SetViewAngle`'s quake handling routes earthquake intensity straight into the controllers:

```
src/rendering/r_utility.cpp:1085   if (vr_quake_haptic_level > 0.0) {
src/rendering/r_utility.cpp:1086       left  = QuakePower(vr_quake_haptic_level, jiggers.Intensity.X, jiggers.Offset.X);
src/rendering/r_utility.cpp:1087       right = QuakePower(vr_quake_haptic_level, jiggers.Intensity.Y, jiggers.Offset.Y);
src/rendering/r_utility.cpp:1090       vrmode->Vibrate(10, 0, (float)left);    // left hand
src/rendering/r_utility.cpp:1091       vrmode->Vibrate(10, 1, (float)right);   // right hand
```

And the camera motion that would normally come with it is **off by default in VR**:
`r_quakeintensity` is `0.0f` with the literal comment "Defaulted to 0 for VR"
(`r_utility.cpp:116`), read as `quakeFactor` at `:1066`.

**What it could become:** `A_QuakeEx` is an ordinary ZScript action function. Put your intensity
in **X for the left hand and Y for the right**, set a short duration and zero damage radius, and
you get **independent per-hand rumble with no view shake** — a punch landing in the hand that
threw it, a parry thump on the shield arm, bow-draw tension, engine rev, a mech's alternating
footfalls (left stride in X, right in Y).

### Use this recipe, not a bare call — the defaults will sabotage you

`A_QuakeEx`'s signature (`wadsrc/static/zscript/actors/actor.zs:1284`) ends with
`… double damageMultiplier = 1, double thrustMultiplier = 0.5, int damage = 0`.

**`thrustMultiplier` defaults to 0.5, and the thrust adds randomised velocity to the player:**

```
src/playsim/a_quake.cpp:191   if (dist < m_DamageRadius && victim->Z() <= victim->floorz)
src/playsim/a_quake.cpp:205       if (!(victim->flags7 & MF7_DONTTHRUST) && m_ThrustMultiplier > 0)
src/playsim/a_quake.cpp:207           DAngle an = victim->Angles.Yaw + DAngle::fromDeg(pr_quake());
src/playsim/a_quake.cpp:208-209       victim->Vel.X += m_Intensity.X * an.Cos() * m_ThrustMultiplier * thrustfalloff;
```

and the same block can damage the player (`:193-203`).

Both are gated on `dist < m_DamageRadius`, so **pass `damrad = 0`** and the whole block is
skipped. Belt and braces: pass `thrustMultiplier = 0` too.

Without that, your haptic confirmation of a dash **shoves the player in a random direction** —
the feedback perturbs the very motion it is confirming. (Found by 09_MOVEMENT, verified here.)

Also:
- **Never pass `QF_GROUNDONLY`** (`constants.zs:675`) — the quake is dropped when the victim is
  off the floor (`a_quake.cpp:354`), which would mute exactly the airborne mechanics (air dash,
  double jump, wall run) that most need the feedback.
- **`QuakePower` returns a random value in `[-intensity, +intensity]`**
  (`r_utility.cpp:911-921`). This is **noise** — good for a thud, a buzz or a rumble, useless
  for a crisp click.

**Other caveats, state them when you use it:** `vr_quake_haptic_level` must be > 0 (a user cvar,
so detect and degrade); `Vibrate` hard-codes a 10 ms pulse at `:1090-1091`, so sustained or
shaped rumble means re-firing each tic; a user who has raised `r_quakeintensity` **will** get
camera shake with it; and this is the OpenVR path (`OpenVRHaptics::Vibrate`,
`gl_openvr.cpp:517-528`), so it inherits the backend question in `00_CAPABILITIES.md` §0.

### The other route: `LineAttack`

`P_LineAttack` buzzes the firing hand whenever a player deals damage, and it is **not** gated on
`LAF_NOINTERACT` (`src/playsim/p_map.cpp:4764-4769`, channel flipped by `LAF_ISOFFHAND` at
`:4767`). So `LineAttack(angle, 0.1, pitch, 1, "None", "NullPuff", LAF_NOINTERACT|LAF_ISOFFHAND)`
gives a clean 150 ms / 0.8 pulse on the chosen hand with no world effect, and it respects the
user's handedness via `vr_control_scheme`. Its duration and intensity are **literals**, so every
pulse feels identical — a light tap and a heavy block are indistinguishable.

**Pick by what you need:** `LineAttack` for a discrete, correctly-handed thump; `A_QuakeEx` when
you need a shaped envelope. (`LineAttack` route found by 07_SHIELDS_PARRY.)

Label it **HACK / S** — it abuses the earthquake system — but it is a HACK that works today and
removes the single biggest feel limitation in this survey.

---

## 2. Psprite layer ids are arbitrary integers, and the gaps are enormous **(mine)**

`PSP_WEAPON = 1`, `PSP_FLASH = 1000`, `PSP_OFFHANDWEAPON = 1000000`
(`wadsrc/static/zscript/constants.zs:798-801`).

The named layers are conventions, not a fixed set. `GetPSprite(id)` / `SetPsprite(id, state)`
take any int. There are 998 free indices between the weapon and its flash, and roughly a million
between the flash and the offhand weapon.

**What it could become:** per-hand stacks of independently animated, independently positioned
overlay layers — a held item, a forearm panel, a glove, a muzzle device, a reload prop — all
without fighting the weapon's own two layers for space.

---

## 3. Four completely free input bits **(mine)**

`BT_USER1` … `BT_USER4` — `wadsrc/static/zscript/constants.zs:859-862`, readable via
`player.cmd.buttons` (`player.zs:3048`) with `oldbuttons` (`player.zs:3064`) for edge detection.

**What it could become:** the cheap answer to half the gesture-recognition problems in this
survey. A pose check gated by a held button is far more reliable at 35 Hz than a pure motion
gesture, and it costs nothing. Before designing a gesture, ask whether a button plus a pose
would do.

---

## 4. `BT_SHOWSCORES` was sacrificed, and the engine says so **(mine)**

`BT_SHOWSCORES = 0,  // replaced with BT_MAINHANDRELOAD` — `constants.zs:857`,
with `BT_MAINHANDRELOAD = 1<<20` at `:869`.

**What it means:** the scores button is gone — any mod binding it is binding nothing. Also a
useful signal about this fork's priorities: it spent a scarce input bit on a per-hand reload,
which tells you the two-handed weapon layer is meant to be used.

---

## 5. Offhand projectile firing already resolves direction correctly **(mine)**

`OffhandDir` is not a loose utility — it is wired into the standard attack functions:
`wadsrc/static/zscript/actors/actions.zs:87-89`, `attacks.zs:476-478`, `attacks.zs:596-597`,
and even a Heretic weapon at `heretic/weaponphoenix.zs:185`. All three basis vectors
(forward, side, up) are derived from it.

**What it could become:** it means `A_FireProjectile` and friends already fire *from the offhand,
in the offhand's direction, with correct offsets* when the weapon is an offhand weapon. Anything
thrown, fired or launched from the free hand is far closer to working out of the box than the
rest of this survey would lead you to expect.

---

## 6. `timefreezer` is a bitmask, not a flag **(mine)**

`native int timefreezer` — `wadsrc/static/zscript/actors/player/player.zs:3079`, with
`PowerTimeFreezer` setting and clearing *bits* per player at
`wadsrc/static/zscript/actors/inventory/powerups.zs:1529`, `:1537`, `:1604`.

**What it could become:** the engine already models "frozen for some participants and not
others" at the bit level. That is the natural first place to look for selective or partial
time effects rather than a global stop. See `10_TIME.md`.

---

## 7. `LineTrace` reports the texture it hit **(mine)**

`FLineTraceData.HitTexture` — `wadsrc/static/zscript/actors/actor.zs:53`, alongside `HitLine`,
`LineSide` and `LinePart` (`:50`, `:58`, `:59`).

**What it could become:** material-aware reactions with no map authoring at all — a whip that
cracks differently on metal, footsteps that match the floor, sparks only on stone, a hookshot
that refuses to bite on glass. The texture name is the material database you already have.

And `HitLine` + `LineSide` is how you recover a **wall normal** at runtime, which is the
foundation under wall-running, rope anchoring and shield alignment. It is the most
load-bearing field in the whole trace struct.

---

## 8. Render-to-texture is available to mods **(mine)**

`SetCameraToTexture(viewpoint, texture, fov)` — `wadsrc/static/zscript/doombase.zs:30`, with
`SetCameraTextureAspectRatio(texture, aspectScale, useTextureRatio)` at `:31`.

**What it could become:** a working mirror (useful well beyond vanity — it is how a VR player
sees their own body), a vehicle rear-view mirror, a mech cockpit display, a security monitor,
a scope picture-in-picture.
