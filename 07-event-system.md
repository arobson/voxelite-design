# Event & Narrative System

---

## Overview

The event system drives all narrative content: quests, branching dialogue, world-state
mutations, and NPC relationships. It is fully data-driven — event trees are YAML files,
complex conditions and effects can defer to Lua scripts. The base game's quest content
uses this system identically to how modders use it.

A companion visual editor (Tauri desktop app) reads and writes the same YAML format.
Hand-editing and visual editing round-trip cleanly — the YAML is the source of truth.

---

## Core Concepts

### Event Trees

An event tree is a directed graph of **nodes** stored in a single YAML file. Each node
is a dialogue beat, condition gate, effect trigger, or branch point. Trees are small and
focused — a multi-stage quest is split across multiple tree files connected by `chain:`
references.

### Markers

Named positions or regions in the world that event trees reference instead of raw
coordinates. Markers are defined in structure metadata, placed by world generation, or
created at runtime by effects. Event YAML always references locations by marker name.

```yaml
# Defined in structure metadata
markers:
  village_gate:   { offset: [12, 0, 3] }
  elder_house:    { offset: [5, 0, 8] }
  bridge_site:    { offset: [-20, 0, 0], radius: 10 }
  goblin_camp:    { offset: [80, 0, -40], radius: 20 }
```

### State Store

Persistent key-value store that event trees read and write. All conditions and effects
operate against this store.

**Scopes:**

| Scope | Persisted with | Example |
|-------|---------------|---------|
| `world` | World save | `world.village_destroyed = true` |
| `player` | Player save | `player.reputation.village_elder = 45` |
| `entity` | Entity data | `entity.times_talked_to = 3` |
| `block` | BlockEntity data | `block.alchemy_table.progress = 0.7` |

**Operations (Lua API):**

```lua
State.set_flag(key, scope)             -- set boolean true
State.clear_flag(key, scope)           -- set boolean false
State.get_flag(key, scope)             -- returns boolean
State.set(key, value, scope)           -- arbitrary value
State.get(key, scope)                  -- returns value or nil
State.increment(key, amount, scope)    -- numeric increment
```

---

## Event Tree YAML Format

### File structure

One tree per file. File path determines registry ID via standard namespacing
(`core:village_crisis.discovery` → `core/quests/village_crisis/discovery.yaml`).

```yaml
# core/quests/village_crisis/discovery.yaml

id: core:village_crisis.discovery
display_name: "Village in Peril"

trigger:
  event: on_player_enter_structure
  structure: core:village
  conditions:
    - flag_not_set: village_crisis_started

nodes:
  start:
    type: dialogue
    npc: core:village_elder
    text: "Strangers rarely come here. We need help..."
    choices:
      - text: "What's wrong?"
        next: explain
      - text: "Not my problem."
        next: refuse

  explain:
    type: dialogue
    npc: core:village_elder
    text: "Creatures from the crystal caves raid us nightly."
    choices:
      - text: "I'll clear the caves."
        next: accept
      - text: "What's in it for me?"
        next: negotiate

  negotiate:
    type: dialogue
    npc: core:village_elder
    text: "We can offer you our finest crystal blade."
    conditions:
      - variable_gte: { key: player.reputation.elder, value: 10 }
    choices:
      - text: "Deal."
        next: accept
      - text: "Not enough."
        next: refuse

  accept:
    type: quest_start
    quest: core:clear_crystal_caves
    effects:
      - set_flag: village_crisis_started
      - journal_entry: "The village elder asked me to clear the crystal caves."
      - notification: "New Quest: Village in Peril"
    chain: core:village_crisis.caves

  refuse:
    type: effects_only
    effects:
      - set_flag: refused_elder
      - increment: { key: player.reputation.elder, amount: -5, scope: player }
```

### Node types

| Type | Purpose |
|------|---------|
| `dialogue` | NPC speaks, player may choose a response |
| `narration` | Text shown to player with no NPC (journal, inner monologue) |
| `condition_gate` | Branching based on state conditions — no player choice |
| `effects_only` | Execute effects and continue (or end) |
| `quest_start` | Begin tracking a quest objective |
| `quest_complete` | Mark a quest as finished, grant rewards |
| `wait` | Pause tree until a condition becomes true (async) |

### Condition types (YAML, declarative)

