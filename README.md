# AlifeTactics: Combat AI for STALKER Anomaly

Vanilla squads do not share hit awareness past audio range. A suppressed shot or a hit on a distant patrolman stays a private problem of one stalker while the rest keep walking. Vanilla stalkers also carry medkits they never use, because the engine ships the consumption config with an empty item list.

AlifeTactics is a combat AI mod built on a shared per-squad memory table. On the first faction-enemy hit, the entire squad is force-disclosed to the shooter and engages, regardless of audio range. A separate per-stalker layer restores the broken medkit consumption path and adds MCM tuning for heal rate and per-rank healing-charge probability.

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
