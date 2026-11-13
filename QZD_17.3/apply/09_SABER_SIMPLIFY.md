# 09 — APPLY: the saber has 137 options and 7 behaviours

**Read the first table and stop if you want. The rest is the evidence.**

Sources, all opened and read for this report:
mod `…/scratchpad/ubqs_gh/QuestSaber` (clone HEAD `6a5c98c`) — `CVARINFO` 177 lines, `MENUDEF`
434, `KEYCONF` 14, `MAPINFO` 4, `MODELDEF` 53,790, `zscript.txt` 6,514;
engine `…/scratchpad/qzd173`, tag `qzd17.3`; context `03_SABER_AND_SAW.md`,
`00_CAPABILITIES.md`, and the mod's own `docs/HANDOFF_polish.md`.
Nothing was written into the mod's tree. Every line below was read, not inferred.

---

## 0. THE NUMBER

| | count | where |
|---|---|---|
| **cvars declared** | **137** | `CVARINFO` — every one of them is read somewhere in `zscript.txt` |
| **menu rows that set a value** | **142** | `MENUDEF`: 45 `Option` + 91 `Slider` + 6 `UBQSPreset` |
| menu pages | 14 | `MENUDEF:179, 209, 233, 250, 262, 274, 291, 312, 328, 352, 367, 396, 412, 421` |
| named value lists | 15 | `MENUDEF:31-136` |
| key binds | 4 | `KEYCONF:11-14` |
| **selectable configurations** | **≈ 10^136** | 91 sliders → 10^117.9 of steps; 45 option rows → 10^17.9 |
| **behaviours a player can actually tell apart** | **7** | §2 |

**137 options. 7 behaviours. He is right.**

### The four fates, against his nine-entry draft

| fate | count | what it means |
|---|---|---|
| **SURVIVES** | **9** | is one of his nine entries, or is the cvar behind one |
| **FIXED** | **97** | disappears into a hardcoded value — in 97 of 97 cases, *today's default* |
| **ADVANCED** | **16** | cvar stays, no menu row; for people who edit configs |
| **DELETED** | **15** | cvar and the code branch it guards both go |

Every one of the 137 has exactly one fate. The full assignment is §5.

**The 97 is the finding.** Not one of those 97 needs a value other than the one it ships
with. `CVARINFO:72-73` says so in the mod's own words:

> *"EVERY DEFAULT BELOW IS THE LITERAL IT REPLACES, to the digit, so nothing changes until he
> moves it"*

**73 of the 137 cvars (53%) are in that block** (`CVARINFO:71-157`). Seventy-three options were
created with the stated intent of changing nothing.

### The health of the 137, on a second axis

| | count | |
|---|---|---|
| **DEAD** — cannot run at shipped defaults | **6** | §3.1 |
| **VESTIGIAL** — live only on a superseded path | **22** | §3.2 |
| **REDUNDANT** — another option already does it | **12** | §3.3 |
| **LOAD-BEARING** — used, distinct, and a player can need it | **14** | §3.5 |
| inert tuning numbers — live, correct, and nobody should touch them | **83** | → FIXED |
| **CONTRADICTORY** — combinations that cannot both hold | **7 clusters** | §3.4 (cross-cutting) |

### The three biggest collapses

1. **Two beam switches become none.** `ubqs_hand_glow` 1.0 / `ubqs_blade_glow` 0.0 /
   `ubqs_blade_tube` / `ubqs_beam_shift` plus 11 world-beam look knobs plus 9 prediction knobs
   = **25 options → 0**. The hand beam is the blade; the world beam already forces itself on at a
   hardcoded `0.6` when the blades cross (`zscript.txt:1662-1663`) and needs no slider to do it.
2. **The block's dead branch takes its own three options with it.** `ubqs_block`,
   `ubqs_block_cone`, `ubqs_block_swung` live in the `else` of
   `if (gw > 0.01)` (`zscript.txt:2325-2340`) where `gw` is `ubqs_blade_guard`, default **6.0**.
   **Unreachable, and the front page's own "Block Assist" preset never reaches it either** —
   all three of its choices set `ubqs_blade_guard` non-zero (`MENUDEF:167-169`). **3 → 0**, and
   the only playsim-RNG draw in the whole block path (`random[UBQSBlock]`, `:2339`) goes with it.
3. **Twenty-six Force options become one.** Keep `ubqs_force` as a cvar; fix the other 25 at
   today's values. The previous session already proved five of them move together — the
   "Force Strength" preset sets `push_power pull_power push_range pull_range grip_mass` as one
   choice (`MENUDEF:173-175`). The other 20 never needed a row.

### Proposed menu size

**His nine, plus two I argue for: 11 entries, one page, no sub-pages.** §6.
Down from 142 rows across 14 pages. The "Tuning and Debug" tree (`MENUDEF:274-434`,
8 pages, 91 sliders) disappears entirely.

---

## 1. HIS DRAFT, MEASURED — three of nine entries break as written

I checked each of the nine against the code. Six are safe. Three need a decision.

### 1.1 `Blade length: 50, 75, 100, 125, 150` — **125 and 150 do not exist**

`ubqs_size` (default `1.0`, `CVARINFO:3`) is not continuous. `ResolveSizeIdx()`
(`zscript.txt:466-478`) **snaps whatever you set to the nearest of five baked sizes**:

```
static double Size(int s) { static const double T[] = { 0.70, 0.85, 1.00, 1.15, 1.30 }; ... }
```
— `zscript.txt:30`

And the blade is `FullLen() = 26.0 * Size(SizeIdx())` map units (`zscript.txt:851`). With
`vr_vunits_per_meter 34.0` (verified, `qzd173/src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:94`)
the five baked blades are:

| index | `ubqs_size` | map units | **real length** |
|---|---|---|---|
| 0 | 0.70 | 18.2 | **54 cm** |
| 1 | 0.85 | 22.1 | **65 cm** |
| 2 | 1.00 (default) | 26.0 | **76 cm** |
| 3 | 1.15 | 29.9 | **88 cm** |
| 4 | 1.30 | 33.8 | **99 cm** |

So as written, his five steps would land like this: 50 → index 0, 75 → index 2, 100 → index 4,
**125 → index 4, 150 → index 4**. The top three entries would all draw and cut the same 99 cm
blade. The player would move the setting and nothing would happen.

