# Engine Architecture

## Core Subsystems

| Subsystem | Responsibility |
|---|---|
| Scene graph / ECS | Entities, components, transforms, parent-child hierarchy |
| Render loop | Fixed-timestep update → variable-rate render |
| Asset pipeline | Load, cache, stream, hot-reload meshes/textures/shaders |
| Resource manager | Ref-counted GPU resource lifetime |
| Command buffer layer | Batch draw calls, sort by PSO/material |
| Camera system | View/projection matrix, frustum culling, LOD |
| Concurrency layer | Worker threads for physics/audio/assets; sync before GPU submit |

---

## Entity-Component-System (ECS)

**ECS** separates data (components) from logic (systems), avoiding deep OOP inheritance hierarchies.

- **Entity** — a lightweight identifier (e.g., an integer ID). No data or behaviour of its own.
- **Component** — a plain data struct attached to an entity (e.g., `Transform`, `Mesh`, `Health`). No logic.
- **System** — a function that iterates over all entities possessing a specific set of components and transforms their data.

**Why ECS over OOP for engines?**

| Problem with OOP | ECS solution |
|---|---|
| Deep inheritance chains → rigid hierarchy | Composition — any combination of components |
| Virtual dispatch → cache misses | Flat arrays per component type → L1 cache friendly |
| Hard to parallelise update logic | Systems with no shared components run concurrently |

**Memory layout:** each component type lives in its own contiguous array. Iterating 10 000 `Transform` components reads sequential memory — no pointer-chasing across the heap. Accessing two component types for the same entity is the main layout challenge; solved by archetype-based ECS (Unreal Mass, Bevy, Flecs) or sparse-set ECS.

**System execution order:**
```
Input → Physics → Animation → Transform propagation → Cull → Render
```
Systems are ordered by dependency, not by entity. A physics system can run entirely before the animation system touches any entity.

---

## Scene Graph

A **scene graph** organises entities in a parent-child transform hierarchy. A child's world transform = parent's world transform × child's local transform. Changing a parent moves all descendants automatically.

ECS and scene graphs are complementary — ECS manages data layout and update logic; the scene graph encodes spatial relationships. Many modern engines layer a scene graph atop an ECS.

---

## Asset Pipeline

Assets flow through three stages before reaching the GPU at runtime:

1. **Source** — raw artist files (`.fbx`, `.psd`, `.wav`). Never shipped.
2. **Cooked / compiled** — engine-specific binary formats optimised for the target platform (compressed textures, triangulated meshes, pre-computed LODs).
3. **Runtime** — GPU resources loaded from cooked data. Can be streamed on demand.

**Hot-reload:** during development the engine watches source files and re-imports changed assets without restarting — critical for iteration speed on shaders and textures.

---

## Streaming and LOD

For large open worlds or geographic renderers, not all geometry can fit in GPU memory at once.

**Level of Detail (LOD):** each mesh has multiple versions at decreasing polygon counts. The engine selects the appropriate LOD based on the object's screen-space size. LOD switches are transparent to the rest of the engine.

**Tile streaming:** a quadtree (or similar spatial structure) divides the world into tiles. As the camera moves, the engine:
1. Queries the quadtree for tiles visible to the current frustum at the current LOD level.
2. Fetches missing tiles from disk or network asynchronously.
3. Evicts tiles that have been out of view long enough from the GPU resource cache.

Lower altitude (closer to the ground) → higher-detail tiles (finer zoom level). Higher altitude → coarser tiles to stay within GPU memory budget.

---

## Camera Fly-Over / Path Animation

Animating a camera smoothly through a sequence of waypoints uses a **Catmull-Rom spline**. Unlike linear interpolation, Catmull-Rom guarantees continuity of velocity (C¹) across waypoints without requiring the artist to set tangent handles.

Each frame, the camera position advances a fixed distance along the spline parameter `t`, the view and projection matrices are recomputed, and the tile streamer reacts to the new camera position and altitude.

**Catmull-Rom formula** (for reference):

```
P(t) = 0.5 × (
    (2·P1) +
    (−P0 + P2)·t +
    (2·P0 − 5·P1 + 4·P2 − P3)·t² +
    (−P0 + 3·P1 − 3·P2 + P3)·t³
)
```

Where P0–P3 are four consecutive waypoints and t ∈ [0, 1] interpolates between P1 and P2.

---

## Concurrency in a Render Engine

**Key constraint:** the GPU command buffer must be submitted from a single thread at a deterministic point in the frame. All concurrent work must finish and hand off results before that point.

**Typical threading model:**

| Thread | Work |
|---|---|
| Main / render thread | Render loop, GPU encoding, present |
| Worker pool | Physics, animation, asset decompression |
| Streaming thread | Disk/network I/O, tile loading |
| Audio thread | Real-time audio mix |

**Synchronisation points:**
- Workers complete their frame's results → signal a barrier → render thread reads them.
- Never block the render thread waiting on I/O; double-buffer async results instead.
- Tile data loaded mid-frame appears in the *next* frame, not the current one.
