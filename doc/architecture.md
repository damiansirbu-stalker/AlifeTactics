# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Independent user-facing systems: a hit-share force-disclosure, a self-heal data + animation layer, a per-rank weapon accuracy curve, a full-file xr_danger override with bug fixes and three toggleable improvements, and a Pattern B planner takeover that injects an alternative combat AI on a configurable share of NPCs (the takeover overrides zero vanilla combat files, so it coexists with vanilla / GAMMA / AI Rework / RCAI / Useful Idiots / Mora). No shared substrate.

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
| `at_combat.script` | feature | done (Pattern B planner takeover; reactive agent over a turret baseline; data-driven maneuver catalog and faction palette) |
| `xr_danger.script` | feature | done (full-file override) |
| `at_jam.script` | feature | done (modded-exes xr_weapon_jam.GetConditionMisfireProbability override; suppresses script-injected NPC misfire) |
| `at_ammo.script` | feature | done (per-NPC virtual ledger drives AP-then-FMJ selection; inventory and corpse boxes clamped to the ledger; veteran-rank gate) |
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
│   │   ├── alifetactics/
│   │   │   ├── at_combat_config.ltx           # Combat numeric tunables
│   │   │   ├── at_combat_doctrine.ltx         # Combat maneuver catalog
│   │   │   └── at_ammo.ltx                    # NPC Ammo tunables
│   │   └── text/eng/ui_st_mcm_at.xml          # English MCM strings
│   ├── scripts/
│   │   ├── _at_deps.script                    # dependency gate
│   │   ├── at_mcm.script                      # MCM configuration
│   │   ├── at_hitresponse.script              # Hit Sharing system
│   │   ├── at_health.script                   # Healing system
│   │   ├── at_accuracy.script                 # Accuracy system
│   │   ├── at_combat.script                   # Combat system (engine half: GOAP takeover, gates, lifecycle)
│   │   ├── at_combat_doctrine.script          # Combat decision half (events, palette, maneuvers, resolvers)
│   │   ├── at_combat_trace.script             # Combat DEBUG tracing + telemetry (noop when off)
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

   - **`force_set_goodwill`** (hostile goodwill): writes RELATION_REGISTRY personal goodwill (`relation_registry.cpp:161-179`). `CAI_Stalker::tfGetRelationType` routes through RELATION_REGISTRY for stalkers so this drives every downstream `is_relation_enemy` check. Gated on `IsStalker(who) AND IsStalker(mem_npc)`: `ForceSetGoodwill` smart_casts both ids to `CSE_ALifeTraderAbstract` and logs an error if either side is non-stalker. Mutant shooters and mutant squadmates both skip the goodwill write.
   - **`enable_memory_object(who, true)`**: toggles `m_enabled` on existing visual/sound/hit memory entries (`memory_manager.cpp:151-156`). No-op when no prior entry. Receiver must be `CCustomMonster` (`script_game_object2.cpp:262`); stalkers and mutants qualify, the actor does not — irrelevant here since `mem_npc` is always a squad member.
   - **`register_in_combat()`**: sets the member's squad_mask bit in `CAgentMemberManager::m_combat_mask` (`agent_member_manager.cpp:114-132`). This is the unlock for engine-native squad memory propagation. With the whole squad's bits set, the next `agent_memory_manager` tick ORs the full combat_mask into the victim's hit-memory entry's `m_squad_mask`, propagating memory of the shooter across every member including distant patrols. Requires `CAI_Stalker` receiver (`script_game_object_inventory_owner.cpp:1945-1955`); safe here because `npc_on_hit_callback` is dispatched only by `motivator_binder` (stalker squads), never `generic_object_binder` (mutant squads), so `mem_npc` is always a stalker.

### Decay and re-engagement

A periodic decay tick walks every `_disclosed[squad_id][shooter_id]` entry and prunes any older than the retention window (MCM-tunable). Pruning clears only the idempotency entry; the goodwill write is RELATION_REGISTRY-persistent and survives independently for the rest of the session.

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
- Hostility for the shooter is pinned by a hostile personal-goodwill write on every squadmate. The override survives community-relation drift and lasts the session.
- Sustained engagement: subsequent hits refresh the timestamp and return early via idempotency.
- After the retention window of no hits from a given shooter, the squad's pin on that shooter expires; the next hit re-fires the full pipeline.
- Mid-fight replenishment: new squad members inherit existing disclosures on spawn.
- Offline-shooter return: when a previously-offline tracked shooter comes back online, the spawn handler replays disclosure to every member of the squads tracking them. Members who joined while the shooter was offline get pinned at this moment.

