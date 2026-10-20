# stp2glb-lite

## Quick start

### Option A: Drag and drop (recommended)

Drop `.stp` / `.step` files straight onto **`drag-and-drop.bat`**. Results appear in a `glb_output`
folder next to the source file. You can drop several files at once.

### Option B: Command line

```bat
stp2glb-lite.exe  input.stp                         :: writes to glb_output next to the source
stp2glb-lite.exe  input.stp  output_dir             :: explicit output directory
stp2glb-lite.exe  input.stp  output_dir  --fast     :: fast mode, coarser tolerance, quicker
```

### Option C: Double-click

Double-click `stp2glb-lite.exe` with no arguments to enter **interactive mode**. Paste or drag in a
file path when prompted. The window stays open, so it will not flash and disappear.

---

## Command-line options

```
stp2glb-lite <input.stp> [output_dir] [options]
```

| Option | Description |
|---|---|
| `--tol <number>` | Chordal tolerance (model units, absolute). Defaults to an adaptive value based on the bounding-box diagonal |
| `--tol-ratio <ratio>` | Chordal tolerance as a fraction of the bounding-box diagonal (default 0.0015) |
| `--ang-tol <degrees>` | Maximum angular step per arc segment (default 30). **A scale-independent safety net** that keeps small holes and small fillets from being drawn as polygons. It only triggers when a first pass at the chordal tolerance shows the edge was flattened into a polygon (turn angle between adjacent segments > 40°), so edges that are already round gain no extra points at all. Set 0 to disable |
| `--fast` | Fast mode: tolerance = 0.6% of the diagonal |
| `--precise` | Precise mode: tolerance = 0.04% of the diagonal |
| `--max-depth <n>` | Maximum adaptive subdivision depth (default 6). Use this first if you want a smaller file; going from depth 6 to 3 roughly cuts the triangle count to one third |
| `--flat` | Flatten everything into a single mesh and skip the assembly node tree. Only needed when a viewer does not honour node matrices |
| `--single-sided` | Drop the double-sided material so the viewer culls back faces. Double-sided is the default; see the next section for why |
| `--name <filename>` | Output file name (`.glb` is appended automatically) |
| `--workers <n>` | Number of parallel workers (default: CPU core count) |
| `--no-normals` | Do not write vertex normals |
| `--no-outline` | Do not write outline curves (B-Rep edges are written by default) |
| `--outline-angle <degrees>` | Draw only edges whose dihedral angle is at least this value. Default 0 = every B-Rep edge; 20–30 filters out the "soft" edges where a fillet meets a tangent plane, for a cleaner picture |
| `--log-dir <dir>` | Log directory (default: `logs` next to the executable) |
| `--history [n]` | Show the last n conversions (default 20) |
| `--inspect` | Structural diagnostics only, no conversion |
| `--diag` | Console / encoding self-check (use this when text renders incorrectly) |
| `--quiet` / `--verbose` | Terse output / debug output |
| `-h` / `-v` | Help / version |

**Choosing a precision level:**

- Large assemblies (tens of MB and up): the default is fine. Add `--fast` if you want it quicker.
- Small parts where fillets and curved detail matter: add `--precise`.
- Tolerance is adaptive. A single global value computed from a large model is never forced onto small
  parts — every face tightens the tolerance to its own scale.

Exit codes: `0` success, `1` bad arguments or environment, `2` problem with the input file,
`3` parse or tessellation failure, `4` write failure.

---

## What the GLB contains

**1. Assembly tree + instancing**
The node hierarchy comes from the STEP assembly relations (`NEXT_ASSEMBLY_USAGE_OCCURRENCE`) and
matches the tree you see in a STEP viewer. Node names are copied verbatim from the STEP product names.
A part used N times stores its geometry once and is placed N times via node matrices, so the file is
far smaller than a flattened version while the expanded triangle count and surface area are identical.

> This is the **assembly structure tree**, not CATIA's **feature tree** (sketch / pad / pocket / hole).
> STEP records only the final B-Rep; the feature tree does not physically exist in the file.

**2. Outline curves (B-Rep edges)**
STEP `EDGE_CURVE` entities are discretised into polylines and attached to the same mesh as a separate
`mode=1 (LINES)` primitive, so node matrices, instancing and the assembly tree all apply as usual.
Edges are deliberately **not** taken from the triangle mesh — that would draw a dense triangle wireframe,
not the edges you see in CAD. Use `--no-outline` to drop them, or `--outline-angle 20` to remove the
soft edges along fillets.

**3. Double-sided material**
Surface faces default to `doubleSided: true`. A large share of automotive STEP data consists of
**surface models with no thickness** (`OPEN_SHELL` / `SHELL_BASED_SURFACE_MODEL`), while the glTF
specification defaults materials to single-sided, so browser viewers cull every face facing away from
the camera — which looks exactly like "solid turned hollow, only the edges remain". Desktop viewers such
as CATIA always render both sides, so the same file can look fine on the desktop and hollow in a browser.
If you have confirmed the model is entirely closed solids and want maximum performance, use
`--single-sided` to remove the flag.

**4. Units and coordinates**
Millimetres and Z-up are inherited from STEP. No unit conversion and no axis transform are applied.

---

## Logs and history

