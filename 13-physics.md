# Physics System

---

## Overview

Physics uses Godot's Jolt Physics engine for all solid-body simulation (character
movement, dropped items, projectiles, collision). Fluid simulation (water, lava) is
a custom cellular automaton system operating on the block grid, independent of Jolt.

Collision shapes are generated per-chunk from the block grid with block-accurate
fidelity — separate from the greedy-meshed render geometry.

The fluid system is designed as a pluggable interface. The launch implementation is
an enhanced cellular automaton (Option A). The interface supports replacement with a
pressure-based simulation (Option B) as a future addition, selectable per-world at
creation time.

---

## Collision System

### Block-accurate collision (separate from render mesh)

Render meshes use greedy meshing to minimize draw calls. Collision geometry is generated
independently with exact block-level fidelity.

**Why separate:** Players expect to stand on exact block edges, jump through 1-block
gaps, and collide with precise block boundaries. Greedy-merged collision faces create
subtle edge-catching bugs where entities snag on invisible seams between merged faces.

**Generation:**
1. For each chunk, iterate all solid blocks
2. For each exposed face (adjacent to air, fluid, or transparent block), emit a face
3. Merge coplanar adjacent faces into rectangular collision quads (simpler merge than
   greedy meshing — only merges faces on the same plane with the same collision properties)
4. Build a `ConcavePolygonShape3D` or compound `BoxShape3D` set from the merged faces
5. Assign to a `StaticBody3D` per chunk

**Optimization:** Only exposed faces generate collision. A solid 16×16×16 chunk of stone
has zero interior collision faces — only the outer shell. A chunk that is entirely air
has no collision body at all.

**Rebuild:** When a block is placed or broken, only the affected chunk's collision shape
is regenerated. Adjacent chunks are checked for newly exposed/hidden faces at the
boundary.

### Collision layers

| Layer | Contains | Collides with |
|-------|----------|--------------|
| 1 — Terrain | Chunk collision bodies | Entities, items, projectiles |
| 2 — Entities | Players, creatures, NPCs | Terrain, entities, projectiles |
| 3 — Items | Dropped item RigidBody3D | Terrain only |
| 4 — Projectiles | Arrows, fireballs | Terrain, entities |
| 5 — Triggers | Area3D zones (markers, AoE) | Entities only |
| 6 — Fluid | Water/lava surface colliders | None (query only — used for swim detection) |

### Block collision properties

Not all blocks have the same collision behavior:

```yaml
# Block collision types (defined per block in block YAML)
collision:
  solid: true          # default — full block collision
  # or:
  solid: false         # no collision (air, flowers, tall grass)
  # or:
  shape: slab_bottom   # half-height collision (bottom slab)
  shape: slab_top      # half-height collision (top slab)
  shape: stair         # stepped collision shape
  shape: fence         # 1.5-block-tall thin collision
  shape: custom        # custom collision mesh from GLB

  # Properties
  friction: 0.6        # surface friction (ice = 0.05, default = 0.6)
  bounce: 0.0          # restitution (slime block = 0.8)
```

Custom collision shapes are defined in the block's GLB file as a secondary mesh
named `collision`. The engine extracts this mesh at import time and uses it as the
collision shape for that block type.

---

## Character Physics

### Player movement

Players use `CharacterBody3D` with `move_and_slide()`.

**Parameters:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| Walk speed | 4.3 m/s | Comparable to Minecraft's 4.317 m/s |
| Sprint speed | 5.6 m/s | ~30% faster, consumes stamina |
| Crouch speed | 1.3 m/s | ~30% of walk speed |
| Jump velocity | 7.0 m/s | Tuned to clear 1.25 blocks |
| Gravity | 24.0 m/s² | Tuned for snappy voxel-scale feel (not real-world 9.8) |
| Terminal velocity | 50.0 m/s | Cap on fall speed |
| Max slope angle | 45° | Slides on steeper surfaces |
| Step height | 0.55 blocks | Auto-step up half-block edges without jumping |
| Swim speed | 2.5 m/s | Reduced in water |
| Climb speed | 3.0 m/s | On ladders/vines |

**Responsive controls:**
- **Coyote time:** 100ms grace period after walking off an edge where jump still works
- **Jump buffering:** If jump is pressed within 100ms before landing, jump executes on land
- **Variable jump height:** Hold jump = full height, tap jump = lower arc (gravity multiplier
  increases when ascending with jump released)
- **Air control:** Reduced but nonzero lateral control while airborne (30% of ground control)

### Slope handling

```
≤ 45°: Walk normally, speed reduced proportionally to incline
> 45°: Slide downward, cannot walk up
  90° (vertical wall): No grip, fall
```

Step height (0.55 blocks) means entities walk up single-block ledges smoothly without
needing to jump. This is critical for natural movement on blocky terrain.

### Fall damage

