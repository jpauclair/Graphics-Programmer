# 2.3 GPU Components

[← Back to Chapter 2](../chapters/02-GPU-Hardware-Architecture.md) | [Main Index](../README.md)

Modern GPUs contain specialized hardware units optimized for specific tasks. Understanding these components helps identify bottlenecks and optimize rendering workloads effectively.

---

## Overview

GPUs are not monolithic processors—they're collections of specialized units working in parallel. Compute units execute shader code, TMUs sample textures, rasterizers convert triangles to pixels, and ROPs write final colors to framebuffers. Each unit has finite throughput; saturating one creates a bottleneck while others idle. A scene with 100M texture samples/frame may be TMU-bound, while a scene with 8K resolution and heavy blending may be ROP-bound.

Balancing workloads across GPU components is key to performance. If vertex shaders consume 2ms and fragment shaders 8ms, the geometry pipeline (rasterizers, TMUs for vertex textures) idles 75% of the time. Modern rendering techniques (deferred shading, compute-based post-processing) shift work between components to maximize utilization. Understanding component throughput helps architects design rendering pipelines that keep all units busy.

Console GPUs have fixed component counts—Xbox Series X has 208 TMUs, 112 ROPs; PlayStation 5 has 144 TMUs, 64 ROPs. PC GPUs vary wildly (RTX 4090 has 512 TMUs, 176 ROPs). Optimization targets differ: console optimization focuses on fixed hardware limits, PC optimization must scale across hardware tiers.

## Key Concepts

- **Texture Mapping Unit (TMU)**: Specialized hardware for texture sampling, filtering (bilinear/trilinear/anisotropic), and mipmap level calculation. Modern GPUs have 144-512 TMUs, each handling 1-4 samples per clock.
- **Render Output Unit (ROP)**: Fixed-function hardware performing depth/stencil tests, blending, and framebuffer writes. Limited count (64-176) makes ROPs potential bottleneck at high resolutions or with heavy blending.
- **Rasterizer**: Converts primitives (triangles) to pixel fragments. Determines coverage, interpolates attributes, generates fragments for shading. Highly optimized fixed-function hardware handling billions of pixels/sec.
- **Compute Unit (CU/SM)**: Programmable processor executing shader code (vertex, fragment, compute). Contains multiple shader cores (32-128), registers, and shared memory. NVIDIA calls them SMs, AMD calls them CUs.
- **Command Processor**: Fetches commands from command buffers, dispatches work to GPU units, manages scheduling. Acts as GPU's "brain", coordinating all components.

## Best Practices

**TMU Optimization:**
- Minimize texture samples in shaders. Each sample costs TMU cycles. Pack textures into channels (R=metallic, G=occlusion, B=detail, A=smoothness) to reduce samples from 4 to 1.
- Use mipmapping always. Without mips, TMU fetches high-resolution texels for distant pixels, wasting bandwidth and cache. Mips improve TMU efficiency by 80%+.
- Leverage anisotropic filtering on PC/current-gen consoles (8x-16x). TMU hardware is optimized for it—quality improvement with minimal cost. On Switch, use 2x-4x max.
- Avoid dependent texture reads (sampling texture A to compute UVs for texture B). Prevents TMU pipelining, serializing samples. Costs 2-3x more than independent samples.

**ROP Optimization:**
- Reduce resolution for ROP-heavy workloads. 4K (8.3M pixels) has 4x the ROP load vs 1080p (2.1M pixels). Use dynamic resolution scaling for 60fps targets.
- Minimize blending operations. Each blend reads and writes framebuffer, doubling ROP bandwidth. Sort opaque front-to-back to leverage early-Z (no blend).
- Limit MRT count. 8 render targets at 4K = 266MB framebuffer writes per frame at 60fps = 16 GB/sec. Exceeds Xbox One's 68 GB/sec significantly.
- Use fast clear values (depth 0.0 or 1.0, color black/white) when possible. ROPs have optimized clear paths for common values, saving bandwidth.

**Rasterizer Efficiency:**
- Avoid tiny triangles (<8 pixels). Rasterizer processes 2x2 pixel quads; small triangles waste quad slots (overshading). Use LOD to merge distant triangles.
- Enable back-face culling for closed meshes. Culls 50% of triangles before rasterization, saving setup cost and fragment shader work.
- Use conservative rasterization (DirectX 12/Vulkan) for voxelization or tight collision bounds. Marks pixels as covered if triangle touches them at all.
- Profile rasterizer utilization (Nsight, RGP). >90% suggests rasterization bottleneck (massive triangle counts or small triangle inefficiency).