---

## Friendly Fire

Friendly-fire damage gate in `at_hitresponse.script`. `npc_on_before_hit` scales `shit.power` by the MCM factor unless the shooter and victim are actually enemies (`attacker:relation(npc) == game_object.enemy` -> full damage). Keyed on per-NPC relation, not community: same-faction NPCs are neutral at worst and never enemy (a loner never enemy to a loner), so they stay protected, while a soured cross-faction pair (a loner vs a hostile Clear Sky) still damages each other. `relation()` is faction-paramount (the community-to-community base dominates personal goodwill). Stalker-vs-stalker only (both `IsStalker`), the actor as shooter is excluded, O(1) with no throttle (a damage block must catch every hit). MCM Combat tab: `friendly_fire_enabled` + `friendly_fire_factor`.

---

## Healing

Per-NPC self-healing. Vanilla `xr_eat_medkit.script` has a working stage machine, but vanilla `ai_tweaks/xr_eat_medkit.ltx [plugin]` lacks the `medkits=` / `bandages=` keys so `parse_list` returns `{}` and the consumption loop iterates zero times.

### Data layer fix

`mod_xr_eat_medkit_at.ltx` is a DLTX overlay on `![plugin]` adding `medkits = medkit, medkit_army, medkit_scientic, medkit_ai1, medkit_ai2, medkit_ai3` and `bandages = bandage`. Boot-time, no runtime toggle.

### Runtime tuning

`at_health.script` installs two hooks on `on_game_start`:

| Hook | Mechanism | What it changes |
|---|---|---|
| Heal rate multiplier | `xr_eat_medkit.heal_hp = _patched_heal_hp` | Per-tick `change_health` scaled by the MCM multiplier, read each tick; rescheduling via the `xr_eat_medkit.heal_hp` lookup keeps every tick on the patched function |
| Bandage tick logging | `xr_eat_medkit.heal_bleed = _patched_heal_bleed` | Logging-only wrapper around vanilla bleed loop |
| Per-rank healing-charge | `RegisterScriptCallback("npc_on_net_spawn", _on_net_spawn)` | Reads `ranks.get_obj_rank_name(npc)`, folds the rank names into MCM tiers, rolls the per-tier chance, replacing vanilla's flat roll. Per-NPC `at_charge_processed` se_var prevents re-roll. |

### Visual layer (Path 1 script-queue overlay)

Two cosmetic cues using `npc:add_animation` directly. No state_mgr, no GOAP, no `state_lib` changes. See `doc/library/modding/state-lib-animations.md` for the Path 1 script-queue overlay mechanism.

| Cue | Trigger | Animation(s) |
|---|---|---|
| Limping | `npc_on_update` per-NPC: per-tick drop on `not alive() or IsWounded or critically_wounded` (bypasses the throttle so the overlay never outlives a wounded transition); a throttled full eligibility check (`health < threshold`, `mental_state() == anim.free`, `body_state() == move.standing`, not zombied, not in smart_cover), periodically re-armed. Eligibility-lost branch drops tracking only; no `clear_animations()` (engine action transition or natural OMF expiry owns cleanup) | a per-slot `dmg_norm` torso overlay chosen from `active_slot()` + `movement_type()`; layers over engine-driven locomotion, legs stay attached to ground |
| Heal anim | One-shot via `_try_play_heal_anim` on the first heal tick. Gated on `not npc:best_enemy()`, `not IsWounded(npc)`, `not npc:critically_wounded()`. No movement freeze, no stage machine, no mid-flight aborts. Engine drains the queue when the gesture ends; action transitions clear it on the way to action_wounded / action_critically_wounded (`stalker_base_action.cpp:24-29`) | a torso medkit / bandage gesture |

Limping is independent of the healing master toggle (its callback registers unconditionally; gated at runtime by `limping_anim_enabled`). Heal cue is gated by `healing_anim_enabled` and the master toggle (it lives inside the heal_hp/heal_bleed patches that only install when healing is enabled).

