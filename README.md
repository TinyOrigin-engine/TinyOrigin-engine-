# Tiny Origin

A 2D game engine and visual editor written in C# (.NET 9.0). Unity-inspired architecture with an Entity-Component pattern, integrated Box2D physics, ImGui-based editor, and runtime C# scripting with hot-reload.

## Tech Stack

| Layer | Technology |
|---|---|
| Language | C# / .NET 9.0 |
| Rendering | Raylib-cs (v8.0.0) |
| Editor UI | ImGui.NET (v1.91.6.1) + rlImGui-cs |
| Physics | Box2D.NET (v3.1.654) |
| Scripting | Roslyn (Microsoft.CodeAnalysis.CSharp.Scripting) |

---

## Architecture Overview

```
editor.cs / main.txt          Entry points
core/                          Engine core systems
  T_components.cs              Component base class + built-in components
  T_gameBlock.cs               GameBlock (entity) + global registry
  T_engine.cs                  Play/Edit mode state management
  T_scene.cs                   Scene container
  T_scenemanager.cs            Multi-scene management
  T_SceneSerializer.cs         JSON scene serialization
  T_scriptmanager.cs           Script discovery, creation, attachment
  T_scriptgenerator.cs         Script code template generation
  T_compiler.cs                Async dotnet build + hot-reload
  T_inputs.cs                  Input wrapper
  T_collisions.cs              Collision utility
phisics/                       Physics system (Box2D wrappers)
  physicsworld.cs              Box2D world manager
  rigidbody.cs                 Dynamic physics body
  staticbody.cs                Static physics body
Scripts/                       User-generated C# scripts
styles/                        ImGui themes
```

---

## Component System

The engine uses a **Unity-inspired Entity-Component pattern**. Components are data containers attached to GameBlocks.

### Base Class

```csharp
public abstract class Component
{
    public string name = "";
    public GameBlock self_gameBlock = null!;  // back-reference to owning entity
}
```

### Built-in Components

| Component | Purpose | Key Fields |
|---|---|---|
| `Transform3D` | Position/rotation/scale (3D vectors, 2D usage) | `Vector3 position, rotation, scale` |
| `Transform2D` | Position/rotation/scale (native 2D) | `Vector2 position, scale; float rotation` |
| `Sprite2D` | Texture rendering | `Texture2D texture, Color color, string texturePath, Vector2 pivotPoint` |
| `Script` | Base for all user scripts | Virtual `Start()` and `Update(float deltaTime)` |

### How Components Are Stored

Each `GameBlock` stores components in a dictionary keyed by type:

```csharp
public Dictionary<Type, Component> components = new();
```

This enforces **one component per concrete type**. API:

```csharp
block.AddComponent<T>(component)   // Add or replace by typeof(T)
block.GetComponent<T>()            // Lookup by type
block.HasComponent<T>()            // Type check
block.RemoveComponent<T>()         // Remove by type
```

Every new `GameBlock` automatically receives a `Transform3D` in its constructor.

### Lifecycle

- `GameBlock.Start()` calls `Script.Start()` on all attached Script components.
- `GameBlock.Update(deltaTime)` calls `Script.Update(deltaTime)` on all attached Script components.
- Non-Script components (`Transform3D`, `Sprite2D`) are pure data holders with no lifecycle hooks.

---

## Block System (GameBlocks)

A **GameBlock** is the fundamental entity in the scene -- what Unity calls a "GameObject." It is a generic container configured entirely by its attached components and scripts.

### GameBlock Definition

```csharp
public class GameBlock
{
    public string id;                           // string identifier (e.g., "player")
    public bool IsActive { get; set; } = true;  // can be deactivated
    public Dictionary<Type, Component> components;
}
```

### Global Registry

```csharp
public static class GameBlockManager
{
    public static readonly List<GameBlock> blocks = new();
}
```

All GameBlocks register themselves on construction. This flat list is used for global lookups.

### Scene Management

A `Scene` holds a `List<GameBlock>` and provides:

```csharp
scene.CreateGameBlock(string id, params Component[])  // Factory method
scene.AddGameBlock(GameBlock) / RemoveGameBlock(GameBlock)
scene.FindBlock(string id)                             // Find by ID
scene.UpdateAll(deltaTime)                             // Tick all blocks
scene.StartAll()                                       // Start all blocks
```

