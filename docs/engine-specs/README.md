# Engine format specifications (Phase 1)

Status: active. The `.mw4` archive container is measured and validated ([mw4-archive.md](mw4-archive.md), seven retail artifacts across all three titles); ERF geometry, mech skeletons, and texture binding are measured and visually verified ([erf-geometry.md](erf-geometry.md)); map archives are measured and cross-map validated ([map-terrain.md](map-terrain.md)); the runtime rendering pipeline is measured from a Direct3D 7 capture ([rendering-pipeline.md](rendering-pipeline.md)); the shell asset tables are measured ([tables.md](tables.md)); animation clip headers are partially measured ([animations.md](animations.md)); the scripting languages are measured at the language level ([scripting.md](scripting.md)); mech composition records are partially measured ([erf-geometry.md](erf-geometry.md)); audio/video formats are measured ([audio-video.md](audio-video.md)); the rest of the checklist is open. Authorized by `decisions/0013-clean-room-engine-reimplementation.md`; phase plan in `active/ENGINE_REIMPLEMENTATION_FEASIBILITY.md`; implementation plan in `active/SDL3_PORT_PLAN.md`.

## Rules

1. Facts only. Every statement in a specification must come from measuring the user's own media copy. No disassembly listings, decompiled code, or community text copied verbatim.
2. Provenance per fact: record the source artifact (role and SHA-256), the tool and version, and the exact command or procedure that produced the observation.
3. Exact counts: parser statements are validated against real files with counts (records, offsets, terminator conditions) and reproduced on a second artifact where one exists.
4. Fail closed on protection: if reading a format requires defeating a protection or using prohibited material, that format stops at `blocked` and its branch parks.
5. No media in the repository: extracted content stays in gitignored local staging (`Installation Files/ReferenceISO/`, plus disposable copies); only structural facts and small derived metadata tables (names, offsets, counts) are committed, and never game bytes.
6. Community material is read only as a falsifiable hypothesis (oracle-only), never as a source of facts or code. Unlicensed code is never copied.
7. Measurement scratch stays untracked under `.local/re/`.

## Method

1. Work from a disposable copy of extracted user media; never modify the source tree.
2. Measure with small disposable readers (for example Python struct parsers) run from `.local/re/`; record the command and the artifact hash in the specification's provenance log.
3. Cross-check every structural claim against a second artifact of the same class (for example a second mission archive) before it becomes a specification statement.
4. Write each format's findings to `docs/engine-specs/<format>.md` with: overview, layout tables, provenance log, validation section with exact counts.
5. Topics that come from community prior art are hypotheses until independently reproduced; mark them as such.

## Target media (user-supplied, never committed)

| Input | Role | Needed for |
|---|---|---|
| Vengeance disc 1 and 2 | base game | archives, missions, shell, movies, audio |
| Vengeance Patch 3 payload | final data state | canonical archive versions |
| Black Knight disc and PR1 payload | expansion | `MW4X` data and script variants |
| Mercenaries disc 1 and 2, PR1 payload | standalone title | archives, missions, campaign |
| Inner Sphere and Clan Mech Paks | optional packs | pack resource overlays |

## Format checklist

| Format | Status | Notes |
|---|---|---|
| `.mw4` archives (`RESOURCE/core.mw4`, `props.mw4`, `textures.mw4`, per-mission `RESOURCE/MISSIONS/*.mw4`) | measured (Mercenaries retail) | [mw4-archive.md](mw4-archive.md); cross-title validation open |
| Asset tables (`tables\*.tbl`: mech, chassis, weapons, subsystems, skins) | measured ([tables.md](tables.md): path/id/name records validated byte-exact across five tables; 21 mechs, 21 chassis, 57 weapons, 6 subsystems, 37 skins; per-mech `.subsystems`/`.damage` records partially decoded incl. per-joint armor and damage-transfer weights) | `missionnames.tbl` layout, cross-title id relations, per-mech record fields open |
| ERF geometry | measured and visually verified (`#FRE` v14, `#RLM` records, per-record positions+UVs, strip-order indices, per-LOD normals, record-name texture chains with the stored-row/GL-v sampling convention, and the `#BBO` joint hierarchy with verified composition; the assembled mech matches the MW4 reference; see [erf-geometry.md](erf-geometry.md)) | pre-count float region, data-block framing fields, `{hint}` semantics, `mad_torso` stray index open |
| `.mw4anim` animation | partially measured ([animations.md](animations.md): clip header, region table, joint directory, and value channels verified/decoded on three clips incl. the fully dumped water clip; joint directory references skeleton site names; 2,375 clips in Mercenaries `props.mw4`) | complete keyframe pairing and track-directory semantics open |
| `.contents` hierarchies | partially measured ([erf-geometry.md](erf-geometry.md): mech `.contents` = fixed 282-byte composition records with `joint_world`, skeleton `.obb` reference, effect-holder and subsystem paths; per-mech `.engine`/`.torso`/`.data`/`.subsystems`/`.damage`/`.instance` and wreck `.obb` siblings located) | field semantics, subsystem/damage layouts open |
| Map terrain and mission data | measured, cross-map validated (terrain geometry decoded, every check 64/64 on four zones; material grid mapped to FGD surface-group ids, confirmed on `ARCTIC01` where 12=snow, 6=rock, 11=airfield; `.tcf` texture list and `.bid` base/mip formats decoded; [map-terrain.md](map-terrain.md); mission metadata decoded in [scripting.md](scripting.md): `.nfm` scenario definitions with `$(ruleset)` variants, help texts, thumbnails) | URBAN groups 7/9, directional-variant selection, `.bid` mip layout, shader blend equation, city-side object records, `GameTypes.h` ruleset table open |
| `.script` shell and mission scripting | measured at the language level ([scripting.md](scripting.md): CRLF text, C preprocessor + `$(NAME)` expansions, `GUI_CREATE`/`EXECUTE`/`DRAW` object blocks, path macros from `stddefs.h`, `localize$(IDS_*)` via `ScriptStrings.h` ids; mission logic = `MISSIONS/*.mw4` archives with ABL FSM scripts, name tables, lightmaps, drop zones, bounds, graph data) | full ABL grammar and engine-function catalogue, execution model open |
| `.animscript` animation state machines | measured at the language level ([scripting.md](scripting.md): `!NAME=value` defines, `[State]` sections with `AnimType`/direction clips, curve enums) | transitions and controller binding open |
| RTP shell layout | reconnaissance ([scripting.md](scripting.md): shell scripts, defs and string-id headers located under `Content\MercsShellScripts\`/`ShellScripts\`) | full shell asset graph open |
| `options.ini` / `optionsx.ini` | open | sections, line-ending requirements, rewrite behavior |
| Save, pilot state, and `.blackmarket` campaign files | open | record schemas |
| Audio (`.wav`) and video (MPEG-1/MP2, `.avi`) | measured ([audio-video.md](audio-video.md): WAVs are MS-ADPCM 4-bit mono 22.05 kHz (decoder required); `.audio`/`.video` entries are effect descriptors; movies are disc-side `CONTENT/MOVIES` MPEG-1 + AVI) | effect-descriptor fields, `{handle}` contents open |

Format names and topics above are measurement targets, several drawn from oracle-only prior art; they are hypotheses until measured.
