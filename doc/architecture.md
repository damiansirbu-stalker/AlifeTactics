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
| `at_combat.script` | feature | Pattern B planner takeover; 12 BEHAVIORS (ADVANCE, TAKE_COVER, FIRE_FROM_COVER, FIRE_HOLD, RETREAT, CLOSE_ASSAULT, SNIPE, ZOMBIE_SHAMBLE, FLANKING, RETREAT_AND_FIRE, FLEE, HOLD_STILL); per-faction lists (military / fanatic / merc / disorganized / coward / tactical default / zombie); SNIPE wires the engine `sniper_fire_mode` flag |
| `xr_danger.script` | feature | done (full-file override) |
| `at_jam.script` | feature | done (modded-exes xr_weapon_jam.GetConditionMisfireProbability override; suppresses script-injected NPC misfire) |
| `at_ammo.script` | feature | done (npc_on_net_spawn: picks highest-k_ap inventory ammo via xinventory.get_ammo_tier_map, calls wpn:set_ammo_type) |
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
│   │   ├── at_ammo.script                     # NPC ammo selection at spawn (NPC Ammo system)
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

Pattern B planner takeover per `doc/library/modding/goap-injection.md:427-447`. Single scheme owns vanilla `action_combat_planner` for a configurable share of NPCs.

The behavior catalog (`BEHAVIORS` table) defines each combat behavior as a row of `{ plan, applies(ctx) }`. Each NPC reads from a per-NPC list (`LIST_TACTICAL`, `LIST_AGGRESSOR`, or `LIST_RETREAT`) chosen by HP and weapon kind. The decide phase walks the list in order, returning the first behavior whose `applies(ctx)` matches the current situation. The behavior's `plan` (state, body, mvt, target) drives engine state writes.

### Product framing: inject not replace

- Single MCM slider `combat_share` (0-100%, default 100%) gates participation per NPC via stable hash `(npc:id() % 1000) < (share * 1000)`. 0% = pure vanilla / GAMMA / whatever modpack combat the user already has. 100% = full takeover.
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
4. Stable per-id hash: `(npc:id() % 1000) < (cfg.combat_share * 1000)` (default 1.0)
5. Pick-fail backoff window expired (linear extension past 3 consecutive failures: 3=10s, 4=20s, 5=30s; resets on successful pick)
6. `npc:best_enemy()` exists and is alive

No distance gate, no community gate, no HP gate, no weapon gate. Every stalker flows through `_allowed_for(ctx)` and per-behavior `applies(ctx)` triggers handle gating. AT controls combat at all ranges; CLOSE_ASSAULT's own `applies` enforces a stop-charging distance against the enemy (default 10m, tunable via `ai_tweaks/at_combat.ltx`).

Returns `(true, "in_range")` on full pass, else `(false, "<reason>")`. All filters dynamic — MCM toggles take effect on next eval. Hash stable per id.

### Action: 5-step tick

`action_at_combat:execute()` runs per scheduled-update tick (variable 1-50Hz per NPC per engine scheduler, typically 10-20Hz for active combat NPCs near actor):

**gather (`_gather_inputs`)** — one luabind round per tick. Reads `npc:position`, `npc:level_vertex_id`, `npc.health`, `best_enemy`, `enemy:position`, `npc:see(enemy)`, `enemy:see(npc)`, `level.high_cover_in_direction(npc_lvid, dir_to_enemy)` (used as `has_high_cover = value >= 0.2`, threshold per `axr_stalker_panic.script:399` precedent), computes `dist`, `dx`, `dz`. Cached for the tick. No re-reads in later steps.

**decide (`_decide_behavior`)** — early returns on `not ctx.enemy` (signals handoff). Otherwise `_allowed_for(ctx)` returns the per-faction list, and the decide loop returns the first behavior whose `applies(ctx)` is true. Returns the behavior NAME (string), or nil for handoff.

`_allowed_for(ctx)` is a pure lookup: `FACTION_LIST[ctx.community] or LIST_TACTICAL_DEFAULT`. No HP, weapon, or community-special-case branches — those gates moved into each behavior's `applies(ctx)`. See the per-faction lists table below.

