# AlifeTactics Architecture

Combat AI replacement for STALKER Anomaly. Built around a shared per-squad memory substrate that current and future combat behaviors all consume. Per-NPC layers (precision, health, damage, fx) sit alongside the squad-scope behaviors.

Built on xlibs (xsquad, xttltable, xtime, xprofiler, xlog, xmcm).

Part of a four-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** tunes existing rates and counts, **AlifeGuard** keeps alife state clean, **AlifeTactics** controls how NPCs fight in combat (this mod).

---

## Status

Version 1.0.0. The substrate, state machine, hit disclosure, MCM, logging, and a console-test harness are implemented; the remaining behaviors (tactical flee, memory persistence, combat scheme selection, global combat tuning, stance/weapon bias, accuracy hook) are designed but not built.

| Module | State |
|---|---|
| `_at_deps.script` | done |
| `at_core_mcm.script` | done |
| `at_squad_memory.script` (substrate) | done |
| `at_state_machine.script` (state machine) | done |
| `at_ext_hitresponse.script` (hit disclosure) | done |
| `at_ext_health.script` (NPC medkit consumption + healing charge controls) | done |
| `configs/ai_tweaks/mod_xr_eat_medkit_at.ltx` (DLTX overlay restoring vanilla consumption path) | done |
| `at_test.script` (console test harness) | done |
| `at_ext_flee.script` (tactical flee, t13) | backlog |
| `at_ext_persistence.script` (memory persistence, t14) | backlog |
| `at_ext_scheme.script` (combat scheme select, t03) | backlog |
| `at_ext_cvars.script` (global combat cvars, t16) | backlog |
| `at_ext_stance.script` (stance/weapon bias, t18) | backlog |
| `at_ext_accuracy.script` (per-NPC accuracy hook, t20) | backlog |

Validate grade S+. Groomed task entries in `stalker-dev/doc/todo/todo-alifetactics-next.md`; brainstorm pool in `todo-alifetactics-backlog.md`.

---

## Three layers

### Layer A: Substrate

One Lua table per squad, keyed by `squad_id`. Holds all combat-relevant collective state for that squad. Every squad-aware behavior reads and writes here.

Schema (as implemented in `at_squad_memory.script`):

| Field | Type | Description |
|---|---|---|
| `per_shooter[shooter_id]` | record | `{ count, last_hit_time, total_damage }` — hit accumulation against this squad |
| `disclosed_shooters[shooter_id]` | bool | Set when this shooter has been disclosed to the squad. Session-persistent (idempotency guard for hit disclosure). |
| `state.current_state` | string | `IDLE` / `ALERTED` / `ENGAGED` / `FLEEING`. Owned by the state machine. |
| `state.state_entered_time` | number | xtime.game_sec when the current state was entered. |
| `state.last_event_time` | number | xtime.game_sec when the latest event was recorded. Drives de-escalation. |
| `state.last_disclosure_time` | number | Set by hit disclosure when a new shooter is disclosed. |
| `members[member_id]` | record | `{ last_health_snapshot, last_hit_time }` — reserved for tactical flee power evaluation. |

Lifecycle:

- Lazy init on first `at_squad_memory.get(squad_id)` (no upfront cost per squad).
- `server_entity_on_unregister` clears the record when the squad despawns.
- 5s decay tick prunes `per_shooter` entries older than `substrate_retention_sec` (MCM, default 60s).
- No save/load persistence; combat state is transient and rebuilds from runtime events.

Memory cost: ~60 bytes per shooter record, ~5 shooters per squad in active combat, ~500 squads online cap. Worst case 150KB.

Anti-pattern: mirror per-NPC state to every squadmate's `db.storage` slot via inform-squad-style propagation. That mirror-state shape has sync issues and duplicates memory across N members. The substrate inverts ownership: state lives at the squad level, members read shared.

### Layer B: Substrate consumers

Behaviors that hook engine callbacks and read or write the substrate. No behavior owns its own scratch state.

