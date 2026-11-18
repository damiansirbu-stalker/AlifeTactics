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

Every release is stable, and a new one can add whole systems or move the settings that drive them.
After every update, open Development in the mod's MCM and reset every setting to its default.

Everyone wants to live.

AlifeTactics is a complete combat system for S.T.A.L.K.E.R. Anomaly, for stalkers and mutants alike, from what they hear and see to how they decide, move, shoot and heal.
In vanilla the enemies all fight the same way, because X-Ray shuffles its combat moves at random.
AlifeTactics adds the decision. Each fighter reads the fight in front of him and picks a move, through the engine's combat systems and hooks built for AlifeTactics and merged into the demonized exes.
It also brings back moves and mechanics that GSC built and the shipped game never ran.

Summary

A stalker decides on Intelligence, which rises with his rank, and on Personality, his faction's character, the two traits that GSC's design documents name.
Combat makes stalkers reload behind cover, fall back to cover when hurt, turn on a player who closes in, pick off an enemy they outrange and press an enemy who can't answer.
Effectiveness makes rank show in every shot. A veteran groups tighter, holds a strafer and leads a runner, and allies stop killing each other in a crossfire.
Perception lets stalkers hear your steps and your reload before they see you, take cover when sniped from far off, turn on the man who shot them and notice a missed knife swing.
Mechanics lets stalkers heal with the medkits they carry, fire the armour-piercing rounds they scavenged, mount the scopes they loot and jam only from real wear.
An artefact feeds its carrier one effect channel, and the air warps around him.
Mutants decide on Intelligence by map danger and Personality by species, and Wild varies each one.
They exploit your vulnerabilities and your bad decisions with new leaps, claws and rear strikes.
Fixes make the unfinished danger reactions, corpse search, targeting and healing work.
Each system has its MCM page, and the design behind them reaches much further.

Combat

Combat is how a stalker moves in a fight, and his rank and faction shape each decision.
In vanilla he re-plans every moment, and a fight turns into strafing and cover-hopping.

General:
Each decision takes one Intelligence roll, and a veteran reads the moment right more often than a rookie, though even a legend gets some of them wrong.
A hurt Duty trooper falls back firing, a hurt bandit runs for home, and a Monolith fighter holds.
  Hit reaction hold: a stalker hit while standing no longer slides across the ground.

Maneuvers:
A maneuver takes a stalker over from X-Ray's planner for one move the engine can't make or never decides on, and it hands him back once the move is done.
However many stalkers fight you, only a few of them make a move on you at once.
  Counterflank: a stalker fighting someone far off turns on a player who walks up on him.
  Reload Cover: caught reloading in an enemy's sights, he finishes the reload on his way to cover.
  Flee: the last man of a beaten squad may sling his weapon and run for a distant friendly base, once the shooting pauses and you are not right on top of him.
  Retreat: hurt and exposed, he falls back to cover still firing.
  Kite: an enemy inside his gun's minimum range makes him back off down a clear lane, still firing.
  Pickoff: outranging an enemy who isn't pressing him, he plants and picks him off with single shots, and he breaks off the moment you aim at him or close in.

Commitment:
Commitment holds a stalker to a plan that works, where vanilla drops it at the smallest change, a teammate crossing his line or a better cover spot a step away.
  Cover pinning: with a clear shot he keeps his cover and stops hopping to the next spot.
  Liquidate vulnerable targets: he stays in his spot and keeps firing at a vulnerable enemy.

Conduct:
Conduct improves choices the engine already makes, with no takeover. Military factions mostly keep its discipline, and bandits and other scavengers mostly don't.
  Cover posture: a trained rifleman crouches for a clear shot and stands to fire over a low wall.
  Weapon spacing: an SMG gunner takes closer cover, and a skilled sniper keeps his distance.

Behaviors:
Push and Pull read who is caught weak in a fight, you or another stalker.
  Push vulnerable targets: his attackers press an enemy who can't answer with heavier bursts, and once they clearly hold the upper hand they move to closer cover.
  Push on reload and empty mag: a reload or an empty magazine starts the press.
  Push weakened targets: a badly hurt enemy draws the press, and healthy attackers close in.
  Pull back while vulnerable: caught weak against a strong enemy, he gives ground until he recovers.

Effectiveness

Effectiveness is how well a stalker performs in combat.
In vanilla these skills are fixed numbers, the same for every NPC whatever his rank, and editing a config only swaps them for other fixed numbers that every NPC shares.
AlifeTactics revives the rank curve for accuracy and adds one to every other skill.
Each stalker fights at the level of his own rank.
A mixed squad will hit you unevenly, the rookie wide and the veteran tight.
It all runs on X-Ray's own aim and fire controls.