**behavior hold** — gate between decide and apply. After a behavior change, the action commits to that name for `random(1500, 3500)` ms before the decide loop's output can flip it again. Prevents sub-second oscillation as NPCs move between adjacent vertices whose `high_cover_in_direction` values straddle the 0.2 threshold. `BYPASSES_HOLD[new]` (RETREAT, RETREAT_AND_FIRE, FLEE — all HP-driven) and nil-handoff skip the gate so HP emergency / terminate paths interrupt any in-flight commitment. `BYPASSES_HOLD_FROM[current]` (CLOSE_ASSAULT) lets distance-driven exits transition immediately to FIRE_HOLD when the NPC reaches the gate distance, instead of overshooting during the commit window. Implemented via `self._behavior_hold_until` on the action; reset in `:initialize`. `_should_hold(new, tg)` extracts the conditional out of the execute hot path.

**apply (`_apply_state`)** — `behavior.plan` returns `{state, body, mvt, target, sniper_aim?}`. Each engine write is gated by change detection:
- `if plan.state ~= self.last_state` then `state_mgr.set_state(npc, plan.state, ..., {fast_set=true})`
- `if plan.body  ~= self.last_body`  then `npc:set_body_state(move[plan.body])`
- `if plan.mvt   ~= self.last_mvt`   then `npc:set_movement_type(move[plan.mvt])`
- `if plan.sniper_aim ~= self.last_sniper_mode` then `npc:sniper_fire_mode(want)` — flips the engine flag at `ai_stalker.h:814` consumed in `ai_stalker_fire.cpp:193, 225, 239`

Zero engine writes when stable. The sniper_aim toggle drives the engine's actual sniper-aim path (m_head.target as aim direction instead of weapon LastFD). Cleared on finalize so vanilla planner resumes without the flag stuck on.

**movement (`_repick`)** — only fires if `tg > self._next_pick` OR another NPC stole our cover (`db.used_level_vertex_ids[target_lvid] ~= npc:id()`). Resolves target via `TARGET_RESOLVERS[plan.target]` (one of `cover_step_fwd`, `cover_nearest`, `step_away`, `charge_enemy`, `cover_flank`, `flee_far`, `hold`). Cover reservation via `_claim_cover` — if another NPC owns the lvid, our pick fails and we push `_next_pick` out 1.5-3.5s to throttle retry. After successful claim: release old lvid, claim new lvid, `set_dest_level_vertex_id(new_lvid)`, `_next_pick = tg + math_random(1500, 3500)`.

| Resolver | Search point | Cover preference | Notes |
|---|---|---|---|
| `cover_step_fwd` | npc + 15m toward enemy | yes (radius 15, dist 1-30 from enemy) | ADVANCE; open-terrain fallback to `vertex_in_direction` |
| `cover_nearest` | npc + bucket offset | yes (two-tier radius 10 then 30) | TAKE_COVER; matches vanilla `find_best_cover` |
| `step_away` | npc + 15m AWAY from enemy | yes | RETREAT, RETREAT_AND_FIRE; open-terrain fallback backward |
| `charge_enemy` | enemy lvid directly | none | CLOSE_ASSAULT, ZOMBIE_SHAMBLE; reservation skipped |
| `cover_flank` | npc + 20m perpendicular (per-id sign) + 5m forward | yes | FLANKING; perp direction `(-dz, dx)/dist` with sign from `id % 2`; open-terrain fallback to lateral vertex |
| `flee_far` | — | none | FLEE; pure `vertex_in_direction` backward 30m, no `best_cover` |
| `hold` | — | — | SNIPE, FIRE_FROM_COVER, FIRE_HOLD, HOLD_STILL; returns existing `target_lvid` or `npc_lvid` |

### Squad spread (lateral hash buckets)

Per-NPC lateral offset perpendicular to the npc->enemy vector spreads squad members across the engagement line. `inputs.bucket = npc:id() % SQUAD_LATERAL_BUCKETS` (5 buckets) is computed once in gather. `_lateral_offset(bucket, dx, dz, dist)` returns `(off_x, off_z)` rotated 90° from `(dx, dz)` and scaled by `(bucket - 2) * SQUAD_LATERAL_STEP_M` (4m), producing offsets in `{-8, -4, 0, +4, +8}` meters.

Applied to the `search_pos` passed to `npc:best_cover` in `cover_step_fwd`, `cover_nearest`, `step_away`. `hold` doesn't apply since it returns the existing target_lvid without a best_cover call.

The offset is stable per id, so the same NPC always lands in the same bucket across save/load. In-squad members fan out deterministically; without the spread, members starting within a few meters of each other run best_cover with near-identical search centers, the engine picks the same cover for the first, reservation forces subsequent members to adjacent lvids on the same cover object, and the squad piles up on one cover edge.

At `dist <= 0` the offset is zero (degenerate same-position case). Cost: 3 mul + 2 div + 2 sub per resolver call, no luabind.

