# 1.4 Fragment Processing

[← Back to Chapter 1](../chapters/01-Graphics-Pipeline-Fundamentals.md) | [Main Index](../README.md)

Fragment processing executes pixel shaders on rasterized fragments to compute final color and depth values. This stage dominates GPU time in most rendering workloads, making optimization critical.

---

## Overview

Fragment shaders (pixel shaders) run once per rasterized pixel, computing color from interpolated vertex data, texture samples, and lighting calculations. Unlike vertex shaders (thousands of invocations), fragment shaders execute millions or billions of times per frame—4K resolution (3840×2160) with 4x overdraw = 33 million invocations. A complex shader taking 500 cycles costs 16.5 billion cycles at 60fps, easily saturating GPU compute units.

Modern GPUs execute fragments in groups called waves, warps, or quads (terminology varies by vendor). NVIDIA uses 32-thread warps; AMD uses 64-thread wavefronts; GPUs process fragments in 2x2 pixel quads for texture derivative calculations (ddx/ddy for mipmapping). This SIMD execution model means all threads in a group execute the same instructions—branches cause divergence where some threads idle while others work, wasting parallelism.

Early-Z optimization checks depth before fragment shading, rejecting occluded fragments to save costly shader execution. Rendering opaque objects front-to-back maximizes early-Z effectiveness. However, alpha testing, discard, or writing depth in shaders disables early-Z, forcing full shader execution for every rasterized fragment. Understanding these interactions is essential for 60fps performance on console and PC.

## Key Concepts

- **Fragment Shader**: Programmable stage computing per-pixel color and depth. Takes interpolated vertex data (UVs, normals, position) and resources (textures, constants), outputs color and optional depth.
- **Early-Z**: Hardware optimization testing fragment depth against Z-buffer before pixel shader execution. Rejects occluded fragments early, saving shader cost. Disabled by alpha testing, discard, or manual depth writes.
- **Pixel Quad**: 2x2 fragment group processed together. Required for texture derivative calculations (mipmap level). Causes overshading—small triangles activate entire quad even if covering <4 pixels.
- **Helper Invocations**: Extra fragments outside triangle bounds but within quads, executed to provide derivatives for neighboring pixels. Don't write outputs but consume shader ALU cycles.
- **Overdraw**: Fragments shaded multiple times per pixel due to overlapping geometry. 4x overdraw means each pixel is shaded 4 times on average—quadrupling fragment shader cost.

## Best Practices

**Shader Complexity Management:**
- Target 200-300 ALU instructions for base pixel shaders on console (150 on Switch, 400-500 on PS5/Series X acceptable). Check compiled shader stats in RenderDoc or PIX.
- Move view-independent calculations to vertex shaders (world-space normals, lightmap UVs) and interpolate. Costs fewer vertex executions than millions of fragment executions.
- Use shader LOD—simpler shaders for distant objects. Unity's shader LOD system or manual variants (`#pragma multi_compile LOD_HIGH LOD_LOW`) reduce complexity based on distance.
- Prefer ALU operations over texture samples (ratio 4:1). Texture fetches cost 10-100x more due to memory latency. Cache thrashing multiplies this cost.

**Early-Z Optimization:**
- Render opaque geometry front-to-back. Sort by depth (camera distance) to maximize early-Z rejection. Can eliminate 70-90% of fragment shading in dense scenes.
- Avoid `discard`, `clip()`, or alpha testing in base pass shaders—they disable early-Z. Use alpha-to-coverage for foliage instead, preserving early-Z.
- Implement Z-prepass (depth-only pass before color) for complex scenes with expensive shaders. Costs extra draw calls but can save 50% total fragment cost when overdraw is high.
- Never write `SV_Depth` (manual depth output) unless required for special effects. It disables early-Z and Hi-Z, forcing full shader execution for every fragment.

**Quad Efficiency:**
- Minimize small triangles (<10 pixels). A 3-pixel triangle wastes 25% of quad (1 pixel idle). Aggregate via LOD or billboards.
- Be aware that discard/clip creates non-uniform control flow. If one quad pixel discards but others don't, entire quad executes full shader to provide derivatives.
- Use `ddx_coarse`/`ddy_coarse` in DirectX 12 when possible—calculates derivatives at 2x2 quad granularity instead of per-pixel, halving derivative cost.