**Two honest ways out, and the second is cheap:**

- **(a) Label the five that exist.** `54 / 65 / 76 / 88 / 99`, or "Short / Shorter / Standard /
  Longer / Long". **MERGE / S.** Costs nothing, but the top is 99 cm and he asked for 150.
- **(b) Bake two more sizes.** Per-size difference in `MODELDEF` is **the `Scale` line and nothing
  else** — `Scale -1.126 1.126 1.126` at 0.70 (`MODELDEF:22`) against `-2.090 2.090 2.090` at 1.30
  (`MODELDEF:9422`); the ratio 2.090/1.126 = 1.856 against 1.30/0.70 = 1.857, so Scale is exactly
  linear in size and **there is no new art.** 50/75/100/125/150 cm need
  `ubqs_size` = 0.654 / 0.98 / 1.31 / 1.63 / 1.96, i.e. two new entries in `T[]`, two new Scale
  values, and a re-run of `ubqs.py`. **FIX DEFAULT / M** — M only because the generator input is
  missing (§4).

**Recommended: (b).** He asked for 150 cm; a 76 cm blade labelled "100" is the kind of thing that
makes a menu untrustworthy.

### 1.2 `Damage: sliderbar 0.5 - 2.0` — **safe, but it is a new cvar's worth of meaning**

`ubqs_damage` is **absolute**, default `30.0` (`CVARINFO:54`), today a 10-100 slider
(`MENUDEF:316`). His 0.5-2.0 is a multiplier. Read at three sites and clamped at none of them:
`zscript.txt:1875` (the cut), `:2016` (the twirl disc, × `ubqs_dmg_twirl`), `:4085` (the thrown
blade, × `ubqs_dmg_thrown`).

**0.5-2.0 × 30 = 15-60. Both endpoints are safe.** At 0.5 a cut does 15 and the per-victim
cooldown is 8 tics (`ubqs_cut_rate`, `:1873`), so an imp at 60 hp dies in 4 cuts ≈ 0.9 s — still a
lightsaber. At 2.0 it is 60, a one-cut imp. Nothing divides by it and nothing clamps it.
**KEEP / S.** Either re-express the cvar as a multiplier, or leave `ubqs_damage` absolute and let
the row write 15…60 while showing 0.5…2.0. The second is less work and loses nothing.

### 1.3 `Max Lock On: 1 - 5` — **safe, exactly**

`ubqs_locks`, default 5 (`CVARINFO:29`), read once: `clamp(UInt("ubqs_locks", 5), 1, 8)`
(`zscript.txt:747`). His range sits strictly inside the code's own clamp and his top is the
shipped default. Nothing breaks at either end. **KEEP / S.**

### 1.4 `Offhand Saber Dynamic Light: On / Off` — **confirmed; the 40 is 1.17 metres**

The coordinator's find stands, with one correction to the line numbers. `zscript.txt:2845-2848`:

```
if (want == 1 && bOffhandWeapon)
{
    let o = Other();
    if (o && (o.HandPos() - HandPos()).Length() < 40.0) want = 0;
}
```

**The unit reasoning, checked on both sides of the subtraction.** `HandPos()` is
`UBQS_Data.HandPosOf(owner, bOffhandWeapon)` (`zscript.txt:825`), which returns
`mo.OffhandPos` or `mo.AttackPos` raw (`:141-146`). The engine writes those in **map units**:
`AttackPos.X = player->mo->X() - (weaponoffset[0] * vr_vunits_per_meter)`
(`qzd173/src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:593`), and the off hand the same
at `:606-607`. `vr_vunits_per_meter` is **34.0** (`hw_vrmodes.cpp:94`). So both terms are map
units and the subtraction is sound — **40.0 map units = 1.176 m.**

One wrinkle that makes it slightly worse, not better: the Z component carries
`/ pixelstretch` (`hw_openxrdevice.cpp:608`), so vertical hand separation reads **1.25× larger**
than life while horizontal reads true. Hands side by side therefore have to be nearly 1.2 m apart
to escape the merge.

**Verdict: the off-hand blade is unlit in every normal dual-wield pose, at the default
`ubqs_lights 1`.** A human dual-wield stance is 0.3-0.7 m. The comment at `:2837-2838` describes
the intent as *"when your two blades are close together only the main hand's lights them both"* —
and 40 cm is what that describes. **40 cm × 34 = 13.6 units.**

**Recommendation: drop the merge entirely. REMOVE / S.** Now that he wants an explicit switch,
the merge is a second, invisible answer to the same question — and he has already noticed it as a
bug. Delete the four lines, read a per-hand light setting instead. If he wants the saving kept,
`13.6` is the corrected number, but two mechanisms for one choice is exactly the complaint.

### 1.5 `Start with Dual Sabers: On / Off` — it is two cvars, and one of them has no home

`ubqs_start` (`CVARINFO:38`, "do you begin holding one", read `zscript.txt:5755`) and
`ubqs_dual_mode` (`CVARINFO:40`, "one saber / Dual key / always out", read `:5753`) are different
questions. Mapping his entry:

- **On** → `ubqs_start true` + `ubqs_dual_mode 2` (always out)
- **Off** → `ubqs_start true` + `ubqs_dual_mode 0` (one saber)

`ubqs_dual_mode 1` — the Dual key, matched to Brutal Doom's `+user3` — then has no menu row, but
the bind still exists (`KEYCONF:11`), so it goes **ADVANCED**, not deleted.
`ubqs_start false` (the saber exists in the world but you pick it up) also goes **ADVANCED**;
"Sabers Off" covers the common case of not wanting it at all.

### 1.6 The other three entries are clean

`Sabers On/Off` → `ubqs_on` (`CVARINFO:37`, read `zscript.txt:5492`). `Mainhand Saber Color` →
`ubqs_color` (0, 8 colours, `:479`). `Offhand Saber Color` → `ubqs_color_off` (2, same accessor).
No endpoint problems; `ColIdx()` clamps to `0..NCOL-1` itself.

---

## 2. THE COMBINATORIAL TRUTH

**Selectable: ≈ 10^136.** 91 sliders, multiplied out at their own declared `MENUDEF` steps, give
10^117.9; the 45 discrete option rows give 10^17.9 (29 of them are plain On/Off).

**Distinct, in play: 7.** Everything the saber does falls into one of these, and no option adds an
eighth:

| | behaviour | entered by | code |
|---|---|---|---|
| 1 | **One saber, blade in hand** — the default | — | `UBQS_Saber`, `zscript.txt:3045` |
| 2 | **Two sabers** | `ubqs_dual_mode` 1/2 | `UBQS_SaberOff`, `:3046` |
| 3 | **Double-ended staff**, and its twirl disc | the Second Blade key / `ubqs_alt 0` | `back`, `:405`; `TwirlDisc`, `:2014` |
| 4 | **Blades crossed → world sabers, cross-guard deflect** | hands together, `ubqs_crossguard` | `Guarding()`, `:2611`; lock `:2602-2638` |
| 5 | **Thrown blade** | the throw gesture | `UBQS_Thrown`, `:3739` |
| 6 | **The Force** — push, pull, choke, jump | off hand / bare hand / off-hand fist | `UBQS_Force`, `:4174` |
| 7 | **The training remote**, scored | the Remote key | `UBQS_Remote`, `:5207` |

Of the 137 options, **9 move you between these seven** and **128 adjust a number inside one of
them.** That is the shape of the problem: the menu is 93% dials and 7% choices.

**And one of the seven is unreachable on a clean profile.** Behaviour 3 needs `back` true, which
needs either the Second Blade key — shipped **unbound** (`KEYCONF:12` adds a bindable row, there is
no default bind; only `AddSlotDefault 1 UBQS_Saber` at `KEYCONF:1`) — or `ubqs_alt 0`, against a
default of 1 (`CVARINFO:6`). The mod's own code says so: `zscript.txt:5566`,
*"so on a default profile the staff twirl could never happen"*. 400 `MODELDEF` FrameIndex lines of
twirl geometry and two cvars (`ubqs_twirl`, `ubqs_dmg_twirl`) sit behind a key nobody has pressed.

---

## 3. THE CLASSIFICATION, WITH THE ABSENCE SHOWN

### 3.1 DEAD — 6

| cvar | default | why it cannot run | cite |
|---|---|---|---|
| `ubqs_block` | 70 | in the `else` of `if (gw > 0.01)`, `gw = ubqs_blade_guard` **6.0** | `zscript.txt:2325`, `:2339` |
| `ubqs_block_cone` | 75.0 | same branch | `:2334` |
| `ubqs_block_swung` | 12 | same branch | `:2338` |
| `ubqs_dual` | true | its only effect is a one-shot migration **that sets it back to true** in the same line | `:5540` |
| `ubqs_twirl` | 2 | needs `back`, which needs an unbound key or `ubqs_alt 0` | `:876`, `:5566`, `KEYCONF:12` |
| `ubqs_dmg_twirl` | 0.8 | read only inside `TwirlDisc` | `:2016` |

The `ubqs_block` cluster is the sharpest of these because **the menu advertises all three
prominently** — "Block Chance", "Block Cone", "Swing Block Time" (`MENUDEF:343-345`) — and the
front page's Block Assist preset writes `ubqs_block` in all three of its choices
(`MENUDEF:167-169`) where it can never be read. The code says it outright at `:2319-2320`:
*"IT REPLACES THE ROLL, it does not stack on it."*

`ubqs_dual`'s migration, in full (`:5536-5541`):
```
let dc = CVar.FindCVar("ubqs_dual");
let mc = CVar.FindCVar("ubqs_dual_mode");
if (dc && mc && !dc.GetBool()) { mc.SetInt(0); dc.SetBool(true); }
```
It is also **the only cvar of the 137 with no menu row at all** (verified: every other name in
`CVARINFO` appears in an `Option`/`Slider`/preset row).

### 3.2 VESTIGIAL — 22

All 22 are read only through `BladeGlow()` (`zscript.txt:1658`) or the world hilt
(`:1586`), and **both are off at shipped defaults**: `ubqs_blade_glow` **0.0** (`CVARINFO:14`)
makes `BladeGlow` return at `:1665`, and `ubqs_world_saber` **false** (`CVARINFO:23`) makes
`WorldUp()` return false at `:1508`.

- **The world beam's look (11):** `ubqs_glow_width` `:1700`, `ubqs_glow_quality` `:1701`,
  `ubqs_glow_dots` `:1743`, `ubqs_glow_motionfade` `:1703`, `ubqs_glow_fade_deg0` `:1359`,
  `ubqs_glow_fade_deg1` `:1360`, `ubqs_glow_fade_widen` `:1706`, `ubqs_glow_fade_jerk` `:1361`,
  `ubqs_glow_overlap_core` `:1719`, `ubqs_glow_overlap_halo` `:1720`,
  `ubqs_blade_glow_color` `:578`/`:1697`.
- **The world beam's motion prediction (8):** `ubqs_glow_predict` `:1427`, `ubqs_glow_lead`
  `:1428`, `ubqs_glow_frame_ms` `:1353`, `ubqs_glow_maxang` `:1354`, `ubqs_glow_maxlin` `:1355`,
  `ubqs_glow_ramp_deg` `:1356`, `ubqs_glow_stop` `:1357`, `ubqs_glow_hitch_ms` `:1358`.
- **(3):** `ubqs_beam_shift` `:1674`, `ubqs_blade_tube` `:495`, `ubqs_blade_glow` `:1660`.

**They are not quite zero, and this is the honest part.** The world beam **does** come up when the
blades cross, and it comes up at a **hardcoded 0.6 regardless of the cvar** (`:1662-1663`):

```
// A LOCK BRINGS THIS ONE UP whatever ubqs_blade_glow says ...
if (Locked() && str <= 0.01) str = 0.6;
```

That is the crossed-blade path, and per the settled design facts it is correct. So the 11 look
knobs do affect the picture during a lock. **The 8 prediction knobs do not even do that**: the
predictor's whole job is to lead a *moving* blade, and `GlowPredict` returns early on its
exact-at-rest branch (`gStill`, `:1419-1421`) — and a lock is held with two blades pressed
together. `GlowTunables()` runs every tic for both hands (`:1126`) and its result is discarded.

`ubqs_blade_tube` is vestigial in the mod's own words (`:485-486`, `:490-494`): the solid tube's
companion — the emitter nub — was *"Gone in code, not by a default"*, and §4 shows the data for it
is still being generated.

