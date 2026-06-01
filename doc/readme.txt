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

NPC weapon accuracy:
  In vanilla every stalker fires with the same dispersion regardless of rank. The engine has a rank-based accuracy curve but it is broken on Anomaly's data: every non-novice stalker clamps to the same value, so a master shoots no tighter than a trainee. The rank dispersion knob in the engine is a dead knob.
  AlifeTactics replaces that curve script-side. On every NPC shot the rank is read and a per-tier multiplier is applied to the engine's computed cone. Defaults run from novice at the vanilla baseline down to legend at roughly a third the cone width. Eight tier sliders in MCM tune each rank independently.
  Master stalkers shoot noticeably tighter than novices. The spread between ranks is configurable per tier, so you can flatten the curve, exaggerate it, or set extremes (laser masters, hopeless rookies).

Squad roles:
  When combat starts each squad member gets a role based on the squad's size and their weapon. Snipers and rocketeers become suppressors (long-range fire from cover). Shotgunners, pistoleers, and SMG carriers become assaulters (they need to close the distance to be effective). Riflemen become flankers (mid-range versatile, arc around). The squad commander is always the lead.
  Distribution scales with squad size. A two-stalker squad has a leader and a fire base. A four-stalker squad adds an assault role and a flanker. Six-stalker squads spread further: two suppressors holding the fire base, one closing the distance, two arcing around to flank. Solos roll by weapon too: snipers always crouch as fire base, shotgunners and pistoleers stand as lead (they have no squad to assault for).
  Roles are assigned once on the first hit against the squad and persist until the squad despawns. New members joining mid-fight default to flanker.
  The behaviors below all read the role. Flanking and crouching are not always-on tricks but doctrinal positions held by specific squad members based on what they carry.

Squad flanking:
  Vanilla NPC cover selection is per-stalker. Each member scores its own cover with no awareness of squadmates, so squads end up stacked behind one wall on one loophole. There is no flanking, no enfilade.
  AlifeTactics dispatches the flankers and assaulters of an engaged squad toward angular positions around the disclosed enemy. Flankers arc to the left and right at sixty to ninety degrees from the enemy bearing. Additional flankers spread at inner angles. Assaulters close the distance straight forward.
  What you see: squads stop clustering on one cover and spread around the engagement. Within a few seconds of an engagement starting, members move to fan out. The lead and suppressors stay put and lay down fire from their initial positions.

Combat crouch:
  Vanilla NPC stance during combat is inherited from current state. A stalker that walked to cover standing stays standing in cover, and gets headshot through the chest-high crate they thought was protecting them.
  AlifeTactics tells suppressors to crouch when the engine selects a static cover action: firing at the enemy, holding position, peeking from cover, waiting in cover, ambushing. Suppressors are the squad's base of fire; they crouch for low silhouette and stable sustained shooting.
  Other roles do not crouch. Movers (lead, assaulter, flanker) keep standing to move fast under fire. The crouch is meaningful: it marks the member holding the fire base.

MCM:
  Five tabs, each with master toggles. Squad Memory: master toggle, substrate retention, state machine timings. Stalker Healing: master toggle, medkit restoration (info-only, boot-time data layer), heal rate multiplier, per-rank healing-charge probability. Weapon Accuracy: master toggle, per-tier dispersion sliders for the eight rank tiers. Combat Tactics: squad flanking toggle, combat crouch toggle. Development: log level.

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
  AI more cover (Mora): forces all ranged NPCs into a static camper scheme via [combat] combat_type, bypassing vanilla's context-based combat selection. NPCs always stand and fire from their initial position regardless of cover, distance, or rank.
  G.A.M.M.A. AI Rework: copies Mora's camper and replaces 60+ per-smart logic files with a single condlist. Long-standing bugs include broken NPC-vs-NPC behavior, save/load issues, db.storage collisions, and memory leaks.
  ReDone Combat AI: copies Mora's camper and overrides 12 core vanilla scripts, creating compatibility issues with other AI mods. Much of its logic is version-gated for 1.5.2 and skipped on 1.5.3.
  Wuut AI Extension: injects a forced-movement scheme that overrides the engine combat planner. Forced movement during combat causes stuck NPCs; Wuut ships explicit stuck-detection logic to mask it.
  NPC_Fleeing: implements squad flee via forced movement. Like other forced-movement combat schemes, NPCs get stuck mid-flee and the mod ships stuck-detection logic to mask it.
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