Accuracy:
His accuracy follows his rank, and the gun keeps its own spread.
A worn AK sprays in anyone's hands.
  Dispersion: a rookie shoots as in vanilla, and each rank above him groups tighter.
  Moving Fire: on the move, a veteran keeps more of his accuracy than a rookie.

Crossfire:
Stalkers stop gunning down the friends and neutrals who step into their line of fire.
Hostile pairs and your own bullets still do full damage.

Reaction:
Reaction is how fast a stalker's gun can follow you.
It closes the strafe and sprint tricks that beat vanilla aim.
  Tracking Speed: a veteran's barrel follows you faster, and when you strafe it stays on you.
  Target Lead: he aims ahead of a runner by his bullet's real flight time. A legend will hit you on the run, while a rookie leads too far and misses in front of you.

Discipline:
Discipline sets how long a stalker's bursts run and how soon the next one comes.
  Fire Discipline: a veteran fires shorter, more frequent bursts that cut the wide last rounds, and every rank hits at least as often per minute as in vanilla.

Range (planned):
In vanilla, a stalker stops treating another stalker as an enemy past a fixed distance, whatever gun he carries, and the limit drops further at night and in rain.
This page will let a long-range stalker answer and open fire as far as his weapon really reaches.

Perception

Perception is what a stalker notices around him and how he reacts to it.
In vanilla a stalker who has not seen you lately drops most of what he hears from you.
AlifeTactics makes the whole soundscape matter, your gunfire, your reloads and the calls of mutants nearby, and it adds new sound events for your footsteps and landings.
The system stays fair, because a sound is never a confirmed enemy, a stalker who hears you only walks over to check the spot, and a careful crouch keeps you silent.
His eyes follow his rank, and darkness and cover still hide you.
It all runs on X-Ray's own senses and on the danger scheme your modpack ships.

Sound:
A hostile stalker may hear you before he ever sees you.
Hearing and sight are separate senses, and a sound only sends him to check the spot.
X-Ray then runs cover and search on what he heard, and once his squad is in a real fight he stops checking noises and keeps watch on the spot from where he stands.
  Neutral Gunfire Alert: a stray bullet of yours that hits near a neutral or a friend puts him on alert, and he moves to cover facing the impact without turning hostile.
  Movement Noise: your steps carry by stance, surface and weather. A walk turns a standing hostile toward you, and a sprint or a hard landing sends him over to check.
  Handling Noise: a reload or an empty click brings him over, and using an item turns him toward you.
  Creature Awareness: a standing stalker glances at a mutant he hears nearby and keeps his post.

Vision:
Vision is how fast a stalker spots a target, and how far out he starts to notice one.
  Vision Speed: a veteran spots you sooner and from farther out, while a rookie sees as in vanilla, and darkness, light and cover still hide you as your setup decides.

Danger:
Danger is how a stalker reacts to a hit, a body or a grenade.
AlifeTactics fixes its broken reactions and adds new ones.
  Distant Hits: a stalker sniped from far off turns and seeks cover.
  Attempted Murder: if your knife swing misses the stalker in front of you, he gets up and faces you with his weapon raised, and a hostile who then sees you fights.

Mechanics

Mechanics are the game rules behind a stalker's kit.
In vanilla many of these rules never run for NPCs, and AlifeTactics makes them work.
AlifePlus fills the Zone with a living economy of trade and loot, and with AlifeTactics everything a stalker has gathered counts once the shooting starts.
Every item in his kit is real and can run out.
A well-stocked veteran is more dangerous than a scavenger with empty pockets.

Healing:
A hurt stalker heals with the medkits and bandages he carries.
He stops healing while an enemy is on him.
Some stalkers of every rank still carry one spare heal when their pockets are empty.
  Limping animation: a badly hurt stalker near you limps out of combat and stops in a fight.
  Healing animation: a stalker who starts a heal standing still plays the medkit or bandage gesture.

Jamming:
On the modded exes an NPC's gun jams from a fake roll, whatever its condition.
  Suppress NPC jams: his gun jams only from real wear, and your own jams stay as they are.

Ammo:
A veteran fights with the rounds he really carries.
  NPC armor-piercing ammo: a veteran loads the armour-piercing rounds he carries, and after each fight he may use up a box of them. Once his supply runs out, he goes back to standard rounds.

Attachments:
A stalker fits the scopes, silencers and grenade launchers he carries onto his own pistol or rifle.
He waits for a quiet moment out of the fight.
  NPC attachment mounting: a looted scope shows on his rifle, and he shoots tighter with it once he stops to aim. Whatever you hand your companion stays as you gave it.