### 3.3 REDUNDANT — 12

| the pair / set | what already does it | cite |
|---|---|---|
| `ubqs_glow_predict` + `ubqs_glow_lead` | **identical**: `lead = pred ? clamp(UF("ubqs_glow_lead",1.0),0,2) : 0.0` — predict off ≡ lead 0 | `:1427-1428` |
| `ubqs_alt` 0 | the **Second Blade key** calls the same `ToggleBack()` | `:728` vs `KEYCONF:12` → `:5569-5581` |
| `ubqs_pull_power` | a pure multiplier of `ubqs_push_power`, both default 1.0; `CVARINFO:125` admits *"there was no pull cvar"* | `:4543` |
| `ubqs_hum_off` | a second slider on the same hum, default 1.0 = "the same" | `:1369` |
| `ubqs_haptic_cut`, `ubqs_haptic_clash` | subsets of `ubqs_haptics`; three switches, one feature | `:1878`, `:2723` vs `:807` |
| `ubqs_vol_swing`, `_hit`, `_clash`, `_force`, `_ignite` (5) | five attenuators that can only attenuate — `CVARINFO:134-135` states no sample has `$volume` above 1.0 — over the engine's own `snd_*` sliders | `:2822`, `:2004`, `:2287`, `:4700`, `:818` |

### 3.4 CONTRADICTORY — 7 clusters, and who silently wins

1. **`ubqs_blade_guard` > 0 kills `ubqs_block` / `_cone` / `_swung`.** *`blade_guard` wins,
   silently.* `:2325`. The player sets Block Chance to 90 and gets geometry.
2. **`ubqs_force 0` overrides `ubqs_alt "Force Push"`** and turns alt into the second-blade
   toggle. *`ubqs_force` wins.* `:722-729` — and the comment says this is deliberate.
3. **`ubqs_dual_mode 2` kills `ubqs_force_hand`.** `wantOff` is true, so `fhand` is false and the
   bare Force hand is never given. *`dual_mode` wins.* `:5807-5810`.
4. **An invisible blade that still kills.** `ubqs_world_saber 0` + `ubqs_blade_tube 0` +
   `ubqs_hand_glow 0` + `ubqs_blade_glow 0` + `ubqs_lights 0` draws **nothing** while `CutAlong`
   runs from `DoEffect` regardless (`:1261`, `:1847`). Every one of those five is reachable from
   the menu (`MENUDEF:215-216, 219, 295-296`). *Nobody wins.* This is the same failure shape the
   mod's own comment calls the worst possible (`:2160-2164`).
