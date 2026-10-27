# Manifesto

AlifeTactics makes the Zone's enemies fight for real.

Everyone in the Zone wants to live.
A stalker caught reloading runs for cover.
A hurt bandit breaks contact while a Duty trooper holds the line.
A dog that cannot reach you gives ground and comes again from another angle.
Every combatant reads the fight it is in and acts in its own favour, and no one stages it for the player.

GSC built this intelligence first and cut it later.
Its AI grew so sharp that stalkers won the firefights and finished the quests before the player could, so the studio reined it in to leave the player room to win [9].
AlifeTactics rebuilds the combat mind GSC designed and shelved.

> "The paradigm of Stalker at the time was that we don't really care about the player. We have this living world."
> Andrii Verpakhovskyi, original game designer [9]

---

## Principles

### Everyone wants to live

Self-preservation is the spine of every decision.
A stalker presses an enemy who cannot answer and falls back when he is the one caught weak.
A mutant punishes your opening and breaks off a fight it cannot win.
Both sides fight on even terms, no edge given and none taken.
The Zone is dangerous because the things in it are trying to survive.

### Intelligent decisions

One formula runs every decision, stalker or mutant.
A premise opens it, the read of the actual fight.
Two things then gate it. Intelligence is competence, rising with rank: how sharply he reads the fight. Personality is character: whether his kind takes this action at all.
It fires only when both pass, a sharp read and a willing character. A passed decision commits to its window, and a missed one lets the opening go.
Rank only improves the odds of a good decision, never grants a new one.
A novice and a legend both act, and the legend is right more of the time.

### Engine first

Every behaviour comes from the engine's own parts.
The GOAP planner, the scheme system, the state machine, and the gulag jobs do the work, and AlifeTactics decides when.
Where the engine had no seam to decide at, the seam went in upstream, as per-NPC hooks written for this mod and merged into the official modded exes.
It never replaces a vanilla script file.
A takeover borrows one NPC for a few seconds of one committed move, then hands him back to the engine's combat brain with cover, flanking, and retreat intact.

### Broken engine mechanics

The engine shipped behaviour that never ran.
Stalker rank should scale accuracy and gun handling, but the engine clamps the rank value until every NPC collapses to one setting.
Other behaviour shipped broken and passed for a feature, a weapon in full condition that jams every few rounds and loops a tactical reload.
AlifeTactics turns the clamp back into a real curve and strips the fake behaviour out.

### GSC's unbuilt design

GSC wrote the combat mind out as a list of improvements and never built most of it.
The list names an accuracy parameter, an intelligence parameter, realistic vision, a rear turn, and combat tuned against every monster type.
It rated every creature for how it fights, an attack and defence profile and a courage score, and left the ratings out of play.
AlifeTactics builds each item on the list.
A stalker's accuracy follows his rank, and a mutant fights with the body GSC wrote for it.

> "Improvements to character combat behaviour: introduce an accuracy parameter.
> Introduce an intelligence parameter.
> Make vision more realistic.
> Adjust combat tactics.
> Adjust turning around to the rear.
> Tune combat functions against every monster type.
> Fix the bugs: the grenade's fall point.
> Fix pistol bursts."
> GSC AI design documents [8]

### Faction and species flavour

Every faction fights its own way, across every choice it makes, not only when it is hurt.
Monolith presses and never yields. Duty holds the line and gives ground in order. Bandits finish a downed man and rout when the fight turns. Ecologists fight poorly and pull back early.
Each faction carries its own chance on every action, rout, give ground, press, finish, hold, so one opening draws a different answer from a merc than from a renegade.
This is GSC's own design, its documents rate every character by type and every creature for courage, the attack-and-defence profile left out of the shipped game.
Every mutant fights to its body, and takes each of its moves at its own chance.
A chimera circles to the flank, leaps and claws without pause, and never routs.
A burer hurls debris with its mind, a bloodsucker turns invisible and strikes from cover, a flesh bolts the moment it is hurt.
Each carries the moves its frame allows and the character to spend them, the mutant twin of a faction's doctrine.

