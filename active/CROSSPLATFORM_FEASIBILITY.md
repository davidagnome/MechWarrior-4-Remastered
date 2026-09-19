# Cross-platform feasibility: Linux release candidate, macOS test

Updated: 2026-09-19. Status: Phase 0, Phase 1, and the Phase 3 macOS test executed; Phase 2 (Linux) runbook and bundle ready. Product changes are limited to the F5 lazy ISO-backend fix and its test.

Purpose: decide whether MechWarrior 4 Remastered can support a Linux installer/launcher that orchestrates Wine, and whether a macOS path exists, without recompiling the games. This document is evidence and a spike plan, not a release claim.

## Premise correction

This repository contains no game or renderer source. The games remain the original 32-bit x86 Windows executables staged from user media; the "remaster" is an installer/launcher plus a compatibility layer. A cross-platform release therefore means a cross-platform installer/launcher and a Wine/CrossOver runtime, not a native game binary. Native Vulkan/MoltenVK game rendering would require a clean-room engine reimplementation and is out of scope.

## Ground truth

| Item | Fact |
|---|---|
| Game binaries | 32-bit x86, DirectDraw/Direct3D 7, SafeDisc retail (`PCGamingWiki` Vengeance, retrieved 2026-09-18) |
| Windows renderer | dgVoodoo2 2.87.5 x86 DirectDraw/D3DImm wrappers with D3D12 backend plus the source-built presentation add-on (`active/ARCHITECTURE.md`; `tools/compatibility/patches/dgVoodoo2-MW4-Presentation.patch`) |
| C# split | `MW4Remastered.Core` targets `net10.0`; Launcher and Installer are WinForms `net10.0-windows`; `MW4Remastered.RtpPatchHost` is self-contained `win-x86`; packaging is Inno Setup (`packaging/MechWarrior4Remastered.iss`) |
| Release pipeline | `tools/package/build-release.ps1:34-43` publishes `win-x64`/`win-x86` self-contained single files and compiles Inno |

## Pinned external evidence (retrieved 2026-09-18)

| Source | Pinned detail | Confidence |
|---|---|---|
| Lutris installer `34602`, "CD + dgVoodoo2" (`lutris.net/games/mechwarrior-4-vengeance`, JSON API) | Last published update `2026-08-06T12:21:14Z`. Wine `win64` prefix, winetricks `quartz amstream directplay`, `ddraw=n,b`, dgVoodoo2 x86 `DDraw.dll`/`D3DImm.dll`, `dgVoodoo.conf` `FPSLimit=30` and watermark off, launch `MW4.exe -noautoconfig`. Known issues: first-launch "Error outside main thread"; multiplayer does not close cleanly | High; primary script |
| PCGamingWiki Vengeance (`pcgamingwiki.com/wiki/MechWarrior_4:_Vengeance`) | Direct3D 7, 32-bit, SafeDisc; `-noautoconfig`/`-noautoconfigx`; `/gosNoJoystick` family; ~30 FPS external cap for physics | High; wiki primary enough |
| Whisky docs maintenance notice (`docs.getwhisky.app/maintenance-notice`) | "Whisky is no longer actively maintained", pinned commit `f157e66`, last change 2025-04-09; repository archived 2025-05-11 | High |
| Apple Game Porting Toolkit page (`developer.apple.com/games/game-porting-toolkit/`) and AppleGamingWiki GPTK page | GPTK 4 current; evaluation environment for Windows games; D3DMetal path is D3D11/D3D12, anti-cheat/aggressive DRM generally unsupported | High for D3D11/12 coverage; D3D7 not covered |
| WindowsForum article on CrossOver 27 (2026-06-12) | CrossOver 27 becomes Apple-Silicon-only, macOS Sonoma+, and removes 32-bit bottles; 64-bit bottles cover "almost all" 32-bit apps; CrossOver 26 is the last Intel/32-bit-bottle line | Medium; secondary source, verify against CodeWeavers notes before relying |
| Local macOS host | macOS 27.0 (26A428), arm64, Rosetta package present, CrossOver 26.3.0.39832 with `bin/wine` CLI and bundled `cabextract`; Whisky support directory present; .NET SDK 10.0.300 absent (pinned `global.json` blocks repo builds) | High; direct inspection |
| Local media | No MW4 ISO/extracted media on this Mac; only `MechWarrior-4-Remastered-main.zip` source archive | High; direct search |

