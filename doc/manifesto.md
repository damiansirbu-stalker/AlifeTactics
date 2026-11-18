# Manifesto

AlifeTactics makes the Zone's enemies fight to stay alive.

In vanilla Anomaly a rookie fires with a legend's spread. Most mutants run straight at you.
AlifeTactics gives every stalker and every mutant a read of the fight, and a reason to act on what it sees.
If you reload in a stalker's sights, he runs for cover. A burer waits for you to look away before it throws.
When a chimera of the pack hits you, the others hold their next leap on you for a few seconds, and each chimera leaps with the set of moves it rolled for itself.

GSC built a mind like this first and cut it later.
Its stalkers outplayed the player so often that the studio held them back, to leave him room to win [9].

> "The paradigm of Stalker at the time was that we don't really care about the player. We have this living world."
> Andrii Verpakhovskyi, original game designer [9]

---

## Principles

### Everyone wants to live

I don't tune these fighters to beat you. I give them reasons to stay alive, and the fight follows from that.
Every decision starts from self-preservation.
Stalkers press an enemy who cannot fire back. If they are the ones reloading or hurt while the enemy is strong, they fall back to cover instead.
Mutants near death may break off when you are far enough away, but a cornered mutant keeps fighting.
Each side reads the same fight and plays it for itself.

GSC's AI brief for its testers lists these states as ready [10]:

> "Advance on the enemy through cover: when the opponent is [cut] weaker or equal.
> Fall back from the enemy through cover: when the opponent is stronger.
> Panic: when the opponent is much stronger. [cut]
> Reload: characters try to get into cover to reload."
> GSC AI brief for testers [10]

### One mind for stalkers and mutants

I wanted one mind for both sides of the Zone, so a stalker and a chimera decide the same way.
A decision starts from an opening, a real read of the fight such as your reload, your turned back or your aim on him.
Personality and Intelligence decide it. GSC's brief gives characters and monsters the same list of traits, and one entry reads "Intelligence affects the tactical tricks applied" [10].
Personality is character, the moves an individual carries at all, rolled once from the chances of the faction or species.
Intelligence is the read, whether a fighter takes this moment right, and it rises with a stalker's rank and with the danger of a mutant's map.
A legend reads more moments right than a rookie, but even a legend misreads some.

Not every behaviour needs both traits.
The moves AlifeTactics fires, such as maneuvers, presses and leaps, need both: the fighter has to carry them and read the moment.
When the read fails, hurt stalkers and mutants may flee instead, because flee is the other face of the same roll.
The habits the engine fires itself, such as cover posture, roll once and hold for the fighter's life.
Wild spreads a mutant's raw numbers with no roll at all.
A repair to vanilla never rolls, because it applies to everyone. Skill grows with rank on every stalker, whatever drives him at the moment.

### Every faction and every species fights its own way

Each faction carries a chance for every move, which is why one opening draws a different answer from each of them.
Monolith fighters are ready to press a helpless enemy, they almost never give ground, and Monolith has no rout at all.
Duty takes cover to reload, gives ground in order, and likes to plant for deliberate shots.
Freedom backs out of close range and keeps its distance.
Mercenaries plant to pick off an enemy they outrange, finish a man who cannot answer, and pull back when they are the weaker side.
Bandits fire through a helpless man's reload, but they are likely to run when the fight turns.
Ecologists never press, and they run as soon as the fight turns against them.

A species carries its moves the same way.
Chimeras claw at point blank, leap without a warning crouch and punish your reload or your turned back. Chimeras have no flee move.
Most snorks steer in the air after you, while dogs seldom leap and seldom run.
A flesh is the likeliest creature to run when it is hurt. Every burer carries its gravity throw, and every bloodsucker its cloak.

This is GSC's design. Its soldiers fight as a squad under a leader who signals each man by gesture and sends them around the flank [12].
Its stalkers follow strategies of their own, and a zombied soldier is a separate type [12].
Every NPC sizes up an enemy by his type, his allies, his cover and his weapon before choosing to attack, defend or run, and GSC sorts character from principled through independent to unprincipled [12].
Its tables score every creature and every kind of human for courage in attack and in defence [8].

### Engine first

AlifeTactics drives X-Ray's combat brain and never replaces a vanilla script file.
I would rather fix the engine than work around it in Lua. Where Lua could not reach a decision, we put the hook into the engine first.
The per-NPC controls for aim, bursts, cover and action switches were written for AlifeTactics and merged into the X-Ray Monolith modded exes.
A maneuver borrows one stalker from the engine's planner for one move. Then it hands him back with the rest of his combat intact.

### GSC's unbuilt list

GSC's AI design documents list the combat work that remained to finish [8].