5. **`ubqs_haptic_power`'s top two thirds do nothing.** Slider 0-3 (`MENUDEF:409`), but
   `amp = clamp(k * 0.85, 0.12, 1.0)` (`:3238`), so anything above ≈1.18 saturates — and the floor
   0.12 means 0.05 is not quiet either. *The clamp wins.* Already recorded as known in
   `docs/HANDOFF_polish.md` ("Review fixes": *"amplitude caps at 1.0, so the top of the power
   slider saturates"*). **FIX DEFAULT / S** independent of everything else here.
6. **Two sliders the code overrules.** `ubqs_glow_fade_deg1` is forced to `fade_deg0 + 1.0`
   (`:1360`) and `ubqs_glow_ramp_deg` to `GLOW_STILLOUTW + 1.0` (`:1356`). *The code wins*, and the
   menu shows the number the player chose.
7. **`ubqs_lights 1` + two sabers: the off light is suppressed at 1.17 m.** *The merge wins.*
   `:2845-2848` — §1.4, and the reason his ninth entry exists.

### 3.5 LOAD-BEARING — 14, and I am being strict

Used, genuinely distinct, and a reasonable player can need a value other than the default:

| | cvar | why |
|---|---|---|
| 1 | `ubqs_on` | the master switch |
| 2 | `ubqs_dual_mode` | one saber or two is content, not tuning |
| 3 | `ubqs_color` | — |
| 4 | `ubqs_color_off` | two blades want two colours |
| 5 | `ubqs_size` | reach: the only number that changes what you can hit |
| 6 | `ubqs_size_off` | a shoto in the off hand; all five sizes are baked for both classes (`:460-462`) |
| 7 | `ubqs_damage` | difficulty |
| 8 | `ubqs_locks` | a real behaviour cap, not a feel number |
| 9 | `ubqs_lights` | performance on Quest, and his off-hand light |
| 10 | `ubqs_swing_cut` | the mod's own *"most-felt number"*, and body-dependent — `:1225-1228` |
| 11 | `ubqs_blade_guard` | the whole geometry of the block; `HANDOFF_polish` leaves the number to him |
| 12 | `ubqs_deflect_aim` | three genuinely different outcomes — at the shooter / where you look / a true blade mirror (`:2112-2128`) |
| 13 | `ubqs_world_saber` | **settled**: the crossed-blade path, and the lever for `03_SABER_AND_SAW` §2.6 |
| 14 | `ubqs_force` | a whole subsystem (`UBQS_Force`, 911 lines) on or off |

Nine of those 14 are his nine entries. Two I argue onto the page (§6.2). Three stay as cvars.
**The remaining 83 cvars are live, correct, and want a constant.**

---

## 4. GENERATED-CODE SMELL — and the generator input is gone

`zscript.txt:4-9` says the lump is generated by `RS_VR_Weapons/_pending/tools/ubqs.py` from
`ubqs_zscript.txt`. `MODELDEF:2` says the same of itself. **Neither `ubqs.py` nor
`ubqs_zscript.txt` is in the repository** (`.gitignore` is 71 bytes; the tree holds no `tools/`).
`MENUDEF:7` names a second absent generator, `tools/rsmenu/presets.py`, and `MENUDEF:138` a third
behaviour of it; `CVARINFO:19` cites `sim_real.py` and `:1711` `tools/joint51vs52.py`.

**This is the maintenance finding, said plainly: 325 KB of ZScript and 1.27 MB of MODELDEF are
committed build output whose build is not. Edit either by hand and the next generator run — on a
machine that still has the input — silently reverts it.** Five tools are cited by name in these
lumps and none can be run from this repository. Any cleanup this report proposes has to be made in
`ubqs_zscript.txt` and `ubqs.py`, wherever they are, or it will not survive.

### 4.1 The clearest piece of generator output: 480 unreachable MODELDEF blocks

`MODELDEF` holds **1,446 model blocks**. Per psprite saber class there are 522; six of its sprite
prefixes carry exactly 80 blocks (1,680 `FrameIndex` lines) each, and they map to four different
`.md3` files:

| prefix | model | can ZScript select it? |
|---|---|---|
| `K` | `ubqs_stub0.md3` | **yes** — `SpriteNow()`, `pre = "K"`, `zscript.txt:495` |
| `S` | `ubqs_saber.md3` | **yes** — `pre = "S"` when `ubqs_blade_tube` is 1, `:495-496` |
| `G` | `ubqs_glow.md3` | **yes** — `A_UBGlow`, `:580` |
| `V` | (twirl geometry) | **yes** — `:546`, but see §2 (unbound key) |
| **`L`** | **`ubqs_stub2.md3`** | **no** |
| **`M`** | **`ubqs_stub4.md3`** | **no** |
| **`N`** | **`ubqs_stub6.md3`** | **no** |

`SpriteNow()` is the only producer of a saber sprite name and `pre` can only ever be `"S"` or
`"K"` (`:489`, `:495`). The strings `L%d`, `M%d`, `N%d` do not occur in `zscript.txt`; the only
`String.Format` calls that build a saber sprite are `:496`, `:546`, `:580`, `:4023`.

**`ubqs_stub2 / 4 / 6` are the emitter-nub lengths that `zscript.txt:490-494` says were removed:**
*"NO NUB (build 54 ... ) ... Gone in code, not by a default"*. The code went. **The generator was
never told**, so it still emits **480 of 1,446 blocks — 33% of MODELDEF, and three of the sixteen
md3 files — for a feature that was deleted.** That is the single clearest "the generator emitted a
family" finding in the mod. **REMOVE / S** (one change in `ubqs.py`).

### 4.2 The rest of the render data, by how often it can be drawn

| blocks | class | drawn when |
|---|---|---|
| 160 | `UBQS_Saber` K + G reachable | always (default) |
| **480** | `L`/`M`/`N` on both saber classes | **never** (§4.1) |
| 160 | `S` on both saber classes | only at `ubqs_blade_tube 1` |
| **320** | `UBQS_WorldSaber` + `…Off` | only at `ubqs_world_saber 1` — **opt-in since 10-29** |
| **64** | `UBQS_BeamGlow` (4 quality × 8 colours × core/halo) | only while `ubqs_blade_glow > 0` **or the blades are crossed** |
| 400 lines | `V` twirl | only with the second blade out (§2) |
| 18 | `UBQS_Thrown`, `UBQS_Remote`, `UBQS_ForceHand` | their own paths |

### 4.3 Four more generator fingerprints

- **The accessor pair, emitted twice.** `UF`/`UInt` on `UBQS_SaberBase` (`:452-453`) and
  `FV`/`IV` on `UBQS_Force` (`:4239-4240`) are **four one-line forwarders to the same two
  functions**, `UBQS_Data.F`/`.I` (`:191`, `:198`). Four names for one job, used 74 / 50 / 32 / 15
  times, beside 29 raw `UBQS_Data.F|I` calls and 32 raw `CVar.FindCVar`. **Six ways to read a
  cvar.** Nobody writes that by hand; a generator that grew two emitters does.
- **The dead haptic class.** `class UBQS_HapticPuff : Actor` (`:3264`) — the string occurs
  **exactly once in 6,514 lines, its own declaration** (verified). The abandoned `LineAttack`
  haptic route. **REMOVE / S**, as `03_SABER_AND_SAW` §2.5 already proposed.
- **Nine options that re-expose nine constants.** `GlowTunables()` (`:1351-1361`) reads nine
  cvars whose defaults are, by the block's own comment, *"the GLOW_* constant it replaces, to the
  digit"*. The constants still exist in the file. This is a generator pass that turned private
  tuning into public API for no caller.
- **Comments that no longer describe the code.** `:515-516` says `ubqs_hand_glow` *"0 (the
  default)"* and `:552` says *"off by default, as he asked"*; it has defaulted to **1.0** since
  10-29 (`CVARINFO:79`). `:485-486` says the tube is off *"because the world glow is the blade"*;
  the world glow is now off too. Three stale statements in the one function that decides what the
  player sees. Not a bug — but the next session reads these, not this report.

---

## 5. THE FULL FATE TABLE — all 137

### SURVIVES (9) — his nine entries

`ubqs_on` · `ubqs_dual_mode` · `ubqs_locks` · `ubqs_damage` · `ubqs_color` · `ubqs_size` ·
`ubqs_color_off` · `ubqs_size_off` · `ubqs_lights`

Plus **one cvar that does not exist yet**: a per-hand light setting for his ninth entry
(`ubqs_lights_off`, or read `ubqs_lights` per instance — `Lights()` is already per-saber,
`:2841`, so this is one line).

### DELETED (15) — cvar and branch both

| cvar | with it goes | cite | cost |
|---|---|---|---|
| `ubqs_dual` | the migration at `:5536-5541` | — | nothing; it is DEAD |
| `ubqs_block` | the whole `else` roll branch `:2330-2340` | — | the old percentage block |
| `ubqs_block_cone` | same | — | nothing; DEAD |
| `ubqs_block_swung` | same, and the twirl-immunity abuse `:2335-2337` | — | nothing; DEAD |
| `ubqs_blade_glow` | the slider; the lock's hardcoded `0.6` becomes the only world-beam strength | `:1660-1663` | a dimmable world beam in normal play — which nobody has had since 10-29 |
| `ubqs_blade_glow_color` | a second colour picker for the same blade | `:578` | a halo in a different colour from the core |
| `ubqs_blade_tube` | `pre = "S"`, and 160 MODELDEF blocks | `:495` | the opaque solid-blade look he rejected (`:490-494`) |
| `ubqs_beam_shift` | `:1668-1674` | — | fitting a hilt that is not Wardust's. One hilt ships |
| `ubqs_glow_dots` | `GlowDots()`, `:1740-1764` | — | the build-48 look, default off |
| `ubqs_glow_predict` | redundant with lead | `:1427` | nothing |
| `ubqs_glow_lead` | prediction fixed on at 1.0 | `:1428` | nothing a lock can show |
| `ubqs_pull_power` | push and pull share one strength | `:4543` | asymmetric push/pull |
| `ubqs_hum_off` | the off hum = the main hum | `:1369` | a quieter second blade |
| `ubqs_haptic_cut` | folded into `ubqs_haptics` | `:1878` | cut buzz without clash buzz |
| `ubqs_haptic_clash` | folded into `ubqs_haptics` | `:2723` | clash buzz without cut buzz |

