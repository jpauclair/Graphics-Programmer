# 2.1 GPU Memory Hierarchy

[← Back to Chapter 2](../chapters/02-GPU-Hardware-Architecture.md) | [Main Index](../README.md)

GPU memory is organized in a hierarchy from fast local registers to slower global VRAM. Understanding this hierarchy is critical for optimizing memory access patterns and achieving peak performance.

---

## Overview

GPU memory systems are designed for massive parallelism, with thousands of threads accessing memory simultaneously. Unlike CPU cache (invisible to programmers), GPU memory hierarchy requires explicit management—choosing the right memory type for data and optimizing access patterns directly impacts performance by 10-100x. The hierarchy spans from registers (1 cycle latency, 0.5-1MB total) to VRAM (400-800 cycles latency, 10-24GB total on consoles).

Modern GPUs hide memory latency through occupancy—running many thread groups simultaneously so that when one group waits for memory, others execute. High occupancy (75%+) requires limiting register usage, shared memory allocation, and keeping threads from diverging. Console profilers reveal occupancy bottlenecks invisible to PC tools, making memory optimization critical for hitting 60fps targets on fixed hardware.

Platform differences are significant: PC GPUs have dedicated VRAM (8-24GB GDDR6) separate from system RAM, while Xbox Series uses unified memory (16GB shared between CPU/GPU), and Switch has tiny 4GB shared memory with limited bandwidth (25.6 GB/s vs PS5's 448 GB/s). Memory optimization strategies must account for these architectural differences.

## Key Concepts

- **VRAM (Global Memory)**: Main GPU memory holding textures, buffers, render targets. Largest capacity (10-24GB) but slowest access (400+ cycles). Bandwidth is finite (336-448 GB/s on consoles)—exceeding it causes stalls.
- **L2 Cache**: Shared cache between all GPU compute units (4-6MB on consoles). Caches VRAM accesses. 80-90% hit rate is good; <60% indicates poor access patterns. Transparent to shader code but critical for performance.
- **L1 Cache / Texture Cache**: Per-compute-unit cache (16-128KB). Texture cache optimized for 2D/3D spatial locality. Mipmapping improves texture cache hits by 5-10x. L1 caches buffer/compute data.
- **Shared Memory (LDS/Groupshared)**: Fast scratchpad memory (32-64KB per compute unit) shared by thread group. Explicitly managed in compute shaders. 10-50x faster than VRAM but limited capacity.
- **Register File**: Ultra-fast per-thread storage (256 registers per thread max). Stores shader variables. Register spilling to memory kills performance—reduce complex calculations or split shaders.
- **Constant Buffers (CBs)**: Read-only uniform data broadcast to all threads. Cached aggressively. PC has 64KB CB limit; use structured buffers for larger data. Updated per-draw for transforms, materials.

## Best Practices

**Memory Access Patterns:**
- Coalesce memory reads—adjacent threads should access adjacent addresses. `texture[threadID]` is perfect; `texture[hash(threadID)]` is 10x slower due to cache thrashing.
- Use texture samplers with mipmapping enabled. Accessing textures without mips causes cache misses (sampling high-res textures for distant objects). Mips improve cache hits by 80-90%.
- Minimize random access. Compute shaders with hash tables or noise lookups destroy cache. Precompute to textures or use coherent access patterns.
- Align buffer elements to 16 bytes (float4). GPU memory controllers optimize for 128-bit reads. Misaligned reads cost extra cycles.

**Register Pressure Management:**
- Target <32 registers per thread for high occupancy (2+ wavefronts per SIMD). Check compiled shader stats in RenderDoc or platform profilers.
- Reduce local variables in shaders. Each variable costs registers. Reuse variables or compute on-the-fly instead of caching intermediate results.
- Use half-precision (`min16float`) for non-critical calculations. Halves register pressure and doubles ALU throughput on consoles.
- Split complex shaders into multiple passes if register spilling occurs. Single 500-instruction shader with spilling is slower than two 250-instruction shaders.

**Shared Memory Usage:**
- Use shared memory for repeated VRAM accesses within thread group. Load from VRAM once, cache in shared memory, read multiple times locally (10x faster).
- Size shared memory carefully: 16-32KB per group is safe; 64KB limits occupancy (only 1 group per compute unit). Profile occupancy to find sweet spot.
- Bank conflicts occur when multiple threads access the same memory bank. Pad arrays to avoid conflicts: `sharedData[threadID * 33]` instead of `[threadID * 32]`.
- Synchronize with `GroupMemoryBarrierWithGroupSync()` after writes before reads. Forgetting causes race conditions and corruption.

**Platform-Specific:**
- **PC NVIDIA**: 10MB L2 cache on RTX 4090. Use wave intrinsics for inter-thread communication. Texture cache favors anisotropic filtering.
- **PC AMD**: Infinity cache (96-128MB on RDNA 3) acts as huge L3. Benefits from large working sets. Use explicit cache control via Vulkan extensions.
- **Xbox Series X/S**: 10GB fast VRAM + 6GB slower (Series X). Place render targets and frequent textures in fast pool. Series S has 8GB total (use aggressive compression).
- **PlayStation 5**: Unified 16GB GDDR6 at 448 GB/s. Fast VRAM but shared with CPU. Target <10GB GPU usage to leave headroom for system and game code.
- **Switch**: 4GB shared LPDDR4 at 25.6 GB/s. Extremely bandwidth-limited. Compress everything, minimize overdraw, use 16-bit render targets.

## Common Pitfalls

**Ignoring Texture Cache**: Sampling textures with computed UVs (Perlin noise `texture.Sample(sampler, sin(pos.xy) * 10)`) causes random memory access. Each fragment samples distant texels, trashing cache. Frame rate drops 50% vs coherent sampling. Solution: Precompute noise to texture, sample with screen UVs for cache-friendly access.

**Excessive Constant Buffer Updates**: Updating 64KB constant buffer every draw call (1,000 draws) uploads 64MB/frame at 60fps = 3.8GB/sec bandwidth. On Switch (25.6 GB/s total), this is 15% of all bandwidth. Solution: Use SRP Batcher or instancing—upload CB once, render many objects.

**Register Spilling**: Pixel shader using 80 registers (limit is 64 on PS4) causes spillage to VRAM. Shader runs 5-10x slower as every spilled variable requires memory load/store. Symptom: High shader instruction count in profiler but low occupancy. Solution: Simplify shader logic, use fewer temporary variables, or split into multiple passes.

## Tools & Workflow

**RenderDoc**: Resource Inspector shows memory allocations (VRAM usage per texture/buffer). "Pipeline State" displays constant buffer contents. Use to verify CB updates and identify large allocations.

**PIX**: Memory view shows VRAM usage breakdown by resource type. "GPU Allocations" timeline reveals memory thrashing or leaks. "Shader Stats" show register usage—target <32 for high occupancy.

**NVIDIA Nsight Graphics**: "Memory" section shows L2 cache hit rates, memory bandwidth utilization, and texture cache efficiency. <80% cache hits indicate optimization opportunities.

**AMD RGP**: "Memory Chart" displays bandwidth consumption per draw. "Wavefront Occupancy" shows register pressure limiting parallelism. "Event Timing" reveals memory-bound shaders (high latency, low ALU).

**Unity Memory Profiler**: Shows texture and mesh memory usage. Identifies large assets consuming VRAM. Use "Compare Snapshots" to track memory leaks or growth over time.

**Platform Profilers**: PlayStation Razor exposes VGPR (vector register) and SGPR (scalar register) usage with occupancy impact. Xbox PIX shows fast vs slow memory pool allocation on Series X/S—optimize pool placement.

## Related Topics

- [2.2 Execution Models](02-02-Execution-Models.md) - How memory latency is hidden via occupancy
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Practical memory optimization techniques
- [5.4 Bandwidth Optimization](05-04-Bandwidth-Optimization.md) - Reducing memory bandwidth usage
- [13.1 Texture Formats and Compression](13-01-Texture-Formats-And-Compression.md) - Compressing textures to save VRAM

---

[← Previous: Chapter 2 Index](../chapters/02-GPU-Hardware-Architecture.md) | [Next: 2.2 Execution Models →](02-02-Execution-Models.md)
