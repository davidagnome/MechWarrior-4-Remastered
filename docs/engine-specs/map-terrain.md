# Map archives and terrain (`RESOURCE/MAPS/*.MW4`)

Status: reconnaissance. The archive inventory, zone-level file formats, the zone cell structure, the zone terrain geometry (grid, vertex, index, and plane encoding), the material grid with its surface-group mapping (cross-map validated on `ARCTIC01`), the texture-list layer, and the `.bid` base/mip formats are decoded from two Vengeance maps; geometry passes every structural check on four zones. Directional street/road variant selection, the URBAN01 groups 7 and 9, the `.bid` trailing data, the city-side zone variants, and several constant fields are open. Facts come from independent measurement; community material was not used.

## Provenance

| Item | Value |
|---|---|
| Source | Vengeance disc 2 `RESOURCE/MAPS/URBAN01.MW4`, 14,872,939 bytes, SHA-256 `919adf85f24289a91859cc64b38fb5f515c7e62efe49d4fda280edff923c7a31` |
| Extraction | `7z e` from the converted disc 2 ISO into the gitignored `Installation Files/ReferenceISO/MW4Vengeance/` tree |
| Listing | `.local/re/vbd_list.py` (untracked scratch), 2026-10-11 |

## Archive inventory (URBAN01.MW4)

445 entries: 164 `.tga` plus 164 `{hint}` companions, 61 `.bid`, 17 `.erf`, 16 `.bsp`, 16 `.material`, and single `.instance`, `.fgd`, `.tcf`, `data{element}`, `data{gamemodel}`, `data`, `erf{zones}`, and `erf` entries.

- **Zones**: 16 named `AA`..`DD` (4×4 grid), each with `<zone>.erf`, `<zone>.material`, and `<zone>.bsp`.
- **Zone list**: `urban01.erf{zones}` (192 bytes) enumerates the 16 zones.
- **Map extents**: `urban01.erf` (52 bytes) is `#FRE` version 14 with header type `0x8C` (140) and contains map-size floats (1280.0, …).
- **Map instance**: `urban01.instance` (100 bytes) uses the same object header as mech instances (identity matrix, per-object record).
- **Layer table**: `urban01.fgd` (354,823 bytes) is a string table naming terrain layers, for example `urban01_color_parking_lite_active`, `arctic05_color_grass_active`, `arctic01_mask_noise256_active`, `urban01_mask_path_active`. Names pair `color_*` and `mask_*` layers with a `_active` suffix.

## Zone files

- `<zone>.erf` (for example `AA.erf`, 38,986 bytes) is `#FRE` version 14 with header `u32@0x08 = 132` — distinct from mech parts (144) and sky (131) — and `u32@0x0C = 1155`. It contains **128 `#RLM` markers**: a zone is 64 cells, each a fixed 122-byte metadata record followed by a variable geometry record. Grouping records are named after their layers, for example `maps\urban01\urban01_detail_active`, and zone textures are named `Maps\urban01\AA\urban01_AA_0_0000`.
- Zone names are `<Z band><X band>`: measured bounds are `AA` = X,Z ∈ [-2560, -1280], `AB` = X [-1280, 0] × Z [-2560, -1280], `BA` = X [-2560, -1280] × Z [-1280, 0], `DC` = X [0, 1280] × Z [1280, 2560] — a 4×4 grid of 1280×1280 zones.
- The mech-part data pattern (`u32 V` + `V`×vec3 + `u32 V` + `V`×vec2) does **not** appear in zone files; terrain geometry is per-cell with explicit u8 triangle triplets (decoded below).
- `<zone>.material` is exactly **65,536 bytes = a 256×256 per-texel material-index grid**. `AA.material` uses only four distinct values (1, 3, 7, 11), so a zone blends a handful of layers; the grid maps every texel of the zone to a layer index.

## Zone ERF cell structure (decoded)

