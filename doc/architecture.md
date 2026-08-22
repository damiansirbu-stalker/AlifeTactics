# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Every system works the same way: it reads the real state of the game (combat events, NPC and player stats, the world, squad, faction, weapons, range, angle, cover) and makes an intelligent decision from it, in the combatant's favor, instead of a script or a die roll. The user-facing systems: a hit-victim turn with a bounded squad investigate (disclosure), a self-heal data + animation layer, a per-rank weapon accuracy curve plus reaction (aim and vision) curves, a danger scheme that layers bug fixes and toggleable improvements onto whichever xr_danger a modpack ships (a runtime patch, not a file override), an intermittent combat takeover that borrows a stalker for one authoritative maneuver the vanilla engine has no mechanism for, and Commitment, a veto that keeps a stalker on a decision that still makes sense instead of the engine's constant re-planning. The takeover overrides zero vanilla combat files, so it works with other combat AI, not against it. The direction is that every layer hooks into the engine and decides intelligently. Where more than one source feeds a single per-NPC value the engine writes once - the damage on a hit, the dispersion on a shot, the vision range at spawn, the carrier aura - a deliberate shared substrate, the effects resolver (`at_effects_resolver.script`), combines those sources highest-wins so they never clash; a value only one source feeds stays in its owning module and writes itself, never touching the resolver.

Built on xlibs (xsquad, xttltable, xtime, xprofiler, xlog, xmcm, xslice, xcreature).

Part of a four-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** tunes existing rates and counts, **AlifeGuard** keeps alife state clean, **AlifeTactics** controls how NPCs fight in combat (this mod).

---

## Invariants

Project-wide constraints. Every system holds all of them; a change that violates one is wrong even when it works.

- **Performance first.** Performance is the top priority and outranks features. A feature that cannot meet the budget below is reworked, replaced, dropped, or removed with an X-Ray engine modification — never kept at the cost of the budget. Only correctness and "never break base gameplay" rank above it. See `doc/standards/code-standards.md` "Performance is the priority".
- **Use the engine, don't work around it.** Every capability comes from the engine and the Anomaly layer first, always through xlibs (here, xcombat); our own code enters only where stock behavior falls short, escalating nudge / correct then, as a last resort, changing the engine itself — the demonized PRs that added the combat seams are exactly this. Never reimplement in script what the engine already does. See `doc/standards/code-standards.md` "Use the engine, don't work around it".
- **No per-frame work. Never.** Ongoing work runs on a scheduled tick (a fixed interval - the vanilla `CreateTimeEvent` queue) or on a discrete engine event (a hit, a shot, a spawn, an option change) - never continuously every frame, and never on a per-frame engine callback, however small the body: dispatch work in front of a throttle IS per-frame work (the pre-2026-07-10 at_combat monitor rode `npc_on_update` and was rebuilt onto a time event for exactly this). We never place code on a path the engine itself runs every frame (a visibility functor, a fire functor). Frame-spreading a bounded one-off batch (xslice, 1 item per frame) is the one allowed use of the frame; it completes and stops. Full rule: `doc/standards/code-standards.md` "No Per-Frame Work".
- **2ms is the ceiling.** Every measured flow targets 0.1ms average per call with a hard 2ms ceiling - an eighth of a 60fps frame. No exceptions: cold start, save load, and level transition all count, and debug-only tools count too. A flow that averages above 0.1ms or ever crosses 2ms is a regression and gets a perf task. When a flow costs too much, the answer is a simpler design, not a faster version of the same one.
- **Measured, not asserted.** Every flow carries a duration field in its DEBUG trace (`walk=us`, `[us]`, `[ms]`); the timers are null objects when debug is off, so measurement costs nothing live. No mechanism (cache, backoff, throttle, precompute) is justified by an unmeasured cost - the decide-path decline backoff was built and removed the same day for this, and the per-row walk timings that replaced it are the gate any future cost mechanism must pass.
- **No file overrides.** AT replaces no vanilla file. Every system attaches by callback, function patch, save-wrap, DLTX overlay, scheme patch, or the time-boxed takeover transaction. Composition with modpacks falls out of the attach mechanism, never out of luck.
- **Engine truth.** Every mechanism claim in this document carries an engine source cite (file:line into xray-monolith or vanilla Anomaly). A behavior that could not be proven from source does not ship; where the engine had no seam, the seam was added upstream first (the demonized PRs: action-switch veto, per-NPC aim and vision, fire-discipline binds).
- **The takeover is a bounded transaction that TRIES to solve its problem.** Vanilla owns every NPC by default. AT borrows one NPC for one committed, time-boxed maneuver and releases it - at most one open maneuver per NPC and at most `max_concurrent_maneuvers` open across all NPCs (a runaway failsafe), ended on arrival, a hard cap, or a broken premise, cleaned up on death and despawn. There is no reseize cooldown: every row's need states the FULL problem and the maneuver's success negates it (kite clears its own weapon minimum, flee routs past the enemy's reach, counterflank flips the engine's enemy selection, a finished pickoff hold resets the stall measurement; retreat is the one row whose need is a STANDING condition - it only grows the distance, so it legitimately re-fires while the NPC stays hurt inside a long weapon's reach), so a solved problem does not re-fire and a recurred or unsolved one legitimately does - three kites under sustained pressure are three correct transactions. Every maneuver is locked to the target it was staged against (`state.enemy_id` resolved via `xcombat.resolve_enemy`, in the shared shell so every current and future row inherits it): the staged target is what the NPC LOOKS at (`look_object`) and what every read and end condition resolves - sight, `fire_make_sense`, the `check_end` premises, `target_lost` - for the maneuver's life, and his death or despawn ends the maneuver at once with vanilla re-selecting from there; which enemy his SHOTS select remains `CEnemyManager`'s own pick, which the takeover does not touch. AT is an interrupt over vanilla, never the combat brain.
- **xcombat boundary.** Every NPC combat command and read goes through an xcombat (xlibs) primitive; AT makes no raw engine combat call. AT owns policy (when, whom, which maneuver); xcombat owns mechanism (how to issue it to the engine).
- **Debug is free when off.** Every trace call gates on one integer compare (`at_debug.on()` / `at_debug.dbg`). The off path builds no string, allocates nothing, and crosses no luabind bridge; an expensive line's whole site is gated by `if at_debug.on()`.

---

## Status

Version 1.1.5.

| Module | Type | State |
|---|---|---|
| `_at_deps.script` | infra | done |
| `at_mcm.script` | infra | done |
| `at_test.script` | infra | done |
| `at_hud.script` | infra | done (live debug HUD; each row coloured by the combat system holding the NPC - maneuver green / commitment blue / conduct mauve - noop unless enabled) |
| `at_debug.script` | infra | done (the one code-trace primitives file: logger, gate, formatters, level lifecycle; every module logs through it) |
| `at_world_trace.script` | infra | done (the world log: slide watchdog + ballistics recorder, one toggle, driver attribution, MCM Development, zero cost off) |
| `at_disclosure.script` | feature | done |
| `at_crossfire.script` | feature | done |
| `at_effects_resolver.script` | substrate | done (the multi-source effects resolver: the `EFFECT` enum, `resolve` (highest-wins), `register`, and the engine WRITES for DAMAGE_DEALT / DAMAGE_RESIST / SHOT_DISPERSION / VISION_RANGE / AURA; sources register lazy providers, the resolver owns the appliers) |
| `at_firstaid.script` | feature | done (active first-aid healing: self-heal rate + per-rank medkit charge + limp/heal anim; also traces the wounded-down medkit consume in `xr_wounded.Cwound_manager:eat_medkit`, the vanilla seam that fires no callback) |
| `at_accuracy.script` | feature | done |
| `at_reaction.script` | feature | done (per-rank aim, target lead, vision, fire discipline) |
| `at_commitment.script` | feature | done (action-switch + cover re-pick vetoes) |
| `at_combat.script` | feature | v2 intermittent takeover: solo maneuver set (counterflank, reload_cover, flee, retreat, kite, pickoff), actor-party fights only; squad coordination and enemy openings are the open phases (todo-combat-takeover-v2.md) |
| `at_combat_doctrine.script` | feature | done (the maneuver catalog + arbiter, at_combat's decision half) |
| `at_danger.script` | feature | done (function-level patch onto the winning xr_danger) |
| `at_noise.script` | feature | done (movement + handling noise hearing, feeds the danger scheme) |
| `at_stance.script` | feature | done (conduct posture: crouch behind real low cover, engine body-state callback) |
| `at_jam.script` | feature | done (NPC misfire suppression via the modded-exes functor) |
| `at_ammo.script` | feature | done (AP fired from carried boxes, box-delete decay) |
| `at_gear.script` | source | done (functional NPC inventory SOURCE: scans carried artefacts/plates/optics, registers lazy providers into the effects resolver; owns the n039 passive-regen write) |
| `zzz_at_firstaid_patch.script` | feature | done (vanilla xr_eat_medkit re-roll suppressor) |
| `configs/ai_tweaks/mod_xr_eat_medkit_at.ltx` | data | done |
| `configs/ai_tweaks/mod_xr_danger_at.ltx` | data | done (delete-lines-only DLTX: drops the collision keys; the copied value rows removed 2026-07-10) |
| `configs/alifetactics/at_combat_config.ltx` | data | done (Combat takeover tunables) |
| `configs/alifetactics/at_ammo.ltx` | data | done (Ammo tunables) |
| `configs/alifetactics/at_reaction.ltx` | data | done (Reaction curves; the MCM slider defaults read from it) |
| `configs/ui/ui_at_stats.xml` | data | done (at_hud layout) |

Backlog (not built):
- Tactical flee (per-squad retreat to friendly smart under power imbalance)
- Memory persistence (extended danger inertion under sustained combat)
- NPC weapon bias (per-NPC callback overrides for loadout selection)

Rejected (not backlogged): combat schemes (per-NPC `combat_type` via condlist). The `script_combat_type` condlist is the most contested combat surface in modpacks - it is GAMMA AI Rework's core mechanism and single-owner by design - and everything a scheme would express is already a maneuver (which forces its action) or a behavior (which composes). A scheme buys nothing here and surrenders composition.

Groomed task entries in `stalker-dev/doc/todo/todo-alifetactics-next.md`; the takeover build plan in `todo-combat-takeover-v2.md`. The 2026-07-02 adversarial review closed 2026-07-08 with all findings terminal; its record lives in stalker-dev git history, and its standing ledger moved to `todo-xlibs-next.md` (x021).

---

## File layout

```
AlifeTactics/
├── doc/
│   ├── architecture.md         (this file)
│   ├── changelog
│   ├── readme.txt
│   └── img/logo.jpg
├── gamedata/
│   ├── configs/
│   │   ├── ai_tweaks/
│   │   │   ├── mod_xr_eat_medkit_at.ltx       # DLTX overlay: vanilla medkit/bandage lists
│   │   │   └── mod_xr_danger_at.ltx           # DLTX overlay paired with the danger fn-patch (at_danger)
│   │   ├── alifetactics/
│   │   │   ├── at_combat_config.ltx           # Combat numeric tunables
│   │   │   ├── at_reaction.ltx                # Reaction + vision curves (MCM slider defaults)
│   │   │   ├── at_ammo.ltx                    # Ammo tunables
│   │   │   └── at_gear.ltx                    # Gear tunables + artefact class tables
│   │   ├── ui/
│   │   │   └── ui_at_stats.xml                # at_hud HUD layout
│   │   └── text/
│   │       ├── eng/ui_st_mcm_at.xml           # English MCM strings
│   │       └── rus/ui_st_mcm_at.xml           # Russian MCM strings
│   ├── scripts/
│   │   ├── _at_deps.script                    # dependency gate
│   │   ├── at_mcm.script                      # MCM configuration
│   │   ├── at_debug.script                    # code-trace primitives (one logger -> alifetactics.log, the on() gate, formatters, level lifecycle)
│   │   ├── at_combat.script                   # Combat > Maneuvers (engine half: GOAP takeover, gates, lifecycle)
│   │   ├── at_combat_doctrine.script          # Combat decision half (catalog, need + destination methods, arbiter)
│   │   ├── at_commitment.script               # Combat > Commitment (action-switch + cover re-pick vetoes)
│   │   ├── at_stance.script                   # Combat > Conduct (cover posture)
│   │   ├── at_accuracy.script                 # Effectiveness > Accuracy
│   │   ├── at_reaction.script                 # Effectiveness > Reaction (aim, lead) + Discipline (fire discipline); vision curves -> Perception > Vision
│   │   ├── at_disclosure.script               # Effectiveness > Disclosure (victim turn + squad investigate)
│   │   ├── at_danger.script                   # Effectiveness > Danger (danger scheme fn-patch)
│   │   ├── at_noise.script                    # Danger > Sound (movement + handling noise hearing)
│   │   ├── at_crossfire.script                # Effectiveness > Crossfire (friendly-fire damage block)
│   │   ├── at_firstaid.script                 # Mechanics > Healing (active first-aid: heal rate, charge, limp/heal anim)
│   │   ├── at_jam.script                      # Mechanics > Jamming (modded-exes xr_weapon_jam override)
│   │   ├── at_ammo.script                     # Mechanics > Ammo (NPC ammo simulation)
│   │   ├── at_gear.script                     # Mechanics > Gear (functional NPC inventory SOURCE -> effects resolver)
│   │   ├── at_effects_resolver.script         # the multi-source effects substrate (resolve/register + the engine writes)
│   │   ├── zzz_at_firstaid_patch.script       # vanilla xr_eat_medkit re-roll suppressor
│   │   ├── at_world_trace.script              # world log: slide watchdog + ballistics recorder -> alifetactics_world.log (MCM Development, off by default)
│   │   ├── at_hud.script                      # live debug HUD (nearby NPC logic/combat/target)
│   │   └── at_test.script                     # console test commands
│   └── textures/
│       └── at_mcm_banner.dds                  # MCM banner
├── LICENSE
└── README.md
```

Namespace: `at_*` (parallel to `ap_*` for AlifePlus, `ag_*` for AlifeGuard, `x*` for xlibs).

---

## User-facing systems

The MCM menu is a six-category gameplay tree plus a Development tab, the canonical structure (source of truth `at_mcm.script`; the page names here are the MCM labels): **Combat** (Maneuvers, Commitment, Conduct, Behaviors), **Effectiveness** (Accuracy, Disclosure, Crossfire, Reaction, Discipline, Range), **Perception** (Sound, Vision, Danger), **Mechanics** (Healing, Jamming, Ammo, Gear), **Effects** (wip), **Mutants** (wip), and **Development** (log level, ballistics debug, world-behaviour debug, debug HUD toggle and position, reset to defaults). Perception is its own category because it is sense-and-reaction, not combat skill: its leaves are Sound (heard gunfire and the movement/handling noise the NPC picks up), Vision (the per-rank vision SPEED and RANGE curves, moved out of Reaction into their own page), and Danger (the distant-hit reaction plus the always-on crash fixes shown as locked toggles and the player-range tuning toggle - the old Hit and Fixes pages folded into one). The distant-hit reaction (duck) lives in Danger; the return-fire-at-reach answer lives in Effectiveness > Range - same scenario, two features, two homes. Behaviors, Effects, Mutants, and the Range page are wip; Commitment, Reaction, and Discipline are built (`at_commitment.script`, `at_reaction.script`), active on exes carrying the n023/n024/n025 binds and inert with an INACTIVE log line on older ones. One visible leaf = one name = one MCM page = one master toggle, and the system sections below follow the category order; the effects resolver is the invisible substrate under the multi-source leaves (Gear feeds it, Vision draws its rank slice from it), with no page of its own. The Scope column is the scope rule below, at a glance.

| System | File | MCM page | Master toggle | Scope | Composition |
|---|---|---|---|---|---|
| Maneuvers | `at_combat.script` | Combat > Maneuvers | `combat_enabled`; per-maneuver toggles | All NPCs | transaction-override |
| Accuracy | `at_accuracy.script` | Effectiveness > Accuracy | per-curve `disp_enabled` / `move_enabled`, no master | All NPCs | callback |
| Disclosure | `at_disclosure.script` | Effectiveness > Disclosure | `disclosure_enabled` | All NPCs | callback |
| Danger | `at_danger.script` | Perception > Sound / Danger | fixes always-on; toggles: Enemy gunfire (Sound), Distant hits + Player ranges (Danger) | All NPCs | scheme-patch |
| Crossfire | `at_crossfire.script` | Effectiveness > Crossfire | `crossfire_enabled` | All NPCs (actor excluded as participant) | callback |
| Commitment | `at_commitment.script` | Combat > Commitment | `commitment_enabled` | All NPCs | callback |
| Conduct | `at_stance.script` | Combat > Conduct | `conduct_posture` | All NPCs | callback |
| Reaction | `at_reaction.script` | Effectiveness > Reaction | per-curve `aim_enabled` / `lead_enabled`, no master (vision -> Perception > Vision, fire discipline -> Effectiveness > Discipline) | All NPCs | callback |
| Discipline | `at_reaction.script` (fire queue) | Effectiveness > Discipline | `discipline_enabled` | All NPCs | callback |
| Vision | `at_reaction.script` (rank) + `at_gear.script` (optics) | Perception > Vision | `vision_enabled` (speed + range curves) | All NPCs | provider -> effects resolver |
| Healing | `at_firstaid.script` | Mechanics > Healing | `healing_enabled` | All NPCs | fn-patch |
| Jamming | `at_jam.script` | Mechanics > Jamming | `jam_enabled` | All NPCs | save-wrap |
| Ammo | `at_ammo.script` | Mechanics > Ammo | `ammo_enabled` | All NPCs | callback |
| Gear | `at_gear.script` | Mechanics > Gear | `gear_enabled` | All NPCs | provider -> effects resolver |

Composition classes, what each does to the surrounding stack: `callback` subscribes to an engine callback and adds to it (composes with any other subscriber); `fn-patch` replaces a vanilla module function, rescheduling through the same lookup so it holds; `save-wrap` saves the prior function and forwards to it when disabled (composes with a prior installer); `transaction-override` suppresses whatever brain is installed, but only for the seconds it holds one NPC; `scheme-patch` installs a generic scheme's binder and evaluators onto whichever file won the MO2 slot, at `on_game_start`, so it layers onto a rival override instead of excluding it (Danger is the one such leaf).

### Scope: every system runs on every fight

No system is actor-gated. The Combat takeover targets whatever enemy the engine selected for the NPC (`best_enemy()` - a stalker, the player, or a mutant); the actor gate it shipped with was removed 2026-07-08 once the measured decide costs showed the complication bought nothing. Disclosure, Healing, Accuracy, and Danger never read who the enemy is. Crossfire, Jamming, and Ammo exclude the player only as a participant (the player's own hits, the player's own weapon) - a participant exclusion, not a target gate.

One community-based exclusion overlays this scope: zombified NPCs (`xcreature.community(npc) == "zombied"`) bail at the entry of Accuracy, Reaction, Disclosure, Crossfire, Commitment's veto, and the Combat Maneuvers (`_can_seize`). A zombie is a mindless shambler, not the deliberate combatant those layers model, so it fires and moves as vanilla drives it - no per-rank curve, no takeover, no shuffle veto. Danger already gates zombied in its vanilla-derived body (the grenade and corpse-alert branches); Mechanics (Healing, Jamming, Ammo) are not zombie-gated, though Healing's limp gate skips zombied on its own.

---

## Combat

Status: built incrementally. On `main`: the GOAP graft (`xcombat.install_takeover`), the maneuver-catalog spine (`at_combat_doctrine`), and the full solo maneuver set - counterflank, reload_cover, flee, retreat, kite, pickoff - with the fire discipline owned by `xcombat.set_combat`. It runs only on fights the actor is party to (2026-07-10; the 2026-07-08/09 any-enemy + seize-radius form is retired); squad coordination and enemy openings remain the open phases. Full phase plan + the decision record: `stalker-dev/doc/todo/todo-combat-takeover-v2.md` (t130-t139 + the Plan section). The full-takeover v1 is preserved on the `combat_takeover` branch.

### Two systems

AT's combat is two independent systems over vanilla, not one:

1. **Maneuvers** - the takeover. AT begins a maneuver on a fighting NPC, drives it to its end, then hands the NPC back. It launches behaviors vanilla lacks or must be forced into (flee, kite, flank, pickoff, suppress). This is most of this section.
2. **Commitment** - the anti-shuffle veto (MCM page: Combat > Commitment). AT never takes the NPC; it vetoes vanilla's own action switches to stop the break-contact shuffle, keeping the NPC on a good action (a player right in front and already being shot) until an important event forces re-evaluation. It runs on every fighting NPC and launches nothing. See "Commitment" below.

The maneuvers impose (block vanilla briefly, run our behavior); the shuffle intervention composes (leave vanilla running, deny only bad switches). The two are separate and can be built and shipped independently.

### Scope: fights the actor is party to

AT seizes only problems the actor is party to - one rule, carried as row data, not as a gate branch. Every catalog row declares whose problem it answers (`vs_actor`): the four fight rows (retreat, flee, kite, pickoff) walk only when the actor IS the NPC's selected enemy (`state.enemy_id == AC_ID`), and counterflank - the actor-party row - walks only when he is NOT (its problem is the actor standing at contact range while the NPC's committed fight is someone farther). NPC-vs-NPC fights are never seized: the triggers that make maneuvers legible - pressing a weapon minimum, chasing a routing man - are things the player does, and an NPC-vs-NPC maneuver reads as generic repositioning (the 2026-07-09 `seize_actor_radius_m` perceivability radius bought a branch, a config key, and a cached distance read on the begin path for that thin payoff; removed 2026-07-10 - the 50m radius survives only as at_firstaid's presentation radius, a different system).

Within a qualifying fight the target is never picked by AT: `best_enemy()` at STAGE time, the same object vanilla combat drives against (counterflank is the one row that stages the ACTOR - the row declares its committed target and the shell stages what the row declares). From staging on, the maneuver is COMMITTED to that target (2026-07-09): `_begin_maneuver` and `_update_maneuver` resolve the staged id via `xcombat.resolve_enemy` instead of re-reading `best_enemy`, so a kiter's LOOK never re-targets mid-maneuver to whoever the brain glanced at (the fire selection stays the engine's), and a dead or despawned target ends the maneuver (reason `target_lost`) with vanilla picking the next fight. A weaponless enemy (a mutant) reads as the rifle range band where a row needs the enemy's weapon, and the reads are otherwise enemy-agnostic. The one other actor read in the decide path is flee's base search scoping to the CURRENT level via the actor's level id - valid because a fleeing NPC is online, and online NPCs are on the actor's level by construction.

### The model: a takeover transaction

Vanilla owns every NPC by default. AT does not run combat; it seizes one NPC for one committed, time-boxed maneuver that TRIES to solve one stated problem, then releases. AT is an interrupt over vanilla, not the combat brain. There is no share knob - an NPC is seized only when a need and a feasibility check both fire for it, so selectivity is inherent. At most one open transaction per NPC, at most `max_concurrent_maneuvers` (20) across all NPCs - a runaway failsafe, never a behavior lever, checked before the catalog walk; with the actor-party scope it should never bind.

The graft is permanent; only the activation is per-seize. `xcombat.install_takeover` plants one evaluator + one action - sharing RESERVED GOAP id **188347** - onto EVERY stalker at spawn (`at_combat._install`, unconditional), and adds a `188347 == false` precondition to the vanilla combat/danger/state/alife chain. The graft then sits dormant: the gate down, `188347` false, vanilla driving. A seize raises the gate (`188347 == true`), which makes the whole vanilla chain unreachable so the grafted action is the only one that can run; release lowers it and vanilla resumes. Grafting on every NPC rather than only seized ones is deliberate - a maneuver begins the instant a need fires, with no per-seize wiring. The id is RESERVED and no other GOAP scheme on the same action manager may reuse it: it was moved off 188200 after that value collided with an external companion scheme's evaluator - because the graft is present even on never-seized companions, the collision silently starved that scheme (a companion frozen out of its own emission-shelter action, traced 2026-08-13).

The transaction law (2026-07-10, replacing the reseize cooldown): a row's need states the FULL problem, and the maneuver's success negates it. Kite's back-off ends outside its own weapon minimum; flee's base is past the enemy's reach by construction; retreat only GROWS the distance - an ~8m rear pull-back cannot exit a rifle's 80m reach term, so a still-hurt NPC under long-reach pressure legitimately re-fires it lap after lap: pressure relief re-applied while the pressure lasts, throttled by the walk cadence and the circuit's displacement gate, not by negation (measured 10 of 19 completed retreats with the need still true, 2026-07-27); counterflank makes the actor SEEN, which flips the engine's own enemy selection; a finished pickoff hold resets the stall measurement, so the next hold needs a fresh 4s standoff. A solved problem reads false at the next begin check and does not re-fire; a recurred problem (the player pressing back inside the minimum) or an unsolved one (a rout that capped in place, the enemy still in reach) legitimately fires again - the trigger is the throttle, and no timer stands between a real problem and its answer. The short circuit is the failsafe for when the model lies: a pick of the same situation with the NPC still within 2m of the previous pick means the last transaction changed nothing (the indoor kite ping-pong between two adjacent vertices, 2026-07-11 GAMMA session - the flee-lane stub was too short to clear the minimum, arrived-but-nothing-moved, 50 kites), and past `circuit_limit` (3) such picks the takeover refuses and vanilla owns him. Displacement is the discriminator, never timing - a legitimate chain under a sprinting player re-fires instantly too, but each transaction MOVED him, which resets the count - and release is world change, not a clock: he moves 2m, the situation changes, or the fight ends. The same displacement-sampling pattern as the stall tracker and the limp drop detectors.

Seizability is a gate separate from the trigger. `_can_seize` (`at_combat.script`) takes an NPC only if it is armed (an unarmed NPC would deadlock, since the engine's own rearm lives in the blocked combat planner), not already in a smart cover (vanilla owns that micro), and not mid-animation (`xcombat.body_busy`: the critical-hit stagger via `xcombat.is_staggering` (the engine's misleadingly named `critically_wounded()` - true only while the ~1s stagger anim plays, not the wounded-down state), a script overlay via `animation_count()`, and - on engine builds exposing the read - the additive hit flinch; a seize writes state and starts movement, which under a playing reaction is the glide by construction, so the animation always finishes first and the begin check re-asks next tick). The same `body_busy` read skips the 200ms state re-apply while a reaction plays, the shape of the existing reloading skip. There is no indoor gate: the old surge-shelter-radius `is_indoor` proxy false-flagged open ground (t97), so the takeover fights everywhere; `xcombat.is_indoor` was rebuilt as a real roof-plus-walls raycast and stays available if a future maneuver needs it.

