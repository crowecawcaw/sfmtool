# Running sfmtool in the browser (interface + engine via WASM)

**Status:** Draft plan — research complete, nothing implemented. CUDA is
explicitly out of scope; everything below targets `wasm32-unknown-unknown`
with WebGPU.

Related: [`../gui/gui-architecture.md`](../gui/gui-architecture.md) (which
already lists "can compile to WASM for web deployment" as a motivation for
the egui + wgpu stack), [`../formats/`](../formats/) (the `.sfmr` / `.sift` /
`.matches` containers the browser must read).

## Goal

Run both halves of sfmtool in a browser tab:

1. **Interface** — the SfM Explorer viewer (`crates/sfm-explorer`), loading
   `.sfmr` files picked or dragged in by the user.
2. **Engine** — the reconstruction pipeline (extract → match → solve),
   compiled to WASM, no server round-trips required for the parts we control.

### Non-goals

- CUDA / native GPU acceleration in the browser (WebGPU only).
- Running the Python CLI layer itself in the browser (Pyodide). The Python
  layer is orchestration; the algorithms it orchestrates are Rust or COLMAP.
  Re-hosting CPython in the browser buys nothing, and the one Python
  dependency that matters (pycolmap) has no wheel that could run there.
- Safari support initially. Target browsers with shipped WebGPU
  (Chromium-based, Firefox ≥ 141).

## Where the code stands today

Findings from auditing the workspace and the Python/Rust boundary:

**The engine is split three ways.** Feature *matching* (sweep, polar, flow,
geometric filtering), optical flow, alignment, triangulation, transforms, and
all file formats are Rust (`sfmtool-core` + format crates) — portable.
Feature *extraction* runs through Python (OpenCV `SIFT_create` in
`src/sfmtool/sift/extract_opencv.py`, or COLMAP via pycolmap in
`extract_colmap.py`). The *solver* is entirely pycolmap:
`pycolmap.incremental_mapping()` in `src/sfmtool/_isfm.py` and
`pycolmap.global_mapping()` (GLOMAP) in `src/sfmtool/_gsfm.py`. Nothing in
this repo implements reconstruction; COLMAP does. No public COLMAP→WASM port
exists.

**The viewer stack is already web-capable upstream.** `sfm-explorer` is raw
winit 0.30 + wgpu 29 + egui 0.34 (custom event loop, not eframe). All three
have first-class web backends: winit renders to a canvas via its web platform
(`EventLoopExtWebSys::spawn_app`), wgpu has `webgpu` (and `webgl`) backends,
and egui runs on the web (egui.rs itself). What blocks compilation today is
mechanical, not architectural:

- `pollster::block_on()` for wgpu adapter/device setup
  (`sfm-explorer/src/lib.rs` and `sfmtool-core/src/optical_flow/gpu/mod.rs`) —
  there is no blocking on the web; init must become `async` and be driven by
  `wasm_bindgen_futures::spawn_local` (native keeps pollster).
- `rfd::FileDialog::pick_file()` (sync) — `rfd::AsyncFileDialog` **does**
  have a wasm backend and returns file *bytes*; alternatively egui's
  dropped-files input carries bytes on web. Either way the load path must
  accept bytes, not paths.
- wgpu features pinned to `dx12, vulkan` — gate those to native, enable
  `webgpu` for wasm.
- `env_logger`, the Windows DirectManipulation module, and `winres` — all
  trivially cfg-gated (DirectManipulation already is).

**Format crates are one seam away from portable.** `zip 2` + `zstd 0.13`;
`sfmr-format/src/archive_io.rs` is already generic over `R: Read + Seek`, but
the public entry points (`read_sfmr(path: &Path)` etc.) hardcode
`std::fs::File`. Adding `read_sfmr_from(reader: impl Read + Seek)` (and the
same for `.sift` / `.matches` / `.camrig`) lets the browser pass
`Cursor<Vec<u8>>`. `zstd-sys` (C) does compile for wasm32 with clang; if that
proves brittle in CI, `ruzstd` is a pure-Rust *decode-only* fallback — enough
for the viewer, not for writing archives.

