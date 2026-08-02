---
name: godot-engine
description: Deep understanding of the Godot engine as a whole — the scene tree, nodes, signals, resources, the node lifecycle, autoloads, physics, input, rendering, and project structure. Use when the user is working in or asking about Godot 4.x and needs engine-level context, architecture decisions, or to fix something that depends on how the engine actually works.
---

# Godot Engine — Mental Model

This skill gives you a coherent mental model of how Godot actually works, so you can reason about architecture, debug odd behaviour, and answer "why does the engine do X?" questions. It targets the **latest Godot 4.x** (verify with `project.godot`'s `config_version` = `5`). There is **no 3.x support** — everything here is Godot 4.x.

## 1. The core model: everything is a tree of nodes

- A **Node** is the base building block — anything with a position, behaviour, or lifecycle. `Node2D`, `Sprite2D`, `CharacterBody2D`, `Control`, `Timer`, etc. are all nodes.
- A **scene** is a reusable, `.tscn`-serialized tree of nodes (plus a root node + metadata).
- An **instance** is a copy of a scene placed inside another scene (the "instanced child" carries a `[node name="X" parent="." instance=ExtResource("1")]`).
- The **SceneTree** (`get_tree()`) is the single global tree formed by the *current scene* merged with autoloads. Every node has exactly one parent except the tree root.
- This tree IS the game. Interacting with the tree is the primary model: adding/removing nodes, reparenting, finding nodes, and traversing.

### Key tree operations (GDScript)
```gdscript
get_tree().current_scene              # the root of the currently running scene
get_parent()                          # parent node
get_node("Path/To/Node")              # resolve a node path (from self)
get_node("/root/AutoloadName")        # absolute path to an autoload
%UniqueName                           # shorthand: get_node("%UniqueName") via %Name
get_tree().get_nodes_in_group("enemies")
add_child(node)                       # add to tree
remove_child(node)                    # detach (node keeps existing but off-tree)
node.queue_free()                     # safe deletion at end of frame
node.set_process(false)               # stop per-frame processing
```

### Node paths
- `"."` self, `".."` parent, `"../Sibling"`, `"Child/SubChild"`.
- **Unique names** (the `%` notation, assigned in the editor): `%Sprite` resolves to the nearest ancestor subtree's uniquely-named node — far more robust than long paths.

## 2. Node lifecycle (order matters)

Built-in callbacks, in the exact order the engine calls them:

1. `_enter_tree()` — node enters the running tree (before it's ready).
2. `_ready()` — **once** per enter, after the whole scene is within the tree. First chance to safely touch children.
3. `_process(delta)` — every rendered frame (delta = seconds), only if processing is enabled.
4. `_physics_process(delta)` — fixed timestep (default 60/s), for physics & anything physics-adjacent.
5. `_input(event)`, `_unhandled_input(event)` — input events (see §6).
6. `_exit_tree()` — node is about to leave the tree.
7. `_notification()` — low-level notification dispatch backing all of the above.

Pitfalls to remember:
- `_ready()` may run **before** a child's `_ready()` completes setup if you do heavy work — children's `_ready()` runs **bottom-up**, parents' runs after children.
- Changing the tree inside `_ready()` (e.g. `add_child`) re-enters children; a node can enter/exit the tree many times in its life, so connect signals in `_ready()` carefully (or use `CONNECT_ONE_SHOT`/redundancy checks).
- **Frame timing:** `_process` runs per rendered frame; `_physics_process` runs on a fixed clock regardless of FPS. Keep physics reactions in `_physics_process`, visual/UI updates in `_process`.

## 3. Signals & the observer pattern

Nodes communicate via **signals** (Godot's built-in event system) instead of reaching into each other's guts. This is a first-class, recommended pattern.

```gdscript
signal health_changed(old_value: int, new_value: int)

func take_damage(amount: int) -> void:
    var old := health
    health -= amount
    health_changed.emit(old, health)

# elsewhere:
player.health_changed.connect(_on_player_health_changed)

func _on_player_health_changed(old_value: int, new_value: int) -> void:
    # UI/square/everything interested updates here
    pass
```

- Connect at runtime, or wire in the editor **Scene** dock. Connected signals that reference a freed node produce errors — disconnect in `_exit_tree()` or free connected objects' callables first.
- Use **callable references**: `.connect(callable)` — never string method names in 4.x (`"method_name"` strings were removed).
- Useful signal flags: `CONNECT_ONE_SHOT`, `CONNECT_DEFERRED` (safe for freeing during emission).

## 4. Resources — data as objects

- A **Resource** is a serialized data object (`ExtResource` in `.tscn`). Textures, `ShaderMaterials`, `Animation`, `GDScript`, `StyleBox`, dictionaries of config, and any custom `extends Resource` class.
- **Shared by reference**: the same resource instance is shared by every scene that references it. Editing it (without `.duplicate()`) affects all owners — a classic bug source.
- `preload()`/`load()` return the shared singleton instance of a resource. Use `resource.duplicate()` before mutating if a copy is intended.
- `.tres` / `.res` are resource files; `.tscn` is a scene file.

## 5. Autoloads (singletons) & global state

- Project settings → Autoload registers a scene/script as a global singleton node added to the tree root at startup. Available as `get_node("/root/Name")` or directly by name in GDScript.
- Use for: game state, event bus (a signal hub), settings, manager services.
- Caution: an autoload's `_ready()` runs at startup; heavy autoloads delay boot. Don't store scene-referencing state in autoloads that points at freed nodes.

## 6. Input system (Godot 4)

- **Input Map** (Project Settings → Input Map): assign named **actions** (e.g. `move_left`) to keys/buttons. Never check raw scancodes in gameplay code — use actions.
- Parse input via callbacks, not polling:
```gdscript
func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("jump"):
        jump()

func _physics_process(_delta: float) -> void:
    var dir := Input.get_axis("move_left", "move_right")  # -1..1
    velocity.x = dir * SPEED
```
- **Event flow**: `_input` (all nodes) → GUI → `_unhandled_input` (nodes/node groups) → default. `set_process_unhandled_input(true)` gates it. `_unhandled_input` is the right place for player/aim handling so UI panels can consume events first.
- **InputEvent types**: `InputEventMouseMotion`, `InputEventKey` (with `echo` repeats), `InputEventScreenTouch`, etc. Check `event.is_action_pressed("x")` vs `is_action_pressed("x", true)` for repeat filtering.

## 7. Physics & movement (Godot 4)

- **CharacterBody2D/3D** — kinematic mover with `move_and_slide()`. Detect ground with `is_on_floor()`, walls with `get_slide_collision()`.
```gdscript
extends CharacterBody2D
var gravity := ProjectSettings.get_setting("physics/2d/default_gravity")
func _physics_process(delta: float) -> void:
    if not is_on_floor():
        velocity.y += gravity * delta
    move_and_slide()
```
- **RigidBody** — fully simulated by the physics engine (add forces, read velocities). Use `apply_central_force`, respect `sleeping`.
- **StaticBody** — immovable collider (walls, floors). `Area2D/3D` — detection zones (no collision response) exposing `body_entered`/`body_exited` signals for triggers, pickups, damage zones.
- **Collision layers/masks**: two ints (bitfields). A body collides with what's on its mask where layer bits match. Bit 1 = value 1, bit 2 = value 2, bit N = `2^(N-1)`.
- Always put both a **collision shape** (actual geometry, `CollisionShape2D`) and a **body** node together; shapes need a parent body.
- **Physics tuning note**: the fixed tick (default 60) runs in `_physics_process`; interpolation smooths rendering between ticks.

## 8. Rendering & scene graph

- Screen space is 2D; a `Camera2D` (or 3D `Camera3D`) determines what renders. `CanvasLayer` lets UI/overlays render above gameplay regardless of world position (put HUD there, with a `Control`).
- **Z-order**: `z_index` and `z_as_relative` control draw order among overlapping 2D nodes; `y_sort` gives depth by Y (top-down games).
- Nodes only render when relevant (culled). Overdraw and too many draw calls (batches) are the main 2D cost; use texture atlases, `Sprite2D`, and `Label` efficiently.
- 3D rendering is a raster/forward+ pipeline driven by **materials** (see the `godot-gdshader` skill). Lighting, shadow settings, and anti-aliasing are project settings.

## 9. Project structure & the editor's data model

Reference layout most Godot projects follow:
```
project.godot          # project config (input map, autoloads, renderer, version)
icon.svg
scenes/                # .tscn scene files
scripts/               # .gd scripts
assets/ (or art/, audio/, fonts/)
shaders/               # .gdshader files
addons/                # plugins
export_presets.cfg     # built when you set up export
```
- `project.godot` is the source of truth for config; it's a plain key/value file (keep it in version control, resolve merge conflicts carefully).
- Changing default renderer (Compatibility / Mobile / Forward+), autoloads, input actions, or physics defaults all live here.
- `.import` files are generated for imported assets (don't hand-edit; reimport in the editor).

## 10. Common engine pitfalls cheat-sheet

| Symptom | Likely cause |
|---|---|
| "Method not found" at runtime | String-based calls in 4.x, or calling before `_ready` (node not in tree) |
| Freeing → "Object was freed" | Keeping a reference after `queue_free()`; free/disconnect references in `_exit_tree()` |
| Resource edited everywhere at once | Sharing a Resource instance (see §4) — `.duplicate()` it |
| Input eaten / not firing | Wrong callback — use `_unhandled_input`; a `Control` with `mouse_filter` STOP on top |
| `is_on_floor()` false when standing still | Not applying gravity + `move_and_slide`, or checking before a physics step |
| Node "not found in the current scene" | Stale path after reparenting — prefer `%UniqueName` |
| Duplicated/corrupted child after copy-paste | In-editor, prefer instancing + "Save Branch as Scene" |

## 11. Godot 4.x (latest) — confirmed facts

This project targets the latest Godot 4.x (`project.godot` `config_version=5`). Expect: typed signals, `@export`, `CharacterBody2D`/`RigidBody2D`/`Area2D`, `%Name` unique nodes, callable-only connects, and `move_and_slide()` on `CharacterBody2D` (velocity built-in). No 3.x APIs are in scope.

## Output guidance
When answering, ground your logic in this model: name the nodes/tree/paths, the signal(s), the lifecycle callback, and the API involved. If something "should work," walk through the tree and the order of operations to find where reality diverges — that's almost always where the bug is. Prefer engine-native idioms over hacks; if you must work around a limitation, say exactly why.

