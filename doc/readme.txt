AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: next (xlibs 1.8.5, demonized 20250908)
GitHub: https://github.com/damiansirbu-stalker/AlifeTactics
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeTactics/issues

Alife Collection:
AlifeAmbience: https://github.com/damiansirbu-stalker/AlifeAmbience
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeCompanions: https://github.com/damiansirbu-stalker/AlifeCompanions
AlifeDiegetic: https://www.moddb.com/mods/stalker-anomaly/addons/diegetic-audio-control-100
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeSpooks: https://github.com/damiansirbu-stalker/AlifeSpooks
AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics
FurnitureFuel: https://github.com/damiansirbu-stalker/FurnitureFuel
JitProfiler: https://github.com/damiansirbu-stalker/JitProfiler
TestZone: https://github.com/damiansirbu-stalker/TestZone
xlibs: https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001

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
- Rank scales accuracy, weapon spread, aim speed, and how tightly a stalker tracks you, which vanilla clamps flat.
- AP ammo is fired from its owner's inventory and consumed, where vanilla gives NPCs infinite cheap rounds and modpacks give infinite AP by rank and map.
- Bugs disguised as features are gone, like fake weapon jamming and random tactical reloading.


Everything is canon, engine-pure, safe, and fast:
- Every behavior is built from pure X-Ray and Anomaly primitives: the GOAP action planner, the xr_logic scheme system, state_mgr, smart terrains and the gulag job system, condlists,
  and DLTX/DXML for data.
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
Maneuver fire bursts by weapon and by skill: burst length and pauses vary shot to shot inside the game's own per-weapon ranges and tighten with rank.
That replaces the uniform burst the game's script machinery applies to every weapon it drives.
Stalkers keep their footing under fire: a hit mid reaction no longer slides a standing stalker across the ground, he stays planted while the animation plays and moves the moment it ends.
The footing hold ships ahead of its engine half and needs a modded exes build carrying engine PR 645. Older builds keep vanilla movement.
The takeover overrides no combat scripts, so it fights side by side with vanilla and works with other combat AI instead of replacing it. Companions are excluded by default.

Commitment:
Vanilla stalkers re-plan the fight every moment, so any small change makes a stalker drop what he is doing and choose again.
Better cover, a flicker of lost sight, or a teammate crossing the line sets him off, which is the twitchy strafing and cover-hopping you see in a firefight.
Many mods answer this by switching the stalker to a camper scheme that pins him in place, muting most of Anomaly's combat variety.
AlifeTactics keeps the engine's full combat AI and instead stops a stalker throwing away a decision that still makes sense.
While what he is doing still works he stays with it, and he switches the instant it stops: he loses sight, the shot is blocked, or the enemy is gone.
It also pins his cover: a stalker firing with a clear shot keeps his spot instead of sliding to a marginally better one, stopping the mid-fight strafe at its source.
When his enemy is caught reloading, out of ammo, staggered, sprinting weapon-down, or with no weapon up, he liquidates: fire held on the window instead of breaking off, repositioning once the enemy can answer again.
He sees a decision through, keeping up his fire, pressing a flank, or finishing a reload, instead of second-guessing himself every frame.

Conduct:
Conduct makes better choices at moments the engine already decides, with no takeover, for stalkers the vanilla engine drives.
Cover posture: experienced riflemen and snipers crouch to steady the shot when the line to the enemy is clear, and stand to fire over low cover that would block a crouched shot.
It replaces vanilla's blind posture picks (crouching behind random bumps and firing into them, or standing tall where a crouch would steady the aim).
They crouch only past close range, staying mobile in a knife fight instead of dropping to a knee at your feet, and only while holding a position rather than on the move to cover.
It reads the game's own cover map toward the enemy, and short-weapon carriers and green ranks keep vanilla behavior.
Weapon spacing: a stalker's cover choices respect what his weapon is good at, submachine gunners accept closer cover so their fire stays effective, and skilled snipers hold extra distance.

