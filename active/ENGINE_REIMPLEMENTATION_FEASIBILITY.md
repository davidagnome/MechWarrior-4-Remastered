# Engine reimplementation feasibility (clean-room track)

Updated: 2026-10-11. Status: accepted by `decisions/0013-clean-room-engine-reimplementation.md`; Phase 0 first pass executed (documentary, recorded below); no engine code exists. This document is a plan and an evidence boundary, not an execution claim.

Purpose: decide whether a clean-room reimplementation of the MechWarrior 4 engine family can give native Windows/Linux/macOS rendering from the user's original media, at what cost, and under which legal and process constraints. The shipped installer/launcher product and the Wine/CrossOver cross-platform track proceed independently and are not affected by anything here.

## Premise

- This project contains no game or renderer source. The original games are 32-bit x86 DirectDraw/Direct3D 7 binaries staged from user media.
- A native engine means independently written code that reads the original data files. It requires the user's media, ships no assets, executes no retail binary as a dependency, and performs no DRM circumvention.
- Changing the stack (SDL3, Vulkan, Metal) does not make code non-derivative. Derivation follows provenance and similarity to protected expression, not the libraries linked against. The clean-room model (specification transfer with separated access) is the accepted way to write independent code, and this plan adopts it.
- The rights position is unchanged from ADR 0002: not public domain, limited historical clearance, user-supplied media required. Phase 0 audits the current rights chain rather than assuming it.

Why the track exists: the executed macOS compatibility spike hit structural blockers a native engine removes. CrossOver 26 cannot decode the startup MPEG-1/MP2 movies without redistributing decoder DLLs the project will not ship, and dgVoodoo's D3D12 path crashes before window creation under vkd3d (`active/CROSSPLATFORM_FEASIBILITY.md` Phase 3 results). A clean engine also stops shipping transformed retail executables entirely.

## Ground truth

| Item | Fact | Source |
|---|---|---|
| Game binaries | 32-bit x86, DirectDraw/Direct3D 7, SafeDisc retail | `active/CROSSPLATFORM_FEASIBILITY.md` |
| Engine source | none in this repository | ADR 0013 context |
| Known data layout | `RESOURCE/*.mw4` archives (core/props/textures plus per-mission maps), `.mpt` tables (mech/chassis/weapons/subsystems), `.script` shell/mission scripting, TGA mission images, WAV music, MPEG/AVI startup movies, INI configuration | install transforms, `MechPakActivationTransform`, media path maps, RTP host |
| Shell/UI | data-driven RTP shell; setup already patches RTP payloads | `MW4Remastered.RtpPatchHost`, `OfficialVengeancePatch3Transform` |
| Runtime oracle | original executes under CrossOver/Wine; main menu and Instant Action user-confirmed on macOS 27 arm64 | `active/CROSSPLATFORM_FEASIBILITY.md` Phase 3 |
| Blockers this track removes | macOS movie decode; dgVoodoo D3D12 under vkd3d | same |

Not yet known (Phase 1 work): verified formats for models/animation, terrain/scene, mission scripts, AI behavior, netcode, and save/pilot state. Phase 0 located candidate public field maps for some of these, but none is licensed for transfer, so specification by measurement remains the path.

## Phase 0 executed results (2026-10-11, first pass)

Evidence level: documentary. All sources retrieved 2026-10-11. This is a first pass, not the completed two-week audit; remaining work is listed under Next actions.

### Rights

| Fact | Evidence |
|---|---|
| MechWarrior 4: Mercenaries (2002) was developed by Cyberlore with FASA Studio and published by Microsoft Game Studios; the 2010 free release was published by Studio MekTek | Wikipedia `MechWarrior 4: Mercenaries`, oldid `1349920173` |
| Microsoft cleared that free release on 2010-04-22 and MekTek published it on 2010-04-30; free availability later ended and later groups could not secure permission to continue it | same article, citing Engadget/Joystiq (2010-04-22) and MekTek notices (2010, 2013) |
| No source found supporting public-domain status or a general redistribution grant | no contrary evidence located; consistent with ADR 0002 |
| The series' electronic rights were negotiated back from Microsoft by Smith & Tinker and then purchased by Piranha Games in February 2011; Piranha has been an Enad Global 7 subsidiary since 2020-2021 | Wikipedia `MechWarrior Online`, oldid `1377487794`; Wikipedia `Piranha Games`, oldid `1346034761` |
| No primary license contracts were reviewed; exact scope (platforms, reserved rights, MW4 coverage) is unverified | recorded gap; carried to Phase 2 legal review |

