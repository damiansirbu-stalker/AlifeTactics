Version: 1.2.1-snapshot (xlibs 1.8.5, demonized 20250908)
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Health: https://damiansirbu-stalker.github.io/AlifeTactics/health/
JitProfiler: https://damiansirbu-stalker.github.io/AlifeTactics/jitprofiler/
Bugs: https://github.com/damiansirbu-stalker/AlifeTactics/issues
Russian / На русском: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/readme_ru.txt

My work:
GitHub: https://github.com/orgs/damiansirbu-stalker/repositories
ModDB: https://www.moddb.com/members/damian-sirbu/addons
Nexus: https://www.nexusmods.com/profile/damiansirbu/mods

My contributions:
X-Ray Monolith: https://github.com/themrdemonized/xray-monolith

Reset MCM settings to defaults after updating.

Everyone wants to live.

AlifeTactics rebuilds how creatures behave and fight in STALKER Anomaly.
Sections marked (planned) are not built yet.

Every system reads the real state of the game, then decides in the combatant's favor rather than by script or die roll.
It weighs combat events, NPC and player stats, the world, squad, faction, weapons, range, angle, and cover.
Maneuvers take a stalker over for an action the vanilla engine has no mechanism for.
Commitment holds a stalker to a decision while it still makes sense.
Threat, accuracy, and the rest read state and decide the same way.

Effects:
- Creatures make emergent decisions from environment, own state, enemy state, squad state, weapons, and faction doctrine.
- Every faction fights its own way: who routs, who presses your weak moment, who plants for deliberate shots, who crouches on the line, who never backs down.
- Nothing is player-centric, and no side gets an advantage or a handicap.
- Creatures use the items they acquired themselves: medkits, ammo, props.
- Rank scales accuracy, weapon spread, aim speed, and how tightly a stalker tracks you, which vanilla clamps flat.
- AP ammo is fired from its owner's inventory and consumed, where vanilla gives NPCs infinite cheap rounds and modpacks give infinite AP by rank and map.
- Bugs disguised as features are gone, like fake weapon jamming and random tactical reloading.


Everything is canon, engine-native, safe, and fast:
- Every behavior is built from pure X-Ray and Anomaly primitives:
  the GOAP action planner (the planner family F.E.A.R. made famous), the xr_logic scheme system, state_mgr, smart terrains and the gulag job system, condlists, and DLTX/DXML for data.
- Nothing is faked and nothing is simulated beside the engine.
- Where the engine had no seam, the seam was added upstream first: per-NPC hooks created in xray-monolith specifically for this mod, merged into the official modded exes.
- It never replaces a vanilla script file. Takeovers block the planner only for their seconds and then hand back, and patches lay onto whichever script your setup ships.
- Every value ships as a formula over the engine's own constants, never an invented number.
- No per-frame Lua. Everything runs on engine callbacks and scheduled passes, and the whole mod costs 5 timer compares per frame, whatever the NPC count.
- AlifeTactics never wraps the engine's visibility function, so no sight test gains a cost or a behavior it did not already have. Your stealth setup computes being seen exactly as before.
- Perception is written as a per-stalker engine field once at spawn, so it multiplies onto whatever detection system is installed instead of replacing it.
- It fixes dozens of vanilla Anomaly bugs: dead branches, wrong calculations, and plain crashes in original code.


Every system below has its own MCM page in this order, where it is toggled and tuned.

Combat

Faction flavor:
Every faction fights its own way. Whether a stalker routs, pulls back, presses your empty magazine, plants for deliberate shots, backs out of close range,
or takes a tactical crouch is a per-faction chance, rolled once per stalker per fight, so each man stays consistent while the fight lasts.
Ecologists run from a losing fight and rarely press yours. Bandits and renegades press hard and rarely hold a disciplined standoff.
The militarized factions crouch on the firing line, withdraw in order, and do not rout. Monolith never backs down. Loners sit in the middle, mercenaries beside them with the military edge.
Zombied carry none of it.
Every number is one line in the faction config, per faction and per behavior, yours to tune.

