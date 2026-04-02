# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VPAW (Virtual Pediatric Airways Workbench) is a 3D Slicer Custom Application for surgical planning of pediatric airway obstruction. It is built on top of 3D Slicer using the SlicerCustomAppTemplate pattern — the top-level `CMakeLists.txt` fetches Slicer sources via FetchContent and adds this project's custom modules and application shell on top.

See `/home/ebrahim/Slicer/CLAUDE.md` for general Slicer architecture, build patterns, and conventions that also apply here.

## Build

VPAW uses the same CMake SuperBuild as Slicer. The local superbuild is at `~/slicer-superbuild-v5.10/`, with the inner build at `~/slicer-superbuild-v5.10/Slicer-build/`.

```bash
# Full rebuild (rarely needed)
cd ~/slicer-superbuild-v5.10
cmake --build . -j$(nproc)

# Inner rebuild (for iterating on C++ or CMake changes)
cd ~/slicer-superbuild-v5.10/Slicer-build
cmake --build . -j$(nproc)
```

All custom modules are pure Python (Scripted), so most development requires no C++ compilation — just restart the application.

## Testing

```bash
cd ~/slicer-superbuild-v5.10/Slicer-build
ctest -R VPAWModel              # run a specific module's test by name regex
ctest -R VPAW                   # run all VPAW tests
ctest -j$(nproc)                # run all tests
```

Tests are defined in each module's `Testing/` directory and registered via `slicer_add_python_unittest()` in CMakeLists.txt.

## Linting

Pre-commit hooks are configured in `.pre-commit-config.yaml`:
- **Python**: ruff (config in `.ruff.toml`, targets Python 3.9, 120-char lines)
- **Markdown/YAML/HTML/CSS/JSON**: prettier (with `--prose-wrap=always`)
- Standard checks: large files, merge conflicts, trailing whitespace, debug statements

```bash
pre-commit run --all-files      # check everything
pre-commit run ruff --all-files # just Python linting
```

## Commit Message Convention

Same as Slicer — prefix every commit with: `BUG:`, `COMP:`, `DOC:`, `ENH:`, `PERF:`, `STYLE:`, or `WIP:`. Subject line in imperative mood, <72 chars.

## Architecture

### Application Shell

`Applications/vpawApp/` — C++ entry point and main window (`qvpawAppMainWindow`). Customizes the Slicer main window with VPAW branding and layout.

### Custom Modules

All custom modules are Scripted (pure Python) and live in `Modules/Scripted/`. They follow Slicer's standard ScriptedLoadableModule pattern with Module/Widget/Logic/Test classes in a single `.py` file, plus a `.ui` file for the GUI and a `CMakeLists.txt` for build integration.

- **Home** — Landing page module. Configures the default UI appearance (hides standard Slicer chrome, applies custom stylesheet). Navigation buttons to other VPAW modules.
- **VPAWModel** — Runs the `pediatric_airway_atlas` pipeline for CT airway data. Handles pip-installing Python dependencies into Slicer's bundled Python, linking the external `pediatric_airway_atlas` source tree, converting FCSV landmarks to P3 format, and running segmentation.
- **VPAWVisualize** — Visualization of CT airway pipeline results (cross-sections, isosurfaces). Has a helper library in `vpawvisualizelib/` with `isosurfaces.py`.
- **VPAWModelOCT** — Similar to VPAWModel but for OCT (Optical Coherence Tomography) data, linking the `OCTSeg` package.
- **VPAWVisualizeOCT** — Visualization of OCT pipeline results.

### Two Imaging Modalities

The modules are split into CT and OCT workflows:
- CT workflow: VPAWModel → VPAWVisualize
- OCT workflow: VPAWModelOCT → VPAWVisualizeOCT

### External Dependencies Fetched at Build Time

The top-level `CMakeLists.txt` fetches several Slicer extensions via FetchContent:
- **SlicerMorph** (ImageStacks module only)
- **SlicerSegmentEditorExtraEffects**
- **SlicerMarkupsToModel**
- **SlicerAirwaySegmentation**

### QSettings Persistence

Modules persist user-configured directory paths and options via Qt's `QSettings`, grouped by module name (e.g., `VPAWModel/PediatricAirwayAtlasDirectory`). The GUI ↔ QSettings synchronization uses a `_updatingGUIFromQSettings` flag to prevent infinite loops.
