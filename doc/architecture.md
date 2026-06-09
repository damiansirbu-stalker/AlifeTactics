# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Independent user-facing systems: a hit-share force-disclosure, a self-heal data + animation layer, a per-rank weapon accuracy curve, a long-range rifle stance crouch (`kind=w_sniper` LTX class: DMRs, battle rifles, bolt-actions), a full-file xr_danger override with bug fixes and three toggleable improvements, and a Pattern B planner takeover that injects an alternative combat AI on a configurable share of NPCs (slider, default 50% — coexists with vanilla / GAMMA / AI Rework / RCAI / Useful Idiots / Mora, zero vanilla file overrides). No shared substrate.

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
| `at_combat.script` | feature | Pattern B planner takeover; ADVANCE and TAKE_COVER wired; RETREAT / SNIPE / CLOSE_ASSAULT / FIRE_FROM_COVER / FIRE_HOLD pending |
| `at_advance.script.bak` | feature | superseded by at_combat (preserved as .bak, doesn't auto-load) |
| `xr_danger.script` | feature | done (full-file override) |
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
│   │   ├── at_stance.script                   # Stance Switch system
│   │   ├── at_combat.script                   # Combat system (Pattern B planner takeover)
│   │   ├── xr_danger.script                   # full-file override (Danger system)
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
| Stance Switch | `at_stance.script` | Stance Switch | `stance_enabled` |
| Combat | `at_combat.script` | Combat | `combat_enabled` |
| Danger | `xr_danger.script` (full-file override) | Danger | bug fixes always-on; three toggleable improvements |

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

Hooks the modded-exe `_G.CAI_Stalker__CombatSetBodyState(npc, wo, body_state)` functor at `stalker_movement_manager_base_inline.h:51-59`. Returns `eBodyStateCrouch` for long-range rifle carriers (LTX `kind=w_sniper`: covers DMRs, battle rifles, bolt-action rifles) when the engine selected Stand for one of the override operators. Engine independently selects crouch for any weapon kind at low cover via `level.high_cover_in_direction` reads; our functor passes that through unchanged.

### Engine call sites

The functor fires from `body_state_combat_override` calls in `stalker_combat_actions.cpp`. Enumerated EWorldOperators that reach the functor: `{12, 14, 17, 20, 21, 22, 23, 25, 27, 28, 39}` (GetItemToKill, MakeItemKilling, GetReadyToKill, RetreatFromEnemy, TakeCover, LookOut, HoldPosition, DetourEnemy, HideFromGrenade, SuddenAttack, ThrowGrenade).

### Override set

```
OVERRIDE_OPS = {
    [OP_LOOKOUT]       = true,  -- 22
    [OP_HOLD_POSITION] = true,  -- 23
}
```

These are the two static-cover firing operators where precision-rifle doctrine calls for crouched fire from cover. Once the functor returns crouch on a LookOut or HoldPosition tick, subsequent KillEnemy, WaitInCover, and HoldAmbushLocation actions inherit the crouched body_state without re-entering the functor.

### Composition chain

`_prev_functor` captures any prior `_G.CAI_Stalker__CombatSetBodyState` installer at `on_game_start`. The chain composes with any other mod touching this seam instead of silently overriding.

### Weapon-kind gate

Reads `kind` from the NPC's `active_item():section()` via `ini_sys:r_string_ex(section, "kind")`. Crouches when `kind == "w_sniper"`. The `w_sniper` LTX classifier is broader than the colloquial "sniper": it covers long-range rifles including DMRs (SVD, SVU, SR25), battle rifles (SVT40, G43), bolt-actions (Mosin, K98), service carbines (SKS), and dedicated sniper rifles (M82, VSSK, M24, SV98, L96A1, TRG, WA2000, M98B, Sig550, Remington700, Gauss). 19 vanilla files plus ~134 GAMMA-stack additions. Other weapon kinds (`w_rifle`, `w_pistol`, `w_shotgun`, `w_smg`, `w_explosive`, `w_knife`) and NPCs without an active item pass through with the engine's chosen body_state. No squad-memory dependency, no role taxonomy. The `w_launcher` kind does not exist in any vanilla or GAMMA LTX file; RPG-7 and GP-25 are `w_explosive`.

Vanilla playtest data validates the gate: 55 FLIP events across {Mosin 46, SKS 5, SVT40 2, SV98 2}, all `weapon_class=sniper_rifle`, scope_status 1 or 2. Engine independently selected crouch for 12+ other weapon kinds (BM-16 shotgun, AEK rifle, PPSh-41 SMG, Fort-500 pistol, etc) at low cover, passed through unchanged.

---

## Combat

Pattern B planner takeover per `doc/library/modding/goap-injection.md:427-447`. Single scheme owns vanilla `action_combat_planner` for a configurable share of NPCs and runs an internal mode state machine. Replaces at_advance entirely. ADVANCE and TAKE_COVER are wired in the decide tree; RETREAT, SNIPE, CLOSE_ASSAULT, FIRE_FROM_COVER, FIRE_HOLD are declared as MODE_PLANS rows but the decide tree does not yet return them.

### Product framing: inject not replace

- Single MCM slider `combat_share` (0-100%, default 50%) gates participation per NPC via stable hash `(npc:id() % 1000) < (share * 1000)`. 0% = pure vanilla / GAMMA / whatever modpack combat the user already has. 100% = full takeover.
- Zero vanilla file overrides beyond xr_danger (separate system). Pattern B preconditions land on top of whatever the user's stack already has.
- Coexists with vanilla, GAMMA, AI Rework, ReDone Combat AI, Useful Idiots, Mora. Their `xr_combat` / `xr_combat_camper` / condlist `script_combat_type` assignments still run; for NPCs outside our share they drive vanilla behavior. For NPCs in our share, vanilla planner is blocked via precondition; their condlist writes still execute every 500ms via `motivator_binder:update` but have no effect because vanilla planner is not running.
- Per-NPC hash deterministic across saves. Same NPC always lands the same side. In-squad A/B testing possible.

### Why Pattern B over the sub-scheme route

SQUAD_CAMPER (commit 3fac38f, deferred at d613386) routed through vanilla `xr_combat_camper` via `script_combat_type` condlist. Two races killed it. **Outer**: vanilla `action_combat_planner` only gated by `xr_evaluators_id.script_combat = false`; predicate momentarily flipping nil let vanilla run one tick of `LookOut`/`HoldPosition`, interrupting state_mgr. **Inner**: vanilla `xr_combat_camper` ships two actions (`action_shoot`, `action_look_around`) that toggle on LOS, each calling `state_mgr.set_state` with different args; LOS flips churned state_mgr inside the sub-scheme.

Pattern B blocks vanilla `action_combat_planner` via precondition `world_property(EVAL_ID, false)`, NOT a flippable gate. The planner re-evaluates preconditions when it re-plans; our eval stays stable across the engagement. One action under our control means no internal toggle. State_mgr is touched only on actual mode transitions, never on every tick.

### Install

Install runs at `npc_on_net_spawn` for every stalker. Adds evaluator + action + Pattern B preconditions to that NPC's `motivation_action_manager`. The `_installed[id]` sentinel prevents duplicate registration on net_spawn re-fires; cleared in `server_entity_on_unregister`. Mutants skipped (`IsStalker` filter) — they have no `action_combat_planner`.

Pattern B preconditions block five vanilla actions when our eval returns true: `stalker_ids.action_combat_planner`, `stalker_ids.action_danger_planner`, `xr_danger.actid`, `xr_actions_id.state_mgr + 2`, `xr_actions_id.alife`. Match `axr_stalker_panic.script:521`.

No per-NPC eligibility flag in storage. Eligibility is decided entirely by the evaluator.

### Evaluator (`_decide_eval`)

Fast-fail chain, in order:

1. `cfg.combat_enabled` master toggle
2. `npc:alive()`
3. `IsWounded(npc)`
4. `character_community(npc)` not "zombied", not "monolith" (those ship their own sub-schemes)
5. Stable per-id hash: `(npc:id() % 1000) < (cfg.combat_share * 1000)` (default 0.5)
6. Pick-fail backoff window expired (set by `_repick` after 3 consecutive failures, 10s backoff)
7. `npc:best_enemy()` exists and is alive
8. Distance to enemy >= `cfg.min_dist` meters (default 8m)

Returns `(true, "in_range")` on full pass, else `(false, "<reason>")`. All filters dynamic — MCM toggles take effect on next eval. Hash stable per id.

### Action: 5-step tick

`action_at_combat:execute()` runs per scheduled-update tick (variable 1-50Hz per NPC per engine scheduler, typically 10-20Hz for active combat NPCs near actor):

**gather (`_gather_inputs`)** — one luabind round per tick. Reads `npc:position`, `npc:level_vertex_id`, `npc.health`, `best_enemy`, `enemy:position`, `npc:see(enemy)`, `enemy:see(npc)`, `level.high_cover_in_direction(npc_lvid, dir_to_enemy)` (used as `has_high_cover = value >= 0.2`, threshold per `axr_stalker_panic.script:399` precedent), computes `dist`, `dx`, `dz`. Cached for the tick. No re-reads in later steps.

**decide (`_decide_mode`)** — pure function returning a mode constant. Flat priority tree, top-down, first match wins, max 2 nesting. Current implementation: `if not enemy then NONE; if dist < min_dist then NONE; if see_me and not has_high_cover then TAKE_COVER; else ADVANCE`. Other branches not yet wired.

**mode hold** — gate between decide and apply. After a mode change, the action commits to that mode for `random(1500, 3500)` ms before the decide tree's output can flip it again. Prevents sub-second oscillation as NPCs move between adjacent vertices whose `high_cover_in_direction` values straddle the 0.2 threshold (advance ↔ take_cover flap observed before the gate landed, eliminated after). MODE_NONE bypasses so terminate paths (no enemy / too close) run unchanged. Implemented via `self._mode_hold_until` on the action; reset in `:initialize`.

**apply (`_apply_state`)** — `MODE_PLANS[mode]` returns `{state, body, mvt, target_kind}`. Each engine write is gated by change detection:
- `if plan.state ~= self.last_state` then `state_mgr.set_state(npc, plan.state, ..., {fast_set=true})`
- `if plan.body  ~= self.last_body`  then `npc:set_body_state(move[plan.body])`
- `if plan.mvt   ~= self.last_mvt`   then `npc:set_movement_type(move[plan.mvt])`

Zero engine writes when stable. Avoids the state_mgr churn that killed SQUAD_CAMPER's vanilla sub-scheme.

**movement (`_repick`)** — only fires if `tg > self._next_pick` OR another NPC stole our cover (`db.used_level_vertex_ids[target_lvid] ~= npc:id()`). Resolves target via `TARGET_RESOLVERS[plan.target_kind]` (currently `cover_step_fwd`, `cover_nearest`, and `hold`). Cover reservation via `_claim_cover` — if another NPC owns the lvid, our pick fails and we push `_next_pick` out 1.5-3.5s to throttle retry. After successful claim: release old lvid, claim new lvid, `set_dest_level_vertex_id(new_lvid)`, `_next_pick = tg + math_random(1500, 3500)`. Same as RCAI's `__keep_point_until` adapted. `cover_nearest` runs a two-tier `npc:best_cover(npc_pos, ene_pos, 10|30, 1, 20)` matching vanilla `find_best_cover` (ai_stalker_cover.cpp:141, 150) — radius 10m then 30m, non-sniper enemy-distance defaults.

**fire** — one `npc:set_sight(look.fire_point, ene_pos + 1.5y)` per tick. Engine fire dispatch runs from `mental_state = danger` + valid sight + LOS + active weapon. No explicit fire call needed.

### MODE_PLANS table

Data-driven. Each mode maps to a plan. Adding a new behavior = add a constant + a row + a decide-tree branch. No new function bodies.

| Mode | State | Body | Mvt | Target |
|---|---|---|---|---|
| RETREAT | panic_in_threat | standing | run | step_away |
| SNIPE | hide_sniper_fire | crouch | stand | hold |
| TAKE_COVER | hide_na | crouch | run | cover_nearest |
| ADVANCE | assault_fire | standing | run | cover_step_fwd |
| CLOSE_ASSAULT | threat_fire | standing | walk | hold |
| FIRE_FROM_COVER | threat_fire | standing | stand | hold |
| FIRE_HOLD | hide_fire | crouch | stand | hold |

body/mvt stored as string keys (`"standing"`, `"crouch"`, `"run"`, etc.), resolved at apply time via `move[name]`. Lets MODE_PLANS construct at module load before engine `move` enum is bound.

### Cover selection: the architectural lever

`npc:best_cover(search_pos, enemy_pos, radius, min_dist, max_dist)` is the same engine primitive vanilla `CStalkerActionTakeCover::execute` uses internally (`stalker_combat_actions.cpp:589`, exposed to Lua via `script_game_object3.cpp:66-79`). The function searches the precomputed cover graph (`m_covers` quadtree) for a cover near `search_pos`, scored by how well it hides from `enemy_pos`.

**Critical insight: `search_pos` is OUR architectural lever, not the NPC's position.**

`npc:best_cover` doesn't care where the NPC currently stands. It finds a cover near whatever search center we pass. The NPC then runs to the lvid we picked. So the behavior the NPC exhibits is determined by where WE point the search.

Phase 1 `cover_step_fwd` resolver computes a step point 15m from NPC toward enemy and passes THAT as `search_pos`. Result: cover is found near a position 15m ahead of NPC. NPC advances 15m, takes cover. Each re-pick steps another 15m forward, progressively closing.

If we passed `npc:position()` as `search_pos`, we'd search near the NPC. NPC would find cover near where they already are, take it, stay there. That's camper behavior.

If we passed `enemy:position()` as `search_pos`, we'd search near the enemy. NPC would charge toward enemy's position to take cover at point-blank. That's close-assault.

If we passed `npc:position() + ADVANCE_STEP * -dir_to_enemy`, we'd step backward. That's retreat-with-cover.

So the same engine API drives every mode by varying `search_pos`. The `TARGET_RESOLVERS` table is data-driven for this reason.

**Cover graph quirks** (cover_manager_inline.h:73-81): the evaluator has inertia. Repeated `best_cover` calls with `search_pos` within `3 * radius` of the previously selected cover tend to return the same lvid. This is fine for us — `_next_pick` throttle means consecutive calls happen 1.5-3.5s apart with the NPC having moved, so `search_pos` shifts enough to escape inertia.

We do NOT replicate vanilla's full `cover_evaluator` weighting (which iterates multiple candidates considering exposed angles, line-of-fire to OTHER enemies in the agent_manager, etc.). Simpler heuristic, slightly worse cover quality, deterministic behavior. Acceptable tradeoff.

### Cover reservation (`db.used_level_vertex_ids`)

Shared global table; vanilla TakeCover and RCAI also write to it. Each NPC marks their chosen lvid: `db.used_level_vertex_ids[lvid] = npc:id()`. When picking, `_claim_cover` checks ownership — if another id owns it, claim fails. NPC retries 1.5-3.5s later via `_next_pick`. Prevents multi-NPC convergence on one lvid (the bug that produced the 300894-cluster in at_advance testing).

`_release_cover(lvid, id)` clears the table when we re-pick or finalize. `server_entity_on_unregister` clears on entity despawn.

### Open-terrain fallback

If `best_cover` returns nothing accessible at the step point (terrain dead zone, cover graph sparse), we fall back to `level.vertex_in_direction(npc_lvid, dir, ADVANCE_STEP_M)` for any accessible vertex in that direction. NPC moves in the open instead of standing still. Pattern: `axr_stalker_panic.script:282`.

If both fail 3x consecutively, `_on_pick_fail` sets `_fail_until[id] = tg + 10000`. Evaluator returns false with reason "fail_backoff" until the window expires. Vanilla planner resumes for that NPC.

### Handoff to vanilla

When evaluator returns false (NPC died, enemy lost, NPC closed within distance floor, hp dropped below wounded threshold, share toggled off, fail backoff window active), Pattern B preconditions no longer block vanilla. Vanilla planner resumes from its normal action set. Vanilla properties (`eWorldPropertyInCover`, `eWorldPropertyLookedOut`, etc.) may have stale values from before our takeover but self-correct on first `best_cover_changed` event per `stalker_combat_planner.cpp:60-64`.

### What we don't touch

- `xr_combat.script`, `xr_combat_camper.script`, `xr_combat_monolith.script`, `xr_combat_zombied.script` — vanilla sub-schemes unchanged
- `xr_combat.set_combat_type` / `motivator_binder:update` 500ms loop — never write `script_combat_type`
- `xr_cover.script`, `xr_smartcover.script`, `xr_combat_ignore.script`, `visual_memory_manager.script` — vanilla files unchanged (contrast with RCAI which overrides all 6)
- Smart terrain configs, job XML, LTX — eligibility is runtime per-NPC, not pre-assigned

### MCM

| Key | Type | Default | Effect |
|---|---|---|---|
| `combat_enabled` | check | true | Master toggle. Effective on next eval tick, no restart |
| `combat_share` | track 0.0-1.0 step 0.05 | 0.5 | Fraction of eligible NPCs using our combat AI. 0 = pure vanilla. 1 = full takeover. Stable per-id hash |
| `min_dist` | track 5-15 step 1 | 8 | Close-combat handoff distance. When owned NPC closes within this, eval returns false and vanilla resumes |

Additional knobs (`retreat_hp`, `advance_dist`) pending the other modes.

---

## Danger

Full-file override of vanilla `xr_danger.script` (Alundaio). Six vanilla bug fixes always-on. Three toggleable improvements behind MCM. Paired LTX (`configs/ai_tweaks/xr_danger.ltx`) with weather-conditional distances and actor-source variant tables.

### Vanilla bugs fixed (always-on)

1. `bd_types` name collision: three perceive-type names overwrite enum values, causing three danger categories to read wrong config sections.
2. `get_danger_time` crashes on mutant corpse: vanilla calls `corpse_object:death_time()` without `IsStalker` guard; trader interface absent on mutants.
3. `eval_danger` nil-NPC guard missing: vanilla crashes when called on a torn-down NPC reference.
4. `eval_danger` non-numeric `danger_time` check missing: vanilla type-asserts on bad return.
5. `danger_intertion_time` condlist param typo: vanilla reads `danger_intertion_time` while LTX section is `[danger_inertion]`; entire actor/distance condlist evaluation was a no-op in vanilla.
6. `npc_on_hit_callback` referenced undefined `who_id` variable: vanilla wrote nil shooter id into `script_danger`. Vanilla callback unregistered entirely; danger pipeline now driven by `npc_on_hear_callback` and `npc_on_death_callback`.

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

## What the engine does and what we feed it

The architecture principle is to feed engine memory and state, not fight it. Per-system summary:

| System | Engine state we write | Engine APIs called |
|---|---|---|
| Hit Sharing | RELATION_REGISTRY personal goodwill, memory entry m_enabled, agent_member_manager m_combat_mask | `force_set_goodwill`, `enable_memory_object`, `register_in_combat` |
| Healing | NPC health field, bleeding field, `healing_charge` se_var | `change_health`, direct `bleeding =` write, `se_save_var` |
| Accuracy | Per-shot dispersion radius via callback return | (subscribes to `npc_shot_dispersion`) |
| Stance Switch | NPC body_state via functor return | (functor at `_G.CAI_Stalker__CombatSetBodyState`) |
| Combat | NPC GOAP action (action_at_combat), Pattern B preconditions on action_combat_planner/action_danger_planner/xr_danger.actid/state_mgr+2/alife, set_dest_level_vertex_id, state_mgr.set_state, set_body_state, set_movement_type, set_sight | GOAP `add_evaluator`/`add_action`/`add_precondition` (evaid/actid 188200), `npc:best_cover`, `level.vertex_in_direction`, `db.used_level_vertex_ids` reservation |
| Danger | NPC danger evaluator/action graft, `script_danger` per-id table for sound-source dispatch | Engine callbacks `npc_on_hear_callback`, `npc_on_death_callback`, GOAP planner graft (evaid/actid 188113) |

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
