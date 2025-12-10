# 1.6 Compute Pipeline

[← Back to Chapter 1](../chapters/01-Graphics-Pipeline-Fundamentals.md) | [Main Index](../README.md)

Compute shaders execute general-purpose parallel workloads on the GPU, bypassing the graphics pipeline. They enable GPU-accelerated physics, post-processing, culling, and procedural generation at massive scale.

---

## Overview

Compute shaders run arbitrary code on GPU compute units, organized into thread groups. Unlike graphics shaders (vertex, fragment) that process predefined data (vertices, pixels), compute shaders operate on generic buffers or textures with explicit thread IDs. This flexibility enables GPU-accelerated algorithms: particle simulation (100K particles at 60fps), compute-based culling (test 50K objects in 0.5ms), image processing (4K blur in 2ms), or procedural content generation.

Modern GPUs contain thousands of shader cores executing compute threads in parallel. NVIDIA RTX 4090 has 16,384 CUDA cores; Xbox Series X has 3,328 stream processors; PlayStation 5 has 2,304. Properly utilizing this parallelism requires understanding thread group sizing, memory access patterns, and synchronization primitives. Poor compute shader design (uncoalesced memory access, excessive barriers) can run slower than CPU, wasting GPU potential.

Async compute allows overlapping compute work with graphics rendering. While the GPU renders geometry (graphics queue), it simultaneously processes post-processing or simulation (compute queue). PlayStation 5 and Xbox Series X support async compute natively, enabling 10-20% better GPU utilization by filling idle compute units during fragment-light rendering passes. Mastering async compute is essential for extracting maximum performance from modern hardware.

## Key Concepts

- **Compute Shader**: Programmable stage executing arbitrary code on GPU. Operates on buffers/textures, not vertices/pixels. Organized into thread groups dispatched by CPU commands.
- **Thread Group**: Collection of threads (typically 64-1024) executing together with shared memory access. Threads within group can synchronize via barriers and communicate via shared memory. Groups execute independently.
- **Dispatch**: CPU command launching compute shader with specified thread group count (X, Y, Z dimensions). Example: Dispatch(16, 16, 1) launches 256 thread groups.
- **Shared Memory (LDS/Groupshared)**: Fast scratchpad memory (32-64KB) shared by all threads in a group. 10-100x faster than global memory. Used for inter-thread communication, caching, or reduction operations.
- **Async Compute**: Executing compute shaders on separate GPU queue parallel to graphics rendering. Requires explicit synchronization (semaphores) between queues to ensure data dependencies.

## Best Practices

**Thread Group Sizing:**
- Use thread group sizes that are multiples of warp/wavefront size: 64 threads (AMD), 32 threads (NVIDIA), or 256/512 for maximum occupancy. Common: `[numthreads(8,8,1)]` = 64, `[numthreads(16,16,1)]` = 256.
- Match thread group size to workload. Image processing: 8x8 or 16x16 tiles. 1D data (particle arrays): 64x1x1 or 256x1x1. 3D data (voxels): 4x4x4 or 8x8x8.
- Avoid tiny thread groups (<32). Wastes GPU occupancy—many compute units idle. Avoid huge thread groups (>1024). Limits occupancy due to register/shared memory per-group limits.
- Profile occupancy with vendor tools (Nsight, RGP). Target 75%+ occupancy for compute-bound work, 50%+ for memory-bound work.

**Memory Access Patterns:**
- Coalesce memory reads/writes—adjacent threads should access adjacent memory. `buffer[threadID]` is perfect; `buffer[threadID * 17]` causes cache thrashing and is 10x slower.
- Use structured buffers (`RWStructuredBuffer<T>`) instead of raw byte buffers. Compiler optimizes access patterns and alignment automatically.
- Minimize random access. Reading textures or buffers with computed indices (hash-based, noise) destroys cache. Precompute lookups when possible.
- Use shared memory for repeated access. Load data from global memory to shared memory once per group, then read from shared memory multiple times. Saves global memory bandwidth.

**Synchronization and Barriers:**
- Use `GroupMemoryBarrierWithGroupSync()` only when necessary—threads must wait for all group threads, stalling execution. Required when threads write to shared memory that other threads will read.
- Avoid synchronization across thread groups—impossible within single dispatch. Split algorithm into multiple dispatches if inter-group communication is needed.
- Minimize atomic operations (`InterlockedAdd`, etc.) on global memory. They serialize access and destroy parallelism. Use shared memory atomics (100x faster) and reduce to global memory once per group.

