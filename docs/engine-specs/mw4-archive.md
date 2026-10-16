# `.mw4` archive container (`#VBD` version 4)

Status: measured and validated on retail artifacts across Mercenaries, Vengeance, and Black Knight (the original seven plus the large archives below; every staged archive now decodes fully, including `textures.mw4` at 298,772,838 decoded bytes and `props.mw4` at 14,527 entries). Official patch payloads remain unprobed (not staged). Facts below come from independent measurement with the Phase 1 probe; no community material or generated code was used.

## Provenance

| Item | Value |
|---|---|
| Source media | Retail Mercenaries disc payload `MSGAME.CAB`, 531,591,572 bytes, SHA-256 `929080121ca2c77f12827ac8b3cd4917b5164209b6ab4a201e4ee27b763b7292` (user-supplied local reference media) |
| Extraction (cabinet) | `7z x -y -o.local/re/media/mercs <cab> "GAME/RESOURCE/core.mw4" "GAME/RESOURCE/MISSIONS/paradise.mw4" "GAME/RESOURCE/MISSIONS/coliseum.mw4"` |
| Extraction (discs) | Vengeance MDF and Black Knight BIN converted to ISO with the untracked raw-sector converter (Mode 1, 2,048-byte data at offset 16), then `7z e` for the listed paths into the ignored staging tree |
| Durable staging | Converted images and extracted artifacts live under the gitignored `Installation Files/ReferenceISO/` tree (never committed): Vengeance `Disc1.iso` SHA-256 `88801c9431764c40e7116b86a0ac1ea504855d30ce70052ee83aa165cea1d152`, `Disc2.iso` `d22aeb31b12d0848c8e260668551d9e8bbfc86f56823bb0fbc5cef6d58300cc2`, Black Knight `MW4BlackKnight.iso` `40a983e99cad67c794e831cd296771ab9a381cb638c980516db5e0e81175c87b` |
| Artifact 1 | `GAME/RESOURCE/core.mw4`, 1,688,217 bytes, SHA-256 `780ef6393291ae2716eea83c9da8b5f1cd7469bd1126e925075ea60a2bf9d0c9` |
| Artifact 2 | `GAME/RESOURCE/MISSIONS/paradise.mw4`, 398,994 bytes, SHA-256 `586a97b27f18171f7b20d31c6b4e12b6e3415757c884dbd57106b1d3ffe522a4` |
| Artifact 3 | `GAME/RESOURCE/MISSIONS/coliseum.mw4`, 58,948 bytes, SHA-256 `d8b318632fc9aec2a25d0dbbfd13bc2355d1507c2b1d4f1a83860e89132ed2c0` |
| Artifact 4 | Vengeance disc 1 `RESOURCE/CORE.MW4`, 981,197 bytes, SHA-256 `411883d06236ff3fc1f0acba9ae52d411bb84949af9f09273a8a2c2115497abd` |
| Artifact 5 | Vengeance disc 2 `RESOURCE/MISSIONS/0104.MW4`, 831,028 bytes, SHA-256 `96fdd8ae97891736e6f54eba82ac09e4ad2f1f780ba7e7da5cb567e4e6017fbc` |
| Artifact 6 | Black Knight `RESOURCE/COREX.MW4`, 1,212,667 bytes, SHA-256 `501bb6b14cddb755516b17f26879112de56261c6298ff5a69fac82bf7b73d3a0` |
| Artifact 7 | Black Knight `RESOURCE/MISSIONS/EDITOR_1.MW4`, 8,056 bytes, SHA-256 `9c0f7e7a4b244b80dd265ee991ffb8995428f4c20c42e7520d9982e5984cc198` |
| Artifact 8 | Mercenaries `GAME/RESOURCE/textures.mw4`, 8,618 entries / 298,772,838 decoded bytes (engine `vbd-cli verify`, 2026-10-11) |
| Artifact 9 | Mercenaries `GAME/RESOURCE/props.mw4`, 14,527 entries / 237,629,556 decoded bytes |
| Artifact 10 | Vengeance disc 2 `RESOURCE/MAPS/URBAN02.MW4`, 14,109,022 bytes, 395 entries / 25,028,977 decoded bytes |
| Artifact 11 | Vengeance disc 2 `RESOURCE/MAPS/ARCTIC01.MW4`, 181 entries / 13,430,112 decoded bytes |
| Artifact 12 | Vengeance disc 2 `RESOURCE/MAPS/DESERT01.MW4`, 199 entries / 11,787,862 decoded bytes |
| Probe | Untracked scratch `.local/re/vbd_probe.py` and `.local/re/vbd_detail.py`; Python 3.14; macOS 27 arm64; 2026-10-11 |
| Extraction validation | All payloads decoded; decoded byte counts equal declared lengths exactly (8,987/8,987 entries) |

Scratch and extracted media are never committed; the commands above plus this document reproduce the measurements.

## File layout

All integers are little-endian. Offsets are absolute file offsets.

| Offset | Size | Field |
|---|---|---|
| `0x00` | 4 | Magic `#VBD` (`23 56 42 44`) |
| `0x04` | 4 | Format version; `4` in every measured artifact |
| `0x08` | 4 | Constant `63` in every measured artifact; semantics unknown |
| `0x0C` | 4 | `directoryEnd`: absolute offset of the end of the directory region |
| `0x10` | 4 | Two identical `u16` halves, each `entryCount + 1` (`0x22EB22EB`, `0x00170017`, `0x001C001C`) |
| `0x14` | 23 + name | Root entry record (same record layout as directory entries), `ordinal = 0` |

`directoryStart = 0x14 + 23 + rootNameLength`; the directory runs until `directoryEnd`.

