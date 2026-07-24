# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A personal reference and study guide for 3D graphics and engine architecture. There is no source code, build system, or test suite — all content is Markdown documentation covering abstract rendering and engine concepts.

## Content map

| File | Contents |
|---|---|
| `Geometry.md` | Vertex, mesh, material, model; index buffer; triangulation; winding order; texture vs material; tile vs mesh face |
| `Math.md` | SIMD; GPU primitive types; coordinate spaces (local → world → camera → clip → screen); camera / view / projection matrices; normal matrix; uniforms; quaternions |
| `RenderPipeline.md` | GPU pipeline stages; shader compilation; PSOs; vertex/index/uniform buffers; draw calls; triple buffering; render graph; render loop; frame budget |
| `EngineArchitecture.md` | Core engine subsystems; ECS pattern; scene graph; asset pipeline; streaming / LOD / quadtree; camera fly-over with Catmull-Rom spline; concurrency model |
| `AdvancedRendering.md` | TAA; PBR / Cook-Torrance BRDF; SSAO; mesh shaders; neural upscaling; SDF text; Gaussian splatting; ray tracing vs rasterisation |
| `Culling.md` | Frustum, occlusion, backface, distance, and layer culling |
| `Resources.md` | Curated video links (engine architecture, GPU pipeline, research-grade rendering) |
| `General.md` | Index redirecting to the files above |
