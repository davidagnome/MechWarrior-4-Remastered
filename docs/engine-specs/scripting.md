# Scripting formats (`.script`, `.animscript`, `.h` defines)

Status: measured at the language level. Both text DSLs, their preprocessor forms, path macros, the localization mechanism, and the mission scenario (`.nfm`) dialect are decoded; full grammar inventories and the execution model are open. Facts come from independent measurement; community material was not used.

## Provenance

| Item | Value |
|---|---|
| Source | Mercenaries `props.mw4`: 132 `.script`, 48 `.animscript` |
| Samples | `mercsshellscripts\AskUpdate.script` (3,671 B), `mercsshellscripts\stddefs.h` (6,695 B), `mercsshellscripts\ScriptStrings.h` (111,955 B), `mechs\atlas\atlas.animscript` (36,734 B), `buildings\air_control_tower\air_control_tower.animscript` (125 B); `MISSIONS\GladiatorPit.nfm` (814 B) with `bat.txt`/`des.txt`/`missionplay.txt` from `MSGAME.CAB` |
| Extraction | `.local/re/vbd_list.py --dump` into the gitignored `.local/re/dump/` scratch |

All samples are CRLF text with no NUL bytes. The shell's include paths reference `Content\MercsShellScripts\` (Vengeance uses its own `Content\...\ShellScripts\` tree).

## Shell and mission scripts (`.script`)

A C-preprocessor-driven, C-like GUI scripting language:

- **Preprocessor**: `#include "Content\\MercsShellScripts\\stddefs.h"`, `#define NAME value`. `stddefs.h` defines the path macros used throughout scripts: `APATH` = `Content\Audio\`, `NPATH` (news audio), `MPATH` = `Content\Movies\`, `MAPPATH` (HUD textures), `SPATH`/`GPATH`/`WPATH`/`FPATH`/`ZPATH`/`FILEPATH`/`MB_*` for shell scripts, graphics, widgets, fonts, mech icons, and MechBay assets.
- **Custom expansions**: `$(NAME)` substitutes preprocessor variables (distinct from C); `localize$(IDS_...)` resolves a localization id; `FPATH expr`, `GPATH expr`, `WPATH expr`, `APATH expr` prefix a path scope onto a string.
- **Localization**: `ScriptStrings.h` is a generated header (`Microsoft Developer Studio ... StringResource.rc`) mapping `IDS_*` names to integer ids only — the localized text itself is not in the file (it lives in Windows string resources).
- **Structure**: top-level `main { ... }` plus per-object sections `o_<Object> { ... }`, each containing lifecycle blocks `GUI_CREATE { }`, `GUI_EXECUTE { }`, `GUI_DRAW { }`; nested statements use C-like control flow (`for`, `if/else`, `+=`, arrays).
- **Types observed**: `int`, `bool`, `object` (with arrays), `sound`, `Font3d`, `pane`, plus object properties (`location`, `region`, `text`, `file`, `total_states`, `framerate`, `alpha`-style custom fields) and calls (`getresx()`, `getresy()`, `initialize()`, `unfocus`, `script_end`).

## Animation state machines (`.animscript`)

Two-part text format (no `#` preprocessor; its own variable syntax):

1. **Definitions**: `!NAME=value` substitutions plus `//---` comment banners. Mech scripts define clip paths via templates, for example `!PATH=Content\Mechs`, `!Back=$(PATH)\$(MECH)\animation\$(MC)_back`, and numeric curve enums (`!BellCurveCurve=8`, `!SpikeCurve=9`, `!LongBumpCurve=10`, `!BumpSpikeCurve=11`, `!SpikeBell=12`).
2. **States**: `[StateName]` INI sections (for example `[StandState]`) with keys: `AnimType` (`FullHeightPoseHolderType`, `CycleType`), `Start`, `Position`, `PitchAttrib=LocalGroundPitch`, `RollAttrib=LocalGroundRoll`, and per-direction clip references `UpAnim`/`DownAnim`/`EvenAnim`/`L...` composed from the variables. The Atlas script has 221 state mentions and 40 transition mentions.

