# Render Pipeline

## The GPU Pipeline Stages

Every triangle passes through a fixed sequence of stages on its way to a pixel:

```
Vertex shader → Rasterisation → Fragment shader → Blending → Framebuffer
```

- **Vertex shader** — runs once per vertex; transforms position from local space to clip space using the MVP matrix; passes attributes (UV, normal) to the next stage.
- **Rasterisation** — GPU determines which pixels each triangle covers; interpolates vertex attributes across the surface.
- **Fragment shader** — runs once per covered pixel; samples textures, computes lighting, outputs a colour value.
- **Blending** — combines the fragment colour with what is already in the framebuffer (required for transparency).

---

## Shader Compilation

Shaders are written in a shading language (GLSL, HLSL, MSL, WGSL, etc.) and compiled into GPU machine code at application load time — never mid-frame. Compilation takes 100ms or more per pipeline state, so all shaders are compiled up-front.

Three kinds of compilation happen at different times in an engine:

1. **Engine source → binary** (build time) — the CPU code is compiled by the host compiler (Clang, MSVC, etc.).
2. **Shaders → GPU bytecode** (load time) — shading language source compiles to GPU-native instructions and is bundled into a Pipeline State Object (PSO).
3. **Asset / style compilation** (pipeline time) — mesh data, textures, and style configs are pre-processed into optimised binary formats before streaming to the device.

---

## Pipeline State Object (PSO)

A **PSO** bundles the vertex shader, fragment shader, blend state, depth/stencil state, and vertex layout into one pre-validated object. When a draw call executes, the GPU has already verified all state combinations are legal.

- Switching PSOs mid-frame is expensive — the GPU must flush and reconfigure.
- **Best practice:** sort draw calls by PSO to minimise switches; group all opaque geometry using the same shader together.

---

## Buffers

A **buffer** is a region of GPU-accessible memory holding typed data. Three key buffer types:

| Buffer | Contents | Frequency |
|---|---|---|
| Vertex buffer | Per-vertex attributes (position, normal, UV) | Once per mesh |
| Index buffer | Integer indices into the vertex buffer | Once per mesh |
| Uniform buffer | Per-frame constants (MVP, camera pos, time) | Once per frame |

On hardware with a unified memory architecture (CPU and GPU share the same DRAM), buffers marked as shared-mode are zero-copy — no upload step needed. On discrete GPUs, data must be staged through a CPU-visible buffer and then blitted to GPU-only memory.

---

## Draw Calls

A **draw call** instructs the GPU to render a set of primitives using the currently bound state (PSO, buffers, textures). With state pre-validated in PSOs, draw calls themselves are near-zero CPU cost. The bottleneck is the number of state changes between them.

**Draw call budget:** modern engines target < 1000–2000 draw calls per frame at 60 Hz. Techniques to reduce them:
- **Instancing** — draw many copies of the same mesh in one call with per-instance data.
- **Batching** — merge multiple meshes that share a material into one vertex buffer.
- **Indirect draw** — GPU decides what to draw; eliminates CPU readback and rebinding.

---

## Triple Buffering

Keep three sets of per-frame uniform buffers in flight simultaneously:

```
Frame N-1: GPU rendering         (buffer index 0)
Frame N:   CPU encoding          (buffer index 1)
Frame N+1: CPU preparing         (buffer index 2)
```

A semaphore (count = 3) prevents the CPU from writing into a buffer the GPU is still reading. This achieves full CPU/GPU overlap without stalls, at the cost of 2–3 frames of input latency.

---

## Render Graph

A **render graph** is a declarative description of all render passes in a frame, with explicit resource read/write annotations per pass. The engine's graph compiler:

- **Derives optimal barrier placement** — inserts memory/execution barriers only where true dependencies exist.
- **Aliases transient resources** — render targets that don't overlap in time (e.g., SSAO scratch buffer, TAA history) share the same GPU memory allocation, reducing peak VRAM 30–50%.
- **Enables pass reordering** — passes with no dependency between them can run in parallel on async compute queues.

Popularised by Frostbite (GDC 2017). Now standard in UE5 (RDG), Unity (RenderGraph), and custom engines.

---

## Render Loop

The heartbeat of the engine — runs every frame at display refresh rate (60 / 90 / 120 Hz):

```
1. Update  — physics, animation, transform propagation, gameplay logic
2. Cull    — remove entities outside the camera frustum (see Culling.md)
3. Encode  — record all draw calls into a command buffer (CPU work)
4. Commit  — submit the command buffer to the GPU
5. Present — display the completed framebuffer when the GPU finishes
```

**Frame budget:**
- 60 Hz → 16.6 ms total
- 120 Hz → 8.3 ms total

Any work that cannot complete within the budget must be offloaded to a background thread and double-buffered — results are consumed in the next frame.

**Fixed vs variable timestep:** physics and gameplay run at a fixed timestep (e.g., 120 Hz) for determinism; rendering runs at whatever rate the display allows. The update loop may tick multiple times between renders if the frame rate drops.

---

## YouTube Resources

- [Introduction to the Render Graph in Unity 6](https://www.youtube.com/watch?v=U8PygjYAF7A)
- [SIGGRAPH 2021 Rendering Engine Architecture Course (playlist)](https://www.youtube.com/playlist?list=PLAOytOz0HZbLaWhVrGEge5_6dNCAzGFYH)
- [Metal Performance Best Practices — Apple 2023](https://www.youtube.com/watch?v=LXTUFmbZwec)
- [WWDC22: Metal Mesh Shaders — Apple](https://www.youtube.com/watch?v=uVfj79_bZsU)
