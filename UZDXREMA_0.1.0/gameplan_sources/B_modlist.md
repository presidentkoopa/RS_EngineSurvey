# B: the mod set and playbook Phase 10 (helper-actor explosion) on UZDXREMA

Scope: the per-tic cost that the owner's 57-file mod set adds, family by family. It covers his own `RSQTHP_Bar`, mk-champions, the
"Legion of Bones" monsters, Universal Entropy's `Shifter`, mk-crits, the UA ambience, and a quick audit of every `RSQ_*` mod
before the uplift. For each, it says what the fork already has and which engine mechanism removes the whole class of cost.

Sources: [F] `/home/claude/uzdxrema` @ e84623f, [FR] `/home/claude/uzdxrema_renderer` @ abb19af, [P] playbook 20 Phase 10,
[M] 10_MEASUREMENTS. [MOD] = the owner's unpacked rotation, `/home/claude/mods_x/<folder>/<path>:lines` (read only).
`RSQ_VisualRewards` in mods_x is byte-identical to the source tree `presidentkoopa/rsq_ubqseries/VisualRewards/pk3` (checked
with `diff -q` on bar.zs and handler.zs).

Per-thinker interpolation-reset share used throughout (**Inference**): 9.02 ms/tic [M] §2 ÷ 53,116 thinkers [M] §1 =
**0.17 µs per thinker removed from the 32..127 lists**.

Owner direction: native engine mechanisms that remove whole classes of cost come first. Mod patches are listed as stopgaps,
or as the port onto the new API. Every sim change must stay deterministic for four-player lockstep.

---

## Delta table