| Behavior | State | Reads | Writes | Trigger |
|---|---|---|---|---|
| HIT_DISCLOSURE | done | disclosed_shooters | per_shooter, disclosed_shooters, state | `npc_on_hit_callback` |
| SQUAD_COMBAT_STATE | done | state (counter), members | state.current_state, state.state_entered_time, state.last_event_time | called from HIT_DISCLOSURE on hit and disclosure events |
| MEMORY_PERSISTENCE | backlog (t14) | state | per_shooter retention multiplier | substrate decay tick |
| TACTICAL_FLEE | backlog (t13) | per_shooter, members, state | state.current_state = FLEEING | throttled 2s tick |
| COMBAT_SCHEME_SELECT | backlog (t03) | state, per_shooter | none | condlist evaluation on danger ticks |

### Layer C: Independent layers (no substrate involvement)

Behaviors operating at game-global scope or one-shot per NPC.

| Behavior | State | Mechanism | Trigger |
|---|---|---|---|
| GLOBAL_COMBAT_TUNING (t16) | backlog | `exec_console_cmd` for PR #523 ai_* cvars | `actor_on_first_update`, MCM change |
| STANCE_WEAPON_BIAS (t18) | backlog | `npc_on_choose_weapon` + `npc_on_combat_set_body_state` overrides | per-NPC callback |
| ACCURACY_HOOK (t20) | backlog | `_g.CAI_Stalker__GetWeaponAccuracy` per-shot Lua hook | per-shot during combat |

---

## Hit disclosure

The headline behavior. When a faction-enemy hits any squad member, the entire squad is force-disclosed to the shooter on hit #1. No rank-tiered threshold; first hit triggers full-squad disclosure.

### What the engine does natively on hit

Before we hook anything, the engine already does most of the work:

1. Hit registered -> `CHitMemoryManager::add` creates a hit_memory entry on the victim (`hit_memory_manager.cpp:95-163`).
2. Friendly-fire filter: returns early if `tfGetRelationType(who) == eRelationTypeFriend` (`hit_memory_manager.cpp:127`).
3. Victim plays hurt sound -> `eStalkerSoundCry` / `eStalkerSoundAlarm`.
4. Audio-range squadmates hear the sound -> their `sound_memory_manager` promotes the source into their hit_memory (`sound_memory_manager.cpp:188`).
5. `enemy_manager` picks the shooter as a selected enemy -> combat planner activates -> `register_in_combat()` flips the member's squad_mask bit (`stalker_combat_planner.cpp:172`).
6. `agent_memory_manager` propagates memory entries across all combat-active squadmates each tick (`agent_memory_manager.cpp:33-42`), gated by combat_mask intersection.

**The engine's native squad disclosure is bounded by audible reach.** Distant patrol squadmates outside sound range, or squadmates against a suppressed weapon, never enter combat_mask and never receive the propagated memory.

### What we add on top

`at_ext_hitresponse.script` hooks `npc_on_hit_callback(npc, amount, dir, who, bone)`:

1. Sanity guards: `amount > 0`, `who` exists, not self-hit.
2. **Faction-relation gate** via `game_relations.is_factions_enemies(npc_community, shooter_community)`. Same-community hits are rejected as friendly fire; neutral/friendly factions never reach the disclosure path. Mirrors the engine's own `tfGetRelationType` friendly-fire skip, applied at our hook entry so substrate is never polluted by accidental hits.
3. Resolve squad via `get_object_squad(npc)`; skip solo NPCs.
4. Substrate writes: increment `record.per_shooter[shooter_id].count`, refresh `last_hit_time`, accumulate `total_damage`. Call `at_state_machine.record_event(squad.id, EVENT.HIT)`.
5. Idempotency check: if `record.disclosed_shooters[shooter_id]` is already set, return.
6. Otherwise, set the flag and call `_disclose(squad, who)` — three engine APIs per online squadmate:

   - `force_set_goodwill(-2000, who)` writes RELATION_REGISTRY personal goodwill minus community baselines, so when `GetAttitude` sums personal + reputation + rank + community + community_to_community the community terms cancel and the final attitude is `-2000 + reputation + rank` — well below enemy threshold (`relation_registry.cpp:161-179`, `relation_registry_inline.h:69-93`). `CAI_Stalker::tfGetRelationType` (`ai_stalker_misc.cpp:92-105`) routes through RELATION_REGISTRY for stalkers, so this drives every downstream `is_relation_enemy` check. **Gated on `IsStalker(who)`**: `RELATION_REGISTRY::ForceSetGoodwill` (`relation_registry.cpp:165-172`) smart_casts both ids to `CSE_ALifeTraderAbstract` and bails with the "cannot convert obj" engine error when either fails; mutants are `CSE_ALifeMonsterAbstract`. For mutant / helicopter / anomaly shooters the goodwill write is skipped — they have no faction relation to write anyway. Substrate updates and the other two engine calls still run.
   - `enable_memory_object(who, true)` toggles `m_enabled` on existing visual/sound/hit memory entries (`memory_manager.cpp:151-156`). No-op when the squadmate has no prior entry; cheap insurance otherwise. Runs for any `who` (mutants included) — the engine call accepts any game_object target.
   - `register_in_combat()` sets the member's squad_mask bit in `CAgentMemberManager::m_combat_mask` (`agent_member_manager.cpp:114-132`). This is the **unlock for engine-native squad memory propagation**: with the whole squad's bits set, the next `agent_memory_manager` tick OR's the full combat_mask into the victim's hit-memory entry's `m_squad_mask`, propagating the memory of the shooter across every member including distant patrols. Runs for any shooter — no `who` argument, just flips the calling stalker's bit.

7. `at_state_machine.record_event(squad.id, EVENT.DISCLOSURE)` — short-circuits the state machine to ENGAGED.

### Net behavior

- Engine handles audio-range squadmates on hit #1 (free, automatic).
- Our hook handles distant patrol squadmates on hit #1 by forcing them into combat_mask, letting the engine's own propagation pipe carry the memory to them.
- Hostility for the shooter is pinned at -2000 personal goodwill on every squadmate — survives community-relation drift, lasts the session.
- Subsequent hits from the same shooter against the same squad no-op via `disclosed_shooters`.

### Cross-squad disclosure (not implemented)

The engine's sound channel already alerts NEARBY non-squad NPCs and squads via `sound_memory_manager`. Suppressed kills and out-of-range distant patrols miss this. AlifeTactics does not extend cross-squad alerting; a queue + periodic-tick design has been considered but defers to the engine sound channel for 1.0.0.

### What we deliberately do NOT do

- `set_relation(enemy, who)` — equivalent to `SetGoodwill(goodwill_enemy_config, who)` (`relation_registry_inline.h:25-44`). Strictly weaker and redundant with `force_set_goodwill`.
- `set_enemy(who)` — `CAI_Bloodsucker`-specific; logs an error for stalkers (`script_game_object_use2.cpp:231-243`).
- Scripted NPC movement — feeds engine memory + combat-mask, lets the engine handle navigation and target selection. The architecture principle is to feed the engine information, not fight it.

---

## Combat state machine

State definitions:

- **IDLE** — no recent combat events. Default on squad spawn.
- **ALERTED** — at least one event recorded. Weapons up, scanning.
- **ENGAGED** — `events_to_engage` (default 3) within `event_window_sec` (default 60s), OR first disclosure event. Active fire exchange.
- **FLEEING** — set by tactical flee when retreating. Priority state.

Transitions (as implemented in `at_state_machine.script`):

```
IDLE     -> ALERTED   on first event recorded
ALERTED  -> ENGAGED   when counter reaches events_to_engage within event_window_sec, OR on disclosure event
ENGAGED  -> ALERTED   after engaged_to_alerted_sec of no events (default 60s)
ALERTED  -> IDLE      after alerted_to_idle_sec of no events (default 120s)
any      -> FLEEING   via explicit set_state from t13 flee
FLEEING  -> ALERTED   via explicit set_state from t13 flee cancel
```

Event counting: `xttltable.create_ttl_counter({ ttl = event_window_sec, clock = xtime.game_sec })`. One counter per squad, lazy-init in `_counters[squad_id]`, cleared on server_entity_on_unregister.

