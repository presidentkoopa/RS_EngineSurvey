# 11 — HUD models owned by a non-weapon actor

**Engine:** `qzd17.3` of `github.com/emawind84/gzdoom` (QuestZDoom PC edition), read-only.
All `path:line` citations are repo-relative at that tag.

**The one question this report answers:** is a HUD model whose owning actor is an ordinary
`Inventory` item — not a `Weapon`, not the ready weapon, not the offhand weapon — drawn at all?

This report corrects `00_CAPABILITIES.md` §2 and `99_OVERLOOKED.md` §1 and §2. The exact
replacement text is in the last section. It takes as given, and did not re-derive, the three
findings the audit already settled: the hand comes from the owner; the targeter band is
persistent but hand-matrixed; `psp.x`/`psp.y` become rotations.

---

## 1. Verdict

**BUILDABLE, with one hard structural constraint that the rig almost certainly tripped over.** A
HUD model owned by a plain `Inventory` item *does* render: nothing in the model path tests for
`Weapon`, for `ReadyWeapon` or for `OffhandWeapon` as a precondition — `FindModelFrame` is handed
`psp->Caller` directly (`src/rendering/hwrenderer/scene/hw_weapon.cpp:922`), MODELDEF accepts any
actor class with defaults (`src/r_data/models.cpp:750-756`), a plain `Inventory` subclass's states
are already flagged `SUF_OVERLAY` (`wadsrc/static/zscript/actors/actor.zs:458`), and the psprite
cull explicitly *keeps* a non-weapon caller as long as the player owns the item
(`wadsrc/static/zscript/actors/player/player.zs:559-563`). It even draws with no weapon in hand at
all.

**The constraint is the layer id.** Both psprite preparation loops stop dead below
`PSP_TARGETCENTER` (`hw_weapon.cpp:830`, `:915`), and the separate pass that *does* cover the
targeter band hardcodes `hudsprite.mframe = nullptr` (`hw_weapon.cpp:1058`), while the only call
site that renders a model is gated on `huds->mframe` being non-null (`hw_weapon.cpp:101-108`). So
**no layer at or above `PSP_TARGETCENTER` can ever draw a model** — it is sprite-only, by
construction, in three independent places. `IsHUDModelForPlayerAvailable` stops at the same
boundary (`src/r_data/models.cpp:1237`), so such a layer does not even switch the renderer into
its HUD-model pass.

**The second trap is the caller, and the ZScript surface makes it easy to hit.** The script
binding `PlayerInfo.GetPSprite(int id)` takes **no** caller argument
(`wadsrc/static/zscript/actors/player/player.zs:3142`), and `player_t::GetPSprite` with a null
`newcaller` on an ordinary layer assigns **`ReadyWeapon`** (`src/playsim/p_pspr.cpp:330-332`).
A mod that creates its layer with `GetPSprite` therefore silently gets a weapon-owned layer: the
MODELDEF keyed on the item's class never matches, and the layer dies on the next weapon switch.
Only `SetPsprite(id, state, pending, newcaller)` (`player.zs:3140`, thunk `p_pspr.cpp:298-308`)
can set a non-weapon owner.

**Which hand:** the main hand, unconditionally. `RenderHUDModel` computes
`int hand = psp->GetCaller() == playermo->player->OffhandWeapon ? 1 : 0;`
(`src/r_data/models.cpp:221`) — a ternary, not a lookup, so anything that is not the offhand
weapon is hand 0. There is no bail-out and no third case. The 2D sprite path has the identical
ternary (`hw_weapon.cpp:256`).

---

## 2. What already exists (cited)

### 2.1 The psprite side — how `Caller` is set

| Route | Caller it produces | Citation |
|---|---|---|
| `player.SetPsprite(id, state, pending, newcaller)` | **`newcaller` as passed** | `player.zs:3140` → `p_pspr.cpp:298-308` → `P_SetPsprite` `p_pspr.cpp:287-292` |
| `player.GetPSprite(id)` from ZScript | **`ReadyWeapon`** (no caller param exists) | `player.zs:3142`; `p_pspr.cpp:330-332` |
| layer `>= PSP_TARGETCENTER`, no caller | `PowerTargeter` item, else **null → returns nullptr** | `p_pspr.cpp:315-321`, `:334` |
| layer `== PSP_OFFHANDWEAPON`, no caller | `OffhandWeapon` | `p_pspr.cpp:326-328` |
| layer `== PSP_STRIFEHANDS`, no caller | the player pawn | `p_pspr.cpp:322-325` |
| `A_Overlay(layer, state)` | **`stateowner`** — the actor whose state is running | `p_pspr.cpp:1225-1243`, `P_CreatePsprite` at `:1240` |

