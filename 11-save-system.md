# Save System & Persistence

---

## Overview

The save system uses a hybrid storage model: YAML for human-readable configuration and
immutable world metadata, SQLite for bulk world data (chunks, entities, block entities).
Saves are prioritized by activity — chunks with active players save most frequently,
chunks with active named NPCs save semi-frequently, and inactive chunks save only on
unload.

---

## Storage Model

### YAML — Configuration and metadata

Human-readable files for data that defines the world at creation time or is useful for
inspection and editing outside the game.

| File | Contents | Mutable? |
|------|----------|----------|
| `world.yaml` | World name, seed, creation date, difficulty settings, world rules | Immutable after creation (settings editable via UI) |
| `mods.yaml` | Mod list with versions at world creation time | Append-only (new mods logged when added) |
| `state.yaml` | World-scope state store (flags, counters, variables) | Yes — written on autosave |
| `players/<uuid>.yaml` | Per-player data: position, inventory, skills, XP, buffs, quest state, player-scope state store | Yes — written on autosave and disconnect |

### SQLite — Bulk world data

A single `world.db` file stores all chunk-level data. SQLite provides atomic writes
(WAL mode), indexed lookups by chunk coordinate, and built-in corruption resistance.

**Tables:**

```sql
-- Block data: palette + RLE compressed binary blob per chunk
CREATE TABLE chunks (
    x INTEGER,
    y INTEGER,
    z INTEGER,
    block_data BLOB,         -- palette + RLE encoded block IDs
    generated_at INTEGER,    -- timestamp of initial generation
    last_modified INTEGER,   -- timestamp of last block change
    PRIMARY KEY (x, y, z)
);

-- Entity data: YAML text per chunk (all entities in that chunk)
CREATE TABLE chunk_entities (
    chunk_x INTEGER,
    chunk_y INTEGER,
    chunk_z INTEGER,
    entity_data TEXT,        -- YAML-formatted entity list
    last_modified INTEGER,
    PRIMARY KEY (chunk_x, chunk_y, chunk_z)
);

-- Block entity data: per-position persistent data for machines, containers, etc.
CREATE TABLE block_entities (
    x INTEGER,
    y INTEGER,
    z INTEGER,
    block_type TEXT,         -- registry ID (e.g. "mymod:alchemy_table")
    data TEXT,               -- YAML-formatted slot/state data
    last_modified INTEGER,
    PRIMARY KEY (x, y, z)
);

-- Navigation cache: precomputed chunk portal data
CREATE TABLE nav_portals (
    chunk_x INTEGER,
    chunk_y INTEGER,
    chunk_z INTEGER,
    portal_data BLOB,        -- binary portal connectivity
    last_modified INTEGER,
    PRIMARY KEY (chunk_x, chunk_y, chunk_z)
);
```

**WAL mode:** SQLite is opened in Write-Ahead Logging mode for crash safety. Writes
are atomic — a crash mid-save cannot corrupt the database.

### Block data encoding — Palette + RLE

Each chunk's block data is stored as a compact binary blob:

```
[palette_size: u16]
[palette_entry_0: string_id → local_index]
[palette_entry_1: string_id → local_index]
...
[rle_data: sequence of (local_index: u16, run_length: u16) pairs]
```

- **Palette:** Maps namespaced block IDs (e.g. `core:stone`) to small local indices
  for this chunk. A chunk with 5 block types has a 5-entry palette.
- **RLE:** Run-length encoded array of local indices in XZY order. Contiguous runs of
  the same block type are stored as a single (index, count) pair.
- **Compression:** A chunk that is entirely air is stored as a ~10 byte blob (1 palette
  entry, 1 RLE pair). A complex surface chunk with 20 block types typically compresses
  to 1-3 KB.

---

## Save Directory Structure

```
saves/
└── my_world/
    ├── world.yaml                ← immutable world metadata
    ├── mods.yaml                 ← mod manifest (versions at creation + additions)
    ├── state.yaml                ← world-scope state store
    ├── world.db                  ← SQLite: chunks, entities, block entities, nav cache
    ├── players/
    │   ├── a7f3c2e8-9b41.yaml   ← player data (UUID-named)
    │   └── b8e4d1f2-3c56.yaml
    └── backups/
        ├── 2026-03-10_14-30-00/  ← pre-session backup (full copy)
        ├── 2026-03-09_19-15-22/
        └── 2026-03-08_10-00-05/
```

---

## World Metadata