## Phase 1 portability audit

### Portable today (pure managed, no Windows API)

- Media recognition and catalog: `MediaCatalog`, `MediaRecognizer`, `DirectoryMediaInventory`, `MediaSelectionSet`, ZIP handling in `IsoArchiveExtractor`, `MechPakIsoProjection`, `MercenariesPr1ArchiveExtractor` (DiscUtils 0.16.13 is pinned in `MW4Remastered.Core.csproj`).
- All executable/data transforms: `VengeanceExecutableTransform`, `MechPakActivationTransform`, `BlackKnightPr1ExecutableTransform`, `MercenariesExecutableTransform`, SafeDisc cipher helpers, official Patch 3/PR1 payload rules.
- Install mechanics: plans, `StagedInstallTransaction`, manifest verifier, overlay/replacement transactions, `OwnedInstallUninstaller`, `InstallDestinationPlanner` (`DriveInfo` free-space works on Unix with Unix paths).
- Launch preparation: `LegacyGameConfiguration` INI writing; `ActiveMonitorResolutionProvider` already falls back to `1024x768` off-Windows (`LegacyGameConfiguration.cs:46`).
- `MW4Remastered.MediaProbe` and the Core test suite both target `net10.0`; the Core suite was executed off-Windows (below), the probe was not exercised in this phase.

### Windows-only surface and replacement direction

| Boundary | Files | Dependency | Replacement direction |
|---|---|---|---|
| ISO mounting | `Media/PowerShellDiskImageBackend.cs:13-65`, composed at `Media/MediaSourceSession.cs:20` | Windows PowerShell `Mount-DiskImage` | DiscUtils ISO reader behind the existing `IDiskImageBackend`; no elevation |
| Registry | `Launch/LegacyGameRegistration.cs:21-70` | `Microsoft.Win32` HKCU 32-bit view | Wine prefix registry via `wine reg add` (as Lutris does), or treat as optional |
| Monitor and window control | `Launch/LegacyGameConfiguration.cs:51-86`, `Launch/GameWindowLifecycleGuard.cs:83-232` | `user32.dll` | Platform window/display services; Wine/gamescope virtual desktop for borderless/Alt-Tab |
| Black Knight capture | `Install/BlackKnightPr1ImageCapture.cs` (job objects at `:248-346`, `version.dll` hook at `:59`) | `kernel32` jobs + x86 capture DLL + protected launcher | Run the whole capture under Wine, or reimplement containment per platform |
| x86 helper processes | `Install/OfficialVengeancePatch3Transform.cs:183-218`, `Install/OfficialMercenariesPr1Transform.cs:124-157`, `Install/OfficialBlackKnightPr1Transform.cs:193-218`, `Install/CabinetPayloadExtractor.cs:11` | `Process.Start` of RTP host, protected launcher, `tar.exe` | Run `MW4RemasteredRtpPatchHost.exe` under Wine (spike) or port the RTP application to managed code; pick platform archive tool |
| Elevation and setup UI | `MW4Remastered.Installer/Program.cs:32-35`, `packaging/MechWarrior4Remastered.iss:24,331` | `WindowsIdentity`, UAC, Inno `runas`, `netsh` firewall | No elevation needed off-Windows; replace Inno with tar/AppImage/Flatpak and per-distro firewall docs |
| Launcher UI | `MW4Remastered.Launcher` (`net10.0-windows`, WinForms, `MainForm.cs`) | WinForms/GDI+ | Cross-platform UI (Avalonia) reusing Core services |

### macOS portability probe (executed)

Environment: macOS 27.0 arm64, .NET SDK 10.0.400, worktree of `HEAD efafa6a` with `global.json` temporarily removed (the repo pins SDK `10.0.300` with `rollForward: disable`).