`GetPSprite` bails out entirely when the resolved caller is null — `if (newcaller == nullptr ||
layer == PSP_CALLERID) return nullptr;` (`p_pspr.cpp:334`). That is the mechanism behind the
`PowerTargeter` trap in `99_OVERLOOKED.md` §1: without the powerup you do not get a layer with a
null caller, you get **no layer at all**.

`pspr->Caller = newcaller` is re-asserted on every `GetPSprite` call (`p_pspr.cpp:347`), so any
other code that touches your layer id without passing a caller will **steal the layer** and
re-point it at `ReadyWeapon`.

### 2.2 What the render loops actually test

Three loops, and they do not agree:

```
hw_weapon.cpp:830   PreparePlayerSprites2D   psp->GetID() < PSP_TARGETCENTER   sprites only (:841 "if (smf) continue")
hw_weapon.cpp:915   PreparePlayerSprites3D   psp->GetID() < PSP_TARGETCENTER   models only  (:925 "if (!smf) continue")
hw_weapon.cpp:1068  PrepareTargeterSprites   FindPSprite(PSP_TARGETCENTER) then every Next
```

- Neither of the first two tests `ReadyWeapon` or `OffhandWeapon` as a *requirement*. The only
  caller comparison is the stabilisation skip, `weaponStabilised && psp->GetCaller() ==
  player->OffhandWeapon` (`:832`, `:917`) — which a non-weapon caller can never satisfy, so a
  non-weapon-owned layer is immune to the two-hand trap. This confirms and extends
  `00_CAPABILITIES.md` gotcha 10a.
- Both require only a non-null state (`:836`, `:921`). `FindModelFrame(psp->Caller, …)` tolerates
  a null caller (`models.cpp:1219-1223`, `if(!thing) return nullptr`), so a null-caller layer
  simply never gets a model.
- `PrepareTargeterSprites` sets **`hudsprite.mframe = nullptr`** at `hw_weapon.cpp:1058` before
  the loop and never reassigns it. `DrawPSprite` only reaches `RenderHUDModel` inside
  `if (huds->mframe)` (`hw_weapon.cpp:101-108`). Hence: targeter band = sprite only, always.
- **A second, separate trap in that pass:** it starts at `player->FindPSprite(PSP_TARGETCENTER)`
  (`:1068`), and `FindPSprite` returns null unless a layer with that **exact** id exists
  (`p_pspr.cpp:241-254`). A layer parked at `PSP_TARGETCENTER+1` or `+2` is therefore drawn **only
  if something else is also holding `PSP_TARGETCENTER`** — normally the crosshair, which the same
  loop skips when `CrosshairImage` is non-null (`:1070`). The persistent band is real, but it is
  reachable only through a layer you do not own.

`IsHUDModelForPlayerAvailable` (`models.cpp:1232-1241`) decides whether the renderer runs its
depth-cleared HUD-model block at all (`src/rendering/hwrenderer/scene/hw_drawinfo.cpp:943-952`,
`:966`, `:1072`) and uses the same `< PSP_TARGETCENTER` bound. A model in the targeter band is
invisible to that test too.

### 2.3 The model side — the hand, and what happens to a non-weapon owner

```
src/r_data/models.cpp:220   VSMatrix objectToWorldMatrix = renderer->GetViewToWorldMatrix();
src/r_data/models.cpp:221   int hand = psp->GetCaller() == playermo->player->OffhandWeapon ? 1 : 0;
src/r_data/models.cpp:223   if (vrmode->GetWeaponTransform(&objectToWorldMatrix, hand)) { … }
src/r_data/models.cpp:229   else if (vrmode->IsVR()) { … world-anchored fallback … }
```

