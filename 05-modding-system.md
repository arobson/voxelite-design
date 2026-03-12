# Modding System Architecture

---

## Overview

The mod system is tiered. Each tier builds on the previous. A modder can participate at
any tier without knowing anything about the tiers above it.

| Tier | Name | Code required | Audience |
|------|------|---------------|----------|
| 1 | Skin & Texture Packs | None | Artists, beginners |
| 2 | Characters, Skins & Behaviors | Lua | Intermediate modders |
| 3 | Items, Objects & New Mechanics | Lua + YAML | Experienced modders |
| 4 | Full Expansions | Lua + YAML + system hooks | Advanced modders |

---

## Mod Load Pipeline

Executed at startup before the main menu. All mods are fully loaded before the world
can be created.

```
1. Discovery    Scan mod directories, read mod.yaml manifests
2. Validation   Schema check, version compat, file type whitelist
3. Dep Graph    Build dependency tree, determine load order
4. Assets       Register texture/mesh/sound overrides (Tier 1)
5. Registries   Register items, blocks, entities, biomes, dimensions
6. Scripts      Load & sandbox Lua VMs, bind engine APIs
7. Systems      Initialize mod systems, call on_register() hooks
8. World Gen    Register terrain generators and structure placers
9. Done         Fire on_mods_loaded event — world creation available
```

**Hot-reload (development builds only):**
- Asset mods (Tier 1): filesystem watcher, live texture reload without restart
- Lua scripts (Tier 2–4): `reload_mod <id>` console command re-runs the Lua file
- Dimension/biome changes: require full world reload

---

## Mod Package Structure

Every mod is a directory (or `.zip`) with this layout:

```
my_mod/
├── mod.yaml          ← required manifest
├── assets/           ← textures, sounds, models
├── biomes/           ← biome YAML definitions
├── blocks/           ← block YAML + Lua
├── items/            ← item YAML + Lua
├── entities/         ← entity YAML + Lua
├── structures/       ← GLB + placement Lua
├── decorators/       ← decorator Lua scripts
├── generators/       ← terrain generator Lua scripts
├── recipes/          ← crafting recipe YAML
├── effects/          ← status effect YAML
├── dimensions/       ← dimension YAML
├── systems/          ← global system Lua scripts
└── api/              ← public API for other mods (expansions only)
```

---

## Tier 1 — Skin & Texture Packs

**No code required.** Pure asset replacement via manifest overrides.

### mod.yaml

```yaml
id: autumn_forest_pack
name: Autumn Forest Retexture
version: 1.0.0
type: assets
author: LeafArtist
game_version: ">=1.0.0"

overrides:
  textures/blocks/grass_block: textures/blocks/grass_autumn.png
  textures/blocks/oak_log:     textures/blocks/oak_log_weathered.png
  sounds/ambient/forest_day:   sounds/forest_wind_autumn.ogg
```

### Supported overrides
- Block textures (albedo, normal, roughness, metallic)
- Character and mob skin textures
- UI icons, inventory art, HUD elements
- Particle textures and sprite sheets
- Audio (SFX, ambient loops, music tracks)
- Skybox cubemaps

### Security
Zero attack surface — no executable code. File types validated against whitelist.
Path traversal rejected (all paths must resolve within mod directory).

---

## Tier 2 — Characters & Behaviors

Introduces new entities with custom meshes, animations, and Lua AI scripts.

### mod.yaml

```yaml
id: crystal_creatures
name: Crystal Creatures
version: 1.0.0
type: content
game_version: ">=1.0.0"

registers:
  entities:
    - entities/crystal_deer
    - entities/snow_rabbit
```

### Entity definition (YAML)

```yaml
# entities/crystal_deer/crystal_deer.yaml

id: mymod:crystal_deer
display_name: Crystal Deer
category: passive
health: 20
speed: 6.5
mesh: entities/crystal_deer/crystal_deer.glb

animations:
  idle:  Idle
  walk:  Walk
  run:   Run
  death: Death
  hurt:  Hurt

sounds:
  idle:  sounds/idle.ogg
  hurt:  sounds/hurt.ogg
  death: sounds/death.ogg

loot_table:
  - item: mymod:crystal_shard
    min: 1
    max: 3
    chance: 0.8
  - item: core:leather
    min: 1
    max: 2
    chance: 1.0

behavior_script: entities/crystal_deer/crystal_deer.lua

spawn_rules:
  biomes: [core:forest, mymod:crystal_tundra]
  light_level_min: 7
  group_min: 2
  group_max: 5
```

### Behavior script (Lua)

