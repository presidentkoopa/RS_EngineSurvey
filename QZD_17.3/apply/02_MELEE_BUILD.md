# 02 — MELEE-ONLY BUILD: developing the foundation that is already there

**What this is.** `01_MELEE.md` designed melee against this engine from scratch. This file is the
*apply* pass: it goes and looks at what the owner actually built, and says what is left.

**Citation roots.**
- **MOD** — a fresh clone of `github.com/presidentkoopa/RSQ_UBQSeries`, **HEAD `6a5c98c`,
  2026-10-30**. Paths relative to its root: `Score/…`, `GunBonsaiOffhandVR/…`, `DistanceGrab/…`,
  `Wheel/…`. **Every mod line number below was read in this copy**, not in the `src/*.zip`
  snapshot of 10-28 (which is two days stale, lacks `Wheel` entirely, and lacks the two newest
  commits — see §0.1).
- **ENGINE** — `qzd173` (tag `qzd17.3`), paths repo-relative: `src/…`,
  `wadsrc/static/zscript/…`. Unchanged.

**Owner's words, 10-27:** *"melee is an excellent example, i had laid the foundation for what could
be a melee only build between kicks, jump kicks, punches, but it never developed."*

---

## 0. THE RECOMMENDATION, UP FRONT

**Build `RSQ_Brawl`: one pk3 that is a *kick* detector and a *guard* detector, and nothing else.**

