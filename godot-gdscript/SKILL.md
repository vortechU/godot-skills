---
name: godot-gdscript
description: Expert GDScript programming for Godot 4.x — language syntax, typed GDScript and static typing, signals, groups, lambdas, annotations, resource scripts, naming conventions, and idiomatic patterns. Use when writing, reading, refactoring, or debugging .gd scripts in Godot 4.x.
---

# GDScript — Idiomatic Programming

Write clean, typed, idiomatic GDScript. Assume **Godot 4.x** unless the project says otherwise.

## 1. Fundamentals

- GDScript is dynamically typed but strongly typed; **add static types everywhere** — your IDE, the compiler, and future-you all benefit. Godot 4 does static analysis and catches many bugs at parse time when types are present.
- Indentation-based (tabs or spaces — pick one, the editor defaults to 4-space). A script must `extends` a base class (e.g. `extends Node2D`) or `class_name` a new one.
- Files: one script per `.gd` file. A script attached to a node instance; a `class_name` makes it a globally available type.

```gdscript
class_name Player
extends CharacterBody2D

signal died
const SPEED := 300.0
@export var jump_force := 400.0
var health := 100

func _ready() -> void:
    print("Hello")
```

## 2. Typed GDScript essentials

```gdscript
var typed_int: int = 1
var typed_float: float = 2.0
var typed_string: String = "hi"
var typed_array: Array[int] = [1, 2, 3]
var typed_dict: Dictionary = {"key": "value"}
var node_ref: Node2D = null
@onready var sprite: Sprite2D = $Sprite          # resolved once at _ready

# Return type + param types + inferred locals via :=
func add(a: int, b: int) -> int:
    var total := a + b          # := infers int
    return total
```

Annotations you'll use constantly:
- `@export var x := 1` — editable in the Inspector. Types: `@export_range(0, 10)` for ranges, `@export_group("Movement")`, `@export var scene: PackedScene`, `@export_enum("A","B")`.
- `@onready var x := get_node(...)` — assign when the node's `_ready` runs (avoids null before ready).
- `@warning_ignore("unused_parameter")` — suppress a specific static warning.
- `@tool` — run the script in the editor (for editor tools/plugins).
- `@static_unload` / `@preserve` — advanced lifetime controls.

## 3. Signals (typed, in 4.x)

Signals can carry typed payloads. Emit with `.emit`, connect with a callable.

```gdscript
signal health_changed(prev: int, now: int)
signal died(reason: String)

func damage(n: int) -> void:
    var prev := health
    health = maxi(health - n, 0)
    health_changed.emit(prev, health)
    if health == 0:
        died.emit("killed")

# connect:
player.health_changed.connect(_on_health_changed)
# one-shot:
player.died.connect(_on_died, CONNECT_ONE_SHOT)
```

- Never connect with strings. In Godot 4 always bind a callable: `connect(_on_x)` or `some_node.some_signal.connect(func(): ...)`.
- Signal parameters can reference types declared with `class_name` for full static checking.

## 4. Data structures & functional tools

```gdscript
var arr: Array[int] = [3, 1, 2]
arr.sort()                          # in-place; arrow form returns copy
var evens := arr.filter(func(x): return x % 2 == 0)
var doubled := arr.map(func(x): return x * 2)
var total := arr.reduce(func(acc, x): return acc + x, 0)
```

- **Dictionaries** are the workhorse data structure (like a JS object). Access via `.get("key", default)` to avoid errors. Iterate `for k in dict:` (keys) or `for k in dict.keys():`.
- **Arrays/TypedArrays** and **PackedArray** variants (`PackedByteArray`, `PackedVector2Array`) — use Packed arrays for performance with bulk numeric data.
- Lambdas (anonymous functions) are first-class `Callable`s — great for short hooks, but prefer named methods for readability beyond a line or two.
- **Loops:**
```gdscript
for i in range(10):          # 0..9
for i in range(1, 10, 2):    # step
for child in get_children(): # iterating nodes
for item in items:           # arrays/dicts
```
- `match` is GDScript's switch, and it's strict about types:
```gdscript
match state:
    "idle": idle()
    "run": run()
    _: default_branch()
```

## 5. Resource scripts & custom classes

Turn data into typed objects by extending `Resource`:

```gdscript
class_name ItemData
extends Resource

@export var display_name: String
@export var icon: Texture2D
@export var value := 0

func describe() -> String:
    return "%s (worth %d)" % [display_name, value]
```

Create instances with `ItemData.new()` and assign exported properties — then you can drag `.tres` files into the Inspector as first-class data assets. This is the idiomatic Godot way to model game data (items, weapons, enemies, dialogue).

## 6. Idioms & conventions

- **Naming:** `snake_case` for vars/functions; `PascalCase` for class names, signals, constants are `UPPER_SNAKE` inside a class; signal names are often verb phrases (`health_changed`).
- **Private-ish**: prefix with `_` (e.g. `_process`) is convention, not enforced.
- **`_init()` vs `_ready()`**: `_init()` runs at object construction (no tree access); `_ready()` runs when entering the tree and can touch nodes. Setup lightweight data in `_init`, node work in `_ready`.
- **Await coroutines**: `await get_tree().create_timer(1.0).timeout` for a timed pause; `await some_signal` to wait for events. Great for sequences without messy state machines.
- **Godot functions you'll reach for:** `randf()`, `randi_range(a,b)`, `clampf(x, a, b)`, `lerpf(a, b, t)`, `move_toward(value, target, delta)`, `fposmod`, `snappedf`. For easing: `tween = create_tween(); tween.tween_property(node, "position", target, 0.5).set_trans(Tween.TRANS_QUAD)`.
- **Avoid**: global mutable static, string `NodePath` manipulation for logic, editing the tree during physics iteration without care, and over-nesting scene nodes for code organisation (use signals/autoloads instead).
- **Performance-aware**: prefer `preload()` for compile-time constant references, `load()` for runtime/optional assets; avoid allocating in `_process`/`_physics_process` hot loops; reuse nodes instead of instantiating per-frame.

## 7. Output guidance
Give complete, runnable snippets with types and correct API for the project's engine version. Explain *why* an idiom is used (typing, signals, autoloads) rather than just pasting code. If a snippet depends on a scene structure/class_name, show the minimal setup so it compiles.

