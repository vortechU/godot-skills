---
name: godot-gdshader
description: Writing GDShader shading-language shaders in Godot 4 — shader types (spatial/canvas_item/particles/sky), vertex/fragment/light functions, uniforms, built-ins, and common techniques like outlines, dissolve, water, and distortion. Use when creating or debugging .gdshader files or ShaderMaterial in Godot.
---

# GDShader — Shading Language

GDShader is Godot's GLSL-like shading language (`.gdshader` files or embedded with `shader_type`). It compiles to the target renderer (Forward+, Mobile, Compatibility). Assume Godot 4.x.

## 1. Anatomy & types

```gdshader
shader_type canvas_item;          // spatial | canvas_item | particles | sky
render_mode unshaded, blend_mix;

uniform vec4 my_color : source_color = vec4(1.0);   // editable in Inspector

void fragment() {
    COLOR = my_color;
}
```

- **`shader_type`** must be the first line. Chooses the available built-ins and functions:
  - `spatial` — 3D meshes (`vertex`, `fragment`, `light`).
  - `canvas_item` — 2D items, sprites, UI (`vertex`, `fragment`, `light`).
  - `particles` — GPU particle logic (`start()`, `process()`).
  - `sky` — sky/background (`fragment`).
- **`render_mode`** sets state flags, e.g. `unshaded` (ignore lighting), `blend_mix|add|sub|mul` (blend mode), `cull_back|front|disabled` (backface culling), `depth_draw_always|never`.
- **Functions:** `vertex()`, `fragment()`, `light()` run per-vertex / per-pixel / per-light respectively. You can omit any of them.

## 2. Types & built-ins in `fragment()` (2D, most common)

- `vec2`/`vec3`/`vec4` (with swizzling `uv.xy`), scalars `float`, `int`, `bool`. Construct with `vec3(1.0, 0.0, 0.0)` or `vec3(1.0)` (splat).
- 2D canvas built-ins:
  - `UV` — texture coordinate (0..1). `FRAGCOORD` — screen pixel coordinate.
  - `COLOR` — the output colour (RGBA). Multiply by texture: `COLOR = texture(TEXTURE, UV);`
  - `TEXTURE` — the sprite's texture. `texture(tex, uv)` samples a `sampler2D` uniform.
  - `VERTEX` (in `vertex()`: position); in 2D you can animate with `VERTEX.y += ...`.
  - `TIME` — elapsed shader time (seconds since shader started) — the basis for most animation.
  - `SCREEN_UV`, `SCREEN_TEXTURE` — read the already-drawn screen (for distortion, holograms).