### ADVANCED (16) — cvar stays, menu row goes

`ubqs_start` (true) · `ubqs_world_saber` (false — **settled, keep**) · `ubqs_swing_cut` (8.0) ·
`ubqs_blade_guard` (6.0) · `ubqs_deflect_aim` (1) · `ubqs_deflect_lead` (3.0 — open question 3 of
`03_SABER_AND_SAW` §5) · `ubqs_throw_min` (1.5) · `ubqs_force` (1) · `ubqs_force_jump` (1) ·
`ubqs_force_gesture` (2.0) · `ubqs_gesture_straight` (0.55) · `ubqs_saber_sound` (0) ·
`ubqs_haptics` (1) · `ubqs_lefty` (-1) · `ubqs_remote_level` (2) · `ubqs_diag` (0)

### FIXED (97) — at today's default, every one

**Beam look, world (9):** `glow_width` 1.2 · `glow_quality` 3 · `glow_motionfade` 1 ·
`glow_fade_deg0` 200 · `glow_fade_deg1` 500 · `glow_fade_widen` 0.25 · `glow_fade_jerk` 1.0 ·
`glow_overlap_core` 0.5 · `glow_overlap_halo` 1.5 — the last two were **measured off the shipped
meshes** with `tools/joint51vs52.py` (`:1708-1718`), so they are a measurement, not a preference.

**Beam prediction (7):** `glow_frame_ms` 11.1 · `glow_maxang` 35 · `glow_maxlin` 4 ·
`glow_ramp_deg` 60 · `glow_stop` 0.6 · `glow_hitch_ms` 120 · **`snap_turn_deg` 45** — that last
one is **load-bearing code**: it also suppresses a cut after a snap turn (`:1096`), which is
gameplay correctness, not a look. Fix it, never delete it.

**Which blade is drawn (1):** `hand_glow` **1.0** — the blade.

**Light and FX (6):** `flash` → **2**, not 1, see below · `trail` 1 · `trail_steps` 2 ·
`trail_life` 6 · `clash_fx` 1.0 · `cut_fx` 1.0.

> **`ubqs_flash` is the one FIXED value I am not setting to today's default, and here is why.**
> At `1` the flashes are conditional on the global blade light: `if (fl == 1 && UInt("ubqs_lights",
> 1) <= 0) return;` (`:2882`). The cvar exists precisely because coupling them was wrong — the
> comment at `:2876-2880` says dropping the light *"silently took the hit, block, deflect, clash,
> guard and burn flashes with it — the player lost his combat FEEDBACK meaning only to lose the
> blade light."* Once his ninth entry can switch a light off, `1` re-opens that trap. **Fix at 2:
> combat feedback always, independent of the light.** Cost: a player who wants no flashes at all
> loses that, and it was never a row he would find.

**Swing and cut (8):** `swing_scale` 1.0 · `cut_rate` 8 · `cut_pad` 3.0 · `cut_shove` 1.0 ·
`cut_lift` 0.35 · `cut_los` 1 · `cut_sustain` 0.6 · `dmg_twirl` 0.8 · `dmg_thrown` 1.5 (9).

**Second blade (2):** `alt` 1 (Force Push — the Second Blade key already covers 0) · `twirl` 2.

**Block, deflect, clash (16):** `deflect` 1 · `deflect_assist` 14 · `block_return` 1 ·
`block_damage` 20 · `block_stagger` true · `stagger_push` 12 · `hitscan_guard` true ·
`parry_speed` 6.0 · `parry_power` 2.0 · `crossguard` 1 · `guard_arc` 100 · `guard_width` 14 ·
`contact` 3.5 · `contact_hilt` 0.75 · `lock_stick` 1.7 · `burn` 1.

> **One balance note while fixing these, not a menu entry.** `ubqs_guard_arc` 100 is 200° of
> *total* immunity measured off the body's yaw, and the code calls it *"the most abusable number in
> the mod, with no way to tighten it"* (`:2275-2277`). Fixing it at 100 ships that as the only
> answer. Worth one headset session before it is welded shut.

**Throwing and lock-on (9):** `throw` 0 (Comes Back) · `throw_spin` 14 · `throw_plane` 0 ·
`throw_double` 1 · `throw_range` 512 · `throw_straight` 0.55 · `throw_blend` 0.6 ·
`throw_speed` 1.0 · `throw_on_lock` 1 · `lock_cone` 14 (10).

**The Force (21):** `force_hand` true · `force_gestures` 1 · `jump_power` 1.0 · `jump_hold` 8 ·
`jump_float` 1.0 · `push_power` 1.0 · `push_range` 448 · `push_damage` 8 · `push_cooldown` 1.5 ·
`push_cone` 40 · `pull_range` 768 · `pull_cone` 30 · `grip_mass` 600 · `grip_damage` 18 ·
`grip_cone` 12 · `grip_range` 1024 · `grip_still` 1.5 · `grip_hold` 8 · `grip_boss` 0 ·
`grip_fling` 3.5 · `grip_slam` 3.0.

> `grip_still` and `grip_hold` look like accessibility knobs and are safe to fix because
> **RSQ_Accessibility already widens them**: `UBQS_A11y.Low(IV("ubqs_grip_hold", 8))` and
> `.Wide(FV("ubqs_grip_still", 1.5))` at `:4426`, driven by `rsqa_gesture_ease`. The a11y pk3
> covers the case; the saber does not need its own slider for it.

**Sound (11):** `hum` 0.45 · `vol_swing` 1.0 · `vol_hit` 1.0 · `vol_clash` 1.0 · `vol_force` 1.0 ·
`vol_ignite` 1.0 · `snd_choke` 1 · `swing_gap` 8 · `swing_spatial` 1 · `doppler` 1.0 ·
`remote_chirp` 1.0.

**Haptics (1):** `haptic_power` 1.0 — **and fix the saturation while you are there** (§3.4 item 5).

**Training remote (2):** `remote_sting` 2 · `remote_warn` 0.