> "Improvements to character combat behaviour: introduce an accuracy parameter.
> Introduce an intelligence parameter.
> Make vision more realistic.
> Adjust combat tactics.
> Adjust turning around to the rear.
> Tune combat functions against every monster type. [cut]"
> GSC AI design documents [8]

We built the stalker side of that list:
- The accuracy and intelligence parameters are Accuracy and Intelligence.
- Realistic vision is Vision.
- The combat tactics are Maneuvers and Commitment.
- The turn to the rear is Counterflank.

The tuning against monster types is still open. A stalker fighting a mutant runs on vanilla tactics.

### Vanilla, repaired

Some of vanilla's combat never worked.
The engine clamps rank below the values Anomaly uses, which leaves every stalker with one shared spread.
The danger scheme crashed on mutant corpses and torn-down stalkers. It froze a stalker who held fire for a friend, and it played the corpse search only on the first body.
We repair each of these, and the Fixes tab lists every repair by name.

---

## Systems

Each MCM tab is a system, in the MCM's order.

### Combat

Combat is how a stalker chooses what to do in a fight.

#### General

Intelligence lives here, the chance that stalkers read a moment right, set by their rank.
The page also stops the hit slide. In vanilla a stalker hit mid-step slides across the ground with his feet still.
With AlifeTactics, hit stalkers stay planted until the hit animation ends, then walk on.

#### Maneuvers

A maneuver takes one stalker from the engine's planner for a single committed move, then gives him back.
Each move answers one real problem, and it ends when the problem does.
- Counterflank: when you reach contact range during a fight with someone far off, the stalker turns on you.
- Reload Cover: a stalker who reloads in a watcher's sight runs to the nearest cover that hides him and reloads on the way.
- Kite: when an enemy gets inside the minimum range of the weapon, the stalker backs out down a clear lane, still firing.
- Retreat: a hurt stalker on an exposed spot gives ground to the best cover behind him.
- Pickoff: a stalker who outranges his enemy in a standoff plants and takes aimed single shots.
- Flee: the last man of a broken squad routs to a friendly base, as long as his faction routs at all.

Maneuvers run in every stalker fight near you, yours and theirs.
A shared budget keeps a mass battle from turning into a flood of moves.

#### Commitment

In vanilla a stalker drops a working shot whenever the engine offers him another cover, and he strafes between covers under fire.
Commitment refuses that switch while his fire works, he sees his enemy and his line is clear. It lets him go the moment one of those fails.
At the end of the hold, Liquidate keeps him firing while his enemy is reloading, out of ammo, staggered, disarmed or turned away.
It runs on two engine hooks written for AlifeTactics, the action switch and the cover re-pick.

#### Conduct

Conduct improves two choices the engine keeps making for itself, with no takeover.
- Cover posture: a trained rifleman crouches only where the shot from the crouch is clear. Where a crouch would put his rounds into the wall, he stands.
- Weapon spacing: a submachine gunner takes closer cover to keep his fire in reach, while a sniper keeps his distance.

A stalker rolls this discipline once for his life. Duty, the army and the ISG almost always have it. A renegade almost never does.

#### Behaviors

Behaviors reads who has the upper hand.
- Push: if a target cannot answer, because he is reloading, out of ammo or badly hurt, his attackers press him with heavier fire from closer cover until he can fire back.
- Pull: if a stalker is the weaker side, reloading or hurt while his enemy is strong, his next cover moves farther back until he recovers.

Pull outranks Push, because self-preservation comes first.

### Effectiveness

Effectiveness is how well a stalker fights, and his rank decides it.
- Accuracy: a rookie fires with the vanilla spread, and each rank above groups tighter. A second curve keeps a veteran steadier when he fires on the move.
- Crossfire: allies and neutrals in one line of fire stop gunning each other down. Hostile pairs and your own bullets still do full damage.
- Reaction: with rank, his barrel follows you faster and holds a strafe better.
  Target Lead aims ahead of a runner by the round's real flight time. A legend hits you on the run, but a rookie leads too far and misses in front of you.
- Discipline: a veteran fires shorter, tighter bursts, while every rank keeps at least vanilla's rounds per minute.

The aim curves start at vanilla for a rookie and stay short of the game's hardcore aim.

### Perception

Perception is what a stalker notices.
On the multi-thread exes a vanilla stalker keeps a heard sound only from an enemy he already remembers. Before first contact he is close to deaf.
- Sound: he hears you before he has seen you. Your footsteps carry by stance, surface and weather, and so does a jump, a reload racked nearby or an empty click.
  If a sound is loud enough, he walks over to check the spot with his weapon ready. A crouch makes no sound at all.
  A shot he hears runs the engine's full plan of cover, look-out, detour and search. He also glances toward a creature he hears.
  GSC's stalker strategies find an enemy by the same sounds: "shot, reload, clicking" and "stomping, jumps" [11].
