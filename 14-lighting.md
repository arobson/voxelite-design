# Voxel Lighting System

---

## Overview

Lighting uses a two-layer approach: a voxel light propagation grid for precise per-block
illumination, and Godot's SDFGI for indirect bounce and ambient fill. The voxel grid is
the ground truth for both visuals and gameplay. SDFGI adds cinematic quality on top.

Entity light sources (held items, spell effects, auras) use Godot OmniLight3D nodes
and are separate from the block light grid.

---

## Design Rationale

### Voxel grid primary, SDFGI secondary

The block light grid is authoritative for all lighting. What the player sees matches
what gameplay systems check.

| Concern | Voxel grid (primary) | SDFGI (secondary) |
|---------|---------------------|-------------------|
| Role | Direct illumination, color, attenuation | Indirect bounce, ambient fill, color bleeding |
| Precision | Exact per-block RGB values | Probe-resolution (soft, approximate) |
| Update speed | Next frame (data texture upload) | 2-4 frames (probe re-evaluation) |
| Underground/caves | Perfect — flood fill works everywhere | Probes can struggle in narrow spaces |
| Gameplay accuracy | Grid value = what spawning/growth checks | Not used for gameplay |
| Failure tolerance | Load-bearing — must work | Enhancement — game works without it |

SDFGI picks up bright surfaces from the voxel-lit geometry and bounces light indirectly.
A torch on a wall lights the wall via the grid; SDFGI bounces warm light around the
corner into the next room. The result is visually rich with precise per-block control.

---

## Light Sources

### Skylight

Sunlight from above. Single-channel (0-15), not RGB.

- Full brightness (15) for blocks with direct vertical line to sky
- Attenuates by 1 per block passing through transparent blocks (water, glass, leaves)
- Stops completely at opaque blocks
- Time-of-day multiplier applied at render time (not stored in grid):

| Time | Skylight multiplier |
|------|-------------------|
| Noon | 1.0 |
| Dawn/Dusk | 0.4 |
| Night | 0.15 |
| Storm | 0.6 |

Skylight is recalculated per column when blocks are placed or broken. Not a BFS — a
simple top-down scan per affected column, fast.

### Block light

Emitted by specific block types. RGB channels, each 0-15. Defined in block YAML.

```yaml
# Block light emission examples
# core/blocks/torch.yaml
id: core:torch
light_emission: 14
light_color: [15, 12, 8]      # warm orange

# core/blocks/glowstone.yaml
id: core:glowstone
light_emission: 15
light_color: [15, 14, 10]     # bright warm white

# mymod/blocks/crystal_block.yaml
id: mymod:crystal_block
light_emission: 12
light_color: [8, 12, 15]      # cool blue

# core/blocks/lava.yaml
id: core:lava
light_emission: 15
light_color: [15, 8, 2]       # deep orange-red

# core/blocks/redstone_lamp.yaml
id: core:redstone_lamp
light_emission: 15
light_color: [15, 13, 10]     # warm white
# conditional: only emits when powered (state-dependent)

# mymod/blocks/neon_green.yaml
id: mymod:neon_green
light_emission: 10
light_color: [4, 15, 4]       # green glow
```

### Entity light sources

Players and NPCs emit light via Godot OmniLight3D nodes, not the block grid.
Entity lights are visual only — they do not modify the grid. A supplementary
gameplay check accounts for entity light when evaluating effective light level.

Sources of entity light:
- **Held items:** Torch in hand, crystal sword, glowing staff
- **Equipment:** Enchanted armor with glow effects
- **Status effects:** Flame aura, holy blessing, magic shield
- **Active abilities:** Spell casting, fire attacks
- **Passive effects:** Crystal attunement, lantern pet

```yaml
# Item with held light
# core/items/torch_item.yaml
id: core:torch_item
held_light:
  color: [1.0, 0.85, 0.6]
  energy: 1.2
  range: 8.0
  attenuation: 0.8

# Item with subtle glow
# mymod/items/crystal_sword.yaml
id: mymod:crystal_sword
held_light:
  color: [0.53, 0.8, 1.0]
  energy: 0.4
  range: 4.0
  attenuation: 1.0

# Status effect with light
# core/effects/flame_aura.yaml
id: core:flame_aura
light:
  color: [1.0, 0.5, 0.2]
  energy: 2.0
  range: 6.0
  flicker: true
  flicker_speed: 8.0
  flicker_intensity: 0.3       # energy varies ±30%
```

