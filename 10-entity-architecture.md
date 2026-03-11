# Entity Architecture

---

## Overview

Entities are everything in the world that isn't a block: creatures, NPCs, players,
dropped items, projectiles, and block entities (machines, containers). The entity system
uses a hybrid architecture: Godot scenes for rendering and physics, a component registry
for fast queries and game state, and a message bus for communication.

---

## Composition Model — Hybrid Node + Component

Each entity is a Godot scene (archetype template) that also registers its components
with a lightweight component registry. Godot handles what it's good at (rendering,
physics, scene tree). The component registry provides what Godot lacks (fast cross-entity
queries, bulk iteration, data-oriented game state).

### Archetype templates

Each archetype is a `.tscn` scene with pre-wired Godot nodes and default components.
Modders select an archetype in their entity YAML and optionally add custom components.

| Archetype | Base node | Default components |
|-----------|-----------|-------------------|
| `creature` | CharacterBody3D | Transform, Physics, Health, Mesh, Animation, AI, Navigation, Sound, LootTable |
| `npc` | CharacterBody3D | All of creature + NPC (dialogue, reputation, schedule, trade) |
| `player` | CharacterBody3D | Transform, Physics, Health, Mesh, Animation, Navigation, Sound, Inventory, Skills, Buffs |
| `item_drop` | RigidBody3D | Transform, Physics, Mesh, ItemStack |
| `projectile` | Area3D | Transform, Physics, Mesh, Damage |
| `block_entity` | StaticBody3D | Transform, Mesh, Inventory (slots), Tickable |

### Modder entity definition

```yaml
# mymod/entities/crystal_golem/crystal_golem.yaml

id: mymod:crystal_golem
display_name: Crystal Golem
archetype: creature

health: 80
speed: 3.0
mesh: entities/crystal_golem/crystal_golem.glb

# Additional components beyond the archetype defaults
components:
  - type: buff_emitter
    config:
      buff: mymod:crystal_shield
      radius: 5.0
  - type: block_breaker
    config:
      hardness_max: 3.0
      speed: 0.5

# Standard creature fields
animations:
  idle: Idle
  walk: Walk
  attack: Attack
  hurt: Hurt
  death: Death

sounds:
  idle: sounds/golem_idle.ogg
  attack: sounds/golem_slam.ogg
  hurt: sounds/golem_crack.ogg
  death: sounds/golem_shatter.ogg

loot_table:
  - item: mymod:crystal_shard
    min: 3
    max: 8
    chance: 1.0
  - item: mymod:resonite_fragment
    min: 1
    max: 2
    chance: 0.15

behavior_script: entities/crystal_golem/crystal_golem.lua

navigation:
  size: large
  can_jump: false
  can_swim: false
  can_climb: false
  can_fly: false
  max_fall: 2
  avoids: [lava, deep_water]

spawn_rules:
  biomes: [mymod:crystal_tundra]
  light_level_max: 8
  group_min: 1
  group_max: 2
```

---

## Component System

### Core components

| Component | Data / Engine | Contents | Godot node? |
|-----------|--------------|----------|-------------|
| **Transform** | Engine | Position, rotation, scale | Node3D (inherent) |
| **Physics** | Engine | Velocity, collision shape, gravity | CharacterBody3D / RigidBody3D |
| **Health** | Data + signals | Current HP, max HP, resistances, combat state timer | No |
| **Mesh** | Engine | 3D model, materials | MeshInstance3D |
| **Animation** | Engine | Current clip, blend tree, state machine | AnimationPlayer + AnimationTree |
| **AI** | Data + Lua | Behavior script, state machine, target | No |
| **Navigation** | Engine | Path, target, speed, size class | NavigationAgent3D-like |
| **Inventory** | Data | Slot array, stack sizes | No |
| **LootTable** | Data | Drop definitions | No |
| **NPC** | Data + Lua | Dialogue trees, reputation, schedule, trade | No |
| **Buffs** | Data | Active buff list, timers | No |
| **Skills** | Data | XP pools, unlocked nodes, cooldowns | No |
| **Sound** | Engine | Sound emitter references | AudioStreamPlayer3D |
| **SpawnRules** | Data | Biomes, light level, group size | No |
| **ItemStack** | Data | Item ID, count (for dropped items) | No |
| **Damage** | Data | Damage amount, type, source (for projectiles) | No |
| **Tickable** | Data | Tick rate, Lua script (for block entities) | No |
| **Ownership** | Data | Owner player ID, command mode | No |