It does not pick a default, does not bail out, and has no third branch: **anything that is not
`OffhandWeapon` is hand 0, the main hand.** `GetWeaponTransform` itself needs no weapon —
`AActor *weap = … ; autoReverse = weap == nullptr || !(…WIF_NO_AUTO_REVERSE)`
(`src/common/rendering/hwrenderer/data/hw_vrmodes.cpp:455-473`) handles a null weapon, and
`GetHandTransform` reads only the controller globals and the viewpoint
(`src/rendering/gl/stereo3d/gl_openvr.cpp:1273-1301`; OpenXR parallel at
`src/common/rendering/hwrenderer/data/hw_openxrdevice.cpp:431-470`). **So a HUD model can draw on
the main hand with no weapon selected at all.** VR-path-only: with no VR mode the base
`GetHandTransform` returns false (`hw_vrmodes.h:89`), both branches fail, and the model is drawn
in view space — `hw_vrmodes.cpp:438-442` is the same fallback that collapses both hand poses onto
the viewpoint.

One VR-path handedness wrinkle worth knowing: the mirror is applied as
`if (!hand && autoReverse) out->scale(-1,1,1)` on the *remapped* index
(`hw_vrmodes.cpp:461-470`). Under the default right-handed `vr_control_scheme < 10` the main hand
remaps to 1, so no mirror. Under a left-handed scheme the main hand remaps to 0 and the model
**is** X-mirrored, with `autoReverse` read from `ReadyWeapon` — i.e. from an actor your item has
nothing to do with, and `true` when no weapon is held. A non-weapon-owned model cannot opt out,
because `WIF_NO_AUTO_REVERSE` is a weapon flag.

### 2.4 The MODELDEF side — no weapon-only gate

`ParseModelDefLump` resolves the class with `PClass::FindClass(sc.String)` and accepts it if it
has defaults (`models.cpp:750-756`); the only error is "Unknown actor type". `hasmodel` is set on
that class's defaults (`models.cpp:1029`, `:1096`). Lookup is by `(type, sprite, frame)` only
(`FindModelFrameRaw`, `models.cpp:1149-1168`). **There is no weapon-only gate anywhere in MODELDEF
parsing or lookup.** `model MyWristPanel { … }` is as valid as `model MyPistol { … }`.

The consequence to design around: the match is on the **item's own class** and on the **sprite and
frame of the state you set on the layer**. A MODELDEF keyed on the item but a state whose sprite
belongs to something else matches nothing, and you fall through to the 2D loop instead.

### 2.5 The culling question — the opposite of the failure

```
wadsrc/static/zscript/actors/player/player.zs:559   if ((pspr.Caller == null ||
player.zs:560        (pspr.Caller is "Inventory" && Inventory(pspr.Caller).Owner != pspr.Owner.mo) ||
player.zs:561        (pspr.Caller is "Weapon" && (pspr.Caller != pspr.Owner.ReadyWeapon && pspr.Caller != pspr.Owner.OffhandWeapon))) ||
player.zs:562       (pspr.ID == PSP_WEAPON && pspr.Caller != pspr.Owner.ReadyWeapon) ||
player.zs:563       (pspr.ID == PSP_OFFHANDWEAPON && pspr.Caller != pspr.Owner.OffhandWeapon))
player.zs:565      pspr.Destroy();
```

Read it exactly. The `is "Weapon"` clause at `:561` — the one that destroys a layer belonging to a
weapon that is not currently selected — **does not apply to a non-weapon caller**. A plain
`Inventory` caller is tested only by `:560`: it survives as long as `Inventory(Caller).Owner` is
the player's pawn. The id clauses at `:562-563` only bind the two reserved weapon layer ids.

**So this is the reverse of the rig's failure. A non-weapon owner does not get culled — it
persists, across weapon switches, for as long as the player holds the item.** That is the real
mechanism behind the "survives weapon switches" property that `99_OVERLOOKED.md` §1 attributed to
the targeter band. Persistence comes from the **caller**, not from the layer id.

Two things still kill it:
- Losing the item. `Owner != pspr.Owner.mo` → destroyed (`player.zs:560`). An item dropped,
  taken or destroyed takes the panel with it.
- `A_ClearOverlays`. It tests **no caller at all** (`p_pspr.cpp:1245-1286`): every layer in range
  gets `SetState(nullptr)`, which destroys it (`p_pspr.cpp:531-534`). With `safety` it breaks out
  at `PSP_TARGETCENTER` (`:1273-1274`) and skips the four reserved ids (`:1275-1276`); without it,
  `start`/`stop` default to `INT_MIN`/`INT_MAX` (`:1256-1259`) and nothing is spared.
  **This is the one genuine advantage of the targeter band, and it is bought at the price of the
  model.**

### 2.6 Flags and bob — a non-weapon owner inherits the *offhand* weapon's bob

Two creation routes give different flags:

- `DPSprite`'s constructor sets the full flag set **only** for a `Weapon` or `PlayerPawn` caller —
  `if (Caller->IsKindOf(NAME_Weapon) || Caller->IsKindOf(NAME_PlayerPawn))` (`p_pspr.cpp:231-232`).
  For anything else `Flags` stays 0. `A_Overlay` goes straight through `P_CreatePsprite`
  (`p_pspr.cpp:1240`), so an `A_Overlay`-created non-weapon layer has **no** `PSPF_ADDBOB` /
  `PSPF_ADDWEAPON`.
- `GetPSprite` then overrides unconditionally, with no class test: on a fresh layer
  `oldcaller` is null, so `newcaller != oldcaller` and `Flags = PSPF_ADDWEAPON|PSPF_ADDBOB|
  PSPF_CVARFAST|PSPF_POWDOUBLE|PSPF_PIVOTPERCENT` (`p_pspr.cpp:355-365`). So a `SetPsprite`-created
  non-weapon layer **does** bob.

And its bob comes from the wrong place. `GetWeaponPosition3D` picks the reference layer with
`w.weapon = psp->GetCaller() == player->ReadyWeapon ? readyWeaponPsp : offhandWeaponPsp;`
(`hw_weapon.cpp:332`) — another two-way ternary, so a non-weapon caller is handed the **offhand
weapon's** psprite. With `PSPF_ADDWEAPON` set, `BobWeapon3D` adds that layer's `wx`/`wy`
(`hw_weapon.cpp:430-434`); the pivot is read from the offhand weapon's `BobPivot3D`, guarded by
`weaponActor->IsKindOf(NAME_Weapon)` (`hw_weapon.cpp:347-358`); and the `ModifyBobLayer3D` /
`ModifyBobPivotLayer3D` virtuals are resolved on the **offhand weapon's class** while being passed
**your** layer id (`hw_weapon.cpp:969-1000`). With no offhand weapon, `w.weapon` is null and
`wx = wy = 0`, pivot zero (`hw_weapon.cpp:360-365`), but `weap.translation`/`rotation` still carry
`P_BobWeapon3D`'s main-weapon bob (`p_pspr.cpp:703-730`).

**Clear `psp.bAddBob` and `psp.bAddWeapon` on a wrist panel** or it will sway with whatever is in
the other hand.

### 2.7 Position and the 8° you did not ask for

Position must come from the MODELDEF, as the audit established. The exact coupling is:

```
hw_weapon.cpp:108   RenderHUDModel(&renderer, huds->weapon, huds->translation,
                      huds->rotation + FVector3(huds->mx / 4., (huds->my - WEAPONTOP) / -4., 0),
                      huds->pivot, huds->mframe);
```

`WEAPONTOP` is `32.` (`src/playsim/p_pspr.h:41`). `huds->my` is the interpolated `psp->y`
(`hw_weapon.cpp:405-407`), and a freshly constructed psprite has `y = 0` (`p_pspr.cpp:180`). So a
new layer is pitched `(0 - 32) / -4 = +8°` until you write `psp.y = 32`. The fixed pre-offsets in
`RenderHUDModel` are `scale(0.01)` then `translate(0, 5, 30)` (`models.cpp:225-227`), after which
`smf->xoffset/yoffset/zoffset` and `vr_3dweaponOffsetX/Y/Z` apply (`models.cpp:246-251`).

---

## 3. Building blocks

| Block | What it gives | Citation |
|---|---|---|
| `SetPsprite(id, state, pending, newcaller)` | the only script route to a non-weapon owner | `player.zs:3140`, `p_pspr.cpp:298-308` |
| `Actor` default state usage `SUF_ACTOR\|SUF_OVERLAY` | a plain `Inventory` item's states are legal overlay states | `actor.zs:458`; checked at `p_pspr.cpp:537-551` |
| MODELDEF on any actor class | the item can carry a HUD model | `models.cpp:750-756`, `:1029` |
| `psprites` cull keeps owned non-weapon callers | persistence across weapon switches | `player.zs:559-563` |
| `weaponStabilised` skip keyed on the caller | a non-weapon-owned layer is immune to the two-hand trap | `hw_weapon.cpp:832`, `:917` |
| Hand matrix needs no weapon | draws with nothing in hand | `hw_vrmodes.cpp:455-473`, `gl_openvr.cpp:1273-1301` |
| `psp.bAddBob` / `psp.bAddWeapon` writable from script | switch off the inherited offhand bob | `p_pspr.cpp:162-163`; `player.zs:2972-2973` |

