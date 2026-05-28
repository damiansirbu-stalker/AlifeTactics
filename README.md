# AlifeTactics: Combat AI for STALKER Anomaly

Vanilla NPCs are not threatening. They miss most of their shots. They walk to cover with the weapon raised. Each one discovers threats on their own. They never retreat tactically. They cannot deal with a player sitting behind a rock at 80 meters.

AlifeTactics is a from-scratch combat AI replacement built on a shared per-squad memory table. Hit disclosure, tactical flee, danger persistence, and combat state all read and write that one table. Per-NPC tuning layers cover precision, health, and damage.

[Bugs, suggestions](https://github.com/damiansirbu-stalker/AlifeTactics/issues)

Requires: Anomaly 1.5.3, Modded exes (themrdemonized fork), [xlibs](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001), MCM

Companion mods:
- AlifePlus ([ModDB](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01)) — reactive A-Life framework. Soft dependency: smart-state predicates degrade gracefully if absent.
- AlifeGuard ([ModDB](https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001)) — population control
- AlifeBalance ([ModDB](https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance)) — respawn pacing

## Documentation

- [readme.txt](doc/readme.txt) — full description, features
- [architecture.md](doc/architecture.md) — substrate-based combat AI architecture
- [changelog](doc/changelog) — version history

## License

PolyForm Perimeter License. See [LICENSE](LICENSE).