Maneuvers:
A maneuver takes one stalker over completely to perform what the vanilla engine cannot, either a mechanic it lacks or a decision it never makes.
Planned: whole squads coordinating their movement and fire, each faction favoring the maneuvers that suit it.

These maneuvers cover the moments vanilla fumbles:
  Counterflank snaps a stalker around when a hostile player stands at contact range while he shoots someone far away.
  Reload Cover sends a stalker caught reloading in a watching enemy's line of fire, yours included, to the cover that hides him from that watcher, and his weapon finishes reloading on the way.
  Flee routs a flee-prone faction to a distant friendly base, because a coward runs before he fights.
  Retreat pulls a steadier one to cover under pressure, and a coward whose escape is cut off does the same.
  Kite backs a stalker out of an enemy that closed too near, still firing.
  Pickoff plants a stalker who has his enemy outranged and picks him off with deliberate single shots, breaking off the moment the threat returns.

Maneuvers run in every stalker fight within 150 meters of you, tunable, whether or not you are the target.
Fights against mutants and fights beyond that range stay vanilla.
One shared allowance bounds everything AlifeTactics starts, against you and against NPCs separately, so a mass battle never drowns either side.
A maneuver fires when its problem is real and ends when it is solved. Keep creating the problem, such as pressing a shotgunner's minimum range, and the answer keeps coming.
The decisions come out looking human. Nobody turns his back on a shooter, two men never take the same cover, nobody plants in his enemy's sight.
Maneuver fire bursts by weapon and by skill: burst length and pauses vary shot to shot inside the game's own per-weapon ranges and tighten with rank.
That replaces the uniform burst the game's script machinery applies to every weapon it drives.
Stalkers keep their footing under fire. A hit mid reaction no longer slides a standing stalker across the ground. He stays planted while the animation plays, and moves the moment it ends.
The footing hold ships ahead of its engine half and needs a modded exes build carrying engine PR 645. Older builds keep vanilla movement.
The takeover overrides no combat scripts, so it fights side by side with vanilla and works with other combat AI. Companions are excluded by default.

Commitment:
Vanilla stalkers re-plan the fight every moment, so any small change makes a stalker drop what he is doing and choose again.
Better cover, a flicker of lost sight, or a teammate crossing the line sets him off, which is the twitchy strafing and cover-hopping you see in a firefight.
Many mods answer this by switching the stalker to a camper scheme that holds him in place, muting most of Anomaly's combat variety.
AlifeTactics keeps the engine's full combat AI and instead stops a stalker throwing away a decision that still makes sense.
While what he is doing still works he stays with it. He switches the moment it stops working, on a lost sight line, a blocked shot, or a vanished enemy.
It also holds his cover. A stalker firing with a clear shot keeps his spot. He no longer slides to a marginally better one, so the mid-fight strafe stops at its source.
When his enemy is caught reloading, out of ammo, staggered, sprinting weapon-down, or with no weapon up, he liquidates.
He keeps firing through the window and repositions once the enemy can answer again.
He sees a decision through without second-guessing himself every frame.

Conduct:
Conduct makes better choices at moments the engine already decides, with no takeover, for stalkers the vanilla engine drives.
Cover posture: experienced riflemen and snipers crouch to steady the shot when the line to the enemy is clear, and stand to fire over low cover that would block a crouched shot.
It replaces vanilla's blind posture picks, such as a crouch behind a random bump, or standing tall where a crouch would steady the aim.
They crouch only past close range and only while holding a position, so they stay mobile in a knife fight and while moving to cover.
It reads the game's own cover map toward the enemy, and short-weapon carriers and green ranks keep vanilla behavior.
Weapon spacing: a stalker's cover choices respect what his weapon is good at, submachine gunners accept closer cover so their fire stays effective, and skilled snipers hold extra distance.

Behaviors:
Stalkers act on weak moments in the fight, in both directions and against any enemy, you or another combatant.
The Push: a stalker whose enemy cannot answer presses him.
Caught reloading, out of ammo, or badly hurt, the target's attackers thicken their fire at close range.
With a clear upper hand (the target hurt or turned away) they move to closer cover.
The Pull is the mirror. A stalker caught reloading or badly hurt while his enemy is strong falls back, his own cover choices landing farther until he recovers.
Everything reverts the moment the target can answer, each attacker presses briefly with a cooldown before pressing again, and each cause has its own switch.

