Version: 1.2.2-snapshot (xlibs 1.9.0, demonized latest)
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Health: https://damiansirbu-stalker.github.io/AlifeTactics/health/
JitProfiler: https://damiansirbu-stalker.github.io/AlifeTactics/jitprofiler/
Bugs: https://github.com/damiansirbu-stalker/AlifeTactics/issues
Russian / На русском: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/readme_ru.txt
Manifesto: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/manifesto.md

---
Alife mods:
  AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
  AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics
  AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
  AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
Diegetic mods:
  DiegeticControl: https://www.moddb.com/mods/stalker-anomaly/addons/diegeticcontrol
  DiegeticAmbience
  DiegeticDread
Tools:
  JitProfiler: https://www.moddb.com/mods/stalker-anomaly/addons/jitprofiler
Libraries:
  xlibs: https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001
Engines:
  X-Ray Monolith: https://github.com/themrdemonized/xray-monolith/pulls?q=is%3Apr+author%3Adamiansirbu+is%3Amerged
  OpenXRay: https://github.com/OpenXRay/xray-16/pulls?q=is%3Apr+author%3Adamiansirbu+is%3Amerged
  IX-Ray: https://github.com/ixray-team/ixray-1.6-stcop/commits/develop?author=damiansirbu
Dependents:
  COMPASS: https://github.com/Crimento/COMPASS
  Nemesis Spawn System: https://github.com/SirEndet/Nemesis-Spawn-System
  Word of Mouth: https://www.moddb.com/mods/stalker-anomaly/addons/word-of-mouth
Integrations:
  Warfare (erepb): https://www.moddb.com/mods/stalker-anomaly/addons/warfare-alife-overhaul-new
  Homestead: https://www.moddb.com/mods/stalker-anomaly/addons/homestead-stalker-settlement-builder
  Stealth Overhaul: https://github.com/Alex-leon1594/Stealth_Overhaul_Reworked
  TALKER: https://www.moddb.com/mods/stalker-anomaly/addons/talker-ai-powered-npc-dialogue-beta
  Factions of the Zone: https://github.com/devinhorowitz/factionsofthezone
  Anomaly Radios Extended: https://www.moddb.com/mods/stalker-anomaly/addons/anomaly-radios-extended-full-remaster-152
Collaborations:
  xAGNA: https://www.moddb.com/mods/stalker-anomaly/addons/xagna

[ Hero image: alifetactics-hero.gif - stalkers fight to survive ]

My work is free and open source, ships in most modpacks, and integrates with established projects.
An organized group copies my work, defames, impersonates and review-bombs me across platforms.
You can help with a review, a rating, a proper bug report, or a coffee on Ko-fi: https://ko-fi.com/damiansirbu

Every release is stable and whole new systems may appear. Always understand the documentation.
After every update, use Reset ALL to Defaults in MCM under Development.

Everyone wants to live.

AlifeTactics rebuilds how creatures behave and fight in STALKER Anomaly.
The engine is strong mechanically, but its enemies fight stereotypically, their actions shuffled and settled by a die roll.
AlifeTactics adds the intelligence to choose the action, and a Personality for each faction and each mutant species over which action it takes at all.
Sections marked (planned) are not built yet.

It rebuilds the combat mind GSC designed and shelved, the intelligence their own AI documents lay out.
GSC built it first and cut it. The AI grew so sharp that stalkers won the firefights before the player could, and the studio reined it in.
Much of it only ever lived as stubs and commented-out code in the engine source, revived here on the current engine.
AlifeTactics is engine-first. Where the engine could not decide, dozens of changes went into X-Ray itself and shipped in the official modded exes.
Every behaviour then runs on the engine's own machinery.

Every system reads the real state of the game, then decides in the combatant's favor rather than by script or die roll.
It weighs combat events, NPC and player stats, the world, squad, faction, weapons, range, angle, and cover.
Maneuvers take a stalker over for an action the vanilla engine has no mechanism for.
Commitment holds a stalker to a decision while it still makes sense.
Mutants run the same decision model, reading your weak moments and answering with their own body's moves.
Threat, accuracy, and the rest read state and decide the same way.

