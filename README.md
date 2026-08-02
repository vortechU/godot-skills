# Godot Skills

A set of skills for building games in **Godot 4.x (latest — currently 4.7)** with GDScript, GDShader, and the engine's systems. No 3.x support.

## Skills

| Skill | Focus |
|---|---|
| **godot-index** | Entry point + quick-reference cheat sheet. Routes a task to the right skill. |
| **godot-engine** | Mental model of the engine: scene tree, nodes, lifecycle, signals, resources, autoloads, physics, input, rendering. |
| **godot-gdscript** | Typed GDScript, signals, idioms, resource scripts, conventions. |
| **godot-gdshader** | GDShader shading language: types, uniforms, vertex/fragment/light, recipes (outline, dissolve, water, particles). |
| **godot-gamedev** | Full game development: architecture, state machines, save/load, export, optimization. |
| **godot-ui** | Control nodes, containers, anchors, Theme/StyleBox, responsive viewport scaling, dynamic UI. |

## Install

```bash
# one skill
npx skills add vortechU/godot-skills@godot-ui

# all skills
npx skills add vortechU/godot-skills --skill '*'
```

You can also install from the git URL: `npx skills add https://github.com/vortechU/godot-skills`


## Usage
The skills are written so that an agent loads them automatically when you ask something Godot-related. `godot-index` is the catch-all entry point.

## License
MIT
