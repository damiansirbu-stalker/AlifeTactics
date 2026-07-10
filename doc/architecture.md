# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Independent user-facing systems: a hit-share force-disclosure, a self-heal data + animation layer, a per-rank weapon accuracy curve, a danger scheme that layers bug fixes and three toggleable improvements onto whichever xr_danger a modpack ships (a runtime patch, not a file override), and an intermittent combat takeover that briefly borrows an NPC from vanilla for one committed maneuver where vanilla is weak (the takeover overrides zero vanilla combat files, so it coexists with vanilla / GAMMA / AI Rework / RCAI / Useful Idiots / Mora). No shared substrate.

Built on xlibs (xsquad, xttltable, xtime, xprofiler, xlog, xmcm, xslice, xcreature).

Part of a four-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** tunes existing rates and counts, **AlifeGuard** keeps alife state clean, **AlifeTactics** controls how NPCs fight in combat (this mod).

---

## Invariants

Project-wide constraints. Every system holds all of them; a change that violates one is wrong even when it works.

- **No per-frame work. Never.** Ongoing work runs on a scheduled tick (a fixed interval - the vanilla `CreateTimeEvent` queue) or on a discrete engine event (a hit, a shot, a spawn, an option change) - never continuously every frame, and never on a per-frame engine callback, however small the body: dispatch work in front of a throttle IS per-frame work (the pre-2026-07-10 at_combat monitor rode `npc_on_update` and was rebuilt onto a time event for exactly this). We never place code on a path the engine itself runs every frame (a visibility functor, a fire functor). Frame-spreading a bounded one-off batch (xslice, 1 item per frame) is the one allowed use of the frame; it completes and stops. Full rule: `doc/standards/code-standards.md` "No Per-Frame Work".
- **2ms is the ceiling.** Every measured flow targets 0.1ms average per call with a hard 2ms ceiling - an eighth of a 60fps frame. No exceptions: cold start, save load, and level transition all count, and debug-only tools count too. A flow that averages above 0.1ms or ever crosses 2ms is a regression and gets a perf task. When a flow costs too much, the answer is a simpler design, not a faster version of the same one.
- **Measured, not asserted.** Every flow carries a duration field in its DEBUG trace (`walk=us`, `[us]`, `[ms]`); the timers are null objects when debug is off, so measurement costs nothing live. No mechanism (cache, backoff, throttle, precompute) is justified by an unmeasured cost - the decide-path decline backoff was built and removed the same day for this, and the per-row walk timings that replaced it are the gate any future cost mechanism must pass.
- **No file overrides.** AT replaces no vanilla file. Every system attaches by callback, function patch, save-wrap, DLTX overlay, scheme patch, or the time-boxed takeover transaction. Composition with modpacks falls out of the attach mechanism, never out of luck.
- **Engine truth.** Every mechanism claim in this document carries an engine source cite (file:line into xray-monolith or vanilla Anomaly). A behavior that could not be proven from source does not ship; where the engine had no seam, the seam was added upstream first (the demonized PRs: action-switch veto, per-NPC aim and vision, fire-discipline binds).
- **The takeover is a bounded transaction that TRIES to solve its problem.** Vanilla owns every NPC by default. AT borrows one NPC for one committed, time-boxed maneuver and releases it - at most one open maneuver per NPC and at most `max_concurrent_maneuvers` open across all NPCs (a runaway failsafe), ended on arrival, a hard cap, or a broken premise, cleaned up on death and despawn. There is no reseize cooldown: every row's need states the FULL problem and the maneuver's success negates it (kite clears its own weapon minimum, retreat/flee push past the enemy's reach, counterflank flips the engine's enemy selection, a finished snipe hold resets the stall measurement), so a solved problem does not re-fire and a recurred or unsolved one legitimately does - three kites under sustained pressure are three correct transactions. Every maneuver is locked to the target it was staged against (`state.enemy_id` resolved via `xcombat.resolve_enemy`, in the shared shell so every current and future row inherits it): the NPC aims at him and only him for the maneuver's life, and his death or despawn ends the maneuver at once (`target_lost`) with vanilla re-selecting from there. AT is an interrupt over vanilla, never the combat brain.
- **xcombat boundary.** Every NPC combat command and read goes through an xcombat (xlibs) primitive; AT makes no raw engine combat call. AT owns policy (when, whom, which maneuver); xcombat owns mechanism (how to issue it to the engine).
- **Debug is free when off.** Every trace call gates on one boolean. The off path builds no string, allocates nothing, and crosses no luabind bridge.

---

## Status

Version 1.1.1.

| Module | Type | State |
|---|---|---|
| `_at_deps.script` | infra | done |
| `at_mcm.script` | infra | done |
| `at_test.script` | infra | done |
| `at_hud.script` | infra | done (live debug HUD, noop unless enabled) |
| `at_disclosure.script` | feature | done |
| `at_crossfire.script` | feature | done |
| `at_healing.script` | feature | done |
| `at_accuracy.script` | feature | done |
| `at_combat.script` | feature | v2 intermittent takeover: solo maneuver set (counterflank, retreat, flee, kite, snipe), actor-party fights only; squad coordination and enemy openings are the open phases (todo-combat-takeover-v2.md) |
| `at_danger.script` | feature | done (function-level patch onto the winning xr_danger) |
| `at_jam.script` | feature | done (NPC misfire suppression via the modded-exes functor) |
| `at_ammo.script` | feature | done (AP fired from carried boxes, box-delete decay) |
| `zzz_at_healing_patch.script` | feature | done (vanilla xr_eat_medkit re-roll suppressor) |
| `configs/ai_tweaks/mod_xr_eat_medkit_at.ltx` | data | done |
| `configs/ai_tweaks/mod_xr_danger_at.ltx` | data | done (delete-lines-only DLTX: drops the collision keys; the copied value rows removed 2026-07-10) |
| `configs/alifetactics/at_combat_config.ltx` | data | done (Combat takeover tunables) |
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
│   │   │   └── at_ammo.ltx                    # Ammo tunables
│   │   ├── ui/
│   │   │   └── ui_at_stats.xml                # at_hud HUD layout
│   │   └── text/
│   │       ├── eng/ui_st_mcm_at.xml           # English MCM strings
│   │       └── rus/ui_st_mcm_at.xml           # Russian MCM strings
│   ├── scripts/
│   │   ├── _at_deps.script                    # dependency gate
│   │   ├── at_mcm.script                      # MCM configuration
│   │   ├── at_combat.script                   # Combat > Maneuvers (engine half: GOAP takeover, gates, lifecycle)
│   │   ├── at_combat_doctrine.script          # Combat decision half (catalog, should/can methods, arbiter)
│   │   ├── at_combat_trace.script             # Combat DEBUG tracing + telemetry (noop when off)
│   │   ├── at_accuracy.script                 # Effectiveness > Accuracy
│   │   ├── at_disclosure.script               # Effectiveness > Disclosure (hit-share force-disclosure)
│   │   ├── at_danger.script                   # Effectiveness > Danger (danger scheme fn-patch)
│   │   ├── at_crossfire.script                # Effectiveness > Crossfire (friendly-fire damage block)
│   │   ├── at_healing.script                  # Mechanics > Healing
│   │   ├── at_jam.script                      # Mechanics > Jamming (modded-exes xr_weapon_jam override)
│   │   ├── at_ammo.script                     # Mechanics > Ammo (NPC ammo simulation)
│   │   ├── zzz_at_healing_patch.script        # vanilla xr_eat_medkit re-roll suppressor
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