De-escalation: 5s tick walks the substrate via `at_squad_memory.iterate`, demotes ENGAGED→ALERTED→IDLE on `last_event_time` timeouts. IDLE and FLEEING are skipped by the tick (FLEEING is owned by t13).

Combat state is per-squad scope. Per-NPC is too granular (squadmates share situational awareness). Per-smart is too coarse (a roaming squad encountering enemies far from any smart still needs combat state).

Independent of AlifePlus smart-state: AlifePlus tracks smart-scope combat; this is squad-scope. Both can be true simultaneously. Consumers needing combined semantics can union them later.

---

## Tactical flee (backlog, t13)

Squad-level power evaluation every 2 seconds. Reads enemy power from substrate `per_shooter` (weighted shooter values) and friend power from `members` (rank * community * health survivors). If `friend_power * threshold_mult * rank_cowardice < enemy_power`, retreat starts.

Mechanism (smart-terrain-aware flee):

- Smart-terrain destination: nearest friendly smart further from enemy than current squad position
- Rear-guard pattern: nearest member to destination stays in cover for configurable seconds
- Smoke grenade chance per rank thrown at farthest member position
- Flee canceled if power swings favorable, or hit-count exceeds threshold

AlifeTactics overrides (planned):

- Monolith and zombied factions never flee
- Squads at AlifePlus ENGAGED smarts never flee (base defenders stay)
- Per-rank cowardice multiplier: novice flees at 0.8x ratio, master at 1.5x

Forced-movement caveat: this is scripted NPC movement, generally fragile. Works in this design because the destination is a structural game element (a smart terrain) not an arbitrary vertex, the update is throttled to 2s, and the AOE-panic API has its own state machine that re-acquires cleanly.

---

## Memory persistence (backlog, t14)

Two-layer extension of danger memory windows so NPCs stay alert longer during sustained engagement.

**Substrate layer (Lua):** `per_shooter` retention scales with combat state. Default 60s. When `state.current_state` is ENGAGED or ALERTED, retention extends to 5 minutes. The substrate's decay tick reads its own state field each pass and applies the multiplier; no callback chain needed.

**Engine layer (LTX):** Provide `gamedata/configs/ai_tweaks/xr_danger.ltx` with state-keyed condlists on `danger_inertion`. Condition functions in `at_cond.script` resolve npc squad and read substrate state via O(1) lookup; evaluated by vanilla `pick_section_from_condlist` only on danger ticks.

Effect: snipe a guard, retreat, wait five minutes, return. Guard still alert (weapons drawn, in cover) instead of patrol idle.

---

## Combat scheme selection (backlog, t03)

Provides a `default_custom_data.ltx` `[combat] combat_type =` override with condlist. Conditions read substrate state and per_shooter, plus rank and distance to enemy:

```
combat_type = {=at_engaged_squad} camper,
              {=at_recent_hit} cover,
              {=at_close_range} default,
              {=at_distant} camper,
              {=at_coward} cover,
              default
```

Predicates in `at_cond.script`:

- `at_recent_hit(npc)` — substrate's per_shooter contains an entry within 15s
- `at_engaged_squad(npc)` — substrate.state.current_state == ENGAGED
- `at_close_range(npc)` — distance to `best_enemy()` < 25m
- `at_distant(npc)` — distance > 50m
- `at_coward(npc)` — rank tier novice/trainee or coward flag set

Predicates use `best_enemy()` so NPC-vs-NPC scheme selection works without changes.

---

## NPC health

Per-NPC self-healing. Vanilla `xr_eat_medkit.script` contains a working stage machine for "find a medkit in inventory, advance to consume_medkit, schedule heal_hp time event for 13 ticks of `change_health(0.05)`" (heal_hp recurses while `left - 1 > 1`, so left=15 produces 13 active ticks before the left=2 call exits). The bug is the data layer: `ai_tweaks/xr_eat_medkit.ltx [plugin]` is provided without `medkits=` / `bandages=` fields, so `parse_list` returns `{}` and the for-loop at `xr_eat_medkit.script:124` never iterates. Only the once-per-life `healing_charge` fallback fires for ~50% of stalkers.

### Data layer fix (t25)

