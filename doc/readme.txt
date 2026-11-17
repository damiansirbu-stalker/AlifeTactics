Version: 1.2.2-snapshot (xlibs 1.9.2, demonized latest)
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

AlifeTactics is a complete combat system for S.T.A.L.K.E.R. Anomaly. It covers the whole fight for stalkers and mutants, from what they hear and see to how they decide, move, shoot and heal.

Vanilla enemies all fight the same way, because X-Ray picks their moves with no logic behind them. AlifeTactics gives them the brain the engine never had.
It is a light, reactive layer that works with X-Ray, through its own hooks in the engine itself.
The core of every decision is Intelligence and Personality, from GSC's own design documents.
Intelligence is the difficulty and rises with rank and map level. Personality is the character of each faction and species, how aggressive or cowardly, and the moves it favours.
It also revives what GSC built and shelved, moves and mechanics the shipped game never ran.

- Combat: stalkers reload behind cover, fall back when hurt, turn on a player who closes in, and pick off an outranged enemy. They hold a plan while it works and press an enemy who can't answer.
- Effectiveness: rank finally shows in aim, tracking, target lead and fire discipline, where the engine clamped every NPC to one setting. Allies stop killing each other in a crossfire.
- Perception: stalkers hear your steps and your reload before they see you, duck when sniped, turn on the man who hit them, and notice a missed knife swing.
- Mechanics: they heal with their own medkits, veterans fire their own armour-piercing rounds, they mount the scopes they loot, and full-condition guns stop jamming.
- Gear: an artefact gives its carrier a real combat edge by its class, binoculars and night vision widen his sight, and a carrier warps the air around him.
- Mutants: they think like stalkers, with their own Intelligence, Personality and Wild variation.
  They punish your reload, your strafe and your perch with leaps, gravity throws, claws and backstrikes from their own bodies, moves vanilla never fires.
- Fixes: vanilla AI bugs repaired in danger, corpse search, targeting, cover fire and healing.
- Engine-native and fast: it runs on X-Ray's own planner and the Anomaly schemes, replaces no vanilla file, and does no per-frame work, whatever the NPC count.

Every system below has its own MCM page, where it is toggled and tuned. The design behind them reaches much further, and each release builds more of it.

Combat

Every move below is a decision. Intelligence decides whether a stalker reads the moment, and his faction's Personality decides whether his kind takes that move.
A hurt Duty trooper falls back to cover still firing, a hurt bandit is quick to break and run for home, and a Monolith fighter holds his ground to the end.

Maneuvers:
A maneuver takes a stalker over from X-Ray's GOAP planner for a move the engine can't make, one he never had, one vanilla never decides on, or one that depends on his squad.
Before he moves, he reads the whole fight, every enemy watching him, his wounds, his weapon's range and his squad. The move ends when its problem is solved.
  Counterflank: a stalker fighting a distant enemy turns to the immediate threat when you close in.
  Reload Cover: caught reloading in an enemy's sights, he runs to the nearest cover that hides him from that enemy and finishes the reload on the way there.
  Retreat: hurt and exposed to a shooter in reach, he falls back to farther cover, still firing.
  Flee: badly hurt and the last of his squad, he breaks contact and heads for a friendly base. It is the panic move, and a sharper stalker is less likely to take it.
  Kite: an enemy inside his weapon's minimum range sends him back down a clear lane, still firing.
  Pickoff: outranging an enemy who isn't pressing him, he plants and picks him off with single shots.
Maneuver fire follows his weapon and rank, with bursts that vary inside the weapon's own range.

Commitment:
Vanilla stalkers drop a good plan at the smallest change, when a teammate crosses the line, sight flickers for a moment or a slightly better cover spot appears.
That is the strafing and cover-hopping of every firefight.
Commitment answers the engine's own "switch now?" with no while the plan still works, through hooks built for AlifeTactics and merged into X-Ray. When it stops working, he switches at once.
  Cover pinning: with a clear shot he keeps his cover and no longer slides to a slightly better spot.
  Liquidate: when his enemy is vulnerable, he holds his spot and fires until the enemy can answer.

Conduct:
Conduct improves choices the engine already makes, with no takeover.
Discipline is personal. Military factions mostly keep it and scavengers mostly don't.
  Cover posture: a trained rifleman crouches for a clear shot and stands to fire over a low wall.
  Weapon spacing: an SMG gunner takes closer cover, and a skilled sniper keeps his distance.

Behaviors:
Push and Pull read who is vulnerable in a fight and work against any enemy, you or another stalker.
  Push: when the target is vulnerable or dying, his attackers press him with heavier fire and move to closer cover once they clearly hold the upper hand.
  Pull: caught weak against a strong enemy, he gives ground to farther cover until he recovers.

Effectiveness

Effectiveness is how well a stalker fights, and his rank should decide it.
The engine has a rank curve for accuracy, but Anomaly's rank values push nearly all NPCs to the same end of it. In vanilla a rookie and a legend shoot alike.
AlifeTactics drives a real curve through the engine's shot callback, and sets each stalker's aim and bursts through hooks built for AlifeTactics and merged into X-Ray.

Accuracy:
A shot's spread is the gun's spread plus the shooter's, and Accuracy scales only the shooter's share. A better gun shoots tighter for a rookie and a legend alike.
  Dispersion: a veteran groups his shots tight, where a rookie sprays wide.
  Moving Fire: a veteran keeps more of his accuracy than a rookie when he has to fire while moving.

Crossfire:
If a stalker walks into a friend's or a neutral's line of fire, the hit doesn't cut him down.
Because it keys on their real relation, a soured pair trades full damage as in vanilla, and the protection never covers the shots that you fire yourself.