---

## Systems

Every system has its own MCM page, in this order. Each reads the live fight and answers it with what the combatant already is, his rank, his weapon, his faction, his body.

### Combat

#### Maneuvers

A maneuver borrows one stalker for a single committed move and hands him back to the engine.
Counterflank snaps him onto a hostile player who reached contact range while he fought someone far off.
Reload Cover sends a stalker caught reloading in a watcher's line to cover that hides him, his weapon loading on the way.
Kite walks a stalker out of an enemy who closed inside his weapon's reach, still firing.
Pickoff plants a stalker who outranges his enemy and takes him with single shots.
Retreat pulls a hurt stalker back to cover under pressure.
Flee routs a skittish faction to a distant friendly base.
The decisions read as human. Nobody turns his back on a shooter, and no one crowds onto cover another man holds.
GSC named this work in one line of the design, adjust combat tactics.
It never built the tactics themselves.
AlifeTactics runs them on the engine's own GOAP planner, borrowing one NPC through hooks added to the modded exes.
The named moves are our design, the fight-read behind them is GSC's.

#### Commitment

Vanilla stalkers re-plan every moment, so a flicker of lost sight makes one drop a good decision and pick again.
Commitment holds a stalker to a decision while it still works and lets him switch the moment it stops.
When his enemy is caught reloading or hurt, he keeps firing through the window and holds his spot.
The engine re-plans the fight every moment, and Commitment damps the wasted re-planning.
It holds through the combat-action-switch and the cover-repick, decision points opened in the modded exes so a working plan survives.

#### Conduct

Conduct sharpens the choices the engine already makes, with no takeover.
A rifleman crouches to steady a clear shot and stands to fire over cover a crouch would hide behind.
A submachine gunner takes closer cover to keep his fire in range, and a sniper holds extra distance.
The cover map and the posture states are the engine's own, and Conduct only chooses better among them.

#### Push and Pull

Push presses a stalker whose enemy cannot answer, thickening his fire and moving him to closer cover.
Pull sends a stalker back when he is the one caught reloading or hurt, his next cover farther out until he recovers.
Both read an advantage the engine tracks and never acts on.
One presses a beaten enemy, the other yields when beaten.

### Effectiveness

#### Accuracy

Accuracy gives each rank a real dispersion curve, because the engine clamps every NPC to one setting on Anomaly's rank values.
A second curve covers fire on the move, and rank buys down the movement penalty.
GSC's design opens with introduce an accuracy parameter, and the engine clamped the rank value until the curve it built collapsed to one setting.

#### Crossfire

Crossfire stops allies cutting each other down in a shared line of fire, keyed on their real relation.
A soured pair still trades fire, and your own shots never count against you.
Crossfire has no design-doc line behind it.
It repairs the engine's friendly-fire handling, keyed on the real relation so hostile pairs still trade fire.

#### Reaction

Reaction sets how fast a stalker swings onto a target and how tightly he holds a strafing one.
It leads a moving target by the round's real flight time, true at high rank and over-led at low.
A player-bias dial tunes how hard NPCs favour you over other combatants, down to treating you like anyone else.
GSC's design calls for an intelligence parameter and realistic vision.
Reaction sets the aim swing and the target lead through per-NPC aim controls added to the modded exes.

#### Discipline

Discipline scales burst length and cadence by rank, inside the engine's own per-weapon bands.
GSC's design lists fix pistol bursts among the combat bugs, and Discipline scales the burst through the engine's own fire-queue control.

### Perception

#### Sound

A hostile stalker reacts to gunfire he hears but cannot see, facing the shots and moving to cover once he sees the shooter.
He hears you move, each footstep carrying by stance, surface, and weather, and a crouch stays silent.
A sound is never a confirmed enemy, so he investigates at a walk.
Stealth is untouched, because Sound never reads vision.
GSC's monster design gives the dogs sharp hearing, and Sound answers the engine hear callback the vanilla AI ignores.
The carry model is ours, each footstep scaled by how a stalker moves and what he crosses.