**Why OmniLight3D for entities, not the grid:**
- Entities move continuously — recalculating BFS flood fill every frame as a player
  walks is prohibitively expensive
- Entity count with light sources is small (players + nearby NPCs) — tens of lights,
  not thousands
- OmniLight3D provides smooth movement, correct distance falloff, and interacts with
  SDFGI naturally
- No mesh or texture update needed when entity moves

---

## Light Propagation — RGB Flood Fill

### Algorithm

Block light propagates via breadth-first flood fill from each light source. Each RGB
channel propagates independently, attenuating by 1 per block.

**Light placement:**

```
Torch placed at position P (emission: 14, color: [15, 12, 8]):
  1. Set light at P = [15, 12, 8]
  2. Add P to BFS queue
  3. For each position in queue:
     a. For each of 6 face-adjacent neighbors:
        - If neighbor is opaque: skip
        - Calculate propagated = [max(0, R-1), max(0, G-1), max(0, B-1)]
        - For each channel: if propagated > current at neighbor:
          - Update neighbor's channel
          - Add neighbor to queue (if not already queued)
  4. Collect set of affected chunks
  5. Re-upload light data textures for affected chunks
```

**Light removal:**

```
Torch broken at position P:
  1. BFS outward from P, collecting all positions where light must be recalculated
     (positions whose light came from P — tracked by comparing levels)
  2. Set collected positions to [0, 0, 0] (clear)
  3. Collect all remaining light sources that border the cleared region
  4. Re-propagate BFS from each bordering light source into the cleared region
  5. Re-upload affected light data textures
```

Light removal is the more expensive operation but is still fast in practice — it only
visits blocks that were lit by the removed source.

### Transparent block attenuation

Some blocks are transparent but attenuate light:

| Block type | Attenuation | Notes |
|-----------|-------------|-------|
| Air | 1 per block | Standard |
| Glass | 1 per block | Full transparency, standard falloff |
| Water | 2 per block | Light fades faster underwater |
| Stained glass | 1 per block + color filter | Multiplies light color by glass tint |
| Leaves | 2 per block | Partial occlusion |
| Ice | 1 per block | Transparent, no extra attenuation |

```yaml
# Block light transparency
# core/blocks/stained_glass_red.yaml
id: core:stained_glass_red
transparent: true
light_attenuation: 1
light_filter: [1.0, 0.2, 0.2]    # multiplies passing light by this color
```

Stained glass color filtering: when light passes through, each RGB channel is
multiplied by the filter value. White light through red glass becomes red light.
Blue light through red glass becomes very dim (blue × red filter ≈ 0).

### Propagation across chunk boundaries

Light propagation crosses chunk boundaries seamlessly. The BFS operates on world
coordinates, not chunk-local coordinates. When propagation enters a neighboring chunk,
that chunk's light data texture is marked dirty and re-uploaded after the BFS completes.

The 3D light data texture uses 1-block padding (18×18×18) to store light values from
neighboring chunks at the boundary. This allows the shader to interpolate smoothly
across chunk edges without visible seams.

### Performance

- A single light source at emission 15 affects at most 15³ ≈ 3,375 blocks (sphere)
- In practice, walls reduce this dramatically — a torch in a room touches 50-200 blocks
- Light updates only occur when light-emitting blocks are placed or broken (infrequent)
- BFS runs on a worker thread; results are applied to data textures on the main thread
- Multiple light changes in the same frame are batched into a single BFS pass
- Data texture upload is 23 KB per chunk — trivial GPU bandwidth

---

## Light Data Texture

Each chunk has a 3D texture storing light values, sampled by the block shader.

### Format

```
Dimensions: 18 × 18 × 18 (16 + 1-block padding on each side)
Format:     RGBA8 (4 bytes per texel)
Size:       18³ × 4 = 23,328 bytes (~23 KB per chunk)
```

