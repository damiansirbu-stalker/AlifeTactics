# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Built around a shared per-squad memory DTO that user-facing systems consume. Five user-facing systems sit on top of one internal core.

Built on xlibs (xsquad, xttltable, xtime, xprofiler, xlog, xmcm, xslice, xcreature).

Part of a four-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** tunes existing rates and counts, **AlifeGuard** keeps alife state clean, **AlifeTactics** controls how NPCs fight in combat (this mod).

---

## Status

Version 1.0.0.

| Module | Type | State |
|---|---|---|
| `_at_deps.script` | infra | done |
| `at_mcm.script` | infra | done |
| `at_squad_memory.script` | internal core | done |
| `at_test.script` | infra | done |
| `at_hitresponse.script` | feature | done |
| `at_health.script` | feature | done |
| `at_accuracy.script` | feature | done |
| `at_maneuver.script` | feature | partial (works mechanically; effectiveness pending runtime verification) |
| `at_stance.script` | feature | done |
| `configs/ai_tweaks/mod_xr_eat_medkit_at.ltx` | data | done |

Backlog (not built):
- Tactical flee (per-squad retreat to friendly smart under power imbalance)
- Memory persistence (extended danger inertion under sustained combat)
- Combat scheme selection (per-NPC combat_type via condlist)

