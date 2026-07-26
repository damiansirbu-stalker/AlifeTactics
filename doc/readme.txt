AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: next (xlibs 1.8.3, demonized 20250908)
GitHub: https://github.com/damiansirbu-stalker/AlifeTactics
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeTactics/issues

Alife Collection:
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics

Reset MCM settings to defaults after updating.

https://www.youtube.com/watch?v=eKpzbmFOFC8

AlifeTactics rebuilds how creatures behave and fight in STALKER Anomaly.
Sections marked (planned) are not built yet.

Every system reads the real state of the game, then decides in the combatant's favor rather than by script or die roll.
It weighs combat events, NPC and player stats, the world, squad, faction, weapons, range, angle, and cover.
Maneuvers take a stalker over for an action the vanilla engine has no mechanism for.
Commitment keeps a stalker on a decision that still makes sense instead of the engine's constant re-planning.
Threat, accuracy, and the rest read state and decide the same way.

Effects:
- Creatures make emergent decisions from environment, own state, enemy state, squad state, weapons, and faction doctrine.
- Nothing is player-centric, and no side gets an advantage or a handicap.
- Creatures use the items they acquired themselves: medkits, ammo, props.
- Rank scales accuracy, weapon spread, and aim speed, which vanilla clamps flat.
- AP ammo is fired from its owner's inventory and consumed, where vanilla gives NPCs infinite cheap rounds and modpacks give infinite AP by rank and map.
- Bugs disguised as features are gone, like fake weapon jamming and random tactical reloading.


Everything is canon, engine-pure, safe, and fast:
- Every behavior is built from pure X-Ray and Anomaly primitives: the GOAP action planner, the xr_logic scheme system, state_mgr,
  smart terrains and the gulag job system, condlists, and DLTX/DXML for data.
- Nothing is faked and nothing is simulated beside the engine.
- Where the engine had no seam, the seam was added upstream first: per-NPC hooks created in xray-monolith specifically for this mod, merged into the official modded exes.
- It never replaces a vanilla script file. Takeovers block the planner only for their seconds and then hand back, and patches lay onto whichever script your setup ships.
- Every value ships as a formula over the engine's own constants, never an invented number.
- Zero per-frame Lua. Everything runs on engine callbacks and scheduled passes, and the whole mod costs 5 timer compares per frame, whatever the NPC count.
- AlifeTactics never wraps the engine's visibility function, so no sight test gains a cost or a behavior it did not already have. Your stealth setup computes being seen exactly as before.
- Perception is written as a per-stalker engine field once at spawn, so it multiplies onto whatever detection system is installed instead of replacing it.
- It fixes dozens of vanilla Anomaly bugs: dead branches, wrong calculations, and plain crashes in original code.


Every system below has its own MCM page in this order, where it is toggled and tuned.

Combat

Maneuvers:
A maneuver takes one stalker over completely to perform what the vanilla engine cannot, either a mechanic it lacks or a decision it never makes.
Planned: whole squads coordinating their movement and fire, each faction fighting in its own flavor and favoring the maneuvers that suit it.

These maneuvers cover the moments vanilla fumbles:
  Counterflank snaps a stalker around when a hostile player stands at contact range while he shoots someone far away.
  Reload Cover sends a stalker caught reloading in your line of fire to the nearest cover instead of standing in the open, and his weapon finishes reloading on the way.
  Flee routs a cautious faction to a distant friendly base, because a coward runs before he fights.
  Retreat pulls a steady one to cover under pressure, and a coward whose escape is cut off does the same.
  Kite backs a stalker out of an enemy that closed too near, still firing.
  Pickoff plants a stalker who has his enemy outranged and picks him off with deliberate single shots, breaking off the moment the threat returns.

Maneuvers run only in fights the player is part of, and NPC-only fights stay vanilla.
A maneuver fires when its problem is real and ends when it is solved. Keep creating the problem, such as pressing a shotgunner's minimum range, and the answer keeps coming.
The decisions come out looking human. Nobody turns his back on a shooter, two men never take the same cover, nobody plants under your crosshair.
Maneuver fire bursts by weapon and by skill: burst length and pauses vary shot to shot inside the game's own per-weapon ranges and tighten with rank,
  in place of the uniform burst the game's script machinery applies to every weapon it drives.