The brief I was handed says the missing layer is detection. **That is wrong, and the correction is
the finding of this report.** A complete two-hand VR punch detector already exists, shipped, in
`GunBonsaiOffhandVR/rsq/zscript/gloves.zs` — velocity by differencing, arming, approach
accumulation, jab/hook/uppercut classification, head-vs-body, counter detection, swept contact
test, delivery, knockback and per-hand haptics. So does the progression (nine upgrade paths), the
stagger/winded/knockdown model (`RSQ_Dazed`), the scoring (27 melee bonuses — counted, not
estimated), the combo ledger, the HUD, **the melee-only mode switch itself** (`rsq_slappers`), **the
clinch** (`DistanceGrab`'s Headlock, written 10-29) and **the weapon picker** (`Wheel`).

What is genuinely absent is a short, nameable list:

| | Missing piece | Label | Cost |
|---|---|---|---|
| 1 | **Kicks, jump kicks, stomps** — named first by the owner, implemented nowhere. | BUILDABLE (jump kick, stomp), HACK (standing kick), WALL (kick *type*) | M |
| 2 | **Guard / block / slip** — no pose test anywhere in the suite. | BUILDABLE | M |
| 3 | **Wall occlusion on the punch contact test** — punches land through walls today. | BUILDABLE | S |
| 4 | **Stance** (orthodox/southpaw) — nothing reads lead-hand depth. | BUILDABLE | S |
| 5 | **Opponents worth boxing** — Doom's grid-chase AI is not a boxing opponent. | BUILDABLE | L |

**Do not build:** a punch detector, a grab/clinch, a scorer, a progression tree, a stagger system,
a combo window, a weapon picker, a HUD, or a haptics API. All nine exist and are correct. A
melee-only build that reinvents any of them is a worse build.

**The order to build in** (each step ships standalone and is provable before the next):

1. **Wall occlusion** on `RSQ_GloveHandler.Contact` — one `LineTrace`, half an hour, fixes a live
   bug (§2.3).
2. **The vertical head channel.** `RSQ_MeleeHandler` already samples `HmdPosition` every tic and
   already stores `headY[]` — and then **never reads it** (written `up_melee.zs:67`, declared
   `:11`, and those are its only two occurrences in the entire mod). Publish head *height*
   velocity the same way it publishes `headMps`. This one field is the input for kicks, jump kicks,
   stomps, knees and ducks. (§2.1)
3. **Jump kick and stomp.** The easiest of the three kicks, not the hardest (§3.2, §3.3).
4. **Standing kick** as a lunge + button, labelled honestly (§3.4).
5. **Guard and slip** (§2.2).
6. **Opponents** (§2.5) — the only L, and the only thing that turns a mechanic set into a *build*.

Steps 1–4 feed the existing seam with **zero changes to Score** (§4). That is the whole argument
for this shape.

### 0.1 What the stale snapshot cost, and what the fresh clone added

Diffed every file I cite (`--strip-trailing-cr`; the zips are LF, the clone CRLF, which masks a
plain `diff` entirely). **Only two files changed, and both changes are on-topic:**

- **`Score/zscript/rsscore/combo.zs` +4 lines.** A new tool id and a new combo:
  ```
  combo.zs:34   //   grab.hold      Distance Grab's off hand holding it (the walk-up grip on a living monster)
  combo.zs:73-75  // the owner, 10-29: "allow offhand to grab enemy up to a point so mainhand can punch them,
                  // have that be a score combo"
  combo.zs:75   reg.OwnCombo("Headlock",  "grab.hold > hands",  0.5);
  ```
  **The owner asked for a melee mechanic the day before yesterday, and it is built** (§1.3a). This
  is the single most important thing the stale snapshot hid.
- **`GunBonsaiOffhandVR/rsq/zscript/core.zs` +13 lines** (`:243-255`): a `weapon.level` /
  `weapon.xp` service answer *"RSQ Wheel's data card"* — the Wheel reading Gun Bonsai levels.
  **Everything I cite in `core.zs` above line 242 is unmoved; everything below shifts by +13**, and
  all of it is re-read below (`RSQ_Buzz` is at `:315`, not `:302`; `RSQ_Dazed` at `:357`, not
  `:344`).

`melee.zs` (1185), `physical.zs` (692), `handler.zs` (1070), `gloves.zs` (362), `up_gloves.zs`
(141), `up_melee.zs` (266), `up_suite.zs` (411), `slappers.zs` (243) and `stylexp.zs` (177) are
**byte-identical** between the two copies, so the coordinator's line counts were right and those
citations needed only re-anchoring, which they got.

**Two conclusions in my first draft rested on the snapshot and are now corrected:** the clinch is
not missing (§1.3a), and `near.zs`'s claim to pre-empt the engine's two-hand steadying does not
hold (§7.3).

### 0.2 The Wheel — does it matter? One line, as asked

**Yes, and favourably: the Wheel is already the melee-only build's weapon picker** — it selects
per hand (`Wheel/zscript/wheel/zscript.zs:974`, `:3385`), honours `bOffhandWeapon` and
`bNoHandSwitch` (`:3542`, `:6994`, `:2301`) which is exactly how the gloves are declared
(`gloves.zs:72`, `+WEAPON.NOHANDSWITCH`), already has an **empty-hand/fist** concept for when a
weapon is thrown back (`Wheel/README.md:23` — *"that hand takes its empty-hand weapon (slot 1, or
BD22 VR's off-hand fist)"*), and reads each weapon's Gun Bonsai level through the service hook added
to `core.zs:243-255` today. **So "fists versus a found pipe" is solved**: with `rsq_slappers` on,
guns are destroyed out of the inventory (`slappers.zs:185`, `w.Destroy()` after stashing the class
name at `:181`), so the ring shows only gloves and whatever the loaded game calls melee — which is
the correct behaviour with no change to either mod. **Nothing to build here; one line of options
text to point the player at it.**

---

## 1. INVENTORY OF THE FOUNDATION — the true list

The owner half-remembers what he built. He built more than he thinks in three places and less than
he thinks in one.

### 1.1 Punches — IMPLEMENTED HERE, fully

`RSQ_GloveHandler` (`gloves.zs:92`) is a complete detector. Not a stub, not a sketch.

| Piece | Where | Note |
|---|---|---|
| Two-hand pose read, gated correctly | `gloves.zs:131-137` `HandPos` | Gates on `OverrideAttackPosDir` **and** `player.PlayInVR` (`:135`), **and** rejects `AttackPos == OffhandPos` (`:136`) — the collapse signature from `hw_vrmodes.cpp:440`. Exactly the filter `01_MELEE` §3.2 demands, independently arrived at. |
| Velocity by differencing | `gloves.zs:167-195` `Read` | Hand measured against `pmo.Prev`, one tic, no averaging. Drops gaps (`:188`), snap turns (`:189`, `SNAP_DEG 15`/tic) and one-tic jumps (`:191`, `JUMP_UNITS 48`). |
| Arm → approach → swing state machine | `gloves.zs:196-248` `Swing` | Arms only after the hand slows below `min_speed * 0.6` and `rehit` tics pass; needs `ARM_M = 0.15 m` of travel toward the target at an average over `rsq_punch_min_speed`. |
| Pixelstretch normalisation | `gloves.zs:199`, `:204` (`v.z *= ps`) | The exact trap `01_MELEE` §0 says costs you the uppercut — already handled. |
| Swept contact test | `gloves.zs:250-273` `Contact` | Nearest approach of the glove's path *this tic* (`prevHand → hand`) to the body's axis; glove a 0.09 m sphere plus `rsq_punch_pad`. A real swept test, not a point test. |
| Classification jab/hook/uppercut | `gloves.zs:142-148` `Classify` | From travel against the line to the target, not from where the player looks. |
| Head vs body | `gloves.zs:302-303` | `upFrac = (at.z - victim.pos.z) / victim.height`; body below 0.6. |
| Counter (victim mid-attack) | `gloves.zs:304` → `RSQM.InAttack` (`core.zs:145`) | |
| One-Two chain | `gloves.zs:308-312` | Other hand within 17 tics keeps the chain; a whiff breaks it (`:274-278`). |
| Star Punch | `gloves.zs:318-319`, bank `:337-351` | Uppercut + trigger squeezed + a banked star. SuperPunchOut's own rule. |
| Delivery | `gloves.zs:326-333` | `SpawnBlood` (`:326`), then one `DamageMobj(g, pmo, dmg, 'Melee', DMG_THRUSTLESS)` (`:327`) with **the glove as inflictor** — this is the seam (§4). Mass-scaled knockback at `:333`. |
| Per-hand haptics | `gloves.zs:329` → `RSQ_Buzz` (`core.zs:315-346`) | §6. |
| Headless test seam | `gloves.zs:111-112` `simOn/simHand`, `:134`, public `Deliver` `:285` | A probe can land a punch with no controller. This is how you gate it (§8). |

**Verdict: punches are done.** The owner should not be told they are missing.

### 1.2 Kicks, jump kicks, stomps, knees — NAMED ONLY

Searched for `kick`, `jumpkick`, `knee`, `stomp` across every `.zs` in the clone. Every hit is a
**string read from somebody else's damage**, or a label:

- `Score/zscript/rsscore/melee.zs:118` — `if (dt == 'Kick' || dt == 'HeadKick' || dt == 'Stomp') return MK_BOOT;`
- `Score/zscript/rsscore/combo.zs:129` — the same three, returning the tool name `"kick"`.
- `Score/zscript/rsscore/melee.zs:560` — `RS_Bonus_Boot`, paying `0.2` for `MK_BOOT`.
- `Score/zscript/rsscore/physical.zs:123` — the same three in the physical family's melee test.
- `GunBonsaiOffhandVR/rsq/zscript/stylexp.zs:129` — `"kick"` routes to the melee XP pool.

Those three damage types are **Brutal Doom's boots** — `melee.zs:12` says so, and says they were
*"measured in brutal22test6's actors, not remembered"*.

**So: there is no kick in the suite. There is a kick-shaped hole that Brutal Doom currently fills,
and that Score, Combo and Style XP are all already wired to pay for.** That is a far better
position than "nothing exists", and it is exactly what §4 trades on.

Knee, stomp and jump kick: **no implementation and no reference at all** beyond the `'Stomp'`
damage type. `01_MELEE` §4.8 designed a knee and a stomp; neither was built.

### 1.3 Gloves, Slappers Only, progression — IMPLEMENTED HERE

| Piece | Where |
|---|---|
| `RSQ_Glove` / `RSQ_GloveOff` weapons (model hangers; draw `TNT1` at `:51`/`:60`, fire nothing) | `gloves.zs:17`, `:68` (`+WEAPON.OFFHANDWEAPON` at `:72`) |
| **Slappers Only — the melee-only mode switch** | `slappers.zs:31` `RSQ_SlappersHandler`, `:33` `On()` |
| Gun pickups replaced with nothing while on | `slappers.zs:16` `RSQ_NoPickup`, `:41` `CheckReplacement`, `:63` |
| Guns stashed by class name and destroyed, restored across levels | `slappers.zs:24` `RSQ_SlapperStash`, `:181`, `:185`, `:211` |
| What still counts as melee while on | `core.zs:30` `RSQM.IsSlapperClass` — asks the loaded game, no per-mod name list |
| Glove upgrades: Haymaker, One-Two, Star Punch, Counterpunch, Body Blow, Knockdown | `up_gloves.zs:16, 34, 53, 78, 104, 123` |
| Any-melee upgrades: Lunge (head speed), Executioner (staggered + low), Adrenaline (slow-mo pulse) | `up_melee.zs:194, 212, 244` |
| Player perks: Evasion (head ≥ 1.2 m/s → less damage), Southpaw | `up_suite.zs:374, 387` (base `RSQ_Perk` `:14`, `ModLoaded` `:17`) |
| Stagger / winded / knockdown / disarmed, per monster | `core.zs:357` `RSQ_Dazed` — `IsStaggered():375`, `IsDown():376`, `Stagger():379`, `Wind():388`, `KnockDown():395` |

**`rsq_slappers` is the melee-only build's chassis and it is already written.** A melee-only build
is not a new mod; it is this toggle plus kicks plus guard.

### 1.3a THE CLINCH — IMPLEMENTED HERE, 10-29. This was invisible in the snapshot.

`DistanceGrab/zscript/rsqdg/near.zs` (377 lines, `RSQDG_Near : EventHandler` at `:24`) is new and
its header is dated by the owner to 10-29. It does three things with a hand that is touching
something; the third is a melee mechanic:

```
near.zs:13-16
//   HEADLOCK     the OFF hand's grip on a living monster no heavier than rsqdg_collar_mass holds it
//                frozen in front of the hand for up to rsqdg_collar_secs, so the main hand can punch
//                it. RSQ_Score is told ("grab.hold"), and "Headlock" (grab.hold > hands) is a combo.
```

Verified in the implementation:

| Piece | Where |
|---|---|
| Off hand only, living, shootable, not a boss, mass ≤ `rsqdg_collar_mass` (default **400**) | `near.zs:125-126`, `:148` |
| Held for `rsqdg_collar_secs` (default **3.0 s**) | `near.zs:326` |
| Released on grip-up, death, or timeout | `near.zs:205`, `:334-335`, `:349` |
| **Score told every 2 s**, so the 3 s chain gap never breaks mid-hold | `near.zs:362-368` (`collarPing` + 70 tics) |
| Scored as the `Headlock` combo | `combo.zs:34`, `:75` |

Note the deliberate split from the *distance* grab, which **refuses** living monsters —
`policy.zs:367-370`, *"LIVING THINGS ARE NOT PROPS… if (a.Health > 0) return null;"*, corpses only
(`:313`, `:372-381`). The clinch is a separate, close-range, off-hand-only mechanic.

**This is `01_MELEE` §4.8's grab, built, and it is a melee-only build's best move: hold them and
punch them.** It also means §2's list is one item shorter than my first draft said.

### 1.4 Score's melee family — SCORES WHAT HAPPENED ELSEWHERE. Confirmed.

The structural claim in the brief is **correct**, and here is the proof.

Score is *told*. The whole event is built from four things the engine hands `WorldThingDied`:

```
Score/zscript/rsscore/handler.zs:264   override void WorldThingDied(WorldEvent e)
Score/zscript/rsscore/handler.zs:326   let ev = RS_ScoreEvent.Create(victim, playerKill ? killer : null,
                                                                     e.Inflictor, victim.DamageTypeReceived);
```

Classification is then a pure read of `damageType` and the inflictor's **class name**:

```
Score/zscript/rsscore/melee.zs:89-120   private static int KindOf(RS_ScoreEvent ev)
melee.zs:92      Name dt = ev.damageType;
melee.zs:94      if (ev.inflictor) inf = ev.inflictor.GetClassName();
melee.zs:95      if (inf ~== "SSQ_Disc")    return MK_DISC;     // ShieldSaw's thrown disc
melee.zs:96      if (inf ~== "UBQS_Thrown") return MK_THROWN;   // QuestSaber thrown
melee.zs:112     if (byPawn && ShieldInHand(ev.killer)) { … }   // ShieldInHand at :123
melee.zs:117     if (dt == 'Saw' || dt == 'Cut')  return MK_BLADE;   // Brutal Doom's axe/chainsaw, BD22 VR swings
melee.zs:118     if (dt == 'Kick' || dt == 'HeadKick' || dt == 'Stomp') return MK_BOOT;  // Brutal Doom's boots
melee.zs:119     if (dt == 'Melee' || dt == 'ExtremePunches')           return MK_FIST;
```

**And there is not one hand read in the whole mod.** Grepped `Score/` for `AttackPos`, `OffhandPos`,
`HmdPosition`, `AttackPitch`, `OffhandAngle`, `OverrideAttackPosDir`: **zero hits in any `.zs`
file.** The only `LineTrace` calls are line-of-sight checks for corpse visibility
(`event.zs:156-159`, `physical.zs:235-238`) — not hit resolution.

This is **deliberate and documented**, and the reason is the better half of the finding:

```
Score/zscript/rsscore/physical.zs:24-26
  // WHY NO POSE. The hand fields (AttackPos, OffhandPos, WF_TWOHANDSTABILIZED) exist on the
  // local machine only; scoring is play scope and must come out the same everywhere. A shot's
  // line, the head's facing and the buttons are all in the playsim already.
```

**That is correct architecture and it must not be moved.** It is also the load-bearing constraint on
the whole build: a detection layer reads render-rate, machine-local state, so it can never live in
Score. It must be a separate mod that *converts* local pose into a playsim fact — a `DamageMobj`
with a damage type and an inflictor — which Score then reads like any other. Which is exactly what
the gloves already do, and what `near.zs:21-23` does for the clinch (*"the hand poses exist only on
the camera player's machine, so that machine decides and sends one network event per press"*).

### 1.5 The honest summary table

| Mechanic | State | Citation |
|---|---|---|
| Punch detect / classify / resolve | **Implemented here** | `gloves.zs:92-364` |
| Jab / hook / uppercut | **Implemented here** | `gloves.zs:142-148` |
| Body vs head shot | **Implemented here** | `gloves.zs:302-303` |
| One-Two / flurry chain | **Implemented here** | `gloves.zs:308-312`; scored `melee.zs:752` |
| Counter (hit mid-wind-up) | **Implemented here** | `gloves.zs:304`, `core.zs:145`; upgrade `up_gloves.zs:78`; scored `melee.zs:725` |
| **Clinch / headlock** | **Implemented here (10-29)** | `near.zs:24-375`; scored `combo.zs:75` |
| Stagger / winded / knockdown | **Implemented here** | `core.zs:357-411`; upgrades `up_gloves.zs:104, 123` |
| Per-hand haptics | **Implemented here** | `core.zs:315-346` |
| Melee-only mode | **Implemented here** | `slappers.zs:31-243` |
| Weapon picking, per hand, fists included | **Implemented here** | `Wheel/zscript/wheel/zscript.zs:974, 3542`; `Wheel/README.md:23` |
| Glove progression (9 paths) | **Implemented here** | `up_gloves.zs`, `up_melee.zs` |
| Head-motion sampling | **Implemented here (horizontal only)** | `up_melee.zs:48-75` |
| Saber cut / throw / Force | **Elsewhere (QuestSaber), scored here** | `melee.zs:95-99` |
| Shield bash / grind / disc / block | **Elsewhere (ShieldSaw), scored here** | `melee.zs:95, 112-115`, `:130-141` |
| Axe / chainsaw / BD22 VR swings | **Elsewhere (Brutal Doom, BD22Q), scored here** | `melee.zs:117`; stacking `melee.zs:489` |
| Fatality | **Elsewhere (Brutal Doom), scored here** | `melee.zs:116` |
| **Kick / head kick / stomp** | **Elsewhere (Brutal Doom) — scored here, detected nowhere in the suite** | `melee.zs:118`, `:560` |
| **Jump kick** | **Only named** — not even a damage type | — |
| **Knee** | **Nowhere** | — |
| **Guard / block / slip / duck** | **Nowhere** | grep: no pose test in `GunBonsaiOffhandVR/` |
| **Stance (orthodox/southpaw)** | **Nowhere** (`up_suite.zs:387` "Southpaw" is an off-hand damage perk, not stance) | — |
| **Wall occlusion on punches** | **Nowhere — live bug** | `gloves.zs:250-273` |

---

## 2. WHAT A MELEE-ONLY BUILD NEEDS THAT DOES NOT EXIST, in order

### 2.1 The vertical head channel — **BUILDABLE / S. Build this first after §2.3.**

`RSQ_MeleeHandler.WorldTick` already does all the work:

```
up_melee.zs:11   double headX[MAXPLAYERS], headY[MAXPLAYERS], headZ[MAXPLAYERS];
up_melee.zs:56   Vector3 h = pmo.HmdPosition;
up_melee.zs:60-61  // horizontal only: index 1 is the height in the headset's own frame
                   double dx = h.x - headX[pn], dz = h.z - headZ[pn];
up_melee.zs:63     if (mps > 8.0) mps = 0;        // a recentre, not a lunge  -- the spike filter
up_melee.zs:66     headMps[pn] = max(mps, headMps[pn] * 0.8);            // decay memory
up_melee.zs:67     headX[pn] = h.x; headY[pn] = h.y; headZ[pn] = h.z;
```

**`headY[pn]` is written and never read** — `:11` and `:67` are its only two occurrences in the
entire mod, verified by grep across `GunBonsaiOffhandVR/`. The comment at `:60` explains why the
sampler is horizontal-only, and it is right about the frame: `HmdPosition` comes from
`VR_SetHMDPosition(hmdPose.m[0][3], m[1][3], m[2][3])`
(`src/rendering/gl/stereo3d/gl_openvr.cpp:2400`) → `src/QzDoom/qzdoom_common.cpp:93` →
`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:445`, so it is an OpenVR Y-up matrix column in
**raw play-space metres**, and `.y` is the head's height.

Two engine properties make this channel *better* than the hands:

- **It is never collapsed.** `VRMode::SetUp` resets `AttackPos`/`OffhandPos` to the body centre on
  every frame (`hw_vrmodes.cpp:440`) before the device code overwrites them — the false-positive
  source `01_MELEE` §0.2(2) warns about. `HmdPosition` is assigned at `hw_vrmodes.cpp:445` from
  `hmdPosition[]` **unconditionally; there is no collapse branch for it.**
- **It is frame-free.** Un-yaw-rotated play space, so a height delta is a height delta with no
  pixelstretch term and no sign ambiguity.

**Deliver:** add `headRiseMps[pn]` (signed, from `h.y`) beside `headMps[pn]`, with the same 8 m/s
spike filter and the same decay. One service accessor so the kick mod can ask without a hard link.
**This is the single highest-leverage line in the whole build.**

> **Correction to `01_MELEE`.** §4.4 says the duck is *"already free"* and §4.8 detects a knee from a
> `player.viewz` drop, citing `gl_openvr.cpp:2512-2517`. That code is inside
> `if (!vr_crouch_use_button)` (`gl_openvr.cpp:2512`), and **`vr_crouch_use_button` defaults to
> `true`** (`hw_vrmodes.cpp:122`). At default settings `crouchfactor` is left alone by the head, so
> `viewz` does **not** follow a physical duck and §4.8's knee test fires never.
> `00_CAPABILITIES` gotcha 4 has this right. **Use `HmdPosition.y`, not `viewz`, for every vertical
> body test.**

### 2.2 Guard, block, slip — **BUILDABLE / M**

Nothing in the suite tests a hand pose against the head. The closest shipped thing is
`RSQ_Upgrade_Evasion` (`up_suite.zs:374`): head moving ≥ 1.2 m/s → take 8 % less per level. **That is
already a slip**, in reduced form — timing without direction, reward without a window.

What to add, reusing the existing samplers:

- **Guard pose.** `(AttackPos - pmo.pos)` and `(OffhandPos - pmo.pos)` are exact and frame-free —
  both hand positions are literally derived as the pawn's position minus the hand's offset
  (`01_MELEE` §3.3, verified against `hw_vrmodes.cpp:440-444`). Require both hands inside the face
  box for 4 consecutive tics. Resolve by overriding `PlayerPawn.DamageMobj`.
- **Slip into a counter window.** `headMps` already exists and already has the decay memory that
  keeps the step alive on the tic the blow lands (`up_melee.zs:66`) — reuse it verbatim.
- **The counter it opens** should call `RSQ_Dazed.Stagger()` (`core.zs:379`) on the attacker, because
  that is the field `RSQ_Upgrade_Executioner` (`up_melee.zs:212`) already reads via
  `IsStaggered()` (`core.zs:375`). Guard then feeds the existing progression with no new state.

**Player knockback on a block** must be `Vel` plus `player.keepmomentum = true`, not
`A_ChangeVelocity` — `01_MELEE` §3.8(a)/(b), engine side
`wadsrc/static/zscript/actors/player/player.zs:1373-1383`, `keepmomentum` native at `player.zs:3108`.

### 2.3 Wall occlusion on the punch contact test — **BUILDABLE / S. Build this first.**

`RSQ_GloveHandler.Contact` (`gloves.zs:250-273`) finds candidates with
`BlockThingsIterator.CreateFromPos` (`:254`) and then does a nearest-approach test. **It never
checks geometry.** `01_MELEE` §3.4 names this exact failure: *"`BlockThingsIterator` is the right
shape (a fat fist is a sphere, not a ray) but it punches through walls."*

**So today a player in Slappers Only can punch a monster through a wall.** This is the only place
the gloves diverge from the survey's own recommendation, and it is a half-hour fix: after `best` is
chosen, one confirming `LineTrace` from `b` to `best`, reject on `TRACE_HitWall / HitFloor /
HitCeiling` closer than the body. Score needs no change; a punch that does not land is not scored.

### 2.4 Stance — **BUILDABLE / S**

`01_MELEE` §4.3's lead-hand-depth method, unchanged, plus the manual override it insists on. Its
only consumer is jab-vs-cross naming, so it is low value until opponents exist — **do it after
§2.5, or not at all.** `Classify` (`gloves.zs:142-148`) deliberately classifies from travel against
the target rather than from stance, and for a Doom map full of imps that is the better call.

### 2.5 Opponents — **BUILDABLE / L. The only large piece, and what "a build" means.**

Everything above is a *mechanic*. A melee-only build needs something to use them on. Doom's
grid-chase AI closes to `CheckMeleeRange` and swings; it does not telegraph, circle, or recover.
`01_MELEE` §4.13 has the full design (ring footwork replacing `A_Chase`, a `Telegraph` state block,
deterministic patterns, two weak windows per attack) and it is correct as written.

Three things the suite already gives it for free, which §4.13 did not know about:

- `RSQ_Dazed` (`core.zs:357`) is the per-monster condition token §4.13 would otherwise need —
  `Stagger` (`:379`), `Wind` (`:388`), `KnockDown` with a **count** (`:395`), `IsDown` (`:376`) for
  the stomp.
- `RSQM.InAttack` (`core.zs:145`) is already the telegraph read, and `RS_Melee.InAttack`
  (`melee.zs:243`) is the same test on the Score side. The counter window is wired end to end;
  only the *telegraph* is missing from the monsters.
- **The clinch** (`near.zs`, §1.3a) already gives a boxing opponent its most important property:
  it can be *held still for three seconds*. Design the roster around that.

`01_MELEE` §4.13's warning stands and should be repeated: **do not retrofit third-party monsters.**
`melee.zs:722`'s comment records this measured on the rig — *"Brutal Doom's monsters jump out of
'Missile' into their own attack states at once: an imp set to Missile on the rig was not in that
sequence a tic later"*. Build a small roster.

---

## 3. KICKS AND JUMP KICKS — the honest verdict

The owner named these first. **The counter-intuitive answer: the jump kick is the easy one and the
standing kick is the hard one.**

### 3.1 What the engine gives you, and what it does not

**Confirmed absent.** Grepped the engine tree for `footpos`, `foot_pos`, `legpos`, `anklepos`,
`hipPos`, `waistPos` and generic tracker poses: **zero hits anywhere in `src/`.** `01_MELEE` W9 says
the same for fingers. There is no lower-body pose on this engine, on either backend.

**Confirmed present, all script-readable and all in the playsim (so replicated and netplay-safe):**

| Input | ZScript | Engine |
|---|---|---|
| `player.jumpTics` | `wadsrc/static/zscript/actors/player/player.zs:3106` | set to `-1` on the jump tic (`player.zs:1553`), decremented each tic (`player.zs:1774`) — **a clean one-tic jump-start edge** |
| `player.onground` | `player.zs:3107` | recomputed every tic, `pos.z <= floorz + 2` (`player.zs:1369`) |
| `Actor.Vel.Z` | `wadsrc/static/zscript/actors/actor.zs:112` | the jump writes it at `player.zs:1550` |
| `player.crouchfactor` | `player.zs:3126` | head-driven **only** at `vr_crouch_use_button 0` (`gl_openvr.cpp:2512-2517`), and that cvar defaults `true` (`hw_vrmodes.cpp:122`) |
| `player.cmd.buttons` | `player.zs:3048` | `BT_USER1..4` at `wadsrc/static/zscript/constants.zs:859-862`; `BT_OFFHANDALTATTACK` at `:867` |

**Confirmed present, render-rate and machine-local:**

| Input | ZScript | Engine |
|---|---|---|
| `HmdPosition` — raw play-space metres, `.y` = height | `actor.zs:277` | `hw_vrmodes.cpp:445` ← `qzdoom_common.cpp:93` ← `gl_openvr.cpp:2400`. **Never collapsed to the body centre** (unlike the hands at `hw_vrmodes.cpp:440`). |

### 3.2 Jump kick — **BUILDABLE / M. The strongest kick in the build.**

Every input a jump kick needs is present, playsim, and already sampled or one line away:

| What it needs | What drives it |
|---|---|
| "The player jumped" | `player.jumpTics == -1` — the exact one-tic rising edge (`player.zs:1553`) |
| "…and is airborne" | `!player.onground` (`player.zs:1369`) |
| "…rising or falling" | `Vel.Z` sign |
| "…committed forward" | `headMps[pn]` — **already computed** (`up_melee.zs:61-66`) |
| "…and the body is extended" | `headRiseMps` from §2.1 (the natural rise of a jump kick) |
| "…into something" | a cone test from the pawn's lower half, resolved with the existing `Contact`-shaped nearest-approach plus the §2.3 `LineTrace` |

Resolve with `DamageMobj(kickInflictor, pmo, dmg, 'Kick')` — see §4. Pay it heavier than a punch,
with a longer `RSQ_Dazed.Stagger`, and let it be the knockdown opener.

**Why this is the easy one:** a jump is a discrete, unambiguous, replicated event with a one-tic
edge. There is no gesture to recognise. Nothing false-positives on it.

### 3.3 Stomp — **BUILDABLE / M**

Landing edge: `!onground` last tic, `onground` this tic, `Vel.Z` was negative. Over a monster with
`RSQ_Dazed.IsDown()` (`core.zs:376`) — the downed state **already exists** and
`RSQ_Upgrade_Knockdown` (`up_gloves.zs:123`) already puts monsters there with a count. Damage type
`'Stomp'`. Score pays `RS_Bonus_Boot` (`melee.zs:560`) with no change.

This is the cleanest mechanic in the plan: both ends already exist and only the middle is missing.

### 3.4 Standing kick — **HACK / M. Label it honestly.**

There is no foot. A standing kick can be driven by:

- a **body lunge**: `headMps` above a threshold with a forward component — the player genuinely
  steps or drives in; plus
- a **button**: `BT_USER1` or `BT_OFFHANDALTATTACK`, which `01_MELEE` §1.5 correctly argues beats
  inventing a gesture; plus
- a **low target cone** from the pawn's feet, so it reads as a kick and not a shove.

That is a *lunge-kick with a commit button*. It will read acceptably — the body motion is real and
the player's intent is explicit — but it is **not** a tracked kick, and nothing can tell a front
kick from a side kick from a roundhouse.

**Do not pretend otherwise, and do not let the kick's visual promise more than the input delivers.**
A kick that plays a roundhouse animation off a forward lunge will feel broken in a way the player
cannot name.

**The alternative, which is worse:** head *height* drop as the trigger (the natural counter-lean of
a kick). It works, but it collides with duck, with the knee, and with the player simply looking
down, and `01_MELEE` §4.12's rule applies — *"if you need a sixth clause, it is not a gesture, it is
a wish."* Use the button.

### 3.5 THE WALL, named

> **WALL — kick *type*, kick *path* and *which foot*.** There is no lower-body pose on this engine.
> Front/side/roundhouse/axe cannot be distinguished; a kick's arc cannot be traced the way a fist's
> is; left and right foot are indistinguishable. No ZScript arrangement recovers this.
>
> **The exact missing piece:** a readonly per-tracker pose pair on `Actor`, filled from the backend's
> generic-tracker poses, alongside the four hand globals written at
> `src/rendering/gl/stereo3d/gl_openvr.cpp:1331-1343, 1733-1770` — e.g. `Actor.TrackerPos(int slot)`
> / `Actor.TrackerAngles(int slot)`, named for what it does (any extra tracked point) rather than for
> feet, so the next caller gets it too.
>
> **This is an engine change and is out of scope for this build.** It is named here so the owner can
> decide separately. The hardware side already exists in his setup — five SlimeVR trackers including
> both feet — so this is a *binding and plumbing* job, not a feature invention, which puts it in the
> same class as `01_MELEE` W2.

**What a melee-only build should ship instead:** jump kick and stomp as first-class, tracked,
unambiguous moves; the standing kick as an explicit committed lunge. Three kicks that each do one
honest thing beat four that guess.

---

## 4. THE SEAM — reuse, do not duplicate

**Score's melee family does not need one line changed to score every mechanic in this plan.** This
is the best news in the report.

### 4.1 What Score accepts

The entire contract is four values:

```
Score/zscript/rsscore/handler.zs:326
  RS_ScoreEvent.Create(victim, killer, e.Inflictor, victim.DamageTypeReceived)
```

A detection layer's whole job is therefore: **turn a local hand/head pose into one `DamageMobj` call
with the right damage type and the right inflictor.** Nothing else crosses the boundary. No service
call, no hard link, no shared header.

### 4.2 What a detection layer reports, and what Score already pays for it

| Detector reports | `DamageMobj(…, damageType)` | Score reads it as | Bonus, unchanged |
|---|---|---|---|
| Punch landed | `'Melee'`, inflictor `RSQ_Glove` | `MK_FIST` (`melee.zs:119`); `IsGlove` (`melee.zs:233`) | Counter `melee.zs:725`, Knockdown `:738`, Flurry `:752`, Combo `:384` |
| **Kick landed** | `'Kick'` | `MK_BOOT` (`melee.zs:118`) | **`RS_Bonus_Boot` (`melee.zs:560`) — already written** |
| **Jump kick landed** | `'Kick'` (or `'HeadKick'` on a high hit) | `MK_BOOT` | same |
| **Stomp landed** | `'Stomp'` | `MK_BOOT` | same |
| Knocked it airborne, then killed it | any melee type, plus the launch touch | `RS_Melee.Airborne` (`melee.zs:180`) + `RS_ScoreShots.Launched` (`:1107`) | **Juggle (`melee.zs:541`) — its comment at `:537-538` already says "your Force, a kick, a punch or a bash"** |
| Knocked it into a crusher/pit | the launch touch | `ev.ringOut`, via `LaunchedBy` (`melee.zs:1116`) | **Ring Out (`melee.zs:632`) — same comment** |
| Clinched it, then punched it | `'Melee'` after `grab.hold` | `combo.zs:34` tool id | **Headlock combo (`combo.zs:75`) — already written** |
| Mixed kicks and punches in one chain | — | `RS_Combo.Classify` returns `"kick"` / `"punch"` (`combo.zs:129-130`) | **Combo ledger, already keyed on exactly these strings** |
| Style XP for a kick | — | `stylexp.zs:129` routes `"kick"` to the melee pool | already written |

**Read the Juggle and Ring Out comments again** (`melee.zs:537-539`, `:628-631`): both say *"your
Force, a kick, a punch or a bash"*. The owner wrote the scoring for kicks he had not yet built. **That
is the foundation he remembers laying**, and the seam is the half of it that got finished.

### 4.3 The one thing the detection layer must also do

`RS_ScoreShots` needs to know a kick *touched* something that lived, for Juggle and Ring Out. It
already learns this for Force/kick/bash blows through its `lnV/lnBy/lnTic` arrays
(`melee.zs:811`, *"bodies a player's Force, kick or bash touched, and lived through"*), expired
against `LAUNCH_TICS` at `:945`, and read by `Launched` (`:1107`) and `LaunchedBy` (`:1116`). A kick
that deals `'Kick'` damage and does not kill passes through `WorldThingDamaged` like anything else.
**So this is already handled too** — verify it in the gate (§8); do not pre-emptively change it.

### 4.4 The rule this sets

> **The detection layer is machine-local and never scores. Score is playsim and never detects.** The
> only thing that crosses is a `DamageMobj`. This is the owner's series rule (each mod works alone,
> they talk through services, never hard links) applied to the one case where even a service would
> be too much coupling — because the engine's own damage path *is* the service.

---

## 5. THE GLOVES ARE THE PROGRESSION — do not invent one

A melee-only build has a progression problem: strip the guns and you strip Doom's reward curve. The
suite already solved it.

**Gun Bonsai levels weapons by use.** The gloves are weapons (`gloves.zs:17`), so they level. Nine
upgrade paths already read the punch the detector wrote onto the glove — possible precisely because
`Deliver` passes the glove as the inflictor (`gloves.zs:327`) and `ModifyDamageDealt` hands the
upgrade that glove (`up_gloves.zs:1-2` says exactly this).

| Path | What it rewards | Where |
|---|---|---|
| Haymaker | throwing it harder (`lastPower`) | `up_gloves.zs:16` |
| One-Two | alternating hands inside half a second (`lastChain`) | `:34` |
| Star Punch | banked stars spent on a triggered uppercut (`lastStar`) | `:53` |
| Counterpunch | hitting mid-wind-up (`lastCounter`) → `RSQ_Dazed.Stagger` | `:78` |
| Body Blow | hits below the chest (`lastBody`) → `RSQ_Dazed.Wind` | `:104` |
| Knockdown | enough damage in 2 s → `RSQ_Dazed.KnockDown(count)` | `:123` |
| Lunge | head speed when the blow lands | `up_melee.zs:194` |
| Executioner | a staggered monster with little left | `up_melee.zs:212` |
| Adrenaline | melee kills → a slow-motion pulse | `up_melee.zs:244` |

**That is a Punch-Out progression already: power, combo, star, counter, body, knockdown.** A
melee-only build's job is to **extend this list, not start a new one.**

**What kicks should add, and nothing more:**

- one or two kick paths on the glove record (a kick is thrown by the same body), reading new
  `RSQ_Glove.lastKick*` fields written by the kick detector the way `lastPower` is written;
- a guard path as an `RSQ_Perk` (`up_suite.zs:14`) — guard belongs to the player, not a weapon,
  which is the distinction `up_suite.zs:5-9` already draws;
- a clinch path, since the clinch now exists (§1.3a) and `up_suite.zs:59+` already shows the pattern
  for Distance Grab perks;
- nothing elemental, nothing that is rate of fire, no new resource — `up_suite.zs:11` already states
  this rule and it should be honoured.

**And `rsq_slappers` is the gate.** Offer the kick and guard paths while it is on. `RSQ_Perk.ModLoaded`
(`up_suite.zs:17`) is already the mechanism for conditionally offering a path.

---

## 6. HAPTICS — which route per blow, and why

Both routes verified in this engine tree at the cited lines.

**Route A — `A_QuakeEx`, shapeable, per-hand, chainable.**
```
src/rendering/r_utility.cpp:1085-1091
  if (vr_quake_haptic_level > 0.0) {
      double left  = QuakePower(vr_quake_haptic_level, jiggers.Intensity.X, jiggers.Offset.X);
      double right = QuakePower(vr_quake_haptic_level, jiggers.Intensity.Y, jiggers.Offset.Y);
      vrmode->Vibrate(10, 0, (float)left);    // left
      vrmode->Vibrate(10, 1, (float)right);   // right
  }
```
`vr_quake_haptic_level` defaults **0.8** (`hw_vrmodes.cpp:127`). 10 ms pulses self-expire, so they
chain into an envelope. **`damrad` must be 0** — the thrust block in `src/playsim/a_quake.cpp` is
gated on `dist < m_DamageRadius`, and without that your haptic feedback shoves the player sideways.

**Route B — `LineAttack`, fixed 150 ms / 0.8, correct handedness.**
```
src/playsim/p_map.cpp:4765-4769
  if (damage > 0) {
      long rightHanded = vr_control_scheme < 10;
      rightHanded = (flags & LAF_ISOFFHAND) ? 1 - rightHanded : rightHanded;
      vrmode->Vibrate(150, rightHanded ? 1 : 0, 0.8);
  }
```
Verified **not** gated on `LAF_NOINTERACT`: `nointeract` is computed at `p_map.cpp:4724`, well before
this block, and this block never reads it. A trace that does nothing in the world still buzzes.

**The channel lock is the deciding constraint:**
```
src/rendering/gl/stereo3d/gl_openvr.cpp:517-520
  void OpenVRHaptics::Vibrate(float duration, int channel, float intensity) {
      if (vibration_channel_duration[channel] > 0.0f) return;
```
**A 150 ms Route-B pulse silently swallows every Route-A pulse on that hand for 150 ms.** Never both
on one hand in one event.

### 6.1 The suite already chose Route A, correctly

```
GunBonsaiOffhandVR/rsq/zscript/core.zs:315-346   RSQ_Buzz
core.zs:321   Default { +NOINTERACTION +NOBLOCKMAP +NOSECTOR +NOGRAVITY +DONTSPLASH +NOTONAUTOMAP RenderStyle "None"; }
core.zs:327   if (!multiplayer) { let cs = CVar.FindCVar("vr_control_scheme"); lefty = cs && cs.GetInt() >= 10; }
core.zs:331   q.left = (hand == 1) != lefty;                      // handedness, by hand
core.zs:332   q.amp = clamp(k, 0.12, 1.0);
core.zs:333   q.total = clamp(tics, 1, 16);
core.zs:343   double a = amp * leftTics / double(total) / 3.2;     // the falling envelope
core.zs:345   A_QuakeEx(left ? a : 0.0, left ? 0.0 : a, 0, 2, 0, 8, "", QF_WAVE, 17.5, 17.5, 1);
```

`damrad = 0`. `tremrad = 8`, tight. `QF_WAVE`, which `01_MELEE` §2 recommends against `QuakePower`'s
randomness. Handedness read from `vr_control_scheme` and **skipped in multiplayer** (`:327`), which
is the netplay rule honoured. This is `01_MELEE` §3.7's envelope, independently built and arguably
better.

It is also an actor with `+NOINTERACTION` (`core.zs:321`), which `00_CAPABILITIES` gotcha 11 calls a
trap — correctly, *for an actor that must stay collidable*. This one must not, so it is the right
call here and the gotcha does not apply.

### 6.2 The per-blow table

| Blow | Route | Shape | Why |
|---|---|---|---|
| **Jab** | A, that hand | peak ~0.5, 3–4 tics, sharp decay | A jab is a *click*. Route B's fixed 150 ms at 0.8 is far too long and too strong — every jab would feel like a cross. `RSQ_Buzz.Start(pmo, h, 0.5 + 0.5*lastPower, 5)` (`gloves.zs:329`) is already this. |
| **Heavy cross / haymaker** | A, that hand | peak 1.0, 7–8 tics, slow swell | The envelope is the whole difference between a jab and a cross. Only Route A can shape it. The existing call already scales peak and length by `lastPower`/`lastStar`. |
| **Hook / uppercut** | A, that hand | as the cross, with the peak one tic later | Reads as the blow arriving at the end of an arc. |
| **Kick / jump kick** | A, **both hands at equal low amplitude** | peak ~0.6, 6 tics | No hand threw it. Putting a kick's buzz in one hand is a lie the player will feel. Both-hands-low reads as a body impact — the same trick the engine itself uses for `weaponStabilised` (`p_map.cpp:4771-4774`). |
| **Stomp** | A, both hands | peak 1.0, 8 tics, slow | The heaviest thing in the build; it should be the heaviest buzz. |
| **Clinch holding** | A, **off hand only**, sustained low | ~0.2, re-fired per tic while held | The clinch lasts up to 3 s (`near.zs:326`) and currently has no feel. A low sustained hum in the gripping hand is the cheapest, clearest "you have hold of it" signal available, and Route A is the only route that can sustain. |
| **Guard absorbing a hit** | A, **both hands** | peak ~0.6, 5–6 tics | Both fists are up; both should thump. `01_MELEE` §4.4's call, and correct. |
| **Guard break** | A, both hands | peak 1.0, 10 tics | Must be distinguishable from a soak by feel alone, because the player's eyes are on the opponent. |
| **Blocked / clang on armour** | A, that hand | peak 0.35, 2 tics, very sharp | A *tick*, not a thump. Deliberately below a landed hit. |
| **A thrown thing connecting (disc, saber)** | B | fixed | The hit already comes from a `LineAttack`-class path; Route B is free and correctly handed, and nothing about a returning disc needs an envelope. |

**And always degrade gracefully.** `vr_quake_haptic_level` is a user cvar. If it is 0 the player gets
no rumble at all and the build must not depend on it — sound plus `freezetics` must carry the blow.
`01_MELEE` §3.7 says this; `RSQ_Buzz`'s clamp (`core.zs:332`) does not read the cvar, so **check it
once and fall back** is a real small addition.

---

## 7. THE TWO-HAND TRAP — does "the fists are not weapons" hold?

**The trap, re-verified in this engine tree:**

```
src/rendering/gl/stereo3d/gl_openvr.cpp:1717-1727
  if (vr_two_handed_weapons && (grip pressed-state changed)) {
      if (grip now pressed) { if (distance < 0.50f) { weaponStabilised = true; } }
      else                  { weaponStabilised = false; }
  }
```
`vr_two_handed_weapons` defaults **true** (`hw_vrmodes.cpp:119`). The bit is re-asserted on every
`P_SetPsprite` (`src/playsim/p_pspr.cpp:512-515`), so script cannot durably clear it.

**A boxing guard is both fists at the face. That is unambiguously inside 0.50 m.** So the question is
real and not theoretical.

### 7.1 The reasoning in `01_MELEE` §0.2(4) is sound — but the gloves did not follow it

`01_MELEE`'s answer was *"the fists are not weapons"*: run everything from a carried `Inventory`
token so there is no offhand `Weapon` for the latch to switch off.

**The gloves are offhand weapons.** `RSQ_GloveOff : RSQ_Glove` has `+WEAPON.OFFHANDWEAPON`
(`gloves.zs:68`, flag at `:72`). So the literal design was not taken. I traced each of the four
consequences of stabilisation against the actual code:

| Stabilisation does | Engine | Does it hit the gloves? |
|---|---|---|
| Offhand weapon **fires nothing** | `player.zs:393-396` returns early for `hand == 1` | **No.** `Deliver` calls `victim.DamageMobj` directly (`gloves.zs:327`). The glove's `Fire` state is a one-tic no-op (`gloves.zs:60`) and nothing routes through it. |
| Offhand weapon **checks no buttons** | `p_pspr.cpp:756-758` returns early for `hand == 1` | **No.** The detector runs in `RSQ_GloveHandler.WorldTick` (`gloves.zs:149`), not in the weapon's button checks. Star Punch reads `pmo.player.cmd.buttons` straight from the tic command (`gloves.zs:318`), which `p_pspr` does not touch. |
| `OffhandWeapon` **pointer cleared** | only by `+Weapon.TwoHanded` (`player.zs:1989-1992`) | **No.** `RSQ_Glove` does not set that flag. `GloveIn` (`gloves.zs:122`) keeps resolving. |
| Offhand weapon's psprite **renders nothing** | `src/rendering/hwrenderer/scene/hw_weapon.cpp:832-835` and `:917-922` both `continue` past any psprite whose `GetCaller() == player->OffhandWeapon` | **YES.** And both skips sit immediately above a `FindModelFrame` call — they are the **model** loops, which is the glove's entire visual: `gloves.zs:12-15` says *"A glove draws nothing (TNT1)… the MODELDEF on these two classes is the whole of what you see."* |

### 7.2 The verdict

> **The reasoning holds for everything that matters, and fails for exactly one thing: the left
> glove's model disappears while the player holds the offhand grip with their hands together.**

The gloves took a different and arguably better route than `01_MELEE` proposed: not *no weapon*, but
an **inert weapon** — a weapon that exists only as a MODELDEF hanger and a Gun Bonsai XP record, with
every behaviour lifted into an `EventHandler`. That keeps the progression (which needs a weapon
identity) while taking the detection out of the latch's reach. **`01_MELEE` §0.2(4) should be amended
to record this as the preferred pattern**, because "no weapon at all" would have cost the gloves
their whole upgrade tree.

### 7.3 A CORRECTION — `near.zs` cannot pre-empt the latch, and may believe it does

`near.zs`'s header claims:

```
near.zs:2-4
// …and while it does them it comes first -- before the engine's two-hand steadying, before the
// distance reach -- so the grip press is EATEN (InputProcess returns true) whenever one of them
// takes it.
```

**Eating the grip in ZScript `InputProcess` cannot prevent `weaponStabilised`.** The two readers are
independent consumers of the same hardware state:

- The stabiliser reads the controller state **directly**:
  `pOffTrackedRemoteNew->ulButtonPressed & ButtonMaskFromId(k_EButton_Grip)`
  (`gl_openvr.cpp:1719`), inside the same per-frame function.
- ZScript's `InputProcess` sees the grip only as a *key event* synthesised separately by
  `Joy_GenerateButtonEvents` → `D_PostEvent` (`gl_openvr.cpp:1543`, `:1508`).

Returning `true` from `InputProcess` consumes the **event**; it does not alter `ulButtonPressed`. So
the header's ordering claim is true for *other ZScript consumers* (the rig, the distance reach) and
**false for the engine's stabiliser**.

Two mitigating facts, so this is not alarmist: `near.zs:17-19` reads the off-hand grip as
`KEY_PAD_RTHUMB` / `KEY_PAD_DPAD_UP` — the **thumbstick click** (`k_EButton_Axis0`,
`gl_openvr.cpp:1594`), not `k_EButton_Grip` — so on many bindings the clinch button and the
stabiliser button are not the same physical control at all; and the clinch holds the monster out in
front of the off hand while the main hand punches, a pose whose hands are typically **well outside**
0.50 m. **The clinch is therefore probably safe in practice. The guard is not**, and that is where
the attention belongs.

### 7.4 What the melee-only build must do

1. **Do not try to clear the bit** — it comes straight back (`p_pspr.cpp:512-515`).
2. **Do not rely on eating the key** — §7.3.
3. **Read it and show it.** `player.WeaponState & WF_TWOHANDSTABILIZED`
   (`wadsrc/static/zscript/constants.zs:1242`). When the guard detector (§2.2) sees a full guard and
   the bit is set, that is a *known, explainable* state, not a bug.
4. **Make the guard look deliberate.** A guard pose will trip this constantly. Either (a) draw the
   guard's visual on a psprite layer whose caller is **not** the offhand weapon — the skip tests the
   *caller*, not the layer id (`hw_weapon.cpp:832`), so a plain overlay layer survives, which is also
   why `07_SHIELDS_PARRY` prefers overlay-plus-world-actor; or (b) tell the player, as `01_MELEE`
   §0.2(4) prescribes for blades.
5. **Say it in the options.** `vr_two_handed_weapons 0` removes the trap entirely and costs a
   melee-only player *nothing* — there is no two-handed gun to stabilise in Slappers Only. **For a
   melee-only build this is the right default to recommend in the mod's own menu text**, and it is
   the one place where the build can make the trap simply not exist.

That last point is the cleanest answer available and it was not open to `01_MELEE`, which had to
assume guns.

---

## 8. THE SMALLEST BUILD THAT PROVES IT

**One pk3. One map. The player kicks something and it reacts, and the kick shows up on the score
HUD with no change to Score.**

### `RSQ_Brawl` — step 0, the probe

Because the punch detector already exists and is already proven, the prototype's job is **only the
new channel**. That makes it much smaller than `01_MELEE` §6's.

1. **A HUD readout** (`RenderOverlay`), printing per tic: `OverrideAttackPosDir`, `PlayInVR`,
   `HmdPosition` (all three components, raw), the derived `headMps` and the **new** `headRiseMps`,
   `player.onground`, `player.jumpTics`, `Vel.Z`, and `WF_TWOHANDSTABILIZED`. **This is the
   deliverable of step 0** — every threshold in §3 is tuned against these numbers and none of them
   can be guessed.
2. **One test monster:** a `Zombieman` subclass with `Speed 0` and a `Telegraph` block, so it stands
   still and can be countered.

### The ordered steps

| Step | What | Proves | Label / cost |
|---|---|---|---|
| **1** | Add the confirming `LineTrace` to `RSQ_GloveHandler.Contact` (§2.3). | Stand behind a wall and punch. Before: it lands. After: it does not. **Keep that comparison** — it is the argument. | BUILDABLE / S |
| **2** | Publish `headRiseMps` from the already-sampled `headY` (§2.1). | The readout moves correctly when the player stands, ducks and jumps — and reads **zero** when they only push the stick, which is what proves it is the body and not the input. | BUILDABLE / S |
| **3** | Jump kick: `jumpTics == -1` → airborne window → low cone → `DamageMobj('Kick')` (§3.2). | Jump at the zombieman and it takes a kick. Check the Score HUD: **`Boot` fires, with nothing added to Score.** That single observation validates the whole §4 seam. | BUILDABLE / M |
| **4** | Stomp on a downed monster (§3.3). | Punch one down with the existing `RSQ_Upgrade_Knockdown`, then land on it. `RSQ_Dazed.IsDown` (`core.zs:376`) gates it; `Boot` pays it. | BUILDABLE / M |
| **5** | Standing kick as lunge + `BT_USER1` (§3.4). | It fires when the player drives in and presses, and **never** when they merely walk. The false-positive rate is the only number that matters here. | HACK / M |
| **6** | Guard: both hands at the face for 4 tics, `DamageMobj` override, soak + both-hand rumble (§2.2). | Guard against the zombieman's swing and take less. Confirm `WF_TWOHANDSTABILIZED` lights on the HUD and **nothing breaks** (§7). | BUILDABLE / M |
| **7** | Kick and guard upgrade paths on the existing Gun Bonsai records, gated on `rsq_slappers` (§5). | The paths are *offered* in Slappers Only and not otherwise. | BUILDABLE / S |
| **8** | One opponent with a `Telegraph` block, ring footwork, a deterministic three-attack pattern (§2.5). | The player slips, counters, clinches and learns it. **This is the step that makes it a build and not a mechanic set.** | BUILDABLE / L |

### What the player does, and what they should feel

**Does:** turns on `rsq_slappers`. Guns vanish; gloves go on both hands; the Wheel now shows only
fists and whatever melee the game has (§0.2). Walks into a room with three opponents that circle
rather than charge. Jabs, crosses, hooks. Guards. Slips. Clinches one with the off hand and works
the body with the main. Jumps and kicks the one winding up. Stomps the one on the floor.

**Should feel:** that the jab is a *click* and the cross is a *swell* in the right hand (§6); that a
kick is a body blow felt in both hands, not a punch; that the clinch hums in the hand that holds it;
that a guard is worth holding; that an opponent telegraphs and that the telegraph is *audible*,
because in a headset audio is the primary tell (`01_MELEE` §4.13).

### Gate it headlessly before the headset

Per the house rule and `01_MELEE` §6: `main_boottest.ps1` / the `gate-a-doom-mod` recipe.
`OverrideAttackPosDir` is false headless, so the hand detectors correctly sit idle — **that is the
pass condition.** But note what makes this prototype *better* to gate than `01_MELEE`'s:

- **The gloves already have a headless seam.** `simOn[]`/`simHand[]` (`gloves.zs:111-112`, consumed
  at `:134`) and a public `Deliver` (`gloves.zs:285`) let a probe feed synthetic hand positions and
  land a real punch with no controller. `RSQ_MeleeHandler` has the same for the head
  (`simHead`/`simHeadMps`, `up_melee.zs:14-15`, consumed at `:54`). **Build the kick detector with
  the same seam from the start**, so steps 3–5 are provable in the log without a headset.
- **`jumpTics`, `onground` and `Vel.Z` are playsim and work headless.** So the jump-kick *trigger*
  can be gated for real: `-Fire`-style button injection plus a jump, then grep the log for the
  `'Kick'` blow landing and for `RS_Bonus_Boot` in Score's debug award path (`handler.zs:645`
  `DebugAward`). **A kick is the one mechanic in this plan that can be fully proven without VR**, and
  that should be exploited.

---

## 9. LABELS, in one table

| Piece | Label | Cost | Note |
|---|---|---|---|
| Punch detect / classify / resolve | **EXISTS** | — | `gloves.zs:92-364` |
| Clinch / headlock | **EXISTS** (10-29) | — | `near.zs:24-375` |
| Per-hand shaped haptics | **EXISTS** | — | `core.zs:315-346`; Route A, `damrad` 0 |
| Stagger / winded / knockdown / count | **EXISTS** | — | `core.zs:357-411` |
| 27 melee bonuses (counted), combo ledger, HUD | **EXISTS** | — | `melee.zs`, `combo.zs`, `hud.zs` |
| Melee-only mode switch | **EXISTS** | — | `slappers.zs:31-243` |
| Weapon picker, per hand, fists included | **EXISTS** | — | `Wheel/zscript/wheel/zscript.zs` |
| Glove + melee progression (9 paths) | **EXISTS** | — | `up_gloves.zs`, `up_melee.zs` |
| Head horizontal speed sampling | **EXISTS** | — | `up_melee.zs:48-75` |
| Kick/stomp scoring seam | **EXISTS** | — | `melee.zs:118, 560`; `combo.zs:129` |
| Wall occlusion on punches | **BUILDABLE** | S | the one live bug |
| Head vertical channel (`headRiseMps`) | **BUILDABLE** | S | `headY` already stored, never read |
| **Jump kick** | **BUILDABLE** | M | `jumpTics` edge is clean and replicated |
| **Stomp** | **BUILDABLE** | M | both ends already exist |
| Guard / block / soak | **BUILDABLE** | M | reuses `RSQ_Dazed.Stagger` |
| Slip timing + counter window | **BUILDABLE** | S | `headMps` + decay already there |
| Clinch haptic hum | **BUILDABLE** | S | the clinch currently has no feel |
| Kick / guard / clinch upgrade paths | **BUILDABLE** | S | extend, do not replace |
| **Standing kick** | **HACK** | M | lunge + button; label it honestly |
| Stance (orthodox/southpaw) | **BUILDABLE** | S | low value until opponents exist |
| Punch-Out opponents | **BUILDABLE** | L | own classes only |
| Slip *direction* (physical vs stick) | **HACK** | S | no play-space yaw — `01_MELEE` W3 |
| Guard's left-glove model under stabilisation | **HACK** | S | overlay layer, or `vr_two_handed_weapons 0` |
| **Kick type / path / which foot** | **WALL** | — | no lower-body pose in the tree; missing piece named in §3.5 |
| Suppressing `weaponStabilised` from script | **WALL** | — | `01_MELEE` W2b; and eating the key does not do it (§7.3) |
| Retrofitting third-party monsters with telegraphs | **WALL** (do not ship) | — | `01_MELEE` §4.13 |
| Direct haptic call with explicit duration | **WALL** (not blocking) | — | `01_MELEE` W2; Route A suffices |

---

## 10. WHERE THESE MODS ARE ALREADY RIGHT

Said plainly, because the temptation in an apply pass is to invent work.

1. **Score's melee scoring belongs in Score, and must not move.** The reason is written down at
   `physical.zs:24-26`: hand poses are machine-local, scoring is play scope and must agree on every
   machine. The separation is not tidiness, it is the netplay rule. `near.zs:21-23` applies the same
   rule to the clinch and gets it right too.
2. **The glove detector independently reinvented the survey's own filters, and got them right.** The
   collapse signature (`gloves.zs:136`), the pixelstretch normalisation (`:204`), the snap-turn and
   one-tic-jump rejects (`:189`, `:191`), the swept contact rather than a point test (`:250-273`).
   `01_MELEE` §3.2 and §3.5 are essentially a derivation of code already written.
3. **The inert-weapon pattern is better than `01_MELEE`'s "no weapon at all."** It keeps the
   progression and still escapes the stabilisation latch everywhere it matters (§7).
4. **The haptics envelope is already the right route, correctly handed and netplay-gated**
   (`core.zs:327-345`), with `damrad = 0` and a tight `tremrad`.
5. **The clinch refuses to be the distance grab.** `policy.zs:367-370` keeps living monsters out of
   the prop system (*"LIVING THINGS ARE NOT PROPS"*) and `near.zs` gives the clinch its own
   close-range, off-hand-only, mass-limited, time-limited rules. That is the right boundary and it
   was drawn deliberately.
6. **Everything is found by name, never linked.** `RS_Svc` (`svc.zs`), `RSQM.Has` (`core.zs`), `~==`
   for Name case-insensitivity (`melee.zs:16-20` explains exactly why, with the measured failure). An
   absent mod costs nothing. The series rule is honoured throughout.
7. **The kick scoring was written before the kick.** `melee.zs:537-538` and `:628-631` both say
   *"your Force, a kick, a punch or a bash"*. `combo.zs:129` and `stylexp.zs:129` already route
   `"kick"`. **This is the foundation the owner remembers laying, and it means a kick detector plugs
   into a socket that is already wired.** That is why the first thing to build is small.
8. **Both mods already carry headless test seams** (`gloves.zs:111-112`, `up_melee.zs:14-15`). A new
   detector should copy that pattern on day one, not retrofit it.

---

## 11. OPEN QUESTIONS FOR THE OWNER

1. **Which backend is the target binary built with?** `01_MELEE` Q1's question, still unanswered and
   still decisive. `USE_OPENXR` is defined only at `mobile/Android_src.mk:15`, and `USE_OPENVR` only
   under `option(ENABLE_OPENVR … NO)` (`src/CMakeLists.txt:418-420`). If the PC target is not built
   `ENABLE_OPENVR=YES`, every hand and head read in this plan is on the non-VR fallback. **The
   gloves' own gate already catches this safely** (`gloves.zs:135-136`), so the build degrades rather
   than breaks — but nothing will *work*.
2. **`vr_two_handed_weapons` — turn it off for the melee-only build?** It costs a melee-only player
   nothing and removes the §7 trap entirely. One line of options text; needs his call.
3. **`vr_crouch_use_button` — which way?** At the default (`true`), a physical duck is view-only, so
   duck/knee must be driven from `HmdPosition.y` (§2.1). At `0` the collision box follows the head for
   free, but unclamped (`01_MELEE` Q6). **This changes the design of duck, knee and stomp**, so decide
   before step 2.
4. **Which standing-kick trigger?** `BT_USER1`, `BT_OFFHANDALTATTACK`, or head-height drop (§3.4). I
   recommend `BT_USER1`: free, explicit, collides with nothing. His choice of binding.
5. **Should `near.zs`'s header comment be corrected?** Its claim to come "before the engine's
   two-hand steadying" is not achievable from `InputProcess` (§7.3). The code is probably fine in
   practice; the comment will mislead the next session.
6. **Is the lower-body tracker binding (§3.5) worth proposing separately?** The hardware is already
   on his body. It is the difference between three honest kicks and a real kickboxing build, and it is
   an engine change, so it is a separate conversation — not this one.