**Not available:** per-layer 3D translation from script (`psp.x`/`psp.y` are rotations,
`hw_weapon.cpp:108`); a model in the targeter band (§2.2); a way to put a non-weapon-owned model on
the **off** hand (§5).

---

## 4. Mechanic designs

### 4.1 The wrist panel — working recipe (BUILDABLE / S)

The whole recipe, as the brief asks for it:

1. **Owner class:** a plain `Inventory` subclass — *not* a `Weapon`, *not* a `StateProvider`
   (`CustomInventory` is a `StateProvider`, `wadsrc/static/zscript/actors/inventory/stateprovider.zs:528`,
   whose `DefaultStateUsage` adds `SUF_ITEM` but keeps `SUF_OVERLAY`, so it also works). Give it
   `+INVENTORY.UNDROPPABLE` and keep it in the player's inventory for life: the cull at
   `player.zs:560` is an ownership test. Do **not** put its display states in a
   `States(Actor)`-qualified block, which would strip `SUF_OVERLAY` and make `SetState` destroy the
   layer with a console message (`p_pspr.cpp:537-543`).
2. **Layer id:** any free integer **strictly below `PSP_TARGETCENTER`**. Use the gap above
   `PSP_OFFHANDWEAPON`, e.g. `1000100`, so a weapon's own overlay ranges and
   `A_ClearOverlays(start, stop)` calls aimed at the 1…1000 band miss it.
3. **Creation — must pass the owner:**
   `player.SetPsprite(PANEL_LAYER, panelItem.ResolveState("Display"), false, panelItem);`
   Never `GetPSprite(PANEL_LAYER)`: it assigns `ReadyWeapon` (`p_pspr.cpp:330-332`).
   Re-assert it each tic from the item's `DoEffect` if `player.FindPSprite(PANEL_LAYER)` is null,
   so it comes back after any `A_ClearOverlays`.
4. **Tidy up after creation:**
   `let psp = player.FindPSprite(PANEL_LAYER); psp.bAddBob = false; psp.bAddWeapon = false;
   psp.y = 32;` — the first two drop the inherited offhand bob (§2.6), the third removes the 8°
   pitch (§2.7).
5. **MODELDEF:** `model MyWristPanelItem { path "models"; model 0 "panel.md3";
   skin 0 "panel.png"; scale …; frameindex PANL A 0 0; offset …; }` — keyed on the **item's**
   class, with a `frameindex` whose sprite and frame are exactly those of the `Display` state. All
   position comes from `offset`/`angleoffset`/`pitchoffset`/`rolloffset`
   (`models.cpp:246-251`, `:268-271`).
6. **Hand:** the **main** hand, always (`models.cpp:221`). On the main hand there is a weapon model
   too, so offset the panel back along the forearm rather than placing it at the grip.
7. **Content:** the readout itself is the model's skin or a per-frame sprite set, selected by
   calling `SetPsprite` again with a different state. There is no script route to draw text onto a
   HUD model.

**Label: BUILDABLE / S.** Every piece is shipped. VR-path-only for the hand matrix; on a non-VR
build the panel renders in view space (§2.3).

### 4.2 The same panel as a sprite in the targeter band (HACK / S)

If surviving a third-party `A_ClearOverlays(0, 0, true)` matters more than being a model, put a
**sprite** layer at `PSP_TARGETCENTER + 1` with the item as caller. It survives the safety break
(`p_pspr.cpp:1273-1274`) and gets `PSPF_CVARFAST|PSPF_POWDOUBLE` (`p_pspr.cpp:360-362`). It is
still on the main hand (`hw_weapon.cpp:256`) and will **not** draw unless something holds
`PSP_TARGETCENTER` itself (`:1068` + `p_pspr.cpp:241-254`) — in practice the crosshair, so this
depends on a layer you do not control. HACK, and the model is off the table.

### 4.3 Belt-and-braces: both (BUILDABLE / S)

Model below the band for the look, sprite in the band as the survivor that re-creates the model
layer from the item's `DoEffect`. Cheap, and it removes the only real weakness of 4.1.

---

## 5. Walls