The takeover overrides no combat scripts, so it fights side by side with vanilla and works with other combat AI instead of replacing it. Companions are excluded by default.

Commitment:
Vanilla stalkers re-plan the fight every moment, so any small change makes a stalker drop what he is doing and choose again.
Better cover, a flicker of lost sight, or a teammate crossing the line sets him off, which is the twitchy strafing and cover-hopping you see in a firefight.
Many mods answer this by switching the stalker to a camper scheme that pins him in place, muting most of Anomaly's combat variety.
AlifeTactics keeps the engine's full combat AI and instead stops a stalker throwing away a decision that still makes sense.
While what he is doing still works he stays with it, and he switches the instant it stops: he loses sight, the shot is blocked, or the enemy is gone.
It also pins his cover: a stalker firing with a clear shot keeps his spot instead of sliding to a marginally better one, stopping the mid-fight strafe at its source.
He sees a decision through, keeping up his fire, pressing a flank, or finishing a reload, instead of second-guessing himself every frame.
It uses action-switch and cover-repick hooks created in xray-monolith for this mod, so on older builds the system stays inactive.

Conduct:
Conduct makes better choices at moments the engine already decides, with no takeover, for stalkers the vanilla engine drives.
Cover posture: experienced riflemen and snipers crouch when real chest-high cover stands between them and their enemy, and stand when it does not,
  replacing vanilla's blind posture picks (crouching behind random bumps and firing into them).
They crouch only at a range that suits the weapon, a rifleman past about 24m and a sniper past about 45m, so they never drop to a knee at your feet in a close fight.
It reads the game's own cover map toward the enemy, and short-weapon carriers and green ranks keep vanilla behavior.

Behaviors (planned):
New actions the vanilla planner lacks, starting with melee and peek.

Effectiveness

Accuracy:
Anomaly's rank curve clamps every NPC to the same dispersion, so accuracy never scaled per rank even though the engine code exists.
AlifeTactics restores a real per-rank curve through the engine's own dispersion callback, tunable per tier.
A second curve covers fire on the move: each rank keeps a share of the movement spread penalty, so rookies spray while repositioning and top ranks cut about a third of it.
The engine has several dispersion variables, including barrel and weapon, and this one is the NPC skill-based dispersion.

Disclosure:
When a faction enemy wounds any squad member, the whole squad learns the shooter at once, even patrol members out of earshot and even against a silenced weapon.
Vanilla only shares a shooter the squadmates actually heard, so distant or suppressed attackers stay unknown.
It works through the engine's own memory rather than by faking relations: the shooter is enabled in each member's memory and registered into the squad's combat,
  and the engine's normal propagation carries it from there.
On demonized 20260722 or newer the shooter also enters their seen memory, which target selection ranks first, so the squad turns on the shooter instead of staying locked on a distant visible enemy.
A stalker with no squad discloses the shooter to himself.
New members inherit the fight when they spawn, and a shooter who went offline and returns is flagged again.
A survivor gate (default on) skips disclosure when the hit kills the victim outright, so a clean kill stays quiet.
The squad can still find you by sound, by sight, or by the body.

Crossfire:
Same-faction fighters no longer cut each other down in a crossfire.
A hit between two NPCs of the same faction deals reduced damage, set by a slider (no damage by default, up to full vanilla).
It keys on their actual relation, so genuinely hostile factions still trade fire while allies stay safe. Your own shots are never affected.

Reaction:
Stalker rank now shapes gun handling, applied per stalker.
Tracking Speed sets how tightly a barrel follows a moving target, from vanilla at the bottom rank to the game's own hardcore AI aim at the top.
Vision Speed sets how fast each rank notices a target at range, up to twice your setup's detection speed at the top.
It scales the rate only: sight range, vision cone, light and darkness response, cover and occlusion, and hearing all stay exactly as your setup has them.
Target Lead aims a stalker ahead of a moving target by the round's real flight time, computed from range and the weapon's bullet speed as it fires.
Higher ranks lead true and hit movers, lower ranks over-lead and overshoot.
Fire Discipline gives higher ranks crisper short bursts at a tighter cadence while low ranks stay vanilla. Defaults keep a rank's rounds per minute at or above vanilla.
Tracking, Vision, Target Lead, and Fire Discipline are per-rank MCM slider curves, each with its own on/off and a short note on the toggle.
It uses per-NPC aim and vision hooks created in xray-monolith for this mod, so on older builds the page has no effect.