Behaviors:
Stalkers act on windows of opportunity in the fight, in both directions.
The Push: the stalkers fighting you punish the moment you cannot answer instead of watching it pass.
Caught reloading, jammed, out of ammo, or badly hurt, their fire thickens at close range, and with a clear upper hand (you weakened, bleeding, or turned away) the attackers move to closer cover.
The Pull is the mirror: a stalker caught reloading or badly hurt under your fire falls back, his own cover choices landing farther from you until he recovers.
Everything reverts the moment the window closes, the moments stay rare rather than constant, and each cause has its own switch.

Effectiveness

Accuracy:
Anomaly's rank curve clamps every NPC to the same dispersion, so accuracy never scaled per rank even though the engine code exists.
AlifeTactics restores a real per-rank curve through the engine's own dispersion callback, tunable per tier.
A second curve covers fire on the move: each rank keeps a share of the movement spread penalty, so rookies spray while repositioning and top ranks cut about a third of it.
The engine has several dispersion variables, including barrel and weapon, and this one is the NPC skill-based dispersion.

Disclosure:
A suppressed hit is a signal, not a broadcast.
The victim turns on the shooter through the engine's own target selection: the real "he hit me" signal outweighs a distant visible enemy, and he returns fire the moment he has line of sight - nothing is revealed through walls.
Squadmates close enough to hear the impact walk over to investigate the shooter's position and open fire only when they actually spot him; distant patrol members are never told.
A loud shot needs no script - the whole area hears it on its own.
A clean instant kill tells no one, and the squad can still find you by sound, by sight, or by the body.
A target-priority dial tunes how strongly NPCs prioritize you over other combatants once you are seen, down to treating you like anyone else.

Crossfire:
Same-faction fighters no longer cut each other down in a crossfire.
A hit between two NPCs of the same faction deals reduced damage, set by a slider (no damage by default, up to full vanilla).
It keys on their actual relation, so genuinely hostile factions still trade fire while allies stay safe. Your own shots are never affected.

Reaction:
Stalker rank now shapes gun handling, applied per stalker.
Tracking Speed sets how fast a barrel moves while tracking, from vanilla at the bottom rank to a decent step above it at the top, well short of the game's hardcore AI aim.
Tracking Lock sets how tightly a barrel holds a strafing target: a novice's aim lags and a moving target loses it exactly like vanilla, a legend holds you across the firing window without tracking you perfectly.
Target Lead aims a stalker ahead of a moving target by the round's real flight time, computed from range and the weapon's bullet speed as it fires.
Higher ranks lead true and hit movers, lower ranks over-lead and overshoot.
Fire Discipline gives higher ranks crisper short bursts at a tighter cadence while low ranks stay vanilla. Defaults keep a rank's rounds per minute at or above vanilla.
Tracking Speed, Tracking Lock, Target Lead, and Fire Discipline are per-rank MCM slider curves. Speed and Lock share one on/off, Lead and Discipline each carry their own.
The two rank vision curves moved to the Perception tab below. Fire Discipline is on its own Discipline tab.

Range (planned):
The game stops NPC fights at a hard range cap regardless of weapon, so a sniper never fires at the distances his rifle exists for.
A stalker sniped from beyond the cap can duck, which is the Danger hit response, but never answer. This page will let long-range NPCs answer and initiate at their weapon's real reach.

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
Reaction follows the evidence: a heard walk or an item used turns him weapon-ready toward the sound.
A sprint, a landing, a racked reload, or an empty click sends him walking over to check the spot.
A sound is never treated as a confirmed enemy: he investigates at a walk, never charges, and does not know where you are.
Nothing changes in combat, and a carry-distance slider scales it, so stealth stays a game of distance and stance instead of NPCs being deaf.
Compatible with stealth mods: stealth in Anomaly is about being seen, through light, cover, and stance, and the sound system never touches vision or detection.
Hearing only adds the short-range sense vanilla lacks, and crouched movement is silent, so the crouched approach your stealth setup allows is never given away by sound.
Neutral and friendly stalkers get one reaction of their own: fire close to them and they go weapon-ready facing your shots instead of ignoring gunfire next to their heads.
They only go alert, never hostile, and settle down when the shooting stops. Every reaction has its own toggle.

