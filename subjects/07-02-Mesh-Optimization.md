# 7.2 Mesh Optimization

[← Back to Chapter 7](../chapters/07-Asset-Optimization.md) | [Main Index](../README.md)

Mesh optimization reduces memory consumption, improves rendering performance through cache efficiency, and enables batching by lowering vertex counts.

---

## Overview

Meshes consume 5-15% of VRAM in typical games (textures dominate at 60-80%). However, mesh optimization has disproportionate impact on performance: poor mesh topology causes vertex cache misses (30-50% performance loss), excessive triangle counts overload rasterization (GPU-bound), and high vertex counts prevent batching (CPU-bound from draw calls). Optimized meshes render 2-5x faster than unoptimized equivalents.

Mesh optimization targets three areas: vertex count reduction (fewer vertices = less memory, faster vertex processing), triangle reordering (improve post-transform vertex cache hits from 50% to 95%), and vertex format optimization (16-bit precision, compressed attributes). Additionally, LOD (Level of Detail) systems dynamically reduce mesh complexity based on distance, providing 3-10x performance improvements in large scenes.

Optimization workflow: audit meshes via Statistics window (show triangle/vertex counts), reduce high-poly meshes in DCC tools (Blender, Maya), enable mesh compression in import settings (50-70% size reduction), implement LOD groups (3-5 levels), and profile with Frame Debugger (identify expensive draws). Modern techniques like GPU instancing allow rendering thousands of meshes with single draw call, but require optimized base meshes to maximize benefits.

## Key Concepts

- **Vertex Count**: Number of unique vertices in mesh. Each vertex costs memory (24-48 bytes) and processing time. Target <10K vertices for characters, <5K for props, <1K for background objects.
- **Triangle Count**: Number of triangles (3 indices each). Determines rasterization cost and fragment workload. Target <20K triangles for characters, <10K for props, <2K for background.
- **Vertex Cache**: GPU cache storing recently transformed vertices (typically 16-32 entries). Reused vertices skip vertex shader. Well-ordered meshes hit 90-95% cache, poorly ordered hit 40-60%.
- **Mesh Compression**: Quantizing vertex attributes (positions, normals, UVs) to 16-bit precision. Reduces mesh size by 40-60% with imperceptible quality loss. Unity import setting: Mesh Compression = High.
- **LOD (Level of Detail)**: Multiple mesh versions at different complexity levels. Engine selects based on screen size/distance. LOD0 (close): full detail. LOD1-2: reduced complexity. LOD3+: very low poly or billboards.

## Best Practices

**Vertex Count Reduction:**
- Model to target budgets: Characters (5-15K vertices), environment props (2-5K), small props (500-2K), particles/billboards (4-12 vertices). Reserve budget for hero assets.
- Remove hidden geometry: Delete back-faces (walls, terrain), interior faces (inside buildings), and occluded details (underside of objects never seen by camera).
- Use normal maps instead of geometry: Bake high-poly details to normal map, use on low-poly mesh. 100K triangle high-poly → 5K triangle low-poly with normal map—indistinguishable at distance.
- Avoid excessive edge loops: Cylindrical objects (columns, pipes) need 8-16 sides typically, not 64. Spheres need 100-300 vertices, not 10,000.
- Optimize for silhouette: Prioritize vertex density on visible silhouette edges. Interior triangles invisible from distance—reduce aggressively.

**Triangle Reordering:**
- Enable mesh optimization in import: Unity 2021+ optimizes automatically. Older Unity: Use MeshOptimizer library or enable "Optimize Mesh" in import settings.
- Vertex cache optimization improves hit rate from 40-60% (random order) to 90-95% (optimized). Results in 30-50% vertex processing speedup (GPU spends less time re-running vertex shaders).
- Manually optimize in DCC tools: Blender "Decimate" modifier has "Optimize" pass. Maya has "Mesh > Cleanup > Optimize".
- Validate optimization: RenderDoc/PIX show vertex reuse metrics. Target >0.9 average reuse ratio (each vertex transformed only ~1.1 times instead of ~2-3 times).

**Vertex Format Optimization:**
- Enable mesh compression: Import Settings > Mesh Compression = High. Quantizes positions, normals, UVs to 16-bit. Reduces size by 40-60%, minimal quality loss.
- Use appropriate precision per attribute: Positions (16-bit relative to bounds), normals (16-bit oct-encoded or 32-bit), UVs (16-bit typical), colors (8-bit).
- Avoid unnecessary vertex attributes: Remove vertex colors if unused (4 bytes per vertex). Remove UVs 2-4 if not needed (8-16 bytes per vertex). Tangents auto-calculated if needed.
- Pack attributes efficiently: 48-byte vertex (pos 12B + normal 12B + UV 8B + tangent 16B) reduces to 28-32 bytes with compression (pos 8B + normal 4B + UV 4B + tangent 8B).

