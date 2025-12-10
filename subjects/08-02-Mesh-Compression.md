# 8.2 Mesh Compression

[← Back to Chapter 8](../chapters/08-Compression-Techniques.md) | [Main Index](../README.md)

Mesh compression reduces memory and storage by quantizing vertex attributes to lower precision, achieving 40-60% size reduction with imperceptible quality loss.

---

## Overview

Uncompressed meshes store vertex attributes at full 32-bit float precision: position (12 bytes), normal (12 bytes), UV (8 bytes), tangent (16 bytes) = 48 bytes per vertex. A 10,000-vertex character = 480KB uncompressed. Mesh compression quantizes attributes to 16-bit precision: position (6-8 bytes), normal (4 bytes), UV (4 bytes), tangent (8 bytes) = 22-24 bytes per vertex. Same character = 220-240KB (50-54% reduction). Across 100 meshes, that's 24MB vs 48MB—50% savings.

Mesh compression works by reducing precision where full 32-bit float is unnecessary. Positions quantize relative to mesh bounds (16-bit sufficient for 99% of meshes). Normals quantize to 16-bit oct-encoding or 32-bit with implicit Z reconstruction. UVs quantize to 16-bit (0-1 range needs minimal precision). Tangents use similar techniques. Quality loss is imperceptible—compression artifacts invisible even on close-up inspection.

Optimization workflow: enable mesh compression in Unity import settings (Mesh Compression = High), validate quality visually (compare compressed vs uncompressed in-game), profile memory savings (Memory Profiler shows mesh sizes), and apply selectively (compress all meshes except specific edge cases like procedurally-modified meshes needing CPU precision).

## Key Concepts

- **Quantization**: Reducing data precision from 32-bit float to 16-bit or 8-bit integer. Positions quantize to mesh bounding box range (min/max), normals to unit sphere, UVs to 0-1 range. Decompression in GPU hardware (free runtime cost).
- **Mesh Compression Settings**: Unity offers Off, Low, Medium, High. High provides maximum compression (50-60% reduction), Low minimal (10-20%). High recommended for all meshes unless specific issues.
- **Vertex Stride**: Bytes per vertex. Uncompressed = 48 bytes typical. Compressed High = 22-28 bytes. Lower stride improves vertex cache efficiency (better GPU performance) and reduces memory bandwidth.
- **Oct-Encoding**: Encoding 3D unit vector (normal) into 2 components using octahedral mapping. Reconstructs third component from other two. 16-bit oct-encoded normal = 4 bytes vs 12 bytes uncompressed (3x savings).
- **Index Buffer Compression**: Index buffer stores triangle indices. Compresses via delta encoding (indices 0,1,2 encode as 0,+1,+1). Unity handles automatically. 16-bit indices for <65K vertices (2 bytes), 32-bit for larger (4 bytes).

## Best Practices

**Compression Level Selection:**
- Use High compression by default: 50-60% size reduction, imperceptible quality loss. Import Settings > Mesh Compression = High. Apply to all meshes (characters, environment, props).
- Use Medium for edge cases: Meshes with extremely fine detail or specific artifacts from High compression (rare). 30-40% reduction. Test visually.
- Disable compression only for procedural meshes: Meshes modified at runtime via CPU (terrain, procedural generation) may need full precision. Disable compression for these specific meshes only.
- Validate quality: Import mesh with High compression, compare in-game vs uncompressed (temporarily disable, re-import, test). Artifacts should be invisible. If visible, reduce to Medium or Off.

**Position Compression:**
- Positions quantize to 16-bit relative to mesh bounds: Unity stores min/max, quantizes vertices to 0-65535 range. Decompresses in GPU: `position = min + (quantized / 65535) * (max - min)`. Precision ~0.0015% of mesh size.
- Works for 99% of meshes: A 10m mesh has ~0.15mm precision (invisible). A 100m mesh has ~1.5mm precision (still invisible at normal viewing distances).
- Issues with huge meshes: 1000m mesh has 15mm precision (potentially visible). Solution: Split large meshes into smaller chunks (<100m each) or disable compression for specific mesh.
- Positions quantized per-mesh: Each mesh has independent bounds. Multi-submesh objects quantize per-submesh. Ensures optimal precision.

**Normal and Tangent Compression:**
- Normals quantize to oct-encoding or 16-bit per-component: Oct-encoding packs unit vector into 2 components (4 bytes total). Alternative: 3 components × 16-bit = 6 bytes (better quality, larger size).
- Tangents compress similarly: 16-bit per-component or oct-encoding. Tangent.w (handedness) stores as 1-bit sign. Total: 4-8 bytes vs 16 bytes uncompressed.
- Quality is excellent: Normal compression artifacts invisible in practice. Lighting and shading appear identical to uncompressed. Validate with side-by-side comparison.
- Reconstruction in GPU: Decompression happens in vertex shader (hardware accelerated). Zero runtime CPU cost. Minimal GPU cost (1-2 ALU instructions per vertex).

