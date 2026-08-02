---
name: godot-index
description: Index and quick reference for the Godot skill set. Routes a task to the right skill (engine, gdscript, gdshader, gamedev, ui), provides a dense cheat sheet of the most-used Godot 4.x APIs and idioms, and confirms the engine version. Use as the entry-point skill whenever the user asks anything Godot-related, to pick the right deeper skill or answer directly from the cheat sheet.
---

# Godot — Index & Quick Reference

This is the **routing + cheat-sheet** skill for the whole Godot set. It tells you which dedicated skill to load and gives an at-a-glance reference so common questions can be answered immediately.

## 1. Skill map — load the right tool

| Task on hand | Load this skill |
|---|---|
| How the engine works / why is my scene behaving oddly / node lifecycle / physics / input flow / rendering | **`godot-engine`** |
| Writing or refactoring `.gd` scripts — typing, signals, idioms, resource scripts | **`godot-gdscript`** |
| Writing or debugging `shader_type` shaders / `ShaderMaterial` / uniforms / visual effects | **`godot-gdshader`** |
| Building a whole game — architecture, state machines, save/load, export, optimization | **`godot-gamedev`** |
| Making or fixing UI — Control nodes, containers, anchors, Theme/StyleBox, viewport scaling | **`godot-ui`** |
| Anything generic ("I want to make X in Godot") | Start here: route via the table above, answer simply from the cheat sheet below |

Rules of thumb:
- **Architecture/scene composition questions** → `godot-gamedev` (big picture) + `godot-engine` (how the tree works).
- **A broken script / API question** → `godot-gdscript`; if it's about why something behaves wrong at runtime, add `godot-engine`.
- **Visual effect or material** → `godot-gdshader` (+ `godot-engine` for renderer quirk checks).
- **A screen/menu/HUD** → `godot-ui` (+ `godot-gdscript` for the wiring code).

## 2. Version facts (always confirm if unsure)

- Targeting the **latest stable Godot 4.x** — currently **4.7.1**. No 3.x anywhere in this skill set.
- `project.godot` `config_version=5` ⇔ Godot 4.x.
- 4.x tells: `@export` (not `export`), typed signals, callable-only `.connect()`, `CharacterBody2D` (velocity built-in), `%UniqueName`, `move_and_slide()` on `CharacterBody2D`.
## 3. Quick-reference cheat sheet (Godot 4.x)

### Node lifecycle (order)
`_enter_tree()` → `_ready()` → `_process(delta)` / `_physics_process(delta)` → `_unhandled_input(event)` → `_exit_tree()`.
- `_ready` once per enter; children `_ready` run **bottom-up**.
- `_process` = per rendered frame; `_physics_process` = fixed 60 Hz.
- Prefer `await` + `create_tween()` for sequences; `await get_tree().create_timer(1.0).timeout` for delays.

### Node access
```gdscript
get_node("Path/Child")      # path resolve
$Child / $"../Sibling"      # shorthand
%UniqueName                 # unique node (robust)
get_tree().get_nodes_in_group("enemies")
add_child(x); remove_child(x); x.queue_free()
```

### Typed GDScript
```gdscript
@export var speed := 300.0          # inspector
@onready var sprite: Sprite2D = $Sprite
func add(a: int, b: int) -> int: return a + b
var evens := [1,2,3,4].filter(func(x): return x % 2 == 0)
```

### Signals
```gdscript
signal died(reason: String)
died.emit("killed")
enemy.died.connect(_on_died)                    # callable, never strings
enemy.died.connect(_on_died, CONNECT_ONE_SHOT)
```

### Movement / physics (CharacterBody2D)
```gdscript
extends CharacterBody2D
var gravity := ProjectSettings.get_setting("physics/2d/default_gravity")
func _physics_process(delta: float) -> void:
    if not is_on_floor(): velocity.y += gravity * delta
    velocity.x = Input.get_axis("move_left", "move_right") * SPEED
    move_and_slide()
```
Bodies: `CharacterBody2D` (kinematic mover), `RigidBody2D` (simulated), `StaticBody2D` (wall), `Area2D` (trigger zones → `body_entered/exited`).

