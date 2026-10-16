# ERF geometry (`#FRE` version 14)

Status: reconnaissance. The outer header, the `#RLM` record chain, per-record position/UV data blocks, strip-order byte indices, and the per-LOD vertex-count plus normals blocks are measured and visually verified, and texture binding is measured and visually confirmed (`@<base>{k}.tga` mech-wide chains sampled through the decoded UVs); the per-record float region, data-block framing fields, and exact texture-level selection remain open, as do the joint/armature transforms required to place limb parts. Facts below come from independent measurement of decoded archive payloads (see [mw4-archive.md](mw4-archive.md) for the container).

## Record families

Mech parts use short record names (for example `arc_lgun`). Cultural tree/prop ERFs (type `145`) share the container and the `[u32 nameLength][name][NUL][four zeros][u32 indexCount][indices]` record prelude but use longer names (for example `tree_arctic_snowy01_branch_mip2`, 31 chars), three LOD records per mesh (24-, 8-, and 4-vertex billboards for the arctic trees), and 16-bit `bpp=32` cutout textures named by the record.

## Where ERF data lives

Measured entry counts by archive (Mercenaries retail, `MSGAME.CAB`; `props.mw4` SHA-256 `2416de0e724fb35615c9ffc93a465d42f6551d38121b92d3c918766671d52081`, `textures.mw4` `6b4a39c98d0502047ae22c96219d8d708e3c85b4c2794d52ce2bf589d60b75a7`, `core.mw4` `780ef6393291ae2716eea83c9da8b5f1cd7469bd1126e925075ea60a2bf9d0c9`):

| Archive | `.erf` entries | Locations |
|---|---|---|
| `props.mw4` | 777 | `buildings\` 418, `vehicles\` 216, `culturals\` 88, `misc\` 54, `dropzones\` 1 |
| `textures.mw4` | 1,519 | `mechs\` 1,513 (mech part meshes), `weapons\` 5, `vehicles\` 1 |
| `core.mw4` | 1 | `mechs\owens\owe_lgun_dam.erf` |
| Mission archives | 2 each | `skies\<map>_sky\*.erf` |

Mech geometry is per part: for example ArcticWolf has 43 ERFs (`arc_hip.erf`, `arc_lgun.erf`, `arc_torso.erf`, and `_dam` variants). `core.mw4`'s `mechs\arcticwolf\*.torso`, `*.engine`, and `*.data` entries are 12-byte stubs, and `*.data{GameModel}` / `*.torso{GameModel}` blobs do **not** carry the `#FRE` magic; they are a different serialized form (samples staged, header bytes recorded below).

## Samples (decoded bytes pinned)

| Sample | Source entry | Decoded size | SHA-256 |
|---|---|---|---|
| Sky | Vengeance `0104.MW4` → `skies\lunar02_sky\lunar02_sky.erf` | 1,667 | `3f98eeafd702530ebabf174bc5453d8e795920cd00dc2da506461064661df803` |
| Gun part | Mercenaries `core.mw4` → `mechs\owens\owe_lgun_dam.erf` | 6,948 | `6c87a073ecfffa67ad6f44861f3385f1b8568bd1b02ddcda14717cc6d5d32c09` |
| Arm gun | Mercenaries `textures.mw4` → `mechs\arcticwolf\arc_lgun.erf` | 11,614 | `f7c4d415302c38336acb7ac6e25ac9519c20f4f38238ea740fdc1f39ae0632d7` |
| Torso | Mercenaries `textures.mw4` → `mechs\arcticwolf\arc_torso.erf` | 33,461 | `fcdcdcc28cb95c941654feb6d343d80593b5ee562e6070212164c53fb015c42a` |

Staged under the gitignored `Installation Files/ReferenceISO/samples/erf/`; nothing here is committed. Reproduction: decode with an archive reader, or `python3 .local/re/vbd_list.py <archive> --dump <entry>` from untracked scratch.

## Measured header layout

