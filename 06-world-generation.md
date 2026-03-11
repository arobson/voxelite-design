# World Generation Architecture

---

## Core Principle: Core Game = First-Party Mod

The engine has **no hardcoded biomes**. The base game's biomes, terrain generators,
structures, and decorators live in `res://core/` and are loaded first — but they use the
exact same YAML + Lua system that community modders use. There is no special engine code
path for built-in content.

**Practical consequence:** To change a base-game biome, you edit a YAML file. A modder
overriding that biome does the same thing. Same tools, same format, same pipeline.

---

## Registry Architecture

All world generation content is registered into global lookup tables at load time.

| Registry | Key format | Populated by |
|----------|-----------|--------------|
| `BiomeRegistry` | `core:forest`, `mymod:crystal_tundra` | All mods including core |
| `GeneratorRegistry` | `core:default_terrain` | All mods including core |
| `StructureRegistry` | `core:dungeon`, `mymod:crystal_tower` | All mods including core |
| `DecoratorRegistry` | `core:oak_tree`, `mymod:ice_spike` | All mods including core |
| `DimensionRegistry` | `core:overworld`, `mymod:crystal_realm` | All mods including core |

The world generator queries these registries. It does not care who registered what.

---

## 8-Stage Generation Pipeline

### Stage 1 — Climate Map
Computed once per world. Three continent-scale noise fields:

```lua
ClimateMap.temperature(wx, wz)  →  0.0 (arctic)  .. 1.0 (tropical)
ClimateMap.humidity(wx, wz)     →  0.0 (arid)     .. 1.0 (swamp)
ClimateMap.altitude(wx, wz)     →  0.0 (sea level) .. 1.0 (high mountain)
```

Low-frequency, large-scale fields — climate zones span thousands of blocks.

### Stage 2 — Biome Selection
For each column, climate values are looked up against all registered biomes. Each biome
declares a climate niche in YAML. Best match wins. Biome blending computed at borders
across a configurable `blend_radius` (default: 24 blocks).

### Stage 3 — Base Terrain (per chunk, worker thread)
Each biome's Lua terrain generator receives chunk coordinates and a noise utility object.
Returns a 16×16 heightmap. Optionally returns a 3D density field for overhangs and
floating islands. At biome borders, height maps from both biomes are blended by the engine
before block placement.

### Stage 4 — Block Placement (per chunk, worker thread)
Engine fills columns using the biome's block palette from YAML: surface block, subsurface
layers, stone fill, ocean floor. Ore veins placed using 3D noise against the biome's ore
table.

### Stage 5 — Cave Generation (per chunk, worker thread)
Default: worm algorithm driven by biome's cave profile YAML. Biomes can supply a
`custom_generator` Lua path to override the worm algorithm entirely.

### Stage 6 — Structure Placement (per chunk, worker thread)
Structures declared in biome YAML placed using seeded placement grid resolved from world
coordinates (so structures spanning multiple chunks remain consistent). Exclusion radius
prevents overlap.

### Stage 7 — Decoration Pass (per chunk, worker thread)
Small Lua decorator functions place surface details: trees, bushes, rocks, tall grass.
Each biome lists decorators in YAML with density weights. Cheapest and most mod-friendly
extension point.

### Stage 8 — Entity Spawning (main thread, post-load)
Initial populations seeded from biome spawn tables when chunk first loads. Ongoing
spawning evaluated every N seconds in loaded chunks. **Mod entities opt in by declaring
`biomes: [core:forest]` in their entity YAML** — no biome file editing required.

---

## Biome YAML Format

### Complete example — base game biome

