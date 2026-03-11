# Player Progression & Survival Mechanics

---

## Design Philosophy: All Carrot, No Stick

The base game never punishes players for lacking something. Systems reward players with
bonuses for engaging with mechanics — better food, higher skills, stronger gear — but
the absence of these is baseline, not a penalty.

- No starvation. No dehydration. No debuffs for ignoring a system.
- Consumables provide stat bonuses, not relief from punishment.
- Death costs your active buff stack, not your inventory.
- Progression makes you stronger. Nothing makes you weaker over time.

This philosophy must be maintained in all base game systems. Modders may implement
punitive mechanics for hardcore difficulty modes via Lua.

---

## Health & Regeneration

### Base health
- Default max health: 100 HP (20 hearts)
- Gear, buffs, and skills can increase max health

### Regeneration model

| Regen source | Rate | During combat? |
|-------------|------|----------------|
| Passive regen | Slow (1 HP/sec base) | **Paused** |
| Food buff | Moderate (varies by food quality) | **Active** |
| Drink buff | Moderate (varies) | **Active** |
| Potion | Fast (varies by potion tier) | **Active** |
| Spell | Varies (instant or over time) | **Active** |
| Skill bonus | Modifier on other sources | Inherits source |

**Combat state:** Player enters combat when they deal or receive damage. Combat state
ends after a configurable timeout with no damage events (default: 5 seconds). Passive
regen resumes when combat state ends.

**Buff stacking:** Multiple regen sources stack additively. A player with a meal buff,
a potion, and a regen enchantment receives all three simultaneously. This rewards
preparation — the player who brews potions, cooks complex meals, and enchants their
gear enters a fight with a significant sustain advantage.

---

## Consumable Buff System

### Food

Eating provides timed stat buffs. No hunger bar. Players eat when they want a boost.

```yaml
# core/items/cooked_steak.yaml
id: core:cooked_steak
category: food
stack_size: 64
icon: textures/items/cooked_steak.png

consumable:
  duration: 300      # 5 minutes
  buffs:
    health_regen: 2.0    # HP per second (active in combat)
    max_health: 10       # +10 max HP while active
    damage: 1.5          # +1.5 melee damage
```

**Food complexity:** Simple foods (roasted meat, baked potato) give one or two small
buffs. Complex recipes with multiple ingredients give compound bonuses:

```yaml
# core/items/hunters_feast.yaml
id: core:hunters_feast
category: food
stack_size: 16

consumable:
  duration: 600
  buffs:
    health_regen: 4.0
    max_health: 20
    damage: 3.0
    move_speed: 0.05
    combat_xp_bonus: 0.10    # +10% combat XP while active
```

### Drinks

Same system as food. Both a food buff and a drink buff can be active simultaneously
(they are separate buff slots).

```yaml
# core/items/crystal_tea.yaml
id: core:crystal_tea
category: drink
stack_size: 32

consumable:
  duration: 480
  buffs:
    magic_regen: 3.0
    magic_power: 0.15       # +15% spell effectiveness
    magic_xp_bonus: 0.10
```

### Potions

Instant or short-duration effects. Stronger than food/drink but shorter and more
expensive to craft.

```yaml
# core/items/healing_potion.yaml
id: core:healing_potion
category: potion
stack_size: 16

consumable:
  instant:
    heal: 40               # instant HP restore
  duration: 30
  buffs:
    health_regen: 8.0      # strong but short
```

### Buff slots

| Slot | Source | Limit |
|------|--------|-------|
| Food | Eating food items | 1 active food buff |
| Drink | Drinking beverages | 1 active drink buff |
| Potion | Drinking potions | 1 active potion buff (new potion replaces) |
| Status effects | Spells, skills, environment | Multiple, stacking by type |
| Equipment | Worn gear enchantments | Always active while equipped |

---

## Death

### Default behavior (configurable per world)

1. Player dies
2. All active buffs are cleared (food, drink, potion, status effects)
3. Inventory is **kept**
4. Player respawns at bed (or world spawn if no bed set)
5. Short respawn invulnerability (3 seconds)

### World difficulty overrides

```yaml
# World creation settings — exposed in world config UI
difficulty:
  death_penalty: buffs_only     # default
  # Options:
  #   buffs_only    — lose active buffs (default)
  #   drop_items    — drop inventory, items persist on ground with despawn timer
  #   drop_and_xp   — drop inventory + lose percentage of unspent XP
  #   keep_all      — no penalty (creative/story mode)
  #   permadeath    — character is deleted (hardcore)

  item_despawn_timer: 300       # seconds, only relevant for drop_items/drop_and_xp
  pvp_enabled: false
  mob_damage_multiplier: 1.0
  xp_multiplier: 1.0
```

Modders can register custom death penalty modes via Lua.

---