Pattern: Godot nodes handle rendering and physics. Data components handle game state.
Systems tick the data components each frame.

### Custom components (mods)

Mods register new component types with a YAML definition and a Lua behavior script:

```yaml
# mymod/components/buff_emitter.yaml

id: mymod:buff_emitter
display_name: Buff Emitter
description: "Emits a status buff to nearby entities"

config_schema:
  buff:
    type: string
    required: true
  radius:
    type: float
    default: 5.0
  interval:
    type: float
    default: 2.0
  target_filter:
    type: string
    default: friendly    # friendly, hostile, all

behavior_script: components/buff_emitter.lua
```

```lua
-- components/buff_emitter.lua
local BuffEmitter = {}

function BuffEmitter.on_tick(entity, component, delta)
  component._timer = (component._timer or 0) + delta
  if component._timer < component.config.interval then return end
  component._timer = 0

  local nearby = World.get_entities_in_radius(
    entity:position(),
    component.config.radius
  )
  for _, target in ipairs(nearby) do
    if component.config.target_filter == "friendly"
       and target:is_friendly_to(entity) then
      target:add_buff(component.config.buff, {
        duration = component.config.interval * 1.5
      })
    end
  end
end

return BuffEmitter
```

### Component queries

The component registry supports fast queries for systems that iterate over entities
with specific component combinations:

```gdscript
# GDScript — engine systems use typed queries
func _process(delta: float) -> void:
    for entity in ComponentRegistry.query([Components.AI, Components.Health]):
        if entity.health.current > 0:
            ai_system.tick(entity, delta)

    for entity in ComponentRegistry.query([Components.Buffs]):
        buff_system.tick(entity, delta)
```

```lua
-- Lua — mods can query (read-only for other mods' entities)
local armed_npcs = ComponentQuery.find({ "health", "npc", "inventory" })
```

---

## Message Bus

All entity communication flows through a central message bus using a command/event
pattern. This architecture prepares for client/server networking — clients send commands,
the host validates and emits events, all systems react to events.

### Commands and events

**Commands** = intent. Not trusted. Validated by the host before producing events.

**Events** = facts. Trusted. Systems react to events to update state.

```
Player Input ──→ Command ──→ Host validates ──→ Event broadcast
AI Decision  ──→ Command ──→                    ├── Entity systems update state
Lua Script   ──→ Command ──→                    ├── Other clients receive & apply
Event Tree   ──→ Command ──→                    ├── Event tree triggers evaluate
                                                ├── Lua on_* callbacks fire
                                                └── Audio/VFX/Animation react
```

### Command validation

Commands carry intent but are never directly applied. The host/server validates
before producing events:

```
AttackCommand { attacker: player_3, target: entity_42, weapon: crystal_sword }
  → Host checks:
    - Is player_3 alive?
    - Is entity_42 in range of crystal_sword's reach?
    - Is attack cooldown elapsed?
    - Does player_3 actually have crystal_sword equipped?
  → Valid: emit DamagedEvent
  → Invalid: reject (optionally send denial to client)
```

### Event types

Events are registered in the registry like everything else. Core events shipped with
the engine:

```yaml
# core/events/events.yaml

events:
  # Entity lifecycle
  - id: core:entity_spawned
    fields: [entity_id, entity_type, position, chunk_id]

  - id: core:entity_died
    fields: [entity_id, killer_id, position, loot]

  - id: core:entity_despawned
    fields: [entity_id, reason]

  # Combat
  - id: core:entity_damaged
    fields: [entity_id, damage, damage_type, source_id, position]

  - id: core:entity_healed
    fields: [entity_id, amount, source]

  # World
  - id: core:block_broken
    fields: [position, block_id, player_id, tool_id]

  - id: core:block_placed
    fields: [position, block_id, player_id]

  # Interaction
  - id: core:entity_interacted
    fields: [player_id, entity_id]

  - id: core:item_picked_up
    fields: [player_id, item_id, count, position]

  - id: core:item_dropped
    fields: [player_id, item_id, count, position]

  # Player
  - id: core:player_entered_area
    fields: [player_id, marker_id]

  - id: core:player_entered_biome
    fields: [player_id, biome_id]

  - id: core:player_died
    fields: [player_id, killer_id, position]

  # State
  - id: core:flag_set
    fields: [key, scope, scope_id]

  - id: core:quest_started
    fields: [player_id, quest_id]

  - id: core:quest_completed
    fields: [player_id, quest_id]
```

Mods register new event types:

```lua
EventBus.register_event("mymod:crystal_resonance_pulse", {
  fields = { "source_position", "radius", "intensity" }
})
```

### Subscription

Systems and Lua scripts subscribe to events they care about:

```lua
-- Entity-scoped subscription (auto-unsubscribed on entity death)
function CrystalGolem.on_spawn(entity)
  entity:on("core:block_broken", function(event)
    if entity:distance_to(event.position) < 15.0 then
      entity:set_state("investigate")
      entity:set_target_position(event.position)
    end
  end)
end

-- Global subscription (mod-scoped, lives for mod lifetime)
EventBus.subscribe("core:entity_died", function(event)
  if event.entity_type == "core:zombie" then
    State.increment("zombies_killed", 1, "world")
  end
end)
```

### Client/server readiness

The bus naturally maps to a networked architecture:

| | Solo / Listen Server | Dedicated Server |
|---|---|---|
| Commands from | Local input + AI | Remote clients + AI |
| Validated by | Local host | Server |
| Events sent to | Local systems | All connected clients |
| Lua runs on | Host | Server only |
| Clients receive | Events directly | Events over network |

Systems that react to events (rather than directly mutating state) will work identically
in solo and multiplayer without modification.

---

## Entity Lifecycle

### Stage 1 — Registration

At mod load time, entity YAML is parsed and stored as a **prototype** in the
EntityRegistry. No Godot scenes are created. Prototypes are data-only blueprints.

```
Mod loads → YAML parsed → EntityRegistry.register("mymod:crystal_golem", prototype)
```

### Stage 2 — Spawning

All spawn requests go through the message bus:

```
SpawnCommand { entity_type, position, source, initial_data }
  → Host validates:
    - Is position in a loaded chunk?
    - Is entity budget for this chunk not exceeded?
    - Is position valid (not inside solid block)?
  → If valid:
    1. Instantiate archetype scene from prototype
    2. Attach components from prototype + any initial_data overrides
    3. Add to Godot scene tree
    4. Register all components in ComponentRegistry
    5. Assign unique entity ID (UUID)
    6. Associate entity with its chunk in ChunkEntityTracker
    7. Emit SpawnedEvent on bus
```

### Stage 3 — Initialization

Systems and Lua react to the SpawnedEvent:

```
SpawnedEvent received by:
  → AI system: initialize state machine, load behavior_script
  → Navigation system: register entity for pathfinding with size class
  → Lua runtime: call on_spawn(entity)
  → Animation system: play idle animation
  → Ownership system: if owned, apply current command mode
```

### Stage 4 — Active life

Each frame, engine systems tick entities with matching components:

| System | Ticks entities with | Action |
|--------|-------------------|--------|
| Physics | Physics | Godot move_and_slide / integrate |
| AI | AI + Health (alive only) | Call Lua on_tick(), evaluate state machine |
| Buff | Buffs | Decrement timers, remove expired, apply effects |
| Navigation | Navigation | Advance along path, request repath if blocked |
| Animation | Animation | Update blend trees from movement/state |
| NPC | NPC | Evaluate schedule, check dialogue conditions |
| Skill | Skills | Tick cooldowns, accumulate XP |
| Sound | Sound | Update 3D positional audio |
| Ownership | Ownership | Execute command mode behavior (follow, defend, patrol, stay) |
| Chunk tracker | Transform | Update chunk association if entity moved across boundary |
| Custom components | (per component) | Call component Lua on_tick() |

