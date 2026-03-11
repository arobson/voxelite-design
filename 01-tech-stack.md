# Tech Stack Decisions

---

## Game Engine — Godot 4

**Decision:** Godot 4

**Rationale:**
- Open source (MIT) — the engine itself can be forked and modified
- `godot_voxel` plugin (by Zylann) is production-tested and feature-rich
- GDScript serves as a natural built-in scripting layer alongside LuaJIT
- Vulkan renderer with PBR, SDFGI global illumination, and volumetric fog out of the box
- PCK file format is a natural mod distribution container
- Actively maintained with a strong indie community

**Rejected alternatives:**
- Unreal Engine 5 — best raw graphics but not voxel-native; 5% royalty; limited modding story
- Unity 6 — technically capable but 2023 runtime fee controversy; closed source; third-party modding tools only
- Custom C++/Rust engine — maximum control but years before playable prototype; out of scope

---

## Core Language — GDScript + GDExtension

**Decision:** GDScript for engine-level code; GDExtension (C++) for performance-critical systems

**Rationale:**
- GDScript integrates directly with Godot's node system, signals, and scene tree
- C++ via GDExtension for: chunk meshing, noise generation, Lua VM embedding, physics-heavy systems
- Avoids premature optimisation — start in GDScript, extract to GDExtension when profiling identifies bottlenecks

---

## Mod Scripting Language — LuaJIT

**Decision:** LuaJIT 2.x (embedded C library)

**Rationale:**
- Industry standard for game modding (Factorio, World of Warcraft, Garry's Mod, Roblox)
- LuaJIT is extremely fast — JIT-compiled, close to native C for hot paths
- Small footprint; easy to sandbox per-mod with isolated `lua_State` per mod
- Large existing modder community already familiar with the syntax
- Tables as first-class objects fit game data models naturally

**Integration approach:**
- Embed LuaJIT via GDExtension
- Each mod gets its own `lua_State` — complete VM isolation
- Engine API exposed via `lua_setfenv` + whitelist of permitted globals
- Dangerous globals stripped: `io`, `os`, `require`, `package`, `debug`, `load`, `dofile`, `loadfile`

**YAML in Lua:**
- `lua-yaml` (pure Lua library) bundled in the mod runtime
- Exposed as `YAML.load(text)` in the mod Lua API
- Mods can parse their own YAML config files from within scripts

---

## Renderer — Godot Vulkan + SDFGI

**Decision:** Godot 4 Forward+ renderer with SDFGI enabled

**Key features used:**
- **SDFGI** (Signed Distance Field Global Illumination) — real-time indirect lighting that bounces off terrain, making voxel worlds look dramatically more atmospheric than Minecraft's flat lighting
- **SSR** (Screen Space Reflections) — for water and crystal surfaces
- **SSAO** (Screen Space Ambient Occlusion) — contact shadows in crevices and caves
- **Volumetric Fog** — biome-specific atmospheric depth
- **Sky shaders** — programmable sky with time-of-day transitions
- **RenderingDevice API** — custom compute shaders for special voxel effects if needed

---

## Voxel Architecture — Hybrid Chunked Meshing

**Decision:** Greedy-meshed chunk grid for placed blocks + smooth terrain via Dual Contouring for natural landscape features

**Chunk specification:**
- Chunk size: 16 × 16 × 16 blocks (standard; tunable)
- Greedy meshing reduces polygon count dramatically vs. naive per-face meshing
- Smooth terrain layer for hills, caves, and organic landforms
- Seam management between blocky and smooth regions handled at the meshing layer
- LOD: reduce chunk resolution with distance (godot_voxel provides this)

**godot_voxel plugin:**
- Use as the foundation for chunk management, meshing, and LOD
- Extend with custom generators via its Lua-exposed generator API

---

## Data Format — YAML

**Decision:** YAML for all data files (biome definitions, item definitions, entity definitions, crafting recipes, mod manifests, etc.)

**Rationale over JSON:**
- Supports comments — modders can document their intent
- No trailing comma errors or quote-every-key discipline required
- More readable for nested structures (loot tables, spawn rules, block palettes)
- Standard choice for configuration in modern tools (GitHub Actions, Docker Compose, etc.)

**Parser stack:**
- Engine side (GDScript): `godot-yaml` GDExtension — wraps `yaml-cpp`, returns Godot Dictionary
- Lua side: `lua-yaml` pure Lua library — bundled in mod runtime, exposed as `YAML.load()`

**File extension:** `.yaml` (not `.yml`) — enforced consistently across core and mods

---

## 3D Asset Format — glTF 2.0

**Decision:** glTF 2.0 binary (`.glb`) as the single supported 3D asset format

**Rationale:**
- Godot 4's glTF importer is first-class — co-maintained with Khronos
- Carries meshes, materials, armatures, animations, and blend shapes in one file
- Binary `.glb` is compact and fast to import
- **Godot 4.2+ runtime loading:** `GLTFDocument.append_from_file()` loads `.glb` files at runtime without the editor — the key enabler for mod 3D assets
- Open standard — no proprietary dependency

**Pipeline:** Blender / MagicaVoxel → `.glb` export → Godot auto-import (editor) or runtime load (mods)

**FBX:** Accepted from internal artists (Godot 4.3+ bundles FBX2glTF); not recommended as a modder-facing format.