Assessment: nothing found blocks a clean-room, user-media-only engine. The posture matches the OpenMW model (independently written engine code, user-supplied original assets, no bundled content), which has operated publicly since 2008. Enforcement remains discretionary; the rights chain is identified only through secondary sources and its contractual scope stays an open gap.

### Prior art

| Item | Classification | Notes |
|---|---|---|
| Public MW4 engine reimplementation | none found | GitHub API searches `mechwarrior4` (1 repository) and `"mechwarrior 4"` (15 repositories) on 2026-10-11 returned tools, bots, translations, and asset rehosts only |
| `voidvoidvoid/MW4_Mercs_Decomp` | prohibited | Decompilation project referenced by the MW4_Tools README ("The repository is associated with the MW4 Mercenaries decompilation project ..."); API and web both return 404 on 2026-10-11 (private, renamed, or removed). Stays prohibited if it reappears; its existence is the main taint-risk finding |
| `anpage/demech2` (C, LGPL-3.0, created 2026-09-25, actively updated) | prohibited | Work-in-progress decompilation of MechWarrior 2, not MW4; recorded because decompilation activity is rising across the franchise within weeks of the MW4 decompilation reference |
| `voidvoidvoid/MechWarrior4_Tools` (Python Blender add-on, no license) | oracle-only | Modern format research and readers: `.erf` geometry, `.mw4anim` animation, `.contents` hierarchies, map terrain, packed "compost" textures, archive access. No license: nothing may be copied, documentation needs independent verification, and the decompilation association rules it out as a spec-transfer source |
| `HelmMemoryCore/Mechwarrior-4-ModdingTools` ("Star League Cache", ModDB rehost, no license) | oracle-only | Vintage format notes (`mechchassistable.mpt.rtf`, chassis/subsystem/weapon documents, table studies) plus binary tools (MW4eXtractor, table editors, ERF viewers); provenance unknown, and it also redistributes system DLLs |
| `RivvidGunner/Mechwarrior-4-ERFEditor` (GPL-3.0), `RivvidGunner/Mechwarrior-4-ERF-Model-Builder`, `narpassword27/MechConverterCS` (no license) | oracle-only | Mesh conversion prior art (ERF, COB, STL) |
| `ZombieRoxtar/MW4-FreeMarket` (CC0-1.0) | usable knowledge | `.blackmarket` campaign-file reader/writer; permissive, campaign-state scope only |
| `dgnorton/DXPort`, `pausenclown/FancyBot`, `Tailgunner/FancyMercs` | oracle-only | DirectPlay-era NAT/multiplayer tooling; netcode field context |
| OpenMW (GPLv3; first release 2008-06-03; 0.50.0 on 2025-11-07; requires original game assets) | process model | The clean-room precedent this track follows |
| DevilutionX (C++, created 2018-08-02; license metadata NOASSERTION) | process caution | Decompilation-derived and tolerated in practice; not a licensing model |
| re3/reVC | cautionary | Take-Two action against the reverse-engineering authors recorded by Wikipedia "Fan game" |
| XWVM | historical only | Its domain is parked for sale as of 2026-10-11; no active source cited |

### Format knowledge

Candidate public field maps exist for archives, ERF geometry, mech skeletons/animations, map terrain and packed textures, `.mpt` tables, weapons/subsystems, and `.blackmarket` state. None is licensed for transfer to this project (only the CC0 and GPL items have stated terms, and no engine license has been adopted). Phase 1 therefore still measures and specifies from user media, but it starts from candidate maps to falsify rather than a blank page. Prohibited decompilation output is not used anywhere.

### Gate result