| Channel | Data | Range |
|---------|------|-------|
| R | Block light red | 0-15 (stored as 0-255, scaled in shader) |
| G | Block light green | 0-15 |
| B | Block light blue | 0-15 |
| A | Skylight | 0-15 |

### Shader sampling

The chunk shader samples the 3D light texture at each fragment's world position:

```glsl
// Chunk fragment shader (simplified)
uniform sampler3D light_data;
uniform float sky_multiplier;    // time-of-day adjustment

void fragment() {
    // Sample light at fragment world position (normalized to chunk 0-1 range)
    vec4 light = texture(light_data, chunk_local_uv);

    // Block light (RGB)
    vec3 block_light = light.rgb * (1.0 / 15.0);

    // Skylight (A channel × time-of-day multiplier)
    float sky = light.a * (1.0 / 15.0) * sky_multiplier;

    // Final light is max of block light and skylight per channel
    // Skylight is white, so it contributes equally to all channels
    vec3 final_light = max(block_light, vec3(sky));

    // Apply minimum ambient (never fully black — even deep caves have faint visibility)
    final_light = max(final_light, vec3(0.02));

    // Modulate albedo by light
    ALBEDO = texture(block_texture, UV).rgb * final_light;
}
```

### Advantages over vertex colors

| Concern | Data texture | Vertex colors |
|---------|-------------|--------------|
| Light change cost | Re-upload 23 KB texture (no remesh) | Rebuild chunk mesh geometry |
| Interpolation | Hardware trilinear filtering (smooth gradients) | Per-face flat or per-vertex Gouraud |
| Mesh coupling | Decoupled — any mesh topology works | Tightly coupled — vertex count affects light resolution |
| Memory | 23 KB per chunk (fixed) | Varies with mesh complexity |
| Visual quality | Smooth light gradients across block faces | Visible faceting on large merged faces |

---

## Skylight Propagation

Skylight uses a simpler algorithm than block light — a top-down column scan rather
than 3D BFS.

### Algorithm

```
For each column (x, z) in the chunk:
  1. Start at the highest block in the column
  2. Scan downward:
     - If block is air: skylight = 15 (direct sky access)
     - If block is transparent: skylight = max(0, skylight - attenuation)
     - If block is opaque: skylight = 0 for all blocks below
  3. Horizontal skylight spread: BFS outward from sky-lit blocks into
     covered areas (overhangs, porches) with standard -1 attenuation
```

**Rebuild trigger:** When a block is placed or broken, only the affected column(s)
need skylight recalculation. Horizontal spread may touch neighboring columns.

---

## Gameplay Light Queries

### Block light level

Game systems query the light grid for gameplay decisions:

```lua
-- Get light level at a world position
Lighting.get_block_level(position)     -- returns 0-15 (max of RGB channels)
Lighting.get_block_color(position)     -- returns [R, G, B] (0-15 each)
Lighting.get_sky_level(position)       -- returns 0-15 (raw, before time multiplier)
Lighting.get_combined_level(position)  -- max(block, sky × time_multiplier)
```

### Effective light level (including entity lights)

Some gameplay checks should account for light from held items and entity effects:

```lua
-- Effective light level including nearby entity light sources
Lighting.effective_level(position)
```

Implementation:
1. Get grid light level at position
2. Query all entity OmniLight3D sources within range
3. For each entity light, calculate contribution at position using inverse-square falloff
4. Return max of grid level and entity contribution

This ensures:
- Holding a torch reduces hostile mob spawning near the player
- An NPC carrying a lantern illuminates their surroundings for gameplay purposes
- The grid is not modified — entity light is transient and moves with the entity

### Usage by game systems

| System | Light query used | Purpose |
|--------|-----------------|---------|
| Mob spawning | `effective_level` | Hostile mobs spawn in low light (< 7) |
| Crop growth | `get_sky_level` | Crops need skylight, not artificial light |
| Entity AI | `get_combined_level` | Creatures avoid/prefer lit areas |
| Block placement | `get_block_level` | Some blocks require minimum light (saplings) |
| Player HUD | `get_combined_level` | Light level display in debug overlay |

---

## Dynamic Lighting Effects

### Flicker