One formula runs every decision in the mod, stalker or mutant.
A premise, the read of the fight, opens the decision. Intelligence, rising with rank, is how sharply he reads it.
Personality is whether his kind takes that action at all. Both must pass, and a passed decision commits to its window.
A bandit and an ecologist do not carry the same instincts.
Each faction carries its own chance on every action, and each mutant species its own chance on every move it has, from 0 (never) to 1 (always).
Rank buys the correct answer more often. The ability set stays fixed across ranks. A novice and a legend both act, and the legend is the one who gets it right.

Effects:
- Creatures make emergent decisions from environment, own state, enemy state, squad state, weapons, and faction doctrine.
- Every faction fights its own way: who routs, who presses your weak moment, who plants for deliberate shots, who crouches on the line, who never backs down.
- Mutants answer your weak moments and punish the cheese: reload and a leap comes, perch on a rock and the jumpers come up after you, move and they steer in the air to follow.
- Every mutant ability value varies inside its species' own envelope. No two spawns read alike. Movement stays the game's own tuning. The variation is one dial.
- Nothing is player-centric, and no side gets an advantage or a handicap.
- Creatures use the items they acquired themselves: medkits, ammo, props.
- Rank scales accuracy, weapon spread, aim speed, and how tightly a stalker tracks you, which vanilla clamps flat.
- AP ammo is fired from its owner's inventory and consumed, where vanilla gives NPCs infinite cheap rounds and modpacks give infinite AP by rank and map.
- Bugs disguised as features are gone, like fake weapon jamming and random tactical reloading.


Every system in this mod is a behaviour GSC designed, or one the engine left unfinished in code. The manifesto documents the source behind each.


Everything is canon, engine-native, safe, and fast:
- Every behavior is built from pure X-Ray and Anomaly primitives:
  the GOAP action planner (the planner family F.E.A.R. made famous), the xr_logic scheme system, state_mgr, smart terrains and the gulag job system, condlists, and DLTX/DXML for data.
- Nothing is faked and nothing is simulated beside the engine.
- Where the engine had no seam, the seam was added upstream first: per-NPC hooks created in xray-monolith specifically for this mod, merged into the official modded exes.
- It never replaces a vanilla script file. Takeovers block the planner only for their seconds and then hand back, and patches lay onto whichever script your setup ships.
- Every value ships as a formula over the engine's own constants, never an invented number.
- No per-frame Lua. Everything runs on engine callbacks and scheduled passes, and the whole mod costs 5 timer compares per frame, whatever the NPC count.
- AlifeTactics never wraps the engine's visibility function. No sight test gains a cost or a behavior it did not already have. Your stealth setup computes being seen exactly as before.
- Perception is written as a per-stalker engine field once at spawn. It multiplies onto whatever detection system is installed instead of replacing it.
- It fixes dozens of vanilla Anomaly bugs: dead branches, wrong calculations, and plain crashes in original code.


Every system below has its own MCM page in this order, where it is toggled and tuned.

Combat

Faction Personality and Intelligence:
Intelligence is how sharply a stalker reads a fight, and it scales with his rank, a base plus a bonus per rank, two sliders on the General page.
A novice fumbles the openings a veteran takes. His judgment emerges from his rank, where vanilla clamps every NPC to one flat setting.
It caps short of certain, and even a legend is never guaranteed the right call.
Then the faction's Personality decides whether his kind takes that action at all, every action carrying a per-faction chance from 0 (never) to 1 (always).
Monolith presses relentlessly, Duty holds the line and gives ground in order, bandits finish a downed man and rout when the fight turns, ecologists pull back early.
One opening draws a different answer by faction.
The per-action faction table is the faction config, yours to tune. Zombied stalkers never participate in anything.

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
One shared allowance bounds everything AlifeTactics starts, against you and against NPCs separately. A mass battle never drowns either side.
A maneuver fires when its problem is real and ends when it is solved. Keep creating the problem, such as pressing a shotgunner's minimum range, and the answer keeps coming.
The decisions come out looking human. Nobody turns his back on a shooter, two men never take the same cover, nobody plants in his enemy's sight.
Maneuver fire bursts by weapon and by skill: burst length and pauses vary shot to shot inside the game's own per-weapon ranges and tighten with rank.
That replaces the uniform burst the game's script machinery applies to every weapon it drives.
Stalkers keep their footing under fire. A hit mid reaction no longer slides a standing stalker across the ground. He stays planted while the animation plays, and moves the moment it ends.
The footing hold ships ahead of its engine half and needs a modded exes build carrying engine PR 645. Older builds keep vanilla movement.
The takeover overrides no combat scripts. It fights side by side with vanilla and works with other combat AI. Companions are excluded by default.

