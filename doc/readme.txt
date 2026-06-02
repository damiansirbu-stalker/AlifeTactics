AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: 1.0.0 (xlibs 1.5.2)
GitHub: https://github.com/damiansirbu-stalker/AlifeTactics
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeTactics/issues

! Reset MCM settings to defaults after updating !

AlifeTactics is a mod composed of several systems, that gives NPC squads coordinated combat behavior.

Squad memory:
  In vanilla each NPC tracks his own threats. The squad has no shared brain. A hit on one stalker stays a problem for one stalker, and the rest of the squad keeps walking.
  Squad memory is one shared table per squad. Every other combat system writes its observations there and reads them back. Hits, shooters, current state, all in one place.
  This is the backbone of the mod. Squads stop acting like four lone wolves wearing the same patches and start acting like a unit with shared awareness. Realistic squad combat is collaborative, and without a shared memory there is nothing to collaborate on.

Squad alarm on hit:
  Vanilla shares hit memory across squadmates within earshot. A suppressor breaks the share. A patrol member thirty meters off the line falls outside its range. The hit stays a private problem of whoever took it.
  On the first faction-enemy hit, AlifeTactics writes the shooter into every squadmate's memory as hostile, audio range or not. The engine combat planner then handles target selection, cover, and return fire on the new information.
  Shooting one stalker from cover or with a suppressor no longer leaves his squad standing around. They turn and engage on the first hit, including the ones who did not see or hear it.

NPC self-healing (medkits AND bandages):
  Vanilla has a working code path for NPCs to consume medkits and bandages from their inventory, but ships the underlying configuration with an empty item list. The consumption loop never iterates, and only a once-per-life fallback ever heals. NPCs carry full medkits and bandages and die clutching them.
  AlifeTactics restores BOTH item lists through a configuration overlay covering the six vanilla medkit sections (medkit, medkit_army, medkit_scientic, medkit_ai1/2/3) and the vanilla bandage. Two MCM sliders tune on top: one scales how fast NPCs heal, one sets a per-rank chance for the lifetime fallback. Defaults match vanilla behavior, so out of the box you only get the fix.
  What actually happens now:
    - Wounded stalkers below 50% HP use medkits to heal up.
    - Bleeding stalkers above the wound threshold use bandages to stop the bleed.
    - Stalkers without either fall back to the lifetime healing charge if their rank rolls allow.
  Visual cues (cosmetic, MCM-toggleable):
    - Stalkers below 65% HP visibly limp when out of combat. A torso slump overlay (visible whether standing or walking; engine drives the legs normally). Re-armed every 20 seconds per NPC.
    - Stalkers play a medkit-injection or bandage-application torso animation when they start a heal cycle. One-shot cue per cycle.
    - Combat NPCs are excluded from limping (mental state changes to danger in combat). Heal cues play in or out of combat.

NPC weapon accuracy:
  In vanilla every stalker fires with the same dispersion regardless of rank. The engine has a rank-based accuracy curve but it is broken on Anomaly's data: every non-novice stalker clamps to the same value, so a master shoots no tighter than a trainee. The rank dispersion knob in the engine is a dead knob.
  AlifeTactics replaces that curve script-side. On every NPC shot the rank is read and a per-tier multiplier is applied to the engine's computed cone. Defaults run from novice at the vanilla baseline down to legend at roughly a third the cone width. Eight tier sliders in MCM tune each rank independently.
  Master stalkers shoot noticeably tighter than novices. The spread between ranks is configurable per tier, so you can flatten the curve, exaggerate it, or set extremes (laser masters, hopeless rookies).

Combat crouch:
  Vanilla NPC stance during combat is inherited from current state. A stalker that walked to cover standing stays standing in cover, and gets headshot through the chest-high crate they thought was protecting them.
  AlifeTactics tells snipers and rocket-launcher carriers to crouch when the engine selects a static cover action: firing at the enemy, holding position, peeking from cover, waiting in cover, ambushing. These weapons are long-range sustained-fire tools; crouching gives low silhouette and stable sustained shooting.
  Other weapons keep vanilla stance. Riflemen, pistoleers, shotgunners, and SMG carriers stand up to move and fire normally. The crouch is meaningful: it marks the squad's fire base by weapon type alone, no role taxonomy needed.

MCM:
  Five tabs, each with a master toggle. Hit Sharing: master toggle, per-shooter memory retention. Healing: master toggle, medkit restoration (info-only, boot-time data layer), heal rate multiplier, per-rank healing-charge probability. Accuracy: master toggle, per-tier dispersion sliders for the eight rank tiers. Stance Switch: master toggle. Development: log level.

Backlog:
  Tactical flee, danger memory persistence, combat scheme selection, NPC weapon bias. The backlog file on GitHub tracks each.

Performance:
  Squad-aware behaviors scale with squad count. NPC count does not enter the cost. Two periodic cleanup ticks run every few seconds. Nothing runs on every frame and nothing polls.

Compatibility:
  Tested with vanilla Anomaly 1.5.3 and GAMMA.
  No base script edits. No engine patches.
  Mid-save install works. Mid-save uninstall is safe.
  Story NPCs, companions, and traders go through the same faction-relation gate as every other stalker, so they do not get caught by the squad alarm.

Disable before installing AlifeTactics:
  AI more cover (Mora): assigns a camper combat scheme via global default_custom_data.ltx [combat] combat_type. The camper action_shoot fires while the NPC has visual on the enemy (state_mgr hide_fire from current cover); when visual breaks, the vanilla engine combat planner takes over. The global LTX overlay overlaps with anything that touches smart-terrain combat selection.
  G.A.M.M.A. AI Rework: layered scheme selector built on Mora's camper pattern. Routes to one of three sub-modes (cover for melee weapons, camper for rifles/snipers/launchers, monolith-specific behaviors) with rank-based dumbness rolls, distance-banded memory timeouts, and faction-aware logic. Overlays default_custom_data.ltx and overrides four vanilla scripts (xr_combat_camper, xr_conditions, xr_danger, schemes_ai_gamma); overlaps the same scripts AlifeTactics integrates with.
  ReDone Combat AI: copies Mora's camper pattern and overrides 12 core vanilla scripts. Much of its logic is version-gated for 1.5.2 and skipped on 1.5.3.
  Wuut AI Extension: injects a forced-movement scheme that grafts a precondition on the engine combat planner. Forced movement during combat can cause stuck NPCs; Wuut ships explicit stuck-detection logic.
  NPC_Fleeing: implements squad flee via forced movement. Like other forced-movement combat schemes, NPCs can get stuck mid-flee and the mod ships stuck-detection logic.
  Dynamic AI Aim Settings / DLTX_JURASZKA Worse NPC vision and accuracy: redundant since Demonized fixed the underlying engine settings in PR #523.

Companion mods:

AlifePlus (reactive A-Life framework): https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeGuard (population control): https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeBalance (respawn pacing): https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance

Requirements:
Anomaly 1.5.3
Demonized modded exes (latest), main or MT
xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Install (MO2):
1. Install xlibs
2. Install AlifeTactics
3. Load order does not matter
4. Configure via MCM

Uninstall (MO2):
Disable or remove in MO2.

Credits:
Altogolik - support, ideas, source materials

Usage and License:
Modpacks: allowed and encouraged. Keep the readme and license files.
Addons, patches, integrations: allowed. Credit "AlifeTactics by Damian Sirbu" visibly on your mod page.
Reproducing the implementation in other software: not allowed, even with credit.
Full license in LICENSE file and on GitHub.