Combat NPCs are excluded by the `mental_state == anim.free` gate. state_mgr drives mental to `anim.danger` in combat states (`state_lib.script:326-340` hide_fire / threat).

---

## Accuracy

Rank-aware NPC dispersion in script. `at_accuracy.script` subscribes to the vanilla `npc_shot_dispersion` callback (declared in `axr_main.script:126`, dispatched from `_g.CAI_Stalker__GetWeaponAccuracy` at `_g.script:1213-1217`).

Why script and not cvars: the engine rank curve degenerates on Anomaly gamedata. `Rank()` clamps to `[0, 100]` at `ai_stalker.cpp:764`, but vanilla `<rank>` intervals run to 26999 (game_relations.ltx:8). All Anomaly NPCs end up at `rank_k = 1.0`, so `m_fRankDisperison` collapses to the constant `dispersion_experienced_k = 0.8`. Cvar tuning is a dead knob.

Math: `out = base * disp`. The engine already multiplied by `m_fRankDisperison` (= 0.8 for every Anomaly NPC after the rank clamp) before the callback fires, plus the per-state factor. We stack `disp` on top — `disp = 1.00` preserves the engine's vanilla cone, lower values tighten it.

Per-rank tiers (novice through legend): higher rank, tighter dispersion. The per-tier factors are MCM tunables.

Per-shot hot path: a rank-name lookup, then pure-Lua scaling of the dispersion the callback hands us.

---

## Combat

GOAP planner takeover (Pattern B), split across two files: `at_combat` (the engine half - the GOAP evaluator + action, the check loop, the per-NPC state store, cover reservation, the handback decision, lifecycle) and `at_combat_doctrine` (the pure decision half - the checks, the maneuver catalog, the faction palette, movement resolution). `_install` grafts one evaluator + one action into each stalker's motivation manager and adds `world_property(EVAL_ID, false)` as a precondition to vanilla `action_combat_planner` / `action_danger_planner` / `xr_danger` / `state_mgr+2` / `alife`. Evaluator true = AT drives the NPC; false = vanilla resumes.

Inversion of control, not a loop. The engine pumps the brain and calls us - the evaluator on every plan solve, the action's `execute` on the NPC's AI scheduler tick while it is the current action (not the render frame, and not a loop we run). Both read `npc:best_enemy()` at entry, which is the engine's per-NPC target from the memory manager one layer below the blocked planner (`enemy_manager.cpp` via `memory_manager.cpp`): AT consumes the target, it never computes it. The takeover owns *what to do*, not *who the enemy is*. See `doc/library/modding/stalker-combat-brain.md`.

The catalog and tunables are data, not code: numeric knobs in `configs/alifetactics/at_combat_config.ltx`, the maneuver catalog in `configs/alifetactics/at_combat_doctrine.ltx`. `load_config` reads both at game start and builds `MANEUVERS` in place; the script holds no literal catalog.

Blocking the combat planner removes the engine's per-tick aimer, so under takeover AT supplies aim, posture, movement, destination, and fire state. The engine keeps the trigger, the per-rank dispersion, and the `can_kill_member` friendly-fire hold. All NPC control routes through xcombat. See `doc/library/modding/npc-combat-control.md`, `npc-combat-effectiveness.md`.

### Invariants

The rules the combat loop must not break. Most encode a mistake already made and reverted; keep them.