**Two real compile blockers, both avoidable.**

- `sfmr-colmap` uses `rusqlite` (bundled SQLite C) — exclude the crate from
  wasm builds. COLMAP *database* interop is meaningless in a browser anyway;
  COLMAP *binary* read/write lives outside this crate's SQLite path.
- `rayon` is pervasive in `sfmtool-core` (~11 modules). This does **not**
  block compilation: rayon ≥ 1.7 falls back to sequential execution on wasm
  targets without atomics. Real parallelism later means
  `wasm-bindgen-rayon` + SharedArrayBuffer, which requires cross-origin
  isolation headers (COOP/COEP) and a nightly-built std — a deliberate
  opt-in, not a prerequisite.

There is currently no `cfg(target_arch = "wasm32")` anywhere in the
workspace.

## Phase 1 — SfM Explorer in the browser

Highest value for least work; everything needed exists upstream.

1. **I/O seam.** Add reader-based entry points to `sfmr-format` and
   `sift-format`; path-based functions delegate to them. Thread the same
   change through `SfmrReconstruction::load` in the explorer's state.
2. **Async GPU init.** Refactor surface/adapter/device setup in
   `sfm-explorer/src/lib.rs` into an `async fn`; native wraps it in
   pollster, web drives it with `spawn_local`. Use winit's
   `spawn_app` on web and attach to a host canvas via
   `WindowAttributesExtWebSys`.
3. **File open.** Replace the sync `rfd` call with `rfd::AsyncFileDialog`
   (works native *and* web) delivering bytes; also accept egui drag-and-drop
   (`DroppedFile::bytes`, populated on web). Loading stays synchronous once
   bytes are in hand — `.sfmr` reads fully into memory already.
4. **Backends and logging.** Feature-split wgpu (`dx12, vulkan` native /
   `webgpu` wasm); `console_log` + `console_error_panic_hook` on web.
   WebGPU only at first — the renderer's pick-buffer and depth readbacks map
   to `mapAsync`, but a WebGL2 fallback would need real rework; skip it.
5. **Build + serve.** `trunk` (or `wasm-pack` + a static `index.html`) with a
   `pixi run gui-web` task. Note: rayon's sequential fallback means the
   point-size KD-tree pass etc. just run single-threaded — fine for a viewer.
6. **Deploy.** A static site; could hang off the existing docs deployment.
   (GitHub Pages cannot set COOP/COEP headers — irrelevant in Phase 1, which
   needs no threads; revisit in Phase 2.)

Deliverable: open a URL, drag in a `.sfmr`, get the full viewer.

## Phase 2 — Engine in the browser: extract + match

1. **`sfmtool-wasm` crate.** A `wasm-bindgen` sibling of `sfmtool-py`
   exporting the same algorithm surface minus COLMAP-DB: format read/write on
   byte buffers, pair matching (`match_image_pair` / batch), optical flow,
   alignment, triangulation, covisibility/frustum pair building. The PyO3
   module already proves these APIs are cleanly callable from outside Rust.
2. **SIFT extraction in Rust.** The missing engine piece that is *not*
   COLMAP. Adopt the `sift-features` crate (pure Rust, explicitly
   OpenCV-compatible output — which matters, because `.sift` files and the
   matching thresholds assume OpenCV/COLMAP SIFT) as a third `feature_tool`
   backend alongside `opencv` and `colmap`. Validate parity on the
   `seoul_bull_sculpture` dataset **natively first** (keypoint/descriptor
   agreement, then end-to-end solve quality), so the browser introduces no
   new algorithm variable. `sift-wgpu` (CPU + WebGPU backends, wasm-ready)
   is a candidate for a GPU fast path later.
3. **Orchestration.** The thin pipeline logic (which pairs to match, file
   naming, workspace layout) lives in Python and won't run in the browser.
   Reimplement the minimal subset in the web app (TypeScript), or — the more
   strategic move — push it down into a Rust `sfmtool-pipeline` module that
   the CLI, PyO3, and wasm builds all share. Start with the former; the
   latter is a refactor to schedule on its own merits.