Little-endian. Offsets verified across all four samples.

| Offset | Size | Observation |
|---|---|---|
| `0x00` | 4 | Magic `#FRE` (bytes `23 46 52 45`) |
| `0x04` | 4 | `14` in all samples (format version) |
| `0x08` | 4 | `144` (`0x90`) in the three mesh parts, `131` (`0x83`) in the sky sample; semantics unknown |
| `0x0C` | 4 | `5` in the three mesh parts, `1` in the sky sample; count-like |
| `0x10` | 16 | Four floats. Torso `(-0.0139, 1.7197, -0.1931, 2.0266)`; arm gun `(0.5272, 0.1169, 0.98, 1.577)`; damaged gun `(0.1828, 0.2013, 0.0679, 1.6164)`; sky `(0.0, 50.0, 0.0, 150.0001)` |
| `0x20` | 2 | Mesh parts only: `u16 = 6` |
| `0x20`/`0x22` | 4 | `#RLM` marker (mesh parts at `0x22`; sky at `0x20`) |
| after marker | 4 | `18` in the three mesh parts, `17` in the sky sample |
| after that | | Mesh parts continue with an eight-byte zero run and small counts; sky continues `0x657` (1,623), `0x4b` (75), `0x6606`, `0x800` … Structure not yet decoded |

Non-ERF counter-samples: `arcticwolf.data{GameModel}` begins `01 00 01 00 01` + `Jammer]`, `arcticwolf.torso{GameModel}` begins `00 00 00 00 01` + `Jammer]`, and `arc_hip.data{GameModel}` begins `00 00 00 00 01` + `nt\mech`. They contain no `#FRE` or `#RLM` marker at any offset.

## Post-`#RLM` records (reconnaissance)

Each measured ERF contains exactly one `#RLM` marker; the following bytes are a sequence of named records. A record begins with `u32 nameLength` + ASCII name, then a fixed prelude relative to the byte after the name:

| Rel. bytes | Field |
|---|---|
| 5 | `00` in every observed record |
| 4 | little-endian `u32 count` |
| 2 | `00 00` |
| rest | index-like data, then float data |

Measured records:

| Sample | Records | Names | Counts | Block sizes |
|---|---|---|---|---|
| Sky (1,667 B) | 6 | `skyl2f1`, `skyl2u1`, `skyl2l1`, `skyl2b1`, `skyl2r1`, `skyl2f1` | 12, 6, 6, 6, 6, 6 | first 334 B, then five 240 B faces |
| `arc_lgun` (11,614 B) | 6 | `@aarc0` ×6 | 168, 144, 78, 78, 42, 24 | 3,229 / 2,421 / 1,667 / 1,507 / 943 / 347 |
| `owe_lgun_dam` (6,948 B) | 6 | `@aowe0` ×6 | 42 ×6 | 1,151 ×5 then 593 |

Each record is preceded by its own vertex-data block: `u32 V` followed by `V` position triples (three floats), then `u32 V` again followed by `V` UV pairs (two floats), framed by small fixed 21–29-byte structures. `V` equals the following record's `max index + 1` in every measured case: all six `arc_torso` records (V = 233, 199, 123, 76, 54, 40), all six `arc_lgun` records (V = 70, 62, 34, 34, 26, 14), and all six `owe_lgun_dam` records (V = 25; those blocks repeat identical data). The sky file instead carries a single 8-vertex data block after the file header, shared by its six face records. Strip indices address their own block's position array directly.

Observations: mech-part counts and block sizes decrease along the record chain, consistent with a LOD chain (hypothesis). Every measured mesh record now has these verified sections:

