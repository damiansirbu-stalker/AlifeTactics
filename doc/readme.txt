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
Cover posture: experienced riflemen and snipers crouch to steady the shot when the line to the enemy is clear, and stand to fire over low cover that would block a crouched shot,
  replacing vanilla's blind posture picks (crouching behind random bumps and firing into them, or standing tall where a crouch would steady the aim).
They crouch only past close range, staying mobile in a knife fight instead of dropping to a knee at your feet, and only while holding a position rather than on the move to cover.
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
Target Lead aims a stalker ahead of a moving target by the round's real flight time, computed from range and the weapon's bullet speed as it fires.
Higher ranks lead true and hit movers, lower ranks over-lead and overshoot.
Fire Discipline gives higher ranks crisper short bursts at a tighter cadence while low ranks stay vanilla. Defaults keep a rank's rounds per minute at or above vanilla.
Tracking, Target Lead, and Fire Discipline are per-rank MCM slider curves, each with its own on/off and a short note on the toggle.
It uses per-NPC aim hooks created in xray-monolith for this mod, so on older builds the page has no effect.
The two rank vision curves moved to the Perception tab below.

Range (planned):
The game stops NPC fights at a hard range cap regardless of weapon, so a sniper never fires at the distances his rifle exists for.
A stalker sniped from beyond the cap can duck, which is the Danger hit response, but never answer. This page will let long-range NPCs answer and initiate at their weapon's real reach.

Resistance (planned):
NPC toughness comes from what they carry: armor and equipment, damage mitigators, and artefacts.

Perception

Perception is how a stalker senses and reacts: what he hears, how his rank shapes his sight, and how he reacts to being hit.
The Sound and Danger reactions are a runtime patch laid onto whichever danger script your modpack ships rather than a replacement, so they work with other combat AI.
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

Vision:
Stalker rank shapes the eyes as well as the trigger, applied per stalker.
Vision Speed sets how fast each rank turns a glimpse into a confirmed threat, from a touch below your setup's detection speed at the low ranks to about 15 percent faster at the top,
  with the middle ranks near your baseline.
It scales the rate only: sight range, vision cone, light and darkness response, cover and occlusion, and hearing all stay exactly as your setup has them.
Vision Range sets how far out each rank begins to notice a threat, the same tight band around your baseline, on engine builds carrying the per-stalker view-distance hook created for this mod,
  inactive on older exes.
Both are per-rank MCM slider curves under one Vision toggle, on their own Vision page.

Danger:
A stalker hit from far off reacts even when vanilla would have him stand still, so sniped at 200m he turns and seeks cover.
The reaction is the duck, and returning fire at that range is the planned Range page above.
A set of always-on fixes clears long-standing crashes and misreads across the danger check and the corpse investigation, and these are not optional, shown as locked toggles.
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
Items a stalker carries become live combat edges, read from each item's own game data, so artefacts from any mod work.
An artefact grants one edge chosen by its anomaly class, scaled by its own tier: gravity, chemical, and armour-plate artefacts cut the damage he takes,
  thermal artefacts tighten his fire, and electric and quest artefacts raise the damage he deals.
Any single artefact tops out at 10 percent and never stacks, the strongest source wins, so gear tilts a fight and never decides one.
Any artefact carrier warps the air around his body, so a distorting stalker is a real, huntable artefact drop.
Binoculars extend his sight range by day and night-vision by night, on engine builds carrying the per-stalker view-distance hook created for this mod, inactive on older exes.
Effect strengths and the artefact class tables are tunable in configs/alifetactics/at_gear.ltx.

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

Incompatible with AlifeTactics (each fights one of its systems - disable it or patch it):
- NPC Limping and Healing (Vodoxleb): the same limp and heal animations from a second system; the two stack and break the heal cue.
- NPC Weapon Jamming, and any mod that jams NPC guns: Anomaly ships no NPC jam animation, so a jammed stalker loops his reload forever;
  AlifeTactics removes NPC jamming for exactly that reason and such mods put it back. Player-side jamming (Weapon Parts Overhaul, the GAMMA jam mods) is a separate system and stays untouched.
- G.A.M.M.A. No NPC Friendly Fire, and any community-based friendly-fire blocker: AlifeTactics already filters friendly hits at the per-NPC relation gate;
  the community version double-filters and mutates NPC relations on hit.
- G.A.M.M.A. NPCs Faster Reactions: raises stalker sight range from 160 to 220m (Monolith 275), which also speeds detection at every distance inside it
  and buries the per-rank Vision Speed curve (a rookie then detects like a veteran). Patch the two range keys back with DLTX; its raised occlusion threshold is the part worth keeping.
- Worse NPC Vision and Accuracy: multiplies the eight NPC dispersion values 6 to 18 times, so every stalker shoots like a blind rookie
  and the per-rank Accuracy curve disappears underneath. Keep its vision half, drop the rest.
- Animated NPC Healing and similar NPC-healer replacements: their healer runs instead, so the heal rate and per-rank charge do nothing where they overlap. Disable one side.
  A DLTX mod that rewrites the vanilla medkit list can also empty AlifeTactics's list, by load order.
