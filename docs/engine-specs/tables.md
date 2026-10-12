# Asset tables (`tables\*.tbl`)

Status: measured. The shell's path/id/name tables are decoded and validated byte-exact across five tables. Facts come from independent measurement; community material was not used.

## Provenance

| Item | Value |
|---|---|
| Source | Vengeance `CORE.MW4`, SHA-256 `411883d06236ff3fc1f0acba9ae52d411bb84949af9f09273a8a2c2115497abd` |
| Extraction | `.local/re/vbd_list.py --dump` into the gitignored `.local/re/dump/` scratch |
| Files | `tables\mechtable.tbl` (1,057 B), `mechchassistable.tbl` (973), `weaponstable.tbl` (4,325), `subsystemtable.tbl` (285), `skinstable.tbl` (992), `missionnames.tbl` (3,742) |

## Record format

**Path tables** (`mechtable`, `mechchassistable`, `weaponstable`, `subsystemtable`) — one record per asset:

```
u32 pathLength; char path[pathLength]; u8 0;
u16 1;                       // occurrence/reference count, observed constant
u16 id;                      // runtime id assigned to the asset
u32 nameLength; char name[nameLength]; u8 0;
```

Validation: parses exactly to the last byte with no residue — 21/21 mechs, 21/21 chassis, 57/57 weapons, 6/6 subsystems. Every referenced path exists in the archive (spot-checked: `Mechs\Argus\Argus.instance`, `Mechs\Atlas\Atlas.data`, `WeaponSubsystems\ACWeaponSubsystem\AC5.data`, `Subsystems\JumpJetSubsystem.data`).

**Skin table** — same family, id-first layout:

```
u32 1; u16 id; u32 1;        // flags/counts, observed constant
u32 nameLength; char name[nameLength]; u8 0;
```

Validation: 37/37 records, clean end. Ids: `95` = `None`, then `48`.. as `01-OpsBlue`, `02-OpsRed`, `03-OpsYellow`, … `36-Arrogant1` (`122`).

## Measured contents

| Table | Records | Id notes |
|---|---|---|
| `mechtable.tbl` | 21 mechs | 914 = Argus, 922 = Atlas, 930 = Awesome (steps of 8) |
| `mechchassistable.tbl` | 21 chassis | 909 = Argus, 917 = Atlas, 925 = Awesome — every chassis id is the mech id − 5 |
| `weaponstable.tbl` | 57 weapons | 866 = AC5, 865 = AC10, 864 = ArtilleryStrike (ids descend with table order) |
| `subsystemtable.tbl` | 6 subsystems | 795 = JumpJets, 790 = AMS, 796 = LAMS |
| `skinstable.tbl` | 37 skins | see above |

`missionnames.tbl` (3,742 B) is the same path-table family but interleaves an extra `u32` between some records; its exact layout is open.

## Per-mech records (`.subsystems`, `.damage`)

Both use the same tagged-record family as mech `.contents` (`erf-geometry.md`): `[u16 20][u16 0x70]` then `u32 2, u32 2, u32 0`, a type id (`0x421` = 1057 for contents, `0x431` = 1073 for subsystems), an identity-matrix float block, and size-prefixed sub-records (for example `[u32 4][u32 152]` inside `.subsystems`).

- **`<mech>.subsystems`** (Mad Cat: 5,018 B): the mech's equipment loadout; record boundaries are size-prefixed and sub-record ids appear in the 900s.
- **`<mech>.damage`** (Mad Cat: 1,688 B): 21 per-joint records (each joint named twice), pairs of sub-records:
  - Armor block: `f32 1.0, 1.0`, a ratio (`0.14`), the length-prefixed joint name, counts `2, 2`, then `f32 armor` (`42.0` for the large arm), an id (1,162), and two structure floats (`21.0, 21.0`).
  - Transfer block: joint name, count `4`, then four entries `[u16 1][u16 location]` with `f32` weights — measured locations `224–227` with weights `0.99, 0.75, 0.5, 0.25` (damage transfer to hit locations).

## Open items

- `missionnames.tbl` record flag layout.
- Whether the adjacent id relations (chassis = mech − 5) hold across titles (Black Knight and Mercenaries archives).
- The `u16 1` field's meaning (reference count or type tag) and id-assignment order across the full shell.
- Per-mech record layouts: `.subsystems` loadout entries, `.damage` field semantics (the ratio, ids, and the hit-location id space), and `.instance` (344 B).
