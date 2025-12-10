# 27.5 Nanite-Like Techniques

[← Back to Chapter 27](../chapters/27-Advanced-Topics.md) | [Main Index](../README.md)

Nanite (Unreal Engine 5's proprietary micro-polygon rasterization system) renders billions of triangles in real-time via automatic LOD, GPU culling, and compute-based rasterization. Nanite-like techniques enable massive geometry complexity without manual LOD creation: meshes are hierarchically clustered, GPU dynamically selects appropriate cluster detail (coarse LOD for distant, fine LOD for near), and performs pixel-perfect visibility testing. This section covers cluster hierarchies, virtual geometry, GPU LOD selection, and cross-platform approximations (UE5 Nanite, custom implementations).

---

## Overview

Traditional geometry management: artists create manual LOD versions (LOD 0 = 1M triangles, LOD 1 = 500k, LOD 2 = 100k, etc.). Painful, memory-intensive, requires re-sculpting. Nanite: automatic LOD based on screen coverage. 10B triangle source asset is clustered into tree (leaf clusters = 128 triangles, parent clusters merge 4 children, etc.), LOD selection is per-cluster by GPU based on view distance, screen coverage, and occlusion. Result: artists sculpt high-detail once, engine automatically scales. Memory: 10B triangles = 1GB Nanite data (vs 5GB+ manual LODs).

Key Nanite features: hierarchical clusters (BVH-like tree), per-cluster LOD decision (GPU culling kernel, <1ms for 1B triangles), micro-polygon rasterization (pixel-level precision, 4-8 triangles per pixel on average), virtual geometry (GPU streams clusters on-demand), and material assignment per-triangle (eliminates batching constraints). Real-world: Unreal Engine 5 games (Lumen, Nanite standard), Demon's Souls remake (PS5 equivalent, custom), many AAA 2023+ titles.

## Key Concepts

- **Cluster Hierarchy**: Binary tree of geometry. Leaf nodes: 128-256 triangles (8-12 vertices per cluster). Parent nodes: union of 4 children. Build: O(log n) levels for n triangles (1B triangles = 30 levels). Cost: one-time offline (build during asset import, <1 minute for 1B triangles). Runtime: GPU traverses tree, selects detail.

- **Per-Cluster LOD Selection**: GPU kernel iterates cluster tree, for each cluster: project bounding cone to screen, measure on-screen coverage, if >4 pixels, subdivide (select child clusters); if <1 pixel, collapse (select parent cluster). Cost: 1-5ms for 1M clusters at 1440p. Deterministic per-frame (no temporal jitter).

- **Micro-Polygon Rasterization**: Nanite rasterizes clusters directly (skip traditional VS->RS->PS for small clusters). Each micro-triangle = 4-8 pixels on average (very small). HZB (hierarchical Z buffer) per-cluster occlusion testing (skip clusters completely hidden). Attribute interpolation per-pixel (vertex shader work pushed to PS, or computed in rasterizer).

- **Virtual Geometry**: Clusters streamed on-demand (GPU memory limited). 1GB Nanite data resident, disk cache. As camera moves, GPU requests clusters, driver streams. Latency: <1 frame (predict camera movement). Fallback: coarse proxy geometry while streaming.

- **Material Per-Triangle**: Traditional: 1 material per mesh or submesh (batch constraint). Nanite: per-triangle material ID (8-bit). Rasterizer outputs material ID, PS fetches material from buffer. No batching constraints, full flexibility.

- **Screen Coverage LOD**: Cluster selection based on screen projection. Metrics: pixel count on-screen, edge coverage (anti-aliasing complexity), specular highlights importance. Fine LOD if edge-heavy geometry on-screen, coarse LOD if flat. Dynamic, frame-to-frame.

## Best Practices

**Building Cluster Hierarchies:**
- Import high-poly mesh (1B+ triangles from ZBrush/Substance). Offline builder:
  1. Compute AABB per triangle (3D morton code, spatial clustering)
  2. Merge nearby triangles into clusters (128-256 tri target)
  3. Build BVH of clusters (tree)
  4. Quantize vertices per cluster (10-bit positions, 8-bit normals)
  5. Compute LOD metrics per cluster (cone angle, bounds, edge density)
  
Time: <1min for 1B triangles on modern CPU (parallel clustering). Storage: ~100B per triangle (10% compression ratio vs raw).

**GPU LOD Selection (Compute Shader):**
```hlsl
[numthreads(256, 1, 1)]
void SelectClusterLOD(uint3 id : SV_DispatchThreadID)
{
    uint clusterIdx = id.x;
    if (clusterIdx >= g_ClusterCount) return;
    
    ClusterBounds bounds = g_ClusterBounds[clusterIdx];
    float4 projBounds = ProjectToScreen(bounds.min, bounds.max, g_ViewProj);
    float screenCoverage = (projBounds.z - projBounds.x) * (projBounds.w - projBounds.y);
    
    // LOD selection: if coverage > threshold, select children; else select this cluster
    if (screenCoverage > 256.0)  // > 16x16 pixels
    {
        // Select children (4 child clusters per parent)
        uint childIdx = clusterIdx * 4;
        for (int i = 0; i < 4; i++)
            g_VisibleClusters.Append(childIdx + i);
    }
    else if (screenCoverage > 1.0)  // > 1 pixel, visible
    {
        g_VisibleClusters.Append(clusterIdx);
    }
    // Else: coverage < 1 pixel, culled
}
```

**Occlusion Testing (Per-Cluster HZB):**
- Build Hi-Z pyramid of depth buffer (8x8 -> 4x4 -> 2x2 -> 1x1). Cost: 33% of depth bandwidth.
- Per-cluster occlusion test: sample Hi-Z at cluster's screen bounds, if max depth > cluster's far-plane depth, potentially visible; else occluded. Cost: 1-2 clocks per cluster (cached).
- Conservative test (no false negatives): if cluster potentially behind other geometry, test conservatively (allow false positives, better to render unnecessary cluster than cull visible one).

**Virtual Geometry (Streaming):**
- Resident set: coarse clusters (LOD 0-2) always in VRAM (10-50MB). Fine clusters on disk.
- Predict camera next-frame position. Queue streaming requests for clusters likely visible.
- Streaming bandwidth: 100-200 MB/s (SSD typical). If camera moves fast, streaming can't keep up; fallback to coarser LOD.
- Latency hiding: render frame N with available clusters, stream frame N+1 clusters in background. One-frame latency acceptable.

**Material Per-Triangle:**
```hlsl
// Per-cluster material assignment
StructuredBuffer<uint> g_ClusterMaterials;  // Cluster -> Material ID

PS_Output PS_Main(PS_Input input)
{
    // input.clusterIdx comes from rasterizer (cluster ID)
    uint materialID = g_ClusterMaterials[input.clusterIdx];
    Material mat = g_Materials[materialID];
    
    float3 color = SampleTexture(mat.diffuseMap, input.uv);
    return float4(color, 1.0);
}
```

## Platform-Specific Guidance

**PC (UE5 Nanite Native):**
- UE5.0+: Nanite fully supported. Automatic LOD, GPU culling, virtual geometry streaming. Enable on all assets (Nanite checkbox in import settings).
- Performance: 1B triangles rendered at 60 fps 1440p with ray-traced shadows (RTX 3080+). Scales with GPU (RTX 2080 Ti: 500M triangles at 60 fps).
- VRAM: 256MB resident clusters, disk cache (SSD required for streaming). 100-200 MB/s streaming bandwidth saturates modern SSD.

**PlayStation 5:**
- Native Nanite support (Unreal Engine 5 PS5 version). Custom SSD enables aggressive streaming (5.5 GB/s). Clusters stream in parallel with frame rendering (minimal latency).
- Performance: PS5 renders 2-4B triangles at 60 fps (1440p). Efficient custom hardware (5.5 TFLOPS ray compute, custom SSD I/O).
- Memory budget: 12GB VRAM available (OS takes 4GB), Nanite clusters ~4-5GB resident + streaming.

**Xbox Series X:**
- Nanite supported (UE5). Similar performance to PS5 (2-4B triangles at 60 fps). Smart Delivery scales Series S vs Series X automatically (Series S: 1-2B triangles, lower LOD).
- DirectStorage support (native, Windows equivalent). Enables ultra-fast streaming (similar to PS5 SSD benefits).
- Optimization: tune LOD bias per platform (Series S more aggressive LOD culling).

**Mobile (Custom Implementation):**
- Full Nanite not feasible (limited VRAM, slow storage). Custom approximation: pre-baked LOD (artist creates 3-4 manual LOD levels). GPU LOD selection (similar to Nanite) with limited LOD levels.
- Performance: 100M-500M triangles on flagship Android (Snapdragon 8 Gen 2), 10M-50M on mid-range. Material per-triangle possible (shader variant, not GPU streaming).
- VRAM budget: 128-256MB total (clusters + materials + textures). Manual LODs acceptable.

## Common Pitfalls

**1. Cluster Overdraw (Redundant Geometry Rendering)**
- *Symptom*: GPU utilization 100%, overdraw 8-10x (should be 1-2x), frame rate <30 fps despite "only" 1M visible triangles.
- *Cause*: LOD selection threshold too conservative. Fine clusters selected even when coarse sufficient (screen coverage > threshold not working correctly).
- *Solution*: Tune LOD threshold. If 256x256 pixel cluster selected when 64x64 sufficient, increase threshold (or reduce screen-space metric). Debug: visualize selected LOD (color per LOD level). Overdraw should be <2x.

**2. Visible Pop-In (LOD Transitions Jarring)**
- *Symptom*: Geometry suddenly pops between LOD levels as camera moves. Noticeable quality drop.
- *Cause*: LOD selection too aggressive. Fine LOD -> coarse LOD transition instant when crossing threshold.
- *Solution*: Use hysteresis (differ thresholds for transitioning in vs out). Or smooth LOD transition (blend fine/coarse over multiple frames). Or increase LOD quality (finer clusters, smaller threshold).

**3. Streaming Latency (Missing Cluster Geometry)**
- *Symptom*: Certain clusters render with wrong LOD or missing entirely as camera moves fast. Visual glitches.
- *Cause*: Streaming can't keep up with camera movement. Predicted clusters not streaming fast enough.
- *Solution*: Increase streaming bandwidth (SSD optimization). Or predict camera further ahead (use velocity history to extrapolate). Or increase resident set size (keep more clusters in VRAM, fewer disk accesses).

**4. Material Binding Per-Triangle Overhead**
- *Symptom*: Pixel shaders take 50-100ms (should be <10ms), despite "simple" material shading.
- *Cause*: Material lookup per-triangle expensive (structure buffer access per pixel). Or shader variants per material (many branches).
- *Solution*: Use single unified material shader (one shader, material parameters per-triangle). Or batch by material (render clusters of same material together, avoid per-pixel branch). Cache material lookups in LDS (groupshared memory).

## Tools & Workflow

**Nanite Cluster Building (Offline):**
```python
# Pseudo-code: cluster builder (offline tool)
def BuildNaniteClusters(mesh_vertices, mesh_indices, cluster_size=128):
    # Spatial clustering via morton codes
    triangles = []
    for i in range(0, len(mesh_indices), 3):
        tri = (mesh_vertices[mesh_indices[i]], 
               mesh_vertices[mesh_indices[i+1]], 
               mesh_vertices[mesh_indices[i+2]])
        morton_code = ComputeMortonCode(TriangleCenter(tri))
        triangles.append((morton_code, tri))
    
    # Sort by morton code (spatial locality)
    triangles.sort(key=lambda x: x[0])
    
    # Create clusters
    clusters = []
    for i in range(0, len(triangles), cluster_size):
        cluster = triangles[i:i+cluster_size]
        clusters.append(Cluster(cluster))
    
    # Build BVH tree
    bvh = BuildBVH(clusters)
    return bvh
```

**Profiling Nanite Rendering:**
- NVIDIA Nsight / PIX: per-cluster rasterization time, LOD selection time (<1ms target).
- Overdraw visualization: render coverage (white = 1x, red = 8x+). Target <2x overdraw.
- Streaming metrics: bytes/frame streamed, latency (frame until cluster available).

## Related Topics

- [27.3 GPU-Driven Rendering](27-03-GPU-Driven-Rendering.md) — LOD selection via GPU culling kernel
- [27.4 Next-Gen Features](27-04-Next-Gen-Features.md) — DirectStorage, mesh shaders enable Nanite-like tech
- [17 Level-Of-Detail Systems](../chapters/17-Level-Of-Detail-Systems.md) — Automatic LOD fundamentals
- [07 Asset Optimization](../chapters/07-Asset-Optimization.md) — Cluster quantization, compression

---

[← Previous: 27.4 Next-Gen Features](27-04-Next-Gen-Features.md) | [Next: 28.1 Performance Metrics →](28-01-Performance-Metrics.md)