```yaml
# world.yaml — created once, immutable (except settings editable via UI)

name: "My World"
seed: 8472910364
created: "2026-03-10T14:30:00Z"
game_version: "1.2.0"
format_version: 1

settings:
  difficulty:
    death_penalty: buffs_only
    pvp_enabled: false
    mob_damage_multiplier: 1.0
    xp_multiplier: 1.0
    item_despawn_timer: 300

  world:
    render_distance: 12          # chunks
    entity_active_distance: 64   # blocks
    entity_dormant_distance: 128 # blocks for named entities
    autosave_interval: 300       # seconds
    max_entities_per_chunk: 32
    max_total_active_entities: 512

  multiplayer:
    max_players: 8
    challenge_scaling: true
    scaling_per_player: 0.7
```

---

## Mod Manifest

```yaml
# mods.yaml — records mod state at world creation and any changes

created_with:
  - { id: core, version: "1.0.0" }
  - { id: crystal_creatures, version: "1.2.0" }
  - { id: autumn_forest_pack, version: "1.0.0" }

history:
  - date: "2026-03-10T14:30:00Z"
    action: world_created
    mods: [core@1.0.0, crystal_creatures@1.2.0, autumn_forest_pack@1.0.0]

  - date: "2026-03-15T09:00:00Z"
    action: mod_added
    mod: { id: magic_system, version: "0.5.0" }

  - date: "2026-03-20T18:30:00Z"
    action: mod_updated
    mod: { id: crystal_creatures, version: "1.3.0", from: "1.2.0" }
```

---

## Player Data

```yaml
# players/a7f3c2e8-9b41.yaml

uuid: "a7f3c2e8-9b41-4d2a-b8f0-1234567890ab"
display_name: "Alex"
last_login: "2026-03-10T16:45:00Z"
play_time: 14520    # seconds

position: [104.2, 65.0, -38.7]
rotation: [0, 1.2, 0]
dimension: core:overworld
spawn_point: [0, 68, 0]

health:
  current: 85
  max: 100

inventory:
  hotbar:
    - { slot: 0, item: core:crystal_sword, count: 1, durability: 380 }
    - { slot: 1, item: core:iron_pickaxe, count: 1, durability: 200 }
    - { slot: 2, item: core:cooked_steak, count: 32 }
    - { slot: 3, item: null }
    # ... slots 4-8
  main:
    - { slot: 0, item: core:oak_log, count: 64 }
    - { slot: 1, item: core:iron_ore, count: 28 }
    # ... up to 27 slots
  armor:
    head: { item: core:iron_helmet, durability: 150 }
    chest: { item: core:iron_chestplate, durability: 200 }
    legs: { item: core:iron_leggings, durability: 175 }
    feet: { item: core:iron_boots, durability: 140 }

skills:
  combat:
    xp: 4520
    level: 12
    points_available: 2
    unlocked:
      - { node: core:combat.steady_hand, rank: 3 }
      - { node: core:combat.thick_skin, rank: 2 }
      - { node: core:combat.whirlwind, rank: 1 }
  harvesting:
    xp: 8200
    level: 18
    points_available: 0
    unlocked:
      - { node: core:harvesting.prospector, rank: 2 }
      - { node: core:harvesting.lumberjack, rank: 3 }
  # ... other categories

active_buffs:
  - { id: core:well_fed, source: core:hunters_feast, remaining: 245.0,
      effects: { health_regen: 4.0, max_health: 20, damage: 3.0 } }
  - { id: core:refreshed, source: core:crystal_tea, remaining: 180.0,
      effects: { magic_regen: 3.0, magic_power: 0.15 } }

active_cooldowns:
  - { skill: core:combat.whirlwind, remaining: 4.2 }

quests:
  active:
    - id: core:clear_crystal_caves
      started: "2026-03-10T15:30:00Z"
      objectives:
        enter_caves: true
        cave_goblins_killed: 7
        goblin_chief_defeated: false
        reported_caves_clear: false
  completed:
    - { id: core:tutorial_basics, completed: "2026-03-10T14:45:00Z" }

state:
  village_crisis_started: true
  refused_elder: false
  reputation:
    village_elder: 25

owned_entities:
  - { uuid: "ent_wolf_01", type: core:wolf, name: "Shadow" }
  - { uuid: "ent_wolf_02", type: core:wolf, name: "Fang" }
```

---

## Chunk Save Priority

Chunks are saved based on activity level. Not all loaded chunks need saving at the
same frequency.

### Priority tiers

| Tier | Condition | Save frequency |
|------|-----------|---------------|
| **Hot** | Chunk has active player(s) | Every autosave interval (default 5 min) |
| **Warm** | No players, but named entities actively changing state since last save | Every 2× autosave interval (default 10 min) |
| **Cold** | Loaded, no activity since last save | On unload only |
| **Frozen** | Not loaded | Never (already on disk) |