```lua
-- entities/crystal_deer/crystal_deer.lua
local CrystalDeer = {}

function CrystalDeer.on_spawn(entity)
  entity:set_state("wander")
  entity:play_sound("idle")
end

function CrystalDeer.on_tick(entity, delta)
  local state = entity:get_state()

  if state == "wander" then
    Behavior.wander(entity, { radius = 12.0, speed = entity.walk_speed })
    local player = entity:nearest_player()
    if player and entity:distance_to(player) < 8.0 then
      entity:set_state("flee")
    end

  elseif state == "flee" then
    local player = entity:nearest_player()
    if player then
      Behavior.flee_from(entity, player, { speed = entity.run_speed })
      if entity.health < 5 then
        entity:emit_particles("crystal_shimmer")
        entity:teleport_random(20.0)
        entity:set_state("wander")
      end
    else
      entity:set_state("wander")
    end
  end
end

function CrystalDeer.on_death(entity, killer)
  entity:emit_particles("crystal_burst", 20)
  if killer and killer:last_damage_type() == "magic" then
    World.spawn_item("mymod:crystal_shard", entity:position(), 5)
  end
end

function CrystalDeer.on_hurt(entity, attacker, damage)
  entity:play_sound("hurt")
  entity:set_state("flee")
end

return CrystalDeer
```

### Tier 2 Lua API surface

```lua
-- Entity state
entity:get_state() / entity:set_state(name)
entity:position() / entity:teleport(pos) / entity:teleport_random(radius)
entity:distance_to(other)
entity:nearest_player() / entity:nearest_entity(type)
entity.health / entity:get_health() / entity:set_health(v)
entity:last_damage_type()

-- Entity presentation
entity:play_sound(key) / entity:stop_sound(key)
entity:play_animation(name) / entity:queue_animation(name)
entity:emit_particles(effect_id, count)

-- Built-in behavior primitives
Behavior.wander(entity, opts)
Behavior.flee_from(entity, target, opts)
Behavior.chase(entity, target, opts)
Behavior.patrol(entity, waypoints, opts)
Behavior.idle(entity, duration)

-- World (read-heavy; limited writes)
World.spawn_item(item_id, position, count)
World.get_block(position)
World.get_biome(position)
World.get_light_level(position)
```

---

## Tier 3 — Items, Objects & New Mechanics

New craftable items, placeable blocks with custom logic, crafting recipes, status effects.

### Item definition (YAML)

```yaml
# items/crystal_sword/crystal_sword.yaml

id: mymod:crystal_sword
display_name: Crystal Sword
category: weapon
mesh: meshes/crystal_sword.glb
icon: textures/items/crystal_sword_icon.png
stack_size: 1
durability: 400

stats:
  damage: 18
  attack_speed: 1.6
  reach: 2.5

tags: [sword, crystal, magic_weapon]
tooltip: "Forged from crystallised mana. Hums faintly."

behavior_script: items/crystal_sword/crystal_sword.lua
on_events: [on_hit, on_equip, on_unequip, on_use]
```

### Item behavior script (Lua)

```lua
local CrystalSword = {}

function CrystalSword.on_equip(player, item)
  player:add_light_source("crystal_glow", { color = "#88ccff", energy = 0.4 })
  player:add_status_effect("mymod:crystal_attunement", { duration = -1 })
end

function CrystalSword.on_unequip(player, item)
  player:remove_light_source("crystal_glow")
  player:remove_status_effect("mymod:crystal_attunement")
end

function CrystalSword.on_hit(player, item, target, damage)
  if math.random() < 0.2 then
    target:add_status_effect("core:slow", { duration = 3.0, strength = 0.5 })
    target:emit_particles("crystal_freeze", 6)
  end
  if not target:has_tag("crystal") then
    Item.consume_durability(item, 2)
  end
end

function CrystalSword.on_use(player, item)
  if Item.is_on_cooldown(item, "burst") then return end
  local nearby = World.get_entities_in_radius(player:position(), 6.0)
  for _, entity in ipairs(nearby) do
    if entity:is_hostile() then
      entity:apply_damage(8, "magic", player)
      entity:add_status_effect("core:slow", { duration = 2.0 })
    end
  end
  World.play_sound("mymod:crystal_burst", player:position())
  World.spawn_particles("crystal_shockwave", player:position())
  Item.start_cooldown(item, "burst", 10.0)
end

return CrystalSword
```

### Block definition (YAML)

