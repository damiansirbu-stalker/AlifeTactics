AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: 1.1.5 (xlibs 1.8.2, demonized 20250908)
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

AlifeTactics is a platform of systems that take how creatures behave and fight in STALKER to another level.
(Note that the readme covers the final shape of the mod, currently it is less than half finalized)

Every AlifeTactics system works the same way: read the real state of the game, then decide intelligently in the combatant's favor, not by script or die roll.
It weighs combat events, NPC and player stats, the world, squad, faction, weapons, range, angle, and cover.
Maneuvers take a stalker over for an action the vanilla engine has no mechanism for.
Commitment keeps a stalker on a decision that still makes sense instead of the engine's constant re-planning.
Threat, accuracy, and the rest read state and decide the same way, and the direction is that every layer will.

The actual effects are
- All creatures will fight and behave much more intelligently, making emergent decisions based on environment, own state, enemy state, squad state, weapons, faction doctrine etc
- Nothing is player centric, everything is fair and there are no advantages or disadvantages given to anyone
- Creatures use real items they acquire themselves (medkits, ammo, props)
- Combat prowess categories that should have scaled by rank (accuracy, weapon spread, aim speed and others) now work for real and without hacks or side effects
- Features that never worked or were hijacked, now work for real (eg AP ammo is used by whoever owns it and is consumed, vanilla has NPCs using infinite cheapest ammo, modpacks have infinite AP ammo per rank and map)
- Bugs disguised as features, like fake weapon jamming or random tactical reloading are eliminated


Everything is 100% canon, engine-pure, safe and fast:
- Every behavior is built from pure X-Ray and Anomaly primitives: the GOAP action planner, the xr_logic scheme system, state_mgr, smart terrains and the gulag job system, condlists, DLTX/DXML for data. Nothing is faked, nothing is simulated beside the engine.
- Where the engine had no seam, the seam was added upstream first: per-NPC hooks created in xray-monolith specifically for this mod, merged into the official modded exes.
- It never replaces a vanilla script file. Takeovers block the planner only for their seconds and hand back; patches lay onto whichever script your setup ships.
- Every value ships as a formula over the engine's own constants, never an invented number.
- Zero per-frame Lua. Everything runs on engine callbacks and scheduled passes; the whole mod costs 3 timer compares per frame, whatever the NPC count.
- Dozens of vanilla Anomaly bugs fixed along the way: dead branches, wrong calculations, plain crashes in original code.


Every system below has its own page in MCM, laid out in this exact order, where it is toggled and tuned.

Combat

Maneuvers:
A maneuver is a complete, authoritative takeover of one stalker to perform what the vanilla engine cannot, whether as a mechanic it lacks or a decision it never makes.
The system reads combat events, NPC and player stats, and the world, then makes an intelligent decision in the combatant's favor.
It weighs squad and faction, squad and enemy state, range, angle, cover, and weapons.
The design goes further than the single stalker: whole squads coordinating their movement and fire, each faction fighting in its own flavor and favoring the maneuvers that suit it.


These maneuvers cover the moments vanilla fumbles (wip, many will come):
  Counterflank snaps a stalker around when a hostile player stands at contact range while he shoots someone far away.
  Reload Cover sends a stalker caught reloading under fire to the nearest cover instead of standing in the open; his weapon finishes reloading on the way.
  Flee routs a cautious faction to a distant friendly base - a coward runs before he fights.
  Retreat pulls a steady one to cover under pressure; a coward whose escape is cut off does the same.
  Kite backs a stalker out of an enemy that closed too near, still firing.
  Snipe holds a stalled marksman on precision aim.