Gear:
In vanilla an artefact works only on the player's belt, and an NPC gets nothing from it.
AlifeTactics translates each artefact into one effect channel an NPC can use, chosen by its anomaly class, and the artefact's tier sets how strong it gets.
The system stays fair, because only the strongest artefact counts on a channel, a cap keeps it from deciding a fight, and every carrier gives himself away.
  Damage resistance: gravity artefacts and armour plates cut the damage he takes.
  Outgoing damage: electric and quest artefacts make his shots hit harder.
  Aim steadiness: thermal artefacts tighten his grouping.
  Health regeneration: chemical artefacts heal him slowly over time.
  Artefact distortion: the air warps around a carrier, and killing him gets you the artefact.
  Binocular spotters: binoculars let him see farther by day.
  Night-vision at night: a night-vision lamp lets him see farther after dark.

Effects (planned)

This tab will add combat feedback you feel, starting with concussion.
A close blast will leave your ears ringing and your sight blurred.

Mutants

Mutant combat is how a creature of the Zone chooses its moves against you or a stalker.
In vanilla X-Ray drives each mutant with a state machine that only charges straight in at melee range, and a player standing on a rock shoots most of them as they come.
AlifeTactics gives mutants the stalkers' decision model and new moves that run through X-Ray's jump and attack controls and play the creature's real animations.

General:
Every mutant watches its enemy, you or a stalker, for an opening such as a weapon aimed at it, a reload or a turned back, a ledge above it, or a man on the run.
  Intelligence: whether it answers an opening with the right move, rising with the danger of the map. If its read fails, it lets the opening go, and a badly hurt one may panic.
  Personality: each species leans toward some moves, and each mutant rolls its set once. Two chimeras of one pack fight differently, and a flesh runs far more often than a pseudodog.
  Wild: how much mutants of one kind differ, in the timing of their moves, the force of their leaps and the reach of their abilities, from one creature to the next.
A mutant spaces its moves apart, and once one mutant's leap, strike or throw hits you, every mutant holds its next one on you for a few seconds while its natural attacks go on.

Species:
Every species has a page: Chimera, Bloodsucker, Burer, Controller, Poltergeist, Dog, Boar, Snork, Pseudodog, Pseudogigant, Cat, Flesh, Fracture, Karlik and Tushkano.
Each subspecies shares the page of its parent species, the lurker with the chimera, the psysucker with the bloodsucker and the psy dog with the pseudodog.

New Moves:
These are moves vanilla never makes for the species, each built on a part of the creature X-Ray already carries, its jump, an unused rear strike or its run-away state.
The chimera, lurker, cat, dog, snork, pseudodog and psy dog share every leap but the Closing leap.
A dog favours the straight leaps, a snork the ones that turn and a chimera the sudden ones.
A mutant leaps only at an enemy it has seen or sensed lately, and close in only up at a ledge.
  High leap: an enemy standing on a rock or a ledge above it draws a jump up after him, and for you or any stalker a rock stops being a safe place to shoot from.
  Punishing leap: it springs without a crouch the moment you aim at it or can't fire back.
  Hunting leap and Feral leap: it turns with a moving enemy through the leap, no harder than a vanilla creature turns, and only Hunting crouches first, a warning you can see.
  Catching leap and Cutoff leap: it springs at the path of a moving enemy, a short step ahead of him, a smarter mutant judges that step better, and only Catching crouches first.
  Closing leap (bloodsucker, psysucker): it closes the gap to an enemy on a ledge, or to a player who aims at it from mid range, and then claws him on the ground.
  Claw (chimera, lurker): a point-blank pounce, the close strike a vanilla chimera lacks. Both creatures leap with no crouch, and their leaps come without warning.
  Backstrike (chimera, lurker, burer, controller): a spin strike at an enemy behind it.
  Flee (dog, boar, snork, pseudodog, psy dog, pseudogigant, cat, flesh, fracture, karlik, tushkano): when its read fails a badly hurt mutant may run, unless it faces you or you are close.

Managed Moves:
Attacks the creature already has, which AlifeTactics fires at the moment the fight calls for them.
  Gravity attack (burer): it throws when you look away from it or can't fire back, at a moment it needs no shield, and never straight after another throw.
  Cloak (bloodsucker, psysucker): if you aim at it or hit it while it closes in, it vanishes for a moment on top of the cloak vanilla times, but never once it is close enough to strike.

Influenced Moves:
Abilities the creature already has, whose numbers Wild spreads between individuals of one kind.
  Gravity numbers and Telekinesis (burer), Psi tube, Stamina and Aura (controller), and Detection (poltergeist) all differ from one creature of a kind to the next.

