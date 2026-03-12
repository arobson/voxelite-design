# Implementation Roadmap

Six phases that layer systems incrementally. Each phase produces a runnable build
and avoids breaking what came before.

---

## Phase 1 — Skeleton (walk around in a voxel world) ✅

1. Godot project setup (folder structure, project.godot, export presets)
2. godot_voxel integration (chunk generation, basic terrain from noise)
3. Basic player controller (CharacterBody3D, WASD, jump, camera)
4. Block-accurate collision (per-chunk StaticBody3D)
5. Basic block interaction (place/break with crosshair raycast)

**Delivered:**
- Procedural terrain generation (simplex noise, 5 block types, sea level with water)
- First-person player controller with physics values from design doc 13
- Coyote time, jump buffering, variable jump height, air control
- Walk / sprint / crouch speeds
- Block breaking (hold left click) and placement (right click)
- Message bus emitting `block_broken` and `block_placed` events
- F3 debug overlay (FPS, position, velocity, target block)
- Mouse capture (Escape to release)

---

## Phase 2 — Core Engine Systems ✅

6. YAML loading (godot-yaml GDExtension integration)
7. Registry system (namespaced ID registration, lookup)
8. Message bus (command/event dispatch)
9. Block registry (blocks defined in YAML, loaded into registry)
10. Item registry + basic inventory (hotbar, pick up/drop)

**Delivered:**
- godot-yaml GDExtension v2.1.2 (RapidYAML C++ parser)
- Base Registry class with namespaced ID support (`namespace:name` format)
- BlockRegistry with bidirectional ID/voxel-index mapping, sort-ordered YAML loading
- ItemRegistry with YAML loading and block placement links
- Registries autoload loads core content from `data/core/` at startup
- GameBus enhanced with command validation layer (`submit_command` + `register_validator`)
- Inventory: 36 slots (9 hotbar + 27 backpack), stacking, add/remove/consume
- Hotbar UI: number keys 1-9, scroll wheel, visual selection highlight
- Block interaction uses registries for hardness-based break time, drops to inventory
- Terrain generator and block library built dynamically from registry data
- 6 block YAML definitions, 4 item YAML definitions
- Debug overlay shows registry stats, held item, target block ID
- Crouch: Ctrl shrinks collision to 1-block-tall, lowers camera, prevents standing under low ceilings
- Sprint: Shift for +60% walk speed (6.88 m/s), disabled while crouching or airborne
- Sprint-to-crouch slide: pressing crouch while sprinting preserves momentum with gradual friction decay
- Movement modes emit GameBus events for future XP accrual (movement skill tree, Phase 5)

---

## Phase 3 — Content Pipeline ✅

11. Mod loader (discover mods, parse mod.yaml, load order)
12. LuaJIT integration (sandboxed VM, basic API surface)
13. Entity system (archetypes, component registry, spawning)
14. Base game as "core" mod (blocks, items, entities in YAML+Lua)

**Delivered:**
- Base game restructured as "core" mod (`mods/core/` with `mod.yaml` manifest)
- Mod loader: directory discovery, manifest parsing, dependency graph, topological sort
- Core always loads first; circular dependency detection; conflict checking
- Registries autoload uses mod loader instead of hardcoded paths
- Old `data/core/` removed — all content loaded through mod pipeline
- Entity registry with YAML prototype loading (directory and flat-file support)
- Component registry with indexed queries (query by component set)
- Entity manager: archetype-based spawning via message bus commands
- Four archetype builders: creature, item_drop, projectile, block_entity
- Component sets auto-built from archetype + prototype data
- Entity spawn/despawn flows through GameBus command validation
- gilzoide/lua-gdextension v0.7.0 (LuaJIT variant) installed and enabled
- ModSandbox: per-mod isolated LuaState, stripped globals (io, os, require, package, debug)
- Command buffer pattern: Lua writes intents, engine validates and applies on main thread
- State snapshot: entity positions, health, components, player position captured per frame
- Behavior system: per-frame Lua ticks with snapshot → tick → drain cycle
- Commands API: spawn, despawn, damage, set_block, move_toward, stop, play_sound, emit_event
- Entity/World snapshot read API: positions, health, components, player position, nearby query
- Wall-clock budget tracking with per-mod timing and budget warnings
- Test creature with Lua behavior script (follows player, hovers above)
- Debug overlay shows mod count, entity registry stats, Lua sandbox timing per mod

---

## Phase 4 — World Systems

15. World generation pipeline (8-stage from design doc 06)
16. Biome system (climate grid, biome selection)
17. Lighting (RGB flood fill, data textures, skylight)
18. Fluid system (enhanced cellular automaton)
19. Save/load (SQLite chunks, YAML player data)

---

## Phase 5 — Gameplay

20. Health, buffs, consumables
21. Combat (melee/ranged, damage types)
    - Wire movement XP accrual (walk, sprint, crouch, jump → Physical XP)
    - Wire crouch noise reduction into stealth/detection system
22. NPC entity subtype (dialogue, reputation, schedule)
23. Event tree system (YAML parser, state store, triggers)
24. Skill/XP system
25. Crafting system (stations, recipes, auto-pull)

---

## Phase 6 — Multiplayer

26. Listen server architecture
27. Client-side prediction
28. Entity replication
29. Chunk streaming (seed + delta)
30. Dedicated server export
