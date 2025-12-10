# 1.2 Vertex Processing

[← Back to Chapter 1](../chapters/01-Graphics-Pipeline-Fundamentals.md) | [Main Index](../README.md)

Vertex processing transforms 3D vertices from model space to screen space, applies skinning/morphing, and passes data to the rasterizer. Efficient vertex processing is critical for geometry-heavy scenes.

---

## Overview

Every vertex in a mesh passes through the vertex shader, which transforms positions (model → world → view → clip space), computes lighting vectors, and prepares interpolated data (UVs, normals, colors) for fragment shaders. On PC and console GPUs, vertex shaders execute on the same ALUs as fragment shaders but typically consume <10% of GPU time—fragment shaders dominate. However, in high-polygon scenes (>1M triangles), vertex processing can bottleneck, especially with complex skinning or per-vertex lighting.

Vertex data format impacts both memory bandwidth and GPU cache efficiency. A character mesh with 10,000 vertices using 64 bytes per vertex (position, normal, tangent, UV, bone weights/indices) occupies 640KB. Multiply by 50 characters and you're transferring 32MB per frame—significant on bandwidth-limited platforms like Switch. Compression reduces memory, bandwidth, and cache misses, improving performance by 20-40% in vertex-bound scenarios.

Modern GPUs fetch vertices from cache before processing. Interleaved vertex formats (all attributes together: PPPNNNUUUU) have better cache locality than split buffers (positions in buffer A, normals in buffer B). However, if shaders access only some attributes (position-only depth prepass), split formats avoid fetching unused data. Platform profilers reveal cache hit rates—target >90% for optimal performance.

## Key Concepts

- **Vertex Shader**: Programmable stage that processes each vertex independently. Takes input attributes (position, normal, UV), outputs clip-space position and varying data for rasterization. Executes once per vertex per frame.
- **Vertex Format**: Memory layout of vertex attributes. Interleaved (all attributes sequential per vertex) vs split (each attribute in separate buffer). Interleaving improves cache hits by 30-50% for full-attribute shaders.
- **Vertex Compression**: Storing attributes in reduced precision (16-bit normals instead of 32-bit floats). Saves memory and bandwidth, requires unpacking in shader. Typically adds 5-10 ALU instructions per vertex with negligible cost.
- **GPU Skinning**: Skeletal animation computed on GPU by blending bone transforms. Each vertex samples 2-4 bone matrices based on weights/indices. Costs 20-40 ALU instructions per vertex but avoids CPU→GPU mesh uploads every frame.
- **Input Assembler**: Fixed-function stage that fetches vertex data from buffers, assembles primitives (triangles), and feeds vertex shaders. Bottlenecks occur with small vertex caches or incoherent vertex ordering.

## Best Practices

**Vertex Shader Optimization:**
- Keep vertex shaders under 100 ALU instructions. Move calculations to fragment shaders if they vary per-pixel (Phong lighting) rather than per-vertex (Gouraud).
- Use built-in functions (normalize, dot, mul) instead of manual implementations—they compile to single instructions on most GPUs.
- Avoid branching in vertex shaders. Branches are cheaper than in fragment shaders (fewer invocations) but still serialize execution within SIMD warps.
- Use half-precision (`min16float` in HLSL) for non-position data like normals and UVs. Doubles ALU throughput on consoles with no visible quality loss.

**Vertex Format Design:**
- Use 16-bit formats where possible: `R16G16B16A16_SNorm` for normals, `R16G16_UNorm` for UVs (0-1 range). Reduces vertex size by 40-50% vs 32-bit floats.
- Pack multiple attributes into unused channels. Store metallic in normal.w, smoothness in UV.z (if UVs use only 2 components).
- Align vertex stride to 16 bytes (one cache line). 48-byte vertices (position 12B + normal 12B + UV 8B + tangent 16B) align perfectly. 52-byte vertices waste cache space.
- For static meshes, bake vertex colors for ambient occlusion (R channel), detail masks (G), or wind animation (B). Costs 4 bytes but eliminates texture samples.

**Compression Strategies:**
- **Positions**: Quantize to 16-bit integers within object bounds. Store bounds in constant buffer, unpack in shader: `pos = asuint(input.pos) / 65535.0 * boundsSize + boundsMin`.
- **Normals**: Use octahedral encoding (2x16-bit). Provides better quality than simple sphere mapping. Decoding costs ~15 ALU instructions, saves 4 bytes per vertex.
- **UVs**: Store as 16-bit UNorm (0-1 range) or SNorm (-1 to 1). For tiling textures (UVs > 1), use SNorm with scale/bias constant.
- **Tangents**: Reconstruct from normal and UV gradients in pixel shader (via ddx/ddy), eliminating 16 bytes per vertex. Increases fragment shader cost by ~10 instructions.