**Budget enforcement:** AI system tracks time spent per entity per mod. Scripts
exceeding budget are throttled (tick rate reduced) and flagged in the debug overlay.

**Entity event subscriptions:** Entities subscribe to bus events on spawn. Subscriptions
are automatically cleaned up on entity death/removal.

### Stage 5 — Dormancy

Entities in loaded chunks beyond a configurable distance from any player enter
**dormant state**:

| | Active | Dormant |
|---|---|---|
| AI ticks | Yes | **No** |
| Physics | Yes | **No** |
| Animation | Yes | **No** |
| Buff timers | Yes | **Paused** |
| Rendered | Yes | **No** |
| Event subscriptions | Active | **Suspended** |
| Persisted | Yes | Yes |

**Dormancy distance:** Configurable per world (default: 64 blocks from nearest player).
Named entities (NPCs, bosses, quest-flagged) and player-owned entities use a larger
dormancy distance (default: 128 blocks) to maintain better responsiveness.

**Wake-up:** When a player moves within active distance, dormant entities resume
immediately. Buff timers are fast-forwarded by elapsed dormancy time. AI state is
preserved — a dormant zombie in "chase" state resumes chasing when it wakes.

### Stage 6 — Chunk unload

When a chunk unloads (no player within load distance), all entities in that chunk
are serialized and saved. **All entities persist — no despawning.**

**Serialization:**

```yaml
# Chunk save data — entities section
# Stored in chunk file alongside block data
entities:
  - uuid: "ent_a7f3c2e8-9b41-4d2a-b8f0-1234567890ab"
    type: core:zombie
    position: [104.2, 65.0, -38.7]
    rotation: [0, 1.2, 0]
    health: { current: 12, max: 20 }
    ai_state: wander
    buffs:
      - { id: core:slow, remaining: 4.2 }
    custom_data: {}

  - uuid: "ent_b8e4d1f2-3c56-4e7b-a901-abcdef012345"
    type: core:village_elder
    position: [112.0, 65.0, -30.0]
    health: { current: 40, max: 40 }
    ai_state: schedule_day
    npc_data:
      reputation: { player_1: 25, player_3: -5 }
      dialogue_state: { village_crisis: accepted }
      trade_inventory: [core:crystal_shard, core:healing_potion]

  - uuid: "ent_c9f5e2a3-4d67-4f8c-b012-fedcba987654"
    type: mymod:crystal_deer
    position: [98.5, 66.0, -42.1]
    health: { current: 20, max: 20 }
    ai_state: wander
    owner: null
    custom_data: {}
```

**Serialization process:**
1. Entity systems pause ticking for the unloading chunk
2. Each entity's components are serialized to YAML-compatible data
3. Lua `on_save(entity)` callback called — mods can write custom_data
4. Entity data appended to chunk save file
5. Entity removed from scene tree and component registry
6. Godot scene freed

### Stage 7 — Chunk reload

When a chunk loads (player moves within load distance):

1. Block data loaded and meshed (standard chunk load)
2. Entity data read from chunk save file
3. For each saved entity:
   a. Look up prototype in EntityRegistry
   b. Instantiate archetype scene
   c. Attach components with saved state (not defaults)
   d. Restore health, AI state, buffs, inventory, custom data
   e. Add to scene tree and component registry
   f. Call Lua `on_restore(entity)` — mods can reinitialize from custom_data
   g. Emit SpawnedEvent (with `restored: true` flag)
4. Restored entities resume from saved state

**Player login chunk loading:** When a player logs in, chunks are loaded in concentric
rings outward from the player's position. The nearest chunks load first so the player
can enter the world immediately without waiting for distant chunks. Entity restoration
happens as each chunk loads — entities far from the player are dormant on load and
impose near-zero cost.

### Stage 8 — Death