```bash
git worktree add --detach <temp>/mw4-portability-probe HEAD
rm <temp>/mw4-portability-probe/global.json
TMPDIR=<temp>/mw4-probe-tmp PATH="<temp>/mw4-probe-bin:$PATH" \
  dotnet run --project tests/MW4Remastered.Core.Tests/MW4Remastered.Core.Tests.csproj --configuration Release
```

(`<temp>/mw4-probe-bin/tar.exe` was a symlink to `/usr/bin/tar`; see F2.)

Results: the full Core suite builds and runs on macOS arm64. Exit code 1 with exactly four failures, all in the two transaction smoke files. Findings:

- **F1 — ancestor symlink rejection (product portability gap).** Without the `TMPDIR` override the suite aborts: `StagedInstallTransaction.RejectReparsePoint` (`Install/StagedInstallTransaction.cs:201-220`) rejects `/var` because macOS symlinks it to `/private/var`. Test temp roots routinely live under such a path. Filtering reparse checks to the install tree or resolving ancestors is needed before any UNIX support.
- **F2 — hardcoded `tar.exe` (product portability gap).** `CabinetPayloadExtractor` defaults to `tar.exe` (`Install/CabinetPayloadExtractor.cs:11`). On macOS a `tar` shim works for the suite's ZIP fixtures, but Linux GNU tar cannot read Microsoft CAB payloads; the real Mercenaries cabinet needs `bsdtar`/libarchive, `cabextract` (bundled with this Mac's CrossOver), or a managed reader.
- **F3 — Windows share-mode fault injection does not port (test-harness gap).** The four failures (`OwnedInstallOverlayTransactionSmoke.cs:55-71`, `OwnedInstallFileReplacementTransactionSmoke.cs:64-80`) hold the manifest open with `FileShare.Read` to force a commit failure. On Unix that lock does not block the `File.Replace`/`File.Move` commit (`OwnedInstallOverlayTransaction.cs:61-66`, `OwnedInstallFileReplacementTransaction.cs:95-145`), so the injected failure never happens and the rollback assertions fail. The product transactions themselves ran; they need injectable failure seams for cross-platform qualification.

Local evidence log: `.local/crossplatform/core-tests-macos.log` (untracked).

Conclusion so far: the managed logic is highly portable; the Windows-only list is small, named, and replaceable behind existing interfaces. No blocker found in Phase 1. Runtime feasibility remains unproven.

## Phase 2 runbook: Linux spike (GPU machine)

Evidence level target: smoke. Do not claim playability from process responsiveness.

Preconditions: same media inputs as the Windows qualification; Vulkan-capable GPU with current driver; record `wine --version`, distro, kernel, GPU driver, and `vulkaninfo --summary` output. Use a disposable prefix and disposable game tree.

1. Build and run the Core suite on Linux from a clean checkout with SDK `10.0.300`; record that only F1/F3-class failures occur, as on macOS.
2. Media intake: run `MW4Remastered.MediaProbe` against extracted user media directories (directory input avoids the PowerShell mount backend). Record recognized layouts and exclusion counts.
3. Transforms: apply project Vengeance/Mercenaries/Black Knight transforms on Linux using extracted media; verify output hashes against the Windows-qualified values (`MW4.exe`, `MW4x.exe`, `MW4Mercs.exe`).
4. Windows helper processes under Wine: run `MW4RemasteredRtpPatchHost.exe` and the Black Knight capture flow inside the prefix; record exit codes and artifact hashes. This is the highest-risk step; a failure here is the primary candidate for Linux-specific work.
5. Runtime: copy the exact dgVoodoo 2.87.5 x86 DLLs, the packaged profile, and the presentation add-on into the staged tree; launch with the project argument set (`-32 -noautoconfig -f <4:3> -gl -GameTime.MaxVariableFps 30 /gosNoJoystick`; Black Knight `-noautoconfigx /gosNoJoystick`).
6. Verify per title: startup cinematics, menu, pilot creation, one mission, 4:3 presentation, watermark absent, seam correction behavior, mouse behavior, Alt-Tab. Capture logs and exact-window screenshots.
7. Try the Lutris oracle configuration as a comparison arm, and record differences from project outputs.