The MCM menu is a five-category gameplay tree plus a Development tab, the canonical structure (source of truth `at_mcm.script`; the page names here are the MCM labels): **Combat** (Maneuvers, Behaviors), **Effectiveness** (Accuracy, Disclosure, Danger, Crossfire, Commitment, Reaction, Range, Resistance), **Mechanics** (Healing, Jamming, Ammo), **Effects** (wip), **Mutants** (wip), and **Development** (log level, debug HUD toggle and position, reset to defaults). Effectiveness > Commitment (the shuffle intervention — its engine keystone n023 is merged, the Lua layer is wip), Behaviors, Effects, Mutants, and the Reaction/Range/Resistance pages are all wip. One leaf = one name = one `at_<leaf>.script` = one MCM page = one master toggle; the system sections below follow the category order. The Scope column is the scope rule below, at a glance.

| System | File | MCM page | Master toggle | Scope | Composition |
|---|---|---|---|---|---|
| Maneuvers | `at_combat.script` | Combat > Maneuvers | `combat_enabled`; per-maneuver toggles | All NPCs | transaction-override |
| Accuracy | `at_accuracy.script` | Effectiveness > Accuracy | `accuracy_enabled` | All NPCs | callback |
| Disclosure | `at_disclosure.script` | Effectiveness > Disclosure | `disclosure_enabled` | All NPCs | callback |
| Danger | `at_danger.script` | Effectiveness > Danger | bug fixes always-on; three toggleable improvements | All NPCs | scheme-patch |
| Crossfire | `at_crossfire.script` | Effectiveness > Crossfire | `crossfire_enabled` | All NPCs (actor excluded as participant) | callback |
| Healing | `at_healing.script` | Mechanics > Healing | `healing_enabled` | All NPCs | fn-patch |
| Jamming | `at_jam.script` | Mechanics > Jamming | `jam_enabled` | All NPCs | save-wrap |
| Ammo | `at_ammo.script` | Mechanics > Ammo | `ammo_enabled` | All NPCs | callback |

Composition classes, what each does to the surrounding stack: `callback` subscribes to an engine callback and adds to it (composes with any other subscriber); `fn-patch` replaces a vanilla module function, rescheduling through the same lookup so it holds; `save-wrap` saves the prior function and forwards to it when disabled (composes with a prior installer); `transaction-override` suppresses whatever brain is installed, but only for the seconds it holds one NPC; `scheme-patch` installs a generic scheme's binder and evaluators onto whichever file won the MO2 slot, at `on_game_start`, so it layers onto a rival override instead of excluding it (Danger is the one such leaf).

### Scope: every system runs on every fight

No system is actor-gated. The Combat takeover targets whatever enemy the engine selected for the NPC (`best_enemy()` - a stalker, the player, or a mutant); the actor gate it shipped with was removed 2026-07-08 once the measured decide costs showed the complication bought nothing. Disclosure, Healing, Accuracy, and Danger never read who the enemy is. Crossfire, Jamming, and Ammo exclude the player only as a participant (the player's own hits, the player's own weapon) - a participant exclusion, not a target gate.

---

## Combat

Status: built incrementally. On `main`: the GOAP graft (`xcombat.install_takeover`), the maneuver-catalog spine (`at_combat_doctrine`), and the full solo maneuver set — counterflank, retreat, flee, kite, snipe — with the fire discipline owned by `xcombat.set_combat`. It runs only on fights the actor is party to (2026-07-10; the 2026-07-08/09 any-enemy + seize-radius form is retired); squad coordination and enemy openings remain the open phases. Full phase plan + the decision record: `stalker-dev/doc/todo/todo-combat-takeover-v2.md` (t130-t139 + the Plan section). The full-takeover v1 is preserved on the `combat_takeover` branch.

### Two systems

AT's combat is two independent systems over vanilla, not one:

1. **Maneuvers** — the takeover. AT begins a maneuver on a fighting NPC, drives it to its end, then hands the NPC back. It launches behaviors vanilla lacks or must be forced into (flee, kite, flank, snipe, suppress). This is most of this section.
2. **Commitment** — the anti-shuffle veto (MCM page: Effectiveness > Commitment). AT never takes the NPC; it vetoes vanilla's own action switches to stop the break-contact shuffle, keeping the NPC on a good action (a player right in front and already being shot) until an important event forces re-evaluation. It runs on every fighting NPC and launches nothing. See "Commitment" below.

The maneuvers impose (block vanilla briefly, run our behavior); the shuffle intervention composes (leave vanilla running, deny only bad switches). The two are separate and can be built and shipped independently.

### Scope: fights the actor is party to

