AlifeTactics: NPC combat behavior for STALKER Anomaly, by Damian
Version: 1.0.0 (xlibs 1.7.6, demonized 20260601)
GitHub: https://github.com/damiansirbu-stalker/AlifeTactics
Changelog: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/changelog
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeTactics/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeTactics/issues

Alife Collection:
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics

! Reset MCM settings to defaults after updating !

https://www.youtube.com/watch?v=eKpzbmFOFC8

AlifeTactics is a collection of fixes and systems focused on NPC combat behavior in STALKER Anomaly.
Defaults match vanilla so the fixes apply on their own with no behavior shift you don't want.

Hit Sharing:

On the first faction-enemy hit, AlifeTactics arms every online squadmate against the shooter, audio range or not. \
Every member's personal goodwill toward the shooter is forced to hostile, the shooter is registered in every member's memory, and the squad's combat-mask bit
is set so the engine's own memory propagation carries the hit across the rest of the squad. 
New squad members spawned mid-fight inherit the squad's active disclosures, and previously offline shooters coming back online get re-disclosed
to the squads tracking them, so engagement state stays consistent across spawn churn.
After 2 game minutes of no further hits from that shooter the squad's pin expires and the next hit re-fires the alert.

The Stealth toggle (MCM, default on) suppresses the squad alert when the hit kills the victim.
Silenced shots, scoped-rifle headshots, and backstabs no longer disclose the shooter to the surviving squadmates.
Squadmates still learn through gunshot sound, corpse discovery, and line of sight.

Healing:

Vanilla and most modpacks rely on a magic medkit use that triggers unreliably, and bandages don't work at
all. About half of NPCs get a one-shot flag at register that lets them heal HP once without consuming any
item, and the flag re-rolls on every save load. Bandages have no equivalent fallback, so bleeding NPCs
in vanilla bleed out unless they happen to have a bandage AND the inventory list is populated. Vanilla
doesn't populate it. AlifeTactics fixes both.

Wounded stalkers below 50% HP consume medkits from their inventory. Bleeding stalkers above the wound
threshold consume bandages. Stalkers carrying neither fall back to a per-rank lifetime healing charge.
The heal rate is MCM-tunable.

Animations were also fixed and are toggleable in MCM. Stalkers below 65% HP visibly limp when out of combat
through a torso overlay the engine layers over normal locomotion, re-armed every 5 seconds per NPC. A
medkit-injection or bandage-application torso animation plays as a one-shot cue when a stalker starts a
heal cycle.

Accuracy:

The engine's rank-based accuracy curve is dead on Anomaly's rank values, making all stalker ranks equal in practice.
AlifeTactics hooks engine internals and allows that to function, while also making it configurable.

AlifeTactics hooks the engine callbacks and applies a per-rank dispersion factor to every NPC shot, MCM configurable.
Masters shoot tighter than novices, and the full spread is configurable per tier. 

Stance Switch:

Stalkers crouch or stand when the engine combat planner picks a static-cover firing or peek action.
The stance carries into killing, waiting in cover, and ambush operators through the engine's body-state inheritance.
The engine calls AT's stance functor on 11 different combat actions. Only 2 of them (look out, hold position) are static-cover firing ops where crouching makes sense. AT overrides those 2, the other 9 pass through unchanged.
The system also considers equipped weapon. Long-range rifle carriers (DMRs, battle rifles, bolt-actions like SVD, SVT40, Mosin, SKS, M82) crouch in cover; short-range carriers pass through with whatever stance the engine picked.

Squad Camper:

Long-range rifle carriers (DMRs, battle rifles, bolt-actions like SVD, SVT40, Mosin, SKS, M82, SR25, and the rest of the kind=w_sniper LTX class) hold cover and fire continuously while they see you, instead of running the standard combat cycle.
A configurable share of other stalkers also holds cover; the rest run the vanilla cycle (take cover, peek, hold, detour, kill).
Pistol, SMG, and shotgun carriers never hold cover regardless of the slider — those weapons are meant to close distance, not plant.
The result is a mix of pinners holding cover (long-range rifle + share of mid-range carriers) and movers advancing on you (close-range weapon carriers + the rest).

Three gates keep the cover-fire behavior in the situations where it makes sense:
Below about 25 meters the cover-fire role is disabled and the NPC runs the vanilla cycle. Planting in cover at point-blank range is exploitable by flanking; vanilla's reactive movement handles short range better.
When a camper loses line of sight to the enemy, the role drops within half a second and the NPC moves to re-acquire instead of rotating in place or freezing in cover.
When the engine's enemy memory expires (no recent sight, sound, or hit), the camper role drops and the NPC returns to idle.

