# Graphics APIs

A comparison of the major real-time GPU programming APIs.

---

## At a Glance

| | Metal | Direct3D 12 | Direct3D 11 | OpenGL | OpenGL ES | Xbox GDK |
|---|---|---|---|---|---|---|
| Vendor | Apple | Microsoft | Microsoft | Khronos | Khronos | Microsoft |
| Platforms | Apple (iOS, macOS, tvOS, visionOS) | Windows 10+ | Windows Vista+ | Windows, Linux, macOS | Android, iOS (legacy) | Xbox Series / One |
| Generation | Modern (low-level) | Modern (low-level) | Mid-level | Legacy | Legacy | Modern (low-level) |
| Shader language | MSL (Metal Shading Language) | HLSL | HLSL | GLSL | GLSL ES | HLSL |
| CPU overhead | Very low | Very low | Medium | High | Medium | Very low |
| Driver complexity | Thin | Thin | Fat | Fat | Fat | Thin |
| First release | 2014 | 2015 | 2009 | 1992 | 2003 | 2020 |

---

## Metal

Apple's low-level GPU API, available on all Apple platforms. Designed from the ground up for Apple Silicon and the unified memory architecture where CPU and GPU share the same DRAM.

**Key characteristics:**
- **Unified memory:** buffers marked `.storageModeShared` are zero-copy — no upload/staging needed on Apple Silicon.
- **Explicit control:** the app manages command buffers, synchronisation, and resource residency explicitly.
- **Tile-based deferred rendering (TBDR):** Apple GPUs process tiles of the framebuffer entirely in on-chip memory before writing to DRAM. Metal exposes tile shaders and imageblocks so apps can exploit this directly (programmable blending, deferred lighting with no DRAM round-trip).
- **MetalFX:** built-in upscaling (spatial and temporal) with no external library.
- **Mesh shaders:** supported on Apple Silicon M-series via the standard Metal pipeline.

**Shader language (MSL):** C++17-based. Runs on CPU at compile time; GPU at runtime. Same struct definitions can be shared between host and shader code without a separate bridge.

**Limits:** Apple platforms only — no path to Windows, Android, or consoles.

---

## Direct3D 12 (DirectX 12)

Microsoft's modern low-level GPU API for Windows 10+ and Xbox Series X/S. Successor to Direct3D 11, designed to eliminate driver-side guessing and give developers explicit control.

**Key characteristics:**
- **Explicit resource barriers:** the app declares every read/write transition; the driver inserts no hidden synchronisation.
- **Descriptor heaps:** all resource views (SRVs, UAVs, samplers) live in GPU-visible descriptor heaps. No per-draw-call driver patching.
- **Work graphs (DX12 Ultimate):** GPU-driven pipelines where nodes schedule work for downstream nodes — eliminates CPU readback for certain compute-heavy workloads.
- **Mesh shaders and ray tracing:** first-class in DirectX 12 Ultimate (supported on Ampere/RDNA2+).
- **Agility SDK:** Microsoft ships driver-independent runtime updates; apps get new features without waiting for OS updates.

**Shader language (HLSL):** compiled offline to DXIL (DXBC for older targets). HLSL 6.x adds wave intrinsics, SM6.6 bindless, SM6.7 samplers.

**Steep learning curve:** explicit resource barriers, pipeline state objects, and command list recording require careful upfront design. Common to use a render graph abstraction on top.

---

## Direct3D 11 (DirectX 11)

The mid-generation API that struck a balance between control and driver automation. Still widely used for Windows games targeting the broadest hardware range.

**Key characteristics:**
- **Driver manages synchronisation:** no explicit resource barriers; the driver tracks resource usage and inserts transitions automatically — simpler to use, higher CPU overhead.
- **Immediate + deferred context:** one immediate context for GPU submission; deferred contexts for multi-threaded command recording (limited and rarely used in practice).
- **Compute shaders:** introduced with DX11, enabling GPGPU workflows before dedicated compute APIs existed.
- **Feature levels:** apps declare a minimum feature level (9.1 through 12.2) and the driver validates capability at device creation.

**Where it still fits:** middleware libraries, legacy codebases, tools that need Windows Vista/7 support, and engines where the DX12 overhead of explicit barriers is not justified by the performance return.

---

## OpenGL

The original cross-platform 3D API, standardised by Khronos. Runs on Windows, Linux, and macOS (deprecated on macOS since 10.14 in favour of Metal).

**Key characteristics:**
- **Global state machine:** a single implicit context holds all state (bound textures, buffers, shaders). Any call can read or modify global state — the root cause of most OpenGL driver overhead.
- **Driver does everything:** the driver tracks resource hazards, compiles shaders lazily, and manages synchronisation internally. Convenient but unpredictable performance.
- **Shader compilation at draw time:** many drivers defer shader compilation until the first draw call using that program — causing hitches mid-game. ARB_program_binary and pipeline state caching are workarounds.
- **Portability:** the widest hardware reach of any GPU API. Still the default on Linux desktops and many embedded systems.
- **Compatibility profile vs core profile:** the compatibility profile keeps all deprecated features (immediate mode, fixed-function pipeline) available. Core profile removes them, producing a cleaner but less portable subset.

**Decline:** Vulkan replaced OpenGL as Khronos's modern API in 2016. New projects on desktop Linux should prefer Vulkan; macOS dropped OpenGL support from GPUs entirely in future macOS versions.

---

## OpenGL ES

The embedded-systems subset of OpenGL, designed for mobile GPUs. The dominant GPU API on Android and the legacy API on iOS before Metal.