**Compatibility (2):** `bd_sword` true · `chainsaw` false — Xim's chainsaw-saber is **auto-detected**
by its own lump or its pickup string (`:5496-5510`), so the switch only ever served the case the
code already handles.

---

## 6. THE MENU HE SHOULD SHIP

### 6.1 The page

```
RSQ SABER

    Sabers                           On
    Start With Dual Sabers           On
    Damage                      [ 1.0 ]        0.5 - 2.0
    Max Lock-On Targets         [  5  ]        1 - 5

    Main Blade Colour                Blue
    Main Blade Length                100
    Off Blade Colour                 Red
    Off Blade Length                 100
    Off Blade Light                  On

    Swing to Cut                [ 8.0 ]        3 - 20      <- proposed
    Blade Block Reach           [  6  ]        0 - 24      <- proposed

    Controls                                               > 4 binds
```

Eleven value rows plus the Controls submenu, in one screen, two groups.
Down from **142 rows across 14 pages**. Defaults are exactly today's, so nobody's save changes.

**Keep the Controls submenu** (`MENUDEF:262-271`, 4 `Control` rows). `KEYCONF:2-4` records why it
exists: *"THE SECOND BLADE AND THE DOUSE ARE THE ONLY INPUTS THIS MOD HAS THAT ARE NOT THE
TRIGGER"*. Drop the duplicate `Command "Summon / Dismiss Remote"` row (`MENUDEF:270`) — the
`Control` row above it binds the same alias.

**Everything else disappears:** `UBQS_Look`, `UBQS_Swing`, `UBQS_TheForce` and the whole
`UBQS_Debug` tree (`UBQS_TLook`, `UBQS_TSwing`, `UBQS_Contact`, `UBQS_Throwing`, `UBQS_TForce`,
`UBQS_Sound`, `UBQS_Trainer`, `UBQS_Predict`) — 10 pages, 91 sliders.
**And the 6 presets go with them** (`MENUDEF:139-177`): a preset is a row that sets five rows that
no longer exist. `ubqs_damage` and `ubqs_size` become real entries instead, which is what the
Damage and Size presets were standing in for.

**Keep `SafeCommand "Reset All"`** (`MENUDEF:288`) — retargeted at the surviving cvars. It is the
one row that earns its place on a page this short, and the previous session built it deliberately
(`HANDOFF_polish`, 10-27: *"one global reset"*).

### 6.2 Does it need anything else? — **Two, and I will defend both. Otherwise: no, nine is right.**

He asked the right question and the answer is close to his draft. The nine cover every *choice*.
What they do not cover is **two thresholds that depend on his body and his load order**, where no
single fixed value can be correct for two players.

**ADDITION 1 — `Swing to Cut` (`ubqs_swing_cut`, 8.0). The strong one.**
The mod says it about itself (`zscript.txt:1225-1228`):

> *"ubqs_swing_cut IS this mod's most-felt number: below it the blade passes through a monster and
> nothing happens, and what counts as a hard swing depends on arm length, standing or seated, and
> the size of the room. Every 'walking registers a swing' and every 'my slash did nothing' lands
> here."*

8 units of tip travel per tic is 8 × 35 / 34 = **8.2 m/s of tip speed**. A standing player with a
full arc clears that; a seated player, or one in a small room keeping the arc short, does not —
and gets a blade that passes through monsters. There is no value that serves both, and the failure
mode is the worst kind: the weapon silently does nothing. **This is the one option a player cannot
get a reasonable experience without.** It also drags the trail with it — the trail threshold is
`cut - 2.0` (`:1247`), the deliberate "you see the arc just before it bites" tell — so one row
tunes the feel and its own feedback together.

**ADDITION 2 — `Blade Block Reach` (`ubqs_blade_guard`, 6.0). The weaker one, but recorded.**
`docs/HANDOFF_polish.md` leaves this to him in writing, under `agent-block` → "Not done / headset":

> *"the leftover tracer hits are shots whose path misses the blade band (6 units each side,
> ubqs_blade_guard, the owner's 'small hitbox'); whether that is too small against BD is his call —
> the slider is already on the menu (Blade Block Reach)."*

Its measured numbers: at 6 units, **22 of 364** Brutal Doom rifle tracers still got through. Six
units is 17 cm each side of a 76 cm blade. Against a different monster set — vanilla Doom, HacX,
the suite — the right band differs, and `03_SABER_AND_SAW` §2.2 shows the swept sampling can still
let a mid-arc target through by up to 12 units of sagitta against this 6-unit tolerance. **It is a
recorded open decision, so deleting the row argues against a decision rather than implementing
one.** If he would rather have nine entries than eleven, this is the one to drop to ADVANCED —
but then §2.2's angular-lerp fix should land first, because the tolerance is doing work the sweep
is not.

**And that is all.** I read all 6,514 lines looking for a third and there is not one. Every other
load-bearing thing either has a home in his nine, has a key bind (second blade, douse, dual,
remote), is answered by the engine already (`ubqs_lefty` reads `vr_control_scheme`,
`zscript.txt:3227-3233`), is answered by RSQ_Accessibility (`UBQS_A11y`, `:6355-6453` — 12
`rsqa_*` settings already widen the saber's gesture thresholds, cap its flashes and scale its
text), or is a number with exactly one right value.

**Nine is a good menu. Eleven is a good menu. Both are right; 142 is not.**

### 6.3 Order of work, if he says go