Fixes

Vanilla leaves many combat behaviours broken, and AlifeTactics carries more than 30 fixes for them.
Each fix is listed on the Fixes tab, and these are the ones you notice.
  Danger scheme: a sniped stalker takes cover, knows his shooter, answers his newest attacker, dodges grenades at full radius, holds fire only for a friend in his line and relaxes after the fight.
  Corpse investigation: stalkers search every corpse they find, every time.
  Target selection: a hit stalker turns on his shooter, and NPCs weigh you like any other enemy.
  Accuracy: rank sets each stalker's spread, where vanilla gave all of them one.
  Combat: stalkers shoot over the low cover they duck behind, and Monolith fighters flip stance less.
  Movement and animation: a hit stalker stops sliding, and no maneuver starts mid-animation.
  Healing: stalkers use their kits, stop to fight, limp only out of combat and bandage standing.
  Mutants: a chimera or lurker rests after each jump, where vanilla let it jump again at once.
The danger checks and the corpse search no longer crash, a dead body no longer spams engine errors, and the danger check reads its config once and keeps it.

Requirements:
Anomaly 1.5.3
Modded exes: themrdemonized 20250908 or newer, or AOEngine v0.5.2 or newer. The full feature set needs the latest demonized build. A feature needing a newer one is inactive or reduced on older exes.
xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Compatibility:
Depends only on xlibs. Install and uninstall mid-save work. Tested: Anomaly 1.5.3, GAMMA, EFP, Zona, Forgotten Zone.
Drop:
- G.A.M.M.A. AI Rework - it replaces the combat AI AlifeTactics works through with old scripts.
  Much of it is copied or dead code, its far stalkers fire without line of sight or stand idle, and its development builds overwrite stalker movement every update.
- RE:DONE Combat AI - it overwrites 17 vanilla scripts whole, moves stalkers onto scripted combat, and sets every stalker's aim from the rank of one stalker.
  Its danger and healing schemes clash with AlifeTactics, and both mods make stalkers flee.
- G.A.M.M.A. No logs, Log spam remover - their _g.script copies drop the hook stalker aim runs on.
  Accuracy, Target Lead and thermal artefacts stop working. No logs also drops the knife hook Attempted murder needs, and functions other scripts call as they load.
Avoid:
- NPC self-heal replacements, such as NPC Limping and Healing (Vodoxleb) - they switch off the medkit scheme Healing runs on and heal stalkers in its place.
  Their limp fights the Healing limp on the same stalker.
- NPC Weapon Jamming, and any mod that adds NPC jams - it jams a gun on a roll, not from wear, which undoes the Jamming system, and a rookie jams every few shots.
- G.A.M.M.A. No NPC Friendly Fire - Crossfire does its job already, and on every hit it turns two feuding stalkers of friendly or neutral factions back to peace.
- G.A.M.M.A. NPCs Faster Reactions - it lengthens stalker sight, and the Vision range and binoculars stretch it further until stalkers spot you across open ground.
Interferes:
- Oleh's Weapons Sounds Tweaks and Fixes (GAMMA) - most of its rifle, SMG and sniper shots carry 5m to other NPCs, while its silenced rifle shots carry 75-120m.
  Stalkers barely hear each other's gunfire, and a mounted silencer makes a stalker louder.
Coexists:
- Ballistics Overhaul (GBOOBS) - it applies your hit damage itself, and AlifeTactics restores the artefact and plate resistance a stalker carries against your shots.
- Useful Idiots - your companions follow its orders, since AlifeTactics leaves companions alone by default, and the friendly-fire fixes of both mods run together.
- Vision and accuracy configs, such as Worse NPC Vision and Accuracy - they set the baseline sight and spread, and Vision and Accuracy scale that baseline by rank.
Everything else coexists, as long as it extends X-Ray and Anomaly and never overrides them.

Not AlifeTactics (base game or your setup):
- Stalkers sliding when shot on the move - the hit animation plays while he keeps walking or running, as in unmodded Anomaly. Hit reaction hold covers a standing or staggered stalker.
- Stalkers ignoring darkness at dusk and dawn - Stealth Overhaul's visibility script uses its night light curve only from 21:00 to 04:00, and its day curve the rest of the time.
- Stalkers seeing you from far off - sight range comes from your creature config, and AlifeTactics adds up to 15 percent by rank and 50 percent with binoculars by day.
- Stalkers reacting to danger from far off - vanilla's danger distance table holds its duration values. A shot or a ricochet then counts as danger out to 300m.

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
