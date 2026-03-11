# UI / GUI System

---

## Overview

The UI system has two layers: core game screens built as Godot scenes by the development
team, and mod UIs declared in YAML with optional Lua scripting. Both layers use the same
theme system for visual consistency.

All interactive UIs support keyboard, mouse, and gamepad navigation.

---

## Architecture

### Core game UI — Godot scenes

The game's own screens (menus, HUD, inventory, settings) are authored as `.tscn` scenes
using Godot's Control node system. The development team has full access to Godot's UI
tooling, animations, and shader effects.

### Mod UI — YAML-declared

Mod UIs are defined in YAML. The engine builds a Godot Control tree at runtime from the
YAML declaration. Modders describe layout and wire data bindings declaratively. Lua
scripting is available for advanced interactivity but is not required for common cases.

This keeps the "data in YAML, logic in Lua" principle for mod-facing content. Modders
do not need Godot's scene editor.

### Theme system

The game provides a default theme (colors, fonts, borders, spacing, control styles).
All UI — core and mod — inherits from this theme. Mods can override individual style
properties per-element. Texture/skin packs (Tier 1 mods) can replace the entire theme.

```yaml
# core/ui/theme.yaml

theme:
  fonts:
    default: fonts/nunito_regular.ttf
    bold: fonts/nunito_bold.ttf
    mono: fonts/jetbrains_mono.ttf
    title: fonts/nunito_bold.ttf

  font_sizes:
    small: 12
    default: 14
    large: 18
    title: 24
    heading: 20

  colors:
    background: "#1a1a2e"
    panel: "#16213e"
    panel_border: "#0f3460"
    text: "#e0e0e0"
    text_dim: "#888888"
    accent: "#e94560"
    accent_hover: "#ff6b81"
    success: "#44cc88"
    warning: "#ffaa33"
    error: "#ff4444"
    slot_background: "#0a0a1a"
    slot_border: "#333355"
    slot_hover: "#444466"
    tooltip_background: "#000000cc"
    tooltip_text: "#ffffff"
    progress_bar: "#44aaff"
    progress_background: "#1a1a2e"

  spacing:
    padding: 8
    margin: 4
    slot_gap: 4
    section_gap: 16

  controls:
    button:
      min_height: 32
      border_radius: 4
      border_width: 1
    slot:
      size: [48, 48]
      border_radius: 2
    progress_bar:
      height: 16
      border_radius: 2
    slider:
      height: 20
      handle_size: 16
    toggle:
      size: [40, 22]
    text_input:
      min_height: 28
      border_radius: 4
    tab:
      min_height: 32
      padding: [12, 6]
```

Mods override theme values per-element:

```yaml
# In a mod GUI definition
style:
  background: "#2a1a0e"        # warm brown for this panel
  accent: "#ffaa33"            # gold accent for this mod's UI
```

---

## Built-In Controls

All controls available to mod YAML definitions:

### Layout controls

| Control | Purpose | Properties |
|---------|---------|-----------|
| `vertical` | Stack children vertically | `spacing`, `padding`, `align` |
| `horizontal` | Stack children horizontally | `spacing`, `padding`, `align` |
| `grid` | Grid layout with columns | `columns`, `spacing`, `cell_size` |
| `scroll` | Scrollable container | `direction` (vertical/horizontal/both), `max_height` |
| `panel` | Styled container with border | `title`, `collapsible` |
| `tabs` | Tabbed container | `tabs: [{label, content}]` |
| `spacer` | Empty space | `size` or `flex` (fill remaining) |
| `separator` | Visual divider line | `direction` (horizontal/vertical) |

### Interactive controls

| Control | Purpose | Properties |
|---------|---------|-----------|
| `button` | Clickable button | `text`, `icon`, `on_click`, `disabled` |
| `toggle` | On/off switch | `label`, `bind`, `on_change` |
| `slider` | Numeric range input | `min`, `max`, `step`, `bind`, `label` |
| `dial` | Rotary knob input | `min`, `max`, `step`, `bind`, `label` |
| `text_input` | Text entry field | `placeholder`, `max_length`, `bind`, `on_submit` |
| `dropdown` | Selection from list | `options`, `bind`, `on_change` |
| `checkbox` | Boolean toggle | `label`, `bind`, `on_change` |