A zone ERF is a sequence of **64 cells**, each an A record (fixed 122 bytes) followed by a B record (variable), alternating for 128 `#RLM` sections. The discriminator is `u32@+8`: `0` for A records, non-zero for B records.

- **A records (64, fixed 122 bytes)**: `#RLM` + `u32 18`, then `u32 0, u32 0, u32 −1, u32 512, u32 992, u32 −1`, then four floats `(-2480.0, 49.0, -2480.0, 114.0176)`; the remainder is a dense metadata region with small floats at mixed byte alignments ending with `114.0176`. Semantics open.
- **B records (64, variable)** — exact byte layout, validated on all 64 cell records in each of the plain zones (city-side zones interleave extra object records; see below):
  1. 25-byte header: `#RLM`, `u32 18`, `u32 payloadSize` (record size − 20), `u32 75`, `u32 0x6801`, then `00 <vertexCount> 00 00 00` (vertex count at +21).
  2. Vertex array: `vertexCount` × 12-byte floats `(X, Y, Z)`; `Y` is the terrain height.
  3. A 29-byte region, constant per zone except one byte: `00 00 00 00 01 <zone> 00 20 00 ff 3f 20 00 ff ff ff ff 00 00 00 00 0f 00 00 00 ff ff ff ff`. The `<zone>` byte is constant per zone and all values ≡ 1 (mod 8): AA `0x11`, AB `0x19`, AC `0x21`, AD `0x29`, BA `0x31`, BB `0x39`, BC `0x41`, BD `0x59`, CA `0x61`, CB `0x69`, CC `0x71`, CD `0x79`, DA `0x81`, DB `0x89`, DC `0x91`, DD `0x99`. Likely an atlas or zone identifier; semantics open.
  4. `u32 nameLength` (33), the shared zone texture name `Maps\urban01\<zone>\urban01_<zone>_0_0000`, NUL.
  5. `u32 1056` (constant in every cell record measured), then `u32 nBytes = 3 × (vertexCount − 2)` and `nBytes` bytes of u8 triangle indices covering `0..vertexCount−1`.
  6. `u32 triangleCount`, then `triangleCount` × 16-byte plane entries `(nx, ny, nz, d)` — `(0, 1, 0, 49)` for every triangle in URBAN01 (flat ground).
  7. Constant prelude (identical in every record examined): `01 00 01 00 ff ff 21 00 | ff ff ff ff 01 00 00 00 0f 00 00 00 ff ff ff ff`, then `u32 nameLength` (34) and the detail-layer name `maps\urban01\urban01_detail_active`, NUL.
  8. A trailing block opening with floats `(12.0, 12.0, 1/64, 1/256)` and a list of ground coordinates; semantics open.
- **Record size**: exactly `426 + 31 × (vertexCount − 4)` in every cell record measured; the per-vertex delta decomposes as 12 (position) + 3 (index bytes) + 16 (plane). The final B record in each plain zone file is 8 bytes smaller than the formula.

## Zone terrain geometry (decoded and cross-validated)

- A zone is an **8×8 grid of 160×160-unit cells covering 1280×1280 units**; each B record is one cell. A minimal cell is a flat quad (4 vertices, 2 triangles); city cells carry subdivision vertices along cell edges (AA cells 0-2 have 9-16 vertices).
- Each cell's triangle list is **explicit u8 triplets** under item 5 above; a fan triangulation of its vertex array.
- Verification: `AA`, `BA`, `AB`, and `DC` pass every structural check **64/64** (vertex-count byte, 29-byte region, `1056`, index run, plane count and values, size formula); AA: 382 vertices / 254 triangles; BA: 256 vertices / 128 triangles; AB and DC are plain quad zones.
- **City-side zones carry extra object records.** `AC`, `BC`, `AD`, and `BD` (X bands C/D — the built-up half of URBAN01) have more than 128 markers (`AC` 178, `BC` 182, `AD` 138, `BD` 144): after a clean A/B cell prefix (AC: 62 pairs) they interleave additional B-like geometry records (0.3–2.7 KB, float-heavy) with larger A-like records (244 and 392 bytes observed). Likely prop/object placements; decode open. `AC` cells 58 and 59 show **real height variation** (`Y` from 49.0 up to about 51.3) within the clean prefix, confirming per-vertex `Y` is terrain height, not a constant.
- Top-down wireframe of the decoded AA mesh: `Screenshots/erf-verification/urban01-zone-AA-terrain-wireframe.png`.