```yaml
# blocks/alchemy_table/alchemy_table.yaml

id: mymod:alchemy_table
display_name: Alchemy Table
mesh: meshes/alchemy_table.glb
hardness: 3.0
tool_required: core:axe
light_emission: 2

behavior_script: blocks/alchemy_table/alchemy_table.lua
on_events: [on_place, on_interact, on_tick, on_break]
tick_rate: 20   # ticks per second

gui: blocks/alchemy_table/alchemy_table_gui.yaml
```

### Block behavior script (Lua)

```lua
local AlchemyTable = {}

function AlchemyTable.on_place(world, pos, player)
  BlockEntity.create(pos, {
    slots    = { input1 = nil, input2 = nil, catalyst = nil, output = nil },
    progress = 0,
    recipe   = nil,
  })
end

function AlchemyTable.on_interact(world, pos, player)
  GUI.open("mymod:alchemy_table", pos, player)
end

function AlchemyTable.on_tick(world, pos, delta)
  local data = BlockEntity.get(pos)
  if not data then return end

  local recipe = Recipes.find_alchemy(data.slots)
  if not recipe then
    data.progress = 0
    BlockEntity.set(pos, data)
    return
  end

  data.progress = data.progress + delta
  if data.progress >= recipe.time then
    data.slots.input1   = nil
    data.slots.input2   = nil
    data.slots.catalyst = Item.consume_one(data.slots.catalyst)
    data.slots.output   = Item.stack(recipe.output, 1)
    data.progress       = 0
    World.play_sound("mymod:alchemy_complete", pos)
    World.spawn_particles("alchemy_poof", pos)
  end

  BlockEntity.set(pos, data)
end

function AlchemyTable.on_break(world, pos, player)
  local data = BlockEntity.get(pos)
  if data then
    for _, item in pairs(data.slots) do
      if item then World.drop_item(item, pos) end
    end
    BlockEntity.destroy(pos)
  end
end

return AlchemyTable
```

### Crafting recipe (YAML)

```yaml
# recipes/crystal_sword.yaml

id: mymod:recipe_crystal_sword
type: shaped
station: core:crafting_table

pattern:
  - " C "
  - " C "
  - " S "

ingredients:
  C: mymod:crystal_shard
  S: core:stick

result:
  item: mymod:crystal_sword
  count: 1
```

### Tier 3 full API surface

```lua
-- Player API
player:position() / player:look_direction()
player:add_status_effect(id, opts) / player:remove_status_effect(id)
player:add_light_source(id, opts)  / player:remove_light_source(id)
player:get_inventory()             / player:give_item(id, count)
player:get_stat(name)              / player:modify_stat(name, delta)

-- Item API
Item.consume_durability(item, amount)
Item.is_on_cooldown(item, key) / Item.start_cooldown(item, key, seconds)
Item.stack(item_id, count)     / Item.consume_one(item_stack)

-- BlockEntity API (persistent data store per block position)
BlockEntity.create(pos, initial_data)
BlockEntity.get(pos) / BlockEntity.set(pos, data)
BlockEntity.destroy(pos)

-- World API
World.get_entities_in_radius(pos, radius)
World.get_block(pos) / World.set_block(pos, block_id)
World.play_sound(id, pos)
World.spawn_particles(effect_id, pos)
World.drop_item(item_stack, pos)

-- Recipe & GUI API
Recipes.find_alchemy(slots)
GUI.open(gui_id, block_pos, player)
```

---

## Tier 4 — Full Expansions

Large-scale mods: new dimensions, biomes, game systems, quest chains, inter-mod APIs.

### mod.yaml

```yaml
id: crystal_realm
name: The Crystal Realm
version: 2.0.0
type: expansion
game_version: ">=1.2.0"

dependencies:
  required:
    - mymod:alchemy_mod@>=1.0.0
  optional:
    - mymod:magic_system@>=0.5.0

load_order: "after:alchemy_mod"

registers:
  dimensions:  [dimensions/crystal_realm]
  biomes:      [biomes/crystal_tundra, biomes/resonance_forest]
  systems:     [systems/resonance_system, systems/faction_system]
  quests:      [quests/quest_chain_main]
  public_api:  api/crystal_api.lua

keybindings:
  - id: crystal_realm:open_resonance_ui
    default_key: R
    description: "Open Resonance Energy panel"
    category: "Crystal Realm"
```

### Global system (Lua)