Commitment:
Vanilla stalkers re-plan the fight every moment. Any small change makes a stalker drop what he is doing and choose again.
Better cover, a flicker of lost sight, or a teammate crossing the line sets him off, which is the twitchy strafing and cover-hopping you see in a firefight.
Many mods answer this by switching the stalker to a camper scheme that holds him in place, muting most of Anomaly's combat variety.
AlifeTactics keeps the engine's full combat AI and instead stops a stalker throwing away a decision that still makes sense.
While what he is doing still works he stays with it. He switches the moment it stops working, on a lost sight line, a blocked shot, or a vanished enemy.
It also holds his cover. A stalker firing with a clear shot keeps his spot. He no longer slides to a marginally better one, and the mid-fight strafe stops at its source.
When his enemy is caught reloading, out of ammo, staggered, sprinting weapon-down, or with no weapon up, he liquidates.
He keeps firing through the window and repositions once the enemy can answer again.
He sees a decision through without second-guessing himself every frame.

Conduct:
Conduct makes better choices at moments the engine already decides, with no takeover, for stalkers the vanilla engine drives.
Whether a fighter holds the discipline is his training, settled once in his life and weighed by faction. Militarized stalkers keep it and scavengers slouch.
Cover posture: experienced riflemen and snipers crouch to steady the shot when the line to the enemy is clear, and stand to fire over low cover that would block a crouched shot.
It replaces vanilla's blind posture picks, such as a crouch behind a random bump, or standing tall where a crouch would steady the aim.
They crouch only past close range and only while holding a position. They stay mobile in a knife fight and while moving to cover.
It reads the game's own cover map toward the enemy, and short-weapon carriers and green ranks keep vanilla behavior.
Weapon spacing: a stalker's cover choices respect what his weapon is good at, submachine gunners accept closer cover so their fire stays effective, and skilled snipers hold extra distance.

Behaviors:
Stalkers act on weak moments in the fight, in both directions and against any enemy, you or another combatant.
The Push: a stalker whose enemy cannot answer presses him.
Caught reloading, out of ammo, or badly hurt, the target's attackers thicken their fire at close range.
With a clear upper hand (the target hurt or turned away) they move to closer cover.
The Pull is the mirror. A stalker caught reloading or badly hurt while his enemy is strong falls back, his cover choices moving farther out until he recovers.
Everything reverts the moment the target can answer, each attacker presses briefly with a cooldown before pressing again, and each cause has its own switch.

Effectiveness

Accuracy:
Anomaly's rank curve clamps every NPC to the same dispersion. Accuracy never scaled per rank even though the engine code exists.
AlifeTactics restores a real per-rank curve through the engine's own dispersion callback.
A second curve covers fire on the move. Each rank keeps a share of the movement spread penalty. Rookies spray while repositioning and top ranks cut about a third of it.
Every rank curve in the mod is two sliders, a base and a percent per rank. Retuning a whole curve is one motion instead of eight.
The engine has several dispersion variables, including barrel and weapon, and this one is the NPC skill-based dispersion.

Crossfire:
Same-faction fighters no longer cut each other down in a crossfire.
A hit between two NPCs of the same faction deals reduced damage, set by a slider (no damage by default, up to full vanilla).
It keys on their actual relation. Genuinely hostile factions still trade fire while allies stay safe. Your own shots are never affected.

