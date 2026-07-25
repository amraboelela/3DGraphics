# Math

## SIMD — Single Instruction, Multiple Data

**SIMD** (Single Instruction, Multiple Data) — one CPU instruction operates on multiple values simultaneously instead of one at a time. A renderer transforms millions of vertices per frame; SIMD makes that practical.

Without SIMD: adding two 4-component vectors requires 4 separate add instructions. With SIMD: 1 instruction. This compounds across millions of vertices per frame.

**GPU primitive types:**

| Type | Components | Bytes | Common use |
|---|---|---|---|
| Scalar (float) | 1 | 4 | Time, alpha, roughness |
| 2D vector | 2 | 8 | UV coordinates, screen position |
| 3D vector | 3 | 12 | Position, normal, RGB colour |
| 4D vector | 4 | 16 | Clip-space position, RGBA |
| 3×3 matrix | 9 | 36 | Normal matrix |
| 4×4 matrix | 16 | 64 | Model / View / Projection / MVP |
| Quaternion | 4 | 16 | Rotation (no gimbal lock) |
| 16-bit float (half) | 1 | 2 | Compressed colour, UV |

Structs crossing the CPU/GPU boundary must use these aligned types — layout mismatches cause silent GPU data corruption.

---

## Coordinate Spaces

Every vertex travels through a chain of coordinate spaces on its way to the screen:

```
Local space →[Model]→ World space →[View]→ Camera space →[Projection]→ Clip space → Screen
```

- **Local space** — positions relative to the object's own origin. An artist works here.
- **World space** — positions in the shared scene. The Model matrix places the object.
- **Camera space** — positions relative to the camera, which sits at the origin facing −Z.
- **Clip space** — positions after perspective divide; visible range is [−1, 1] in all axes (NDC).
- **Screen space** — final 2D pixel coordinates after the viewport transform.

The combined **MVP matrix** (Model × View × Projection) transforms a vertex from local space to clip space in one operation.

---

## Camera, View Matrix, Projection Matrix

**Camera** — the conceptual eye: position in the world, look-at direction, up vector, field of view, near/far clip planes. Not a matrix itself — just the data used to build the matrices.

**View matrix** — transforms world space into camera space, making the camera the origin facing −Z. Built from camera position, look-at target, and up vector. All scene geometry is expressed relative to the camera after this transform.

**Projection matrix** — transforms camera space into clip space, applying perspective so far objects appear smaller. Defined by:
- Field of view (e.g. 60°)
- Aspect ratio (e.g. 16:9)
- Near clip plane (e.g. 0.1 m)
- Far clip plane (e.g. 1000 m)

**Perspective vs orthographic projection:**

| | Perspective | Orthographic |
|---|---|---|
| Appearance | Far objects appear smaller | No foreshortening |
| Use | 3D games, cinematics | CAD, 2D UIs, isometric |
| Clip volume | Frustum (truncated pyramid) | Box |

---

## Normal Matrix

Normal vectors cannot use the Model matrix directly — the translation component of a 4×4 matrix would corrupt them. The **normal matrix** is the inverse transpose of the 3×3 upper-left portion of the Model matrix. It correctly transforms normals even when the model is non-uniformly scaled or sheared.

---

## Uniforms

**Uniforms** are per-frame constants that every shader reads without change across all vertices/fragments in a draw call. Typical contents:

- MVP matrix (or separate Model, View, Projection)
- Normal matrix
- Camera position in world space
- Time (for animated effects)
- Light direction / colour

Uniforms are uploaded once per frame into a small GPU-accessible buffer and bound to both the vertex and fragment stages. They are the primary mechanism for communicating per-frame state from the CPU to the GPU.

---

## Quaternions

A quaternion represents a 3D rotation as a 4-component value (x, y, z, w). Compared to Euler angles:

- **No gimbal lock** — axes never align and lose a degree of freedom.
- **Smooth interpolation** — SLERP (spherical linear interpolation) produces natural in-between rotations.
- **Compact** — 4 floats vs a 3×3 rotation matrix (9 floats).

Euler angles are fine for authoring (artist-facing); quaternions are used at runtime.

---

## YouTube Resources

- [What Are SIMD Instructions?](https://www.youtube.com/watch?v=vIRjSdTCIEU)
- [4× Code Performance with SIMD](https://www.youtube.com/watch?v=Imj4ROIiMw0)