```
Fall distance < 3 blocks:    No damage
Fall distance 3-20 blocks:   (distance - 3) × 2 damage
Fall distance > 20 blocks:   Lethal (unless mitigated)

Mitigated by:
  - Land Softly skill (25%/50%/75% reduction per rank)
  - Feather Fall buff/enchantment
  - Water landing (no fall damage)
  - Slime block landing (bounces, no damage)
  - Hay bale landing (80% reduction)
```

### Entity movement

Creatures and NPCs use `CharacterBody3D` with the same `move_and_slide()` pipeline.
Speed, step height, and slope limits are configured per entity type in the navigation
profile.

**Entity-specific physics:**

| Entity type | Physics body | Notes |
|-------------|-------------|-------|
| Player | CharacterBody3D | Full movement suite |
| Creature/NPC | CharacterBody3D | AI-driven, navigation profile controls capabilities |
| Dropped item | RigidBody3D | Bounces, rolls, settles. Lock to sleep after settling |
| Projectile | Area3D + raycast | No rigid body — moves via code, checks collision via raycast per frame |
| Block entity | StaticBody3D | No movement — static collision only |

### Dropped item physics

Items dropped by players or from entity death use `RigidBody3D`:

- Initial velocity: tossed in player's look direction (or random scatter on death)
- Bounce off terrain and settle via Jolt physics
- After settling (velocity near zero for 1 second), body enters sleep mode (zero CPU)
- Item pickup radius: 1.5 blocks. Player walks near, item is collected
- Item magnet: items within 3 blocks of a player are slowly pulled toward them
- Despawn timer: configured per world (default: 300 seconds), disabled if world
  death penalty is `keep_all`

---

## Projectile Physics

Projectiles use raycasting rather than rigid bodies for performance and precision.

### Arrow / bolt

```
Per frame:
  1. Calculate next position from velocity + gravity
  2. Raycast from current position to next position
  3. If hit terrain: embed arrow (visual), trigger BlockHitEvent
  4. If hit entity: apply damage, trigger DamagedEvent
  5. If no hit: update position, continue

Properties:
  speed: 40.0 m/s (initial)
  gravity: 24.0 m/s² (same as player — arcs naturally)
  drag: 0.01 (slight slowdown over distance)
  damage_falloff: 0.8 (damage multiplier at max range)
  max_range: 80 blocks (despawn beyond this)
```

### Thrown items (potions, grenades)

Same raycast system but with:
- Lower initial speed (20 m/s)
- Higher gravity arc
- AoE effect on impact (splash radius)
- Particle trail during flight

### Homing projectiles (magic)

```
Per frame:
  1. Calculate direction to target
  2. Rotate current velocity toward target (turn rate per frame)
  3. Raycast forward
  4. Standard hit detection

Turn rate determines tracking strength — slow turn = easily dodged
```

---

## Fluid System

### Architecture — Pluggable interface

The fluid system is a registered system that ticks fluid block state. The interface
is implementation-agnostic:

```gdscript
# FluidSystem interface
class_name FluidSystem

# Called every fluid tick for each loaded chunk with fluid blocks
func tick_chunk(chunk: ChunkData, delta: float) -> Array[BlockChange]:
    # Returns list of block state changes
    # Engine applies changes to world and notifies clients
    pass

# Called when a block adjacent to fluid is placed or broken
func on_neighbor_changed(position: Vector3i, chunk: ChunkData) -> Array[BlockChange]:
    pass
```

Both the enhanced cellular automaton and a future pressure-based system implement
this interface. The world selects which implementation to use at creation time.

```yaml
# World creation setting
physics:
  fluid_model: simple         # "simple" = enhanced cellular automaton (default)
  # fluid_model: pressure     # future: pressure-based simulation
  fluid_tick_rate: 4          # fluid updates per second (independent of game tick)
```

### Client-side fluid simulation

Fluid simulation is **deterministic**: same block state + same neighbors = same flow
result, every time. Clients run the fluid simulation locally without waiting for
server updates.

```
Block broken adjacent to water:
  → Server validates block break, emits BlockBrokenEvent
  → Server runs fluid tick for affected area
  → Client receives BlockBrokenEvent
  → Client runs identical fluid tick locally
  → Both arrive at the same fluid state (deterministic)
  → No fluid-specific network traffic needed
```

This follows the same principle as seed-based chunk streaming: deterministic systems
replicate by running the same algorithm on both sides rather than transmitting results.

**Edge case — desync recovery:** If a client's fluid state ever diverges from the
server (bug, packet loss during block update), the next chunk delta sync corrects it.
The server's chunk state is always authoritative — the client just doesn't need to
wait for it under normal operation.

### Block state for fluids

Each fluid block stores:

