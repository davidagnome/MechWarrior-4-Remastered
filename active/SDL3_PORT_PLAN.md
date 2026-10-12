# SDL3 engine port plan

Updated: 2026-10-11. Status: proposed plan under ADR 0013 (clean-room track); the stack is fixed by `decisions/0014-engine-stack-and-licensing.md` (C++20, GPL-3.0, SDL3 + SDL_GPU). No engine code exists. This document is a plan; measured facts live in `docs/engine-specs/`.

## 1. Purpose and scope

Plan the full port of the MechWarrior 4 engine family (Vengeance, Black Knight, Mercenaries, both Mech Paks) as an independently written C++20 engine that reads the user's original media, ships no game assets, and runs natively on macOS arm64, Linux x64, and Windows x64. The 12-week vertical slice (feasibility doc Phase 2) is expressed as milestones M0–M6 followed by the track decision gate; expansion subsystems are sketched in section 7 and gated separately.

Not in scope: shipping assets, executing or patching retail binaries, DRM circumvention, and campaign, MechLab, or netcode before their gates.

## 2. Decisions

Firm (`decisions/0014-engine-stack-and-licensing.md`):

| Choice | Value |
|---|---|
| Language | C++20 |
| License | GPL-3.0 |
| Platform/renderer | SDL3 + SDL_GPU (Metal on macOS, Vulkan on Linux, D3D12 on Windows); raw Vulkan plus MoltenVK recorded as the alternative if SDL_GPU gaps appear |
| Shader pipeline | Offline compilation with SDL_shadercross to SPIR-V/MSL/DXIL |
| Video decoding | plmpeg preferred; FFmpeg considered only with recorded LGPL/GPL obligations |
| Build | CMake pinned to exact SDL3, shadercompiler, and decoder versions |
| Repository | Engine in its own repository created at M0; this repository keeps plans, specifications, and provenance |

Deferred: repository name; minimum OS/GPU targets; CI matrix details; final micro-architecture choices at M1/M2.

## 3. Architecture

| Module | Responsibility |
|---|---|
| `platform` | SDL3 init, window, input, main loop, logging |
| `gpu` | SDL_GPU device, swapchain, pipelines, framebuffers |
| `shaders` | Shader sources and per-backend artifact selection |
| `assets` | VBD v4 archive mount, ERF loader, TGA loader, `#BBO` skeleton, texture chains; spec-driven |
| `scene` | Transform composition, part assembly, scene graph |
| `render` | Mesh upload and passes (opaque textured first), camera |
| `video` | plmpeg playback into a texture |
| `game` | Slice behavior: free camera and basic mech movement |
| `parity` | Oracle harness and recorded comparisons |
| `tools` | CLI inspector over the asset library (list, extract, verify) |

Data flow: user media root → archive mount (`core.mw4` and friends) → resource resolution → typed loading (ERF → skeleton composition → scene) → render.

Measured conventions the engine must implement (from `docs/engine-specs/`):

- Archives: `#VBD` v4 container with LZW payloads and `directoryEnd + dataOffset` addressing (`mw4-archive.md`).
- ERF: `#FRE` v14; each record is preceded by its own `[u32 V][positions][u32 V][UVs]` data block; strip-order u8 indices; per-record `u32 V` plus unit normals; `#BBO` joint matrices `(r0,r1,r2,t)` composed parent×local with `v' = R·v + t` (row convention); torso-group parts are stored in mech space; the ankle 45° yaw splays the feet (ADR-level facts in `erf-geometry.md`).
- Textures: record names select `textures\@<base>{k}.tga` chains by record order; TGAs are sampled in stored row order with GL-style v (`row = (1−v)·H`), 32-bit BGRA and RLE supported.
- Maps: zone ERFs (`#FRE` type 132) with fully decoded terrain geometry (64-cell zones, 4×4 grid of 1280-unit zones, vertex arrays, u8 triangle triplets, per-triangle planes; validated on four zones), per-zone terrain texture sets (macro + 2×2 detail tiles), the `.tcf` texture list, the 256×256 `.material` overlay grid mapped to FGD surface-group ids (cross-map validated on `ARCTIC01`: 12 = snow, 6 = rock, 11 = airfield), and `.bid` textures decoded to RGB555 color / 8-bit masks with mips; the directional street/road variant selection, the URBAN01 groups 7/9, the `.bid` mip layout, and the shader blend equation are the remaining M3 work (`map-terrain.md`).

## 4. Clean-room workflow