Maneuvers run only in fights the player is part of; NPC-only fights stay fully vanilla.
A maneuver fires when its problem is real and ends when it is solved; if you keep creating the problem (keep pressing a shotgunner's minimum range), the answer keeps coming.
The decisions come out looking human. Nobody turns his back on a shooter, two men never take the same cover, a sniper never plants under your crosshair.
Maneuver fire bursts by weapon: pistols and rifles fire real bursts, sniper rifles single shots, in place of the uniform burst the game's script machinery uses for every weapon it drives.
The takeover overrides no combat scripts, so it fights side by side with vanilla and works with other combat AI instead of replacing it. Companions are excluded by default.

Commitment:
Vanilla stalkers re-plan the fight every moment, so any small change makes a stalker drop what he is doing and choose again.
Better cover, a flicker of lost sight, a teammate crossing the line - and off he goes, the twitchy strafing and cover-hopping you see in a firefight.
Many mods answer this by switching the stalker to a camper scheme that pins him in place, muting most of Anomaly's combat variety.
AlifeTactics keeps the engine's full combat AI and instead stops a stalker throwing away a decision that still makes sense.
While what he is doing still works he stays with it, and he switches the instant it stops: he loses sight, the shot is blocked, or the enemy is gone.
It also pins his cover: a stalker firing with a clear shot keeps his spot instead of sliding to a marginally better one, stopping the mid-fight strafe at its source.
The result is a stalker who sees a decision through - keeps up his fire, presses a flank, finishes a reload - instead of second-guessing himself every frame.
Based on action-switch and cover-repick hooks created in xray-monolith for this mod; on older builds the system stays inactive.

Behaviors (planned):
New actions the vanilla planner lacks, like melee, hopefully peek etc.

Effectiveness

Accuracy:
Anomaly's rank curve clamps every NPC to the same dispersion, accuracy never scaled per rank, even though the engine code exists.
AlifeTactics restores a real per-rank curve through the engine's own dispersion callback, tunable per tier.
A second curve covers fire on the move: each rank keeps a share of the movement spread penalty, so rookies spray while repositioning and top ranks lose about half the penalty.
Note that the engine has many dispersion variables (eg barrel, weapon, npc), this is the NPC skill-based dispersion.

Disclosure:
When a faction enemy wounds any squad member, the whole squad learns the shooter at once, even patrol members out of earshot and even against a silenced weapon.
Vanilla only shares a shooter the squadmates actually heard, so distant or suppressed attackers stay unknown.
It works through the engine's own memory, not by faking relations: the shooter is enabled in each member's memory and registered into the squad's combat, and the engine's normal propagation carries it from there.
On a recent engine build the shooter also enters their seen memory, the class target selection ranks first, so the squad turns on the shooter instead of staying locked on a distant visible enemy.
A stalker with no squad discloses the shooter to himself.
New members inherit the fight when they spawn, and a shooter who went offline and returns is flagged again.
A survivor gate (default on) skips disclosure when the hit kills the victim outright, so a clean kill stays quiet.
The squad can still find you by sound, by sight, or by the body.

Crossfire:
Same-faction fighters no longer cut each other down in a crossfire.
A hit between two NPCs of the same faction deals reduced damage, set by a slider (no damage by default, up to full vanilla).
It keys on their actual relation, so genuinely hostile factions still trade fire while allies stay safe.
Your own shots are never affected.

Reaction:
Stalker rank now shapes gun handling, applied per stalker.
Tracking Speed sets how tightly a barrel follows a moving target, from vanilla at the bottom rank to the game's own hardcore AI aim at the top.
Vision Speed sets how fast each rank notices a target at range, up to twice your setup's detection speed at the top.
Target Lead aims a stalker ahead of a moving target by the round's real flight time, computed from range and the weapon's bullet speed as it fires.
Higher ranks lead true and hit movers, lower ranks over-lead and overshoot.
Fire Discipline gives higher ranks crisper short bursts at a tighter cadence while low ranks stay vanilla.
Defaults keep a rank's rounds per minute at or above vanilla.
Tracking, Vision, Target Lead, and Fire Discipline are per-rank MCM slider curves, each with its own on/off and a short note on the toggle.
Based on per-NPC aim and vision hooks created in xray-monolith for this mod; on older builds the page has no effect.

Range (planned):
Engagement distance. The game stops NPC fights at a hard range cap regardless of weapon, so a sniper never fires at the distances his rifle exists for, and a stalker sniped from beyond the cap can duck (the Danger hit response) but never answer. This page will let long-range NPCs answer and initiate at their weapon's real reach.

Resistance (planned):
NPC toughness from what they carry: armor and equipment, damage mitigators, and artefacts.

Danger

Danger is how a stalker reacts to a threat he perceives - by sound, by hit, or (later) by sight. It is a runtime patch laid onto whichever danger script your modpack ships, not a replacement, so it works with other combat AI. Detection distances stay owned by your setup's danger config; AlifeTactics adds the reactions, never the tuning.

Sound:
Vanilla NPCs ignore gunfire they hear but cannot see. A hostile stalker now reacts to an enemy's shots without line of sight: he turns to face the gun and takes a threat stance, and moves to cover once he can see the shooter. He is responding to the sound, not to sight of you. Gunfire from neutral or friendly stalkers, including your own, does not alarm them - reactions follow the engine's relation rule, same as vanilla.
Hostile stalkers also hear you move. Each footstep carries by your stance, the surface, and the weather: sneaking is near-silent, sprinting on metal carries far, rain muffles everything, and a jump landing is the loudest thing you can do.
They react to handling noise too - a reload racked nearby, an empty click, an item used - at shorter reach.
A hostile who hears you turns to the sound and goes on alert; he does not magically know where you are. Nothing changes in combat, and a carry-distance slider scales it, so stealth stays a game of distance and stance instead of NPCs being deaf.
Neutral and friendly stalkers get one reaction of their own: fire close to them and they go weapon-ready facing your shots instead of ignoring gunfire next to their heads. Alert is all it is - they never turn hostile and settle down when the shooting stops.
Every reaction has its own toggle.

Hit:
A stalker hit from far off reacts even when vanilla would have him stand still - sniped at 200m, he turns and seeks cover. The reaction is the duck; returning fire at that range is the planned Range page above.

Fixes:
A set of always-on fixes clears long-standing crashes and misreads across the danger check and the corpse investigation; these are not optional. One tuning toggle lets danger you cause read your setup's separate player-specific ranges where the config provides them.

Mechanics

Healing:
Wounded NPCs actually heal, with the items they carry.
Vanilla's magic medkit fires unreliably and bandages do nothing, so bleeding stalkers die that should not.
AlifeTactics has them spend real medkits when injured and real bandages when bleeding, falling back to a per-rank charge only when empty.
The heal rate is tunable, and fixed limp and heal animations show it, out of combat only.

Jamming:
Eliminates the fake NPC jams and the tactical-reload loop they cause.
On modded exes an NPC rolls a per-rank jam from ammo spent, so even a full-condition rifle chokes and reloads every few rounds.
AlifeTactics turns that roll off for NPCs, so they now jam only for real and reload only once the magazine is spent, which you can confirm in the debug log.
Your own weapon still jams normally when worn.

Ammo:
NPCs fire the ammunition they actually carry, not infinite generic rounds.
Veteran-rank and higher stalkers use armor-piercing rounds from their own inventory, with real ballistics, and fall back to standard rounds once it runs out.
That AP comes from trade and looting through the Alife Collection, so what an NPC scavenged shapes how dangerous he is.
NPCs drop no AP as loot.
Rank threshold and consumption rate are tunable in configs/alifetactics/at_ammo.ltx.

Effects (planned):
Player-facing combat feedback, concussion first (tinnitus and blur).

Mutants (planned):
Mutant combat behavior.

Intended setup:
AlifeTactics is designed against vanilla Anomaly on the latest demonized build, and that is what it is tuned for.
It runs on AOEngine and older demonized builds with fallbacks, where a feature that needs a newer hook stays inactive or reduced.
It coexists with other combat AI mods, but vanilla plus AlifeTactics is the intended experience.

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
- G.A.M.M.A. No NPC Friendly Fire, or any community-based friendly-fire blocker: AlifeTactics already filters same-community and friendly-faction hits at the per-NPC relation gate. The community version double-filters and mutates NPC relations on hit, fighting the relation gate. Disable it.

Coexists:
- xrMPE Animations (ANOMALY-GAMMA): replaces the stalker animation files AlifeTactics's hurt and heal poses play from.
  Every animation AlifeTactics uses exists in its files (verified), so the poses keep working and take on xrMPE's look.
- Other combat AI (G.A.M.M.A. AI Rework, ReDone Combat AI, RE:VISION, AI More Cover):
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
- Global aim tuners (the game's own Hardcore AI aim option, REDONE's aim system, any ai_aim console tuning):
  Reaction writes per-stalker engine fields; a stalker it touched uses its own values and everything else keeps following the global tuning.
  AlifeTactics never writes a value worse than that global baseline, so raising game difficulty is always respected.
- Detection mods (GAMMA Stealth Overhaul and the GAMMA thresholds): Reaction's vision speed is a multiplier
  on your setup's own detection result, shipped at 1.00 = unchanged. Raising it in MCM stacks ON TOP of such overhauls.
- GAMMA's "No logs" and "Log spam remover" (disabled by default): their outdated _g.script removes the engine's
  dispersion forwarder and silently disables the Accuracy system. Keep them disabled.
- g_ai_unlimited_ammo set to 0 (a console cvar on newer engine builds makes NPCs consume real inventory rounds):
  the Ammo system detects the setting and goes inert, so carried AP is not drained twice. At 1 (the default) Ammo runs normally.

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

AlifeTactics works deep in the engine, and combat is the hardest part of Anomaly to diagnose.
Before reporting a combat or AI issue, confirm it is actually this mod: reproduce it, disable AlifeTactics, reproduce again.
If it persists, it is not this mod. Much of what gets blamed on combat mods is the engine, the modpack, or another mod, and only the log tells them apart.
The cleanest test setup is vanilla Anomaly plus xlibs plus AlifeTactics.

Include:
- Exact steps to reproduce, from a new game or a named save, with expected and actual result.
- Confirmation that the issue disappears with AlifeTactics disabled.
- xray.log and the mod debug log (MCM log level DEBUG), plus engine build, modlist, load order.
- Describe the behavior. With hundreds of mods and overrides, only the log shows whether this mod was involved and what caused it.