`SceneManager` (static) manages multiple scenes by name, tracks the active scene, and maintains a load history.

### Example Blocks (from scene files)

```
physics_world  -> Transform3D + PhysicsWorld script (manages Box2D world)
player         -> Transform3D + Sprite2D + RigidBody script
ground         -> Transform3D + Sprite2D + StaticBody script
bob            -> Transform3D + Sprite2D + RigidBody script
```

There is no specialized "block type" system. The type of a block is entirely determined by its attached components.

---

## Scripting System

The scripting system is the most extensible part of the engine. Users write C# classes that inherit from `Script` and get compiled, loaded, and attached to GameBlocks at runtime.

### Writing a Script

```csharp
using T_Component;
using T_GameBlock;
using T_INPUT;

namespace CustomScripts
{
    public class PlayerMovement : Script
    {
        public float speed = 100f;  // public fields appear in inspector

        public override void Start() { }

        public override void Update(float deltaTime)
        {
            Transform3D t = self_gameBlock.GetComponent<Transform3D>();
            if (Input.IsKeyDown(KeyboardKey.D))
                t.position.X += speed * deltaTime;
            if (Input.IsKeyDown(KeyboardKey.A))
                t.position.X -= speed * deltaTime;
        }
    }
}
```

### Script Lifecycle

| Method | When Called |
|---|---|
| `Start()` | Once when the block enters Play mode |
| `Update(float deltaTime)` | Every frame during Play mode |

### Creating Scripts (Editor)

1. Click **"Create New Script"** in the editor.
2. Enter a name for the script.
3. The engine generates a `.cs` file in the `Scripts/` directory from a template.
4. `dotnet build` is triggered automatically in the background.
5. On success, the script type is discovered and can be attached to any GameBlock.

### ScriptManager

The `ScriptManager` handles the full script lifecycle:

- **Discovery**: Scans all loaded assemblies for classes inheriting from `Script`.
- **Directory Scanning**: Checks the `Scripts/` directory for `.cs` files and tracks pending scripts.
- **Instantiation**: `CreateScript(string name)` creates instances via `Activator.CreateInstance()`.
- **Attachment**: `AttachScript(string name, GameBlock target)` attaches a script to a block.
- **Placeholder Scripts**: While a script is pending compilation, a `PlaceholderScript` is created that logs warnings.
- **Dynamic Wrappers**: `DynamicScriptWrapper` polls every 2 seconds trying to load the actual script (for hot-reload).

### Hot-Reload Workflow

```
User creates script -> .cs file written to Scripts/
                    -> dotnet build runs asynchronously
                    -> ScriptManager.ReloadScripts() re-scans assemblies
                    -> New script type available immediately
                    -> Attach to any GameBlock
```

### Custom Attributes

```csharp
[ScriptCategory("Rigidbody")]  // Categorizes scripts in the editor UI
public class MyCustomBody : Script { ... }
```

### Built-in Scripts (Physics)

The physics system is implemented as Script subclasses, demonstrating the extensibility of the system:

- **`PhysicsWorld`** -- Creates and steps the Box2D world. Configurable gravity and pixels-per-meter ratio.
- **`RigidBody`** -- Dynamic physics body with box/circle/capsule shapes.
- **`StaticBody`** -- Static (immovable) physics body with box/circle shapes.

---

## Physics System

Box2D.NET integration provides 2D rigid body physics.

### PhysicsWorld

- Creates a `b2World` with configurable gravity (default: `0, -9.81`).
- Steps the world each frame with `b2World_Step` (4 velocity iterations, clamped to max 1/30s delta).
- `SteppedThisFrame` flag prevents multiple steps per frame.
- `Generation` counter tracks world recreations for body initialization guards.

### Body Types

| Body | Box2D Type | Shapes | Key Properties |
|---|---|---|---|
| `RigidBody` | `b2_dynamicBody` | box, circle, capsule | density 1.0, friction 0.3, gravity scale 1 |
| `StaticBody` | `b2_staticBody` | box, circle | friction 0.3, immovable |

### Coordinate Conversion

Screen pixels are converted to/from Box2D meters using a `pixelsPerMeter` ratio (default: 50). The Y-axis is inverted (screen Y-down -> Box2D Y-up). Half-width/half-height offsets are applied for proper centering.

### Shape Visualization

