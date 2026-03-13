# Model Rigger — 3D Rigging & Animation Tool

A browser-capable desktop application for rigging glTF models and authoring
animation clips. Runs in the browser for zero-install access and ships as a
compiled Tauri desktop app for modders who need filesystem integration.

---

## Purpose

Blender is the recommended rigging tool (see `04-asset-pipeline.md`), but it
has a steep learning curve for modders who only need to rig a simple creature or
character. The Model Rigger fills this gap: open a `.glb`, paint weights, author
animation clips, and export a rigged `.glb` — without installing Blender.

This tool dogfoods the same Tauri shell used by the Event Tree Editor, consistent
with the "dogfood everything" principle in `08-tooling.md`.

---

## Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Shell | Tauri 2.x (Rust) | Native filesystem, small binary (~5 MB), cross-platform |
| Frontend | SvelteKit + TypeScript | Lighter than React, excellent SSR/SPA hybrid, file-based routing |
| 3D engine | Babylon.js 7.x | Best-in-class browser rigging/skeletal API; GPU picking; WebGPU-ready |
| Styling | Tailwind CSS 4.x | Utility-first, fast iteration on editor chrome |
| Build | Vite (via SvelteKit) | Sub-second HMR in browser dev mode |
| Testing | Vitest + Playwright | Unit tests for state logic; E2E for critical editor flows |
| Packaging | Tauri bundler | `.deb`, `.rpm`, `.AppImage`, `.dmg`, `.msi` from a single `make package` |

**Why Babylon.js over Three.js:** Babylon has a first-class `Skeleton`,
`Bone`, and `VertexData` API designed for skinned meshes. Weight painting,
morph targets, and animation clip management are built in — not third-party
plugins. The `GLTFFileLoader` and `GLTF2Export` extensions handle round-trip
fidelity with `.glb` without custom serialisation.

**Why SvelteKit over React:** The Event Tree Editor (React) is a node-graph
application with complex drag-and-drop state. The rigger is a more traditional
tool application — panels, inspectors, timelines — where Svelte's reactive
stores and component model reduce boilerplate significantly without the overhead
of React's reconciler inside a Babylon.js render loop.

---

## Architecture

```
tauri-shell (Rust)
└── IPC commands: open_file, save_file, watch_directory, recent_files
    └── SvelteKit SPA (served from Tauri webview or browser)
        ├── /  — workspace landing (open file, recent files)
        ├── /editor  — main rigging workspace
        │   ├── BabylonViewport.svelte  — canvas, scene, camera
        │   ├── BoneTree.svelte         — skeleton hierarchy panel
        │   ├── WeightPainter.svelte    — weight painting mode overlay
        │   ├── AnimationTimeline.svelte — clip editor (bottom panel)
        │   └── Inspector.svelte        — bone properties, transform values
        └── /preview  — animation playback preview (shareable URL in browser mode)
```

### State model

All editor state lives in Svelte stores, not Babylon scene state. The Babylon
scene is the render target only — authoritative data flows from stores → scene,
never the other way.

```
stores/
  model.ts        — loaded GLTFFileLoader result, mesh list, material list
  skeleton.ts     — bone hierarchy, selected bone, bone transforms
  weights.ts      — per-vertex weight map (boneIndex[], weight[])
  animation.ts    — clip list, active clip, current frame, keyframes
  ui.ts           — active panel, active tool (select/rotate/paint), viewport mode
```

### Babylon ↔ Svelte bridge

A thin `scene-bridge.ts` module subscribes to Svelte stores and applies updates
to the live Babylon scene. This keeps reactive UI logic out of the render loop
and keeps Babylon objects from leaking into Svelte components.

---

## Feature Scope

### Core features (Phases 1–3)

- **Model import:** Open `.glb` or `.obj` file; renders mesh, materials,
  and existing skeleton if present. OBJ is mesh-only on import — the rig is
  added inside the tool. Export is always `.glb` regardless of import format.
  Note: `@babylonjs/loaders` does not include a polygon-mesh PLY loader (its
  SPLAT loader handles Gaussian-splat `.ply` only); PLY mesh import is deferred
  to a post-launch phase if there is modder demand.