- Log directory: `logs\` next to the executable, one file per day (e.g. `2026-10-20.log`).
- History: `logs\history.jsonl`, one JSON object per line, recording input/output paths, size, triangle
  count, elapsed time and status.
- Show history: `stp2glb-lite.exe --history 20`

Results are written to a temporary file first and then renamed atomically, so a failure midway never
leaves a half-written file behind.

---

## Performance reference

Measured on this machine (Windows x64), default precision, outline curves enabled:

| File | Source size | Output GLB | Time | Triangles |
|---|---|---|---|---|
| `CAD1784883508577.stp` body-in-white assembly | 72.0 MB | 8.5 MB | ~7 s | 254,962 |
| `GUARD_RESULT_MMA_HB51_070.stp` welding fixture assembly | 128.7 MB | 14.2 MB | 11.1 s | 399,598 |
| `GUARD_RESULT(1).stp` welding fixture assembly | 1.43 GB | 65.3 MB | ~7 min | 2,008,343 |
| `as1-oc-214.stp` (official CAx-IF sample) | 434 KB | 44.7 KB | 0.1 s | 930 |

Output is typically 5%–25% of the source size, depending on how densely the surface control points are
packed. On large files most of the time goes into parsing and face extraction; tessellation itself is a
small fraction.

---

## Known limitations

- Supports **boundary representation (B-Rep)** geometry in STEP AP203 / AP214 / AP242. Pure wireframe
  models, `GEOMETRIC_CURVE_SET` and similar data with no faces produce the message "no faces available
  for tessellation".
- **Colours / materials / layers are not preserved.** Only geometry, normals and outline curves are written.
- Only geometry that appears **in the assembly tree** is written. "Orphan copies" that are geometrically
  identical to an assembly component but take part in no assembly relation are dropped as redundant (the
  log prints `redundant representations : dropped N ...`). This is intentional — such copies would stamp
  an extra copy of the model at the part's original coordinates.
- **Sibling geometric representations of the same part are all kept by default.** When the log shows
  `sibling representations : kept N ...`, those representations could not be proven duplicates, so they
  are kept on the principle of "rather one extra layer of coincident faces than lost detail", at the cost
  of a slightly larger GLB. For aggressive de-duplication, set the environment variable
  `STP2GLB_DEDUP_ALT=1`.
- A small number of faces fall back to **planar approximation** (about 0.24% in a large real-world
  regression run) and are therefore slightly off in shape. One such class is **multi-turn helical
  surfaces** (for example thread flanks, where the v parameter is arc length and one face wraps several
  turns): the parameter-space inverse mapping lands on the wrong turn and the local iteration cannot
  recover.
- Zero-area degenerate faces (`SKIP: zero-area degenerate face`) are skipped and not counted as failures.
- **"Looser tolerance, denser mesh" is a real anomaly.** In assemblies with a very large number of faces
  the effective tolerance of each individual face is pinned, so raising the global tolerance has no effect
  on that set of small faces and can push some faces from a structured grid back to ear clipping plus
  subdivision. An explicit `--fast` may therefore increase the file size. When no precision option is
  given explicitly, a built-in size-ratio guard adjusts subdivision depth automatically to keep things in
  check. Once `--tol/--fast/--precise` is given explicitly, the guard stays out of the way by design — use
  `--max-depth` to control density in that case.
- **Self-intersecting boundary loops** (bow-tie boundaries caused by inconsistent sampling) cannot be
  correctly triangulated by ear clipping in principle, so the algorithm forces progress and may produce
  locally overlapping triangles. The conversion log prints
  `tessellation precision : N faces with holes ...`, which is 0 on the standard samples and common large
  files.
- **Parameter-domain degenerate holes** (inner loops whose projected area collapses to zero) are skipped,
  deliberately: falling back to planar approximation for every one of them would sharply increase the
  number of approximated faces. If every hole on a face is degenerate, that face drops to the planar
  path (which uses 3D loops and is unaffected).
- `DEGENERATE_TOROIDAL_SURFACE` is not supported (the surface itself is degenerate).
- A statically linked build involves no injection, but Windows SmartScreen may warn about an unknown
  publisher the first time you run it. Choose "Run anyway".

---

## Folder layout

```
stp2glb-lite\
├─ stp2glb-lite.exe    main program (4.0 MB, no install, no runtime, copy and run)
├─ drag-and-drop.bat   drop .stp / .step files onto it
├─ README.md
├─ logs\               created at runtime: daily logs + history.jsonl
└─ src\                complete Go source
   ├─ main.go
   ├─ internal\        step kernel / glb writer / console / logging
   └─ tools\           batch regression, Khronos validation, rendering, per-face diagnostics
```

Building it yourself:

```bash
cd src
go build -o stp2glb-lite.exe .
```

Standard library only — no third-party dependencies, no CGO, no C compiler.

---

## Self-check

Convert any sample; the end of the console output should show `Faces : ok N / failed 0`, the size ratio
and the elapsed time:

```bat
stp2glb-lite.exe  your_file.stp  out
```

To look at a file's structure without converting it:

```bat
stp2glb-lite.exe  --inspect  your_file.stp
```

This prints the entity total, face / shell / assembly-node counts, the distribution of geometric entities
and the bounding box, and also flags reused assembly components, redundant copies and orphan shapes.

**If text renders incorrectly** (question marks, hollow boxes, a run of diamonds with dots inside):

```bat
stp2glb-lite.exe  --diag
```

It prints the detected console / code-page state and a few sample lines. Under normal conditions you
should see `[info] Input file : ...`.