```lua
-- systems/resonance_system.lua
local ResonanceSystem = {}
local resonance = {}

function ResonanceSystem.on_register()
  HUD.register_element("crystal_realm:resonance_bar", {
    scene  = "res://mods/crystal_realm/hud/resonance_bar.tscn",
    anchor = "bottom_left",
    offset = { x = 10, y = -60 }
  })
  Events.register("on_player_join",  ResonanceSystem.on_player_join)
  Events.register("on_player_leave", ResonanceSystem.on_player_leave)
  Events.register("on_world_tick",   ResonanceSystem.on_world_tick)
end

function ResonanceSystem.on_player_join(player)
  resonance[player.id] = { current = 50, max = 100, regen = 2.0 }
  HUD.show_element("crystal_realm:resonance_bar", player)
end

function ResonanceSystem.on_world_tick(delta)
  for pid, data in pairs(resonance) do
    local player = World.get_player(pid)
    local rate   = data.regen
    if World.get_biome(player:position()) == "crystal_realm:crystal_tundra" then
      rate = rate * 2.5
    end
    data.current = math.min(data.current + rate * delta, data.max)
  end
end

function ResonanceSystem.consume(player, amount)
  local data = resonance[player.id]
  if not data or data.current < amount then return false end
  data.current = data.current - amount
  return true
end

return ResonanceSystem
```

### Public inter-mod API (Lua)

```lua
-- api/crystal_api.lua
-- Other mods load this via: local CrystalAPI = Mods.require("crystal_realm")

local ResonanceSystem = require("systems/resonance_system")
local FactionSystem   = require("systems/faction_system")

return {
  get_resonance      = ResonanceSystem.get,
  consume_resonance  = ResonanceSystem.consume,
  get_faction_rep    = FactionSystem.get_reputation,
  modify_faction_rep = FactionSystem.modify_reputation,
  teleport_to_realm  = function(player)
    Dimensions.teleport(player, "crystal_realm:crystal_realm")
  end,
}
```

### Available Tier 4 system hooks

```lua
Events.register("on_player_join",    fn)
Events.register("on_player_leave",   fn)
Events.register("on_player_death",   fn)
Events.register("on_world_tick",     fn)  -- every frame
Events.register("on_day_change",     fn)  -- in-game day boundary
Events.register("on_season_change",  fn)
Events.register("on_chunk_load",     fn)
Events.register("on_chunk_unload",   fn)
Events.register("on_mods_loaded",    fn)  -- after all mods load

Dimensions.register(id, dimension_yaml_table)
Dimensions.teleport(player, dimension_id)
HUD.register_element(id, opts)
HUD.show_element(id, player) / HUD.hide_element(id, player)
Mods.require(mod_id)          -- load another mod's public API
```

---

## Lua Runtime Architecture

### GDExtension foundation