## XP & Skill System

### XP Categories

XP is earned contextually by performing activities. Each category has its own XP pool
and its own skill tree.

| Category | XP earned by | Example activities |
|----------|-------------|-------------------|
| **Physical** | Movement, athletics | Running, jumping, climbing, swimming, falling |
| **Magic** | Using magic | Casting spells, using magic items, enchanting |
| **Crafting** | Creating items | Crafting at any station, smelting, brewing, cooking |
| **Harvesting** | Gathering resources | Mining ore, chopping trees, farming, foraging, fishing |
| **Combat** | Fighting | Killing mobs, hunting animals, PvP, taking damage |
| **Engineering** | Building systems | Placing machines, wiring, building automation, complex structures |

**XP scaling:** Early levels require little XP. Later levels require progressively more.
There is no level cap — players who invest more time continue to progress. Diminishing
returns on XP per-action prevent trivial grinding (killing the same weak mob 10,000
times gives less XP per kill over time for that mob type).

**XP is not a spendable currency.** XP accumulates in each category. When thresholds are
reached, skill points are awarded in that category. Skill points are spent to unlock
nodes in the corresponding skill tree.

### Skill Trees

Each XP category has a skill tree containing passive bonuses and active abilities.
Trees are defined in YAML. Modders can add new trees or append nodes to existing trees.

**Tree structure:** Nodes are arranged in tiers. Higher tiers require minimum category
level and prerequisite nodes. Within a tier, players choose which nodes to unlock —
but given enough time, all nodes are reachable. No forced specialization. No respec.

```yaml
# core/skills/combat_tree.yaml

id: core:combat
display_name: Combat
icon: textures/skills/combat.png
xp_category: combat

tiers:
  - tier: 1
    level_required: 1
    nodes:
      - id: core:combat.steady_hand
        type: passive
        display_name: Steady Hand
        description: "Increases melee damage."
        icon: textures/skills/steady_hand.png
        cost: 1
        ranks:
          - { damage_bonus: 0.05 }         # +5% melee damage
          - { damage_bonus: 0.10 }
          - { damage_bonus: 0.15 }

      - id: core:combat.thick_skin
        type: passive
        display_name: Thick Skin
        description: "Reduces physical damage taken."
        icon: textures/skills/thick_skin.png
        cost: 1
        ranks:
          - { physical_resistance: 0.05 }
          - { physical_resistance: 0.10 }
          - { physical_resistance: 0.15 }

  - tier: 2
    level_required: 5
    nodes:
      - id: core:combat.whirlwind
        type: active
        display_name: Whirlwind Slash
        description: "Spin attack hitting all enemies within range."
        icon: textures/skills/whirlwind.png
        cost: 2
        requires: [core:combat.steady_hand]
        cooldown: 12.0
        ranks:
          - { damage: 15, radius: 3.0 }
          - { damage: 22, radius: 3.5 }
          - { damage: 30, radius: 4.0, knockback: 2.0 }
        behavior_script: skills/combat/whirlwind.lua

      - id: core:combat.battle_regen
        type: passive
        display_name: Battle Regeneration
        description: "Regenerate health slowly during combat."
        icon: textures/skills/battle_regen.png
        cost: 2
        requires: [core:combat.thick_skin]
        ranks:
          - { combat_regen: 0.5 }     # HP/sec during combat
          - { combat_regen: 1.0 }
          - { combat_regen: 1.5 }

  - tier: 3
    level_required: 15
    nodes:
      - id: core:combat.executioner
        type: passive
        display_name: Executioner
        description: "Bonus damage to low-health targets."
        icon: textures/skills/executioner.png
        cost: 3
        requires: [core:combat.whirlwind]
        ranks:
          - { bonus_damage_below_25pct: 0.15 }
          - { bonus_damage_below_25pct: 0.25 }
```

### Skill node types

| Type | Description | Example |
|------|-------------|---------|
| `passive` | Always active once unlocked | +10% melee damage, +5% move speed |
| `active` | Placed on hotbar, used with a key | Whirlwind slash, magic shield, ground slam |
| `recipe_unlock` | Unlocks a crafting recipe | Crystal armor requires Crafting tier 3 |
| `ability_upgrade` | Modifies an existing active ability | Whirlwind now applies slow effect |

### Skill ranks

Most nodes have multiple ranks (levels). Each rank costs additional skill points and
provides a stronger version of the bonus. The node definition specifies the bonus per
rank as an array.

### Skill requirements as barrier to entry

Skills gate access to specific activities:

```yaml
# An item that requires a minimum skill level to craft
# core/items/crystal_armor.yaml

id: core:crystal_armor_chestplate
category: armor

crafting_requirements:
  skill: core:crafting
  level: 15
  node_required: core:crafting.crystal_smithing   # must have this node unlocked

# Higher skill levels improve outcome
crafting_bonuses:
  level_20: { durability_bonus: 0.10 }     # +10% durability
  level_30: { durability_bonus: 0.20, chance_bonus_enchant: 0.15 }
```