The glide-stop closes the case those gates cannot reach: the ENGINE's own mover starting a standing NPC mid-hit-reaction (the footless glide - `body_busy` defers AT's writes, but the vanilla planner is engine-side C++ and defers to nothing). The engine half is one neutral lever, `npc:set_movement_hold(bool)` (engine PR, wrapped as `xcombat.set_movement_hold`): while true, `parse_velocity_mask` routes into its Stand branch - speed 0, movement type Stand, path and destination preserved, so the NPC resumes his route on release. Every decision is Lua-side in `at_combat.script`: `_check_hold` holds during the crit stagger, or during the additive flinch on a body already standing (`npc:movement_type() == move.stand` is the standing proxy - Lua cannot read NPC speed, `GetMovementSpeed` is actor-only; a moving NPC is never held, freezing a runner mid-stride is the same artifact from the other side). `npc_on_hit_callback` sets the hold the frame the hit lands (waiting for the next monitor pass leaves up to 200ms of glide - exactly the window), then the monitor pass maintains it and releases the moment the reaction ends; a per-NPC mirror (`state.move_hold`) makes the write transition-only. The engine flag persists while the NPC is online and has no decay, so every path that stops maintaining it clears it first: the monitor release, death, `npc_on_net_destroy`, `server_entity_on_unregister`, and the feature toggle-off - a stale true would plant the NPC forever. On an exe without the bind the wrapper no-ops and vanilla movement (the glide) is the fallback. Debug `[CMB] hold` lines trace each transition with its cause (stagger/flinch/hit/released); off the debug gate the steady-state cost per NPC per pass is one table read and one subtraction.

Three states per NPC, real cost only in the last:
- Not fighting - one `best_enemy` read per begin check, nothing else.
- Fighting, no maneuver open - the begin_maneuver check runs (600ms), engine-memory reads only, no raycasts.
- Maneuver running - the update_maneuver and end_maneuver checks run (200ms each): re-apply the row's state, hand back on arrival, the cap, or a broken premise.

### The decision pipeline