**UV Compression:**
- UVs quantize to 16-bit: 0-1 range quantized to 0-65535. Precision = 1/65535 ~0.0015%. More than sufficient for textures (even 8K textures = 0.0015% × 8192 = 0.12 pixels).
- Multiple UV sets compress independently: UV0, UV1, UV2, UV3 each quantize separately. Lightmap UVs (UV1) compress without affecting base UVs (UV0).
- Tiling UVs (>1.0) supported: Unity tracks UV range, quantizes to actual min/max (not forced 0-1). UVs spanning 0-10 quantize to that range.
- Quality validation: Inspect textures in-game. No seams, no distortion. UV compression is safest compression type (UVs are inherently discrete, not continuous).

**Vertex Color and Additional Attributes:**
- Vertex colors compress to 8-bit per channel: RGBA = 4 bytes (vs 16 bytes float). Quality sufficient for vertex colors (smooth gradients, AO baking).
- Blend weights/indices compress: Skinning weights quantize to 8-bit or 16-bit. Bone indices remain 8-bit or 16-bit (no compression needed, already minimal).
- Additional attributes (UV2, UV3, colors): All compress via quantization. Unity handles automatically based on Mesh Compression setting.

**Platform-Specific:**
- **All Platforms**: Mesh compression works universally. GPU decompression supported on all modern hardware (DX11+, OpenGL ES 3.0+, Metal, Vulkan).
- **PC**: Enable High compression. PC GPUs handle decompression efficiently. Memory and bandwidth savings beneficial even on high-end hardware (enables more complex scenes).
- **Consoles**: High compression recommended. Xbox/PlayStation GPUs decompress in vertex fetch units. No performance cost. Critical for Switch (memory constrained).
- **Mobile**: High compression essential. Reduces memory (critical on 2-4GB devices) and bandwidth (primary mobile bottleneck). Quality loss invisible on small screens.

## Common Pitfalls

**Compression Disabled by Default**: Developer imports meshes without changing default settings. Unity defaults to Mesh Compression = Off on some platforms. Meshes occupy 2x memory unnecessarily. Symptom: Memory Profiler shows meshes at full size despite expecting compression. Solution: Create Texture Preset with Mesh Compression = High, apply on import.

**Read/Write Enabled with Compression**: Developer enables Read/Write for procedural mesh modification. Unity stores both compressed GPU version and uncompressed CPU version. Mesh occupies 2x-2.5x memory (uncompressed CPU + compressed GPU). Symptom: Memory usage higher than expected despite compression. Solution: Disable Read/Write unless CPU access required. For procedural meshes needing CPU access, disable compression entirely (not both).

**Huge Meshes with Compression Artifacts**: Terrain mesh or massive building mesh >1000m spans. Position quantization to 16-bit produces 15-30mm precision. Vertices visibly snap to quantized positions (jittering, Z-fighting). Symptom: Visual glitches on huge meshes. Solution: Split large meshes into chunks <100m each, or disable compression for specific mesh. Unity's terrain system handles this automatically.

## Tools & Workflow

**Unity Mesh Import Settings**: Select mesh asset > Inspector > Meshes tab > Mesh Compression dropdown (Off, Low, Medium, High). Apply, reimport mesh. Compare file size before/after.

**Memory Profiler**: Capture snapshot > Native Memory > Mesh. Shows per-mesh memory usage. Compressed meshes show reduced size (22-28 bytes per vertex vs 48 bytes uncompressed).

**Frame Debugger**: Select draw call > Mesh section shows vertex stride. Compressed mesh: stride ~24 bytes. Uncompressed: stride ~48 bytes. Validate compression applied.

**Vertex Buffer Inspector**: RenderDoc or PIX shows vertex buffer layouts. Compressed mesh: positions 16-bit, normals 16-bit or oct-encoded, UVs 16-bit. Uncompressed: all 32-bit float.

**Mesh Compression Preset**: Create Import Preset with Mesh Compression = High, Read/Write = false. Right-click preset > Apply to folder. Batch-applies compression to all meshes. Saves manual per-mesh configuration.

**Build Size Report**: Unity Build Report (File > Build Settings > Build > check "Development Build" + "Autoconnect Profiler"). Shows per-mesh contribution to build size. Compressed meshes significantly smaller.

**Visual Comparison Script**: Custom editor script to instantiate two GameObjects (one compressed, one uncompressed), render side-by-side. Validates no visual difference from compression. Useful for QA validation.

## Related Topics

- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Mesh optimization strategies
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory reduction techniques
- [5.4 Bandwidth Optimization](05-04-Bandwidth-Optimization.md) - Bandwidth savings from reduced vertex size
- [17.1 LOD Systems](17-01-LOD-Systems.md) - LOD combined with compression

---

[← Previous: 8.1 Texture Compression Formats](08-01-Texture-Compression-Formats.md) | [Next: 8.3 Audio Compression →](08-03-Audio-Compression.md)
