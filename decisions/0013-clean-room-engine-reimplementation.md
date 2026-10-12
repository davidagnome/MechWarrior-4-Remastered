# ADR 0013: Clean-room engine reimplementation track

- Status: accepted
- Date: 2026-10-11

## Context

The shipped product installs user media and launches the original 32-bit Windows executables through a compatibility layer. `active/CROSSPLATFORM_FEASIBILITY.md` records that a native game binary is out of scope (`:9`) because this repository contains no engine source, and the executed macOS spike then proved the compatibility route has structural platform blockers: startup MPEG-1/MP2 movies cannot decode under CrossOver 26 without redistributing DLLs the project will not ship, and the pinned dgVoodoo D3D12 path crashes before window creation under vkd3d.

A native engine would remove those blockers, but only if it is written independently of the original implementation. Changing languages, libraries, or renderer APIs does not reset derivation: code translated or rewritten from decompiled source remains an adaptation of protected expression regardless of how different the resulting stack looks, and enforcement history (for example the 2021 takedown of reverse-engineered Grand Theft Auto III/Vice City ports) shows that reimplementation alone does not guarantee tolerance. The durable precedent is the clean-room model used by projects such as OpenMW: independently written engine code, user-supplied original data files, no bundled game content.

Rights evidence has not changed (`decisions/0002-user-supplied-media-and-release-boundary.md`): the games are not public domain, the 2010 MekTek Mercenaries clearance was limited, and releases require user-supplied media. A clean-room engine that reads those data files removes the grayest step in the current no-disc work because it never needs to defeat or transform SafeDisc/C-Dilla protection.

## Decision

Establish a separate, time-boxed feasibility track for a clean-room engine reimplementation, described in `active/ENGINE_REIMPLEMENTATION_FEASIBILITY.md`. Until its decision gate passes and a follow-up ADR accepts a prototype:

- The shipped installer/launcher and the Wine/CrossOver cross-platform work continue unchanged; no product source, packaging, release scope, or test surface is affected.
- Research is limited to observable behavior and file-format analysis of user media and of the original software running as intended. Decompiled source, disassembly listings, and translated code never enter the repository, committed documents, or auxiliary-model prompts.
- Written specifications are the only transfer channel from research to implementation; implementation code is written from those specifications and public documentation without reading reverse-engineering scratch output.
- The engine requires original user media, ships no game assets, does not execute a retail binary as a dependency, and performs no DRM circumvention.
- Renderer direction for the spike is SDL3 with SDL_GPU (native Metal on macOS, Vulkan/D3D12 elsewhere); raw Vulkan plus MoltenVK is recorded as the alternative pending spike evidence.
- An accepted engine becomes its own component repository through a later ADR; this repository owns only the plan and cross-references until then.

## Alternatives

- Keep the Wine/CrossOver route as the only cross-platform path. Viable, but macOS movie decode and presentation currently have no redistributable fix, and the 32-bit compatibility stack is a moving target.
- Decompilation-based port (DevilutionX/XWVM-style). Rejected: distributing decompiled or translated source distributes a derivative work.
- Rewriting from decompiled code without clean-room separation. Rejected: the stack change does not reset derivation.
- A third-party engine or wrapper. Does not address the 32-bit Direct3D 7 executable boundary the current product already fights.

## Consequences

- Adds a research track with real cost and no product guarantee; each phase has kill criteria and the track can be parked without touching the product.
- Format specifications become public project artifacts and must contain facts and measurements only, never disassembly or decompiled code.
- Honest scale is recorded: a vertical slice is months of focused effort; parity across Vengeance, Black Knight, Mercenaries, and both packs is multi-year, with campaign and DirectPlay-era multiplayer as the largest risks.
- The release boundary is unchanged: user-supplied media, no assets, no serials, no DRM components.
- If accepted later, the engine would not replace the installer/launcher contract; it would consume the same user media through new intake in its own repository.

## Rollback

The track is documentation-only until its gates pass. Parking or rejecting it changes no shipped behavior; the ADR status becomes rejected or superseded and the plan document records the evidence. No product source depends on this track.

## Verification

- Phase 0 completes a rights and prior-art audit with pinned citations, including classifying community material as usable documentation, oracle-only, or prohibited.
- Phase 1 format specifications reproduce parsing of the user's own media with exact counts and commit no disassembly, decompiled code, or media.
- Phase 2 demonstrates the vertical slice from original user media on macOS arm64, Linux, and Windows, with the original running under Wine as the behavior oracle only.
- Repository scans of the track's artifacts find no proprietary content, disassembly, local absolute paths, or executable content beyond what release policy already allows.