- **Skeleton creation:** Add root bone; attach child bones; position via
  gizmo (translate, rotate)
- **Bone parenting:** Drag-and-drop reparenting in the bone tree panel
- **Pose mode vs. rest mode:** Toggle; rest pose edits bone transforms, pose
  mode is for animation
- **Weight painting:** Brush-based; per-vertex weight visualised as heatmap;
  supports add, subtract, smooth operations; normalise weights on stroke end
- **Animation clips:** Create/delete named clips; set frame range; insert
  keyframes (bone translate, rotate, scale); timeline scrub
- **Export:** Write rigged, animated `.glb` via `GLTF2Export`; preserves
  original materials and mesh data

### Post-launch features (Phase 4+)

- **Shape keys / morph targets:** Add blend shape to mesh; keyframe weight
  over time for facial animation
- **IK handles:** Simple two-bone IK chains for limbs; bake IK to FK on export
- **Animation retargeting:** Remap bone names from Mixamo or other rigs to
  a target skeleton layout
- **Pose library:** Save/load named rest poses; useful for building varied NPCs
  from one base mesh
- **Browser sharing:** Generate a shareable URL with the `.glb` embedded
  (base64 or presigned blob URL) for preview-only links

---

## Phased Implementation Plan

### Phase 1 — Project Foundation

Stand up the complete project skeleton so every subsequent phase has a working
dev loop and known build targets.

**Steps:**

1. Scaffold SvelteKit project with TypeScript and Tailwind CSS
2. Add Babylon.js as a dependency; create `BabylonViewport.svelte` that renders
   a basic scene (ground plane, directional light, arc-rotate camera)
3. Wrap the SvelteKit app in a Tauri 2.x shell; verify hot-reload in browser
   mode and Tauri dev mode both work
4. Write `Makefile` with all targets documented (see Makefile section below)
5. Add Vitest for unit tests; add Playwright for E2E tests with a smoke test
   that verifies the Babylon canvas mounts
6. Set up `package.json` scripts that the Makefile delegates to
7. Establish directory layout and lint/format config (ESLint + Prettier,
   enforced via pre-commit hook)

**Deliverable:** `make dev` opens the app in browser with a visible 3D viewport.
`make test` passes. `make build` produces a deployable SPA. `make build-app`
produces a native binary.

---

### Phase 2 — Model Loading & Scene Setup

**Steps:**

1. Implement `model.ts` store with open/close model lifecycle
2. Wire `BabylonViewport` to load `.glb`, `.obj`, and `.ply` files via
   `SceneLoader.ImportMeshAsync` — all three are handled by `@babylonjs/loaders`
   without additional dependencies. Browser uses `File` input; Tauri uses
   native `open_file` IPC with all three extensions in the filter.
3. Display mesh, materials, and original armature (if present) in viewport
4. Add `BoneTree.svelte` panel: renders imported skeleton hierarchy using Svelte
   recursive component; clicking a bone selects it and highlights it in viewport
5. Add `Inspector.svelte` panel: shows selected bone's name, local transform
   (position/rotation/scale) with editable number inputs
6. Implement gizmo overlay: translate and rotate gizmos on selected bone using
   Babylon's `PositionGizmo` and `RotationGizmo`
7. Implement Tauri `save_file` IPC command; wire a save button that exports the
   current scene back to `.glb` using `GLTF2Export`

**Deliverable:** Open a `.glb`, inspect and rename bones, move bones with
gizmos, save the modified file.

---

### Phase 3 — Skeleton Authoring & Weight Painting

**Steps:**

1. **Skeleton authoring:** Add bone creation (click-to-place root; click mesh
   surface to place child bone at intersection point); bone deletion; bone
   reparenting via bone tree drag-and-drop
2. **Rest vs. pose mode toggle:** Toolbar button; rest mode edits `Bone.getLocalMatrix()`,
   pose mode is read-only during skeleton authoring