**Barrier to entry:** Some recipes, workstations, and activities require a minimum
category level or a specific unlocked node. A level 1 crafter cannot forge crystal
armor — they need to progress the crafting tree first.

**Skill-scaled bonuses:** Beyond the barrier, higher levels improve outcomes. A level 30
crafter forging the same crystal armor gets bonus durability and a chance at a free
enchantment. A level 50 harvester mining the same ore vein gets bonus yield. This
rewards continued investment without locking content behind extreme grind walls.

### Base game skill trees

| Tree | Example passives | Example actives |
|------|-----------------|-----------------|
| **Physical** | Land Softly (fall dmg, noise), Swift Stride (speed), Deep Breath (swim time), Climber (climb speed) | Dodge Roll, Sprint Burst, Power Jump |
| **Magic** | Mana Pool (max mana), Spell Focus (cast speed), Crystal Attunement (magic dmg) | Magic Shield, Blink (short teleport), Mana Surge |
| **Crafting** | Efficient Crafter (chance not to consume materials), Quality Touch (bonus stats), Batch Production (craft multiples) | Masterwork (guaranteed quality bonus on next craft) |
| **Harvesting** | Prospector (ore highlight), Lumberjack (wood yield), Green Thumb (crop yield), Lucky Strike (rare drops) | Vein Miner (mine connected ore blocks), Harvest Sweep (AoE gather) |
| **Combat** | Steady Hand (damage), Thick Skin (resistance), Battle Regen, Executioner, Weapon Mastery per type | Whirlwind Slash, Shield Bash, Power Shot (ranged), War Cry (AoE buff) |
| **Engineering** | Efficient Wiring (reduced power loss), Overclocked (machine speed), Structural Integrity (build range) | Turret Deploy, Drone Scout, Remote Trigger |

---

## Combat System

### Basic combat

- Real-time first-person melee and ranged
- Left click: primary attack (weapon swing / bow draw)
- Right click: secondary (shield block / weapon special)
- Attack cooldown per weapon (`attack_speed` stat) — no spam clicking
- Directional knockback based on hit direction
- Hit feedback: enemy flash, sound, particle, screen shake (subtle)

### Combat depth (moderate)

- **Shield blocking:** Hold right click with shield equipped. Reduces incoming damage
  by shield's `block_value`. Stamina cost per blocked hit. Shield can break if durability
  runs out.
- **Dodge roll:** Active ability unlocked in Physical tree. Quick directional roll with
  brief invulnerability frames. Stamina cost. Cooldown.
- **Weapon combos:** 2-3 hit chains with different animations and damage values. Final
  hit in chain deals bonus damage. Interrupting the chain resets it.
- **Ranged combat:** Bows with draw time affecting damage. Crossbows with fixed damage
  but reload time. Throwables (potions, grenades via engineering).

### Damage types

| Type | Sources | Resistance from |
|------|---------|----------------|
| `physical` | Melee weapons, arrows, falling, mob melee | Armor, skills |
| `magic` | Spells, crystal weapons, enchantments | Magic resistance gear, skills |
| `fire` | Lava, fire enchantments, fire mobs | Fire resistance enchant, potions |
| `ice` | Frost spells, cold biome hazards | Cold resistance gear |
| `poison` | Spider bites, poison potions, swamp gas | Antidote potion, skills |
| `void` | Falling out of world, endgame hazards | Void-touched armor only |

Modders add new damage types to the registry. Armor and status effects provide
resistance multipliers per damage type.

### Stamina (lightweight)

- Stamina bar: 100 base (modifiable by gear/skills)
- Consumed by: sprinting, dodge rolling, shield blocking
- **Not** consumed by: mining, basic attacks, jumping, swimming
- Fast regen (3 seconds to full from empty, pauses briefly on use)
- Running out of stamina: cannot sprint or dodge until partial recovery. No other penalty.

---

## Multiplayer Balance

### Group play

Players with different specializations complement each other:
- A high-combat player tanks and deals damage
- A high-crafting player provides superior gear and consumables
- A high-harvesting player gathers resources efficiently
- A high-magic player provides healing, buffs, crowd control
- A high-engineering player builds defenses and automation

Groups can take on challenges (dungeons, bosses, events) that reward all participants.
Challenge difficulty scales with group size (configurable per world).

### Solo play

Solo players can complete all content. No content is group-locked. However:
- Challenges designed for groups are harder solo — requires better gear, buffs, and play
- Solo players naturally spread XP across more categories (they must do everything)
- A solo player who focuses on combat + harvesting can fight and gear up effectively
- Boss encounters have solo-scaled variants with reduced health/damage