Phase 0 closed 2026-10-11 at documentary level with conditions. The rights chain reaches an identifiable current holder (Piranha Games, an Enad Global 7 subsidiary) with no permissive grant and no enforcement action found toward MW4 reimplementations; exact contractual scope is unverified and carried as a Phase 2 legal-review item. The decompilation projects (`MW4_Mercs_Decomp`, `anpage/demech2`) stay on the exclusion list backed by the contributor attestation; decompilation activity is rising across the franchise and remains a standing taint risk. No unlicensed community code is used, and every community format claim must be independently reproduced. Phase 1 is authorized and its scaffold is committed.

## Clean-room protocol

1. Two workstreams. Research may inspect binaries, run instrumented originals, and capture behavior. Implementation reads only published specifications and public documentation.
2. Research output is written specifications under version control (`docs/engine-specs/`); never decompiled listings, symbol dumps, or code fragments.
3. Reverse-engineering scratch (databases, notes, captures) stays untracked under `.local/re/`; only sanitized specifications migrate into the repository.
4. Every specification fact carries provenance: what was measured, on which exact-hash artifact, with which tool and version.
5. Existing reverse-engineering-derived project code (executable transforms, Mech Pak site matching) must not be copied into the engine; it is another observation of the same binaries, not engine source.
6. Auxiliary models may review sanitized specifications and tests only. No binaries, disassembly, media, or memory dumps in prompts, per `AGENTS.md` routing rules.
7. Contributors attest that implementation changes were written without access to decompiled code.
8. Specifications are reviewed for legal hygiene (facts and identifiers needed for interoperability only) before commit.

## Phases

### Phase 0: rights and prior-art audit (timebox: 2 weeks; evidence level: documentary)

Tasks: identify the current franchise/electronic rights holders and the exact scope of any historical clearance; catalog prior MW4 reimplementation attempts, community format documentation, and modding tools; classify each as usable public documentation, oracle-only (binary tools used for output comparison), or prohibited (decompiled source); collect process oracles (OpenMW, DevilutionX, XWVM) and the cautionary enforcement case.

Deliverable: findings recorded here with pinned citations.

Gate: no adverse rights finding and a lawful path to the required format knowledge; otherwise park the track with the recorded evidence.

### Phase 1: format and behavior specifications (timebox: 4 weeks; evidence level: static/focused)

Tasks: specify `.mw4` archive structure, `.mpt` tables, model/geometry/animation formats, terrain and mission `.mw4` files, `.script` scripting, RTP shell layout, INI/save formats, audio, and video. Validate each parser statement against the user's real files with exact counts.

Deliverable: `docs/engine-specs/` documents (location confirmed 2026-10-11; method and checklist scaffold committed as `docs/engine-specs/README.md`).

Status: two formats in progress 2026-10-11. The `.mw4` archive container is specified and validated on seven retail artifacts across all three titles (`docs/engine-specs/mw4-archive.md`); ERF geometry is in reconnaissance (`docs/engine-specs/erf-geometry.md`: `#FRE` header, `#RLM` named object records, mech parts with six decreasing LOD-like blocks, per-record positions+UVs, strip-order indices and vertex-count+normals blocks verified in every measured LOD and visually confirmed by rendering, with `@<base>{k}.tga` texture chains located) with mech part meshes located in `textures.mw4`; map archives are measured at the geometry and material levels (inventory, zone file formats, fully decoded zone terrain — 64-cell zones, 4×4 grid of 1280-unit zones, vertex arrays, u8 triangle triplets, per-triangle planes — plus per-zone terrain texture sets, the 61-entry `.tcf` texture list, the 256×256 `.material` overlay grid, and the `.fgd` material database, in `docs/engine-specs/map-terrain.md`; exact material mapping, `.bid` trailing data, and the blend equation still open; `.bid` textures decode to RGB555/8-bit with mips and render correctly); official patch payloads and the remaining checklist are open. Measurements commit only facts and provenance; converted disc images, extracted archives, and decoded samples are staged under the gitignored `Installation Files/ReferenceISO/` tree.

Gate: every format required for the slice is readable; fail closed on any format whose reading would require defeating protection.

