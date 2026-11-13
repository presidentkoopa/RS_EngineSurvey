# QuestZDoom 17.3 — what you can build on it

A teardown of emawind84's QuestZDoom (tag `qzd17.3`) answering one question per topic:
**what can we build as a mod on this exact engine, and how?**

Eleven reports, every capability claim cited to `path:line` in the engine source. Nothing here
is a feature request — it is what the shipped engine already does, and exactly where it stops.

**This is an index, not a wiki.** Find your mechanic in the tables below, go to that section,
and the design is there: what the player physically does, how script detects it, how it
resolves, the tuning knobs, and the real function names.

---

## Read these two first, whatever you are building

| File | Why |
|---|---|
| **[surveys/00_CAPABILITIES.md](surveys/00_CAPABILITIES.md)** | The shared foundation. **§0 is a go/no-go** — read it before writing code. §7 is the thirteen gotchas that silently eat prototypes. |
| **[surveys/11_HUD_MODELS.md](surveys/11_HUD_MODELS.md)** | Who owns a psprite layer decides which hand it rides, whether it survives a weapon switch, and whether it can be a model at all. Read before building any wrist panel, gauge or held instrument. |
| **[surveys/99_OVERLOOKED.md](surveys/99_OVERLOOKED.md)** | Mechanics nobody asked for, found while reading the source. **§1a is the best single find in the survey.** |
| **[surveys/VR_MELEE_ON_QUESTZDOOM_2026-10-24.md](surveys/VR_MELEE_ON_QUESTZDOOM_2026-10-24.md)** | **Melee in depth.** Part One: every hand, hit, swept-test, feedback and netplay primitive ZScript can reach on stock QZD 17.3, cited to `file:line`. Part Two: UltraBadassQuestSaber, RS_ShieldSawQ and the BD22 Quest weapons compared technique by technique, with the trap table and what a fourth implementation should be. Read with [01_MELEE](surveys/01_MELEE.md). **One claim is wrong** (`A_QuakeEx` intensity is not an int): [its corrections](surveys/VR_MELEE_ON_QUESTZDOOM_2026-10-24_CORRECTIONS.md). |

### The five facts that shape every design here

1. **Both hands are fully tracked**, plus the head — `AttackPos`/`AttackAngle`/`AttackPitch`/`AttackRoll`
   and `OffhandPos`/`OffhandAngle`/`OffhandPitch`/`OffhandRoll`, `actor.zs:268-275`.
   Gate every read on `OverrideAttackPosDir` (`actor.zs:276`).
2. **No velocity fields, 35 Hz sampling.** You difference positions yourself. Test the *chord*
   between samples, never the point, or fast motion passes through targets.
3. **Collision is an axis-aligned square, not a cylinder**, and it cannot be rotated
   (`p_map.cpp:1417-1419`).
4. **Haptics have no ZScript binding — but two back doors work today.** `LineAttack` with
   `LAF_NOINTERACT|LAF_ISOFFHAND` gives a clean per-hand thump (`p_map.cpp:4764-4769`);
   `A_QuakeEx` gives a shapeable envelope with the VR camera shake off by default
   (`r_utility.cpp:1085-1091`, `:116`) — but **pass `damrad = 0`** or it shoves the player.
   Both in [99_OVERLOOKED §1a](surveys/99_OVERLOOKED.md).
5. **Script cannot move the VR view.** Pitch and roll are overwritten from the HMD every frame
   (`hw_entrypoint.cpp:143`). Screen shake and camera kick are walls, not choices.

---

## "I want to make…" → where to look

| If you are building | Start here | Then |
|---|---|---|
| **Super Punch-Out, Streets of Rage, Condemned** | [01_MELEE](surveys/01_MELEE.md) §4.1-4.8 | §0.2(4) the trap that kills fist mechanics; [VR_MELEE_ON_QUESTZDOOM](surveys/VR_MELEE_ON_QUESTZDOOM_2026-10-24.md) for every primitive in depth |
| **Swordplay, lightsabers, Zelda** | [01_MELEE](surveys/01_MELEE.md) §4.9+ | §2 swept-chord tracing — this is how fast swings hit |
| **Castlevania / Indiana Jones whip** | [05_GRAPPLE_ROPES](surveys/05_GRAPPLE_ROPES.md) §4.3 | [01_MELEE](surveys/01_MELEE.md) for the crack attack |
| **Rocket Jockey** | [05_GRAPPLE_ROPES](surveys/05_GRAPPLE_ROPES.md) **§4.6** | its own full design; the hardest case in the survey |
| **Zelda hookshot, Thief rope arrows** | [05_GRAPPLE_ROPES](surveys/05_GRAPPLE_ROPES.md) §4.1, §4.4 | §3.1 for the wall-normal function |
| **Interstate 76, Matrix highway chase** | [02_VEHICLES](surveys/02_VEHICLES.md) §4.1, §4.9 | **§4.3a — a steering wheel is inside the trap** |
| **Motocross, ExciteBike** | [02_VEHICLES](surveys/02_VEHICLES.md) §4.5, §4.6 | bike bars are safe by construction; see §4.3a |
| **Lucky & Wild** | [02_VEHICLES](surveys/02_VEHICLES.md) §4.10 | uses the shipped two-weapon system |
| **Mechs, cockpits** | [03_MECHS](surveys/03_MECHS.md) §4 | §0 the view-height hinge — one launch settles it |
| **Bullet time, the Matrix lobby** | [10_TIME](surveys/10_TIME.md) §4.1 | buildable today in pure ZScript |
| **Gravity gloves, throwing, human shields** | [06_GRAB_INTERACTION](surveys/06_GRAB_INTERACTION.md) | §2 interpolation decides whether it feels solid |
| **Bows, crossbows, grenades, reload verbs** | [08_BOWS_OFFHAND](surveys/08_BOWS_OFFHAND.md) | draw power = \|AttackPos − OffhandPos\| |
| **Shields, parry, Captain America** | [07_SHIELDS_PARRY](surveys/07_SHIELDS_PARRY.md) | the Hexen Centaur already ships a timed parry |
| **Dashes, slides, wall running** | [09_MOVEMENT](surveys/09_MOVEMENT.md) | §0 two traps that kill every momentum mechanic |
| **Flight, jetpacks, Iron Man** | [04_FLIGHT_6DOF](surveys/04_FLIGHT_6DOF.md) | translation is cheap; rotation is a hard wall |

