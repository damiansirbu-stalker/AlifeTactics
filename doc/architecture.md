# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Independent user-facing systems: a hit-share force-disclosure, a self-heal data + animation layer, a per-rank weapon accuracy curve, a full-file xr_danger override with bug fixes and three toggleable improvements, and an intermittent combat takeover that briefly borrows an NPC from vanilla for one committed maneuver where vanilla is weak (the takeover overrides zero vanilla combat files, so it coexists with vanilla / GAMMA / AI Rework / RCAI / Useful Idiots / Mora). No shared substrate.

Built on xlibs (xsquad, xttltable, xtime, xprofiler, xlog, xmcm, xslice, xcreature).

Part of a four-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** tunes existing rates and counts, **AlifeGuard** keeps alife state clean, **AlifeTactics** controls how NPCs fight in combat (this mod).

---

## Invariants

- **No steady-state per-frame work.** Ongoing work runs on a throttled tick (a fixed interval) or on a discrete engine event (hit, shot, spawn, option change); it never runs continuously every frame. A per-frame engine callback (`npc_on_update`) is used only as a carrier that throttles before doing anything, and we never place our code on a path the engine runs every frame (a visibility or fire functor). Frame-spreading a bounded one-off batch (xslice, 1 item per frame) to avoid a single-frame spike is the one allowed use of the frame; it completes and stops. Full rule and rationale: `doc/standards/code-standards.md` "No Per-Frame Work".

---

## Status

Version 1.0.0.

| Module | Type | State |
|---|---|---|
| `_at_deps.script` | infra | done |
| `at_mcm.script` | infra | done |
| `at_test.script` | infra | done |
| `at_hud.script` | infra | done (live debug HUD: nearby NPCs with logic scheme, combat operator, and target; noop unless enabled) |
| `at_hitresponse.script` | feature | done |
| `at_health.script` | feature | done |
| `at_accuracy.script` | feature | done |
| `at_combat.script` | feature | v2 intermittent takeover: solo maneuver set (kite, snipe, retreat, flee) + fire gate + flee disengage, actor-scoped, validates S+, pending playtest; squad coordination and enemy openings are the open phases (see Combat section + todo-combat-takeover-v2.md) |
| `xr_danger.script` | feature | done (full-file override) |
| `at_jam.script` | feature | done (modded-exes xr_weapon_jam.GetConditionMisfireProbability override; suppresses script-injected NPC misfire) |
| `at_ammo.script` | feature | done (AP fired from carried boxes; per-engagement rank/rpm-weighted box-delete decay reverts to FMJ when out; veteran-rank gate; no death hook) |
| `zzz_at_health_patch.script` | feature | done (vanilla xr_eat_medkit re-roll suppressor) |
| `configs/ai_tweaks/mod_xr_eat_medkit_at.ltx` | data | done |
| `configs/ai_tweaks/xr_danger.ltx` | data | done |
| `configs/alifetactics/at_combat_config.ltx` | data | done (Combat takeover tunables) |
| `configs/ui/ui_at_stats.xml` | data | done (at_hud layout) |

Backlog (not built):
- Tactical flee (per-squad retreat to friendly smart under power imbalance)
- Memory persistence (extended danger inertion under sustained combat)
- NPC weapon bias (per-NPC callback overrides for loadout selection)

Rejected (not backlogged): combat schemes (per-NPC `combat_type` via condlist). The `script_combat_type` condlist is the most contested combat surface in modpacks - it is GAMMA AI Rework's core mechanism and single-owner by design - and everything a scheme would express is already a maneuver (which forces its action) or a behavior (which composes). A scheme buys nothing here and surrenders composition.

Groomed task entries in `stalker-dev/doc/todo/todo-alifetactics-next.md`; the takeover build plan in `todo-combat-takeover-v2.md`; the 2026-07-02 adversarial review in `todo-alifetactics-fable-review.md`.

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
│   │   │   └── at_ammo.ltx                    # NPC Ammo tunables
│   │   ├── ui/
│   │   │   └── ui_at_stats.xml                # at_hud HUD layout
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

Each system has its own file, its own MCM tab, and one master toggle. The Scope column is the actor-scope rule below, at a glance.

