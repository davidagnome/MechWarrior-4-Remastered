# ADR 0014: Engine implementation stack and licensing

- Status: accepted
- Date: 2026-10-11

## Context

ADR 0013 accepted a clean-room engine feasibility track and deferred the implementation stack and licensing. The port plan (`active/SDL3_PORT_PLAN.md`) needs firm inputs: language, license, renderer direction, and where the code lives. The track follows the OpenMW precedent (independently written engine code, user-supplied original media, no bundled assets), and the release boundary (`decisions/0002-user-supplied-media-and-release-boundary.md`) already excludes media and DRM material.

## Decision

- Implementation language: **C++20**.
- Engine license: **GPL-3.0**, matching the OpenMW precedent and keeping distributed engine code open.
- Platform and renderer: **SDL3 with SDL_GPU** (native Metal on macOS, Vulkan on Linux, D3D12 on Windows). Raw Vulkan plus MoltenVK is recorded as the alternative if SDL_GPU gaps appear.
- Shaders: offline compilation with SDL_shadercross to SPIR-V/MSL/DXIL.
- Video decoding: plmpeg preferred; FFmpeg only with recorded LGPL/GPL obligations.
- Repository boundary: the engine is developed in **its own repository**, created when Phase 2 starts; this repository keeps the plan, specifications, and provenance. The root repository remains the orchestration/source repository for the installer/launcher product.
- Clean-room separation carries into the engine repository: implementation is written only from `docs/engine-specs/`; `.local/re` research scratch is never copied; contributors attest to no access to decompiled code.

## Alternatives

- C or Rust: viable, but rejected in favor of SDL3's native idiom, the graphics/shader tooling ecosystem, and alignment with the reference and parity tooling.
- Permissive engine license (MIT/Zlib): rejected to keep derivative engine code open and to match the precedent; dependency licenses must be GPL-compatible.
- Developing the engine inside this repository: rejected to keep release policy (Defender scanning, payload allowlists) and the clean-room separation straightforward.

## Consequences

- GPL-3.0 constrains third-party code choices: no copying from permissive-only or proprietary sources.
- A second repository adds coordination overhead; the specifications in this repository are the contract.
- The slice can start at Phase 2 with no further stack decisions.

## Rollback

Until the engine repository is created, nothing in the product changes. Reversing the language or license before code publication requires only a superseding ADR and plan edits.

## Verification

- The port plan cites these decisions and contains no contradicting stack choices.
- Engine repository creation (Phase 2 start) references this ADR and the clean-room protocol.