```
Entity health reaches 0
  → DeathCommand emitted
  → Host validates (entity exists, is alive)
  → Lua on_death(entity, killer) called
  → Loot table evaluated, items dropped via SpawnCommand (item_drop entities)
  → DiedEvent emitted on bus
  → Event tree triggers checked (on_entity_death)
  → XP awarded to killer (if player)
  → Entity removed from component registry
  → Entity removed from chunk entity tracker
  → Entity save data removed from chunk
  → Godot scene freed
```

### Stage 9 — Cleanup

On entity removal (death, manual despawn, world unload):
- All bus subscriptions for this entity are unregistered
- All component data is freed from the registry
- Godot scene is `queue_free()`'d
- If the entity was owned by a player, the ownership record is updated

---

## Player-Owned Entities

Players can own entities (tamed animals, hired guards, summoned creatures). Owned
entities have an `Ownership` component with a command mode.

### Command modes

| Mode | Behavior |
|------|----------|
| **Follow** | Entity follows the owner, maintaining a configurable distance. Avoids combat unless attacked. |
| **Defend** | Entity follows the owner. Attacks any entity that damages the owner. Returns to owner after combat. |
| **Stay** | Entity remains at its current position. Idle behavior. Does not move or engage. |
| **Patrol** | Entity patrols a radius around its current position. Proactively attacks hostile mobs within that radius. |

### Command mode YAML

```yaml
# Ownership component configuration per entity type
ownership:
  tameable: true
  tame_item: core:raw_meat           # item used to tame
  tame_chance: 0.33                  # chance per attempt
  command_modes: [follow, defend, stay, patrol]
  patrol_radius: 16.0               # blocks
  follow_distance: 4.0              # blocks behind owner
  defend_aggro_radius: 12.0         # attacks hostiles within this range of owner
  leash_distance: 32.0              # max distance before teleporting to owner
```

### Ownership behavior

```lua
-- Engine-level ownership system (not mod Lua — runs as core system)
-- Simplified illustration of command mode logic

function OwnershipSystem.tick(entity, delta)
  local owner = entity:get_owner()
  if not owner or not owner:is_online() then
    -- Owner offline: default to stay behavior
    Behavior.idle(entity)
    return
  end

  local mode = entity:get_command_mode()
  local dist = entity:distance_to(owner)

  -- Leash: teleport to owner if too far
  if dist > entity.ownership.leash_distance then
    entity:teleport(owner:position())
    return
  end

  if mode == "follow" then
    if dist > entity.ownership.follow_distance then
      Behavior.follow(entity, owner, { speed = entity.walk_speed })
    else
      Behavior.idle(entity)
    end

  elseif mode == "defend" then
    -- Check if owner was recently damaged
    local attacker = owner:get_last_attacker()
    if attacker and attacker:is_alive() then
      Behavior.chase(entity, attacker, { speed = entity.run_speed })
      if entity:distance_to(attacker) < entity.attack_range then
        entity:attack(attacker)
      end
    elseif dist > entity.ownership.follow_distance then
      Behavior.follow(entity, owner, { speed = entity.walk_speed })
    else
      Behavior.idle(entity)
    end

  elseif mode == "stay" then
    Behavior.idle(entity)

  elseif mode == "patrol" then
    -- Attack nearby hostiles
    local hostile = entity:nearest_hostile(entity.ownership.defend_aggro_radius)
    if hostile then
      Behavior.chase(entity, hostile, { speed = entity.run_speed })
      if entity:distance_to(hostile) < entity.attack_range then
        entity:attack(hostile)
      end
    else
      Behavior.wander(entity, {
        origin = entity:get_patrol_origin(),
        radius = entity.ownership.patrol_radius,
        speed = entity.walk_speed,
      })
    end
  end
end
```

### Owned entity persistence

Player-owned entities persist in chunk data like all other entities. The Ownership
component saves owner player ID, current command mode, and patrol origin. When the
chunk reloads, ownership is restored.

If the owner is offline, owned entities default to "stay" behavior and are dormant
if outside active range of other players.

### Ownership Lua API