**fire** — one `npc:set_sight(look.fire_point, ene_pos + 1.5y)` per tick. Engine fire dispatch runs from `mental_state = danger` + valid sight + LOS + active weapon. No explicit fire call needed.

### BEHAVIORS catalog

Data-driven. Each row: `name = { plan, applies(ctx), [sniper_aim] }`. Adding a behavior = add a row + add the name to the relevant list(s). No new function bodies.

| Behavior | State | Body | Mvt | Target | applies(ctx) | sniper_aim |
|---|---|---|---|---|---|---|
| ADVANCE | assault_fire | standing | run | cover_step_fwd | `dist > ADVANCE_DIST_M or not see` | — |
| TAKE_COVER | hide_na | crouch | run | cover_nearest | `see_me and not has_high_cover` | — |
| SNIPE | hide_sniper_fire | crouch | stand | hold | `weapon_kind == "w_sniper" and see` | **true** |
| FIRE_FROM_COVER | threat_fire | standing | stand | hold | `has_high_cover and see` | — |
| FIRE_HOLD | hide_fire | crouch | stand | hold | `see` | — |
| RETREAT | panic | standing | run | step_away | `hp_frac < RETREAT_HP_FRAC` | — |
| CLOSE_ASSAULT | assault_fire | standing | run | charge_enemy | `AGGRESSOR_KINDS[weapon_kind]` (pistol, shotgun, SMG, knife) AND `dist > 10` | — |
| ZOMBIE_SHAMBLE | raid_fire | standing | walk | charge_enemy | always | — |
| FLANKING | assault_fire | standing | run | cover_flank | `dist > ADVANCE_DIST_M and not see and has_high_cover` | — |
| RETREAT_AND_FIRE | sneak_fire | crouch | walk | step_away | `hp_frac < RETREAT_HP_FRAC and (id % 10) < 5` | — |
| FLEE | sprint | standing | run | flee_far | `hp_frac < RETREAT_HP_FRAC` | — |
| HOLD_STILL | hide_na | crouch | stand | hold | always | — |

body / mvt stored as string keys (`"standing"`, `"crouch"`, `"run"`, etc.), resolved at apply time via `move[name]`. Lets BEHAVIORS construct at module load before engine `move` enum is bound.

### Per-faction lists

Module-level constants. `_allowed_for(ctx)` returns the list for the NPC's community from `FACTION_LIST`; unknown communities fall back to `LIST_TACTICAL_DEFAULT`. No per-tick allocation. Order is priority — first applies-true wins.

| List | Order (priority) | Communities |
|---|---|---|
| `LIST_MILITARY` | RETREAT_AND_FIRE, SNIPE, FLANKING, FIRE_FROM_COVER, TAKE_COVER, FIRE_HOLD, ADVANCE | `army`, `army_npc`, `dolg`, `freedom`, `isg` |
| `LIST_FANATIC` | CLOSE_ASSAULT, SNIPE, FLANKING, FIRE_FROM_COVER, TAKE_COVER, FIRE_HOLD, ADVANCE | `monolith`, `greh`, `greh_npc` |
| `LIST_MERC` | RETREAT_AND_FIRE, RETREAT, SNIPE, FLANKING, FIRE_FROM_COVER, TAKE_COVER, FIRE_HOLD, ADVANCE | `killer` |
| `LIST_DISORGANIZED` | RETREAT_AND_FIRE, RETREAT, CLOSE_ASSAULT, ADVANCE, FIRE_FROM_COVER, FIRE_HOLD | `bandit`, `renegade` |
| `LIST_COWARD` | RETREAT_AND_FIRE, FLEE, FIRE_FROM_COVER, TAKE_COVER, FIRE_HOLD, HOLD_STILL | `ecolog`, `csky` |
| `LIST_TACTICAL_DEFAULT` (fallback) | RETREAT_AND_FIRE, RETREAT, SNIPE, ADVANCE, TAKE_COVER, FIRE_FROM_COVER, FIRE_HOLD | `stalker` (loner), any unmapped |
| `LIST_ZOMBIE` | ZOMBIE_SHAMBLE | `zombied` |

Faction signatures (what the player sees):

