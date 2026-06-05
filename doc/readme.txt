AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: 1.0.0 (xlibs 1.5.2, demonized 20260601)
GitHub: https://github.com/damiansirbu-stalker/AlifeTactics
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeTactics/issues

Alife Collection:
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeTactics: TBD

! Reset MCM settings to defaults after updating !

AlifeTactics is a mod composed of several systems, that gives NPC squads coordinated combat behavior.

Squad alarm on hit:
  Vanilla shares hit memory across squadmates within earshot. A suppressor breaks the share. A patrol member thirty meters off the line falls outside its range. The hit stays a private problem of whoever took it.
  On the first faction-enemy hit, AlifeTactics writes the shooter into every squadmate's memory as hostile, audio range or not. The engine combat planner then handles target selection, cover, and return fire on the new information.
  Shooting one stalker from cover or with a suppressor no longer leaves his squad standing around. They turn and engage on the first hit, including the ones who did not see or hear it.

NPC self-healing (medkits AND bandages):
  Vanilla has a working code path for NPCs to consume medkits and bandages from their inventory, but ships the underlying configuration with an empty item list. The consumption loop never iterates, and only a once-per-life fallback ever heals. NPCs carry full medkits and bandages and die clutching them.
  AlifeTactics restores BOTH item lists through a configuration overlay covering the six vanilla medkit sections (medkit, medkit_army, medkit_scientic, medkit_ai1/2/3) and the vanilla bandage. MCM controls on top: heal-rate multiplier, four per-rank charge-chance sliders for the lifetime fallback (novice, experienced, veteran, master) replacing vanilla's flat 50%, and toggles for limping and the heal-cue animation. Defaults match vanilla behavior, so out of the box you only get the fix.
  What actually happens now:
    - Wounded stalkers below 50% HP use medkits to heal up.
    - Bleeding stalkers above the wound threshold use bandages to stop the bleed.
    - Stalkers without either fall back to the lifetime healing charge if their rank rolls allow.
  Visual cues (cosmetic, MCM-toggleable):
    - Stalkers below 65% HP visibly limp when out of combat. A torso slump overlay (visible whether standing or walking; engine drives the legs normally). Re-armed every 5 seconds per NPC.
    - Stalkers play a medkit-injection or bandage-application torso animation when they start a heal cycle. One-shot cue per cycle.
    - Combat NPCs are excluded from limping (mental state changes to danger in combat). Heal cues play in or out of combat.

NPC weapon accuracy:
  In vanilla every stalker fires with the same dispersion regardless of rank. The engine has a rank-based accuracy curve but it is broken on Anomaly's data: every non-novice stalker clamps to the same value, so a master shoots no tighter than a trainee. The rank dispersion knob in the engine is a dead knob.
  AlifeTactics replaces that curve script-side. On every NPC shot the rank is read and a per-tier dispersion factor is applied to the engine's computed cone. Defaults run from novice at the vanilla baseline down to legend at roughly a third the cone width. Eight tier sliders in MCM tune each rank independently.
  Master stalkers shoot noticeably tighter than novices. The spread between ranks is configurable per tier, so you can flatten the curve, exaggerate it, or set extremes (laser masters, hopeless rookies).

Combat crouch:
  Vanilla NPC stance during combat is inherited from current state. A stalker that walked to cover standing stays standing in cover, and gets headshot through the chest-high crate they thought was protecting them.
  AlifeTactics tells snipers and rocket-launcher carriers to crouch when the engine combat planner selects a static-cover firing or peek action (LookOut, HoldPosition). The crouch carries forward into KillEnemy, WaitInCover, and HoldAmbushLocation through the engine's body_state inheritance. These weapons are long-range sustained-fire tools; crouching gives low silhouette and stable sustained shooting.
  Other weapons keep vanilla stance. Riflemen, pistoleers, shotgunners, and SMG carriers stand up to move and fire normally. The crouch is meaningful: it marks the squad's fire base by weapon type alone, no role taxonomy needed.

MCM:
  Five tabs, each with a master toggle. Hit Sharing: master toggle, forget-shooter timer in game minutes. Healing: master toggle, heal-rate multiplier, four per-rank charge-chance sliders, limping toggle + threshold, heal-cue animation toggle. Accuracy: master toggle, per-tier dispersion sliders for the eight rank tiers. Stance Switch: master toggle. Development: log level, reset-to-defaults button.

Backlog:
  Tactical flee, danger memory persistence, combat scheme selection, NPC weapon bias. The backlog file on GitHub tracks each.

Performance:
  Hit Sharing fires on the engine hit callback and scales with shots fired against squads. Per-NPC behaviors (Accuracy, Healing, Stance Switch) fire on their natural engine events: accuracy per bullet, stance per body-state set, healing tick per time-factor. None of these iterate the NPC list every frame. One periodic cleanup tick (disclosure decay, every 5 real seconds) walks tracked squads and prunes expired shooter entries.

Compatibility:
  Tested with vanilla Anomaly 1.5.3 and GAMMA.
  No base script edits. No engine patches.
  Mid-save install works. Mid-save uninstall is safe.
  Friendly fire and same-community hits are rejected at the faction-relation gate before any squad-memory write or disclosure. Story NPCs, companions, and traders go through the same gate as every other stalker. A bandit hitting a friendly stalker still arms that stalker's squad against the bandit; the friendly stalker hitting their own squadmate is ignored.

Disable before installing AlifeTactics:

  Redundant with engine settings:
    Dynamic AI Aim Settings, DLTX_JURASZKA Worse NPC vision and accuracy. Demonized PR #523 exposes the underlying ai_aim_* / ai_vision_speed_boost / ai_search_inertia_time cvars in the engine Settings > Stalkers menu. AlifeTactics's accuracy hook also writes per-shot dispersion via the npc_shot_dispersion callback. Either path stacks on the other.

  Forced-movement schemes (stuck-NPC risk):
    Wuut AI Extension. Grafts a forced-movement precondition onto the engine combat planner; includes its own stuck-detection because forced movement during combat can leave NPCs stuck.
    NPC_Fleeing. Squad flee via forced movement; same stuck risk, includes its own stuck-detection.

Compatible today, will overlap once the per-NPC camper scheme is added:
  AI more cover (Mora). Assigns the vanilla camper combat scheme via global default_custom_data.ltx [combat] combat_type. Engine-documented machinery. AlifeTactics's planned per-NPC camper writes script_combat_type from a binder; two writers on the same field.
  G.A.M.M.A. AI Rework. Layered scheme selector on Mora's pattern with rank-weighted dispatch. Overrides xr_combat_camper, xr_conditions, xr_danger, schemes_ai_gamma. The xr_combat_camper override removes action_look_around, which AlifeTactics's planned camper depends on to scan when sight breaks. The xr_danger override file-replaces a script AlifeTactics will surgically monkey-patch.
  ReDone Combat AI. Copies Mora's pattern; overrides 12 vanilla scripts. Much of its logic is 1.5.2-gated and skipped on 1.5.3.

Requirements:
Anomaly 1.5.3
demonized 20260601+ (https://github.com/themrdemonized/xray-monolith)
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
