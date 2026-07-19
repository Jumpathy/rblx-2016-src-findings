

## The Three Executables

### RobloxStudio
> The game editor you used to build Roblox games.

This is the biggest output. It's a Qt desktop app (the ribbon toolbar, property panel, explorer
tree, script editor — all Qt widgets) wrapping the full game engine. When you hit "Play Solo" it
literally boots the engine inside the editor window. It can work mostly offline for building and
testing `.rbxl` place files, though it'll complain about network calls it can't complete.

### WindowsClient
> The game player — what ran when you clicked "Play" on the website.

Bare-bones compared to Studio. No editor UI — just the engine + a render window. It was hardwired
to talk to `www.roblox.com` to authenticate, fetch the place, and stream assets. On its own today
it can't load anything useful because the 2016 backend is gone. But the rendering, input, and
physics code is all here.

### RCCService
> The dedicated game server, meant to run in Roblox's cloud.

"RCC" stands for Roblox Cloud Compute. It exposes a SOAP web service (via gSOAP) so that
Roblox's backend infrastructure could spin it up, tell it what game to run, and manage it remotely.
It was never designed to run standalone — it expects the full Roblox web stack behind it. You can
technically launch it locally but it won't do much without something orchestrating it. It also has
a `ThumbnailGenerator` in it, which was used server-side to render preview images of places.

---

## The Core Libraries (the engine itself)

These all compile as static `.lib` files and get linked into whichever exe needs them.

### Base
> The absolute foundation. Math, types, utilities.

Things like `CFrame` (the position+rotation type used everywhere), `Vector3`, `RbxAssert`, string
formatting, and CPU/hardware detection. Everything else in the engine depends on Base. Nothing
fancy — just the bedrock that everything else is built on.

### App
> The game engine. This is the big one.

Everything you interacted with as a game developer lives in here. It's organized into subsystems:

- **`v8datamodel/`** — every Roblox object/service (Part, Workspace, Players, Camera, GUI
  objects, AnimationController, etc.). The "DataModel" is the hierarchy of all objects in a
  running game.
- **`v8world/`** — the physics world: bodies, contacts, collision, buoyancy.
- **`v8kernel/`** — low-level physics kernel: constraint solving, joints, forces.
- **`v8tree/`** — the instance tree itself: parenting, replication flags, property change signals.
- **`script/`** — the Lua scripting layer: VM integration, Lua-to-C++ bridge, sandboxing,
  debugger hooks.
- **`reflection/`** — the system that exposes C++ properties/methods/events to Lua. Every time
  you do `part.Size = ...` in a script, this is what handles it.
- **`humanoid/`** — the humanoid state machine: Running, Jumping, Freefall, Swimming, Ragdoll,
  etc. Jump height, walk speed, and fall physics are all in here.
- **`gui/`** — the in-game 2D GUI system (ScreenGui, Frame, TextLabel, etc.), separate from
  Studio's editor UI.
- **`security/`** — Lua sandbox levels (game script vs. plugin vs. core script) and API access
  control.

### App.BulletPhysics
> Bullet physics library, Roblox-modified — but only for collision detection.

Roblox didn't use Bullet to solve physics (forces, joints, constraints). They used Bullet only
for collision detection — figuring out which shapes are touching and where. Their own custom PGS
(projected Gauss-Seidel) solver in `App/v8kernel/` handles everything else. The solver config
and all its tunable defaults live in `App/include/solver/SolverConfig.h`.

### Network
> Multiplayer: replication, physics sync, and the RakNet UDP layer.

Built on top of RakNet (a UDP networking library). Handles:
- **Replicator** — syncs the DataModel between server and clients. When a part moves on the
  server, `ServerReplicator` packages that change up; `ClientReplicator` applies it on the other
  end.
- **PhysicsSender / PhysicsReceiver** — stream physics state (positions, velocities) across the
  network, with several strategies (round-robin, error-compensated, top-N-errors).
- **GameConfigurer** — the entry point that sets up all the `roblox.com` API URLs when the
  engine starts. This is the first file to touch if you want to redirect to a local server.

### CSG
> Constructive Solid Geometry — the Union and Negate tools.

When you union two parts together or negate one out of another in Studio, this library does the
geometry math. It wraps `sgCore.dll` (a closed-source CSG backend that was reverse-engineered for
this repo) and exposes the result back into the DataModel as a `CSGMesh` or `SpecialMesh`.

### Log
> The logging system.

`FastLog` / `FastLogStream` — a lightweight, low-overhead logging layer used throughout the
engine for debug output and diagnostics. Most of the `FASTLOG(...)` calls you see throughout the
source go through here.

---

## Rendering

The rendering stack is split into four layers under `Rendering/`:

### GfxBase
> The abstract graphics interface — hardware-agnostic types and handles.

Defines the `IRenderer` interface, texture handles, mesh handles, adornments (the debug drawing
primitives like lines and boxes you see in Studio). Nothing here knows whether it's talking to
D3D9, D3D11, or OpenGL.

### GfxCore
> The actual GPU driver layer.

Has three backends: `D3D9/`, `D3D11/`, and `GL/`. Handles render targets, framebuffers,
geometry buffers, and shader binding. This is where draw calls actually hit the GPU.

### GfxRender
> Scene rendering: the material system, lighting, G-buffer.