```yaml
# core/biomes/forest.yaml

id: core:forest
display_name: Temperate Forest
category: land

climate:
  temperature: { min: 0.4,  max: 0.7  }
  humidity:    { min: 0.50, max: 0.80 }
  altitude:    { min: 0.10, max: 0.60 }
  weight: 1.0

terrain:
  generator: generators/default_terrain.lua
  base_height: 68
  height_variation: 18
  smoothness: 0.65       # 0 = jagged, 1 = flat

blocks:
  surface:          core:grass_block
  subsurface:       core:dirt
  subsurface_depth: 4
  fill:             core:stone
  underwater:       core:gravel
  beach:            core:sand
  beach_depth:      3
  bedrock:          core:bedrock

ores:
  - { block: core:coal_ore, vein: 8, count: 20, y_range: [0, 128] }
  - { block: core:iron_ore, vein: 5, count: 14, y_range: [0, 64]  }
  - { block: core:gold_ore, vein: 4, count: 4,  y_range: [0, 32]  }

caves:
  density:        0.012
  radius:         { min: 1.5, max: 3.5 }
  branch_chance:  0.25
  aquifer_chance: 0.08
  lava_min_y:     12
  custom_generator: null   # override with lua path for special caves

structures:
  - { id: core:forest_cottage, chance: 0.004, separation: 200 }
  - { id: core:ruined_tower,   chance: 0.002, separation: 400 }

decorators:
  - { id: core:oak_tree,    density: 0.08 }
  - { id: core:birch_tree,  density: 0.03 }
  - { id: core:tall_grass,  density: 0.30, cluster: true }
  - { id: core:wildflowers, density: 0.04 }
  - { id: core:boulder,     density: 0.005 }

atmosphere:
  sky_color:     "#7ab8e8"
  fog_color:     "#c8dde8"
  fog_density:   0.001
  ambient_light: 0.6
  water_color:   "#3a7fc1"
  water_opacity: 0.7

  sounds:
    day:   [core:bird_chirp, core:wind_leaves]
    night: [core:cricket, core:owl_hoot]
    rain:  [core:rain_forest]

  particles:
    - { id: core:firefly,      density: 0.002, conditions: { time: [night] } }
    - { id: core:falling_leaf, density: 0.015 }

spawning:
  passive:
    - { entity: core:deer,   group: [2,4], weight: 10, light_min: 9 }
    - { entity: core:rabbit, group: [2,6], weight: 12, light_min: 9 }
  hostile:
    - { entity: core:zombie,   group: [1,3], weight: 8, light_max: 7, time: [night] }
    - { entity: core:skeleton, group: [1,3], weight: 8, light_max: 7, time: [night] }
```

### Mod-added biome

Same format exactly. Biome participates in overworld generation automatically based on
its declared climate niche — no core file editing required.

```yaml
# mymod/biomes/crystal_tundra.yaml

id: mymod:crystal_tundra
display_name: Crystal Tundra
category: land

climate:
  temperature: { min: 0.0, max: 0.25 }
  humidity:    { min: 0.1, max: 0.45 }
  altitude:    { min: 0.2, max: 0.85 }
  weight: 0.8

terrain:
  generator: generators/crystal_terrain.lua  # custom generator
  base_height: 72
  height_variation: 28
  smoothness: 0.4

blocks:
  surface:          mymod:crystal_grass
  subsurface:       mymod:crystal_dirt
  subsurface_depth: 3
  fill:             mymod:crystal_stone
  underwater:       mymod:frozen_gravel
  bedrock:          core:bedrock            # can reference core blocks

ores:
  - { block: mymod:crystal_ore,  vein: 6, count: 18, y_range: [0, 200] }
  - { block: mymod:resonite_ore, vein: 3, count: 6,  y_range: [0, 48]  }
  - { block: core:iron_ore,      vein: 4, count: 8,  y_range: [0, 48]  }

caves:
  density: 0.008
  radius:  { min: 2.0, max: 5.0 }
  custom_generator: generators/crystal_cave_gen.lua

structures:
  - { id: mymod:crystal_tower,  chance: 0.006, separation: 180 }
  - { id: core:ruined_tower,    chance: 0.001, separation: 500 }

decorators:
  - { id: mymod:crystal_cluster, density: 0.06, cluster: true }
  - { id: mymod:ice_spike,       density: 0.02 }
  - { id: core:boulder,          density: 0.003 }

atmosphere:
  sky_color:     "#a8c8f8"
  fog_color:     "#d0e8ff"
  fog_density:   0.003
  ambient_light: 0.5

  sounds:
    day:   [mymod:crystal_hum, core:wind_cold]
    night: [mymod:crystal_resonance]

  particles:
    - { id: mymod:crystal_mote, density: 0.01 }
    - { id: core:snowflake,     density: 0.02 }

spawning:
  passive:
    - { entity: mymod:crystal_deer, group: [2,5], weight: 8 }
  hostile:
    - { entity: mymod:crystal_golem, group: [1,2], weight: 5, light_max: 8 }
    - { entity: core:skeleton,       group: [1,3], weight: 6, light_max: 7 }
```

### Biome override — extend without replacing

```yaml
# mymod/biomes/overrides/core_forest.yaml

override: core:forest   # target biome; only declared fields are changed

blocks:
  surface: mymod:autumn_grass   # replace surface block

decorators:
  append:                        # add to existing list, not replace
    - { id: mymod:autumn_leaves_pile, density: 0.12 }
    - { id: mymod:mushroom_ring,      density: 0.008 }

structures:
  append:
    - { id: mymod:witch_hut, chance: 0.002, separation: 300 }

spawning:
  passive:
    append:
      - { entity: mymod:forest_spirit, group: [1,1], weight: 2 }

atmosphere:
  fog_color: "#d4a870"
  sky_color: "#e8c090"
```

---

## Terrain Generator Lua API

### Default terrain generator

