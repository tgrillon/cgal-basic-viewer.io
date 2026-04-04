# CGAL — GLFW Basic Viewer Wiki

## What is the GLFW Basic Viewer?

The GLFW Basic Viewer is a lightweight, interactive 3D/2D visualization component built for the CGAL library (Computational Geometry Algorithms Library). It lets you display any CGAL geometric data structure — surface meshes, point sets, linear cell complexes, arrangements, and more — in a real-time OpenGL window with no boilerplate.

It is the second backend for the CGAL `Basic_viewer` infrastructure, alongside the older Qt backend. Both share the same `Graphics_scene` data model but are otherwise independent. The GLFW backend has no dependency on Qt: it uses GLFW for windowing, glad for OpenGL loading, and Eigen for math — all vendored directly under `include/CGAL/GLFW/vendor/`.

### Key capabilities

- Interactive orbit and free-fly camera
- Phong-shaded faces, cylinder edges, sphere vertices
- Interactive clipping plane with four display modes
- Keyframe-based camera animation 
- Screenshot / headless rendering (no display required)

### Table of Contents

| Page | When to read it |
|------|----------------|
| [Getting Started](#page-2--getting-started) | First time setup, running examples |
| [Architecture Overview](#page-3--architecture-overview) | Understanding how the pieces fit |
| [Graphics Scene](#page-4--graphics-scene) | How to populate a scene with geometry |
| [Rendering Pipeline](#page-5--rendering-pipeline) | How a frame is produced |
| [Shader System](#page-6--shader-system) | GLSL shaders, lighting model |
| [Camera System](#page-7--camera-system) | Orbit, free-fly, projection modes |
| [User Interaction](#page-8--user-interaction) | Full keyboard/mouse reference |
| [Clipping Plane](#page-9--clipping-plane) | Section cuts through geometry |
| [Animation System](#page-10--animation-system) | Keyframe camera animation |
| [Configuration & Settings](#page-11--configuration--settings) | Compile-time and runtime tunables |
| [Extending the Viewer](#page-12--extending-the-viewer) | New data structures, custom colors, screenshots |
| [Known Limitations & TODO](#page-13--known-limitations--todo) | Open issues and planned work |

---

## Page 2 — Getting Started

### Dependencies

| Dependency | Role | Provided |
|------------|------|----------|
| CGAL | Geometry kernel, `Graphics_scene` | External (required) |
| OpenGL 4.3+ | Rendering API | System |
| GLFW 3.x | Window, input, OpenGL context | Vendored under `include/CGAL/GLFW/vendor/glfw/` |
| glad | OpenGL function loader | Vendored under `include/CGAL/GLFW/vendor/glad/` |
| Eigen 3 | Linear algebra (`vec3f`, `mat4f`, `quatf`) | External (required, part of CGAL) |
| stb_image_write | PNG/JPEG screenshot writing | Vendored (single-header) |

Because GLFW and glad are vendored, you do not need to install them separately. On Linux you still need the system OpenGL and X11/Wayland dev libraries for GLFW to link against.

### Building with CMake

The examples in `examples/Basic_viewer/` demonstrate the recommended CMake setup.

```cmake
cmake_minimum_required(VERSION 3.1...3.23)
project(my_viewer)

find_package(CGAL REQUIRED COMPONENTS Core)
find_package(Eigen3 3.1.0 REQUIRED)

add_executable(my_program main.cpp)
target_link_libraries(my_program PRIVATE CGAL::CGAL Eigen3::Eigen)
```

From the build directory:

```sh
cmake .. -DCMAKE_BUILD_TYPE=Release
make draw_surface_mesh_height
./draw_surface_mesh_height path/to/mesh.off
```

### Running the bundled examples

| Binary | What it shows |
|--------|--------------|
| `draw_surface_mesh_height` | Surface mesh colored by vertex height via `Graphics_scene_options` |
| `draw_surface_mesh_small_faces` | Surface mesh with small faces highlighted |
| `draw_surface_mesh_vcolor` | Surface mesh with per-vertex colors |
| `draw_mesh_and_points` | Mixed scene: a mesh and a point cloud together |
| `draw_several_windows` | Two independent viewer windows running in the same process |
| `basic_viewer_glfw_screenshot` | Headless rendering — writes a PNG without showing a window |

### Minimal usage pattern

```cpp
#include <CGAL/Surface_mesh.h>
#include <CGAL/draw_surface_mesh.h>

typedef CGAL::Simple_cartesian<double> Kernel;
typedef CGAL::Surface_mesh<Kernel::Point_3> Mesh;

int main()
{
  Mesh sm;
  CGAL::IO::read_polygon_mesh("elephant.off", sm);

  // Option 1: one-liner
  CGAL::draw(sm);

  // Option 2: more control
  CGAL::Graphics_scene scene;
  CGAL::add_to_graphics_scene(sm, scene);

  CGAL::Basic_viewer bv(scene, "My Viewer");
  bv.draw_vertices(true);
  bv.show();

  return 0;
}
```

`CGAL::draw()` is a thin wrapper that creates the `Graphics_scene`, fills it, constructs a `Basic_viewer`, and calls `show()`.

---

## Page 3 — Architecture Overview

### Component map

```
┌─────────────────────────────────────────────────────────────────┐
│  User Application                                               │
│    CGAL::draw(mesh)  /  Basic_viewer bv(scene)                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                    Graphics_scene
               (CPU-side geometry buffers)
                            │
                    Basic_viewer   ◄─── inherits from: Input
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
      Camera          Clipping_plane   Animation_controller
    (Orbiter /          (Line_renderer      (keyframe
     FreeFly)            + plane math)     SLERP/LERP)
         │
      Shader ×11
   (GLSL programs,
    compiled at init)
         │
      VAO[5] + VBO[N]
   (GPU geometry buffers)
         │
    OpenGL 4.3 (via glad)
    GLFW window + input
```

### Namespace layout

```
CGAL::
  GLFW::
    Basic_viewer        ← the renderer (Basic_viewer.h + Basic_viewer_impl.h)
  Graphics_scene        ← scene data container (Graphics_scene.h)
  Graphics_scene_options← coloring/visibility policy (Graphics_scene_options.h)
  Buffer_for_vao        ← CGAL geometry → float arrays (Buffer_for_vao.h)

(internal, not in CGAL::)
  Camera                ← Camera.h
  Input                 ← Input.h
  Clipping_plane        ← Clipping_plane.h
  Animation_controller  ← Animation_controller.h
  Line_renderer         ← Line_renderer.h
  Shader                ← Shader.h
  VAO / BufferObject    ← buffer/VAO.h, buffer/BufferObject.h
```

### Header-only design

Nearly all code lives in `.h` files as inline or template implementations. There is no separately compiled `.cpp` to link — including `Basic_viewer.h` transitively pulls in `Basic_viewer_impl.h`, all internal headers, `Basic_shaders.h` (GLSL source as `const char[]`), vendored GLFW and glad. This makes integration into a CGAL package straightforward at the cost of longer initial compile times.

### Math types (Eigen)

All vector/matrix math uses Eigen aliases defined in `utils.h`:

```cpp
using vec2f = Eigen::Vector2f;
using vec2i = Eigen::Vector2i;
using vec3f = Eigen::Vector3f;
using vec4f = Eigen::Vector4f;
using mat4f = Eigen::Matrix4f;
using quatf = Eigen::Quaternionf;
```

---

## Page 4 — Graphics Scene

### Role

`Graphics_scene` (in `include/CGAL/Graphics_scene.h`) is a pure data container. It holds CPU-side float arrays that will be uploaded to the GPU. The viewer (`Basic_viewer`) reads from it but never writes to it.

You populate a `Graphics_scene` by calling the `add_to_graphics_scene()` free function, which is specialized for each CGAL data structure (`Surface_mesh`, `Polyhedron_3`, `Point_set_3`, `Linear_cell_complex`, `Arrangement_2`, etc.).

### Primitive types

The scene organizes geometry into five primitive types:

| Type | Description | Typical use |
|------|-------------|-------------|
| Points | 3D point positions | Vertices, point clouds |
| Segments | Pairs of 3D endpoints | Edges of meshes |
| Rays | Origin + direction | Half-lines, normals overlay |
| Lines | Infinite lines (stored as two points) | Axes, guides |
| Faces | Triangulated polygon faces | Mesh faces |

### Buffer layout

Each primitive type gets its own position buffer and color buffer. Faces also get two normal buffers (flat and smooth). The `BufferIndices` enum in `Graphics_scene` indexes into a flat array:

```cpp
enum BufferIndices {
  BEGIN_POS,
    POS_POINTS,
    POS_SEGMENTS,
    POS_RAYS,
    POS_LINES,
    POS_FACES,
  END_POS,
  BEGIN_COLOR,
    COLOR_POINTS,
    COLOR_SEGMENTS,
    COLOR_RAYS,
    COLOR_LINES,
    COLOR_FACES,
  END_COLOR,
  FLAT_NORMAL_FACES,
  SMOOTH_NORMAL_FACES,
  LAST_INDEX
};
```

The number of GPU VBOs is computed automatically from this layout:

```cpp
NB_GL_BUFFERS = (END_POS - BEGIN_POS) + (END_COLOR - BEGIN_COLOR) + 2; // +2 for normals
```

### Bounding box

`Graphics_scene` tracks the axis-aligned bounding box of all added geometry. `Basic_viewer` reads this at construction to auto-fit the camera via `Camera::lookat(pmin, pmax)`.

### 2D detection

`Graphics_scene::is_two_dimensional()` returns `true` when all z-coordinates are zero (or absent). The viewer uses this to automatically switch to an orthographic camera and lock rotation to roll-only.

### Populating a scene

**With a CGAL data structure:**

```cpp
CGAL::Graphics_scene scene;
CGAL::add_to_graphics_scene(my_mesh, scene);
// add more objects to the same scene:
CGAL::add_to_graphics_scene(my_point_set, scene);
```

**With `Graphics_scene_options` for custom colors:**

`Graphics_scene_options<DS, VertexDescriptor, EdgeDescriptor, FaceDescriptor>` is a template policy class. You override its lambda fields to control per-element visibility and color:

```cpp
struct My_options : public CGAL::Graphics_scene_options<Mesh, ...>
{
  My_options(const Mesh& sm)
  {
    // color each face according to height
    this->colored_face = [](const Mesh&, Mesh::Face_index) { return true; };
    this->face_color   = [](const Mesh& m, Mesh::Face_index fi) -> CGAL::IO::Color {
      // ... compute color from fi ...
      return CGAL::IO::Color(r, g, b);
    };
  }
};

// Use it:
CGAL::add_to_graphics_scene(mesh, scene, My_options(mesh));
```

The full `draw_surface_mesh_height.cpp` example demonstrates this pattern: it seeds a `CGAL::Random` from a face's average Y-height and returns a random color, producing a height-map visualization.

### Buffer_for_vao

`Buffer_for_vao` (in `include/CGAL/Buffer_for_vao.h`) is the utility that converts CGAL geometric types (points, segments, polygon faces, etc.) into the flat float arrays that `Graphics_scene` stores. Key responsibilities:

- Triangulating non-triangular faces (using constrained Delaunay triangulation for 2D, fan triangulation for 3D)
- Computing per-face flat normals (Newell's method for robustness with non-planar faces)
- Lifting 2D geometry to z=0 in 3D space

---

## Page 5 — Rendering Pipeline

### Frame lifecycle

The `show()` loop runs until the GLFW window is closed:

```
while (!glfwWindowShouldClose(window))
{
  deltaTime = currentTime - lastTime;

  handle_events(deltaTime)       ← poll GLFW, dispatch input actions
  if (need_update())
    render_scene(deltaTime)      ← update camera/clipping, draw, swap buffers
  print_application_state(...)   ← optional HUD to stdout
}
```

`need_update()` returns `true` when the camera or clipping plane still has pending motion (target ≠ current position/orientation), preventing needless redraws when the scene is static.

### render_scene()

```
render_scene(deltaTime)
  └─ camera.update(deltaTime)           ← smooth toward target state
  └─ clipping_plane.update(deltaTime)   ← same
  └─ animation: if running → apply keyframe pose to camera
  └─ compute_model_view_projection_matrix(deltaTime)
       └─ ModelMatrix     = Identity  (scene in world space)
       └─ ViewMatrix      = camera.view()
       └─ ProjectionMatrix= camera.projection(width, height)
       └─ ViewProjection  = Projection * View
  └─ draw(deltaTime)
  └─ glfwSwapBuffers()
```

### draw() — render order

Render order matters for correct blending of transparent clipping-plane mode:

1. Clear color and depth buffers
2. Faces (`draw_faces()`) — with optional clipping-plane logic
3. Edges (`draw_edges()`) — cylinder shader if `m_DrawCylinderEdge`, plain line otherwise
4. Rays (`draw_rays()`)
5. Lines (`draw_lines()`)
6. Vertices (`draw_vertices()`) — sphere shader if `m_DrawSphereVertex`, plain point otherwise
7. Clipping plane quad (`render_clipping_plane()`) — if enabled and visible
8. World axis (three colored lines at origin) — if enabled
9. XY grid — if enabled
10. Normals arrows — if `m_DisplayFaceNormal`
11. Triangle wireframe — if `m_DrawMeshTriangles`

### Clipping-plane face rendering

When a clipping plane is active, faces are drawn in two passes using `draw_faces_bis(mode)`:

| Mode | Triangles rendered |
|------|--------------------|
| `DRAW_INSIDE_ONLY` | Fragments where `dot(pos - planePoint, planeNormal) > 0` |
| `DRAW_OUTSIDE_ONLY` | Fragments where `dot(pos - planePoint, planeNormal) < 0` |
| `DRAW_ALL` | All fragments |

The `CLIPPING_PLANE_SOLID_HALF_TRANSPARENT_HALF` display mode renders inside solid, outside at `m_Transparency = 0.5` alpha. The `SOLID_HALF_WIRE_HALF` mode renders outside as a wireframe instead. No geometry pre-processing occurs — the decision is made per-fragment in the shader.

### VAO/VBO layout

```
VAO_POINTS   → VBO[POS_POINTS],   VBO[COLOR_POINTS]
VAO_SEGMENTS → VBO[POS_SEGMENTS], VBO[COLOR_SEGMENTS]
VAO_RAYS     → VBO[POS_RAYS],     VBO[COLOR_RAYS]
VAO_LINES    → VBO[POS_LINES],    VBO[COLOR_LINES]
VAO_FACES    → VBO[POS_FACES],    VBO[COLOR_FACES],
               VBO[FLAT_NORMAL_FACES], VBO[SMOOTH_NORMAL_FACES]
```

GPU buffers are allocated with `glGenBuffers` / `glGenVertexArrays` once in `initialize_buffers()`, then populated in `load_scene()` via `glBufferData(..., GL_STATIC_DRAW)`. The scene geometry is treated as static — a full `load_scene()` is needed to update it after construction.

### Screenshot mode

`make_screenshot(filePath)` runs a single frame without a visible window:

```
constructor calls initialize(screenshotOnly=true)
  → window created hidden (no event callbacks registered)
  → camera smoothness disabled (instant position)
make_screenshot(path)
  → draw()
  → glfwSwapBuffers()
  → capture_screenshot(path)   ← stb_image_write PNG
```

---

## Page 6 — Shader System

### Overview

All GLSL source code lives as `const char[]` string literals in `include/CGAL/Basic_shaders.h`. Shaders are compiled at viewer initialization time by `compile_shaders()`. There are no external `.glsl` files.

The system targets GLSL 1.50 (`#version 150`, OpenGL 3.2 core), but also provides compatibility variants for features that need OpenGL 4.3 (specifically `gl_PointSize` and some geometry shader features). The viewer detects the available version at runtime:

```cpp
if (openglMajorVersion > 4 || (openglMajorVersion == 4 && openglMinorVersion >= 3))
  m_IsOpengl4_3 = true;
```

and selects the appropriate shader source accordingly (e.g. `VERTEX_SOURCE_COLOR` vs `VERTEX_SOURCE_COLOR_COMP`).

### Shader inventory

| Shader variable | Stages | Purpose |
|----------------|--------|---------|
| `m_ShaderFace` | V + F | Phong-shaded mesh faces with clipping |
| `m_ShaderSphere` | V + G + F | Point → sphere expansion |
| `m_ShaderCylinder` | V + G + F | Segment → cylinder expansion |
| `m_ShaderPl` | V + F | Plain points/lines (no geometry shader) |
| `m_ShaderLine` | V + G + F | Variable-width lines via line-width geometry shader |
| `m_ShaderPlane` | V + F | Clipping plane quad |
| `m_ShaderGrid` | V + G + F | World axis / XY grid via arrow geometry shader |
| `m_ShaderArrow` | V + G + F | Normal arrows (world axis, helpers) |
| `m_ShaderNormal` | V + G + F | Per-face normal display arrows |
| `m_ShaderTriangles` | V + G + F | Triangle wireframe overlay |

### Phong lighting (face shader)

The face fragment shader implements Blinn-Phong lighting:

```glsl
vec3 L = normalize(u_LightPos.xyz - vs_fP.xyz);  // light direction
vec3 V = normalize(-vs_fP.xyz);                   // view direction
vec3 R = reflect(-L, normal);                     // reflection

vec4 diffuse  = max(dot(normal, L), 0.0) * u_LightDiff * fColor;
vec4 ambient  = u_LightAmb * fColor;
vec4 specular = pow(max(dot(R, V), 0.0), u_SpecPower) * u_LightSpec;

out_color = diffuse + ambient + specular;
```

The light position `u_LightPos` is passed as a `vec4`. When the w-component is `0`, it is treated as a directional light (direction = xyz); when `w = 1`, it is a point light. The default setting is `{0, 0, 0, 0}` — a directional light aligned with the camera.

### Clipping plane in shaders

Every face/point/line shader receives two uniforms: `u_ClipPlane` (plane normal as xyz, distance as w) and `u_PointPlane` (a point on the plane). The fragment position in local space (`ls_fP`) is tested:

```glsl
float onPlane = sign(dot(ls_fP.xyz - u_PointPlane.xyz, u_ClipPlane.xyz));
// onPlane =  1 → inside (solid side)
// onPlane = -1 → outside (transparent/wire side)

if (u_RenderingMode == (onPlane + 1) / 2) {
  discard;          // fragment on the wrong side
}
if (u_RenderingTransparency < 1.0) {
  out_color.a = u_RenderingTransparency;
}
```

The `u_RenderingMode` maps to the `RenderingMode` enum: `DRAW_ALL = -1`, `DRAW_INSIDE_ONLY = 0`, `DRAW_OUTSIDE_ONLY = 1`.

### Geometry shader use (sphere / cylinder)

Using geometry shaders to expand primitives avoids pre-tessellating the CPU-side data. A point in the VAO is a single `vec3`; the sphere geometry shader generates a camera-facing quad around it and the fragment shader shades it as a sphere. A segment is two `vec3`s; the cylinder geometry shader generates the six triangles of a rectangular tube. This trades some GPU computation for significantly smaller CPU-side buffers.

---

## Page 7 — Camera System

### Two types, two modes

The camera has two orthogonal axes of variation:

**Type** — how movement is interpreted:
- `ORBITER` (default): the camera revolves around a fixed center point. Zoom changes the distance to that center. Arrow keys pan the center laterally.
- `FREE_FLY`: the camera has a free position in space. Scroll moves forward along the viewing direction. Arrow keys move the camera position.

**Mode** — projection:
- `PERSPECTIVE` (default): `perspective(FOV=45°, aspect, znear, zfar)`. Natural for 3D.
- `ORTHOGRAPHIC`: `ortho(-halfW, +halfW, -halfH, +halfH, znear, zfar)`. Automatically selected for 2D scenes.

Toggle type with the `T` key at runtime, or call `camera_position()` / `scene_center()` before `show()`.

### Internal state

```
Orbiter position  = (m_Position.x, m_Position.y, m_Size)
                     ↑ lateral pan              ↑ distance

Free-fly position = (m_Position.x, m_Position.y, m_Position.z + m_Size)
```

Orientation is stored as a single quaternion `m_Orientation` (Eigen `Quaternionf`). Pitch and yaw are accumulated as float scalars and converted to quaternion increments each frame:

```cpp
// Orbiter: rotates in world space
quatf pitchQ(AngleAxisf(pitchDelta, UnitX));
quatf yawQ  (AngleAxisf(yawDelta,   UnitY));
m_Orientation = pitchQ * yawQ * m_Orientation;

// Free-fly: pitch around local right axis, yaw around world up (locks roll)
quatf pitchQ(AngleAxisf(pitchDelta, get_right()));
quatf yawQ  (AngleAxisf(yawDelta,   UnitY));
m_Orientation = m_Orientation * pitchQ * yawQ;
```

### Smoothing

Every quantity has a current value and a target value. Each frame, the current value is nudged toward the target by a smoothing factor:

```cpp
m_Size     += m_ZoomSmoothFactor        * (m_TargetSize     - m_Size);
m_Position += m_TranslationSmoothFactor * (m_TargetPosition - m_Position);
// rotation uses exponential smoothing on pitch/yaw scalars
```

Default smoothing factors (all configurable at runtime):

| Factor | Default | Range |
|--------|---------|-------|
| Zoom | 0.10 | 0.01 – 1.0 |
| Rotation | 0.25 | 0.01 – 1.0 |
| Translation | 0.25 | 0.01 – 1.0 |

A factor of `1.0` means instant (no smoothing). `disable_smoothness()` forces all to `1.0`, which is used by `make_screenshot()`.

### Near/far plane computation

The clip planes are computed from the scene radius to avoid precision issues:

```
znear = max(0.1,   distance(center, eye) - 2 * radius)
zfar  = max(100.0, distance(center, eye) + 2 * radius)
```

### Constraint axis

Rotation can be constrained to a single axis (cycle with `Shift+C`):

| Axis | Orbiter | Free-fly |
|------|---------|----------|
| None | Full rotation | Full rotation |
| Right | Pitch only | Pitch only |
| Up | Yaw only | Yaw only |
| Forward | Roll (pitch - yaw combined) | — (not available, skipped) |

### Align to nearest axis

`align_to_nearest_axis()` snaps the camera to the closest cardinal view (+X, -X, +Y, -Y, +Z, -Z) by comparing the current forward and up vectors against all six unit axes and computing the corresponding canonical quaternion. Useful for getting clean orthographic-like views without switching mode.

### Runtime API

```cpp
bv.scene_center({cx, cy, cz});       // set orbit center
bv.scene_radius(r);                   // set orbit radius
bv.camera_position({x, y, z});        // set position (z used as distance for orbiter)
bv.camera_orientation(forward, upAngle);
bv.zoom(z);                           // set distance/size directly
bv.two_dimensional();                 // switch to orthographic + forward-axis constraint
bv.align_camera_to_clipping_plane();  // orient camera to face the clipping plane
```

---

## Page 8 — User Interaction

### Input system architecture

`Basic_viewer` inherits from `Input`, which implements an action-based input system. The core idea: raw key/mouse events are never handled directly. Instead, they are matched against a registry of `InputBinding` objects, each associated with an abstract `ActionEnum` integer. When a binding fires, the virtual methods `start_action()`, `action_event()`, and `end_action()` are called on `Basic_viewer`.

This means:
- The same action can have multiple bindings (e.g., both keyboard and mouse trigger the same effect)
- Modifier keys (Ctrl, Alt, Shift) create priority-sorted multi-key chords
- Longer chords (3 keys) shadow shorter ones (1 key) for the same primary key — preventing ambiguity

`HOLD` bindings fire every frame while the key is held. `RELEASE` bindings fire once on press.

Double-click is detected by comparing timestamps: two presses of the same button within 250 ms triggers `double_click_event()`.

### Keyboard shortcuts reference

#### Application

| Key | Action |
|-----|--------|
| `Escape` | Exit |
| `H` | Print help to stdout (all bindings with descriptions) |
| `I` | Toggle application state display (HUD) |
| `F11` | Toggle fullscreen |
| `Ctrl+S` | Screenshot (save PNG to `cgal_basic_viewer.png`) |

#### Scene display

| Key | Action |
|-----|--------|
| `V` | Toggle vertices |
| `E` | Toggle edges |
| `F` | Toggle faces |
| `R` | Toggle rays |
| `L` | Toggle lines |
| `N` | Toggle face normal arrows |
| `O` | Toggle flat/smooth shading |
| `Alt+N` | Toggle per-vertex normals direction (reverse) |
| `Alt+E` | Toggle cylinder edge rendering |
| `Alt+V` | Toggle sphere vertex rendering |
| `M` | Toggle mono/per-element color |
| `Shift+N` | Toggle normal mono color |
| `G` | Toggle world axis display |
| `Shift+G` | Toggle XY grid display |
| `Shift+T` | Toggle face triangulation display |
| `Home` | Fit entire scene in view |

#### Lighting

| Key | Action |
|-----|--------|
| `+` / `-` | Increase/decrease all light components |
| `R` (hold) | Increase red component |
| `G` (hold) | Increase green component |
| `B` (hold) | Increase blue component |
| `Shift+R/G/B` | Decrease respective component |

#### Primitive sizes

| Key | Action |
|-----|--------|
| `Shift+Up` / `Shift+Down` | Increase/decrease vertex size |
| `Ctrl+Up` / `Ctrl+Down` | Increase/decrease edge size |

#### Camera

| Key / Mouse | Action |
|-------------|--------|
| Left mouse drag | Rotate camera |
| Middle mouse drag | Translate camera |
| Right mouse drag | Rotate clipping plane (when active) |
| Scroll | Zoom (orbiter) / move forward (free-fly) |
| Arrow keys | Translate camera (Up/Down/Left/Right) |
| `PageUp` / `PageDown` | Move forward/backward (free-fly) |
| `T` | Switch orbiter ↔ free-fly |
| `P` | Switch perspective ↔ orthographic |
| `Shift+C` | Cycle rotation constraint axis |
| `Backspace` | Reset camera and clipping plane |
| Double-click (left) | Change pivot/orbit center to clicked point |
| `Shift+←/→` | Increase/decrease translation speed |
| `Alt+←/→` | Increase/decrease rotation speed |
| `Ctrl+←/→` | Adjust rotation smoothness |
| `Ctrl+Alt+←/→` | Adjust translation smoothness |

#### Clipping plane

| Key | Action |
|-----|--------|
| `C` | Cycle clipping plane display mode (Off → Solid+Transparent → Solid+Wire → Solid only → Off) |
| `Alt+C` | Toggle clipping plane quad visibility |
| `Shift+Alt+C` | Cycle constraint axis for clipping plane (None → Right → Up) |
| Right mouse drag | Rotate clipping plane |
| `Shift+Right` | Translate along plane normal |
| `Ctrl+Right` | Translate along camera forward direction |
| `Shift+Alt+R` | Reset clipping plane |

#### Animation

| Key | Action |
|-----|--------|
| `Shift+S` | Save current camera pose as a keyframe |
| `Shift+A` | Play / stop animation |
| `Shift+X` | Clear all keyframes |

### QWERTY / AZERTY support

The viewer defaults to QWERTY. Calling `bv.azerty_layout()` (or `bv.set_keyboard_layout(AZERTY)`) remaps:

| AZERTY key | Treated as (QWERTY) |
|------------|---------------------|
| `A` | `Q` |
| `Q` | `A` |
| `Z` | `W` |
| `W` | `Z` |
| `,` | `M` |
| `M` | `;` |

### Adding custom bindings

```cpp
// In a subclass or directly:
add_keyboard_action({GLFW_KEY_X, GLFW_KEY_LEFT_SHIFT}, InputMode::RELEASE, MY_ACTION);
add_description(MY_ACTION, "My Section", "Does something cool");
```

The `add_description()` call registers the binding text automatically from the key codes, so `print_help()` will show the correct key names.

---

## Page 9 — Clipping Plane

### Concept

The clipping plane is an infinite mathematical plane that divides the scene into two half-spaces: "inside" and "outside." The viewer uses it to cut through geometry and inspect cross-sections without modifying any mesh data.

### Display modes

Cycling through `C` advances through four modes (stored in `DisplayMode`):

| Mode | Description |
|------|-------------|
| `CLIPPING_PLANE_OFF` | No clipping, full geometry visible |
| `CLIPPING_PLANE_SOLID_HALF_TRANSPARENT_HALF` | Inside half solid, outside half at 50% alpha |
| `CLIPPING_PLANE_SOLID_HALF_WIRE_HALF` | Inside half solid, outside half as wireframe |
| `CLIPPING_PLANE_SOLID_HALF_ONLY` | Only inside half rendered (hard cut) |

When mode is not `OFF`, the shader `u_RenderingMode` uniform switches between `DRAW_INSIDE_ONLY` and `DRAW_OUTSIDE_ONLY` across the two face draw passes.

The plane's visual quad (a colored rectangle showing plane position and orientation) is drawn by `render_clipping_plane()` using `m_ShaderPlane`. Its visibility is independent from the clipping mode and toggled separately with `Alt+C`.

### Implementation: Clipping_plane

`Clipping_plane` extends `Line_renderer` (to inherit VAO/VBO machinery for drawing the plane quad). Its internal state mirrors `Camera`: quaternion orientation, current and target position/pitch/yaw, and independent smoothing factors.

- Default position: origin `(0, 0, 0)`
- Default normal: +Z axis `(0, 0, 1)`

The plane equation passed to shaders is:

```cpp
vec4f m_ClipPlane  = {nx, ny, nz, 0};   // plane normal in world space
vec4f m_PointPlane = {px, py, pz, 1};   // point on the plane
```

`get_normal()` derives the world-space normal by rotating `UnitZ` by the orientation quaternion:

```cpp
return (m_Orientation * vec3f::UnitZ()).normalized();
```

`get_matrix()` produces a `mat4f` (translation × rotation) used to render the plane quad.

### Constraint axis

The clipping plane has its own two-state constraint (cycle with `Shift+Alt+C`):

| Constraint | Effect |
|------------|--------|
| None | Full rotation (pitch + yaw) |
| Right | Pitch only (tilt forward/back) |
| Up | Yaw only (spin in place) |

### Translation modes

Three independent translation modes:

1. **Along plane normal** (`Shift+Right` / `Shift+Left`): `TargetPosition -= Size * normal * speed`
2. **Along camera forward** (`Ctrl+Right` / `Ctrl+Left`): uses the camera's current forward vector as translation direction
3. **In-plane** (middle mouse drag while clipping plane active): translates along the plane's local Right and Up axes

### Runtime API

```cpp
bv.display_mode(DisplayMode::CLIPPING_PLANE_SOLID_HALF_TRANSPARENT_HALF);
bv.clipping_plane_orientation({0, 1, 0});     // horizontal cut (XZ plane)
bv.clipping_plane_translate_along_normal(1.5f);
bv.draw_clipping_plane(true);                  // show/hide plane quad
bv.align_camera_to_clipping_plane();           // snap camera to face plane
```

`clipping_plane()` returns a `CGAL::Plane_3<Local_kernel>` suitable for use in geometry algorithms.

---

## Page 10 — Animation System

### Concept

`Animation_controller` records a sequence of camera keyframes (position + orientation quaternion) and smoothly interpolates between them over a configurable duration to produce a camera flythrough.

### Data structure

```cpp
struct AnimationKeyFrame {
  vec3f position;     // camera position at this keyframe
  quatf orientation;  // camera orientation at this keyframe
};

std::vector<AnimationKeyFrame> m_KeyFrames;
```

### Workflow

1. Navigate the camera to a desired viewpoint.
2. Press `Shift+S` → `save_key_frame()` → calls `m_AnimationController.add_key_frame(camera.get_position(), camera.get_orientation())`.
3. Repeat steps 1–2 for each waypoint. At least 2 keyframes are required before playback can start.
4. Press `Shift+A` → `run_or_stop_animation()` → starts playback from the beginning.
5. Press `Shift+A` again to stop. Press `Shift+X` to clear all keyframes.

### Interpolation

The timestamp between consecutive frames is:

```
timestamp = total_duration_ms / (num_keyframes - 1)
```

Each frame of playback:

1. Compute elapsed time since `start()`.
2. Compute fractional keyframe index `t = elapsed / timestamp`.
3. Select the surrounding two keyframes: `floor(t)` and `ceil(t)`.
4. Local `t_local = t - floor(t)` in `[0, 1]`.
5. Interpolate:

```cpp
rotation    = keyframe0.orientation.slerp(t_local, keyframe1.orientation);
translation = lerp(keyframe0.position, keyframe1.position, t_local);
```

6. Apply to camera via `m_Camera.set_position(translation)` and `m_Camera.set_orientation(rotation)`.

When the end of the sequence is reached, the animation stops and resets to frame 0. It does not loop automatically.

### Timing

Default duration: 5 seconds.

```cpp
bv.animation_duration(std::chrono::milliseconds(10000)); // 10s tour
```

The timestamp is recomputed whenever `set_duration()` is called or a new keyframe is added, so keyframes are always evenly spaced in time regardless of when you adjust the duration.

### Camera mode note

Animation drives the camera regardless of whether it is in `ORBITER` or `FREE_FLY` mode, by directly setting position and orientation. The smoothing system is still active during playback, so very short animations with large orientation changes may appear slightly lagged. Calling `bv.camera_position()` before `show()` pre-sets the camera without triggering the animation system.

---

## Page 11 — Configuration & Settings

All configurable defaults live in `include/CGAL/GLFW/bv_settings.h`. Every macro uses a `#ifndef` guard, so you can override any of them in your `CMakeLists` before including the viewer headers.

### Window

| Macro | Default | Description |
|-------|---------|-------------|
| `CGAL_WINDOW_WIDTH_INIT` | `500` | Initial window width in pixels |
| `CGAL_WINDOW_HEIGHT_INIT` | `450` | Initial window height in pixels |
| `CGAL_WINDOW_SAMPLES` | `1` | MSAA sample count (1 = off) |

### Primitive sizes

| Macro | Default | Description |
|-------|---------|-------------|
| `CGAL_SIZE_VERTICES` | `7.0f` | Point size in pixels |
| `CGAL_SIZE_EDGES` | `1.1f` | Edge line width |
| `CGAL_SIZE_RAYS` | `3.1f` | Ray line width |
| `CGAL_SIZE_LINES` | `3.1f` | Line line width |
| `CGAL_SIZE_NORMALS` | `0.2f` | Normal arrow line width |
| `CGAL_NORMAL_HEIGHT_FACTOR` | `0.02f` | Normal arrow length as fraction of scene size |
| `CGAL_NORMALS_MONO_COLOR` | `{220, 20, 20}` | Default color for normal arrows (red) |

### Lighting

| Macro | Default | Description |
|-------|---------|-------------|
| `CGAL_LIGHT_POSITION` | `{0, 0, 0, 0}` | Light position (w=0: directional) |
| `CGAL_AMBIENT_COLOR` | `{0.6, 0.5, 0.5, 0.5}` | Ambient RGBA |
| `CGAL_DIFFUSE_COLOR` | `{0.9, 0.9, 0.9, 0.9}` | Diffuse RGBA |
| `CGAL_SPECULAR_COLOR` | `{0.0, 0.0, 0.0, 1.0}` | Specular RGBA (off by default) |
| `CGAL_SHININESS` | `1.0f` | Specular exponent |

### Camera

| Macro | Default | Description |
|-------|---------|-------------|
| `CGAL_CAMERA_FOV` | `45.0f` | Field of view in degrees |
| `CGAL_CAMERA_RADIUS` | `5.0f` | Initial orbit radius / scene size estimate |
| `CGAL_CAMERA_TRANSLATION_SPEED` | `1.0f` | Translation speed multiplier |
| `CGAL_CAMERA_ROTATION_SPEED` | `270.0f` | Rotation speed in degrees/second |
| `CGAL_CAMERA_TRANSLATION_SMOOTHNESS` | `0.25f` | Translation smooth factor (0.01–1.0) |
| `CGAL_CAMERA_ROTATION_SMOOTHNESS` | `0.25f` | Rotation smooth factor |
| `CGAL_CAMERA_ZOOM_SMOOTHNESS` | `0.10f` | Zoom smooth factor |

### Clipping plane

| Macro | Default | Description |
|-------|---------|-------------|
| `CGAL_CLIPPING_PLANE_RENDERING_TRANSPARENCY` | `0.5f` | Alpha of the transparent half |
| `CGAL_CLIPPING_PLANE_TRANSLATION_SPEED` | `1.0f` | Translation speed |
| `CGAL_CLIPPING_PLANE_ROTATION_SPEED` | `270.0f` | Rotation speed (deg/s) |
| `CGAL_CLIPPING_PLANE_TRANSLATION_SMOOTHNESS` | `1.0f` | Translation smooth factor |
| `CGAL_CLIPPING_PLANE_ROTATION_SMOOTHNESS` | `0.35f` | Rotation smooth factor |

### Overriding at build time

```cmake
target_compile_definitions(my_program PRIVATE
  CGAL_WINDOW_WIDTH_INIT=1280
  CGAL_WINDOW_HEIGHT_INIT=720
  CGAL_WINDOW_SAMPLES=4
  CGAL_CAMERA_FOV=60.0f
)
```

### Complete runtime setter reference

All state can be overridden after construction but before `show()`:

```cpp
// Window
bv.window_size({1280, 720});
bv.azerty_layout();

// Primitive sizes
bv.size_vertices(10.f);
bv.size_edges(2.f);
bv.size_rays(4.f);
bv.size_lines(4.f);
bv.normal_height_factor(0.05f);

// Visibility toggles
bv.draw_vertices(true);
bv.draw_edges(true);
bv.draw_faces(true);
bv.draw_rays(false);
bv.draw_lines(false);
bv.draw_world_axis(true);       // default: true
bv.draw_xy_grid(false);         // default: false
bv.draw_mesh_triangles(false);

// Shading
bv.flat_shading(true);
bv.use_default_color(false);
bv.reverse_normal(false);

// Lighting
bv.light_position({0, 10, 10, 1}); // point light above
bv.light_ambient({0.4f, 0.4f, 0.4f, 1.f});
bv.light_diffuse({0.8f, 0.8f, 0.8f, 1.f});
bv.light_specular({0.5f, 0.5f, 0.5f, 1.f});
bv.light_shininess(32.f);

// Camera
bv.scene_center({0, 0, 0});
bv.scene_radius(5.f);
bv.camera_position({0, 5, 10});
bv.zoom(3.f);
bv.two_dimensional();   // sets orthographic + forward constraint

// Clipping plane
bv.display_mode(DisplayMode::CLIPPING_PLANE_SOLID_HALF_ONLY);
bv.clipping_plane_orientation({0, 1, 0});
bv.draw_clipping_plane(true);

// Animation
bv.animation_duration(std::chrono::milliseconds(8000));
```

---

## Page 12 — Extending the Viewer

### Supporting a new CGAL data structure

To make `CGAL::draw(my_type)` and `CGAL::add_to_graphics_scene(my_type, scene)` work, specialize these free functions. The pattern from existing specializations:

```cpp
// In your header:
namespace CGAL {

template <class DS, class GSOptions>
void add_to_graphics_scene(const DS& ds,
                           Graphics_scene& gs,
                           const GSOptions& opt)
{
  // iterate over ds, call gs.add_point(...), gs.add_segment(...),
  // gs.add_face(...), etc.
}

// Convenience overload with default options:
template <class DS>
void add_to_graphics_scene(const DS& ds, Graphics_scene& gs)
{
  add_to_graphics_scene(ds, gs, Graphics_scene_options<DS, ...>());
}

template <class DS>
void draw(const DS& ds, const char* title = "CGAL Basic Viewer")
{
  Graphics_scene scene;
  add_to_graphics_scene(ds, scene);
  Basic_viewer bv(scene, title);
  bv.show();
}

} // namespace CGAL
```

The `Graphics_scene` insertion API to use:

```cpp
gs.add_point(p);
gs.add_point(p, color);
gs.add_segment(p1, p2);
gs.add_segment(p1, p2, color);
gs.add_ray(p, direction);
gs.add_line(p1, p2);
gs.add_face_begin();
  gs.add_point_in_face(p, normal);   // once per vertex of the face
gs.add_face_end();
```

Faces must be added in counterclockwise winding order. `Buffer_for_vao` handles triangulation internally.

### Custom per-element coloring

Create a struct that inherits `CGAL::Graphics_scene_options` and override the lambda fields:

```cpp
struct My_options :
  public CGAL::Graphics_scene_options<Mesh,
                                      Mesh::Vertex_index,
                                      Mesh::Edge_index,
                                      Mesh::Face_index>
{
  My_options()
  {
    // Draw edge i only if it is not on the boundary
    this->draw_edge = [](const Mesh& m, Mesh::Edge_index e) -> bool {
      return !CGAL::is_border(e, m);
    };

    // Color each face by its index mod 3
    this->colored_face = [](const Mesh&, Mesh::Face_index) { return true; };
    this->face_color = [](const Mesh&, Mesh::Face_index fi) -> CGAL::IO::Color {
      const CGAL::IO::Color palette[3] = {
        CGAL::IO::Color(255,0,0), CGAL::IO::Color(0,255,0), CGAL::IO::Color(0,0,255)
      };
      return palette[fi % 3];
    };
  }
};
```

Available policy lambdas (all have sensible defaults):

| Lambda | Signature | Default |
|--------|-----------|---------|
| `draw_vertex` | `(DS, VertexDescriptor) → bool` | `true` |
| `draw_edge` | `(DS, EdgeDescriptor) → bool` | `true` |
| `draw_face` | `(DS, FaceDescriptor) → bool` | `true` |
| `colored_vertex` | `(DS, VertexDescriptor) → bool` | `false` |
| `colored_edge` | `(DS, EdgeDescriptor) → bool` | `false` |
| `colored_face` | `(DS, FaceDescriptor) → bool` | `false` |
| `vertex_color` | `(DS, VertexDescriptor) → Color` | random |
| `edge_color` | `(DS, EdgeDescriptor) → Color` | random |
| `face_color` | `(DS, FaceDescriptor) → Color` | random |

### Headless screenshot

```cpp
CGAL::Graphics_scene scene;
CGAL::add_to_graphics_scene(mesh, scene);

CGAL::Basic_viewer bv(scene, "screenshot");
bv.camera_position({0, 5, 10});
bv.draw_vertices(false);
bv.make_screenshot("output.png");
```

`make_screenshot` creates a hidden GLFW window (no display server required in principle, though Wayland screenshot support is still a known issue — see [Known Limitations](#page-13--known-limitations--todo)). The output is a PNG written via `stb_image_write`.

### Multiple windows

Two `Basic_viewer` objects can coexist in the same process. They each own an independent GLFW window, input system, and OpenGL context. Call `show()` on each in sequence, or manage your own loop:

```cpp
// From draw_several_windows.cpp:
CGAL::Basic_viewer bv1(scene1, "Window 1");
CGAL::Basic_viewer bv2(scene2, "Window 2");
bv1.show();   // blocks until window 1 is closed
bv2.show();   // then shows window 2
```

For truly simultaneous windows you would need to manage the GLFW event loop yourself, polling both windows in a single loop — the current `show()` implementation is single-window blocking.

### Modifying shaders

All GLSL source is in `include/CGAL/Basic_shaders.h` as string literals. To add a post-process effect, change the lighting model, or add a custom rendering mode, edit the appropriate `const char[]` and recompile. Shaders are compiled at `Basic_viewer` construction time — there is no hot-reload.

---

## Page 13 — Known Limitations & TODO

The following is drawn directly from `include/CGAL/TODO.md`, supplemented with architectural observations.

### GLFW-specific

- **Auto keyboard layout detection:** the viewer currently requires the user to explicitly call `bv.azerty_layout()` for AZERTY keyboards. Automatic detection from the OS has not been implemented.
- **Set pivot point via click (`Shift+Right`):** changing the orbit center by double-clicking on a mesh point is listed as pending.
- **Wayland screenshot:** `make_screenshot` may not work correctly under Wayland compositors. X11 is the reliable path currently.

### Shared with Qt viewer

- **Mesh selection:** interactive selection of vertices, edges, or faces is not yet supported in either backend.
- **Line/ray width shaders:** only `Segment` primitives use the `LINE_WIDTH` geometry shader (which expands lines to quads for variable width). `Ray` and `Line` primitives still use plain `gl_LineWidth`, which is deprecated and ignored on many drivers. The fix is to apply the same geometry-shader expansion to all three.
- **Compatibility shaders:** several GLFW-specific shaders (sphere, cylinder, line-width, arrow, normal, triangle) do not yet have OpenGL 3.2-compatibility variants — the viewer falls back to simpler rendering if OpenGL 4.3 is not available.

### Graphics scene

- **No per-element size hints:** `Graphics_scene` does not store per-point or per-segment size hints. Vertex and edge sizes are viewer-global. Adding a per-element size component to the scene buffers would allow mixing large vertices with small ones in the same scene.

### Architectural notes

- **Static scene:** `load_scene()` uploads geometry to the GPU once with `GL_STATIC_DRAW`. Dynamic scenes (geometry that changes per frame) require a full re-initialization. There is no incremental update path.
- **Single-process multi-window:** the `show()` method is blocking. Running two windows simultaneously requires managing the GLFW poll loop manually.
- **No pick/ray-cast:** there is no built-in mechanism to find which face/vertex the mouse cursor is pointing at (required for selection features).
- **Shader recompilation:** shaders are compiled from source at every `Basic_viewer` construction. For applications that create many viewers (e.g., batch screenshot workflows), this adds fixed overhead per viewer.