Range (planned):
The game stops NPC fights at a hard range cap regardless of weapon, so a sniper never fires at the distances his rifle exists for.
A stalker sniped from beyond the cap can duck, which is the Danger hit response, but never answer. This page will let long-range NPCs answer and initiate at their weapon's real reach.

Resistance (planned):
NPC toughness comes from what they carry: armor and equipment, damage mitigators, and artefacts.

Danger

Danger is how a stalker reacts to a threat he perceives, by sound, by hit, or later by sight.
It is a runtime patch laid onto whichever danger script your modpack ships rather than a replacement, so it works with other combat AI.
Detection distances stay owned by your setup's danger config, and AlifeTactics adds the reactions, never the tuning.

Sound:
Vanilla NPCs ignore gunfire they hear but cannot see.
A hostile stalker now reacts to an enemy's shots without line of sight: he turns to face the gun and takes a threat stance, and moves to cover once he can see the shooter.
He is responding to the sound rather than to sight of you.
Gunfire from neutral or friendly stalkers, including your own, does not alarm them, because reactions follow the engine's relation rule, same as vanilla.
Hostile stalkers also hear you move.
Each footstep carries by your stance, the surface, and the weather: crouched movement is silent, sprinting on metal carries far, rain muffles everything, and a jump landing is loudest of all.
Walking carries 5m as the base, sprinting multiplies it by 1.6, a jump landing carries 10m, and crouched movement is silent, always.
Surfaces scale it: metal x1.25, wood x1.15, water x1.35, grass x0.7, dirt and sand x0.8, and rain cuts carry by up to 40 percent.
They react to handling noise too, at shorter reach: a racked reload carries 8m, an empty click 6m, an item used 5m.
Handling noise goes through each stalker's own ears, so a setup that deafens NPC hearing quiets these sounds with it.
Reaction follows the evidence: a heard walk or an item used turns him weapon-ready toward the sound,
  while a sprint, a landing, a racked reload, or an empty click sends him walking over to check the spot.
A sound is never treated as a confirmed enemy: he investigates at a walk, never charges, and does not know where you are.
Nothing changes in combat, and a carry-distance slider scales it, so stealth stays a game of distance and stance instead of NPCs being deaf.
Compatible with stealth mods: stealth in Anomaly is about being seen, through light, cover, and stance, and the sound system never touches vision or detection.
Hearing only adds the short-range sense vanilla lacks, and crouched movement is silent, so the crouched approach your stealth setup allows is never given away by sound.
Neutral and friendly stalkers get one reaction of their own: fire close to them and they go weapon-ready facing your shots instead of ignoring gunfire next to their heads.
They only go alert, never hostile, and settle down when the shooting stops. Every reaction has its own toggle.

Hit:
A stalker hit from far off reacts even when vanilla would have him stand still, so sniped at 200m he turns and seeks cover.
The reaction is the duck, and returning fire at that range is the planned Range page above.

Fixes:
A set of always-on fixes clears long-standing crashes and misreads across the danger check and the corpse investigation, and these are not optional.
One tuning toggle lets danger you cause read your setup's separate player-specific ranges where the config provides them.
One of those fixes shortens reactions instead of lengthening them.
Vanilla's danger table gives three of its entries the same key, so two threat types are read under the wrong range: a ricochet at the 150m sight range instead of its own 4m,
  and an attacked ally at 150m instead of 50m.
That is why a vanilla stalker can wheel around at a spark off a wall halfway across the map.
AlifeTactics separates the keys, so both reactions happen at the distance they were written for. A modpack that already corrected this sees no change.

Mechanics

Healing:
Wounded NPCs heal with the items they carry.
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
That AP comes from trade and looting through the Alife Collection, so what an NPC scavenged shapes how dangerous he is. NPCs drop no AP as loot.
Rank threshold and consumption rate are tunable in configs/alifetactics/at_ammo.ltx.

Gear:
Items a stalker carries become live benefits, read from each item's own game data, so artefacts from any mod work.
Armor artefacts and inserts reduce the damage he takes, fire artefacts raise the damage he deals, electric artefacts sharpen his aim and eyes,
  psy artefacts keep him from panicking, healing artefacts speed his recovery, acid artefacts feed him armor-piercing rounds, stamina artefacts quicken his trigger,
  gravity artefacts steady his fire, and binoculars or night-vision lamps sharpen his spotting.
