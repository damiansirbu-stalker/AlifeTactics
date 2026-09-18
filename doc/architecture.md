# AlifeTactics Architecture

## Overview

AlifeTactics is a combat AI mod for STALKER Anomaly. Every system works the same way. It reads the real state of the game and decides in the combatant's favor.
No scripted sequence and no die roll ever stands in for that read. The inputs are combat events, NPC and player stats, the world, the squad, the faction, both sides' weapons, range, angle, and cover.

AT replaces no vanilla file. Every system attaches to an engine seam and composes, so the mod runs under any combat brain a modpack ships: vanilla, GAMMA AI Rework, ReDone Combat AI.
The one override is the Maneuvers takeover, which borrows one NPC for one committed, time-boxed maneuver and hands him back.

The systems:

```
AlifeTactics
├── Substrate (no user surface)
│   ├── at_core.script              the per-stalker store and the 200ms monitor
│   ├── at_faction.script           per-faction behavior chances
│   ├── at_effects_resolver.script  the multi-source effects combiner
│   ├── xcombat (xlibs)             the boundary for every engine combat call
│   └── the GOAP graft              the takeover control point (xcombat.script classes, at_maneuvers.script consumer)
├── Combat
│   ├── Maneuvers       the takeover
│   ├── Commitment      the anti-shuffle veto
│   ├── Conduct         cover posture and weapon spacing
│   └── Push and Pull   pressing the enemy's weak moment, escaping your own
├── Effectiveness
│   ├── Accuracy        rank dispersion curves
│   ├── Reaction        aim tracking, lock, target lead
│   ├── Disclosure      the hit-victim turn and squad investigate
│   ├── Crossfire       friendly-fire damage gate
│   └── Discipline      rank burst shape
├── Perception
│   ├── Sound           movement and handling noise hearing
│   ├── Vision          rank vision speed and range
│   └── Danger          the danger-scheme runtime patch
├── Mechanics
│   ├── Healing         active first aid
│   ├── Jamming         NPC misfire suppression
│   ├── Ammo            NPC AP ammo simulation
│   └── Gear            functional NPC inventory
├── Effects (wip)       decided category, no built system yet
├── Mutants (wip)       decided category, no built system yet
└── Observability (dev tooling, off in play)
    ├── at_debug.script        code tracing, one logger to alifetactics.log
    ├── at_world_trace.script  the slide watchdog and the ballistics recorder, to alifetactics_world.log
    ├── at_hud.script          the live debug HUD
    └── at_test.script         console test commands
```

The document follows this order. Each section names its level in full before opening any element.

Built on xlibs (xcombat, xsquad, xttltable, xtime, xprofiler, xlog, xmcm, xslice, xcreature). AT is part of a four-mod alife family.
AlifePlus extends A-Life with new behaviors. AlifeBalance tunes rates the engine already owns. AlifeGuard releases and repairs alife state. AlifeTactics controls how NPCs fight.

## Integration model

Every system touches stalker combat through one of seven methods, each a distinct way a mod reaches into the xray combat brain.
The grip axis orders the methods by how much of the engine's combat decision chain each one displaces.

```
grip     method             AT systems                             what it does
strong   forced action      Maneuvers                              grafts one evaluator and action at reserved GOAP id 188347 and
         (takeover)                                                precondition-blocks the vanilla chain; holds one NPC for seconds, then releases
         engine-hook veto   Commitment                             answers allow or deny before the engine commits one proposed decision:
                                                                   the action switch, the best-cover re-pick
         per-NPC bind       Reaction, Vision, Disclosure,          plants a standing per-NPC parameter; the engine consults it with no Lua
                            Gear regen, Ammo, the move-hold,       on the hot path (aim, vision, burst scale, selection weights, ammo type)
                            the Push fire bump
         callback adjust    Accuracy, Crossfire, Conduct,          scales or answers one per-event value inside an engine callback (dispersion,
                            the Push band, Pull, Effects resolver  hit power, a distance band, a posture ask)
         function patch     Danger, Healing, Jamming               replaces one anomaly-layer module function in place; the patch reschedules
                            (Sound feeds the patched scheme)       through the same lookup, so it holds
weak     squad simulate     the flee holster re-assert             lays a state overlay on a driven NPC; no planner is touched
-        inject             none, unused by design                 a competing action can lose the solve to another mod's action; AT forces
                                                                   (Maneuvers) or composes (everything else), never competes
```

Where each method lands, top of the stack to the bottom:

```
layer                                                              methods landing here
AlifeTactics   at_*.script - policy only
xcombat        xlibs - one wrapper per engine call                 the boundary every method crosses
anomaly (Lua)  xr_danger, xr_eat_medkit, xr_weapon_jam,            function patch replaces a module function;
               state_mgr, the scheme binder                        squad simulate lays its state overlay
luabind        the engine callbacks and setters                    callback adjust, veto, and bind cross here
xray (C++)     the GOAP planners, CEnemyManager, sight, vision,    forced action blocks the planners; a veto denies
               fire, the weapon handler                            one decision; a bind is consulted every frame
```

- One override. Only forced action blocks the vanilla chain, and only for the seconds a maneuver holds. Release lifts the block and the installed brain resumes. Every other method leaves it running.
- Veto vs bind. A veto answers allow or deny on a transition the engine already proposed. A bind plants standing state the engine consults on its own.
- Provenance is per-seam, never a category. A demonized seam probes at load (type(fn) == "function") and goes inert on a floor exe, one INACTIVE line. Both shapes have vanilla members
  (wpn:set_ammo_type is a vanilla bind).
- Seams before workarounds. Where Lua cannot reach a C++ decision point (the action switch, the cover re-pick, per-NPC aim), the seam is added to the demonized build first and consumed second.
- xcombat boundary. No AT system makes a raw engine combat call. Every method is issued through an xcombat (xlibs) primitive. AT owns policy (when, whom, which maneuver). xcombat owns mechanism
  (how it reaches the engine).

## Control model

Every ongoing behavior runs on a vanilla time event or reacts to a discrete engine event. Nothing runs per frame (the Invariants). One loop drives combat.
The service loops run beside it, each inert until its feature or its toggle needs it.

```
loop                       owner and cadence                    walks
_run_monitor               at_core.script, 200ms                the combat records: facts, then maneuvers, then behaviors
_run_monitor               at_ammo.script, 5s                   the spawn-filled roster: the AP ammo tick
_run_monitor               at_healing.script, 200ms             the spawn-filled roster: the limp pose and its drop detectors
_run_pass                  at_sound.script, 500ms (tick_sec)    online stalkers inside the accumulated noise radius
_run_monitor + _run_watch  at_world_trace.script, 150ms + 60ms  near-actor NPCs; registered only while the world-trace toggle is on
_update_hud                at_hud.script, 500ms                 the capped visible row set; debug HUD only
```

The per-event systems hold no loop. Their cadences live in their sections - at_conduct's 3s decide hold, the danger scheme's per-solve evaluation, at_healing's per-heal event chain.

The combat loop:

```
events (write the record)                     _run_monitor (at_core.script:97; one vanilla time event, 200ms)
npc_on_net_spawn    -> the record             re-arm FIRST (_reset_monitor)
npc_on_hit_callback -> hit_at, hit_by         for each record in _npc_states:
reload start/stop   -> reloading, reload_at     dead -> clear maneuvers, clear behaviors, done
(PR #611; INACTIVE without it)                  write actor_dist_sqr
                                                refresh best_enemy_id (per 600ms)
                                                gate up   -> at_maneuvers.run_maneuver (update + end checks, 200ms each)
                                                gate down -> at_maneuvers.try_maneuver (begin check, 600ms)
                                                always    -> at_behaviors.update_npc

teardown (npc_on_net_destroy / server_entity_on_unregister; death clears the systems and keeps the record until despawn)
  at_maneuvers.clear_npc -> at_behaviors.clear_npc -> the reverse-index edge -> the record -> xcombat.release_takeover
```

- Re-arm first. ProcessEventQueue (_g.script:364) has no error protection. An expired event that errors would re-fire and re-error every frame and starve every time event in the game.
  Re-armed, a fault costs one aborted pass (at_core.script:99).
- One compare per frame. The whole monitor costs one due-time compare inside ProcessEventQueue's walk, registered from actor_on_reinit (bind_stalker_ext.script:26).
  Time events do not survive a save load, so _apply_enabled arms the loop at actor_on_first_update.
- The loop runs while any client system needs it. Combat, push, or pull enabled arms it. All three off stops it outright (at_core.script:208-213).
- Three states per NPC, real cost only in the last. Not fighting costs one best_enemy read per begin check. Fighting with no maneuver open runs the begin check on engine-memory reads alone,
  no raycasts. A running maneuver runs the update and end checks, which re-apply the row's state and watch for the end.