Vision:
Stalker rank shapes the eyes as well as the trigger, applied per stalker.
Vision Speed sets how fast each rank turns a glimpse into a confirmed threat, from your setup's own detection speed at the bottom rank (a novice matches it) to about 21 percent faster at the top.
No rank notices slower than your baseline.
It scales the rate only: sight range, vision cone, light and darkness response, cover and occlusion, and hearing all stay exactly as your setup has them.
Vision Range sets how far out each rank begins to notice a threat, the same band, from your baseline at novice to about 15 percent farther at the top.
Both are per-rank MCM slider curves under one Vision toggle, on their own Vision page.

Danger:
A stalker hit from far off reacts even when vanilla would have him stand still, so sniped at 200m he turns and seeks cover.
The reaction is the duck, and returning fire at that range is the planned Range page above.
A set of always-on fixes clears long-standing crashes and misreads across the danger check and the corpse investigation, and these are not optional, shown as locked toggles.
One tuning toggle lets danger you cause read your setup's separate player-specific ranges where the config provides them.
One of those fixes shortens reactions instead of lengthening them.
Vanilla's danger table gives three of its entries the same key, so two threat types are read under the wrong range.
A ricochet reads at the 150m sight range instead of its own 4m, and an attacked ally at 150m instead of 50m.
That is why a vanilla stalker can wheel around at a spark off a wall halfway across the map.
AlifeTactics separates the keys, so both reactions happen at the distance they were written for. A modpack that already corrected this sees no change.

Mechanics

Healing:
Wounded NPCs heal with the items they carry.
Vanilla's medkit heal fires unreliably and bandages do nothing, so bleeding stalkers die that should not.
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
Rank threshold and consumption rate are tunable in configs/alifetactics/at_ammo_config.ltx.

Gear:
Items a stalker carries give him combat advantages, read from each item's own game data, so artefacts from any mod work.
An artefact grants one advantage chosen by its anomaly class, scaled by its own tier: gravity, chemical, and armour-plate artefacts cut the damage he takes.
Thermal artefacts tighten his fire, and electric and quest artefacts raise the damage he deals.
Any single artefact tops out at 10 percent and never stacks, the strongest source wins, so gear tilts a fight and never decides one.
A chemical artefact instead heals its carrier slowly over time, in place of the damage cut above.
Any artefact carrier warps the air around his body, so a distorting stalker is a real, huntable artefact drop.
Binoculars extend his sight range by day and night-vision by night.
Effect strengths and the artefact class tables are tunable in configs/alifetactics/at_gear.ltx.

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

Performance:
Performance comes first, ahead of any feature. Every combat command goes through xcombat into the engine's own mechanisms, nothing runs per frame, and every flow is timed with a hard 2ms ceiling.
When a feature cannot fit the budget it is reworked, replaced, or removed with an X-Ray engine modification rather than allowed to slow the game.
It is measured on the engine built from the latest source with no multithreading and no optimizations, so the timings are worst-case, and the optimized multithreaded build you run is always faster.

Compatibility:
Tested with vanilla Anomaly 1.5.3 and GAMMA, and installing or uninstalling mid-save works.
The takeover leaves combat scripts vanilla, and the danger rework patches at runtime, so AlifeTactics layers cleanly onto other combat and AI mods.
Friendly fire and same-community hits are filtered at the faction-relation gate.
Story NPCs, companions, traders, and squadmates are never armed against their own faction.

Disable or patch these, each one breaks an AlifeTactics system:
- NPC Limping and Healing (Vodoxleb): plays its own limp and heal animations, the same ones the Healing system uses. The two stack and the heal cue breaks. Disable one side.
- NPC Weapon Jamming, and any mod that jams NPC guns: Anomaly has no NPC jam animation, so a jammed stalker loops his reload with no end.
  The Jamming system removes NPC jamming for that reason, and these mods put it back. Player-side jamming (Weapon Parts Overhaul, the GAMMA jam mods) is a separate system and stays untouched.