Groomed task entries in `stalker-dev/doc/todo/todo-alifetactics-next.md`; brainstorm pool in `todo-alifetactics-backlog.md`.

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
│   │   │   └── mod_xr_eat_medkit_at.ltx       # DLTX overlay: vanilla medkit/bandage lists
│   │   └── text/eng/ui_st_mcm_at.xml          # English MCM strings
│   ├── scripts/
│   │   ├── _at_deps.script                    # dependency gate
│   │   ├── at_mcm.script                      # MCM configuration
│   │   ├── at_squad_memory.script             # DTO + role assignment + decay
│   │   ├── at_hitresponse.script              # Hit Sharing system
│   │   ├── at_health.script                   # Healing system
│   │   ├── at_accuracy.script                 # Accuracy system
│   │   ├── at_maneuver.script                 # Maneuver system
│   │   ├── at_stance.script                   # Stance Switch system
│   │   └── at_test.script                     # console test commands
│   └── textures/
│       └── at_mcm_banner.dds                  # MCM banner
├── LICENSE
└── README.md
```

Namespace: `at_*` (parallel to `ap_*` for AlifePlus, `ag_*` for AlifeGuard, `x*` for xlibs).

---

## Five user-facing systems

Each system has its own file, its own MCM tab, and one master toggle. Plus one internal core that all squad-aware systems read from.

| System | File | MCM Tab | Master toggle |
|---|---|---|---|
| Hit Sharing | `at_hitresponse.script` | Hit Sharing | `hit_share_enabled` |
| Healing | `at_health.script` | Healing | `healing_enabled` |
| Accuracy | `at_accuracy.script` | Accuracy | `accuracy_enabled` |
| Maneuver | `at_maneuver.script` | Maneuver | `maneuver_enabled` |
| Stance Switch | `at_stance.script` | Stance Switch | `stance_enabled` |

Plus `at_squad_memory.script` — internal core. No MCM exposure. Always on.

---

## Squad Memory (internal core, no MCM)

The DTO. One Lua table per squad, keyed by `squad.id`. Holds the shared per-squad state. Lazy-initialized on first `get(squad_id)`. Cleared on `server_entity_on_unregister`.

Record schema:

| Field | Type | Written by | Read by | Description |
|---|---|---|---|---|
| `per_shooter[shooter_id]` | record | hit_share | maneuver | `{ count, last_hit_time, total_damage }` — accumulated hits per shooter against this squad |
| `disclosed_shooters[shooter_id]` | bool | hit_share | hit_share | session-persistent idempotency guard |
| `engaged_until` | number | hit_share | maneuver | xtime.game_sec when active combat ends; set to `now + ENGAGED_WINDOW_SEC` on every kept hit |
| `roles[member_id]` | string | squad_memory | maneuver, stance | tactical role per member: `lead` / `suppressor` / `assaulter` / `flanker` |

Role assignment (`assign_roles`) is called from hit_share on first disclosure per squad. Idempotent — repeats no-op. Roles never change once assigned. New members joining the squad after assignment default to FLANKER.

Decay tick (every 5s) prunes `per_shooter` entries older than `substrate_retention_sec` (MCM-tunable, default 60s).

The substrate has no on/off toggle. The DTO is always available; if hit_share is disabled, nothing writes to it, but consumers can still call `get()` safely (record will be empty).

### Tactical role distribution

Doctrine reference: US Army FM 3-21.8 small-unit infantry tactics — fire and movement, base of fire + maneuver element split.

| Role | Function |
|---|---|
| Lead | commander, holds front |
| Suppressor | base of fire from cover (low silhouette, sustained fire) |
| Assaulter | closes distance, runs toward enemy |
| Flanker | maneuvers to angular position |

Distribution scaled by squad size:

| Squad size | Lead | Suppressor | Assaulter | Flanker |
|---|---|---|---|---|
| 1 (solo) | 50% | 50% | 0 | 0 |
| 2 | 1 | 1 | 0 | 0 |
| 3 | 1 | 1 | 0 | 1 |
| 4 | 1 | 1 | 1 | 1 |
| 5 | 1 | 2 | 1 | 1 |
| 6 | 1 | 2 | 1 | 2 |
| 7 | 1 | 2 | 1 | 3 |
| 8+ | 1 | 3 | 1 | (rest) |

Solo NPCs (size 1) roll by weapon: sniper/launcher always Suppressor, pistol/shotgun/SMG always Lead, rifle and other 50/50 Lead-or-Suppressor.

Non-commander role assignment is weapon-driven via the `kind` field on the NPC's active item:

| Weapon kind | Preferred role |
|---|---|
| `w_sniper`, `w_launcher` | Suppressor |
| `w_pistol`, `w_shotgun`, `w_smg`, `w_knife` | Assaulter |
| `w_rifle` | Flanker |

Assignment is greedy: each member tries their preferred role first; cap overflow falls through to Flanker.

---

## Hit Sharing

Hooks `npc_on_hit_callback`. When a faction-enemy hits any squad member, the entire squad is force-disclosed to the shooter on hit #1. Extends the engine's audio-range squad disclosure to distant patrol members and suppressed-weapon victims.

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
2. **Faction-relation gate** via `game_relations.is_factions_enemies(npc_community, shooter_community)`. Same-community hits rejected. Mirrors the engine's friendly-fire skip at our hook entry so substrate is never polluted by accidental hits.
3. Resolve squad via `get_object_squad(npc)`; skip solo NPCs.
4. Substrate writes: increment `record.per_shooter[shooter_id].count`, refresh `last_hit_time`, accumulate `total_damage`.
5. Write `record.engaged_until = now + ENGAGED_WINDOW_SEC` — marks the squad as in active combat for the next window.
6. Idempotency check: if `record.disclosed_shooters[shooter_id]` is set, return.
7. Otherwise, set the flag, call `at_squad_memory.assign_roles(squad)`, and call `_disclose(squad, who)` — three engine APIs per online squadmate:

   - **`force_set_goodwill(-2000, who)`** — writes RELATION_REGISTRY personal goodwill (`relation_registry.cpp:161-179`). `CAI_Stalker::tfGetRelationType` routes through RELATION_REGISTRY for stalkers so this drives every downstream `is_relation_enemy` check. Gated on `IsStalker(who)`: `ForceSetGoodwill` smart_casts both ids to `CSE_ALifeTraderAbstract` and logs an error for mutants. For mutant / helicopter / anomaly shooters, the goodwill write is skipped.
   - **`enable_memory_object(who, true)`** — toggles `m_enabled` on existing visual/sound/hit memory entries (`memory_manager.cpp:151-156`). No-op when no prior entry; cheap insurance otherwise. Works for any shooter type.
   - **`register_in_combat()`** — sets the member's squad_mask bit in `CAgentMemberManager::m_combat_mask` (`agent_member_manager.cpp:114-132`). This is the unlock for engine-native squad memory propagation: with the whole squad's bits set, the next `agent_memory_manager` tick ORs the full combat_mask into the victim's hit-memory entry's `m_squad_mask`, propagating memory of the shooter across every member including distant patrols.

### Net behavior

- Engine handles audio-range squadmates on hit #1 (free, automatic).
- Our hook handles distant patrol squadmates on hit #1 by forcing them into combat_mask, letting the engine's own propagation pipe carry the memory.
- Hostility for the shooter is pinned at -2000 personal goodwill on every squadmate — survives community-relation drift, lasts the session.
- Subsequent hits from the same shooter against the same squad no-op via `disclosed_shooters`.

---

## Healing

Per-NPC self-healing. Vanilla `xr_eat_medkit.script` has a working stage machine, but vanilla `ai_tweaks/xr_eat_medkit.ltx [plugin]` ships without `medkits=` / `bandages=` keys so `parse_list` returns `{}` and the consumption loop iterates zero times.

### Data layer fix

`mod_xr_eat_medkit_at.ltx` is a DLTX overlay on `![plugin]` adding `medkits = medkit, medkit_army, medkit_scientic, medkit_ai1, medkit_ai2, medkit_ai3` and `bandages = bandage`. Boot-time, no runtime toggle.

### Runtime tuning

`at_health.script` installs two hooks on `on_game_start`:

| Hook | Mechanism | What it changes |
|---|---|---|
| Heal rate multiplier | `xr_eat_medkit.heal_hp = _patched_heal_hp` | Per-tick `change_health(0.05 * mult)` reads MCM each tick; reschedule via `xr_eat_medkit.heal_hp` lookup propagates the patch through all 13 ticks |
| Bandage tick logging | `xr_eat_medkit.heal_bleed = _patched_heal_bleed` | Logging-only wrapper around vanilla bleed loop |
| Per-rank healing-charge | `RegisterScriptCallback("npc_on_net_spawn", _on_net_spawn)` | Reads `ranks.get_obj_rank_name(npc)`, folds 8 rank names into 4 MCM tiers, rolls per-tier chance, overrides vanilla's flat 50% roll. Per-NPC `at_charge_processed` se_var prevents re-roll. |

---

## Accuracy

Rank-aware NPC dispersion in script. `at_accuracy.script` subscribes to the vanilla `npc_shot_dispersion` callback (declared in `axr_main.script:126`, dispatched from `_g.CAI_Stalker__GetWeaponAccuracy` at `_g.script:1213-1217`).

Why script and not cvars: the engine rank curve degenerates on Anomaly gamedata. `Rank()` clamps to `[0, 100]` at `ai_stalker.cpp:761`, but vanilla `<rank>` intervals run to 26999 (game_relations.ltx:8). All Anomaly NPCs end up at `rank_k = 1.0`, so `m_fRankDisperison` collapses to the constant `dispersion_experienced_k = 0.8`. Cvar tuning is a dead knob.

Math: `out = (base / 0.8) * mult` — divides out the engine's baked-in rank step, applies our per-tier multiplier.

8 tiers (novice / trainee / experienced / professional / veteran / expert / master / legend) with defaults from 1.00 down to 0.38.

Per-shot hot path. Cost ~1.5μs per call when DEBUG off (2 luabind crossings via `ranks.get_obj_rank_name`, the rest pure Lua).

---

## Maneuver

Squad-coordinated combat positioning. Per-NPC GOAP injection on `motivation_action_manager`.

### Design

Each non-commander member's evaluator returns true when:
1. Maneuver is enabled
2. NPC is alive and not wounded
3. NPC has a `best_enemy` (engine memory)
4. Squad exists and is not Monolith/zombied
5. Squad is currently engaged (`xtime.game_sec() < record.engaged_until`)
6. Member's role is Assaulter or Flanker (Lead and Suppressor pass through)
7. 2-second throttle expired for this squad
8. Substrate has a primary enemy (highest count in `per_shooter`)

Action `:execute()`:
1. Compute direction from NPC to primary enemy
2. Rotate by role-specific angle (Assaulter: 0°; Flanker: outer ±60-90° for ranks 1/2, inner ±15-55° alternating for 3+)
3. Pick a level vertex via `level.vertex_in_direction(base_vid, rotated_dir, radius)` with fallback walk-down
4. Call `npc:set_dest_level_vertex_id(vid)` and `npc:set_path_type(game_object.level_path)`
5. Call `state_mgr.set_state(npc, "assault_fire", nil, nil, {look_object = npc:best_enemy()}, {fast_set = true})` so the NPC fires while moving

### GOAP wiring

Action preconditions: `property_alive=true`, `property_enemy=true`, `property_danger=true`, `MANEUVER_EVAID=true`. Effect: `MANEUVER_EVAID=false`.

`configure_actions` grafts `MANEUVER_EVAID=false` precondition onto five existing actions:
- `stalker_ids.action_combat_planner`
- `stalker_ids.action_danger_planner`
- `xr_actions_id.state_mgr + 2`
- `xr_actions_id.alife`
- `xr_danger.actid`

This is the pattern from `axr_fight_from_cover.script` (ReDone Combat AI / Wuut AI Extension). Blocking the combat_planner via precondition forces the engine planner to run our action first to satisfy the new requirement. Our action's effect provides `MANEUVER_EVAID=false`, satisfying combat_planner's grafted precondition. Combat_planner is then unblocked but the evaluator re-queries on the next planner tick and returns true again (squad still engaged), so our action runs every tick as long as the squad is engaged.

Without this graft (combat_planner unblocked), the engine planner picks combat_planner over our action because combat_planner's effect (`property_enemy=false`) directly satisfies the parent goal direction while our action's effect does not.

### Binder

Per-NPC bind via `npc_on_net_spawn`. Deferred to after `actor_on_first_update` via `_first_update_fired` flag to avoid modifying GOAP during LSS save-restoration. The xslice sweep at first_update binds pre-existing NPCs in batches of 5 per frame. Late spawns hit the immediate-bind path.

`FACTION_SKIP` excludes Monolith and zombied stalkers from binding (they have their own combat archetypes).

---

## Stance Switch

Hooks the modded-exe `_G.CAI_Stalker__CombatSetBodyState(npc, wo, body_state)` functor at `stalker_movement_manager_base_inline.h:51-59`. Returns `eBodyStateCrouch` for Suppressor-role NPCs when the engine selected Stand for a static-cover operator.

### Engine call sites

The functor fires from `body_state_combat_override` calls in `stalker_combat_actions.cpp`. Enumerated EWorldOperators that reach the functor: `{12, 14, 17, 20, 21, 22, 23, 25, 27, 28, 39}` (GetItemToKill, MakeItemKilling, GetReadyToKill, RetreatFromEnemy, TakeCover, LookOut, HoldPosition, DetourEnemy, HideFromGrenade, SuddenAttack, ThrowGrenade).

Of these, the only static-cover ops are LookOut (22) and HoldPosition (23). The remaining operators are movement transients where crouch is doctrinally wrong (slows movement under fire).

### Override set

```
OVERRIDE_OPS = {
    [OP_LOOKOUT]       = true,  -- 22
    [OP_HOLD_POSITION] = true,  -- 23
}
```

Once a Suppressor is crouched via LookOut or HoldPosition, subsequent actions (KillEnemy, WaitInCover, HoldAmbushLocation) inherit the body_state from the previous action — the engine doesn't call `set_body_state` for these. So Suppressor stays crouched through sustained fire by inheritance.

### Composition chain

`_prev_functor` captures any prior `_G.CAI_Stalker__CombatSetBodyState` installer at `on_game_start`. The chain ensures we compose with any other mod touching this seam instead of silently overriding.

### Role gate

Reads role via `at_squad_memory.get_role(squad_id, npc_id)`. Returns vanilla body_state for Lead / Assaulter / Flanker / unbound NPCs. Suppressor returns Crouch when engine picked Stand.

---

## What the engine does and what we feed it

The architecture principle is to feed engine memory and state, not fight it. Per-system summary:

| System | Engine state we write | Engine APIs called |
|---|---|---|
| Hit Sharing | RELATION_REGISTRY personal goodwill, memory entry m_enabled, agent_member_manager m_combat_mask | `force_set_goodwill`, `enable_memory_object`, `register_in_combat` |
| Healing | NPC health field, bleeding field, `healing_charge` se_var | `change_health`, direct `bleeding =` write, `se_save_var` |
| Accuracy | Per-shot dispersion radius via callback return | (subscribes to `npc_shot_dispersion`) |
| Maneuver | NPC dest level_vertex_id, path_type, animation state | `set_dest_level_vertex_id`, `set_path_type`, `state_mgr.set_state` |
| Stance Switch | NPC body_state via functor return | (functor at `_G.CAI_Stalker__CombatSetBodyState`) |

The engine then runs its own combat detection (property_enemy, m_combat_mask, agent_memory propagation) on the state we wrote. No system reimplements engine behavior; each one nudges engine state to produce the desired outcome.

---

## Logging

Each module owns its own `xlog.get_logger("AT.X", { outfile = "alifetactics.log" })` facade. All write to `alifetactics.log` with distinct prefixes:

- `AT.MCM` — MCM configuration
- `AT.MEM` — Squad Memory DTO + decay
- `AT.HIT` — Hit Sharing
- `AT.HEALTH` — Healing
- `AT.ACC` — Accuracy
- `AT.MANEUVER` — Maneuver
- `AT.STANCE` — Stance Switch
- `AT.TEST` — console test harness

MCM `log_level` (ERROR/WARN/INFO/DEBUG) controls verbosity. Each module subscribes to `on_option_change` and `mcm_option_restore_default` to refresh its level and derived `_dbg` flag.

`xprofiler.new_if(_dbg)` wraps profile-relevant code paths. Null singleton when DEBUG off (zero luabind); real `profile_timer` when on. All `log.debug` calls gated by `if _dbg then` so format strings are never built when off.

### Key debug events

- `[DECAY]` — substrate decay tick stats
- `[CLEAR]` — substrate clear on entity unregister
- `[ROLES]` — role assignment per squad with distribution
- `[HIT]` — hit handler reject reasons or already-disclosed cases (includes `engaged_until`)
- `[DISCLOSURE]` — full-squad disclosure with member count and `engaged_until`
- `[UNREGISTER]` — substrate cleanup on entity despawn
- `[HEAL]` `hp_tick`, `complete`, `bleed_tick`, `bleed_complete`
- `[CHARGE]` — healing charge rolls
- `[PATCH]` — install messages for xr_eat_medkit patches
- `[ACC]` — per-shot accuracy calculation
- `[SWEEP]` — xslice bind sweep status
- `[BIND]` — per-NPC GOAP bind
- `[MANEUVER]` `eval_true`, `init`, `dispatch`, `no_vid`, `skip`, `set_state`, `set_state_skip`
- `[STANCE]` — body_state override fires

---

## Test infrastructure

`at_test.script` provides console commands invoked via `run_string at_test.<func>()`:

- `at_spawn()` — spawn `bandit_sim_squad_novice` 50m ahead
- `at_spawn_veteran()` — `bandit_sim_squad_veteran` 50m ahead
- `at_spawn_master()` — `bandit_sim_squad_alpha` 50m ahead
- `at_spawn_far()` — novice 100m ahead
- `at_dump()` — log all substrate records (squad_id, engaged state, per_shooter count, engaged_until)

---

## See also

- Task queue: `stalker-dev/doc/todo/todo-alifetactics-next.md`
- Brainstorm pool: `stalker-dev/doc/todo/todo-alifetactics-backlog.md`
- Engine PR queue: `stalker-dev/doc/todo/todo-demonized-exes.md`
- xlibs architecture: `stalker-mods/xlibs/doc/architecture.md`
- AlifePlus architecture: `stalker-mods/AlifePlus/doc/architecture.md`
- AlifeGuard architecture: `stalker-mods/AlifeGuard/doc/architecture.md`
- AlifeBalance architecture: `stalker-mods/AlifeBalance/doc/architecture.md`
