# 2.2 Execution Models

[← Back to Chapter 2](../chapters/02-GPU-Hardware-Architecture.md) | [Main Index](../README.md)

GPUs execute thousands of threads in parallel using SIMD/SIMT architecture. Understanding execution models is essential for writing shaders that maximize hardware utilization and avoid performance pitfalls.

---

## Overview

GPU architecture fundamentally differs from CPUs: instead of 8-16 powerful cores executing independent threads, GPUs contain thousands of simpler cores executing groups of threads in lockstep. NVIDIA calls these groups "warps" (32 threads), AMD uses "wavefronts" (64 threads), and conceptually they execute the same instruction simultaneously across all threads. This SIMT (Single Instruction Multiple Thread) model achieves massive parallelism but requires careful shader design to avoid inefficiencies.

When a shader branch causes some threads to take one path and others to take another (divergence), the hardware serializes execution—first executing one path with some threads masked off, then the other path with remaining threads masked. If 16 threads branch left and 16 branch right, you've effectively halved performance. Modern games minimize branching via shader variants, branchless math (lerp/step), or reorganizing data to reduce divergence.

GPU occupancy measures how well threads fill execution units. Low occupancy (<50%) means the GPU runs fewer threads than hardware supports, leaving compute units idle and failing to hide memory latency. High register usage, excessive shared memory, or large thread groups reduce occupancy. Console profilers show occupancy directly—targeting 75%+ ensures the GPU stays busy while memory accesses complete.

## Key Concepts

- **SIMT (Single Instruction Multiple Thread)**: Execution model where groups of threads (warps/wavefronts) execute the same instruction simultaneously. Differs from SIMD (Single Instruction Multiple Data) by allowing thread-local program counters for flexibility.
- **Warp/Wavefront**: Group of threads executing together. NVIDIA uses 32-thread warps, AMD uses 64-thread wavefronts. All threads in group execute the same instruction each cycle (divergence causes masking).
- **Thread Divergence**: When threads within warp/wavefront take different code paths (branches). Hardware serializes execution—both paths run, each with some threads masked. Halves or quarters effective throughput.
- **GPU Occupancy**: Percentage of maximum threads actively executing on GPU. 100% occupancy = all thread slots filled. Limited by register usage, shared memory, or thread group size. Target 75%+ for memory-bound workloads.
- **Streaming Multiprocessor (SM/Compute Unit)**: GPU building block containing shader cores, registers, and shared memory. NVIDIA RTX 4090 has 128 SMs; Xbox Series X has 52 compute units. Each executes multiple warps/wavefronts concurrently.

## Best Practices

**Minimizing Thread Divergence:**
- Avoid per-pixel branches in fragment shaders. Use shader variants (`#pragma multi_compile`) to create specialized shaders instead of runtime branching.
- Prefer branchless math: `result = lerp(a, b, condition)` instead of `if (condition) result = a; else result = b;`. Same result, no divergence.
- Group similar work together. Sort particles by shader complexity before rendering—simple particles in one batch, complex in another. Reduces divergence within warps.
- Use early-out in loops only if most threads exit early. `for (int i = 0; i < maxLights; i++) { if (i >= numLights) break; }` is worse than uniform `for (int i = 0; i < NUM_LIGHTS; i++)`.

**Maximizing Occupancy:**
- Limit register usage to 32-48 per thread (platform-dependent). Check shader compilation stats. Excess registers reduce occupancy exponentially.
- Use appropriate thread group sizes for compute shaders: 64 or 256 threads (multiples of warp/wavefront size). Avoid tiny groups (<32) or huge groups (>1024).
- Reduce shared memory allocation. 64KB per group limits occupancy to 1 group per compute unit (assuming 64KB LDS total). Use 16-32KB for better occupancy.
- Profile occupancy in vendor tools (Nsight, RGP, Razor). If <50%, investigate register pressure, shared memory, or thread group configuration.