`<zone>.bsp` (818 bytes for `AA`) is also decoded: `u16 3`, then three 16-byte planes `(0, 1, 0, 49)`, then 64 × 12-byte cell records `[u32 8, u16 1, u16 class, u32 0]` with `class ∈ {0, 1, 2}` (AA: 7 cells class 0, 49 class 1, 8 class 2). Purpose unknown — per-cell classification or portal data. Total: 2 + 48 + 768 = 818 bytes exactly.

## Material grid and texture binding (decoded in part)

- **Zone terrain textures**: each zone has five 256×256 24-bit uncompressed TGAs — `<zone>_0_0000` (a macro city image) and `<zone>_1_0000`, `_1_0001`, `_1_0100`, `_1_0101` (a 2×2 detail-tile set). The zone ERF references only `_0_0000`. Contact sheet: `Screenshots/erf-verification/urban01-zone-AA-terrain-textures.png`.
- **`urban01.index.tcf`** (4,681 bytes) is a plain-text CRLF list of **61 entries** (`[k]`, `Name=`, `Type=`, `Width=`, `Height=`); the names match the 61 `.bid` files in `textures\composttexture\` (`Type=1` color, `2` decal, `3`/`5` masks). Entry order defines the compost texture indices.
- **`<zone>.material`** (65,536 bytes = 256×256, 5 world units per texel): per-texel overlay selector. URBAN01 uses only six **odd** values `{1, 3, 5, 7, 9, 11}` in every zone. Spatial check against the rendered grid (`Screenshots/erf-verification/urban01-zone-AA-material-grid.png`): `11` = the complete street and road network (40–44%), `7` = block interiors (54%), `1` = parking lanes plus the airport apron (thin strips in AA, a large filled region in AC), `3` = lawns, `5` = dirt, `9` = concrete patches.
- **`.fgd`** (354,823 bytes) is a texture/material database: records alternate short surface definitions (`background`, `grass`, `dead grass`, `path`, `dirt256a`, `concrete128x128`, `street_horizontal64/128/256`, …) and texture records carrying a binding field pattern `[id][flags][hash][1]`. Id `11` binds every citystreet/cityroad variant, `3` binds parking_lite + concrete, `1` binds medians and the noise mask, `5` binds the mask set, `32` the light map.
- **`urban01.data{GameModel}`** (184 bytes; tagged binary plus script text) carries `class=mechwarrior4::mwmap`, `tcffilename=urban01.index.tcf`, `waterlevel=48.000000`, `underwatertexture=maps\urban01\ur…`.
- **Material value = FGD surface-group id** (the `[id]` bound on the FGD's texture records). Evidence chain: (a) value `1` is green in the zone macro lightmaps of both `AA` and `AC` (hue `(26,39,25)` / `(39,49,32)`, green-dominant), matching the only green surface/texture pair (`grass` ↔ `arctic05_color_grass_active`, bound to group `1` with the noise mask and medians); the "value = TCF entry index" reading is dead because TCF entry 1 (`parking_lite`) is neutral gray. (b) The `dead grass` surface record carries the id pair `(3, 3)` and value `3` is the lawn patches. (c) Value `11` is the whole paved network; every street/road texture is bound to group `11`. (d) **ARCTIC01 settles it across maps**: its values `{6, 10, 11, 12}` are group ids bound in its own FGD, and its macro aerial shows 12 = snow field, 6 = rock outcrops, 11 = airfield apron. URBAN01 uses the same six ids in all 16 zones (`{1, 3, 5, 7, 9, 11}`).
- **Surface-variant selection resolves through the archive**: the map packs exactly one variant per surface (its `.tcf` lists, for example, a single `citystreet_color_horizontal` at 64×32), so a `.material` id maps to the one packed `.bid` without further disambiguation. The FGD's 64/128/256 variants are alternatives for other maps or quality settings.
- Macro-lightmap patch comparison (value 11 vs 7) and grid renders: `Screenshots/erf-verification/urban01-macro-material-classes.png`, `…-zone-AA-material-grid.png`.
- **Blending model (working hypothesis, unverified):** the zone TGAs behave like baked lighting/detail layers rather than the final color map — they are **not** quadrants of `urban01_light_active` (tested: mean absolute luminance difference 64/255), and that light texture (1024² = 4×4 zones × 256²) is a separate map-wide light. The likely runtime composite is the per-texel surface texture (tiled at the surface's declared tile size) modulated by the macro/light, plus the `urban01_detail_active` detail layer. Confirming the equation needs the renderer.

## `.bid` packed textures (base level decoded)

The 61 `textures\composttexture\*.bid` files hold the TCF textures; sizes are fixed per dimensions and type (no compression). Layout:

1. **Base level at offset 0**, row-major, sized `w × h` per the TCF entry:
   - `Type=1` (color) and `Type=2` (decals): **16-bit little-endian RGB555** (`0RRRRRGGGGGBBBBB`, bit 15 unused). Verified on three files: the all-white runway (every word `0x7FFF`), the intersection (all pixels near-gray once decoded, with 21% white markings — asphalt plus crosswalk stripes), and `parking_lite` (smooth asphalt).
   - `Type=3`/`Type=5` (masks): **8-bit luminance** (`urban01_mask_path_active` is exactly 512×512 = 262,144 bytes; the 16×16 turn-arrow mask shows the arrow glyph in its first 256 bytes).
2. **A mip pyramid follows the base at the same pixel format.** Verified on the 32×32 intersection: the 16×16 mip starts at byte 2048 and the 8×8 mip at 2560, each a correct downscale. Rendering evidence: `Screenshots/erf-verification/urban01-bid-base-format-validation.png`, `…-bid-mip-chain-probe.png`.
3. **Trailing data beyond the base is further image content whose exact layout is unresolved but non-blocking.** Rendering the 32×32 intersection's trailing 430 bytes as a 16×16 16-bit image yields a coherent blurry image (a correct continuation of the pyramid), so the tail is mip/light content, not a table; sizes are deterministic per dimensions and type (for example every 64×64 road texture is 11,672 bytes). Type-3 tails are exactly `4·w + 4` bytes; Type-1/2 tails exceed their computed 16-bit mip chains, and the 1024² light texture's tail is *smaller* than a full chain — the pyramid policy is not uniform. The base level alone is sufficient for full-resolution rendering; mip layout is a later polish item.

## Cross-map validation (ARCTIC01.MW4)

A second Vengeance map (`ARCTIC01.MW4`, 8,527,088 bytes, SHA-256 `c6270d446c3a43714763041654936c616a50d3ad9f22ec3d9710ac40f4ad110a`; 181 entries) confirms the model and sharpens the material mapping:

- Structure matches: 4 zones (`AA`..`BB`, a 2×2×1280 grid), each with `.erf`, `.material`, `.bsp`; one `arctic01.index.tcf` (18 entries), one `arctic01.fgd` (5,082 bytes, 141 records), plus a new `.mlr` entry and 18 `.bid` files. Zone ERFs are stored uncompressed in the archive. The 1280-unit zone/letter bands scale with map size (a 2×2 map uses letters `A`/`B` for the same 1280-unit cells).
- Geometry format holds on the second map: the B-record header, vertex-count byte, and `(X, Y, Z)` arrays parse exactly (zone `AA` = X, Z ∈ [-1280, 0]; first cell at [-1280..-1120]²), and this map carries **real terrain elevation** — sampled vertex heights run 58.0–61.6 instead of URBAN's flat 49. ARCTIC zones have 172 `#RLM` markers (64 cells plus 108 extra records), so the extra-record pattern is common to both maps.
- Value-11 turf (roads/airfield) appears only in zones `AA` and `BA`, matching the macro aerials; the tile-placement list's coordinates could not be mapped to world space despite trying several offsets and orientations, so its units remain open.
- **`.material` value = FGD surface-group id** — settled visually: the map's values `{6, 10, 11, 12}` are the ids bound in its FGD (`rock` → 6; `road_*`/`tarmac` → 11; `snow`/`ice` → 12). Rendering the macro aerial (`Screenshots/erf-verification/arctic01-AA-macro-aerial.png`) over the material grid (`…-AA-material-grid.png`) shows value 12 as the **snow field** (76%), 6 as the **rock outcrops** (20%), 11 as the **airfield apron/runway** (3%), 10 as small road patches. Value 10 has no located FGD binding yet.
- The FGD also contains a **tile-placement list** (thousands of records: short surface name + two u16 coordinates + trailing fields — 6,987 entries for URBAN01). Coordinate units are unresolved (values exceed the map bounds at 1:1), so it is not yet usable as ground truth; it likely records the authoring stamps fed to the tile compiler.
- URBAN01's group bindings found so far: **1** = grass/noise/medians, **3** = parking_lite + concrete, **5** = the mask set, **11** = the paved set (all street/road textures + one concrete + parking_dark), **32** = light. Groups **7** (the 54% block ground) and **9** remain unlocated in the FGD even after full-file scans.
- FGD surface records carry the surface's **tiled footprint**: the numbers match the texture's size times its tile count (`street_intersection` = 32×32, equal to its TCF size; `grass` = 2048×2048 = 8×8 copies of its 256² texture; `background` = 8192 = 32×32 copies). A handful of records also carry a small repeated pair after the footprint (`dead grass` 3, `path` 2, `light` 3) — consistent with per-surface tile counts rather than ids.