1. **A model in the targeter band is a WALL.** Three independent gates:
   `hw_weapon.cpp:1058` (`mframe = nullptr`), `hw_weapon.cpp:830` and `:915` (loop bound),
   `models.cpp:1237` (same bound in `IsHUDModelForPlayerAvailable`). The missing engine piece, if
   it ever matters, is a model-capable targeter pass — i.e. `PrepareTargeterSprites` doing the
   `FindModelFrame` lookup the other two loops do and the pass bounds extended past
   `PSP_TARGETCENTER`. **Say WALL first; this is not the primary answer, §4.1 is.**
2. **A non-weapon-owned model on the OFF hand is a WALL.** The hand is a two-way ternary against
   `OffhandWeapon` (`models.cpp:221`; sprites the same at `hw_weapon.cpp:256`). There is no field,
   flag or layer-id convention that selects hand 1 for a non-weapon caller. Missing piece: a
   per-psprite hand override, or widening that ternary to read a hand from the layer. Today the
   only way onto the off hand is to be the offhand `Weapon` — which re-exposes the whole
   `weaponStabilised` trap (`00_CAPABILITIES.md` gotcha 10a).
3. **Script-driven 3D placement is a WALL** — already established, `hw_weapon.cpp:108`. Position is
   MODELDEF-static; the only per-tic degrees of freedom a mod has are the two rotations smuggled
   through `psp.x`/`psp.y`.
4. **Left-handed mirroring cannot be opted out of** for a non-weapon owner, because `autoReverse`
   is read from `ReadyWeapon`'s `WIF_NO_AUTO_REVERSE` (`hw_vrmodes.cpp:458-470`). Cosmetic, but it
   will flip a panel's lettering for left-handed players.

---

## 6. Smallest prototype

**One launch. Two cheap tests, in this order.**

**Test A — is it the layer id?** Keep everything else identical and move the layer from whatever
the rig used to `1000100` (below `PSP_TARGETCENTER`). If the model appears, the cause was
`hw_weapon.cpp:1058` / the loop bound at `:915`. This is the single cheapest confirming test and
the one to run first.

**Test B — is it the caller?** From the item's `DoEffect`, each tic:

```
let psp = player.FindPSprite(PANEL_LAYER);
Console.Printf("panel: psp=%s caller=%s ready=%s", psp ? "yes" : "NULL",
    psp && psp.Caller ? psp.Caller.GetClassName() : "none",
    player.ReadyWeapon ? player.ReadyWeapon.GetClassName() : "none");
```

Three readings, three diagnoses: `psp=NULL` → `GetPSprite` returned null because the resolved
caller was null (`p_pspr.cpp:334`, the `PowerTargeter` path); `caller=` the **weapon's** class →
created via `GetPSprite`, caller defaulted to `ReadyWeapon` (`p_pspr.cpp:330-332`), so the MODELDEF
on the item never matches; `caller=` the item and still nothing drawn → the MODELDEF key is wrong,
so re-check that the `frameindex` sprite and frame match the state actually set (`models.cpp:1157-1165`).

A sprite-only fallback state on the same layer separates "no layer" from "layer but no model" in
one glance: if the sprite draws and the model does not, the MODELDEF lookup failed
(`hw_weapon.cpp:841` sent it to the 2D loop); if nothing draws at all, there is no layer.

---

## 7. Open questions

1. **Is a `CustomInventory` safe as the owner?** Its `DefaultStateUsage` is
   `SUF_ACTOR|SUF_OVERLAY|SUF_ITEM` (`stateprovider.zs:528`), so overlay states are legal, and
   `is "Inventory"` at `player.zs:560` covers it. Not tested here: whether its `Use` state chain
   fighting over the same psprite ids causes interference. A plain `Inventory` avoids the question.
2. **Does anything in the shipped `wadsrc` ZScript call `A_ClearOverlays` without `safety`?** That
   decides how often §4.3's re-creation path actually fires. Not surveyed — out of scope here.
3. **Dynamic lighting on a non-weapon-owned HUD model.** `hw_GetDynModelLight(playermo, …)` is
   called for the model path (`hw_weapon.cpp:987-998`) with the player pawn, not the caller, so it
   should behave — but it is untested on this owner shape.
4. **Quest/OpenXR parity.** Every hand-matrix citation here has an OpenXR twin
   (`hw_openxrdevice.cpp:431-470`), but the file is not compiled on PC
   (`00_CAPABILITIES.md` §0), so the Quest behaviour of §4.1 is inferred from the parallel code,
   not observed.

---

## Edits this report requires elsewhere