- G.A.M.M.A. "No logs" and "Log spam remover" (disabled by default): their outdated _g.script removes the engine's dispersion forwarder and silently disables the Accuracy system. Keep them disabled.

Independent of AlifeTactics (these behave the same with it or without it):
- RE:DONE Combat AI: writes the four global ai_aim console variables as degrees where the engine reads radians (25 to 79 times the defaults),
  never restores them, and the game saves them into user.ltx, so the damage outlives uninstalling. Restore by hand:
  ai_aim_max_angle 0.7854, ai_aim_min_angle 0.19635, ai_aim_min_speed 0.24, ai_aim_predict_time 0.4. Its combat half coexists with AlifeTactics.
- Improved Visual Awareness, Stealth 2.31, RE:VISION, RE:DONE Combat AI - any TWO mods shipping the visibility script: only one copy loads,
  and the loser's whole stealth model silently stops existing while its MCM sliders remain. Pick one.
- enemy_shoot_back, NPC Improvements, Kebab's NPC Overhaul - any mod shipping a full m_stalker.ltx: the same single-winner problem
  for sight range, vision cone and dispersion; the losing copy contributes nothing.

Coexists:
- xrMPE Animations and other NPC animation packs: every pose AlifeTactics plays exists in their files (verified), so its cues take on their look.
  Their bigger, longer hit and hurt animations can make the gliding and staggering artifacts more visible,
  but that class is base-game hit handling and shows without any mods - see "Not AlifeTactics" below.
- G.A.M.M.A. AI Rework, RE:DONE Combat AI, RE:VISION, AI More Cover: the takeover blocks the combat planner only while a maneuver runs, then hands back to their combat AI;
  the danger rework patches their danger script at load and reads their ranges instead of substituting its own.
  Where one of them also ships a visibility script or rewrites the aim globals, that part is covered above - the combat half coexists.
- Wuut AI Extension, NPC_Fleeing, Mora's AI More Covered: add planner actions; suppressed only for the seconds a maneuver runs, then they resume. Disable Combat in MCM to leave them fully in charge.
- No More Companion Friendly Fire: a different axis (actor-to-companion damage); AlifeTactics never touches your shots.
- Tougher Important NPCs and Companions: damage reduction on the same callback, composes.
- Dynamic AI Aim Settings: perception tweaks that compose with the dispersion fix.
- The game's Hardcore AI aim option and any ai_aim console tuning: Reaction writes per-stalker fields and never below the global baseline, so raising difficulty is always respected.
- GAMMA Stealth Overhaul and detection threshold mods: Vision Speed multiplies your setup's own detection result as a rank curve (rookie unchanged, legend up to twice as fast).
  Set every rank to 1.00, or clear the toggle, to hand acquisition back entirely.
- g_ai_unlimited_ammo set to 0 (newer engine builds): the Ammo system detects it and goes inert so carried AP is not drained twice; at the default 1 it runs normally.

Not AlifeTactics (base-game behavior):
- Stalkers gliding or staggering oddly when shot: the engine starts or continues movement while a hit reaction animation still plays,
  and the hit impulse shoves the body with no animation at all. Present in unmodded Anomaly; animation packs (xrMPE among them) can make it more visible because their reactions are longer and larger.
  AlifeTactics never starts a maneuver or re-applies state on an animating body, so it adds no instances of its own.
- Stalkers who ignore darkness: your modpack's visibility script applies its darkness curve only between 21:00 and 04:00, so dusk and dawn compute as full daylight.
  In GAMMA the lever is michiko_patch on the Stealth MCM page, off by default. AlifeTactics deliberately runs no code on the per-sight-test path, so it cannot reach this.
- Stalkers who see across open ground: sight range from your setup's creature config, not acquisition speed - see NPCs Faster Reactions above.
- Stalkers who react to danger from across the map: vanilla fills its danger reaction-distance table with duration values, so a corpse, a ricochet or a shot counts as danger out past 300m.
  AlifeTactics reads whatever your setup ships; GAMMA's Stealth Overhaul rewrites the band to 75-125m, the scale it was meant to be.

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
Open a report at https://github.com/damiansirbu-stalker/AlifeTactics/issues/new/choose, or ask on the GAMMA, EFP, Anomaly, and Zona Discord servers. Read this readme and the MCM options first.

Combat is the hardest thing in Anomaly to diagnose, so first confirm it is this mod: reproduce, disable AlifeTactics, reproduce again. If it persists it is not this mod. The cleanest test is vanilla Anomaly plus xlibs plus AlifeTactics.

Include: exact repro steps (new game or named save, expected vs actual), confirmation the issue disappears with AlifeTactics off, engine build, modlist, load order, xray.log, and the mod debug log. Only the log shows whether this mod was involved.

The debug log is required: set the MCM log level to DEBUG, reproduce, then back to WARN. DEBUG is not free. It writes a timed line for every evaluation and hitches single-threaded exes, and the millisecond figures include the tracing itself, so treat them as relative.