### Phase 2: vertical slice (timebox: 12 weeks; evidence level: component/smoke)

Scope: an SDL3 application using SDL_GPU; load one mech and one map from user media; free camera plus basic mech movement; play one original movie via FFmpeg or plmpeg; run on macOS arm64, Linux, and Windows. Parity is checked against the original under Wine through recorded side-by-side comparisons, not pixel equality. Full architecture and milestones M0–M6: `active/SDL3_PORT_PLAN.md`; stack and licensing fixed by `decisions/0014-engine-stack-and-licensing.md` (C++20, GPL-3.0, SDL3 + SDL_GPU).

Non-goals: simulation parity, AI, campaign, MechLab, multiplayer, save compatibility.

Gate: the slice runs on all three platforms from original user media. Then the track-level decision gate: pursue, park, or reject via a follow-up ADR. No prototype beyond the slice is authorized here.

### Phase 3: expansion gates (only after the decision gate)

Ordered by risk: simulation core (movement, collision, weapons, heat, damage), then MechLab, then campaign/market state, then netcode (DirectPlay-era multiplayer has dead services and needs a replacement design). Each gate requires its own acceptance evidence, and none of these promises a finished game.

## Scale

- Slice: single-digit person-months for one to three focused contributors.
- Parity across three titles and two packs: multi-year effort; OpenMW-class projects are measured in a decade. Campaign and multiplayer are the dominant risks.
- This track commits to nothing beyond the phase timeboxes; the decision gate exists so the track can stop cheaply.

## Risks

- Enforcement risk exists even for clean-room work; the track depends on the audit, not optimism.
- Clean-room hygiene is difficult in an LLM-assisted workflow; provenance and access discipline are the mitigation.
- Decompilation activity across the MechWarrior franchise is increasing (two references in 2026), raising both taint risk and the chance of external attention; the exclusion list and provenance rules are the mitigation.
- Format dead ends are possible where resources are packed or compressed without public documentation.
- With assets out of scope, visual improvement can only come from the renderer, not new artwork.
- User media variance: retail discs, official patches, and pack versions must map to supported inputs before any engine claim.

## Open questions

- Which community format claims can be independently reproduced, and which need fresh measurement?
- Do any primary rights documents or explicit statements from the current rights holder clarify the scope over MW4's engine and data files?
- Is a second contributor or reviewer required to make the clean-room separation meaningful in practice?
- Minimum viable multiplayer: skirmish-only first, or design a replacement protocol layer?
- What macOS packaging, GPU minimums, and SDL_GPU backend maturity apply to the target OS versions?
- What legal review, if any, should precede Phase 2?

## Next actions

1. Completed: ADR 0013 accepted; Phase 0 closed 2026-10-11; Phase 1 scaffold committed at `docs/engine-specs/README.md`.
2. Completed: `active/SDL3_PORT_PLAN.md` (full-port architecture, milestones M0–M6, clean-room workflow for implementers) and proposed `decisions/0014-engine-stack-and-licensing.md` (C++20, GPL-3.0, SDL3 + SDL_GPU, engine repository at M0).
3. Next slice: continue Phase 1 — pin the directional street/road variant selection, URBAN01's material groups 7/9, and the blend equation (the remaining M3 blocker; geometry, the material grid's surface-group mapping — cross-map validated on ARCTIC01 — and `.bid` pixel formats are decoded); the shell asset tables (`docs/engine-specs/tables.md`), animation clip headers (`docs/engine-specs/animations.md`: 2,375 clips, header/region/joint-directory/value channels verified, full keyframe pairing open), and the scripting languages (`docs/engine-specs/scripting.md`: `.script` shell GUI DSL and `.animscript` state machines, language-level) are decoded, along with per-mech composition and simulation records (`.contents`, `.damage` armor/damage-transfer tables, `.subsystems` loadouts — partial); full keyframe pairing and the remaining checklist items are next; official patch-payload archive validation remains open.
4. Keep the decompilation exclusion list and independent-verification rules in force; revisit the rights posture if primary-scope evidence or a rights-holder statement appears.
5. Keep the current product and Wine/CrossOver tracks unaffected.