### Display controls

| Control | Purpose | Properties |
|---------|---------|-----------|
| `label` | Text display | `text`, `font_size`, `align`, `color`, `bind` |
| `icon` | Image display | `texture`, `size`, `tint` |
| `progress_bar` | Progress indicator | `bind`, `min`, `max`, `color`, `label` |
| `item_icon` | Item with count badge | `bind` (item stack), `size` |
| `entity_preview` | 3D entity render | `entity_type`, `rotation`, `size` |
| `tooltip` | Hover information | Applied via `tooltip` property on any control |

### Inventory controls

| Control | Purpose | Properties |
|---------|---------|-----------|
| `slot` | Single inventory slot | `id`, `size`, `tooltip`, `read_only`, `filter` |
| `slot_grid` | Grid of inventory slots | `id`, `columns`, `slots`, `slot_size` |
| `player_inventory` | Standard player inventory panel | `compact` (bool) |
| `recipe_list` | Scrollable recipe browser | `station`, `filter_tags`, `on_select` |
| `recipe_detail` | Selected recipe with craft button | `bind_recipe`, `auto_pull` |

---

## Inventory & Container Interaction

### Layout convention

When a player interacts with a container or crafting station, the screen displays:
- **Left side:** Player's inventory
- **Right side:** Container/station interface

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌──────────────────┐    ┌────────────────────────┐ │
│  │ Player Inventory │    │  Container / Station   │ │
│  │                  │    │                        │ │
│  │  [48 slots grid] │    │  (mod-defined layout)  │ │
│  │  6 columns × 5   │    │                        │ │
│  │                  │    │                        │ │
│  │  ── hotbar ──    │    │                        │ │
│  │  [9 slots]       │    │                        │ │
│  │                  │    │                        │ │
│  │  ── equipment ── │    │                        │ │
│  │  [head][chest]   │    │                        │ │
│  │  [legs][feet]    │    │                        │ │
│  └──────────────────┘    └────────────────────────┘ │
│                                                     │
└─────────────────────────────────────────────────────┘
```

The player inventory panel is provided by the engine automatically. Mod GUIs define
only the right-side content. The engine handles the split layout, drag-and-drop between
sides, and responsive sizing.

### Drag and drop

All slot-based interfaces support drag-and-drop:

- **Click and drag** an item stack to move it
- **Drag to another slot** to place or swap
- **Drag between player inventory and container** to transfer
- **Right-click drag** to split a stack (take half)
- **Shift-click** to quick-transfer a full stack to the other side
- **Drop outside window** to drop item into the world

Drag and drop works identically for core UI and mod-defined UIs. The engine handles
all drag logic — mod YAML only declares slots and their IDs.

### Item filtering

Slots can declare accepted item types:

```yaml
- type: slot
  id: fuel
  filter:
    tags: [fuel, burnable]
  tooltip: "Place fuel here"

- type: slot
  id: sword_slot
  filter:
    category: weapon
    tags: [sword]
  tooltip: "Sword only"
```

Dragging an invalid item to a filtered slot shows a rejection visual (red tint, bounce
back). The engine enforces filtering — mods cannot bypass it.

---

## Crafting System UI

### Recipe auto-pull from nearby containers

Crafting stations do not require players to manually drag items into input slots.
Instead, the station searches nearby container inventories to satisfy recipe
requirements.

**Search order for ingredients:**
1. Player's own inventory
2. Containers within a configurable radius of the crafting station (default: 4 blocks)
3. Other crafting stations within radius (their internal inventories)

**Crafting flow:**

```
1. Player opens crafting station
2. Right panel shows recipe browser (filterable, scrollable)
3. Recipes are color-coded:
   - Green:  all ingredients available (player + nearby containers)
   - Yellow: some ingredients available
   - Red:    missing critical ingredients
   - Gray:   skill level too low to craft