3. **`weights.ts` store:** Stores a `Float32Array` of weights per vertex per
   bone (matches glTF `WEIGHTS_0`/`JOINTS_0` layout); initialised to zero when
   a new skeleton is created on an unrigged mesh
4. **Weight painting mode:** Activate via toolbar; raycasting picks triangles
   under the mouse; Gaussian brush accumulates weight on surrounding vertices
   within brush radius; weight normalisation runs after each stroke
5. **Heatmap visualisation:** A custom Babylon material reads the weight texture
   and renders a heatmap overlay on the mesh while painting; toggled off in
   normal view
6. **Weight transfer:** If a mesh has an existing skeleton, transfer weights
   to the new skeleton via nearest-bone proximity (basic auto-skinning fallback)
7. **Vertex weight panel:** Click a vertex to see its full bone→weight list in
   the Inspector; manually edit individual weights

**Deliverable:** Create a skeleton from scratch on an unrigged mesh; paint
weights; inspect per-vertex data; export the skinned mesh.

---

### Phase 4 — Animation Timeline

**Steps:**

1. **`animation.ts` store:** Clip list (name, frameStart, frameEnd); active
   clip; current frame; keyframe map `{clipId → {frame → {boneId → Transform}}}`
2. **`AnimationTimeline.svelte`:** Bottom panel; horizontal track per bone that
   has keyframes; scrubber; frame counter input; play/pause/stop buttons;
   loop toggle
3. **Keyframe insertion:** Press `I` with a bone selected to insert a keyframe
   at the current frame capturing the bone's current transform
4. **Playback engine:** A Babylon `Observable` on `scene.onBeforeRenderObservable`
   advances the frame counter at 24 fps; interpolates bone transforms between
   keyframes (linear, then Bézier as stretch goal)
5. **Clip management:** New clip dialog (name, frame range); duplicate clip;
   delete clip; rename clip
6. **Multi-bone selection:** Shift-click bones in the tree to select multiple;
   batch-insert keyframes; batch transform (useful for posing)
7. **Clip preview export:** Export a single animation clip (or all clips) as
   part of the `.glb`; clips appear as named animations in Godot's
   `AnimationPlayer`

**Deliverable:** Author a walk cycle or simple NPC idle animation; export the
`.glb`; import into Godot and verify `AnimationPlayer` picks up the named clips.

---

### Phase 5 — Polish, Distribution & Tauri Integration

**Steps:**

1. **Recent files list:** Tauri backend writes a `recent.yaml` file to the
   app's data directory; landing page renders it with thumbnails (first frame
   of the model rendered to a canvas, saved as a data URL)
2. **File watching:** Tauri `watch_directory` command; if the open `.glb` is
   modified on disk externally, prompt "Reload from disk?" (matches Event Tree
   Editor behaviour)
3. **Keyboard shortcuts:** Full shortcut map documented in UI; configurable via
   a settings panel (stored in `settings.yaml`)
4. **Undo/redo:** Command pattern — every user action is a reversible command
   object pushed onto a stack; `Ctrl+Z` / `Ctrl+Y`
5. **Auto-updater:** Tauri's built-in updater checks the GitHub releases endpoint
   on startup; background download; prompt to restart
6. **Packaging:** `make package` produces platform-native installers for
   Linux (`.deb`, `.rpm`, `.AppImage`), macOS (`.dmg`), Windows (`.msi`)
7. **Browser-only mode:** SvelteKit adapter-static build produces a standalone
   SPA deployable to any static host; Tauri-specific IPC calls are gated behind
   an `isTauri()` guard and fall back to browser `<input type="file">` and
   `URL.createObjectURL()` download

**Deliverable:** Full tool ships to modders as a signed desktop app. Browser
version is deployable as a hosted tool. Both targets built and packaged from
`make package`.

---

## Makefile

The Makefile is the single entry point for all developer and CI operations.
All targets are documented with `##` comments so `make help` prints a formatted
summary.