The Lua runtime is built on [gilzoide/lua-gdextension](https://github.com/gilzoide/lua-gdextension)
with the LuaJIT variant. This provides the GDExtension lifecycle, Godot type marshalling
(Variant ↔ Lua for Vector3, Color, Dictionary, Array, Callable), and multiple `LuaState`
instances. A custom `ModSandbox` wrapper layer adds sandboxing, budget enforcement, and the
engine API surface.

### ModSandbox

Each mod gets a `ModSandbox` instance wrapping an isolated `LuaState`:

```
ModSandbox
├── LuaState (LuaJIT, isolated lua_State)
├── Engine API tables (Entity, World, Behavior, Events, etc.)
├── Pre-loaded mod data (YAML → Lua tables, loaded at pipeline step 5)
├── Command buffer (write intents collected during tick)
├── Wall-clock budget tracker
└── Memory allocator with cap
```

### No filesystem access

Mods have **no runtime filesystem API**. All mod data files (YAML definitions, Lua scripts,
lookup tables) are loaded by the engine during the mod load pipeline (steps 4-6) and injected
into the mod's Lua state as tables. This eliminates an entire class of security concerns —
no path validation, no scoping, no read-only wrappers needed.

Mods that ship large datasets (e.g., thousands of entries) have their data pre-loaded into
Lua tables at mod init. The 32MB memory budget per mod is generous for data tables.

### Command buffer pattern

Lua scripts never directly mutate game state. All writes flow through a command buffer:

```
Lua on_tick() → reads state snapshot → emits commands to buffer
  → Frame ends
  → Engine drains command buffers on main thread
  → Commands validated → events emitted → state updated
```

This decouples Lua execution from state mutation, which:
- Prevents race conditions if Lua ticks are parallelized later
- Matches the message bus command/event pattern already in the engine
- Ensures all state changes are validated before application

### State snapshot (read-only)

Before Lua ticks run, the engine captures a read-only snapshot of relevant game state:

```
Snapshot contents:
├── Entity positions, rotations, velocities
├── Entity health, AI state, buff lists
├── Nearby block state (within active range)
├── Player positions and states
├── Light levels (cached from lighting system)
└── Biome data (cached from world gen)
```

Lua API calls like `entity:position()`, `entity:distance_to()`, `World.get_block()`,
`World.get_light_level()`, and `World.get_entities_in_radius()` read from the snapshot,
not from the live Godot scene tree or PhysicsServer. This means:

- **Physics lookups are free** — no PhysicsServer calls from Lua, just cached data reads
- **Consistent state** — all Lua scripts in a frame see the same snapshot, no order-dependent
  bugs from one script's changes affecting another's reads
- **Thread-safe** — snapshot is immutable during Lua execution, enabling future parallelism

Snapshot scope scales with need. Phase 3 starts with entity positions and nearby blocks.
Future phases expand to include spatial indices for radius queries, block lighting, etc.

### Threading model

**Phase 3 (initial):** All Lua ticks run on the main thread. The command buffer and snapshot
interfaces are in place from day one, but execution is sequential.

**Future phases:** Because each mod has an isolated `lua_State` and reads from an immutable
snapshot, mod ticks can be fanned out to Godot's WorkerThreadPool:

```
Per frame:
  1. Main thread: capture state snapshot
  2. Worker threads: tick each mod's Lua scripts against snapshot
     └── Each mod writes to its own command buffer (no contention)
  3. Main thread: drain all command buffers, validate, apply
  4. Main thread: emit events, update scene tree
```

This is a deployment change, not an architecture change — the same ModSandbox interfaces
work in both single-threaded and multi-threaded modes.

**What runs on worker threads (future):**
- Entity AI ticks (per-mod `lua_State` isolation)
- Terrain generators (already async in godot_voxel)
- System hooks (`on_world_tick`)
- Custom component ticks

**What stays on the main thread (always):**
- Command validation and state mutation
- Scene tree modifications (spawn/despawn)
- Animation, sound, VFX triggers
- Event broadcast

---

## Security Model

### Lua Sandbox
- Each mod gets its own isolated `lua_State` — complete VM isolation
- Stripped globals: `io`, `os`, `require`, `package`, `debug`, `load`, `dofile`, `loadfile`
- No filesystem access — engine pre-loads all mod data during the load pipeline
- Network: none. No sockets, no HTTP
- Native libraries: mods cannot load `.dll`/`.so` files
- Cross-mod: only via declared public APIs (`Mods.require`) — no direct state sharing
- Error isolation: all Lua calls wrapped in `lua_pcall`; errors caught, logged, mod's
  tick skipped. One mod crashing never affects other mods or the engine

### CPU & Memory Budgets

| Context | Budget |
|---------|--------|
| Entity behavior scripts | 0.1ms per entity per tick |
| Block `on_tick` scripts | 0.5ms per block entity per tick |
| System `on_world_tick` total | 2ms per frame across all mods |
| Terrain generators | 5ms per chunk (worker thread) |
| Memory per mod | 32MB default (server-configurable) |

**CPU enforcement:** Wall-clock timing with per-call measurement. The engine measures elapsed
time around each Lua call (`on_tick`, `on_spawn`, etc.) and accumulates per mod per frame.
This preserves LuaJIT's JIT compilation — using Lua debug hooks would disable the JIT
entirely, defeating the purpose of choosing LuaJIT over PUC Lua.

Mods exceeding budget are flagged in the debug overlay with per-mod timing breakdowns.
Repeat offenders are throttled (tick rate halved, then quartered). Server operators can
force-disable mods via console.

**Memory enforcement:** Custom allocator via `lua_setallocf` tracks and caps per-mod
allocation. Allocation failures in Lua are caught by `lua_pcall` error handling.

### Asset Validation
- Allowed file types: `.png`, `.jpg`, `.ogg`, `.wav`, `.mp3`, `.glb`, `.yaml`, `.lua`
- Image max dimensions: 4096×4096
- GLB files: parsed against glTF schema; embedded scripts rejected
- Path traversal: all file references validated to remain within mod directory
- `mod.yaml` `id` field must match directory name exactly

### Dependency Management
- Circular dependency detection at load time
- Semver version range checks for required dependencies
- `conflict:` declarations respected
- Explicit `load_order: before/after` declarations; remainder topologically sorted
- Duplicate registry IDs: warning logged; last-registered wins (configurable)
- Expansion mods must declare all system registrations in `mod.yaml` — no dynamic injection

### Multiplayer
- Server is authoritative — Lua scripts run server-side only
- Clients must have matching mod list to join; mismatch gives clear error message
- Item/block/entity definitions validated against server registry on client join
- Cosmetic GUI mods may run client-side
