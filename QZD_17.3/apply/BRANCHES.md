# Branches on RSQ_UBQSeries — what to test

Six branches on `github.com/presidentkoopa/RSQ_UBQSeries`, each cut from `main` at `3e8bbff`
and independent of the others. Each carries its rebuilt pk3, so you can download a branch and
load it straight away.

**None of this is compiled.** There is no `zip` and no Linux rig on the machine these were
written on, so no ZScript parse ran and nothing was worn. The first real check is you loading
it. If one throws on load, the error names the file and line.

**`Wheel/` and `Grapple/` are untouched in all six.** You are working in both.

| branch | pk3 | what changed |
|---|---|---|
| `saber-menu-simplify` | `RSQ_Saber.pk3` | 137 cvars → 26, 142 rows / 14 pages → 11 rows / 1 page |
| `distancegrab-fixes` | `RSQ_DistanceGrab.pk3` | Headlock blocks projectiles; cheaper near sweep |
| `stabilisation-gate` | `RSQ_TacticalRig.pk3`, `RSQ_LaserSight.pk3` | off hand held while the engine has it |
| `gunbonsai-occlusion` | `RSQ_GunBonsaiVR.pk3` | punches no longer land through walls |
| `slowmo-fixes` | `RSQ_SlowMo.pk3` | frozen-actor gravity, friction guards, CHAN_AUTO pitch |
| `shieldsaw-sweep` | `RSQ_ShieldSaw.pk3` | grind swept, per-victim rate, held-prop fixes |

---

## What to look for, per branch

### `saber-menu-simplify`
One options page instead of fourteen. **The off blade should now light** — it never did at the
default, because the merge that suppressed it triggered within 40 map units, and that is 1.17 m.
Clash haptics must still fire; that was the one cvar whose fallback did not match its default,
and removing it without the paired edit would have silently killed them.

Blade length rows are labelled with the five lengths that exist — 54 / 65 / 76 / 88 / 99 cm.
99 cm is the longest the mod can draw. Extending needs `tools/ubqs.py`, which is not in the repo.

### `distancegrab-fixes`
**Hold a monster and let something shoot a fireball at you.** It should now stop. It never did:
`bTHRUACTORS` on the held body made the engine skip collision on the OR of both actors' flags
before any other test, while hitscans never consulted it — so bullets stopped and projectiles
did not.

**Expect one feel change at point-blank.** With that flag gone, a monster held close sits inside
your own footprint, `TryMove` refuses, and the body lags at arm's length instead of following
in. Recorded as D33. If it reads badly the answer is a bigger stand-off, not the flag back.

### `stabilisation-gate`
**Brace a rifle — off-hand grip, hands together — then cross-draw from a hip.** Before, you got
a weapon that rendered nothing, fired nothing and answered nothing until you let go. Now the
draw waits for the grip and arrives when you release. The holster still wins the press; only the
raise waits.

Same for the off-hand laser: it now switches off cleanly with its dot, glow, light and ring,
instead of the beam vanishing while the dot kept drawing from an invisible gun.

Return is within a few tics of release rather than instant — the flag script reads only
refreshes when a weapon layer changes state.

### `gunbonsai-occlusion`
**Punch at a thin wall with an enemy behind it.** The hit should not land.

Set `rsq_punch_debug 1` first. A blocked punch prints a line. **No line means the check did not
fire, not that the wall was clear** — that distinction is how you tell a working build from one
that silently did not load.

Known limit: shove your controller physically *past* the wall plane and the trace starts on the
far side and reports clear. Punch *at* a wall, not through it.

### `slowmo-fixes`
Three things, all audible or visible:
- an airborne monster is no longer kicked downward when a sphere ends
- a Lost Soul holds its charge speed instead of bleeding it
- **`CHAN_AUTO` sounds now drop in pitch with the clock** — landing thuds, grunts, the mod's own
  cues. They never did: the loop started at channel 1 and `CHAN_AUTO` is channel 0.

### `shieldsaw-sweep`
**Sweep the saw across a monster quickly.** The grind should catch it; it used to sample a single
point at 8.75 Hz and skip about 11.6 units per evaluation at 3 m/s.

**Grab the shield.** It should no longer visibly jump — the world prop now honours the same six
offset sliders as the drawn one.

Two things to watch, both named in `ShieldSaw/HANDOFF.md`: the grind sweeps along the plate's
*absolute* path, so walking forward sweeps it back along where the plate really went (one-line
alternative named if it reads wrong); and if the held trim rotates the wrong way for your hand,
two signs are named to flip. The positional slide that stops the jump does not depend on them.

---

## Suggested order

Test them one at a time. Six branches of unverified change landing together is how a load error
in one gets blamed on another.

1. **`saber-menu-simplify`** — biggest visible change, smallest risk. It is almost all data
   lumps, and a deleted cvar is a no-op here because the call sites carry their own fallbacks.
2. **`slowmo-fixes`** — three narrow guards, nothing structural.
3. **`gunbonsai-occlusion`** — one line plus a helper, with a debug print to prove it fired.
4. **`stabilisation-gate`** — two mods, but each is one gate.
5. **`distancegrab-fixes`** — carries the point-blank feel change.
6. **`shieldsaw-sweep`** — the most code, and the one with a derived-not-measured sign in it.

---

## Still open

Not attempted, and each is bigger than a fix:

- **The hand-claim protocol** (`10_HAND_OWNERSHIP.md`) — three mods answer "is this hand busy"
  three private ways, with a fourth predicate inside DistanceGrab. Six collisions, five proven.
  Wheel should publish first.
- **`player.keepmomentum`** — absent from all 152 ZScript files in the repo, so `vel.XY` is
  wiped every grounded tic. Belongs in the tether rework.
- **The swung-object swept chord** in DistanceGrab — "hit things with them" has no hit test at
  all today. Cost M.
- **GunBonsai and Score chains and combos** — the big pass, deliberately last.
