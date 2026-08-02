---
name: godot-ui
description: Building and editing UI in Godot 4.x — Control nodes, containers, anchors & layout modes, the Theme/StyleBox theming system, viewport scaling for responsive UI, ui_* input actions, focus navigation, popups/tooltips, and building dynamic UIs from GDScript. Use when creating or debugging menus, HUDs, settings screens, dialogs, or any user interface.
---

# Godot UI — Control Nodes, Layout & Theming

Complete guidance for building interfaces in **Godot 4.x (latest)**, both in the editor and from GDScript. UI in Godot is built from **`Control` nodes** living on a **`CanvasLayer`** or as children of a `Control` root. Works with the `godot-engine`, `godot-gdscript`, and `godot-gdshader` skills.

## 1. The UI model: Control nodes

- **`Control`** is the base UI node. Everything visible/interactive (buttons, labels, panels, sliders, text boxes) extends it. Key traits inherited by all controls: **anchors, offsets, size, and a position in screen space**, plus `gui_focus`, tooltips, and mouse behaviour (`mouse_filter`).
- Common built-in controls: `Button`, `Label`, `Panel`/`PanelContainer`, `TextureRect`, `LineEdit`, `TextEdit`, `RichTextLabel`, `ProgressBar`, `HSlider`/`VSlider`, `CheckButton`/`CheckBox`, `OptionButton`, `SpinBox`, `ColorRect`, `ScrollContainer`.
- **`Control` vs `Node2D`:** use `Control` (UI units, percent/px layout) for interfaces; use `Node2D`/`Sprite2D` only for world-space elements. Don't fight the UI system by hand-placing sprites as a "poor man's UI."
- A UI scene is typically a `Control` (or `CanvasLayer`) root containing a tree of controls.

## 2. Layout: containers vs anchors

Two complementary layout tools:

### Containers (automatic layout)
Containers arrange their children automatically and recompute on resize — the sensible default for menus and screens:
- `VBoxContainer` / `HBoxContainer` — stack children vertically / horizontally.
- `GridContainer` — grid by column count.
- `MarginContainer` — fixed padding around a single child.
- `CenterContainer` — center a child.
- `PanelContainer` — a styled backdrop that wraps its child (`Panel` + container in one).
- `ScrollContainer` — scrollable viewport for oversized content.

Inside a box container, children control their share via `size_flags_horizontal/vertical` (`SIZE_EXPAND`, `SIZE_FILL`, `SIZE_SHRINK_CENTER`, `SIZE_EXPAND_FILL`) and `custom_minimum_size` (min size in px). Example menu:
```gdscript
# build a centered column of buttons over a backdrop (code form)
var root := Control.new()
var margin := MarginContainer.new()
root.add_child(margin)
var col := VBoxContainer.new()
margin.add_child(col)
for label in ["New Game", "Options", "Quit"]:
    var b := Button.new()
    b.text = label
    b.pressed.connect(func(): print("pressed ", label))
    col.add_child(b)
```

### Anchors & offsets (manual layout)
For full control a control pins to its parent via **anchors** (0..1 fractions) + **offsets** (px from the anchor):
- `anchor_left/right/top/bottom` in `[0,1]`. E.g. `anchor_right=1` pins the right edge to the parent's right.
- Offsets = px inset from each anchor. **Common presets** (the anchor presets dropdown): Full Rect (0,0,1,1), Center (0.5,0.5,0.5,0.5), Top Wide, Bottom, Left, Right, etc.
- `Control.set_anchors_preset(Control.PRESET_FULL_RECT)` applies a preset programmatically.
- **Layout modes:** "Anchors" (manual, default) vs "Container" (auto once placed in a container). Setting a control's `layout_mode` to `1` (anchors) or `2` (container) controls how the editor exposes offsets. In code, use `custom_minimum_size` + `size_flags` inside containers, and anchors+offsets for free-positioned controls.

## 3. The three UI zones (scales/layers)

For crisp, responsive UIs:
- **CanvasLayer** — put UI here so it never scrolls/moves with the world (HUD always on screen).
- **A single root `Control`** with **Full Rect** anchors is the universal full-screen container; add a `Control`-scaling/`canvas_items` stretch so UI scales with resolution (see "Viewport scaling" below).
- Mixed scale zones: `Control.SCALE_MODE_*` lets one canvas mix pixel-perfect and scalable zones (advanced).

## 4. Styling: Theme, StyleBox, and selectors

Godot separates **structure/logic** (Control nodes) from **appearance** (Theme):
- A **`Theme`** resource holds styles for many `Control` types at once — the "skinning" system. One theme can style every button/label in the project. Set it node-locally, on a sub-tree, or globally via **Project Settings → GUI → Theme → Custom** (and `ThemeDB` in code).
- **`StyleBox`** (rectangular appearance: fill, borders, corner radius, shadows) is the atomic visual unit. Types: `StyleBoxFlat` (fills + rounded corners + borders), `StyleBoxTexture` (image-based), `StyleBoxLine`.
- Controls expose **type variations** (`Button`, `Button/Skill`, etc.) and a huge set of **theme items** (colors, fonts, icons, styleboxes, constants like `font_size`). Each control can override via `add_theme_*_override()`:
```gdscript
btn.add_theme_color_override("font_color", Color(1, 0.8, 0.2))
btn.add_theme_stylebox_override("normal", my_stylebox)   # states: normal/hover/pressed/focus/disabled
lbl.add_theme_font_size_override("font_size", 22)
```
- **State-based styling**: buttons have styleboxes per state (`normal`, `hover`, `pressed`, `focus`, `disabled`); sliders have `grabber`, `track`; `ProgressBar` has `fill`/`background`.
## 5. Viewport scaling & responsive UI (Godot 4.x)