- **MILITARY** — cover-disciplined, sniping, flanks. Wounded → pulls back firing. Never panics, never charges. Half of low-HP NPCs fight on (RETREAT_AND_FIRE hash-miss, no other HP behavior in list).
- **FANATIC** — same skill as military, no retreat. Close weapon → charges always. Fights to death.
- **MERC** — military skillset + breaks. Panic RETREAT at low HP (military doesn't).
- **DISORGANIZED** — no sniping, no flanking, no cover-seek. Charge with shotguns, advance with rifles, panic when hurt.
- **COWARD** — never advance, never charge, never snipe (untrained, no long-range fire even with a sniper rifle). FLEE outright at low HP (weapon strapped, 30m sprint). HOLD_STILL when LOS lost (crouch in place).
- **TACTICAL_DEFAULT** — average. Mix of tactical + panic. No flank, no charge.
- **ZOMBIE** — mindless walk-and-fire.

HP-low (HP < `RETREAT_HP_FRAC`, default 0.25 from `ai_tweaks/at_combat.ltx`) traces per list:

| List | 50% (id-hash hit) | 50% (id-hash miss) |
|---|---|---|
| MILITARY | RETREAT_AND_FIRE | fall through → SNIPE/FLANKING/cover/ADVANCE (fight on) |
| FANATIC | (no HP-driven behavior in list — fall through immediately) | combat behaviors (fight on) |
| MERC | RETREAT_AND_FIRE | RETREAT (panic flee to cover) |
| DISORGANIZED | RETREAT_AND_FIRE | RETREAT |
| COWARD | RETREAT_AND_FIRE | FLEE (sprint, weapon down) |
| TACTICAL_DEFAULT | RETREAT_AND_FIRE | RETREAT |
| ZOMBIE | (no HP gate) | ZOMBIE_SHAMBLE (shamble until dead) |

The 50/50 split comes from RETREAT_AND_FIRE.applies = `hp_frac < RETREAT_HP_FRAC and (id % 10) < 5`. Same NPC always lands the same side across save/load.

### Sniper behavior: real engine sniper-aim

When SNIPE fires, `_apply_state` calls `npc:sniper_fire_mode(true)`. The engine flag at `ai_stalker.h:814` is consumed in `ai_stalker_fire.cpp:193, 225, 239`:

- When the flag is set, the engine swaps the aim direction from `weapon->get_LastFD()` (weapon barrel direction) to `movement().m_head.target.yaw/pitch` (target head direction).
- Practical effect: the NPC aims where the head IS GOING TO point (target position via state_mgr's `look_object`), not where the weapon currently happens to face. More precise aim, tracks the target rather than the barrel.

When SNIPE exits (transition to any other behavior or finalize), `npc:sniper_fire_mode(false)` is called so the engine flag doesn't persist when vanilla planner resumes.

The flag is the only engine-level sniper mechanism. `weapon="sniper_fire"` in the state def is treated identically to `weapon="fire"` by `state_mgr_weapon` (both pass the `unstrapped or fire or sniper_fire` evaluator the same way). The state-name choice is cosmetic; the engine flag is what changes behavior.

Replaces the old `at_stance.script` functor (dropped 2026-06-09 commit `591b6e0`) which forced crouch on sniper LookOut / HoldPosition. That functor was unreachable for in-share NPCs anyway because Pattern B blocks the `action_combat_planner` sub-tree where `body_state_combat_override` is called from.

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

When evaluator returns false (NPC died, enemy lost, hp dropped below wounded threshold, share toggled off, fail backoff window active), Pattern B preconditions no longer block vanilla. Vanilla planner resumes from its normal action set. Vanilla properties (`eWorldPropertyInCover`, `eWorldPropertyLookedOut`, etc.) may have stale values from before our takeover but self-correct on first `best_cover_changed` event per `stalker_combat_planner.cpp:60-64`.

### What we don't touch

- `xr_combat.script`, `xr_combat_camper.script`, `xr_combat_monolith.script`, `xr_combat_zombied.script` — vanilla sub-schemes unchanged
- `xr_combat.set_combat_type` / `motivator_binder:update` 500ms loop — never write `script_combat_type`
- `xr_cover.script`, `xr_smartcover.script`, `xr_combat_ignore.script`, `visual_memory_manager.script` — vanilla files unchanged (contrast with RCAI which overrides all 6)
- Smart terrain configs, job XML, LTX — eligibility is runtime per-NPC, not pre-assigned

### MCM

| Key | Type | Default | Effect |
|---|---|---|---|
| `combat_enabled` | check | true | Master toggle. Effective on next eval tick, no restart |
| `combat_share` | track 0.0-1.0 step 0.05 | 1.0 | Fraction of eligible NPCs using our combat AI. 0 = pure vanilla. 1 = full takeover. Stable per-id hash |

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

`at_ammo.script` picks the highest armor-piercing ammo type in an NPC's inventory at spawn and sets it on the active weapon. The engine cascade then drives natural consumption: real AP first, then degraded variants, then FMJ, then unlimited fallback if the trader flag is set.

### Ranking

Uses xlibs `xinventory.get_ammo_tier_map(wpn_sec, 3)` (`xinventory.script:312-359`). Sorts `ammo_class` sections by `k_ap` ascending (cost tiebreaker), buckets into 3 tiers via `floor((i-1) * N / count) + 1`. Tier 3 = highest `k_ap` (clean AP). Tier 2 = mid (typically `_bad` variants of AP, or clean FMJ for non-AP-carrying weapons). Tier 1 = lowest (`_verybad` variants, basic FMJ in mixed lists). Cached per `(weapon_sec, n_tiers)`.

Same primitive consumed by `AlifePlus/ap_ext_trade.script:64` (`_buy_ammo_tier`) for trader ammo purchases. Shared ranking semantics across the mod family.

### Pick

`npc_on_net_spawn` handler:

1. MCM gate (`ammo_enabled`).
2. `IsStalker(npc)` filter; mutants skipped.
3. `wpn = npc:active_item()`, `IsWeapon(wpn)` filter.
4. `accepted = xinventory.get_ammo_classes(wpn:section())` — set form `{section → true}`.
5. `tier_map = xinventory.get_ammo_tier_map(wpn:section(), 3)`.
6. Walk `npc:iterate_inventory`, track section with highest `tier_map[sec]` that's also in `accepted`.
7. Map best section to its index in the ordered `xinventory.get_ammo_sections` list (1-based Lua → 0-based engine: `i - 1`).
8. If `idx ~= wpn:get_ammo_type()`, call `wpn:set_ammo_type(idx)`.

### Engine cascade

`wpn:set_ammo_type(idx)` writes `m_ammoType = idx` (`Weapon.h:1100`). The reload pipeline reads it:

- **AP from inventory:** `WeaponMagazined.cpp:298-347` `TryReload` line 314: `m_pInventory->GetAny(m_ammoTypes[m_ammoType])`. Line 323: triggers `SwitchState(eReload)`. `ReloadMagazine` line 525-531 pulls real AP boxes; line 562+ decrements per round.
- **Fallback through tiers:** When AP gone, line 329-340 walks `m_ammoTypes` for any inventory match, defers via `m_set_next_ammoType_on_reload`. Same again at line 533-545 inside `ReloadMagazine`. `m_ammoType` ratchets to whichever type still has rounds.
- **Unlimited fallback:** All inventory drained AND `unlimited_ammo()` true (trader flag set per-NPC at `object_handler.cpp:78`). Line 520 skips the inventory search block. Line 559-560: `m_DefaultCartridge.Load(m_ammoTypes[m_ammoType], ...)` rebuilds the magic cartridge from current `m_ammoType` (whichever tier the NPC was last firing from). Mag refills from it.

The cheap-fallback at the end happens automatically: by the time inventory is fully drained, `m_ammoType` has ratcheted down to the lowest tier that had any rounds (typically FMJ for most NPCs since loadouts are FMJ-heavy). For NPCs spawning with only AP, the unlimited fallback stays on AP — we do not patch this case.

### Verification

Empirically verified by `tmp/probe_ammo.script`: PMM-carrying NPC, `wpn_pmm` `ammo_class` has 9 entries (FMJ/PMM/AP × clean/bad/verybad). Before: `get_ammo_type=3` (clean PMM). After `set_ammo_type(0)`: `get_ammo_type=0`. After 3 seconds and multiple NPC update ticks: `get_ammo_type=0` still. The engine binding holds; no internal logic resets the value.

### MCM

| Key | Type | Default | Effect |
|---|---|---|---|
| `ammo_enabled` | check | true | On: pick best inventory ammo at NPC spawn. Off: no spawn-time action. Already-set ammo types on existing NPCs are untouched either way (one-shot per spawn). |

### Scope and limits

- Fires once per spawn. Does not re-pick if an NPC loots better ammo mid-game.
- Active weapon only. If NPC switches weapons later, the new weapon retains its vanilla default ammo type.
- No interaction with the `xr_weapon_jam` patch in `at_jam`; they target different engine paths and compose freely.

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
| NPC Ammo | CWeapon `m_ammoType` field via `wpn:set_ammo_type(idx)` | `npc:active_item`, `npc:iterate_inventory`, `wpn:set_ammo_type`, `wpn:get_ammo_type` plus xlibs `xinventory.get_ammo_tier_map` / `get_ammo_sections` / `get_ammo_classes` |

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