Reaction:
Stalker rank now shapes gun handling, applied per stalker.
Tracking Speed sets how fast a barrel moves while tracking, from vanilla at the bottom rank to a decent step above it at the top, well short of the game's hardcore AI aim.
Tracking Lock sets how tightly a barrel holds a strafing target. A novice's aim lags and a moving target loses it, exactly like vanilla.
A legend holds you across the firing window without tracking you perfectly.
Target Lead aims a stalker ahead of a moving target by the round's real flight time, computed from range and the weapon's bullet speed as it fires.
Higher ranks lead true and hit movers, lower ranks over-lead and overshoot.
Fire Discipline scales burst size and cadence per rank.
The shipped defaults are near-flat. Rank barely alters fire out of the box. The sliders allow the full spread.
Defaults keep a rank's rounds per minute at or above vanilla.
Tracking Speed, Tracking Lock, Target Lead, and Fire Discipline are each a base plus percent-per-rank slider pair. Speed and Lock share one on/off, Lead and Discipline each carry their own.
The two rank vision curves moved to the Perception tab below. Fire Discipline is on its own Discipline tab.

Range (planned):
The game stops NPC fights at a hard range cap regardless of weapon. A sniper never fires at the distances his rifle exists for.
A stalker sniped from beyond the cap can duck, which is the Danger hit response, but never answer. This page will let long-range NPCs answer and initiate at their weapon's real reach.

Perception

Perception is how a stalker senses and reacts. It covers what he hears, how his rank shapes his sight, and how he reacts to being hit.
The Sound and Danger reactions are a runtime patch laid onto whichever danger script your modpack ships. They work with other combat AI.
Detection distances stay owned by your setup's danger config, and AlifeTactics adds only the reactions.

Sound:
In vanilla a stalker hears a shot anywhere inside its reach and answers it with the engine's own cover, look-out, detour and search. AlifeTactics leaves that plan alone.
Everything quieter barely reaches him: a running step only inside about 6 m, a reload inside 4 m, a walk never.
On the multi-thread exes his sound memory also refuses every sound from a player he has not seen or heard for a minute (two on GAMMA). A quiet sound never starts anything.
AlifeTactics answers that refusal. For half a second after a sound of yours loud enough to keep, the hostile hearer's enemy check says yes for you.
His sound memory then stores the sound and the engine runs its own reaction, from the first step, with no earlier sighting.
What the engine still cannot keep, your footsteps and handling noise, stamps the danger scheme at the source with an evidence grade the scheme already runs.
A faint stamp turns a standing stalker weapon-ready toward its source, and solid evidence sends him over to check the spot himself.
A moving stalker ignores a faint stamp, and no sound alone ever drives a run.
Nearby creature cries draw a lighter response, a head-turn a standing stalker gives out of combat that leaves him in place and outside the danger scheme.
Mutant cries always earn that glance, another stalker's only when the two are hostile.
Your bullet landing within 30 m of a neutral or friendly stalker puts him on alert without turning him hostile. Off, he ignores it as vanilla does.
A step's noise scales with stance, surface, and weather, and a crouch makes none at all.
The stamp channel stays inside the scripts and never writes the engine's bounded sound memory.
Sound never touches vision, seen-memory, or the sight test. It reads the config's own hearing sensitivity. A setup that deafens the AI quiets this too.

Vision:
Stalker rank shapes the eyes as well as the trigger, applied per stalker.
Vision Speed sets how fast each rank turns a glimpse into a confirmed threat, from your setup's detection speed at the bottom rank (a novice matches it) to about 21 percent faster at the top.
Every rank notices at least as fast as your baseline.
It scales the rate only. Sight range, vision cone, light and darkness response, cover and occlusion, and hearing all stay exactly as your setup has them.
Vision Range sets how far out each rank begins to notice a threat, the same band, from your baseline at novice to about 15 percent farther at the top.
Both are base plus percent-per-rank slider pairs under one Vision toggle, on their own Vision page.

