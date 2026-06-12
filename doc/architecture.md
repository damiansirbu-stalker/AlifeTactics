# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Independent user-facing systems: a hit-share force-disclosure, a self-heal data + animation layer, a per-rank weapon accuracy curve, a full-file xr_danger override with bug fixes and three toggleable improvements, and a Pattern B planner takeover that injects an alternative combat AI on a configurable share of NPCs (slider, default 100% — coexists with vanilla / GAMMA / AI Rework / RCAI / Useful Idiots / Mora, zero vanilla file overrides). No shared substrate.

Built on xlibs (xsquad, xttltable, xtime, xprofiler, xlog, xmcm, xslice, xcreature).

Part of a four-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** tunes existing rates and counts, **AlifeGuard** keeps alife state clean, **AlifeTactics** controls how NPCs fight in combat (this mod).

---

## Status

Version 1.0.0.

| Module | Type | State |
|---|---|---|
| `_at_deps.script` | infra | done |
| `at_mcm.script` | infra | done |
| `at_test.script` | infra | done |
| `at_hitresponse.script` | feature | done |
| `at_health.script` | feature | done |
| `at_accuracy.script` | feature | done |
| `at_combat.script` | feature | Pattern B planner takeover; 14 behaviors + 8 state machines + per-faction × per-environment rule lists; delta-driven decide with squad-uniqueness skip |
| `xr_danger.script` | feature | done (full-file override) |
| `at_jam.script` | feature | done (modded-exes xr_weapon_jam.GetConditionMisfireProbability override; suppresses script-injected NPC misfire) |
| `at_ammo.script` | feature | done (per-NPC virtual ledger drains highest-tier ammo each 5s combat tick down to MIN_SPARE; inventory boxes clamped to ledger to defeat engine try_advance_ammo top-up; npc_on_death_callback trims all boxes to MIN_SPARE so they survive decide_items_to_keep; AP sections gated by `min_ap_rank` LTX (default 17499 = RANK_MASTER)) |
| `zzz_at_health_patch.script` | feature | done (vanilla xr_eat_medkit re-roll suppressor) |
| `configs/ai_tweaks/mod_xr_eat_medkit_at.ltx` | data | done |
| `configs/ai_tweaks/xr_danger.ltx` | data | done |

