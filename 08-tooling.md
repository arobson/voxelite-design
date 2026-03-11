# Creator Tooling

---

## Principle: Dogfood Everything

Every tool used to build the base game is shipped to modders. The development team does
not have access to internal-only authoring tools. If a workflow is painful with the
shipped tools, the tools are fixed — not bypassed.

---

## Platform Strategy

**The game is the runtime. Desktop apps are the workshop.**

Creator tools are web applications wrapped in Tauri for native distribution. This gives:
- Familiar web UI frameworks for complex editors (node graphs, forms, previews)
- Full filesystem access via Tauri's Rust backend
- Cross-platform: Windows, macOS, Linux
- Small binary (~5MB shell vs ~150MB Electron)
- No server required — tools work offline, read/write directly to mod directories
- Auto-updater for tool releases

**Development workflow:** Each tool is built as a Vite + web framework app. During
development, run in a browser with hot reload. For distribution, wrap in Tauri. Same
codebase, two targets.

---

## Tool Inventory

### Ships at launch

| Tool | Platform | Purpose |
|------|----------|---------|
| **Event Tree Editor** | Tauri app | Visual node-graph editor for quest/dialogue/event trees |
| **Mod Validator** | CLI | Schema validation, file whitelist, dependency checks |
| **Mod Scaffolder** | CLI | `voxelite create-mod mymod` generates directory structure |
| **In-game Console** | Godot (in-game) | Lua REPL, debug commands, entity/block inspector |
| **In-game Debug Overlay** | Godot (in-game) | FPS, chunk stats, registry browser, mod budget monitor |

### Ships post-launch

| Tool | Platform | Purpose |
|------|----------|---------|
| **Biome Preview** | Tauri app | Fly-cam terrain visualization for a single biome's parameters |
| **Structure Editor** | Tauri app or in-game | Build structures, place markers, export as .glb + metadata |
| **Item/Block Designer** | Tauri app | Visual form for YAML definitions, preview icons and stats |
| **Mod Workshop** | Web app (hosted) | Browse, install, rate, and publish mods |

---

## Event Tree Editor

The flagship creator tool. A visual node-graph editor for authoring event trees, NPC
dialogue, quest chains, and branching narratives.

### Tech stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Shell | Tauri 2.x | Native filesystem, small binary, cross-platform |
| Backend | Rust (Tauri) | File watching, YAML read/write, directory traversal |
| Frontend | React + TypeScript | Largest ecosystem, strong typing for complex UI |
| Node graph | React Flow | MIT licensed, battle-tested, built for node editors |
| YAML | `js-yaml` (browser) | Parse YAML → JS objects, serialize back cleanly |
| Styling | Tailwind CSS | Utility-first, fast iteration on editor UI |

### Features

**Core editing:**
- Open a mod directory — sidebar shows all event tree files
- Render tree nodes as a visual graph with draggable layout
- Add, edit, delete, connect nodes via drag
- Inline editing of dialogue text, conditions, and effects
- Typed node palette — drag `dialogue`, `condition_gate`, `effects_only`, etc. onto canvas
- Choice branches rendered as labeled output ports
- Auto-layout with manual override

**Chain references:**
- `chain:` references shown as labeled exit ports linking to other tree files
- Click a chain port to navigate to the linked tree
- Breadcrumb navigation for moving between linked trees
- Dangling chain references (target file missing) highlighted as warnings

**Conditions and effects:**
- Condition builder: dropdown for condition type, fields for parameters
- Effect builder: same pattern — select effect type, fill in parameters
- `custom_script` fields show the Lua filename with a button to open in external editor
- Conditions shown as colored badges on nodes; effects shown as an inline list

**Validation:**
- Real-time schema validation as you edit
- Unreachable nodes highlighted
- Missing required fields flagged
- Condition/effect parameter type checking
- Chain reference validation (target tree exists, entry node exists)

**YAML round-trip:**
- Opening a file parses YAML and renders the graph
- Saving serializes the graph back to clean, human-readable YAML
- Preserves comments from hand-edited files where possible
- Diff-friendly output: stable key ordering, consistent indentation

**File watching:**
- Tauri backend watches the mod directory for external changes
- If a file is modified outside the editor, prompt to reload
- Enables workflow: edit YAML in VS Code, see changes in editor (and vice versa)

### YAML ↔ Visual mapping

The visual editor maps 1:1 to the YAML structure defined in `07-event-system.md`:

```
YAML node            →  Visual node on canvas
node.choices[]       →  Output ports (one per choice)
node.next            →  Single output port (auto-advance)
node.chain           →  Exit port with file reference
node.conditions[]    →  Condition badges on node
node.effects[]       →  Effect list panel on node
trigger:             →  Special "start" node at top of graph
```

No hidden state. What you see in the editor is exactly what's in the YAML.

---

## Mod Validator (CLI)

A standalone command-line tool that validates a mod directory against the game's schemas.
Runs locally during development and in CI pipelines for mod publishing.

```bash
# Validate a mod directory
voxelite-validate ./my_mod

# Output
✓ mod.yaml — valid manifest
✓ entities/crystal_deer — valid entity definition
✓ items/crystal_sword — valid item definition
✗ quests/broken_quest.yaml — node "accept" references unknown chain target
✗ biomes/bad_biome.yaml — climate.temperature.min must be <= max
⚠ assets/unused_texture.png — not referenced by any definition

Summary: 4 checked, 2 errors, 1 warning
```

**Checks performed:**
- `mod.yaml` manifest schema and required fields
- All registered YAML files validated against their type schema (biome, entity, item, etc.)
- Event tree validation: reachable nodes, valid chain references, known condition/effect types
- File type whitelist enforcement
- Asset reference validation (referenced files exist)
- Dependency version range parsing
- Namespace validation (`id` field matches mod directory name)
- Path traversal detection

**Distribution:** Single binary. Built with Rust (shares schema definitions with Tauri tools)
or bundled Node.js via Bun single-file executable.

---

## Mod Scaffolder (CLI)

Generates a new mod directory with the correct structure and a starter `mod.yaml`.

```bash
# Create a new mod
voxelite create-mod crystal_creatures --type content

# Output
Created crystal_creatures/
  ├── mod.yaml
  ├── assets/
  ├── entities/
  ├── items/
  ├── blocks/
  ├── recipes/
  └── README.md

Edit mod.yaml to configure your mod, then run:
  voxelite-validate ./crystal_creatures
```

**Mod types generate different scaffolds:**
- `assets` — Tier 1: just `mod.yaml` + `assets/` with override examples
- `content` — Tier 2/3: entities, items, blocks, recipes directories + example files
- `expansion` — Tier 4: full directory structure including dimensions, systems, api/

---

## In-Game Console

Built into the Godot client. Toggled with backtick (`` ` ``) key. Available in
development builds and optionally in release builds (server-configurable).

**Features:**
- Lua REPL — execute arbitrary Lua in the game context
- Tab completion for API functions and registry IDs
- Command history (persisted to user config)
- Built-in commands:

```
/spawn <entity_id> [count]           — spawn entity at crosshair
/give <item_id> [count]              — add item to inventory
/tp <x> <y> <z>                      — teleport player
/tp <marker_name>                    — teleport to named marker
/biome                               — print current biome
/time <set|add> <value>              — set or advance time of day
/weather <clear|rain|storm>          — set weather
/reload_mod <mod_id>                 — hot-reload a mod's Lua scripts
/inspect                             — print registry info for block/entity at crosshair
/state <get|set|flag> <key> [value]  — read/write state store
/trigger <tree_id>                   — manually trigger an event tree
/quest <list|complete|reset> [id]    — quest debugging
```

---

## In-Game Debug Overlay

Toggle with `F3`. Shows real-time diagnostic information.

**Panels:**
- **Performance:** FPS, frame time, draw calls, triangle count
- **World:** player position, current chunk, biome, light level, time of day
- **Chunks:** loaded count, meshing queue depth, LOD distribution
- **Entities:** active count per type, budget usage per mod
- **Mods:** loaded mod list, per-mod CPU/memory usage, error log
- **Registry:** searchable browser of all registered IDs (blocks, items, entities, biomes)
- **State:** live view of state store flags and variables (filterable)
- **Events:** active event trees, recent triggers, state mutations log

---

## Shared Infrastructure

### Schema definitions

YAML schemas for all game data types (biome, entity, item, block, event tree, quest,
mod manifest, etc.) are defined once and shared across:

- Game engine (GDScript/C++ — validates at load time)
- Mod validator CLI
- Event tree editor (client-side validation)
- Future Tauri tools

**Schema format:** YAML schema files stored in a shared repository or package. Each tool
consumes them in its native language. The schemas are the single source of truth for what
constitutes valid YAML. No JSON files anywhere in the project — all data files use YAML.

### Mod directory convention

All tools expect the standard mod directory layout defined in `05-modding-system.md`.
The scaffolder generates it, the validator checks it, the event editor navigates it,
and the game engine loads from it.
