---
name: godot-gamedev
description: End-to-end Godot game development — project architecture, scene composition, autoloads, state machines, input/UI patterns, save systems, audio, animation, export/build, and optimization. Use when planning, building, or shipping a full game project in Godot rather than a single feature.
---

# Godot Game Development — From Idea to Ship

A practical playbook for building complete, well-architected Godot games. Works with the `godot-engine`, `godot-gdscript`, and `godot-gdshader` skills.

## 1. Architecture: composition over inheritance

Godot nudges you toward **scene composition**: build small, single-purpose scenes and nest/instance them, communicating via signals and autoloads.

- **One scene = one cohesive behaviour** (a `Player.tscn`, an `Enemy.tscn`, a `HUD.tscn`). Keep scenes small and self-contained.
- **Prefer composition**: a player made of child nodes (Sprite, CollisionShape, Camera, AudioPlayer) + a script, not a deep inheritance chain.
- **Autoloads for shared systems**: an `EventBus` (signal hub), a `GameState` (score/health/progress), a `SaveManager`, an `AudioManager`. This replaces messy global references and lets any node reach "the game" cleanly.
- **Signals are your API**: nodes announce (`emitted`) what happened; others listen. This keeps dependencies one-directional and testable.

A clean minimal skeleton:
```
autoload/
  EventBus.gd          # all cross-cutting signals live here
  GameState.gd
scenes/
  main/Main.tscn       # root scene
  player/Player.tscn
  ui/HUD.tscn
  ui/MainMenu.tscn
scripts/
resources/
assets/
```

## 2. State machines for gameplay logic

Finite state machines keep character/AI/UI logic readable. A minimal robust pattern:

```gdscript
# State.gd
class_name State
extends Node
func enter() -> void: pass
func exit() -> void: pass
func physics_update(_delta: float) -> void: pass

# Player.gd
extends CharacterBody2D
@onready var states: Node = $States
var current: State

func ready_sequence() -> void:
    current = states.get_node("Idle")
    current.enter()

func change_state(new_state: String) -> void:
    current.exit()
    current = states.get_node(new_state)
    current.enter()
```
Using **child `State` nodes** (not just enums) keeps each state's code isolated, and you can use `AnimationTree` states in parallel. For simple cases an enum + `match` (see the gdscript skill) is enough — pick by complexity.

## 3. Input, UI, and feel

- Define **action-based input** in the Input Map (never raw keys), and support keyboard + gamepad by mapping multiple inputs to one action.
- **UI tree** (Godot 4): `Control` nodes with containers (`VBoxContainer`, `HBoxContainer`, `GridContainer`, `MarginContainer`) + `Theme` for consistent styling. Anchors/containers make responsive layouts automatic.
- **CanvasLayer for HUD** so UI never scrolls with the world; pause the game with `get_tree().paused` and set `process_mode` on nodes (e.g. `PROCESS_MODE_ALWAYS` for a pause menu).
- **Feel (game feel)**: use `Tween` for juicy animations (bounce on land, hit flash via modulate/shader), screen shake via camera offset, hit-stop (brief `Engine.time_scale`), and audio feedback. These small touches define quality.
- **Camera**: `Camera2D` with `position_smoothing_enabled` and limits; for sidescrollers lock to gameplay bounds.

## 4. Save systems

Godot's idiomatic save = serialize game data to a `JSON`/`ConfigFile`/custom resource in `user://`.

```gdscript
func save_game(path: String = "user://save.json") -> void:
    var data := {
        "level": current_level,
        "player": {"pos": player.global_position, "health": player.health},
        "unlocked": unlocked_items,
    }
    var f := FileAccess.open(path, FileAccess.WRITE)
    f.store_string(JSON.stringify(data))

func load_game(path: String = "user://save.json") -> void:
    if not FileAccess.file_exists(path):
        return
    var f := FileAccess.open(path, FileAccess.READ)
    var data: Variant = JSON.parse_string(f.get_as_text())
    # -- apply back to nodes --
```
- `user://` is the platform-agnostic user-data dir (Workspace/user-data). Never write to `res://` (read-only at runtime).
- Serialize only data (not nodes); re-instantiate nodes on load. For simple games, `ConfigFile` is even easier than JSON.

## 5. Audio, animation, particles

- **Audio**: `AudioStreamPlayer`(s) on scene nodes or a pooled `AudioManager` autoload; `AudioStreamPlayer2D` for positional audio. Use buses (Master/Music/SFX) from the Audio tab so players can mute SFX from music.
- **Animation**: `AnimationPlayer` for arbitrary property animation (UI, doors, cutscenes); `Tween` for short scripted changes; `AnimationTree` for blend trees (character locomotion). Use `animation_finished` signal to chain to gameplay code.
## 6. Performance & optimization

Optimize only when needed — but design to avoid the common traps:

- **Draw calls**: batch 2D via texture atlases; reuse sprites; combine meshes in 3D. Use `RemoteTransform`/`MultiMeshInstance2D` for many identical objects.
- **Physics**: keep rigid bodies sleeping; avoid many `Area2D` overlapping; use `PhysicsServer2D` queries (`intersect_point`, `space_state`) instead of many body nodes where appropriate.
- **Processing**: disable unneeded processing (`set_process(false)`), stop hidden cameras/timers. Avoid allocations and `get_node` string lookups in hot loops (cache references in `@onready`).
- **Loading**: `preload()` at compile time for always-needed assets; `load()` lazily for big/optional content; consider resource deduplication for large open worlds.
- **Use the editor's profilers**: the Debugger → Monitors and GPU/CPU profilers show hot spots. Profile on the actual target (mobile is dramatically slower than desktop).
- **Rendering defaults**: adjust `max dynamic lights`, shadow settings, `hdr`, and anti-aliasing in Project Settings (rendering) for the target hardware.

## 7. Export & shipping

- Set up **export presets** (Project → Export) for each platform (Windows, Linux, macOS, web, Android, iOS). Install the platform templates via the Godot version manager / templates dialog.
- **Export a release-optimized build**: choose the appropriate renderer at export (`--rendering-driver`), strip dev-only logging (`OS.is_debug_build()` guards), and check `export_presets.cfg` into version control.
- **Version control**: keep `project.godot`, scripts, scenes, and assets in git; `.godot/` (editor cache) should be gitignored. Use `.gdignore` for folders to exclude.
- **Web export**: serve the generated `.html` + `.pck`; mind wasm size and that GDScript is compiled to wasm. Enable compression/export options to shrink the pck.
- **Mobile targets**: test on device — GPU/memory differ hugely from desktop; keep vertex counts and draw calls low, manage memory via resource unloading.

## 8. Project lifecycle checklist

1. **Prototype** — prove the core loop with placeholder art; validate fun before polish.
2. **Structure** — set up folders, autoloads, input map, and git from day one.
3. **Build vertical slice** — one playable chunk (menus → gameplay → game-over → save).
4. **Polish & juice** — animation, audio, shaders, game feel.
5. **Test/balance** — iterate on difficulty, profiles, and bugs.
6. **Export & ship** — build presets, QA the release build, then stay in touch for patches.

## 9. Output guidance
When helping build a game, first clarify: engine version, genre/target platform, and whether it's 2D or 3D — these drive every decision (renderer, input, physics). Then give architecture guidance (scenes/autoloads/signals) before code, and supply runnable scripts. Prefer engine-native systems (scenes, signals, tweens, resources) over hand-rolled frameworks. Keep partial/prototype steps small so the user can run and verify as they go.