Any single item tops out around 7 percent and a whole loadout around 15, so gear tilts a fight, never decides one.
Any artefact carrier warps the air around his body, so a distorting stalker is a real, huntable artefact drop.
Effect strengths are tunable in configs/alifetactics/at_gear.ltx.

Effects (planned):
Player-facing combat feedback, starting with concussion: tinnitus and blur.

Mutants (planned):
Mutants get the same combat treatment as stalkers.

Intended setup:
AlifeTactics is built and tuned against vanilla Anomaly on the latest demonized build.
It runs on AOEngine and older demonized builds with fallbacks, where a feature that needs a newer hook stays inactive or reduced.
It coexists with other combat AI mods, but vanilla plus AlifeTactics is the intended experience.

Requirements:
Anomaly 1.5.3
Modded exes (themrdemonized 2025.9.10 or newer, or AOEngine v0.55 or newer)
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
Tested with vanilla Anomaly 1.5.3 and GAMMA, and installing or uninstalling mid-save works.
The takeover leaves combat scripts vanilla, and the danger rework patches at runtime, so AlifeTactics layers cleanly onto other combat and AI mods.
Friendly fire and same-community hits are filtered at the faction-relation gate.
Story NPCs, companions, traders, and squadmates are never armed against their own faction.

Conflicts (choose one):
- NPC Limping and Healing (Vodoxleb): the same limp and heal animations from a different system. The two stack and break the heal cue.
- NPC Weapon Jamming, or any mod that jams NPC guns: Anomaly ships no jam animation for NPCs, so a jammed stalker loops the reload forever instead of clearing the stoppage.
  AlifeTactics removes NPC jamming for that reason and the mod puts it back. Player-side jamming (Weapon Parts Overhaul and the GAMMA jam mods) is a separate system and is untouched.
- G.A.M.M.A. NPCs Faster Reactions: its creature config is a copy of GAMMA's own Stealth Overhaul file with the stalker sight range raised from 160 to 220, Monolith to 275.
  Everything else people credit to it, the 175-degree vision cone included, belongs to the file it copied and stays in place when it is disabled.
  An unaware stalker picks you up at 222m where the copied file stopped at 162m, an alerted one at 268m instead of 195m.
  The engine divides by that same range while it accumulates detection, so they also notice faster at every distance inside it: 1.3 to 1.4 times faster at 100m, about twice as fast at 150m.
  That sits on the same accumulator as AlifeTactics's Vision Speed curve, which spans 1.0 at rookie to 2.0 at legend, so a rookie detects like a veteran and rank stops reading as skill.
  Its raised occlusion threshold is the part worth keeping: one bush breaks line of sight in combat where two were needed before.
  Patch the two range keys back with DLTX rather than deleting the file.
- Worse NPC Vision and Accuracy: the vision half is sound, but the same file multiplies the eight NPC dispersion values by 6 to 18 times, taking aimed standing fire from 0.42 to 7.5.
  Every stalker then shoots like a blind rookie whatever his rank, and any rank-based accuracy curve laid over that base disappears underneath it. Keep the vision half, drop the rest.
- RE:DONE Combat AI: its aim system overwrites the four global ai_aim console variables with values 25 to 79 times the engine defaults,
  because the engine reads them as radians and the mod writes them as though they were degrees.
  It never reads your existing values, never restores them, and the game saves them into user.ltx, so the change outlives uninstalling the mod.
  Its own MCM toggle does not gate the code that writes them.
  If you have run it, restore the stock values by hand: ai_aim_max_angle 0.7854, ai_aim_min_angle 0.19635, ai_aim_min_speed 0.24, ai_aim_predict_time 0.4.
- Any second mod that ships its own copy of the visibility script (Improved Visual Awareness, Stealth 2.31, RE:DONE Combat AI, RE:VISION):
  that one file turns light, cover, movement and distance into being seen, and only one copy can load.
  Install two and one mod's whole stealth model stops existing, with no error, so you keep its MCM sliders and lose everything behind them. Pick one.