**Compute Unit Utilization:**
- Balance vertex/fragment/compute workloads. If fragment shaders dominate (10ms) and vertex shaders are light (1ms), CUs idle during vertex processing. Consider compute-based vertex processing.
- Use async compute to fill idle CUs. During shadow map rendering (fragment-light), run post-processing compute shaders in parallel.
- Monitor CU utilization in profilers. <70% suggests underutilization—add work or investigate bottlenecks (memory, TMU, ROP).

**Platform-Specific:**
- **Xbox Series X**: 52 CUs, 208 TMUs, 112 ROPs. Well-balanced for 4K rendering. Use VRS (Variable Rate Shading) to reduce fragment workload.
- **Xbox Series S**: 20 CUs, 80 TMUs, 64 ROPs. Target 1440p max—4K saturates ROPs. Reduce shader complexity to match lower CU count.
- **PlayStation 5**: 36 CUs, 144 TMUs, 64 ROPs. High clock (2.23 GHz) compensates for fewer CUs. ROP-limited at 4K with heavy blending—use dynamic resolution.
- **Switch**: 2 CUs (256 CUDA cores), 32 TMUs, 16 ROPs. Extremely limited. Target 720p, minimize texture samples (2-3 per shader), avoid blending.

## Common Pitfalls

**TMU Saturation from Excessive Sampling**: Material shader samples 8 textures (albedo, normal, metallic, roughness, AO, height, detail albedo, detail normal). At 1080p = 16.6M samples per texture = 133M samples total per frame. Xbox Series X (208 TMUs at 1.825 GHz) can theoretically do 380M samples/clock, but in practice 133M samples costs ~3ms. With other rendering, TMU bottleneck emerges. Solution: Pack textures aggressively, use fewer samples, or optimize material complexity via shader LOD.

**ROP Bottleneck at 4K**: Rendering at 4K (8.3M pixels) with deferred G-buffer (4 MRTs, 16 bytes/pixel each) writes 531MB per frame. At 60fps = 31.9 GB/sec bandwidth, plus reads for blending. PlayStation 5's 448 GB/sec can handle this, but older consoles (Xbox One: 68 GB/sec) cannot. Symptom: Profiler shows low GPU utilization but poor frame rate. Solution: Reduce resolution, use fewer/smaller MRTs, or implement forward+ rendering.

**Ignoring Command Processor Overhead**: Submitting 5,000 draw calls per frame creates command processor bottleneck—it spends cycles parsing commands instead of executing rendering. Even with fast CUs/TMUs/ROPs, command processor serializes work. Symptom: CPU-bound but GPU shows availability. Solution: Reduce draw calls via batching, instancing, or SRP Batcher.

## Tools & Workflow

**NVIDIA Nsight Graphics**: "Range Profiler" shows per-unit utilization (SM, TMU, ROP, Rasterizer). Identify saturated units (>90% utilization). "Memory" tab shows bandwidth usage—high bandwidth with low SM utilization suggests ROP/TMU bottleneck.

**AMD RGP**: "Summary" page displays component utilization overview. "Event Timing" shows per-draw component usage. "Wavefront Occupancy" focuses on CU utilization. Filter by component to isolate bottlenecks.

**PIX**: "GPU Timeline" shows command processor activity and unit scheduling. "Timing Data" breaks down draw calls by component (vertex processing, rasterization, fragment shading, ROP). Essential for Xbox optimization.

**PlayStation Razor**: Hardware counters expose TMU cache hit rates, ROP throughput, and CU stalls. "Pipeline" view shows component balance across frame. Use to verify workload distributes evenly.

**RenderDoc**: "Pipeline State" shows active GPU stages per draw. "Statistics" reveal primitive counts (rasterization load) and sample counts (TMU load). Use to estimate component pressure before profiling.

## Related Topics

- [2.1 GPU Memory Hierarchy](02-01-GPU-Memory-Hierarchy.md) - Memory systems feeding GPU components
- [2.2 Execution Models](02-02-Execution-Models.md) - How compute units execute shader code
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Identifying which component limits performance
- [1.4 Fragment Processing](01-04-Fragment-Processing.md) - How fragments flow through components

---

[← Previous: 2.2 Execution Models](02-02-Execution-Models.md) | [Next: 2.4 Console-Specific Architecture →](02-04-Console-Specific-Architecture.md)