Effectiveness

Accuracy:
Anomaly's rank curve clamps every NPC to the same dispersion, so accuracy never scaled per rank even though the engine code exists.
AlifeTactics restores a real per-rank curve through the engine's own dispersion callback, tunable per tier.
A second curve covers fire on the move. Each rank keeps a share of the movement spread penalty, so rookies spray while repositioning and top ranks cut about a third of it.
The engine has several dispersion variables, including barrel and weapon, and this one is the NPC skill-based dispersion.

Crossfire:
Same-faction fighters no longer cut each other down in a crossfire.
A hit between two NPCs of the same faction deals reduced damage, set by a slider (no damage by default, up to full vanilla).
It keys on their actual relation, so genuinely hostile factions still trade fire while allies stay safe. Your own shots are never affected.

Reaction:
Stalker rank now shapes gun handling, applied per stalker.
Tracking Speed sets how fast a barrel moves while tracking, from vanilla at the bottom rank to a decent step above it at the top, well short of the game's hardcore AI aim.
Tracking Lock sets how tightly a barrel holds a strafing target. A novice's aim lags and a moving target loses it, exactly like vanilla.
A legend holds you across the firing window without tracking you perfectly.
Target Lead aims a stalker ahead of a moving target by the round's real flight time, computed from range and the weapon's bullet speed as it fires.
Higher ranks lead true and hit movers, lower ranks over-lead and overshoot.
Fire Discipline scales burst size and cadence per rank.
The shipped defaults are near-flat, so rank barely alters fire out of the box. The sliders allow the full spread.
Defaults keep a rank's rounds per minute at or above vanilla.
Tracking Speed, Tracking Lock, Target Lead, and Fire Discipline are per-rank MCM slider curves. Speed and Lock share one on/off, Lead and Discipline each carry their own.
The two rank vision curves moved to the Perception tab below. Fire Discipline is on its own Discipline tab.

Range (planned):
The game stops NPC fights at a hard range cap regardless of weapon, so a sniper never fires at the distances his rifle exists for.
A stalker sniped from beyond the cap can duck, which is the Danger hit response, but never answer. This page will let long-range NPCs answer and initiate at their weapon's real reach.

Perception

Perception is how a stalker senses and reacts. It covers what he hears, how his rank shapes his sight, and how he reacts to being hit.
The Sound and Danger reactions are a runtime patch laid onto whichever danger script your modpack ships, so they work with other combat AI.
Detection distances stay owned by your setup's danger config, and AlifeTactics adds only the reactions.

Sound:
Vanilla NPCs ignore gunfire they hear but cannot see.
A hostile stalker now reacts to an enemy's shots without line of sight. He turns to face the gun and takes a threat stance, and moves to cover once he can see the shooter.
He is responding to the sound rather than to sight of you.
Gunfire from neutral or friendly stalkers, including your own, does not alarm them, because reactions follow the engine's relation rule, same as vanilla.
Hostile stalkers also hear you move.
Each footstep carries by your stance, the surface, and the weather.
Crouched movement is silent and sprinting on metal carries far. Rain muffles everything, and a jump landing is loudest of all.
Walking carries 5m as the base, sprinting multiplies it by 1.6, a jump landing carries 10m, and crouched movement is silent, always.
Surfaces scale it, metal x1.25, wood x1.15, water x1.35, grass x0.7, dirt and sand x0.8. Rain cuts carry by up to 40 percent.
They react to handling noise too, at shorter reach. A racked reload carries 8m, an empty click 6m, an item used 5m.
Handling noise goes through each stalker's own ears, so a setup that deafens NPC hearing quiets these sounds with it.
Reaction follows the evidence. A heard walk or an item used turns him weapon-ready toward the sound.
A sprint, a landing, a racked reload, or an empty click sends him walking over to check the spot.
A sound is never treated as a confirmed enemy. He investigates at a walk and does not know where you are.
The active reaction lasts around 10 seconds, then he settles into a standing watch until the memory fades.
Stalkers whose squadmates are actually fighting skip the investigation entirely and hold a watch stance. The fight is the information.
Every sound reaction in the mod obeys the same rule, including the sounds other mods and quests feed in, and it caps at a walk-over check.
Your companions are the one exception and still run when called to help.
Standing stalkers also notice nearby creatures by sound, within 5 to 10 meters. Mutant footsteps, voices, and death cries always draw a glance toward the sound,
and another stalker's sounds draw it only when that stalker is an enemy of the hearer, so a camp never startles at its own chatter.
The reaction stops at a glance. A stalker in his own fight ignores it, and so does one on the move, so patrols and traveling squads keep their stride. Companions are excluded.
A stalker starting a walk-over check calls it out, so you hear the reaction as well as see it.
Nothing changes in combat, and a carry-distance slider scales it, so stealth stays a game of distance and stance.
Compatible with stealth mods: stealth in Anomaly is about being seen, through light, cover, and stance, and the sound system never touches vision or detection.
Hearing only adds the short-range sense vanilla lacks, and crouched movement is silent, so the crouched approach your stealth setup allows is never given away by sound.
Neutral and friendly stalkers get one reaction of their own. Fire close to them and they go weapon-ready facing your shots.
They go alert but stay friendly. They settle down when the shooting stops. Every reaction has its own toggle.

