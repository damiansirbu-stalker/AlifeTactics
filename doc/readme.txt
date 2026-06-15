AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: 1.0.0 (xlibs 1.7.6, demonized 20250908, AOEngine v0.55)
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

AlifeTactics is a collection of fixes and systems focused on NPC combat behavior in STALKER Anomaly.
Defaults match vanilla so the fixes apply on their own with no behavior shift you don't want.

Hit Sharing:

On the first faction-enemy hit, AlifeTactics arms every online squadmate against the shooter, audio range or not. \
Every member's personal goodwill toward the shooter is forced to hostile, the shooter is registered in every member's memory, and the squad's combat-mask bit
is set so the engine's own memory propagation carries the hit across the rest of the squad. 
New squad members spawned mid-fight inherit the squad's active disclosures, and previously offline shooters coming back online get re-disclosed
to the squads tracking them, so engagement state stays consistent across spawn churn.
After 2 game minutes of no further hits from that shooter the squad's pin expires and the next hit re-fires the alert.

The Stealth toggle (MCM, default on) suppresses the squad alert when the hit kills the victim.
Silenced shots, scoped-rifle headshots, and backstabs no longer disclose the shooter to the surviving squadmates.
Squadmates still learn through gunshot sound, corpse discovery, and line of sight.

Healing:

Vanilla and most modpacks rely on a magic medkit use that triggers unreliably, and bandages don't work at
all. About half of NPCs get a one-shot flag at register that lets them heal HP once without consuming any
item, and the flag re-rolls on every save load. Bandages have no equivalent fallback, so bleeding NPCs
in vanilla bleed out unless they happen to have a bandage AND the inventory list is populated. Vanilla
doesn't populate it. AlifeTactics fixes both.

Wounded stalkers below 50% HP consume medkits from their inventory. Bleeding stalkers above the wound
threshold consume bandages. Stalkers carrying neither fall back to a per-rank lifetime healing charge.
The heal rate is MCM-tunable.

Animations were also fixed and are toggleable in MCM. Stalkers below 65% HP visibly limp when out of combat
through a torso overlay the engine layers over normal locomotion, re-armed every 5 seconds per NPC. A
medkit-injection or bandage-application torso animation plays as a one-shot cue when a stalker starts a
heal cycle.

Accuracy:

The engine's rank-based accuracy curve is dead on Anomaly's rank values, making all stalker ranks equal in practice.
AlifeTactics hooks engine internals and allows that to function, while also making it configurable.

AlifeTactics hooks the engine callbacks and applies a per-rank dispersion factor to every NPC shot, MCM configurable.
Masters shoot tighter than novices, and the full spread is configurable per tier. 

Combat:

This is the core of AlifeTactics. In a firefight, every move a stalker makes - posture, where it walks, when it shoots - is run by the engine's built-in combat planner. AlifeTactics takes that planner over: on the NPCs in its share it blocks the engine's combat and danger planners outright and drives the NPC itself. It is a full takeover, not a tweak layered on top of vanilla, which is what lets it give NPCs coherent faction tactics instead of the usual sit-in-cover-and-barely-shoot.

It coexists with everything. A slider in MCM (default 100%) picks which NPCs run AlifeTactics combat and which stay on vanilla or your modpack's combat, by a stable per-NPC hash so the same NPC always lands the same side across save and load. AlifeTactics overrides no combat scripts, so out-of-share NPCs are completely untouched - you can watch AT and vanilla NPCs fight in the same battle. Set the share to 0 to disable combat without removing the mod.

Your companions are left out of the takeover by default (MCM > Combat), so they keep their own follow and combat behavior under your command. Turn the toggle off to let companions fight with AlifeTactics doctrine too.

Each faction fights to its own doctrine. Military, Duty, Freedom, ISG, mercenaries, and Monolith fight from cover and keep firing while they reposition, steady under fire, and outdoors they flank: breaking toward the enemy from the side to close the distance, or running wide to come at an enemy they have lost sight of. Monolith presses forward firing and never retreats. Bandits, renegades, and Sin come head-on. Loners, ecologists, and Clear Sky fight more cautiously and fall back when badly hurt. Zombies walk straight at the enemy firing, no cover, no retreat. The set of moves an NPC can pick from is filtered by its faction, its weapon class, and whether the fight is indoors or out, and one is drawn at random, so squadmates spread across the options instead of all reacting the same way. At the start of a fight each NPC commits to an opening move - hold and fire, take cover, advance, or flank - then reacts to how the fight develops.

Each NPC keeps its own weapon on you. Because taking the planner over also removes the engine's own aimer, AlifeTactics re-points the weapon at the enemy itself, at roughly human reaction speed - so NPCs track a moving target and trail a strafe instead of snapping to where you were or holding a frozen aim. They are not aimbots; the reaction lag is deliberate and tunable.

Snipers (the rifles the game classes as sniper weapons - SVD, SVT40, Mosin, SKS, M82 and the rest) get their own behavior: with line of sight they crouch and switch to the engine's sniper-aim mode, which aims down the head line instead of the looser barrel line for tighter shots at range.

Friendly fire between same-faction NPCs is blocked by default (MCM > Combat). An NPC that hits a teammate of its own faction does no damage, so squads stop dropping their own in crossfire. A slider scales it back up to full vanilla damage if you want it. Your own shots are never touched.

Danger:

A full-file override of Anomaly's xr_danger.script with bug fixes always-on and three improvements toggleable in MCM > Danger.

Six bug fixes are always on.
Three danger categories (direct hit, bullet ricochet, attacked nearby) were silently reading the wrong config row and reacting identically.
Mutant corpses crashed the evaluation on death-time reads.
The evaluator crashed when called on a torn-down NPC reference.
The evaluator also crashed when corpse death-time returned a non-numeric value.
Danger-state transitions reset only the upper-body animation tier and left a stale lower-body pose visible across the change.
The hit callback referenced an undefined variable and silently dropped responses on every hit.

Danger-state transitions snap into place instead of soft-blending so the reaction delay across state changes is gone.

The paired xr_danger.ltx widens corpse inertion to 15 game minutes (vanilla 12 seconds) and ricochet inertion to 10 minutes.
Detection distances respond to weather so stalkers see less in storms.

MCM-toggleable improvements (default on).
Hits override combat_ignore so allies of allies still get a danger response.
Gunshot reports register through the script danger pipeline gated on whether the actor is aiming.
Actor-sourced danger uses its own inertion and ignore tables so encounters with the player tune independently of NPC-vs-NPC.

Weapon Jam:

NPCs fire 2-3 rounds, jam, and reload in a tight loop, even from full-condition rifles. The Demonized modded exes add a per-rank jam roll on top of the engine's condition-based one, fired from ammo spent rather than weapon damage. AlifeTactics suppresses the script-side roll for NPCs (MCM > Fixes > Weapon Jam, default on). Actor jams from damaged weapons stay vanilla.

Requires demonized exes 2026.6.1 or later for this fix specifically. The engine hook it uses was added in that release. Loading on an older demonized or on AOEngine is harmless but does nothing.

NPC Ammo:

Vanilla NPCs always fire the first ammo type in their weapon's list. AlifeTactics promotes veteran-rank and higher NPCs who carry armor-piercing ammo to fire AP-class rounds with real ballistics (MCM > Fixes > NPC Ammo, default on).

A veteran or higher merc or military soldier carrying AP will fire AP at the player for a short window, then revert to vanilla magic FMJ when their virtual AP supply runs out. A novice bandit who looted an AP box still fires vanilla FMJ from it. The rank threshold and per-weapon drain rates are tunable in configs/alifetactics/at_ammo.ltx.

NPC corpses keep at least 3 rounds of each ammo box so the player loots the remainder instead of just the procedural spawn from death_manager.

The fixed AP section list covers vanilla Anomaly calibers (5.45x39, 5.56x45, 7.62x39, 7.62x51, 7.62x54, 7.92x33, 9x18, 9x19, 9x39, 12.7x55, 12x76). Add modpack calibers to the script's AP_SECTIONS table when needed.

Performance:

Most systems above hook on engine callbacks but are throttled and situational.
All operations are done through xlibs, which encapsulates best practices for working with the engine and what Anomaly uses internally.
All operations are also traced for performance and duration, viewable when debug logging is on.
Tests showed the performance impact is almost nonexistent.

Compatibility:

Tested with vanilla Anomaly 1.5.3 and GAMMA. Mid-save install and uninstall both work. Friendly fire and same-community hits are
filtered at the faction-relation gate, so story NPCs, companions, traders, and squadmates are never armed against their own faction.

AlifeTactics overrides one Anomaly script (xr_danger.script) and hooks engine APIs for self-healing, accuracy, and combat AI. Most
mods install alongside without issue. The cases below are worth knowing about.


Conflicts (pick one - running both breaks both):

NPC Limping and Healing (Vodoxleb). Drives the same limp and heal animations AlifeTactics does, through a different system. Both
running stacks animations and breaks the heal cue.


Load order matters (both safe to install, the one loaded later wins):

ReDone Combat AI, G.A.M.M.A. AI Rework. Both also override xr_danger.script. Put AlifeTactics later in MO2 if you want AlifeTactics's
xr_danger fixes; put it earlier if you prefer the other mod's xr_danger.


May have issues (untested combinations, listed because they touch the same systems):

Animated NPC Healing, NPC Animation Overhaul Part 1. Replace NPC heal/movement animations through different paths. May fight
AlifeTactics's heal and limp torso cues.

Wuut AI Extension, NPC_Fleeing. Add forced-movement actions to the combat planner. AlifeTactics blocks the vanilla combat planner
for the NPCs in its share, so these mods may not drive in-share NPCs as their authors intended. Out-of-share NPCs are unaffected.
If you want full Wuut or NPC_Fleeing behavior, set AlifeTactics's combat share to 0 in MCM.

Requirements:
Anomaly 1.5.3
demonized 20250908+ (https://github.com/themrdemonized/xray-monolith) OR AOEngine v0.55+ (https://github.com/Mirrowel/AOEngine-Assets)
xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Install (MO2):
1. Install xlibs
2. Install AlifeTactics
3. Load order does not matter
4. Configure via MCM

Uninstall (MO2):
Disable or remove in MO2.

Credits:
Altogolik - support, ideas, source materials

Usage and License:
Modpacks: allowed and encouraged. Keep the readme and license files.
Addons, patches, integrations: allowed. Credit "AlifeTactics by Damian Sirbu" visibly on your mod page.
Reproducing the implementation in other software: not allowed, even with credit.
Full license in LICENSE file and on GitHub.