```yaml
# Fluid block state
block_type: core:water        # or core:lava, or mod-registered fluid
fluid_level: 7                # 0 (nearly empty) to 7 (full/source)
is_source: true               # source blocks don't drain
flow_direction: [0, 0, -1]    # cached dominant flow direction (for rendering)
```

### Enhanced cellular automaton (launch implementation)

The base fluid simulation with level equalization for enclosed spaces.

**Core rules:**

```
1. Source blocks (level 7, is_source = true):
   - Never drain
   - Spread to adjacent air/fluid blocks

2. Downward flow (highest priority):
   - Fluid above air → fill air block below (level 7)
   - Fluid always falls until hitting solid ground

3. Horizontal spread:
   - Fluid spreads to adjacent air blocks at (current_level - 1)
   - Maximum spread distance from source: 7 blocks (water), 3 blocks (lava)
   - Prefers spreading toward nearby edges (drop detection — looks ahead N blocks
     for a drop and preferentially flows that direction)

4. Level equalization (enhancement over basic Minecraft):
   - In an enclosed space, fluid finds its own level
   - Connected fluid blocks equalize: if two adjacent fluid blocks have different
     levels and no downward exit, level averages between them
   - This means filling a U-shaped channel results in equal water level on both
     sides, not Minecraft's infinite-source quirks

5. Drain:
   - Non-source fluid blocks with no adjacent higher-level fluid drain by 1 level
     per fluid tick
   - Level 0 → air (block removed)

6. Source generation:
   - Two adjacent source blocks flowing into the same air block create a new source
     (infinite water — standard mechanic, configurable per fluid type)
   - Lava does NOT generate new sources (default)
```

**Tick rate:** Fluid updates run at a separate rate from the game tick (default: 4
updates/second). Water flows faster than lava:

| Fluid | Spread per tick | Ticks between updates |
|-------|----------------|----------------------|
| Water | 1 block | 1 (every fluid tick) |
| Lava | 1 block | 3 (every 3rd fluid tick) |

**Performance:** Only chunks containing fluid blocks are ticked. A chunk with no fluid
blocks has zero fluid simulation cost. Active fluid (currently flowing/changing) is
tracked — once fluid settles (no state changes for 2 ticks), the chunk is removed from
the fluid tick list until a neighbor change reactivates it.

### Fluid rendering

Water and lava are rendered with custom shaders, not standard block meshes:

**Water:**
- Semi-transparent surface mesh generated from fluid levels (not full-block faces)
- Surface height interpolated from fluid levels of adjacent blocks for smooth surface
- Animated UV scrolling for flow direction
- Fresnel-based transparency (more opaque at glancing angles)
- Tinted by biome water color from atmosphere settings
- Underwater fog with caustic light patterns
- Refraction distortion of blocks seen through water

**Lava:**
- Opaque emissive surface
- Animated flow texture
- Emission glow (contributes to block lighting)
- Heat shimmer particle effect above surface
- Ignites flammable blocks within 2 blocks (fire spread system)

### Fluid interaction with entities

```
Entity enters water block:
  → Physics: gravity reduced to 25%, drag increased
  → Movement: swim speed replaces walk speed
  → State: entity.in_water = true
  → Sound: splash SFX, underwater ambient
  → Camera: underwater tint and fog (if player)
  → Oxygen: timer starts (player only, from progression doc)

Entity enters lava block:
  → Damage: continuous fire damage (configurable per entity — fire immune mobs ignore)
  → State: entity.on_fire = true (persists for 5 seconds after leaving lava)
  → Movement: speed reduced to 50%
  → Visual: fire particle effect on entity

Fluid pushes entities:
  → Entities in flowing (non-source) water are pushed in flow_direction
  → Push strength proportional to fluid_level
  → Swim input can overcome weak currents, not strong ones
```

### Water source placement

Players can place and break water/lava source blocks:

- **Bucket item:** Right-click water source → pick up (block becomes air, bucket becomes
  water bucket). Right-click with water bucket → place source block.
- **Infinite water:** Two source blocks flowing into the same position create a third
  source. Standard voxel game mechanic for player convenience.
- **Lava:** Not infinitely renewable (default). Bucketable but scarce.

---

## Future: Pressure-Based Fluid Simulation (Option B)

The pressure-based system is a planned upgrade that replaces the cellular automaton
with a more physically accurate model. It implements the same `FluidSystem` interface
and is selectable at world creation.

### Design notes for future implementation

**Pressure model:**
- Each fluid block stores pressure value in addition to level
- Pressure propagates through connected fluid blocks
- Fluid flows from high pressure to low pressure
- Enclosed columns of water exert downward pressure (hydrostatic)
- U-bends work: water in a sealed U-shaped tunnel equalizes height on both sides
  through pressure, not just adjacent level averaging
- Siphons work: a full pipe over a wall drains to the lower side

**Key differences from enhanced cellular automaton:**

