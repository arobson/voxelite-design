# Voxel Survival Game — Architecture Overview

This document set captures all architecture decisions made during the planning phase.
Provide these files to Claude Code as project context before beginning implementation.

---

## Document Index

| File | Contents |
|------|----------|
| `00-overview.md` | This file — index and guiding principles |
| `01-tech-stack.md` | Engine, language, renderer, and core library decisions |
| `02-audio.md` | Audio library architecture and format strategy |
| `03-input.md` | Input system design |
| `04-asset-pipeline.md` | 3D modeling, animation, texturing, and import pipeline |
| `05-modding-system.md` | Four-tier mod architecture, Lua API, security model |
| `06-world-generation.md` | Biome system, terrain pipeline, noise API, YAML format |
| `07-event-system.md` | Event trees, NPC dialogue, quest system, state store, markers |
| `08-tooling.md` | Creator tools strategy, event editor, CLI tools, in-game debug |
| `09-player-progression.md` | Survival mechanics, buff system, XP, skill trees, combat, death |
| `10-entity-architecture.md` | Composition model, components, message bus, lifecycle, navigation, ownership |
| `11-save-system.md` | Hybrid storage, chunk priority saves, backups, mod data migration |
| `12-networking.md` | Server model, ENet/Steam protocol, prediction, replication, chunk streaming |
| `13-physics.md` | Jolt collision, character physics, pluggable fluid system, fire spread |
| `14-lighting.md` | RGB block light, skylight, data textures, SDFGI integration, entity lights |
| `15-ui-system.md` | Theme system, mod YAML GUIs, drag-and-drop, crafting auto-pull, HUD |
| `16-performance.md` | Hardware tiers, frame budgets, quality presets, memory, optimization |

---

## Guiding Principles

These principles should inform every implementation decision:

### 1. Core game = first-party mod
The base game's content (biomes, items, entities, structures) is defined using the exact
same YAML + Lua system that community modders use. No hardcoded special cases. The engine
only knows how to run the pipeline — it does not care whether content comes from `core` or
a user mod.

### 2. Data in YAML, logic in Lua
All declarative data (stats, block palettes, climate ranges, crafting recipes, spawn tables)
lives in `.yaml` files. All procedural logic (terrain generators, AI behavior, item effects,
machine ticks) lives in `.lua` scripts. The two are always separate files.

### 3. Open formats throughout
- Assets: **glTF 2.0 / .glb** for all 3D content
- Data: **YAML** for all configuration, definitions, and schemas — no JSON anywhere
- Scripting: **LuaJIT** for all game logic exposed to mods
- Audio: **OGG Vorbis** as the primary format

### 4. Registries, not hardcoding
Blocks, items, entities, biomes, structures, decorators, effects — everything is registered
into a named registry at load time using a namespaced ID (e.g. `core:oak_log`,
`mymod:crystal_sword`). Game systems query registries; they never reference content by
position in an array or by a magic integer.

### 5. Mod safety by default
Each mod's Lua runs in an isolated VM with a stripped global environment. CPU and memory
budgets are enforced. Mods can only write to the world through a narrow, auditable API
surface. A broken or malicious mod cannot crash the engine or corrupt other mods.

### 6. Additive, not replacement
Mods add to registries. They do not replace core systems unless they explicitly opt into
a system-level extension point (e.g. a custom terrain generator for their own biome). A
mod adding a new structure to the forest biome uses `append:` in an override YAML — it
does not rewrite the biome file.

### 7. Game is the runtime, tools are the workshop
Creator tools are standalone desktop apps (Tauri) and CLIs — not embedded in the game
binary. The game focuses on being a great runtime. Authoring tools are web apps wrapped
in Tauri for native distribution. The development team dogfoods every tool — if a workflow
requires internal-only tooling, the shipped tools are inadequate and must be fixed.

### 8. Seed-stable generation
World generation is deterministic. Given the same seed, the world must be identical
regardless of which mods are loaded or in what order. Noise functions are pure — they
receive only world coordinates and seed, never mutable state.

---

## Technology Stack Summary

| Concern | Choice |
|---------|--------|
| Game engine | Godot 4 |
| Core game language | GDScript + GDExtension (C++ where performance critical) |
| Mod scripting language | LuaJIT (embedded, sandboxed) |
| Renderer | Godot Vulkan — Forward+ with SDFGI global illumination |
| Voxel architecture | Hybrid chunked meshing (greedy blocks + smooth terrain) |
| 3D asset format | glTF 2.0 / .glb |
| Data / config format | YAML |
| Primary audio | Godot AudioServer (built-in) |
| Adaptive music | FMOD Studio (via fmod-for-godot plugin) |
| Input system | Godot InputMap (built-in) |
| YAML parser (engine) | godot-yaml GDExtension (wraps yaml-cpp) |
| YAML parser (Lua) | lua-yaml (pure Lua, bundled in mod runtime) |
| Creator tools | Tauri 2.x (Rust + React) desktop apps |
| Event tree editor | React Flow node graph in Tauri shell |
