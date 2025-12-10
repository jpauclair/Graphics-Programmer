# 5.1 CPU-Side Optimization

[← Back to Chapter 5](../chapters/05-Performance-Optimization.md) | [Main Index](../README.md)

CPU-side optimization focuses on reducing the cost of submitting rendering work to the GPU, a critical bottleneck in complex 3D scenes across all platforms.

---

## Overview

Every object rendered requires CPU work to prepare draw calls, set GPU state, and validate resources. On PC and consoles, the graphics API (DirectX, GNM, NVN) introduces overhead for each draw call—typically 0.1-1.0ms per thousand draws. In a scene with 5,000 objects, this can consume 5-10ms of your 16ms frame budget, leaving insufficient time for gameplay, physics, and GPU rendering.

CPU optimization strategies reduce draw call count through batching, minimize state changes via material sharing, and leverage modern rendering techniques like GPU instancing and the SRP Batcher. Console platforms benefit more from these optimizations due to lower CPU clock speeds (Xbox Series X: 3.8GHz vs high-end PC: 5.5GHz+), making efficient rendering architecture essential for hitting 60fps targets.

Unity's rendering pipeline performs frustum culling, LOD selection, and occlusion testing on CPU before submitting draws. Optimizing these systems—especially for open-world or dense indoor scenes—can reclaim significant frame time and improve GPU utilization by reducing idle periods.

## Key Concepts

- **Draw Call**: CPU command instructing GPU to render specific geometry with a material/shader. Each call has overhead from API validation and GPU state changes.
- **Batch**: Group of objects rendered with a single draw call. Static batching combines meshes at build time; dynamic batching combines at runtime.
- **SRP Batcher**: Unity's URP/HDRP optimization that caches material properties in GPU memory, eliminating per-object CPU uploads. Requires compatible shaders.
- **GPU Instancing**: Rendering multiple copies of a mesh with one draw call. Each instance can have unique transforms and properties (color, metallic, etc.).
- **Culling**: Process of determining which objects are visible before rendering. Frustum culling removes off-screen objects; occlusion culling removes objects blocked by others.

## Best Practices

**Draw Call Reduction:**
- Aim for <1,500 draw calls on PC, <1,000 on Xbox One/PS4, <500 on Switch. Next-gen consoles (PS5/Series X) handle 2,000-3,000 efficiently.
- Use Unity Frame Debugger to identify high draw call sources. Sort by "Total" column to prioritize optimization effort.
- Combine materials where possible. Two objects with identical shaders/textures but different material assets generate separate draws—merge them.
- Apply texture atlasing for environments. A city scene with 50 building materials can become 5-10 atlased materials, reducing draws by 80%.

**Batching Strategy Selection:**
- **Static Batching**: Use for environment geometry (buildings, rocks, trees). Increases memory by duplicating shared meshes but eliminates draw call overhead entirely. Ideal for console with tight CPU budgets.
- **Dynamic Batching**: Unity automatically batches small meshes (<300 vertices). Good for particles and UI, but breaks easily (different scales, materials, lightmaps). Monitor Frame Debugger "Batching" reasons for breaks.
- **GPU Instancing**: Best for repeated objects (foliage, crowds, props). Enable "Enable GPU Instancing" on materials. Use MaterialPropertyBlock for per-instance variation (color, wind offset). Requires instancing-compatible shaders.
- **SRP Batcher**: Enable in URP/HDRP settings. Fastest option for objects with SRP-compatible shaders. Check Shader Graph's "SRP Batcher Compatible" in inspector. Reduces CPU time per draw from ~0.5ms to ~0.05ms.

**Culling Optimization:**
- Bake occlusion data for dense scenes (cities, interiors). Unity's built-in occlusion culling is CPU-heavy but effective—aim for 30-50% cull rate to justify cost.
- Set aggressive camera far plane distances (500-1000m instead of 10,000m). Culls distant objects and improves depth buffer precision.
- Use Layer Culling Distances in camera settings to cull small objects (debris, small props) at 50-100m. Players won't notice but saves hundreds of draws.

**Platform-Specific:**
- **Xbox/PlayStation**: Prioritize static batching and SRP Batcher over dynamic batching. Console CPUs benefit more from eliminating draws than minimizing mesh data.
- **Switch**: Use GPU instancing extensively. Switch's Tegra GPU handles instancing well but struggles with high draw call counts (>500 impacts frame rate).
- **PC**: Balance draw calls with batch size. High-end PCs handle 3,000+ draws easily, but min-spec CPUs (4-core) struggle past 1,500.

## Common Pitfalls

**Over-Batching Static Geometry**: Static batching duplicates mesh data in memory. Batching a 100MB building set across 50 instances creates 5GB of mesh data, causing memory pressure on consoles. Use instancing for repeated assets instead—it shares the original mesh and costs only transform data.

**Ignoring SRP Batcher Compatibility**: Many custom shaders break SRP Batcher by using per-material CBUFFER declarations incorrectly. Use Shader Graph or follow SRP shader templates. A scene with 2,000 incompatible draws runs 3-5x slower than SRP Batcher equivalents. Check Frame Debugger for "SRP Batcher compatibility: No" warnings.

**Excessive Dynamic Batching Reliance**: Dynamic batching has strict requirements (same material, small vertex count, uniform scale). It breaks silently, leaving you with unbatched draws. Modern projects should use SRP Batcher or instancing—dynamic batching is legacy and unreliable for optimization.

## Tools & Workflow

**Unity Frame Debugger** (Window > Analysis > Frame Debugger): Shows every draw call with batch/instance counts. "Why this draw call can't be batched" reveals breaking conditions. Use in Play Mode to profile actual game scenes, not isolated test scenes.

**Unity Profiler - Rendering Module**: "Batches" counter shows total draw calls. Compare "Batches" vs "Saved by batching" to measure effectiveness. "SetPass calls" indicates material changes—target <500 on console, <1,000 on PC.

**SRP Batcher Profiler** (Window > Analysis > Rendering Debugger > SRP Batcher): Shows per-batch GPU memory usage and compatibility status. Green batches are optimal; red indicates incompatible shaders needing fixes.

**PIX/RenderDoc**: Capture a frame to see actual GPU draw calls submitted. Unity's Frame Debugger shows logical draws; external tools reveal what hardware sees after batching/instancing. Validates optimization effectiveness.

**Jobs System Integration**: Use `IJobParallelFor` for custom culling or LOD calculations. Burst-compiled jobs run 10-20x faster than MonoBehaviour loops. Unity's rendering jobs (culling, skinning) automatically parallelize on 4+ cores.

## Related Topics

- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Optimizing shader and GPU execution
- [6.1 CPU Bottlenecks](06-01-CPU-Bottlenecks.md) - Identifying when CPU limits rendering
- [17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) - Level of detail systems
- [23.1 Unity Jobs System](23-01-Unity-Jobs-System.md) - Multithreading rendering work

---

## Related Topics

- [6.1 CPU Bottlenecks](06-01-CPU-Bottlenecks.md) - CPU bottleneck identification
- [17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) - LOD system details
- [23.1 Unity Jobs System](23-01-Unity-Jobs-System.md) - Jobs System deep dive

---

[← Previous: Chapter 5 Index](../chapters/05-Performance-Optimization.md) | [Next: 5.2 GPU-Side Optimization →](05-02-GPU-Side-Optimization.md)