| Behavior | Cellular automaton | Pressure-based |
|----------|--------------------|---------------|
| U-bend equalization | Approximate (adjacent averaging) | Correct (pressure propagation) |
| Siphons | Do not work | Work correctly |
| Sealed pressurized rooms | Water sits at fill level | Water exerts pressure on walls |
| Upward flow in sealed pipes | Not possible | Possible (pressure-driven) |
| CPU cost per fluid block | Very low | Moderate |
| Simulation radius | Full chunk | May need distance limiting |

**Performance considerations:**
- Pressure propagation is iterative — may need multiple iterations per tick to converge
- Simulation radius around players may need to be smaller than chunk load distance
- Fluid tick rate may need to be lower (2/sec vs 4/sec)
- Worker thread recommended for pressure calculations

**Interface compatibility:**
- Same `FluidSystem` interface: `tick_chunk()` and `on_neighbor_changed()`
- Same block state: `fluid_level` + `is_source` (adds `pressure: float`)
- Same rendering: surface mesh from fluid levels (unchanged)
- Same entity interaction: swim detection, push, damage (unchanged)
- Same client-side determinism: pressure algorithm is pure function of block state

**World creation toggle:**

```yaml
physics:
  fluid_model: pressure
  fluid_tick_rate: 2             # lower rate for heavier simulation
  pressure_iterations: 4         # convergence iterations per tick
  pressure_sim_radius: 64       # blocks from nearest player
```

Worlds created with `simple` fluid model can be converted to `pressure` model —
existing fluid block states are valid input for either system. The pressure system
will re-evaluate and settle fluid over several ticks after conversion.

---

## Fire Spread

Lava and fire blocks can ignite flammable blocks. Fire is a block state, not an entity.

```yaml
# Block flammability (defined in block YAML)
flammable: true
burn_chance: 0.3          # chance to catch fire per fire tick when adjacent to fire
burn_time: 100            # ticks before block is destroyed by fire
fire_spread_chance: 0.1   # chance to spread fire to adjacent flammable blocks
```

**Fire tick:** Runs at 1 tick/second. For each fire block:
1. Decrement burn timer on the burning block
2. If timer reaches 0, block is destroyed (replaced with air)
3. Check adjacent blocks for flammability
4. Roll spread chance — if passed, adjacent block catches fire
5. Fire blocks with no adjacent flammable blocks extinguish after 3 seconds

**Rain:** Extinguishes exposed fire blocks. Weather system sets a flag per chunk;
fire tick checks the flag before spreading.

**Client-side:** Fire spread is deterministic (seeded RNG from block position + tick
count). Clients simulate locally, same as fluids.

---

## Physics Lua API

```lua
-- Raycast
Physics.raycast(origin, direction, max_distance, collision_mask)
  -- Returns: { hit: bool, position: vec3, normal: vec3, block: block_id or nil,
  --            entity: entity_id or nil }

-- Area queries
Physics.overlap_sphere(center, radius, collision_mask)
  -- Returns: list of { entity_id, position }

Physics.overlap_box(center, half_extents, collision_mask)

-- Fluid queries
Fluid.get_level(position)           -- fluid level at position (0-7, 0 if no fluid)
Fluid.get_type(position)            -- "water", "lava", or nil
Fluid.get_flow_direction(position)  -- vec3 flow direction
Fluid.is_source(position)           -- bool

-- Fluid manipulation (mod API)
Fluid.place_source(position, fluid_type)    -- place a source block
Fluid.remove_source(position)               -- remove a source (fluid drains)

-- Custom fluid registration (mods)
Fluid.register("mymod:acid", {
  display_name = "Acid",
  spread_distance = 5,
  tick_interval = 2,
  generates_sources = false,
  damage = { type = "poison", amount = 4, interval = 0.5 },
  color = "#44ff22",
  opacity = 0.8,
  shader = "res://mods/mymod/shaders/acid.gdshader",   -- optional custom shader
})
```

---

## Interaction with Other Systems

### Navigation rebuild

When blocks change (placed, broken, fluid flow), the navigation system is notified:

```
BlockChangedEvent → Navigation system
  → Mark affected chunk nav data dirty
  → Rebuild portals on worker thread
  → Invalidate active paths through affected area
```

Fluid blocks are treated as passable for swimming entities and impassable for
non-swimming entities in the navigation system.

### Collision rebuild

```
BlockChangedEvent → Collision system
  → Regenerate collision shape for affected chunk
  → Update StaticBody3D shape
  → Adjacent chunk boundary faces rechecked
```

### Sound

- Block collision: material-specific footstep sounds (wood, stone, grass, sand, metal)
- Fluid: splash on enter/exit, underwater ambient, flow sound near moving water
- Impact: falling item sounds based on item weight and surface material
- Physics-driven sounds are positional (3D audio via AudioStreamPlayer3D)