**Overdraw Reduction:**
- Profile overdraw with RenderDoc or PIX "Overdraw" view. Target <2x average for opaque, <5x for scenes with heavy transparency.
- Sort opaque front-to-back as mentioned. Sort transparents back-to-front only (required for alpha blending correctness).
- Use GPU occlusion culling (Hi-Z on consoles) to eliminate objects behind opaque surfaces before issuing draws.
- Reduce particle overdraw—use simpler shaders, fewer particles, or alpha-to-coverage tricks. Particle-heavy scenes easily hit 10-20x overdraw in screen center.

**Platform-Specific:**
- **PC DirectX 12**: Use wave intrinsics (`WaveActiveSum`, `WaveReadLaneFirst`) for efficient inter-fragment communication. Requires shader model 6.0+.
- **Xbox Series X/S**: Variable Rate Shading (VRS) reduces shading rate in peripheral vision or low-detail areas. Saves 30-40% fragment cost with careful tuning.
- **PlayStation 5**: Async compute allows overlapping post-processing (compute shaders) with geometry rendering. Schedule post-processing dispatches during fragment-heavy draws.
- **Switch**: Fragment shading is primary bottleneck. Keep shaders under 150 instructions, minimize texture samples (2-3 max), use 16-bit precision aggressively.

## Common Pitfalls

**Unbounded Loops in Pixel Shaders**: `for (int i = 0; i < numLights; i++)` where `numLights` is dynamic (from constant buffer) prevents compiler unrolling and causes terrible performance. GPUs execute loops sequentially per thread. A 10-iteration loop in a 200-instruction shader becomes 2000 instructions. Solution: Use fixed loop counts (`#define MAX_LIGHTS 8`) or compute shaders for heavy lighting.

**Texture Thrashing with Random Access**: Sampling textures with computed UVs (noise patterns, procedural distortion) destroys texture cache coherency. Neighboring fragments sample distant texels, causing cache misses. Symptom: Low occupancy, high memory latency in profilers. Solution: Use texture arrays or 3D textures with coherent sampling patterns, or precompute to render targets.

**Ignoring Helper Invocations**: Small triangles near edges activate helper fragments outside triangle bounds. These execute the full shader (for derivatives) but don't write outputs. A 5-pixel triangle might activate 8 fragments (3 helpers). Profilers show high fragment count but low pixel output. Solution: Merge small triangles via LOD, use instanced billboards for distant objects.

## Tools & Workflow

**RenderDoc Pixel Debugger**: Right-click pixel > "Debug Pixel" to step through shader execution. Shows all variables, texture samples, and control flow. Essential for fixing visual bugs or understanding performance.

**PIX Shader Analysis**: Select draw call, click "Shader Perf" to see ALU, texture, and memory costs per shader. "Occupancy" view reveals register pressure or memory latency limiting parallelism. Green (>75% occupancy) = good, red (<50%) = investigate.

**Unity Frame Debugger**: GPU Profiler shows millisecond cost per draw. Sort by cost to identify expensive shaders. Click draw to see shader code and assigned material. Quickly identifies which shaders need optimization.

**NVIDIA Nsight Graphics**: Pixel history shows all fragment shader executions for a pixel—reveals overdraw and shading order. Shader Profiler breaks down instruction costs (ALU vs texture vs memory). Use "PC Sampling" for hotspot identification.

**Platform Profilers**: PlayStation Razor shows wavefront occupancy (parallelism efficiency) and VGPR usage (register pressure). Xbox PIX exposes early-Z rejection rates—low rates suggest front-to-back sorting issues or depth writes disabling optimization.

## Related Topics

- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Optimizing fragment shader performance
- [12.3 Shader Optimization Techniques](12-04-Advanced-Shader-Techniques.md) - Advanced shader optimization
- [1.5 Output Merger](01-05-Output-Merger.md) - Next pipeline stage (depth/blend)
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Identifying fragment-bound scenarios

---

[← Previous: 1.3 Primitive Assembly and Rasterization](01-03-Primitive-Assembly-Rasterization.md) | [Next: 1.5 Output Merger →](01-05-Output-Merger.md)
