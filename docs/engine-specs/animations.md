# Animation clips (`.mw4anim`)

Status: partially measured. The clip header, region table, and joint directory are decoded and verified on three clips; value channels and the track directory are partially decoded from the fully dumped water clip. Facts come from independent measurement; community material was not used.

## Provenance

| Item | Value |
|---|---|
| Source | Mercenaries `props.mw4` (14,527 entries; 2,375 `.mw4anim`, 48 `.animscript`, 1 `.animskript`) |
| Samples | `mechs\atlas\animation\atl_back.mw4anim` (6,679 B), `atl_backstand.mw4anim` (3,231 B), `missions\gladiatorpit\swamp_water_detail.mw4anim` (249 B) |
| Extraction | `.local/re/vbd_list.py --dump` into the gitignored `.local/re/dump/` scratch |

Mech clips live at `mechs\<mech>\animation\<clip>.mw4anim`; prop, vehicle, building, and mission clips follow the same pattern under their own trees (for example `buildings\air_control_tower\animation\actlight_rotate.mw4anim`, `missions\<mission>\<name>_water_detail.mw4anim`).

## Clip header (verified on all three samples)

Fields are byte-packed; a single pad byte at `+0x0c` misaligns everything that follows, so every later field starts one byte late.

| Offset | Type | Meaning | atl_back | atl_backstand | water_detail |
|---|---|---|---|---|---|
| `+0x00` | u32 | constant `0x200` | 512 | 512 | 512 |
| `+0x04` | u32 | constant `0x100` | 256 | 256 | 256 |
| `+0x08` | u32 | 0 | 0 | 0 | 0 |
| `+0x0c` | u8 | 0 (pad; shifts the fields below) | 0 | 0 | 0 |
| `+0x0d` | f32 | duration/blend scalar (seconds) | 1.2 | 0.0 | 3.333 |
| `+0x11` | u32 | file size − 1 (exact in all samples) | 6,678 | 3,230 | 248 |
| `+0x15` | u32 | 0 | 0 | 0 | 0 |
| `+0x19` | u32 | constant `100` | 100 | 100 | 100 |
| `+0x1d` | u32 | region 1 offset | 316 | 316 | 112 |
| `+0x21` | u32 | region 2 offset | 386 | 386 | 124 |
| `+0x25` | u32 | region 3 offset | 5,122 | 2,382 | 196 |
| `+0x29` | u32 | region 4 offset | 6,326 | 2,878 | 216 |
| `+0x2d` | char[] | clip name, NUL-terminated | `atl_back` | `atl_backstand` | `Desert_water_detail` |

Between the name and region 1 is a **gap of uninitialized bytes** — `0xFF` fill in the atlas clips, tool garbage in the water clip (which embeds a stray filter string `IGHTS;*.MLR;*.MW4ANIM;*.OBB;*.STL;*.`).

## Regions

- **Region 1 — joint directory.** Names of the joints the clip drives, interleaved with u16 fields (approximately `[u16 id][u16 ?]` per entry; ids in the 9,1xx range like the asset-table ids). The atlas clips list exactly the foot sites `site_rfoot` and `site_lfoot` — the same site-joint names as the skeleton (`erf-geometry.md`), confirming clips reference skeleton joints by name. The water clip lists `joint_water`.
- **Region 2 — float track data.** Floating-point values at one-byte-late alignment (for example 7.35 and 9.6 in `atl_back`); the water clip's values alternate around ±0.5 and ±1.0, consistent with UV-scroll keys for a tiling water surface.
- **Region 3 — track/value storage** (floats normalized 0–1 observed: 0.0333, 0.3, 0.5, 1.0).
- **Region 4 — event/track table** running to EOF: repeating 16-byte records `[u32 16][u32 start][u32 end][4 flag/tag bytes]` in the atlas clip (start/end stepping by about 52–64 per record, from 1000 upward).

## Channel layout (water clip, fully dumped at 249 bytes)

The `swamp_water_detail` clip exposes the channel shape in miniature (all values start one byte late, consistent with the header pad):

- **Region 1** (`0x70`): `00` + `joint_water` — one joint reference.
- **Region 2** (`0x7c`): 18 floats as (x, y, z) triples — two `(-0.25, 0.0, z)` triples (z = -0.0, -1.0) followed by twelve ±0.5 values; consistent with quarter/half UV-scroll offsets.
- **Region 3** (`0xc4`): track data; the water clip ends this region with the pair `3.3333` (its header scalar) and `0.0333` (1/30) — a time/step table at the region boundary — while walk/back/pose clips start it with zero-valued floats.
- **Region 4** (`0xd8`, first byte belongs to the region-3 float): small integer descriptor words `12, 0, 0`, then `16, 24, 8` and `03 02 01 00`-style tag bytes (counts/offsets; 12 matches the twelve ±0.5 values). In the atlas walk clip the same region is a run of 16-byte records `[u32 16][u32 offset][u32 span][4 tag bytes]` whose spans step by ~52–64 from 1000 upward — likely the per-track directory.

## Open items

- Complete keyframe decoding (times ↔ values pairing, rotations) and how regions 2–4 concatenate.
- The u16 id fields in the joint directory and the `[u32 16][offset][span][tags]` descriptor records' exact semantics.
- The constant `100` field (present in every sampled clip) and the `0x0c` pad byte's purpose; why region offsets point one byte before their data.
- The duration/blend scalar's exact role: measured `1.2` for both walk and back cycles, `3.333` for the stand pose and the water detail, `0.0` for the static backstand — consistent with seconds, but a blend time cannot be excluded.