Small building scripts use the same INI style with a single state (for example `ACTLight_Rotate` under `CycleType`), referencing `Content\Buildings\...\animation\...`.

## Scenario definitions (`.nfm`) and mission metadata

Mercenaries ships `GAME/RESOURCE/MISSIONS/*.nfm` inside `MSGAME.CAB` (LZX cabinet; 85 files: `core.mw4`, 12 `MAPS/*.mw4`, and the mission metadata set). Each map gets a `.nfm` scenario file plus help text and a thumbnail:

- `.nfm` is CRLF text in the same `!NAME=value` preprocessor dialect as `.animscript`: `!include=Content\GameTypes.h`, `!preservecase=true`, then an `[info]` section (`author`, `info`, `link`, `longname`) and a `[scenarios]` section of named blocks, e.g. `GladiatorPit1={ [server] name=… ruleset=$(StockDestruction) teamAllowed=0 teamcount=0 scenariotext=resource\missions\des.txt }`.
- Measured patterns: four scenario variants per map (solo/team × Destruction/Battle), each referencing a ruleset variable (`$(StockDestruction)`, `$(StockTeamDestruction)`, `$(StockBattle)`, `$(StockTeamBattle)`) defined in `Content\GameTypes.h` (location open — not inside the cabinet), and a help text path.
- Companion files: `des.txt`/`tdes.txt`/`bat.txt`/`tbat.txt` (plain CRLF help text with a title and `---` underline), `missionplay.txt` (generic multiplayer play text), and `<Name>.tga` (37,676-byte mission thumbnails). Mission logic itself lives in `MISSIONS/<name>.mw4` archives.

## Mission archives and ABL scripts

Mission logic ships as `MISSIONS/<name>.mw4` archives (a `#VBD` container, for example Mercenaries `alpine.mw4`, 245,565 bytes, 21 entries) holding the mission's world:

- `missions\<name>\<name>.contents` — the mission object tree (182,542 bytes for Alpine).
- `<name>{NameTable}` — the mission string pool (`dropteam1`, `playerstart`, `SecondaryLanceStart`, …): `[u32 length][name]` entries.
- Day/night lightmaps: `<name>.lights` and `<name>night.lights` (131,356 bytes each) — a single `#RLM` light-definition header (sun color ≈ `(0.73, 0.76, 0.89)` with intensity, two 4-float direction sets, the name `lookuplight`, map extents `2560`/`1280`, a `4, 4` grid field, and `65`) followed by 131,352 bytes of lightmap samples (sample layout open).
- `DropZones\*.contents` — spawn points as transforms (quaternion + position floats; `playerstart.contents` = 916 bytes).
- `<name>.bounds` / `mission.bounds` (36-byte boxes), `<name>graph.data` (905,860 bytes — graph/nav data), `<name>.data{GameModel}` (mission settings), and `scripts\<Name>.abl` + `scripts\player.abl`.
- Skies referenced from shared archives (for example `skies\thark04_sky\*.erf`).
- **Object placement records** (mission `.contents`) — decoded across Alpine (Mercenaries) and 0101 (Vengeance): a 2-byte lead, then **fixed-stride 100-byte records** (`u32@0 = 100` type tag). Each record = 28-byte header (`[100][2][2][0][archetype index][0/1024][link field]`), a **48-byte transform** (three orthonormal basis rows, yaw-only in samples, with the translation in the 4th column: `X@+0x3C`, `Y@+0x40`, `Z@+0x44` relative to the record), and a 24-byte tail. The **object's name is the `u16` NameTable entry index at `+92`** (verified on both titles: Alpine resolves 1,283 `tree_arctic_*` tree placements + 510 `Player`/`PlayerAI` spawn markers + dropzone entries with 1,804 unique indices total; Vengeance 0101 resolves its base layout — `alarm_tower_0004`, `vehicle_hangar`, `factory4_0002`, `base_bldg`, `fuel_tank`, `lunar_tower`, `Objective Tower 1…3`, `Listening Post` — with 90 unique indices 0–89). `u16@+86` is an asset/variant id in a wider space (e.g. 3,163+ for 0101's objects), and `u32@+24` points at other record data in the same file (link/child; schema open). The archetype index selects the object class (`469` = player/AI spawn records; `1169` = map objects — trees on Alpine, base buildings on lunar01; `1160` = 9 spawn definitions on 0101). **Vengeance mission trees are not contiguous**: the player record sits at the head and the object list follows after a ~20 KB gap, so the scan walks the file and skips gaps.
- **Cultural tree meshes** (`props.mw4` `culturals\`): single families ship a `<family>.erf` — three LOD records (24-vertex branch billboards + two coarser billboards) whose record names are the 32-bit cutout textures (`tree_arctic_snowy01_branch_mip2`, `tree_arctic_snowy01_mip1`; textures in `TEXTURES.MW4`, alpha-cutout billboards, record 0 = detail). The four `*_multi*` families ship no mesh of their own: their `.video` is a **cluster descriptor** — a 12-byte head then fixed 117-byte stubs (`#FRE` v14 + `#RLM` 18), each carrying a sub-mesh **archive entry ordinal** (`u16 @+0x33`) and a 48-byte local transform (`@+0x3D`). `tree_arctic_snowy_multi02` clusters five sub-trees (snowy03, snowy01, snowy01, snowy02, snowy01). Ordinals resolve within the containing archive (Vengeance and Mercenaries number their props archives independently).
- **Drop zone `.contents`** (`DropZones/*.contents`): `u32 count` then `count` × 48-byte transforms (no header). `playerstart.contents` = 19 spawn transforms; first at `(-1353.65, 79.73, 179.72)`.
- The `culturals` objects also ship `.instance` (100-byte record, identity transform), `.data`, `.data{Element}`, `.data{GameModel}` (composition list of same-archive ordinals plus ids `542`/`1169`), and `.obb` (collision). The `.video` stub's `u32 542` field is unexpanded. The map's `<map>culturals.mlr` layer, the `.contents` tail, and per-instance versus variant-slot semantics of the `+86` index remain open.

**ABL** is an FSM scripting language (Pascal-flavoured). Full working sample (`Alpine.abl`, 499 bytes):

```
fsm Alpine : integer;
     const
        #include_ <content\ABLScripts\mwconst.abi>
     type
        #include_ <content\ABLScripts\mwtype.abi>
function init;
    code
        SetupScoring_TeamDestruction;
        TeamSetNav(1, ena_Nav_Center);
endfunction;
state startState;
    code
        revealnavpoint(ena_Nav_Center);
endstate;
state deadState;
    code
endstate;
endfsm.
```

- Structure: `fsm <name> : <type>;` header, `const`/`type` sections with `#include_ <path>` (angle-bracket include), `function init;` (runs at start), and `state <name>; code … endstate;` blocks; `endfsm.` terminator. Statements are calls to engine functions (`SetupScoring_TeamDestruction`, `TeamSetNav(team, nav)`, `revealnavpoint`) using `ena_*` enum constants defined in `mwconst.abi` / `mwtype.abi` (22,607 / 352 bytes in `CORE.MW4`).
- The same language drives AI (`ai\*.abl` in `CORE.MW4`: `DodgeBot`, `KillerBot`, `RookieSniperBot`, …).

## Open items

- Complete grammar inventory of `.script` statements and the execution/scheduling model; event handlers beyond `GUI_CREATE`/`EXECUTE`/`DRAW`.
- The integer id spaces in `ScriptStrings.h` and where the localized text lives per release (Windows resources).
- `.animscript` transition table syntax and curve enum semantics; how states bind to `.mw4anim` clips and to the locomotion controller.
- Full ABL grammar and the engine-function catalogue (scoring, nav, objectives beyond the sampled calls); `mwconst.abi`/`mwtype.abi` enum inventory.
- Mission archive internals: the `.contents` record tail semantics (link/flag fields, the 1,540-byte PlayerAI tail), instance-name resolution robustness for every record, where the archetype/cultural **meshes** live (asset ids 542/1169, the `.mlr` layer, or a full-install archive), `graph.data`, the `.lights` sample layout (sample count/grid, day/night deltas), and bounds boxing. The zone-ERF object records (`+12 = 116`) are a separate small format (four-vertex quads with flags) and are not yet placed.