- Any mod that ships a full m_stalker.ltx (enemy_shoot_back, NPC Improvements, Kebab's NPC Overhaul): the same single-winner problem for sight range, vision cone and dispersion.
  The copy that loses contributes nothing, so the NPC tuning you believe you installed may not be running.
- G.A.M.M.A. No NPC Friendly Fire, or any community-based friendly-fire blocker: AlifeTactics already filters same-community and friendly-faction hits at the per-NPC relation gate.
  The community version double-filters and mutates NPC relations on hit, fighting the relation gate. Disable it.

Coexists:
- xrMPE Animations (ANOMALY-GAMMA): replaces the stalker animation files AlifeTactics's hurt and heal poses play from.
  Every animation AlifeTactics uses exists in its files (verified), so the poses keep working and take on xrMPE's look.
- Other combat AI (G.A.M.M.A. AI Rework, RE:DONE Combat AI, RE:VISION, AI More Cover): the takeover blocks the combat planner only while it runs a maneuver, then hands back to their combat AI.
  The danger rework patches their xr_danger at load, so their danger layer keeps running too.
  Where one of them owns the danger config, AlifeTactics reads its ranges, including its separate player-specific ones, instead of substituting numbers of its own.
  The ones that also ship a visibility script or rewrite the global aim variables are covered under Conflicts above, where their combat half coexists and the rest does not.
- Wuut AI Extension, NPC_Fleeing, Mora's AI More Covered: add planner actions. While AlifeTactics runs a maneuver its own action is suppressed for those seconds, then resumes.
  Disable Combat in MCM to leave them in charge.
- Mods that replace NPC healing with their own path (Animated NPC Healing and similar): their healer runs, so AlifeTactics's heal rate and per-rank charge do nothing where they overlap.
  Disable one side. A DLTX mod that rewrites the vanilla medkit list can also empty AlifeTactics's list, by load order.
- No More Companion Friendly Fire: a different axis (actor-companion damage), and AlifeTactics never touches your shots.
- Tougher Important NPCs and Companions: damage reduction through npc_on_before_hit, composes.
- Dynamic AI Aim Settings: perception tweaks that compose with the dispersion fix.
- Global aim tuners (the game's own Hardcore AI aim option, any ai_aim console tuning): Reaction writes per-stalker engine fields,
  so a stalker it touched uses its own values and everything else keeps following the global tuning.
  AlifeTactics never writes a value worse than that global baseline, so raising game difficulty is always respected. RE:DONE's aim system is the exception and is covered under Conflicts.
- Detection mods (GAMMA Stealth Overhaul and the GAMMA thresholds): Reaction's vision speed multiplies your setup's own detection result.
  It ships as a rank curve, unchanged at rookie and up to twice as fast at legend, so out of the box a high-rank stalker crosses your setup's own threshold sooner while a rookie behaves as before.
  Set every rank to 1.00, or clear the Vision Speed toggle, to hand acquisition back to your detection mod entirely.
- GAMMA's "No logs" and "Log spam remover" (disabled by default): their outdated _g.script removes the engine's dispersion forwarder and silently disables the Accuracy system. Keep them disabled.
- g_ai_unlimited_ammo set to 0 (a console cvar on newer engine builds makes NPCs consume real inventory rounds): the Ammo system detects the setting and goes inert, so carried AP is not drained twice.
  At 1 (the default) Ammo runs normally.

Not related to AlifeTactics:
- Stalkers who ignore darkness: the visibility script your modpack ships decides this, and the copies in common use only apply their darkness curve between 21:00 and 04:00.
  Dusk and dawn are computed as full daylight, which is why a stalker picks you out of near-black at last light and again before sunrise.
  In GAMMA the lever is the michiko_patch option on the Stealth MCM page, which widens the window and steepens the curve, and it ships off.
  AlifeTactics cannot reach this, because changing it means running code on every sight test in the game, which the mod deliberately does not do.
- Stalkers who see across open ground: that is sight range, set in your setup's creature config, not acquisition speed. See NPCs Faster Reactions under Conflicts.

FAQ:
Do I need modded exes?
  Yes. AlifeTactics needs themrdemonized modded exes (2025.9.10 or newer) or AOEngine (v0.55 or newer). Vanilla Anomaly does not expose the APIs it relies on.

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
- Describe the behavior, because with hundreds of mods and overrides only the log shows whether this mod was involved and what caused it.
