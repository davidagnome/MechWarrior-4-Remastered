# Audio and video assets

Status: measured. Sound storage format, effect descriptors, and movie inventory are decoded; effect-descriptor field semantics and handle companions are open. Facts come from independent measurement; community material was not used.

## Provenance

| Item | Value |
|---|---|
| Audio | Mercenaries `props.mw4`: 2,348 `.wav` + 2,348 `.wav{handle}` entries; sample `audio\cmp_core_breached.wav` (20,976 B) |
| Effect descriptors | Vengeance `CORE.MW4`: `effects\ac_flare\ac10_flare.video` (136 B), `…ac10_flare.audio` (128 B) |
| Video | Disc images: `CONTENT/MOVIES/*.MPG` (numbered `C00007`–`C00038`), `BURNLO_1.AVI`; `GOODIES/MC2/MC2.MPG` on disc 2 |

## Sound files (`.wav`)

Standard RIFF/WAVE containers whose audio payload is **Microsoft ADPCM** — not PCM:

- `fmt ` chunk size 50 bytes (WAVEFORMATEXTENSIBLE): format tag `2` (MS ADPCM), 1 channel, 22,050 Hz, average bytes/s 11,156, block align 512, 4 bits per sample; extension with samples-per-block 1,012 and standard coefficient tables.
- Consequence for the engine: SDL_audio does not ingest MS-ADPCM directly; an MS-ADPCM decoder (or conversion at load time in memory, never re-encoding files) is required for playback.
- Each `.wav` has a 10-byte `{handle}` companion entry (authoring metadata; contents open).

## Effect descriptors (`.audio`, `.video`)

The `.audio` and `.video` archive entries are **not media**; they are small effect descriptors (128 and 136 bytes) in the tagged-record family seen in mech data:

- Common wrapper: `u32 1`, a type id (`0x22F` audio / `0x221` video), a record id, zero, count fields, an identity-matrix float block, then a trailing id pair.
- `.audio` additionally carries float parameters (values like `2.0, 1.0, 0.5, 0.8, 0.525` and `400.0/401.0`) plus two `u16` sound-reference ids — consistent with volume/pitch ranges and the WAV references.
- `.video` embeds a small `#FRE` v14 stub (`#RLM` record with type `10`, sentinels) — the effect's visual resource plug.

## Movies

Movies live outside the `.mw4` archives on the discs (`CONTENT/MOVIES/`): numbered MPEG-1 files `C00007.MPG`–`C00038.MPG` plus `BURNLO_1.AVI`, and goodies (`GOODIES/MC2/MC2.MPG`). The implementation plan plays these via plmpeg (MPEG-1 + MP2 audio) with no re-encoding (see `active/SDL3_PORT_PLAN.md`).

## Open items

- Effect-descriptor field semantics (the type ids, parameter meanings, and the sound-id spaces).
- `{handle}` companion contents (10 bytes) and their role.
- Movie inventory completeness across titles (Black Knight and Mercenaries payloads) and whether any movie lives inside archives.
- Whether any WAVs use PCM at all, and the full range of ADPCM parameters across titles.
