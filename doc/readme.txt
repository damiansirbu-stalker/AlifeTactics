AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: 1.0.0 (xlibs 1.5.2)
GitHub: https://github.com/damiansirbu-stalker/AlifeTactics
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
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

NPC self-healing:
  Vanilla has a working code path for NPCs to consume medkits and bandages from their inventory, but ships the underlying configuration with an empty item list. The consumption loop never iterates, and only a once-per-life fallback ever heals. NPCs carry full medkits and die clutching them.
  AlifeTactics restores the missing item list through a configuration overlay covering the vanilla medkits and the bandage. Two MCM sliders tune on top: one scales how fast NPCs heal, one sets a per-rank chance for the lifetime fallback. Defaults match vanilla behavior, so out of the box you only get the fix.
  Wounded stalkers actually heal themselves now. Stalkers carrying medkits use them. Stalkers without one fall back to the charge if their rank rolls allow.

MCM:
  Three tabs. Squad Tactics: substrate retention, state machine timings, hit disclosure toggle. Stalker Healing: medkit restoration (info-only, boot-time data layer), heal rate multiplier, per-rank healing-charge probability. Development: log level.

Backlog (not in 1.0.0):
  Tactical flee, danger memory persistence, combat scheme selection, global combat tuning, stance and weapon bias, per-NPC rank-aware dispersion. The backlog file on GitHub tracks each.

Performance:
  Squad-aware behaviors scale with squad count. NPC count does not enter the cost. Two periodic cleanup ticks run every few seconds. Nothing runs on every frame and nothing polls.

Compatibility:
  Tested with vanilla Anomaly 1.5.3 and GAMMA.
  No base script edits. No engine patches.
  Mid-save install works. Mid-save uninstall is safe.
  Story NPCs, companions, and traders go through the same faction-relation gate as every other stalker, so they do not get caught by the squad alarm.

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