Light-emitting blocks and entity lights can flicker for atmospheric effect:

```yaml
# Block with flicker
id: core:torch
light_emission: 14
light_color: [15, 12, 8]
light_flicker:
  enabled: true
  speed: 6.0                # oscillation speed
  intensity: 0.15            # emission varies ±15%
  pattern: noise             # noise, sine, candle
```

**Block flicker is shader-side only.** The light grid stores the base emission value.
The shader adds per-frame variation using a noise function seeded by block position.
This means flicker is free — no grid recalculation, no texture re-upload.

**Entity flicker** (torch in hand, flame aura) modulates the OmniLight3D energy
property directly each frame.

### Conditional light emission

Some blocks emit light only when in a specific state:

```yaml
# Redstone lamp — only emits when powered
id: core:redstone_lamp
light_emission: 15
light_color: [15, 13, 10]
light_condition: powered       # only emits when block state "powered" is true

# Furnace — emits when smelting
id: core:furnace
light_emission: 10
light_color: [15, 10, 6]
light_condition: active
```

When a conditional block's state changes, the light grid runs a removal or placement
BFS for that source.

---

## Mod API

### Light queries

```lua
-- Read light values
Lighting.get_block_level(position)        -- 0-15 (max of RGB)
Lighting.get_block_color(position)        -- [R, G, B] each 0-15
Lighting.get_sky_level(position)          -- 0-15
Lighting.get_combined_level(position)     -- max(block, sky × time)
Lighting.effective_level(position)        -- combined + entity lights
```

### Custom light-emitting blocks

Modders define light emission in block YAML — no Lua required:

```yaml
id: mymod:aurora_crystal
light_emission: 12
light_color: [6, 15, 12]        # teal glow
light_flicker:
  enabled: true
  speed: 2.0
  intensity: 0.3
  pattern: sine                   # slow pulsing
```

### Entity light manipulation

```lua
-- Add/remove light sources on entities (via Lua API from progression doc)
player:add_light_source("crystal_glow", {
  color = { 0.53, 0.8, 1.0 },
  energy = 0.4,
  range = 4.0,
  attenuation = 1.0,
})
player:remove_light_source("crystal_glow")

-- Modify existing entity light
player:set_light_property("crystal_glow", "energy", 0.8)
player:set_light_property("crystal_glow", "range", 6.0)
```

### Custom light filter blocks

```lua
-- Register a block that filters light with custom color
-- (handled via YAML light_filter, but Lua can modify dynamically)
Block.set_light_filter(position, { 0.2, 1.0, 0.2 })   -- green filter
Block.clear_light_filter(position)
```

---

## Interaction with Other Systems

### Chunk meshing

Light data is decoupled from chunk mesh geometry. When light changes:
- Light data texture is re-uploaded (23 KB, fast)
- Chunk mesh is NOT rebuilt
- Visual update appears next frame

When blocks change (place/break):
- Chunk mesh IS rebuilt (geometry changed)
- Light propagation runs (BFS for placed/removed light sources)
- Light data texture updated
- Both happen, but independently

### SDFGI integration

SDFGI probes read the lit geometry as their input. When block light changes:
1. Data texture updates → chunk surfaces appear brighter/darker
2. SDFGI probes detect the brightness change on subsequent probe updates (2-4 frames)
3. Indirect bounce adjusts automatically
4. No explicit SDFGI notification needed — it reacts to what it sees

### Networking

Light propagation is deterministic. Clients run propagation locally when they
receive `BlockPlacedEvent` or `BlockBrokenEvent` — no light-specific network traffic.
Same principle as fluid simulation: deterministic systems replicate by running the
same algorithm on both sides.

### Performance budget

| Operation | Cost | Frequency |
|-----------|------|-----------|
| Single light placement BFS | 0.1-0.5ms | On block place (infrequent) |
| Single light removal + repropagate | 0.3-1.0ms | On block break (infrequent) |
| Data texture upload (per chunk) | ~0.05ms | After light change |
| Shader sampling (per fragment) | Negligible | Every frame |
| Skylight column rescan | ~0.01ms per column | On block change |
| Entity light query | ~0.02ms | Per gameplay check |