Vision:
Stalker rank shapes the eyes as well as the trigger, applied per stalker.
Vision Speed sets how fast each rank turns a glimpse into a confirmed threat, from your setup's own detection speed at the bottom rank (a novice matches it) to about 21 percent faster at the top.
No rank is slower to notice than your baseline.
It scales the rate only. Sight range, vision cone, light and darkness response, cover and occlusion, and hearing all stay exactly as your setup has them.
Vision Range sets how far out each rank begins to notice a threat, the same band, from your baseline at novice to about 15 percent farther at the top.
Both are per-rank MCM slider curves under one Vision toggle, on their own Vision page.

Danger:
A stalker hit from far off reacts even when vanilla would have him stand still, so sniped at 200m he turns and seeks cover.
The reaction is the duck, and returning fire at that range is the planned Range page above.
One tuning toggle lets danger you cause read your setup's separate player-specific ranges where the config provides them.
A Player target pull dial tunes how strongly NPCs prioritize you over other combatants once you are seen, down to treating you like anyone else.
A hit victim turns on his shooter through the engine's own target selection, always on: the real "he hit me" signal outweighs a distant visible enemy,
and he returns fire the moment he has line of sight. Nothing is revealed through walls, and a suppressed kill tells no one.
The vanilla danger-check and corpse-investigation fixes are listed under Fixes to Vanilla below.

Mechanics

Healing:
Wounded NPCs heal with the items they carry.
Vanilla's medkit heal fires unreliably and bandages do nothing, so bleeding stalkers die that should not.
AlifeTactics has them spend real medkits when injured and real bandages when bleeding, falling back to a per-rank charge only when empty.
The heal rate is tunable. Fixed limp and heal animations show it, out of combat only.

Jamming:
Eliminates the fake NPC jams and the tactical-reload loop they cause.
On modded exes an NPC rolls a per-rank jam from ammo spent, so even a full-condition rifle chokes and reloads every few rounds.
AlifeTactics turns that roll off for NPCs, so they now jam only for real and reload only once the magazine is spent, which you can confirm in the debug log.
Your own weapon still jams normally when worn.

Ammo:
NPCs fire the ammunition they actually carry.
Veteran-rank and higher stalkers use armor-piercing rounds from their own inventory, with real ballistics, and fall back to standard rounds once it runs out.
That AP comes from trade and looting through the Alife Collection, so what an NPC scavenged shapes how dangerous he is. NPCs drop no AP as loot.
Rank threshold and consumption rate are tunable in the ammo config under configs/alifetactics/.

Gear:
Items a stalker carries give him combat advantages, read from each item's own game data, so artefacts from any mod work.
An artefact grants one advantage chosen by its anomaly class, scaled by its own tier. Gravity and armour-plate artefacts cut the damage he takes.
Thermal artefacts tighten his fire, and electric and quest artefacts raise the damage he deals.
Any single artefact tops out at 10 percent, and only the strongest applies. Gear tilts a fight without deciding it.
A chemical artefact heals its carrier slowly over time.
Any artefact carrier warps the air around his body, so a distorting stalker is a real, huntable artefact drop.
Binoculars extend his sight range by day and night-vision by night.
Effect strengths and the artefact class tables are tunable in the gear config under configs/alifetactics/.