**Key characteristics:**
- **Subset of desktop OpenGL:** floating-point textures, geometry shaders, and many extensions are optional or absent. Precision qualifiers (`lowp`, `mediump`, `highp`) are mandatory because mobile shaders target different hardware tiers.
- **Tile-based GPUs:** most mobile GPUs (ARM Mali, Qualcomm Adreno, Imagination PowerVR) are tile-based. OpenGL ES has no way to express tile-local operations — a key reason Apple moved to Metal (which exposes imageblocks) and Khronos introduced Vulkan.
- **EGL:** the platform-agnostic surface and context management layer that sits alongside OpenGL ES.
- **Versions:**
  - ES 2.0 — programmable shaders, the universal baseline for Android/iOS.
  - ES 3.0 — instancing, multiple render targets, transform feedback.
  - ES 3.1 — compute shaders, indirect draw.
  - ES 3.2 — tessellation, geometry shaders, ASTC texture compression.

**Current state:** still the baseline for Android targeting older devices. New Android development increasingly uses Vulkan. iOS dropped OpenGL ES with iOS 12 (deprecated; removed from the simulator in Xcode 14).

---

## Xbox GDK (Game Development Kit)

The modern API for Xbox Series X/S and Xbox One, built on top of DirectX 12 with Microsoft-proprietary extensions for console-specific hardware.

**Key characteristics:**
- **DirectX 12 as the foundation:** Xbox GDK exposes the same command list, PSO, and descriptor heap model as PC DirectX 12 — cross-platform code is achievable with thin platform layers.
- **Xbox-specific extensions:**
  - **Texture-space shading:** shade at a different resolution than screen space; reuse shading across frames for static surfaces.
  - **ESRAM (Xbox One only):** fast on-die memory for render targets; requires explicit placement.
  - **Velocity Architecture (Xbox Series):** DirectStorage for GPU decompression of assets directly from NVMe; SFS (Sampler Feedback Streaming) for fine-grained texture residency.
  - **Variable Rate Shading (VRS):** shade tiles at reduced rates based on content; saves GPU time in low-detail areas.
- **Certification requirements:** games must meet Microsoft TRC (Title Requirement Checklist) including memory limits, suspend/resume handling, and performance targets.
- **Shared codebase with PC:** GDK replaces the older XDK and ERA model. A single solution can target both Xbox and Windows 10/11 with conditional compilation for console-specific paths.

---

## Abstraction Level Comparison

```
High abstraction (easy, less control)
  │
  ├── OpenGL / OpenGL ES   — driver manages everything; implicit state machine
  │
  ├── Direct3D 11          — driver handles synchronisation; some explicit state
  │
  ├── Direct3D 12          — explicit barriers, heaps, residency; thin driver
  ├── Metal                — explicit command buffers, synchronisation, TBDR exposure
  └── Xbox GDK             — DX12 + console extensions; direct hardware access
  │
Low abstraction (harder, maximum control)
```

---

## Choosing an API

| Goal | Recommended API |
|---|---|
| Apple platforms (iOS, macOS, visionOS) | Metal |
| Windows PC, broad hardware | Direct3D 11 (safe) or Direct3D 12 (performance) |
| Xbox console | Xbox GDK (Direct3D 12) |
| Cross-platform desktop (Windows + Linux) | Vulkan (or OpenGL for legacy reach) |
| Android modern | Vulkan |
| Android legacy / broad device support | OpenGL ES 3.x |
| Cross-platform mobile + desktop (single codebase) | Vulkan or a cross-API abstraction layer (bgfx, SDL_GPU, WebGPU) |

---

## Shader Language Comparison

| API | Language | Compilation |
|---|---|---|
| Metal | MSL (C++17-based) | Offline to AIR (Apple Intermediate Representation), then GPU-native at PSO creation |
| Direct3D 12 / 11 | HLSL | Offline to DXIL (DX12) or DXBC (DX11) via DXC; driver translates to GPU ISA |
| Xbox GDK | HLSL | Same as DX12; Xbox shader compiler adds console-specific optimisations |
| OpenGL | GLSL | Source string compiled by the GPU driver at runtime — unpredictable latency |
| OpenGL ES | GLSL ES | Same driver-side compilation; precision qualifiers required |

HLSL, GLSL, and MSL are syntactically similar (C-like structs, swizzles, built-in vector math) but differ in binding models, precision handling, and available intrinsics.

---

## Memory Model Comparison

| API | CPU↔GPU memory | Notes |
|---|---|---|
| Metal (Apple Silicon) | Unified (shared DRAM) | Zero-copy for `.storageModeShared`; no staging buffer needed |
| Metal (Intel Mac) | Discrete | Requires staging buffer + blit for GPU-private resources |
| Direct3D 12 | Discrete (PC) or unified (Xbox Series) | `D3D12_HEAP_TYPE_UPLOAD` for CPU-write; `DEFAULT` for GPU-only |
| OpenGL | Driver-managed | `glBufferData` with usage hints; driver decides placement |
| OpenGL ES | Driver-managed | `GL_STREAM_DRAW` / `GL_STATIC_DRAW`; no explicit control |
| Xbox GDK | Unified (GDDR6 shared) | Title manages memory pools; no OS virtual memory overhead |

---

## YouTube Resources

- [Metal Performance Best Practices — Apple 2023](https://www.youtube.com/watch?v=LXTUFmbZwec)
- [WWDC22: Metal Mesh Shaders — Apple](https://www.youtube.com/watch?v=uVfj79_bZsU)
- [Programming Metal in iOS — Full Playlist](https://www.youtube.com/playlist?list=PL23Revp-82LJG3vcDPm8w7b5HTKjBOY0W)
- [Swift Game Engine with Metal Intro](https://www.youtube.com/watch?v=PcA-VAybgIQ)
- [Beginner Tutorial: Your First DirectX 12 Application in C++](https://www.youtube.com/watch?v=3ubqb13Cix4)