### Resources / data
```gdscript
class_name ItemData
extends Resource
@export var display_name: String
@export var icon: Texture2D
# share-with-care: resources are shared by reference — .duplicate() before mutating
```

### Autoloads
Register under Project Settings → Autoload → then `EventBus.some_signal` / `GameState.score` anywhere.

### Save / load (JSON, user://)
```gdscript
var f := FileAccess.open("user://save.json", FileAccess.WRITE)
f.store_string(JSON.stringify(data))
var d: Variant = JSON.parse_string(FileAccess.open("user://save.json", FileAccess.READ).get_as_text())
```

### GDShader (2D tint / animate)
```gdshader
shader_type canvas_item;
uniform vec4 tint : source_color = vec4(1.0);
void fragment() { COLOR = texture(TEXTURE, UV) * tint; }
```
Set from code: `sprite.material.set_shader_parameter("tint", Color(1,0.5,0.5))`.
Types: `spatial` (3D, use `ALBEDO`/`ROUGHNESS`/`EMISSION`), `canvas_item` (2D, use `COLOR`), `particles`, `sky`. Built-ins: `UV`, `TIME`, `VERTEX`, `SCREEN_TEXTURE`.

### UI (Control / layout / styling)
- Containers: `VBoxContainer`, `HBoxContainer`, `GridContainer`, `MarginContainer`, `CenterContainer`, `PanelContainer`, `ScrollContainer`; child sizing via `size_flags` + `custom_minimum_size`.
- Full-rect anchors: set `anchor_right=1, anchor_bottom=1` (or preset `PRESET_FULL_RECT`).
- Style: `Theme` resource + `StyleBoxFlat`; per-node `add_theme_color_override("font_color", Color(...))`, `add_theme_stylebox_override("normal", sb)`, `add_theme_font_size_override("font_size", 22)`.
- Responsive: Project Settings → Display → Window → Stretch Mode `canvas_items` + Stretch Aspect `keep`.
- `mouse_filter` = `IGNORE` on decorative overlays, else they swallow clicks.
- Popups: `PopupMenu`, `ConfirmationDialog` (→ `confirmed`), `FileDialog`, `PopupPanel`; tooltips via `tooltip_text`.

### Common gotchas
- Resources are **shared by reference** → `.duplicate()` before editing.
- Freed notes → "Object was freed" — disconnect in `_exit_tree()`.
- UI scrolling with world → missing `CanvasLayer`.
- Input dead → full-rect `Control` with `mouse_filter` STOP — set IGNORE.
- Prefer `%UniqueName` over long paths; cache node refs in `@onready`.
- Optimize: `preload()` for constants, `load()` for optional; avoid allocs in per-frame loops; disable unneeded `set_process(false)`.

## 4. How the skills combine (workflow)

Use this playbook to answer Godot questions holistically:
1. **Identify domain** with the §1 table. Answer immediately from the cheat sheet if it's a quick fact.
2. **For code-level answers**, write a complete solution grounded in `godot-gdscript`/`godot-engine` idioms: name the nodes/`%UniqueName`, the signal, and the lifecycle callback involved.
3. **For architecture**, lean on `godot-gamedev` (scenes/autoloads/signals/state machines) before writing code.
4. **For visuals/UI**, pair `godot-ui` (structure + theming + stretch) with `godot-gdshader` (materials/shader effects) as needed.
5. **Verify version** (`config_version=5`), then confirm uncommon APIs against `docs.godotengine.org` before recommending — especially anything from the newest 4.7.x.

## Output guidance
Always: (1) state which skill(s) you're drawing from, (2) confirm the target is Godot 4.x, (3) give complete, runnable code with types, and (4) when something could be done several ways, recommend the engine-native approach (scenes, signals, tweens, resources, Theme) and say why. Keep partial steps small so the user can run and verify incrementally.