`AlifeTactics/gamedata/configs/ai_tweaks/mod_xr_eat_medkit_at.ltx` is a DLTX overlay on `![plugin]` adding `medkits = medkit, medkit_army, medkit_scientic, medkit_ai1, medkit_ai2, medkit_ai3` and `bandages = bandage`. After the engine's base+mod merge, `parse_list` returns the actual list. No script change to vanilla. AlifePlus restocks these item sections via `ap_trade_policy.ltx` (medkit category) so NPCs carry the items they now actually use.

### Runtime tuning (t27)

`at_ext_health.script` installs two independent hooks in `on_game_start`:

| Hook | Mechanism | What it changes |
|---|---|---|
| Heal rate multiplier | Direct assignment `xr_eat_medkit.heal_hp = _patched_heal_hp` | Per-tick `change_health(0.05 * mult)` reads MCM each tick; reschedule via `xr_eat_medkit.heal_hp` lookup propagates the patch through all 13 ticks |
| Bandage tick logging | Direct assignment `xr_eat_medkit.heal_bleed = _patched_heal_bleed` | Same logic as vanilla (`npc.bleeding = 0.07` per tick for 13 ticks), wrapped to emit `[HEAL] bleed_tick` / `bleed_complete` log lines + xprofiler timing so bandage consumption is observable end-to-end. No behavior change. |
| Per-rank healing-charge | `RegisterScriptCallback("npc_on_net_spawn", _on_net_spawn)` | Reads `ranks.get_obj_rank_name(npc)` and folds the 8 vanilla rank names (novice / trainee / experienced / professional / veteran / expert / master / legend) into the 4 MCM tiers (novice / experienced / veteran / master). Rolls the MCM-configured chance, overrides vanilla's flat 50% roll. Per-NPC `at_charge_processed` se_var prevents re-roll across save/load and offline/online transitions. |

Why direct assignment on `heal_hp` and not `xevent.hook`: we substitute the body entirely rather than wrap. Vanilla's recursive scheduling does name lookup on `heal_hp` at each call, so reassigning the module-table entry propagates through the recursion. No chain stacking needed.

Why `npc_on_net_spawn` and not `server_entity_on_register`: `character_rank()` requires the game_object, which doesn't exist at server-entity-register time. `npc_on_net_spawn` fires once the game_object comes online, which is the earliest point we can read rank reliably.

### MCM surface

Under `Stalker Healing` tab — three sections.

`Medkit Restoration` (info-only): describes the DLTX overlay restoring vanilla item lists. Boot-time data layer, no runtime toggle.

`Heal Rate`:

| Key | Range | Default | Effect |
|---|---|---|---|
| `heal_rate_multiplier` | 0.5 - 3.0 (step 0.1) | 1.0 | Multiplies per-tick `change_health` |

`Healing Charge`:

| Key | Range | Default | Effect |
|---|---|---|---|
| `charge_chance_novice` | 0 - 100 (step 5) | 50 | Roll target for novice tier |
| `charge_chance_experienced` | 0 - 100 (step 5) | 50 | Roll target for experienced tier |
| `charge_chance_veteran` | 0 - 100 (step 5) | 50 | Roll target for veteran tier |
| `charge_chance_master` | 0 - 100 (step 5) | 50 | Roll target for master tier |

All defaults match vanilla behavior. Sliders are tuning surface, not feature toggles.

### Tracing

Following the convention from `at_squad_memory._decay_tick` and `at_state_machine._deescalate_tick`:

- `[PATCH]` install — info on success per patched function, warn if `xr_eat_medkit.heal_hp` or `heal_bleed` unavailable
- `[HEAL] hp_tick id=X mult=Y health=Z left=N [Tms]` — medkit heal tick, debug-gated, with `xprofiler.new_if(_dbg):get_ms()` timing
- `[HEAL] complete id=X reason=R left=N [Tms]` — medkit heal end, debug-gated, reason in {`no_obj`, `dead`, `ticks_exhausted`}
- `[HEAL] bleed_tick id=X bleeding=Z left=N [Tms]` — bandage bleed tick, debug-gated
- `[HEAL] bleed_complete id=X reason=R left=N [Tms]` — bandage bleed end, debug-gated, same reasons
- `[CHARGE] id=X rank=R tier=T chance=C rolled=V granted=B` — debug-gated, per roll
- `[CHARGE] already_processed id=X name=N` — debug-gated, per save-load skip