Danger:
A stalker hit from far off reacts even when vanilla would have him stand still. Sniped at 200m he turns and seeks cover.
The reaction is the duck, and returning fire at that range is the planned Range page above.
A Player target pull dial tunes how strongly NPCs prioritize you over other combatants once you are seen, down to treating you like anyone else.
A hit victim turns on his shooter through the engine's own target selection, always on: the real "he hit me" signal outweighs a distant visible enemy,
and he returns fire the moment he has line of sight. Nothing is revealed through walls, and a suppressed kill tells no one.
The vanilla danger-check and corpse-investigation fixes are listed under Fixes to Vanilla below.

Mechanics

Healing:
Wounded NPCs heal with the items they carry.
Vanilla's medkit heal fires unreliably and bandages do nothing. Bleeding stalkers die that should not.
AlifeTactics has them spend real medkits when injured and real bandages when bleeding, falling back to a per-rank charge only when empty.
The heal rate is tunable. Fixed limp and heal animations show it, out of combat only.

Jamming:
Eliminates the fake NPC jams and the tactical-reload loop they cause.
On modded exes an NPC rolls a per-rank jam from ammo spent. Even a full-condition rifle chokes and reloads every few rounds.
AlifeTactics turns that roll off for NPCs. They now jam only for real and reload only once the magazine is spent, which you can confirm in the debug log.
Your own weapon still jams normally when worn.

Ammo:
NPCs fire the ammunition they actually carry.
Veteran-rank and higher stalkers use armor-piercing rounds from their own inventory, with real ballistics, and fall back to standard rounds once it runs out.
That AP comes from trade and looting through the Alife Collection. What an NPC scavenged shapes how dangerous he is. NPCs drop no AP as loot.
Rank threshold and consumption rate are tunable in the ammo config under configs/alifetactics/.

Attachments:
Stalkers fit the scopes, silencers and grenade launchers they carry onto their own guns.
Vanilla mounts only a fitting silencer or launcher, and only on an NPC's main gun.
AlifeTactics adds scopes and the second gun. A stalker mounts them once he is out of a fight and his hands are free, and a looted or bought scope shows on his rifle at the next quiet moment.
Companions mount what they loot. What you hand a companion stays exactly as you gave it.
A mounted scope tightens a stalker's aim by about a third when he shoots from a stop, as the base game does for scoped NPCs. It does not let him see farther on most rifles.

Gear:
Items a stalker carries give him combat advantages, each read from its own game data. An artefact from any mod works the same way.
An artefact grants one advantage chosen by its anomaly class, scaled by its own tier. Gravity and armour-plate artefacts cut the damage he takes.
Thermal artefacts tighten his fire, and electric and quest artefacts raise the damage he deals.
Any single artefact tops out at 10 percent, and only the strongest applies. Gear tilts a fight without deciding it.
A chemical artefact heals its carrier slowly over time.
Any artefact carrier warps the air around his body. A distorting stalker is a real, huntable artefact drop.
Binoculars extend his sight range by day and night-vision by night.
Effect strengths and the artefact class tables are tunable in the gear config under configs/alifetactics/.

Mutants

Vanilla mutants are a straight line. They run at you, they swing, and they die to anyone who stands on a rock or strafes on cooldown.
AlifeTactics gives every mutant the decision model stalkers run. It reads your reload, your aim, your high ground, and your strafe, and it answers with moves its own body already has,
the ones vanilla fires blindly and the ones vanilla never fires at all.
The engine drives every mutant as it always has. AlifeTactics scales a mutant's stats. It commands moves the engine itself validates and runs. It never takes a mutant over.
Rank is the map tier, from 0 on the starting maps to 5 at the endgame. A Cordon dog barely changes while a deep-Zone chimera is far deadlier.
The whole system needs the latest demonized build and stays inactive on older exes and on AOEngine.

Power:
A mutant hits harder and takes less as the Zone deepens. Damage dealt and damage resistance each grow by a percent-per-rank slider.
Rank 0 stays vanilla, and every value multiplies the installed balance. A mutant overhaul keeps its numbers.