```yaml
conditions:
  - flag_set: village_crisis_started
  - flag_not_set: betrayed_elder
  - counter_gte: { key: goblins_killed, value: 10 }
  - counter_lte: { key: player.deaths, value: 3 }
  - variable_gte: { key: player.reputation.elder, value: 20 }
  - variable_range: { key: player.reputation.elder, min: 10, max: 50 }
  - has_item: { item: core:crystal_shard, count: 5 }
  - lacks_item: { item: core:goblin_trophy }
  - in_biome: core:forest
  - in_structure: core:village
  - time_of_day: [night]
  - weather: [rain, storm]
  - quest_active: core:clear_crystal_caves
  - quest_complete: core:clear_crystal_caves
  - distance_to_marker: { marker: goblin_camp, max: 50 }
  - entity_alive: { entity: marker:goblin_chief }
  - custom_script: quests/village_crisis_special_check.lua
```

All standard conditions are evaluated by the engine. `custom_script` defers to a Lua
function that returns `true`/`false` for cases the declarative system cannot express.

### Effect types

```yaml
effects:
  # State mutations
  - set_flag: village_saved
  - clear_flag: goblin_threat_active
  - increment: { key: elder_reputation, amount: 10, scope: player }
  - set_variable: { key: village_state, value: "rebuilt", scope: world }

  # World mutations
  - spawn_entity: { entity: core:village_guard, at: marker:village_gate }
  - remove_entity: { at: marker:goblin_camp_leader }
  - place_structure: { structure: core:rebuilt_bridge, at: marker:bridge_site }
  - destroy_blocks: { at: marker:old_bridge, radius: 10 }
  - set_biome_override: { area: marker:corrupted_zone, biome: mymod:corrupted_forest }

  # Player effects
  - give_item: { item: core:crystal_sword, count: 1 }
  - remove_item: { item: core:goblin_key, count: 1 }
  - add_status_effect: { effect: core:blessing, duration: 600 }
  - unlock_recipe: { recipe: core:crystal_armor }
  - teleport_player: { to: marker:cave_entrance }

  # Narrative
  - journal_entry: "The village is safe."
  - notification: "Quest Complete: Village in Peril"
  - play_sound: { sound: core:quest_complete }
  - camera_event: { type: pan_to, target: marker:rebuilt_bridge, duration: 3.0 }

  # Flow control
  - trigger_tree: core:village_crisis.aftermath
  - delay: { seconds: 5, then: core:village_crisis.celebration }
  - custom_script: quests/village_crisis_special_effect.lua
```

---

## Chain References

Trees connect to other trees via `chain:` on any node. When a node with `chain:` is
reached, the current tree ends and the referenced tree begins.

```
discovery.yaml ──chain──→ caves.yaml ──chain──→ aftermath.yaml
                                          └──chain──→ betrayal.yaml (alt path)
```

In the visual editor, chain references appear as labeled exit ports on the source node
and entry ports on the target tree. The editor can navigate between linked trees.

Modders can extend existing quest chains by adding new branches:

```yaml
# mymod/quests/overrides/village_crisis_discovery.yaml

override: core:village_crisis.discovery

nodes:
  negotiate:
    choices:
      append:
        - text: "I know a secret way in. Let me show you."
          conditions:
            - flag_set: mymod:knows_secret_path
          next: mymod_secret_path

  mymod_secret_path:
    type: effects_only
    effects:
      - set_flag: mymod:using_secret_path
      - journal_entry: "I offered to show the elder the hidden cave entrance."
    chain: mymod:village_crisis.secret_route
```

---

## NPC Entity Subtype

NPCs are entities registered in the entity registry with an `npc:` block that connects
them to the event system.

### NPC entity definition

```yaml
# core/entities/village_elder/village_elder.yaml

id: core:village_elder
display_name: Village Elder
category: npc
health: 40
speed: 3.0
mesh: entities/village_elder/village_elder.glb
invulnerable: true    # quest NPCs cannot be killed by default

animations:
  idle:  Idle
  walk:  Walk
  talk:  Talk
  wave:  Wave

sounds:
  greeting: sounds/elder_greeting.ogg
  farewell: sounds/elder_farewell.ogg

behavior_script: entities/village_elder/village_elder.lua

npc:
  portrait: textures/portraits/village_elder.png

  dialogue_trees:
    - tree: core:village_crisis.discovery
      priority: 10
      conditions:
        - flag_not_set: village_crisis_started
    - tree: core:village_crisis.progress_check
      priority: 5
      conditions:
        - quest_active: core:clear_crystal_caves
    - tree: core:village_elder.idle_chat
      priority: 0    # fallback when no other tree matches

  relationship:
    track: true
    initial_reputation: 0
    key: player.reputation.village_elder    # state store key

  schedule:
    day:   { location: marker:village_square, behavior: wander, radius: 8 }
    night: { location: marker:elder_house,    behavior: idle }

  trade:
    enabled: true
    inventory_pool: core:village_elder_trades
```