| # | piece | label | cost |
|---|---|---|---|
| 1 | Split `ubqs_lights` per hand; **delete the 40.0-unit merge** (`:2845-2848`) | **REMOVE** | **S** |
| 2 | Delete the `else` roll branch (`:2330-2340`) and its 3 cvars | **REMOVE** | **S** |
| 3 | Delete `ubqs_dual`, `ubqs_blade_tube`, `ubqs_beam_shift`, `ubqs_glow_dots`, `ubqs_blade_glow_color`, `ubqs_glow_predict`/`_lead`, `ubqs_pull_power`, `ubqs_hum_off`, `ubqs_haptic_cut`/`_clash`, `ubqs_blade_glow`, `UBQS_HapticPuff` | **REMOVE** | **S** |
| 4 | Stop `ubqs.py` emitting `L`/`M`/`N` — 480 MODELDEF blocks, 3 md3 files | **REMOVE** | **S** |
| 5 | Inline the 97 FIXED cvars as constants, `ubqs_flash` at **2** | **MERGE** | **M** |
| 6 | Rewrite `MENUDEF` to §6.1; retarget Reset All | **MERGE** | **M** |
| 7 | Bake `ubqs_size` 1.63 and 1.96 so 125 and 150 exist (generator only, no art) | **FIX DEFAULT** | **M** |
| 8 | `ubqs_damage` as a 0.5-2.0 multiplier of 30 | **FIX DEFAULT** | **S** |
| 9 | Fix the `haptic_power` saturation at `:3238` | **FIX DEFAULT** | **S** |
| 10 | Correct the three stale comments at `:485`, `:515`, `:552` | **KEEP** | **S** |
| — | **All of it in `ubqs_zscript.txt` / `ubqs.py`, not in the committed lumps** (§4) | — | — |

---

## 7. WHAT NOT TO TOUCH

Read this list and trust that the cleanup above does not go near any of it.

1. **The sprite / world split, exactly as he stated it.** The saber is drawn on the sprite layer;
   the hilt is a psprite **all the time**; when the blades **cross**, they become world sabers.
   That is `WorldUp()` (`:1506-1512`), `A_UBShow`'s blanking (`:543`), `UBQS_WorldSaber`
   (`:3537`) and the lock's forced world beam (`:1662-1663`). **Nothing here is a defect and
   nothing in this report proposes changing it.** `ubqs_world_saber` keeps its cvar; it only loses
   a menu row.
2. **`GrippedInput()`, `zscript.txt:2173-2194`**, with its statement of the two-hand trap at
   `:2160-2164`. It re-drives paint, throw and recall from `p.cmd.buttons` for as long as
   `WF_TWOHANDSTABILIZED` is set, because the engine takes the off hand's weapon machinery away
   (`A_WeaponReady` `weapons.zs:415-418`, `FireWeapon` `player.zs:393-396`,
   `P_CheckWeaponButtons` `p_pspr.cpp:755-757` — all verified). Touch nothing in it.
3. **The swept blade segments, all three paths.** `CutLine` `:1894-1921` (3 positions per tic),
   `Deflect` `:2086-2095`, `SweptSep` `:2451-2469`. These exist because a hard swing used to miss
   (`:1894-1900`). **Any "simplification" that reduces them to one sample re-opens that.**
4. **The deflection capsule.** `SegSeg(blade, shot) <= radius + ubqs_blade_guard` — `Deflect`
   `:2094-2096`, `ContactBlock` `:2559`, `StopsHitscan` `:2483`. One width, three paths, all
   deterministic. And `BatBack` `:2105-2157`, including the true blade mirror
   `n = blade × (v × blade)` at `:2126-2128`.
5. **`ubqs_snap_turn_deg`'s second job.** It is in the prediction family and reads like a look
   knob, but `:1096` uses it to suppress a cut after a snap turn. **Fix it at 45; never delete
   it.** Same for `UBQS_FrameKeeper` (`:5882`), which measures the swing in the tracking frame so
   a turn is not a cut.
6. **`ubqs_cut_rate`'s role.** `:1867-1868`: *"the real cap on the saber's rate of killing, and the
   reason holding the blade inside a monster does not delete it."* FIXED at 8 means the constant
   stays; the per-victim `cutWho`/`cutWhen` logic (`:1892-1893`, `:1958-1959`) stays with it.
7. **`UBQS_PassThru` (`:6454`) and the contact block's `+REFLECTIVE +THRUREFLECT` loan**
   (`:2565`). Measured at 0 copies and 0 explosions over 17 shots per class
   (`HANDOFF_polish`, `agent-polish`). It is the one thing the saber does better than the saw.
8. **`WorldMirror(off)` in `BladeDirOf` (`:34-73`).** Left-handed swing frames were up to ~70° off
   the drawn blade before it (`HANDOFF_polish`, `agent-lh`, with the per-frame table).
9. **The one haptic funnel.** `Rumble()` `:806-812` → `UBQS_Buzzer` `:3210-3262`, `QF_WAVE`,
   `damrad = 0`, `tremrad 8`. Deleting `ubqs_haptic_cut`/`_clash` removes two gates **above** it
   and changes nothing inside it. **Do not add a haptic hum and do not add a `LineAttack` pulse** —
   `gl_openvr.cpp:519-520` drops a buzz on a busy channel.
10. **`UBQS_A11y` (`:6355-6453`) and every `.Low()` / `.Wide()` call site.** Fixing
    `grip_still`/`grip_hold`/`throw_straight`/`gesture_straight` as constants is safe **only
    because** the a11y layer still scales them at `:842`, `:4415`, `:4426`, `:4662`. Keep those
    wrappers on the constants.
11. **`UBQS_SaberService` (`:6025`) and `UBQS_Memory` (`:6072`).** Other mods read them:
    `RSQ_TopHPBars` reads `"gripped"` and `RS_Score` reads `"blocked"`, `"perfect"`, `"pulled"`,
    `"thrown.leg"` (`HANDOFF_polish`, 10-27). Cross-mod API. Do not rename.
12. **`UBQS_Upg` (`:6187`) → `RSQ_MeleeUpgradeService`.** The Gun Bonsai patch's melee upgrades
    key off it by name.
13. **The `ubqs_dual_out` / `ubqs_dual_in` / `rsqa_alt` netevents (`:5584`, `:5550`, `:6271`).**
    Not in `KEYCONF`; they come from outside the mod. No `KEYCONF` cleanup may assume the four
    aliases are the only senders.

---

## 8. WHAT THIS REPORT DOES NOT CLAIM

- **No engine change is proposed.** Every engine line cited was read to verify a claim.
- **The 10-29 psprite flip is treated as correct**, per the settled design facts. The one thing it
  costs — the off-hand blade vanishing under SteamVR's `weaponStabilised` — is already
  `03_SABER_AND_SAW` §2.6's finding and its fix is `ubqs_world_saber`, which this report keeps.
- **Nothing was written into the mod's tree.**
- **One thing I could not check:** whether the generated `ubqs_zscript.txt` carries different
  defaults from the committed `CVARINFO`. The generator input is not in the repository (§4), so
  every default above is the one a player actually gets, read from the shipped `CVARINFO`.
