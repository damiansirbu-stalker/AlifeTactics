# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Built around a shared per-squad memory DTO that user-facing systems consume. Four user-facing systems sit on top of one internal core.

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
| `at_stance.script` | feature | done |
| `configs/ai_tweaks/mod_xr_eat_medkit_at.ltx` | data | done |

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
│   │   │   └── mod_xr_eat_medkit_at.ltx       # DLTX overlay: vanilla medkit/bandage lists
│   │   └── text/eng/ui_st_mcm_at.xml          # English MCM strings
│   ├── scripts/
│   │   ├── _at_deps.script                    # dependency gate
│   │   ├── at_mcm.script                      # MCM configuration
│   │   ├── at_hitresponse.script              # Hit Sharing system
│   │   ├── at_health.script                   # Healing system
│   │   ├── at_accuracy.script                 # Accuracy system
│   │   ├── at_stance.script                   # Stance Switch system
│   │   └── at_test.script                     # console test commands
│   └── textures/
│       └── at_mcm_banner.dds                  # MCM banner
├── LICENSE
└── README.md
```

Namespace: `at_*` (parallel to `ap_*` for AlifePlus, `ag_*` for AlifeGuard, `x*` for xlibs).

---

## Four user-facing systems

Each system has its own file, its own MCM tab, and one master toggle.

| System | File | MCM Tab | Master toggle |
|---|---|---|---|
| Hit Sharing | `at_hitresponse.script` | Hit Sharing | `hit_share_enabled` |
| Healing | `at_health.script` | Healing | `healing_enabled` |
| Accuracy | `at_accuracy.script` | Accuracy | `accuracy_enabled` |
| Stance Switch | `at_stance.script` | Stance Switch | `stance_enabled` |

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
| Limping | `npc_on_update` per-NPC throttled to 1s; eligibility = `health < threshold` AND `mental_state() == anim.free` AND `body_state() == move.standing` AND not zombied AND not wounded AND not in smart_cover. Re-arms every 5s | `dmg_norm_torso_1_idle_0` (torso overlay; layers over engine-driven locomotion, legs stay attached to ground) |
| Heal anim | First tick of `_patched_heal_hp` / `_patched_heal_bleed` (`left == 15`); engine drains the queue naturally after | `dmg_norm_torso_11_attack_0` (medkit) / `norm_torso_12_attack_0` (bandage) |

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

## Stance Switch

Hooks the modded-exe `_G.CAI_Stalker__CombatSetBodyState(npc, wo, body_state)` functor at `stalker_movement_manager_base_inline.h:51-59`. Returns `eBodyStateCrouch` for sniper and launcher carriers when the engine selected Stand for one of the override operators.

### Engine call sites

The functor fires from `body_state_combat_override` calls in `stalker_combat_actions.cpp`. Enumerated EWorldOperators that reach the functor: `{12, 14, 17, 20, 21, 22, 23, 25, 27, 28, 39}` (GetItemToKill, MakeItemKilling, GetReadyToKill, RetreatFromEnemy, TakeCover, LookOut, HoldPosition, DetourEnemy, HideFromGrenade, SuddenAttack, ThrowGrenade).

### Override set

```
OVERRIDE_OPS = {
    [OP_LOOKOUT]       = true,  -- 22
    [OP_HOLD_POSITION] = true,  -- 23
}
```

These are the two static-cover firing operators that actually call `body_state_combat_override` in `stalker_combat_actions.cpp`. Once the functor returns crouch on a LookOut or HoldPosition tick, subsequent KillEnemy, WaitInCover, and HoldAmbushLocation actions inherit the crouched body_state without re-entering the functor.

### Composition chain

`_prev_functor` captures any prior `_G.CAI_Stalker__CombatSetBodyState` installer at `on_game_start`. The chain composes with any other mod touching this seam instead of silently overriding.

### Weapon-kind gate

Reads `kind` from the NPC's `active_item():section()` via `ini_sys:r_string_ex(section, "kind")`. Crouches when `kind == "w_sniper"` or `kind == "w_launcher"`. All other weapon kinds (including `w_rifle`, `w_pistol`, `w_shotgun`, `w_smg`, `w_knife`) and NPCs without an active item pass through with the engine's chosen body_state. No squad-memory dependency, no role taxonomy.

---

## What the engine does and what we feed it

The architecture principle is to feed engine memory and state, not fight it. Per-system summary:

| System | Engine state we write | Engine APIs called |
|---|---|---|
| Hit Sharing | RELATION_REGISTRY personal goodwill, memory entry m_enabled, agent_member_manager m_combat_mask | `force_set_goodwill`, `enable_memory_object`, `register_in_combat` |
| Healing | NPC health field, bleeding field, `healing_charge` se_var | `change_health`, direct `bleeding =` write, `se_save_var` |
| Accuracy | Per-shot dispersion radius via callback return | (subscribes to `npc_shot_dispersion`) |
| Stance Switch | NPC body_state via functor return | (functor at `_G.CAI_Stalker__CombatSetBodyState`) |

The engine then runs its own combat detection (property_enemy, m_combat_mask, agent_memory propagation) on the state we wrote. No system reimplements engine behavior; each one nudges engine state to produce the desired outcome.

---

## Logging

Each module owns its own `xlog.get_logger("AT.X", { outfile = "alifetactics.log" })` facade. All write to `alifetactics.log` with distinct prefixes:

- `AT.MCM`: MCM configuration
- `AT.HIT`: Hit Sharing
- `AT.HEALTH`: Healing
- `AT.ACC`: Accuracy
- `AT.STANCE`: Stance Switch
- `AT.TEST`: console test harness

MCM `log_level` (ERROR/WARN/INFO/DEBUG) controls verbosity. Each module subscribes to `on_option_change` and `mcm_option_restore_default` to refresh its level and derived `_dbg` flag.

`xprofiler.new_if(_dbg)` wraps profile-relevant code paths. Null singleton when DEBUG off (zero luabind); real `profile_timer` when on. All `log.debug` calls gated by `if _dbg then` so format strings are never built when off.

### Key debug events

- `[HIT]`: hit handler reject reasons or refreshed-disclosure cases
- `[DISCLOSURE]`: full-squad disclosure with member count
- `[INHERIT]`: new squad member inherits disclosed shooters on spawn
- `[REDISCLOSE]`: previously-offline shooter came back online, replayed disclosure to tracked squads
- `[DECAY]`: decay tick pruned entries past retention threshold
- `[UNREGISTER]`: disclosure cleanup on entity despawn
- `[HEAL]`: `seq_start`, `seq_gesture`, `seq_end`, `seq_abort`, `hp_tick`, `complete`, `bleed_tick`, `bleed_complete`
- `[CHARGE]`: healing charge rolls per NPC
- `[LIMP]`: limp gained / lost / anim queued per NPC
- `[PATCH]`: install messages for xr_eat_medkit patches
- `[ACC]`: per-shot accuracy calculation
- `[STANCE]`: body_state override fires
- `[SPAWN]`: at_test squad spawn
- `[DUMP]`: at_test disclosed-squad dump

---

## Test infrastructure

`at_test.script` provides console commands invoked via `run_string at_test.<func>()`:

- `at_spawn()`: spawn `bandit_sim_squad_novice` 50m ahead
- `at_spawn_veteran()`: `bandit_sim_squad_veteran` 50m ahead
- `at_spawn_master()`: `bandit_sim_squad_alpha` 50m ahead
- `at_spawn_far()`: novice 100m ahead
- `at_dump()`: log all disclosed-squad records (squad_id, shooter count)

---

## See also

- Task queue: `stalker-dev/doc/todo/todo-alifetactics-next.md`
- Brainstorm pool: `stalker-dev/doc/todo/todo-alifetactics-backlog.md`
- Engine PR queue: `stalker-dev/doc/todo/todo-demonized-exes.md`
- xlibs architecture: `stalker-mods/xlibs/doc/architecture.md`
- AlifePlus architecture: `stalker-mods/AlifePlus/doc/architecture.md`
- AlifeGuard architecture: `stalker-mods/AlifeGuard/doc/architecture.md`
- AlifeBalance architecture: `stalker-mods/AlifeBalance/doc/architecture.md`