**Async Compute:**
- Schedule async compute during fragment-light rendering (shadows, early geometry passes) to fill idle compute units.
- Use semaphores/fences to synchronize queues. Ensure compute writes complete before graphics reads (e.g., compute culling results must finish before indirect draw).
- Profile GPU timeline (PIX, Nsight) to verify overlap. Poorly scheduled async compute may run serially, adding latency instead of parallelism.
- Limit async compute to ~30% of GPU time to avoid starving graphics queue or causing memory contention.

**Platform-Specific:**
- **PC DirectX 12**: Use `ID3D12CommandQueue` with `D3D12_COMMAND_LIST_TYPE_COMPUTE` for async compute. Insert fences to synchronize with graphics queue.
- **Xbox Series X/S**: Hardware supports 2 async compute queues. Use for post-processing during geometry rendering or pre-frame culling during previous frame present.
- **PlayStation 5**: GNM exposes low-level compute queue control. GNMX provides higher-level API. Schedule async compute carefully to avoid memory bandwidth saturation.
- **Switch**: Limited compute performance (256 CUDA cores, 768 MHz). Use compute shaders sparingly—prefer CPU Jobs or simplified algorithms. Good for small workloads (particle updates <10K).

## Common Pitfalls

**Uncoalesced Memory Access**: Compute shader reading buffer with stride of 33 elements (`buffer[threadID * 33]`) causes each thread to access different cache lines. Memory bandwidth utilization drops to 10-20%. Symptom: Compute shader runs 5-10x slower than expected despite low ALU cost. Solution: Restructure data layout for sequential access (`buffer[threadID]`) or use shared memory caching.

**Excessive Thread Group Synchronization**: Calling `GroupMemoryBarrierWithGroupSync()` 10 times in shader because of complex inter-thread dependencies serializes execution. Each barrier stalls all threads, waiting for slowest. Symptom: Low GPU occupancy, high idle time in profiler. Solution: Redesign algorithm to minimize barriers (2-3 max), or split into multiple dispatches.

**Ignoring Occupancy Limits**: Compute shader uses 64KB shared memory per group, but GPU only has 64KB total per compute unit. Only 1 group can run per unit—occupancy is 12.5% instead of 100%. Symptom: Compute shader runs 5x slower than expected despite correct algorithm. Solution: Reduce shared memory usage (8-16KB per group), use more thread groups with less memory each.

## Tools & Workflow

**PIX Compute Analysis**: "Compute Shader" view shows thread group dimensions, occupancy, and execution time. "Memory" tab reveals bandwidth usage and cache hit rates. Use "GPU Timeline" to verify async compute overlap with graphics work.

**RenderDoc Compute Debugging**: Set dispatch as current event, use "Mesh Output" to visualize output buffers as 3D data. "Pipeline State" shows bound resources (buffers, textures). "Shader Debugger" steps through compute shader code for specific thread.

**NVIDIA Nsight Compute**: Deep dive into compute shader performance. "Occupancy" shows active wavefronts vs theoretical max. "Memory Workload Analysis" reveals uncoalesced access patterns. "Scheduler Statistics" shows warp stalls and issue efficiency.

**AMD RGP**: "Wavefront Occupancy" timeline shows compute shader parallelism over time. "Event Timing" sorts dispatches by cost. "Barrier Analysis" reveals synchronization overhead—identify excessive barriers or serialization.

**Unity Profiler**: "GPU Profiler" shows compute dispatches as separate entries. Millisecond timing lets you identify expensive dispatches. Use `CommandBuffer.BeginSample` to label dispatches for easier identification.

## Related Topics

- [1.1 Rendering Pipeline Overview](01-01-Rendering-Pipeline-Overview.md) - Command buffer and async compute architecture
- [23.1 Unity Jobs System](23-01-Unity-Jobs-System.md) - CPU parallelism alternative to GPU compute
- [24.2 Async Compute](24-02-Async-Compute.md) - Advanced async compute techniques
- [25.1 GPU-Driven Rendering](25-01-GPU-Driven-Rendering.md) - Compute-based culling and indirect rendering

---

[← Previous: 1.5 Output Merger](01-05-Output-Merger.md) | [Next: Chapter 2 - GPU Hardware Architecture →](../chapters/02-GPU-Hardware-Architecture.md)