Both `RigidBody` and `StaticBody` render wireframe collision outlines (red lines/circles) when `show_shape = true`.

---

## Editor

The editor (`editor.cs`, ~1470 lines) is a full ImGui-based visual editor.

### Panels

| Panel | Description |
|---|---|
| **Toolbar** | Play/Stop, FPS, fullscreen, Save/Load/Rebuild, build status |
| **Hierarchy** | Tree list of all blocks, component icons, create new blocks |
| **Inspector** | Selected block's components with editable fields (auto-generated via reflection) |
| **Viewport** | Scene rendering, click/drag to move blocks in Edit mode |

### Inspector (Reflection-Based)

The inspector uses .NET reflection to render appropriate ImGui controls for any public field:

| Field Type | ImGui Control |
|---|---|
| `float` | `DragFloat` |
| `int` | `DragInt` |
| `string` | `InputText` |
| `bool` | `Checkbox` |
| `Color` | `ColorEdit4` |
| `enum` | `Combo` |

This means any public field on any component or script is automatically editable in the inspector with zero extra code.

### Block Drag & Drop (Edit Mode)

Click and hold on a block in the viewport to drag it. Collision detection uses `PointRotatedRec()` (inverse rotation + AABB check) to determine which block is under the cursor.

---

## Scene Serialization

Scenes are saved as human-readable JSON using `System.Text.Json` and reflection.

### Save Format

```json
{
  "Name": "level",
  "Blocks": [
    {
      "Id": "player",
      "IsActive": true,
      "Components": [
        {
          "Type": "T_Component.Transform3D",
          "Properties": {
            "position": { "X": 0, "Y": 0, "Z": 0 },
            "rotation": { "X": 0, "Y": 0, "Z": 0 },
            "scale": { "X": 1, "Y": 1, "Z": 1 }
          }
        },
        {
          "Type": "RigidBody",
          "Properties": {
            "shape": "box",
            "width": 2.2,
            "height": 0.4
          }
        }
      ]
    }
  ]
}
```

### Serialization Details

- Uses **reflection** to read all public instance fields from each component.
- Special handling for `Vector2`, `Vector3`, `Vector4`, `Color` (serialized as anonymous objects).
- `Sprite2D` reloads its texture from `texturePath` on load.
- Script types are resolved by scanning loaded assemblies, with fallback to `ScriptManager.CreateScript()`.

---

## Play/Edit Mode

The engine maintains two states managed by `EngineState`:

### Enter Play Mode

1. Auto-saves the current scene to `scene.json`.
2. Creates a backup at `_temp_backup.json`.
3. Stores the in-memory scene reference.
4. Calls `Start()` on all blocks.
5. Physics simulation begins.

### Exit Play Mode

1. Clears all blocks from `GameBlockManager`.
2. Reloads the scene from `_temp_backup.json`.
3. Replaces the active scene.
4. Deletes the temp backup file.

This allows the editor to run game logic and physics in Play mode while preserving the original scene state for Edit mode.

---

## Input System

A thin static wrapper around Raylib:

```csharp
Input.IsKeyPressed(key)   // Single frame press
Input.IsKeyDown(key)      // Held down
Input.IsKeyReleased(key)  // Single frame release
Input.IsKeyUp(key)        // Not held
Input.IsFullscreen        // Fullscreen state
Input.ToggleFullscreen()  // Toggle fullscreen (F11)
```

---

## Window & Rendering

- Window: **1280x720**, 60 FPS target.
- F11 toggles fullscreen.
- Rendering uses `Raylib.DrawTexturePro()` for 2D sprite rendering.
- Despite `Transform3D` using `Vector3`, rendering is effectively **2D** (X/Y position, Z rotation).
- Textures are loaded via `Raylib.LoadImage` -> `Raylib.LoadTextureFromImage` -> `Raylib.UnloadImage`.

---

## Quick Start

```bash
# Build and run the editor
dotnet run --project tinyOrigin.csproj

# Or run the minimal runtime (no editor)
# Rename main.txt to Program.cs, then:
dotnet run
```

### Editor Controls

| Key | Action |
|---|---|
| F11 | Toggle fullscreen |
| Play button | Enter Play mode |
| Stop button | Exit Play mode (restores scene) |
| Click + Drag | Move block (Edit mode only) |
| Rebuild | Recompile scripts |