4. **Storage.** Map the workspace-directory concept onto OPFS (origin-private
   file system): images in, `.sift`/`.matches` cached alongside, results
   downloadable. Keeps the "workspace" mental model intact.
5. **Threads (opt-in).** Exhaustive matching is embarrassingly parallel and
   single-threaded WASM will hurt on the 85-image dataset. Enable
   `wasm-bindgen-rayon` behind a feature: COOP/COEP headers (or the
   `coi-serviceworker` shim where headers can't be set), atomics-enabled std.
   Ship single-threaded first; measure; then turn this on.

Deliverable: drop images in the tab, get `.sift` + `.matches` out — same
bytes the native pipeline would produce.

## Phase 3 — Solve in the browser (the hard part)

The solver is COLMAP/GLOMAP C++ behind pycolmap. Three options:

- **A. Server-side solve.** Browser extracts/matches/views; a small service
  runs `sfm solve` on uploaded matches. Pragmatic escape hatch, ships
  immediately after Phase 2, but it isn't "engine in browser".
- **B. Emscripten-compile COLMAP.** Ceres, Eigen, and SQLite individually
  compile under Emscripten, but no one has shipped COLMAP this way: the
  FreeImage/glog/boost dependency chain, pthread requirements (COOP/COEP),
  the wasm32 4 GB memory ceiling, and a binary likely in the tens of MB.
  High risk, low control, and we'd own a gnarly fork. Not recommended.
- **C. Rust-native solver in `sfmtool-core`.** The repo already has
  triangulation with observability diagnostics, epipolar geometry, RANSAC
  infrastructure, and photometric refinement. The genuinely missing piece is
  sparse bundle adjustment (Levenberg–Marquardt with a Schur-complement
  solve) plus incremental bookkeeping (two-view init, PnP registration
  loop). This is months of work, but it is the only option that *also* pays
  off outside the browser: it removes the pycolmap dependency from the
  native pipeline entirely and gives the project its own solver. It aligns
  with the direction the Rust core has been growing in.

**Recommendation:** ship Phases 1–2 with solve out of the browser (optionally
via A), and treat C as the long-term engine goal. Don't pursue B.

## Cross-cutting

- **CI.** Add a `cargo check --target wasm32-unknown-unknown` job (excluding
  `sfmr-colmap`, `sfmtool-py`) as soon as Phase 1 lands, so the wasm build
  can't rot.
- **Binary size.** zstd, the `image` codec set, egui, and wgpu add up; audit
  `image` default features (the viewer needs JPEG/PNG only) and build with
  `opt-level = "z"` + `wasm-opt` for the web profile.
- **Memory.** wasm32 caps at 4 GB. The checked-in datasets are comfortable;
  full-res multi-hundred-image workspaces are not a Phase 2 target.
- **Specs.** When Phase 1 starts, split a concrete `gui-web` spec out of this
  draft and update `gui-architecture.md` (event-loop and file-dialog sections
  change).

## Suggested first milestone

Phase 1, scoped to: `pixi run gui-web` serving the explorer locally via
trunk, WebGPU only, `.sfmr` load via file picker and drag-and-drop. It
touches the fewest crates (`sfmr-format`, `sift-format`, `sfm-explorer`),
needs no threads, no SQLite decisions, no new algorithms — and it proves the
whole toolchain (winit/wgpu/egui on canvas, zip+zstd in wasm) before any
engine work begins.

## Open questions

- Does `zstd-sys` build cleanly for wasm32 in our CI image, or do we standardize
  on `ruzstd` for reads and keep writes native-only until needed?
- Where does the web app live — `crates/sfm-explorer` behind cfg, or a thin
  `crates/sfm-explorer-web` wrapper crate? (Leaning wrapper: keeps the
  Windows-specific code and `winres` build script out of the wasm graph.)
- Is OPFS adoption worth it in Phase 2, or is in-memory + explicit
  download/upload enough for the first cut?