4. Player selects a recipe
5. Recipe detail panel shows:
   - Required ingredients with source indicators
     "Iron Ingot ×3 (inventory: 2, chest: 1)"
   - Required skill level and current level
   - Output item preview with stats
   - Craft button (enabled when all requirements met)
6. Player clicks Craft
7. Engine pulls ingredients from sources in search order
8. Output item placed in player inventory (or dropped if full)
9. Crafting XP awarded
```

```yaml
# Crafting station block definition
id: core:crafting_table
gui: blocks/crafting_table/crafting_table_gui.yaml

crafting:
  type: crafting_table         # recipe station type
  pull_radius: 4               # blocks — search for ingredients in nearby containers
  categories: [shaped, shapeless]   # recipe types this station supports
```

### Recipe browser GUI (engine-provided component)

```yaml
# blocks/crafting_table/crafting_table_gui.yaml

id: core:crafting_table_gui
display_name: Crafting Table
size: [400, 450]

layout:
  type: vertical
  padding: 10
  children:
    - type: label
      text: "Crafting Table"
      font_size: 18
      align: center

    - type: tabs
      tabs:
        - label: "All"
          content:
            type: recipe_list
            id: recipes
            station: core:crafting_table
            on_select: select_recipe

        - label: "Weapons"
          content:
            type: recipe_list
            id: recipes_weapons
            station: core:crafting_table
            filter_tags: [weapon]
            on_select: select_recipe

        - label: "Armor"
          content:
            type: recipe_list
            id: recipes_armor
            station: core:crafting_table
            filter_tags: [armor]
            on_select: select_recipe

        - label: "Tools"
          content:
            type: recipe_list
            id: recipes_tools
            station: core:crafting_table
            filter_tags: [tool]
            on_select: select_recipe

    - type: separator

    - type: recipe_detail
      id: selected_recipe
      auto_pull: true
      pull_radius: 4
```

### Ingredient source visualization

When a recipe is selected, the detail panel shows where each ingredient will be
pulled from:

```
┌─────────────────────────────────────┐
│  Crystal Sword                      │
│  ┌──────┐                           │
│  │ icon │  Damage: 18               │
│  │      │  Speed: 1.6               │
│  └──────┘  Durability: 400          │
│                                     │
│  Ingredients:                       │
│  [Crystal Shard] ×2  ✓ inventory    │
│  [Stick]         ×1  ✓ chest (2m)   │
│                                     │
│  Skill: Crafting Lv.15 ✓ (Lv.18)   │
│                                     │
│  Bonus: +10% durability (Lv.20+)    │
│                                     │
│  [ Craft ]                          │
└─────────────────────────────────────┘
```

---

## HUD System

### Core HUD layout

The HUD is always visible during gameplay. Elements are anchored to screen edges.

```
┌──────────────────────────────────────────────────────┐
│ [Health bar]              [Compass/Direction]  [Time]│
│ [Stamina bar]                           [Minimap]   │
│ [Active buffs row]                                   │
│                                                      │
│                                                      │
│                                                      │
│                      [Crosshair]                     │
│                                                      │
│                                                      │
│                                                      │
│ [Quest tracker]                     [Chat window]    │
│                                                      │
│           [Hotbar: 1-9 slots with selection]         │
│           [XP bar / current category]                │
└──────────────────────────────────────────────────────┘
```

### HUD elements

| Element | Anchor | Contents |
|---------|--------|----------|
| Health bar | Top-left | Red bar, numeric value, heart icon |
| Stamina bar | Below health | Green bar, shown only when not full |
| Buff row | Below stamina | Icons for active buffs with remaining duration |
| Compass | Top-center | Cardinal directions, marker indicators |
| Time | Top-right | In-game time, day count |
| Minimap | Top-right (below time) | Top-down chunk view, entity dots, markers |
| Crosshair | Center | Context-sensitive (changes on hover over interactable) |
| Quest tracker | Bottom-left | Active quest objectives with progress |
| Chat | Bottom-right | Message history, input field |
| Hotbar | Bottom-center | 9 item slots, selected slot highlighted |
| XP indicator | Below hotbar | Current XP category and progress to next level |

### HUD toggle

`F1` hides all HUD elements. `F3` shows the debug overlay (from tooling doc).
Individual elements can be toggled in settings.

### Mod HUD elements

Mods register HUD elements that appear alongside core elements:

```yaml
# Mod HUD element definition
id: crystal_realm:resonance_bar
type: progress_bar
anchor: bottom_left
offset: { x: 10, y: -80 }     # relative to anchor
size: [150, 14]
z_order: 10                     # layering priority

