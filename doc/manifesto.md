# Manifesto

AlifeTactics makes the Zone's enemies fight for real.

Everyone in the Zone wants to live.
A stalker caught reloading runs for cover.
A hurt bandit breaks contact while a Duty trooper holds the line.
A dog that cannot reach you gives ground and comes again from another angle.
Every combatant reads the fight it is in and acts in its own favour, and no one stages it for the player.

GSC built this intelligence first and cut it later.
Its AI grew so sharp that stalkers won the firefights and finished the quests before the player could, so the studio reined it in to leave room to play [9].
AlifeTactics rebuilds the combat mind GSC designed and shelved.

> "The paradigm of Stalker at the time was that we don't really care about the player. We have this living world."
> Andrii Verpakhovskyi, original game designer [9]

---

## Principles

### Everyone wants to live

Self-preservation is the spine of every decision.
A stalker presses an enemy who cannot answer and falls back when he is the one caught weak.
A mutant punishes your opening and breaks off a fight it cannot win.
No side gets an advantage and no side carries a handicap.
That keeps the fight fair.
The Zone is dangerous because the things in it are trying to survive.

### Intelligent decisions

One formula runs every decision, stalker or mutant.
A premise opens it, the read of the actual fight.
Intelligence decides it, a base chance plus a bonus per rank.
A passed decision commits to its window, and a missed one lets the opening go.
No decision fires from a script, and none fires from a coin flip.
Rank only improves the odds of a good decision.
A novice and a legend both act, and the legend is right more of the time.

### Engine first

Every behaviour comes from the engine's own parts.
The GOAP planner, the scheme system, the state machine, and the gulag jobs do the work, and AlifeTactics decides when.
Where the engine had no seam to decide at, the seam went in upstream, as per-NPC hooks written for this mod and merged into the official modded exes.
It never replaces a vanilla script file.
A takeover borrows one NPC for a few seconds of one committed move, then hands him back to the engine's combat brain with cover, flanking, and retreat intact.

### Performance is the first invariant

The whole mod costs a handful of timer compares per frame, whatever the number of NPCs.
No work runs every frame.
The work runs on engine callbacks and scheduled passes, and a flow that cannot hold its budget gets rewritten or dropped.
Performance outranks every feature, and the build measures it on unoptimized single-threaded exes.

### Reviving dead mechanics

The engine shipped behaviour that never ran.
Stalker rank should scale accuracy and gun handling, but the engine clamps the rank value until every NPC collapses to one setting.
Other behaviour shipped broken and passed for a feature, a weapon in full condition that jams every few rounds and loops a tactical reload.
AlifeTactics turns the clamp back into a real curve and strips the fake behaviour out.

### Building what GSC designed

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

## Mechanics

Every mechanic reads the live fight and answers it with what the combatant already is, his rank, his weapon, his faction, his body.

### Maneuvers

A maneuver takes one stalker over to do the thing vanilla cannot, then returns him.
Counterflank spins him onto a hostile player who closed to contact range while he shot someone far off.
Reload Cover sends a stalker caught reloading in a watcher's line to cover that hides him, and the weapon finishes loading on the way.
Flee routs a skittish faction to a distant friendly base.
Retreat pulls a steadier one back to cover under pressure.
Kite walks a stalker out of an enemy who closed inside his weapon's reach, still firing.
Pickoff plants a stalker who outranges his enemy and takes him with single shots, breaking off the moment the threat returns.

The decisions come out looking human.
Nobody turns his back on a shooter.
No one crowds onto cover another man holds.
No one plants in his enemy's line of sight.

### Commitment

Vanilla stalkers re-plan the fight every moment, so a flicker of lost sight or a teammate crossing the line makes one drop a good decision and pick again.
That is the twitchy strafing and constant cover changes of a vanilla firefight.
AlifeTactics holds a stalker to a decision while it still works and lets him switch the moment it stops working.
When his enemy is caught reloading, out of ammo, staggered, or with his weapon down, he keeps firing through the window.

### Conduct

Conduct sharpens the choices the engine already makes, with no takeover.
A rifleman crouches to steady a clear shot and stands to fire over cover a crouch would hide behind.
A submachine gunner accepts closer cover so his fire stays in range, and a sniper holds extra distance.

### Push and Pull

A stalker whose enemy cannot answer presses him, thickening his fire and moving to closer cover.
A stalker caught reloading or hurt while his enemy is strong pulls back, his cover choices landing farther until he recovers.
Both revert the moment the fight evens out.

### Skill by rank

Rank scales what a fighter does with his eyes and his trigger, one curve per NPC.
Accuracy gives each rank a real dispersion curve, because the engine's own curve collapses flat on Anomaly's rank values.
Reaction sets how fast he swings onto a target and how tightly he holds a strafing one.
It also sets how far he leads a mover by the round's real flight time.
Discipline shapes his burst length and cadence.
Vision sets how fast he turns a glimpse into a confirmed threat and how far out he first notices one.
A novice fumbles and a legend is sharp, and no rank is ever certain.

### Crossfire

Allies stop cutting each other down in a shared line of fire.
A hit between two stalkers of the same side deals reduced damage, down to none.
It reads their real relation, so a soured pair still trades fire while true allies stay safe, and your own fire never triggers it.

### Perception

A hostile stalker reacts to gunfire he hears but cannot see, turning to face the shots and moving to cover once he sees the shooter.
He hears you move, each footstep carrying by your stance, the ground, and the weather, and a crouch is silent.
A sound is never a confirmed enemy, so he investigates at a walk and settles into a watch when it fades.
A stalker hit from far off reacts and seeks cover where vanilla left him standing.

### Healing, Jamming, Ammo, Gear

A wounded NPC heals with the medkits and bandages he carries, pausing under fire.
A weapon in full condition no longer jams on a fake roll.
A veteran loads armour-piercing rounds from his own inventory and spends them.
A stalker is as dangerous as the gear he scavenged.
An artefact he carries grants one combat edge by its class, and binoculars and night vision widen what he sees.

### Mutants

Vanilla mutants charge in a straight line and go down to any player on a rock.
A player who strafes on a timer kills one the same way.
AlifeTactics gives every mutant the same decision model the stalkers run.
It reads your reload, your aim, your perch, your strafe, and your weapon's reach, and answers with the moves its own body carries.
Reload and a leap comes.
Perch on a rock and the jumpers climb after you.
Hold your aim on a burer and the gravity throw answers.
The engine still drives every mutant.
AlifeTactics scales its stats and commands the moves.
The engine validates and runs each one.
Power grows with the map's danger, so a dog in Cordon barely changes while a chimera deep in the Zone is far deadlier.
Every ability value varies inside the species' own envelope, so no two spawns read alike.

---

## Sources

- [3] Oles Shishkovtsov (engine programmer), ModDB
  https://www.moddb.com/features/stalker-interview
- [8] GSC Game World internal AI design documents, pre-release, circa 2002-2006.
  The combat tables rate every creature for attack and defence, a courage score, and a hunger that drives hunting and flight.
  Leaked in the "Koan's Gift" package and catalogued on the STALKER wiki.
- [9] Andrii Verpakhovskyi (original S.T.A.L.K.E.R. game designer), Eurogamer, reported by VideoGamer
  https://www.videogamer.com/news/original-stalker-a-life-so-advanced-that-players-were-constantly-outsmarted-by-ai/
