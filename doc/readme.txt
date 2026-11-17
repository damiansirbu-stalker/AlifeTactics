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
An artefact gives its carrier one combat edge, and the air warps around him.
Mutants decide on Intelligence by map danger and Personality by species, and Wild varies each one.
They exploit your vulnerabilities and your bad decisions with new leaps, claws and rear strikes.
Fixes to Vanilla make the unfinished danger reactions, corpse search, targeting and healing work.
Each system has its MCM page, and the design behind them reaches much further.

Combat

Combat is how a stalker moves in a fight, and his rank and faction shape each decision.
In vanilla he re-plans every moment, and a fight turns into strafing and cover-hopping.

General:
Each decision takes one Intelligence roll, and a veteran reads the moment right more often.
A hurt Duty trooper falls back firing, a hurt bandit runs for home, and a Monolith fighter holds.

Maneuvers:
A maneuver takes a stalker over from X-Ray's planner for one move the engine can't make or never decides on, and it hands him back once the move is done.
  Counterflank: a stalker fighting someone far off turns on a player who walks up on him.
  Reload Cover: caught reloading in an enemy's sights, he finishes the reload on his way to cover.
  Flee: the last man of a beaten squad may sling his weapon and run for a friendly base.
  Retreat: hurt and exposed, he falls back to cover still firing.
  Kite: an enemy inside his gun's minimum range makes him back off down a clear lane, still firing.
  Pickoff: outranging an enemy who isn't pressing him, he plants and picks him off with single shots.

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

AlifePlus fills the Zone with a living economy of trade and loot, and AlifeTactics makes everything a stalker has gathered count once the shooting begins.
Mechanics are the game rules behind a stalker's kit, from his medkits and the jams in his gun to the ammunition he loads and the artefacts in his pockets.
In vanilla many of these rules don't run for NPCs, and AlifeTactics makes them work.

Healing:
A hurt stalker heals with the medkits and bandages he carries, and pauses while an enemy is on him.
In vanilla the healing script reads empty item lists, and a bleeding man died with full pockets.
  Heal Rate: how fast a medkit brings him back.
  Healing Charge: with empty pockets, some stalkers of every rank still carry one spare heal.
  Animations: out of combat a hurt man limps, and a healing one plays the medkit or bandage gesture.

Jamming:
In X-Ray every NPC rolls a fake jam from the ammo he spends, and that per-rank roll has nothing to do with the real condition of the weapon in his hands.
Even a full-condition rifle chokes every few rounds and he reloads a half-full magazine. AlifeTactics removes the fake jam, and a stalker reloads only when his magazine is spent.

Ammo:
A veteran loads the armour-piercing rounds he carries, with their real ballistics, and the ammunition he has scavenged decides how dangerous he is to you.
Once his own supply runs out, he goes back to the standard rounds every stalker fires.

Attachments:
A stalker fits the scopes, silencers and grenade launchers he carries onto his own guns once the fight is over and his hands are free, at the next quiet moment.
With a scope mounted he shoots tighter once he stops to aim.
Your companion mounts what he loots, and whatever you hand him stays exactly as you gave it.

Gear:
An NPC has no use for most artefact properties, like carry weight or stamina. AlifeTactics translates every artefact the same way, by its class and tier.
The anomaly class picks one combat edge and the tier sets its strength. Only the strongest counts, and a cap keeps any one artefact from deciding a fight.
  Damage Resistance: gravity artefacts and armour plates cut the damage he takes.
  Outgoing Damage: electric and quest artefacts make his shots hit harder.
  Aim Steadiness: thermal artefacts tighten his grouping.
  Health Regeneration: chemical artefacts heal him slowly over time.
  Artefact Distortion: the air warps around a carrier, and killing him gets you the artefact.
  Binocular Spotters: binoculars let him see farther by day.
  Night-vision at Night: a night-vision lamp lets him see farther after dark.

Mutants

Vanilla mutants run an engine state machine that shuffles their moves at random.
They charge straight at you, and anyone on a rock or strafing in time beats them.
AlifeTactics gives mutants the stalkers' decision model, the same Intelligence and Personality.
It answers a vulnerable enemy with moves its own body carries.

General:
Every mutant reads when its enemy, you or a stalker, is reloading, empty or turned away. It also reads where that enemy stands and how he moves around it.
  Intelligence: whether it answers with the right move, rising with the danger of the map.
  Personality: each species leans toward its own moves. Each individual rolls its set once as it enters the fight, and two chimeras from one pack fight differently.
  Wild: how much a mutant's numbers vary, from every spawn the same to each one different.
A pack takes turns on you, and its moves come one after another.

Species:
Each species has its own page with the moves its body carries, sorted into three groups.

New Moves:
Completely new moves, built on dead moves the engine carries and leaves unused in vanilla.
  High Leap (chimera, lurker, cat, dog, snork, pseudodog, psy-dog): a mutant below a perched shooter crouches and springs up after him, and a rock stops being a safe place to shoot from.
  Punishing Leap (chimera, lurker, cat, dog, snork, pseudodog, psy-dog): if you aim at it, reload or turn your back, it pounces at once, with no wind-up at all.
  Hunting and Feral Leap (chimera, lurker, cat, dog, snork, pseudodog, psy-dog): it steers in the air after you as you run, and only Hunting crouches first.
  Catching and Cutoff Leap (chimera, lurker, cat, dog, snork, pseudodog, psy-dog): it leaps at the spot where you will be, and only Catching crouches before it jumps.
  Closing Leap (bloodsucker, psysucker): it closes the gap with a leap of its own, then claws.
  Claw (chimera, lurker): a pounce at point-blank range.
  Backstrike (chimera, lurker, burer, controller): a spin strike at an enemy circling behind it.
  Flee (dog, boar, snork, pseudodog, psy-dog, pseudogiant, cat, flesh, fracture, karlik, tushkano): a dying mutant breaks off when it can get away, and a cornered one fights on.

Managed Moves:
The creature's own attacks, which AlifeTactics fires at the moment the fight calls for them.
  Gravity Throw (burer): it answers your reload or your turned back with its gravity throw.
  Cloak (bloodsucker, psysucker): a second cloak, on top of the invisibility vanilla times on its own. Aim at it or hit it while it closes in, and it vanishes for a beat.

Influenced Moves:
The creature's own abilities, whose numbers Wild spreads between individuals.
The burer's gravity and telekinesis, the controller's psi attack and stamina, and the poltergeist's detection all differ between individuals of a kind.

Fixes to Vanilla:
AlifeTactics repairs bugs across many vanilla systems, each listed on the MCM Fixes tab: the danger scheme, the corpse search, target choice, cover posture and healing.
Behaviours that were unfinished or not working now work.
Stalkers heal with the medkits and bandages they carry, search every corpse they find, and dodge a grenade at the full distance the game intended for it.
A stalker holds fire only for a friend standing between him and his target, and campers and mounted gunners shoot again when other stalkers stand near them.
Stalkers also react to the right threat. One who is shot knows who shot him and turns on him, answers the newest attacker, and stops healing to fight an enemy who appears.
Animation bugs are fixed. A stalker no longer slides across the ground when he is hit or while he bandages, stops limping when a fight starts, and leaves the combat stance after it.
Stability and speed improve too. Vanilla's crashes in the danger checks and the corpse search are fixed, a dead body stops spamming engine errors, and the danger check parses its config once.

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