Three edits. Each quotes the current text exactly and gives its replacement. **Not applied by this
report.**

### Edit 1 — `00_CAPABILITIES.md` §2.3 "Models"

**Current text (lines 344-349), wrong by omission — it does not say that the owner decides the
hand, that the owner need not be a weapon, or that the targeter band cannot carry a model:**

> MODELDEF attaches models to weapon states as normal. The per-frame hand pose means a model
> drawn on the offhand follows the real controller without any script work. Driving a model's
> position or rotation *per tic from script* is where it gets interesting, and the topic reports
> that need it (vehicles, mechs, shields) each work out their own route.

**Replacement:**

> MODELDEF attaches models to actor classes, and **not only to weapons** — the parser accepts any
> actor class with defaults and the only error is "Unknown actor type"
> (`src/r_data/models.cpp:750-756`). So a plain `Inventory` item can own a HUD model, and a plain
> `Inventory` item's states are already legal overlay states, because `Actor`'s
> `DefaultStateUsage` is `SUF_ACTOR|SUF_OVERLAY` (`actor.zs:458`).
>
> **The hand is decided by the owner, as a two-way ternary against the offhand weapon:**
> `int hand = psp->GetCaller() == playermo->player->OffhandWeapon ? 1 : 0;`
> (`src/r_data/models.cpp:221`; the 2D sprite path is identical at
> `src/rendering/hwrenderer/scene/hw_weapon.cpp:256`). There is no third case: **anything that is
> not the offhand weapon rides the main hand**, including a non-weapon owner. Putting a
> non-weapon-owned model on the off hand is a WALL. The hand matrix itself needs no weapon
> (`hw_vrmodes.cpp:455-473`), so a HUD model draws with nothing in hand at all.
>
> **Two hard constraints before you build one:**
> - **A layer at or above `PSP_TARGETCENTER` cannot draw a model.** Both preparation loops stop
>   below it (`hw_weapon.cpp:830`, `:915`), the targeter pass hardcodes `mframe = nullptr`
>   (`hw_weapon.cpp:1058`), and the model call site is gated on `mframe` (`hw_weapon.cpp:101-108`).
> - **`PlayerInfo.GetPSprite(int id)` takes no caller and defaults it to `ReadyWeapon`**
>   (`player.zs:3142`; `p_pspr.cpp:330-332`). Use
>   `SetPsprite(id, state, pending, newcaller)` (`player.zs:3140`) or the layer is weapon-owned and
>   your MODELDEF never matches.
>
> Driving a model's position or rotation *per tic from script* is where it gets interesting —
> `psp.x`/`psp.y` become **rotations**, not translations (`hw_weapon.cpp:108`), so position comes
> from the MODELDEF — and the topic reports that need it (vehicles, mechs, shields) each work out
> their own route. Full trace in `11_HUD_MODELS.md`.

### Edit 2 — `99_OVERLOOKED.md` §1, the "What it could become" paragraph

**Current text (lines 24-26), wrong: it offers the band as the home for persistent hand-mounted
instruments, including "a hand model", which that band cannot draw:**

> **What it could become:** persistent **hand-mounted instruments** — a wrist panel, a permanent
> gauge, a held tool, a hand model that survives weapon switches and overlay clears without a
> custom statusbar.

**Replacement:**

> **What it could become — but read the correction below first:** persistent hand-mounted
> **sprite** instruments. **The band cannot draw a model**, and it is not where persistence comes
> from.
>
> **Correction (11_HUD_MODELS).** Two things were wrong here.
>
> 1. **No layer at or above `PSP_TARGETCENTER` can carry a HUD model.** `PrepareTargeterSprites`
>    hardcodes `hudsprite.mframe = nullptr` (`hw_weapon.cpp:1058`) and the only call site that
>    renders a model is gated on `huds->mframe` (`hw_weapon.cpp:101-108`); the two loops that *do*
>    look up models stop below the band (`hw_weapon.cpp:830`, `:915`), as does
>    `IsHUDModelForPlayerAvailable` (`src/r_data/models.cpp:1237`). So "a hand model" in this band
>    is a **WALL**. A wrist panel built as a *model* must live below `PSP_TARGETCENTER`.
> 2. **Surviving weapon switches does not need this band at all — it comes from the CALLER.** The
>    psprite cull destroys a layer whose caller is a `Weapon` that is not currently selected, but
>    that clause does not apply to a non-weapon caller: a plain `Inventory` caller is tested only
>    for ownership, `Inventory(pspr.Caller).Owner != pspr.Owner.mo`
>    (`wadsrc/static/zscript/actors/player/player.zs:559-563`). **A layer owned by a held
>    non-weapon `Inventory` item persists across weapon switches at any layer id**, including the
>    model-capable range below the band.
>
> **What the band is still genuinely good for:** surviving `A_ClearOverlays(…, safety)`
> (`p_pspr.cpp:1273-1274`), which nothing below it does — `A_ClearOverlays` tests no caller at all
> (`p_pspr.cpp:1245-1286`). That is the whole trade: sprite in the band and survive overlay
> clears, or model below the band and re-create it when something clears you. The sound design is
> a model layer below the band owned by an undroppable `Inventory` item, re-asserted from that
> item's `DoEffect`. Recipe in `11_HUD_MODELS.md` §4.1.