| System | File | MCM Tab | Master toggle | Scope | Composition |
|---|---|---|---|---|---|
| Hit Sharing | `at_hitresponse.script` | Hit Sharing | `hit_share_enabled` | All NPCs | callback |
| Healing | `at_health.script` | Healing | `healing_enabled` | All NPCs | fn-patch |
| Accuracy | `at_accuracy.script` | Accuracy | `accuracy_enabled` | All NPCs | callback |
| Combat | `at_combat.script` | Combat | `combat_enabled` | Actor only | transaction-override |
| Danger | `xr_danger.script` (full-file override) | Fixes > Danger | bug fixes always-on; three toggleable improvements | All NPCs | excludes (full override) |
| Weapon Jam | `at_jam.script` | Fixes > Weapon Jam | `jam_enabled` | All NPCs | save-wrap |
| NPC Ammo | `at_ammo.script` | Fixes > NPC Ammo | `ammo_enabled` | All NPCs | callback |

Composition classes, what each does to the surrounding stack: `callback` subscribes to an engine callback and adds to it (composes with any other subscriber); `fn-patch` replaces a vanilla module function, rescheduling through the same lookup so it holds; `save-wrap` saves the prior function and forwards to it when disabled (composes with a prior installer); `transaction-override` suppresses whatever brain is installed, but only for the seconds it holds one NPC; `excludes` is a full-file override, mutually exclusive in MO2 with any other mod that overrides the same file (Danger is the one such leaf, until it becomes a monkey-patch).

### Actor scope: only the Combat takeover is actor-gated

The Combat takeover is the sole actor-scoped system. It seizes an NPC exactly when that NPC's current enemy is the player, gated by one predicate, `_target_eligible(enemy)` at `at_combat.script:73-74`; in an NPC-vs-NPC fight it never engages. This is a target gate (the NPC's enemy must be the actor), not a participant exclusion.

Every other system runs on all NPCs in any fight, NPC-vs-NPC included. Hit Sharing, Healing, Accuracy, and Danger never read who the enemy is. Friendly Fire, Weapon Jam, and NPC Ammo exclude the player only as a participant (the player's own hits, the player's own weapon); that is the inverse of an actor-only scope, not an instance of it. So widening or keeping the Combat gate changes the takeover alone and touches none of the conduct, mechanics, or fixes systems.

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

Status: built incrementally. On `main`: the GOAP graft (`xcombat.install_takeover`), the maneuver-catalog spine (`at_combat_doctrine`), and the full solo maneuver set — kite, snipe, retreat, flee — with the fire-discipline gate (`fire_make_sense` at the decision point) and flee's disengage. Actor-scoped, validates S+, not yet playtested, so squad coordination and enemy openings remain the open phases. Full phase plan + the decision record: `stalker-dev/doc/todo/todo-combat-takeover-v2.md` (t130-t137 + the Plan section). The full-takeover v1 is preserved on the `combat_takeover` branch.

### Scope: vs the player, decoupled

The takeover is designed against the player and, for now, only runs when an NPC's enemy is the actor. NPC-vs-NPC is dropped (too many corner cases; the actor is always online, has a stable id, never despawns mid-fight, and exposes facing/health/reload). The restriction is one predicate, `_target_eligible(enemy)`; every maneuver, viability check, and read takes a generic `enemy` game_object, so widening the scope later is a one-line change. No readme or MCM surface mentions it.

This actor gate is unique to the takeover. The conduct, mechanics, and fixes systems (Accuracy, Hit Sharing, Healing, Danger, Weapon Jam, NPC Ammo) are unscoped and run on every NPC regardless of who it fights. See "Actor scope" under User-facing systems for the full rule.

### The model: a takeover transaction

Vanilla owns every NPC by default. AT does not run combat; it seizes one NPC for one committed, time-boxed maneuver, then releases. AT is an interrupt over vanilla, not the combat brain. There is no global share knob — an NPC is seized only when a trigger and a viability check both fire for it, so selectivity is inherent. At most one open transaction per NPC.

Three NPC states, real cost only in the last:
- Idle — not in a fight, zero AT work.
- Monitored — in a fight; the decision pipeline runs throttled, engine-memory reads only, no raycasts.
- Under-transaction — one committed maneuver running to its end.

### The decision pipeline