### Dirty tracking

Chunks flag themselves dirty when state changes occur:

```
Block placed/broken       → chunk.dirty = true, chunk.last_modified = now
Entity state changed      → chunk.dirty = true, chunk.last_modified = now
BlockEntity state changed → chunk.dirty = true, chunk.last_modified = now
Named NPC performs action  → chunk.dirty = true, chunk.last_modified = now
```

**"Actively changing state"** means the chunk has received a write event since its last
save. A named NPC standing idle does not make the chunk warm. A named NPC completing a
trade, modifying inventory, or advancing an event tree does.

Unchanged chunks (dirty = false) are never re-serialized regardless of tier.

### Save tick

```
On autosave tick (every autosave_interval):
  1. Save all hot dirty chunks → clear dirty flag
  2. Save warm dirty chunks past their interval → clear dirty flag
  3. Skip cold chunks
  4. Save all player data (always, regardless of changes)
  5. Save world-scope state store if dirty

On chunk unload:
  → Save chunk immediately regardless of tier/dirty state

On player disconnect:
  → Save player data immediately
  → Save chunk player was in (if dirty)

On manual save / world quit:
  → Full flush: all loaded chunks, all player data, all state
  → SQLite checkpoint (flush WAL to main database file)

On world open (pre-session):
  → Create timestamped backup before any writes
```

---

## Chunk Loading Strategy

### Player login

When a player logs in, chunks load in concentric rings outward from the player's
saved position:

```
Ring 0: chunk containing player (immediate — blocks login until ready)
Ring 1: 8 surrounding chunks (high priority)
Ring 2: 16 surrounding chunks (medium priority)
Ring 3+: remaining chunks up to render distance (background, streamed)
```

The player can enter the world as soon as Ring 0 is loaded and meshed. Surrounding
chunks stream in progressively. Entity restoration happens per-chunk as each loads —
distant entities are dormant on load and impose near-zero cost.

### Chunk load/unload distance

```
                    ┌─────────────────────┐
                    │  Render distance     │  Chunks loaded, entities active
                    │  (12 chunks default) │  within entity_active_distance
                    │                      │
                    │    ┌───────────┐     │
                    │    │  Active   │     │  Entities fully ticking
                    │    │  (64 blk) │     │
                    │    │  Player   │     │
                    │    └───────────┘     │
                    │                      │
                    │  Dormant zone        │  Entities loaded but dormant
                    │  (64-128 blk named)  │
                    └─────────────────────┘
                       Unload boundary      Chunks serialized and freed
                    (render distance + 2)
```

Chunks are unloaded when they exceed render distance + 2 chunks from all players
(hysteresis prevents rapid load/unload at the boundary).

---

## Backup System

### Pre-session backups

Every time a world is opened, the save system creates a complete copy of the save
directory before any new writes occur. Backups are timestamped.

```
saves/my_world/backups/
├── 2026-03-10_14-30-00/    ← most recent
│   ├── world.yaml
│   ├── mods.yaml
│   ├── state.yaml
│   ├── world.db
│   └── players/
├── 2026-03-09_19-15-22/    ← previous session
└── 2026-03-08_10-00-05/    ← oldest kept
```

**Retention:** 3 most recent pre-session backups are kept. Oldest is deleted when a
new backup is created. Count is configurable in world settings.

**Recovery UI:** The world selection screen shows available backups for each world.
Player selects a timestamped backup to restore. The current save is replaced with the
backup contents. The replaced save becomes a new backup entry (so the restore itself
is reversible).

### SQLite integrity

- WAL mode ensures atomic writes — a crash mid-save cannot corrupt the database
- On world open, `PRAGMA integrity_check` runs against `world.db`
- If integrity check fails, player is prompted to restore from a backup

### Chunk checksums

Each chunk stores a CRC32 checksum of its block data. On load, the checksum is verified.
If a chunk fails validation:

1. Log the corruption with chunk coordinates
2. Notify the player: "Chunk at (X, Z) was corrupted and has been regenerated"
3. Regenerate the chunk from world seed (terrain, ores, structures, decorators)
4. Entity and BlockEntity data for that chunk is lost (logged as warning)

This is a last resort — between SQLite WAL and pre-session backups, chunk corruption
should be extremely rare.

---

## Mod Data Persistence

### Mod data in saves

Mod-defined content persists through the standard systems:

- **Mod blocks** — stored as namespaced IDs in chunk palettes (`mymod:crystal_ore`)
- **Mod entities** — stored in chunk entity data with full type and state
- **Mod BlockEntities** — stored in block_entities table with type and state
- **Mod state store entries** — stored in `state.yaml` or player files
- **Mod quest progress** — stored in player files
- **Mod skill trees / XP** — stored in player files under their category

### Mod removal — dedicated tool

Removing a mod from a world with existing save data is a destructive operation that
cannot be safely automated in all cases. A dedicated CLI tool handles mod removal
with explicit user confirmation.

```bash
voxelite-mod-remove ./saves/my_world --mod crystal_creatures

Analyzing world save for mod "crystal_creatures"...

Found references:
  Chunks:
    - 847 chunks contain blocks from crystal_creatures
    - 12,340 block instances (crystal_ore, crystal_grass, crystal_dirt, crystal_stone)
  Entities:
    - 23 crystal_deer instances across 15 chunks
    - 4 crystal_golem instances across 3 chunks
    - 1 crystal_tower structure (3 chunks)
  Block entities:
    - 0 block entities from crystal_creatures
  Player data:
    - Player "Alex": 3 inventory items, 1 active quest, 2 state flags
    - Player "Sam": 1 inventory item
  State store:
    - 4 world-scope flags referencing crystal_creatures

Actions that will be taken:
  - Replace crystal_creatures blocks with core:stone (subsurface) or core:air (surface)
  - Remove crystal_creatures entities from chunk data
  - Remove crystal_creatures items from player inventories
  - Abandon crystal_creatures quests in player data
  - Remove crystal_creatures state flags
  - Remove crystal_creatures from mods.yaml

WARNING: This operation is irreversible. A backup will be created before proceeding.

Proceed? [y/N]
```

**Limitations:** The tool handles data removal but cannot undo gameplay consequences.
If a mod's quest chain unlocked a recipe or changed world state that other systems
depend on, the world may be in an inconsistent narrative state. The tool logs all
changes so the player understands what was affected.

**Not all removals are safe.** The tool warns when:
- A mod provided a dimension that players may have built in
- A mod provided a custom terrain generator used by existing biomes
- Other mods declare the removed mod as a dependency
- The mod provided system-level hooks that other mods rely on

In these cases the tool refuses to proceed and explains why.

### Mod addition to existing world

Adding a mod to an existing world is non-destructive:
- New blocks, items, entities are available immediately
- New biomes appear in newly-generated chunks only (existing chunks unchanged)
- New entity spawn rules take effect in loaded chunks
- The addition is logged in `mods.yaml` history

### Mod version updates

Mods can provide migration scripts for version transitions:

```yaml
# mod.yaml
migrations:
  "1.2.0 → 1.3.0":
    renames:
      blocks:
        mymod:crystal_rock: mymod:crystal_stone    # block ID renamed
      items:
        mymod:crystal_gem: mymod:crystal_shard      # item ID renamed
    script: migrations/1_2_to_1_3.lua               # complex migrations
```

```lua
-- migrations/1_2_to_1_3.lua
-- Runs once on world load when mod version changes

function migrate(world)
  -- Convert old alchemy table data format to new format
  local tables = world:find_block_entities("mymod:alchemy_table")
  for _, be in ipairs(tables) do
    if be.data.fuel then
      -- v1.3 removed fuel requirement
      be.data.fuel = nil
      be:save()
    end
  end
end
```

Migrations run after mod loading but before the world becomes playable. The mod
version in `mods.yaml` is updated after successful migration.

---

## Lua Save/Load API

### Entity save hooks

```lua
-- Called when entity is serialized (chunk unload or autosave)
function CrystalGolem.on_save(entity)
  return {
    rage_level = entity._rage_level,
    home_position = entity._home_pos,
    kills_since_spawn = entity._kills,
  }
end

-- Called when entity is restored from save
function CrystalGolem.on_restore(entity, saved_data)
  entity._rage_level = saved_data.rage_level or 0
  entity._home_pos = saved_data.home_position
  entity._kills = saved_data.kills_since_spawn or 0
end
```

### State store persistence

The state store API (defined in `07-event-system.md`) persists automatically:

```lua
-- These persist to state.yaml or player files without explicit save calls
State.set_flag("village_crisis_started", "world")
State.set("reputation.elder", 25, "player")
State.increment("goblins_killed", 1, "player")

-- All state is loaded on world open and saved on autosave/quit
```

### World save events

```lua
-- Mods can hook into save events for custom persistence
Events.register("on_world_save", function()
  -- Flush any mod-specific cached state
end)

Events.register("on_world_load", function()
  -- Initialize mod state from restored data
end)
```