1. **The engine owns the clock.** It pumps `execute()` on the NPC's AI scheduler tick - not the render frame, not a loop we run. Add no loop, and no throttle in front of the checks.
2. **Everything is a check** - `{ name, period, wait, run }`. Aim is a check (the fastest, ~200ms). "Is the maneuver finished" is a check too (throttled by its caller, with the `maneuver_max_ms` stuck cap). Nothing per-tick runs outside the check list.
3. **One throttle per check** - its own `period`. Never a master gate in front of the checks.
4. **Each check reads only what it needs, when its period fires.** There is no observe / read-all phase and no shared context. Two checks that coincide on a pass and want the same value re-read it (a couple of cheap getters); we do not cache across checks.
5. **No allocation on the per-tick path.** `vector()` / `{}` happen only inside a maneuver commit (the destination geometry), never on the turret tick - the seen turret rides the engine object-track and a blind NPC sets no aim, so neither allocates.
6. **Maneuvers are committed.** While one is in flight only aim and the hard-stop handback run; tactical checks wait for `path_completed` or the stuck cap. No soft reason interrupts a committed maneuver.
7. **The turret is the baseline.** Aim at the enemy and fire when the shot is clear (sight-gated + wall-gated), hold weapon-up otherwise; a flee (STOW) is the one maneuver that suppresses aim and faces the run path. Checks are negative - they fire on a problem, never on an opportunity.
8. **Handback is not a combat check.** "Should AT drive this NPC at all" is a separate throttled decision (`_decide_takeover` in `_on_update`), cached in `state.eval` for the evaluator to read.
9. **Consume engine primitives, do not reinvent them.** `best_enemy` (target), perception / memory, `can_kill_member` (friendly fire), `best_cover`, `register_in_combat` (squad memory-sharing - joined on `initialize`, left on `finalize`, since the blocked combat planner no longer does it). Reusable engine wraps live in xcombat; AT calls them.

### The model: a reactive agent

AT is a reactive agent over a turret baseline. The default behavior is the turret — aim at the engine's current enemy and let it fire; the NPC does nothing else until a problem appears. The reactions are deliberately sparse and negative-only: a maneuver runs only when a check detects something wrong (a grenade down nearby, the enemy inside the close band, the firing line blocked, the NPC hurt, the enemy lost), never on an opportunity. There is no "enemy exposed, push" trigger and no resting maneuver — finish a maneuver with nothing wrong and the NPC is the turret again.

The checks are condition-action rules over that baseline. Each is a predicate that, when it holds, fires the maneuver its faction palette assigns. A maneuver, once started, is committed: it runs to completion and is never interrupted mid-flight; only a hard stop (dead, wounded, unarmed, disabled, companion) breaks it. So the agent is reactive in *what* triggers it and deliberate in *how long it holds*.

### Versus vanilla: commitment, not per-frame recompute

Both the engine's C++ combat planner and Anomaly's Lua combat schemes are per-frame controllers: every frame they re-read perception and re-pick the whole tactical state (a state + destination + aim) from the current instant, holding almost no memory of what they were doing. Fluid, but it dithers when perception flickers (the enemy weaving through line of sight, marginal cover) — the plan flips and the NPC abandons a half-finished move, the headless-chicken stutter.

AT grafts in the same way (blocks the combat planner, issues the same engine commands) but inverts the decision model: it picks a *maneuver* — a multi-second committed action (take that cover and fire, flank to there) — and runs it to completion, re-deciding only when the maneuver finishes (`npc:path_completed` or the stuck cap) or a hard stop hits. Commit, not recompute.

This is a deliberate alternative on the commitment-vs-reactivity axis, not a strict upgrade. Better for the goal — stable, deliberate, aggressive maneuvering (real advance / flank / cover-to-cover a per-frame recompute cannot sustain, plus a push toward the enemy the engine never does) — at the cost of moment-to-moment reactivity inside a committed maneuver, which the reactive checks and the handback partially restore.

Cost model (reasoned, not benchmarked): the commit model is cheaper per frame than a per-frame recompute — the heavy decision (the checks, palette, resolve, best_cover) runs only when a maneuver completes, every few seconds; between commits only the cheap aim re-point runs per tick, and the per-NPC eval is throttled with the expensive bridge calls gated behind it. It is still a Lua layer over the engine (which keeps running perception, memory, and the trigger), so total per-NPC cost is not necessarily below vanilla — the claim is the decision loop is lighter, not the whole NPC.

### The check loop

`execute()` runs one loop over the check list each AI tick. Every check carries its own `period`, a `run`, and a `wait` flag. A check runs when its period has elapsed AND it is eligible this pass: aim (`wait=false`) runs every tick — you keep aiming while you move; the tactical checks (`wait=true`) run only once the current maneuver is finished. Each check reads its own world when it fires; there is no shared read-all phase. When a tactical check's predicate holds and the palette has a maneuver for it, that maneuver is a candidate; among the candidates that fired this pass one is chosen at random and committed. A finished flee (weapon stowed) with nothing pending re-arms to `hold_fire` so the NPC never idles weapon-down.

