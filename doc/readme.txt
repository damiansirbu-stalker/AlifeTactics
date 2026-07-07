AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: 1.0.0 (xlibs 1.7.6, demonized 20250908)
GitHub: https://github.com/damiansirbu-stalker/AlifeTactics
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeTactics/issues

Alife Collection:
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics

! Reset MCM settings to defaults after updating !

https://www.youtube.com/watch?v=eKpzbmFOFC8

It wants you dead. It won't cheat to get there.

AlifeTactics makes the Zone fight back, and it never cheats to do it.
Difficulty is not a raised number. It is built from systems that scale and interact.
Rank drives how well a stalker aims, what ammunition he spends, and how readily he heals.
NPCs fight with what they actually carry, looted and traded through the Alife Collection rather than granted.
These advantages stack, so a threat is a product of the situation, not a slider.
The shortcuts are gone from both sides. NPCs heal with real medkits, fire the rounds they loaded, and act only on what they have seen or heard.
Range no longer buys a free shot, a wounded man's squad turns on the source, and the frozen or forgetful behavior you used to exploit is fixed.
A stalker also stops reacting to one cue in isolation.
His move is weighed against his own condition, the enemy's range and whether it closes, the cover around him, his weapon, and his faction's temperament.
Even the darkness and the weather count, setting how far he can see.
Steady factions hold and take cover, cautious ones break and run, fanatics never break.
Twitching in place becomes a committed maneuver: give ground, take cover, hold a sniper's line, or rout.
Every fight is assembled from these systems, not a script, so no two play out the same.
It is unpredictable, and it is fair, which is the hardest kind of fight there is.

Every system below has its own page in MCM, laid out in this exact order, where it is toggled and tuned.

Combat

Maneuvers:
Vanilla stalkers re-decide the fight every frame and end up dithering, ducking and popping as they catch and lose sight of you.
AlifeTactics borrows a single stalker for one committed maneuver, then hands control back, so the move actually finishes.
A maneuver fires only when the ground, the ranges, and the stalker's own state say it gains him something.
Four maneuvers cover the moments vanilla fumbles.
  Kite backs a stalker out of an enemy that closed too near, still firing.
  Retreat pulls a steady faction to cover under pressure.
  Flee routs a cautious one to a distant friendly base.
  Snipe holds a stalled marksman on precision aim.
The decisions come out looking human. Nobody turns his back on a shooter at arm's length. A rout picks a base away from you, so every stride gains distance. Two men never take the same cover. When no move pays, AlifeTactics leaves the stalker alone and vanilla fights on.
A borrowed stalker fires with the engine's own aim, but holds fire with no clear shot, and lowers the weapon only to run.
The takeover overrides no combat scripts, so it fights side by side with vanilla and layers cleanly over AI overhauls.
Companions sit it out by default.

Behaviors (planned):
New actions the vanilla planner lacks, like grenades and melee.

Effectiveness

Accuracy:
Masters finally out-shoot novices.
Anomaly's rank curve clamps every NPC to the same dispersion, so rank never touched aim.
AlifeTactics restores a real per-rank curve through the engine's own dispersion callback, tunable per tier.

Disclosure:
A firefight spreads through a squad the way it should, and a clean kill stays quiet.
Wound one stalker and the whole squad turns on you, even a patrol out of earshot or the victim of a silenced round.
AlifeTactics works through the engine's own memory: forced goodwill, a registered shooter, and the squad's combat-mask set.
The engine propagates the contact from there.
New members inherit the fight on spawn, and a shooter who logs out and returns is flagged again.
A survivor gate (default on) keeps a kill that drops the target outright from disclosing you, so silenced shots and backstabs stay silent.
The squad can still find you the honest way, by sound, by sight, or by the body.

Danger:
Stalkers read danger the way the engine always meant them to.
AlifeTactics reworks Anomaly's danger scheme as a runtime patch, laid onto whichever danger script a modpack ships instead of replacing the file.
A set of always-on fixes clears long-standing crashes and misreads across the danger check and the corpse investigation.
The paired config scales detection with weather, so stalkers see less in a storm, and gives player encounters their own tables.
Three of the improvements are optional: a direct hit answers at any range, nearby gunfire draws a response, and actor-sourced danger reads the separate tables.

Crossfire:
Squads stop dropping their own in the middle of a firefight.
A stalker that hits a same-faction teammate deals no damage, keyed on relation rather than community.
A soured cross-faction pair still trades fire, while true allies stay safe.
A slider scales it back toward vanilla, and your own shots are never touched.

Commitment (planned):
NPCs hold a good action instead of shuffling between cover, on the demonized action-switch hook.
The engine hook is merged, the layer is next.

Reaction (planned):
Per-NPC aim speed and vision.

Mechanics

Healing:
Wounded NPCs actually heal, with the items they carry.
Vanilla's magic medkit fires unreliably and bandages do nothing, so bleeding stalkers die that should not.
AlifeTactics has them spend real medkits below half health and real bandages when bleeding, falling back to a per-rank charge only when empty.
The heal rate is tunable, and fixed limp and heal animations show it, out of combat only.