1. **Indices**: `count` bytes, one per index, 0-based, in **triangle-strip order**: consecutive values extend a strip (triangle `i` uses vertices `idx[i], idx[i+1], idx[i+2]`), not independent triples. `max index + 1` defines the record's vertex count and matches the normals length in all twelve measured mesh records.
2. **Vertex count and normals**: a `u32 V` immediately followed by `V` float triples, each of unit length (|v|² = 1 within 2%). The run length equals the index maximum + 1 in every record: `arc_lgun` V = 70, 62, 34, 34, 26, 14; `owe_lgun_dam` V = 25 in all six records.
3. **A float region between the `u16 0` after the strip indices and the count**: smallest `arc_lgun` record holds eight 16-byte entries whose first three floats are unit vectors at a constant 4-float stride; longer records hold larger similar regions (57 floats in `owe_lgun_dam`, 225 in `arc_lgun` LOD0). Content unidentified.

The bytes after a record's normals block are the next record's vertex-data block; the final record has none (the normals blocks of `arc_lgun` LOD5 and `owe_lgun_dam` record 6 end exactly at the file end). The sky's first face reads cleanly as `u16` indices `1,2,3,2,1,4,5,6,7,6,5,4` (two quads split into four triangles), followed by `00 00` and `u32 0` before float data. Header `u32@0x0C` is `5` for both mesh parts and `1` for the sky, plausibly the highest LOD index (hypothesis). Float values include magnitudes 0, 0.26, 0.97, 1, 2.395, 24.25, 100, and 131,071 sentinels.

## Material and texture binding (reconnaissance)

- Each mech's part records share one base name (`@aarc0` in both measured ArcticWolf part files; `@aowe0` and `@aatl0` for Owens and Atlas). The name string is identical in all six records, and the same base is reused by that mech's other parts: the texture is a mech-wide atlas, with part UVs selecting regions.
- `textures.mw4` carries a texture chain per part base: `textures\@<base>{k}.tga` for k = 0..5, each with a 4-byte `{hint}` companion entry. Mercenaries retail contains 532 such entries across 46 bases.
- Measured headers, all 32-bit BGRA images: `@aarc0` is 512×512 RLE (TGA type 10) and `@aarc1..5` are 256/128/64/32/16 uncompressed (type 2); `@aowe0` and `@aatl0` are 512×512 uncompressed with the same halving chain. Decoded sizes equal width × height × 4 + 44.
- TGA origin bits vary per file: `@aarc0` and `runninglight` are top-left; `@amad0`, `@aowe0`, and `cdash1` are bottom-left. The game samples each TGA's **stored row order** directly with GL-style v (sampler row = `(1−v)·H`); a loader must not normalize rows to top-down first, or bottom-origin textures get double-flipped. Getting this wrong moved the Mad Cat launcher-face UV island onto the legs (black missile-hole grids appeared on the legs while the pods showed armor); with stored order preserved, both shoulder pods display their hole grids and the legs return to camo (`madcat-reference-comparison.png`; root-cause diagnostics `madcat-leg-uv-on-atlas.png` and `madcat-atlas-grid.png`).
- Record names are texture-entry names. `mad_torso.erf` alternates `@amad0` (exterior atlas) and `@pilot` (cockpit interior, `textures\@pilot.tga` exists) across 13 records; each `@pilot` record has its own small vertex block (the first holds 12 vertices), and the other measured parts repeat one name six times.
- Working hypothesis: the record name identifies the texture entry, repeated records with the same name advance one level of the `@<base>{k}.tga` chain per LOD, and `@pilot` names the cockpit-interior texture.
- Record names can also reference unrelated texture entries: `madcat_cage.erf` records are named `cdash1` (→ `textures\cdash1.tga`, extra levels 2–5 exist) and `runninglights.erf` records are named `RunningLight` (→ `textures\runninglight.tga`).
- Textured rendering confirms the binding: sampling `@aarc0.tga` through the decoded UVs reproduces aligned armor details and painted markings on both ArcticWolf parts in all four views (see `Screenshots/erf-verification/arcticwolf-*-textured-4views.png`).

## Joint transforms (`#BBO`)

`<mech>.data{HierarchicalOBB}` (for example `madcat.data{HierarchicalOBB}`) is the joint hierarchy:

- Header: magic `#BBO`, u32 version `3`.
- Records: joint name (NUL-terminated), one pad byte, then a 64-byte matrix — three rows of `(r0, r1, r2, t)`, row-major 3×4 — followed by a 16-byte extents block `(hx, hy, hz, radius)` for leaf OBBs; some records carry extra trailing data (child lists). All 31 Mad Cat joints appear in hierarchy order (legs, hip, torso, head, torso fronts, specials, arms).
- Verified: composing `joint_luleg → joint_ldleg → joint_lankle → joint_litoe/lbtoe/lotoe` and applying each joint's composed matrix to its part turns the previously overlapping leg parts into a correct articulated reverse-joint leg (`madcat-leg-transform-diagnostic.png`); the arm chain (`joint_luarm → joint_lgunabove → joint_lgun`) transforms its parts coherently as well (`madcat-arms-transform-diagnostic.png`).
- Parent links from `contents[joint_*]{armature}` build the tree `joint_vel → joint_root → {joint_hipbelow → hip → hipabove → torsobelow → torso → torsoabove, joint_luleg/ruleg → ldleg/rdleg → ankle → belowankle → toes}`; `joint_torsoabove` children include head, torso fronts, specials, cage, and both arm chains (`joint_luarm → joint_lgunabove → joint_lgun`, mirrored right). Missing joints (`vel`, `root`, `cage`) act as identity.
- Composition and convention: world = parent × local; vertex transform `v' = R·v + t`. Limb parts are joint-local; the torso group renders correctly at its stored positions (its chain is near identity), and pods/lights are left untransformed.
- The only non-identity rotations in the Mad Cat skeleton are the two ankles — `joint_lankle` and `joint_rankle`, each a 45° rotation about Y, mirrored. They **splay the feet outward**: applying them with the row convention (`v' = R·v + t`) yields the reference's wide toe-out stance, while the transposed form twists both feet inward, and removing the rotation leaves the feet parallel-forward (variants: `madcat-ankle-variants.png`, top view: `madcat-feet-splay-top.png`). The row convention is therefore confirmed for the whole skeleton.
- Result: all 22 parts assemble into a standing mech (`madcat-assembled-textured-4views.png`; side view shows the reverse-joint legs), matching the MW4: Mercenaries reference in layout and stance (`madcat-reference-comparison.png`). The reference render is a **posed promotional image** (walking stride, rear leg extended with the shin and heel plates flaring back); the bind pose is neutral, and the difference is pose, not orientation.

## Validation (visual)

The decoded per-record positions rendered with strip-order indices produce coherent shapes in four-view orthographic renders: `arc_torso` reads as a boxy mech torso with two shoulder pods and a cockpit panel, `arc_lgun` as a housing with a long barrel, and `owe_lgun_dam` as a broken gun chunk. Grouping the same indices as independent triples renders triangle soup; the strip interpretation was identified from that comparison. Sampling `textures\@aarc0.tga` (512×512 RLE, 32-bit BGRA) through the decoded UVs renders both ArcticWolf parts with aligned armor detail, confirming the texture binding. The iconic Mad Cat also verifies: `mad_torso.erf` renders with its characteristic cockpit canopy and side launcher pods, and `mad_lgun.erf` as its weapon housing, both textured from the `@amad` chain. Merging all 22 LOD0 Mad Cat part files with no transforms places the torso group plausibly — torso, shoulder launcher pods, and running lights land in a shared space — and the `#BBO` joint transforms then assemble all 22 LOD0 parts into a standing mech (side view shows the reverse-joint legs); the untransformed limb pile is kept as `madcat-legs-diagnostic.png`. Comparison against the MW4: Mercenaries Mad Cat model render (`madcat-reference-comparison.png`, reference also kept as `madcat-reference-mw4mercs.png`) matches the reference layout and stance, with launcher-hole grids on both shoulder pods and coherent part textures (`madcat-texture-diagnostic.png`); the residual difference is the reference's green skin and smoother shading versus the dark `@amad0` camo. Renders are kept for review in the gitignored `Screenshots/erf-verification/` folder (`arcticwolf-*`, `owens-lgun-dam-strip-4views.png`, `madcat-torso-*`, `madcat-lgun-textured-4views.png`, `madcat-assembled-strip-4views.png`, `madcat-assembled-textured-4views.png`, `madcat-legs-diagnostic.png`, `madcat-reference-comparison.png`, `madcat-reference-mw4mercs.png`, plus the triple-grouping comparison); the scratch renderer is untracked at `.local/re/erf_render.py`, `.local/re/assemble.py`, and `.local/re/compare.py`.

