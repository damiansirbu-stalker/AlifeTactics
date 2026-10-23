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
Intelligence decides it, a base chance plus a bonus per rank.
A passed decision commits to its window, and a missed one lets the opening go.
Rank only improves the odds of a good decision.
A novice and a legend both act, and the legend is right more of the time.

### Engine first

Every behaviour comes from the engine's own parts.
The GOAP planner, the scheme system, the state machine, and the gulag jobs do the work, and AlifeTactics decides when.
Where the engine had no seam to decide at, the seam went in upstream, as per-NPC hooks written for this mod and merged into the official modded exes.
It never replaces a vanilla script file.
A takeover borrows one NPC for a few seconds of one committed move, then hands him back to the engine's combat brain with cover, flanking, and retreat intact.

### Performance is the first invariant

The whole mod costs a handful of timer compares each frame, whatever the number of NPCs, and does its real work only on engine callbacks and scheduled passes.
A flow that cannot hold its budget gets rewritten or dropped.
Performance outranks every feature, and the build measures it on unoptimized single-threaded exes.

### Broken engine mechanics

The engine shipped behaviour that never ran.
Stalker rank should scale accuracy and gun handling, but the engine clamps the rank value until every NPC collapses to one setting.
Other behaviour shipped broken and passed for a feature, a weapon in full condition that jams every few rounds and loops a tactical reload.
AlifeTactics turns the clamp back into a real curve and strips the fake behaviour out.

### GSC's unbuilt design

GSC rated every creature for how it fights and never wired the ratings into play.
The design gave each species an attack and defence profile, a courage score, and a hunger that drives it to hunt or flee.
The engine gave them all one charge and one death.
AlifeTactics gives each creature the decision its design always described, so a mutant fights with the body and the temperament GSC wrote for it.

> "A complex AI system with a life simulation system that enables any monster and NPC to act independently on any game level,"
> "with creatures having a full life cycle where they hunt, feed, take rest, and sleep."
> Oles Shishkovtsov, ModDB interview [3]

### Faction and species flavour

Every faction fights its own way.
Some factions rout when they are hurt.
Others pull back to cover and keep firing.
Monolith does neither and holds its ground.
Every mutant fights to its body.
A chimera circles to the flank, and a burer hurls debris with its mind.
A bloodsucker turns invisible and strikes from cover.
Each fights to the moves its own frame carries.

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

#### Commitment

Vanilla stalkers re-plan every moment, so a flicker of lost sight makes one drop a good decision and pick again.
Commitment holds a stalker to a decision while it still works and lets him switch the moment it stops.
When his enemy is caught reloading or hurt, he keeps firing through the window and holds his spot.

#### Conduct

Conduct sharpens the choices the engine already makes, with no takeover.
A rifleman crouches to steady a clear shot and stands to fire over cover a crouch would hide behind.
A submachine gunner takes closer cover to keep his fire in range, and a sniper holds extra distance.

#### Push and Pull

Push presses a stalker whose enemy cannot answer, thickening his fire and moving him to closer cover.
Pull sends a stalker back when he is the one caught reloading or hurt, his next cover farther out until he recovers.

### Effectiveness

#### Accuracy

Accuracy gives each rank a real dispersion curve, because the engine clamps every NPC to one setting on Anomaly's rank values.
A second curve covers fire on the move, and rank buys down the movement penalty.

#### Crossfire

Crossfire stops allies cutting each other down in a shared line of fire, keyed on their real relation.
A soured pair still trades fire, and your own shots never count against you.

#### Reaction

Reaction sets how fast a stalker swings onto a target and how tightly he holds a strafing one.
It leads a moving target by the round's real flight time, true at high rank and over-led at low.
A player-bias dial tunes how hard NPCs favour you over other combatants, down to treating you like anyone else.

#### Discipline

Discipline scales burst length and cadence by rank, inside the engine's own per-weapon bands.

### Perception

#### Sound

A hostile stalker reacts to gunfire he hears but cannot see, facing the shots and moving to cover once he sees the shooter.
He hears you move, each footstep carrying by stance, surface, and weather, and a crouch stays silent.
A sound is never a confirmed enemy, so he investigates at a walk.
Stealth is untouched, because Sound never reads vision.

#### Vision

Vision gives each rank an acquisition speed and a view range, every tier at or above the engine baseline.

#### Danger

A stalker sniped from across the map now ducks and seeks cover where vanilla left him standing.
A hit victim turns on the man who shot him, over any distant enemy he can see.

### Mechanics

#### Healing

A wounded NPC heals with the medkits and bandages he carries, pausing under fire, with limp and heal animations.

#### Jamming

A weapon in full condition no longer chokes on a fake misfire.

#### Ammo

A veteran fires armour-piercing rounds from his own inventory and spends them.
A stalker is as dangerous as the ammunition he scavenged.

#### Gear

An artefact grants one combat edge by its class, capped so it tilts a fight and never decides it.
The carrier warps the air around him, a real and huntable drop.
Binoculars and night vision widen what he sees.

### Mutants

AlifeTactics gives every mutant the decision model the stalkers run, reading your reload, your aim, your perch, and your strafe, then answering the cheese each one invites.
Perch on a rock and the jumpers climb after you.
Hold your aim on a burer and its gravity throw answers.
A shotgun aimed in range stops a leap, and a shotgunner turned away invites one.
The engine still drives every mutant.
AlifeTactics scales its stats and commands the moves. The engine runs them itself.

#### Power

Power grows with the map's danger, so a dog in Cordon barely changes while a chimera deep in the Zone is far deadlier.

#### Wild

Wild varies each ability value inside the species' own envelope, so no two spawns read alike.

#### Moves

The leap family reads your attention.
A turned back gets the telegraphed Leap.
A watching enemy gets the Jump with no wind-up.
A strafing one gets the version that steers in the air.
A burer answers a held aim or a perched shooter with its Gravity throw.
Charge, Backstrike, Claw, Cloak, and Retreat round out the set, each the creature's own.

---

## Sources

- [3] Oles Shishkovtsov (engine programmer), ModDB
  https://www.moddb.com/features/stalker-interview
- [8] GSC Game World internal AI design documents, pre-release, circa 2002-2006.
  The combat tables rate every creature for attack and defence, a courage score, and a hunger that drives hunting and flight.
  Leaked in the "Koan's Gift" package and catalogued on the STALKER wiki.
- [9] Andrii Verpakhovskyi (original S.T.A.L.K.E.R. game designer), Eurogamer, reported by VideoGamer
  https://www.videogamer.com/news/original-stalker-a-life-so-advanced-that-players-were-constantly-outsmarted-by-ai/