```lua
-- Taming
entity:tame(player)                          -- force tame (for scripts)
entity:is_tameable()                         -- check if entity type supports taming
entity:is_owned()                            -- check if owned by any player
entity:get_owner()                           -- returns owner player or nil

-- Commands
entity:set_command_mode(mode)                -- "follow", "defend", "stay", "patrol"
entity:get_command_mode()                    -- returns current mode
entity:set_patrol_origin(position)           -- set center of patrol area
entity:get_patrol_origin()                   -- returns patrol center

-- Player API
player:get_owned_entities()                  -- returns list of owned entities
player:get_owned_entities_in_radius(radius)  -- nearby owned entities
```

### Ownership UI

Player interacts with an owned entity to open a command radial menu:

```
         [Follow]
            |
[Stay] --- Pet --- [Defend]
            |
         [Patrol]
```

The radial menu is a core UI element. Modders can add custom command modes to specific
entity types, and they appear as additional options in the radial.

---

## Navigation — Layered Pathfinding

### Two-layer system

Entity pathfinding operates on two layers to balance performance and accuracy across
all terrain types including caves, overhangs, and indoor spaces.

**Layer 1 — Chunk connectivity graph (coarse):**
- Precomputed when a chunk is generated or modified
- Each chunk stores **portals**: walkable block positions on chunk boundaries connecting
  to adjacent chunks
- Portals linked into a graph of chunk-to-chunk connectivity
- A* on this graph produces a sequence of chunks for long-distance paths
- Rebuild cost: only the modified chunk + immediate neighbors

**Layer 2 — Block-level A* (fine):**
- Within a chunk or across 2-3 adjacent chunks
- Standard A* on the voxel grid
- Walkability check: block is solid, blocks above have clearance for entity size class
- Supports: jumping up 1 block, dropping down (configurable per entity), ladders,
  water swimming
- Path cache: frequently-used paths within a chunk cached until chunk is modified

### Pathfinding request flow

```
Entity wants to reach target 200 blocks away:
  1. Coarse: A* on chunk graph → [chunk_A, chunk_B, chunk_C, chunk_D]
  2. Fine: A* from entity position to portal leading to chunk_B
  3. Entity follows fine path
  4. On reaching chunk_B, compute next fine path to chunk_C portal
  5. Repeat until target reached or path fails
```

### Terrain change invalidation

```
Player breaks block at position P:
  → Chunk containing P marks nav data dirty
  → Adjacent chunks sharing boundary with P's chunk mark portals dirty
  → Worker thread: rebuild affected chunk portals
  → Active paths through affected chunks invalidated and re-requested
```

### Entity size classes

| Size | Clearance | Examples |
|------|-----------|---------|
| `small` | 1×1×1 | Rabbit, chicken, spider |
| `medium` | 1×2×1 | Player, zombie, deer, NPC |
| `large` | 2×2×2 | Crystal golem, horse, bear |
| `huge` | 3×3×3 | Boss mobs, dragons |

```yaml
# Entity navigation profile
navigation:
  size: medium
  can_jump: true
  jump_height: 1
  can_swim: true
  can_climb: false       # ladders, vines
  can_fly: false
  max_fall: 4            # blocks before entity avoids the drop
  avoids: [lava, cactus, deep_water]
```

### Lua pathfinding API

```lua
-- Async path request (callback fires when computed)
Navigation.request_path(entity, target_position, function(path)
  if path then
    entity:follow_path(path)
  else
    entity:set_state("blocked")
  end
end)

-- Synchronous checks
Navigation.is_reachable(entity, target_position)
Navigation.nearest_walkable(position, radius)
Navigation.distance_along_path(entity, target_position)  -- path distance, not euclidean
```

---

## Entity Budget

To prevent performance degradation, the engine enforces entity budgets.

| Budget | Default | Configurable |
|--------|---------|-------------|
| Max active entities per chunk | 32 | Yes (world settings) |
| Max total active entities | 512 | Yes (world settings) |
| Max dormant entities per chunk | 128 | Yes (world settings) |
| AI tick budget per entity | 0.1ms | Yes (per mod) |
| AI tick budget total per frame | 4ms | Yes (world settings) |

When budgets are reached:
- Spawn system stops spawning in affected chunks
- Farthest entities from any player are dormanted first
- Debug overlay shows budget usage per mod and per entity type
- No entities are deleted — only dormanted or spawn-blocked
