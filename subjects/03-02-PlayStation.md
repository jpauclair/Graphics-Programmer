# 3.2 PlayStation

[← Back to Chapter 3](../chapters/03-Platform-Specific-Graphics-APIs.md) | [Main Index](../README.md)

PlayStation platforms use Sony's proprietary GNM/GNMX APIs and PSSL shader language, providing low-level GPU control and platform-specific optimizations unavailable on PC.

---

## Overview

GNM is PlayStation's low-level graphics API, similar to Vulkan or DirectX 12 in design philosophy. It exposes direct control over GPU command buffers, memory management, and synchronization. GNMX is a higher-level helper library built on GNM, providing convenience functions while maintaining performance. Most developers use GNMX for productivity while dropping to GNM for performance-critical sections.

PSSL (PlayStation Shader Language) is an HLSL-like language compiled to GCN (PS4) or RDNA 2 (PS5) GPU instructions. It supports platform-specific extensions like wavefront intrinsics, explicit cache control, and compute queue management. PS5's PSSL compiler generates highly optimized code leveraging RDNA 2 features: ray tracing, mesh shaders (via primitive shaders), and variable rate shading.

PS5's architecture (36 CUs at 2.23 GHz, 448 GB/s bandwidth, 16GB unified memory) differs from Xbox Series X (52 CUs at 1.825 GHz, 560 GB/s peak). PS5 prioritizes clock speed over CU count, making it more efficient for shader-heavy workloads but less parallel for compute. Understanding these tradeoffs guides optimization—complex shaders perform relatively better on PS5, massive parallel compute favors Xbox.

## Key Concepts

- **GNM**: Low-level graphics API providing direct GPU control. Command buffer construction, explicit memory management, and manual synchronization. Analogous to Vulkan's thin driver model.
- **GNMX**: Higher-level library wrapping GNM with helper functions. Provides resource management, context abstraction, and common rendering patterns. Productivity-focused while maintaining performance.
- **PSSL (PlayStation Shader Language)**: HLSL-like shader language compiling to GCN/RDNA ISA. Supports extensions like wavefront operations, explicit data movement, and compute queue control.
- **Geometry Engine**: PS5 hardware performing primitive culling and tessellation before rasterization. Culls back-facing/off-screen primitives automatically, saving 10-30% rendering cost.
- **Async Compute**: Explicit compute queue separate from graphics queue. Schedule particle simulation or post-processing during shadow/geometry rendering for 15-25% GPU utilization improvement.

## Best Practices

**GNM/GNMX Programming:**
- Use GNMX for most rendering work. Its command buffer wrappers and state management reduce boilerplate vs raw GNM while maintaining near-identical performance.
- Drop to GNM for high-frequency calls (updating descriptors in tight loops, submitting thousands of draws). GNMX overhead (function call + validation) adds microseconds per call.
- Leverage persistent command buffers. Record static scene command buffers once, resubmit each frame. Eliminates per-frame recording cost (1-3ms for complex scenes).
- Implement multi-threaded command buffer recording. GNMX supports parallel recording via separate contexts. Scale to 6-7 CPU cores (reserve 1-2 for game logic and OS).
- Profile with Razor to validate optimizations. "Command Buffer" view shows submission times; "Wavefront Occupancy" reveals shader efficiency.

**PSSL Shader Optimization:**
- Use wavefront intrinsics for efficient parallelism: `S_VOTE`, `S_REDUCE`, `DS_SWIZZLE`. Enables inter-thread communication within 64-thread wavefront without LDS (shared memory).
- Leverage 16-bit scalar operations (`s16`, `u16`) for registers and ALU. Halves register pressure and doubles throughput vs 32-bit ops on RDNA 2.
- Explicit cache control via `__builtin_amdgcn` intrinsics. Bypass L1 for streaming data or enforce L2 caching for frequent accesses. Improves bandwidth efficiency by 10-20%.
- Use PSSL's data sharing extensions for compute. Explicit LDS management and wavefront-level atomics outperform generic shared memory patterns.