bind: crystal_realm.resonance.current
max_bind: crystal_realm.resonance.max
color: "#aa44ff"
label: "Resonance"
icon: textures/hud/resonance_icon.png

visibility:
  conditions:
    - flag_set: crystal_realm_unlocked
```

---

## Mod GUI Definition Format

### Complete example — Alchemy Table

```yaml
# blocks/alchemy_table/alchemy_table_gui.yaml

id: mymod:alchemy_table
display_name: Alchemy Table
size: [320, 300]

layout:
  type: vertical
  padding: 12
  children:
    - type: label
      text: "Alchemy Table"
      font_size: 18
      align: center

    - type: separator

    - type: horizontal
      spacing: 16
      align: center
      children:
        - type: vertical
          spacing: 4
          children:
            - type: label
              text: "Ingredients"
              font_size: 12
              color: text_dim
            - type: horizontal
              spacing: 4
              children:
                - type: slot
                  id: input1
                  tooltip: "First ingredient"
                - type: slot
                  id: input2
                  tooltip: "Second ingredient"

        - type: vertical
          align: center
          children:
            - type: icon
              texture: textures/gui/arrow_right.png
              size: [24, 24]
            - type: progress_bar
              id: progress
              size: [24, 6]
              bind: block_entity.progress
              max: block_entity.recipe_time

        - type: vertical
          spacing: 4
          children:
            - type: label
              text: "Result"
              font_size: 12
              color: text_dim
            - type: slot
              id: output
              read_only: true
              tooltip: "Alchemy result"

    - type: separator

    - type: horizontal
      align: center
      spacing: 8
      children:
        - type: label
          text: "Catalyst:"
        - type: slot
          id: catalyst
          filter:
            tags: [catalyst]
          tooltip: "Required catalyst (not consumed)"

    - type: spacer
      size: 8

    - type: label
      id: status_text
      text: ""
      align: center
      font_size: 12
      bind: block_entity.status_message
```

### Data binding

Controls auto-bind to data sources using dot-notation paths:

```yaml
# Auto-bind to block entity data
bind: block_entity.progress          # reads from BlockEntity data at this key
bind: block_entity.slots.input1      # binds slot to block entity inventory slot

# Auto-bind to player data
bind: player.health.current
bind: player.skills.combat.level

# Auto-bind to state store
bind: state.player.reputation.elder
bind: state.world.village_state

# Auto-bind to mod system data
bind: crystal_realm.resonance.current
```

Binding is two-way for interactive controls (slots, sliders, toggles, text input).
Display controls (labels, progress bars) bind one-way (read only).

The engine updates bound controls automatically when the source data changes. No Lua
required for standard data display and interaction.

### Lua scripting for advanced UI

When YAML data binding is insufficient, mod GUIs can use Lua for complex logic:

```yaml
# GUI definition with Lua handler
id: mymod:enchanting_table
display_name: Enchanting Table
size: [400, 350]
behavior_script: blocks/enchanting_table/enchanting_gui.lua

layout:
  type: vertical
  children:
    - type: slot
      id: item_slot
      on_change: on_item_changed     # calls Lua function

    - type: scroll
      max_height: 200
      children:
        - type: vertical
          id: enchantment_list
          # populated dynamically by Lua

    - type: button
      id: enchant_button
      text: "Enchant"
      on_click: on_enchant_clicked
      disabled: true