## Mech composition (`.contents`)

Each mech also carries `mechs\<mech>\<mech>.contents` — a fixed **282-byte composition record** (identical size for all 21 mechs; header words identical across mechs: 280, 2, 2, 0, 1057, 0; single mech-specific bytes at offsets 0 and 0x1B/0x1C). It embeds:

- `joint_world` — the root joint name.
- The skeleton reference `…s\<mech>\<mech>_skeleton.obb`, stored in a fixed field at 0xA6; when the full path exceeds the field it is truncated **from the front** (measured: `s\madcat\madcat_skeleton.obb`, `las\atlas\atlas_skeleton.obb`). No matching standalone `.obb` entry exists in `CORE.MW4` — the skeleton data itself lives in the `data{HierarchicalOBB}` stream (see above); where the `.obb` path resolves is open.
- `content\effects\materialeffectholders\mech_foot_large.contents` — present in `CORE.MW4` (80-byte effect-holder records exist for every weapon/material effect).
- `Content\subsystems\…` paths for the mech's equipment; the last string is truncated at the file end.

Related per-mech assets in `CORE.MW4`: `<mech>.engine` / `.torso` / `.data` (12 bytes each), `<mech>.subsystems` (equipment loadout, for example 5,018 bytes for the Mad Cat), `<mech>.damage` (1,688 bytes), `<mech>.instance` (344 bytes), and `<mech>_destroyed_solid.obb` (78-byte wreck solids).

## Open items

- What the pre-count float region represents (per-vertex frames, joints, or per-LOD transforms) and why its size grows faster than V.
- What the 21–29-byte structures framing each vertex-data block contain (small fixed fields including byte pairs `61 44` / `61 45` and `73 04`).
- Whether `u32@0x0C` (5 for mesh parts, 1 for the sky) indexes the LOD chain, and what the sixth sky record repeats `skyl2f1` for.
- Whether the record chain is a LOD chain and what determines its length; the sixth sky record repeats `skyl2f1`.
- Semantics of `0x08` (144 vs 131), the header floats, and the many all-ones sentinel values.
- Confirm the record-index-to-texture-level selection (per-LOD image or mip level) and the meaning of the 4-byte `{hint}` companions (measured `00 09 02 00` for `@aarc0.tga`).
- Remaining assembly questions: `mad_torso` LOD0's single stray index (182, one past its 161-vertex block — exactly one dropped triangle). The corrected TGA-origin/sampling handling resolved the pod/leg texture misatlas; the remaining visible differences from the reference render are its green skin and smoother shading versus the dark `@amad0` camo, and its walking pose versus the bind pose (`madcat-reference-legs-zoom.png`).
- Why `mad_torso.erf` LOD0 carries a single index value of 182 (one past its 161-vertex block, at position 544 of 546); every other index is in range.
- Whether `.mw4anim`, `.lights`, `.bounds`, and `.data` share related section markers.
- `.contents` field semantics (the mech-specific bytes, the subsystem id slots), where the `<mech>_skeleton.obb` path resolves, and the per-mech `.subsystems` / `.damage` / `.instance` record layouts.
- A maintained ERF measurement tool (the analysis scripts are untracked scratch).