- G.A.M.M.A. No NPC Friendly Fire, and any community-based friendly-fire blocker: the Crossfire system already filters friendly hits at the per-NPC relation gate.
  The community version filters a second time and rewrites NPC relations on each hit. Disable it.
- G.A.M.M.A. NPCs Faster Reactions: raises stalker sight range from 160 to 220m (Monolith 275).
  That also raises detection speed at every distance inside the range and overrides the Vision Speed curve, so every rank detects at the raised range.
  Patch the two range keys back with DLTX. Its raised occlusion threshold is the part worth keeping.
- Worse NPC Vision and Accuracy, the accuracy half (the DLTX_JURASZKA build included): multiplies the eight base NPC dispersion values in m_stalker.ltx by 6 to 18 times.
  The Accuracy system multiplies the engine shot-dispersion cone by a per-rank factor on top of that inflated base, so every rank fires far too wide and the rank spread is swamped.
  Keep its vision changes and remove its dispersion (accuracy) DLTX, or turn the Accuracy system off in MCM to keep its dispersion. The vision half is separate and does not touch AlifeTactics.
- Animated NPC Healing, and NPC-healer replacements: their healer runs in place of the Healing system, so the heal rate and per-rank charge do nothing where they overlap. Disable one side.
  A DLTX mod that rewrites the vanilla medkit list can also empty the AlifeTactics list by load order.
- G.A.M.M.A. "No logs" and "Log spam remover" (off by default): their outdated _g.script drops the engine dispersion forwarder, which silently disables the Accuracy system. Keep them off.

Works alongside, with a note:
- RE:DONE Combat AI: its combat half composes with AlifeTactics. Separately, it writes the four global ai_aim console variables in degrees where the engine reads radians (25 to 79 times the defaults),
  never restores them, and the game saves them into user.ltx, so the change outlives uninstalling. Restore by hand:
  ai_aim_max_angle 0.7854, ai_aim_min_angle 0.19635, ai_aim_min_speed 0.24, ai_aim_predict_time 0.4.
- The game's Hardcore AI aim option, and any ai_aim console tuning: the Reaction system writes per-stalker fields and never below the global baseline, so a higher difficulty is always kept.
- GAMMA Stealth Overhaul, and detection-threshold mods: Vision Speed multiplies your setup's own detection result as a rank curve (novice unchanged, legend about 21 percent faster).
  Set every rank to 1.00, or clear the toggle, to hand acquisition back to your setup.
- Improved Visual Awareness, Stealth 2.31, RE:VISION, RE:DONE Combat AI, and any other mod shipping the visibility script: only one copy loads.
  The loser's stealth model stops running while its MCM sliders stay on screen. Pick one.
- enemy_shoot_back, NPC Improvements, Kebab's NPC Overhaul, and any mod shipping a full m_stalker.ltx: the same single-winner rule for sight range, vision cone, and dispersion.
  The losing copy does nothing.
- Useful Idiots, and any companion mod with a GOAP surge or shelter scheme: AlifeTactics reserves GOAP id 188347 for its combat takeover, planted on every NPC and dormant until a maneuver runs.
  Other schemes on the stalker planner must not reuse that id. It was moved off 188200, which Useful Idiots uses for its emission-shelter scheme.
  On the old id the two clashed and companions could not take cover during emissions.
- xrMPE Animations, and other NPC animation packs: every pose AlifeTactics plays exists in their files (verified), so its cues take on their look.
  Their larger, longer hit and hurt animations can make the base-game gliding and staggering more visible, which is base-game hit handling (see "Not AlifeTactics" below),
  not anything AlifeTactics adds.