AT seizes only problems the actor is party to - one rule, carried as row data, not as a gate branch. Every catalog row declares whose problem it answers (`vs_actor`): the four fight rows (retreat, flee, kite, snipe) walk only when the actor IS the NPC's selected enemy (`state.enemy_id == AC_ID`), and counterflank - the actor-party row - walks only when he is NOT (its problem is the actor standing at contact range while the NPC's committed fight is someone farther). NPC-vs-NPC fights are never seized: the triggers that make maneuvers legible - pressing a weapon minimum, chasing a routing man - are things the player does, and an NPC-vs-NPC maneuver reads as generic repositioning (the 2026-07-09 `seize_actor_radius_m` perceivability radius bought a branch, a config key, and a cached distance read on the begin path for that thin payoff; removed 2026-07-10 - the 50m radius survives only as at_healing's presentation radius, a different system).

Within a qualifying fight the target is never picked by AT: `best_enemy()` at STAGE time, the same object vanilla combat drives against (counterflank is the one row that stages the ACTOR - the row declares its committed target and the shell stages what the row declares). From staging on, the maneuver is COMMITTED to that target (2026-07-09): `_begin_maneuver` and `_update_maneuver` resolve the staged id via `xcombat.resolve_enemy` instead of re-reading `best_enemy`, so a kiter never re-aims mid-maneuver at whoever the brain glanced at, and a dead or despawned target ends the maneuver (reason `target_lost`) with vanilla picking the next fight. A weaponless enemy (a mutant) reads as the rifle range band where a row needs the enemy's weapon, and the reads are otherwise enemy-agnostic. The one other actor read in the decide path is flee's base search scoping to the CURRENT level via the actor's level id - valid because a fleeing NPC is online, and online NPCs are on the actor's level by construction.

### The model: a takeover transaction

Vanilla owns every NPC by default. AT does not run combat; it seizes one NPC for one committed, time-boxed maneuver that TRIES to solve one stated problem, then releases. AT is an interrupt over vanilla, not the combat brain. There is no share knob — an NPC is seized only when a need and a feasibility check both fire for it, so selectivity is inherent. At most one open transaction per NPC, at most `max_concurrent_maneuvers` (20) across all NPCs — a runaway failsafe, never a behavior lever, checked before the catalog walk; with the actor-party scope it should never bind.

The transaction law (2026-07-10, replacing the reseize cooldown): a row's need states the FULL problem, and the maneuver's success negates it. Kite's back-off ends outside its own weapon minimum; retreat's cover and flee's base are past the enemy's reach by construction; counterflank makes the actor SEEN, which flips the engine's own enemy selection; a finished snipe hold resets the stall measurement, so the next hold needs a fresh 4s standoff. A solved problem reads false at the next begin check and does not re-fire; a recurred problem (the player pressing back inside the minimum) or an unsolved one (a rout that capped in place, the enemy still in reach) legitimately fires again — the trigger is the throttle, and no timer stands between a real problem and its answer.

Seizability is a gate separate from the trigger. `_can_seize` (`at_combat.script`) takes an NPC only if it is armed (an unarmed NPC would deadlock, since the engine's own rearm lives in the blocked combat planner) and not already in a smart cover (vanilla owns that micro). There is no indoor gate: the old surge-shelter-radius `is_indoor` proxy false-flagged open ground (t97), so the takeover fights everywhere; `xcombat.is_indoor` was rebuilt as a real roof-plus-walls raycast and stays available if a future maneuver needs it.

Three states per NPC, real cost only in the last:
- Not fighting — one `best_enemy` read per begin check, nothing else.
- Fighting, no maneuver open — the begin_maneuver check runs (600ms), engine-memory reads only, no raycasts.
- Maneuver running — the update_maneuver and end_maneuver checks run (200ms each): re-apply the row's state, hand back on arrival, the cap, or a broken premise.

### The decision pipeline

Every maneuver is two methods, split so the need is always cheap and the geometry is always bounded (2026-07-10, replacing the one-row rotation). `should_maneuver` is the need: a compare over memoized reads (positions, distance, health, my and the enemy's weapon kind, the actor position and relation) that states the row's FULL problem and returns the situation name — actor_close, too_close, hurt, stalled — or nil; it is forbidden the raycast/path/search class of work. `can_maneuver` is the feasibility: it owns the geometry (and the premise reads that cost luabind — flee's and snipe's under-fire pair, snipe's actor-aim read) and returns the destination vertex when the maneuver is doable here, or nil to decline — one pass answers both "can he?" and "where to?", and the vertex feasibility validated is the vertex the engine executes (resolve once, never re-resolve).

Per begin_maneuver check (on the monitor pass, 600ms) `resolve_maneuver` arbitrates the catalog in priority order — counterflank, retreat, flee, kite, snipe — from a per-NPC cursor. A row out of scope (`vs_actor` against the walk's fight class), or whose need is absent, or whose MCM toggle or faction/weapon palette rejects it (each maneuver has its own checkbox on Combat > Maneuvers; a disabled row traces as `off`, an out-of-scope one as `scope`) falls through to the NEXT row in the SAME tick, so a row that does not apply never delays the rows below it. The first row whose need holds runs its `can_maneuver` — the ONE geometry probe this tick — and ends the walk, pick or decline. In both cases the cursor moves past that row: a declined row (flee under fire, no rear cover) defers to the next candidate and retries after at most one lap (~3s), and a picked row hands the NEXT decision to the row below it — which is what escalates a still-hurt timid NPC from retreat to flee with no escalation state. The cursor resets to the top when the NPC leaves the fight. Cost per tick is bounded by construction — at most one lap of need compares (microseconds) plus at most one geometry probe — so no budget machinery exists on this path. Row order is the priority: counterflank answers an ignored enemy actor at contact range before anything else, retreat answers pressure before the rout (first retreat, then flee if possible), kite answers a violated weapon minimum, and snipe sits last so a pressed sniper retreats and only the merely-stalled one plants. There is no shared read-all: each row reads its own world, and a value two rows share — the faction, my and the enemy's weapon kind, the npc-enemy distance, the actor position — is memoized lazily on the walk, for that walk only. With debug tracing on, the decision line carries every examined row plus per-layer `should=`/`can=` microseconds on the row that ran its feasibility — any future cost mechanism on this path (a decline backoff was built and removed 2026-07-08) must first be justified by those numbers. The things vanilla does well — the opener, re-target, search, turret, grenade dodge — are not situations AT answers (the grenade trigger was removed 2026-07-10: vanilla's own grenade reaction is faster than a staged walk-to-cover, and a seized NPC eating a rare grenade was already the accepted trade).

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
      catalog walk from the per-NPC cursor, priority: counterflank -> retreat -> flee -> kite -> snipe
          per row: scope (vs_actor vs the fight class) -> toggle ->
                   should_maneuver (need) -> palette -> can_maneuver (feasibility)
          need absent = next row, SAME tick
          can_maneuver nil = decline: cursor past the row, tick over
          can_maneuver vertex = picked: cursor past the row
      picked -> stage row + destination + the row's committed target, raise the gate
  engine plan solve -> grafted action initialize
      -> _begin_maneuver: send_to(dest) + apply_state    (the one engine write)
  maneuver open -> update check (200ms): row.update re-applies fire/posture/movement
                -> end check (200ms): arrived, cap, or broken premise -> _end_maneuver
      -> gate down, cover reservation released, stall tracker reset, vanilla resumes
```

### The maneuvers, and how each one works

The catalog holds five built maneuvers — counterflank, retreat, flee, kite, snipe, in priority order. The base-of-fire and maneuver-element maneuvers (suppress, assault, push, flank) are later squad phases, not yet in the catalog. Each built maneuver is one row in `at_combat_doctrine.script`: its `should_maneuver` (the need it answers — the FULL problem, whose negation is the maneuver's success), a faction/weapon palette, its `can_maneuver` (feasibility; where it sends the NPC), an optional `should_end` (the premise re-checked while it runs), and the fire/posture/movement it drives.

Each row opens on its need: **actor_close** (an enemy actor inside `counterflank_actor_dist_m` while the committed target is farther than him) for counterflank, **hurt** (health below `hurt_frac` AND the enemy inside his own weapon's reach) for retreat and flee, **too_close** (enemy inside MY weapon's minimum) for kite, **stalled** (held position ~4s since the last maneuver, the enemy not closing, and inside MY weapon's band) for snipe. The walk takes the first in-scope row whose need holds, whose palette matches, and whose `can_maneuver` yields a destination.

| Maneuver | Fires on | Applies to | Runs to | Weapon; move | Ends on |
|---|---|---|---|---|---|
| counterflank | actor_close | any NPC fighting someone other than the actor | holds its own spot, aimed at the actor | fire; still | 3s hold |
| retreat | hurt, enemy in reach | brave factions; timid ones on the first pressure decision | cover behind him, never closer to the enemy (reserved) | fire; walk | arrival or 8s |
| flee | hurt, enemy in reach, as the last man, once the shooting pauses | timid factions | a friendly base 100m+ away, rear-biased | holstered; run | arrival or 20s |
| kite | too_close | any NPC | a clear back-lane, a weapon-set distance to the rear (shotgun 4m .. sniper 10m) | fire; walk | arrival or 8s |
| snipe | stalled, past the enemy's weapon maximum, not under fire, not under the actor's aim | w_sniper only | holds its own spot | precision-fire; still | 8s hold or broken premise |

- **counterflank** is the actor-party hold: an enemy actor inside `counterflank_actor_dist_m` (5) while the NPC's committed target is FARTHER than him — he is shooting past the man who can kill him first, the enemy_manager's seen-now scoring artifact (a seen distant target outranks the unseen man on his shoulder, enemy_manager.cpp:110-175). The row stages the ACTOR, aims at him for a 3s hold with no movement; the turn makes him SEEN, and seen at contact range wins the engine's own selection outright, so at hand-back `best_enemy` IS the actor and vanilla drives the new fight — the transaction's success is the engine changing its mind. One raycast at feasibility (a wall between them means facing him is theater, decline); the need is pure math over the walk memo plus one relation read paid only inside the radius. Universal palette; only fires when the NPC is already fighting someone else, so player stealth against idle NPCs is untouched.
- **retreat** is the standing-line pressure response: badly hurt with the enemy inside his own weapon's reach, pull back to cover BEHIND you while still firing — the search centers 8m to the rear - deeper than the 7m cover-search radius, so the circle excludes the cover the NPC already holds (at 4m best_cover kept re-picking his own cover and every retreat declined toward_enemy, 2026-07-11) - and the winning vertex must be farther from the enemy than the NPC stands, so a retreat always grows the distance; toward-the-shooter cover declines instead. It reserves the cover vertex so two NPCs never pick the same spot. Factions: army, dolg, freedom, killer, isg, stalker, bandit, greh, plus the timid factions (a hurt ecolog pulls back armed before he routs — retreat is the first pressure answer for everyone eligible).
- **flee** is the rout, and a rout is for the broken: hurt with the enemy in reach, only the last man (squadless, or sole survivor of his squad), and only once the shooting pauses — a fresh hit or a perceived shot from that enemy (`xcombat.is_under_fire`, the NPC's own danger memory) declines it, so nobody turns his back mid-burst; the declined row defers and is re-asked a lap later, so the rout fires when the fire lifts. He runs HOLSTERED to a friendly base (no enemy squad stationed) at least 100m away, biased to the rear hemisphere so every stride gains distance, and declines when no such base is reachable. Factions: ecolog, clear sky, renegade. Monolith and zombied get neither retreat nor flee — fanatics do not break.
- **kite** answers the enemy getting inside your minimum range: back off a weapon-set distance (`KITE_BACK`: shotgun 4m, pistol 5, SMG 6, rifle 8, sniper 10), still firing the whole way — a visible fighting withdrawal that always ends well outside the weapon's minimum. The old form backed off "minimum minus current distance", which in practice was a 2m hop; replaced 2026-07-09. Universal — any faction, any weapon; a gun inside its minimum is half useless no matter what the enemy holds. The back-off negates `too_close` by construction, so a completed kite does not re-fire — and the player pressing back in re-creates the problem, so sustained pressure produces kite after kite with no dead window. `find_flee_lane` runs a two-phase search — straight-back then +-45 then +-90 at the full distance, the same fan at half distance, then the longest straight stub — and declines only when fully boxed in.
- **snipe** is a fire-mode hold, not a movement: a sniper stuck in a standoff plants and takes precision shots — but only past the ENEMY's weapon maximum, where the enemy cannot effectively answer (past 15m against a shotgun, past 80m against a rifle), only with the enemy inside his OWN band, and only while unbothered: under fire or under the actor's crosshair (`snipe_targeted_deg`) it declines, and the same premise is re-checked while the hold runs (`should_end`) — a hit landing or the player's aim arriving mid-hold ends it now, not at the cap. A finished hold resets the stall tracker, so the next plant needs a fresh 4s standoff — vanilla owns the gap. The fire discipline downgrades it to weapon-up (no shot) when there is no clear line, so it never fires at a wall.

retreat sits above flee in the catalog: a hurt timid NPC pulls back to cover first, and because a pick moves the cursor past its row, his NEXT pressure decision starts at flee — first retreat, then the rout, with no escalation state. A blocked flee (squad stands, under fire, point-blank, no base) defers the same way. Brave factions never flee (flee's palette rejects them and the walk falls through in the same tick). When a palette matches but every feasibility declines, AT leaves that NPC to vanilla.

### The advantage rules

A need says a situation exists; it does not say the maneuver pays. Each maneuver therefore carries rules — enemy-related (distance, the enemy's weapon, the actor's aim) and squad-related (last man, a standing line) — so that running it is an advantage for this NPC here, not a reflex. All of them live in the `can_maneuver` methods (the seize path), so they cost nothing on the monitor and each traces its pass or decline when debug is on (`can_counterflank` / `can_kite` / `can_snipe` / `can_retreat` / `can_flee` lines): counterflank refuses a wall between him and the actor, kite retreats its weapon's set distance and always clears its minimum, snipe refuses to plant inside the enemy's working range, under fire, or under the actor's crosshair, retreat refuses cover toward the shooter, flee refuses to rout under fire or while a squadmate stands. The reads are the `WEAPON_RANGES` table (both sides' weapon kinds are cached reads), the squad member count, the NPC's own danger memory, and the actor's facing — one raycast in counterflank feasibility, nothing per frame. Later maneuvers (suppress, assault, flank) get held to the same bar at design time.

### Flee hand-back

Flee holds no enemy suppression and no clock of its own; it is a generic maneuver like the other three. While it runs, the GOAP graft already blocks vanilla combat, so the NPC cannot turn and fight (the holster re-assert keeps the weapon down and the facing on the run path). It ends on the same conditions as any row — arrival at the base, the timeout cap, or a lost target — and hands back with all its state cleared.

On hand-back the engine decides, and the disengagement gates are these: against a MONSTER enemy, the engine's own `max_ignore_distance` (75m, `m_stalker.ltx:415`, applied in `CEnemyManager::useful` for the stalker-vs-monster clause only, enemy_manager.cpp:76-82); against another STALKER, the script-side 100m cutoff in whichever `xr_combat_ignore.script` won the MO2 slot (vanilla `:245`, `dist > 10000` squared, inside the engine's `useful_callback`); against the ACTOR there is NO unconditional distance rule (vanilla limits actor fights to 100m only at night or in rain, `xr_combat_ignore.script:229-231`) — a daytime flee from the player relies on the NPC having run holstered and blind, so the memory decays unrefreshed. A flee that reached its 100m+ base clears the first two gates outright. A flee that capped in place (boxed in, or the base too far to reach online) hands back next to the enemy and fights — and since the need (hurt, enemy in reach) still holds, he attempts escape again when the feasibility gates allow: the correct read of a cornered man, not a bug. There is deliberately no post-hand-back ignore window; the earlier `flee_enemy` / `flee_until` hold was removed (2026-07-10) because it existed only to make a failed escape look like a clean one.

### The maneuver pattern

Every maneuver sets its state ONCE when AT takes the NPC (the GOAP action's `initialize` — the single place AT writes the engine, setting the move and fire state), carries an `ends_on` (`arrival` for movers, `time` for holds) and a `max_ms` cap. The GOAP graft `execute()` stays empty (no per-frame code). The destination is set once and the engine walks the NPC there; the combat state is re-applied by the row's `update` (`doctrine.apply_state`) on the 200ms update_maneuver check. The fire DISCIPLINE lives inside `xcombat.set_combat` (2026-07-10; AT holds zero fire logic — a row declares only its INTENT: FIRE, SNIPE, READY, STOW): a FIRE/SNIPE intent follows vanilla `kill_enemy`'s order — an enemy SEEN right now is fired on with no further gate (no distance term — point-blank fires; `fire_make_sense`'s 2.5m bail is a smart-cover rule that must never gate a seen enemy), only the blind case consults `fire_make_sense` (its occlusion pick stops shooting the wall he ducked behind, its 10s automatic-weapon window sustains suppression at last-known), else the intent degrades to READY, weapon up, eyes on the enemy, until sight returns. The periodic work is a catalog property: each row declares its own `update` (or none), never a fire-keyed branch in the shell.

Flee is the against-the-grain maneuver (face away, weapon down); its update is a BLOCK — keep the weapon down — not a sight re-drive. The takeover block stops `kill_enemy` from aiming, but that alone does not hold: set once, the weapon comes back up and the NPC re-aims (the observed bug). So flee re-asserts the HOLSTERED `sprint` state (weapon strapped = physically cannot aim or fire) on the update_maneuver check, throttled — 200ms is enough; demonized re-applies it every frame (`demonized_stalker_aoe_panic.script:327`). A holstered weapon with no target faces the run path on its own; AT never steers the sight, it just keeps the gun down. Flee routes to a non-hostile friendly base or smart at least `flee_base_min_dist_m` (100m) away — a real rout to safety, not a near hop (`_can_flee` → `find_friendly_base` biased to the rear hemisphere; it declines when none qualifies). This is exactly the demonized / redone panic mechanism: block the same combat planners AT blocks (combat/danger/xr_danger/state_mgr+2/alife, `demonized_stalker_aoe_panic.script:426`), re-assert `sprint`, route far away. Flee is not the only per-period case: every current row's `update` is `apply_state` — a firing maneuver re-checks its shot, flee re-asserts the holster. The GOAP `execute()` stays empty for all of them; the periodic work runs on the update_maneuver check (the monitor pass), not the GOAP action.

The monitor watches the end on the end_maneuver check — `is_arrived` (over `path_completed`) for movers, elapsed time for holds, the row's `should_end` premise where one is declared (snipe) — and ends the maneuver, handing the NPC back. The monitor is `_run_monitor` (`at_combat.script`), one vanilla time event every 200ms walking `_npc_states` — the store `_install` creates on spawn and the destroy/unregister handlers drop, so the walk touches only NPCs AT tracks, resolved per pass via `level.object_by_id` (nil-guarded; a gone id vanishes with its state). Every timed operation is a registered check row in `CHECKS` with its own period — begin_maneuver (no maneuver open, 600ms — quantized to the pass, the config states the truth), update_maneuver and end_maneuver (a maneuver running, 200ms each, the few NPCs mid-maneuver, so hand-back is prompt). Per frame the whole monitor costs one timer compare inside vanilla's own `ProcessEventQueue` walk (`_g.script:364`, driven from the actor update at `bind_stalker_ext.script:26`); time events do not survive a save load, so `_start_monitor` arms it at `actor_on_first_update`, and a disabled feature stops the pass entirely (`_apply_enabled` ends every open maneuver, then `_stop_monitor`). Aborts come free from the action's preconditions failing (alive, armed, not wounded, has-enemy); a held NPC simply eats a rare grenade rather than AT re-implementing vanilla's reactions. A mover's end position is sticky for free; a held mode reverts on release.

### Commitment

The second system, separate from the maneuvers, the anti-shuffle veto, and the design for the WIP Effectiveness > Commitment page. The engine keystone is merged (the `npc_on_combat_action_switch` veto, demonized n023); AT's Lua subscriber is not yet built, so nothing below runs today. What it does when built: vanilla ties sustained fire to being in cover — `kill_enemy` requires `InCover = true` (`stalker_combat_planner.cpp:342-344`) — and the best-cover point churns, so NPCs break contact and shuffle toward sketchy cover even while they are winning the shooting. The shuffle intervention fixes this in place, with no takeover.

It runs on the engine's action-switch veto (`npc_on_combat_action_switch`, the demonized keystone n023): the callback fires before the combat planner swaps actions, and AT returns `allow = false` to keep the current action. So AT denies a switch when the NPC is doing something good — a player in front and already being fired on, a chosen action mid-run — and allows it only when an important event (a hit taken, a grenade, the enemy lost) warrants re-evaluation. The rule lives entirely in the callback: "never switch away while X holds," or "only switch under our conditions."

The veto only HOLDS the current action; it can never make a new one start, so it launches no maneuver. That is the whole distinction from the takeover: the maneuvers select and run a behavior, the shuffle intervention only keeps vanilla committed to one it already picked. It runs on every fighting NPC, seized or not, and composes with modpacks (it denies switches, it grafts nothing) — including GAMMA AI Rework, whose camper action lives in the same planner and is untouched by a veto that only refuses switches. This is what makes vanilla's own maneuvers commit and shrinks the takeover to the behaviors vanilla genuinely lacks.

### Squad coordination, decentralized

A later squad phase, not yet built: the base-of-fire and maneuver-element maneuvers it needs (suppress, assault, push, flank) are not in the catalog. The design: a squad fighting the player coordinates fire and movement, but with no central brain. Role eligibility is biased by `get_squad_ordinal`; a maneuver-element's viability requires that the enemy is being pinned, so one NPC flanking alone (a death wish) cannot fire. The reliable pin signal is an AT NPC committed to a base-of-fire maneuver; the coordination is emergent from each NPC's local read of its squad.

### The GOAP graft (the control point)

The graft adds one evaluator and one action per stalker and `world_property(EVAL_ID, false)` as a precondition on each entry of `xcombat.get_blocked_planners()`. While the per-NPC gate flag is true the vanilla combat/danger/alife chain is gated off and the grafted action is the only producer of the `EVAL_ID=false` the brain now requires, so it runs; clear the flag and vanilla resumes. The graft mechanism is encapsulated in `xcombat.install_takeover(npc, spec)` / `release_takeover(npc)`, where the spec is `{ gate, on_begin }` — the gate flag the evaluator polls and the one-time maneuver start the action's `initialize` calls; AT owns the spec, xcombat owns the GOAP classes.

### Layer arbitration

Vanilla orders its own schemes against each other with explicit cross-preconditions in `configure_actions`; AT's Combat layers need the same rule, stated before the Behaviors leaf gets its first action. The rule: **maneuvers outrank behaviors.** `xcombat.get_blocked_planners()` lists vanilla planner ids only, so a future Behaviors-leaf injected action would not be blocked while a maneuver holds the NPC — the solver could route through the behavior instead of the takeover action. Enforcement is one condition per layer: a Behaviors action's evaluator returns false while the takeover gate is up (`at_combat.get_maneuver(id)` non-nil). The rule binds now; the enforcement code is built with the first Behaviors graft.

A structural fact that falls out of the same block: Maneuvers and Commitment are mutually exclusive per NPC per moment by construction. The n023 action-switch veto (`npc_on_combat_action_switch`) fires only when the combat planner proposes swapping actions, and a blocked planner never switches — so the veto never fires for a seized NPC. Consequence: Commitment cannot police takeover quality; a takeover's fire discipline belongs at the maneuver's own decision points (`apply_state`), never at the veto seam.

### xcombat boundary

AT owns what to do; xcombat (xlibs) owns how to issue it to the engine. Every NPC command and read — weapon state, aim, movement, cover and clear-shot search, the line-of-fire and memory reads, arrival, the cover reservation, the enemy-state reads — goes through an xcombat primitive; AT makes no raw engine combat call. New primitives for this rebuild: `install_takeover`/`release_takeover`, `is_arrived`, `is_reloading`, `is_bleeding`, `is_moving`, and suppressive fire via `set_combat`; the rest is reuse.

One deliberate future exception would live outside xcombat: a sniper-reach extension that forces `is_enemy = true` so a planted sniper can engage past the engine's own enemy-distance gate. It is NOT built. Today the snipe maneuver only fires against an enemy the engine has already selected as `best_enemy`, so its reach is bounded by the engine's `is_enemy` range — real, but shorter than a sniper's effective range. If added, it would sit on the `on_enemy_eval` engine callback seam and AT would register and own it directly (like `npc_on_hit_callback`), not as an xcombat primitive: xcombat stays stateless by design — it holds no live-event callback or ownership table on its own behalf, so a stateful `set_enemy_eval` would break that contract.

### Identity and rejected alternatives

The identity the takeover is built for: recognizable committed maneuvers, composition under modpacks (it overrides zero combat files), and solving the shuffle (vanilla twitching between actions instead of committing). Everything above serves those three; a change that trades any of them away is out of scope.

The continuous `script_combat_type` scheme (GAMMA AI Rework, ReDone Combat AI) is the rejected alternative, and a 2026-07-05 read of both confirmed why: it owns an NPC's whole combat single-ownedly, and either does less than vanilla (GAMMA's thin camper sets one state) or reimplements it worse (ReDone's fat `get_combat_movement` and global-cvar aim). The intermittent takeover borrows an NPC for one maneuver where vanilla is weak and hands back, so vanilla's own aim, fire discipline, cover cycle, and squad coordination run the rest of the time - the maneuvers without deleting the strengths.

The considered-and-rejected alternative to the top-level planner block is rx-style injection inside the combat sub-planner (`cast_planner` on `action_combat_planner`, then `add_evaluator`/`add_action` there, per Rulix's `rx_combat.script:327-353`). It preserves what the top-level block loses - `CStalkerCombatPlanner::update`'s side effects, `react_on_grenades` / `react_on_member_death` at `stalker_combat_planner.cpp:102-105`, and the initialize/finalize mask and danger inertion. But it arbitrates against whatever a modpack grafts inside that same planner, so it surrenders exactly the robustness the takeover was chosen for. The top-level block wins for the GAMMA audience, at the cost of suppressing those `update` side effects for the seconds it holds - which is why a transaction stays narrow and brief.

---

## Accuracy

Rank-aware NPC dispersion in script. `at_accuracy.script` subscribes to the vanilla `npc_shot_dispersion` callback (declared in `axr_main.script:126`, dispatched from `_g.CAI_Stalker__GetWeaponAccuracy` at `_g.script:1213-1217`).

Why script and not cvars: the engine rank curve degenerates on Anomaly gamedata. `Rank()` clamps to `[0, 100]` at `ai_stalker.cpp:764`, but vanilla `<rank>` intervals run to 26999 (game_relations.ltx:8). All Anomaly NPCs end up at `rank_k = 1.0`, so `m_fRankDisperison` collapses to the constant `dispersion_experienced_k = 0.8`. Cvar tuning is a dead knob.

Math: `out = base * disp`. The engine already multiplied by `m_fRankDisperison` (= 0.8 for every Anomaly NPC after the rank clamp) before the callback fires, plus the per-state factor. We stack `disp` on top — `disp = 1.00` preserves the engine's vanilla cone, lower values tighten it.

Per-rank tiers (novice through legend): higher rank, tighter dispersion. The per-tier factors are MCM tunables.

Per-shot hot path: a rank-name lookup, then pure-Lua scaling of the dispersion the callback hands us.

---

## Disclosure

MCM page: Effectiveness > Disclosure (`at_disclosure.script`). Hooks `npc_on_hit_callback`. When a faction-enemy hits any squad member, the entire squad is force-disclosed to the shooter on hit #1. Extends the engine's audio-range squad disclosure to distant patrol members and suppressed-weapon victims.

### The flow

```
npc_on_hit_callback (a faction enemy hits a stalker)
  -> gate: amount > 0, shooter exists, not a self-hit, is_factions_enemies
  -> key = squad id, or the victim's own id when squadless
  -> survivor gate on: defer one frame, drop the disclosure if the hit killed
  -> first hit for (key, shooter): xcombat.disclose_enemy per online member
         enable_memory_object + register_in_combat
         (the engine's own memory propagation carries it from there)
     repeat hit: refresh the timestamp, nothing else
  -> decay tick (5s): prune shooters idle past the retention window
spawn: a new member inherits the squad's live disclosures;
       a returning offline shooter is re-disclosed to every tracking squad
```

### What the engine does natively on hit

1. Hit registered → `CHitMemoryManager::add` creates a hit_memory entry on the victim (`hit_memory_manager.cpp:95-163`).
2. Friendly-fire filter: returns early if `tfGetRelationType(who) == eRelationTypeFriend` (`hit_memory_manager.cpp:127`).
3. Victim plays hurt sound → `eStalkerSoundCry` / `eStalkerSoundAlarm`.
4. Audio-range squadmates hear the sound → their `sound_memory_manager` promotes the source into their hit_memory (`sound_memory_manager.cpp:188`).
5. `enemy_manager` picks the shooter as a selected enemy → combat planner activates → `register_in_combat()` flips the member's squad_mask bit (`stalker_combat_planner.cpp:172`).
6. `agent_memory_manager` propagates memory entries across all combat-active squadmates each tick (`agent_memory_manager.cpp:33-42`), gated by combat_mask intersection.

**The engine's native squad disclosure is bounded by audible reach.** Distant patrol squadmates outside sound range, or squadmates against a suppressed weapon, never enter combat_mask and never receive the propagated memory.

### What we add on top

1. Sanity guards: `amount > 0`, `who` exists, not self-hit.
2. **Faction-relation gate** via `game_relations.is_factions_enemies(npc_community, shooter_community)`. Same-community hits rejected. Mirrors the engine's friendly-fire skip at our hook entry.
3. Resolve the disclosure key: the squad id via `get_object_squad(npc)`, or the victim's own id when squadless — a solo victim discloses to himself (squad and NPC server ids share one id space, so the map never collides).
4. Write/refresh timestamp: `_disclosed[key][shooter_id] = xtime.game_sec()`. Every hit refreshes.
5. If the entry existed before the write (idempotency hit): return. This audience already engaged this shooter in this fight.
6. Otherwise (first hit, or first hit since decay): `xcombat.disclose_enemy` per online squadmate (or the solo victim). Relation-clean — no goodwill or community write; the enemy_manager engages the injected object because faction enmity already holds (`is_relation_enemy` via the faction-dominated summed attitude). Two engine APIs per receiver:

   - **`enable_memory_object(who, true)`**: toggles `m_enabled` on existing visual/sound/hit memory entries (`memory_manager.cpp:151-156`). Its teeth: re-enables memory a combat-ignore suppressed, so an "ignored" shooter becomes engageable again. Receiver must be `CCustomMonster` (`script_game_object2.cpp:262`).
   - **`register_in_combat()`**: sets the member's squad_mask bit in `CAgentMemberManager::m_combat_mask` (`agent_member_manager.cpp:114-132`) — the unlock for engine-native squad memory propagation across every member including distant patrols. Requires `CAI_Stalker` receiver; safe because `npc_on_hit_callback` is dispatched only by `motivator_binder` (stalker squads). Routed through the alive-guarded `xcombat.register_in_combat`.

   What disclosure deliberately does NOT do: change enemy RANKING. The selection pays "currently seen" ~1000 points against ~5-100 for "hit me" (`enemy_manager.cpp:110-175`), and `visible_now` is live vision state no script write reaches — so a disclosed-but-unseen shooter still loses to a seen enemy. The engine's own `make_object_visible_somewhen` (`memory_manager.cpp:345`) closes exactly that gap and is not script-bound: the n027 demonized PR binds it, and disclosure adopts it on merge.

### Decay and re-engagement

A periodic decay tick walks every `_disclosed[key][shooter_id]` entry and prunes any older than the retention window (MCM-tunable). Pruning clears only the idempotency entry; no relation state was written, so nothing else needs unwinding.

After decay, the next hit from that shooter against that squad triggers a fresh `_disclose` call. Distant patrol squadmates get re-pinned into combat_mask for the new engagement.

### Spawn handler (mid-engagement replenishment + offline-shooter return)

`npc_on_net_spawn` fires for every stalker spawn (dispatched by `motivator_binder`). Two paths run for each spawned NPC:

1. **Inherit from squad** (case 1): if the spawning NPC's squad has active disclosures, apply `_disclose_to_member` for each disclosed shooter that resolves online. The replacement inherits the squad's combat state without waiting for the next hit.
2. **Re-disclose on shooter return** (case 2): walk every tracked squad's disclosed map. If the spawning NPC's id is present (the NPC is a previously-offline shooter coming back online), replay `_disclose_to_member` for every online squadmate of those squads. Covers members who joined while the shooter was offline.

Both paths short-circuit quickly when no entries match. Most spawn events trigger zero work.

**Mutant-shooter return is not re-disclosed by this handler.** Mutants dispatch `monster_on_net_spawn` (`bind_monster.script:298`) which AT does not subscribe to. Sustained engagement is not lost: combat_mask bits set at original hit time persist on squad members, and mutant-vs-stalker faction enmity drives target acquisition independently. The only gap is a stalker member who joins the tracking squad while the mutant shooter is offline: that member never gets disclosed to the returning mutant.

### Net behavior

- Engine handles audio-range squadmates on hit #1 (free, automatic).
- Our hook handles distant patrol squadmates on hit #1 by forcing them into combat_mask, letting the engine's own propagation pipe carry the memory; a squadless victim gets the same pair himself.
- No relation is written: faction enmity already makes the disclosed shooter engageable, and the memory enable defeats a combat-ignore suppression.
- Sustained engagement: subsequent hits refresh the timestamp and return early via idempotency.
- After the retention window of no hits from a given shooter, the squad's pin on that shooter expires; the next hit re-fires the full pipeline.
- Mid-fight replenishment: new squad members inherit existing disclosures on spawn.
- Offline-shooter return: when a previously-offline tracked shooter comes back online, the spawn handler replays disclosure to every member of the squads tracking them. Members who joined while the shooter was offline get pinned at this moment.

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
    eval_danger -> npc_on_eval_danger (third-party veto seam) -> best_danger type
        -> inertion + ignore tables (the winning ai_tweaks\xr_danger.ltx rows)
        -> danger_flag
    at_action_danger:execute -> per-type response
        (scripted / grenade / corpse / attacked / attack_sound alert)
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
5. The vanilla hit callback passed an undefined `who_id` and wrote a nil shooter id into `script_danger` on every hit. AT does not register it, and the patched `set_script_danger` rejects a nil `who_id` (`at_danger.script:99-102`), so the winning file's own copy of that callback cannot corrupt the entry either.
6. Animstate reset missing on danger-state transitions: vanilla `state_mgr.set_state` calls did not invoke `sm.animstate:set_state(nil, true) + set_control()`, leaving stale lower-body animation visible across the transition. AT calls the reset at every state-change site (`at_danger.script:417-419, 487-489, 524-526, 741-773`).
7. `at_action_danger:finalize` wiped the whole shared `db.used_level_vertex_ids` reservation map in vanilla, clobbering every other system's cover claims (the Combat takeover's, vanilla's). AT releases only the vertices this NPC owns (`at_danger.script:837-845`).
8. The corpse action crashed on the teardown race: a corpse despawning between the evaluator pass and the action execute left a nil danger object (`bdo:id()`) or a nil storage entry in the stage-6 look_position. Both paths are guarded.
9. The corpse force-hostile loop reused the acting NPC variable for squad members, so later corpse stages drove the last member (or nil) instead of the acting NPC. The loop uses its own local.
10. Corpse stage 6 sent the NPC to the just-cleared `st.lvid` instead of the cover vertex `try_go_cover` found, so the found cover was never used.
11. Performance: vanilla re-parsed the danger inertion and ignore-distance condlists on every evaluation, per NPC per plan solve. The strings are fixed after the DLTX merge, so they parse once into a memo (`_parse_cached`) and every later evaluation is a table lookup; only the condition evaluation (weather, actor state) still runs per call.
12. `script_danger` entries are dropped on entity unregister. Vanilla kept a dead perceiver's entry until expiry, so an id recycled inside the inertion window inherited a scripted danger the new NPC never perceived.

### Extension callback

`eval_danger` fires `npc_on_eval_danger` (`at_danger.script:265-267`) with `flags.ret_value = true` before it evaluates; a subscriber that sets `flags.ret_value = false` suppresses danger for that NPC on that tick. AT preserves this vanilla seam — `axr_main.script:125` declares it and vanilla `xr_danger.script:268` fires it from the same evaluator — so third-party subscribers keep working when AT's patch replaces `eval_danger`.

### Improvements (MCM Effectiveness > Danger, default on)

- `danger_hit_bypass`: a direct hit is danger at any distance - the branch returns true past the relation, combat-ignore and ignore-distance gates (being hit is proof of range); only the scripted combat-ignore override still suppresses it. The division of labor around a hit: at_disclosure owns "the squad learns the shooter" (faction enemies, combat memory, rangeless by construction); hit_bypass owns "the victim ducks even when he cannot fight back" (a neutral or combat-ignored shooter, where combat cannot engage); the future Range page owns answering fire at weapon reach.
- `danger_attack_sound`: script_action_danger_alert dispatch for the `attack_sound` danger type, admitted by `at_evaluator_check_danger` while the toggle is on (2026-07-10: the dispatch alone was unreachable - the evaluator admitted only attacked/corpse/grenade, so the improvement never fired). Reaction distance is the winning config's `attack_sound` row. Vanilla had no script handler for this danger type. The handler's actor-aim gate is dormant: the relation gate upstream blocks non-enemy actor sources; opening that path is a future feature.
- `danger_actor_tables`: read separate inertion and ignore tables from `[danger_inertion_actor]` and `[danger_object_actor]` when the danger source is the actor - meaningful where the installed config differentiates them (GAMMA AI Rework does; vanilla ships identical copies, so the toggle is a no-op there).

### Paired LTX

`configs/ai_tweaks/mod_xr_danger_at.ltx` is delete-lines only (2026-07-10): `![section]` override-merge on all four danger sections, each dropping only the dead `hit`/`sound`/`visual` keys (PerceiveType names colliding with EDangerType values 0/1/2; nothing reads them after the bd_types fix). AT ships NO danger values - every detection distance and inertion comes from whichever `xr_danger.ltx` won the MO2 slot plus later DLTX overlays: GAMMA plays AI Rework's tuning unchanged, vanilla plays vanilla's true-name rows (the rows the collision always hid), Stealth Overhaul plays xcvb's. The 1.1.0 value rows were a verbatim GAMMA AI Rework copy; on non-GAMMA setups they cut effective reaction ranges 3-50x (the 2026-07-10 user report), so they were removed and the setup owns the tuning. `!![section]` is NOT full-section replacement in this engine's DLTX - it deletes the section outright and discards the keys under it (`Xr_ini.cpp:721-739`, the 2026-07-09 nil-src condlist flood).

Later-alphabet DLTX overlays on the same sections still take precedence, load-order-wise, as with any DLTX stack (in GAMMA, Useful Idiots' `mod_xr_danger_z_idiots.ltx` owns `[danger_inertion]` this way).

### Composition

`at_danger.script` carries the `-- @override` marker only to exempt its vanilla-derived code from AT-native style rules and the stub load test; it is not a VFS whole-file override. Because AT patches the winning file at runtime instead of replacing it, the Danger system layers onto GAMMA AI Rework or REDONE rather than excluding them: AT's evaluator and action logic wins while the rival's file stays loaded and its own perception callbacks keep running. The one thing the patch cannot do that a file override could is suppress the winner's danger callbacks, which surfaces only as REDONE's fixed hit callback adding a second, harmless trigger of the same react-to-a-hit intent. The MCM Effectiveness > Danger page describes the always-on fixes and the three improvement toggles.

---

## Crossfire

Friendly-fire damage gate in `at_crossfire.script`. `npc_on_before_hit` scales `shit.power` by the MCM factor unless the shooter and victim are actually enemies (`attacker:relation(npc) == game_object.enemy` -> full damage). Keyed on per-NPC relation, not community: same-faction NPCs are neutral at worst and never enemy (a loner never enemy to a loner), so they stay protected, while a soured cross-faction pair (a loner vs a hostile Clear Sky) still damages each other. `relation()` is faction-paramount (the community-to-community base dominates personal goodwill). Stalker-vs-stalker only (both `IsStalker`), the actor as shooter is excluded, O(1) with no throttle (a damage block must catch every hit). MCM page: Effectiveness > Crossfire (`crossfire_enabled` + `crossfire_factor`).

---

## Healing

Per-NPC self-healing. Vanilla `xr_eat_medkit.script` has a working stage machine, but vanilla `ai_tweaks/xr_eat_medkit.ltx [plugin]` lacks the `medkits=` / `bandages=` keys so `parse_list` returns `{}` and the consumption loop iterates zero times.

### Data layer fix

`mod_xr_eat_medkit_at.ltx` is a DLTX overlay on `![plugin]` adding `medkits = medkit, medkit_army, medkit_scientic, medkit_ai1, medkit_ai2, medkit_ai3` and `bandages = bandage`. Boot-time, no runtime toggle.

### Runtime tuning

`at_healing.script` installs two hooks on `on_game_start`:

| Hook | Mechanism | What it changes |
|---|---|---|
| Heal rate multiplier | `xr_eat_medkit.heal_hp = _patched_heal_hp` | Per-tick `change_health` scaled by the MCM multiplier, read each tick; rescheduling via the `xr_eat_medkit.heal_hp` lookup keeps every tick on the patched function |
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
| Limping | the limp monitor pass (`_run_monitor`, a vanilla time event every 200ms over the spawn-filled roster): the drop detectors run on the pass itself while the pose is worn (wounded/dead; commanded gait changed since add; stand-variant drifted off its add anchor; walk/run-variant stopped displacing over a 1s sample) - every drop calls `clear_animations()`; a 1s eligibility check on a per-NPC stamp (`health < threshold`, no `best_enemy`, `mental_state() == anim.free`, `body_state() == move.standing`, not zombied, not in smart_cover), re-armed every 5s | a per-slot `dmg_norm` hurt pose (clutch-the-torso) chosen from `active_slot()` + `movement_type()`. A queued script animation suspends the engine's whole animation selection (legs included, stalker_animation_manager_update.cpp:232), so the overlay must die the moment its gait stops matching - that is what the drop detectors do |
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

## What the engine does and what we feed it

The architecture principle is to feed engine memory and state, not fight it. Per-system summary:

| System | Engine state we write | Engine APIs called |
|---|---|---|
| Disclosure | memory entry m_enabled, agent_member_manager m_combat_mask | `enable_memory_object`, `register_in_combat` |
| Healing | NPC health field, bleeding field, `healing_charge` se_var | `change_health`, direct `bleeding =` write, `se_save_var` |
| Accuracy | Per-shot dispersion radius via callback return | (subscribes to `npc_shot_dispersion`) |
| Combat | NPC GOAP action (at_combat_action), Pattern B preconditions on action_combat_planner/action_danger_planner/xr_danger.actid/state_mgr+2/alife, set_dest_level_vertex_id, state_mgr.set_state, set_body_state, set_movement_type, set_sight, `m_sniper_fire_mode` flag | GOAP `add_evaluator`/`add_action`/`add_precondition` (custom evaid/actid), `npc:best_cover`, `level.vertex_in_direction`, `npc:sniper_fire_mode`, `db.used_level_vertex_ids` reservation |
| Danger | Danger scheme evaluators/action installed onto the winning `xr_danger` binder, `script_danger` per-id table for sound-source dispatch | Patches `xr_danger.{setup_generic_scheme,add_to_binder,configure_actions,reset_generic_scheme,get_danger_time,set_script_danger,has_danger}`; relies on the winner's `npc_on_hear_callback` / `npc_on_death_callback` feeders |
| Jamming | Module-level function table on `xr_weapon_jam` | Lua function assignment (`xr_weapon_jam.GetConditionMisfireProbability = ...`) read by engine functor lookup at `Weapon.cpp:1781` |
| Ammo | CWeapon `m_ammoType` field via `wpn:set_ammo_type(idx)` (re-keys `m_DefaultCartridge` for magic refill ballistics); per-engagement box-delete decay (`alife_release` one AP box on a rank/rpm-weighted roll); reverts to `m_ammoType = 0` when the section is empty | `npc:active_item`, `wpn:set_ammo_type`, `wpn:get_ammo_type`, `wpn:get_ammo_count_for_type`, `npc:best_enemy`, `npc:character_rank`, `npc:iterate_inventory`, `alife_release` |

The engine then runs its own combat detection (property_enemy, m_combat_mask, agent_memory propagation) on the state we wrote. No system reimplements engine behavior; each one nudges engine state to produce the desired outcome.

---

## See also

- Task queue: `stalker-dev/doc/todo/todo-alifetactics-next.md`
- Takeover build plan: `stalker-dev/doc/todo/todo-combat-takeover-v2.md`
- Adversarial review: `stalker-dev/doc/todo/todo-alifetactics-fable-review.md`
- Engine PR queue: `stalker-dev/doc/todo/todo-demonized-exes.md`
- xlibs architecture: `stalker-mods/xlibs/doc/architecture.md`
- AlifePlus architecture: `stalker-mods/AlifePlus/doc/architecture.md`
- AlifeGuard architecture: `stalker-mods/AlifeGuard/doc/architecture.md`
- AlifeBalance architecture: `stalker-mods/AlifeBalance/doc/architecture.md`