#### Vision

Vision gives each rank an acquisition speed and a view range, every tier at or above the engine baseline.
GSC's design asks to make vision more realistic.
Vision scales the acquisition rate and the view range per rank on the engine's own visual memory, never below the baseline.

#### Danger

A stalker sniped from across the map now ducks and seeks cover where vanilla left him standing.
A hit victim turns on the man who shot him, over any distant enemy he can see.
GSC's design lists the grenade fall point among the combat bugs to fix.
Danger repairs that read and the rest of the xr_danger scheme, and routes a hit back to its shooter through the engine's danger-in-direction evaluator.

### Mechanics

#### Healing

A wounded NPC heals with the medkits and bandages he carries, pausing under fire, with limp and heal animations.
GSC's offline loop has the hurt stalker use a medkit when his health runs low.
The eat-medkit state is the engine's own, driven off the items the NPC actually carries.

#### Jamming

A weapon in full condition no longer chokes on a fake misfire.
This fix has no design-doc line behind it.
It strips a script-injected misfire the modded exes roll onto full-condition weapons, which passed for a feature.

#### Ammo

A veteran fires armour-piercing rounds from his own inventory and spends them.
A stalker is as dangerous as the ammunition he scavenged.
GSC's trader design has stalkers buy ammo and equipment and carry it.
Ammo spends what a stalker holds, and the armour-piercing decay is our extension of that supply.

#### Gear

An artefact grants one combat edge by its class, capped so it tilts a fight and never decides it.
The carrier warps the air around him, a real and huntable drop.
Binoculars and night vision widen what he sees.
GSC's combat tables rate equipment for effectiveness, and the monster design gives artefacts their field effects.
Gear reads each item's own data for one capped edge, and the advantages are our design on that base.

### Mutants

AlifeTactics gives every mutant the decision model the stalkers run, reading your reload, your aim, your perch, and your strafe, then answering the opening each one invites.
Perch on high ground and the jumpers come up after you.
Catch yourself reloading or turn your back near a burer and its gravity throw answers.
The engine still drives every mutant.
AlifeTactics scales its stats and commands the moves. The engine runs them itself.
This is the strongest-grounded system in the mod.
Every move traces to a line of GSC's monster design, the burer's gravity field, the fracture's strike at a turned back, the bloodsucker's night hunt, the chimera's flank.

#### Power

Power grows with the map's danger, so a dog in Cordon barely changes while a chimera deep in the Zone is far deadlier.

#### Wild

Wild varies each ability value inside the species' own envelope, so no two spawns read alike.

#### Moves

The leap family reads your aim.
Anything short of a dead aim gets the telegraphed Leap you can dodge.
Aim right at it and it drops the wind-up and lunges.
Strafe and it steers in the air to track you.
A burer answers a reloading or turned-away player with its Gravity throw.
Charge, Backstrike, Claw, Cloak, Retreat, and the hit-and-run Breakoff round out the set, each the creature's own.

---

## Sources

- [8] GSC Game World internal AI design documents, pre-release, circa 2002-2006.
  Ai.doc has the combat-improvements list, monstry.doc the creatures and their moves, Killing_NPC.doc the offline heal loop, traders.doc the trade economy.
  The EFC tables hold the combat ratings and probability data: creature_courage and WeaponSuccessProbability.
  Leaked in the "Koan's Gift" package, published on the Gameru.net forum, and catalogued on the STALKER wiki.
- [9] Andrii Verpakhovskyi (original S.T.A.L.K.E.R. game designer), Eurogamer, reported by VideoGamer
  https://www.videogamer.com/news/original-stalker-a-life-so-advanced-that-players-were-constantly-outsmarted-by-ai/
