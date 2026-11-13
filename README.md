# RS Engine Survey

Code-cited technical teardowns of the engines behind the Radiant Silvergun Doom VR work.
Every claim in these documents points to `file:line` in the engine's own source; READMEs and
design notes are treated as leads, never as evidence.

| Folder | Engine | Source read | What it answers |
|---|---|---|---|
| [QZD_17.3/](QZD_17.3/) | QuestZDoom 17.3 (emawind84, tag `qzd17.3`) | Quest standalone engine | What can be built as a mod on this exact engine, and how |
| [Helion_1.2.0/](Helion_1.2.0/) | Helion 1.2.0 (C#, OpenGL) | `Helion-Engine/Helion` at `5edd437` | How a Doom engine reaches its published frame rates: the reference design |
| [UZDXREMA_0.1.0/](UZDXREMA_0.1.0/) | UZDXREMA (UZDoom 5.0 + DoomXR VR, Vulkan, OpenXR) | branches `main`, `renderer`, `netplay`, `selaco` | How it runs, every hot path, hooks and dead code, what slows it down, measured; the 90 Hz gameplan, implementer checklist and sim lane handoff |
| REMAROTH_x.y.z/ | REMAROTH (Realms of the Haunting as a native VR game) | planned | |
| REMASELACO_x.y.z/ | REMASelaco (engine fork running Selaco natively) | planned | |