Every tactical check is a negative condition — a problem to correct (the last row, `is_context_changed`, is the exception: it rebuilds the palette on its period and never fires a maneuver):

| Check | Problem it detects | Reaction |
|---|---|---|
| engage | no maneuver yet (fight opener) | pick an opening maneuver |
| is_grenade_near | a grenade is down nearby | retreat: kite (brave) or flee (timid) |
| is_too_close | enemy inside the close band | step back while firing |
| is_target_changed | the engine selected a new enemy | re-pick on the new target |
| is_hurt | health below the hurt threshold | retreat: kite (brave) or flee (timid) |
| is_blocked_wall | a wall on the firing line | lateral step / re-cover |
| is_blocked_friendly | a teammate on the firing line | lateral step |
| is_enemy_unseen | the enemy unseen past the lost-sight window | close in (advance / flank) |
| is_context_changed | indoor-outdoor / weapon bucket changed | rebuild the eligible maneuver list |

The check periods and the thresholds they read (the hurt fraction, the range band, the lost-sight window) are `at_combat_config.ltx` tunables, not fixed here.

"Finished" is `_is_maneuver_finished`: `npc:path_completed()` (the engine path-traversal flag, which self-clears when a new destination is set, so a committed move reads finished only on arrival and a non-moving command at once), or no maneuver set, or the `maneuver_max_ms` stuck cap forcing it after a move that never arrives. See `doc/library/modding/npc-combat-control.md`.

### Aim — the turret (`_aim_at_enemy`)

The aim check, every `aim_period_ms`: choose the weapon state and point the weapon. It applies the maneuver's fire state only when the NPC both sees the enemy (`_can_see` — the throttled `npc:see`) AND has a clear static line (`_has_wall_between`); otherwise it drops to READY — weapon up, no trigger, same posture/movement. That mirrors the engine's `KillEnemy::execute` and every vanilla scheme: fire on sight, hold weapon-up otherwise — no firing into walls, no holster.

Who points the weapon depends on sight. While the NPC sees the enemy in a fire state, `set_combat` hands the state a `look_object`, so the engine's own object-track owns the aim — it re-reads the enemy each frame and leads it, and the engine fires along that sight with its per-rank dispersion (snipers add `sniper_fire_mode`, head-line aim, through the SNIPE fire state). When the NPC is blind or pre-maneuver it drops to READY and sets no aim target; the engine's own direction sub-planner then faces it down `path_dir`. The takeover stamps no static last-known point: a script-set `eSightTypeFirePosition` is never the direction the state's own sub-planner wants for a no-look-target state, so it is overwritten within a frame of being set (proven against `state_mgr_goap.script:486-499` + `state_mgr_direction.script:160-217`). A flee (STOW) is the exception: aim is suppressed, `set_combat` passes no look target, the state resolves to `path_dir`, and the NPC turns its back and runs; finishing with nothing pending re-arms the turret to `hold_fire`. The wall gate is coarse — a chest-height static ray that false-positives on a window or railing the round would clear — so the precise check is the engine's `can_kill` geometry, pending a fork PR.

### Maneuvers

Each maneuver is one section in `at_combat_doctrine.ltx`, carrying `handles` (the checks it answers), `move` (the destination resolver), `fire` (shoot / snipe / stow), and the selection tags factions / weapons / env. Posture + speed are rolled at maneuver start and held (crouch / run chances and the stance-hold window are `at_combat_config.ltx` tunables; crouch always walks, since the engine has no crouch+run fire state). The catalog itself is data, not architecture — read `at_combat_doctrine.ltx` for the live maneuver list, their checks, and faction tags.