Pass: all three titles reach a mission from project-produced trees under Linux without a Windows VM, with presentation at least equal to the current Windows baseline. Fail: enumerated blockers with logs.

Executable bundle: `.local/crossplatform/linux-bundle/` (untracked) contains `linux-spike.sh`, a self-contained `linux-x64` publish of the Core staging runner, the x86 window-lifecycle probe, the x86 RTP host, and the byte-exact profile. Run `./linux-spike.sh --disc1 <path> --disc2 <path> --work <dir>` and return `<work>/evidence`; it stages through the project transforms, verifies the pinned hashes, and records both the WineD3D and dgVoodoo arms.

## Phase 3 runbook: macOS test (this machine, time-boxed)

Preconditions: user media copied to this Mac (none present today); CrossOver 26.3 is the last line with 32-bit bottle support per the secondary CrossOver 27 report; Rosetta is installed.

1. Create a new bottle (`mw4-feasibility`); do not touch the existing `uaf-oracle` bottle.
2. Stage a disposable tree from extracted media and project transforms (a native `dotnet` runner can execute `MW4Remastered.Core` staging logic once a non-Windows disk-image backend exists; otherwise stage manually).
3. Install/run `MW4.exe` 32-bit under the bottle; record whether the legacy loader and the project's no-disc output execute at all.
4. Rendering arms: CrossOver WineD3D path first; then dgVoodoo 2.87.5 D3D12 through CrossOver's D3DMetal libraries (`/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/lib64/apple_gptk/external`).
5. Record exact versions and results; no time-box extension without new evidence.

No-go triggers: 32-bit execution fails under Rosetta/CrossOver; D3D7 cannot render acceptably through available arms; or the only working configuration depends on the CrossOver-26 32-bit path that CrossOver 27 removes and Rosetta is phasing out. On any no-go, document a community recipe if one works and recommend macOS stays unsupported.

## Phase 3 executed results: macOS (2026-09-18)

Environment: macOS 27.0 arm64, CrossOver 26.3.0.39832, Rosetta, disposable 32-bit WoW64 bottle `mw4-feasibility`. Media: user-supplied Vengeance `.mdf/.mds` discs converted to ISO with a local raw-sector converter (Mode 1, 2048-byte payload) and extracted with 7z. Evidence logs under `.local/crossplatform/evidence/`.

Passed:

- Native macOS staging: `VengeanceRetailExecutableTransform` produced exact SHA-256 `48ba4baf0894f0c14c44ddaf503cf953f59be3b919817156f1f11ea9b0df0de4` from the two real discs; `VengeanceInstallPlanBuilder` plus `StagedInstallTransaction` committed 231 files and `InstallStatusReader` verified Vengeance `Ready` with `MW4.exe` and a valid manifest. This ran through an untracked local runner (`.local/crossplatform/MacSpike`) because the shipped inspector default constructs the Windows-only ISO backend eagerly (F5).
- The game launches under CrossOver 26.3 on Apple Silicon with a real, visible, non-minimized window and no display-mode change (2560x1440@144). The project's `WindowLifecycleProbe`, patched for x86 (F8), recorded the snapshots in `.local/crossplatform/evidence/mw4-baseline.json`.
- Native macOS tooling after the F5 fix: `MW4Remastered.MediaProbe` recognized both extracted Vengeance discs (`RecognizedWithExcludedContent`, layout ids `vengeance-disc-1`/`vengeance-disc-2`, `SECDRV.SYS` excluded). The platform-neutral contract suites (`PresentationCompatibilityContract`, `PackagingContract`, `ReleaseTreePolicy`, `UserFlowContract`) pass on macOS under PowerShell 7.6.6.
- User-verified with the movie diagnostic flag `/gosnovideo`: the Vengeance main menu renders and Instant Action launches. The game is playable in a window on macOS.
- User-verified blocker without the flag: `STOP: You do not have the correct video codec installed to play 'Content\Movies\Gameopen.mpg'`.

Failed or blocked:

- dgVoodoo 2.87.5 (exact pinned hashes) under CrossOver: the D3D12 backend initializes through vkd3d, but the game exits with `0xC0000005` before creating a window; the synthetic D3D7 probe reproduced a surface failure (`0x887600D4`) and page fault. Switching the profile to `OutputAPI=d3d11_fl11_0` hung the probe beyond 300 seconds. The project's borderless 4:3 presentation has no working macOS path yet.
- MPEG-1/MP2 movie playback: CrossOver's bundled GStreamer ships only `libgstavi.dylib` with no MPEG-PS demux or MPEG video/audio decoders. Registering x86 LAV Filters in the prefix did not help because Wine's builtin quartz uses the GStreamer path rather than native DirectShow filters. CrossOver's `winegstreamer.so` does not honor an external GStreamer (no `WINE_GSTREAMER_DIR` support), replacing app-bundle plugins would break signing, and winetricks cannot drive CrossOver's wine wrapper. The only untested route is shipping native DirectShow DLLs, which the project will not redistribute. A macOS release needs additional GStreamer plugins from the Wine/CrossOver vendor or a different runtime.
- CrossOver 26.3 creates only 32-bit WoW64 prefixes here (`Bottle:Arch=x86_64` was ignored), so any x64 helper must wait for a 64-bit-bottle route or stay x86 (F8).

New portability findings:

- **F4 — single-file .NET fails under CrossOver.** The self-contained `win-x86` single-file `MW4RemasteredRtpPatchHost.exe` aborted with `Failed to load System.Private.CoreLib.dll (0x80070005)` from both `Z:` and `C:` paths; the folder-published build ran and printed its usage line. Wine packaging of the RTP host must use folder publish (or a different launcher).
- **F5 — eager Windows ISO backend (resolved).** `MediaSourceSessionFactory()` constructed `PowerShellDiskImageBackend` in the constructor, so directory media inspection threw `PlatformNotSupportedException` on macOS and would throw on any Wine prefix without Windows PowerShell. Fix applied: backend construction is deferred through `Lazy<OwnedIsoMediaSessionFactory>`, so directory sessions never construct it and only ISO/ZIP media resolve it, with a focused Core test. Verified on macOS: the Core suite shows no new failures and `MW4Remastered.MediaProbe` natively recognized both extracted Vengeance discs (`RecognizedWithExcludedContent`, `SECDRV.SYS` reported for exclusion).
- **F6 — profile hash mismatch (resolved).** The committed `assets/compatibility/dgVoodoo-MW4.conf` was the all-LF form (21,507 bytes, `7ae6dba6…`), while the release gate requires `fd9413ae…`. The shipped `0.6.44` setup was downloaded (SHA-256 `a3fdde1f…` verified), installed into a disposable 64-bit CrossOver bottle with `/VERYSILENT /ACCEPTLICENSE=1 '/MEDIAFILES=C:\dummy.iso'`, and its payload inspected. The shipped `Compatibility\dgVoodoo2\dgVoodoo.conf` is 21,831 bytes with mixed line endings (324 CRLF plus 37 LF) and hashes exactly `fd9413ae…`; the content is otherwise identical. Fix applied: the asset now carries the exact shipped bytes, `.gitattributes` marks it `-text` so no platform normalizes it, and `PresentationCompatibilityContract.Tests.ps1` asserts the file's real SHA-256 against the lock. A staged-blob check produced `fd9413ae…`.
- **F7 — stale mirror in the lock (resolved).** `third_party/dgVoodoo2.lock.json` `binaryArchiveMirror` pointed at a `dgVoodoo2_87_5.zip` path that returns 404. It now points at the upstream GitHub release `v2.87.5` asset, whose archive and `MS/x86` DLL hashes verified against the lock byte-for-byte.
- **F8 — WindowLifecycleProbe is x64-only.** It P/Invokes `GetWindowLongPtr`, which does not exist in 32-bit `user32`, so it cannot run in a 32-bit prefix (nor on Windows x86). A local one-line fallback to `GetWindowLongW` enabled the x86 run; the tool should select the entry point by pointer size.


## Legal and provenance notes

