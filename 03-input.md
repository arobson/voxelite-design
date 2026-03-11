# Input System

---

## Decision: Godot 4 InputMap (built-in, no third-party library needed)

Godot 4's built-in input system fully covers all requirements for keyboard, mouse, and
gamepad support. No additional library is required.

---

## Core Concept — Abstract Actions

All input is mapped through **abstract actions** defined in the InputMap. A single action
can simultaneously accept a keyboard key, mouse button, and gamepad button. Game code
never checks for specific physical keys — only action names.

```gdscript
# Correct — device-agnostic
if Input.is_action_pressed("jump"):
    player.jump()

# Never do this — device-specific, breaks gamepad/rebinding
if Input.is_key_pressed(KEY_SPACE):
    player.jump()
```

---

## Input Categories

### Keyboard + Mouse
- Raw key events via `InputEventKey`
- Mouse buttons via `InputEventMouseButton`
- Mouse motion via `InputEventMouseMotion` — provides relative delta for camera look
- Scroll wheel for hotbar scrolling, zoom
- Mouse capture: `Input.mouse_mode = Input.MOUSE_MODE_CAPTURED` — one line for first-person cursor lock

### Gamepad
- Auto-detected on all platforms via SDL2 controller database (no setup code)
- PlayStation, Xbox, Switch Pro, and generic USB controllers work with correct button labels
- Analog sticks: raw axis values + deadzone-processed values
- Custom deadzone thresholds configurable per-action in InputMap
- Rumble via `Input.start_joy_vibration(device, weak, strong, duration)`

---

## Action Map (default bindings)

All actions defined here are rebindable by the player. Mods can register additional
actions through the mod API.

```yaml
# Defined in the InputMap — not hardcoded in scripts
actions:
  # Movement
  move_forward:     [W, gamepad:left_stick_up]
  move_backward:    [S, gamepad:left_stick_down]
  move_left:        [A, gamepad:left_stick_left]
  move_right:       [D, gamepad:left_stick_right]
  jump:             [Space, gamepad:button_a]
  sprint:           [Shift, gamepad:left_stick_click]
  crouch:           [Ctrl, gamepad:button_b]

  # Interaction
  interact:         [E, gamepad:button_x]
  attack_primary:   [mouse:left, gamepad:right_trigger]
  attack_secondary: [mouse:right, gamepad:left_trigger]
  use_item:         [F, gamepad:button_y]

  # Inventory
  open_inventory:   [Tab, gamepad:button_select]
  hotbar_next:      [mouse:scroll_down, gamepad:right_bumper]
  hotbar_prev:      [mouse:scroll_up,   gamepad:left_bumper]
  hotbar_1..9:      [1..9]

  # UI / System
  open_map:         [M, gamepad:dpad_down]
  open_crafting:    [C, gamepad:dpad_left]
  pause:            [Escape, gamepad:start]
  toggle_hud:       [F1]
  screenshot:       [F2]
```

---

## Player Rebinding

Godot does not provide a pre-built rebinding UI, but provides all the necessary API.
A rebinding screen must be built early in development — players and modders expect it.

**Implementation approach:**
```gdscript
# Detect next input event for rebinding
func _await_rebind(action_name: String) -> void:
    _rebinding = action_name
    # Set engine to pass-through mode — next InputEvent is captured

func _input(event: InputEvent) -> void:
    if _rebinding:
        InputMap.action_erase_events(_rebinding)
        InputMap.action_add_event(_rebinding, event)
        _save_bindings()
        _rebinding = ""

# Persist custom bindings to user config
func _save_bindings() -> void:
    var cfg = ConfigFile.new()
    for action in InputMap.get_actions():
        for event in InputMap.action_get_events(action):
            cfg.set_value("bindings", action, event)
    cfg.save("user://input_config.cfg")
```

---

## Mod Input API

Mods can read input state and register new actions, but cannot register raw hardware
bindings directly. All mod actions are declared in `mod.yaml` and registered into the
InputMap by the engine on behalf of the mod.

```yaml
# mymod/mod.yaml
keybindings:
  - id: mymod:open_resonance_ui
    default_key: R
    default_gamepad: dpad_right
    description: "Open Resonance Energy panel"
    category: "Crystal Realm"
```

```lua
-- Mods read actions via the Input API
function ResonanceSystem.on_world_tick(delta)
  if Input.is_action_just_pressed("mymod:open_resonance_ui") then
    GUI.open("mymod:resonance_panel", nil, local_player)
  end
end
```

**Restrictions:**
- Mods cannot call `InputMap.action_erase_events()` on core or other mods' actions
- Mods cannot read raw hardware state (`Input.is_key_pressed`) — action API only
- Mod keybindings appear in the rebinding screen under their declared `category`

---

## Camera Input

First-person camera look uses mouse relative motion:

```gdscript
func _input(event: InputEvent) -> void:
    if event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
        rotate_y(-event.relative.x * mouse_sensitivity)
        camera.rotate_x(-event.relative.y * mouse_sensitivity)
        camera.rotation.x = clamp(camera.rotation.x, -PI/2, PI/2)
```

Gamepad camera uses right stick axis values with configurable sensitivity and optional
aim-assist smoothing.