## Open items

- URBAN01 groups `7` (54% of the grid) and `9` (1%) — no FGD binding found in full-file scans; likely the block-ground and parking ground groups.
- How a surface selects among directional street/road variants (horizontal vs vertical vs intersections) per texel; whether the FGD tile-placement list (short name + u16 coordinates) can serve as ground truth (its coordinate units exceed map bounds at 1:1 and are unresolved); FGD binding-id `[id][flags]` field semantics; where `light` (id 32) applies.
- `.bid` trailing data after the base and mip pyramid (Type-3 tails are `4·w+4` bytes; Type-1/2 tails exceed their mip chains; the 1024² light tail is smaller than a full chain), and the FGD record schema (`surface` fields, binding `[id][flags][hash][1]` fields).
- The terrain shader blend equation: how the `_0_0000` macro, the `_1_00xx` 2×2 detail tiles, the material overlays, and the detail layer (`urban01_detail_active`) combine.
- Semantics of the B-record constants `75`, `0x6801`, `1056`, the 29-byte region, the zone byte, and the trailing block `(12.0, 12.0, 1/64, 1/256)` coordinate list.
- A-record field semantics (122-byte form; the 244/392-byte forms in city-side zones), and the extra object records in `AC`/`BC`/`AD`/`BD` (prop mesh + metadata decode).
- Role of the 16 `<zone>.bsp` files and their per-cell `class ∈ {0,1,2}` field; exact map extents in `urban01.erf`; the `erf{zones}` one-char list.
- Whether every map archive shares this layout, whether other maps use the 8×8/160 grid or varied cell sizes, and how zone records stream from disk (load order).