### Challenge scaling

```yaml
# World settings
multiplayer:
  challenge_scaling: true          # scale difficulty with group size
  scaling_base: 1                  # solo baseline
  scaling_per_player: 0.7          # each additional player adds 70% of base difficulty
  # 2 players = 1.7x, 3 players = 2.4x, 4 players = 3.1x
  max_scaling: 5.0
```

---

## Material Tiers

| Tier | Materials | Key unlocks | How obtained |
|------|-----------|------------|-------------|
| 0 | Wood, plant fiber, stone | Basic tools, workbench | Surface gathering |
| 1 | Copper, tin → bronze | Furnace, bronze weapons/armor | Shallow mining |
| 2 | Iron | Anvil, iron gear, engineering basics | Medium-depth mining |
| 3 | Steel, silver | Advanced stations, magic basics | Deep mining, alloy crafting |
| 4 | Crystal, mithril | Enchanting table, crystal gear | Crystal caves + quest unlock |
| 5 | Resonite, void-touched | Dimension portal, legendary gear | Boss drops, deep exploration |

### Progression gating

Early tiers (0–2) use **crafting chains**: each tier's tools mine the next tier's ore.
Familiar, intuitive, no friction.

Mid tiers (3–4) add **exploration and quest gates**: crystal forging requires finding
crystal caves AND completing a quest that teaches the technique. This demonstrates the
event system as a progression mechanic.

Late tier (5) requires **boss encounters and dimensional exploration**: resonite is found
in alternate dimensions accessed via portal. Void-touched materials drop from endgame
bosses. This requires group play or exceptional solo skill.

### Skill integration

Material tiers interact with the skill system:
- Tier 3+ crafting recipes require minimum Crafting skill level
- Higher skill levels when crafting yield stat bonuses on the crafted item
- Some ore types require minimum Harvesting level to mine efficiently (below the
  threshold, mining is slow and yields less — not blocked entirely)
- Enchanting requires Magic skill investment

---

## Lua API Extensions

### Buff API

```lua
-- Register a custom buff type
Buffs.register("mymod:berserker_rage", {
  display_name = "Berserker Rage",
  icon = "textures/buffs/berserker.png",
  slot = "status",            -- food, drink, potion, status, equipment
  stackable = false,
  effects = {
    damage_bonus = 0.30,
    damage_taken_multiplier = 1.15,   -- modders CAN make punitive buffs
    move_speed = 0.10,
  }
})

-- Apply buff to player
player:add_buff("mymod:berserker_rage", { duration = 30 })
player:remove_buff("mymod:berserker_rage")
player:has_buff("mymod:berserker_rage")
player:get_active_buffs()
```

### Skill tree API

```lua
-- Register a new skill tree (mod adds an entire new category)
Skills.register_tree("mymod:necromancy", {
  display_name = "Necromancy",
  icon = "textures/skills/necromancy.png",
  xp_category = "magic",        -- shares XP with magic, or:
  -- xp_category = "mymod:dark_arts"  -- register a new XP category
})

-- Register a new XP category
XP.register_category("mymod:dark_arts", {
  display_name = "Dark Arts",
  earned_by = { "mymod:on_undead_summon", "mymod:on_soul_harvest" },
})

-- Append a node to an existing tree
Skills.append_node("core:combat", {
  id = "mymod:vampiric_strike",
  tier = 3,
  type = "passive",
  requires = { "core:combat.executioner" },
  ranks = {
    { lifesteal = 0.05 },
    { lifesteal = 0.10 },
  }
})

-- Check player skill state
local level = player:get_skill_level("combat")
local has_node = player:has_skill("core:combat.whirlwind")
local rank = player:get_skill_rank("core:combat.whirlwind")  -- 0 if not unlocked
```

### Combat API

```lua
-- Register a custom damage type
Damage.register_type("mymod:lightning", {
  display_name = "Lightning",
  icon = "textures/damage/lightning.png",
  default_resistance = 0.0,
})

-- Custom active ability script
-- skills/combat/whirlwind.lua
local Whirlwind = {}

function Whirlwind.on_activate(player, skill_data)
  local rank = skill_data.current_rank
  local radius = rank.radius
  local damage = rank.damage

  player:play_animation("whirlwind_slash")
  World.play_sound("core:whirlwind", player:position())
  World.spawn_particles("core:slash_arc", player:position())

  local targets = World.get_entities_in_radius(player:position(), radius)
  for _, entity in ipairs(targets) do
    if entity:is_hostile() then
      entity:apply_damage(damage, "physical", player)
      if rank.knockback then
        local dir = entity:position() - player:position()
        entity:apply_knockback(dir:normalized() * rank.knockback)
      end
    end
  end
end

return Whirlwind
```
