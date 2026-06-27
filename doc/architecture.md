# AlifeTactics Architecture

Combat AI mod for STALKER Anomaly. Independent user-facing systems: a hit-share force-disclosure, a self-heal data + animation layer, a per-rank weapon accuracy curve, a full-file xr_danger override with bug fixes and three toggleable improvements, and an intermittent combat takeover that briefly borrows an NPC from vanilla for one committed maneuver where vanilla is weak (the takeover overrides zero vanilla combat files, so it coexists with vanilla / GAMMA / AI Rework / RCAI / Useful Idiots / Mora). No shared substrate.

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
| `at_combat.script` | feature | re-architecture in progress (intermittent takeover transaction model — see Combat section + t122; shipped code is still the prior full-takeover) |
| `xr_danger.script` | feature | done (full-file override) |
| `at_jam.script` | feature | done (modded-exes xr_weapon_jam.GetConditionMisfireProbability override; suppresses script-injected NPC misfire) |
| `at_ammo.script` | feature | done (AP fired from carried boxes; per-engagement rank/rpm-weighted box-delete decay reverts to FMJ when out; veteran-rank gate; no death hook) |
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

Status: re-architecture in progress. The shipped code in `at_combat.script` / `at_combat_doctrine.script` still implements the prior full-takeover (AT as the combat brain over a turret baseline); this section describes the target design now being built. Full design and ordered build plan: `stalker-dev/doc/todo/todo-alifetactics-next.md` t122. The full-takeover code is preserved on the `combat_takeover` branch and in `.deleted/`.

### The model: a takeover transaction

Vanilla owns every NPC by default (autocommit). AT does not run combat; it briefly borrows one NPC inside a transaction — BEGIN one maneuver, run it atomically to completion, COMMIT — then hands control back. AT is an interrupt handler over vanilla, not the combat brain. Between transactions AT is absent: no engine state written, no aim set, nothing while the gate is off.

Three NPC states, real cost only in the last:
- Idle — not in a fight, zero AT work.
- Monitored — in a fight; one cheap throttled predicate "should I BEGIN now", engine-memory reads only, no raycasts.
- Under-transaction — one committed maneuver: aim upkeep, drive to the destination, watch for completion.

### Fight detection

Idle and Monitored switch off the engine combat mask, which Hit Sharing's `register_in_combat` already sets on every faction-enemy hit (`at_hitresponse` -> `xcombat.disclose_enemy`). A hit is fight-start regardless of shooter range, so no separate long-range entry is needed. Edge-triggered; while Idle the NPC is invisible to AT.

### BEGIN — the trigger decides when

While Monitored, a predicate fires only where vanilla is weak or absent: too_close, hurt, stalled (holding too long, no re-engage), grenade, and hit-from-unanswerable-range (range defense, reading the hit vector not `best_enemy`). On a fire, AT picks the faction-flavored intervention and BEGINs by flipping the GOAP gate. The things vanilla does well — the opener, re-target, search, cover, turret, grenade throw and dodge, squad coordination — are not triggers.

### The body, COMMIT, ABORT

The maneuver body is atomic and isolated: it runs to completion, and no new trigger or perception flicker interrupts it mid-flight — this is what kills the per-frame headless-chicken dither. COMMIT releases the NPC in a state vanilla accepts so it does not undo the maneuver: open-ending maneuvers (flee, kite, advance) need no reconcile; the one cover-ending maneuver (flank) delivers the NPC to the engine's own `best_cover` so vanilla continues the cover cycle without relocating. ABORT is immediate on a hard stop (dead, wounded, unarmed, no_enemy), even mid-body. At most one open transaction per NPC.

### Maneuvers

Keep only what vanilla cannot do: flee (timid rout-to-base), kite (back-and-fire — vanilla `get_distance` is dead), flank (disciplined — vanilla detour is gated and rare), advance (open charge — vanilla never advances in the open). Drop every duplicate of a vanilla operator (turret, cover-seek, sidestep). Two families: pure-AT (flee, kite, advance — end in the open, AT drives start to finish) and cover-handoff (flank — deliver to engine cover, release). Advance ends in the open, never in held forward cover, so vanilla's evaluator never disagrees and walks the NPC back.

### The GOAP graft (the control point, unchanged)

`_install` grafts one evaluator and one action per stalker and adds `world_property(EVAL_ID, false)` as a precondition on the vanilla combat and danger planners. This precondition block is the only GAMMA-proof control point, so it is kept. The evaluator is true only while a transaction is open. The trigger detection (decide to BEGIN) runs always-on in `_on_update` under vanilla, engine-memory only, throttled; the thin runner (aim upkeep and completion watch) runs in the action's execute, only while the transaction holds.

### Range defense and active reach

Range defense is a trigger inside the model: a hit from a range the NPC cannot answer BEGINs a break-contact-to-cover transaction. The complementary active reach — letting an NPC register and shoot back at a far shooter by widening the combat-ignore gate (LOS-gated) — is a separate below-planner feature (t92), not part of the takeover.

### xcombat boundary

AT owns what to do; xcombat (xlibs) owns how to issue it to the engine, and is where precision wrappers live (for example `can_kill` shot-clearance over a coarse ray). Every NPC command — weapon state, aim, movement, cover and clear-shot search, the line-of-fire and memory reads, the cover reservation — goes through an xcombat primitive; AT makes no raw engine combat call.

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
- Brainstorm pool: `stalker-dev/doc/todo/todo-alifetactics-backlog.md`
- Engine PR queue: `stalker-dev/doc/todo/todo-demonized-exes.md`
- xlibs architecture: `stalker-mods/xlibs/doc/architecture.md`
- AlifePlus architecture: `stalker-mods/AlifePlus/doc/architecture.md`
- AlifeGuard architecture: `stalker-mods/AlifeGuard/doc/architecture.md`
- AlifeBalance architecture: `stalker-mods/AlifeBalance/doc/architecture.md`