- 3D spatial built-ins: `NORMAL`, `ALBEDO`, `METALLIC`, `ROUGHNESS`, `EMISSION`, `SPECULAR`, `ALPHA`, `VIEW` (view direction), `POSITION`, `VERTEX`, `UV`, `NORMAL_MAP`.
- **3D output discipline:** in `fragment()` you set material properties (`ALBEDO`, `ROUGHNESS`, `METALLIC`, `EMISSION`, `ALPHA`) rather than `COLOR` (that's 2D-only). In `light()` you accumulate `DIFFUSE_LIGHT` / `SPECULAR_LIGHT`.

## 3. Common 2D techniques

**Uniform/colour tint:**
```gdshader
shader_type canvas_item;
uniform vec4 tint : source_color = vec4(1.0);
void fragment() {
    COLOR = texture(TEXTURE, UV) * tint;
}
```

**Outline (edge detection via neighbour sampling + alpha):**
```gdshader
shader_type canvas_item;
uniform vec4 outline_color : source_color = vec4(0.0, 0.0, 0.0, 1.0);
void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    float a = tex.a;
    float edge = 0.0;
    // textureSize() returns ivec2 — cast to float before dividing.
    float spread = 2.0 / float(textureSize(TEXTURE, 0).x);   // pixel size
    edge = max(edge, texture(TEXTURE, UV + vec2(spread, 0.0)).a);
    edge = max(edge, texture(TEXTURE, UV - vec2(spread, 0.0)).a);
    edge = max(edge, texture(TEXTURE, UV + vec2(0.0, spread)).a);
    edge = max(edge, texture(TEXTURE, UV - vec2(0.0, spread)).a);
    COLOR = mix(tex, outline_color, edge * (1.0 - a));
}
```

**Dissolve (animated cutout using a noise/pattern texture):**
```gdshader
shader_type canvas_item;
uniform sampler2D dissolve_texture : hint_white;
uniform float dissolve_amount : hint_range(0.0, 1.0) = 0.0;
uniform vec4 burn_color : source_color = vec4(1.0, 0.4, 0.1, 1.0);
void fragment() {
    float d = texture(dissolve_texture, UV).r;
    float is_visible = step(dissolve_amount, d);
    COLOR = texture(TEXTURE, UV);
    COLOR.a *= is_visible;
    // glowing edge:
    float edge = smoothstep(dissolve_amount, dissolve_amount + 0.1, d) - smoothstep(dissolve_amount + 0.1, dissolve_amount + 0.2, d);
    COLOR.rgb = mix(COLOR.rgb, burn_color.rgb, edge);
}
```

## 4. Vertex shaders & vertex animation

`vertex()` runs before `fragment()` and lets you move geometry. In spatial shaders:
```gdshader
shader_type spatial;
uniform float wave_strength : hint_range(0.0, 2.0) = 0.5;
void vertex() {
    VERTEX.y += sin(VERTEX.x * 5.0 + TIME * 3.0) * wave_strength;
}
```

In 2D canvas shaders, `VERTEX` is in canvas (pixel) space — animate sprites' vertices directly, and remember to transform normals if you move geometry (2D is usually fine without).

## 5. Particles shaders (GPU)

```gdshader
shader_type particles;
uniform float spread = 1.0;
void process() {
    // RESTART when a particle spawns (flag: use `if (RESTART)` or the restart variant)
    if (RESTART) {
        VELOCITY = vec3(0.0, 1.0, 0.0) * spread;
    }
    // accelerate downward over life:
    VELOCITY.y -= 9.8 * DELTA;
    TRANSFORM = TRANSFORM * translate(VELOCITY * DELTA);
}
```
Key particle built-ins: `RESTART` (true on spawn), `TRANSFORM`, `VELOCITY`, `LIFETIME`, `DELTA` (sim step), `CUSTOM`. Rendering/materials live on the GPU process' `ParticlesMaterial` or a `ShaderMaterial` on the mesh.

## 6. Using shaders in your project

- Create a **ShaderMaterial** on a node (`Sprite2D`, `MeshInstance3D`, `CanvasItem`), attach a `.gdshader` resource or write inline.
- **Uniforms** become Inspector fields. Set them from code:
  ```gdscript
  sprite.material.set_shader_parameter("dissolve_amount", progress)
  ```
  (or `$Cell.material.` via an exported `ShaderMaterial`). Never mutate `material` from many places without `.duplicate()` if sharing.
- **Editor hint:** `: hint_range`, `: source_color`, `: hint_white`, `: hint_albedo`, `: hint_normal`, `: hint_roughness` make uniforms designer-friendly.
- Shaders must be valid for the **current renderer**; the Compatibility renderer has a smaller feature set than Forward+ (e.g. no compute-capable particles, limited instancing). Match precision: prefer `mediump`/`highp` care only for mobile.

## 7. Debugging shaders
- A shader that fails to compile shows a red error in the Shader editor — the reported line usually points at the real culprit; fix the first error, recompile.
- Verify uniforms are actually exposed: `material.get_shader_parameter("name")`.
- For subtle visual issues, isolate: comment out `render_mode`, simplify to a flat color, then reintroduce parts.
- Remember `TIME` is in seconds and always increasing — retrigger animations with `fposmod(TIME, period)`.

## 8. Output guidance
Write complete shaders with a `shader_type`, sensible `render_mode`, and `uniform`s with good `: hint_*` defaults so they're usable from the Inspector. Explain which function (`vertex`/`fragment`/`light`) does the work. When giving GLSL-style math, keep it concise and correct for the chosen `shader_type`. Flag any feature that won't work on the Compatibility renderer if relevant to the user's target platform.

