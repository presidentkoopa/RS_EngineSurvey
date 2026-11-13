# Corrections to VR_MELEE_ON_QUESTZDOOM_2026-10-24.md

The document itself is kept exactly as written. Its corrections live here.

## 1. `A_QuakeEx` intensity is NOT an int. Sub-1.0 rumble is not silent. (2026-11-02)

**The claim:** Part Two §8, ZScript and the VM: "`A_QuakeEx` intensity is an int. X = left
controller, Y = right. [verified] — `r_utility.cpp:1085`. Several shipped sub-1.0 strengths may
therefore be silent (unresolved)." §10 lists the same point as not established.

**What the source says.** QuestZDoom 17.3 (emawind84/gzdoom `9afc750917bf54870f2d16b7ba0d00f2b77e6fb5`,
tag `qzd17.3`, the commit the RSQ_WeaponDepot test rig pins). The intensity is a double the whole way
from the script call to the controller:

| Step | Where | Type |
|---|---|---|
| The ZScript declaration | `wadsrc/static/zscript/actors/actor.zs:1284`: `A_QuakeEx(double intensityX, double intensityY, double intensityZ, int duration, ...)` | double |
| The native reads it | `src/playsim/p_actionfunctions.cpp:3405-3407`: `PARAM_FLOAT(intensityX)` / `Y` / `Z` | double |
| The quake stores it | `src/playsim/mapthinkers/a_quake.cpp:63`: `m_Intensity = DVector3(intensityX, intensityY, intensityZ)` (`P_StartQuakeXYZ`, `:431`, takes doubles) | double |
| The haptic reads it | `src/rendering/r_utility.cpp:1086-1087`: `double left = QuakePower(vr_quake_haptic_level, jiggers.Intensity.X, ...)`, and the same for `right` | double |
| It reaches the controller | `r_utility.cpp:1090-1091`: `vrmode->Vibrate(10, 0, (float)left)` and `(10, 1, (float)right)` | float |
| `QuakePower` | `r_utility.cpp:911-922`: `factor * (offset + random(-intensity, +intensity))`, with no rounding | double |

So a strength of 0.3 arrives as a pulse of up to 0.3 × `vr_quake_haptic_level`. The real limits
stand as the document describes them elsewhere: 10 ms pulses re-fired every tic, a random value in
±intensity each frame, the user's `vr_quake_haptic_level` (default 0.8, 0 turns it off), and camera
shake only if the user has raised `r_quakeintensity`.

**Who it affects:** SABER's cut rumble (0.9), SHIELD's grind, sweep and block buzzes, and Super
Punch-Out's Arcade rumble (`SPO_Rema.Buzz`, which divides its strengths by the quake gain). None of
them is silent on this account.

*Checked by reading the source at the pinned commit. Not measured in a headset: the rig's
`QzDoom_Vibrate` is an empty stub (`tools/rig/engine/glue/qz_linux_glue.cpp:125`).*