Reaction:
Reaction is how a stalker's gun follows a moving target. A rookie tracks as in vanilla, and even a legend stays well short of the game's hardcore aim setting.
  Tracking Speed: a veteran's barrel follows you faster as you move.
  Tracking Lock: if you strafe, a veteran holds you, where a rookie's aim lags and loses you.
  Target Lead: he aims ahead of a runner by the round's real flight time. A legend aims where you will be, and the lower his rank, the farther ahead of you he shoots.

Discipline:
Discipline is how a stalker works the trigger, the length of his bursts and the pause between them.
  Burst Size: a veteran fires shorter bursts and drops the stray rounds at the end of a long one.
  Fire Cadence: a veteran pauses less between bursts to keep his volume of fire up.
The defaults are close to flat and keep the hits per minute of each rank at or above vanilla, and the sliders widen the gap between a rookie and a legend.

Range (planned):
Vanilla's combat-ignore script drops another stalker as an enemy past a fixed distance, whatever gun he carries, and closer still at night and in rain.
This page will let a long-range stalker answer and open fire at his weapon's real reach.

Perception

Perception is what a stalker hears and sees, and how he reacts to it.
Every sound you make now counts. A careful crouch keeps you hidden, a careless sprint gives you away, and in a firefight your noise tells him where to look.
AlifeTactics patches these reactions into whichever danger script your modpack ships.

Sound:
In vanilla, a stalker who hasn't seen you lately drops your sounds, even a reload behind his back.
AlifeTactics lets a hostile stalker hear you, and he answers with the engine's cover and search.
  Neutral Gunfire Alert: a stray bullet of yours alerts a neutral or friend, who won't turn hostile.
  Movement Noise: a hostile hears your steps by stance, surface and weather, and a crouch is silent. A walk turns a standing hostile toward you, and a sprint or a jump landing sends him over to check.
  Handling Noise: your reload or empty click brings him over, and using an item turns him toward you.
  Creature Awareness: a standing stalker glances at a mutant he hears nearby and keeps his post.

Vision:
Vision is how fast a stalker turns a glimpse into a confirmed threat, and how far out he starts to notice one. Both scale with rank on the engine's own visual memory.
  Vision Speed: a veteran confirms you sooner, and a rookie matches your setup's baseline.
  Vision Range: a veteran starts noticing you from farther out, while light, darkness and cover still hide you from him exactly as your own setup decides.

Danger:
Danger is the vanilla scheme that makes a stalker react to a hit, a corpse or a grenade. AlifeTactics repairs its bugs, listed under Fixes, and adds the reactions below.
  Distant Hits: a stalker sniped from far off turns and seeks cover, where vanilla left him standing.
  Attempted Murder: if your knife swing misses the stalker in front of you, he gets up, raises his weapon and faces you, and a hostile who then sees you fights.
  Player Target Pull: vanilla stalkers turn on you before any other enemy. AlifeTactics removes that player bias, and you count like any other fighter in the fight.

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

General:
Observing is free, deciding is hard. Every mutant sees your reload from the first fight, and Intelligence is the chance it answers with the RIGHT move, a base plus a bonus per map rank.
Personality then sets which moves its species favours, a chimera leaping and clawing without pause where a flesh bolts early. The final chance is Intelligence times Personality.
A missed roll lets the opening go. A dull mutant wastes chance after chance while a sharp one punishes each.
Every move you see has a real cause behind it, and a cap holds a pack to waves.
Wild is how much a mutant's own ability numbers vary between spawns, the gravity, telekinesis, aura, tube, stamina, and detection values.
At 0 a kind is identical and learnable. At 100 no two spawns read alike.

Species:
Each species gets its own page carrying only the moves its body has, in three groups. New moves are the attacks the vanilla game never fires for it.
Managed moves are its own attacks AlifeTactics decides when to fire. Influenced moves are its own abilities whose numbers Wild spreads between individuals.
The leap family reads your aim, your movement, and your perch. Punishing leap strikes at once, with no wind-up, when you aim at it, reload, or turn your back.
Hunting leap crouches, then steers mid-air to follow you while you move. Feral leap does the same with no warning.
Catching leap crouches, then leaps at where you will be while you move. Cutoff leap does the same with no warning.
High leap springs up at a shooter above it, with a wind-up. Climbing a rock or a ledge stops being safe. The bloodsucker closes the gap with its own Closing leap, then claws.
Every leaping species except the bloodsucker can take each of the six shared leaps. Its Personality, rolled once per individual, decides which ones it carries.
A chimera rarely telegraphs, a dog jumps straight and open, a snork steers, a cat strikes from your blind side.
Ten seconds pass between any two of a mutant's own leaps, and twenty before it repeats the same one.
Backstrike arms a spin strike on an enemy circling behind, once per individual, on the four species whose models carry the motions. Claw gives the chimera the point-blank pounce vanilla never gave it.
Flee breaks a dying mutant out of the fight when it is far enough to escape. It runs on the inverse of Intelligence: the dull mutant panics, the sharp one presses on. A mutant with no escape fights to the end.
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
Send the traces to my Anomaly Discord thread: https://discord.com/channels/456765861953536020/1512539453791670402
Join the server first through the Anomaly Discord invite: https://discord.gg/c4RuJNs
Or file a defect with the same information on GitHub: https://github.com/damiansirbu-stalker/AlifeTactics/issues
Attach xray.log, the mod log, the engine build, the modlist, and the load order.
For deep technical details and mechanisms, check the architecture docs on GitHub.

Tags: alife, combat-ai, mutant-ai, mutants, npc, goap, tactical, realistic, emergent, engine-native, self-preservation, accuracy, stealth, perception, cover, faction, loot, ammo, performance, save-safe, reverse-engineering
