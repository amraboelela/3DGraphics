# Culling

**Culling** is the process of skipping the rendering of objects the camera cannot see. The goal is to reduce CPU and GPU work by eliminating geometry that will not contribute to the final image.

## 1. Frustum Culling

The camera can only see objects inside its viewing pyramid (the **view frustum**). Anything outside is discarded before any vertex or fragment work.

```
        Camera
          |
         / \
        /   \
       /     \
      /       \
     /         \
    +-----------+
   Objects inside → Rendered
   Objects outside → Culled
```

**Example:** facing north, a building directly behind the camera is completely outside the frustum — the engine skips it entirely.

Frustum culling is performed automatically by virtually every real-time engine.

---

## 2. Occlusion Culling

An object may be inside the frustum but completely hidden behind another object. Occlusion culling detects this and skips the draw call.

```
Camera

   ███████ Wall
      Car
```

Without occlusion culling, the car is submitted to the GPU and its pixels are discarded in the depth test — wasted work. With occlusion culling, the car is never submitted at all.

Particularly effective in:
- Dense urban environments
- Indoor scenes (rooms, corridors, buildings)

Occlusion culling is typically baked offline (for static scenes) or computed with GPU occlusion queries at runtime.

---

## 3. Backface Culling

Every triangle has a front side and a back side, determined by **winding order**. If the camera sees the back face of a triangle it can be discarded before rasterisation.

```
Camera →

▲  Front face  → Render
▼  Back face   → Cull
```

Since most 3D models are closed surfaces, the inside faces are never visible. Backface culling halves the number of triangles that reach the fragment shader. Enabled by default in almost every engine and graphics API.

---

## 4. Distance Culling

Objects beyond a configurable distance threshold are skipped entirely. Typically applied to small or dense objects where the visual contribution at distance is negligible:

- Grass and ground cover beyond a few hundred metres
- Small props (rocks, debris)
- Decorations

Often combined with **Level of Detail (LOD)**: instead of a hard cut-off, the mesh switches to progressively simpler versions with distance before disappearing entirely. Fog can mask the transition.

---

## 5. Layer Culling

A camera can be configured to render only specific object categories (layers). Multiple cameras can each render a different subset and composite the results:

```
Camera A        Camera B
- World   ✓     - UI     ✓
- UI      ✗     - World  ✗
- Enemies ✓     - Enemies ✗
```

Useful for rendering UI on top of the world without running depth tests against scene geometry, or for split-screen with different visible sets per player.

---

## Combined Effect — Example

A city scene with 20 000 buildings, 100 000 trees, and 500 cars. The camera is at street level facing one direction.

| Culling stage | Objects removed |
|---|---|
| Frustum culling | All geometry behind or beside the camera |
| Occlusion culling | Cars and buildings hidden behind other buildings |
| Distance culling | Trees and small props beyond the draw distance |
| Backface culling | Interior-facing triangles of every visible mesh |

Result: roughly 2 000 of the original 120 500 objects reach the GPU — a 98% reduction in draw calls.

---

## Culling Pipeline

```
Scene Objects
      │
      ▼
Frustum Culling
      │
      ▼
Occlusion Culling
      │
      ▼
Distance / Layer Culling
      │
      ▼
Backface Culling
      │
      ▼
GPU Rasterisation
      │
      ▼
Screen
```

Culling never deletes objects — it simply skips rendering work for geometry that cannot contribute to the final image. It is one of the most effective levers for real-time rendering performance.
