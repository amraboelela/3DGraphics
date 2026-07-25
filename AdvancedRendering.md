# Advanced Rendering

## TAA — Temporal Anti-Aliasing

Accumulates multiple slightly-jittered frames using a **history buffer** to reconstruct a high-quality image. Each frame the camera's projection matrix is offset by a sub-pixel amount (the jitter); the history buffer blends the new frame with past frames weighted by a confidence factor.

- Near-free vs 4–8× MSAA in pixel cost.
- Requires a **velocity buffer** to reproject history onto moving geometry correctly.
- **Neighbourhood clamping** rejects history samples that differ too much from the current frame's local neighbourhood, preventing ghosting.
- Main artifact: ghosting on fast-moving objects when reprojection fails.

---

## PBR — Physically Based Rendering

Models light interaction using the same physical laws regardless of lighting conditions, so materials look consistent under any illumination.

**Cook-Torrance BRDF** is the standard real-time PBR model:

```
f(l, v) = (D · F · G) / (4 · (n·l) · (n·v))
```

Where:
- **D** — Normal Distribution Function: how many microfacets are aligned with the halfway vector (controls roughness).
- **F** — Fresnel term: ratio of light reflected vs refracted at grazing angles.
- **G** — Geometry term: accounts for microfacet self-shadowing.

**Artist-facing parameters:**
- `metallic` (0 = plastic/dielectric, 1 = metal) — drives whether base colour acts as albedo or specular tint.
- `roughness` (0 = mirror, 1 = diffuse) — drives the width of specular highlights.

**Image-Based Lighting (IBL):** pre-computes diffuse irradiance and specular radiance from an environment map into cubemaps, then samples them at runtime for ambient lighting that matches the environment.

---

## SSAO — Screen-Space Ambient Occlusion

Darkens crevices and contact areas cheaply by sampling the depth buffer in a hemisphere around each pixel and checking how many samples are occluded by nearby geometry.

- Runs entirely in screen space — no scene geometry query needed.
- Cheap: one depth-buffer pass + blur.
- **Limitation:** only sees what's on screen — geometry offscreen can't occlude on-screen pixels.
- Typical blur radius: 4–16 samples; results blurred with a bilateral filter to avoid darkening across depth discontinuities (edges).

---

## Mesh Shaders

Replace the traditional vertex + tessellation pipeline with a two-stage programmable pipeline:

- **Task shader (amplification)** — decides how many mesh shader threadgroups to launch; used for culling at the meshlet level before any vertex work.
- **Mesh shader** — generates vertices and primitives directly on the GPU; no vertex buffer required.

**Benefits:** eliminates the CPU-side tessellation and draw-call overhead for high-density geometry. The GPU can cull individual meshlets (clusters of ~64–128 triangles) in the task stage, which is far more granular than per-object frustum culling on the CPU.

Available on recent GPU generations (Turing/RDNA2/Apple Silicon M-series).

---

## Neural Upscaling (DLSS / FSR style)

Renders the scene at a reduced resolution (e.g., 50–75% of native) and reconstructs the full-resolution image using a trained neural network or spatial filter.

- **DLSS** (NVIDIA) — transformer-based network using motion vectors and depth; runs on dedicated tensor cores.
- **FSR** (AMD) — spatial upscaler (no ML); uses a spatial sharpening filter; hardware-agnostic.
- **MetalFX Upscaling** (Apple) — temporal or spatial upscaler integrated into the Metal API.

**Why it matters on mobile:** rendering at 75% resolution and upscaling costs roughly 56% of native pixel work, saving significant battery without perceptible quality loss.

---

## SDF Text — Signed Distance Field Text

Glyphs are pre-rasterised into a texture where each texel stores the **signed distance to the nearest glyph outline** (negative = inside, positive = outside, zero = on the edge).

At runtime, the fragment shader reads the SDF texture and thresholds at 0.5 to reconstruct a crisp edge at any size — one texture sample is sufficient. Compared to traditional bitmap fonts:

- **Resolution-independent:** the same texture looks sharp at 12pt and 200pt.
- **Cheap effects:** outline, drop shadow, and glow are free — adjust the threshold and add a colour pass.
- **Single atlas:** one 512×512 SDF atlas covers all sizes; bitmap fonts need a separate atlas per size.

**Tradeoff:** sharp corners require a larger SDF texture or a multi-channel SDF (MSDF) to preserve corner sharpness.

---

## Gaussian Splatting

Represents a scene as a collection of 3D Gaussian ellipsoids rather than triangle meshes. Each Gaussian stores position, covariance (shape/orientation), opacity, and spherical harmonic coefficients (view-dependent colour).

**Rendering:** each Gaussian is projected as a 2D Gaussian onto the screen, sorted by depth, and alpha-composited front-to-back. No rasterisation or ray tracing — entirely a splatting pass.

**Strengths:**
- Very fast novel-view synthesis from a set of input photographs.
- Smooth, continuous appearance without visible triangle edges.

**Weaknesses:**
- Large memory footprint (millions of Gaussians per scene).
- Hard to edit or animate (no mesh topology).
- Does not integrate with traditional rasterisation pipelines.

Used primarily for static scene reconstruction and cinematic novel-view synthesis; not yet a replacement for mesh-based real-time rendering.

---

## Ray Tracing vs Rasterisation

| | Rasterisation | Ray Tracing |
|---|---|---|
| Approach | Project geometry to screen, shade pixels | Cast rays from the eye, trace bounces |
| Shadows | Shadow maps (approximation) | Exact — ray hit test per light |
| Reflections | Reflection maps, SSR (approximate) | Exact — recursive rays |
| GI | Baked lightmaps, SSAO, probes (approx) | Path tracing — physically exact |
| Cost | Cheap — hardware optimised for decades | Expensive — feasible only with dedicated RT cores |
| Use | Real-time games, apps | Offline rendering, hybrid (shadow/reflection rays only) |

Hybrid rendering: use rasterisation for primary visibility (it's fast), then use ray tracing only for secondary effects (shadows, reflections, ambient occlusion) where rasterisation approximations break down.

---

## YouTube Resources

- [TAA — Temporal Anti-Aliasing Deep Dive](https://www.youtube.com/watch?v=WG8w9Yg5B3g)
- [PBR Explained in 3 Minutes](https://www.youtube.com/watch?v=_ZbkOZNgwNk)
- [Mesh Shaders — How Do They Work?](https://www.youtube.com/watch?v=8NNnMwcb-hM)
- [Upscaling Explained — DLSS vs FSR](https://www.youtube.com/watch?v=Chpb3yNypxY)
- [Glyphs, Shapes, Fonts, Signed Distance Fields](https://www.youtube.com/watch?v=1b5hIMqz_wM)
- [Advances in Neural Rendering — SIGGRAPH 2021](https://www.youtube.com/watch?v=otly9jcZ0Jg)