Fixes to Vanilla:
AlifeTactics corrects dozens of vanilla Anomaly and xray defects, grouped below by the system each repairs.
Each shows as a locked toggle on the Fixes tab. The always-on ones cannot be turned off. The rest are switched on their own pages.

Danger scheme:
- A name collision made three danger categories read the wrong range, so a ricochet read at 150m, not its own 4m. Each now reads its own range.
- A mutant corpse no longer crashes the danger time check.
- A torn-down NPC reference no longer crashes the evaluator.
- A bad danger time value no longer crashes the evaluator.
- The hit callback no longer corrupts danger memory with a missing shooter.
- A stalker is no longer evaluated for danger after he dies.
- A stalker who held fire because a friend crossed his line no longer freezes in a combat stance after the fight. He returns to normal when his enemy is gone.
- A stalker held fire for any visible non-enemy standing close in any direction, or standing on his aim line beyond the target, and the hold re-armed endlessly in a crowd.
  He now holds and steps aside only for a friend actually between him and his target.
  The false holds also silenced campers, wounded-finishing, mounted guns and anti-helicopter fire. Those behaviors now run again.
- A danger transition no longer leaves a stale lower-body animation playing.
- Leaving danger clears only its own cover reservation, not every stalker's.
- A stalker attacked again later reacts to the new attacker, not his first attacker's old position, and the corpse search plays for every corpse, not only his first.
- The grenade dodge distance read on the wrong scale. A stalker now dodges within the intended radius and faces distant grenades.
- The danger check parses its config once and caches the result.
- A stalker sniped from far off reacts and seeks cover, where vanilla left him standing.

Corpse investigation:
- A despawning corpse no longer crashes the investigator.
- The squad member who found the body investigates it, not the wrong one.
- The investigator walks to the cover it found, not a cleared spot.
- The corpse reaction survives firing before its stage is set.
- A dead stalker's danger no longer lingers on a reused id.

Target selection:
- A shot-at stalker turns on the man who hit him, not a distant target he can see.
- NPCs no longer converge on you far harder than on each other.

Accuracy:
- A real per-rank accuracy curve replaces vanilla's flat clamp.

Combat:
- A stalker no longer fires into the low cover he ducks behind.
- A stalker fires over low cover he can shoot across. The shot check reads eye height, not chest height.

Movement and animation:
- A hit no longer slides a standing stalker across the ground.
- A maneuver never starts on a stalker still playing an animation.

Healing:
- The medkit and bandage lists the engine dropped are restored.
- The bandage gesture waits until the stalker stands still.
- A hurt stalker drops his limp in a fight.
- A healing stalker no longer freezes or ignores an enemy that appears mid-gesture.
- The save-load charge re-roll is suppressed.

Effects (planned):
Player-facing combat feedback, starting with concussion: tinnitus and blur.

Mutants (planned):
Mutants get the same combat treatment as stalkers.

Intended setup:
AlifeTactics is built and tuned against vanilla Anomaly on the latest demonized build.
It runs on AOEngine and older demonized builds with fallbacks, where a feature that needs a newer hook stays inactive or reduced.
It coexists with other combat AI mods, but vanilla plus AlifeTactics is the intended setup.

Requirements:
Anomaly 1.5.3
Modded exes: themrdemonized 20250908 or newer, or AOEngine v0.55 or newer. The full feature set needs the latest demonized build. A feature that needs a newer one stays inactive on older exes.
xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Compatibility:
Depends only on xlibs. Install and uninstall mid-save work. Tested: Anomaly 1.5.3, GAMMA, EFP, Zona, Forgotten Zone.
Disable (conflict, superseded, problematic):
- Animated NPC Healing, and NPC-healer replacements - run in place of the Healing system.
- G.A.M.M.A. AI Rework - blocks the combat planner and re-enables broken vanilla subschemes, so NPCs shoot through cover and never flank.
  It also keeps them out of your fights past 173m and reseeds RNG at load, on dead code with some bugs.