xprofiler timing is a no-op singleton when `_dbg` is false (zero luabind crossings); real `profile_timer` when on. Debug log generation is uniformly gated `if _dbg then log.debug(...) end`.

### What's intentionally not changed

- The `eat_medkit:update` stage machine and its gates (alive, not trader/zombied, not `IsWounded(npc)`, combat-filter from LTX `in_combat`/`out_combat`).
- LTX threshold values (`medkit_health = 65`, `bandage_bleeding = 0.15`). Threshold tuning would require monkey-patching `_eating.max_h` / `_eating.min_b` which are module-locals not exposed externally. Output-level tuning (heal rate multiplier) covers most of the player intent.
- `heal_bleed` behavior (vanilla `npc.bleeding = 0.07` per tick). We patch it for logging only; the bleeding value and tick count are unchanged. The engine field is a hard set, not a delta, so an MCM multiplier would be unintuitive.

---

## Engine constraint

Once the engine combat planner activates, it overrides script states. The architecture minimizes conflict by feeding engine memory (information) rather than forcing engine state (mental state, destinations). Where forced state is needed (tactical flee), the design uses the Demonized AOE panic API which has its own state machine that re-acquires cleanly.

The hit-disclosure design follows this principle: `enable_memory_object` and `register_in_combat` are the engine's own mechanisms — we just guarantee they fire across the whole squad. `force_set_goodwill` writes into the engine's relation registry; the engine reads it during its standard combat evaluation.

---

## Per-NPC versus per-squad scope decision matrix

| Concern | Scope | Layer |
|---|---|---|
| "This NPC was hit by X" | per-squad (accumulates across members) | substrate per_shooter |
| "Is this NPC in combat right now" | per-squad | substrate state |
| "Has this shooter been disclosed to this squad already" | per-squad | substrate disclosed_shooters |
| "Which medkit/bandage sections can NPCs consume" | global (loaded once at boot) | t25 DLTX overlay on `[plugin]` |
| "How fast does this NPC's heal-over-time tick" | global multiplier (MCM-tunable, read per tick) | t27 `_patched_heal_hp` |
| "Does this NPC get the lifetime healing_charge fallback" | per-NPC, decided per rank tier at first net spawn | t27 `_on_net_spawn` + `at_charge_processed` flag |
| "What weapon should this NPC carry" | per-NPC | t18 callback |
| "How fast does the engine aim" | global | t16 cvar |
| "How wide is this NPC dispersion cone" | per-NPC, per-shot (rank tier curve) | t20 Lua hook |

When a concern straddles scopes (e.g. "should this NPC have tighter aim because his base is under attack"), it requires either a per-NPC mechanism with a substrate read (script hook reading squad state), or a new engine PR exposing per-NPC accessors. Global engine cvars are never modulated by per-squad state.

---

## Accuracy hook (backlog, t20)

Rank-aware NPC dispersion lives in script, not in engine cvars. Engine path: `CAI_Stalker::GetWeaponAccuracy()` at `xray-monolith/src/xrGame/ai/stalker/ai_stalker_fire.cpp:77` computes per-shot dispersion (radians; lower = tighter) and dispatches to `_g.CAI_Stalker__GetWeaponAccuracy(npc, wpn, base, body_state, movement_type)` at lines 135-139. Called from `CWeapon::GetFireDispersion()` at `WeaponDispersion.cpp:45` per shot, not per frame; luabind crossing cost is microseconds at typical combat firing rates.

