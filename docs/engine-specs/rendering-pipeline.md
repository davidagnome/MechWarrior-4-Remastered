# Rendering pipeline (DirectDraw 7 / Direct3D 7)

Status: measured 2026-10-13 from an apitrace capture of Vengeance `1.06.11.0220`
(mission load and gameplay on `lunar01`, 800x600x16) plus the project's own
captures. Facts only; no listing content.

## API surface

- `DirectDrawCreateEx(IID_IDirectDraw7)`; the 3D device is created through
  `IDirect3D7::CreateDevice` as `IID_IDirect3DTnLHalDevice` (hardware) after an
  `IID_IDirect3DRGBDevice` probe. Presentation uses `IDirectDrawSurface7::Flip`
  on a 16-bit flip chain.
- **Depth buffer: 16-bit.** The attached Z surface at 800x600 reports
  `lPitch = 1600` (2 bytes/pixel). The product's qualified Windows dgVoodoo
  profile compensates (`DepthBuffersBitDepth = forcemin24bit`,
  `SmoothedDepthSampling = true`); renderers that honor D16 (for example
  CrossOver/wined3d) show distance depth-fighting, consistent with field
  reports.

## Geometry submission

- **Every draw submits pre-transformed, CPU-lit vertices**
  (`D3DFVF_XYZRHW | D3DFVF_DIFFUSE | D3DFVF_SPECULAR | D3DFVF_TEX1`, triangle
  lists / indexed triangle lists). The engine performs its own transform and
  lighting (it also ships a software rasterizer, `blade.dll`); Direct3D only
  rasterizes screen-space triangles.

## Texture stages

- Stage 0 only: `COLOROP = ALPHAOP = MODULATE`, `COLORARG1 = ALPHAARG1 =
  TEXTURE`, `COLORARG2 = ALPHAARG2 = DIFFUSE` — the fragment is
  `texel × vertex diffuse`; scene lighting lives in the vertex colors.
- Stages 1 and 2 are always `DISABLE` in the capture; **no hardware
  multi-texturing occurs anywhere**.

## Render states (observed set)

- Blending: only the pairs `{SRCALPHA, INVSRCALPHA}` (transparent overlays and
  cutouts) and `{ONE, ONE}` (additive effects); no multiply/dest-color blending
  is ever set.
- `ALPHATESTENABLE` for cutouts; point filtering (`MIN/MAG/MIPFILTER = 1`);
  `CULLMODE = none`; world pass `ZFUNC = LESSEQUAL` with `ZWRITEENABLE = 1`;
  2D overlays use `ZFUNC = ALWAYS` with `ZWRITEENABLE = 0`; `LIGHTING = 0`
  (no hardware lighting); fog disabled by default.

## Textures

- Uploads go through `Lock` / `memcpy` / `Unlock` with data identical to the
  decoded files: zone macro TGAs match byte-for-byte after surface-format
  conversion (correlation 1.0, per-pixel ratio 1.00; 32-bit BGRA and/or 16-bit
  565 depending on the surface), and the `_1_*` detail tiles match byte-exactly
  as 16-bit 565. Nothing is baked into the textures at load.
- Dominant upload shapes: 256x256x2 (131,072 bytes) and 256x256x4 (262,144);
  1024x1024 8-bit (1,048,576-byte) uploads occur at map load (light-map
  candidates; the on-disk form `textures\composttexture\lunar01_light_active.bid`
  is 1,559,064 bytes whose base is 16-bit, so the runtime 8-bit upload is a
  derived form).

## Consequences for the reimplementation

- The material "blend equation" is settled: there is no multi-texture or
  blend-based layer composition. Terrain presentation =
  per-zone macro texture × per-vertex lighting (light map sampled CPU-side)
  + per-cell detail tiles selected through the `.material`/FGD system
  + alpha-blended overlay passes (decal/`.bid` layers, roads, tarmac).
- The engine should therefore sample the 8-bit light map (and sun
  color/direction) into vertex diffuse colors and draw with stage-0
  `texel × diffuse`, matching the measured pipeline.

## Capture recipe (research tooling)

`apitrace 14.0` win32 build (`sha256 3f2548621ea25a714198ac367f429f6c6f095f04fbeb2a610e534131ed8febbe`),
its `ddraw.dll` wrapper placed in the game directory, launched under the
CrossOver bottle with `--dll "ddraw=n,b;d3dimm=b;d3d8=b;d3d9=b"` and
`apitrace trace -a d3d7 -o <trace> MW4.exe /gosnovideo`. Trace artifacts stay
untracked under `.local/`.