**LOD Implementation:**
- Use 3-5 LOD levels: LOD0 (0-10m): full detail. LOD1 (10-50m): 50% triangles. LOD2 (50-100m): 25% triangles. LOD3 (100-200m): 10% triangles. LOD4 (200m+): billboard/impostor.
- Set LOD transition distances based on screen coverage: LOD transitions when object occupies 50%, 25%, 10%, 5% of screen height respectively. Unity LODGroup component calculates automatically.
- Generate LODs in DCC tools: Blender "Decimate" modifier with target triangle ratios. Maya "Reduce" tool. Or use Unity's built-in LOD generation (Simplygon integration in Unity 2020+).
- Profile LOD effectiveness: Statistics window shows current triangles/vertices. Move camera, verify counts drop as objects distance. Target 3-10x reduction at max distance.

**Read/Write Enabled Management:**
- Disable "Read/Write Enabled" by default: Import Settings > Read/Write Enabled = false. Enabling duplicates mesh in CPU RAM + GPU VRAM (2x memory).
- Enable only when needed: Procedural mesh modification, collision mesh access, runtime mesh analysis. Most meshes never need CPU access after upload.
- Symptom of Read/Write enabled globally: Memory Profiler shows mesh occupying double expected memory. Solution: Audit all meshes, disable unless explicitly required.

**Platform-Specific:**
- **PC**: Handle 50-100K triangles per mesh on modern GPUs. Still optimize for min-spec (GTX 1060 level): Target <20K triangles for characters, <50K for complex environment pieces.
- **Xbox Series X/S**: Series X handles complex meshes well (RDNA 2 architecture). Series S more limited (fewer CUs). Target <15K triangles for Series S characters, <30K for environments.
- **PlayStation 5/4**: PS5 similar to Series X (<20K characters). PS4 (GCN architecture) slower at vertex processing. Target <10K triangles for PS4 characters, aggressive LOD.
- **Switch**: Vertex processing limited (Maxwell-based GPU). Target <5K triangles for characters, <2K for props. LOD essential—implement 4-5 levels with aggressive transitions.

## Common Pitfalls

**Read/Write Enabled on All Meshes**: Developer enables globally for one procedural mesh script, forgetting to disable per-mesh. Now every mesh occupies double memory (CPU + GPU copy). 500MB mesh memory becomes 1GB. Symptom: Memory usage doubles, Memory Profiler shows meshes with "Instance Count = 2". Solution: Disable Read/Write Enabled in import settings except specific procedural meshes.

**No LOD Implementation**: Game renders full-detail 20K triangle characters at 200m distance (occupying 10 pixels on screen). 100 characters = 2M triangles for 1,000 pixels (2,000 triangles per pixel overdraw). GPU overwhelmed. Symptom: Frame rate drops in distant views with many objects. Solution: Implement LOD groups with 3-5 levels, transition at 10m, 50m, 100m, 200m.

**Unoptimized Mesh Topology**: Artist models in DCC, exports without optimization. Vertex order random, cache hit rate 40-60%. Each vertex transformed 2-3 times (should be 1.1 times). Symptom: Frame Debugger shows high vertex processing cost. Solution: Enable "Optimize Mesh" in Unity import settings or use MeshOptimizer library to reorder vertices/triangles.

## Tools & Workflow

**Unity Statistics Window**: Game View > Stats button (top-right). Shows real-time triangle/vertex counts, draw calls, batches. Essential for profiling mesh complexity in scene.

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows per-draw triangle/vertex counts. Click draw, see mesh details. Identify expensive draws (>100K triangles per draw).

**Memory Profiler**: Captures mesh memory usage. Native Memory > Mesh shows per-mesh size. Sort by size, identify oversized meshes (50MB mesh for small prop).

**LODGroup Component**: Attach to GameObject with multiple mesh children (LOD0, LOD1, LOD2, etc.). Configure transition percentages (50%, 25%, 10%). Scene view shows LOD colors (blue = LOD0, green = LOD1, yellow = LOD2, red = LOD3).

**Blender Decimate Modifier**: Select mesh > Modifiers > Decimate > Ratio (0.5 = 50% triangles). Apply modifier, export. Generates LOD levels quickly.

**MeshOptimizer**: Unity package (com.unity.meshopt.decompress) or standalone library. Reorders vertices/triangles for optimal cache performance. Reduces vertex transform cost by 30-50%.

**RenderDoc Mesh Viewer**: Capture frame > Event > Mesh Viewer tab. Shows vertex buffer, index buffer, vertex reuse statistics. "Cache Utilization" metric indicates optimization quality (target >90%).

**Simplygon**: Commercial mesh optimization tool integrated in Unity 2020+ (requires license). Automatic LOD generation, vertex welding, triangle reduction. Accessed via Inspector > LOD Group > Generate LODs.

## Related Topics

- [17.1 LOD Systems](17-01-LOD-Systems.md) - Comprehensive LOD strategies
- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - Draw call reduction via batching
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Vertex-bound scenario identification
- [2.3 GPU Components](02-03-GPU-Components.md) - Vertex processing hardware

---

[← Previous: 7.1 Texture Optimization](07-01-Texture-Optimization.md) | [Next: 7.3 Shader Optimization →](07-03-Shader-Optimization.md)