Per monitor tick (npc_on_update, throttled) the pipeline evaluates pluggable predicates grouped as own_state, own_geometry, enemy_state, team_state. There is no shared read-all: each predicate reads its own world, and a value reused within a cycle is memoized lazily for that cycle only. Each maneuver carries a trigger (a cheap predicate — a situation is present: too_close, hurt, grenade, enemy_unseen, stalled, an enemy opening) and a resolver (where the maneuver sends the NPC). The resolver IS the viability gate: it returns a destination when the maneuver is doable here, or nil to decline — one pass answers both "does it make sense?" and "where to?", no separate viability predicate. A maneuver fires only when its trigger passes and its resolver yields a destination. The things vanilla does well — the opener, re-target, search, turret, grenade dodge — are not triggers.

### The maneuvers and their roles

Eight maneuvers (names locked) across three tactical roles:
- Base of fire (hold and fire to pin the enemy): suppress, snipe.
- Maneuver element (reposition offensively, only safe under a base of fire): assault (open charge, no cover), push (advance to cover), flank.
- Break contact (individual survival): kite (back ~6m), retreat (to cover), flee (rout).

too_close always answers with kite; the pressure response (retreat vs flee) is per-faction temperament (brave factions retreat to cover, timid ones flee, monolith and zombied do neither). snipe is a fire-mode hold, not a movement, answering a stalled sniper. The solo triggers are settled: kite = raw proximity (enemy inside the weapon's minimum range), snipe = stalled for `w_sniper`, retreat and flee = hurt or grenade split by faction temperament. The squad and enemy-opening maneuvers are the remaining phases.

### The maneuver pattern

Every maneuver follows one shape, no per-maneuver invention and no per-frame code: a `start` issued ONCE when AT takes the NPC (the GOAP action's `initialize` — the single place AT writes the engine, setting the move and fire state), an `ends_on` (`arrival` for movers, `time` for holds), and a `max_ms` cap. The action's `execute` does nothing — the engine's own per-frame state machine carries the maneuver (aims at the enemy, fires, walks); AT runs zero per-frame logic. The monitor watches the end on its throttled tick — `is_arrived` (over `path_completed`) for movers, elapsed time for holds — and COMMITs, lowering the gate. Two throttle rates: the decide tick (gate down, 500ms, runs for every fighting NPC) and the watch tick (gate up, 200ms, runs only for the few NPCs mid-maneuver, so hand-back is prompt). No early release, no soft interrupt. Aborts come free from the action's preconditions failing (alive, armed, not wounded, has-enemy); a held NPC simply eats a rare grenade rather than AT re-implementing vanilla's reactions. A mover's end position is sticky for free; a held mode reverts on release, which is correct.

### Squad coordination, decentralized

A squad fighting the player coordinates fire and movement, but with no central brain. Role eligibility is biased by `get_squad_ordinal`; a maneuver-element's viability requires that the enemy is being pinned, so one NPC flanking alone (a death wish) cannot fire. The reliable pin signal is an AT NPC committed to a base-of-fire maneuver; the coordination is emergent from each NPC's local read of its squad.

### The GOAP graft (the control point)

The graft adds one evaluator and one action per stalker and `world_property(EVAL_ID, false)` as a precondition on each entry of `xcombat.get_blocked_planners()`. While the per-NPC gate flag is true the vanilla combat/danger/alife chain is gated off and the grafted action is the only producer of the `EVAL_ID=false` the brain now requires, so it runs; clear the flag and vanilla resumes. The graft mechanism is encapsulated in `xcombat.install_takeover(npc, spec)` / `release_takeover(npc)`, where the spec is `{ gate, on_begin }` — the gate flag the evaluator polls and the one-time maneuver start the action's `initialize` calls; AT owns the spec, xcombat owns the GOAP classes.

### xcombat boundary

AT owns what to do; xcombat (xlibs) owns how to issue it to the engine. Every NPC command and read — weapon state, aim, movement, cover and clear-shot search, the line-of-fire and memory reads, arrival, the cover reservation, the enemy-state reads — goes through an xcombat primitive; AT makes no raw engine combat call. New primitives for this rebuild: `install_takeover`/`release_takeover`, `is_arrived`, `is_reloading`, `is_bleeding`, `is_moving`, and suppressive fire via `set_combat`; the rest is reuse.

The enemy-eval override (flee's disengage and the t92 sniper force-enable) is the one deliberate exception. It is not an xcombat primitive: it lives on the `on_enemy_eval` engine callback seam, so AT registers and owns it directly like any other callback subscription (`npc_on_hit_callback`, `npc_on_net_spawn`). One AT subscriber holds a per-NPC override table and dispatches both directions — a fleeing NPC forces `result=false` (checked first), a sniper-in-range forces `result=true`. xcombat stays stateless by design: it holds no live-event callback or ownership table on its own behalf, so a stateful `set_enemy_eval` would break that contract. This matches `xcombat.on_action_switch`, which registers the caller's function statelessly rather than owning the state.

### Identity and rejected alternatives

The identity the takeover is built for: recognizable committed maneuvers, composition under modpacks (it overrides zero combat files), and solving the shuffle (vanilla twitching between actions instead of committing). Everything above serves those three; a change that trades any of them away is out of scope.

The continuous `script_combat_type` scheme (GAMMA AI Rework, ReDone Combat AI) is the rejected alternative, and a 2026-07-05 read of both confirmed why: it owns an NPC's whole combat single-ownedly, and either does less than vanilla (GAMMA's thin camper sets one state) or reimplements it worse (ReDone's fat `get_combat_movement` and global-cvar aim). The intermittent takeover borrows an NPC for one maneuver where vanilla is weak and hands back, so vanilla's own aim, fire discipline, cover cycle, and squad coordination run the rest of the time - the maneuvers without deleting the strengths.

The considered-and-rejected alternative to the top-level planner block is rx-style injection inside the combat sub-planner (`cast_planner` on `action_combat_planner`, then `add_evaluator`/`add_action` there, per Rulix's `rx_combat.script:327-353`). It preserves what the top-level block loses - `CStalkerCombatPlanner::update`'s side effects, `react_on_grenades` / `react_on_member_death` at `stalker_combat_planner.cpp:102-105`, and the initialize/finalize mask and danger inertion. But it arbitrates against whatever a modpack grafts inside that same planner, so it surrenders exactly the robustness the takeover was chosen for. The top-level block wins for the GAMMA audience, at the cost of suppressing those `update` side effects for the seconds it holds - which is why a transaction stays narrow and brief.

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
7. `action_danger` finalize wiped the whole shared `db.used_level_vertex_ids` reservation map, clobbering every other system's cover claims (the Combat takeover's, vanilla's). AT releases only the vertices this NPC owns (`xr_danger.script:882-890`).

### Extension callback

`eval_danger` fires `npc_on_eval_danger` (`xr_danger.script:311-313`) with `flags.ret_value = true` before it evaluates; a subscriber that sets `flags.ret_value = false` suppresses danger for that NPC on that tick. AT adds this seam so another system can veto danger evaluation without overriding the file.

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

Veteran-rank-and-up NPCs fire AP from the loose ammo they carry; each engagement has a rank- and fire-rate-weighted chance to consume one AP box, until the NPC runs out and reverts to vanilla magic FMJ. NPCs drop no AP boxes as loot.

Engine context: while `unlimited_ammo()` is TRUE (the stalker default, `ai_stalker.cpp:78,1585`) the magic refill at `WeaponMagazined.cpp:559-571` copies `m_DefaultCartridge` keyed from `m_ammoTypes[m_ammoType]` and never consumes inventory, and the reload does not re-derive `m_ammoType` -- so `wpn:set_ammo_type(idx)` re-keys what is fired and holds for the online session. `get_ammo_count_for_type` sums loose belt+ruck boxes only (`Weapon.cpp:1727`), so the AP an NPC carries comes from AlifePlus trade/loot; vanilla gives NPCs zero loose ammo (`xrs_rnd_npc_loadout.script:215`, ammo-give block commented out).

### Budget is the inventory

No virtual ledger. The budget is the NPC's AP boxes themselves (box_size 15 rifle / 16 pistol; stocked to 1 box by trade, up to 3 by looting, per the box-aligned policy unification). `_find_ap` returns the first `AP_SECTIONS` entry in the weapon's `ammo_class` with a loose count > 0. `AP_SECTIONS` is the hardcoded clean-AP set; degraded `_bad` / `_verybad` variants are excluded.

### Tick algorithm

`pick(npc, now)` is the public entry, subscribed to `npc_on_update` (per-NPC throttle). Gates: cached `ammo_enabled`, `alive`/`IsStalker`, `character_rank() >= min_ap_rank`, `IsWeapon(active_item)`, non-empty `ammo_class`. Then it splits on `best_enemy()`:

- Combat tick: on combat entry or weapon change, `_find_ap` caches `idx`/`sec`; while AP is carried, hold `m_ammoType = idx` (re-asserted only if changed). AP is held for the whole fight, no mid-fight revert.
- Peace tick: once `best_enemy()` has been nil past `peace_debounce_ms`, the engagement ends -- roll `_decay_chance`; on a hit, `alife_release` one AP box of `sec`; if that section is now empty, set `m_ammoType = 0`.

### Decay chance

`ap_decay_base * (rpm / rpm_ref) * rank_weight`, clamped to [0,1]. `rpm` is the weapon's effective fire rate (bolt 25, SVD 60, rifle 600, SMG/MG 900). `rank_weight` lerps from 1.0 at `min_ap_rank` to `rank_weight_floor` at `rank_ceiling`. So fast weapons burn AP quickly and high rank conserves it -- a legend bolt-action shoots AP almost always. Deleting a whole box is permanent: `try_advance_ammo` (`object_actions.cpp:131-169`) refills rounds inside surviving boxes but cannot recreate a deleted box, so counting boxes never fights the top-up.

### No death hook

Vanilla `decide_items_to_keep` (`xr_motivator.script:362` -> `death_manager.script:457`) `alife_release`s every ammo box with > 5 rounds on death. An AP box is 15 rounds, so NPCs drop no AP boxes with no module help. `npc_on_death_callback` fires at `xr_motivator.script:396`, after that release, so a death-time trim could not preserve AP anyway -- the old hook was removed.

### Rank gate

`character_rank() >= min_ap_rank`, veteran by default (12000 = veteran floor in `configs/creatures/game_relations.ltx [game_relations] rating`; professional caps at 11999, legend starts at 27000). Below-threshold NPCs early-exit at the rank read.

### Tunables

`gamedata/configs/alifetactics/at_ammo.ltx`: `min_ap_rank`, `rank_ceiling`, `ap_decay_base`, `rpm_ref`, `rank_weight_floor`, `peace_debounce_ms`. Script-side fallbacks match so a missing key or file won't crash.

### MCM

`ammo_enabled` master toggle (cached, refreshed on `on_option_change`): off, `pick` early-exits with no `m_ammoType` writes and no box deletion.

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
| Hit Sharing | RELATION_REGISTRY personal goodwill, memory entry m_enabled, agent_member_manager m_combat_mask | `force_set_goodwill`, `enable_memory_object`, `register_in_combat` |
| Healing | NPC health field, bleeding field, `healing_charge` se_var | `change_health`, direct `bleeding =` write, `se_save_var` |
| Accuracy | Per-shot dispersion radius via callback return | (subscribes to `npc_shot_dispersion`) |
| Combat | NPC GOAP action (at_combat_action), Pattern B preconditions on action_combat_planner/action_danger_planner/xr_danger.actid/state_mgr+2/alife, set_dest_level_vertex_id, state_mgr.set_state, set_body_state, set_movement_type, set_sight, `m_sniper_fire_mode` flag | GOAP `add_evaluator`/`add_action`/`add_precondition` (custom evaid/actid), `npc:best_cover`, `level.vertex_in_direction`, `npc:sniper_fire_mode`, `db.used_level_vertex_ids` reservation |
| Danger | NPC danger evaluator/action graft, `script_danger` per-id table for sound-source dispatch | Engine callbacks `npc_on_hear_callback`, `npc_on_death_callback`, GOAP planner graft |
| Weapon Jam | Module-level function table on `xr_weapon_jam` | Lua function assignment (`xr_weapon_jam.GetConditionMisfireProbability = ...`) read by engine functor lookup at `Weapon.cpp:1781` |
| NPC Ammo | CWeapon `m_ammoType` field via `wpn:set_ammo_type(idx)` (re-keys `m_DefaultCartridge` for magic refill ballistics); per-engagement box-delete decay (`alife_release` one AP box on a rank/rpm-weighted roll); reverts to `m_ammoType = 0` when the section is empty | `npc:active_item`, `wpn:set_ammo_type`, `wpn:get_ammo_type`, `wpn:get_ammo_count_for_type`, `npc:best_enemy`, `npc:character_rank`, `npc:iterate_inventory`, `alife_release` |

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