| ID | Item | Fork status | Where in fork | What is left | Done when (measurable) | Don't touch | Netplay/determinism impact | Est. saving on modlist (Inference, with derivation) |
|---|---|---|---|---|---|---|---|---|
| P10.0 | `TickBehaviors` per-tic heap arrays | Not started (stock code unchanged) | [F] src/playsim/p_mobj.cpp:712-751, call :4827 | Inline-capacity stack arrays (re-entrant). Needed before data-only Behaviors replace inventory helpers. | 0 allocations/tic from `TickBehaviors` with 5k data-only behaviors (0.5 alloc counter) | Behavior semantics, script API | None (same calls, same order) | ~0 today (no mod in the set uses Behaviors); enables P10.6 |
| P10.1 | Native per-actor overlays (HP bars, icons) | **Partial: a different mechanism exists.** Attached billboards follow an actor at render rate, but they are unculled, have O(n) handles and no viewer mask. | [F] wadsrc/static/zscript/doombase.zs:1100-1131 (`AttachBillboard`); src/scripting/vmthunks.cpp:3109-3117 (`FindBillboardByID` linear), :3207-3220; src/p_tick.cpp:267-297 (`TickBillboards`), :762; src/rendering/hwrenderer/scene/hw_drawinfo.cpp:3546-3663 (`DispatchBillboards`, no frustum or visibility cull, `PointInSector` per billboard); src/p_saveg.cpp:1142 (serialized) | Option A: harden billboards (see F-MOD-1). Option B (recommended for the RSQ uplift): playbook `ActorOverlay` slots, emitted from `RenderThings`. | `RSQTHP_Bar` thinkers = 0; bars drawn only for actors in visible subsectors; 0 script per bar per tic | Avatar/model draw path (render lane rule) | Overlay state is play state and serialized. Per-viewer bits (viewer mask, far size) are resolved at render time from `consoleplayer`. | RSQTHP 16.46 + 3,622×0.17 µs = **17.1 ms → ~0**; champion_Icon 4.79 + 0.28 = **5.1 ms → 0** |
| P10.1i | Interim overlay drawing through the sprite path | Not started | [F] hw_bsp.cpp:914-960 `RenderThings` (insertion point after the sprite `Process`) | Emit 1-3 `HWSprite` quads per active slot (frame, fill, icon). | Visual match with `RSQTHP_Bar` on the rig (bridge `boottest` frames) | Model draw | Render only | Enables P10.1 before Phase 7 |
| P10.2 | Attachments (follow the master without ticking) | Not started for actors. Render-only attachment exists as billboards (P10.1 row). | none | `AttachTo(master, ofs, flags)` plus one native pass after `RunThinkers` ([F] p_tick.cpp:881). Needed for champion icons and leaves, the aggro visual, BD head hitboxes and `RS_ScoreHeadSensor`. | 0 VM calls per attached helper per tic | Hitbox collision semantics (other agent's BD22 area) | Deterministic (creation order, after the thinkers) | champion_Leaves 2.18 + 0.17; AggroVisual part of 4.3; plus BD hitboxes (other report) |
| P10.3 | Tickless actors with wake-up | **Partial: GZSelaco thinker sleep ported.** | [F] src/playsim/dthinker.cpp:97-140, :161-162, :1234-1262; statnums.h:56-59 (`STAT_SLEEP` 60, `STAT_SLEEP_FOREVER` 61); ZScript doombase.zs:284-292 | No automatic wake on `SetState`/damage. Sleepers sit in 60/61, inside the default iterator range ([F] dthinker.cpp:1617-1631), so the interpolation reset still walks them. For helpers that never move, park below 32 (they still draw). | A parked helper costs 0 µs/tic in `RunThinkers` and in the reset | Sleep ordering (GZSelaco sorted insert left out on purpose, dthinker.cpp:97-99) | Deterministic and saved ([F] dthinker.cpp:1082-1088) | Used by the ports below (IA_Sound, controllers with `eftic`>1) |
| P10.4 | Throttled statnums | Not started | none (only per-thinker `Sleep(n)`) | `Level.SetStatTickInterval` in STAT_USER 70..90 ([F] statnums.h:61-62) | Ambience/AI directors tick 1/N | — | Deterministic if keyed on `maptime` plus list order | Small for this set (IA_Sound is better parked) |
| P10.5 | Per-class lists and transient iterators | Not started | none | Native per-class lists. `BlockThingsIterator` for "near me" scans. | No `ThinkerIterator.Create("Actor")` in any per-tic path of the set | — | Order must be the spawn serial | mk-champions Pink: ~1.7 full 53k walks per tic (derivation in Findings §2.4); SlowMo `Gather` |
| P10.6 | Damage-reactive helpers become events and Behaviors | Not started (stock events exist) | — | crits: two inventory items per monster | 11,332 inventory thinkers → 0 (F-MOD-8 does it engine-side for every such item) | — | Deterministic | 1.57 + 11,332×0.17 = **3.5 ms** |
| P10.7 | "One helper per N actors" (`Shifter`) | Not started | — | Engine: per-actor sound-pitch and tic scale (F-MOD-9). The mod port drops `Shifter`. | 7,686 thinkers → 0 | — | Tic scale is sim state (deterministic). Pitch is output only. | 2.23 + 7,686×0.17 = **3.5 ms** on [FR]. On [F] main, much more: F-MOD-5. |
| P10.8 | Script allocation hygiene | Not started (no per-class alloc counter on main) | — | Measure first: `ThinkerIterator.Create` in head sensors, `String.Format` in `Tint`, `A_RadiusGive` item churn | allocs/tic per class | — | — | in the rows above |
| P10.9 | "Monster replacements at 8× stock cost" | **Re-diagnosed: not Legion of Bones code.** | [MOD] 2.1-Enemies_LegionOfBones_v1.05/decorate/imp.txt:1-61 (plain DECORATE, no Tick) vs [F] p_mobj.cpp:4870-4881 (inventory `DoEffect` loop inside the owner's tick) | The cost is the inventory `DoEffect` chain the mod set hangs on every monster, mostly mk-crits `isAttacking()`. It is booked to the monster's class. | SkeletonImp µs/tic within 2× of DoomImp no-mod (0.31 µs [M] §3) | LoB gameplay | Deterministic patch (cached states) | 0.5-1.0 µs × 5,666 monsters = **2.8-5.7 ms** (spread over all monster classes, ~11.3 ms listed for 4 LoB classes) |
| P10-R1 | `RSQTHP_Bar`, transitional port (no engine change, stock-compatible) | Not started (bars tick; damnums/words already use the parked pattern) | [MOD] RSQ_VisualRewards/zscript/rsqthp/bar.zs:165-362; handler.zs:463-511, 581-603; parked-pattern precedent: rsqthp/damnums/glyphs.zs:12-18, rsqvr/words.zs:29, :326, :385 | Baseline fix, parked bars and parts, one driver at statnum 124, no `consoleplayer` in play | Bars at the Sunder MAP15 start ≈ number of hurt monsters (0 at map start); ≤0.5 µs per live bar per tic | Look and options | **Fixes a latent desync** (F-MOD-10) | 17.1 ms → **≤0.2 ms** at the start; ≤0.5 µs × live bars in a fight |
| P10-R2 | `RSQTHP_Bar` on native overlays (uplift) | Not started | P10.1 | Policy stays script (event-driven). Drawing becomes native. | 0 VM per bar per tic; 0 bar actors | — | Viewer mask at render | → 0 |
| F-MOD-1 | Billboard hardening (makes billboards usable at 10^3-10^4) | Not started | [F] vmthunks.cpp:3109-3117; hw_drawinfo.cpp:3546-3663; hw_sprites.cpp:2876-2905 (always fullbright, translucent) | O(1) id→index map; cull attached billboards by `attachedTo->subsector` `SSRF_PROCESSED` ([F] hw_bsp.cpp:1121-1142, hw_drawinfo.h:88); use `attachedTo->Sector` instead of `PointInSector`; viewer mask; far-scale curve; `SetBillboardAttachOffset` | 7k attached billboards cost < 0.3 ms/eye CPU when 200 are visible | Panels/wheel/hand-locked billboards | Render only, except the setter API (play) | Avoids a +3-4 ms/eye regression if bars move onto billboards as they are today (Findings §1.6) |
| F-MOD-2 | `WorldStep` runs after `RunThinkers` | Done (fork-only) | [F] p_tick.cpp:872 `WorldTick`, :881 `RunThinkers`, :896 `WorldStep`; events.cpp:2109-2117 | Use it as the bar and attachment driver slot on REMA. Stock-compatible mods use a statnum-124 thinker. | — | World clock semantics (only on world steps) | Deterministic | enabler |
| F-MOD-3 | Client-side thinkers off the deterministic path | Done (stock 5.0 + fork clock) | [F] p_tick.cpp:155-204; p_mobj.cpp:5939-5960 `SpawnClientSide`; codegen.cpp:8459-8482 (`crandom`/`cfrandom`) | Ports of purely cosmetic helpers (leaves, aggro rings, sparkles) | sim thinkers −976 leaves, −aggro visuals | — | Must use `cfrandom`; may cull by the local camera | Leaves 2.35 ms sim → client side (cullable) |
| F-MOD-4 | Netplay-safe compat copies of third-party mods | Done (infrastructure) | [F] tools/netplay/compat/README.md:1-13 | Owner-maintained patched copies (`*_REMA.pk3`) for Champions, crits, UE, UA | `tools/netplay/run.sh` IN SYNC with the patched set | Originals untouched | Every peer loads the same copy. Re-record replays. | enabler |
| F-MOD-5 | Per-source sound channel lists | **On [FR] only, missing on [F] main** | [FR] src/common/audio/sound/s_sound.cpp:74, :558, :939, :1089-1102; [F] s_sound.cpp:1073-1085 (walks all channels) | Port to main (or merge via the netplay lane) | `Shifter` ≤ 0.3 µs/call on main | — | Output only | [M] numbers come from `doomxr_rend1113a` (renderer build). On main, `Shifter` cost 174 ms/tic on Abandon MAP13 before the fix ([FR] docs/plans/SIM_AT_MOD_SCALE.md §1). |
| F-MOD-6 | `EventHandler.Find` is O(handlers) | Not started | [F] src/events.cpp:1702-1721 | ~171 handler names in the set's MAPINFOs. Add a class→handler map. | `Find` O(1) | — | None | bar.zs:215 calls it per bar per tic: 3,622 × ~0.4 µs ≈ 1.4 ms (Inference; it vanishes with P10-R1 anyway) |
| F-MOD-7 | Champions makes every monster translucent | Not started (mod) | [MOD] 2.2-Enemies-mk-champions[20210309]/zscript/champions/handler.txt:400-403 | Patch out, or set `STYLE_Normal` unless alpha<1 | Sorted translucent sprite count | — | Render only | GPU/sort, not tic (renderer report) |
| F-MOD-8 | Park owned inventory whose `Tick` is a no-op | Not started (new, engine) | [F] wadsrc/static/zscript/actors/inventory/inventory.zs:113-147 (`Inventory.Tick` is ZScript: one VM call per item per tic) | Park on `AttachToOwner` when tics==-1, DropTime==0 and the class does not override `Tick`; unpark on `SetState`/detach | csh ×2, champion tokens, LD tokens, `RSQTHP_ChampKind`… 0 µs/tic | Item `DoEffect` (runs from the owner, [F] p_mobj.cpp:4870-4881, unchanged) | Same results. List order changes, so re-baseline the replays. | ≥ 11,332 items: 1.57 + 1.93 = **3.5 ms**, more for other no-op items |
| F-MOD-9 | Per-actor `SoundPitchScale` and `StateTicScale` | Not started (new, engine) | (new fields in `AActor`; applied in `S_StartSound` and `SetState`) | UE port: set once at spawn; delete `Shifter` | 7,686 thinkers → 0 | — | Tic scale is sim (deterministic). Pitch is output. | 3.5 ms (P10.7) |
| F-MOD-10 | `consoleplayer` in play scope in the set | Hazard (mods) | [MOD] RSQ_VisualRewards/zscript/rsqthp/bar.zs:226-246, :476-477; handler.zs:515-518, :548-577; 2.4-Enemies_UniversalEntropy/randomise.zs:9, :19-23 | Bars and parts differ per peer. `TraceAims` spawns bars on one peer only, and UE then draws `frandom` twice for every spawned thing (randomise.zs:22-23), which advances the sim RNG on that peer alone. | `run.sh` IN SYNC with Aimed Only on | — | **Desync with `rsqthp_aimed 1`** (Inference from code) | correctness |
| F-MOD-11 | RSQTHP "full health" is `SpawnHealth()` | Bug (mod) | [MOD] RSQ_VisualRewards/zscript/rsqthp/handler.zs:503-507; 2.4-Enemies_UniversalEntropy/randomise.zs:39-44 (`ue_resize 5`: every monster's health ×size, `StartHealth` untouched); [F] p_mobj.cpp:9159-9174 | Undamaged monsters all get a bar (and a wrong partial or overheal look). Use an undamaged baseline. | 0 bars at a fresh map start | Overheal strip for turned legendaries | Deterministic | Most of the 3,622 bars (Findings §1.2) |

---

## Findings

### 1. `RSQTHP_Bar` (owner's mod): 16.46 ms, 3,622 thinkers [M] §3

#### 1.1 What it is

One sprite actor per monster with a bar, plus up to nine `RSQTHP_Part` sprite actors: the overheal strip, markers, status,
boss crown, DOWN mark, tier gem, champion icon, mutation icon and damage chunk. These are listed at [MOD]
RSQ_VisualRewards/zscript/rsqthp/bar.zs:59-60. Each part has an empty `Tick` override (bar.zs:549), which still costs one VM call
per tic plus its share of the reset.

The bar runs everything in its own `Tick` → `Update` every tic (bar.zs:165-176, 178-362):

- `h.FullOf(who, peak)` → `LDCandidate` → up to two `FindInventory` walks per tic (handler.zs:208-218).
- Sprite-top measurement on every sprite-frame change (`TopOf`, bar.zs:199-203, 507-523): `GetSpriteTexture`, `TexMan`
  queries, and the voxel table lookup. Monsters change frame every 2-10 tics.
- `EventHandler.Find("RSQVR_FxHandler")` whenever the ceiling is above the monster, which is nearly always (bar.zs:213-219).
  That is a linear walk of the handler list ([F] events.cpp:1702-1710). The set registers ~171 handler names (grep of all
  `AddEventHandlers` in mods_x).
- `SetOrigin` when moved, `Prev` copy, camera-distance scale, visibility, fade and flash (bar.zs:220-277).
- Every 4th tic: series markers through services by name (bar.zs:321-360 → handler.zs:606-640). That is 1-9 `Service.GetInt`
  string-keyed calls, `BossMarkOf` (2-3 more), `TierOf`, `ChampFrame`, and `MutFrame`, which does 4 string→class conversions
  plus `FindInventory` (handler.zs:180-189).
- `SyncParts`: per part, `SetOrigin`, `Prev`, scale and style (bar.zs:470-504).

The owner's own bench on the rig was 1.36 µs per bar per tic with 170 bars ([MOD-src] VisualRewards/HANDOFF.md:386-388). The
full set on his PC measures 4.54 µs (16.46 ms / 3,622). **Inference:** the gap is the services the full series answers
(markers, tiers, miniboss), the per-tic handler `Find` added for "UNDER THE CEILING" (dated 11-11 in bar.zs:210-212, after the
bench), and the 11-10 side icons.

#### 1.2 Why there are 3,622 bars at a map start with no fight

The design is "a bar only once health differs from spawn health" (handler.zs:6-12). The 16-tic poll compares
`a.health != a.SpawnHealth()` (handler.zs:503-507). `SpawnHealth()` uses `StartHealth` or the class default ([F]
p_mobj.cpp:9159-9174).

Universal Entropy resizes every non-friendly monster: `ue_resize = 5` against `random(1,5)` is always true ([MOD]
2.4-Enemies_UniversalEntropy/CVARINFO:5; randomise.zs:39). It sets `a.Health = round(size * a.Health)` and never touches
`StartHealth` (randomise.zs:41-44). Champions raises health for Bronze ×2 and Titan, and also rewrites `StartHealth` at its
first controller tic ([MOD] 2.2-…/champs/champ_bronze.txt:30; baseclass.txt:321-327). LegenDoom candidates sit at 300%.

So nearly every monster's health ≠ `SpawnHealth()` within 16 tics of spawn, and nearly every monster gets a bar. It also draws
wrong: a resized monster at full health shows a partial bar or an overheal strip.

The survey's "LegenDoom / TopHPBars" link is therefore secondary. LegenDoom supplies only the 300% candidates, which the
code already treats specially (handler.zs:14-22); UE is the bulk.

**Inference:** with a baseline fix, the bar count at the Sunder MAP15 start drops from 3,622 to ≈0, because the player has
not moved [M] preamble. The 16.46 ms drops with it.

#### 1.3 Netplay defects in today's bar (F-MOD-10)

The bar is a playsim actor, but it reads `players[consoleplayer]`:

- the camera distance sets its `Scale` (bar.zs:226-234);
- it hides your own bar (bar.zs:239-240);
- Distance Limit (bar.zs:245);
- side-icon placement uses the camera angle (bar.zs:476-477).

`CullFar` (handler.zs:515-544) and `TraceAims` (handler.zs:548-577) are consoleplayer-only. `TraceAims` calls `Ensure`, which
**spawns a bar on one peer only** (handler.zs:575, :590).

Universal Entropy's `WorldThingSpawned` runs for that bar (it is neither friendly nor a player) and calls `frandom` twice on
the shared RNG before any filtering (randomise.zs:19-23). So the peers' RNG streams diverge: a desync with
`rsqthp_aimed 1`.

Even without Aimed Only, actor fields such as `Scale`, `bInvisible`, `Alpha`, `shown` and `lastFlash` differ per peer. That
breaks any whole-world hash (`VRSyncLog_OnTick` hashes every actor when `vr_synclog` is set, survey 08 §3.2) and makes
savegames differ.

The README's claim "Bars … are play-side and run the same on every machine" ([MOD-src] VisualRewards/README.md:344) does not
hold for these fields. There is precedent: the netplay lane already had to ship a client-side Damnums copy for exactly this
pattern ([F] tools/netplay/compat/README.md:5-9).

#### 1.4 Rewrite (a): stock engine, keeps the look, a fraction of the cost (P10-R1)

Principles, all taken from the owner's own damnums and words drivers, which already park glyphs below statnum 32 and run one
driver at 125 ([MOD] rsqthp/damnums/glyphs.zs:12-18; rsqvr/words.zs:29, :385):

1. **Baseline, not `SpawnHealth`.** A monster's "full" is its undamaged health. The baseline starts at its health when
   tracked and rises while it is undamaged (spawn-time buffs: UE, Champions, LegenDoom picks). A turned legendary keeps the
   owner's rule: drawn against `SpawnHealth()` with the overheal strip.
2. **No bar or part ever ticks.** Bars and parts are parked at statnums 27 and 26 (below `STAT_FIRST_THINKING`; 28 and 30
   are already taken by words and glyphs). The engine never ticks them, never resets their interpolation, and the default
   `ThinkerIterator` never sees them.
3. **One driver after the actors.** A `Thinker` at statnum 124 (stock-portable; on REMA, `WorldStep` is the same slot,
   [F] p_tick.cpp:896). It updates position and `Prev` only for bars whose monster moved, the frame only on a health change,
   and markers every 4th tic only for bars within 1536 units of some player.
4. **No `consoleplayer` in play.** Settings are server cvars already ([MOD-src] VisualRewards/pk3/CVARINFO.txt:3-23).
   - Far Size, Nearest 24 and Distance Limit use the nearest *player pawn*, which is the same on every peer.
   - Aimed Only traces from **every** player's hands (`AttackPos`/`OffhandPos` come from usercmd, so they agree across peers).
   - Side icons use the stock view-relative `SpriteOffset` ([F] wadsrc/static/zscript/actors/actor.zs:412; applied in
     hw_sprites.cpp:1569-1570) instead of the camera's right vector, so every eye of every peer draws them correctly.
   - Bars over players are dropped: a per-viewer "hide my own bar" cannot be one shared sprite (it comes back in (b) as a
     viewer mask).
5. **Cache the cross-mod lookups:** the `RSQVR_FxHandler`, services and token classes, as transient fields refilled after a
   load.

```zscript
// ---- handler.zs (deltas) ----------------------------------------------------------------
const STAT_BAR    = 27;    // below STAT_FIRST_THINKING: never ticked, never interpolation-reset (words 28, glyphs 30)
const STAT_PART   = 26;
const STAT_DRIVER = 124;   // after STAT_DEFAULT (100): monsters have moved
const MARK_NEAR   = 1536.0;

Array<int> monBase;        // undamaged baseline, same index as mons (replaces the SpawnHealth test)
Array<bool> monHurt;       // damaged at least once since tracked
transient RSQVR_FxHandler fx; transient bool fxLooked;
RSQTHP_Driver driver;

RSQVR_FxHandler Fx() { if (!fxLooked) { fxLooked = true; fx = RSQVR_FxHandler(EventHandler.Find("RSQVR_FxHandler")); } return fx; }

private void Track(Actor a) { mons.Push(a); monBar.Push(null); monPeak.Push(0); monBase.Push(max(1, a.health)); monHurt.Push(false); }

// the poll (replaces handler.zs:502-508): a bar only when hurt below the baseline, or a turned legendary off its spawn health
//   if (!b || b.who != a)
//   {
//       if (!monHurt[pollAt] && a.health > monBase[pollAt]) monBase[pollAt] = a.health;   // spawn-time buff: raise "full"
//       bool turned = ldTransformed && a.FindInventory(ldTransformed);
//       if (a.health < monBase[pollAt] || (turned && a.health != a.SpawnHealth()))
//           monBar[pollAt] = Ensure(a, false, monBase[pollAt]);
//   }
// WorldThingDamaged: set monHurt[idx] = true (idx = mons.Find(a); once per damaged monster) before Ensure.

override void WorldLoaded(WorldEvent e)
{
    // ... existing body ...
    if (!driver) { driver = RSQTHP_Driver(new("RSQTHP_Driver")); driver.h = self; driver.ChangeStatNum(STAT_DRIVER); }
}

// In Ensure(), right after the spawn (handler.zs:590):
//     b.ChangeStatNum(STAT_BAR); b.full = peak;  (no b.Update() here: the driver places it this tic)

Actor NearestPawn(Vector3 at, out double d)
{
    Actor best = null; d = 1e12;
    for (int i = 0; i < MAXPLAYERS; i++)
    {
        if (!playeringame[i] || !players[i].mo) continue;
        double dd = (players[i].mo.Pos - at).Length();
        if (dd < d) { d = dd; best = players[i].mo; }
    }
    return best;
}

void StepBars()    // called by the driver once per tic, after every actor has ticked
{
    if (!on) { for (int i = 0; i < bars.Size(); i++) if (bars[i]) bars[i].SetHidden(true); return; }
    bool markTic = (level.maptime & 3) == 0;
    for (int i = bars.Size() - 1; i >= 0; i--)
    {
        let b = bars[i];
        if (!b) { bars.Delete(i); continue; }
        let m = b.who;
        if (!m || m.health <= 0) { b.Destroy(); bars.Delete(i); continue; }
        b.Step(self, markTic);
    }
}

class RSQTHP_Driver : Thinker
{
    RSQTHP_Handler h;
    override void Tick() { if (h) h.StepBars(); }
}
```

```zscript
// ---- bar.zs (deltas): RSQTHP_Bar keeps its fields; Tick/Update are replaced by Step -----------------------------
int full;                  // the baseline it is drawn against (handler)
Vector3 lastMonPos, lastMonPrev;
double lastUp;

override void Tick() {}    // parked: never called; kept so an unparked bar is inert

void SetHidden(bool hide) { if (bInvisible != hide) { bInvisible = hide; SyncPartFlags(); } }

void Step(RSQTHP_Handler h, bool markTic)
{
    // ---- frame: only when something it shows changed (as BuildFrame today)
    int hp = who.health;
    bool turned = h.ldTransformed && who.FindInventory(h.ldTransformed);
    int mx = turned ? max(1, who.SpawnHealth()) : max(1, full);
    bool inv = who.bInvulnerable, al = h.ally && who.bFriendly;
    if (!built || hp != lastHp || mx != lastMax || inv != lastInv || h.ver != lastVer || al != lastAlly)
    {
        built = true; lastHp = hp; lastMax = mx; lastInv = inv; lastVer = h.ver; lastAlly = al;
        BuildFrame(hp, mx, inv);
        lenK = LenOf(who.SpawnHealth());
    }
    // ---- height: only on a sprite or scale change (TopOf as today)
    if (!haveTop || who.sprite != lastSpr || who.frame != lastFrm || who.Scale.Y != lastSY)
    {
        lastSpr = who.sprite; lastFrm = who.frame; lastSY = who.Scale.Y;
        topTarget = TopOf(who);
    }
    top = haveTop ? top + (topTarget - top) * 0.5 : topTarget; haveTop = true;
    double up = top * (1.0 + h.offset) + who.GetBobOffset();
    if (who.ceilingz > who.Pos.z)
    {
        let fx = h.Fx();                                         // cached: no handler walk per bar
        double over = fx ? fx.StackOver(who) : 0.0;
        up = max(who.Height * 0.5, min(up, who.ceilingz - who.Pos.z - (16.0 * Scale.Y + (over > 0 ? over + 10.0 : 0.0) + 2.0)));
    }
    // ---- place: the parked bar is never reset by the engine, so Prev is written whenever the monster's ends changed
    if (who.Pos != lastMonPos || who.Prev != lastMonPrev || up != lastUp)
    {
        lastMonPos = who.Pos; lastMonPrev = who.Prev; lastUp = up;
        SetOrigin((who.Pos.xy, who.Pos.z + up), false);
        Prev = (who.Prev.xy, who.Prev.z + up);
        SyncPartPositions();                                     // parts: SetOrigin + Prev, offsets via SpriteOffset
    }
    // ---- deterministic visibility and size: nearest player, never consoleplayer
    double d; h.NearestPawn(Pos, d);
    double s = h.scale;
    if (h.far > 1.0 && d > 256) s *= 1.0 + (h.far - 1.0) * min(1.0, (d - 256) / 768.0);
    if (Scale.Y != s || Scale.X != s * lenK) { Scale = (s * lenK, s); SyncPartScale(); }
    bool hide = (h.limit && d > h.limitDist) || culled
             || (h.aimedOnly && level.maptime - aimTic > RSQTHP_Handler.AIM_LINGER && !h.DownMarked(who));
    shown = hide ? 0 : min(shown + 1, FADE);
    SetHidden(hide);
    StepStyle(h);                                                // stealth / low pulse / fade / flash as bar.zs:249-277
    // ---- markers: every 4th tic, and only where some player can see them
    if (markTic && d < RSQTHP_Handler.MARK_NEAR) StepMarkers(h); // bar.zs:321-360 unchanged, parts spawned parked
}

// parts: spawned with ChangeStatNum(RSQTHP_Handler.STAT_PART). The side icons stop using the camera:
//   p.SpriteOffset = ((right ? 1 : -1) * (c0 + slot * SIDE_STEP), 0);   // view-relative, per eye, per peer
```

The handler's aim tracing loops `for (pn = 0; pn < MAXPLAYERS; pn++)` over `players[pn]` instead of `consoleplayer`, so
`Ensure(a, true)` runs identically on every peer. `CullFar` measures from `NearestPawn`.

**Cost (Inference):**

- Per live bar per tic: one driver-loop iteration plus 4-6 native field compares, and `SetOrigin` only for moved monsters.
  The owner's own VM bench of the lighter per-bar `Update` is 1.36 µs (HANDOFF.md:386). This path skips the
  `FullOf`/`Find`/services work on most tics, so **≈0.3-0.6 µs**.
- At the Sunder MAP15 start the baseline fix leaves ≈0 bars, so 16.46 + 0.61 (3,622 × 0.17) ≈ **17.1 ms → <0.1 ms**. The poll
  is `n/16` monsters a tic, as today.
- In a fight with 500 hurt monsters: 500 × 0.5 µs = 0.25 ms (today: 500 × 4.54 = 2.3 ms).
- Thinkers: −3,622 bars and every part.
- Determinism: new and old bars are different mods, so old demos with the old pack will not replay. Re-record the replay
  suite with the new pack.

#### 1.5 Rewrite (b): native, for the uplift (P10-R2, recommended)

The bar is pure presentation with per-viewer parts. Per the owner's direction, the permanent form is engine-side drawing with
script-side *policy*:

- **Engine (new): `ActorOverlay` slots.** The playbook's mechanism 1 ([P] 20 Phase 10 "Mechanisms" 1): a lazily allocated
  slot block behind one `AActor` pointer, serialized with the actor. Fields on top of the playbook's:
  - `FrameTex[]`, a 22-frame strip: the frame is chosen natively from `health / base` in 5% steps, so coldcite's art is kept
    exactly;
  - `Base` (the baseline above, written by script once);
  - `ViewerMask`, filtered by `consoleplayer` at render, which brings back "hide my own bar" and per-player Aimed Only;
  - `FarScale(from, to, k)`, resolved per eye from that eye's distance (restores Far Size per viewer);
  - `TopMode = SPRITE_TOP`: the renderer reads the sprite frame it is actually drawing for the actor, which removes `TopOf`
    and the smoothing script;
  - `Markers` bits with an icon atlas.
- **Renderer:** emit from `HWDrawInfo::RenderThings` right after the actor's own sprite ([F] hw_bsp.cpp:914-960, inside the
  `CurrentMapSections` test at :947). The bar is then drawn only when its monster's subsector is in view (the same culling the
  sprite gets, 192 sprites in the modded frame [M] §1) and lit from `thing->Sector` with no `PointInSector`. The position is
  `thing->InterpolatedPosition(TicFrac)`, the same call billboards use ([F] hw_drawinfo.cpp:3639), so the bar can never trail.
  "The renderer may only draw" holds: it reads health, sprite and pose, and writes nothing.
- **Script policy (cheap, event-driven):** `WorldThingSpawned` sets the baseline; `WorldThingDamaged`/`Revived` enable the
  slot; markers are pushed by the series mods through a native `Actor.SetOverlayMarkers(slot, bits)` instead of being polled
  through services. No per-tic work.
- **Cost:** 0 µs/tic sim; ≤ 3 quads per visible bar per eye.

#### 1.6 Interim on the existing fork billboards (only with F-MOD-1)

`Level.AttachBillboard(actor, offset, …, BB_TEXTURE, frameTex.GetIndex(), …)` exists and follows the actor at render rate
([F] doombase.zs:1130; hw_drawinfo.cpp:3637-3640). It must not be used at bar scale as it is:

- **Setters are linear.** `FindBillboardByID` scans the array (vmthunks.cpp:3109-3117), so per-tic `SetBillboardAlpha`
  (fade, pulse) on 7k billboards is O(n²): 7,244² ≈ 5.2·10^7 compares a tic.
- **No visibility cull.** `DispatchBillboards` submits every billboard each eye, with a `PointInSector` each and no frustum
  or BSP-visibility test (hw_drawinfo.cpp:3576-3661). Only optional distance and count caps exist (`rs_bb_cullradius`,
  `rs_bb_maxpanels`, :3543-3544, both off).
  - **Inference:** 7,244 quads at ~0.5 µs ≈ 3.6 ms per eye of scene build, against today's bars, which only exist for
    visible subsectors.
- **No attach-offset setter.** An attached billboard's `pos` is overwritten every tic from `attachOffset` (p_tick.cpp:284), so
  the sprite-top height cannot be updated without remove and re-attach.
- **Always fullbright and translucent** (hw_sprites.cpp:2880-2899), which changes the "Bright off" look.

F-MOD-1 fixes all four. Doing that is roughly the same work as P10.1i, so for HP bars prefer the overlay slots, and keep
billboards for panels.

### 2. mk-champions ([MOD] 2.2-Enemies-mk-champions[20210309]): ~25 ms listed

#### 2.1 Structure

- **Who becomes a champion:** every `bISMONSTER && bCOUNTKILL` monster, with chance `(4+sk·10)·(sk+1)`/256 for
  `sk = SKILLP_ACSReturn` (handler.txt:413-419). At UV (`ACSReturn` 3) that is 136/256 = **53% of monsters**. This matches
  1,646 icons [M] §3.
- **Per champion:** a `champion_BaseController` **plain `Thinker`** (handler.txt:434). It is not an actor, so it pays no
  reset; its cost is pure script.
- **Also per champion:** two inventory items (`PersistentInfo`, a token: tokens.txt:1-60), a `champion_Icon` actor
  (baseclass.txt:78-95), and per colour extras.

`champion_BaseController.Tick` runs every tic (baseclass.txt:303-378):

- `CountInv(ld_token2)` twice, where `ld_token2` is a `string`, so each call converts string→class at run time;
- `champion_SpawnParticles()` → `A_SpawnParticle` every tic when `champion_Particles 1` (baseclass.txt:331-332, :100);
- health compares;
- `TickEffect` every `eftic` tics;
- for Titans, `A_RadiusGive("champion_AggroEffect", 384, RGF_MONSTERS)` **every tic** (baseclass.txt:372-377).

#### 2.2 The hot colours

- **Cyan, 7.16 ms / 122 = 59 µs each.** `eftic = 1` and `A_RadiusThrust(192, 384)` every tic (champ_cyan.txt:34, :53-56). In
  the fork's `P_RadiusAttack` that is a `FMultiBlockThingsIterator` over a 768-unit box plus a **`P_CheckSight` per thing in
  range** ([F] src/playsim/p_map.cpp:8542 ff., sight test at ~:8622). In Sunder's hordes that is dozens of sight checks per
  champion per tic.
  - It also spawns 8 `champion_Leaves` per Cyan (champ_cyan.txt:42-45). 122 × 8 = 976, which equals the measured 976.
  - Each leaf runs `Super.Tick()` with a 1-tic animation loop, `A_SetAngle`, and `A_Warp(AAPTR_MASTER, WARPF_INTERPOLATE)` every
    tic (champ_cyan.txt:63-116): 2.2 µs each.
- **Silver, 73 µs.** The same thrust, with a species filter (champ_silver.txt:28, :47-49).
- **DarkRed, 20.7 µs**, although its `TickEffect` is one compare on most tics (champ_darkred.txt:53-60). That makes it the
  best estimate of the **base controller cost, ~20 µs**.
  - **Inference:** too high for the code shown, which points at the VM interpreter (JIT state unknown, [M] §5) or the
    particle path. It needs `vm_profile`.
- **`champion_Icon`, 4.79 ms / 1,646 = 2.9 µs.** `Super.Tick()` plus `A_Warp` every tic. It is `+NOINTERACTION` without
  `+NOBLOCKMAP`, so every warp also relinks the blockmap (extrafx.txt:32-81).
  - RSQTHP already hides these icons and draws its own (handler.zs:190-205). Yet they still tick, and RSQTHP's `WorldTick`
    walks all 1,646 of them with a `FindInventory` each, every tic (handler.zs:466, :192-204).
- **`champion_Aggro*`, ~4.3 ms, ~2,900 calls.** The Titan's per-tic `A_RadiusGive` hands each monster in 384 units a
  `CustomInventory` that is created and consumed every tic (allocation churn). Each such monster also holds an `AggroPower`
  powerup and an `AggroVisual` actor that runs `SetOrigin` and `CountInv("…")` every tic (extrafx.txt:341-447).

#### 2.3 Unlisted controllers, a likely large share of the 78 ms

The listed controllers are Cyan, DarkRed and Silver: 245 of them. The colour weights are 14 × 10 + 3 + 2 + 1 = 146
(cvarinfo.txt), and Cyan's 122 at weight 10 implies **~1,780 controllers** in total. That leaves ~1,535 controllers of other
colours that the [M] table does not list.

**Inference:** at the ~20 µs base cost derived from DarkRed, that is ~31 ms, roughly 40% of the unattributed 78.3 ms. Here
78.3 = 130.8 − the 52.5 ms of listed entries, carried by 53,116 − 35,681 = 17,435 unlisted thinkers.

This hypothesis is the first thing to check (Open questions Q1).

#### 2.4 Pink: a whole-world scan, every 70 tics per Pink champion

`ThinkerIterator.Create("Actor")` with `Distance2D`, `FindState("Raise")` and two `CountInv` (string) per actor
(champ_pink.txt:38, :55-75).

**Inference:** weight 10/146 × 1,780 ≈ 122 Pink champions, so 122/70 ≈ **1.7 full walks of 53k thinkers per tic**. At ~0.1 µs
per VM iteration that is ~9 ms/tic on average, and spiky. The rest of the list:
- Violet: a projectile fan every 30 tics;
- Grey: `CheckSight` every 8;
- White and DarkGreen: spawns every 10.

#### 2.5 Fix: engine mechanisms first, then the port (owner-maintained `2.2-…_REMA.pk3` per [F] tools/netplay/compat)

| Cost | Engine mechanism (permanent) | Port onto it / stopgap |
|---|---|---|
| Icons 4.79 + reset 0.28 | P10.1 overlay slot (`champion_SpawnIcon` → `SetOverlay`) | Today, with zero code: server cvar `champion_Icons 0` (cvarinfo.txt:7). RSQTHP reads the type from `champion_*Token` (tokens.txt:71-94: the token index equals the icon frame) instead of the icon actors, which also deletes `ChampIconsTick` from RSQTHP's `WorldTick` |
| Leaves 2.18 + 0.17 | F-MOD-3 client-side thinkers; P10.2 render-only attachment | Spawn with `SpawnClientSide` and use `cfrandom` in `PostBeginPlay`. Cull per client beyond 2048 from the local camera with client-side `Sleep`. |
| Cyan/Silver thrust (≈ (59−20)×122 + (73−20)×21 ≈ 5.9 ms) | Phase 9 per-tic sight cache (pairs reused across 122 champions); native fast-reject for `RTF_NOTMISSILE` | Gate: `if (no player pawn within 384 + 512) return;`. It is deterministic (player pawns are sim state) and keeps the gameplay everywhere a player is. |
| Aggro ~4.3 | P10.2 (visual), F-MOD-8 (power parked) | Titan `A_RadiusGive` every 35 tics: the power lasts 10 s (`Powerup.Duration -10`, extrafx.txt:395-401), so the gameplay is identical apart from the `DamageFactor` count lag. `AggroVisual` becomes client-side. |
| Base ~20 µs × ~1,780 | P10.5 batch: one native-backed `WorldStep` loop over a compact controller array; F-MOD-6; JIT verified | Port: cache `class<Inventory>` tokens in `PostBeginPlay`; `champion_Particles 0` (server cvar) or client-side particles. Detect hit and death with `WorldThingDamaged`/`Died`. Then `Sleep(eftic-1)` between effects for every colour with `eftic` > 1 (fork `Sleep`, [F] doombase.zs:284-292). |
| Pink scans | P10.5 per-class lists | `BlockThingsIterator.Create(champion, radius*radfactor)`: the radius is ~4-8 × the monster radius, so it visits a handful of things instead of 53k |

**Netplay:** every row above is a sim change with fixed rules (no `consoleplayer`, sim RNG untouched, except
`cfrandom` in client-side leaves). Champions already holds global per-machine state via `server` cvars only. Re-record the
replays with the patched copy.

### 3. "Legion of Bones" monsters, ~11.3 ms: really the per-monster inventory chain (P10.9)

`SkeletonImp` is plain DECORATE: no `Tick`, no ZScript, `A_Look` every 10 tics, `A_Chase` every 2
([MOD] 2.1-Enemies_LegionOfBones_v1.05/decorate/imp.txt:1-61). The pk3's only ZScript is a 12-line `InfightController`
(ZSCRIPT:1-12). So its 2.46 µs (4.24 ms / 1,724) against `DoomImp`'s 0.31 µs [M] §3 is not LoB.

Every owned inventory item's `DoEffect` is called from **inside the owner's `AActor::Tick`** ([F] p_mobj.cpp:4870-4881), so
`profilethinkers` books it to the monster's class. Every monster carries:

- mk-crits `csh_critController`. Its `doEffect` calls `isAttacking()` = 2 × `FindState` plus 2 × `InStateSequence` **every
  tic** ([MOD] 4.01-…/csh_classes/core.txt:206-209, :229-278). `FindState` scans the class's label list; `InStateSequence`
  walks a BD22 state chain.
- `csh_Brain`: `ModifyDamage` only (core.txt:1-181).
- `Shifter`: its own `Tick`, measured separately.
- Champions tokens (cheap after the first `DoEffect`, tokens.txt:33-59).
- LegenDoom tokens (other report).

**Inference:** crit `DoEffect` ≈ 0.5-1.0 µs × 5,666 monsters (11,332 crit items ÷ 2, handler.txt:3-10) = **2.8-5.7 ms**,
spread over all monster classes.

**Fix:**
- *Port (crits copy):* resolve `stMissile = owner.FindState("Missile")` and `stMelee` once in `AttachToOwner`. Recompute
  `isAttacking` only when `owner.curState != lastState`. The results are identical, so this is deterministic even against
  the old version.
- *Engine (permanent):* `Actor.StateSequenceID` / cached label lookups are a Phase 9 native fast path. F-MOD-8 parks the two
  crit items themselves.

### 4. Universal Entropy `Shifter`: 2.23 ms, 7,686 thinkers (P10.7, F-MOD-5, F-MOD-9)

`Shifter` is given to every resized monster **and every solid decoration** (`ue_decor_resize 5`)
([MOD] 2.4-Enemies_UniversalEntropy/randomise.zs:55, :81; CVARINFO:9). Each tic it runs `A_SoundPitch(CHAN_VOICE)`,
`A_SoundPitch(CHAN_WEAPON)` and a tic rescale on state change (randomise.zs:146-166).

- On [FR] the sound queries walk only the source's own channels ([FR] s_sound.cpp:1089-1102), so 0.29 µs each.
- **On [F] main they walk every channel** ([F] s_sound.cpp:1073-1085). The render lane measured the main-equivalent at
  **174 ms/tic** on Abandon MAP13 ([FR] docs/plans/SIM_AT_MOD_SCALE.md §1).
- All [M] numbers come from the renderer build `doomxr_rend1113a` [M] preamble. **Main cannot reproduce them until
  `SourceChannels` is ported** (F-MOD-5).

**Fix:**
- *Engine:* F-MOD-9, `AActor::SoundPitchScale` (applied where `S_StartSound` sets a source's pitch) and `StateTicScale`
  (applied in `SetState`, rounded as `Shifter` does). Deterministic; serialized.
- *Port:* `UE_randomisation_EventHandler` sets the two fields and gives no item. That is −7,686 thinkers: 2.23 + 1.31 =
  **3.5 ms**.
- *Stopgap without the engine:* decorations get no `Shifter` (shift_speed 0 has nothing to rescale, and solid decorations
  rarely voice), and monsters re-pitch only when `curState` changes. That roughly halves the cost.
- *Netplay note:* `players[consoleplayer].mo` is used for friend tests in play (randomise.zs:9, :19). In co-op every pawn is
  a friend, so the result agrees, but the port should test against `players[0..3]`.

### 5. crits (`csh_critController` + `csh_Brain`): 1.57 ms, 11,332 thinkers (P10.6, F-MOD-8)

Both are `Inventory` items with no `Tick` override, so each runs the ZScript `Inventory.Tick` ([F]
inventory.zs:113-147), a VM call that does nothing for an owned item with a `-1` state: 0.14 µs [M] §3. Each also pays the
0.17 µs reset share. The real work is `DoEffect` (§3) and `ModifyDamage` (event-like already).

**Engine (F-MOD-8, permanent):**
- On `AttachToOwner`, an item whose class keeps `Inventory.Tick` (compare the vtable slot once per class), whose state has
  `tics == -1`, and whose `DropTime == 0`, moves to a parked statnum (e.g. 25). It moves back on `SetState` to a timed state,
  on `DetachFromOwner`, or on a `DropTime` write.
- `DoEffect` still runs from the owner (unchanged), so behaviour is identical. Only the item's list position changes when
  it unparks: deterministic, but re-baseline the replays.
- This one change covers crits, every champion token and `PersistentInfo`, `RSQTHP_ChampKind`, the LegenDoom tokens, and
  any future token mod.
- **Saving (Inference):** ≥ 11,332 × (0.14 + 0.17) = 3.5 ms; more with the other no-op items (count via Q4).

**Port-only stopgap:** merge `csh_Brain` into `csh_critController` (one item per monster): −5,666 thinkers ≈ 1.75 ms.

### 6. UA ambience `IA_Sound`: 0.89 ms, 2,544 thinkers

`IA_Sound` is a DECORATE actor with **no states and no `NOINTERACTION`/`NOBLOCKMAP`**
([MOD] 3.09-…_UA_UniversalAmbience/decorate.txt:1-8). So it is a full-path actor in the blockmap, spawned per qualifying
sector flat at `WorldLoaded` and per item species, and given one looping ambient sound
(idleambiance_events.zs:315-361, :365-384). It never moves. Its loops are the evicted channels the render lane found
(SIM_AT_MOD_SCALE §1).

**Port:** in the handler, after `A_StartSound`, call `ptr.ChangeStatNum(29)` (parked, still a sound source), and in
DECORATE add `+NOINTERACTION +NOBLOCKMAP`. That gives 0.89 + 2,544 × 0.17 = **1.3 ms → 0**. It also removes 2,544 things from
every `BlockThingsIterator`/`P_CheckPosition` that passes through those blocks. Deterministic (no RNG; the sound is output).

**Engine (generic):** a never-moving actor with no states (or a `-1` state) is a dormancy candidate for Phase 9 step 2 (other
report). The UA port is the same idea done by hand.

### 7. `RSQ_*` uplift audit (per-tic hooks; hidden sim cost to keep out of REMA)

Counts over every script file per mod (`override void Tick`, `WorldTick`/`WorldStep`, `ThinkerIterator.Create`,
`BlockThingsIterator.Create`, traces, `EventHandler.Find`/`ServiceIterator.Find`):

| Mod | lines | Tick | WorldTick | ThIt | BTIt | traces | Find |
|---|---|---|---|---|---|---|---|
| VisualRewards | 7,285 | 11 | 6 | 1 | 0 | 3 | 29 |
| Saber | 8,795 | 13 | 6 | 3 | 19 | 2 | 16 |
| Grenade | 9,133 | 13 | 2 | 5 | 5 | 3 | 15 |
| Melee | 6,750 | 7 | 9 | 0 | 15 | 5 | 21 |
| DistanceGrab | 10,772 | 2 | 12 | 3 | 10 | 1 | 31 |
| TacticalRig | 13,502 | 7 | 8 | 3 | 1 | 0 | 35 |
| Whip | 8,287 | 5 | 4 | 3 | 9 | 12 | 18 |
| Score | 12,618 | 6 | 6 | 3 | 2 | 2 | 29 |
| Monsters | 81,370 | 4 | 0 | 5 | 0 | 1 | 6 |
| (others) | | ≤8 | ≤5 | ≤5 | ≤6 | ≤5 | ≤35 |

Per-tic or per-monster cost to fix before or while uplifting:

- **RSQ_Score head sensors.** `RS_ScoreHeadSensor` is a per-monster `+SHOOTABLE` helper that runs `SetOrigin` every tic
  (`Vec3Angle` of its master) ([MOD] RSQ_Score/zscript/rsscore/head.zs:585-660). On its first tic it runs
  `ThinkerIterator.Create("RS_ScoreHeadSensor")` over **all** thinkers (head.zs:651-653).
  - That is O(N_monsters × N_thinkers) at a map start: **Inference:** 3,600 × 53k ≈ 1.9·10^8 iterations, seconds of stall.
  - Native: P10.2 attachment with collision. Dedupe with a token on the master instead of the scan.
- **RSQ_SlowMo `Gather`.** While slow-mo runs, a full `ThinkerIterator("Actor")` every 17 tics with `act.Find` per actor
  (O(N·M)) ([MOD] RSQ_SlowMo/zscript/rsqsm/clock.zs:1240, :1358-1375). Its own comment measures 1.5 ms at 3,543 actors;
  ×15 at 53k. Native: per-class lists, or a spatial query around the players.
- **RSQ_Miniboss.** `BfScan`/`Enrage`/`TierUp` walk all actors. They are event-triggered (handler.zs:135, :448), so not per
  tic. Fine as is; use per-class lists on uplift.
- **RSQ_Whip `FireballsWatch`.** Walks all actors only on the off→on edge (whip.zs:661-673). Fine.
- **RSQ_VisualRewards.** Bars (§1).
  - The words driver reads per-player user cvars through `players[consoleplayer]` in play (rsqvr/words.zs:390-399). Move it
    to UI, or read each player's own cvar (`CVar.GetCvar(name, players[i])`).
  - `consoleplayer` appears in play-scope paths across the series: Wheel 69, TacticalRig 60, DistanceGrab 38, VisualRewards 35
    hits (grep). Each needs the F-MOD-10 audit before netplay is frozen.
- **GlowInTheDark.** `ddz.zs:24` iterates `Lighting` thinkers (a small class). No per-tic cost seen. On uplift it belongs
  with Radiance in the renderer lane.

**Uplift recommendation per hot RSQ system:**
- **HP bars:** native overlay slots (§1.5).
- **Damage numbers and words:** already parked VisualThinkers with one driver. Keep as script, or move to native billboards
  after F-MOD-1. Make them client-side (the Damnums compat precedent).
- **Head sensors:** native attachments.
- **SlowMo gather:** per-class lists.
- **Markers/services:** a native per-actor marker bitfield set by the owning mod, replacing string-keyed service polls.

### 8. What the fork already has that helps (verified)

| Facility | Where | Use here |
|---|---|---|
| Parked statnums < 32 (stock) | [F] statnums.h:34-42; `ChangeStatNum` allows them for actors except `STAT_STATIC` (dthinker.cpp:1359-1387) | Bars, parts, IA_Sound |
| Thinker `Sleep`/`SleepIndefinite`/`Wake` (GZSelaco port) | [F] dthinker.cpp:97-140, :1188-1272; doombase.zs:284-292 | Controllers with `eftic`>1; client-side leaves |
| `WorldStep` after the thinkers | [F] p_tick.cpp:896; events.cpp:1079, :2109-2117 | Driver slot (bars, batched controllers) |
| Attached, view-locked and hand-locked billboards, groups, serialized | [F] doombase.zs:886-1300; g_levellocals.h:349-447; p_saveg.cpp:1142 | Panels; bars only after F-MOD-1 |
| Client-side thinkers on their own world clock | [F] p_tick.cpp:155-204 | Cosmetic helpers |
| `SpriteOffset` (view-relative) | [F] actor.zs:412; hw_sprites.cpp:1569-1570 | Bar side icons without the camera |
| Netplay compat-copy mechanism | [F] tools/netplay/compat/README.md | Versioned patched third-party mods |
| Effects governor | [F] src/common/rendering/hwrenderer/data/hw_effectsgovernor.h:15-63 | **Does not touch script/sim work** (presentation only), so it cannot rescue the tic |
| Per-source sound lists, catch-up cap | [FR] only (F-MOD-5) | Needed on main |

Not present on either branch: overlays, actor attachments, throttled statnums, per-class lists, transient iterators, an
inventory fast path, or a `TickBehaviors` fix.

---

## 90 Hz contribution

- **Frame budget:** 11.1 ms at 90 Hz (the governor's own budget, hw_effectsgovernor.h:28-29). The tic budget is 28.6 ms at
  35 Hz.
- **Without the sim thread** (Phase 8 / SIM_AT_MOD_SCALE §3.2), a tic lands inside one headset frame. It must fit beside the
  render CPU (no-mod flat scene build 3.8 ms [M] §1, doubled per eye without multiview). That leaves **≈ 5 ms for the whole
  tic**, which this area cannot reach alone.
- **With the sim thread,** the tic must fit in its 28.6 ms slot with headroom. The playbook targets thinkers ≤ 20 ms and the
  whole tic ≤ 25 ms median ([P] Phase 10 success metric).

**This area's budget:** all mod helper families together **≤ 5 ms/tic** on Sunder MAP15 with the full set, and thinkers ≤ 25k.

| Family | Today ms/tic (+ reset share) | Target ms/tic | Method | Who | Netplay notes |
|---|---|---|---|---|---|
| RSQTHP_Bar (+parts) | 16.46 + 0.61 | <0.1 at start; ≤0.25 with 500 hurt | Now: P10-R1 (baseline, parked, driver, no consoleplayer). Uplift: native overlay slots P10-R2 | Owner (VisualRewards) then engine | Removes a latent desync; re-record replays |
| champion_Icon | 4.79 + 0.28 | 0 | `champion_Icons 0` + RSQTHP reads tokens; later overlay slot | Owner (cvar + VisualRewards) | Server cvar: same on all peers |
| champion_Leaves | 2.18 + 0.17 | 0 sim | Client-side spawn (F-MOD-3); later render-only attachment | Owner (compat copy) | `cfrandom`; local culling allowed |
| Cyan/Silver thrust | 8.69 | ≤1.5 | Player-proximity gate (copy); Phase 9 sight cache (engine) | Owner + engine | Gate on pawn positions only |
| DarkRed + ~1,535 unlisted controllers | 2.11 + (~31, hypothesis) | ≤2 | Cached token classes, `champion_Particles 0`, event hit/death, `Sleep(eftic-1)`, JIT check; engine batch (P10.5), F-MOD-6 | Owner copy + engine | Fork `Sleep` is saved and deterministic |
| Pink scans | ~9 avg, spiky (hypothesis) | ~0 | `BlockThingsIterator`; engine per-class lists | Owner copy | Result order changes, deterministic |
| champion_Aggro* | ~4.3 | ≤0.4 | Titan give every 35 tics; visual client-side; F-MOD-8 | Owner copy + engine | Gameplay-equivalent |
| "LoB" monsters (crit `DoEffect`) | 11.29 (of which ~2.8-5.7 crit) | −2.8…5.7 | Cache states (copy); native label cache | Owner copy + engine | Same results |
| Shifter | 2.23 + 1.31 [FR]; catastrophic on [F] | 0 | F-MOD-5 port; F-MOD-9 fields; UE copy drops the item | Engine + owner copy | Tic scale deterministic |
| csh_* items | 1.57 + 1.93 | 0 | F-MOD-8 parked owned inventory (or merge items) | Engine | Order change: re-baseline |
| IA_Sound | 0.89 + 0.43 | 0 | Park + `NOBLOCKMAP` (UA copy) | Owner copy | None |
| RSQ head sensors, SlowMo gather | not in [M] (load stall, slow-mo only) | 0 per tic | Attachments; per-class lists | Owner (uplift) + engine | — |

**Sum (Inference):** listed families today ≈ 52.5 ms + their reset share ≈ 7.0 ms. After this plan: ≤ 5 ms. Thinkers drop by
≈ 3,622 + 1,646 + 976 + ~2,900 + 7,686 + 11,332 + 2,544 ≈ **30,700**, from 53,116 to ~22k. If the unlisted-controller
hypothesis holds, another ~25-30 ms goes.

**Gap:** nothing in this table is done on main. The fork has the enabling primitives: parked statnums, `Sleep`, `WorldStep`,
client-side thinkers, billboards and compat copies. It lacks the four class-wide engine mechanisms: overlays, parked owned
inventory, per-actor pitch/tic scale, and per-class lists. It also lacks the render lane's sound fix on main.

---

## Open questions (exact runs on the owner's PC via the bridge's `boottest`)

1. **Full per-class list (attributes the 78 ms):** Sunder MAP15, the owner's 57-file `.zdl`,
   `-Exec` with `profilethinkers 200` after 60 tics (the stock command prints all classes). Save the whole table, not the top
   entries. Confirms or kills the "~1,535 unlisted champion controllers ≈ 31 ms" hypothesis and finds BD22/XMVE helpers.
2. **JIT state and VM share:** read the startup log line for `vm_jit`, then rerun Q1 with `+vm_jit 0` and `+vm_jit 1`. If
   DarkRed's ~20 µs halves with the JIT, the controller base cost is interpreter-bound.
3. **Bar count with the baseline fix:** same run as Q1, plus a quick `RSQTHP_Handler.spawnedBars` dump through `-Exec`
   (`netevent`, or the existing test harness probe, test/probe.zs). Run with `ue_resize 0` vs 5 to confirm UE drives the 3,622
   bars (expect ≈ hurt-monster count, i.e. ~0, with `ue_resize 0`).
4. **Owned no-op inventory count:** a one-off script dump: count owned `Inventory` whose class keeps `Inventory.Tick`, state
   tics −1, `DropTime` 0, on Sunder MAP15 at tic 60. Sizes F-MOD-8.
5. **Crit `DoEffect` share:** Q1 with `csh_critical_enabled 0` vs 1 (if the copy guards `doEffect` on it). The delta in the
   monster classes' µs is the inventory-chain cost booked to LoB.
6. **Pink spikes:** `ticprofile` per-tic series (renderer build) over 300 tics with `champion_weight_Pink 0` vs 10. The p95
   tic delta is the scan cost.
7. **Main vs renderer, Shifter:** on a main build, `profilethinkers` on Abandon MAP13 with UE loaded. Confirms F-MOD-5 before
   any [M] number is quoted for main.
8. **Netplay:** `tools/netplay/run.sh` with RSQ_VisualRewards and `rsqthp_aimed 1` on the BD22 net set (README "NETSET=1").
   Expect a desync today (F-MOD-10), and IN SYNC with P10-R1.
9. **Billboard scale test (before any bar moves onto billboards):** `bb_spawn`-style script that attaches 7,000 billboards on
   Sunder MAP15. Read `cpu_scene_ms` with `perf_track 1`, with `rs_bb_cullradius 0` vs 2048. Sizes F-MOD-1.