```makefile
# =============================================================================
# Voxelite Model Rigger — Build System
#
# Targets:
#   make help          Print this help message
#   make dev           Run browser dev server with hot reload (fastest loop)
#   make dev-app       Run Tauri dev mode (native window + hot reload)
#   make test          Run all tests (unit + E2E)
#   make test-unit     Run Vitest unit tests only
#   make test-e2e      Run Playwright E2E tests only
#   make lint          Run ESLint + Prettier check
#   make fmt           Auto-format all source files
#   make build         Build SvelteKit SPA (static adapter, browser-only)
#   make build-app     Build Tauri desktop binary (current platform)
#   make package       Build and package for all platforms (requires cross env)
#   make clean         Remove all build artifacts
# =============================================================================

.PHONY: help dev dev-app test test-unit test-e2e lint fmt build build-app package clean

# Default target — print help
help: ## Print this help message
	@grep -E '^[a-zA-Z_-]+:.*##' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS = ":.*##"}; {printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2}'

# ---------------------------------------------------------------------------
# Development
# ---------------------------------------------------------------------------

dev: ## Run browser dev server with hot reload (fastest dev loop — no Tauri)
	pnpm dev

dev-app: ## Run Tauri dev mode: native window with hot-reload frontend
	pnpm tauri dev

# ---------------------------------------------------------------------------
# Testing
# ---------------------------------------------------------------------------

test: test-unit test-e2e ## Run all tests

test-unit: ## Run Vitest unit tests (stores, state logic, export helpers)
	pnpm vitest run

test-unit-watch: ## Run Vitest in watch mode (interactive, use during TDD)
	pnpm vitest

test-e2e: ## Run Playwright E2E tests (requires a running build)
	pnpm playwright test

# ---------------------------------------------------------------------------
# Code quality
# ---------------------------------------------------------------------------

lint: ## Run ESLint and Prettier in check mode (no writes)
	pnpm eslint src --ext .ts,.svelte
	pnpm prettier --check src

fmt: ## Auto-format source files with Prettier
	pnpm prettier --write src

# ---------------------------------------------------------------------------
# Builds
# ---------------------------------------------------------------------------

build: ## Build SvelteKit SPA (static adapter, browser-only distributable)
	pnpm build

build-app: ## Compile Tauri desktop binary for the current platform
	pnpm tauri build --no-bundle

# ---------------------------------------------------------------------------
# Packaging / Distribution
# ---------------------------------------------------------------------------

package: ## Build + package native installers for all platforms
	# Builds .deb, .rpm, .AppImage (Linux), .dmg (macOS), .msi (Windows)
	# Requires the Tauri cross-compilation environment:
	#   https://tauri.app/v2/guides/building/cross-platform
	pnpm tauri build

# ---------------------------------------------------------------------------
# Housekeeping
# ---------------------------------------------------------------------------

clean: ## Remove all build artifacts (build/, .svelte-kit/, src-tauri/target/)
	rm -rf build .svelte-kit
	cargo clean --manifest-path src-tauri/Cargo.toml
```

### pnpm scripts (package.json)

The Makefile delegates to pnpm scripts so the tool also works without `make`
(CI environments, Windows developers not using WSL):

```jsonc
{
  "scripts": {
    "dev":           "vite dev",
    "build":         "vite build",
    "preview":       "vite preview",
    "test":          "vitest run && playwright test",
    "vitest":        "vitest",
    "playwright":    "playwright test",
    "lint":          "eslint src --ext .ts,.svelte",
    "fmt":           "prettier --write src",
    "tauri":         "tauri"
  }
}
```

---

## Directory Layout