Jamming:
NPC rifles stop jamming every few rounds.
The demonized exes roll a per-rank jam from ammo spent, so even a pristine rifle chokes and reloads in a loop.
AlifeTactics suppresses that roll for NPCs, while your own jams from a worn weapon stay vanilla.
It needs demonized 2026.6.1 or newer and stays inactive on older builds.

Ammo:
The armor-piercing round that kills you was looted, not spawned.
Veteran and higher NPCs fire the AP they actually carry, with real ballistics, until it runs out and they fall back to standard rounds.
That ammo comes from the world through the Alife Collection, so who scavenged what genuinely changes the fight, and NPCs drop no AP as loot.
Rank threshold and drain are tunable in configs/alifetactics/at_ammo.ltx.

Range and Resistance under Effectiveness, and the Effects and Mutants categories, are reserved in the menu for systems still to come.

Under the hood:
Every system rides the engine's own callbacks, event-driven and effectively free at runtime, and it replaces no files.
Compatibility falls out of that: nothing is replaced, so nothing collides.
Every operation is traced for duration, visible with debug logging on.

Intended setup:
AlifeTactics is designed against vanilla Anomaly on the latest demonized build, and that is what it is tuned for.
It runs on AOEngine and older demonized builds with fallbacks, where a feature that needs a newer hook stays inactive or reduced.
It coexists with AI-overhaul combat mods, but vanilla plus AlifeTactics is the intended experience.

Requirements:
Anomaly 1.5.3
Modded exes (themrdemonized 20250908 or newer, or AOEngine v0.55 or newer)
xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Install (MO2):
1. Install xlibs
2. Install AlifeTactics
3. Load order does not matter
4. Configure via MCM

Uninstall (MO2):
Disable or remove in MO2.

Compatibility:
Tested with vanilla Anomaly 1.5.3 and GAMMA. Mid-save install and uninstall both work.
The takeover leaves combat scripts vanilla, and the danger rework patches at runtime, so AlifeTactics layers cleanly onto other combat and AI mods.
Friendly fire and same-community hits are filtered at the faction-relation gate.
Story NPCs, companions, traders, and squadmates are never armed against their own faction.

Conflicts (choose one):
- NPC Limping and Healing (Vodoxleb): the same limp and heal animations from a different system. The two stack and break the heal cue.
- NPC Weapon Jamming, or any rank-based NPC jam mod: Anomaly has no jam animation, so jammed NPCs reload forever. AlifeTactics removes it, the mod re-adds it.

Coexists:
- AI-overhaul combat (G.A.M.M.A. AI Rework, ReDone Combat AI, RE:VISION, AI More Cover):
  The takeover blocks the combat planner only while it runs a maneuver, then hands back to their combat AI.
  The danger rework patches their xr_danger at load, so their danger layer keeps running too.
- Wuut AI Extension, NPC_Fleeing, Mora's AI More Covered: add planner actions.
  While AlifeTactics runs a maneuver its own action is suppressed for those seconds, then resumes.
  Disable Combat in MCM to leave them fully in charge.
- Mods that replace NPC healing with their own path (Animated NPC Healing and similar):
  Their healer runs, so AlifeTactics's heal rate and per-rank charge do nothing where they overlap.
  Disable one side. A DLTX mod that rewrites the vanilla medkit list can also empty AlifeTactics's list, by load order.
- No More Companion Friendly Fire: a different axis (actor-companion damage). AlifeTactics never touches your shots.
- Tougher Important NPCs and Companions: damage reduction through npc_on_before_hit, composes.
- Dynamic AI Aim Settings, Worse NPC Vision and Accuracy: perception tweaks that compose with the dispersion fix.

FAQ:
Do I need modded exes?
  Yes. AlifeTactics needs themrdemonized modded exes (20250908 or newer) or AOEngine (v0.55 or newer). Vanilla Anomaly does not expose the APIs it relies on.

Credits:
Altogolik - support, ideas, source materials

Usage and License:
Modpacks: allowed and encouraged. Keep the readme and license files.
Addons, patches, integrations: allowed. Credit "AlifeTactics by Damian Sirbu" visibly on your mod page.
Reproducing the implementation in other software: not allowed, even with credit.
Full license in LICENSE file and on GitHub.

Reporting issues and suggestions:
Open a bug report or a suggestion at https://github.com/damiansirbu-stalker/AlifeTactics/issues/new/choose.
Also discussed on the GAMMA, EFP, Anomaly, and Zona Discord servers.

Before posting, read this readme and the MCM options.

AlifeTactics is a sophisticated mod that works deep in the engine, and combat is one of the hardest parts of Anomaly to reason about.
Before you report a combat or AI issue as AlifeTactics, confirm it is actually AlifeTactics.
The surest test: reproduce it, then disable AlifeTactics and reproduce again.
If the behavior stays, it is not this mod.
A minimal setup is best: vanilla Anomaly plus xlibs plus AlifeTactics, where nothing else can be the cause.
Much of what gets reported against combat mods turns out to be the engine, the modpack, or another mod.
The log is what tells them apart.

Include:
- Exact steps to reproduce, from a new game or a named save, with expected and actual result.
- Confirmation that the issue disappears with AlifeTactics disabled.
- xray.log and the mod debug log (MCM log level DEBUG), plus engine build, modlist, load order.
- Describe the behavior. With hundreds of mods and overrides, only the log shows whether this mod was involved and what caused it.