General:
Observing is free, deciding is hard. Every mutant sees your reload from the first fight, and Intelligence is the chance it answers with the RIGHT move, a base plus a bonus per map rank.
Personality then sets which moves its species favours, a chimera leaping and clawing without pause where a flesh bolts early. The final chance is Intelligence times Personality.
A missed roll lets the opening go. A dull mutant wastes chance after chance while a sharp one punishes each.
Every move you see has a real cause behind it, and a cap holds a pack to waves.
Wild is how much a mutant's own ability numbers vary between spawns, the gravity, telekinesis, aura, tube, stamina, and detection values.
At 0 a kind is identical and learnable. At 100 no two spawns read alike.

Species:
Each species gets its own page carrying only the moves its body has. First come the managed moves, the creature's own attacks and the values AlifeTactics varies.
Then come the new moves the vanilla game never fires for it.
The leap family reads your aim, your movement, and your perch. Sudden leap strikes at once, with no wind-up, when you aim at it, reload, or turn your back.
Adjusting leap steers mid-air to follow you while you move. Intercepting leap aims at where you will be when you run in a line.
Charged leap, the one leap with a wind-up, springs up at a shooter above it. Climbing a rock or a ledge stops being safe. The bloodsucker closes the gap with its own Closing leap, then claws.
Backstrike arms a spin strike on an enemy circling behind, once per individual, on the four species whose models carry the motions. Claw gives the chimera the point-blank pounce vanilla never gave it.
Flee breaks a dying mutant out of the fight, rarely, and only when it is far enough to escape. A mutant with no escape fights to the end.
Cloak sends a bloodsucker invisible when you aim at it or hit it while it is still closing. It re-appears a beat later, and never vanishes once it is already at strike range.
Beyond the moves, each species' own ability numbers breathe: the burer's gravity and telekinesis, the controller's psi tube and stamina and auras, the poltergeist's detection,
each behind its own row, each a bounded variation over the installed balance.
Movement stays the game's own, since varying what the engine's path and attack timing depend on only ever made mutants worse.

Fixes to Vanilla:
AlifeTactics corrects dozens of vanilla Anomaly and xray defects, grouped below by the system each repairs.
Each shows as a locked toggle on the Fixes tab. The always-on ones cannot be turned off. The rest are switched on their own pages.

Danger scheme:
- A name collision made three danger categories read the wrong range. A ricochet read at 150m, not its own 4m. Each now reads its own range.
- A mutant corpse no longer crashes the danger time check.
- A torn-down NPC reference no longer crashes the evaluator.
- A bad danger time value no longer crashes the evaluator.
- A stalker hit by a bullet knows who shot him. He looks at where the shooter is now and takes cover from him.
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

Intended setup:
AlifeTactics is built and tuned against vanilla Anomaly on the latest demonized build.
It runs on AOEngine and older demonized builds with fallbacks, where a feature that needs a newer hook stays inactive or reduced.
It coexists with other combat AI mods, but vanilla plus AlifeTactics is the intended setup.

Requirements:
Anomaly 1.5.3
Modded exes: themrdemonized 20250908 or newer, or AOEngine v0.5.2 or newer. The full feature set needs the latest demonized build. A feature that needs a newer one stays inactive on older exes.
xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Compatibility:
Depends only on xlibs. Install and uninstall mid-save work. Tested: Anomaly 1.5.3, GAMMA, EFP, Zona, Forgotten Zone.
Drop:
- NPC self-heal replacements - run in place of the Healing system.
- G.A.M.M.A. No NPC Friendly Fire, and community friendly-fire blockers - re-filter the friendly hits Crossfire already handles and rewrite NPC relations on hit.
- G.A.M.M.A. NPCs Faster Reactions - raises sight range and detection. NPCs see across open ground and swamp the Vision curve. No toggle.
- NPC Limping and Healing (Vodoxleb) - stacks the limp and heal animation and breaks the heal cue.
- NPC Weapon Jamming, and any NPC-jam mod - re-adds the jams the Jamming system removes, looping reloads with no end.
- RE:DONE Combat AI - a full combat-AI takeover: full-file overrides of the combat schemes, danger, visual memory, GOAP state and aim, replacing the whole layer Alife Tactics works at.
- Worse NPC Vision and Accuracy, and any mod with its own NPC vision config - override the Vision and Accuracy systems. No toggle.
- Companion Spotters - replaces vanilla NPC movement game-wide so one bug breaks every NPC, does per-frame work plus a full world scan,
  and misroutes companions on large maps with a broken vertex check.
