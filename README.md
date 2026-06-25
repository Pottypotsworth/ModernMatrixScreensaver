# Modern Matrix

A native, Apple-Silicon-only recreation of XScreenSaver's **GLMatrix** — the 3D
"digital rain" from the title sequence of *The Matrix* — rebuilt from scratch in
Swift + Metal for macOS 26 (Tahoe).

It exists because the original GLMatrix, while still the best-looking version, is
an x86-64 binary running under Rosetta inside Apple's screensaver host. On a
multi-display Apple-Silicon Mac that combination produces the "boxed across two
screens" rendering, runaway memory, and general flakiness. This version is native
`arm64`, leak-free, and written to behave well despite the known bugs in Apple's
`legacyScreenSaver` host (which are real on Sonoma, Sequoia, and Tahoe).

## What you get

- **`Modern Matrix.saver`** — the installable screensaver (System Settings ▸ Screen Saver).
- **`Modern Matrix.app`** — the companion app: live preview + settings editor (and an offscreen `--snapshot` mode). It's how you configure the saver (see Known limitations).

Both are built from one shared core — a portable C engine (`core/mmcore.c`) wrapped in a
Swift/Metal layer — by a single script, no `.xcodeproj`. That same C engine compiles into
the planned Windows `.scr`, so the rain's behaviour is defined once for both platforms.

## Features

All of the original GLMatrix controls, plus modern additions:

| Control | Notes |
| --- | --- |
| Glyph density | 60–900 falling columns |
| Glyph speed | fall rate + glyph mutation rate |
| Matrix encoding | Matrix · Binary · Hexadecimal · Decimal · DNA · Unicode katakana |
| Fog | depth-based fade |
| Waves | travelling brightness wave along the columns |
| Panning | slow camera drift between framed views |
| Textured / Wireframe | glyph texture vs. cell outlines |
| Show frame rate | FPS overlay |
| **Bloom glow** *(new)* | HDR bright-pass + separable Gaussian glow |
| **HDR highlights** *(new)* | `rgba16Float` + EDR so leaders pop on capable displays |

Glyphs are generated at runtime with Core Text (half-width katakana + digits,
mirrored like the film), so they stay crisp at any resolution.

## Build & install

```sh
./build.sh            # build harness + saver (debug)
./build.sh run        # build + launch the dev harness window
./build.sh snapshot   # render one frame to build/snapshot.png
./build.sh install    # release-build + install BOTH the saver and the app
./build.sh clean
```

After `install`:
- The screensaver appears in **System Settings ▸ Screen Saver ▸ Other ▸ Modern Matrix**.
- **Configure it with `Modern Matrix.app`** (installed to /Applications): a live preview
  beside the settings form. macOS 26's System Settings does **not** open a third-party
  saver's **Options…** sheet, so the app is the way to change settings — edits write to
  the saver's own settings domain and are picked up the next time the saver runs.

To uninstall: `rm -rf ~/Library/Screen\ Savers/Modern Matrix.saver "/Applications/Modern Matrix.app"`

### Snapshot flags (handy for previewing settings)

```sh
BIN="build/Modern Matrix.app/Contents/MacOS/MatrixRainHarness"
"$BIN" --snapshot out.png --encoding 5 --density 0.85 --panning --warmup 140
# flags: --encoding 0..5  --density 0..1  --speed 0..1
#        --wireframe --no-bloom --no-fog --no-textured --no-waves --panning --warmup N
```

## Building the Windows version (.scr)

Most of the Windows port is already written: the simulation, settings model, and glyph
encodings live in **portable C at [`core/mmcore.c`](core/mmcore.c)**. To build the
screensaver, read **[`PORTING.md`](PORTING.md)** — a complete, self-contained blueprint
(start at §0) — plus [`core/README.md`](core/README.md). You **link `core/mmcore.c`** for the
rain itself and implement only the Windows-specific parts: a Direct3D 11 renderer, a
DirectWrite glyph atlas, a Win32 config dialog, and the `.scr` host shell (in a `windows/`
folder). Requires Visual Studio 2022 with the "Desktop development with C++" workload.

## Known limitations (macOS 26)

These are **macOS platform limitations, not bugs in Modern Matrix** — they affect
every third-party screensaver, and both are confirmed by Apple's own engineers:

- **The catalogue thumbnail is a generic "blue spiral galaxy," not the rain.** In
  System Settings ▸ Screen Saver, the little grid icon for *any* code-based third-party
  saver on macOS 26 is a fixed placeholder. Apple removed custom thumbnails for `.saver`
  plug-ins when they rebuilt Screen Saver settings; an Apple engineer states plainly that
  *"there's no supported way to replace the default thumbnail."* (Savers that still show a
  real thumbnail were cached from **before** macOS 26 — the capability is gone for anything
  new.) Everything *else* shows the real rain: the **live preview** when Modern Matrix is
  selected, the **companion app**, and the **actual running screensaver**. Only that one
  tiny catalogue icon is the placeholder.
  Ref: [Apple Developer Forums thread 806641](https://developer.apple.com/forums/thread/806641).

- **The "Options…" button in System Settings does nothing.** macOS 26's System Settings
  doesn't present a third-party saver's configuration sheet at all. **Configure Modern
  Matrix from the companion `Modern Matrix.app`** instead — your changes are written to the
  saver's settings and picked up the next time it runs.

## Requirements

- Apple Silicon Mac, macOS 26+
- Xcode 26 toolchain (Swift + the Metal toolchain component for `metallib`)

## Layout

```
core/              shared C engine (mmcore) — linked into macOS AND the Windows .scr
  mmcore.h / mmcore.c    portable C99: simulation, settings + derived, encodings, constants
Sources/Core/      macOS Swift/Metal layer over the C engine
  Settings.swift         Swift model (UI + JSON persistence) + bridge to mmcore's MMSettings
  GlyphAtlas.swift       Core Text glyph atlas (the code points come from mmcore)
  Renderer.swift         Metal pipeline, camera/panning, bloom, HDR; drives the mmcore sim
  ShaderTypes.swift      Uniforms layout (the glyph-instance struct lives in mmcore)
  MathUtil.swift         simd matrix helpers
  SettingsUI.swift       shared SwiftUI settings form (used by app + saver)
  RainView.swift         shared live Metal preview (companion app + Options sheet)
  DisplayLinkDriver.swift, FPSOverlay.swift
Sources/Saver/     ScreenSaverView subclass + configure-sheet wrapper
Sources/Harness/   companion app — live preview + settings, --snapshot mode
Resources/Shaders.metal
build.sh             compiles mmcore (clang) + Swift (swiftc) + Metal, assembles the bundles
```

## Notes on the screensaver host

Rendering is driven by the host timer (`animateOneFrame`) as a baseline, with a
view-bound `CADisplayLink` as a smoothness optimisation gated by a watchdog — the
host's display link frequently never fires (verified on macOS 26), so the timer is
what guarantees frames. The view tears down in both `stopAnimation` and
`viewDidMoveToWindow(nil)` (since `stopAnimation` is no longer reliably called),
keeps all state per-instance (no globals that leak across the host's repeated
instantiations), and lowers density automatically for the small System Settings preview.