```

```lua
-- blocks/enchanting_table/enchanting_gui.lua
local EnchantGUI = {}

function EnchantGUI.on_item_changed(gui, slot_id, item)
  local list = gui:get("enchantment_list")
  list:clear()

  if not item then
    gui:get("enchant_button"):set_disabled(true)
    return
  end

  -- Query available enchantments for this item type
  local enchantments = Enchanting.get_available(item)
  for _, ench in ipairs(enchantments) do
    local affordable = Enchanting.can_afford(ench, gui.player)
    list:add_child({
      type = "horizontal",
      spacing = 8,
      children = {
        { type = "checkbox", id = "ench_" .. ench.id, label = ench.display_name,
          disabled = not affordable },
        { type = "label", text = "Cost: " .. ench.xp_cost .. " XP",
          color = affordable and "text" or "error" },
      }
    })
  end

  gui:get("enchant_button"):set_disabled(#enchantments == 0)
end

function EnchantGUI.on_enchant_clicked(gui)
  local item = gui:get_slot("item_slot")
  local selected = {}
  for _, ench in ipairs(Enchanting.get_available(item)) do
    if gui:get("ench_" .. ench.id):is_checked() then
      table.insert(selected, ench.id)
    end
  end

  if #selected > 0 then
    Enchanting.apply(item, selected, gui.player)
    gui:get("enchant_button"):set_disabled(true)
    gui:refresh()
  end
end

return EnchantGUI
```

---

## NPC Dialogue UI

The dialogue UI is a core screen driven by event tree data. It's not mod-YAML-defined
because it has specific presentation requirements (portrait, animated text, choice
buttons).

### Layout

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│                    (game world visible behind)        │
│                                                      │
│  ┌──────────────────────────────────────────────────┐│
│  │ ┌────────┐                                       ││
│  │ │Portrait│  Village Elder                        ││
│  │ │        │                                       ││
│  │ └────────┘  "Strangers rarely come here.         ││
│  │              We need help..."                    ││
│  │                                                  ││
│  │  ┌─────────────────────────────────────────────┐ ││
│  │  │ > What's wrong?                             │ ││
│  │  │ > Not my problem.                           │ ││
│  │  │ > [Persuade - Lv.10] Tell me more.          │ ││
│  │  └─────────────────────────────────────────────┘ ││
│  └──────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────┘
```

### Features
- NPC portrait from entity YAML `npc.portrait` field
- Text appears with typewriter effect (skippable)
- Choices shown as a vertical list of buttons
- Skill-gated choices show requirement and current level
- Condition-failed choices are hidden (not grayed — player doesn't see paths they can't take)
- Keyboard navigation: number keys select choices, Enter advances
- Gamepad: D-pad to select, A to confirm

### Connection to event tree system

The dialogue UI reads directly from the active event tree node:
- `node.text` → displayed text
- `node.npc` → portrait and name lookup
- `node.choices[]` → choice buttons (filtered by conditions)
- `node.effects[]` → executed when node is entered
- `node.next` / `node.chain` → navigation on choice selection

---

## Skill Tree UI

The skill tree screen is a core screen with a specialized layout.

### Layout

```
┌──────────────────────────────────────────────────────┐
│  [Physical] [Magic] [Crafting] [Harvest] [Combat] [Eng]│
│  ─────────────────────────────────────────────────────│
│                                                      │
│  Available Points: 3           Combat Level: 12      │
│                                                      │
│    ┌──────────┐                                      │
│    │ Steady   │──────┐                               │
│    │ Hand III │      │                               │
│    │ ✓✓✓     │      ▼                               │
│    └──────────┘  ┌──────────┐                        │
│                  │Whirlwind │                        │
│    ┌──────────┐  │ Slash I  │──────┐                 │
│    │ Thick    │  │ ✓        │      │                 │
│    │ Skin II  │  └──────────┘      ▼                 │
│    │ ✓✓      │               ┌──────────┐            │
│    └──────────┘──────────────│Executioner│            │
│                              │  Locked   │            │
│                              │ Lv.15 req │            │
│                              └──────────┘            │
│                                                      │
│  ┌─ Tooltip ──────────────────────────────────────┐  │
│  │ Whirlwind Slash (Rank 1/3)                     │  │
│  │ Spin attack hitting all enemies within 3.0m    │  │
│  │ Damage: 15  Cooldown: 12s                      │  │
│  │ Next rank: Damage 22, Radius 3.5m (1 point)    │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

### Features
- Tab bar for each XP category
- Zoomable/pannable node graph (same interaction model as the event tree editor)
- Nodes show: icon, name, current rank, unlock state
- Connections show prerequisite relationships
- Hover for detailed tooltip
- Click to spend points (confirmation prompt)
- Locked nodes show requirement (level, prerequisites)
- Mod-added trees appear as additional tabs
- Mod-appended nodes appear in existing trees seamlessly

---

## Responsive Scaling

All UI scales with a configurable UI scale factor:

```yaml
# User settings
ui:
  scale: 1.0            # 0.75, 1.0, 1.25, 1.5, 2.0
  tooltip_delay: 0.5    # seconds before tooltip appears
  chat_opacity: 0.7     # background opacity of chat window
  hud_opacity: 0.9
```

Layout containers use relative sizing and anchoring. The engine scales all pixel
values by the UI scale factor. Mod GUIs declare sizes in logical pixels — the engine
handles DPI scaling.

---

## Gamepad UI Navigation

All UI screens support gamepad navigation:

- **D-pad / left stick:** Move focus between controls
- **A button:** Activate focused control (press button, toggle, pick up item)
- **B button:** Back / close current screen
- **X button:** Context action (split stack, quick-transfer)
- **Y button:** Secondary action (drop item, view details)
- **Bumpers:** Switch tabs
- **Start:** Close all UI, return to game

Focus is visualized with a highlight border around the currently selected control.
The focus system auto-navigates in the logical direction of input (spatial navigation
for grids, sequential for lists).

Drag-and-drop on gamepad: press A to pick up item, navigate to target slot, press A
to place. Visual cursor shows the held item.

---

## Mod GUI Lua API

```lua
-- Opening and closing
GUI.open(gui_id, block_pos, player)          -- open a block GUI
GUI.open_screen(gui_id, player)              -- open a standalone screen
GUI.close(player)                            -- close current GUI
GUI.is_open(player)                          -- returns bool

-- Dynamic UI manipulation (from behavior scripts)
gui:get(control_id)                          -- get control by id
gui:set_value(control_id, value)             -- set control value
gui:get_value(control_id)                    -- get control value
gui:set_text(control_id, text)               -- set label text
gui:set_visible(control_id, visible)         -- show/hide control
gui:set_disabled(control_id, disabled)       -- enable/disable control

-- Slot operations
gui:get_slot(slot_id)                        -- get item in slot
gui:set_slot(slot_id, item_stack)            -- set item in slot
gui:clear_slot(slot_id)                      -- empty a slot

-- Dynamic children (for lists, scroll content)
gui:get(container_id):clear()                -- remove all children
gui:get(container_id):add_child(definition)  -- add child from table
gui:get(container_id):remove_child(index)    -- remove specific child

-- Event callbacks
gui:on(control_id, event, callback)          -- register event handler
-- Events: "click", "change", "hover_enter", "hover_exit", "submit"

-- Refresh
gui:refresh()                                -- re-read all bindings, update display
```

### Crafting API (for custom crafting stations)

```lua
-- Query recipes available at a station
Crafting.get_recipes(station_type)                  -- all recipes for station type
Crafting.get_recipes(station_type, { tags = {"weapon"} })  -- filtered

-- Check ingredient availability
Crafting.check_ingredients(recipe, player, pull_radius)
  -- Returns: { available: bool, sources: { ingredient_id: { source, count } } }

-- Execute crafting
Crafting.craft(recipe, player, pull_radius)
  -- Pulls ingredients from player + nearby containers
  -- Places result in player inventory
  -- Awards crafting XP
  -- Returns: { success: bool, item: result_item }
```
