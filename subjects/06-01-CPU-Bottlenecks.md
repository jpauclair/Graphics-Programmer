# CPU Bottlenecks

[← Back to Chapter 6](../chapters/06-Bottleneck-Identification.md) | [Main Index](../README.md)

Identifying CPU bottlenecks is essential for balanced performance. When the CPU limits frame rate, GPU optimizations provide no benefit—you must optimize CPU workloads instead.

---

## Overview

CPU bottlenecks occur when the CPU takes longer than the GPU to complete frame work, leaving the GPU idle waiting for rendering commands. This manifests as low GPU utilization (30-60%) despite poor frame rates. Unity's rendering architecture uses two primary threads: the main thread (gameplay, scripts, culling) and the render thread (API calls, command buffers). If either thread exceeds your frame budget, you're CPU-bound.

Console CPUs are weaker than high-end PC CPUs, making CPU optimization critical. Xbox Series S runs at 3.6GHz (3.4GHz SMT mode), PlayStation 5 at 3.5GHz, and Switch at 1.0GHz. Complex game logic or inefficient rendering submission can easily consume 16ms per frame on these platforms. PC development must target 4-core CPUs (min-spec) while optimizing for 8+ cores on recommended specs.

Distinguishing between main thread and render thread bottlenecks guides optimization strategy. Main thread issues require gameplay code optimization or Jobs System parallelization. Render thread issues need draw call reduction, batching, or SRP Batcher implementation.

## Key Concepts

- **Main Thread**: Executes game logic, physics, scripts, culling, and rendering decisions. Runs synchronously with gameplay—stalls directly impact frame rate.
- **Render Thread**: Translates Unity's rendering commands into graphics API calls (DirectX, GNM, NVN). Submits draws, sets GPU state, uploads buffers. Runs parallel to main thread but can bottleneck independently.
- **Gfx.WaitForPresent**: CPU waiting for GPU to finish previous frame before submitting new work. Indicates GPU-bound scenario, not CPU bottleneck.
- **Thread Synchronization**: Points where main and render threads wait for each other. Excessive sync causes one thread to idle while the other works.
- **Job System Overhead**: Unity Jobs have scheduling cost. Tiny jobs (<0.1ms) waste time on overhead instead of parallelism gains.

## Key Concepts (Additional)

- **Script Update Cost**: MonoBehaviour Update/LateUpdate execution time. Poorly optimized scripts (FindObjectOfType, GetComponent per frame) dominate main thread.
- **Physics Simulation**: FixedUpdate and physics engine (PhysX/Box2D) cost. Complex scenes with hundreds of rigidbodies can consume 5-10ms.

## Best Practices

**Identifying CPU Bottlenecks:**
- Open Unity Profiler (Window > Analysis > Profiler). If "GPU" time is much lower than total frame time, you're CPU-bound.
- Check "Rendering" module for "Gfx.WaitForPresent". High values (>3ms) indicate GPU-bound. Low values (<1ms) confirm CPU bottleneck.
- Use PIX or platform profilers to view CPU/GPU timelines. Gaps where GPU is idle reveal CPU bottlenecks. Continuous GPU work with idle CPU suggests GPU bottleneck.
- Profile both main and render threads. Sort Profiler by "Self Time" to find heaviest functions.

**Main Thread Optimization:**
- Move expensive logic to Jobs System. Pathfinding, physics queries, and AI decisions parallelize well with `IJobParallelFor`.
- Reduce `Update()` calls by caching references (save `GetComponent` results). Use object pooling to avoid `Instantiate`/`Destroy` overhead (0.1-1ms per call).
- Optimize physics by reducing FixedUpdate rate (60Hz to 30Hz for non-critical objects), using simplified colliders (boxes/spheres instead of mesh colliders), and disabling physics for off-screen or distant objects.
- Batch script updates using custom manager patterns. Instead of 1,000 enemies each calling Update, one manager updates an array in a Burst job.