`move` dispatches to a resolver: hold = own node; advance_open = the NPC->enemy line, `advance_standoff_m` short of the enemy; advance_cover = cover anchored at that forward point; cover = `find_cover` near the NPC; step_back / withdraw = a short back-step away; flee = run `flee_distance_m` away from the danger (the grenade itself when fleeing a grenade, else the enemy) down a lane checked clear for the full distance (`find_flee_lane` fans the heading off straight-away in `flee_lane_arc_deg` steps to `flee_lane_max_spread_deg` until one has a reachable vertex that far out with no static wall on the line, so the NPC does not turn and run into a corner), re-firing on arrival so the NPC keeps running until it breaks contact; step_side = `find_shot`; flank = lateral + forward offset (side by squad bucket parity); flank_cover = `find_cover` at that flank offset. advance_cover and flank_cover_fire search radially around their anchor for now; the directional forward/lateral cover search is a separate task. A resolve that returns the own node holds and fires in place (never fails to vanilla). Catalog in `at_combat_doctrine.ltx`.

### Doctrine (faction palette)

No groups, no lean flags. Each maneuver lists the communities it belongs to; an NPC's palette = the maneuvers matching its (community, weapon bucket, indoor/outdoor). `pick_maneuver` rolls a random eligible maneuver, cached per NPC until the palette rebuilds. A check with no eligible maneuver is a no-op for that faction (how doctrine emerges):

Which factions fight from cover vs the open, who flanks, who falls back when hurt and who flees — those are the per-maneuver `factions` / `weapons` / `env` tags in `at_combat_doctrine.ltx`, not duplicated here. Behavior is emergent from the tags, with no group or lean code: a faction reacts to a check only if some maneuver in its palette handles it.

Retreat is exactly two behaviors: brave factions kite (`back_open_fire`: withdraw, weapon up, still firing); timid factions flee (`flee_open_stow`: stow the weapon and run `flee_distance_m` away from the danger down a clear lane via the `flee` resolver, re-firing on arrival to keep running). Both answer `is_hurt` and `is_grenade_near`; fearless factions (zombied, monolith) do neither.

### Cover

`find_cover(npc, enemy_pos, search_pos)` (xlibs): `best_cover` locates the nearest obstacle around `search_pos` (defaults to the NPC), then a ring of probes around it (8 directions × increasing radius) returns the nearest free vertex with a clear shot — the NPC stands there with cover adjacent. No clear ring vertex → hold and fire. flank_cover passes `search_pos` = the flank offset, advance_cover the forward point. `find_shot` is the same ring centred on the NPC (the sidestep), no cover required. The search is radial around the anchor today; the directional forward/lateral cover search is a separate task.

### Handback to vanilla (`_decide_takeover`)

Hard stops first, each returning `(false, reason)`: `combat_enabled` → id-hash vs `combat_share` → `alive` → not a companion → armed (else **unarmed** — AT blocks the engine's own rearm, so a weaponless NPC must yield). `wounded` is not checked here: it is a GOAP precondition on the action (`sidor_wounded_base = false`), engine-gated for free. Then two soft handbacks, both suppressed while a maneuver is in flight: **no_enemy** — `best_enemy()` is nil or dead (the engine's enemy manager has no target — died, despawned, or forgotten past its inertia window); and **lost_sight** — the enemy is still remembered but unsensed past `lost_sight_ms`. Lost-sight reads the engine's own memory clock, `time_global() - npc:memory_time(enemy)`, which aggregates sight, sound, and hit (`memory_manager.cpp`) — the same signal vanilla uses to disengage (`xr_combat_ignore.script`); `best_enemy` is memory-derived with inertia (`enemy_manager.cpp`), so it does not flicker. The read is wrapped in `xcombat.get_unseen_ms`. The decision is recomputed off the hot path by `_on_update` (`npc_on_update`, throttled to `eval_period_ms`); the engine-polled evaluator only reads the cached `state.eval`, so each `actual()` poll costs a flag read, not a recompute.

**Maneuver transaction.** While a committed maneuver is still in flight (`_is_maneuver_running`: a maneuver set, the path not yet completed, within the stuck cap), both soft handbacks (`no_enemy`, `lost_sight`) are suppressed — a maneuver is never broken mid-flight; only the hard stops (`dead`, `wounded`, `unarmed`, `disabled`, `companion`) interrupt it. The chase target itself comes from memory: while the enemy is not currently seen, a move resolver targets `npc:memory_position(enemy)` (last-known), not the live position (`xcombat.get_track_pos`) — so a pursuit heads to where the enemy was lost and then yields to vanilla search, instead of trailing the live, fleeing target up a stairwell forever.

### Lifecycle