Backlog (not built):
- Tactical flee (per-squad retreat to friendly smart under power imbalance)
- Memory persistence (extended danger inertion under sustained combat)
- Combat scheme selection (per-NPC combat_type via condlist)
- NPC weapon bias (per-NPC callback overrides for loadout selection)

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
│   │   │   ├── mod_xr_eat_medkit_at.ltx       # DLTX overlay: vanilla medkit/bandage lists
│   │   │   └── xr_danger.ltx                  # paired with xr_danger override
│   │   └── text/eng/ui_st_mcm_at.xml          # English MCM strings
│   ├── scripts/
│   │   ├── _at_deps.script                    # dependency gate
│   │   ├── at_mcm.script                      # MCM configuration
│   │   ├── at_hitresponse.script              # Hit Sharing system
│   │   ├── at_health.script                   # Healing system
│   │   ├── at_accuracy.script                 # Accuracy system
│   │   ├── at_combat.script                   # Combat system (Pattern B planner takeover)
│   │   ├── xr_danger.script                   # full-file override (Danger system)
│   │   ├── at_jam.script                      # modded-exes xr_weapon_jam override (Weapon Jam system)
│   │   ├── at_ammo.script                     # NPC ammo simulation (NPC Ammo system)
│   │   ├── zzz_at_health_patch.script         # vanilla xr_eat_medkit re-roll suppressor
│   │   └── at_test.script                     # console test commands
│   └── textures/
│       └── at_mcm_banner.dds                  # MCM banner
├── LICENSE
└── README.md
```

Namespace: `at_*` (parallel to `ap_*` for AlifePlus, `ag_*` for AlifeGuard, `x*` for xlibs).

---

## User-facing systems

Each system has its own file, its own MCM tab, and one master toggle.

| System | File | MCM Tab | Master toggle |
|---|---|---|---|
| Hit Sharing | `at_hitresponse.script` | Hit Sharing | `hit_share_enabled` |
| Healing | `at_health.script` | Healing | `healing_enabled` |
| Accuracy | `at_accuracy.script` | Accuracy | `accuracy_enabled` |
| Combat | `at_combat.script` | Combat | `combat_enabled` |
| Danger | `xr_danger.script` (full-file override) | Fixes > Danger | bug fixes always-on; three toggleable improvements |
| Weapon Jam | `at_jam.script` | Fixes > Weapon Jam | `jam_enabled` |
| NPC Ammo | `at_ammo.script` | Fixes > NPC Ammo | `ammo_enabled` |

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
2. **Faction-relation gate** via `game_relations.is_factions_enemies(npc_community, shooter_community)`. Same-community hits rejected. Mirrors the engine's friendly-fire skip at our hook entry.
3. Resolve squad via `get_object_squad(npc)`; skip solo NPCs.
4. Write/refresh timestamp: `_disclosed[squad_id][shooter_id] = xtime.game_sec()`. Every hit refreshes.
5. If the entry existed before the write (idempotency hit): return. The squad already engaged this shooter in this fight.
6. Otherwise (first hit, or first hit since decay): call `_disclose(squad, who)`. Three engine APIs per online squadmate:

   - **`force_set_goodwill(-2000, who)`**: writes RELATION_REGISTRY personal goodwill (`relation_registry.cpp:161-179`). `CAI_Stalker::tfGetRelationType` routes through RELATION_REGISTRY for stalkers so this drives every downstream `is_relation_enemy` check. Gated on `IsStalker(who) AND IsStalker(mem_npc)`: `ForceSetGoodwill` smart_casts both ids to `CSE_ALifeTraderAbstract` and logs an error if either side is non-stalker. Mutant shooters and mutant squadmates both skip the goodwill write.
   - **`enable_memory_object(who, true)`**: toggles `m_enabled` on existing visual/sound/hit memory entries (`memory_manager.cpp:151-156`). No-op when no prior entry. Receiver must be `CCustomMonster` (`script_game_object2.cpp:262`); stalkers and mutants qualify, the actor does not — irrelevant here since `mem_npc` is always a squad member.
   - **`register_in_combat()`**: sets the member's squad_mask bit in `CAgentMemberManager::m_combat_mask` (`agent_member_manager.cpp:114-132`). This is the unlock for engine-native squad memory propagation. With the whole squad's bits set, the next `agent_memory_manager` tick ORs the full combat_mask into the victim's hit-memory entry's `m_squad_mask`, propagating memory of the shooter across every member including distant patrols. Requires `CAI_Stalker` receiver (`script_game_object_inventory_owner.cpp:1945-1955`); safe here because `npc_on_hit_callback` is dispatched only by `motivator_binder` (stalker squads), never `generic_object_binder` (mutant squads), so `mem_npc` is always a stalker.

### Decay and re-engagement

Decay tick fires every 5 seconds. Walks every `_disclosed[squad_id][shooter_id]` entry; prunes any whose timestamp is older than `hit_share_retention_min` minutes (MCM-tunable, default 2 game minutes). Pruning clears only the idempotency entry. Goodwill -2000 is RELATION_REGISTRY-persistent and survives independently for the rest of the game session.

After decay, the next hit from that shooter against that squad triggers a fresh `_disclose` call. Distant patrol squadmates get re-pinned into combat_mask for the new engagement.

### Spawn handler (mid-engagement replenishment + offline-shooter return)

`npc_on_net_spawn` fires for every stalker spawn (dispatched by `motivator_binder`). Two paths run for each spawned NPC:

1. **Inherit from squad** (case 1): if the spawning NPC's squad has active disclosures, apply `_disclose_to_member` for each disclosed shooter that resolves online. The replacement inherits the squad's combat state without waiting for the next hit.
2. **Re-disclose on shooter return** (case 2): walk every tracked squad's disclosed map. If the spawning NPC's id is present (the NPC is a previously-offline shooter coming back online), replay `_disclose_to_member` for every online squadmate of those squads. Covers members who joined while the shooter was offline.

Both paths short-circuit quickly when no entries match. Most spawn events trigger zero work.

**Mutant-shooter return is not re-disclosed by this handler.** Mutants dispatch `monster_on_net_spawn` (`bind_monster.script:298`) which AT does not subscribe to. Sustained engagement is not lost: combat_mask bits set at original hit time persist on squad members, and mutant-vs-stalker faction enmity drives target acquisition independently. The only gap is a stalker member who joins the tracking squad while the mutant shooter is offline: that member never gets disclosed to the returning mutant.

### Net behavior

- Engine handles audio-range squadmates on hit #1 (free, automatic).
- Our hook handles distant patrol squadmates on hit #1 by forcing them into combat_mask, letting the engine's own propagation pipe carry the memory.
- Hostility for the shooter is pinned at -2000 personal goodwill on every squadmate. The override survives community-relation drift and lasts the session.
- Sustained engagement: subsequent hits refresh the timestamp and return early via idempotency.
- After `hit_share_retention_min` game minutes of no hits from a given shooter, the squad's pin on that shooter expires; the next hit re-fires the full pipeline.
- Mid-fight replenishment: new squad members inherit existing disclosures on spawn.
- Offline-shooter return: when a previously-offline tracked shooter comes back online, the spawn handler replays disclosure to every member of the squads tracking them. Members who joined while the shooter was offline get pinned at this moment.

---

## Healing

Per-NPC self-healing. Vanilla `xr_eat_medkit.script` has a working stage machine, but vanilla `ai_tweaks/xr_eat_medkit.ltx [plugin]` lacks the `medkits=` / `bandages=` keys so `parse_list` returns `{}` and the consumption loop iterates zero times.

### Data layer fix

`mod_xr_eat_medkit_at.ltx` is a DLTX overlay on `![plugin]` adding `medkits = medkit, medkit_army, medkit_scientic, medkit_ai1, medkit_ai2, medkit_ai3` and `bandages = bandage`. Boot-time, no runtime toggle.

### Runtime tuning

`at_health.script` installs two hooks on `on_game_start`:

| Hook | Mechanism | What it changes |
|---|---|---|
| Heal rate multiplier | `xr_eat_medkit.heal_hp = _patched_heal_hp` | Per-tick `change_health(0.05 * mult)` reads MCM each tick; reschedule via `xr_eat_medkit.heal_hp` lookup propagates the patch through all 13 ticks |
| Bandage tick logging | `xr_eat_medkit.heal_bleed = _patched_heal_bleed` | Logging-only wrapper around vanilla bleed loop |
| Per-rank healing-charge | `RegisterScriptCallback("npc_on_net_spawn", _on_net_spawn)` | Reads `ranks.get_obj_rank_name(npc)`, folds 8 rank names into 4 MCM tiers, rolls per-tier chance, overrides vanilla's flat 50% roll. Per-NPC `at_charge_processed` se_var prevents re-roll. |

### Visual layer (Path 1 script-queue overlay)

Two cosmetic cues using `npc:add_animation` directly. No state_mgr, no GOAP, no `state_lib` changes. See `doc/library/modding/state-lib-animations.md` for the Path 1 script-queue overlay mechanism.

| Cue | Trigger | Animation(s) |
|---|---|---|
| Limping | `npc_on_update` per-NPC: per-tick drop on `not alive() or IsWounded or critically_wounded` (bypasses the throttle so the overlay never outlives a wounded transition); 1s-throttled full eligibility check (`health < threshold`, `mental_state() == anim.free`, `body_state() == move.standing`, not zombied, not in smart_cover). Re-arms every 5s. Eligibility-lost branch drops tracking only; no `clear_animations()` (engine action transition or natural OMF expiry owns cleanup) | `dmg_norm_torso_<slot>_<idle\|walk\|run>_0` per `_pick_limp_anim` reading `active_slot()` + `movement_type()` (torso overlay; layers over engine-driven locomotion, legs stay attached to ground) |
| Heal anim | One-shot via `_try_play_heal_anim` from the first tick of `_patched_heal_hp` / `_patched_heal_bleed` (`left == 15`). Gated on `not npc:best_enemy()`, `not IsWounded(npc)`, `not npc:critically_wounded()`. No movement freeze, no stage machine, no mid-flight aborts. Engine drains the queue when the gesture ends; action transitions clear it on the way to action_wounded / action_critically_wounded (`stalker_base_action.cpp:24-29`) | `dmg_norm_torso_11_attack_0` (medkit) / `norm_torso_12_attack_0` (bandage) |

Limping is independent of the healing master toggle (its callback registers unconditionally; gated at runtime by `limping_anim_enabled`). Heal cue is gated by `healing_anim_enabled` and the master toggle (it lives inside the heal_hp/heal_bleed patches that only install when healing is enabled).

Combat NPCs are excluded by the `mental_state == anim.free` gate. state_mgr drives mental to `anim.danger` in combat states (`state_lib.script:326-340` hide_fire / threat).

---

## Accuracy

Rank-aware NPC dispersion in script. `at_accuracy.script` subscribes to the vanilla `npc_shot_dispersion` callback (declared in `axr_main.script:126`, dispatched from `_g.CAI_Stalker__GetWeaponAccuracy` at `_g.script:1213-1217`).

Why script and not cvars: the engine rank curve degenerates on Anomaly gamedata. `Rank()` clamps to `[0, 100]` at `ai_stalker.cpp:764`, but vanilla `<rank>` intervals run to 26999 (game_relations.ltx:8). All Anomaly NPCs end up at `rank_k = 1.0`, so `m_fRankDisperison` collapses to the constant `dispersion_experienced_k = 0.8`. Cvar tuning is a dead knob.

Math: `out = base * disp`. The engine already multiplied by `m_fRankDisperison` (= 0.8 for every Anomaly NPC after the rank clamp) before the callback fires, plus the per-state factor. We stack `disp` on top — `disp = 1.00` preserves the engine's vanilla cone, lower values tighten it.

8 tiers (novice / trainee / experienced / professional / veteran / expert / master / legend) with defaults from 1.00 down to 0.38.

Per-shot hot path. Cost ~1.5μs per call when DEBUG off (2 luabind crossings via `ranks.get_obj_rank_name`, the rest pure Lua).

---

## Combat

GOAP planner takeover. One evaluator + one action installed per stalker. While the evaluator returns true, vanilla `action_combat_planner` / `action_danger_planner` / xr_danger / state_mgr / alife are precondition-blocked. When false, vanilla resumes.

### Eligibility (`_decide_eval`)

Fast-fail chain, in order; first failing returns `(false, reason)`:

| Step | Check | Reason on fail |
|---|---|---|
| 1 | `cfg.combat_enabled` | `disabled` |
| 2 | `(npc:id() % 1000) < (cfg.combat_share * 1000)` | `hash_out` |
| 3 | `npc:alive()` | `dead` |
| 4 | `IsWounded(npc)` is false | `wounded` |
| 5 | no active pick-fail backoff | `fail_backoff` |
| 6 | `npc:best_enemy()` exists and alive | `no_enemy` |

### Decision model

| Layer | Role | Lives in |
|---|---|---|
| State machines | 8 categorical variables, each with hysteresis | `_read_*` in `at_combat.script` |
| Rule list | Per-faction × per-environment ordered `{ when, do_, on_delta?, unique? }` rows | `MILITARY_OUTDOOR`, `MILITARY_INDOOR`, `STALKER_DEFAULT` in `at_combat.script` |
| Behavior catalog | 14 named `{ plan }` rows (pure data) | `BEHAVIORS` in `at_combat.script` |
| Resolvers | 9 target functions returning a level vertex id | `TARGET_RESOLVERS` in `at_combat.script` |
| Primitives | Weapon classification, raycasts, cover graph, squad helpers | `xlibs/xcombat.script` |

### State machines

8 variables, each discrete with hysteresis. Read in cost order; first delta detected sets `first_delta`; all variables still updated each tick.

| Variable | States | Hysteresis | Reads |
|---|---|---|---|
| health | HEALTHY / WOUNDED | drop at hp < 0.50; recover at hp > 0.55 | `npc.health` |
| environment | OUTDOOR / INDOOR | lvid-cached | `xcombat.is_indoor(pos)` |
| contact | VISUAL / NO_VISUAL | 500ms debounce on losing | `npc:see(enemy)` |
| squad | INTACT / DEPLETED / DECIMATED | none (integer ratio) | `xcombat.squad_alive_count` / `xcombat.squad_initial` |
| friendly_line | CLEAR / BLOCKED | 2-tick clear debounce | `xcombat.friendly_in_line` |
| range_fit | IN_ZONE / TOO_FAR / TOO_CLOSE | 3m deadband on boundaries | `xcombat.weapon_range` + dist |
| threat | SAFE / THREATENED | 1s see_me drop, 3s hit lock | `enemy:see(npc)` + recent-hit timestamp |
| position | IN_COVER / EXPOSED | 3 consecutive raycast misses | `xcombat.has_occluder_between` |

### Throttle

| Gate | Value | Source |
|---|---|---|
| Per-NPC | 1500ms | `throttle_npc_ms` in `at_combat.ltx` |

Stamped on entry once a live enemy is confirmed, so a no-delta steady state costs one state read per window, not one per frame. Sole governor; total cost is bounded by the in-combat NPC count, so no global gate is needed.

### Per-tick flow

1. Per-NPC throttle gate — return early on miss
2. `best_enemy` check — return on nil; stamp the per-NPC throttle on pass
3. `_state[id]` init on first tick (forces `first_delta = "init"`)
4. `_update_state` reads 8 state machines, tracks `first_delta`
5. No delta → return (current behavior preserved)
6. Look up rules: `FACTION_RULES[env][community]` with `[env .. "_DEFAULT"]` fallback
7. `_decide(rules, state, first_delta, npc_id)` walks rules
8. `_switch(picked, ctx)` if behavior changed; else `_repick` if the behavior tracks a moving target (charge) or cover was stolen
9. `npc:set_sight(look.fire_point, ene_pos + 1.5y)` for fire dispatch

### Decide algorithm

```
fallback = nil
for rule in rules:
    if rule.on_delta and rule.on_delta != first_delta: skip
    if not rule.when(state): skip
    if rule.unique != false and xcombat.squadmate_in(squad_id, npc_id, rule.do_):
        fallback = fallback or rule.do_   -- eligible but taken
        continue
    return rule.do_