**Render Thread Optimization:**
- Reduce draw calls via SRP Batcher, GPU instancing, or batching (see [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md)).
- Minimize material SetPass calls by sharing materials. Each unique material forces a GPU state change (0.05-0.2ms on console).
- Use CommandBuffers for complex multi-pass rendering instead of multiple cameras. Cameras have high overhead (culling, sorting per camera).
- Enable "Graphics Jobs" in Player Settings (PC/Console). Parallelizes render thread work across multiple cores, reducing single-thread bottleneck by 30-50%.

**Platform-Specific:**
- **Xbox Series X/S**: Enable SMT (Simultaneous Multithreading) for 16 logical cores. Schedule Jobs on worker threads, leaving main/render threads for critical work.
- **PlayStation 5**: Use 6-7 cores for game threads (OS reserves 2 cores). Profile with Razor to identify core utilization imbalance.
- **Switch**: Extremely CPU-limited (4 ARM cores, 1GHz). Minimize script updates, simplify physics, and use aggressive culling. Target <10ms main thread.
- **PC**: Use Unity's Job System WorkerThreadCount API to scale parallelism. Detect core count at startup and adjust job batch sizes accordingly.

## Common Pitfalls

**Assuming GPU Bottleneck**: Profiling shows 50fps with beautiful graphics, so you optimize shaders—but frame rate doesn't improve. Always check Profiler first. Many "GPU-heavy" scenes are actually CPU-bound due to thousands of draw calls or expensive scripts. GPU optimization is wasted effort if CPU is the bottleneck.

**Ignoring Render Thread**: Main thread runs at 10ms, GPU at 8ms, but frame time is 16ms. The hidden culprit: render thread at 16ms submitting 3,000 draw calls. Enable "Other Threads" in Profiler to see render thread cost. Without this visibility, you'll optimize the wrong thread.

**Physics Overdoing**: Setting 200 objects to continuous collision detection (CCD) for "better quality" adds 5-8ms to FixedUpdate. Use CCD only for fast-moving critical objects (player, bullets). Static and slow-moving objects use discrete collision. Profile Physics.Simulate in Profiler to catch this.

## Tools & Workflow

**Unity Profiler - CPU Module**: Shows main thread functions sorted by cost. "Self Time" reveals direct cost; "Total Time" includes children. Click markers to see callstacks. Enable "Deep Profiling" for per-function cost, but only in small test scenes (massive overhead).

**Unity Profiler - Rendering Module**: "RenderLoop.Draw" shows render thread cost. "Batches" and "SetPass calls" indicate draw call overhead. "Gfx.WaitForPresent" distinguishes CPU vs GPU bottleneck.

**PIX for Windows/Xbox**: Timeline view shows CPU threads and GPU execution simultaneously. Zoom into frame to see exact main/render thread work. Right-click functions for details. Xbox PIX provides hardware performance counters for CPU cache misses and branch mispredictions.

**Platform Profilers**: PlayStation Razor shows per-core utilization and thread scheduling. Switch NNGFX Debugger highlights CPU wait states. These tools reveal threading inefficiencies invisible in Unity Profiler.

**Frame Comparison**: Profile identical scene at different resolutions (720p vs 4K). If frame time doesn't change, you're CPU-bound (resolution doesn't affect CPU). If 4K is slower, you're GPU-bound (more pixels to shade).

## Related Topics

- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - Techniques to reduce CPU rendering cost
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Identifying GPU-bound scenarios
- [23.1 Unity Jobs System](23-01-Unity-Jobs-System.md) - Parallelizing CPU work
- [24.1 Multithreading Rendering](24-01-Multithreading-Rendering.md) - Render thread optimization

---

[← Previous: Chapter 6 Index](../chapters/06-Bottleneck-Identification.md) | [Next: 6.2 GPU Bottlenecks →](06-02-GPU-Bottlenecks.md)