### Edit 3 — `99_OVERLOOKED.md` §1, the trap paragraph

**Current text (lines 36-38), half right — the caller does default to `PowerTargeter`, but the
consequence is not "a null caller and gets culled", and the second, bigger trap in that pass is
missing:**

> **The trap:** a layer created at or above `PSP_TARGETCENTER` defaults its `Caller` to the
> player's `PowerTargeter` inventory item (`p_pspr.cpp:310-318`). Without that powerup the layer
> has a null caller and gets culled. Pass `newcaller` explicitly when creating it.

**Replacement:**

> **Two traps, and the first one is worse than stated.** A layer created at or above
> `PSP_TARGETCENTER` without an explicit caller defaults it to the player's `PowerTargeter` item
> (`p_pspr.cpp:315-321`). Without that powerup you do not get a null-caller layer that is later
> culled — `GetPSprite` **returns `nullptr` and creates nothing** (`p_pspr.cpp:334`). Pass
> `newcaller` explicitly, which from ZScript means
> `SetPsprite(id, state, pending, newcaller)` (`player.zs:3140`): the `GetPSprite(int id)` binding
> has **no caller parameter at all** (`player.zs:3142`) and for an ordinary layer defaults the
> caller to `ReadyWeapon` (`p_pspr.cpp:330-332`), which both ties the layer to the current weapon
> and makes a MODELDEF keyed on your own class fail to match.
>
> **Second trap:** the targeter pass starts at `player->FindPSprite(PSP_TARGETCENTER)`
> (`hw_weapon.cpp:1068`), and `FindPSprite` matches that id **exactly** (`p_pspr.cpp:241-254`). A
> layer parked at `PSP_TARGETCENTER+1` or `+2` is therefore drawn only while something else holds
> `PSP_TARGETCENTER` — normally the crosshair, which the same loop then skips when
> `CrosshairImage` is non-null (`hw_weapon.cpp:1070`). The persistent band is real, but reaching it
> depends on a layer you do not own.

### Edit 4 — `99_OVERLOOKED.md` §2, "What it could become"

**Current text (lines 130-132), incomplete — it invites exactly the mistake the rig made:**

> **What it could become:** per-hand stacks of independently animated, independently positioned
> overlay layers — a held item, a forearm panel, a glove, a muzzle device, a reload prop — all
> without fighting the weapon's own two layers for space.

**Replacement:**

> **What it could become:** per-hand stacks of independently animated overlay layers — a held item,
> a forearm panel, a glove, a muzzle device, a reload prop — without fighting the weapon's own two
> layers for space.
>
> **Three qualifications, all load-bearing (11_HUD_MODELS):**
> - **"Independently positioned" is wrong for models.** `psp.x`/`psp.y` become **rotations** on a
>   HUD model (`hw_weapon.cpp:108`); 3D position comes from the MODELDEF only.
> - **"Per-hand" is not a property of the layer id.** Both the model path (`models.cpp:221`) and the
>   sprite path (`hw_weapon.cpp:256`) choose the hand by comparing the layer's **caller** to
>   `OffhandWeapon`. A layer id does not select a hand, and a non-weapon owner always lands on the
>   **main** hand.
> - **Keep your ids strictly below `PSP_TARGETCENTER`** if the layer is to carry a model
>   (`hw_weapon.cpp:830`, `:915`, `:1058`). The roughly one million free indices above
>   `PSP_OFFHANDWEAPON` are the right place: model-capable, and clear of weapon overlay ranges and
>   of `A_ClearOverlays` calls aimed at the 1…1000 band.