- Implementation code is written only from `docs/engine-specs/` and public documentation. The `.local/re` research scratch (probes, scratch renderer) is never copied into the engine repository and is not an implementation base.
- Spec changes follow measure → specification commit here → implementation commit there; every parser statement carries provenance.
- Contributors attest that changes were written without access to decompiled code; the prohibited list (`MW4_Mercs_Decomp`, `anpage/demech2`, associated material) is enforced.
- Auxiliary models may review sanitized specifications and tests only.

## 5. Milestones (slice)

| Milestone | Budget | Deliverable and acceptance |
|---|---|---|
| M0 Repository and archive CLI | ~2 weeks | Engine repository created per ADR 0014; CMake with pinned SDL3/shadercompiler; CI builds on three OSes; CLI mounts user `core.mw4` and lists entries. Acceptance: exact entry counts and hashes reproduce spec values (for example 8,938 entries for Mercenaries `core.mw4`); parser unit tests green |
| M1 Window and GPU bring-up | ~1–2 weeks (window + SDL_GPU clear + textured triangle verified locally on macOS arm64 with the Metal backend — `openmw4-app --screenshot` renders the checker-mapped triangle offscreen and the swapchain smoke runs; the SPIR-V/DXIL paths via SDL_shadercross and the Linux/Windows runs remain) | SDL3 window with SDL_GPU clear and a textured triangle through the shader pipeline. Acceptance: runs on macOS arm64, Linux x64, Windows x64 |
| M2 Mech render | ~2–3 weeks (verified: ERF/`#BBO`/TGA/chains/assembly plus a GPU viewer that renders the 22-part Mad Cat textured with a depth buffer from user media — `openmw4-app --mech …`; remaining: lighting/shading and stance parity polish) | ERF loader, `#BBO` skeleton, TGA chains; the standing Mad Cat renders. Acceptance: side-by-side parity with the MW4: Mercenaries reference (layout, stance, launcher-hole grids) |
| M3 Map render | ~2–3 weeks (first light verified: the zone parser matches the spec totals exactly and `openmw4-app --map … --zone AA` renders the URBAN01 zone terrain textured with its zone texture from user media — `Screenshots/engine/m3-urban01-AA-zone.png`; remaining: per-texel material/`.bid` overlay blending, detail layers, directional street/road variants) | One zone and one full map with FGD-based layer blending. Acceptance: scale and orientation match the original under Wine at fixed cameras |
| M4 Movie playback | ~1 week (video and audio verified: pl_mpeg pinned `c871f2be…` decodes a mission cinematic and renders it letterboxed (`Screenshots/engine/m4-movie-frame.png`), MP2 audio decodes (44.1 kHz, peak 0.595) and streams via SDL audio with ~30 fps pacing; remaining: audible hardware smoke, `GAMEOPEN.MPG`) | `GAMEOPEN.MPG` plays from user media via plmpeg. Acceptance: plays on all three platforms without external codecs. Audio note: in-game WAVs are MS-ADPCM (4-bit, mono 22.05 kHz) and need a decoder — see `docs/engine-specs/audio-video.md` |
| M5 Input and movement | ~1–2 weeks (groundwork verified: world-space MVP uniform path with an orbit camera and keyboard controls; perspective renders of the mech and zone — `Screenshots/engine/m5-*.png`; movement/collision remain) | SDL3 keyboard/mouse, free camera, basic mech movement in the map. Acceptance: recorded interactive smoke |
| M6 Three-platform smoke and gate | ~1 week | Clean-checkout runs on three OSes with an evidence bundle, then the track decision gate (pursue, park, or reject per the feasibility doc) |

## 6. Testing and parity

- Unit: parser tests with exact counts against spec-pinned artifacts; golden struct dumps for regressions.
- Component: headless asset-load tests (archive → ERF → scene) with hashed summaries.
- Parity: fixed-camera comparisons against the original under Wine/CrossOver; recorded side-by-sides, not pixel equality.
- CI: build and unit tests on three OSes; GPU smoke depends on runner availability and coverage gaps are recorded explicitly.

## 7. Expansion preview (post-gate)

Ordered by risk: simulation core (movement, collision, weapons, heat, damage), mission-script VM, MechLab tables, campaign state, netcode replacement design, and save formats. Each gate requires its own specifications and acceptance evidence; this plan promises none of them.

## 8. Dependencies and licensing

SDL3 (Zlib), SDL_shadercross (license pinned at M0), CMake (BSD-3), plmpeg (pinned at M0). FFmpeg (LGPL/GPL) is used only if plmpeg fails, with obligations recorded first. The engine is GPL-3.0; every dependency must be GPL-compatible.

## 9. Risks