**PS5 Advanced Features:**
- Enable Geometry Engine primitive culling via `PrimitiveSetup` flags. Automatic back-face and frustum culling before rasterization. Minimal CPU cost, 10-30% fragment shader savings.
- Use primitive shaders (PS5's mesh shader equivalent) for LOD or procedural geometry. Directly emit primitives from compute-like shader, bypassing vertex processing.
- Leverage async compute aggressively. PS5's RDNA 2 async compute is highly efficient. Schedule compute during fragment-light rendering (shadows, depth prepass) to fill idle CUs.
- Exploit fast VRAM (448 GB/s unified). PS5 has no slow memory pool like Xbox—all memory is equally fast. Simplifies allocation vs Xbox Series X's tiered pools.

**PS4 Optimization (GCN Architecture):**
- Async compute is critical on GCN. PS4 has 18 CUs; many workloads leave CUs idle. Overlap compute with graphics to maximize utilization.
- Use ACE (Asynchronous Compute Engine) queues explicitly. GNM provides 8 compute queues; schedule non-graphics work (physics, culling, post-processing) across queues.
- Minimize VGPR (vector register) usage. PS4 GCN has 256 VGPRs per wavefront; exceeding limits halves occupancy. Target <128 VGPRs for complex shaders.
- Leverage PS4 Pro's additional CUs (36 vs 18). Scale compute workloads linearly; graphics workloads benefit from checkerboard rendering or higher resolution.

**Platform-Specific:**
- **PS5**: 16GB unified GDDR6, 36 CUs at 2.23 GHz, RDNA 2 features (ray tracing, primitive shaders, VRS). Target 4K/60fps or 1440p/120fps. Use Geometry Engine and async compute extensively.
- **PS4 Pro**: 8GB GDDR5, 36 CUs at 911 MHz, GCN architecture. Target 1440p-4K checkerboard. Use ID buffer for reconstruction; async compute for parallelism.
- **PS4 Base**: 8GB GDDR5 (1GB reserved for OS), 18 CUs at 800 MHz. Target 1080p/30fps or 900p/60fps. Aggressive optimization required—texture compression, LOD, simplified shaders.

## Common Pitfalls

**Synchronization Errors in Async Compute**: Launching compute shader writing to buffer, then immediately rendering reading same buffer without semaphore causes race condition. Compute may not finish before graphics reads, causing flicker or corruption. GNM requires explicit semaphores between queues. Solution: Insert `submitAndFlip` barriers or use semaphores to ensure compute completes before graphics reads.

**Excessive VGPR Usage on PS4**: PSSL shader uses 160 VGPRs per thread. GCN allows max 256, so occupancy drops to 1 wavefront per SIMD (should be 2-4). Performance halves due to inability to hide memory latency. Symptom: Low wavefront occupancy in Razor (<40%). Solution: Simplify shader, split into passes, or use lower precision (f16 halves VGPR usage).

**Ignoring Geometry Engine on PS5**: Rendering with Geometry Engine disabled (default in some middleware) forces CPU culling and wastes hardware culling capability. Symptom: High CPU culling cost (1-2ms) despite PS5 hardware support. Solution: Enable Geometry Engine via GNMX primitive setup flags, offload culling to GPU.

## Tools & Workflow

**Razor**: PlayStation's primary profiler. "GPU Timeline" shows graphics and compute queue work. "Wavefront Occupancy" reveals shader efficiency (target >70%). "Command Buffer" view displays submission times and sync points. Essential for PS5/PS4 optimization.

**Tuner**: CPU/GPU profiler showing both timelines. Identifies CPU-GPU sync issues and async compute opportunities. "Memory" tab displays VRAM usage and bandwidth. Use to optimize resource allocation.

**Orbis Shader Compiler**: Offline PSSL compiler producing GCN/RDNA bytecode. Unity invokes automatically, but standalone compilation debugs complex shaders. Outputs register usage, occupancy estimates, and optimization warnings.

**GNM Validation Layer**: Debug layer catching API misuse (invalid states, resource hazards, synchronization errors). Enable during development, disable for release builds. Catches issues invisible without hardware validation.

**PlayStation SDK Profiler**: Low-level profiler exposing hardware counters (CU utilization, cache hit rates, memory bandwidth). More detailed than Razor but requires deeper graphics knowledge. Use for final optimization pass.

## Related Topics

- [2.4 Console-Specific Architecture](02-04-Console-Specific-Architecture.md) - PlayStation GPU architecture details
- [4.2 Console Profiling Tools](04-02-Console-Profiling-Tools.md) - Razor and Tuner profiling
- [24.2 Async Compute](24-02-Async-Compute.md) - Async compute strategies for PlayStation
- [12.1 Unity Shader Languages](12-01-Unity-Shader-Languages.md) - Shader language comparison

---

[← Previous: 3.1 DirectX (PC/Xbox)](03-01-DirectX.md) | [Next: 3.3 Nintendo Switch →](03-03-Nintendo-Switch.md)