```lua
-- generators/default_terrain.lua
local DefaultTerrain = {}

function DefaultTerrain.heightmap(cx, cz, params, noise)
  local h = {}
  for x = 0, 15 do
    h[x] = {}
    for z = 0, 15 do
      local wx = cx * 16 + x
      local wz = cz * 16 + z
      local continent = noise.simplex(wx * 0.0008, wz * 0.0008) * 20
      local mid       = noise.fractal(wx * 0.005,  wz * 0.005, 4, 0.55) * params.height_variation
      local fine      = noise.simplex(wx * 0.04,   wz * 0.04) * 2.5
      h[x][z] = math.floor(params.base_height + continent + mid + fine)
    end
  end
  return h
end

-- Return nil to use heightmap only (no overhangs)
function DefaultTerrain.density(cx, cy, cz, params, noise)
  return nil
end

return DefaultTerrain
```

### Custom terrain with 3D density (floating islands)

```lua
-- Returns 3D density field for overhangs and floating shards
function CrystalTerrain.density(cx, cy, cz, params, noise)
  local d = {}
  for x = 0, 15 do
    d[x] = {}
    for y = 0, 15 do
      d[x][y] = {}
      for z = 0, 15 do
        local wx, wy, wz = cx*16+x, cy*16+y, cz*16+z
        local shard = 0
        if wy > 100 then
          shard = noise.simplex3(wx*0.04, wy*0.06, wz*0.04)
          shard = shard * math.max(0, 1 - (wy - 100) / 40)
        end
        d[x][y][z] = shard > 0.62 and 1 or 0
      end
    end
  end
  return d
end
```

---

## Noise API Reference

All noise functions available in terrain generator scripts. Deterministic — same inputs
always produce the same outputs. Seed is bound at world creation, not passed per-call.

```lua
-- 2D
noise.simplex(x, z)                              →  [-1, 1]
noise.fractal(x, z, octaves, persistence)        →  [-1, 1]
noise.ridged(x, z)                               →  [-1, 1]  -- sharp peaks
noise.billow(x, z)                               →  [-1, 1]  -- rounded bumps
noise.white(x, z)                                →  [0, 1]   -- uncorrelated

-- 3D
noise.simplex3(x, y, z)                          →  [-1, 1]
noise.fractal3(x, y, z, octaves, persistence)    →  [-1, 1]
noise.cellular3(x, y, z)                         →  { distance, id }
noise.white3(x, y, z)                            →  [0, 1]

-- Domain warping
noise.warp2(x, z, strength, freq)               →  { wx, wz }

-- Example: ridged mountains with domain warp
local w      = noise.warp2(wx * 0.005, wz * 0.005, 20, 0.003)
local ridges = noise.ridged(w.wx, w.wz) * 40
local base   = noise.fractal(wx * 0.006, wz * 0.006, 4, 0.5) * 20
local height = base_height + base + ridges
```

---

## Dimension Definition

```yaml
# core/dimensions/overworld.yaml

id: core:overworld
display_name: Overworld

height:
  min: -64
  max: 320
  sea_level: 64

climate:
  temperature_scale: 0.0008
  humidity_scale:    0.0009
  altitude_scale:    0.0006
  blend_radius:      24        # blocks of biome blending at borders

biome_categories: [land, ocean]
biomes: all                    # include all registered biomes of eligible categories
                               # or: biomes: [core:forest, mymod:crystal_tundra, ...]

global_ores:
  - { block: core:bedrock, y_range: [0, 1], fill: true }

default_atmosphere:
  sky_color:   "#7ab8e8"
  fog_density: 0.0005

generator: null   # null = standard climate → biome → terrain pipeline
```

---

## Core Game mod.yaml

The base game registers its content exactly like any user mod:

```yaml
# core/mod.yaml

id: core
display_name: "Base Game"
version: 1.0.0
type: core          # load priority: always first; otherwise identical to expansion

registers:
  biomes:
    - biomes/forest
    - biomes/desert
    - biomes/tundra
    - biomes/plains
    - biomes/swamp
    - biomes/mountains
    - biomes/ocean
    - biomes/jungle
    - biomes/savanna
    - biomes/mushroom_island

  generators:
    - generators/default_terrain
    - generators/ocean_terrain
    - generators/mountain_terrain

  structures:
    - structures/forest_cottage
    - structures/ruined_tower
    - structures/dungeon
    - structures/village

  decorators:
    - decorators/oak_tree
    - decorators/birch_tree
    - decorators/cactus
    - decorators/tall_grass
    - decorators/boulder

  dimensions:
    - dimensions/overworld
    - dimensions/underworld
```

---

## Seed Stability Contract

This constraint must be maintained throughout implementation:

1. Noise functions are **pure** — they receive only world coordinates and seed, never
   mutable global state
2. Terrain generators must be **referentially transparent** — same inputs, same output
3. Decorator and structure placement must use **seeded RNG derived from world position**,
   not a shared global RNG state
4. Mod load order must **not affect terrain output** for the same seed
5. Adding or removing a mod must not change terrain in worlds created before that mod
   was installed (for already-generated chunks)