### Entry record

| Rel. offset | Size | Field |
|---|---|---|
| `+0x00` | 8 | 64-bit stamp, identical for every entry in the measured archives; decodes as a Windows FILETIME (`2002-09-30T21:17:35.117372`). Likely a pack build stamp; per-entry variation not yet observed |
| `+0x08` | 4 | `unpackedLength` (decoded byte length) |
| `+0x0C` | 4 | `storedLength` (bytes present in the file) |
| `+0x10` | 4 | `dataOffset` relative to `directoryEnd` |
| `+0x14` | 2 | `ordinal`: root is `0`; directory entries are `1..N` in directory order (validated exactly) |
| `+0x16` | 1 | `nameLength` (ASCII byte count) |
| `+0x17` | n | Name bytes: ASCII, no NUL, `\` separator (7 of 8,938 core names use `/` instead) |

Records are concatenated with no padding and no terminator; their total size equals `directoryEnd - directoryStart` exactly.

### Payload region

- Payload bytes for an entry start at `directoryEnd + dataOffset`.
- The root entry's payload is first (`dataOffset = 0`): a NUL-terminated ASCII string naming a related archive. For mission archives it names the map archive (`resource\maps\trop01.mw4`, `resource\maps\colsm01.mw4`); for `core.mw4` it is a single `00` byte (empty string).
- Payloads are contiguous from `directoryEnd` to end of file: zero gaps, zero overlaps, zero trailing bytes, and `dataOffset` order equals directory order in all three artifacts.
- If `storedLength == unpackedLength` the payload is literal. Otherwise it is LZW-compressed; no case of `storedLength > unpackedLength` was observed.

## Compression

LZW with these measured properties (all payloads decoded to their exact declared lengths):

- Codes are packed least-significant-bit first in a continuous little-endian bit stream.
- Initial code width 9 bits; codes `0x100` = clear, `0x101` = end, first dictionary entry `0x102`.
- Clear resets width and dictionary and is followed by one literal byte code.
- Width increases to 10, 11, and finally 12 after a dictionary add once the next free index reaches `0x200`, `0x400`, and `0x800`; dictionary tops out at 4096 entries.
- Standard self-referential case (`code == next free index`) expands as previous string plus its own first byte.

## Measurements

| Artifact | Entries | `directoryEnd` | `directoryStart` | Literal | LZW | Decoded bytes | Max expansion |
|---|---|---|---|---|---|---|---|
| `core.mw4` | 8,938 | 631,031 | 60 | 2,133 | 6,805 | 2,515,786 | 7.96x |
| `paradise.mw4` | 22 | 1,478 | 73 | 4 | 18 | 1,863,771 | 9.52x |
| `coliseum.mw4` | 27 | 1,840 | 73 | 4 | 23 | 714,498 | 26.24x |
| Vengeance `CORE.MW4` | 5,933 | 417,980 | 60 | 1,491 | 4,442 | 1,269,734 | 5.98x |
| Vengeance `0104.MW4` | 98 | 6,685 | 69 | 77 | 21 | 1,688,187 | 7.86x |
| Black Knight `COREX.MW4` | 6,885 | 487,328 | 61 | 1,714 | 5,171 | 1,620,226 | 7.95x |
| Black Knight `EDITOR_1.MW4` | 16 | 1,182 | 80 | 3 | 13 | 540,839 | 277.42x |

Core top-level name components: `mechs` 5,416, `effects` 2,978, `weaponsubsystems` 255, `weapons` 146, `ablscripts` 63, `subsystems` 37, `campaigns` 13, `tables` 10, `vehicles` 6, `ai` 4, plus a few single files.

## Validation

- Decode: every entry in all seven original artifacts decodes to exactly its declared `unpackedLength` (21,919 of 21,919), including the 8 KB Black Knight template archive; the five additional archives (Artifacts 8–12) also decode completely (the 300 MB `textures.mw4` among them).
- Structure: decoded `tables\mechtable.mpt` begins `24 00 00 00` = 36, followed by the 36-character ASCII path `Mechs\ArcticWolf\ArcticWolf.instance` and a NUL byte, proving the decode reproduces structured content rather than matching lengths by chance.
- Directory: record sizes sum exactly to the declared directory length; ordinals are exactly `1..N`.
- Cross-title: all seven artifacts share version 4 and the fixed `u32@0x08 = 63` value; `u32@0x10` is the duplicated `entryCount + 1` in every case; root payload semantics hold (empty for `core`/`corex`, parent-map reference for missions).
- Payload bounds: every payload lies inside the file; regions are contiguous and non-overlapping; the file ends exactly at the last payload.

## Open items

- Semantics of header field `u32@0x08 = 63`.
- Whether the entry stamp varies per entry in other archives (identical in all measured artifacts).
- Meaning of the duplicated `entryCount + 1` field at `0x10` (ordinal high-water mark or version pair).
- Root payload semantics for other archive kinds (only missions and `core.mw4` measured).
- The 7 `/`-separated names in core (mount-prefixed effect paths).
- Official patch payloads: Vengeance Patch 3 and Mercenaries PR1 `core.mw4`, Black Knight PR1 `corex.mw4` (not staged; the only unprobed class). The large `props.mw4`/`textures.mw4` and three further map archives are verified (Artifacts 8–12).
- Commit a maintained measurement tool (the probes are untracked scratch).

Cross-check only: the project-owned Mech Pak transform reads this container with the same field offsets and an equivalent LZW variant (`src/MW4Remastered.Core/Install/MechPakActivationTransform.cs:416-577`); the specification above stands on the measurements, not on that code.