**Skinning Optimization:**
- Limit bone influences to 2-4 per vertex. More influences add 10-20 ALU instructions each—diminishing visual returns.
- Use bone index compression: 8-bit indices (255 bones max) instead of 32-bit. Saves 12 bytes per vertex for 4-bone skinning.
- Implement vertex shader instancing for crowds—upload bone matrices as instance data, saving CPU→GPU bandwidth.

**Platform-Specific:**
- **PC DirectX 12**: Use structured buffers for bone matrices instead of constant buffers (4KB limit). Enables 1,000+ bones for complex rigs.
- **Xbox Series X/S**: Leverage mesh shaders for LOD or geometry amplification. Bypasses input assembler and vertex shader, providing more control.
- **PlayStation 5**: Use fast VRAM for frequently-accessed vertex buffers (character skinning). Slower system RAM acceptable for static environment meshes.
- **Switch**: Minimize vertex size aggressively—32 bytes ideal, 48 acceptable. Switch's 25.6 GB/s bandwidth is 10-20x lower than PS5/Xbox.

## Common Pitfalls

**Overusing 32-bit Floats**: High-precision floats for every attribute waste bandwidth. Normals have 4.2 billion possible values in Float32—only need 65,536 directions visible to humans. Using Float32 normals on 100,000-vertex meshes transfers 1.2MB instead of 600KB (Float16)—double the bandwidth for zero visual improvement.

**Skinning Too Many Vertices CPU-Side**: CPU skinning seems simpler but requires uploading entire mesh to GPU every frame. A 10,000-vertex character at 60fps uploads 38MB/sec (10K verts × 64 bytes/vert × 60fps). GPU skinning uploads only bone matrices (256 bones × 64 bytes = 16KB), saving 38MB/sec per character. With 10 characters, you've saved 380MB/sec—critical on console bandwidth.

**Ignoring Vertex Cache**: GPUs cache processed vertices (typically 16-32 entries). If triangles reference vertices in random order, cache misses force redundant vertex shader executions. Run mesh optimization tools (DirectXMesh Optimize, Forsyth algorithm) to reorder indices for 50-80% fewer vertex shader invocations. Unity does this automatically on import.

## Tools & Workflow

**Unity Mesh Importer**: "Read/Write Enabled" doubles mesh memory (keeps CPU copy). Disable for static meshes. "Optimize Mesh" reorders vertices for cache efficiency. "Compression" applies lossy quantization—test visual quality carefully.

**RenderDoc**: Mesh Viewer shows vertex buffer layout and contents. "Input Assembler" tab reveals vertex format and stride. Right-click mesh in event list to inspect vertex data before/after vertex shader.

**PIX Mesh Visualizer**: Shows mesh with color-coded vertex cache hits (green) vs misses (red). Use "Vertex Reuse" metric to measure optimization effectiveness—target >1.5 (each vertex used by 1.5+ triangles on average).

**NSight Graphics**: "Vertex Cache Analyzer" reveals cache hit rate and optimal index reordering. "Vertex Attribute Fetcher" shows memory bandwidth consumed per vertex—use to identify compression opportunities.

**DirectXMesh Library**: Use `OptimizeFaces()` and `OptimizeVertices()` in asset pipeline to reorder indices for cache efficiency. Reduces vertex shader invocations by 40-60% on typical meshes. Unity calls this internally, but custom mesh generators should manually optimize.

## Related Topics

- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Vertex cache optimization and LOD generation
- [7.4 Animation Optimization](07-04-Animation-Optimization.md) - Skinning and blend shape optimization
- [12.2 Shader Types](12-02-Shader-Types.md) - Vertex shader programming patterns
- [13.2 Vertex Compression](08-02-Mesh-Compression.md) - Advanced compression techniques

---

## Related Topics

- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Mesh optimization strategies
- [7.4 Animation Optimization](07-04-Animation-Optimization.md) - Animation performance
- [12.2 Shader Types](12-02-Shader-Types.md) - Shader programming

---

[← Previous: 1.1 Rendering Pipeline Overview](01-01-Rendering-Pipeline-Overview.md) | [Next: 1.3 Primitive Assembly and Rasterization →](01-03-Primitive-Assembly-Rasterization.md)