- The map material binding (directional street/road variant selection, blend equation) is the M3 blocker; terrain geometry, the material grid-to-surface mapping, and the `.bid` pixel formats are decoded.
- SDL_shadercross maturity and version availability across the three OSes.
- CI GPU coverage is likely partial; a manual hardware matrix is needed for M6.
- Decoder licensing constraints affect later packaging on macOS and Windows.
- Scope: campaign and netcode remain multi-year risks beyond the slice.

## 10. Open questions

- ~~Engine repository name and visibility at M0~~ — locked: `openmw4`, private first (2026-10-11).
- Minimum GPU and OS versions per platform.
- Whether the parity harness may reuse this repository's tooling strictly as test scaffolding (never as implementation code).
- Final video decoder choice after the M4 spike.

### M0 setup checklist

M0 does not depend on the open Phase 1 items — the `.mw4` archive spec is validated across seven retail artifacts, so the archive CLI can start now. **Status 2026-10-11: bootstrap bundle prepared and acceptance-verified locally** in `.local/engine-bootstrap/openmw4/` (LICENSE, README, CMake, `src/archive/vbd.{h,cpp}`, `src/erf/erf.{h,cpp}` + `src/erf/bbo.{h,cpp}` + `src/tga/tga.{h,cpp}` (M2 groundwork, validated against the pinned sample numbers, the Mad Cat skeleton, and the origin-bit TGA samples), `tools/vbd-cli`, `tests`, CI workflow, pinned `docs/specs/`, `docs/M0-EVIDENCE.md`); unit tests pass and `vbd-cli verify` reproduces the spec counts and byte totals exactly for Vengeance `CORE.MW4` (5,933 entries / 1,269,734 bytes) and Mercenaries `core.mw4` (8,938 entries / 2,515,786 bytes). The bundle is an initialized git repository (branch `main`, first commit `f7f4e0f`, 40 files, no media/build artifacts) and a clean-checkout configure/build/test pass; remaining at repository creation: create the private `openmw4` repo on the host, add the remote, push, and record the source-docs commit pin in `docs/specs/SOURCE.txt`. Decisions and steps:

1. **Repository decision (locked 2026-10-11)**: name **`openmw4`**, **private first** (flip to public only after a clean-room/rights review), GPL-3.0-only, copyright line "openmw4 engine contributors" pending the owner's name.
2. **Bootstrap contents**:
   - `LICENSE` (GPL-3.0 text) and SPDX headers (`GPL-3.0-only`).
   - `README.md` stating the clean-room posture: reads user-supplied media only, ships no game assets, no DRM circumvention, prohibited references listed (`MW4_Mercs_Decomp`, `anpage/demech2`), and a pointer to the spec set.
   - `docs/specs/` — a **pinned copy** of `docs/engine-specs/` at a recorded commit of this repository (the only transfer channel).
   - `CMakeLists.txt` (C++20), pinned dependencies: SDL3 and SDL_shadercross with recorded hashes, plmpeg at M4.
   - Layout: `src/archive/` (VBD reader), `src/erf/`, `src/scene/`, `tools/vbd-cli/`, `tests/` (parser tests with the spec-pinned counts and hashes), `.gitignore` that refuses media (`*.mw4`, `*.erf`, `*.tga`, `Installation Files/`).
   - CI: build + unit tests on macOS arm64, Ubuntu x64, Windows x64.
3. **M0 acceptance**: the CLI mounts a user `core.mw4` and lists entries whose counts and hashes reproduce the spec-pinned values (8,938 entries for Mercenaries `core.mw4`; all 21,919 payloads decode to declared lengths), with parser unit tests green.
4. **Clean-room workflow for the new repository**: specs (facts) only from this repository; never copy `.local/re/` scratch; every parser statement traceable to a spec statement; contributor checklist for provenance.

## 11. Next actions

1. Review and accept or amend `decisions/0014-engine-stack-and-licensing.md`.
2. Finish the Phase 1 blockers: the map material binding first (geometry, the material grid, and `.bid` pixel formats are decoded; directional-variant selection, URBAN01 groups 7/9, and the blend equation remain), then official patch-payload archive validation. The shell asset tables (`docs/engine-specs/tables.md`, including per-mech `.damage`/`.subsystems` records), animation clip headers (`docs/engine-specs/animations.md`, partial), the scripting languages and mission `.nfm` scenarios (`docs/engine-specs/scripting.md`), and mech composition records (`docs/engine-specs/erf-geometry.md`, partial) are decoded; the Mercs `MSGAME.CAB` mission metadata is staged.
3. At Phase 2 start, create the private `openmw4` repository and copy the prepared bootstrap bundle from `.local/engine-bootstrap/openmw4/` (M0 acceptance already verified locally); record the source-docs commit pin in `docs/specs/SOURCE.txt`.