The scene graph, material rendering (brick, plastic, neon, metal, etc.), lighting passes, and
shadow maps. Each material in `shaders/source/*.hlsl` is tied into here.

### AppDraw
> High-level draw calls and debug rendering.

Sits on top of GfxRender. Handles part drawing, adornments, selection highlights, and billboards.
The bridge between "game object wants to be drawn" and "GfxCore submits draw call."

### RbxG3D + g3d
> Roblox's fork of the G3D graphics math library.

G3D provides things like `Vector3`, `Matrix4`, `CoordinateFrame`, `AABox`, `Sphere` on the
graphics side. Roblox forked it and modified it for their needs. Used heavily throughout the
rendering pipeline.

### ShaderCompiler
> Compiles HLSL shaders at build time.

Runs during the build (via `buildshaders.bat`) to compile the `.hlsl` files in `shaders/source/`
into binary shader bytecode that GfxCore loads at runtime. If you want to add or tweak a material
shader, edit the `.hlsl` and rebuild shaders.

---

## Studio-Specific Code

### RobloxStudio (the project folder)
> All the editor UI on top of the engine.

This is everything you see in the Studio window that isn't the game viewport:
- The ribbon toolbar (`RobloxRibbonMainWindow`)
- The Explorer tree (`RobloxTreeWidget`)
- The Properties panel (`RobloxPropertyWidget`)
- The script editor (`ScriptTextEditor`, syntax highlighting, IntelliSense)
- The Output window, Diagnostics view, Find dialog
- Cloud Edit / Team Create widgets
- Mobile device emulator
- Plugin hosting (`RobloxPluginHost`)
- The embedded browser pane (Qt WebKit, used for login and asset pages)

It's all Qt widgets. The `moc_*.cpp` files are Qt's "meta-object compiler" output — auto-generated
glue code, not hand-written.

### BuiltInPlugins / StudioPlugins
> Roblox's own Studio plugins, written in Lua.

These are `.rbxmx` files (Roblox model XML). Things like the terrain tools, the Transform
dragger, and the physics analyzer. They run inside Studio's plugin sandbox, the same as any
third-party plugin would.

---

## Supporting Code

### ClientBase
> Shared startup/config code for the client executables.

`MachineConfiguration` (hardware detection), `RenderSettingsItem` (stores and loads graphics
quality settings), and `ReflectionMetadata` (the XML metadata file that documents all Lua-
accessible classes and properties — this is what populates Studio's Object Browser).

### ClientShared
> Thin shared utilities between WindowsClient and Studio.

Data serialization helpers, an InfluxDB metrics helper, SDL gamepad support, a minimal JSON
parser, and string conversion utilities. Small glue code that both client executables needed but
didn't belong in the core engine.

### GameChat
> The in-game voice/text chat system.

Separate from the DataModel's `ChatService`. Has its own network layer (`ChatNetwork`),
audio thread (`ChatAudioThread`), client (`ChatClient`), and user serialization. Used for the
real-time chat alongside gameplay.

---

## Third-Party / Contribs

These live in `Contribs/` and are external libraries the engine depends on:

| Library | What it does |
|---|---|
| `boost_1_56_0` | C++ utilities: threads, signals, filesystem, smart pointers, lexical cast |
| `SDL2` | Window creation, input handling, gamepad support |
| `Qt` (DLLs) | The entire Studio editor UI framework |
| `openssl` | HTTPS/TLS for all web API calls |
| `curl` / `curllib` | HTTP client (all requests to roblox.com go through libcurl) |
| `fmod` | Audio engine (sound playback, 3D audio) |
| `zlib` | Compression (place files, network packets) |
| `boostlibs` | Pre-built Boost static libraries |
| `RakNet` | UDP networking library (lives in `Network/raknet/`) |
| `sgCore` | CSG geometry backend (DLL, was closed-source, reverse-engineered here) |
| `LibOVR / OpenVR / VrApi` | Oculus/SteamVR/Gear VR support in the rendering stack |

---

## How It All Connects

```
RobloxStudio.exe
  |
  +-- Studio UI (Qt widgets, ribbon, panels)  [RobloxStudio/]
  |
  +-- Game Engine
  |     +-- Base          (math, types)
  |     +-- App           (DataModel, Lua VM, physics world, humanoid, GUI)
  |     +-- App.BulletPhysics  (collision detection shapes)
  |     +-- Network       (RakNet, replication, physics sync)
  |     +-- CSG           (union/negate geometry)
  |     +-- Log           (logging)
  |
  +-- Rendering
        +-- GfxBase / GfxCore / GfxRender / AppDraw
        +-- RbxG3D / g3d  (graphics math)
        +-- ShaderCompiler (HLSL -> bytecode)

WindowsClient.exe
  +-- Same engine libs as above, minus Studio UI
  +-- ClientBase / ClientShared (config, machine detection)

RCCService.exe
  +-- Same engine libs, headless (no rendering)
  +-- gSOAP SOAP server interface
  +-- ThumbnailGenerator (offscreen rendering for place previews)
```

The short version: **Base** is the foundation, **App** is the engine, **Network** makes it
multiplayer, **Rendering** draws it, **CSG** handles unions, and then **RobloxStudio**,
**WindowsClient**, and **RCCService** are three different shells around that same engine core for
three different jobs.