return fallback
```

### BEHAVIORS catalog (14 entries)

Naming: `MOVE_COVER_QUALIFIER`. `MOVE` ∈ {ADVANCE, RETREAT, HOLD}. `COVER` ∈ {COVER, OPEN}. `QUALIFIER` is posture or distinguishing action.

| Behavior | state | body | mvt | target | sniper_aim |
|---|---|---|---|---|---|
| ADVANCE_COVER_STAND | assault_fire | standing | run | cover_step_fwd | — |
| ADVANCE_COVER_FLANK | assault_fire | standing | run | cover_flank | — |
| ADVANCE_OPEN_RUN | assault_fire | standing | run | charge_enemy | — |
| ADVANCE_OPEN_WALK | raid_fire | standing | walk | charge_enemy | — |
| ADVANCE_OPEN_STEP | assault_fire | standing | walk | step_toward_short | — |
| RETREAT_COVER_STAND | panic | standing | run | step_away | — |
| RETREAT_COVER_CROUCH | sneak_fire | crouch | walk | step_away | — |
| RETREAT_OPEN_STAND | sprint | standing | run | flee_far | — |
| RETREAT_OPEN_STEP | assault_fire | standing | walk | step_away_short | — |
| HOLD_COVER_CROUCH | hide_na | crouch | run | cover_nearest | — |
| HOLD_OPEN_CROUCH | hide_fire | crouch | stand | hold | — |
| HOLD_OPEN_SNIPE | hide_sniper_fire | crouch | stand | hold | true |
| HOLD_OPEN_HIDE | hide_na | crouch | stand | hold | — |
| SUPPRESS_OPEN_CROUCH | hide_fire | crouch | stand | hold | — |

`body` / `mvt` stored as strings; resolved at apply time via `move[name]`.

### Target resolvers (9 entries)

| Resolver | Behavior(s) | Search shape |
|---|---|---|
| cover_step_fwd | ADVANCE_COVER_STAND | step 15m toward enemy → best_cover, fallback vertex_in_direction |
| cover_flank | ADVANCE_COVER_FLANK | 20m perpendicular (per-id sign) + 5m forward → best_cover |
| charge_enemy | ADVANCE_OPEN_RUN, ADVANCE_OPEN_WALK | enemy lvid directly (no reservation) |
| step_toward_short | ADVANCE_OPEN_STEP | 4m toward enemy, no cover |
| step_away | RETREAT_COVER_STAND, RETREAT_COVER_CROUCH | step 15m back → best_cover |
| flee_far | RETREAT_OPEN_STAND | 30m backward, no cover |
| step_away_short | RETREAT_OPEN_STEP | 4m back, no cover |
| cover_nearest | HOLD_COVER_CROUCH | npc + lateral bucket → best_cover (two-tier radius) |
| hold | HOLD_OPEN_CROUCH, HOLD_OPEN_SNIPE, HOLD_OPEN_HIDE, SUPPRESS_OPEN_CROUCH | current `target_lvid` or `npc_lvid` |

Squad lateral spread applied to cover-seeking resolvers via `xcombat.lateral_offset(bucket, ...)`.

Resolvers compute the intent geometry (forward / lateral / back offsets, enemy axis, squad spread); the move itself routes through the robust xlibs primitives — `cover_vertex` expands its radius and skips reserved cover, `vertex_toward` decreases its step until a free node validates, and `_repick` issues the move via `send_to`, which reroutes to the nearest accessible node so a blocked exact target degrades instead of failing. Charge (`charge_enemy`) re-resolves each tick to follow the moving enemy.

### Rule lists (v1)

Shipping rule lists. Other communities fall through to `STALKER_DEFAULT`.

| List | Communities | Environment |
|---|---|---|
| `MILITARY_OUTDOOR` | army, army_npc, dolg, freedom, isg | OUTDOOR |
| `MILITARY_INDOOR` | army, army_npc, dolg, freedom, isg | INDOOR |
| `STALKER_DEFAULT` | all unmapped | OUTDOOR + INDOOR |

Doctrine summary:

- **Military outdoor**: cover-disciplined, snipes, flanks, walks back firing when wounded; advances cover-to-cover, never in the open.
- **Military indoor**: charge with close-range weapons, no flank/snipe, crouch-fire from current spot.
- **Stalker default**: balanced, simple priority (cover when threatened, advance when too far, fire when visual, hide as fallback).

### Pattern B preconditions

`world_property(EVAL_ID, false)` added to:
- `stalker_ids.action_combat_planner`
- `stalker_ids.action_danger_planner`
- `xr_danger.actid`
- `xr_actions_id.state_mgr + 2`
- `xr_actions_id.alife`

When AT eval returns true, these vanilla actions are blocked from running. When false, no precondition match → vanilla resumes normally.

### Lifecycle

| Callback | Effect |
|---|---|
| `npc_on_net_spawn` | install evaluator + action + preconditions; sentinel-guarded |
| `npc_on_net_destroy` | clear `_installed[id]` so next online cycle re-installs |
| `server_entity_on_unregister` | clear all per-id tables; release cover reservation via action finalize |
| `actor_on_first_update` | reset all module-level tables; load tunables; refresh log level |
| `npc_on_hit_callback` | stamp `state._threat_hit_at` for 3s lock window |
| `on_option_change` / `mcm_option_restore_default` | refresh log level / `_dbg` flag |

### Resolver failure handling

| Trigger | Effect |
|---|---|
| Resolver returns nil | `_fail_streak[id]` increments |
| Streak ≥ 3 | `_fail_until[id]` set with linear backoff (3→10s, 4→20s, ...) |
| Eval during backoff | returns `(false, "fail_backoff")` → vanilla resumes |
| Successful pick | `_fail_streak[id] = 0` |

Resolvers return a reason code on failure (`no_cover`, `no_flank_room`, `no_path_to_enemy`, `adjacent`, ...). With `log_level = DEBUG`, the decide loop emits per-tick `[DECIDE]` (full state snapshot + picked behavior), `[RESOLVE]` (ok / fail with reason + streak / contested), and `[BACKOFF]` lines to `alifetactics.log`. All trace points are gated, noop when DEBUG is off.

### Sniper-aim engine flag

`HOLD_OPEN_SNIPE` sets `plan.sniper_aim = true`. `_apply_plan` calls `npc:sniper_fire_mode(true)`. Engine swaps aim direction from weapon barrel to head target (via `state_mgr.set_state(..., { look_object = enemy })`). Cleared on behavior change away from sniper or on `finalize`.

### What we don't touch

- vanilla `xr_combat.script`, `xr_combat_camper.script`, `xr_combat_monolith.script`, `xr_combat_zombied.script`
- `xr_cover.script`, `xr_smartcover.script`, `xr_combat_ignore.script`, `visual_memory_manager.script`
- smart terrain configs, job XML, condlists
- `script_combat_type` writes from other mods (they still execute; vanilla planner is blocked so they have no effect for in-share NPCs)

### MCM

| Key | Type | Default | Effect |
|---|---|---|---|
| `combat_enabled` | check | true | Master toggle. Effective on next eval tick |
| `combat_share` | track 0.0-1.0 step 0.05 | 1.0 | Stable per-id hash share of NPCs using AT combat; 0 = pure vanilla, 1 = full takeover |

### xcombat primitives (in xlibs)

| Surface | Purpose |
|---|---|
| `AGGRESSOR_KINDS`, `SNIPER_KINDS`, `RANGED_KINDS`, `WEAPON_RANGES` | Weapon classification constants |
| `weapon_kind(npc)` | Active weapon kind, TTL-cached |
| `weapon_range(kind)` | `{ min, max }` table per kind |
| `has_occluder_between(a, b)` | Static-geometry raycast between positions |
| `friendly_in_line(npc, npc_pos, ene_pos, thresh)` | Squadmate-on-firing-line projection check |
| `is_indoor(pos)` | Level-name table + surge-shelter smart proximity |
| `vertex_toward(npc, lvid, dir, step)` | Nearest free accessible vertex toward `dir`, decreasing the step until one validates |
| `cover_vertex(npc, pos, ene, max_r)` | Nearest free cover facing the enemy, expanding radius, skips reserved vertices |
| `send_to(npc, vid)` | Move toward `vid`, reroute to nearest accessible node if blocked; never fails |
| `claim_cover` / `release_cover` / `cover_stolen` | `db.used_level_vertex_ids` reservation wrappers (shared with vanilla and other mods) |
| `squad_alive_count(squad_id)` | TTL-cached live count |
| `squad_initial(squad_id, set?)` | Initial squad size per combat session |
| `squadmate_in` / `record_pick` / `clear_pick` | Per-squad behavior census |
| `lateral_offset` | Perpendicular spread for cover search |
| `INDOOR_LEVELS` | Level-name table for full-indoor levels |

---

## Danger

Full-file override of vanilla `xr_danger.script` (Alundaio). Six vanilla bug fixes always-on. Three toggleable improvements behind MCM. Paired LTX (`configs/ai_tweaks/xr_danger.ltx`) with weather-conditional distances and actor-source variant tables.

### Vanilla bugs fixed (always-on)

1. `bd_types` name collision: three perceive-type names overwrite enum values, causing three danger categories to read wrong config sections.
2. `get_danger_time` crashes on mutant corpse: vanilla calls `corpse_object:death_time()` without `IsStalker` guard; trader interface absent on mutants.
3. `eval_danger` nil-NPC guard missing: vanilla crashes when called on a torn-down NPC reference.
4. `eval_danger` non-numeric `danger_time` check missing: vanilla type-asserts on bad return.
5. `npc_on_hit_callback` referenced undefined `who_id` variable: vanilla wrote nil shooter id into `script_danger`. Vanilla callback unregistered entirely; danger pipeline now driven by `npc_on_hear_callback` and `npc_on_death_callback`.
6. Animstate reset missing on danger-state transitions: vanilla `state_mgr.set_state` calls did not invoke `sm.animstate:set_state(nil, true) + set_control()`, leaving stale lower-body animation visible across the transition. AT calls the reset at every state change site (`xr_danger.script:459-464, 528-534, 805-818`).

### Improvements (MCM Danger tab, default on)

- `danger_hit_bypass`: direct hits bypass the combat-ignore distance gate. Sniped NPCs respond regardless of attacker distance.
- `danger_attack_sound`: script_action_danger_alert dispatch for `attack_sound` danger type. Includes actor-aim gate (dot product > 0.85) so actors walking past with rifle out do not trigger cover-seek. Vanilla had no script handler for this danger type.
- `danger_actor_tables`: read separate inertion and ignore tables from `[danger_inertion_actor]` and `[danger_object_actor]` when danger source is the actor. Tune player encounters independently of NPC-vs-NPC.

### Paired LTX

`configs/ai_tweaks/xr_danger.ltx` ships values aligned to GAMMA AI Rework's base:
- Weather-conditional ignore distances (rain/storm reduces detection)
- Separate actor-source tables that respond to `actor_enemy` condition
- Dead `hit`/`sound`/`visual` keys (PerceiveType names; collide with EDangerType enum values) are dropped

In GAMMA, Useful Idiots DLTX (`mod_xr_danger_z_idiots.ltx`) overlays `[danger_inertion]` (all 8 keys) and 2 keys of `[danger_inertion_actor]` (bullet_ricochet, attack_sound) with per-NPC condlists. The `{=is_gamma}` branch falls through to the same values shipped here, so non-companion non-redone-combat behavior matches our base. Companions get 4-8s inertion via the `{=npc_companion}` branch.

### Composition

The override is marked `-- @override` so the validator skips inherited vanilla style warnings. Conflicts with mods that override `xr_danger.script` (ReDone Combat AI, GAMMA AI Rework). MCM Danger tab describes always-on fixes and the three improvement toggles.

---

## Weapon Jam

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

| Key | Type | Default | Effect |
|---|---|---|---|
| `jam_enabled` | check | true | On: override returns 0 for NPC misfire probability. Off: forward to original modded-exes function. Effective on next engine functor call, no restart. |

---

## NPC Ammo

Master-rank NPCs fire AP from inventory until depleted, then revert to vanilla magic FMJ. Player loots the remainder.

Engine context: NPCs do not consume real inventory ammo while `unlimited_ammo()` is TRUE, which is the default for every stalker (`xrServer_Objects_ALife_Monsters.cpp:2164`). The magic refill at `WeaponMagazined.cpp:520-571` produces copies of `m_DefaultCartridge`, keyed from `m_ammoTypes[m_ammoType]` at line 559-560. Setting `m_ammoType` re-keys those copies; inventory drain in this script is cosmetic for corpse-loot composition.

### Fixed AP set

`AP_SECTIONS` is a hardcoded set of clean armor-piercing cartridge sections enumerated from vanilla Anomaly `configs/items/weapons/*.ltx` (14 entries covering 5.45x39, 5.56x45, 7.62x39, 7.62x51, 7.62x54, 7.92x33, 9x18, 9x19, 9x39, 12.7x55, 12x76). Degraded `_bad` / `_verybad` variants are intentionally excluded — they're treated as "carry but don't promote". Add new calibers to the table when modpacks introduce them.

### Per-NPC state

`_state[id] = { idx, sec, left }`. Set when a master-rank NPC is first observed carrying any `AP_SECTIONS` entry in inventory; nil otherwise (those NPCs run vanilla forever). Cleared on `server_entity_on_unregister`. Not save-persisted.

### Tick algorithm

`pick(npc)` is the public entry, subscribed to `npc_on_update` (5s throttle per NPC, gated on `best_enemy()`). Each tick:

1. MCM gate (`ammo_enabled`), `npc:alive()` and `IsStalker(npc)` filter, `npc:character_rank() >= min_ap_rank` filter — sub-master returns immediately.
2. `IsWeapon(npc:active_item())` filter.
3. If `_state[id]` nil: scan `ammo_class` for the first entry in `AP_SECTIONS` with inventory count > 0; seed `_state[id]` with that idx, section, and starting count. If no AP found, leave `_state[id]` nil so the NPC stays on vanilla.
4. Drain `_state[id].left` by `DRAIN_PER_KIND[kind]`, clamped at `MIN_SPARE`.
5. If `left > MIN_SPARE`: set `m_ammoType = idx` and `_sync_box(npc, sec, left)` to defeat engine `try_advance_ammo` top-up.
6. Else: set `m_ammoType = 0` (vanilla magic FMJ for the rest of the NPC's online session).

Cost per tick per NPC: ~5-10 luabind. Sub-master NPCs early-exit at rank check; only luabind is `character_rank()`. For 50 active combat NPCs at 0.2Hz with mixed rank: ~30 luabind/sec total.

### Rank gate

`npc:character_rank() >= min_ap_rank`. Default `min_ap_rank = 17499` (= `RANK_MASTER`, sourced from `AlifePlus/ap_core_const.script:15`). Below-master NPCs never enter the picker. Their inventory AP boxes are untouched during life and trimmed at death by the death hook.

This intentionally restricts AP to master+ to prevent armor degradation across the whole NPC population. Tune via LTX.

### Death hook

`npc_on_death_callback` walks the dead NPC's inventory and trims every ammo box where `ammo_get_count() > MIN_SPARE` down to MIN_SPARE. Runs before `motivator_binder:death_callback`'s `decide_items_to_keep` call at `xr_motivator.script:362`. Boxes ≤ 5 survive `death_manager.script:456-460`'s `>5` deletion filter, so the depletion trail persists as corpse loot.

Without this hook the engine's `try_advance_ammo` top-up at the last reload before death would leave boxes at `boxSize` and `decide_items_to_keep` would release them entirely. Player would see only `death_manager.try_spawn_ammo`'s procedural box. With the hook, player loots the actual remainder.

### Tunables (`gamedata/configs/alifetactics/at_ammo.ltx`)

| Key | Default | Effect |
|---|---|---|
| `min_spare` | 3 | Floor for inventory boxes. Must be ≤ 5 to survive `decide_items_to_keep`. |
| `min_ap_rank` | 17499 | `npc:character_rank()` threshold for AP eligibility. |
| `drain_w_pistol` | 5 | Rounds drained per 5s tick when active weapon kind is `w_pistol`. |
| `drain_w_shotgun` | 2 | Same for `w_shotgun`. |
| `drain_w_smg` | 10 | Same for `w_smg`. |
| `drain_w_rifle` | 8 | Same for `w_rifle`. |
| `drain_w_sniper` | 2 | Same for `w_sniper`. |

Script-side fallback defaults match the LTX defaults so a missing key or file won't crash.

### MCM

| Key | Type | Default | Effect |
|---|---|---|---|
| `ammo_enabled` | check | true | Master toggle. Off: tick early-exits, no `m_ammoType` writes, no inventory clamping, no death-time trim. Effective on next tick. |

### Scope and limits

- Active weapon only. NPC swapping weapons mid-fight: state stays bound to the original AP section / idx until offline cycle resets state.
- Drain is by-time, not by-shots-fired. No NPC fire callback exists in vanilla. A rifle NPC firing constantly drains the same as one in cover. Acceptable proxy.
- Save/load resets `_state` unless marshal hook is added. Re-seeds on next tick for any NPC still carrying AP.
- Degraded `_ap_bad` / `_ap_verybad` variants are not promoted. NPCs carrying only degraded variants fall through to vanilla magic FMJ and the degraded boxes survive death hook trimming for player loot.
- No interaction with `at_jam`; different engine paths, compose freely.

---

## What the engine does and what we feed it

The architecture principle is to feed engine memory and state, not fight it. Per-system summary:

| System | Engine state we write | Engine APIs called |
|---|---|---|
| Hit Sharing | RELATION_REGISTRY personal goodwill, memory entry m_enabled, agent_member_manager m_combat_mask | `force_set_goodwill`, `enable_memory_object`, `register_in_combat` |
| Healing | NPC health field, bleeding field, `healing_charge` se_var | `change_health`, direct `bleeding =` write, `se_save_var` |
| Accuracy | Per-shot dispersion radius via callback return | (subscribes to `npc_shot_dispersion`) |
| Combat | NPC GOAP action (action_at_combat), Pattern B preconditions on action_combat_planner/action_danger_planner/xr_danger.actid/state_mgr+2/alife, set_dest_level_vertex_id, state_mgr.set_state, set_body_state, set_movement_type, set_sight, `m_sniper_fire_mode` flag | GOAP `add_evaluator`/`add_action`/`add_precondition` (evaid/actid 188200), `npc:best_cover`, `level.vertex_in_direction`, `npc:sniper_fire_mode`, `db.used_level_vertex_ids` reservation |
| Danger | NPC danger evaluator/action graft, `script_danger` per-id table for sound-source dispatch | Engine callbacks `npc_on_hear_callback`, `npc_on_death_callback`, GOAP planner graft (evaid/actid 188113) |
| Weapon Jam | Module-level function table on `xr_weapon_jam` | Lua function assignment (`xr_weapon_jam.GetConditionMisfireProbability = ...`) read by engine functor lookup at `Weapon.cpp:1781` |
| NPC Ammo | CWeapon `m_ammoType` field via `wpn:set_ammo_type(idx)` (re-keys `m_DefaultCartridge` for magic refill ballistics); per-NPC virtual ledger drives section selection; inventory boxes clamped to ledger via `obj:ammo_set_count` to defeat `try_advance_ammo` top-up; death hook trims boxes to MIN_SPARE for corpse loot survival | `npc:active_item`, `wpn:set_ammo_type`, `wpn:get_ammo_type`, `wpn:get_ammo_count_for_type`, `npc:best_enemy`, `npc:character_rank`, `npc:iterate_inventory`, `obj:ammo_get_count`, `obj:ammo_set_count` |

The engine then runs its own combat detection (property_enemy, m_combat_mask, agent_memory propagation) on the state we wrote. No system reimplements engine behavior; each one nudges engine state to produce the desired outcome.

---

## See also

- Task queue: `stalker-dev/doc/todo/todo-alifetactics-next.md`
- Brainstorm pool: `stalker-dev/doc/todo/todo-alifetactics-backlog.md`
- Engine PR queue: `stalker-dev/doc/todo/todo-demonized-exes.md`
- xlibs architecture: `stalker-mods/xlibs/doc/architecture.md`
- AlifePlus architecture: `stalker-mods/AlifePlus/doc/architecture.md`
- AlifeGuard architecture: `stalker-mods/AlifeGuard/doc/architecture.md`
- AlifeBalance architecture: `stalker-mods/AlifeBalance/doc/architecture.md`