Project Settings → Display → Window controls resolution scaling so UI/2D looks right at any window size:
- **Stretch Mode = `canvas_items`** — the whole canvas scales from a base resolution you choose (e.g. 1920×1080); UI and world scale together. Best all-round default.
- **Stretch Aspect** = `keep` (letterbox), `expand`, or `keep_width/height` — pick how it fills the window.
- For **pixel-art**: `viewport` stretch with integer scaling + a low base resolution, and `canvas_items` only for UI that should stay crisp at native res.
- In code, read the effective size with `get_viewport().get_visible_rect().size` and react to resize via `get_viewport().size_changed`.
- **Fonts**: use `theme_override_fonts/font` with dynamic font sizes and enable font fallbacks so text scales cleanly.

## 6. Input, focus & navigation

- UI uses the built-in `ui_*` actions — `ui_left/right/up/down`, `ui_accept`, `ui_cancel`, `ui_focus_next/prev`, `ui_select`, `ui_text_*` — mapped automatically and remappable. Buttons respond to `ui_accept` when focused; `ui_cancel` is the standard "back/close."
- **Focus**: controls accept focus (keyboard/gamepad) automatically where relevant. Control tab order via `focus_neighbor_*` and `focus_next/prev`; ensure menus are navigable by full keyboard.
- **Mouse**: `mouse_filter` = `IGNORE` (clicks pass through — set on decorative panels), `STOP` (consume), `PASS` (consume + bubble). A full-screen `Control` with STOP blocking input is a classic "why won't my game take input" bug — set transparent overlays to IGNORE.
- **Guiding events**: capture clicks with button `pressed`/`button_down`/`toggled`, `LineEdit.text_changed`, `Slider.value_changed`, `OptionButton.item_selected`, `CheckButton.toggled`.

## 7. Popups, dialogs & tooltips

- **`PopupMenu`** — dropdown/context menus; **`ConfirmationDialog`** / **`AcceptDialog`** for yes/no & info dialogs (connect to `confirmed` signal); **`FileDialog`** for open/save; **`PopupPanel`** for custom popovers.
- **Tooltips**: `Control.tooltip_text` shows a built-in tooltip; customize with a custom `Control.tooltip_text` + `make_custom_tooltip()` override.
- Hide/show with `popup()` / `popup_centered()` / `hide()`; use `popup_centered_clamped()` to keep dialogs on-screen. Popups belong to their own top-level window — set `exclusive` if you want a modal feel.
- Close on outside-click or `ui_cancel` via `allow_search`/`hide_on_parent_focus` properties.

## 8. Building dynamic UI from GDScript

Beyond static scenes, you often build UI at runtime (inventory, chat, settings rows):
```gdscript
func build_inventory(items: Array[ItemData]) -> void:
    var grid := GridContainer.new()
    grid.columns = 4
    for item in items:
        var cell := Button.new()
        cell.text = item.display_name
        cell.tooltip_text = item.describe()
        cell.pressed.connect(_on_item_clicked.bind(item))
        grid.add_child(cell)
    %InventoryPanel.add_child(grid)

func _on_item_clicked(item: ItemData) -> void:
    # handle click
    pass
```
- Instantiate with `Control.new()` (or `preload("res://scenes/ui/MyControl.tscn").instantiate()`), add as child, and **always free** with `queue_free()` when replacing to avoid leaks.
- Assign to unique nodes with `%Name` so you can update them from anywhere in code.
- Connect signals with bound args (`.bind(...)`) so a shared handler knows which item/widget fired.

## 9. Common UI pitfalls

| Symptom | Likely cause |
|---|---|
| UI scrolls/moves with camera | Forgot to put UI under a `CanvasLayer` |
| Buttons won't receive clicks | Parent `Control` has `mouse_filter` STOP and covers them; or `z_index`/draw-order puts something above |
| Game input dead after adding a panel | Full-rect `Control` with STOP intercepting everything — set IGNORE |
| Layout jumps at different window sizes | No stretch mode set; base resolution vs window mismatch in Display settings |
| Text blurry on high-DPI | `canvas_items` scale off (or mix scales); enable editor font oversampling settings |
| Container children overlap | Missing `size_flags`/`custom_minimum_size`, or mixing manual offsets inside a container |

## 10. Godot 4.7 note
This skill targets the current latest stable, **Godot 4.7.x** (`project.godot` `config_version=5`). The 4.7 release's headline work was lighting & cameras; UI tooling (Theme editor, container/anchor presets, StyleBox controls) is mature across 4.x and as documented here. When someone cites a specific 4.7.x UI feature, verify it against `docs.godotengine.org` rather than assuming it exists.

## Output guidance
For UI work, always give layered guidance: (1) the Control/scene structure, (2) the layout approach (container vs anchor) and stretch mode, (3) theming via a shared Theme where useful, (4) the input/focus wiring, then (5) minimal runnable GDScript. Prefer built-in controls + a Theme over hand-drawn styling. Show both the editor steps and the equivalent code so the result is maintainable and can be built dynamically when needed.

