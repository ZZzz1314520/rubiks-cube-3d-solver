# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file HTML 3D Rubik's Cube solver. No build tools, no local dependencies — open `index.html` directly or serve via any static HTTP server.

## How to run

```bash
# In project root, either:
start index.html          # Windows — open directly in browser

# Or via Python:
python -m http.server 8765  # Then visit http://127.0.0.1:8765/index.html
```

There are no build, lint, or test commands — this is a zero-config static page.

## Architecture

Everything lives in `index.html` (~1487 lines). The structure:

**CSS (lines 8–355):** Dark-themed UI with CSS custom properties (`--bg`, `--panel`, `--accent`, etc.). CSS Grid layout: 3D stage on the left, control panel (340px) on the right. Responsive breakpoint at 840px stacks vertically. Includes loading spinner, step navigation, and keyboard-hint styles.

**HTML (lines 366–418):** Canvas for Three.js rendering + sidebar with stats, speed slider, status text, dynamic move list, undo button, and step navigation controls (⏮⏸⏭). Loading spinner overlay shown during boot.

**JavaScript `<script type="module">` (lines 421–1487):** The full app, organized as:

1. **UI refs** (`ui` object) — cached DOM element references including step nav and loader.
2. **Constants** — cube geometry (`CUBE_GAP`, `CUBELET_SIZE`), face colors, face-to-axis mappings (`FACE_DATA`), and move definitions.
3. **Boot sequence** (`boot()`) — checks CDN load, dynamic-imports Three.js from jsdelivr, shows loading spinner, initializes solver, validates move definitions, then hides spinner.
4. **3D scene** (`initScene()`) — PerspectiveCamera, WebGLRenderer with ACES tone mapping and PCF soft shadows, multi-light setup, ground shadow plane.
5. **Cube model** (`createCube()`) — 27 cubelets as Three.js Groups. `cubeModel` is the cubejs `Cube` instance for logical state.
6. **Interaction** — pointer events on canvas via Three.js Raycaster. Drag-to-turn with `inferMoveFromDrag()` and `fallbackFaceMove()`. Blocked during `isBusy` or `isSolving`.
7. **Layer rotation** (`rotateLayer()`) — pivots matching cubelets, animates with `easeInOutCubic()`, snaps positions to grid.
8. **Solver** (`solveCube()`) — calls `cubeModel.solve()`, verifies solution. `runSolve()` orchestrates playback. Supports pause/resume (`toggleSolvePause()`), step forward/back (`stepSolveNext()`/`stepSolvePrev()`), and jump-to-step via clicking move pills (`navigateToSolveStep()`).
9. **Undo** (`undoLastMove()`) — maintains `moveHistory[]` stack. `invertMove()` computes inverse ("R"↔"R'", "R2"↔"R2"). Ctrl+Z or ↩ button.
10. **Keyboard shortcuts** (`onKeyDown()`) — R/U/D/L/F/B for moves (Shift=prime, Alt=double), Space=scramble, Enter=solve, Backspace=reset, Ctrl+Z=undo, arrows=orbit.
11. **Validation** (`validateMoveDefinitions()`) — on boot, cross-checks visual move definitions against cubejs solver state.
12. **Scramble** (`scrambleCube()`) — 24 random moves, resets solve state and hides step nav.

## Key data structures

- **`FACE_DATA`**: Maps face letter → `{axis, axisIndex, layer, baseAngle}`. Critical: `baseAngle` defines the visual rotation direction. All move animations and validations derive from this.
- **`FACE_BY_LAYER`**: Maps `"axisIndex:layer"` → face letter. Inverse lookup for physical-to-logical conversion.
- **`cubelets[]`**: Array of Three.js Group objects. Each's `userData.coord` tracks its logical `{x, y, z}` position in the 3×3 grid.
- **cubejs `Cube`**: The source of truth for solve state. `cubeModel.asString()` returns the 54-character facelet string; `cubeModel.solve()` returns space-separated move strings.

## Dependencies (CDN, no install needed)

- **Three.js 0.160**: 3D rendering (dynamic `import()` from jsdelivr)
- **cubejs 1.3.2**: Cube state model + Kociemba solver (`<script>` tags with `onerror` fallback)

## Async/busy guard

Two flags control concurrency:
- **`isBusy`**: `true` during any animation (rotation). Blocks new `applyMove()` and user interactions.
- **`isSolving`**: `true` during the entire solve sequence lifecycle (including when paused). Blocks manual moves, undo, scramble, and reset. Clears on `finishSolve()` or when user exits solve mode.

`setControlsEnabled()` disables/enables buttons based on both flags. Solve-mode buttons (step nav) remain usable while `isSolving` is true but disable appropriately based on `solveStepIndex`.

## Key state variables

- **`moveHistory[]`**: Stack of user moves for undo. Not populated during solve playback.
- **`solveSequence[]`**: The current solution move array. `solveStepIndex` tracks position (-1 = start, n-1 = last step complete).
- **`solvePaused`**: `true` when auto-play is paused. User can manually step forward/back or click move pills to jump.