**Efficient SIMT Programming:**
- Design shaders for uniform execution. All threads processing similar data (geometry shaders, particle updates) perform well. Wildly different data causes divergence.
- Use wave/warp intrinsics for efficient inter-thread communication: `WaveActiveSum`, `WaveReadLaneAt` (DirectX), `subgroupAdd` (Vulkan). Operates within warp without shared memory.
- Leverage SIMT for reductions: 32 threads can sum 32 values in 5 steps using wave intrinsics (log₂ complexity). Faster than sequential CPU code.
- Avoid atomics when possible. Atomic operations serialize access, destroying SIMT parallelism. Use wave intrinsics or separate passes instead.

**Platform-Specific:**
- **PC NVIDIA**: 32-thread warps, 128 warps per SM (4,096 threads max per SM on Ada). Use `SM_6_0` wave intrinsics for warp-level operations.
- **PC AMD**: 64-thread wavefronts, wave32 mode available on RDNA 2+ for 32-thread execution (better occupancy for small kernels). Infinity Cache helps hide latency.
- **Xbox Series X/S**: 64-thread wavefronts (RDNA 2 architecture). 52 CUs on Series X, 20 on Series S. Use async compute to fill idle CUs during rendering.
- **PlayStation 5**: 64-thread wavefronts, 36 CUs at 2.23 GHz (very high clock). Fewer CUs than Xbox but faster execution. Occupancy matters more due to fewer parallel threads.
- **Switch**: 32-thread warps, 256 CUDA cores total (Tegra X1). Extremely limited parallelism—optimize for low thread counts and simple shaders.

## Common Pitfalls

**Dynamic Branching on Texture Data**: Fragment shader branches based on texture color (`if (albedo.r > 0.5)`) causes maximum divergence—neighboring pixels sample different colors, taking different paths. Every 2x2 quad diverges. Performance drops 50-75%. Solution: Use texture data for blending (`result = lerp(a, b, albedo.r)`) or restructure algorithm to avoid branches.

**Over-Optimization with Wave Intrinsics**: Using wave intrinsics for 2-3 value reductions adds complexity and limits portability for negligible gain. Wave intrinsics shine for 32-64+ element reductions or broadcast patterns. For small operations, standard code is clearer and sufficient. Profile before optimizing.

**Ignoring Occupancy Warnings**: Shader compiles with warning "occupancy limited by register usage (48 registers)". Ignoring this means shader runs at 50% occupancy—half the threads possible. GPU sits 50% idle while memory fetches complete. Symptom: Low GPU utilization in profiler despite heavy workload. Solution: Simplify shader or split into passes.

## Tools & Workflow

**NVIDIA Nsight Compute**: "Occupancy" section shows achieved vs theoretical occupancy with limiting factors (registers, shared memory, blocks). "Warp State Statistics" reveals divergence—"% warps divergent" should be <10%.

**AMD RGP**: "Wavefront Occupancy" timeline shows occupancy over frame execution. Color-coded: green (high), yellow (medium), red (low). "Event Timing" highlights shaders with poor occupancy for investigation.

**PIX for Xbox**: "GPU Timeline" shows compute unit utilization. Gaps indicate low occupancy. "Shader Stats" display register counts and occupancy impact. Use "Experiments" to test shader modifications live.

**PlayStation Razor**: "Wavefront Occupancy" view shows precise occupancy per shader. "VGPR/SGPR Usage" reveals register pressure. "CU Utilization" indicates whether 36 CUs are fully utilized.

**RenderDoc**: "Shader Messages" window shows compilation warnings including occupancy limiters. "Pipeline State" displays thread group dimensions for compute shaders. Validate sizing is optimal for target hardware.

**Unity Shader Compilation**: Enable "Show generated code" in shader inspector. Look for register counts and warnings. Use `#pragma target 5.0` for advanced features but beware register usage implications.

## Related Topics

- [2.1 GPU Memory Hierarchy](02-01-GPU-Memory-Hierarchy.md) - Memory systems and latency hiding via occupancy
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Optimizing shaders for SIMT execution
- [12.3 Shader Optimization Techniques](12-03-Shader-Optimization-Techniques.md) - Advanced shader optimization
- [1.6 Compute Pipeline](01-06-Compute-Pipeline.md) - Compute shader thread groups and execution

---

[← Previous: 2.1 GPU Memory Hierarchy](02-01-GPU-Memory-Hierarchy.md) | [Next: 2.3 GPU Components →](02-03-GPU-Components.md)
