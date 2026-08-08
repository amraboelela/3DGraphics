# Geometry

## Vertex, Mesh, Material, Model

A **vertex** is one point in 3D space with attributes (position, normal, UV coordinates, etc.).

A **mesh** is a collection of vertices plus an index buffer describing how they connect into triangles.

A **material** describes how the surface looks when light hits it — which shaders to run and which textures to use.

A **model** is a higher-level container bundling one or more meshes together with their materials, skeleton, animations, and transform hierarchy — what you import from a `.usdz`, `.fbx`, or `.obj` file.

```
Model (e.g. person.usdz)
 ├── Mesh: body skin    + Material: skin shader + albedo texture
 ├── Mesh: eyes         + Material: glossy eye shader
 ├── Mesh: hair         + Material: alpha-blended hair shader
 └── Mesh: clothing     + Material: fabric shader (swappable)
```

**Why multiple meshes for one character?**
- Different body parts need different shaders (hair uses alpha transparency; eyes need specular highlights).
- Clothing meshes can be swapped independently.
- LOD — you can replace just the hair mesh with a lower-poly version at distance.
- Rigging — a skeleton deforms each mesh; separating them makes weight painting easier.

**Typical vertex attributes:**

| Attribute | Description |
|---|---|
| position | XYZ location in local space |
| normal | Surface direction for lighting |
| UV (texcoord) | Texture coordinate (0–1 range) |
| tangent | For normal mapping; perpendicular to normal |
| color | Per-vertex colour tint |

---

## Triangulation — Why Triangles?

3D modeling tools work with quads and n-gons because they're easier to sculpt. But the GPU only draws triangles, so the engine **triangulates** before uploading: every quad is split into 2 triangles, every n-gon into however many needed. This happens once at asset import time.

```
Quad:          Triangulated:
A --- B        A --- B
|     |   →    |  ↗  |
D --- C        D --- C
Indices: [A,B,D,  B,C,D]
```

**Why triangles?** Three points always define exactly one flat plane. A quad can be warped (non-planar), making shading ambiguous. Triangle rasterisation is also simple and parallelisable — GPU hardware is optimised for it.

---

## Index Buffer

An index buffer stores integers that reference positions in the vertex array, rather than duplicating vertex data per face.

A cube has 8 unique vertices but 36 index entries (6 faces × 2 triangles × 3 indices). Indices reuse vertices to avoid duplicating data, cutting memory and bandwidth.

---

## Winding Order

Counter-clockwise winding = front face. The GPU back-face culls clockwise triangles (facing away from the camera) — halving fragment work. Wrong winding = mesh invisible from outside.

---

## Texture vs Material

A **texture** is just an image stored on the GPU — a grid of pixels (texels). It has no meaning on its own.

A **material** is the recipe that says *how to use* one or more textures together to produce a final surface appearance. The material owns the textures and tells the shader which image to sample for colour, which for bump detail, which for shininess.

```
Texture = raw image data (pixels)
Material = shader + textures + scalar parameters
           → combined by the fragment shader to produce final pixel colour
```

Analogy: a texture is a tin of paint; a material is the painter's instructions — which tin, where, matte or gloss.

**Common material texture maps:**

| Texture | Effect |
|---|---|
| Albedo / diffuse | Surface colour in white light |
| Normal map | Fake surface bumps without extra geometry |
| Roughness map | Per-pixel shiny vs matte |
| Metallic map | Per-pixel metal vs non-metal |
| Emissive map | Self-illuminated areas (windows, signs) |
| Ambient occlusion | Pre-baked shadow in crevices |

---

## Mesh Face vs Tile

| | Mesh face | Tile |
|---|---|---|
| What it is | One triangle (3 vertices) | A geographic region's worth of scene data |
| Scale | Microscopic — part of a model | Kilometres of real-world area |
| Content | 3 vertices + indices | Many meshes, textures, labels, metadata |
| Loaded | At model import time | Streamed on demand as camera moves |
| LOD | Fewer faces at distance | Lower zoom level (coarser) tile at distance |

---

## YouTube Resources

- [OpenGL 3D Game Tutorial: Rendering with Index Buffers](https://www.youtube.com/watch?v=z2yFlvkBbmk)
- [UV Maps Explained](https://www.youtube.com/watch?v=Yx2JNbv8Kpg)