`npc_on_net_spawn` installs (sentinel-guarded); `npc_on_update` recomputes the takeover decision per NPC (throttled to `eval_period_ms`, off the engine-polled evaluator's hot path); `npc_on_net_destroy` clears install + releases the cover reservation; `server_entity_on_unregister` drops the NPC's state; `actor_on_first_update` resets the trace and refreshes the log level — it must not wipe the install/state tables (the spawn/destroy lifecycle maintains those; wiping them here dropped every NPC installed during level load); tunables load in `on_game_start`; `on_option_change` / `mcm_option_restore_default` refresh the log level. Each `action:initialize()` (combat start) registers the NPC in the squad combat mask (the blocked combat planner no longer does, so this restores squad memory-sharing - `register_in_combat`, the engine's own call at `stalker_combat_planner.cpp:182`) and clears the state's per-fight maneuver fields (`maneuver`, `dest`, `enemy_id`) so `engage` reopens; the weapon/palette cache persists, and `finalize` unregisters from the mask and clears the maneuver on handback so the in-flight guard never reads a stale one.

### Tracing

At DEBUG, `at_combat_trace` writes one `scan` line per decision pass (the readings the checks saw, which check fired, which maneuver — or `-` for a turret tick), a `decide` line per committed maneuver (the rolled posture/speed plus the apply/resolve ms), an `eval` line on each takeover/handback transition, and an `install` line per takeover graft (attempt / no_manager / installed). No aggregate counters and no console command — the log lines are the telemetry. Noop when DEBUG is off (no string, no alloc on the off path). The MCM tab settings are dumped once at `actor_on_first_update` as a single INFO `[MCM]` line.

### Configuration model

The takeover is gated by three MCM toggles — a master enable, a per-id `combat_share` split between AT and vanilla, and a companion skip. Everything else is data, not code: numeric tuning in `at_combat_config.ltx`, the maneuver catalog in `at_combat_doctrine.ltx`. The scripts carry only script-side fallbacks, so the values themselves live in the LTX, never here.

### xcombat boundary

AT owns *what to do*; xcombat (xlibs) owns *how to issue it to the engine*. Every NPC command — weapon state, aim, movement, cover and clear-shot search, the line-of-fire and enemy-memory reads, the cover reservation — goes through an xcombat primitive; AT makes no raw engine combat call. The primitive surface and its per-call costs are documented in `stalker-mods/xlibs/doc/architecture.md`, not duplicated here.

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
- `danger_attack_sound`: script_action_danger_alert dispatch for `attack_sound` danger type. Includes an actor-aim gate (the actor must be facing the NPC) so actors walking past with rifle out do not trigger cover-seek. Vanilla had no script handler for this danger type.
- `danger_actor_tables`: read separate inertion and ignore tables from `[danger_inertion_actor]` and `[danger_object_actor]` when danger source is the actor. Tune player encounters independently of NPC-vs-NPC.

### Paired LTX

`configs/ai_tweaks/xr_danger.ltx` ships:
- Weather-conditional ignore distances (rain/storm reduces detection)
- Separate actor-source tables that respond to `actor_enemy` condition
- Dead `hit`/`sound`/`visual` keys (PerceiveType names; collide with EDangerType enum values) are dropped

DLTX overlay mods that replace `[danger_inertion]` take precedence over these base values; absent those, the values here apply.

### Composition

The override is marked `-- @override` so the validator skips inherited vanilla style warnings. Conflicts with any mod that also overrides `xr_danger.script`. MCM Danger tab describes always-on fixes and the three improvement toggles.

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

`jam_enabled` master toggle: on, the override returns 0 for NPC misfire probability; off, it forwards to the original modded-exes function. Effective on the next engine functor call.

---

## NPC Ammo

Veteran-rank-and-up NPCs fire AP from inventory until depleted, then revert to vanilla magic FMJ. Player loots the remainder.

Engine context: NPCs do not consume real inventory ammo while `unlimited_ammo()` is TRUE, which is the default for every stalker (`xrServer_Objects_ALife_Monsters.cpp:2164`). The magic refill at `WeaponMagazined.cpp:520-571` produces copies of `m_DefaultCartridge`, keyed from `m_ammoTypes[m_ammoType]` at line 559-560. Setting `m_ammoType` re-keys those copies; inventory drain in this script is cosmetic for corpse-loot composition.

### Fixed AP set

`AP_SECTIONS` is a hardcoded set of clean armor-piercing cartridge sections enumerated from vanilla Anomaly `configs/items/weapons/`, covering the base calibers. Degraded `_bad` / `_verybad` variants are intentionally excluded — they're treated as "carry but don't promote". Add new calibers to the table when modpacks introduce them.

### Per-NPC state

`_state[id] = { idx, sec, left }`. Set when a veteran-rank NPC is first observed carrying any `AP_SECTIONS` entry in inventory; nil otherwise (those NPCs run vanilla forever). Cleared on `server_entity_on_unregister`. Not save-persisted.

### Tick algorithm

`pick(npc)` is the public entry, subscribed to `npc_on_update` (slow per-NPC throttle, gated on `best_enemy()`). Each tick:

1. MCM gate (`ammo_enabled`), `npc:alive()` and `IsStalker(npc)` filter, `npc:character_rank() >= min_ap_rank` filter — sub-master returns immediately.
2. `IsWeapon(npc:active_item())` filter.
3. If `_state[id]` nil: scan `ammo_class` for the first entry in `AP_SECTIONS` with inventory count > 0; seed `_state[id]` with that idx, section, and starting count. If no AP found, leave `_state[id]` nil so the NPC stays on vanilla.
4. Drain `_state[id].left` by `DRAIN_PER_KIND[kind]`, clamped at `MIN_SPARE`.
5. If `left > MIN_SPARE`: set `m_ammoType = idx` and `_sync_box(npc, sec, left)` to defeat engine `try_advance_ammo` top-up.
6. Else: set `m_ammoType = 0` (vanilla magic FMJ for the rest of the NPC's online session).

Sub-rank NPCs early-exit at the rank check, paying only the `character_rank()` read; the full tick runs on a slow per-NPC throttle, gated on having an enemy.

### Rank gate

`npc:character_rank() >= min_ap_rank`, the veteran band by default (the threshold maps onto the rank ratings in `configs/creatures/game_relations.ltx`). Below-threshold NPCs never enter the picker; their inventory AP boxes are untouched during life and trimmed at death by the death hook.

This intentionally restricts AP to veteran+ to prevent armor degradation across the whole NPC population. Tune via LTX.

### Death hook

`npc_on_death_callback` walks the dead NPC's inventory and trims every ammo box where `ammo_get_count() > MIN_SPARE` down to MIN_SPARE. Runs before `motivator_binder:death_callback`'s `decide_items_to_keep` call at `xr_motivator.script:362`. Boxes ≤ 5 survive `death_manager.script:456-460`'s `>5` deletion filter, so the depletion trail persists as corpse loot.

Without this hook the engine's `try_advance_ammo` top-up at the last reload before death would leave boxes at `boxSize` and `decide_items_to_keep` would release them entirely. Player would see only `death_manager.try_spawn_ammo`'s procedural box. With the hook, player loots the actual remainder.

### Tunables

`gamedata/configs/alifetactics/at_ammo.ltx`: the box floor, the rank threshold, and a per-weapon-kind drain rate. The floor must stay at or below the `decide_items_to_keep` deletion cutoff so corpse boxes survive. Script-side fallbacks match so a missing key or file won't crash.

### MCM

`ammo_enabled` master toggle: off, the tick early-exits with no `m_ammoType` writes, no inventory clamping, and no death-time trim.

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
| Combat | NPC GOAP action (at_combat_action), Pattern B preconditions on action_combat_planner/action_danger_planner/xr_danger.actid/state_mgr+2/alife, set_dest_level_vertex_id, state_mgr.set_state, set_body_state, set_movement_type, set_sight, `m_sniper_fire_mode` flag | GOAP `add_evaluator`/`add_action`/`add_precondition` (custom evaid/actid), `npc:best_cover`, `level.vertex_in_direction`, `npc:sniper_fire_mode`, `db.used_level_vertex_ids` reservation |
| Danger | NPC danger evaluator/action graft, `script_danger` per-id table for sound-source dispatch | Engine callbacks `npc_on_hear_callback`, `npc_on_death_callback`, GOAP planner graft |
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