```
tools/model-rigger/
├── Makefile                    ← single source of truth for all build commands
├── package.json
├── pnpm-lock.yaml
├── svelte.config.ts            ← static adapter for browser build
├── vite.config.ts
├── tsconfig.json
├── .eslintrc.cjs
├── .prettierrc
├── playwright.config.ts
│
├── src/                        ← SvelteKit frontend
│   ├── app.html
│   ├── app.css                 ← Tailwind base import
│   ├── lib/
│   │   ├── babylon/
│   │   │   ├── scene-bridge.ts ← Svelte store → Babylon scene sync
│   │   │   ├── gizmos.ts       ← bone gizmo lifecycle
│   │   │   ├── weight-viz.ts   ← heatmap shader material
│   │   │   └── export.ts       ← GLTF2Export wrapper
│   │   ├── stores/
│   │   │   ├── model.ts
│   │   │   ├── skeleton.ts
│   │   │   ├── weights.ts
│   │   │   ├── animation.ts
│   │   │   └── ui.ts
│   │   ├── components/
│   │   │   ├── BabylonViewport.svelte
│   │   │   ├── BoneTree.svelte
│   │   │   ├── WeightPainter.svelte
│   │   │   ├── AnimationTimeline.svelte
│   │   │   └── Inspector.svelte
│   │   └── ipc.ts              ← Tauri IPC wrapper with browser fallbacks
│   └── routes/
│       ├── +layout.svelte
│       ├── +page.svelte         ← landing: open file, recent files
│       └── editor/
│           └── +page.svelte     ← main rigging workspace
│
├── src-tauri/                  ← Tauri Rust backend
│   ├── Cargo.toml
│   ├── tauri.conf.yaml         ← Tauri config (YAML, not JSON)
│   └── src/
│       ├── main.rs
│       └── commands/
│           ├── file.rs         ← open_file, save_file IPC commands
│           ├── watch.rs        ← directory watcher
│           └── recent.rs       ← recent files list (reads/writes recent.yaml)
│
└── tests/
    ├── unit/                   ← Vitest: store logic, export helpers
    └── e2e/                    ← Playwright: smoke tests, editor flows
```

---

## Browser vs. Desktop Feature Matrix

| Feature | Browser | Desktop (Tauri) |
|---------|---------|-----------------|
| Open `.glb` / `.obj` / `.ply` | `<input type="file">` | Native file dialog |
| Save `.glb` | Download via blob URL | Write directly to disk |
| Recent files | `localStorage` | `recent.yaml` in app data dir |
| File watching | Not supported | Tauri `watch_directory` IPC |
| Auto-updater | N/A (reload page) | Tauri updater |
| Offline use | Requires CDN or local server | Fully offline |
| Installation | Zero (open URL) | Platform installer |

The `ipc.ts` module provides a unified API surface; components never call Tauri
directly:

```typescript
// ipc.ts
import { isTauri } from './env'

export async function openFile(): Promise<File | null> {
  if (isTauri()) {
    const path = await invoke<string>('open_file', { filters: ['glb'] })
    return path ? fetchLocalFile(path) : null
  }
  return showBrowserFilePicker(['glb'])
}
```

---

## Integration with Asset Pipeline

The Model Rigger fits into the pipeline documented in `04-asset-pipeline.md`
as an alternative to Blender for the rigging and animation steps:

```
Model                  Rig & Animate              Export    Import / Load
──────                 ─────────────              ──────    ─────────────
MagicaVoxel       →   Model Rigger (this tool) →  .glb  →  Godot (runtime)
Blender (mesh)         (or Blender Armatures)
```

The exported `.glb` is identical in structure to a Blender export — Godot's
`GLTFDocument` importer handles both without distinction. Animation clips appear
as named `AnimationPlayer` actions.

---

## Relation to Tooling Strategy

This tool follows all conventions established in `08-tooling.md`:

- Same Tauri shell, same filesystem IPC pattern as Event Tree Editor
- Tauri config uses YAML (`tauri.conf.yaml`) — no JSON config files
- Browser dev mode for fast iteration; Tauri wraps the same codebase for distribution
- Ships to modders through the same distribution channel as other creator tools
- Auto-updater on the same release track

The choice of SvelteKit (rather than React, used by Event Tree Editor) is
acceptable because the two tools are independent applications that share no
frontend code. SvelteKit's lighter reactive model is a better fit for a
timeline-and-panels tool than a node-graph editor.