- G.A.M.M.A. No logs, Log spam remover - full-file _g.script overrides that drop the modded-exes weapon-accuracy hook the Accuracy system reads.
  Accuracy goes dead, and No logs (a stale 1.5.1 copy) reverts newer engine callbacks the whole game needs.
Avoid:
- G.A.M.M.A. AI Rework - hands combat to broken scripted subschemes in place of the engine brain. Past 173m a normal NPC gets a camper that fires through cover or a "cover" scheme that freezes it.
  Monolith, Sin and ISG run an assault-rush at any range that can crash. It also reseeds the global RNG at load and is half dead code.
Interferes:
- Useful Idiots - a well-liked companion UI, fine to keep. Two combat caveats: companions keep its combat modes when the takeover skips them, and it replaces 1 of Crossfire's 3 friendly-fire fixes.
  Combat runs cleanest with it off, but it works.
- Oleh's Weapons Sounds Tweaks and Fixes (GAMMA) - NPCs hear a shot through every layer of its sound, each as far as that file's AI range.
  Its unsilenced shot carries about 5 m and its echo tail about 50 m, while its silenced shot carries 75-110 m. Vanilla runs the other way, about 200 m against 7-20 m.
  NPCs react to each other's normal gunfire only inside about 50 m, and a mounted silencer makes a stalker easier to hear.
Everything else coexists, as long as it extends X-Ray and Anomaly and never overrides them.

Not AlifeTactics (base game or your setup):
- Stalkers gliding or staggering when shot - the engine moves the body while a hit animation plays, present in unmodded Anomaly.
- Stalkers ignoring darkness - your visibility script applies its darkness curve only from 21:00 to 04:00. Dusk and dawn read as full daylight.
- Stalkers seeing across open ground - sight range comes from your creature config, not from acquisition speed.
- Stalkers reacting to danger across the map - vanilla's danger table uses duration values. A shot counts as danger past 300m.

How It's Built:

The code and patterns are original, built on best practices from the best STALKER modders and hands-on reverse-engineering of X-Ray.
The design stays engine-native and minimal, with event-native pub/sub not polling, work spread across frames via deferred queues and rate limiters, and per-level caches replacing world scans.
The raycasting and range math are hand-written and load-tested live, following the engine's own standards and flags.
Where scripting hits an engine limit, the fix is made in X-Ray itself, in the modded exes.
Performance is the first invariant, so every flow stays under 2ms or the build rewrites or drops it, profiled continuously with JitProfiler and hand-tested on unoptimized, single-threaded exes.
Every mod carries OpenTelemetry-style tracing and performance monitoring, spanning world events and every flow, gated by the log level so it costs nothing when off.
Every commit runs the full pipeline locally and in CI, with luacheck, a custom STALKER selene build, and a load test on engine stubs.
Rule layers then check Lua practice, engine truth, conventions, contracts, release, security, and docs.
Every rate, threshold, and toggle is exposed through MCM or LTX with nothing left hard-coded, and it writes no engine values, keeping its state within engine bounds so a save can never corrupt.
It runs on one xlibs rulebook shared across the whole mod family, the same protection, distances, faction logic, and combat reads in every mod.
It depends on no other mod, not even the author's own, and needs only X-Ray and xlibs beneath it.
See the Health and JitProfiler links up top for every test and smoke result, and the mod's real CPU and allocation cost.

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

Tags: alife, combat-ai, mutant-ai, mutants, npc, goap, tactical, realistic, emergent, engine-native, self-preservation, accuracy, stealth, perception, cover, faction, loot, ammo, performance, save-safe, reverse-engineering
