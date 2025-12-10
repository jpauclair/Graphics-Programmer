# GPU-Side Optimization

[← Back to Chapter 5](../chapters/05-Performance-Optimization.md) | [Main Index](../README.md)

Optimizing GPU performance requires understanding shader execution, memory access patterns, and hardware architecture across PC and console platforms.

---

## Overview

GPU-side optimization focuses on reducing the computational cost of shaders and improving memory access efficiency. Modern GPUs execute thousands of threads in parallel, but inefficient shaders can create bottlenecks that waste this parallelism. Understanding the balance between ALU operations, texture sampling, and memory bandwidth is critical for achieving optimal performance across DirectX 12 (PC/Xbox), GNM/GNMX (PlayStation), and NVN (Switch).

The GPU processes vertices and fragments in waves or warps (groups of 32-64 threads). When threads within a wave diverge due to branches or when memory access patterns are incoherent, performance suffers dramatically. Console platforms provide lower-level control over GPU resources, allowing for more aggressive optimization than PC, but the fundamental principles remain consistent across all platforms.

## Key Concepts

- **Shader Complexity**: Total instruction count including ALU operations, texture samples, and memory accesses. Measured in cycles per pixel/vertex.
- **Overdraw**: Pixels shaded multiple times due to depth sorting or transparency. Can cost 2-10x rendering time on fragment-heavy scenes.
- **Texture Sampling Cost**: Memory fetches are 10-100x slower than ALU operations. Cache misses multiply this cost significantly.
- **Register Pressure**: Shaders using too many registers reduce thread occupancy, limiting parallelism and hiding memory latency poorly.
- **Thread Divergence**: Branches causing different threads in a wave to execute different code paths, serializing execution and wasting GPU cycles.

## Best Practices

**Shader Optimization:**
- Keep pixel shaders under 200 instructions for mobile/Switch, 300-400 for PS4/Xbox One, 500+ acceptable for PS5/Series X.
- Prefer ALU operations over texture samples (4:1 ratio is often optimal). Use texture combiners and packing techniques.
- Move calculations to vertex shaders when possible—vertices are processed far less frequently than fragments.
- Use half-precision (float16) on consoles for color/normal calculations to double ALU throughput.

**Overdraw Reduction:**
- Enable GPU occlusion culling on PC/Xbox (PIX shows overdraw heatmaps). Use conservative depth output when available.
- Sort opaque objects front-to-back to leverage early-Z rejection. PlayStation's depth prepass can eliminate 70-90% of fragment work.
- Minimize transparent objects and sort back-to-front only when necessary. Consider alpha-to-coverage for foliage.

**Memory Access:**
- Sample textures at mipmap levels matching screen resolution. Avoid negative LOD bias which forces high-resolution samples.
- Group texture samples at the start of shaders to hide latency with ALU work. Use texture arrays instead of bindless on consoles.
- Align buffer reads to 128-bit boundaries (float4). Xbox/PlayStation hardware optimizes for vectorized loads.

**Platform-Specific:**
- **PC/Xbox**: Use DirectX 12 wave intrinsics for efficient parallel reductions. Enable variable rate shading for Series X.
- **PlayStation**: Utilize async compute for post-processing while geometry renders. PSSL provides explicit wavefront control.
- **Switch**: Minimize fragment shader complexity aggressively. Use lower precision and reduce texture samples to 2-3 per shader.

## Common Pitfalls

**Dynamic Branching in Pixel Shaders**: Branches are cheap only if all threads in a wave take the same path. Per-pixel branches based on texture data or interpolated values cause massive slowdowns. Use static branches (shader variants) instead, or restructure code to use lerp/step functions that execute both paths.

**Unoptimized Texture Sampling**: Sampling multiple textures with different UV coordinates thrashes texture cache. Pack related textures into channels (R=metallic, G=occlusion, B=detail, A=smoothness) to reduce samples from 4 to 1. On consoles, this can improve frame time by 1-2ms in material-heavy scenes.

**Ignoring Register Limits**: PS4 allows 48 vector registers per shader; using 49 halves occupancy and doubles render time. Use RenderDoc or console profilers to monitor register usage. Split complex shaders into multiple passes if necessary.

## Tools & Workflow

**PIX for Windows/Xbox**: Capture GPU timeline and analyze shader execution. The "Shader Performance" view shows instruction counts, register usage, and occupancy. Use "Experiments" feature to test shader modifications live.

**RenderDoc**: Inspect pixel history to see overdraw and identify expensive shaders. Shader debugger lets you step through HLSL and see register values, revealing inefficient calculations.

**Unity Frame Debugger**: Shows batch-level GPU cost. Enable "GPU Profiler" in Play Mode to see per-draw millisecond timing. Identify heavy draws and optimize those shaders first.

**Platform Profilers**: PlayStation Razor shows wavefront occupancy and VGPR usage. NVIDIA Nsight Graphics reveals warp stalls and memory bottlenecks on PC. Switch NNGFX Debugger highlights fragment shader hotspots.

**Shader Variants**: Use `#pragma multi_compile` for quality levels. Strip unused variants in builds. SRP lets you control variant generation programmatically to reduce build sizes and load times.

## Related Topics

- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - Reducing CPU overhead
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Identifying GPU-bound scenarios
- [12.3 Shader Optimization Techniques](12-04-Advanced-Shader-Techniques.md) - Advanced shader optimization
- [13.1 Texture Formats and Compression](08-01-Texture-Compression-Formats.md) - Optimizing texture memory

---

[← Previous: 5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) | [Next: 5.3 Memory Optimization →](05-03-Memory-Optimization.md)
