# AlifeTactics: NPC combat behavior for STALKER Anomaly

Independent systems that change how stalkers fight: an intermittent combat takeover, per-rank accuracy, squad-wide hit disclosure, a danger scheme rework, friendly-fire protection, real self-healing, NPC jam removal, and NPC armor-piercing ammo.
It overrides no combat files, so it layers onto vanilla, GAMMA, and the AI overhauls.

[ModDB](https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics) | [Nexus](TBD) | [Releases](https://github.com/damiansirbu-stalker/AlifeTactics/releases) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/AlifeTactics/issues)

Requires: Anomaly 1.5.3, [demonized 20250908+](https://github.com/themrdemonized/xray-monolith) OR [AOEngine v0.55+](https://github.com/Mirrowel/AOEngine-Assets), [xlibs 1.7.6](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001), MCM

## Systems

Every system has its own MCM page with a master toggle, grouped as in the menu.

### Combat

Maneuvers: borrows one stalker for one committed maneuver where vanilla dithers, then hands control back.
Kite reopens range, retreat pulls to rear cover, flee routs the last man to a distant friendly base, snipe holds a marksman on precision aim.
A maneuver runs only when it gains the stalker something, and when no move pays, vanilla fights on untouched.

### Effectiveness

Accuracy: a real per-rank dispersion curve, tunable per tier.
Anomaly's own rank curve collapses every NPC to the same value, so rank never touched aim.

Disclosure: wounding one squad member turns the whole squad on the shooter, distant patrols and silenced hits included.
A survivor gate keeps a clean kill silent.

Danger: a rework of the danger scheme, patched at runtime onto whichever danger script the modpack ships.
Always-on crash and misread fixes, plus three optional improvements.

Crossfire: same-faction hits deal no damage, keyed on actual relation.
Hostile pairs still trade fire, and your own shots are never touched.

### Mechanics

Healing: wounded NPCs spend the medkits and bandages they carry, with limp and heal animations.
A per-rank charge covers the empty-handed.

Jamming: removes the per-rank NPC jam roll that chokes even pristine rifles.
Your own jams stay vanilla.

Ammo: veteran and higher NPCs fire the armor-piercing rounds they carry until they run out, and drop none of it as loot.

### Development

Log level and a live combat debug HUD.

The Behaviors, Commitment, Reaction, Range, and Resistance pages and the Effects and Mutants categories are reserved for systems still to come.

## Alife Collection

- [AlifePlus](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01)
- [AlifeBalance](https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance)
- [AlifeGuard](https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001)
- [AlifeTactics](https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics)

## Documentation

- [readme.txt](doc/readme.txt): full description, features
- [architecture.md](doc/architecture.md): combat AI architecture
- [changelog](doc/changelog): version history

## License

PolyForm Perimeter License. See [LICENSE](LICENSE).