### NPC interaction flow

1. Player presses `interact` while looking at NPC
2. Engine checks NPC's `dialogue_trees` list in priority order
3. First tree whose `conditions` all pass is activated
4. Tree nodes execute (dialogue, effects, state changes)
5. If no tree matches, NPC plays a generic idle line

### NPC relationship

The `relationship:` block opts the NPC into reputation tracking. Reputation is stored in
the state store under the declared key. Event trees can check and modify reputation via
standard conditions and effects. Reputation affects:

- Which dialogue trees are available (gated by `variable_gte` conditions)
- Trade prices (configurable per NPC via Lua)
- NPC behavior toward the player (friendly, neutral, hostile thresholds)

### NPC Lua API

```lua
-- Available in entity behavior scripts and event custom_scripts
npc:get_reputation(player)
npc:set_reputation(player, value)
npc:modify_reputation(player, delta)
npc:get_current_dialogue_tree()
npc:get_schedule_state()         -- "day" or "night" (or custom)
npc:force_dialogue(tree_id, player)
npc:set_trade_inventory(pool_id)
```

---

## Quest Tracking

Quests are a thin layer on top of the event/state system. A quest is not a separate data
type — it is a set of conventions applied to event trees and state flags.

### Quest definition

```yaml
# core/quests/clear_crystal_caves.yaml

id: core:clear_crystal_caves
display_name: "Clear the Crystal Caves"
description: "Eliminate the creatures raiding the village from the crystal caves."
icon: textures/quests/crystal_caves.png
category: main    # main, side, misc

objectives:
  - id: enter_caves
    text: "Enter the crystal caves"
    type: flag
    flag: entered_crystal_caves

  - id: kill_goblins
    text: "Defeat cave goblins (0/10)"
    type: counter
    key: cave_goblins_killed
    target: 10

  - id: defeat_chief
    text: "Defeat the Goblin Chief"
    type: flag
    flag: goblin_chief_defeated

  - id: return_to_elder
    text: "Return to the Village Elder"
    type: flag
    flag: reported_caves_clear

rewards:
  - give_item: { item: core:crystal_sword, count: 1 }
  - give_item: { item: core:gold_coin, count: 50 }
  - increment: { key: player.reputation.village_elder, amount: 20, scope: player }
  - unlock_recipe: { recipe: core:crystal_armor }
```

Objectives are checked against the state store. The HUD quest tracker reads objective
definitions and renders progress automatically. When all objectives are complete, the
quest is marked completable — the final event tree node handles the `quest_complete`
effect and reward distribution.

---

## Trigger System

Event trees activate in response to triggers. Triggers are registered at mod load time
based on the `trigger:` block in each tree file.

### Trigger types

| Trigger | Fires when |
|---------|-----------|
| `on_player_enter_structure` | Player enters a structure's bounding area |
| `on_player_enter_biome` | Player enters a new biome |
| `on_player_enter_marker` | Player enters a marker's radius |
| `on_interact_entity` | Player interacts with a specific entity type |
| `on_interact_block` | Player interacts with a specific block type |
| `on_item_pickup` | Player picks up a specific item |
| `on_entity_death` | A specific entity type dies |
| `on_block_place` | A specific block type is placed |
| `on_block_break` | A specific block type is broken |
| `on_quest_complete` | A specific quest is completed |
| `on_flag_set` | A specific state flag becomes true |
| `on_time_of_day` | In-game time reaches a threshold |
| `on_day_change` | In-game day boundary |
| `manual` | Only triggered by `trigger_tree` effect or Lua call |

All triggers support a `conditions:` block for additional gating.

---

## Modder Extension Points

Modders interact with the event system at three levels:

1. **YAML only** — write event tree files, define quests, set up NPC dialogue. No Lua
   required for straightforward branching narratives.

2. **YAML + Lua conditions/effects** — use `custom_script:` for conditions or effects
   that the declarative system cannot express.

3. **Lua system hooks** — register custom trigger types, add new condition evaluators,
   or build entirely scripted narrative systems on top of the state store.

```lua
-- Register a custom trigger type
Events.register_trigger("mymod:on_full_moon", function(callback)
  Events.register("on_day_change", function()
    if World.get_moon_phase() == "full" then
      callback()
    end
  end)
end)
```
