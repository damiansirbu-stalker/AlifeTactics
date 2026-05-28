AlifeTactics: Combat AI replacement for STALKER Anomaly, by Damian
Version: 1.0.0
GitHub: https://github.com/damiansirbu-stalker/AlifeTactics
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeTactics/issues

Vanilla NPCs in Anomaly are not threatening. They miss most of their shots. They walk
to cover with the weapon raised, looking silly. Each NPC discovers threats on its own
so squadmates remain oblivious to attacks on their teammates. They never retreat
tactically. They cannot deal with a player sitting behind a rock at 80 meters.

AlifeTactics rebuilds combat AI from the ground up around a single shared data
structure: a per-squad memory table. Every behavior reads and writes that table
instead of each one carrying its own scratch state. Hits get logged once at the
squad level. Disclosure thresholds accumulate across the whole squad. Combat state
escalates from idle through alerted to engaged based on event volume. Retreat
decisions read squad power against enemy power. Danger memory retention scales with
combat state. One source of truth, many small behaviors.

Hit disclosure:
  When N hits from a single shooter accumulate against a squad (across any members),
  the shooter is added to every squadmate's engine memory as a known hostile. The
  engine combat planner takes over from there: cover selection, return fire, flank
  decisions all work because the engine now knows who the enemy is. No scripted
  movement, no forced destinations. Threshold scales with rank: a master figures
  out the shot direction after one hit, a novice after four or five. Cross-NPC
  accumulation means three different snipers each hitting one squadmate trips
  disclosure for the whole squad.

Squad combat state:
  Tracks per-squad state through IDLE, ALERTED, ENGAGED, FLEEING based on event
  volume. State drives retention scaling, scheme selection, flee gating. Per-squad
  scope: squadmates share situational awareness without depending on smart-terrain
  proximity.

Tactical flee:
  Outnumbered squads retreat to the nearest friendly smart terrain further from
  the enemy than the squad's current position. The strongest member stays as rear
  guard, others sprint to the destination. Smoke grenade masks the retreat. Per-rank
  cowardice multiplier on the trigger threshold. Monolith and zombied never flee
  by faction character. Squads at engaged smart terrains stay to defend.

Memory persistence:
  Danger memory windows extend during sustained engagement. Substrate-side: hit
  records stay disclosed for five minutes instead of one when squad is in ENGAGED
  state. Engine-side: provides an xr_danger.ltx with smart-state-keyed condlists
  extending danger_inertion. Effect: snipe a guard, retreat, wait five minutes,
  return. Guard still alert with weapons drawn instead of patrol idle.

Scheme selection:
  Provides a default_custom_data.ltx override switching NPCs between cover, camper, and
  default combat schemes based on recent hit window, squad combat state, distance to
  enemy, and rank. NPC-vs-NPC works by default.

Global combat tuning:
  Pushes about 16 engine combat cvars to tuned defaults at boot. Aim inertia, burst
  delay, hold position time, grenade throw delay, search inertia, target-switching
  delays, dispersion endpoints. MCM picks between Vanilla, Tactical, Hardcore, or
  Custom (per-cvar sliders) presets. No state coupling - the cvars stay set for the
  session because they are global engine state, not per-squad concerns.

Grenade flush:
  Vanilla NPCs in cover often cannot fire on hidden enemies because the engine
  raycast hits their own cover. The engine has full grenade-throwing AI built to
  flush such targets, but most loadouts have no grenades so the AI never triggers.
  AlifeTactics guarantees 1-2 grenades on enemy NPCs of rank experienced or higher
  and enables their grenade-throwing flag. Effect: cover-camping at the same
  position stops working after the first thirty seconds.

Stance and weapon bias:
  Crouched with a scoped weapon is 12-15 times more accurate than walking with iron
  sights. Engine auto-selection picks weapons reasonably but inconsistently. NPCs
  crouch sometimes but often stand exposed in cover. AlifeTactics overrides the
  weapon-choice callback to force scoped weapons at distance for higher-rank NPCs
  and the body-state callback to force crouch when cover supports a kneeling
  firing position.

NPC self-healing:
  Vanilla xr_eat_medkit contains a working code path for NPCs to consume medkits
  and bandages from their inventory when wounded, but vanilla provides the [plugin]
  section without the medkit/bandage item lists. parse_list returns empty, the
  for-loop never iterates, and NPCs never consume real items -- only a one-shot
  healing_charge fallback fires for half of stalkers. AlifeTactics adds the
  missing item lists via a DLTX overlay (medkit, medkit_army, medkit_scientic,
  the three medkit_ai variants, and bandage), restoring the intended behavior
  without changing vanilla script. Two runtime tuning layers on top: heal rate
  multiplier (scales the per-tick HP recovery, 0.5x-3.0x), and per-rank
  healing-charge probability (replaces vanilla's flat 50% roll with four
  per-tier sliders for novice/experienced/veteran/master). Defaults match
  vanilla behavior exactly; sliders are opt-in tuning.

Performance:
  All squad-aware behaviors are O(squads online) at worst, not O(NPCs online).
  Per-frame work is bounded by the substrate decay tick (every 5 seconds) and the
  flee evaluation (every 2 seconds per squad). The hit callback fires only when
  someone gets shot. The vision rewrite is deferred precisely because get_visible_value
  runs every frame for every NPC-actor pair; the other subsystems land first.

Companion mods:

AlifePlus (reactive A-Life framework) -- https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeGuard (population control) -- https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeBalance (respawn pacing) -- https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance

AlifePlus is a soft dependency. Smart-state predicates that hook AlifePlus's n118
smart combat state degrade gracefully (return false) if AlifePlus is absent.

Requirements:
- Anomaly 1.5.3 with Modded exes (themrdemonized fork)
- xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
- MCM