- Wine is not bundled; CrossOver is commercial and cannot be redistributed by this project. Whisky is unmaintained and archived.
- dgVoodoo terms already permit shipping individual files with the game; the project's own add-on is source-built.
- No game media, serials, or extracted proprietary trees leave local storage or enter the repository; auxiliary models receive sanitized text only.

Working macOS reproduction (local state, 2026-09-18):

- Bottle: `~/Library/Application Support/CrossOver/Bottles/mw4-feasibility`; staged tree `C:\MW4Test\vengeance`; HKCU `MechWarrior Vengeance` registration written.
- The pinned dgVoodoo files are installed in the tree; rename `DDraw.dll`, `D3DImm.dll`, `D3D8.dll`, `D3D9.dll`, and `dgVoodoo.conf` to `*.bak` so Wine's built-in DirectDraw is used.
- Launch: `CX_BOTTLE=mw4-feasibility wine 'C:\MW4Test\vengeance\MW4.exe' -32 -noautoconfig -f 1920x1440 -gl -GameTime.MaxVariableFps 30 /gosNoJoystick`; add `/gosnovideo` to bypass the unplayable startup movie.

## Media input paths and extracted directories

The worker layer already accepts extracted directories: `MediaSourceSessionFactory.Open` handles `Directory.Exists` (`src/MW4Remastered.Core/Media/MediaSourceSession.cs:41-49`), and `MediaSourceInspector.Inspect` dispatches through the same session type used for ISO and ZIP. The Inno media page is the only restriction: its file dialog filters `*.iso;*.zip`, while `/MEDIAFILES=<path>` accepts any path and `NextButtonClick` only rejects an empty list. A directory path passed through `/MEDIAFILES` therefore reaches the same recognition and exact-hash gates as an ISO. The shipped installer was exercised this way for payload extraction.

Making extracted directories a first-class, documented input would require: an "Add folder" affordance and validation in `packaging/MechWarrior4Remastered.iss`, explicit README/BUILDING wording, and an ADR because `AGENTS.md` currently scopes accepted input to original ISOs or recognized archival ZIP layouts. The transformations and manifests already fail closed on unrecognized or modified trees, so the main trade-off is contract scope and user guidance, not missing verification.

## Open questions

- Does the Black Knight capture (job-object containment plus SafeDisc-era protected launcher) survive under Wine, or is a platform-specific capture needed?
- How should movie decode be supplied on macOS and Linux: bundled GStreamer plugins, a Wine build with full GStreamer, or an in-prefix decoder, given Wine's builtin quartz ignores native LAV filters?
- Is there a workable macOS presentation path without dgVoodoo (WineD3D windowed, Wine virtual desktop at a 4:3 size, or a fixed dgVoodoo configuration), and can the project's borderless 4:3 look be reproduced?
- Do mouse, joystick, cursor capture, and Alt-Tab behave acceptably under Wine without the Windows window-lifecycle guard?
- Which distribution format is supportable: tar + desktop entry, AppImage, or Flatpak, and what replaces Inno's uninstall registration?
- Why does `assets/compatibility/dgVoodoo-MW4.conf` not hash to the value required by the release gate (F6), and which bytes were actually packaged in `0.6.44`?

## Next actions

1. Verify the F6/F7 fix on Windows: confirm `build-release.ps1` passes its profile hash gate and commit the asset, `.gitattributes`, test, and lock changes. The owning contract suites already pass on macOS via PowerShell 7.6.6 (presentation, packaging, release-tree, user-flow).
2. Review this document and choose the Linux packaging and UI direction (ADR under `decisions/`), including whether extracted directories become a documented input.
3. Run the Phase 2 spike on the GPU Linux machine with user media (`.local/crossplatform/linux-bundle/linux-spike.sh`) and return the evidence directory.
4. If Linux proceeds, plan the OS-abstraction work (disk-image backend, archive tool, capture containment) plus F1/F4/F8 fixes as a named slice.
5. For macOS, scope a follow-up slice for movie decode (F9) and presentation (dgVoodoo or virtual-desktop) before making any support claim.
