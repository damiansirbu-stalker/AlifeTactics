# AlifeTactics: NPC combat behavior for STALKER Anomaly

Vanilla squads do not share hit awareness past audio range. A suppressed shot or a hit on a distant patrolman stays a private problem of one stalker while the rest keep walking. Vanilla stalkers also carry medkits they never use, because the engine ships the consumption config with an empty item list.

AlifeTactics is a combat AI mod with squad-scope and per-stalker behaviors. The squad-scope layer shares hit awareness: on the first faction-enemy hit, the entire squad is force-disclosed to the shooter and engages, regardless of audio range. Two per-stalker layers sit alongside. One restores the broken medkit consumption path and tunes heal rate plus per-rank healing-charge probability. The other applies a per-rank weapon accuracy curve so master stalkers shoot noticeably tighter than novices (the vanilla engine rank knob is dead on Anomaly's rank values).

[ModDB](TBD) | [Nexus](TBD) | [Releases](https://github.com/damiansirbu-stalker/AlifeTactics/releases) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/AlifeTactics/issues)

Requires: Anomaly 1.5.3, [demonized 20260601+](https://github.com/themrdemonized/xray-monolith), [xlibs 1.5.2](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001), MCM

Alife Collection:
- [AlifePlus](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01)
- [AlifeBalance](https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance)
- [AlifeGuard](https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001)
- [AlifeTactics](TBD)

## Documentation

- [readme.txt](doc/readme.txt): full description, features
- [architecture.md](doc/architecture.md): substrate-based combat AI architecture
- [changelog](doc/changelog): version history

## License

PolyForm Perimeter License. See [LICENSE](LICENSE).