---

## Every mechanic, by label

**EXISTS** — already in the engine · **BUILDABLE** — script, with named APIs ·
**HACK** — works but abuses something · **WALL** — needs an engine change
Cost: **S** an afternoon · **M** a few days · **L** a week or more

### Fighting and blades — [01_MELEE](surveys/01_MELEE.md)

| Mechanic | Label | Cost |
|---|---|---|
| Two-hand tracking, per-hand weapons and buttons, hand-origin traces | EXISTS | — |
| `freezetics` hit-stop, forced pain, physical duck and lean | EXISTS | — |
| Punch detection and classification (jab/cross/hook/uppercut, body vs head) | BUILDABLE | M |
| Orthodox vs southpaw stance | BUILDABLE | S |
| Guard, slips, weaving | BUILDABLE | S |
| Combos, timing windows, counter-punches | BUILDABLE | M |
| Stagger, knockdown, wall bounce | BUILDABLE | M |
| Grabs, throws, knees, stomps | BUILDABLE | M |
| Blades: edge vs flat, thrust vs slash, clash, parry, deflection | BUILDABLE | M |
| Punch-Out opponents as their own classes | BUILDABLE | M |
| Per-hand rumble via `A_QuakeEx` | HACK | S |
| Slip *direction* | HACK | S |
| Knockdown retrofitted onto third-party monsters | HACK | L |
| Sub-tic hand history (curved gestures only) | WALL | — |
| `weaponStabilised` cannot be suppressed | WALL | — |
| Play-space yaw · prone model · finger tracking · analogue trigger | WALL | — |

### Vehicles — [02_VEHICLES](surveys/02_VEHICLES.md)

| Mechanic | Label | Cost |
|---|---|---|
| Speed (no practical cap in normal play) · traffic as actors | EXISTS | S |
| **Vehicle as player movement override** ← the supported shape | BUILDABLE | M |
| Two-handed handlebars / wheel · throttle on per-hand triggers | BUILDABLE | S |
| Hand-mounted handlebar models | BUILDABLE | S |
| Lean from `HmdPosition` · wheelies, jumps, landing angle | BUILDABLE | M |
| Interstate 76 hardpoints · Lucky & Wild · modular damage · car-as-mech | BUILDABLE | M |
| Riding a separate vehicle actor (strictly worse than the override) | HACK | L |
| Jumping between moving vehicles | HACK | L |
| Head-locked cockpit · analogue trigger value · rider carry | WALL | — |

### Mechs — [03_MECHS](surveys/03_MECHS.md)

| Mechanic | Label | Cost |
|---|---|---|
| Weapon hardpoints per arm (shipped offhand system) | EXISTS | — |
| Player scale and view height · heat · stomps · enter/exit via `MorphPlayer` | BUILDABLE | S |
| Cockpit built from spawned actors · grabbed levers · legs-follow-torso · per-limb damage | BUILDABLE | M |
| Cockpit as a psprite · screen shake / bob / view kick · native second facing | WALL | — |
| `player.camera` on a non-player actor (kills all VR input) | WALL | — |

### Flight — [04_FLIGHT_6DOF](surveys/04_FLIGHT_6DOF.md)

| Mechanic | Label | Cost |
|---|---|---|
| Plane-style "fly where you look" · throttle/stick split | EXISTS | S |
| Jetpack · hover | BUILDABLE | S |
| Iron Man palm flight, both hands | BUILDABLE | M |
| View pitch/roll in VR · room-scale Z · Descent-style zero-g | WALL | — |

### Grapples, ropes, whips — [05_GRAPPLE_ROPES](surveys/05_GRAPPLE_ROPES.md)