MCM Squad Camper tab: master toggle and camper share slider (0.1-0.9, default 0.50).
At default 0.50 about half of non-sniper stalkers hold cover at qualifying ranges, the other half run the vanilla cycle.
Lower the camper share for aggressive squads (most NPCs advance), raise it for defensive squads (most NPCs plant).
Sniper-class carriers ignore this slider but still respect the line-of-sight and distance gates.

The system drives the vanilla xr_combat_camper sub-scheme that ships with Anomaly but is dormant by default.
Compatible with GAMMA AI Rework and AI more cover (Mora). Both install their own per-NPC condlist into the same engine seam; AlifeTactics installs its own predicate per stalker, overriding theirs. When AlifeTactics is disabled in MCM, the predicate returns false and the vanilla planner runs.

Danger:

A full-file override of Anomaly's xr_danger.script with bug fixes always-on and three improvements toggleable in MCM > Danger.

Six bug fixes are always on.
Three danger categories (direct hit, bullet ricochet, attacked nearby) were silently reading the wrong config row and reacting identically.
Mutant corpses crashed the evaluation on death-time reads.
The evaluator crashed when called on a torn-down NPC reference.
The evaluator also crashed when corpse death-time returned a non-numeric value.
Danger-state transitions reset only the upper-body animation tier and left a stale lower-body pose visible across the change.
The hit callback referenced an undefined variable and silently dropped responses on every hit.

Danger-state transitions snap into place instead of soft-blending so the reaction delay across state changes is gone.

The paired xr_danger.ltx widens corpse inertion to 15 game minutes (vanilla 12 seconds) and ricochet inertion to 10 minutes.
Detection distances respond to weather so stalkers see less in storms.

MCM-toggleable improvements (default on).
Hits override combat_ignore so allies of allies still get a danger response.
Gunshot reports register through the script danger pipeline gated on whether the actor is aiming.
Actor-sourced danger uses its own inertion and ignore tables so encounters with the player tune independently of NPC-vs-NPC.

Performance:

Most systems above hook on engine callbacks but are throttled and situational.
All operations are done through xlibs which encapsulates best practices in workign with the engine and what Anomaly uses internally.
Also all operations are traced for performance and duration and can be checked if debug logging is activated.
Tests showed the perfromance impact is almost nonexistent.

Compatibility:

Tested with vanilla Anomaly 1.5.3 and GAMMA. One base script override: xr_danger.script (vanilla file with bug fixes and three toggleable
improvements; see MCM > Danger). No engine patches. Mid-save install and uninstall both work. Friendly fire and same-community hits are
filtered at the faction-relation gate, so story NPCs, companions, traders, and squadmates are never armed against their own faction.

Disable before installing AlifeTactics:

Redundant with engine settings: Dynamic AI Aim Settings, DLTX_JURASZKA Worse NPC vision and accuracy. PR #523 exposes the aim, vision, and
search-inertia cvars in engine Settings, and AT's accuracy curve also writes per-shot dispersion. Either path stacks on the other.

Forced-movement schemes with stuck-NPC risk: Wuut AI Extension, NPC_Fleeing. Both graft forced movement onto the combat planner.

NPC self-heal overlaps: NPC Limping and Healing (Vodoxleb), Animated NPC Healing, NPC Animation Overhaul Part 1. Parallel heal schemes
double-heal, full-file animation overrides conflict, combat planners gated on the heal evaluator can freeze NPCs after combat.

G.A.M.M.A. AI Rework: ships its own xr_danger.script and xr_danger.ltx overrides. GAMMA AI Rework cannot be cleanly disabled in current GAMMA
builds (removal crashes), so the recommended setup is to load AlifeTactics AFTER GAMMA AI Rework in MO2. Our xr_danger files win the overlay,
GAMMA's other AI rework files (xr_combat_camper, xr_conditions, schemes_ai_gamma, default_custom_data.ltx) continue to run unchanged. Net:
AlifeTactics's xr_danger fixes + GAMMA's other AI tactics. Tested working with this load order.

Compatible but overlapping with AT Squad Camper:

AI more cover (Mora): installs its own condlist into the vanilla combat_type seam. AT installs its predicate into the same seam on each
stalker net-spawn and overrides Mora's. Mora's behavior is replaced; uninstall AT to restore.

G.A.M.M.A. AI Rework (the camper side): same seam, same overlap. AT's predicate per stalker replaces theirs. When AT is disabled in MCM,
the predicate returns false and the vanilla planner runs (GAMMA's condlist is not restored mid-session, but the engine returns to vanilla
behavior).

ReDone Combat AI: copies Mora, overrides 12 scripts. Much is 1.5.2-gated and skipped on 1.5.3.

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