Every maneuver is two methods, split so the need is always cheap and the geometry is always bounded (2026-07-10, replacing the one-row rotation). `check_need` is the need: a compare over memoized reads (positions, distance, health, my and the enemy's weapon kind, the actor position and relation) that states the row's FULL problem and returns the situation name - actor_close, too_close, hurt, stalled - or nil; it is forbidden the raycast/path/search class of work. `find_destination` is the feasibility: it owns the geometry (and the premise reads that cost luabind - flee's and pickoff's under-fire pair, pickoff's actor-aim read) and returns the destination vertex when the maneuver is doable here, or nil to decline - one pass answers both "can he?" and "where to?", and the vertex feasibility validated is the vertex the engine executes (resolve once, never re-resolve).

Per begin_maneuver check (on the monitor pass, 600ms) `resolve_maneuver` arbitrates the catalog in priority order - counterflank, reload_cover, flee, retreat, kite, pickoff - from a per-NPC cursor. A row out of scope (`vs_actor` against the walk's fight class), or whose need is absent, or whose MCM toggle or faction/weapon palette rejects it (each maneuver has its own checkbox on Combat > Maneuvers; a disabled row traces as `off`, an out-of-scope one as `scope`) falls through to the NEXT row in the SAME tick, so a row that does not apply never delays the rows below it. The first row whose need holds runs its `find_destination` - the ONE geometry probe this tick - and ends the walk, pick or decline. In both cases the cursor moves past that row: a declined row (flee under fire, no rear cover) defers to the next candidate and retries after at most one lap (~3s), and a picked row hands the NEXT decision to the row below it - which is what gives a cowardly NPC whose rout was blocked his retreat fallback with no escalation state. The cursor resets to the top when the NPC leaves the fight. Cost per tick is bounded by construction - at most one lap of need compares (microseconds) plus at most one geometry probe - so no budget machinery exists on this path. Row order is the priority: counterflank answers an ignored enemy actor at contact range before anything else, flee answers pressure first for the cowardly factions (a coward runs before he fights; everyone else falls through its palette in the same tick) with retreat the fallback when the rout is blocked, kite answers a violated weapon minimum, and pickoff sits last so a pressed NPC retreats and only the merely-stalled one plants. There is no shared read-all: each row reads its own world, and a value two rows share - the faction, my and the enemy's weapon kind, the npc-enemy distance, the actor position - is memoized lazily on the walk, for that walk only. With debug tracing on, the decision line carries every examined row plus per-layer `need=`/`dest=` microseconds on the row that ran its feasibility - any future cost mechanism on this path (a decline backoff was built and removed 2026-07-08) must first be justified by those numbers. The things vanilla does well - the opener, re-target, search, turret, grenade dodge - are not situations AT answers (the grenade trigger was removed 2026-07-10: vanilla's own grenade reaction is faster than a staged walk-to-cover, and a seized NPC eating a rare grenade was already the accepted trade).

### The flow

One maneuver, birth to hand-back:

```
_run_monitor (a vanilla time event, every 200ms; per frame the whole monitor costs
              ONE timer compare inside vanilla's own ProcessEventQueue walk)
  for each tracked NPC (_npc_states), run the checks due for its state:
  no maneuver open -> begin check (600ms)
      not fighting (no best_enemy, no recent hit)        -> skip
      not seizable (unarmed / smart cover)               -> skip
      concurrency cap reached (max_concurrent_maneuvers) -> skip
      catalog walk from the per-NPC cursor, priority: counterflank -> reload_cover -> flee -> retreat -> kite -> pickoff
          per row: scope (vs_actor vs the fight class) -> toggle ->
                   check_need (need) -> palette -> find_destination (feasibility)
          need absent = next row, SAME tick
          find_destination nil = decline: cursor past the row, tick over
          find_destination vertex = picked: cursor past the row
      picked -> stage row + destination + the row's committed target, raise the gate
  engine plan solve -> grafted action initialize
      -> _begin_maneuver: send_to(dest) + apply_state    (the one engine write)
  maneuver open -> update check (200ms): row.update re-applies fire/posture/movement
                -> end check (200ms): arrived, cap, or broken premise -> _end_maneuver
      -> gate down, cover reservation released, stall tracker reset, vanilla resumes
```

### The maneuvers, and how each one works

The catalog holds six built maneuvers - counterflank, reload_cover, flee, retreat, kite, pickoff, in priority order. The base-of-fire and maneuver-element maneuvers (suppress, assault, push, flank) are later squad phases, not yet in the catalog. Each built maneuver is one row in `at_combat_doctrine.script`: its `check_need` (the need it answers - the FULL problem, whose negation is the maneuver's success), a faction/weapon palette, its `find_destination` (feasibility; where it sends the NPC), an optional `check_end` (the premise re-checked while it runs), and the fire/posture/movement it drives.

Each row opens on its need: **actor_close** (an enemy actor inside `counterflank_actor_dist_m` while the committed target is farther than him) for counterflank, **reloading** (the NPC's own weapon mid-reload, stamped by the n031 engine events) for reload_cover, **hurt** (health below `hurt_frac` AND the enemy inside his own weapon's reach) for retreat and flee, **too_close** (enemy inside MY weapon's minimum) for kite, **stalled** (held position ~4s since the last maneuver, the enemy not closing, and inside MY weapon's EFFECTIVE range) for pickoff. The walk takes the first in-scope row whose need holds, whose palette matches, and whose `find_destination` yields a destination.

| Maneuver | Fires on | Applies to | Runs to | Weapon; move | Ends on |
|---|---|---|---|---|---|
| counterflank | actor_close | any NPC fighting someone other than the actor | holds its own spot, aimed at the actor | fire; still | 3s hold |
| reload_cover | reloading, under fire | any NPC | the nearest cover | weapon up, no fire; run | arrival, reload done, or 8s |
| flee | hurt, enemy in reach, as the last man, once the shooting pauses | cowardly factions, their first answer | a friendly base 100m+ away, rear-biased | holstered; run | arrival or 20s |
| retreat | hurt, enemy in reach | brave factions; cowardly ones when their rout is blocked | cover behind him, never closer to the enemy (reserved) | fire; walk | arrival or 8s |
| kite | too_close | any NPC | a clear back-lane, a weapon-set distance to the rear (shotgun 4m .. sniper 10m) | fire; walk | arrival or 8s |
| pickoff | stalled, comfortably past the enemy's EFFECTIVE range, not under fire, not under the actor's aim | any NPC | holds its own spot | deliberate single shots; still | 8s hold or broken premise |

- **counterflank** is the actor-party hold: an enemy actor inside `counterflank_actor_dist_m` (5) while the NPC's committed target is FARTHER than him - he is shooting past the man who can kill him first, the enemy_manager's seen-now scoring artifact (a seen distant target outranks the unseen man on his shoulder, enemy_manager.cpp:110-175). The row stages the ACTOR, aims at him for a 3s hold with no movement; the turn makes him SEEN, and seen at contact range wins the engine's own selection outright, so at hand-back `best_enemy` IS the actor and vanilla drives the new fight - the transaction's success is the engine changing its mind. No feasibility geometry: the old wall raycast was structurally blind across the whole 5m trigger radius (its clearances cannot report an obstacle below ~3.75m) and is removed; the need is pure math over the walk memo plus one relation read paid only inside the radius. OPEN DEFECT, root cause diagnosed 2026-07-28: the NPC is sight-GLUED to his committed enemy. Every counterflank update sample in the diagnostic session (16 of 16, 2 holds) shows the sight manager serving the OTHER enemy's id - `stype=4` (object sight) or `stype=9` (fire-object sight, the weapon-fire class) - never the staged actor, so the ordered look never wins arbitration, the turn never starts, and the direction gate (state_mgr.script:318) withholds every shot (56 of 119 holds ended cap-with-0-shots, 2026-07-27). Turning speed and geometry are exonerated (stand-in-danger body turn runs at a full rotation per second, parse_velocity_mask; `select_speed` never slows large angles, sight_manager.cpp:80-100). Open question: who re-asserts the foreign object sight during the hold and what beats it - no trigger suppression as a workaround (user ruling 2026-07-28). Universal palette; only fires when the NPC is already fighting someone else, so player stealth against idle NPCs is untouched.
- **reload_cover** is the vulnerable-window reposition (the n031 consumer): a stalker caught reloading in the player's line runs to nearby concealment (`xcombat.find_cover` with `selection = "nearest", firing = false`: the nearest reachable cover that hides him from the enemy, not the best-hidden one anywhere in the radius) instead of standing in the open, and the engine finishes the reload on its own timer during the move. The need is a pure-Lua read of the reload stamp the n031 engine events write (`npc_on_weapon_reload_start`/`_stop`, PR #611, subscribed through `xcombat.on_reload_start`/`on_reload_stop`); feasibility re-confirms with the authoritative `xcombat.is_reloading` (`get_state() == 7` - `eReload` is 7, not the 5 the old at_jam trace compared, which is `eFire`), requires the exposure gate - `xcombat.actor_can_hit`: the player holds a firearm, faces him within `reload_cover_cone_deg`, and the shot line between them is clear (one facing read + one raycast, paid only at reload feasibility). The leading indicator, and the only evidence that matches the maneuver's player-only scope: the earlier hit/heard-shot/danger stack counted fire from ANY source (third-party crossfire fed a player-scoped decision) and reacted only after the danger had already materialized. Reloading out of the player's line declines and vanilla reloads in place - and claims the cover vertex like retreat. Its `check_end` hands back the moment the reload finishes - the gun is up again and vanilla's own fire beats finishing our walk at weapon-up. It runs weapon-up-no-fire (a reloading gun cannot shoot) at a run (the window is ~2s). Universal palette; sits directly under counterflank because a reloading NPC under fire is at his most vulnerable, outranking the pressure rows. On an exe without the reload events the stamp never sets and the row is inert (INACTIVE logged once at boot).
- **retreat** is the standing-line pressure response: badly hurt with the enemy inside his own weapon's reach, pull back to cover BEHIND you while still firing - the search centers 8m to the rear - deeper than the 7m cover-search radius, so the circle excludes the cover the NPC already holds (at 4m best_cover kept re-picking his own cover and every retreat declined toward_enemy, 2026-07-11) - and the winning vertex must be farther from the enemy than the NPC stands, so a retreat always grows the distance; toward-the-shooter cover declines instead. It reserves the cover vertex so two NPCs never pick the same spot. Factions: army, dolg, freedom, killer, isg, stalker, bandit, greh, plus the cowardly factions as their FALLBACK - a hurt ecolog whose rout is blocked fights from cover instead.
- **flee** is the rout, and a rout is for the broken: hurt with the enemy in reach, only the last man (squadless, or sole survivor of his squad), and only once the shooting pauses - a fresh hit or a perceived shot from that enemy (`xcombat.is_under_fire`, the NPC's own danger memory) declines it, so nobody turns his back mid-burst; the declined row defers and is re-asked a lap later, so the rout fires when the fire lifts. He runs HOLSTERED to a friendly base (no enemy squad stationed) at least 100m away, biased to the rear hemisphere so every stride gains distance, and declines when no such base is reachable. Factions: ecolog, clear sky, renegade. Monolith and zombied get neither retreat nor flee - fanatics do not break.
- **kite** answers the enemy getting inside your minimum range: back off a weapon-set distance (`KITE_BACK`: shotgun 4m, pistol 5, SMG 6, rifle 8, sniper 10), still firing the whole way - a visible fighting withdrawal that always ends well outside the weapon's minimum. The old form backed off "minimum minus current distance", which in practice was a 2m hop; replaced 2026-07-09. Universal - any faction, any weapon; a gun inside its minimum is half useless no matter what the enemy holds. The back-off negates `too_close` by construction, so a completed kite does not re-fire - and the player pressing back in re-creates the problem, so sustained pressure produces kite after kite with no dead window. `find_flee_lane` runs a two-phase search - straight-back then +-45 then +-90 at the full distance, the same fan at half distance, then the longest CLEAR straight stub (raycast-validated like the fan phases) - and declines only when fully boxed in; the destination is accepted only if standing on it puts the enemy back OUTSIDE the weapon's minimum (destination-to-enemy > min, measured in `_find_kite_lane` at decision time), so arriving negates `too_close` by construction and the arrived-but-nothing-moved ping-pong cannot be accepted.
- **pickoff** is a deliberate-fire hold, not a movement: any stalker who has his enemy outranged in a standoff stands his ground and picks him off with single aimed shots. The range term reads EFFECTIVE range - where the enemy's weapon is still genuinely dangerous, not where bullets stop existing (a shotgun blast at 25m still arrives, spread thin) - so the advantage is statistical, never immunity, and the break-offs stay strict. Range hysteresis, two thresholds: he plants only comfortably past the enemy's effective range (`pickoff_enter_factor`, 1.2x), and the hold ends when the enemy closes back inside 1.0x - between the two nothing flaps. He also needs the enemy inside his OWN weapon's effective range, and to be unbothered: under fire or under the actor's crosshair (`pickoff_targeted_deg`) it declines, and the same premises re-check while the hold runs (`check_end`) - a hit landing, the player's aim arriving, or the enemy closing in ends it now, not at the cap. His fire is 1 round per pull at a deliberate pause rolled fresh each shot (`pickoff_interval_min/max_ms`, the engine's own `[fire_queue_params]` sniper-band rhythm), tightened by the rank factors; his accuracy is the rank dispersion curve from a still stance - no engine cheat mode (the `sniper_fire_mode` story-scene flag is deliberately unused, see `npc-combat-effectiveness.md`). A finished hold resets the stall tracker, so the next plant needs a fresh 4s standoff - vanilla owns the gap. The fire discipline downgrades it to weapon-up (no shot) when there is no clear line, so it never fires at a wall.

flee sits above retreat in the catalog (2026-07-11): a coward runs before he fights, so a hurt ecolog asks the rout FIRST, and a blocked flee (squad stands, under fire, point-blank, no base) moves the cursor to retreat - he fights from cover only when he cannot run, with no escalation state. Brave factions lose nothing to flee's position (its palette rejects them and the walk falls through in the same tick). When a palette matches but every feasibility declines, AT leaves that NPC to vanilla.

### The advantage rules

A need says a situation exists; it does not say the maneuver pays. Each maneuver therefore carries rules - enemy-related (distance, the enemy's weapon, the actor's aim) and squad-related (last man, a standing line) - so that running it is an advantage for this NPC here, not a reflex. All of them live in the `find_destination` methods (the seize path) or, where the rule is pure math, in the need itself, so they cost nothing on the monitor and each traces its pass or decline when debug is on (`can_counterflank` / `can_kite` / `can_pickoff` / `can_retreat` / `can_flee` lines): kite refuses any destination that would leave the enemy still inside its weapon minimum, pickoff refuses to plant inside 1.2x the enemy's effective range, under fire, or under the actor's crosshair, retreat refuses cover toward the shooter, flee refuses to rout under fire or while a squadmate stands. The reads are the `WEAPON_RANGES` table (both sides' weapon kinds are cached reads), the squad member count, the NPC's own danger memory, and the actor's facing - no raycast in counterflank, nothing per frame. Later maneuvers (suppress, assault, flank) get held to the same bar at design time.

### Flee hand-back

Flee holds no enemy suppression and no clock of its own; it is a generic maneuver like the other three. While it runs, the GOAP graft already blocks vanilla combat, so the NPC cannot turn and fight (the holster re-assert keeps the weapon down and the facing on the run path). It ends on the same conditions as any row - arrival at the base, the timeout cap, or a lost target - and hands back with all its state cleared.

On hand-back the engine decides, and the disengagement gates are these: against a MONSTER enemy, the engine's own `max_ignore_distance` (75m, `m_stalker.ltx:415`, applied in `CEnemyManager::useful` for the stalker-vs-monster clause only, enemy_manager.cpp:76-82); against another STALKER, the script-side 100m cutoff in whichever `xr_combat_ignore.script` won the MO2 slot (vanilla `:245`, `dist > 10000` squared, inside the engine's `useful_callback`); against the ACTOR there is NO unconditional distance rule (vanilla limits actor fights to 100m only at night or in rain, `xr_combat_ignore.script:229-231`) - a daytime flee from the player relies on the NPC having run holstered and blind, so the memory decays unrefreshed. A flee that reached its 100m+ base clears the first two gates outright. A flee that capped in place (boxed in, or the base too far to reach online) hands back next to the enemy and fights - and since the need (hurt, enemy in reach) still holds, he attempts escape again when the feasibility gates allow: the correct read of a cornered man, not a bug. There is deliberately no post-hand-back ignore window; the earlier `flee_enemy` / `flee_until` hold was removed (2026-07-10) because it existed only to make a failed escape look like a clean one.

### The maneuver pattern

Every maneuver sets its state ONCE when AT takes the NPC (the GOAP action's `initialize` - the single place AT writes the engine, setting the move and fire state), carries an `ends_on` (`arrival` for movers, `time` for holds) and a `max_ms` cap. The GOAP graft `execute()` stays empty (no per-frame code). The destination is set once and the engine walks the NPC there; the combat state is re-applied by the row's `update` (`doctrine.apply_state`) on the 200ms update_maneuver check. The fire DISCIPLINE lives inside `xcombat.set_combat` (2026-07-10; AT holds zero fire logic - a row declares only its INTENT: FIRE, SNIPE, READY, STOW): a reloading weapon degrades the intent to READY first - a fire goal issued mid-reload CANCELS the engine reload (proven 2026-07-21: chained maneuver applications killed reloads at ~116ms held, leaving NPCs racking empty guns), so the discipline lets the reload finish and fire resumes on the next periodic re-apply - then a FIRE/SNIPE intent follows vanilla `kill_enemy`'s order - an enemy SEEN right now is fired on with no further gate (no distance term - point-blank fires; `fire_make_sense`'s 2.5m bail is a smart-cover rule that must never gate a seen enemy), only the blind case consults `fire_make_sense` (its occlusion pick stops shooting the wall he ducked behind, its 10s automatic-weapon window sustains suppression at last-known), else the intent degrades to READY, weapon up, eyes on the enemy, until sight returns. A `can_kill_enemy` gate on the seen branch was tried 2026-07-11 and reverted 2026-07-12 on measured + source evidence (t152): the engine itself never gates fire on that read - its sole consumer is sight AIM-POINT selection (`sight_action.cpp:408`; when the clear-shot check fails it re-aims at the target's visible point for 1500ms), and while walking the ray follows the head's CURRENT sight angles, which lag a strafing target - so the gate muted fire through the engine's normal lag-and-displace windows (73% of reads blocked, many at body-angle 0). The read is an aim-quality question, valid on an NPC whose vanilla planner is aiming - which is why Commitment may use it and the takeover fire path never does. The periodic work is a catalog property: each row declares its own `update` (or none), never a fire-keyed branch in the shell.

The burst shape is the second half of that discipline: every state without an animation-specific entry falls to the generic `{5,300,0}` override (`state_mgr.script:322`) - full-auto on a pistol, a burst on a sniper rifle. `at_combat.script` monkey-patches `state_mgr_weapon.get_queue_params`: for a maneuver-driven NPC each query ROLLS burst size and pause fresh inside the engine planner's own per-weapon `[fire_queue_params]` medium band (`xcombat.FIRE_QUEUE`, `m_stalker.ltx:826-909`) - the same per-burst variance the vanilla planner has - then multiplies by the Fire Discipline rank factors, read through `at_reaction.queue_scales` (one owner of the tier tables, toggle-governed, pure Lua so it works on every exe; `max(1, ...)` rounding keeps 1-round bursts alive). The pickoff row swaps the weapon band for its own single-shot band. This is what closes the Fire Discipline gap in "The rank curves under a maneuver": both fire paths obey the one rank model. Animation-tuned states keep their values, the `aim_time` machinery runs unchanged, NPCs outside a maneuver pass through byte-identical. Tunables: `at_combat_config.ltx` (`queue_w_*` pins a kind to fixed values; `pickoff_interval_min/max_ms`). Traced per application as `[QUE]` with the row and the rolled, scaled pair.

Flee is the against-the-grain maneuver (face away, weapon down); its update is a BLOCK - keep the weapon down - not a sight re-drive. The takeover block stops `kill_enemy` from aiming, but that alone does not hold: set once, the weapon comes back up and the NPC re-aims (the observed bug). So flee re-asserts the HOLSTERED `sprint` state (weapon strapped = physically cannot aim or fire) on the update_maneuver check, throttled - 200ms is enough; demonized re-applies it every frame (`demonized_stalker_aoe_panic.script:327`). A holstered weapon with no target faces the run path on its own; AT never steers the sight, it just keeps the gun down. Flee routes to a non-hostile friendly base or smart at least `flee_base_min_dist_m` (100m) away - a real rout to safety, not a near hop (`_find_flee_base` → `find_friendly_base` biased to the rear hemisphere; it declines when none qualifies). This is exactly the demonized / redone panic mechanism: block the same combat planners AT blocks (combat/danger/xr_danger/state_mgr+2/alife, `demonized_stalker_aoe_panic.script:426`), re-assert `sprint`, route far away. Flee is not the only per-period case: every current row's `update` is `apply_state` - a firing maneuver re-checks its shot, flee re-asserts the holster. The GOAP `execute()` stays empty for all of them; the periodic work runs on the update_maneuver check (the monitor pass), not the GOAP action.

The monitor watches the end on the end_maneuver check - `is_arrived` (over `path_completed`) for movers, elapsed time for holds, the row's `check_end` premise where one is declared (pickoff) - and ends the maneuver, handing the NPC back. The monitor is `_run_monitor` (`at_combat.script`), one vanilla time event every 200ms walking `_npc_states` - the store `_install` creates on spawn and the destroy/unregister handlers drop, so the walk touches only NPCs AT tracks, resolved per pass via `level.object_by_id` (nil-guarded; a gone id vanishes with its state). Every timed operation is a registered check row in `CHECKS` with its own period - begin_maneuver (no maneuver open, 600ms - quantized to the pass, the config states the truth), update_maneuver and end_maneuver (a maneuver running, 200ms each, the few NPCs mid-maneuver, so hand-back is prompt). Per frame the whole monitor costs one timer compare inside vanilla's own `ProcessEventQueue` walk (`_g.script:364`, driven from the actor update at `bind_stalker_ext.script:26`); time events do not survive a save load, so `_start_monitor` arms it at `actor_on_first_update`, and a disabled feature stops the pass entirely (`_apply_enabled` ends every open maneuver, then `_stop_monitor`). Aborts come free from the action's preconditions failing (alive, armed, not wounded, has-enemy); a held NPC simply eats a rare grenade rather than AT re-implementing vanilla's reactions. A mover's end position is sticky for free; a held mode reverts on release.

### Commitment

The second system, separate from the maneuvers: the anti-shuffle veto (`at_commitment.script`, MCM Combat > Commitment, built 2026-07-11). The engine keystone is merged (the `npc_on_combat_action_switch` veto, demonized n023); the subscriber registers through `xcombat.on_action_switch`, which probes the seam and reports INACTIVE on exes without it. The problem it fixes: vanilla ties sustained fire to being in cover - `kill_enemy` requires `InCover = true` (`stalker_combat_planner.cpp:342-344`) - and the best-cover point keeps re-picking, so NPCs break contact and shuffle toward sketchy cover even while they are winning the shooting. The shuffle intervention fixes this in place, with no takeover. The deny rule holds three transitions, each proven from the planner preconditions (the census-derived table below, 2026-07-15). Two are fire-action exits - `kill_enemy` / `kill_if_not_visible` toward `take_cover` or `get_ready_to_kill` - held while the NPC still SEES its enemy (`visible_now`, the exact bit `kill_enemy` fires on, `stalker_combat_actions.cpp:483`) and no non-enemy blocks the lane (`can_kill_member`). The `-> take_cover` hold carries one extra gate, `fire_make_sense` (`ai_stalker_fire.cpp:894-937`): it holds only when the aim-direction firing lane is clear out to the enemy (`pick_distance` is a raycast to the first occluder, `:684-808`) and the floor gap is reasonable, because a held fire action fires on `visible_now` even at an occluded target, and pinning an NPC to shoot its own cover is worse than letting it reposition. The third hold is the flank: `detour_enemy -> take_cover`, held only while the NPC is BLIND (`detour` runs at `SeeEnemy=false` by precondition), released the instant the enemy is re-seen - the inverse of the fire gate. `can_kill_enemy` is never a gate: it raycasts along the gun's CURRENT aim direction (`g_fireParams`, `ai_stalker_fire.cpp:811-824`), so it reads false through the aim-lag of a moving target (the t152 mute-fire finding), and survives only as a debug read on the hold line. The cap (`commitment_hold_s`, `_hold_cap_ms`) is a TIME-TO-LIVE on the refusal, not a commanded hold duration: the veto is a bare `if` in the action-switch hook that returns `allow = false`, and it keeps returning false for every proposed cover-seeking switch as long as the advantage holds. The cap is only the ceiling on how long ONE continuous refusal may last before the veto relents and lets a cover move through (so cover quality is allowed to matter again eventually); it never counts down to a forced release. A hold ends the instant any condition falsifies (sight lost, a teammate crossing the lane, the engine stops proposing the switch) - usually well under the cap. Set the cap high and block unconditionally and the NPC holds his firing spot indefinitely; the timer is a relent valve, never the hold length. A recent-hit standdown was tried and removed 2026-07-12: "any hit in the last 3s permits leaving" kept the veto disabled in the one fight it exists for, the one the player is shooting in. Every other transition passes untouched; release needs no machinery because each condition falsifies itself. The debug traces carry the tuning evidence: declines (which condition blocked a would-be veto), the transition histogram (what real shuffling consists of), and the best-cover re-picks.

#### The watched set: which transitions are vetoed, and why

The watched set was chosen from a real transition histogram (one GAMMA bench, vanilla brain, actor as the NPC's enemy, 2026-07-13; the structure drives the decisions, not the exact counts). "From-state fires?" is the load-bearing column: only an action whose `execute` calls `fire()` is worth holding, because a held action keeps executing (the engine proof is in `doc/library/modding/stalker-combat.md`, "Holding an action").

| Transition | Count | From-state fires? | Decision |
|---|---|---|---|
| take_cover -> kill_enemy | 177 | arriving to fire | pass; this is the return to fire |
| get_ready -> take_cover | 118 | no; `InCover` already false | never veto; deadlock |
| kill_enemy -> get_ready | 88 | yes; leaving fire | VETO; the prize |
| take_cover -> look_out | 68 | lost-sight cycle | pass |
| look_out -> hold_position | 38 | `SeeEnemy` false | pass |
| take_cover -> get_ready | 19 | no | pass |
| look_out -> get_ready | 18 | `SeeEnemy` false | pass |
| sudden_attack -> take_cover | 14 | ambush | defer |
| kill_enemy -> take_cover | 7 | yes; leaving fire | VETO; InCover drop while seeing |
| kill_enemy -> crit_wounded | 5 | wounded | never veto |

The rules that explain every row, from the 2026-07-15 census (1468 switches, which widened the table above to ~42 transitions; the shape held):

1. **Hold the two fire-action exits.** `kill_enemy` / `kill_if_not_visible` fire on `visible_now` regardless of `InCover`, so holding one keeps the NPC shooting. `-> take_cover` (`InCover` dropped on a best-cover re-pick) holds on `sees + fire_make_sense + clear lane`; `-> get_ready` (`ReadyToKill` dropped on reload/empty/misfire) holds on `sees + clear lane` and reloads the NPC in place (the object handler reloads independent of the combat action) instead of the `get_ready -> take_cover -> kill_enemy` detour. The `fire_make_sense` gate on `-> take_cover` is the clear-lane refinement: hold only when the shot path is unobstructed to the enemy, and let an occluded NPC (one shooting its own cover) take the cover move.
2. **Hold the flank.** `detour_enemy -> take_cover` abandons the engine flank for the cover shuffle. `detour` runs blind (`SeeEnemy=false` precondition), so hold while STILL blind and release on re-sight - the inverse of the fire gate. No arrival read is needed: a completed detour advances to `search` (allowed), and the cap relents a stuck flank.
3. **Never veto `get_ready -> take_cover`.** `get_ready_to_kill` set `InCover = false` and `take_cover` is the ONLY action that restores it; blocking that edge strands a non-firing NPC that can never satisfy `kill_enemy`'s `InCover` precondition. The shuffle is caught one hop upstream at `kill_enemy -> get_ready`, so `get_ready` is never entered and this edge is never proposed.
4. **The rest passes for free.** The lost-sight cycle (`take_cover -> look_out`, `look_out -> hold_position`, the `-> get_ready` edges out of non-fire states) runs with `SeeEnemy = false`, so the `sees` gate declines it. Survival and opportunity transitions (`-> crit_wounded`, grenade dodge, `-> retreat`, `-> get_distance`) leave from a non-fire action (holding buys no fire) or an unwatched from-action, so they never enter the hold path. `sudden_attack -> take_cover` is deferred: low volume, ambush semantics not yet analysed.

The debug traces make the veto legible per transition (noop when the log level is below DEBUG): the hold line names the refused edge (`op=kill_enemy refused=get_ready`), the release and cap lines carry a per-edge refusal breakdown (`refused=get_ready=4 take_cover=1`), the decline line names which advantage condition blocked a would-be veto, and the transition histogram logs every real action change so the watched set stays chosen from evidence.

#### The cover re-pick veto: denying the cause

MCM: `commitment_cover_pin`, its own toggle under the Commitment master (so the switch-seam veto and the cover pin can be compared in isolation). The second seam (`npc_on_best_cover_repick`, the n029 veto, PR #607, unmerged): the `kill_enemy -> take_cover` shuffle STARTS at a best-cover re-pick - `best_cover()` finds a different point, `on_best_cover_changed` clears `InCover`/`LookedOut`/`PositionHolded` (`stalker_combat_planner.cpp:58-64`), and only then does the planner propose the exit the switch veto refuses. The engine callback fires BEFORE that reset, from the sole `on_best_cover_changed` call site (`ai_stalker_cover.cpp` `best_cover()`), and `flags.allow = false` keeps the held cover - so `InCover` never drops and the exit is never proposed: the cause denied instead of the symptom held. `_on_cover_repick` (`at_commitment.script`) denies under the SAME advantage gate as the fire-exit hold - sees + friendly lane clear + `fire_make_sense` - and only while the NPC's current combat action is a fire action. The callback hands over only `(npc, flags)`, so the current action is shadowed per NPC from `npc_on_combat_action_changed` (`_cur_op`); re-picks fire exclusively from the running action's `execute()`, which is also why every other caller of `best_cover()` - `take_cover`'s own walk, `look_out`, `hold_position`, and `hide_from_grenade` - keeps vanilla re-picks: denying during a grenade dodge would pin the NPC's grenade cover to its old fire spot. A continuous cover refusal is bounded by the same `_hold_cap_ms` relent valve as the switch hold (`_cover_hold`, closed by any actual cover change via `npc_on_best_cover_changed`); the cap matters because two re-pick drivers are standing state, not events - an enemy inside 3m of the held cover (`MIN_SUITABLE_ENEMY_DISTANCE`, `ai_stalker_cover.cpp:237`) and a smart cover whose loophole is gone re-invalidate on every action execute while denied. A maneuver-seized NPC never reaches this seam (blocked planner, no combat action executes), the same exclusivity as the switch veto. One engine path deliberately bypasses the veto: the actuality check's advance search (`ai_stalker_cover.cpp:278`) can swap the held pointer toward nearer cover without firing `on_best_cover_changed` - it resets no props, so it is not a shuffle driver, and the veto's guarantee is "no props reset without consent," not a frozen pointer. On an exe without the seam the registration probe fails and Commitment logs the cover pin INACTIVE, holding the shuffle at the switch seam only.

It runs on the engine's action-switch veto (`npc_on_combat_action_switch`, the demonized keystone n023): the callback fires before the combat planner swaps actions, and AT returns `allow = false` to keep the current action. So AT denies a switch when the NPC is doing something good - a player in front and already being fired on, a chosen action mid-run - and allows it only when an important event (a hit taken, a grenade, the enemy lost) warrants re-evaluation. The rule lives entirely in the callback: "never switch away while X holds," or "only switch under our conditions."

The veto only HOLDS the current action; it can never make a new one start, so it launches no maneuver. That is the whole distinction from the takeover: the maneuvers select and run a behavior, the shuffle intervention only keeps vanilla committed to one it already picked. It runs on every fighting NPC, seized or not, and composes with modpacks (it denies switches, it grafts nothing) - including GAMMA AI Rework, whose camper action lives in the same planner and is untouched by a veto that only refuses switches. This is what makes vanilla's own maneuvers commit and shrinks the takeover to the behaviors vanilla genuinely lacks.

### Conduct: cover posture

The third combat system (`at_stance.script`, MCM Combat > Conduct): small habits applied to VANILLA-driven NPCs at moments the engine already decides - no takeover, no held actions. The engine fires `npc_on_combat_set_body_state` whenever a vanilla combat action sets the body state (the `COMBAT_BODY_STATE_OVERRIDE` forwarder in `callbacks_gameobject.script`; a maneuver-held NPC never reaches it, his planner is blocked and his row sets posture - clean exclusivity for free). The subscriber overrides the posture for long-weapon carriers (`w_rifle`/`w_sniper`) of experienced tier and up, on `hold_position` ONLY - the one op whose ask is provably stationary (it sets movement STAND in the same initialize, stalker_combat_actions.cpp:843-850). The 2026-07-28 audit of every ask site removed the other three: take_cover asks per execute only while FAR from cover (the walking phase, engine proposes STAND for speed, :575-585), get_ready asks while walking/running to position (:362-380), and look_out asks once then MOVES to the lookout spot (:704-709) - answering crouch in any of those slows the engine's own move (never slow a mover, user-ruled). A flat distance floor keeps it a firing-line behaviour: it crouches only past `CONDUCT_FLOOR_M` (20m), so a close fight stays standing and mobile instead of crouching at the actor's feet. Flat, not a fraction of weapon range (the earlier 0.3x-effective form, 2026-07-29): the mobility boundary is the same absolute distance for every weapon, and the fraction sent the sniper floor to ~45m - exactly where a sniper benefits from crouch most, the inversion. The decision is HELD, not per-ask: the engine re-asks its combat body state several times a second while an action runs, and the crouch-shot line toward the enemy flips clear/blocked second to second, so a stateless per-ask decision flapped crouch/stand continuously (1658 overrides in one field session, 2026-07-23 - the up-down shuffle, with the T-pose and glide artifacts of posture contention under hit animations). The built form re-decides once per `DECIDE_MS` (3s), held purely by TIME (an op-change early re-decide was tried and removed the same day: combat ops ping-pong several times a second and bypassed the hold), by casting a crouch-eye shot ray to the enemy (`xcombat.has_obstacle_to_target` at `xcombat.BODY_Y.CROUCH_EYE`, the real line-of-fire test, not the baked cover field): crouch only when that line is CLEAR so the low stance steadies the aim, stand when it is blocked because a low wall would eat his own shot. A per-NPC disposition roll (`CROUCH_CHANCE` 0.30, cached once) crouches only some eligible stalkers, so a firing line mixes standing and crouched shooters instead of the whole squad dropping at once. Every ask during the hold answers the held posture, so the applied value never alternates. Both directions override vanilla's blind pick - its crouch behind a random bump (the 1.0.0 `at_stance` bug: it crouched by op alone, never checking the geometry, and NPCs fired into the bump) and its standing tall in the open where a crouch would steady the aim, wasting the largest legitimate accuracy gain the engine has (stillness + crouch, `m_stalker.ltx:476-483`). Zombied are controlled to a forced STAND at `hold_position` regardless of weapon or tier - a mindless shambler never holds a deliberate firing-line crouch - as a held decision through the SAME freeze/hold/trace path as an eligible NPC (a second control policy, not a special-case bypass: the `want` is forced to STAND instead of read from the crouch-shot ray, everything else is shared, so a zombie mid-flinch holds its posture like anyone else). The danger-duck reaction (`at_danger`, a direct `set_body_state` on a startle) is a reaction not deliberate posture and is left intact. Traced `[STANCE]` per override, debug-gated. On an exe without the seam the callback never fires and the module is inert.

### The rank curves under a maneuver

A maneuver changes who drives movement and fire intent; it does not exempt the NPC from the skill model. Four of the five rank curves apply at the engine-parameter or per-shot-callback level and reach every NPC shot regardless of what drives it: Accuracy dispersion and Moving Fire (`npc_shot_dispersion`, fired inside the engine's weapon-accuracy calculation on every shot), Tracking Speed and Target Lead (per-NPC engine aim parameters via `set_aim_params`), and Vision Speed (the `get_visible_value` multiplier). A novice under a maneuver keeps its novice-tier dispersion, tracking, lead, and vision.

The fifth curve - Fire Discipline - reaches maneuver fire through a second application point: it scales the ENGINE combat planner's burst picker (`set_fire_queue_scale`) for vanilla-driven NPCs, and maneuver fire - which runs through `state_mgr_weapon.get_queue_params` instead - gets the same per-tier factors multiplied onto at_combat's rolled bands via `at_reaction.queue_scales`. One rank model, two application points, no duplicated tier tables (see "The maneuver pattern", burst shape).

### Squad coordination, decentralized

A later squad phase, not yet built: the base-of-fire and maneuver-element maneuvers it needs (suppress, assault, push, flank) are not in the catalog. The design: a squad fighting the player coordinates fire and movement, but with no central brain. Role eligibility is biased by `get_squad_ordinal`; a maneuver-element's viability requires that the enemy is being pinned, so one NPC flanking alone (a death wish) cannot fire. The reliable pin signal is an AT NPC committed to a base-of-fire maneuver; the coordination is emergent from each NPC's local read of its squad.

### The GOAP graft (the control point)

The graft adds one evaluator and one action per stalker and `world_property(EVAL_ID, false)` as a precondition on each entry of `xcombat.get_blocked_planners()`. While the per-NPC gate flag is true the vanilla combat/danger/alife chain is gated off and the grafted action is the only producer of the `EVAL_ID=false` the brain now requires, so it runs; clear the flag and vanilla resumes. The graft mechanism is encapsulated in `xcombat.install_takeover(npc, spec)` / `release_takeover(npc)`, where the spec is `{ gate, on_begin }` - the gate flag the evaluator polls and the one-time maneuver start the action's `initialize` calls; AT owns the spec, xcombat owns the GOAP classes.

### Layer arbitration

Vanilla orders its own schemes against each other with explicit cross-preconditions in `configure_actions`; AT's Combat layers need the same rule, stated before the Behaviors leaf gets its first action. The rule: **maneuvers outrank behaviors.** `xcombat.get_blocked_planners()` lists vanilla planner ids only, so a future Behaviors-leaf injected action would not be blocked while a maneuver holds the NPC - the solver could route through the behavior instead of the takeover action. Enforcement is one condition per layer: a Behaviors action's evaluator returns false while the takeover gate is up (`at_combat.get_maneuver(id)` non-nil). The rule binds now; the enforcement code is built with the first Behaviors graft.

A structural fact that falls out of the same block: Maneuvers and Commitment are mutually exclusive per NPC per moment by construction. The n023 action-switch veto (`npc_on_combat_action_switch`) fires only when the combat planner proposes swapping actions, and a blocked planner never switches - so the veto never fires for a seized NPC. Consequence: Commitment cannot police takeover quality; a takeover's fire discipline belongs at the maneuver's own decision points (`apply_state`), never at the veto seam.

### xcombat boundary

AT owns what to do; xcombat (xlibs) owns how to issue it to the engine. Every NPC command and read - weapon state, aim, movement, cover and clear-shot search, the line-of-fire and memory reads, arrival, the cover reservation, the enemy-state reads - goes through an xcombat primitive; AT makes no raw engine combat call. New primitives for this rebuild: `install_takeover`/`release_takeover`, `is_arrived`, `is_reloading`, `is_bleeding`, `is_moving`, `get_health_frac`, and suppressive fire via `set_combat`; the rest is reuse.

One deliberate future exception would live outside xcombat: a sniper-reach extension that forces `is_enemy = true` so a planted sniper can engage past the engine's own enemy-distance gate. It is NOT built. Today the pickoff maneuver only fires against an enemy the engine has already selected as `best_enemy`, so its reach is bounded by the engine's `is_enemy` range - real, but shorter than a sniper's effective range. If added, it would sit on the `on_enemy_eval` engine callback seam and AT would register and own it directly (like `npc_on_hit_callback`), not as an xcombat primitive: xcombat stays stateless by design - it holds no live-event callback or ownership table on its own behalf, so a stateful `set_enemy_eval` would break that contract.

### Identity and rejected alternatives

The identity the takeover is built for: recognizable committed maneuvers, composition under modpacks (it overrides zero combat files), and solving the shuffle (vanilla twitching between actions instead of committing). Everything above serves those three; a change that trades any of them away is out of scope.

The continuous `script_combat_type` scheme (GAMMA AI Rework, ReDone Combat AI) is the rejected alternative, and a 2026-07-05 read of both confirmed why: it owns an NPC's whole combat single-ownedly, and either does less than vanilla (GAMMA's thin camper sets one state) or reimplements it worse (ReDone's fat `get_combat_movement` and global-cvar aim). The intermittent takeover borrows an NPC for one maneuver where vanilla is weak and hands back, so vanilla's own aim, fire discipline, cover cycle, and squad coordination run the rest of the time - the maneuvers without deleting the strengths.

The considered-and-rejected alternative to the top-level planner block is rx-style injection inside the combat sub-planner (`cast_planner` on `action_combat_planner`, then `add_evaluator`/`add_action` there, per Rulix's `rx_combat.script:327-353`). It preserves what the top-level block loses - `CStalkerCombatPlanner::update`'s side effects, `react_on_grenades` / `react_on_member_death` at `stalker_combat_planner.cpp:102-105`, and the initialize/finalize mask and danger inertion. But it arbitrates against whatever a modpack grafts inside that same planner, so it surrenders exactly the robustness the takeover was chosen for. The top-level block wins for the GAMMA audience, at the cost of suppressing those `update` side effects for the seconds it holds - which is why a transaction stays narrow and brief.

---

## Accuracy

Rank-aware NPC dispersion in script. `at_accuracy.script` subscribes to the vanilla `npc_shot_dispersion` callback (declared in `axr_main.script:126`, dispatched from `_g.CAI_Stalker__GetWeaponAccuracy` at `_g.script:1213-1217`).

Why script and not cvars: the engine rank curve degenerates on Anomaly gamedata. `Rank()` clamps to `[0, 100]` at `ai_stalker.cpp:764`, but vanilla `<rank>` intervals run to 26999 (game_relations.ltx:8). All Anomaly NPCs end up at `rank_k = 1.0`, so `m_fRankDisperison` collapses to the constant `dispersion_experienced_k = 0.8`. Cvar tuning is a dead knob.

Math: `out = base * disp * move`. The engine already multiplied by `m_fRankDisperison` (= 0.8 for every Anomaly NPC after the rank clamp) before the callback fires, plus the per-state factor. We stack two rank curves on top:

- `disp` - the flat rank curve, applied to every shot. `disp = 1.00` preserves the engine's vanilla cone, lower values tighten it, above 1.00 loosens. Defaults spread wide so rank reads over the ~10-point positional noise (proven 2026-07-14 by the 0.2-vs-1.3 controlled test): novice 1.20 (sprays) -> legend 0.20 range, shipped 1.20 / 1.05 / 0.90 / 0.75 / 0.60 / 0.47 / 0.34 / 0.22 across the eight tiers.
- `move` - the moving-fire curve, applied only while the shooter's `move_type` is walk/run (`ai_stalker_fire.cpp:81-104`). Cancels part of the movement spread penalty per rank: novice 1.00 keeps the full vanilla ~2x penalty, legend 0.70 keeps ~1.8x of it - every rank still pays visibly for firing on the move, so the planted-fire systems (Pickoff, Commitment, stance) keep their edge. Shipped 1.00 / 0.96 / 0.91 / 0.87 / 0.83 / 0.79 / 0.74 / 0.70. Error budget: `doc/library/modding/npc-combat-effectiveness.md` "The error budget per shot".

Per-rank tiers (novice through legend): higher rank, tighter dispersion. Each curve has its own MCM toggle (`disp_enabled`, `move_enabled`) with no page master; `_on_npc_shot_dispersion` applies each factor only when its toggle is on, a disabled factor staying 1.0.

Per-shot hot path: a rank-name lookup, a move_type compare, then pure-Lua scaling of the dispersion the callback hands us.

---

## Reaction

MCM page: Effectiveness > Reaction (`at_reaction.script`, built 2026-07-11). Per-NPC, rank-tiered aim, target lead, vision, and fire discipline through merged engine binds (#594, #603). Aim, vision, and fire discipline are set once per `npc_on_net_spawn` (the fields are not serialized; the game object is reconstructed on re-online, so every spawn re-sets them); the target lead is recomputed live on the fire seam (t156, below). The spawn body is public as `at_reaction.apply(npc)` because the fields are never re-read: a runtime rank change (`npc:set_character_rank`, the test bench) leaves the old tier applied until a caller re-keys it. Five values per tier:

- **Tracking speed** (`xcombat.set_aim_params` min_speed, rad/s): how fast the sight moves inside the `min_angle` lock band in `select_speed` (`sight_manager.cpp:80-101`). 0.24 is the vanilla default; the curve runs novice 0.24 to legend 1.50, a decent step short of the 2.5 hardcore-aim option. `max_angle` passes -1 (follow the live `ai_aim_max_angle`); `min_angle` now carries the Tracking Lock curve (below) rather than -1. Alone at the vanilla lock band, min_speed is near-cosmetic - tested 2026-07-12 (forced A/B, 10x spread): no change to first-shot latency (the band sits inside the trigger cone `fire_angle` 0.3927, so the first shot never waits for it) and no strafing hit% change. It becomes real once Tracking Lock widens the band, which is why the two ship together and share `aim_enabled`.
- **Tracking lock** (`xcombat.set_aim_params` min_angle, rad, 2026-08-05): the gap size below which the barrel tracks at min_speed with no deceleration (`ai_aim_min_angle`, `sight_manager.cpp:76`). Vanilla 0.196 = half the fire cone, so a novice's barrel eases off at the cone edge and a strafer can juke it; the curve widens the band toward the fire cone (novice 0.196 to legend 0.40, past `fire_angle` 0.3927 so a legend holds a strafing target across the firing window). Every value stays under `max_angle` 0.785 so the natural fast swing-in survives - `xcombat.set_aim_params` asserts on any angle past PI (above PI the band collapses to always-on, silently - the failure the game's hardcore option abuses with 17). `_resolve_min_angle` returns -1 (follow the global) when the hardcore option is already stickier, so a player's choice is never downgraded and no above-PI value ever reaches the assert.
- **Target lead** (predict_time, seconds, t156): the aim point is the target's visible position plus its horizontal velocity x predict_time (`predict_object_position`, `sight_action.cpp:383-384`). Recomputed per firing NPC on `npc_shot_dispersion` (throttled 1s, `_on_shot_lead`) as `clamp(range / bullet_speed, 0, 0.5) * lead_<tier>`: the physically-correct lead is the bullet's flight time (the weapon section's `bullet_speed` times the loaded round's `k_bullet_speed`, `ShootingObject.cpp:168` and `Level_Bullet_Manager.cpp:66`, so subsonic/AP ammo scales the lead; per-kind fallback when the section lacks the key), so the lead adapts to range and round speed on its own and the per-rank `lead_*` factor (2.00 novice -> 1.00 legend) is the only skill lever - legend leads true and hits movers, low ranks over-lead so their rounds overshoot a crossing target, and the error compounds with the wide low-rank dispersion cone. Bounded and range-scaled, unlike the retired fixed 0.40->0.28 table that over-led every rank at close range. The recompute re-passes the rank tracking speed so `min_speed` does not revert to the global; nothing is set at spawn (no enemy yet, predict follows the global until the first shot).
- **Vision speed** (`xcombat.set_vision_speed`): a factor on `get_visible_value` accumulation (`visual_memory_manager.cpp:365-379`), applied after whichever detection stack the install runs (engine formula or Lua functor) - a relative rank spread over the installed baseline, the only semantics the seam supports. It is REACTION (how fast a stalker turns a glimpse into a confirmed threat), not eyesight: detection timing only, never aim or accuracy - `m_vision_speed` is read solely in `get_visible_value`, and the sight, fire, and dispersion paths never touch it. Retuned 2026-07-29 from the old 1.00-2.00 (every tier at-or-above vanilla, so a legend at 2.00 fast-confirmed targets through weakly-occluding foliage - Feel_Vision gates line of sight on material transparency, and a bush authored weak is defeated fast by a high multiplier) to a band re-centered 2026-08-05 on novice = vanilla: novice 1.00 to legend 1.21, uniform 0.03 step, every tier at-or-above vanilla so no rank detects slower than the base game. Inert inside `always_visible_distance`.
- **Fire discipline** (`_qsize` / `_qinterval`, t154): two per-rank queue scales set at spawn via `xcombat.set_fire_queue_scale` (PR #603), applied in `select_queue_params` (`stalker_combat_action_base.cpp:246-254`) after the weapon-type/distance band pick - `_qsize` multiplies burst size, `_qinterval` the inter-burst pause. High ranks fire shorter bursts at a tighter cadence. Scope: vanilla-planner fire only; a `state_mgr` fire state (maneuver override) reaches the object handler with explicit params and bypasses `select_queue_params`. Non-degradation invariant: defaults keep `_qsize >= _qinterval` per tier, so rounds/min (proportional to size/(interval + size*dt)) stays >= vanilla while a shorter burst sheds only the dispersed tail rounds (past `base_dispersioned_bullets_count`, `WeaponMagazined.cpp:797-800`), raising per-shot% -> hits/min >= vanilla. `_qinterval` floored at 0.60 so bursts never merge into continuous fire.

The rejected delivery, for the record: driving the global `ai_aim_*` cvars from per-NPC update callbacks (one NPC at a time owns 4 globals, actor-only, reset every actor update - a writer war), with values degenerate in the radians domain (min_angle above PI kills the blend band; min_speed above every animation speed disables damping identically for every rank), and perception through a per-frame `get_visible_value` functor - the pattern code-standards bans. Reaction takes the per-rank CONCEPT, per-NPC: aim and vision set-once at spawn, the target lead recomputed on the fire seam.

Each curve has an MCM toggle - `aim_enabled` (tracking speed and lock together), `vision_enabled`, `lead_enabled`, `discipline_enabled` - and its own per-rank slider set (`rct_aim_*`, `rct_track_*`, `rct_vision_*`, `rct_lead_*`, `rct_qsize_*`/`rct_qinterval_*`); there is no page master. The controls span three pages under the one `at_reaction.script`: aim and lead on Effectiveness > Reaction, the vision speed/range curves on Perception > Vision, and fire discipline (burst size + cadence) on its own Effectiveness > Discipline page. `apply()` sets tracking speed and lock behind `aim_enabled` and vision behind `vision_enabled`, `_on_shot_lead` runs behind `lead_enabled`. Each subsystem's explanation is the hover hint on its toggle, not a separate row.

On an exe without the binds both wrappers return false and Reaction logs INACTIVE once; vanilla behavior applies untouched.

---

## Disclosure

MCM page: Effectiveness > Disclosure (`at_disclosure.script`). The hit-victim turn plus a bounded squad investigate on suppressed attacks, all through engine perception and selection - no relation writes, no memory injection, no squad-wide combat-mask forcing. The earlier force-disclosure model (squad-wide `disclose_enemy` on hit #1, retention map, spawn inherit, shooter re-disclose) is retired: it force-ENGAGED distant patrol members with no perceptual basis, and its victim-turn leg (`make_enemy_visible`) was disproven at source - `make_object_visible_somewhen` saves and RESTORES the prior visible bit (`memory_manager.cpp:355,361`), so for an unseen shooter the "seen" promotion was a no-op and selection still ranked him ~1000 behind any seen enemy. The module also hosts the target-priority fairness dial (the player-magnet bias).

### The flow

```
npc_on_net_spawn (stalker, non-zombied)
  -> xcombat.set_hit_redirect(npc, 900, 60)      when Disclosure + Turn are on, else (-1) = vanilla
  -> xcombat.set_visible_enemy_bias(npc, dial, -1)   the player-pull dial, npc side vanilla

npc_on_hit_callback (any hit on a stalker)
  -> gate: enabled, not from_death_callback, amount > 0, not self, victim not zombied,
           npc:relation(who) >= 2 (per-NPC hostility, not community)
  -> loud shot (unsuppressed stalker/actor weapon): return - engine gunfire hearing owns it
  -> defer one frame: victim dead -> nothing (native death sound / corpse discovery own it)
     victim alive:
       floor exe + Turn on -> scripted danger at the KNOWN shooter position + register_in_combat
                              (victim only - the S1 fallback turn)
       squadmates within earshot of the victim -> xr_danger.set_script_danger(member, ..., "solid")
                              (walk-investigate the shooter position; engage only on real perception)
```

### The victim turn: a standing selection lever, not a memory write

`npc:set_hit_redirect(max, falloff)` (PR #636, merged; `enemy_manager.cpp:149-167`) scales the engine's own "this object hit me" term in `CEnemyManager::evaluate`: the last attacker (`memory().hit().last_hit_object_id()`) within `falloff` metres gets up to `max` subtracted from its cost, decaying to zero at `falloff`. At 900/60 (probe-proven) a close attacker outranks a fully-visible distant enemy, so the victim flips SELECTION on the real hit signal - even a victim already committed to another enemy, which no script-side seed can reach. Nothing stamps a sighting: `fire_make_sense` still requires real line of sight or a genuine last-seen, so there is no through-cover fire and no wallhack. The lever is standing per-NPC engine state, written once per online at `npc_on_net_spawn` (not serialized, the enemy manager is rebuilt on re-online); MCM off writes the -1 sentinel = the vanilla -5/-100 hit step. Zombied keep vanilla selection.

**Floor exes** (no bind): the S1 fallback runs per admitted hit - `xr_danger.set_script_danger` at the KNOWN shooter position (the danger action's look order wins because the danger action is the selected action - an ordered `set_sight` would lose arbitration to the committed enemy's sight) plus `xcombat.register_in_combat`. Ceiling, documented: a victim already fighting another enemy stays on his fight (the danger scheme yields to combat); only the selection lever reaches him.

### The squad half: investigate, not engage

A suppressed hit on a surviving victim stamps his squadmates within earshot of the VICTIM (~15m, tunable) with the graded scripted danger at the SHOOTER's position - the same `set_script_danger` idiom the noise system uses, grade "solid", so the reaction walks the position weapon-up (t163 "raid") and never runs. The stamp expires on its own inertion; a member who actually perceives the shooter escalates to combat natively. A member already fighting is untouched by construction - the danger scheme does not run for an NPC with a combat enemy. Per-member re-stamp throttle; zombied skipped.

**The silent/loud gate.** An unsuppressed shot from a human (stalker or actor) seeds nothing - the engine's own gunfire perception covers it twice: per-listener `attack_sound` danger entries, and the ally-relay (`CStalkerSoundDataVisitor` - a listener adopts the enemy a fighting ally has selected, `stalker_sound_data_visitor.cpp:30-60`). Suppressed-now is the `utils_item.has_attached_silencer` shape (`utils_item.script:414-420`): an integral silencer (`weapon_silencer_status() == 1`) or an attachable one currently mounted (`== 2` + `weapon_is_silencer()`). A shooter without a ranged weapon in hand - a mutant, a knife - is silent by definition.

**Survivor semantics.** A hit that killed the victim seeds nothing (checked one frame deferred - `alive()` is still true inside the killing hit's callback). A clean suppressed kill tells no one; the squad can still find the body through the engine's native death sound and corpse discovery.

### The target-priority dial

`npc:set_visible_enemy_bias(actor_bias, npc_bias)` (PR #637, merged; `enemy_manager.cpp:175-184`) replaces the hardcoded "prefers whoever sees me" terms: vanilla subtracts 900 when the ACTOR sees the NPC but only 300 for another NPC - a ~3x baked player magnet. The MCM dial (0-900, default 900 = vanilla) writes the actor side per-NPC at the same net_spawn seam; the npc side stays vanilla. Lower values treat the player like any other combatant.

### Net behavior

- The victim turns on a close attacker through the engine's own selection; fire needs real line of sight.
- Only squadmates who could plausibly have noticed (earshot of the victim, suppressed case) investigate the shooter's position; engagement requires real perception. Distant patrols are never told.
- Loud shots are the engine's business end to end - no script double-fire.
- No relation writes (the original goodwill-write era corrupted saved relations and is long gone), no memory injection, no forced combat-mask, no retention state: the module keeps only a per-member stamp throttle and counters (`get_stats` for `at_test.at_dump`).

---

## Danger

`at_danger.script` installs AlifeTactics's danger scheme as a function-level patch, at `on_game_start`, onto whichever `xr_danger.script` won the MO2 virtual filesystem (vanilla, GAMMA AI Rework, or REDONE Combat AI). AT no longer ships `xr_danger.script`, so it does not compete for that slot. Vanilla bug fixes run always-on; three improvements sit behind MCM toggles. The paired DLTX overlay (`configs/ai_tweaks/mod_xr_danger_at.ltx`) is delete-lines only: AT ships no danger values, so detection distances and inertion stay owned by the setup's own `xr_danger.ltx` (GAMMA plays AI Rework's tuning unchanged, vanilla plays vanilla's rows).

### The flow

```
on_game_start: at_danger points the winning xr_danger's entry points at itself
    setup_generic_scheme / add_to_binder / configure_actions / reset_generic_scheme /
    get_danger_time / set_script_danger / has_danger
bind time (modules.script): _G["xr_danger"].add_to_binder -> AT's evaluators + action
    installed into every stalker's motivation manager under the danger scheme ids
runtime, per plan solve:
    eval_danger -> npc_on_eval_danger (third-party veto seam)
        -> live script_danger stamp = TRUE on its own (the stamp is an ACTIVATOR;
           vanilla's set_script_danger only redirected an already-active danger action,
           so stamp reactions silently died on modpacks that mute engine sound
           perception - 2026-07-25 reporter fix, same reorder in the selector and execute)
        -> else best_danger type
        -> inertion + ignore tables (the winning ai_tweaks\xr_danger.ltx rows)
        -> danger_flag
    at_action_danger:execute -> script_danger stamp first, else per-type response
        (grenade / corpse / attacked / attack_sound alert)
    combat-safe by GOAP construction: the action requires property_enemy false
feeders: the winner's own hear + death callbacks
    -> patched set_script_danger -> script_danger table
```

### How the patch installs

The danger implementation lives privately in `at_danger.script` (file-locals plus `at_`-prefixed evaluator and action classes). At `on_game_start` the module points the winning `xr_danger`'s generic-scheme entry points at its own versions: `setup_generic_scheme`, `add_to_binder`, `configure_actions`, `reset_generic_scheme`, `get_danger_time`, `set_script_danger`, `has_danger`. Each is dispatched by a live `_G["xr_danger"].fn` lookup at bind and reset time (`modules.script:110,152,167,211`; `xr_logic.script:294`), and every script runs `setfenv`'d to its own module table (`script_storage.cpp:44-63`), so the winner's own internal bare calls resolve to the patched members too. The winner's `add_to_binder` is never called; AT's binds AT's evaluators and action into every stalker's motivation manager under the danger scheme's fixed ids.

AT does not register the hear or death callbacks. It relies on the winning file's `npc_on_hear_callback` and `npc_on_death_callback`, vanilla-derived in every known `xr_danger`, which feed AT's `script_danger` table through the patched `set_script_danger` and write `killer_last_known_position` into `db.storage`. The winning file's other danger callbacks (GAMMA AI Rework's torch and weight perception, REDONE's per-frame perception) keep running untouched.

### Vanilla bugs fixed (always-on)

1. `bd_types` name collision: the perceive-type names `visual`/`sound`/`hit` share enum values 0/1/2 with danger types in the single `danger_object` enum (`danger_object.h:18-35`, `memory_space_script.cpp:130-147`), so three danger categories read the wrong config section. AT's table drops the three perceive entries.
2. `get_danger_time` crashes on mutant corpse: vanilla calls `corpse_object:death_time()` without `IsStalker` guard; trader interface absent on mutants.
3. `eval_danger` nil-NPC guard missing: vanilla crashes when called on a torn-down NPC reference.
4. `eval_danger` non-numeric `danger_time` check missing: vanilla type-asserts on bad return.
5. The vanilla hit callback passed an undefined `who_id` and wrote a nil shooter id into `script_danger` on every hit. AT does not register it, and the patched `set_script_danger` rejects a nil `who_id` at its top, so the winning file's own copy of that callback cannot corrupt the entry either.
6. Animstate reset missing on danger-state transitions: vanilla `state_mgr.set_state` calls did not invoke `sm.animstate:set_state(nil, true) + set_control()`, leaving stale lower-body animation visible across the transition. AT calls the reset at every state-change site (every `sm.animstate:set_state(nil, true)` + `set_control()` pair in the danger action handlers).
7. `at_action_danger:finalize` wiped the whole shared `db.used_level_vertex_ids` reservation map in vanilla, clobbering every other system's cover claims (the Combat takeover's, vanilla's). AT releases only the vertices this NPC owns (the owner-filtered loop in `at_action_danger:finalize`).
8. `script_action_danger_corpse` compared `st.stage >= 4` bare (vanilla `xr_danger.script:539`) and crashed the game when the corpse reaction fired before the scheme's initialize set the stage; since stage only ever holds 1 or 2, the bare compare could only crash, never pass. AT reads `(st.stage or 0)`.
8. The corpse action crashed on the teardown race: a corpse despawning between the evaluator pass and the action execute left a nil danger object (`bdo:id()`) or a nil storage entry in the stage-6 look_position. Both paths are guarded.
9. The corpse force-hostile loop reused the acting NPC variable for squad members, so later corpse stages drove the last member (or nil) instead of the acting NPC. The loop uses its own local.
10. Corpse stage 6 sent the NPC to the just-cleared `st.lvid` instead of the cover vertex `try_go_cover` found, so the found cover was never used.
11. Performance: vanilla re-parsed the danger inertion and ignore-distance condlists on every evaluation, per NPC per plan solve. The strings are fixed after the DLTX merge, so they parse once into a memo (`_parse_cached`) and every later evaluation is a table lookup; only the condition evaluation (weather, actor state) still runs per call.
12. `script_danger` entries are dropped on entity unregister. Vanilla kept a dead perceiver's entry until expiry, so an id recycled inside the inertion window inherited a scripted danger the new NPC never perceived.

### Extension callback

`eval_danger` fires `npc_on_eval_danger` with `flags.ret_value = true` before it evaluates; a subscriber that sets `flags.ret_value = false` suppresses danger for that NPC on that tick. AT preserves this vanilla seam - `axr_main.script:125` declares it and vanilla `xr_danger.script:268` fires it from the same evaluator - so third-party subscribers keep working when AT's patch replaces `eval_danger`.

### Improvements (MCM Danger category, default on)

- `danger_hit_bypass` (MCM Danger > Hit, "Distant hits"): a direct hit is danger at any distance - the branch returns true past the relation, combat-ignore and ignore-distance gates (being hit is proof of range); only the scripted combat-ignore override still suppresses it. The division of labor around a hit: at_disclosure owns "the squad learns the shooter" (faction enemies, combat memory, rangeless by construction); hit_bypass owns "the victim ducks even when he cannot fight back" (a neutral or combat-ignored shooter, where combat cannot engage); the future Range page owns answering fire at weapon reach.
- `danger_attack_sound` (MCM Danger > Sound, "Enemy gunfire"): reacts to enemy gunfire the NPC heard but cannot see. Engine side: a heard `SOUND_TYPE_WEAPON_SHOOTING` becomes an `attack_sound` danger at the shot's position (`danger_manager.cpp:301-304`) - the perception, the hearing range, and the danger object are the engine's, always were. AT side: the scripted danger scheme never reacted to that type - the evaluator admitted only attacked/corpse/grenade, so the danger was produced and then ignored (2026-07-10 fix). AT admits `attack_sound` (`at_evaluator_check_danger`, toggle-gated) and routes it to `script_action_danger_alert`, and drops the inherited non-enemy aim gate so enemy fire triggers without aim (the gate stays dormant behind the upstream relation gate; admitting non-enemy sources is t150). Vanilla shipped no handler for this danger type. The reaction: a hostile stalker who cannot see the shooter turns to face the sound and holds a threat stance (the no-sight branches of `script_action_danger_alert`); the move-to-cover and strafe follow once he can see the enemy (the `npc:see(be)` stage-1 path). Reaction distance is the winning config's `attack_sound` row. Movement noise is the Noise hearing section's job (`at_noise.script`, below).
- `danger_actor_tables` (MCM Danger > Fixes, "Player ranges"): read separate inertion and ignore tables from `[danger_inertion_actor]` and `[danger_object_actor]` when the danger source is the actor - meaningful where the installed config differentiates them (GAMMA AI Rework does; vanilla ships identical copies, so the toggle is a no-op there). Gated by a liveness probe at `_install_patches()`: the actor ignore table is honored only when the winning `xr_danger` module carries its own `DangerIgnoreActor` field, proof it consumed the section itself. A config that ships `[danger_object_actor]` without reading it (a stale copy next to a retuned base table) keeps its base tuning instead of having the unused section resurrected.
- `danger_neutral_gunfire` (MCM Danger > Sound, "Neutral gunfire alert", t150): a NEUTRAL or FRIENDLY stalker goes on alert to the PLAYER's shots fired close by. The one admission through `is_danger`'s relation gate: an `attack_sound` danger whose source is the actor at relation < enemy passes within `NEUTRAL_GUNFIRE_RADIUS_SQR` (30m - deliberately its own small constant, never the config's combat-range attack_sound row, so camps do not alarm at 300m). The reaction is alert and NOTHING more: the alert handler's non-enemy branch sets a weapon-ready stance facing the sound (`threat_na`) and returns - no repositioning, no relation or goodwill write, decaying with the inertion. Scoped to the actor: NPC-vs-NPC neutral fire stays ignored. Each gunfire population obeys its own toggle - the `attack_sound` admission and dispatch run when EITHER toggle is on, and the handler head routes enemy sources through `danger_attack_sound` and non-enemy actor sources through `danger_neutral_gunfire`, so neither leaks through the other's switch.

### Noise hearing (at_noise, MCM Danger > Sound)

The t160 build (2026-07-19): hostile stalkers hear the player's movement and handling noise. `at_noise.script` is feeder-only - it adds NO reaction machinery; both signals stamp `xr_danger.set_script_danger` (the patched entry at_danger owns), so the reaction (orient to the heard position, alert stance, reposition - `script_action_danger_scripted`), its decay (`danger_inertion`), the combat gate (the scheme runs only with no `best_enemy` - hearing is an out-of-combat sense), the zombied and wounded exclusions, and the unregister cleanup (fix 12) are all the existing scheme's.

Two signals:

- **Movement** (`danger_move_noise`): landings included - `actor_on_land` (`Actor_Movement.cpp:90` via `_g.script`) accumulates a flat `LAND_RADIUS_M` (10m) thud on the same path, the loudest movement sound the actor makes. Steps: the VANILLA per-step event `actor_on_footstep` (`step_manager.cpp:209` fires `_G.CActor__FootstepCallback` for the actor per step of the legs animation; `_g.script:1260` forwards it) accumulates a noise radius - `BASE_RADIUS_M` (5m walking) x stance (crouch 0 - crouched movement is SILENT, so the crouched stealth-kill approach that vision-based stealth is built around is never defeated by hearing; sprint 1.6 - stance scaling per the engine's own abandoned footstep-hearing design, `CROUCH_SOUND_FACTOR`/`ACCELERATED_SOUND_FACTOR` at `ai_sounds.h:81-82`, defined and never consumed) x surface material (the step's material name arrives with the event: metal/wood/water louder, grass/dirt/sand quieter) x the MCM multiplier x the install-hearing scale (t170: the winning config's own stalker hearing sensitivity over vanilla's - `[stalker] sound_threshold` and the `[stalker_sound_perceive]` npc factor, the two keys the engine admission consumes at `sound_memory_manager.cpp:168-180`; derived once at first update, clamped 0.25-2.0, factor 1.0 on vanilla/GAMMA/Stealth Overhaul tuning - so a pack that deafens NPC hearing deafens the synthesized radii identically, where the engine-heard handling rows already inherit it through the ear). A scheduled pass (500ms time event, the at_combat monitor shape; never per-frame) masks the accumulated radius by rain (`level.rain_factor`), then walks `db.OnlineStalkers` and stamps every alive, actor-hostile stalker inside it - per-NPC re-alert throttled (4s), luabind reads only inside the radius. Standing still produces no step events, so the steady-state cost is zero.
- **Handling** (`danger_heard_actions`): `npc_on_hear_callback` (the vanilla ear: `callback.sound` -> `motivator_binder:hear_callback` -> `xr_hear.hear_callback`, which fires the callback for every TYPED sound the NPC's sound perception accepted - attenuation and the `[stalker_sound_perceive]` thresholds already applied). The actor's `WPN_reload` (8m), `WPN_empty` (6m), and `ITM_use` (5m) types stamp the same scripted danger at the sound's position; every other type costs one table read.

Every noise stamp carries an evidence grade, and the scripted reaction runs at that grade (t163). The design law: a sound is a position ESTIMATE, never a confirmed enemy, so no sound-only stamp may produce a run state. FAINT (walking steps, `ITM_use`) turns the NPC weapon-ready toward the heard position (`threat_na` facing the STAMPED position, not the source's live one) and nothing more. SOLID (sprint steps, landings, `WPN_reload`, `WPN_empty`) runs the existing reposition in `script_action_danger_scripted` at `raid` (walk, weapon up) in place of `assault`, arrival scan and decay unchanged. The grade rides an optional strength argument on `set_script_danger`, stored per stamp; a caller that passes none (the winning `xr_danger`'s own feeds, impact sounds) gets the original assault path unchanged. Strong evidence (gunfire, impacts, hits) never enters this path at all - it arrives as engine danger types with their own handlers. A solid event bypasses the per-NPC re-alert throttle over a standing faint alert, so a sprint past an already-listening NPC escalates now instead of after the throttle window.

The boundary that keeps stealth playable, stated as what we never do: footsteps are NEVER typed into the engine sound space. A typed hostile footstep would land an `EnemySound` danger per step and thrash the 32-slot sound memory (`danger_manager.cpp:322-327`) - the reason GSC shipped footsteps with `sg_SourceType = -1` and the reason the reaction is script-scoped: relation-enemy only, out of combat only, alert-not-omniscient (the NPC turns toward a position, he does not acquire the player), throttled, decaying. Neutral NPCs never react to the player's noise, and crouched movement is silent outright. The radii ship provisional; the playtest owns the balance numbers.

Stealth compatibility: stealth in Anomaly is a VISION system - the engine's `CVisualMemoryManager` accumulates visibility from light, cover, stance, and speed, with the math delegated to the `visual_memory_manager.get_visible_value` script hook that stealth overhauls replace. The sound system never touches that hook, vision configs, or seen-memory. The mod's systems that DO touch detection are orthogonal to the light model: Reaction's Vision Speed scales each rank's detection SPEED (the same formula, faster or slower), and disclosure writes seen-memory only after the player HIT someone - neither changes what light and cover let an NPC see. Hearing adds the one sense vanilla lacks at radii (5-10m) far below any vision range, crouch is silent, and the scan has no occlusion test so the radii double as the wall policy. A crouched approach a stealth setup permits by light and cover is therefore never revealed by sound.

### Paired LTX

`configs/ai_tweaks/mod_xr_danger_at.ltx` is delete-lines only (2026-07-10): `![section]` override-merge on all four danger sections, each dropping only the dead `hit`/`sound`/`visual` keys (PerceiveType names colliding with EDangerType values 0/1/2; nothing reads them after the bd_types fix). AT ships NO danger values - every detection distance and inertion comes from whichever `xr_danger.ltx` won the MO2 slot plus later DLTX overlays: GAMMA plays AI Rework's tuning unchanged, vanilla plays vanilla's true-name rows (the rows the collision always hid), Stealth Overhaul plays xcvb's. The 1.1.0 value rows were an exact GAMMA AI Rework copy; on non-GAMMA setups they cut effective reaction ranges 3-50x (the 2026-07-10 user report), so they were removed and the setup owns the tuning. `!![section]` is NOT full-section replacement in this engine's DLTX - it deletes the section outright and discards the keys under it (`Xr_ini.cpp:721-739`, the 2026-07-09 nil-src condlist flood).

Later-alphabet DLTX overlays on the same sections still take precedence, load-order-wise, as with any DLTX stack (in GAMMA, Useful Idiots' `mod_xr_danger_z_idiots.ltx` owns `[danger_inertion]` this way).

### Composition

`at_danger.script` carries the `-- @override` marker only to exempt its vanilla-derived code from AT-native style rules and the stub load test; it is not a VFS whole-file override. Because AT patches the winning file at runtime instead of replacing it, the Danger system layers onto GAMMA AI Rework or REDONE rather than excluding them: AT's evaluator and action logic wins while the rival's file stays loaded and its own perception callbacks keep running. The one thing the patch cannot do that a file override could is suppress the winner's danger callbacks, which surfaces only as REDONE's fixed hit callback adding a second, harmless trigger of the same react-to-a-hit intent. The MCM Danger category (Sound, Hit, Fixes leaves) describes the always-on fixes and the three improvement toggles.

---

## Crossfire

Friendly-fire damage gate in `at_crossfire.script`. `npc_on_before_hit` scales `shit.power` by the MCM factor unless the shooter and victim are actually enemies (`attacker:relation(npc) == game_object.enemy` -> full damage). Keyed on per-NPC relation, not community: same-faction NPCs are neutral at worst and never enemy (a loner never enemy to a loner), so they stay protected, while a soured cross-faction pair (a loner vs a hostile Clear Sky) still damages each other. `relation()` is faction-paramount (the community-to-community base dominates personal goodwill). Stalker-vs-stalker only (both `IsStalker`), the actor as shooter is excluded, O(1) with no throttle (a damage block must catch every hit). MCM page: Effectiveness > Crossfire (`crossfire_enabled` + `crossfire_factor`).

---

## Healing

Per-NPC self-healing. Vanilla `xr_eat_medkit.script` has a working stage machine, but vanilla `ai_tweaks/xr_eat_medkit.ltx [plugin]` lacks the `medkits=` / `bandages=` keys so `parse_list` returns `{}` and the consumption loop iterates zero times.

### Data layer fix

`mod_xr_eat_medkit_at.ltx` is a DLTX overlay on `![plugin]` adding `medkits = medkit, medkit_army, medkit_scientic, medkit_ai1, medkit_ai2, medkit_ai3` and `bandages = bandage`. Boot-time, no runtime toggle.

### Runtime tuning

`at_firstaid.script` installs two hooks on `on_game_start`:

| Hook | Mechanism | What it changes |
|---|---|---|
| Heal rate multiplier | `xr_eat_medkit.heal_hp = _patched_heal_hp` | Per-tick `change_health` scaled by the MCM multiplier, read each tick; rescheduling via the `xr_eat_medkit.heal_hp` lookup keeps every tick on the patched function |
| Engaged-pause | inside `_patched_heal_hp` | A fight must be able to end: the hp tick PAUSES while the NPC is actively engaged - a live `best_enemy` with the weapon in his hands (`weapon_unstrapped`, the state_mgr.script:396 read) or an enemy inside 5m regardless of weapon (a mutant mauling him). Pause = `ResetTimeEvent` on the firing event + return false, so the same event stays queued with a pushed timer - no health applied, no tick consumed, the charge waits; a same-key re-create from inside the callback would be silently skipped (`_g.script:345`) and kill the chain. Bleed staunching deliberately stays vanilla: pausing it would turn every pressed fight into a bleed-out lottery. Field case: the melee-locked NPC pair that healed through punch damage at 3x and could not die (reporter, 2026-07-21) |
| Bandage tick logging | `xr_eat_medkit.heal_bleed = _patched_heal_bleed` | Logging-only wrapper around vanilla bleed loop |
| Per-rank healing-charge | `RegisterScriptCallback("npc_on_net_spawn", _on_net_spawn)` | Reads `ranks.get_obj_rank_name(npc)`, folds the rank names into MCM tiers, rolls the per-tier chance, replacing vanilla's flat roll. Per-NPC `at_charge_processed` se_var prevents re-roll. |

### The flow

```
heal:  vanilla xr_eat_medkit stage machine (untouched)
         -> patched heal_hp / heal_bleed (rate multiplier, duration trace)
         -> consume_medkit save-wrap logs item=<section> | CHARGE
            (a real inventory item released vs the per-rank fallback)
limp:  monitor pass (a vanilla time event, 200ms, over the spawn-filled roster)
         eligibility check (1s, per-NPC stamp): hurt + no enemy + calm + standing
             -> queue the hurt pose for the current gait
         drop detectors (every pass): wounded/dead, gait changed,
             drifted off the stand anchor, stopped displacing
             -> clear_animations
            (a queued script animation suspends the engine's whole animation
             selection, so the pose must die the moment its gait stops matching)
```

### Visual layer (Path 1 script-queue overlay)

Two cosmetic cues using `npc:add_animation` directly. No state_mgr, no GOAP, no `state_lib` changes. See `doc/library/modding/state-lib-animations.md` for the Path 1 script-queue overlay mechanism.

| Cue | Trigger | Animation(s) |
|---|---|---|
| Limping | the limp monitor pass (`_run_monitor`, a vanilla time event every 200ms over the spawn-filled roster): the drop detectors run on the pass itself while the pose is worn (wounded/dead; an enemy is selected or the mental state leaves free, i.e. a fight began - moved onto this fast pass so a limp does not linger up to a second into combat and slide when the NPC is shot; commanded gait changed since add; stand-variant drifted off its add anchor; walk/run-variant stopped displacing over a 1s sample) - every drop calls `clear_animations()`; a 1s eligibility check on a per-NPC stamp (`health < threshold`, no `best_enemy`, `mental_state() == anim.free`, `body_state() == move.standing`, not zombied, not in smart_cover), re-armed every 5s | a per-slot `dmg_norm` hurt pose (clutch-the-torso) chosen from `active_slot()` + `movement_type()`. A queued script animation suspends the engine's whole animation selection (legs included, stalker_animation_manager_update.cpp:232), so the overlay must die the moment its gait stops matching - that is what the drop detectors do |
| Heal anim | One-shot via `_try_play_heal_anim` on the first heal tick. Gated on `not npc:best_enemy()`, `not IsWounded(npc)`, `not npc:critically_wounded()`. No movement freeze, no stage machine, no mid-flight aborts. Engine drains the queue when the gesture ends; action transitions clear it on the way to action_wounded / action_critically_wounded (`stalker_base_action.cpp:24-29`) | a torso medkit / bandage gesture |

Limping is independent of the healing master toggle (its monitor arms unconditionally; gated at runtime by `limping_anim_enabled` - one boolean per pass when off). Heal cue is gated by `healing_anim_enabled` and the master toggle (it lives inside the heal_hp/heal_bleed patches that only install when healing is enabled).

Combat NPCs are excluded by the `mental_state == anim.free` gate. state_mgr drives mental to `anim.danger` in combat states (`state_lib.script:326-340` hide_fire / threat).

---

## Jamming

Suppresses the modded-exes script-injected NPC misfire path. `at_jam.script` `on_game_start` replaces `xr_weapon_jam.GetConditionMisfireProbability` with a function that returns 0. The engine's per-shot misfire roll for non-actor weapons becomes 0 unconditionally.

### What the modded-exes overlay does natively

Themrdemonized's `gamedata/scripts/xr_weapon_jam.script` (packed in `00_modded_exes_gamedata.db0`) defines `GetConditionMisfireProbability(weapon, npc, base_value)`, which the engine looks up by name at `Weapon.cpp:1781` and calls per-shot. The function fires the `npc_get_misfire_probability` callback; the default subscriber `npc_misfire` computes `chance = clamp(base_ch_<rank> * ammo_spent, 0, max_ch_<rank>)` from per-rank settings in `ai_tweaks/xr_weapon_jam.ltx`. On a roll hit, `t.ret_value = 1` is written, the engine treats the next shot as a definite misfire (`bMisfire = true` in `CheckForMisfire` at `Weapon.cpp:1800`), `StopShooting()` fires, the NPC's `Ready1` evaluator (`object_property_evaluators.cpp:117`) returns false, and the planner schedules a reload to clear the misfire flag. NPCs at full weapon condition still misfire every 2-3 rounds based purely on ammo spent.

### What we override

`at_jam.script` saves the original function reference at install and replaces it with one that reads `at_mcm.get_config("jam_enabled")`:

- `jam_enabled == true` (default): return 0 regardless of weapon, npc, or base_value.
- `jam_enabled == false`: forward `(weapon, npc, base_value)` to the saved original. Modded-exes behavior restored.

The engine gates the functor call to non-actor parents at `Weapon.cpp:1778`. Actor weapons retain their full vanilla condition-based misfire roll. No code path runs for actor through this override.

### Version requirement

The engine functor lookup at `Weapon.cpp:1781` was added in demonized commit `f27211ad`, first released as tag `2026.6.1` (2026-06-01). Before that release the script handled misfires via `npc_on_update` + `itm:unload_magazine()` (the older mag-dump approach); `GetConditionMisfireProbability` did not exist as a module field. Our override installs to a missing field on older releases, the engine never calls it, no error fires. AlifeTactics's `DEMONIZED_MIN_VERSION` is not raised for this; the feature is informational at the dep gate layer. On AOEngine the `xr_weapon_jam` module is not loaded (no modded-exes db0 overlay), and the `if xr_weapon_jam then` guard skips the install entirely.

### MCM

`jam_enabled` master toggle: on, the override returns 0 for NPC misfire probability; off, it forwards to the original modded-exes function. Effective on the next engine functor call.

---

## Ammo

Veteran-rank-and-up NPCs fire AP from the loose ammo they carry; each engagement has a rank- and fire-rate-weighted chance to consume one AP box, until the NPC runs out and reverts to vanilla magic FMJ. NPCs drop no AP boxes as loot.

Engine context: while `unlimited_ammo()` is TRUE (the stalker default, `ai_stalker.cpp:78,1585`) the magic refill at `WeaponMagazined.cpp:559-571` copies `m_DefaultCartridge` keyed from `m_ammoTypes[m_ammoType]` and never consumes inventory, and the reload does not re-derive `m_ammoType` -- so `wpn:set_ammo_type(idx)` re-keys what is fired and holds for the online session. `get_ammo_count_for_type` sums loose belt+ruck boxes only (`Weapon.cpp:1727`), so the AP an NPC carries comes from AlifePlus trade/loot; vanilla gives NPCs zero loose ammo (`xrs_rnd_npc_loadout.script:215`, ammo-give block commented out).

### Budget is the inventory

No virtual ledger. The budget is the NPC's AP boxes themselves, stocked by AlifePlus trade and looting. `_find_ap` returns the first `AP_SECTIONS` entry in the weapon's `ammo_class` with a loose count > 0. `AP_SECTIONS` is the clean-AP set; degraded `_bad` / `_verybad` variants are excluded.

### Tick algorithm

`pick(npc, now)` is the public entry, driven by the ammo monitor pass (a vanilla time event every 5s over the spawn-filled roster; at_test drives it directly). Gates: cached `ammo_enabled`, `alive`/`IsStalker`, `character_rank() >= min_ap_rank`, `IsWeapon(active_item)`, non-empty `ammo_class`. Then it splits on `best_enemy()`:

- Combat tick: on combat entry or weapon change, `_find_ap` caches `idx`/`sec`; while AP is carried, hold `m_ammoType = idx` (re-asserted only if changed). AP is held for the whole fight, no mid-fight revert.
- Peace tick: once `best_enemy()` has been nil past `peace_debounce_ms`, the engagement ends -- roll `_decay_chance`; on a hit, `alife_release` one AP box of `sec`; if that section is now empty, set `m_ammoType = 0`.

### Decay chance

`ap_decay_base * (rpm / rpm_ref) * rank_weight`, clamped to [0,1]. `rpm` is the weapon's effective fire rate; `rank_weight` lerps from 1.0 at `min_ap_rank` to `rank_weight_floor` at `rank_ceiling`. So fast weapons burn AP quickly and high rank conserves it -- a legend bolt-action shoots AP almost always. Deleting a whole box is permanent: `try_advance_ammo` (`object_actions.cpp:131-169`) refills rounds inside surviving boxes but cannot recreate a deleted box, so counting boxes never fights the top-up.

### No death hook

Vanilla `decide_items_to_keep` (`xr_motivator.script:362` -> `death_manager.script:457`) `alife_release`s every ammo box with > 5 rounds on death. An AP box is 15 rounds, so NPCs drop no AP boxes with no module help. `npc_on_death_callback` fires at `xr_motivator.script:396`, after that release, so a death-time trim could not preserve AP anyway -- the old hook was removed.

### Rank gate

`character_rank() >= min_ap_rank`, veteran by default (the veteran floor in `configs/creatures/game_relations.ltx [game_relations] rating`). Below-threshold NPCs early-exit at the rank read.

### Tuning

The numeric tunables live in `configs/alifetactics/at_ammo.ltx` with matching script-side fallbacks. The `ammo_enabled` MCM toggle early-exits `pick` with no `m_ammoType` writes and no box deletion.

### Scope and limits

- Active weapon only; state re-scans on weapon change or next engagement.
- Decay is per-engagement, not per-shot (no NPC fire callback exists). A continuous siege counts as one engagement.
- Save/load resets `_state`, but depletion lives in the inventory (boxes are released, not tracked), so it persists for free; `_state` re-seeds on the next engagement.
- Requires a loose-AP source (AlifePlus trade/loot) and the magazine system off; with NPC ammo encapsulated in magazine items, `get_ammo_count_for_type` reads 0 and the NPC stays on FMJ.
- No interaction with `at_jam`; different engine paths, compose freely.

---

## Effects resolver

`at_effects_resolver.script` is the deliberate shared substrate for the combat effects MORE THAN ONE source feeds. It owns the `EFFECT` enum, the highest-wins combiner `resolve`, the source registry `register`, and the engine WRITES for these effects; the sources own the scans. Four invariants govern it:

- **I1.** An effect is a per-NPC value MULTIPLE sources feed and the resolver combines highest-wins at a fixed engine point. Most are numeric multipliers on a value the engine already computes (damage on a hit, dispersion on a shot, vision range at spawn), combined MAX or, for a reduction effect, MIN; the aura is a PRESENCE effect (boolean, OR-combined, applied by a particle call). A single-source value is NOT an effect - it stays in its owning module and writes itself.
- **I2.** Medkit healing is an ACTION, owned by `at_firstaid`: the NPC consumes an item and runs the `xr_eat_medkit` chain. Its heal rate and charge chance are parameters of that action, not effects.
- **I3.** The entry rule: a value enters the resolver ONLY when more than one source feeds it. One source means one writer and no clash, so it stays home and never touches `resolve`. This is why aim, vision speed, fire discipline, the move penalty, and passive regen are NOT effects.
- **I4.** Passive regen is a distinct engine lever from medkit healing: the condition velocity `m_fV_HealthRestore` (`EntityCondition.cpp:642`), not the `xr_eat_medkit` action. `at_gear` owns it (n039).

The effect set (multi-source only): DAMAGE_DEALT (special + electro artefacts), DAMAGE_RESIST (gravi + plates + chemical-interim), SHOT_DISPERSION (rank + thermal), VISION_RANGE (rank + binoc-day + NVG-night), AURA (every artefact class, OR-combined). NOT effects, single-source, written by their own module: AIM_SPEED + AIM_LEAD, VISION_SPEED, FIRE_BURST/INTERVAL (`at_reaction`), the move penalty (`at_accuracy`), PASSIVE_REGEN (`at_gear`, via n039).

### resolve and register

Sources sign up at `on_game_start` via `register(effect, provider)`, where a provider is `fn(npc) -> value|nil`. `resolve(npc, effect)` looks up the registry, calls each provider, and keeps the STRONGEST - the maximum for higher-is-better effects, the minimum for the reduction effects (DAMAGE_RESIST, SHOT_DISPERSION), a boolean OR for the aura. Sources never sum and never multiply against each other, and there is no clamp: the ceiling is the largest single tier value, which highest-wins enforces for free. A provider that does not apply returns nil and drops out; a real per-NPC penalty (a novice rank curve above 1.0) survives because it is a contributing value, not an absent sentinel.

### Performance spine: lazy providers, resolve at apply, the applier never scans

Each provider is LAZY: it walks the NPC's inventory (or reads rank) once on the first call for that NPC and caches the result in the SOURCE's own cache; later calls are table reads. `resolve` re-combines the cached answers - a few table reads plus max/min, microseconds on the hot path. So the one inventory walk per NPC happens inside a source provider on first use, never in an applier: the scan stays in the source, the apply in the applier, and they meet only at `resolve`. The net_spawn applier is the de-facto first caller, so it triggers the walk once and caches the whole record; by the time a hit or shot fires in combat, the before_hit and shot appliers read that cache and never walk mid-callback.

### The appliers (the resolver owns these engine writes)

- **net_spawn** (`npc_on_net_spawn`): VISION_RANGE via `xcombat.set_view_distance_factor` (n038, no-op on a floor exe), and the AURA via `xcreature.attach_particle` on the configured bone, or the first default fallback bone the skeleton has. resolve returns nil for a former carrier, so the vision bind is written to the neutral value and a dropped optic never keeps a stale factor. The aura is START-ONLY: it attaches for a carrier and re-attaches each online because the particle dies with the game object, with no stop path. A creature particle has no safe stop, because stop_particles trips the engine bone assert whenever the bone is not renderable (proven live on a healthy NPC), so a carrier that drops its artefact keeps the glow until it despawns.
- **before_hit** (`npc_on_before_hit` / `actor_on_before_hit`): the victim's DAMAGE_RESIST (down) and the attacker's DAMAGE_DEALT (up), each scaling `shit.power` through `xcombat.scale_hit_power`. Both multiply, so they compose.
- **shot** (`npc_shot_dispersion`): SHOT_DISPERSION scaling `temp_disp.dispersion` through `xcombat.scale_dispersion`. `at_accuracy` writes the single-source MOVE penalty on the same callback; both are multiplies, so subscriber order is irrelevant.

The resolver owns the aura attach (the `_emitting` map holds the bones attached per carrier, bone resolution over the configured list with a default fallback, start-only with no stop) and a public `refresh(npc)` a source calls after it invalidates its own cache on a gear change, to re-push the spawn binds. Death and unregister only clear the emitting mark, they never call stop_particles. Every apply carries a null-object `at_debug` timer (measured, 0.1ms avg / 2ms ceiling); a stagger returns only if a measured crowd-load first-online burst crosses it.

## Gear

MCM: Mechanics > Gear (`at_gear.script`). `at_gear` is the functional-inventory SOURCE: it walks what a stalker CARRIES and registers lazy providers into the effects resolver; it writes no multi-source engine field itself. An artefact grants one combat edge chosen by its anomaly CLASS, scaled by the section's intrinsic `tier` (tier x 2%, the ladder 4/6/8% over vanilla tier 2-4, a 10% ceiling at tier 5, D4). The class -> effect map (D6): gravi -> DAMAGE_RESIST, thermal -> SHOT_DISPERSION, electro and the six quest specials -> DAMAGE_DEALT, ballistic plates -> DAMAGE_RESIST, binoculars -> VISION_RANGE (day), NVG -> VISION_RANGE (night), chemical -> DAMAGE_RESIST (the D18 interim) or passive regen when the n039 bind is present, and every artefact -> the carrier AURA. Detection is by exact vanilla section name, read from the class lists in `at_gear.ltx`; a modded section not listed grants nothing. The class -> effect assignment is a design choice and lives in the script; the section membership is data in the ltx.

Providers register at `on_game_start`: DAMAGE_DEALT, DAMAGE_RESIST, SHOT_DISPERSION, VISION_RANGE, AURA. Each reads the one cached record for the NPC (the lazy scan) and applies the live toggles: `gear_enabled` (master), then one per combat effect - `gear_resist_enabled` (armour-plate and shielding-artefact damage resist, both behind this one toggle), `gear_dealt_enabled` (outgoing damage), `gear_disp_enabled` (aim steadiness) - plus `gear_binoc_enabled` / `gear_nvg_enabled` (optics) and `gear_aura_enabled` (the aura). The record is toggle-independent (raw strengths plus presence flags), so an option change never forces a re-scan. Eligibility rides in the providers per D23: the combat effects gate on a live, non-actor, non-zombied stalker; the AURA gates stalker + non-actor only, since a zombie still physically carries the artefact and the glow marks the loot. The one inventory walk keeps the strongest strength per effect (a channel never stacks across items - strongest wins, no aggregate cap), and the chemical strength is routed after the walk: to passive regen when the n039 bind is probed present, else folded into the resist pool.

The one single-source value `at_gear` owns and writes itself (never through the resolver, I3) is PASSIVE_REGEN: at net_spawn, behind `gear_regen_enabled`, it writes `xcombat.set_health_restore_boost` (n039), inert until that engine bind exists (a one-time `type(fn)=="function"` probe routes chemical, so the write is dead and chemical stays on DAMAGE_RESIST on today's exes). Optics read day/night live at the VISION_RANGE provider (`level.get_time_hours`) so a binoc/NVG carrier's range tracks the clock without a polling tick (D20). Cache: `npc_on_item_take`/`npc_on_item_drop` re-resolve a cached id (they fire for pickups, drops, and NPC-to-NPC `transfer_item` trades) and hand the spawn-bind re-apply to `at_effects_resolver.refresh`; `server_entity_on_unregister` drops the record; `at_gear.refresh(npc)` is the public re-resolve for a scripted grant with no take event. The aura particle name (`anomaly2\burer_prepare`, bone-verified) lives in `at_gear.ltx` because the carrier set is a gear fact, but the resolver applies it; only a config-referenced particle name is valid (a name living solely in `particles.xr` is an engine fatal with no script-side check, `r4.cpp:738-748`, and `anomaly2\burer_prepare` qualifies through `m_burer.ltx:272` `Particle_Gravi_Prepare`). A distorting stalker IS a visible, huntable artefact drop.

Carrier persistence: benefit items stay on carriers through the per-category COUNT-BAND policies, not a protection allowlist (there is no `xinventory_protected.ltx`). `xinventory.get_category` classifies each item; artefacts resolve to `artefact`, binoculars to `weapon`, NVG torches to `device`, armour plates (`af_kevlar`/`af_plates`/`af_kevlar_up`/`af_plates_up`/`fieldcraft_plate_attch`) to their own class. Nothing is `untouchable` by kind (that bucket is only quest/anim/blacklist plus runtime story-id/companion/strapped). Each policy keeps a per-key floor and sheds the surplus: the AlifeGuard cull (`ag_inventory_policy.ltx`) caps `artefact = 3`; the AlifePlus trade/barter/stash/loot policies keep 1 (`0,1`). Because binoculars, NVG, and plates fall into `weapon`/`device`/other categories that are otherwise sold or cap-1, each of the five NPC-inventory policies carries explicit SECTION rows (`wpn_binoc`, `device_torch_nv_*`, the five plate sections) so they survive their category's rule. The player-facing shelves (`ap_market_policy`, `ap_outpost_stock_policy`) never touch NPC-carried gear. Circulation is loot and combat only.

---

## Ballistics

The ballistics recorder is a section of `at_world_trace.script` (MCM Development > World behaviour debug, the one world-trace toggle). An outcome recorder over the engine's shot and impact feeds, reads only, OFF by default with zero cost (no callbacks registered until the toggle). Two gates: the toggle drives CAPTURE (in-memory counters + a 200ms actor-velocity poll for the aim split), the log level drives OUTPUT (per-bullet lines at DEBUG, minute tables at INFO to `alifetactics_world.log`). No on-screen panel (removed 2026-07-14). Per minute-table tier line: hit% split still/moving, arrival conversion, damage, hits per minute, per-driver hit rates, the burst histogram per weapon kind, plus the per-bullet aim split - `ang` (mean fired-direction error off the shooter->actor line) and `ahead` (% of bullets passing in front of actor travel = lead overshoot vs tracking lag, computed only while the actor moves). `at_world_trace.reset()` zeroes the counters and restarts the session clock - the bench-run boundary, since module state survives a save load within one game session.

Per bullet fired (`npc_shot_dispersion`) it records tier, weapon kind, shooter motion, and the driver context read directly from the modules at capture time - `at_combat.get_maneuver` / `at_commitment.get_hold` / vanilla - so no cross-log correlation ever happens. Per bullet landed (`bullet_on_impact`): hit on the actor or a near miss inside 10m. Per actor hit: damage attributed to the shooter's tier. The minute tables carry per-tier hit% split standing/moving (the Moving Fire differential), arrival conversion, damage, hits per minute, per-driver hit rates, and the burst-length histogram per weapon kind (the scheme fire-discipline evidence).

### Observability: measure outputs, never intent

Two concerns, two modules, two log files, no logging-only middle files. CODE tracing - what the mod's code decides and does - goes to `alifetactics.log` through the one primitives file `at_debug.script` (one logger, the `at_debug.on()` gate, shared `flag`/`show` formatters, the log level refreshed mod-wide from one lifecycle); every gameplay module calls `at_debug.dbg/info/warn` at its own sites, in its own words, and no module owns a logger or a debug boolean. WORLD tracing - whether the fight physically looks right, measured from positions and bullets - goes to `alifetactics_world.log` through `at_world_trace.script` alone (the slide watchdog + the ballistics recorder, one toggle, one logger). One is the code's account of itself, the other is the world's account of the code.

The 2026-07-24 lesson governs every "does it look/work right" trace: measure what the NPC actually DID (where its body moved, where its bullets went, whether a need cleared), never what the engine INTENDS (movement_type target, body_state, animation_count). Intent fields read as frozen whenever an animation plays, so a healthy vanilla NPC and a genuinely stuck one give identical reads - the earlier watchdog judged appearance from those fields and produced fake conclusions at volume. The three signals below are outputs; none is an appearance guess.

**Slide watchdog** (`at_world_trace.script`, MCM Development > World behaviour trace). Reads-only, OFF by default, zero cost until the toggle. It reports ONE thing, the one visual defect provable from cheap reads: a SLIDE, the body travelling a real distance while its movement_type is never a locomotion type (walk/run/steal) - it moved with no walk cycle, the visible glide. Measured from POSITION, an output that cannot go stale; violations only to `alifetactics_world.log`, each with its driver (`at_combat.get_maneuver` / `at_commitment.get_hold` / vanilla). Two detectors: a 150ms monitor pass logs one line per slide EPISODE when displacement from the start of a non-locomotion stretch crosses 1m (a still NPC never accumulates displacement, so it never trips), and a 60ms hit-triggered watcher (`SLIDE_ONHIT`) catches the sub-pass knockback (the displacement persists; `overlay_at_hit` is context, not verdict). Everything the old watchdog inferred from intent fields (posture flap, weapon flap, misaim, overlay pin, frozen-reload) is deleted.

**Firing the wrong way** (`at_world_trace.script`, ballistics section). The angle between a round's own direction and the shooter->best_enemy line, measured from the BULLET on `bullet_on_impact`, not from facing. A large angle is the NPC firing away from its target, the provable form of "shooting the wall". Reported per rank tier as the average off-target angle and the fraction of rounds past 50deg, beside the actor-relative aim split.

**need_cleared** (`at_combat.script`, at maneuver end). Did the maneuver negate the situation it fired on? `at_combat_doctrine.recheck_need` re-runs the row's own `check_need` with a fresh memo; a need still holding at hand-back (`need_cleared=n`) is the churn signal - the maneuver ran and did not solve its problem. Debug-path only, on the `end` trace line.

---

## What the engine does and what we feed it

The architecture principle is to feed engine memory and state, not fight it. Per-system summary:

| System | Engine state we write | Engine APIs called |
|---|---|---|
| Disclosure | Per-NPC CEnemyManager selection fields at net_spawn (hit-redirect, visible-enemy bias); per-hit `script_danger` investigate stamps on earshot squadmates; floor fallback only: victim combat registration | `xcombat.set_hit_redirect`, `xcombat.set_visible_enemy_bias`, `xr_danger.set_script_danger`; floor: `xcombat.register_in_combat` |
| Healing | NPC health field, bleeding field, `healing_charge` se_var | `change_health`, direct `bleeding =` write, `se_save_var` |
| Accuracy | Per-shot dispersion radius via callback return: the single-source MOVE penalty written directly, the rank cone registered as a SHOT_DISPERSION provider | (subscribes to `npc_shot_dispersion`; `at_effects_resolver.register`) |
| Reaction | Per-NPC aim (min_speed + min_angle + predict), vision speed, fire-queue scales at net_spawn; the rank VISION_RANGE slice registered as a provider | `xcombat.set_aim_params`, `set_vision_speed`, `set_fire_queue_scale`; `at_effects_resolver.register` |
| Gear | No multi-source write of its own: registers DAMAGE_DEALT/RESIST/SHOT_DISPERSION/VISION_RANGE/AURA providers; writes only its single-source passive regen at net_spawn | `xinventory.iterate_inventory`, `at_effects_resolver.register`, `xcombat.set_health_restore_boost` (n039, inert) |
| Effects resolver | Per-hit `shit.power` (attacker DAMAGE_DEALT up, victim DAMAGE_RESIST down), per-shot `temp_disp.dispersion` (SHOT_DISPERSION), per-spawn view-distance factor (VISION_RANGE) and carrier particle (AURA) | `xcombat.scale_hit_power`, `xcombat.scale_dispersion`, `xcombat.set_view_distance_factor`, `npc:start_particles` |
| Combat | NPC GOAP action (at_combat_action), Pattern B preconditions on action_combat_planner/action_danger_planner/xr_danger.actid/state_mgr+2/alife, set_dest_level_vertex_id, state_mgr.set_state, set_body_state, set_movement_type, set_sight | GOAP `add_evaluator`/`add_action`/`add_precondition` (custom evaid/actid), `npc:best_cover`, `level.vertex_in_direction`, `db.used_level_vertex_ids` reservation |
| Conduct | Combat body-state override per action application (`csbs_flags.body_state`) | (subscribes to `npc_on_combat_set_body_state`); `xcombat.has_obstacle_to_target` (crouch-eye shot ray) |
| Danger | Danger scheme evaluators/action installed onto the winning `xr_danger` binder, `script_danger` per-id table for sound-source dispatch | Patches `xr_danger.{setup_generic_scheme,add_to_binder,configure_actions,reset_generic_scheme,get_danger_time,set_script_danger,has_danger}`; relies on the winner's `npc_on_hear_callback` / `npc_on_death_callback` feeders |
| Jamming | Module-level function table on `xr_weapon_jam` | Lua function assignment (`xr_weapon_jam.GetConditionMisfireProbability = ...`) read by engine functor lookup at `Weapon.cpp:1781` |
| Ammo | CWeapon `m_ammoType` field via `wpn:set_ammo_type(idx)` (re-keys `m_DefaultCartridge` for magic refill ballistics); per-engagement box-delete decay (`alife_release` one AP box on a rank/rpm-weighted roll); reverts to `m_ammoType = 0` when the section is empty | `npc:active_item`, `wpn:set_ammo_type`, `wpn:get_ammo_type`, `wpn:get_ammo_count_for_type`, `npc:best_enemy`, `npc:character_rank`, `npc:iterate_inventory`, `alife_release` |

The engine then runs its own combat detection (property_enemy, m_combat_mask, agent_memory propagation) on the state we wrote. No system reimplements engine behavior; each one nudges engine state to produce the desired outcome.

---

## See also

- Task queue: `stalker-dev/doc/todo/todo-alifetactics-next.md`
- Takeover build plan: `stalker-dev/doc/todo/todo-combat-takeover-v2.md`
- Adversarial reviews: closed (2026-07-02, 2026-07-08, 2026-07-23); records in stalker-dev git history, deferred release-gate items in `todo-alifetactics-next.md` (t167)
- Engine PR queue: `stalker-dev/doc/todo/todo-demonized-exes.md`
- xlibs architecture: `stalker-mods/xlibs/doc/architecture.md`
- AlifePlus architecture: `stalker-mods/AlifePlus/doc/architecture.md`
- AlifeGuard architecture: `stalker-mods/AlifeGuard/doc/architecture.md`
- AlifeBalance architecture: `stalker-mods/AlifeBalance/doc/architecture.md`
