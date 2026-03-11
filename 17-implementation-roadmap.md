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

## Phase 2 — Core Engine Systems

6. YAML loading (godot-yaml GDExtension integration)
7. Registry system (namespaced ID registration, lookup)
8. Message bus (command/event dispatch)
9. Block registry (blocks defined in YAML, loaded into registry)
10. Item registry + basic inventory (hotbar, pick up/drop)

---

## Phase 3 — Content Pipeline

11. Mod loader (discover mods, parse mod.yaml, load order)
12. LuaJIT integration (sandboxed VM, basic API surface)
13. Entity system (archetypes, component registry, spawning)
14. Base game as "core" mod (blocks, items, entities in YAML+Lua)

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