- G.A.M.M.A. No NPC Friendly Fire, and community friendly-fire blockers - re-filter the friendly hits Crossfire already handles and rewrite NPC relations on hit.
- G.A.M.M.A. No logs and Log spam remover - GAMMA log-suppression mods, off by default, that disable the Accuracy system if enabled.
- G.A.M.M.A. NPCs Faster Reactions - raises NPC sight range and detection, so NPCs see across open ground and swamp the Vision curve.
- NPC Limping and Healing (Vodoxleb) - stacks the limp and heal animation and breaks the heal cue.
- NPC Weapon Jamming, and any NPC-jam mod - re-adds the jams the Jamming system removes, looping reloads with no end.
- RE:DONE Combat AI - drives combat aim itself and leaves the game's aim settings altered on removal.
- Useful Idiots (bellyillish) - a broad combat-AI overhaul that races the Danger scheme.
- Worse NPC Vision and Accuracy, and any mod with its own NPC vision config - override the Vision and Accuracy systems.
It coexists with everything else.

Not AlifeTactics (base game or your setup):
- Stalkers gliding or staggering when shot - the engine moves the body while a hit animation plays, present in unmodded Anomaly.
- Stalkers ignoring darkness - your visibility script applies its darkness curve only from 21:00 to 04:00, so dusk and dawn read as full daylight.
- Stalkers seeing across open ground - sight range comes from your creature config, not from acquisition speed.
- Stalkers reacting to danger across the map - vanilla's danger table uses duration values, so a shot counts as danger past 300m.

How It's Built:

Although it started from work by Demonized, Alundaio, and Tronex, the current code and patterns are original, learned through reverse-engineering X-Ray, load testing, and custom X-Ray changes.
The design favors the engine's own mechanisms and minimal intervention, with event-native pub/sub over polling.
Work spreads across frames through deferred queues and rate limiters, while per-level caches replace world scans.
The raycasting and range math are hand-written and tested live, and the code follows the engine's own standards and flags.
Performance is the first invariant. Every flow stays under 2ms, and the build rewrites or drops anything that misses.
Profiled continuously with JitProfiler, an engine-native profiler. Manual tests run on unoptimized, single-threaded exes.
The code carries tracing and monitoring from the ground up, with every flow timed off the log level.
Every commit runs the full pipeline locally and in CI: luacheck, a Selene build compiled for STALKER with flags the public build lacks, and a load test that runs every script against engine stubs.
Rule layers then check crash safety, hotpath cost, engine correctness, complexity, architecture contracts, security, and the docs.
Every mod is configurable through MCM or LTX, down to each rate, threshold, and toggle, with nothing tunable left hard-coded.
The mod avoids writing engine values, holding its own state in parallel. Any value it must change stays inside the engine's own bounds, so save corruption is impossible.
The family runs on one rulebook through xlibs. Every rule, policy, and check is one shared implementation, the same protection, distances, faction logic, and combat reads in every mod.
It depends on no other mod, not even the author's own. The only shared layers are X-Ray and xlibs.

That pipeline runs on every commit and publishes what it finds. The header links a live health page and a JitProfiler capture of the mod's real CPU and allocation cost.

Credits:
Altogolik provided support, ideas, and source materials.

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "AlifeTactics by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  The full license is in the LICENSE file and on GitHub.

Diagnostics and reporting:
Every release goes through careful engineering and testing, but bugs can still slip through.
To report one, reproduce with debug logging on, and the world log where the mod has one.
First rule this mod out: reproduce with it off, then on. The cleanest test is this mod alone on vanilla and xlibs.
Send the traces on the Anomaly Discord, or file a defect on GitHub with the same information.
Attach xray.log, the mod log, the engine build, the modlist, and the load order.
For deep technical details and mechanisms, check the architecture docs on GitHub.

Tags: alife, combat-ai, mutant-ai, mutants, npc, goap, tactical, realistic, emergent, engine-native, self-preservation, accuracy, stealth, perception, cover, faction, loot, ammo, performance, save-safe