| Mechanic | Label | Cost |
|---|---|---|
| Hookshot, player-pull | BUILDABLE | S |
| Hookshot including enemy-pull · whip · rope arrows · hand-over-hand | BUILDABLE | M |
| Swinging pendulum with momentum carry | BUILDABLE | M |
| **Rocket Jockey** — moving anchor, tension, snap | BUILDABLE | L |
| Corner-wrapping rope | BUILDABLE | L |
| Rope rendering (segment chain; no line primitive) | HACK | M |
| Haptics · multiplayer hand-driven ropes | WALL | — |

### Grabbing and interaction — [06_GRAB_INTERACTION](surveys/06_GRAB_INTERACTION.md)

| Mechanic | Label | Cost |
|---|---|---|
| Human shields · gravity-glove ray (`TRF_USEWEAPON`) | EXISTS | — |
| Windows through two-sided lines | BUILDABLE | S |
| Grab inert objects · grab live enemies · throwing · catching projectiles | BUILDABLE | M |
| Hand-driven door and switch activation | BUILDABLE | M |
| Physically pushing or pulling doors | HACK | M |
| Breaking through one-sided walls (no runtime geometry) | WALL | — |

### Shields and parry — [07_SHIELDS_PARRY](surveys/07_SHIELDS_PARRY.md)

| Mechanic | Label | Cost |
|---|---|---|
| Projectile interception · hitscan interception · dynamic resizing | EXISTS | S |
| Parry with reflection (`P_ReflectOffActor`) | EXISTS | S |
| Shield bash (`LAF_ISOFFHAND`) · deployable shield · durability | BUILDABLE | S |
| Plate following the hand · thrown and returning shield · sword-and-board | BUILDABLE | M |
| Melee block (player-side cone) | HACK | S |
| Blocking line of sight, melee, splash, or aggro · reflecting hitscans | WALL | — |

### Bows and offhand verbs — [08_BOWS_OFFHAND](surveys/08_BOWS_OFFHAND.md)

| Mechanic | Label | Cost |
|---|---|---|
| Grenade pin-pull · lighter dwell · flare strike · shotgun pump · slide rack | BUILDABLE | S |
| Crossbow cranking | BUILDABLE | S–M |
| Bow nock / draw / release · magazine insertion | BUILDABLE | M |
| `bTwoHanded` for a bow (it deletes the hand you need) | WALL | — |

### Movement — [09_MOVEMENT](surveys/09_MOVEMENT.md)

| Mechanic | Label | Cost |
|---|---|---|
| Ground dash · air dash · backstep · double jump | BUILDABLE | S |
| Slide | BUILDABLE | M |
| Wall run | BUILDABLE | L |
| Eye-space vignette · view pitch/roll/yaw · room-scale Z | WALL | — |

### Time — [10_TIME](surveys/10_TIME.md)

| Mechanic | Label | Cost |
|---|---|---|
| Global freeze with hand-picked exceptions (`Level.SetFrozen` + `timefreezer`) | BUILDABLE | S |
| Per-actor slow via `freezetics` duty cycle | BUILDABLE | S |
| Catching a bullet · two-handed melee in slow-mo | BUILDABLE | M |
| Bounded replay of a few actors | BUILDABLE | L |
| Global time scale (`i_timescale` unreachable from script) | WALL | — |
| Rewind · smooth slowed motion (no interpolation stretch) | WALL | — |

---

## The engine asks, ranked

Every report was told to name the missing piece behind a WALL. These came up most:

1. **A ZScript binding for `VRMode::Vibrate`** (`hw_vrmodes.h:94`). Named by five reports as the
   highest-value addition. The implementation already exists and is cvar-gated — this is a
   thunk, not a feature. Something like `PlayerInfo.Vibrate(hand, duration, intensity)`:
   additive, default-inert, useful to every mod.
2. **A script-settable view-angle offset applied after the HMD pose is read.** Unlocks screen
   shake, camera kick, vehicle and mech feel, dash roll — currently all walls.
3. **A mod-settable gate on the two-hand stabilisation condition** (`gl_openvr.cpp:1717`).
4. **Per-actor interpolation stretch** — would make slow-motion smooth instead of strobing.
5. **Runtime geometry creation** — breaking through walls, real rope rendering.

---

## Before anyone commits a week

From `00_CAPABILITIES.md` §0, and it takes five minutes:

1. Confirm whether the target binary was built with `USE_OPENVR`, `USE_OPENXR`, or neither.
   On PC the OpenXR path **is not compiled at all** (`USE_OPENXR` is defined only in
   `mobile/Android_src.mk:15`), and OpenVR is off by default (`src/CMakeLists.txt:418`).
2. Print `AttackPos` and `OffhandPos` from `DoEffect` for a few tics in the real build. They
   must move, and they must differ from each other. If they are equal you are on the non-VR
   fallback and every hand-driven design here reads the viewpoint instead of a hand.

---

*Surveyed against tag `qzd17.3` of github.com/emawind84/gzdoom. The engine was read, never
edited. Where a report could not find something, it says so rather than guessing.*