- Measured, not asserted. The [MON] span logs tick_avg and tick_max per 5s window when debug is on, against the 0.1ms-average and 2ms-ceiling budget (at_core.script:69-79).
- Three loop shapes were built and rejected, DO NOT RE-ATTEMPT. An npc_on_update subscriber crossed C++ to Lua per online stalker per frame before any throttle could bail (30 stalkers at 60fps is
  1800 crossings a second against the time event's one compare). A round-robin sweep off the actor's actor_on_update frame callback coupled flee's 200ms holster re-assert to the online crowd size,
  so a fleeing stalker got his weapon down later the more NPCs were online. Per-NPC time events cost N due-time compares per frame where the single pass costs one, for the same honored cadences.
- A pure event model is impossible. Stalled-for-4s, hurt-with-the-enemy-in-reach, and the time caps are continuous conditions with no event edge, so bounded polling is required and the single time
  event is its minimum.

## Data & ownership

One store, one pass. _npc_states (at_core.script:20) holds one record per tracked stalker, created on npc_on_net_spawn and deleted in the one teardown sequence. Every field has one writer.

```
_npc_states[id] field           writer                   meaning
id, ran_at{}                    at_core.script (spawn)   identity; the per-check clocks (at_maneuvers stamps them per check)
hit_at, hit_by                  at_core.script (hit)     the last landed hit: time and shooter id
reloading, reload_at            at_core.script (reload)  the PR #611 reload edges
actor_dist_sqr                  the loop                 squared distance to the player, written once per pass
best_enemy_id, best_enemy_at    the loop                 the engine selection fact, refreshed per 600ms; the loop is its only writer
gate, the maneuver fields       at_maneuvers.script      the takeover gate, the open maneuver, its committed target (enemy_id), the move hold
push_*, pull*                   at_behaviors.script      the open press and pull windows
```

- The reverse index. _best_enemy_of maps a target id to the set of NPCs holding it as best enemy. The loop maintains both directions of the edge (at_core.script:41-62).
  get_threats returns candidates only. The reader verifies every predicate live at its decision, so a stale entry costs one failed compare.
- The start budget. One sliding-window pool (_starts, an xttltable counter) split into vs_player and vs_npc buckets, so neither class can starve the other.
  6 starts per 10s per bucket by default, 0 lifts the cap (alifetactics\at_maneuvers_config.ltx, [at_core]). Starts only spend: a maneuver grant, a first press write, a pull open.
  Re-applies and engine re-grants never count. can_start() is the cheap pre-walk bail.
- Why one store. A second pass would mean a second time event, duplicate timestamp subscribers, and a second id-keyed store to drift.
  The behaviors' per-NPC fields stay on the shared record with at_behaviors.script as their only writer.

The publics every system reads (at_core.script:230-295):

```
check_start(bucket) -> bool                room left in the bucket's window
add_start(bucket)                          spend one start
can_start() -> bool                        either bucket still admits a start
get_combat_record(id) -> record|nil        the per-stalker record
get_combat_records() -> store              read-only, for walkers (the debug HUD)
get_best_enemy(id) -> id|nil               the selection fact
get_threats(id) -> bucket|nil              the reverse-index candidates
has_recent_hit(state, now, ms) -> bool     hit_at inside the caller's window
is_enemy_vulnerable(enemy) -> bool[,term]  the one vulnerability rule (below)
is_in_gate(state) -> bool                  actor_dist_sqr against the one gate radius
```

- One vulnerability rule, branched per kind (at_core.script:274-289). The actor answers on the weapon block terms (xcombat.get_block_reason - unarmed, reloading, empty magazine) plus sprinting
  and climbing. A stalker answers on the block terms plus the animation terms (xcombat.is_body_busy). A mutant never answers, because it has no weapon state to read.
  Every term reads objective state and no threshold enters the rule.
- One gate radius. is_in_gate compares the record's actor_dist_sqr against gate_radius_m (150 default, the sniper band maximum, so the gate never excludes a fight the weapon bands model).
  The combat systems' scope and the debug HUD's display set read the same fact through the same public.

## Invariants

Project-wide constraints. Every system holds all of them, and a change that violates one is wrong even when it works.

- Performance first. Performance outranks features. A feature that cannot meet the budget below is reworked or dropped or handed to an engine PR, and it is never kept at the cost of the budget.
  Only correctness and never-break-base-gameplay rank above it.
- Use the engine, do not work around it. Every capability comes from the engine and the anomaly layer first, always through xlibs, and our own code enters only where stock behavior falls short.
  Never reimplement in script what the engine already does.
- No per-frame work, ever. Ongoing work runs on a scheduled vanilla time event or a discrete engine event, never continuously and never on a per-frame engine callback. Dispatch work in front of a
  throttle IS per-frame work (the at_core monitor moved off npc_on_update for exactly this). Frame-spreading a bounded one-off batch (xslice, one item per frame) is the one allowed use of the frame.
- The 2ms ceiling. Every measured flow targets 0.1ms average per call with a hard 2ms ceiling, an eighth of a 60fps frame. Cold start, save load, and level transition all count, and debug-only tools
  count too.
- The 20v20 bound. The whole mod holds its parameters in a 40-combatant fight (at_test.start_arena(40)) - the monitor pass averages 0.1-0.4ms and never crosses a hard 4ms ceiling. 50v50 is rejected
  as a test scale, because anomaly itself malfunctions at 100 combatants. The [MON] span at 20v20 is the acceptance test for any change that widens scope or adds per-NPC work.
- Measured, not asserted. Each phase carries ONE aggregate duration in its DEBUG trace, and the timers are null objects when debug is off, so measurement costs nothing live. A mechanism like a cache
  or a throttle is never justified by an unmeasured cost. The decide-path decline backoff was built and removed the same day for exactly this.
- No file overrides. AT replaces no vanilla file. Every system attaches by a callback, a function patch, a save-wrap, a DLTX overlay, a scheme patch, or the time-boxed takeover, so composition with
  modpacks falls out of the attach mechanism.
- Engine truth. Every mechanism claim in this document carries an engine source cite. A behavior that could not be proven from source does not ship, and where the engine had no seam, the seam was
  added upstream first (the demonized PRs).
- The takeover is a bounded transaction against one stated problem. Vanilla owns every NPC by default. AT borrows one NPC for one committed, time-boxed maneuver and releases it, at most one open
  maneuver per NPC, the start budget bounding the total, ended on arrival, a cap, or a broken premise, cleaned up on death and despawn. There is no reseize cooldown, because every row's need states
  the full problem and the maneuver's success negates it. A maneuver that fails to solve it is still a valid transaction. Every maneuver is locked to the target it was staged against. AT stays an
  interrupt over vanilla and never becomes the combat brain.
- xcombat boundary. Every NPC combat command and read goes through an xcombat (xlibs) primitive. AT owns policy (when, whom, which maneuver), xcombat owns mechanism (how to issue it to the engine).
- Debug is free when off. Every trace call gates on one integer compare (at_debug.is_on()). The off path crosses no luabind bridge and computes nothing that exists only for the trace.
- Every vanilla or xray fix is enumerated on all three surfaces. A fix to a vanilla or xray defect appears in readme.txt under Fixes to Vanilla, in the changelog as a Fixed vanilla bug line, and on
  the MCM Fixes tab (the always-on ones as locked toggles). No fix is invisible, and a fix is not done until it is on all three.

## File and config layout

```
AlifeTactics/gamedata/
├── scripts/
│   ├── _at_manifest.script          identity data (name, version, xlibs)
│   ├── _at_init.script              the dependency gate and compatibility floor
│   ├── at_mcm.script                the MCM tree and the one config-defaults source
│   ├── at_debug.script              the code-trace primitives (one logger, the on() gate)
│   ├── at_core.script               the per-stalker store, the 200ms monitor, the start budget
│   ├── at_maneuvers.script          the takeover lifecycle, the row methods, the arbiter
│   ├── at_maneuvers_config.script   the catalog rows (structure LTX cannot hold)
│   ├── at_faction.script            faction flavor (get_faction_chance, has_flavor)
│   ├── at_behaviors.script          the Push and the Pull
│   ├── at_commitment.script         the anti-shuffle veto (switch and cover re-pick)
│   ├── at_conduct.script            cover posture and weapon spacing
│   ├── at_accuracy.script           rank dispersion and moving-fire curves
│   ├── at_reaction.script           aim, lead, vision, and fire discipline (one file, three pages)
│   ├── at_disclosure.script         the hit-victim turn and squad investigate
│   ├── at_danger.script             the danger-scheme function patch (@override)
│   ├── at_sound.script              movement and handling noise hearing
│   ├── at_crossfire.script          the friendly-fire damage gate
│   ├── at_healing.script            active first aid (rate, charge, limp and heal anims)
│   ├── zzz_at_healing_patch.script  the vanilla on_register re-roll suppressor
│   ├── at_jam.script                the modded-exes misfire suppressor
│   ├── at_ammo.script               NPC AP ammo simulation
│   ├── at_gear.script               the functional-inventory source
│   ├── at_effects_resolver.script   the multi-source effects substrate
│   ├── at_compat.script             the grok_bo hit-pipeline compatibility wrapper
│   ├── at_world_trace.script        the slide watchdog and ballistics recorder
│   ├── at_hud.script                the live debug HUD
│   └── at_test.script               console test commands
├── configs/
│   ├── ai_tweaks/                   the DLTX overlays (mod_xr_danger_at.ltx, mod_xr_eat_medkit_at.ltx)
│   ├── alifetactics/                the per-system numeric tunables (at_<system>_config.ltx)
│   ├── ui/ui_at_stats.xml           the debug-HUD layout
│   └── text/eng,rus               the MCM strings
└── textures/                        the MCM banner
```

- Namespace: at_* (parallel to ap_* for AlifePlus, ag_* for AlifeGuard, x* for xlibs).
- Config pairing, per system: at_<system>.script holds the logic, at_<system>_config.ltx holds the numbers, and at_<system>_config.script exists only where LTX cannot hold the shape (the maneuver
  catalog rows reference methods). The MCM defaults in at_mcm.script are the ONE source for the per-tier tables, so no LTX or script copy of those tables exists to drift.
- The compatibility floor (_at_init.script). AT requires xlibs >= 1.8.5 and modded exes - demonized >= build 20250908, or AOEngine (probed via get_aoe_version). The dep gate asserts these at boot and
  refuses to run below them. The platform status has three tiers: full at or above TARGET (build 20260809), fallback below it (partially compatible), and blocked with no modded exes.
- The floor is fixed, and features never raise it. A feature that needs a post-baseline engine symbol probes for it (type(fn) == "function", not a version compare) and goes inert with an INACTIVE
  log line when it is absent. The readme Compatibility block states the floor and the fallback for each system.

## Substrate

The substrate carries no user surface. at_core.script is described above (Control model, Data & ownership). Faction flavor, the effects resolver, the xcombat boundary, and the GOAP graft follow.

### Faction flavor

at_faction.script decides WHETHER a behavior triggers for this NPC, before the behavior's own mechanics run - the flavor-first law. Every AT decision can carry a per-faction chance.

- Data. at_faction_config.ltx: section = faction, key = the decision, value 0 to 1. Lookup reads the faction's key, then [default], then 1 (at_faction.script:13-23). Chances cache per faction and key.
- The roll. has_flavor(npc, key) rolls once per NPC per key and holds the boolean 300s in an xttltable, so one stalker either carries a behavior for the whole fight or does not
  (at_faction.script:27-36). No fight outlives the hold, and the TTL avoids policing the fight edge, which flickers across lulls. 0 and 1 answer without touching the roll table.
- The keys. Six maneuver_* keys (the first stage of the catalog walk, trace stage flavor), behavior_push and behavior_pull, conduct_crouch, and eight participation keys
  (accuracy, reaction, disclosure, crossfire, gear, healing, commitment, danger) carried at 1 by every faction and 0 by [zombied] - the file is the zombied participation filter.
- Three zombied sites stay code because they are behavior, not participation: at_conduct.script's forced-STAND posture branch and spacing skip row, and at_danger.script's corpse-danger condition.
- The character lives in the numbers, all user-set: the militarized factions carry posture discipline and never rout, the flee-prone run first, the push skews to the aggressive factions.

### Effects resolver

at_effects_resolver.script combines the combat effects MORE THAN ONE source feeds and owns their engine writes. The sources own the scans. Four invariants govern it:

- I1. An effect is a per-NPC value multiple sources feed, combined at a fixed engine point (at_effects_resolver.script:8-23):
  max, min for a reduction (DAMAGE_RESIST, SHOT_DISPERSION), boolean OR for the aura.
- I2. Medkit healing is an action owned by at_healing.script - the NPC consumes an item and runs the xr_eat_medkit chain. Its rate and charge chance are parameters of that action, never effects.
- I3. A value enters the resolver ONLY when more than one source feeds it. One source means one writer and no clash, so it stays in its owning module. This is why aim, vision speed, fire discipline,
  the move penalty, and passive regen never touch resolve.
- I4. Passive regen is a distinct engine lever from medkit healing: the condition velocity (m_fV_HealthRestore, EntityCondition.cpp:642), owned by at_gear.script through the n039 bind.

```
sources (lazy, cached in the source)      resolver (at_effects_resolver.script)     engine writes (the appliers)
at_accuracy   rank SHOT_DISPERSION        register(effect, provider)                net_spawn: set_view_distance_factor + the aura particle
at_reaction   rank VISION_RANGE slice     resolve(npc, effect) keeps the            before_hit: apply_hit_power, victim DAMAGE_RESIST then
at_gear       artefact classes feeding    strongest contribution; a provider        attacker DAMAGE_DEALT (actor seam: attacker side only)
              DEALT, RESIST, DISPERSION,  returns nil and drops out; no clamp       shot: apply_dispersion, SHOT_DISPERSION
              VISION_RANGE, AURA          (the ceiling is the largest tier value)
```

The effect set, each with its combine mode and sources:

```
effect           combine   sources
DAMAGE_DEALT     max       at_gear: electro and the quest-special artefact classes
DAMAGE_RESIST    min       at_gear: gravi, ballistic plates, the chemical interim
SHOT_DISPERSION  min       at_accuracy: the rank cone; at_gear: thermal
VISION_RANGE     max       at_reaction: the rank slice; at_gear: binoculars by day, NVG by night
AURA             OR        at_gear: every artefact class (plates and optics emit none)
```

- The performance spine. Each provider walks the NPC once on first call and caches in its own module. resolve re-combines cached answers - table reads plus max or min.
  The net_spawn applier is the first caller in practice, so the walk lands there, never inside a hit or shot callback (at_effects_resolver.script:147-165).
  Every applier runs under a null-object xprofiler timer (at_effects_resolver.script:62,81,113). A stagger gets built only if a measured first-online burst crosses the budget.
- The shot seam carries two independent multipliers, at_accuracy's single-source move penalty and the resolver's SHOT_DISPERSION. Both multiply, so subscriber order is irrelevant.
- The aura is start-only. The particle attaches at net_spawn on the configured bone or the first fallback the skeleton accepts, and dies with the game object. No stop path exists,
  because stop_particles trips the engine bone assert on a non-renderable bone (proven live). Death and unregister only clear the emitting mark (at_effects_resolver.script:48-59, 123-129).
- apply_binds(npc) re-pushes the spawn binds after a source invalidates its cache on a gear change (at_effects_resolver.script:168-171). resolve returning nil writes the neutral value,
  so a dropped optic never keeps a stale factor.
- Dropped, do not re-add: MORALE (nothing in the NPC simulation consumed it), HEAL_RATE (reached across the module boundary into the medkit action, which I2 forbids),
  gear feeding VISION_SPEED (single-source rank work per I3 - gear drives vision RANGE instead).

at_compat.script quarantines the one foreign-mod coupling. G.A.M.M.A.'s grok_bo recomputes shit.power from the weapon and self-applies the damage, discarding the resolver's DAMAGE_RESIST
scale on player-to-NPC hits - only that cell. The wrapper installs at actor_on_first_update, after every other mod's wrap of grok_bo is in place, so it captures the FINAL chained handler
deterministically. It swaps the handler through Unregister and Register, because reassigning the module field never enters the call path.
It forwards ALL the arguments, because a chained handler reads flags.ret_value and dropping it is a crash.
It measures the health delta grok_bo dealt and heals back the resisted fraction (at_compat.script:13-32).
Inert without grok_bo. ADB needs no shim - it reads shit.power, and at_ loads before grok_ by name.

### xcombat boundary

The boundary rule is stated in the Integration model. The primitives cover weapon state, aim, movement, cover and clear-shot search, line-of-fire and memory reads, arrival, the cover reservation,
and the enemy-state reads. The full primitive surface is xlibs' own architecture doc.

One deliberate future exception would live outside xcombat: a sniper-reach extension forcing is_enemy true past the engine's enemy-distance gate. It is NOT built.
If added, AT would register and own it directly on the on_enemy_eval seam, because xcombat stays stateless by design - it holds no live-event callback and no ownership table on its own behalf.

### GOAP graft

The takeover control point: while the gate is up, the solver can finish only through AT's one action.

```
xcombat.register_takeover(npc, spec)    per stalker at net_spawn (at_maneuvers._register_graft); spec = { gate, on_begin, on_release }
  evaluator at id 188347                polls spec.gate - one flag read per plan solve
  action at id 188347                   initialize runs the one-time writes and spec.on_begin; execute stays empty; finalize runs spec.on_release
  the block                             every blocked-list action gains the precondition 188347 == false

gate down    188347 false    the vanilla chain solves as always; the graft sits dormant
seize        188347 true     every blocked action is unselectable; the graft action is the only path to the goal; its initialize starts the maneuver
release      188347 false    vanilla resumes from wherever the NPC stands
```

- The action's own writes, once per seize at initialize (xcombat.script:771-782): clear_animations, set_desired_position and direction, set_path_type(level_path), set_mental_state(anim.danger),
  register_in_combat - buying back the squad memory-sharing a planner block loses - then spec.on_begin.
- Grafting every stalker at spawn rather than only seized ones is deliberate: a maneuver begins the instant a need fires, with no per-seize wiring. The one exception is a companion while
  combat_ignore_companions is on - the seize gate already excludes companions, so a graft would only park a dead evaluator, action, and block precondition on his action manager.
  A companion recruited after spawn keeps his graft, harmless since the reserved id no longer clashes.
- The block list, in full (xcombat.get_blocked_planners, xcombat.script:712-732): the combat planner, the danger planner, alife, xr_danger, state_mgr+1 and +2, the monolith, zombied,
  and camper sub-scheme actions, axr_fight_from_cover, the smartcover action, both xrs_facer actions, xrs_kill_wounded, rx_ff.
- The block set is per NPC, and that is a crash constraint. Adding the same world-property condition twice to one action THROWs (condition_state_inline.h:44-53), so a global
  already-blocked flag cannot exist. apply_takeover_block re-runs at every seize to catch an action bound by a later configure_schemes (a gulag job change, xr_logic.script:279-295).
- The id is RESERVED. 188347 replaced 188200 after that value collided with an external companion scheme's evaluator - the graft sits even on never-seized companions, so the collision
  silently starved that scheme. A live [CMB] escape WARN fires once per maneuver when a foreign operator holds the slot under an open maneuver.
- The graft is permanent and single-consumer. register_takeover asserts on a second differing spec. release_takeover only clears the install tracking, because a graft cannot be unwired
  from a live action manager - a respawn builds a fresh one (xcombat.script:800-812, 855-870).
- AT never writes a combat planner property, InCover above all. The engine auto-resets InCover on every best-cover re-pick (stalker_combat_planner.cpp:58-64), so a script write dies within
  a frame and the NPC fights its own cover cycle. AT does GROSS placement and releases. Vanilla's take_cover runs the cover micro-cycle from wherever the NPC was left.
- The rejected alternative, kept as a guard: rx-style injection inside the combat sub-planner (cast_planner, rx_combat.script:327-353) preserves CStalkerCombatPlanner::update's side
  effects (react_on_grenades, react_on_member_death, stalker_combat_planner.cpp:104-105) but arbitrates against whatever a modpack grafts in the same planner.
  The top-level block wins for the GAMMA audience at the cost of suppressing those reactions for the seconds a maneuver holds - which is why a transaction stays narrow and brief.
- Layer arbitration. Maneuvers outrank behaviors: the block list holds vanilla ids only, so a future Behaviors graft enforces the rule with one condition - its evaluator returns false
  while the gate is up. Maneuvers and Commitment are mutually exclusive per NPC by construction. A blocked planner never switches, so the action-switch veto never fires for a seized NPC,
  and Commitment cannot police takeover quality - a takeover's fire discipline belongs at the maneuver's own decision points.

## Combat

Four systems. Maneuvers imposes (block vanilla briefly, run our behavior). Commitment, Conduct, and Push and Pull compose (leave vanilla running, deny or bend single decisions).
Shared scope rules, stated once:

- The gate. The maneuvers, the Push, and the Pull open only inside gate_radius_m of the player (at_core.is_in_gate).
  The gate is a performance concession. A maneuver far from the player would be correct, only unobserved, so the radius bounds cost and leaves the meaning intact.
  Counterflank keeps vs_actor row data and walks only when the actor is NOT the committed fight.
- Mutant-enemy fights are excluded before the catalog walk (IsStalker on the selection, at_maneuvers.script:648, the standing maneuvers-vs-mutants ruling).
  Push and Pull open only on a human target - the actor or a stalker, two plain compares. Fights past the gate stay vanilla.
- Zombied NPCs carry 0 on every participation key (Faction flavor), so no maneuver, no veto, no press reaches them.
  A weaponless enemy reads as the rifle range band where a row needs the enemy's weapon.
- No system walks NPC pairs assessing each other. Every scan reads the NPC's own record. Events exist only to stamp true transients (hits, reload edges, both written by at_core.script).

### Maneuvers

- Purpose: launch committed behaviors vanilla lacks or must be forced into - counterflank, reload_cover, flee, retreat, kite, pickoff. Vanilla owns every NPC by default.
  AT borrows one NPC for one committed, time-boxed maneuver against one stated problem, then releases.
  AT is an interrupt over vanilla. Vanilla stays the combat brain.
- Method: forced action through the GOAP graft (Substrate). Only the activation is per-seize. The graft is permanent.
- Seam: the graft gate plus apply_takeover_block at every seize. The reload events (npc_on_weapon_reload_start/_stop, PR #611 - the reload_cover row is inert without them, INACTIVE at boot).
  The movement hold uses xcombat.set_movement_hold (the glide-stop below), and the burst shape patches state_mgr_weapon.get_queue_params.
  The id registers in state_mgr.combat_action_ids (at_maneuvers.script:901-903) so the state machinery's idle evaluator never unwinds a maneuver's state.
- State/Cost: the maneuver fields on the combat record (gate, maneuver, dest, enemy_id, trigger, the stall and repeat trackers, cover_lvid, move_hold).
  Begin check 600ms, update and end checks 200ms each, all inside the one monitor pass.
  Cost per begin check is bounded by construction. It spends at most one lap of need compares plus at most one geometry probe.

Mechanism, the begin decision (at_maneuvers.script:636-656). Fighting = a live best_enemy or a hit within hit_fight_ms. Not fighting resets the cursor and the trackers.
The enemy must be the actor or a stalker. _check_seize_block bails on an unseizable body or an exhausted budget before any walk.
_can_seize (at_maneuvers.script:79-85) takes an NPC only if armed (an unarmed NPC would deadlock - the engine's own rearm lives in the blocked combat planner),
outside any smart cover (vanilla owns that micro), and free of a playing animation (xcombat.is_body_busy: the crit stagger, a script overlay,
the additive flinch where the exe exposes it - a seize under a playing reaction is the glide by construction).

The catalog walk (_resolve_maneuver, at_maneuvers.script:447-484). Rows in priority order - counterflank, reload_cover, flee, retreat, kite, pickoff - from a per-NPC cursor.
Per row, in order: scope (the gate, or the fight class for vs_actor), toggle, flavor (at_faction.has_flavor, consulted before the need - the flavor decides whether the behavior triggers at all),
check_need, palette, find_destination (at_maneuvers.script:388-402). A row that fails scope, toggle, flavor, or need falls through to the NEXT row in the SAME check.
The first row whose need holds runs its find_destination - the one geometry probe this check - and ends the walk, pick or decline.
Both move the cursor past the row: a declined row defers to the next candidate and retries after at most one lap,
and a picked row hands the NEXT decision to the row below it - a cowardly NPC whose rout was blocked gets his retreat fallback with no escalation state.
The cursor resets to the top when the NPC leaves the fight. Values two rows share (faction, weapon kinds, positions, the threat set) memoize lazily on the walk, for that walk only (_reset_memo).
With debug on, the decision line carries every examined row's stage plus the whole-walk microseconds,
and it prints only when some row got past its need or a row picked - the all-quiet walks stay silent.

Every row is two methods, split so the need is always cheap and the geometry is always bounded.
check_need is a compare over memoized reads that states the row's FULL problem and returns the situation name - actor_close, reloading, hurt, too_close, stalled - or nil.
The raycast, path, and search class of work is forbidden in it.
find_destination owns the geometry and the premise reads that cost luabind, and returns the destination vertex or nil to decline - one pass answers "can he?" and "where to?",
and the vertex it validated is the vertex the engine executes, resolved a single time at the decision.

The threat set (at_maneuvers.script:130-170).
A decision's candidates are the union of my selection (best_enemy_id), everyone holding me as best enemy (at_core.get_threats), and the player while hostile to me (xcreature.is_actor_enemy).
The union is candidates only - _find_closest_threat verifies every predicate live at the decision,
so a stale index entry costs one failed compare - and the closest qualifying member is declared by the row (_set_staged_threat) and locked by the shell as the committed target.

The seize (_try_seize, at_maneuvers.script:615-634). The committed target resolves as the actor for a vs_actor row, else the staged threat, else the selection.
Its budget bucket (vs_player or vs_npc) must admit the start. A refused pick releases its claimed cover and traces [CMB] limited.
Admission re-applies the takeover block (a later-bound scheme action gets its precondition), raises the gate, and stamps the row, destination, and target on the record.

From staging on the maneuver is COMMITTED to that target: _start_maneuver and _update_maneuver resolve the staged id via xcombat.resolve_enemy and never re-read best_enemy,
so the LOOK never re-targets mid-maneuver to whoever the brain glanced at, and a dead or despawned target ends the maneuver (target_lost) with vanilla picking the next fight.
The staged target is what the NPC looks at and what every read and end condition resolves - sight, fire_make_sense, the check_end premises - for the maneuver's life.
Which enemy his SHOTS select remains CEnemyManager's own pick, which the takeover does not touch.

The start (the graft action's initialize -> _start_maneuver, at_maneuvers.script:559-575) resolves the target or stops as target_lost.
It sends the destination once (xcombat.set_destination - a substituted vertex traces as dest_substituted) and applies the row's state once
(_apply_state -> xcombat.set_combat { fire, posture, movement, enemy }). The engine walks the NPC there on its own.
The graft action's execute stays empty. An engine re-grant of the same open maneuver counts and traces as a reenter. It opens no new transaction.

The update check (200ms, at_maneuvers.script:672-702): re-apply the row's state (every row's update is _apply_state - a firing maneuver re-checks its shot, flee re-asserts the holster).
The re-apply SKIPS while the weapon reloads - re-applying mid-reload costs a weapon-pose transition,
and the READY degrade at reload start plus FIRE at reload end read as two visible dips per reload on small magazines - and pauses while a hit reaction plays (is_body_busy.
The [CMB] pause line keeps skipped passes visible).
Two WARN watchdogs run on the same check. escape fires when an unblocked action holds the slot under an open maneuver, so a denylist gap reports itself.
sight_lost fires on three consecutive samples serving another object's sight under a firing row, the committed-target sight regression.
It samples BEFORE the re-apply, because a post-apply read would only echo our own write.

The end check (200ms, at_maneuvers.script:704-727): wounded, the row's check_end premise, arrival (xcombat.is_arrived over path_completed) for ends_on arrival rows, or the cap (the row's timeout,
else MANEUVER_CAP_MS 8000 - unreachable today, kept so a future row that omits one caps at a sane bound and never reaches nil arithmetic). target_lost ends at once.
_stop_maneuver lowers the gate, releases the claimed cover, resets the stall tracker, and with debug on re-runs the row's own check_need fresh: need_cleared=n at hand-back is the unsolved signal.
Vanilla resumes from wherever the NPC stands - a mover's end position is sticky for free, a held mode reverts on release.
Aborts come free from the graft action's preconditions failing (alive, not wounded). A held NPC eats the rare grenade. AT re-implements none of vanilla's reactions.

The fire discipline lives in xcombat.set_combat - a row declares only its INTENT (FIRE, SNIPE, READY, STOW).
A reloading weapon degrades the intent to READY first, because a fire goal issued mid-reload CANCELS the engine reload (chained applications killed reloads, leaving NPCs racking empty guns),
so the discipline lets the reload finish and fire resumes on the next re-apply. A seen enemy is fired on with no further gate - no distance term, point-blank fires.
fire_make_sense's 2.5m bail is a smart-cover rule that must never gate a seen enemy.
Only the blind case consults fire_make_sense (its occlusion pick stops shooting the wall he ducked behind, its 10s automatic-weapon window sustains suppression at last-known),
else the intent degrades to READY, weapon up, eyes on the enemy, until sight returns.
A can_kill_enemy gate on the seen branch was tried and reverted on measured plus source evidence. The engine never gates fire on that read - its sole consumer is sight aim-point selection
(sight_action.cpp:408), and while walking the ray follows the head's current sight angles, which lag a strafing target - so the gate muted fire through the engine's normal lag-and-displace windows.
The read is an aim-quality question, valid on an NPC whose vanilla planner is aiming - which is why Commitment may use it and the takeover fire path never does.

The burst shape (_compute_queue_params, at_maneuvers.script:39-66).
Every state without an animation-specific entry falls to the generic {5,300,0} override (state_mgr.script:322) - full-auto on a pistol, a burst on a sniper rifle.
The patch on state_mgr_weapon.get_queue_params rolls burst size and pause fresh per query inside the engine planner's own per-weapon [fire_queue_params] medium band (xcombat.FIRE_QUEUE,
m_stalker.ltx:826-909) - the same per-burst variance the vanilla planner has - then multiplies by the Fire Discipline rank factors (at_reaction.get_queue_scales, the one owner of the tier tables.
Max(1) rounding keeps 1-round bursts alive). The pickoff row swaps the weapon band for its own single-shot band (pickoff_interval_min/max_ms, the engine's own sniper-band rhythm).
queue_w_* fixes a kind to constant values.
Animation-tuned states keep their values, NPCs outside a maneuver pass through byte-identical, and the patch's weapon-kind read adds no crash surface over the original,
which makes the same class of member calls on the same NPC here.

The glide-stop (at_maneuvers.script:729-752, 773-781) closes the one case the seize gates cannot reach, the ENGINE's own mover starting a standing NPC mid-hit-reaction.
is_body_busy defers AT's writes, but the vanilla planner is engine-side C++ and defers to nothing.
The engine half is one neutral lever, npc:set_movement_hold(bool), wrapped as xcombat.set_movement_hold. While true, parse_velocity_mask routes into its Stand branch - speed 0, movement type Stand,
path and destination preserved, so the NPC resumes his route on release.
Every decision is Lua-side. _check_hold holds during the crit stagger (xcombat.is_staggering - the engine's misleadingly named critically_wounded(), true only while the ~1s stagger anim plays,
and false in the wounded-down state), or during the additive flinch on a body already standing (npc:movement_type() == move.stand is the standing proxy - Lua cannot read NPC speed,
GetMovementSpeed is actor-only. A moving NPC is never held, freezing a runner mid-stride is the same artifact from the other side).
npc_on_hit_callback sets the hold the frame the hit lands - waiting for the next pass leaves up to 200ms of glide,
exactly the window - then the pass maintains it and releases the moment the reaction ends. A per-NPC mirror (state.move_hold) makes the write transition-only.
The engine flag persists while the NPC is online and has no decay, so every path that stops maintaining it clears it first: the release, death, net_destroy, unregister,
and the toggle-off (within one pass, so the switch can never strand a planted NPC). On an exe without the bind the wrapper no-ops and the vanilla glide is the fallback.
[CMB] hold traces each transition with its cause. Off debug the steady-state cost per NPC per pass is one table read and one subtraction.

The catalog (at_maneuvers_config.script - the rows reference the methods, which at_maneuvers binds at _load_config. Structure LTX cannot hold):

```
maneuver      fires on                                  applies to                                   runs to                          weapon; move             ends on
counterflank  actor_close: enemy actor inside 5m        any NPC fighting someone other than          holds its own spot, aimed        fire; still              3s hold
              while the committed target is farther     the actor                                    at the actor
reload_cover  reloading, a watcher has him in sight     any NPC                                      the nearest hiding cover         weapon up, no fire; run  arrival, reload done, or 8s
flee          hurt, enemy in reach, last man,           faction flavor - the flee-prone,             a friendly base 100m+ away,      holstered; run           arrival or 20s
              once the shooting pauses                  their first answer                           rear-biased
retreat       hurt, enemy in reach                      faction flavor - the flee-prone's fallback   cover behind him, never closer   fire; walk               arrival or 8s
                                                                                                     to the enemy (reserved)
kite          too_close: nearest threat inside          universal - a gun inside its minimum         a clear back-lane, a weapon-set  fire; walk               arrival or 8s
              MY weapon's minimum                       is half useless whatever the enemy holds     distance to the rear
pickoff       stalled ~4s, comfortably past the         faction flavor - the disciplined more        holds its own spot               deliberate single        8s hold or broken premise
              enemy's EFFECTIVE range, unbothered                                                                                     shots; still
```

Per row, the mechanism the table cannot carry:

- counterflank answers the actor-party hold: an enemy actor inside counterflank_actor_dist_m (5) while the NPC's committed target is FARTHER - he is shooting past the man who can kill him first,
  the enemy manager's seen-now scoring artifact (a seen distant target outranks the unseen man on his shoulder, enemy_manager.cpp:110-175).
  The row stages the ACTOR and aims at him for a 3s hold with no movement.
  The turn makes him SEEN, and seen at contact range wins the engine's own selection outright,
  so at hand-back best_enemy IS the actor and vanilla drives the new fight - the transaction's success is the engine changing its mind.
  No feasibility geometry: the old wall raycast was structurally blind across the whole 5m trigger radius (its clearances cannot report an obstacle below ~3.75m) and is removed.
  The need is pure math over the walk memo plus one relation read paid only inside the radius.
  It only fires when the NPC is already fighting someone else, so player stealth against idle NPCs is untouched.
- reload_cover answers the vulnerable window (the PR #611 consumer): a stalker caught reloading in a watching threat's line runs to nearby concealment while the engine finishes the reload on its own
  timer. The need is a pure-Lua read of the reload stamp, stale past reload_stale_ms.
  Feasibility re-confirms with the authoritative xcombat.is_reloading (get_state() == 7 - eReload is 7. An older trace compared 5, which is eFire),
  requires the exposure self-check - a threat-set member has him in sight (xcombat.is_in_sight at reload_cover_cone_deg: facing within the cone plus a clear shot line.
  One facing read and one raycast per candidate, paid only at reload feasibility.
  An in-hands weapon term was removed - what the viewer holds never discriminates in combat) - and the cover must hide from THAT member (xcombat.find_cover selection nearest,
  firing false - the nearest reachable cover that hides him. The best-hidden one anywhere in the radius loses to reach). Reloading unobserved declines and vanilla reloads in place.
  A taken cover claims its vertex. Its check_end hands back the moment the reload finishes - the gun is up again and vanilla's own fire beats finishing our walk at weapon-up.
  It sits directly under counterflank because a reloading NPC under fire is at his most vulnerable, outranking the pressure rows.
- retreat answers standing-line pressure: badly hurt (health below hurt_frac) with a threat inside his weapon's reach, pull back to cover BEHIND you while still firing.
  The search centers retreat_rear_m (8) to the rear - deeper than the 7m cover-search radius,
  so the circle excludes the cover the NPC already holds - and the winning vertex must be farther from the enemy than the NPC stands (toward-the-shooter cover declines),
  so a retreat always grows the distance. The vertex is reserved (xcombat.register_cover) so two NPCs never pick the same spot.
- flee is the rout, and a rout is for the broken: hurt with the threat in reach, only the last man (xsquad.is_last_man - squadless or sole survivor),
  and only once the shooting pauses - a fresh hit or a perceived shot (xcombat.is_under_fire, his own danger memory) declines it, so nobody turns his back mid-burst.
  The declined row defers and is re-asked a lap later, so the rout fires when the fire lifts.
  He runs HOLSTERED to a friendly base with no enemy squad stationed, at least flee_base_min_dist_m (100) away,
  rear-biased so every stride gains distance (xsmart.find_friendly_base scoped to the actor's level - valid because a fleeing NPC is online, and online NPCs are on the actor's level by construction).
  No reachable base declines.
  flee sits above retreat in the catalog because a coward runs before he fights. A hurt flee-prone NPC asks the rout FIRST,
  and a blocked flee moves the cursor to retreat - he fights from cover only when he cannot run.
- kite answers the enemy inside your minimum: back off a weapon-set distance (kite_distance_m_*: shotgun 4, pistol 5, SMG 6, rifle 8, sniper 15),
  still firing the whole way - a visible fighting withdrawal.
  xcombat.find_flee_lane runs a two-phase search - straight-back then +-45 then +-90 at full distance, the same fan at half distance,
  then the longest clear raycast-validated stub - and declines only when every phase is boxed in.
  At 0m separation the lane direction falls back to the OPPOSITE of the NPC's facing - a fighting NPC faces his enemy, and the old raw-facing fallback kited INTO him.
  The destination is accepted only if standing on it puts the enemy back OUTSIDE the weapon's minimum (measured at decision time),
  so arriving negates too_close by construction and the arrived-but-nothing-moved ping-pong cannot be accepted.
- pickoff is a deliberate-fire hold, not a movement: a stalker who has his enemy outranged in a standoff (held position ~4s since the last maneuver, the enemy not closing,
  inside MY weapon's effective range) plants and picks him off with single aimed shots.
  The range term reads EFFECTIVE range, the distance where the enemy's weapon is still genuinely dangerous, so the advantage is statistical and the break-offs stay strict,
  and the break-offs stay strict.
  Range hysteresis, two thresholds: he plants only comfortably past the enemy's effective range (pickoff_enter_factor,
  1.2x) and the hold ends when the enemy closes back inside 1.0x (pickoff_exit_sqr) - between the two nothing flaps.
  Under fire or in the actor's sight (xcombat.is_in_sight at pickoff_targeted_deg - facing plus clear line, so a crosshair through a wall never counts) it declines,
  and the same premises re-check while the hold runs (check_end), ending the hold the moment any of them trips - well before the cap.
  His fire is 1 round per pull at a deliberate pause rolled fresh each shot.
  His accuracy is the rank dispersion curve from a still stance - no engine cheat mode (the sniper_fire_mode story-scene flag is deliberately unused, npc-combat-effectiveness.md).
  A finished hold resets the stall tracker, so the next plant needs a fresh 4s standoff - vanilla owns the gap.
  need_recheck is off for pickoff. Its need is the stall measurement its own end resets, so a hand-back recheck would always read cleared.

Choices, one decision per line:

- The transaction law: a row's need states the FULL problem and the maneuver's success negates it.
  Kite's back-off ends outside its own minimum, flee's base is past the enemy's reach by construction, counterflank flips the engine's own selection, a finished pickoff resets the stall measurement.
  A solved problem reads false at the next begin check. A recurred or unsolved one legitimately re-fires - three kites under sustained pressure are three correct transactions.
- retreat is the one row whose need is a STANDING condition: an ~8m pull-back cannot exit a rifle's reach term, so a still-hurt NPC under long-reach pressure legitimately re-fires it lap after lap.
  That is pressure relief re-applied while the pressure lasts, throttled by the walk cadence and the repeat limit, its only throttles.
- There is no reseize cooldown. The trigger is the throttle, and no timer stands between a real problem and its answer.
- Displacement is the discriminator, never timing.
  The repeat limit (_check_repeat_limit, at_maneuvers.script:429-445): a pick of the same situation with the NPC still within 2m of the previous pick means the last transaction changed nothing,
  and past repeat_limit (3) such picks the takeover refuses and vanilla owns him.
  A legitimate chain under a sprinting player re-fires instantly too, but each transaction MOVED him, which resets the count - the same displacement-sampling pattern as the stall tracker.
- The advantage rules: a need says a situation exists. It does not say the maneuver pays.
  Each row's enemy and squad terms (distance, the enemy's weapon, the actor's aim, last man, a standing line) live in find_destination or, where the rule is pure math,
  in the need - so they cost nothing on the monitor, and each traces its pass or decline (the can_* lines). Later rows (suppress, assault, flank) are held to the same bar at design time.
- The things vanilla does well - the opener, re-target, search, turret, grenade dodge - are not situations AT answers.
  The grenade trigger was removed because vanilla's own grenade reaction is faster than a staged walk-to-cover, and a seized NPC eating a rare grenade was already the accepted trade.
- There is no indoor gate: the old surge-shelter-radius is_indoor proxy false-flagged open ground, so the takeover fights everywhere.
  xcombat.is_indoor was rebuilt as a real roof-plus-walls raycast and stays available if a future row needs it.
- A per-NPC decline BACKOFF was built and removed (commit 95f0717): stamping each row's decline complicated the arbitration against a cost nobody had measured.
  Re-add one only if the walk-microsecond traces show a standing decline burning real frame time. Take the implementation from that commit as-is.
- There is no post-hand-back ignore window. The earlier flee_enemy hold existed only to make a failed escape look like a clean one.
  A flee that capped in place hands back next to the enemy and fights - and since the need still holds, he attempts escape again when the gates allow: the correct read of a cornered man.
  On hand-back the engine decides: against a monster the engine's own max_ignore_distance (75m, m_stalker.ltx:415, applied in CEnemyManager::useful for the stalker-vs-monster clause only,
  enemy_manager.cpp:76-82). Against a stalker the script-side 100m cutoff in whichever xr_combat_ignore.script won the MO2 slot applies (vanilla :245).
  Against the ACTOR no unconditional distance rule exists (vanilla limits actor fights to 100m only at night or in rain,
  xr_combat_ignore.script:229-231) - a daytime flee from the player relies on the NPC having run holstered and blind, so the memory decays unrefreshed.
  A flee that reached its 100m+ base clears the first two gates outright.
- The sight-glue defect is FIXED in xcombat, not policed here.
  CSightManager is a single slot, last-writer-wins, no priority, no expiry (sight_manager.cpp:233-242), and the look order travelled only through the state machinery's direction_turn,
  whose preconditions (state_mgr_goap.script:468-481) can hang for a whole hold - a mid-hold reload suffices - leaving the direction gate (state_mgr.script:318) to withhold every shot.
  xcombat.set_combat now sets the committed target's sight on every firing apply (_set_committed_sight), mirroring look_at_object's own branch so the two writers dedup,
  and stamping point_obj_dir so the fire gate releases even when direction_turn never runs. The sight_lost watchdog guards the fix as a regression detector.
  Turning speed and geometry were exonerated. Do not re-suspect them. Stand-in-danger body turn runs at a full rotation per second,
  and select_speed never slows large angles (sight_manager.cpp:80-100).
- If a cover readout is ever re-instrumented on the decide path, take it at DECIDE time, before any seize: under a takeover block the planner's cover flags can only decay,
  so the pre-seize value is the truthful one.
- Flee's update is a BLOCK - keep the weapon down - not a sight re-drive.
  The takeover block stops kill_enemy from aiming, but set once, the weapon comes back up and the NPC re-aims (the observed bug),
  so flee re-asserts the HOLSTERED sprint state (weapon strapped = physically cannot aim or fire) on the update check, throttled to its 200ms cadence.
  Demonized re-applies the same state every frame (demonized_stalker_aoe_panic.script:327), so the throttled form is amply fresh.
  A holstered weapon with no target faces the run path on its own. AT never steers the sight.
  This is the demonized panic mechanism: block the same planners, re-assert sprint, route far away.
- The continuous script_combat_type scheme (GAMMA AI Rework, ReDone Combat AI) is the rejected alternative, and a read of both confirmed why: it owns an NPC's whole combat single-ownedly,
  and either does less than vanilla (GAMMA's thin camper sets one state) or reimplements it worse (ReDone's fat get_combat_movement and global-cvar aim).
  The intermittent takeover borrows an NPC where vanilla is weak and hands back. Vanilla's own aim, fire discipline, cover cycle, and squad coordination run the rest of the time.
- hit_fight_ms exists in BOTH the maneuvers and behaviors config files deliberately: the fight-entry window on the far-shooter case for the walk,
  the pull's incoming-pressure window for at_behaviors - one number today, two independently tunable meanings.

### Commitment

- Purpose: hold a stalker on a winning combat action so it keeps firing, rather than break contact and shuffle toward fresh cover while it is winning the shooting.
  It never seizes and never launches a behavior. It denies vanilla's own bad switches and leaves everything else running.
- Method: engine-hook veto. Two seams, each a callback that fires before the engine commits a decision, and AT answers deny by setting flags.allow = false.
- Seam: the action-switch veto (npc_on_combat_action_switch, PR #595, via xcombat.on_action_switch - INACTIVE without it, the toggle inert) and the cover re-pick veto
  (npc_on_best_cover_repick, PR #607, via xcombat.on_cover_repick - the cover-pin seam logs INACTIVE and the shuffle holds at the switch seam only).
- State/Cost: per-id hold records (_hold, _cover_hold) plus the shadowed current action (_cur_op). No loop. Every read runs inside the callback the engine already fires.

Mechanism, the switch veto (_on_action_switch, at_commitment.script:154-193). The deny rule holds three transitions, each proven from the planner preconditions.
Two are fire-action exits - kill_enemy or kill_if_not_visible toward take_cover or get_ready_to_kill.
A fire exit holds while the NPC still SEES its enemy (visible_now, the exact bit kill_enemy fires on, stalker_combat_actions.cpp:483) and no teammate blocks the lane (can_kill_member).
The -> take_cover hold carries one extra gate, fire_make_sense (ai_stalker_fire.cpp:894), so it holds only when the firing lane is clear to the enemy.
An NPC pinned to shoot its own cover is worse off than one that repositions.
The third hold is the flank - detour_enemy -> take_cover - held only while the NPC is BLIND (detour runs at SeeEnemy = false by precondition) and released the instant the enemy is re-seen.
All the rest passes untouched, and each condition falsifies itself, so release needs no machinery.

The cap (commitment_hold_s, _hold_cap_ms) is a TIME-TO-LIVE on the refusal. The veto is a bare if in the callback that returns allow = false.
It keeps returning false for every proposed cover-seeking switch while the advantage holds.
The cap only bounds how long ONE continuous refusal may last before the veto relents and lets a cover move through, so cover quality matters again eventually.
A hold ends the instant any condition falsifies - sight lost, a teammate crossing the lane, the engine stops proposing the switch - usually well under the cap.
An ALLOWED switch closes the refusal window so the next hold starts a fresh cap.

The LIQUIDATE deferral (_try_extend_hold, at_commitment.script:83-94). At the relent moment, and only there so the deny stream under the base cap pays no extra read,
the veto consults at_core.is_enemy_vulnerable. While the enemy cannot return fire the relent defers, bounded by VULNERABLE_EXTEND_MS (4000ms) past the cap.
Both seams carry the deferral, each tracing it once per hold, and on the debug HUD the Commitment token becomes LIQUIDATE while a deferral holds. Its own toggle is commitment_liquidate.

The cover re-pick veto (_on_cover_repick, at_commitment.script:236-269). The kill_enemy -> take_cover shuffle STARTS at a best-cover re-pick,
which clears InCover before the planner proposes the exit the switch veto refuses. kill_enemy requires InCover=true (stalker_combat_planner.cpp:344), so losing it forces the exit.
The re-pick callback fires BEFORE that reset, so keeping the held cover means InCover never drops and the exit is never proposed. The veto denies the cause upstream of the symptom.
It denies under the SAME advantage gate as the fire-exit hold and only while the NPC's current combat action is a fire action.
The callback hands over only (npc, flags), so the current action is shadowed per NPC from npc_on_combat_action_changed (_cur_op).
The engine reports no initial action (stalker_combat_planner.cpp:107 fires only past the first update on current != previous), so the shadow is nil for a fresh NPC and its re-picks pass.
A continuous cover refusal is bounded by the same relent valve (_cover_hold, closed by any actual cover change via npc_on_best_cover_changed).
The cap is needed because two re-pick drivers are standing conditions, each re-invalidating on every action execute while denied.
They are an enemy inside 3m of the held cover (MIN_SUITABLE_ENEMY_DISTANCE, ai_stalker_cover.cpp:237) and a smart cover whose loophole is gone.
A maneuver-seized NPC never reaches this seam - its blocked planner runs no combat action.
One engine path deliberately bypasses the veto. The actuality check's advance search (ai_stalker_cover.cpp:278) swaps the held pointer toward nearer cover without firing on_best_cover_changed.
It resets no props, so it is not a shuffle driver. The veto's guarantee is that no props reset without consent, and the held pointer may still move.

The watched set, from a real transition histogram. From-state fires is the load-bearing column, because only an action whose execute calls fire() is worth holding.

```
transition                  from-state fires   decision
take_cover -> kill_enemy    arriving to fire   pass - this is the return to fire
get_ready -> take_cover     no (InCover false) never veto - blocking it deadlocks the NPC
kill_enemy -> get_ready     yes (leaving fire) VETO - the prize; reload in place beats the detour
kill_enemy -> take_cover    yes (leaving fire) VETO - InCover drop while seeing
detour -> take_cover        blind flank        VETO while blind, release on re-sight
lost-sight and survival     from a non-fire op pass - the sees gate declines them
```

Choices, one decision per line:

- Catch the shuffle one hop UPSTREAM, at the fire action's own exit (kill_enemy -> get_ready), never at the fat get_ready -> take_cover edge. get_ready_to_kill sets InCover false and
  take_cover is the ONLY action that restores it, so vetoing that edge strands a non-firing NPC that can never satisfy kill_enemy. Caught upstream, get_ready is never entered.
- can_kill_enemy is never a gate. It raycasts along the gun's CURRENT aim direction (ai_stalker_fire.cpp:811), so it reads false through the aim-lag of a moving target (the mute-fire finding).
  It survives only as a debug read on the hold line. The hold gates on the same signal the engine fires on (sees plus can_kill_member for the lane).
- Set the cap high and block unconditionally and the NPC holds his firing spot forever. The timer is a relent valve, never the hold length.
- A recent-hit standdown was tried and removed. "Any hit in the last 3s permits leaving" kept the veto disabled in the one fight it exists for, the one the player is shooting in.
- Commitment cannot police a takeover. A blocked planner never proposes a switch, so the veto never fires for a seized NPC (the Layer arbitration fact under GOAP graft).
- The evidence traces register on every exe, because the shuffle they measure exists without the veto seam. The decision-point census logs every proposed switch with its situational reads
  as DATA that never gates, before the enabled gate, so a run with Commitment OFF measures the unhelped stream. Its under-fire read is a landed-hit stamp, because is_under_fire is dead in
  combat - the danger manager ignores the selected enemy's hit and sound dangers (danger_manager.cpp:350).

### Conduct

- Purpose: two small habits on VANILLA-driven NPCs at moments the engine already decides. Cover posture (crouch or stand at a firing line) and weapon spacing (the cover-distance band by
  weapon and rank). No takeover, no held actions.
- Method: callback adjust. Each habit answers one per-event value inside an engine callback the vanilla planner fires.
- Seam: posture on npc_on_combat_set_body_state (the COMBAT_BODY_STATE_OVERRIDE forwarder). Spacing on npc_on_get_min_combat_dist and npc_on_get_max_combat_dist (#563 forwarders, via
  xcombat.on_get_min_combat_dist and on_get_max_combat_dist - INACTIVE on floor exes, the engine bands apply). A maneuver-held NPC never reaches either - its planner is blocked.
- State/Cost: the held posture decision (_decision), the eligibility cache (_eligible), the TTL spacing row (_spacing), plus the push and pull overlays.
  No loop. Each answer runs inside the engine callback.

Mechanism, posture (_on_combat_body_state, at_conduct.script:98-135). It overrides the posture for long-weapon carriers (w_rifle, w_sniper - a shotgunner's fight is movement,
so short weapons keep vanilla's pick) of experienced tier and up, on hold_position ONLY - the one op whose ask is provably stationary (stalker_combat_actions.cpp:843).
The decision is HELD, re-decided once per DECIDE_MS (3s) by casting a crouch-eye shot ray to the enemy (xcombat.has_shot_obstacle at CROUCH_EYE).
Crouch only when that line is CLEAR so the low stance steadies the aim, stand when a low wall would eat his own shot.
Past a flat CONDUCT_FLOOR_M (20m) only, so a close fight stays standing and mobile.
A per-NPC disposition roll (the conduct_crouch faction flavor, cached once per life) crouches only some eligible stalkers, so a firing line mixes standing and crouched shooters.
While a hit reaction plays (is_body_busy) the posture FREEZES, holding the last decision. Zombied are forced to STAND through the same held path.

Mechanism, weapon spacing (_on_min_dist and _on_max_dist, at_conduct.script:170-204). A standing per-NPC cover-distance band from weapon and rank,
answered through the engine's own combat-distance asks (compute_enemy_distances, ai_stalker_cover.cpp:91).
Every handler SCALES the handed base and never replaces it, so the engine's own weapon-type semantics survive underneath.
The fork already places shotgun and pistol and sniper bands through its cvars, and the disposition adds the two reads it lacks - the SMG (tightened x0.6) and the sniper minimum raised by rank.
Rows derive lazily per NPC with a 5s TTL, so a mid-life weapon change re-keys itself.
The same handler answers the Push window's per-NPC max override and the Pull band as separate overlay entries.
A cleared overlay restores the band and leaves the disposition beneath untouched, because the disposition row is never overwritten.

Choices, one decision per line:

- Posture answers hold_position ONLY, from an audit of every ask site. take_cover and get_ready ask while walking to position, and look_out asks once then MOVES, so answering crouch in any of
  those slows the engine's own move - never slow a mover, user-ruled.
- The decision is HELD, not per-ask. The engine re-asks its combat body state several times a second, and the crouch-shot line flips clear and blocked second to second, so a stateless per-ask
  decision flapped crouch and stand continuously (the up-down shuffle with T-pose and glide artifacts).
- Both directions override vanilla's blind pick. Vanilla crouched behind a random bump without checking geometry (the at_stance bug, NPCs fired into the bump) and stood tall in the open where
  a crouch would steady the aim, wasting the largest legitimate accuracy gain the engine has (stillness plus crouch, m_stalker.ltx:476).
- The floor is flat, not a fraction of weapon range. A fraction sent the sniper floor to ~45m, exactly where a sniper benefits from crouch most - the inversion.
  The mobility boundary is the same absolute distance for every weapon.
- Spacing SCALES, never replaces. The engine owns the weapon-type semantics, and the disposition only adds the two reads the fork lacks.

### Push and Pull

- Purpose: press a stalker whose committed target cannot answer (Push), and fall back a stalker caught in his own weak moment while his target is strong (Pull).
  Two effectors on vanilla-driven NPCs, no takeover.
- Method: Push fire is a per-NPC bind (set_fire_queue_scale). Push band and Pull band are callback adjusts through the Conduct spacing handler.
  The press causes are POLLED from the target, never evented.
- Seam: no engine seam of its own. at_behaviors subscribes to nothing. at_core.update_npc calls at_behaviors.update_npc per record per pass, one idempotent call that applies and restores.
- State/Cost: the per-NPC mirrors on the combat record (push_fire, push_band, push_cause, push_at, push_until, pull, pull_at, pull_until). Every state is on the record or one poll away.

Mechanism, Push (_update_push_npc, at_behaviors.script:171-196). Opens only inside the gate and only on a human target.
The press cause is polled from the target - at_core.is_enemy_vulnerable under push_reload, or the standing weak condition (get_health_frac <= push_weak_frac, clearing past a hysteresis margin)
under push_finisher. Two effectors run on the target. The FIRE bump is range-gated by the burst-tail law (inside push_close_m the burst grows and the pause shrinks,
out to push_far_m only the pause shrinks). The BAND bump (set_push_max) is target-facing advantage gated - the finisher self-check, the target bleeding,
or the target's back turned past push_back_deg. The press is per attacker, no shared window object.
His first write against a target spends one start from the shared budget, caps at push_window_max_ms, and cools down push_press_cooldown_ms on his own record.
A press whose cause clears restores every write within one pass. Snipers are fire-only.
The restore re-applies the rank values through at_reaction.get_queue_scales, so the queue returns to its tier value. A raw 1.0 would wipe the rank curve.

Mechanism, Pull (_try_open_pull and _update_pull_npc, at_behaviors.script:219-260). It runs the same scan from the victim's side.
My own record shows me reloading, or hurt with a hit within hit_fight_ms and my health at or below pull_weak_frac.
My target is strong (get_health_frac >= pull_strong_frac), my faction rolls me in, the budget admits - then my accepted cover minimum rises
(set_pull_band, pull_band_k x my current distance, floored and capped) so my own re-picks land farther, held at least pull_hold_min_ms, cooled down pull_cooldown_ms after it clears.

Choices, one decision per line:

- Scan-only, no windows. Events exist only to stamp true transients (hits, reload edges, both written by at_core), and every decision scans the target or the record. A press has no shared
  window object, so per-tactical-reload re-presses stay per attacker and never make the press continuous.
- A seized NPC never presses or pulls. state.maneuver is the first check of both scans, and a seize clears his writes with the cooldown started, so the maneuver alone answers the moment.
- Priority is one-directional. Maneuver decisions never read press or pull state. Pull outranks push per NPC, both at_conduct handlers apply the pull overlay after the push cap, so
  self-preservation wins over aggression.
- The same vulnerable moment feeds three verbs with no conflict, all reading at_core.is_enemy_vulnerable - Commitment's liquidate holds fire on it, the press thickens fire into it, a future
  maneuver may move on it.
- Every write follows the stale-lever discipline: transition-only against the per-NPC mirrors, cleared on cause end, toggle-off, seize, death, despawn, unregister.
- The fire-shape vocabulary keeps one owner per shape so no future row rebuilds a fire pattern on its own.
  DUMP is full-auto volume, legitimate only at point-blank. PUSH is the press's range-gated bump.
  MEASURED is the suppress shape, a steady constant cadence at a known position (the future squad base-of-fire row).

## Effectiveness

Five per-NPC skill layers on vanilla-driven fire and perception. Each reaches every NPC shot regardless of what drives it.
A maneuver changes who drives movement and fire intent, and it never exempts the NPC from the skill model.
Reaction, Vision, and Discipline share one file (at_reaction.script) - Vision's mechanism lives under Perception.

### Accuracy

- Purpose: rank-aware NPC dispersion in script, because the engine rank curve degenerates on Anomaly gamedata.
- Method: two callback adjusts on the per-shot dispersion. The flat rank curve registers as a resolver provider, the moving-fire curve writes on the shot seam directly.
- Seam: npc_shot_dispersion (declared axr_main.script:126, dispatched from _g.CAI_Stalker__GetWeaponAccuracy at _g.script:1213-1217), fired per bullet from the engine weapon-accuracy calc.
- State/Cost: no record state. Per shot it does a rank-name lookup, a move_type compare, and pure-Lua scaling of the dispersion the callback hands over.

Mechanism. out = base * disp * move. disp is the flat rank curve applied to every shot, registered into the effects resolver as SHOT_DISPERSION (multi-source, min-combined with thermal).
move is the moving-fire curve applied only while the shooter's move_type is walk or run (ai_stalker_fire.cpp:81-104).
It writes directly on the shot seam, because it is single-source AND keyed on the per-shot move_type a resolver provider never receives
(walk = 0, run = 1, stand = 2, ai_monster_space.h:31-33, the move_type <= run gate). Both gate on the accuracy faction key.

Choices, one decision per line:

- Script, not cvars, because the engine rank curve is a dead knob. Rank() clamps to [0,100] (ai_stalker.cpp:764) but Anomaly rank intervals run to 26999 (game_relations.ltx:8),
  so every NPC lands at rank_k = 1.0 and m_fRankDisperison collapses to the constant dispersion_experienced_k = 0.8. Scaling the novice config knob changes nothing.
- The 16 per-tier values live in the at_mcm defaults table (the ONE config source).
  The script's _disp and _move tables are deliberately empty and fill at refresh, so no LTX or script copy can drift.
- disp registers into the resolver (min with thermal). move writes directly, because it is single-source AND keyed on the per-shot move_type a resolver provider never sees.

### Reaction

- Purpose: per-NPC rank-tiered aim tracking speed, tracking lock, and target lead. It shapes how tightly a rank follows a moving target and how far it leads one.
- Method: per-NPC bind. Aim and lock set once at net_spawn (apply). Target lead recomputed live on the fire seam.
- Seam: set_aim_params (PR #594) at net_spawn and on npc_shot_dispersion (the lead recompute, throttled 1s). The fields are not serialized, so every spawn re-sets them.
  On an exe without the bind the wrapper returns false and Reaction logs INACTIVE once.
- State/Cost: the tier cache and the per-NPC lead throttle (_next_lead, _tier_cache). apply reads the tier once per spawn. The lead recompute runs at most once per second per firing NPC.

Mechanism, three values. Aim tracking speed (min_speed, rad/s) is how fast the sight moves inside the min_angle lock band in select_speed (sight_manager.cpp:80-101), curve novice 0.24 to legend 1.50.
The tracking lock (min_angle, rad) is the gap below which the barrel tracks at min_speed with no deceleration (ai_aim_min_angle, sight_manager.cpp:76).
Vanilla 0.196 is half the fire cone, and the curve widens the band toward the fire cone (novice 0.196 to legend 0.40, past fire_angle 0.3927 so a legend holds a strafing target through the window).
Every value stays under max_angle 0.785 so the natural fast swing-in survives.
Target lead (predict_time, seconds) sets the aim point to the target's visible position plus its horizontal velocity times predict_time (predict_object_position, sight_action.cpp:383-384).
It recomputes per firing NPC on npc_shot_dispersion (throttled 1s, _on_shot_lead) as clamp(range / bullet_speed, 0, 0.5) times the rank lead factor.
bullet_speed is the weapon section's value times the loaded round's k_get_bullet_speed (ShootingObject.cpp:168, Level_Bullet_Manager.cpp:66).
Subsonic and AP ammo scale the lead, with a per-kind fallback (KIND_BSPEED) when the section lacks the key.
The per-rank lead factor (2.00 novice to 1.00 legend) is the only skill lever, so a legend leads true and low ranks over-lead a crossing target.

Choices, one decision per line:

- min_speed alone at the vanilla lock band is near-cosmetic - the band sits inside the trigger cone (fire_angle 0.3927), so the first shot never waits for it.
  It becomes real once Tracking Lock widens the band, which is why the two ship together and share aim_enabled.
- _resolve_min_angle returns -1 (follow the global) when the hardcore-aim option is already stickier, so a player's choice is never downgraded and no above-PI value reaches the assert.
  set_aim_params asserts on any angle past PI, because above PI the band silently collapses to always-on.
- The rejected delivery is driving the global ai_aim_* cvars from per-NPC update callbacks, a writer war (one NPC owns 4 globals, actor-only, reset every actor update) with values degenerate
  in the radians domain. Reaction takes the per-rank CONCEPT per-NPC - aim set-once at spawn, the target lead recomputed on the fire seam.
- The lead recompute re-passes the rank tracking speed so min_speed does not revert to the global. Nothing is set at spawn for lead, because there is no enemy yet.
- ONE FILE FOR THREE SYSTEMS is a ruled exception to the file-per-system law. set_aim_params writes aim, lock, and lead in one engine call.
  apply reads the tier once and sets every curve in one spawn body. One loader walks the seven curves over one LTX section and one rct_* MCM family.
  A split would duplicate the resolvers or add a fourth shared file.

### Disclosure

- Purpose: the hit-victim turn plus a bounded squad investigate on suppressed attacks, all through engine perception and selection. It makes no relation write and no memory injection.
  It never forces a squad combat-mask.
- Method: per-NPC bind (two standing CEnemyManager selection levers) plus per-hit script_danger stamps.
- Seam: net_spawn writes the two levers. npc_on_hit_callback drives the stamps, deferred one frame.
- State/Cost: the per-member re-stamp throttle (_stamp_at) and counters. The levers write once per online spawn. The stamps run only on an admitted suppressed hit.

Mechanism, the victim turn. set_hit_redirect(max, falloff) (PR #636, enemy_manager.cpp:149-167) scales the engine's own this-object-hit-me term in CEnemyManager::evaluate.
The last attacker within falloff metres gets up to max subtracted from its cost, decaying to 0 at falloff.
At 900/60 a close attacker outranks a fully-visible distant enemy, so the victim flips SELECTION on the real hit signal, even a victim already committed to another enemy.
No sighting is stamped, because fire_make_sense still requires real line of sight, so there is no through-cover fire.
The lever is standing per-NPC engine state, written once at net_spawn (not serialized). MCM off writes the -1 sentinel, which is the vanilla -5/-100 hit step.

Mechanism, the squad half. A suppressed hit on a surviving victim stamps his squadmates within earshot of the VICTIM (~15m, tunable) with graded scripted danger at the SHOOTER's position
(set_script_danger grade "solid", so the reaction walks the position weapon-up and never runs).
The stamp expires on its own inertion, and a member who actually perceives the shooter escalates to combat natively.
A member already fighting is untouched by construction, because the danger scheme does not run for an NPC with a combat enemy.

Mechanism, the gates. The silent gate seeds nothing on an unsuppressed human shot, because the engine's own gunfire perception covers it twice
(per-listener attack_sound danger entries, and the ally-relay CStalkerSoundDataVisitor, stalker_sound_data_visitor.cpp:30).
Suppressed-now is the utils_item.has_attached_silencer shape (an integral silencer, or an attachable one currently mounted).
The survivor gate handles a lethal hit. A hit that killed the victim seeds nothing (checked one frame deferred, because alive() is still true inside the killing hit's callback).
The deferral looks up CreateTimeEvent live, because demonized_time_events replaces the functions at runtime and a cached local would capture the dead originals.
The floor-exe fallback runs per admitted hit, the victim only, through set_script_danger at the KNOWN shooter position plus register_in_combat.

Mechanism, the target-priority dial. set_visible_enemy_bias(actor_bias, npc_bias) (PR #637, enemy_manager.cpp:175-184) replaces the hardcoded prefers-whoever-sees-me terms.
Vanilla subtracts 900 when the ACTOR sees the NPC and 300 for another NPC, a ~3x baked player magnet.
The MCM dial (0-900, default 900 = vanilla) writes the actor side per-NPC at the same net_spawn seam, and the npc side stays vanilla. Lower values treat the player like any other combatant.

Choices, one decision per line:

- The earlier force-disclosure model is retired. It force-ENGAGED distant patrol members with no perceptual basis, and its victim-turn leg (make_enemy_visible) was disproven at source -
  make_object_visible_somewhen saves and RESTORES the prior visible bit (memory_manager.cpp:355,361), so for an unseen shooter the "seen" promotion was a no-op and selection still ranked him
  ~1000 behind any seen enemy.
- No relation writes. The original goodwill-write era corrupted saved relations and is long gone. The module keeps only a per-member stamp throttle and counters.
- Keyed on per-NPC hostility (npc:relation(who) >= enemy), not community, so the victim turns on a real attacker of any faction.

### Crossfire

- Purpose: a friendly-fire damage gate, so same-faction NPCs do not kill each other through the engine's own imperfect avoidance.
- Method: callback adjust on the incoming hit.
- Seam: npc_on_before_hit, O(1) with no throttle, because a damage block must catch every hit.
- State/Cost: no state. One relation read and one multiply per stalker-vs-stalker hit.

Mechanism (at_crossfire.script:7-26). It scales shit.power by the crossfire_factor unless the shooter and victim are actually enemies (attacker:relation(npc) == game_object.enemy keeps full damage).
It runs stalker-vs-stalker only (both IsStalker), with the actor as shooter excluded.

Choices, one decision per line:

- Keyed on per-NPC relation, not community. Same-faction NPCs are neutral at worst and never enemy (a loner is never enemy to a loner), so they stay protected, while a soured cross-faction pair
  (a loner against a hostile Clear Sky) still damages each other. relation() is faction-paramount, because the community-to-community base dominates personal goodwill.

### Discipline

- Purpose: a per-rank fire-discipline curve - burst size and inter-burst cadence by rank. High ranks fire shorter bursts at a tighter cadence.
- Method: per-NPC bind (set_fire_queue_scale) for vanilla-driven fire, plus the shared tier tables that maneuver fire consumes.
- Seam: set_fire_queue_scale (PR #603) at net_spawn (apply -> _apply_discipline), applied in select_queue_params (stalker_combat_action_base.cpp:246-254) after the weapon-type and distance band pick.
  It logs INACTIVE without #603.
- State/Cost: the tier cache shared with Reaction. The scales are set once per spawn and read by the engine every planner solve.

Mechanism (at_reaction.script:185-198, 298-311). Two per-rank scales - _qsize multiplies burst size, _qinterval the inter-burst pause.
_qinterval is floored at DISC_INT_FLOOR (0.60) so bursts never merge into continuous fire.
get_queue_scales is the ONE owner of the tier tables (returning (1,1) when off or tierless, so callers multiply blindly), consumed by the at_maneuvers burst shape and the at_behaviors push restore.
One rank model feeds two application points.

Choices, one decision per line:

- Scope is vanilla-planner fire only. A state_mgr fire state (a maneuver override) reaches the object handler with explicit params and bypasses select_queue_params.
  This is why maneuver fire multiplies the same per-tier factors through get_queue_scales.
- Non-degradation invariant: defaults keep _qsize >= _qinterval per tier, so rounds per minute (proportional to size / (interval + size*dt)) stays at or above vanilla while a shorter burst sheds
  only the dispersed tail rounds (past base_dispersioned_bullets_count, WeaponMagazined.cpp:797-800), raising per-shot accuracy so hits per minute stay at or above vanilla.

## Perception

Sense-and-reaction, separate from combat skill. Sound and Vision feed detection. Danger owns the reaction scheme both of them stamp into.

### Sound

- Purpose: hostile stalkers hear the player's movement and handling noise, and every stalker notices nearby creature sounds. It adds the one sense vanilla lacks at close range.
- Method: feeder only. It adds no reaction machinery. Both signals stamp xr_danger.set_script_danger (the entry at_danger owns), so the reaction, decay, combat gate, and cleanup are the
  existing scheme's.
- Seam: actor_on_footstep (step_manager.cpp:209) and actor_on_land (Actor_Movement.cpp:90) for movement, npc_on_hear_callback (the vanilla ear) for handling and creature sounds.
- State/Cost: the per-hearer re-alert throttle and the accumulated step radius. A 500ms time event (TICK_SEC) stamps the movement radius.
  A standing NPC makes no step events, so its steady-state cost is 0.

Mechanism, movement (_on_footstep, at_sound.script:91-113). The vanilla per-step event accumulates a noise radius - BASE_RADIUS_M (5m walking) times stance (crouch 0 so crouched movement is
SILENT, sprint 1.6) times surface material (the step's material name, metal and wood and water louder) times the MCM multiplier times the install-hearing scale.
The install-hearing scale (_compute_hear_scale) is the winning config's own stalker hearing sensitivity over vanilla ([stalker] sound_threshold and the [stalker_sound_perceive] npc factor,
the two keys the engine admission consumes at sound_memory_manager.cpp:168), derived once, clamped 0.25-2.0. Landings add a flat LAND_RADIUS_M (10m) thud on the same path.
A scheduled pass (500ms) masks the accumulated radius by rain (level.rain_factor), walks db.OnlineStalkers, and stamps every alive actor-hostile stalker inside it (per-NPC re-alert throttled 4s).

Mechanism, handling and creatures (_on_hear, at_sound.script:162-181). The actor's WPN_reload (8m), WPN_empty (6m), and ITM_use (5m) types stamp the same scripted danger at the sound's position.
Creature awareness admits by type alone (only MST_* rows exist in CREATURE_ACTIONS).
Field-confirmed: the engine tags STALKER footsteps as MST_step and stalker voices as MST_talk, so the same bit carries stalkers and mutants alike.
MST_step (5m), MST_talk (8m), MST_eat (5m), MST_die (10m) all stamp FAINT, so no creature sound produces movement.
Creature sounds alert every stalker EXCEPT companions, and carry a longer per-hearer throttle (CREATURE_REALERT_MS 8000ms).

Choices, one decision per line:

- Every noise stamp carries an evidence grade, and the scripted reaction runs at that grade. A sound is a position ESTIMATE, so no sound-only stamp may produce a run state.
  FAINT (walking steps, ITM_use) turns the NPC weapon-ready toward the heard position. SOLID (sprint steps, landings, WPN_reload, WPN_empty) runs the reposition at raid (walk, weapon up).
- Footsteps are NEVER typed into the engine sound space. A typed hostile footstep would land an EnemySound danger per step and thrash the 32-slot sound memory (danger_manager.cpp:322-327).
  This is the reason GSC shipped footsteps with sg_SourceType = -1, and the reason the reaction is script-scoped (relation-enemy only, out of combat only, alert-not-omniscient, throttled, decaying).
- Crouched movement is silent outright (CROUCH_FACTOR 0), so a crouched stealth-kill approach that vision-based stealth is built around is never defeated by hearing.
  Neutral NPCs never react to the player's noise.
- Stealth compatibility. Stealth in Anomaly is a VISION system (CVisualMemoryManager). The sound system never touches that hook, vision configs, or seen-memory. Hearing adds the one sense at
  radii (5-10m) far below any vision range, and the scan has no occlusion test, so the radii double as the wall policy.

### Vision

- Purpose: a per-rank vision curve on two axes - acquisition SPEED (how fast a glimpse becomes a confirmed threat) and view RANGE (how far detection can begin). It is REACTION timing, never
  aim or accuracy.
- Method: Vision Speed is a per-NPC bind. Vision Range is a resolver provider (multi-source with optics). Both live in at_reaction.script (the shared-file exception), with the RANGE provider
  registered into the effects resolver.
- Seam: set_vision_speed at net_spawn (a factor into get_visible_value, visual_memory_manager.cpp:365-379). set_view_distance_factor at net_spawn through the resolver (n038). INACTIVE without
  the binds.
- State/Cost: the tier read at spawn. No per-frame work, because the factors are standing per-NPC engine state the engine consults on its own.

Mechanism. Vision Speed multiplies the per-update increment to the detection accumulator (visual_memory_manager.cpp:365-379), the rate at which confidence builds.
It applies after whichever detection stack the install runs. The band centers on novice = vanilla (novice 1.00 to legend 1.21, uniform step), every tier at or above vanilla so no rank detects slower.
It is inert inside always_visible_distance.
Vision Range multiplies object_visible_distance (the distance at which detection can begin), registered as the rank slice of the VISION_RANGE effect (max-combined with the gear optics under Gear).

Choices, one decision per line:

- Vision Speed is detection timing only. m_vision_speed is read solely in get_visible_value, and the sight, fire, and dispersion paths never touch it. It shapes how fast a stalker turns a
  glimpse into a confirmed threat, and it leaves his aim alone.
- The band stays modest (legend 1.21, not 2.00), because vision speed accelerates confirmation on any sightline the ray ALREADY reaches. A wide spread would fast-confirm targets through
  weakly-occluding foliage, since Feel_Vision gates line of sight on material transparency and a bush authored weak is defeated fast by a high multiplier.
- Vision cannot defeat occlusion. The accumulation formula carries no transparency term, so a high vision speed never makes an NPC see a target whose ray is blocked by cover. The Reaction rank
  curves and the disclosure seen-memory are orthogonal to the light model, so neither changes what light and cover let an NPC see.

### Danger

- Purpose: layer bug fixes and toggleable improvements onto whichever xr_danger a modpack ships (vanilla, GAMMA AI Rework, REDONE Combat AI), so the danger scheme reacts proportionately to
  what a stalker perceives. Vanilla bug fixes run always-on. Improvements sit behind MCM toggles.
- Method: function patch. At on_game_start AT points the winning xr_danger's generic-scheme entry points at its own versions. It ships no xr_danger.script of its own, so it does not compete for
  the MO2 slot.
- Seam: the seven patched entry points (setup_generic_scheme, add_to_binder, configure_actions, reset_generic_scheme, get_danger_time, set_script_danger, has_danger). The paired DLTX overlay
  (mod_xr_danger_at.ltx) is delete-lines only. It relies on the winner's own npc_on_hear_callback and npc_on_death_callback feeders, which AT does not register.
- State/Cost: the per-id script_danger table, the danger-scheme storage (danger_flag), and the parse cache (_parse_cached).
  eval_danger runs per NPC per plan solve, so its cost is the tightest budget in the mod.

Mechanism, how the patch installs (_install_patches, at_danger.script:1088-1116). The module points the winning xr_danger's seven entry points at its own versions.
Each is dispatched by a live _G["xr_danger"].fn lookup at bind and reset time.
Every script runs setfenv'd to its own module table (script_storage.cpp:44), so the winner's own internal bare calls resolve to the patched members too.
The winner's add_to_binder is never called. AT binds AT's evaluators and action into every stalker's motivation manager under the danger scheme's fixed ids.
The install also registers the monolith sub-scheme's four actions in state_mgr.combat_action_ids (state_mgr.script:10-18), which vanilla omits.
The idle evaluator forces st.combat=false for a current action absent from that table (state_mgr.script:93-96) and unwinds its state, so the omitted monolith actions cause the crouch-aim shuffle.
The install also wraps two rx_ff members - the abandoned-raid finalize and the dont-shoot evaluator.

```
on_game_start: at_danger points the winning xr_danger's entry points at itself
  setup_generic_scheme / add_to_binder / configure_actions / reset_generic_scheme / get_danger_time / set_script_danger / has_danger
bind time: the winning xr_danger's add_to_binder (now AT's) binds AT's evaluators and action into every stalker's motivation manager
runtime, per plan solve:
  eval_danger -> verdict first (eval_danger_raw), then npc_on_eval_danger on a would-be-true verdict (a veto clears the inertion latch)
    -> live script_danger stamp = TRUE on its own (the stamp is an ACTIVATOR, so a stamp reaction survives on modpacks that mute engine sound perception)
    -> else best_danger type -> inertion and ignore tables (the winning ai_tweaks/xr_danger.ltx rows) -> danger_flag
  at_action_danger:execute -> script_danger stamp first, else per-type response (grenade / corpse / attacked / attack_sound alert)
  combat-safe by GOAP construction: the action requires property_enemy false
feeders: the winner's own hear and death callbacks -> patched set_script_danger -> script_danger table
```

Mechanism, the sound reaction ladder. A heard sound buys attention in proportion to its evidence. It never triggers the full fighting-stance theater.
The grade is set at the single entry every feeder passes through. The patched set_script_danger defaults an ungraded stamp to "solid".
A companion defaults to "rush" so an ordered assist keeps its urgency.
faint is a look at the position only (a same-state set_state call, so a glance costs no re-plan). solid is a walk-over investigate (raid), then the standing scan.
rush is the assault run, companions only.
The active theater is time-boxed per episode (active_until = time_global() plus 8500-12500ms on the first solid pass, the alert machine's own stage-2 give-up band).
Episode open is also the voice moment, one "search" bark. Past the window the NPC settles into threat_na (standing, weapon ready, watching) until the config inertion decays.
The config window keeps owning MEMORY (danger_flag still suppresses looting and sitting) and no longer owns the body.

Mechanism, the squad stand-down gate (_is_squad_engaged, at_danger.script:278-297). While any squadmate holds a live best_enemy, the sound-fed dispatch sets watch and skips the theater,
so mid-battle bystanders no longer run noise choreography inside a live fight. rush stamps are exempt, because the live companion assist fires exactly when the squad is engaged.
The result memoizes 500ms per squad. The gate extends the engine's own personal-combat yield (stalker_danger_planner.cpp:55) to squad combat, which the engine lacks.
Grenade, corpse, and attacked reactions are confirmed threats and keep vanilla pacing.

Mechanism, the eval ordering (eval_danger, at_danger.script:369-388). eval_danger computes its result first and fires npc_on_eval_danger only when the result would be true.
The idle case broadcasts nothing.
A subscriber that sets flags.ret_value = false suppresses danger for that NPC on that solve, and a vetoed result clears the inertion latch so the veto leaves no stale true-window.
Every registered consumer in the field is a pure veto (Useful Idiots' companion patch, Duty Expansion's danger_ignore, Stealth Overhaul Reworked's corpse suppression),
so vetoing a false result was dead work broadcast for every online stalker per solve.

Vanilla bugs fixed, always-on. Every one is a crash, misread, or dropped behavior in the winning xr_danger or in stock Anomaly.

1. bd_types collision. The perceive-type names visual, sound, and hit share enum values 0/1/2 with danger types in the single danger_object enum
   (danger_object.h:18-35, memory_space_script.cpp:130), so three danger categories read the wrong config section. AT's table drops the three perceive entries.
2. get_danger_time crash on a mutant corpse. Vanilla calls corpse_object:death_time() with no IsStalker guard, and the trader interface is absent on mutants. AT guards it (at_danger.script:86-97).
3. eval_danger nil-NPC guard. Vanilla crashes when called on a torn-down NPC reference. AT returns false (at_danger.script:370-374).
4. eval_danger non-numeric danger_time. Vanilla type-asserts on a bad return. AT checks the type and returns false (at_danger.script:345-348).
5. The vanilla hit callback passed an undefined who_id on every hit. AT does not register it, and the patched set_script_danger accepts a nil who_id as a source-less stamp (the vanilla quest form),
   reading the stamped position alone.
6. Animstate reset missing on danger-state transitions. Vanilla left stale lower-body animation across the transition. _set_state_once applies the reset on every flip (at_danger.script:250-272).
7. at_action_danger:finalize wiped the whole shared db.used_level_vertex_ids map in vanilla, clobbering every other system's cover claims.
   AT releases only the vertices this NPC owns (at_danger.script:861-867).
8. script_action_danger_corpse compared st.stage >= 4 bare in vanilla and crashed when the corpse reaction fired before initialize set the stage. AT reads (st.stage or 0) (at_danger.script:573).
9. The corpse action crashed on the teardown race. A corpse despawning between the evaluator pass and the execute left a nil danger object or storage entry. Both paths are guarded.
10. The corpse force-hostile loop reused the acting NPC variable for squad members, so later stages drove the last member. The loop uses its own local (at_danger.script:576-581).
11. Corpse stage 6 sent the NPC to the just-cleared st.lvid in vanilla. AT sends it to the cover vertex try_go_cover found (at_danger.script:670-675).
12. Performance. Vanilla re-parsed the inertion and ignore-distance condlists on every evaluation. The strings are fixed after the DLTX merge, so they parse once into _parse_cached and every later
    evaluation is a table lookup (at_danger.script:27-35).
13. script_danger entries dropped on entity unregister. Vanilla kept a dead perceiver's entry until expiry, so a recycled id inherited a scripted danger the new NPC never perceived.
    AT drops it on unregister (at_danger.script:1040-1042).
14. Episode state cleared at finalize. Vanilla reset only stage, so last_pos held the first attacker's position forever and searched stuck after one search.
    AT clears every episode field (at_danger.script:848-856).
15. Grenade far gate compared a meters constant against a squared distance in vanilla (a literal 70 against distance_to_sqr, effectively 8.4m).
    AT names GRENADE_FAR_M and squares it at the compare (at_danger.script:23,520).
16. rx_ff abandoned-raid freeze (stock Anomaly, outside xr_danger). action_verso:finalize frees its cover vertex but restores no state, so a stalker whose friendly-fire hold ends with his enemy
    gone stays parked in "raid" and the engine slides him. Rulix's original CoP finalize ended with a calm set_state("idle") the Anomaly port dropped.
    AT wraps rx_ff.action_verso.finalize and restores the calm state when no enemy remains (at_danger.script:1064-1071).
17. rx_ff dont-shoot over-hold. rx_ff's friendly-fire hold is cruder than the engine's lane check and silences NPCs the engine would clear. AT wraps rx_ff.evaluator_dont_shoot.evaluate to hold only
    when can_kill_member agrees a squadmate is in the lane (at_danger.script:1076-1086).

Improvements (MCM Perception, default on):

- danger_hit_bypass (Distant hits). A direct hit is danger at any distance - the branch returns true past the relation, combat-ignore, and ignore-distance gates, because being hit is proof of
  range (at_danger.script:201-205). at_disclosure owns learning the shooter across the squad. hit_bypass owns the victim ducking even when he cannot fight back.
  Answering fire at the shooter's range is a separate concern that neither owns, so a duck is the whole reaction here.
- danger_attack_sound (Enemy gunfire). Reacts to enemy gunfire the NPC heard but cannot see. The engine produces the attack_sound danger (danger_manager.cpp:301) and vanilla shipped no handler.
  AT admits it, routes it to the alert action, and drops the inherited non-enemy aim gate.
  A hostile stalker who cannot see the shooter turns to face the sound and holds a threat stance.
  The move-to-cover follows once he can see the enemy.
- danger_actor_tables (Player ranges). Reads separate inertion and ignore tables when the danger source is the actor, meaningful where the config differentiates them (GAMMA AI Rework does, vanilla
  ships identical copies). Gated by a liveness probe (_actor_tables_live), honored only when the winning xr_danger carries its own DangerIgnoreActor field.
- danger_neutral_gunfire (Neutral gunfire alert). A NEUTRAL or FRIENDLY stalker goes on alert to the PLAYER's shots fired within NEUTRAL_GUNFIRE_RADIUS_SQR (30m, its own small constant). The reaction
  is alert and nothing more, a weapon-ready stance facing the sound that decays with the inertion. It is scoped to the actor, so NPC-vs-NPC neutral fire stays ignored.

Choices, one decision per line:

- The extension callback is preserved with one ordering change. eval_danger fires npc_on_eval_danger with flags.ret_value = true, and a subscriber that sets false suppresses danger for that NPC.
  AT fires it only on a would-be-true result, because every field consumer is a pure veto and broadcasting a false result was dead work.
- Composition layers, it does not exclude. at_danger carries the @novalidate marker only to exempt its vanilla-derived code from AT-native style rules and the stub load test, and it is not a VFS
  whole-file override. AT patches the winner at runtime, so Danger layers onto GAMMA AI Rework or REDONE while the rival's file stays loaded and its own perception callbacks keep running. The one
  thing the patch cannot do that a file override could is suppress the winner's danger callbacks, which surfaces only as REDONE's fixed hit callback adding a second harmless trigger.
- The paired DLTX ships NO danger values. mod_xr_danger_at.ltx is delete-lines only (the ![section] delete, Xr_ini.cpp:721), dropping the dead perceive keys.
  Every detection distance and inertion comes from whichever xr_danger.ltx won the slot, so GAMMA plays AI Rework's tuning unchanged and vanilla plays vanilla's true-name rows.
- The action is combat-safe by GOAP construction. It requires property_enemy false, so it never runs for an NPC with a combat enemy.

## Mechanics

Four systems that act on the object layer, below the combat decision. Each names its own method - a function patch or a monitor field-write - drawn from outside the combat taxonomy.

### Healing

- Purpose: per-NPC self-healing - a heal-rate multiplier, an engaged-pause so a fight can end, a per-rank medkit-charge grant, and a cosmetic limp pose and heal gesture.
- Method: function patch (three xr_eat_medkit members), a spawn callback (the charge roll), a monitor field-write (the limp pose), and one wrapped wounded method.
- Seam: xr_eat_medkit.heal_hp, heal_bleed, and consume_medkit patched at on_game_start (at_healing.script:200-215). npc_on_net_spawn drives the charge roll. A 200ms monitor drives the limp pose.
  zzz_at_healing_patch unregisters the vanilla xr_eat_medkit.on_register roll.
- State/Cost: the spawn-filled roster (_npc_states) and the limp records (_limping). The heal loop runs on vanilla's own event chain. The limp monitor runs every 200ms.

Mechanism, the data-layer fix. Vanilla ai_tweaks/xr_eat_medkit.ltx [plugin] lacks the medkits= and bandages= keys, so parse_list returns {} and the consumption loop iterates 0 times.
mod_xr_eat_medkit_at.ltx (a DLTX overlay on ![plugin]) adds them, boot-time. The paired zzz_at_healing_patch unregisters vanilla's on_register roll, because vanilla rolls math.random() > 0.5 for
healing_charge on EVERY server_entity_on_register and alife re-fires that for every restored entity on save load (alife_storage_manager.cpp:160), so vanilla re-rolls per load and can re-grant one.
The zzz_ prefix loads it after xr_eat_medkit.

Mechanism, runtime tuning (installed at on_game_start). The heal-rate hook replaces xr_eat_medkit.heal_hp, so each change_health is scaled by the multiplier.
Each heal_hp firing reschedules through the xr_eat_medkit.heal_hp lookup, which keeps it on the patched function.
The engaged-pause lives inside _update_heal_hp - the heal_hp firing PAUSES while the NPC is actively engaged (a live best_enemy with the weapon in his hands, or an enemy inside 5m of any kind).
The pause is a ResetTimeEvent on the firing event plus a return false, so the same event stays queued with a pushed timer. Bleed staunching stays vanilla.
The per-rank charge grant runs on npc_on_net_spawn. It reads the rank tier and rolls the per-tier chance, which supersedes vanilla's flat roll.
A per-NPC at_charge_processed se_var prevents a re-roll.
The wounded-consume trace wraps xr_wounded.Cwound_manager.eat_medkit, the only signal for a medkit handed via the help dialog or burned by the autoheal.

Mechanism, the visual layer (Path 1 script-queue overlay, at_healing.script:285-422). The limp pose runs on the 200ms monitor. Its drop detectors (wounded or dead, a changed gait, a drift off the
stand anchor, a stop in displacement) each clear the animations. A 1s eligibility check gates it (hurt, no enemy, calm, standing, not zombied, not in smart_cover).
The pose is a per-slot dmg_norm hurt animation chosen from active_slot() and movement_type(). The heal gesture plays once on the first heal_hp firing.
It plays only when the NPC has no enemy, is neither wounded nor critically wounded, holds an empty animation queue, and stands still.

Choices, one decision per line:

- The heal loop runs on vanilla's stage machine untouched. AT patches only the rate and adds the pause. Pausing bleed staunching would turn every pressed fight into a bleed-out lottery, so bleed
  stays vanilla.
- A queued script animation suspends the engine's whole animation selection (stalker_animation_manager_update.cpp:232), so the limp pose must die the moment its gait stops matching.
  That is what the drop detectors do.
- Both animations play only within ANIM_ACTOR_RADIUS_M (50m) of the actor, a presentation-only gate. The healing SIMULATION (the hp and bleed loops, item consumption) is never distance-gated, so
  off-screen stalkers heal identically.
- Limping is independent of the healing master toggle. Its monitor arms unconditionally and is gated at runtime by limping_anim_enabled, one boolean per pass when off.
- The patches install UNCONDITIONALLY at on_game_start. With the master off the same installed functions run exact vanilla semantics, so an MCM flip applies on the next heal_hp firing with no restart.

### Jamming

- Purpose: suppress the modded-exes script-injected NPC misfire path, so an NPC at full weapon condition does not misfire every 2-3 rounds on ammo spent alone.
- Method: function patch on the modded-exes functor.
- Seam: xr_weapon_jam.GetConditionMisfireProbability = _compute_misfire_chance (at_jam.script:29), the functor the engine looks up by name and calls per shot at Weapon.cpp:1781. It logs INACTIVE
  when the functor is absent (vanilla Anomaly or AOEngine).
- State/Cost: no state. One snapshot boolean read per NPC shot.

Mechanism (at_jam.script:12-30). The wrapper returns 0 while jam_enabled, so the engine's per-shot misfire roll for non-actor weapons becomes 0.
With the toggle off it forwards (weapon, npc, base_value) to the saved original, restoring modded-exes behavior.
The engine gates the functor call to non-actor parents at Weapon.cpp:1778, so actor weapons keep their full vanilla condition-based misfire roll.

Choices, one decision per line:

- The install captures the original FIRST and installs only when it is a real function, because a wrapper closing over a nil original is harmless while jam is enabled and a nil-call crash the
  moment it is disabled (the disabled path forwards to the original). An absent functor logs a WARN and the module stays inert.
- MIN_DEMONIZED_VERSION is not raised for this. The feature is informational at the dep-gate layer, so a floor exe runs the mod with jamming inert rather than failing the gate.

### Ammo

- Purpose: veteran-and-up NPCs fire AP from the loose ammo they carry, with one rank-and-rate-weighted box decay per fight, until the NPC runs out and reverts to vanilla magic FMJ.
- Method: monitor field-write (wpn:set_ammo_type) plus an inventory box-delete.
- Seam: a 5s monitor over the spawn-filled roster (update_npc). wpn:set_ammo_type(idx) re-keys what is fired. alife_release deletes a box. It logs inert when g_ai_unlimited_ammo is 0, because the
  engine then consumes real inventory rounds and the whole-box decay would double-consume.
- State/Cost: the per-NPC ammo state (_state) and the roster (_npc_ids). The monitor runs every 5s. Save load resets _state, and depletion lives in the inventory, so it persists for free.

Mechanism (at_ammo.script:116-220). While unlimited_ammo is TRUE (the stalker default, ai_stalker.cpp:78) the magic refill copies m_DefaultCartridge keyed from m_ammoType (WeaponMagazined.cpp:559).
It consumes no inventory, and the reload does not re-derive m_ammoType. wpn:set_ammo_type(idx) re-keys what is fired and holds it for the online session.
The budget is the NPC's AP boxes themselves, with no virtual ledger. The combat pass
(on combat entry or weapon change) caches idx and sec via _find_ap and holds m_ammoType = idx while AP is carried. The peace pass (once best_enemy has been nil past peace_debounce_ms) rolls
_compute_decay_chance, releases one AP box of sec on a hit (alife_simulator_script.cpp:288), and sets m_ammoType = 0 when the section is now empty.
The decay chance is ap_decay_base times (rpm / rpm_ref) times rank_weight, so fast weapons burn AP quickly and high rank conserves it.

Choices, one decision per line:

- The budget is the inventory, no ledger. Deleting a whole box is permanent, because try_advance_ammo (object_actions.cpp:131-169) refills rounds inside surviving boxes but cannot recreate a
  deleted box, so counting boxes never fights the top-up.
- AP_SECTIONS is the clean-AP set (box_size 15 rifle and 16 pistol), and the degraded _bad and _verybad variants are excluded by design. The set is exported, so the test AP-arming helper and the
  fire path agree on one caliber list.
- No death hook. Vanilla decide_items_to_keep already releases every ammo box over 5 rounds on death (an AP box is 15 rounds), and npc_on_death_callback fires after that release, so a death-time
  trim could not preserve AP anyway.
- Decay is per-engagement, not per-shot, because no NPC fire callback exists. A continuous siege counts as one engagement.
- The dependency: a loose-AP source in the NPC's inventory and the magazine system off. Vanilla gives NPCs 0 loose ammo, so the AP an NPC carries comes from a trade-and-loot source, and with
  ammo encapsulated in magazine items get_ammo_count_for_type reads 0 (Weapon.cpp:1727) and the NPC stays on FMJ.

### Gear

- Purpose: the functional-inventory SOURCE. An artefact grants one combat edge chosen by its anomaly CLASS, scaled by the section's tier. Gear registers lazy providers into the effects resolver and
  writes only its single-source passive regen itself.
- Method: five resolver providers (DAMAGE_DEALT, DAMAGE_RESIST, SHOT_DISPERSION, VISION_RANGE, AURA) plus the single-source regen bind.
- Seam: register at on_game_start. npc_on_item_take and npc_on_item_drop re-resolve a cached record. net_spawn writes the regen. A one-time n039 probe routes the chemical channel.
- State/Cost: the resolved records (_resolved) and the class map (_kind). The one inventory walk happens on first resolve per NPC and caches, so a hit or shot reads the cache.

Mechanism (at_gear.script:55-166). The class-to-effect map: gravi to DAMAGE_RESIST, thermal to SHOT_DISPERSION, electro and the quest specials to DAMAGE_DEALT, ballistic plates to DAMAGE_RESIST,
binoculars to VISION_RANGE by day, NVG to VISION_RANGE by night, chemical to DAMAGE_RESIST or passive regen when the n039 bind is present, and every artefact class to the carrier AURA.
The strength is the section's tier times a 2% step, capped at 10% (tier x 2%, the 4/6/8% ladder over vanilla tier 2-4).
Detection is by exact vanilla section name from the class lists in at_gear_config.ltx.
The one inventory walk keeps the strongest strength per effect (a channel never stacks across items).
Optics read day and night live at the VISION_RANGE provider (_is_night), so a carrier's range tracks the clock without a polling pass.
Passive regen is the one single-source value Gear writes itself through xcombat.set_health_restore_boost (n039), the neutral 0 for a non-carrier so a re-online never keeps a stale boost.

Choices, one decision per line:

- The class-to-effect assignment is a design choice and lives in the script. The section membership is data in the ltx, so a modded section not listed grants nothing.
- Eligibility is checked inside the providers. The combat effects gate on a live, non-actor, non-zombied stalker. The AURA gates stalker and non-actor only, because a zombie still physically carries
  the artefact and the glow marks the loot.
- The record is toggle-independent (raw strengths plus presence flags), so an option change never forces a re-scan.
- The chemical strength routes after the walk - to passive regen when the n039 bind is probed present, else folded into the resist pool. On today's exes the write is dead and chemical stays on
  DAMAGE_RESIST.
- The aura particle name lives in at_gear_config.ltx because the carrier set is a gear fact, but the resolver applies it (Substrate, Effects resolver). Only a config-referenced particle name is
  valid, because a name living solely in particles.xr is an engine fatal with no script-side check (r4.cpp:738-748).
- Carrier persistence is enforced by the loot and trade inventory-policy floors, not by Gear. Gear only marks the carrier, and circulation stays loot-and-combat only. AT does not implement those
  policies.

## Observability

Dev tooling, off in play. It measures outputs, because an intent field (movement_type target, body_state, animation_count) reads as frozen whenever an animation plays, so a healthy vanilla NPC and a
genuinely stuck one give identical reads. Two concerns split into two modules and two log files, with no logging-only middle files. CODE tracing (what the mod's code decides and does) goes to
alifetactics.log through at_debug. WORLD tracing (whether the fight physically looks right, measured from positions and bullets) goes to alifetactics_world.log through at_world_trace.

The tracing law. Standing traces cover a system AS A FLOW - one line per transition (begin, end, reenter, release, pause, the decision line when a row fires) and one aggregate measurement per
phase (the whole-walk walk=us, the [MON] span). Depth instrumentation (per-stage timers, per-pass field dumps, counters) exists only while an issue is under investigation and comes back out with it.
git history keeps the implementation. The WARN watchdogs (_check_sight_lost, the escape warn) are regression detectors that stay even when the flow traces are stripped.

### at_debug

One primitives file, so no gameplay module owns a logger or a debug boolean. It holds one logger to alifetactics.log and the at_debug.is_on() gate (one integer compare against DEBUG_LEVEL 5).
It holds the shared format_flag and show formatters. It refreshes the log level mod-wide from one lifecycle. update_config reads the MCM log level and sets the logger's flush_on_level.
TRACE flushes every line, and the ERROR default leaves xlog buffering below it. Every gameplay module calls at_debug.debug, info, or warn at its own sites, in its own words.

### at_world_trace

An outcome recorder over the engine's shot and impact feeds, reads only, off by default with no callbacks registered until the toggle. Two gates split the work.
The toggle drives CAPTURE (in-memory counters plus a 200ms actor-velocity poll for the aim split), and the log level drives OUTPUT (per-bullet lines at DEBUG, minute tables at INFO).
There is no on-screen panel.

The slide watchdog reports the visual defects provable from cheap reads. A SLIDE is the body travelling a real distance while its movement_type is never a locomotion type, measured from POSITION.
FAST is the rate against the fastest [stalker_movement_speeds] speed times a 1.25 tolerance, WHATEVER the movement type - the walked-glide the SLIDE anchor cannot see.
A 150ms monitor logs one line per slide episode. A 60ms hit-triggered watch samples the body for 600ms after every nearby hit and writes one raw record at close (health, overlay count, movement type).
The monitor also logs [DANGER-HELD] on change for near-actor NPCs - the held danger's type, source, distance, age, and whether the NPC entered AT's own danger scheme (at_danger.has_danger).
The old watchdog inferred appearance from these intent fields, and all of it is deleted, because those fields read as frozen whenever an animation plays.

The ballistics recorder reads the driver context directly from the modules at capture time (at_maneuvers.get_maneuver, at_commitment.get_hold, vanilla), so no cross-log correlation ever happens.
Per bullet fired (npc_shot_dispersion) it records tier, weapon kind, shooter motion, and the driver. Per bullet landed (bullet_on_impact) it records a hit on the actor or a near miss inside 10m.
The minute tables carry per-tier hit% split still and moving, arrival conversion, damage, hits per minute, per-driver hit rates, the burst-length histogram per weapon kind, and two aim axes.
ang and ahead are actor-relative (the fired-direction error and the lead overshoot).
WRONGWAY is enemy-relative - the angle between the round's own direction and the shooter-to-best_enemy line, measured from the BULLET, the provable form of shooting the wall.
at_world_trace.reset() zeroes the counters and restarts the session clock, the bench-run boundary.

need_cleared is the third output (at_maneuvers.script, at maneuver end). at_maneuvers re-runs the row's own check_need with a fresh memo, and need_cleared=n at hand-back is the unsolved signal - the
maneuver ran and did not solve its problem.

### at_hud

Two lines per NPC on one shared 2-column grid, so every column keeps its position by construction. There are no header rows. The grouping is pure ORDER - AT-driven first, then anyone fighting, then
idle (only with HUD_SHOW_IDLE), nearest first within each group, capped at MAX_ROWS with a "+N more" overflow line. Line A (bright) carries rank, name, and hp beside the SYSTEMS cell, every AT combat
system active on the NPC as one comma-joined list. Line B (dim) carries target, distance, and sight word beside scheme and mental. The dominant driver colours the whole row, green for a maneuver,
blue for Commitment, amber for the Push window, mauve for Conduct, and untouched for plain vanilla.

The cost design keeps a huge crowd cheap. A candidate pass over at_core.get_combat_records() (already the IsStalker-filtered tracked set) reads only the record's own facts (actor_dist_sqr against the
gate, maneuver, best_enemy_id) with no raycast, no name, no operator, and the expensive _build_row runs only for the capped visible set. The window is built once (UIDebugHUD), and a refresh only
rewrites text and colors. The refresh runs every 0.5s (the takeover state changes on a 200ms cadence and maneuvers live 2-3s, so a slower repaint missed whole maneuvers), visibility-gated (hidden
while the PDA is open), the time event re-armed first.
The display radius is the gate radius through at_core.is_in_gate, one radius with no HUD-private number. HUD_SHOW_IDLE is a code constant, because every key in the at_mcm defaults table must have a
tree widget and it never had one.

### at_test

Console commands (@export console, run via run_string). The create_squad_* family creates a squad at the actor's vertex and teleports it distance and angle away.
Every spawned member is gear-parented BEFORE online with one artefact per anomaly class plus a kevlar plate and both optics (EFFECT_GEAR), so every resolver channel fires on the real net_spawn path.
start_arena(n) maintains a constant fight around the actor - n/2 side-a (army for retreat plus the three flee-prone factions) against n/2 monolith, the universal aggressor.
add_ap_ammo arms nearby stalkers with the AP their weapon fires plus a veteran rank, reading at_ammo.AP_SECTIONS so arming and firing agree.
run_lab stocks every nearby stalker per pass (a medkit and bandage, AP plus the veteran rank, and one palette weapon per squad).
create_squad_unarmed strips a spawned squad 3 seconds after online to prove the unarmed seize decline.
No console telemetry aggregate exists by design, because the DEBUG alifetactics.log already carries the per-decision lines.

## Engine write surface

The principle is to feed engine memory and state, then let the engine run its own combat detection (property_enemy, m_combat_mask, agent_memory propagation) on what was written.
No system reimplements engine behavior. Each one writes engine state to produce the outcome.

```
system         engine state written                                          key calls
Combat         a GOAP action graft at id 188347, the block preconditions,    add_evaluator/add_action/add_precondition, best_cover,
               destination, fire/posture/movement state, the movement hold   set_dest_level_vertex_id, state_mgr.set_state, set_movement_hold
Commitment     nothing; it denies a proposed action switch or cover re-pick  the npc_on_combat_action_switch and npc_on_best_cover_repick vetoes
Conduct        the combat body-state answer, the cover-band min/max answers  npc_on_combat_set_body_state, on_get_min/max_combat_dist, has_shot_obstacle
Push, Pull     per-NPC fire-queue scales, the cover-band overlays            set_fire_queue_scale, at_conduct.set_push_max/set_pull_band
Accuracy       per-shot dispersion (the move penalty direct, rank a source)  npc_shot_dispersion, at_effects_resolver.register
Reaction       per-NPC aim, vision speed, fire-queue scales at net_spawn     set_aim_params, set_vision_speed, set_fire_queue_scale, register
Disclosure     CEnemyManager selection at net_spawn, per-hit danger stamps   set_hit_redirect, set_visible_enemy_bias, xr_danger.set_script_danger
Crossfire      the incoming hit power on a friendly hit                      npc_on_before_hit (shit.power scale)
Danger         the danger evaluators and action on the winning binder       patches xr_danger's seven entry points, the script_danger table
Healing        the NPC health and bleeding fields, the healing_charge var    change_health, bleeding =, se_save_var
Jamming        a module function on xr_weapon_jam                            xr_weapon_jam.GetConditionMisfireProbability (read at Weapon.cpp:1781)
Ammo           the CWeapon m_ammoType field, a per-fight box delete          wpn:set_ammo_type, alife_release
Gear           no multi-source write; five providers plus single regen      iterate_inventory, register, set_health_restore_boost
Effects        per-hit shit.power, per-shot dispersion, spawn range + aura   apply_hit_power, apply_dispersion, set_view_distance_factor, start_particles
resolver
```

## See also

- xlibs: the xlibs mod's doc/architecture.md - the xcombat primitive surface every method above is issued through.
- The engine source: themrdemonized's xray-monolith fork - the GOAP planners, CEnemyManager, sight_manager, and every cpp/h line cited above.
- Vanilla Anomaly: the unpacked gamedata scripts (xr_danger, state_mgr, xr_combat) the function patches attach to.