Why script and not cvars: the engine rank curve degenerates on modded gamedata. `Rank()` clamps to [0,100] but modded `<rank>` values are in the thousands, so `rank_k = 1.0` for every NPC and `m_fRankDisperison` collapses to the constant `expirienced_rank_dispersion = 0.8`. A cvar scaling the novice endpoint would be a dead knob. The Lua hook receives the full pre-baked `base` per shot and can replace it with any curve. See `todo-demonized-exes.md` n014 (DROPPED, PR #544 closed unmerged) for the rejected engine-cvar approach.

---

## File layout

```
AlifeTactics/
├── doc/
│   ├── architecture.md         (this file)
│   ├── changelog               (version history)
│   ├── readme.txt              (long-form description)
│   └── img/
│       └── logo.jpg            (800x400 banner for moddb)
├── gamedata/
│   ├── configs/
│   │   ├── ai_tweaks/
│   │   │   └── mod_xr_eat_medkit_at.ltx   # DLTX overlay: vanilla medkit/bandage lists (t25)
│   │   └── text/
│   │       ├── eng/ui_st_mcm_at.xml       # English MCM strings
│   │       └── rus/ui_st_mcm_at.xml       # Russian MCM strings (windows-1251)
│   ├── scripts/
│   │   ├── _at_deps.script                # xlibs dependency gate
│   │   ├── at_core_mcm.script             # MCM definition
│   │   ├── at_squad_memory.script         # substrate (t22)
│   │   ├── at_state_machine.script        # state machine (t23)
│   │   ├── at_ext_hitresponse.script      # hit disclosure (t24)
│   │   ├── at_ext_health.script           # NPC health controls (t27)
│   │   └── at_test.script                 # console test commands
│   └── textures/
│       └── at_mcm_banner.dds              # MCM banner (512x50 DXT5)
├── LICENSE
└── README.md
```

Backlog scripts (not yet present):

```
├── at_ext_flee.script                # tactical flee (t13)
├── at_ext_persistence.script         # memory persistence (t14)
├── at_ext_scheme.script              # combat scheme select (t03)
├── at_ext_cvars.script               # global combat tuning (t16)
├── at_ext_stance.script              # stance/weapon bias (t18)
├── at_ext_accuracy.script            # rank-aware dispersion hook (t20)
└── at_cond.script                    # condlist condition functions (t03, t14)
```

Namespace: `at_*` (parallel to `ap_*` for AlifePlus, `ag_*` for AlifeGuard, `x*` for xlibs).

---

## Logging and test infrastructure

Each module owns its own `xlog.get_logger("AT.X", { outfile = "alifetactics.log" })` facade. All five modules write to the same `alifetactics.log` with distinct prefixes (AT.MCM, AT.MEM, AT.STATE, AT.HIT, AT.TEST). MCM `log_level` controls verbosity; each module subscribes to `on_option_change` and `mcm_option_restore_default` to refresh its level and derived `_dbg` flag at runtime.

`xprofiler.new_if(_dbg)` wraps the decay and de-escalation ticks. Null-singleton when DEBUG is off (zero luabind), real profile_timer when DEBUG is on. Timing logged in the DEBUG line itself.

`at_test.script` provides console commands invoked via `run_string at_test.<func>()`:

- `at_spawn()` — spawn `bandit_sim_squad_novice` 50m ahead
- `at_spawn_veteran()` — `bandit_sim_squad_veteran` 50m ahead
- `at_spawn_master()` — `bandit_sim_squad_alpha` 50m ahead
- `at_spawn_far()` — novice 100m ahead
- `at_dump()` — log all substrate records to alifetactics.log

Spawn pattern: create at actor's vertex, teleport to actor:position() + actor:direction() * distance via `TeleportSquad`. Off-AI-map targets leave the squad at the actor.

---

## See also

- Task queue: `stalker-dev/doc/todo/todo-alifetactics-next.md` (in-flight, groomed)
- Brainstorm pool: `stalker-dev/doc/todo/todo-alifetactics-backlog.md`
- Engine PR queue: `stalker-dev/doc/todo/todo-demonized-exes.md`
- xlibs architecture: `stalker-mods/xlibs/doc/architecture.md`
- AlifePlus architecture: `stalker-mods/AlifePlus/doc/architecture.md`
- AlifeGuard architecture: `stalker-mods/AlifeGuard/doc/architecture.md`
- AlifeBalance architecture: `stalker-mods/AlifeBalance/doc/architecture.md`