- Neutral gunfire alert: a neutral or friendly stalker turns weapon-ready toward the shots you fire near him.
- Vision: rank sets how fast he turns a glimpse into a confirmed threat, and how far out he starts to notice one. Every rank sees at least as well as vanilla.
- Danger: AlifeTactics layers its repairs onto whichever danger script your modpack ships.
  A stalker hit from far away turns and takes cover. A hit stalker turns on the man who shot him.
  If you swing a knife at a stalker and miss, he is alerted, and a hostile who sees you fights.

### Mechanics

Mechanics is the gear a stalker carries, and what it does for him in a fight.
- Healing: a hurt stalker heals with the medkits and bandages he carries. Vanilla ships the healing item lists empty, so its stalkers never used them.
  He pauses while he fights, and he limps with his weapon in hand.
  GSC wrote the wounded stalker's drill around that medkit: "find cover [cut] draw a medkit and heal" [11].
- Jamming: a worn NPC weapon misfires by its condition, the same way yours does. The jam the modded exes inject for NPCs is gone.
- Ammo: a veteran loads the armor-piercing rounds he carries, and he spends them fight by fight.
- Attachments: out of a fight, a stalker fits the scope, silencer or launcher he looted. A scoped stalker aims tighter from a stop, while a silenced one is heard closer in.
- Gear: an artefact gives its carrier one combat edge by its class, less damage taken, tighter fire or more damage dealt. Binoculars and night vision extend his sight.
  Carriers have a shimmer of warped air around them, and you can hunt them for what they carry.

### Mutants

Mutants run the same mind as stalkers, with no takeover. We keep X-Ray's mutant AI. The engine still moves every mutant, and we pick the moment and the move.

#### General

A mutant's Intelligence rises with the danger of its map, and its Personality comes from its species.
With Wild on, two mutants of a kind seldom fight alike.
Mutants space their moves apart, and after one of their moves hits you, the pack gives you a moment before the next.

#### Moves

Every species page splits its moves into three kinds.
- New moves are attacks vanilla never fires for the creature, built on parts X-Ray already carries: its jump, an unused rear strike, its run-away state.
- Managed moves are attacks it already has, which AlifeTactics fires at the moment the fight calls for them.
- Influenced moves are abilities whose numbers Wild spreads between individuals of one kind.

The leap family is the largest new move. It reads your perch, your aim, your reload and your movement.
- High leap springs up at you on a rock or a ledge.
- Punishing leap strikes when you aim at it, reload, sprint or turn your back.
- Hunting and Feral leap steer in the air after you. Feral leap gives no warning.
- Catching and Cutoff leap jump to where you will be. Cutoff leap gives no warning.
- Closing leap carries a bloodsucker to you before its claws strike.

The other new moves answer a single opening.
- Claw: a chimera or a lurker pounces at point blank. Both rest after each jump, where vanilla let them jump again at once.
- Backstrike: a chimera, lurker, burer or controller spins on you when you get behind it.
- Flee: a mutant near death runs when you are far enough away.

The managed moves are the burer's gravity throw and the bloodsucker's cloak.
The burer throws when you look away from it or cannot answer. The bloodsucker vanishes when you aim at it or hit it as it closes in.
The influenced moves are the burer's gravity numbers and telekinesis, the controller's psi tube, stamina and aura, and the poltergeist's detection.

GSC's monster design gives the burer its telekinetic strike and gravity field, and the bloodsucker its invisibility.
It has the fracture strike at a man who turns his back, the opening Punishing leap reads.
It calls the flesh a coward, and in AlifeTactics the flesh is the creature most likely to run [8].

---

## Sources

- [8] GSC Game World internal AI design documents, pre-release, circa 2002-2006.
  Ai.doc has the combat-improvements list, and monstry.doc the creatures and their moves.
  The EFC table creature_courage scores every creature and kind of human for attack and defence courage.
  Leaked in the "Koan's Gift" package, published on the Gameru.net forum, and catalogued on the STALKER wiki.
- [9] Andrii Verpakhovskyi (original S.T.A.L.K.E.R. game designer), Eurogamer, reported by VideoGamer
  https://www.videogamer.com/news/original-stalker-a-life-so-advanced-that-players-were-constantly-outsmarted-by-ai/
- [10] GSC Game World, AI technical brief for testers (Technical Brief_ai), from the same archive as [8].
  Section 3 lists the attack and defence states, marked ready. Section 8 lists the characteristics of characters and monsters.
- [11] GSC Game World, Stalker Strategies (ai/monsters), from the same archive as [8]: the sound types an enemy is found by, and the wounded stalker's drill.
- [12] GSC Game World, from the same archive as [8]: FSM Strategies and Soldier rules hold the soldier squad's doctrine.
  The offline AI holds the danger estimate by character type, and Ai.doc the character types.