- g_ai_unlimited_ammo set to 0 (newer engine builds): the Ammo system detects it and goes inert, so carried AP is not drained twice. At the default 1 it runs normally.
- G.A.M.M.A. Ballistics Overhaul and Close Quarter Combat (both ship the grok_bo hit system): a hit you land on an NPC is recomputed from the weapon's own values and applied by grok_bo itself, which cancels that NPC's artefact damage resistance. AlifeTactics restores it: it takes over the grok_bo NPC hit and reapplies the resistance to the damage grok_bo actually dealt, so a geared NPC still resists your shots. It hooks whichever of the two wins load order, is inactive when neither is installed, and changes nothing for hits between NPCs.
- G.A.M.M.A. Actor Damage Balancer: finalizes damage the player takes, reading the hit's power and then applying the damage itself. A modifier this mod makes to a hit against the player still reaches final damage, because the balancer reads that power before applying it and this mod's scripts (at_, ap_) load ahead of grok_ by name. That order is fixed by the file names, so it holds on any standard install. Only a damage mod whose scripts sort ahead of both could take it over.

Works as-is, no setup:
- G.A.M.M.A. AI Rework, RE:DONE Combat AI, RE:VISION, AI More Cover, Wuut AI Extension, NPC_Fleeing, Mora's AI More Covered, No More Companion Friendly Fire, Tougher Important NPCs and Companions, Dynamic AI Aim Settings.
  The takeover blocks the combat planner only while a maneuver runs, then hands back, and the danger rework reads their ranges instead of substituting its own.
  Turn Combat off in MCM to leave a planner-action mod fully in charge. Where one also ships a visibility script or the aim globals, that part is under "Works alongside" above.

Not AlifeTactics (base-game behavior):
- Stalkers gliding or staggering oddly when shot: the engine starts or continues movement while a hit reaction animation still plays, and the hit impulse shoves the body with no animation at all.
  Present in unmodded Anomaly. Animation packs (xrMPE among them) can make it more visible because their reactions are longer and larger.
  AlifeTactics never starts a maneuver or re-applies state on an animating body, so it adds no instances of its own.
- Stalkers who ignore darkness: your modpack's visibility script applies its darkness curve only between 21:00 and 04:00, so dusk and dawn compute as full daylight.
  In GAMMA the lever is michiko_patch on the Stealth MCM page, off by default. AlifeTactics deliberately runs no code on the per-sight-test path, so it cannot reach this.
- Stalkers who see across open ground: sight range from your setup's creature config, not acquisition speed. See NPCs Faster Reactions above.
- Stalkers who react to danger from across the map: vanilla fills its danger reaction-distance table with duration values, so a corpse, a ricochet or a shot counts as danger out past 300m.
  AlifeTactics reads whatever your setup ships. GAMMA's Stealth Overhaul rewrites the band to 75-125m, the scale it was meant to be.
- Stalkers who lose interest oddly, or whose pursuit persistence differs run to run:
  GAMMA's Stealth Overhaul also carries an older xr_combat_ignore that wins the script slot in every GAMMA install and overrides the modded-exe version,
  removing its enemy-id validation and randomizing the combat-memory window each session.
  AlifeTactics reads whichever xr_combat_ignore won and never replaces it.

FAQ:
Do I need modded exes?
  Yes. AlifeTactics needs themrdemonized modded exes (2025.9.10 or newer) or AOEngine (v0.55 or newer). Vanilla Anomaly does not expose the APIs it relies on.

Credits:
Altogolik: support, ideas, source materials

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "AlifeTactics by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  Full license in LICENSE file and on GitHub.

Reporting issues and suggestions:
Open a report at https://github.com/damiansirbu-stalker/AlifeTactics/issues/new/choose, or ask on the GAMMA, EFP, Anomaly, and Zona Discord servers. Read this readme and the MCM options first.

Combat is the hardest thing in Anomaly to diagnose, so first confirm it is this mod: reproduce, disable AlifeTactics, reproduce again. If it persists it is not this mod.
The cleanest test is vanilla Anomaly plus xlibs plus AlifeTactics.

Include: exact repro steps (new game or named save, expected vs actual), confirmation the issue disappears with AlifeTactics off, engine build, modlist, load order, xray.log, and the mod debug log.
Only the log shows whether this mod was involved.

The debug log is required: set the MCM log level to DEBUG, reproduce, then back to WARN. DEBUG is not free.
It writes a timed line for every evaluation and hitches single-threaded exes, and the millisecond figures include the tracing itself, so treat them as relative.
