# 26.3 Performance Debugging & Bottleneck Identification

[← Back to Chapter 26](../chapters/26-Debugging-Techniques.md) | [Main Index](../README.md)

Performance debugging = identify bottleneck (CPU, GPU, or memory = one limits frame time). Methodology: (1) measure current performance (baseline = 30 FPS = 33ms), (2) profile (Profiler shows where time spent = CPU or GPU), (3) identify bottleneck (if CPU >80% = CPU-bound, if GPU >80% = GPU-bound), (4) optimize bottleneck (if CPU-bound = reduce draw calls, if GPU-bound = simpler shaders), (5) repeat until target met. Trap: optimizing non-bottleneck (optimize CPU when GPU-bound = wasted effort).

---

## Overview

Bottleneck Framework: Frame = CPU work + GPU work (parallel = max(CPU, GPU) = frame time). If CPU = 20ms, GPU = 10ms = CPU-bound (frame = 20ms, GPU idle last 10ms). Optimization: reduce CPU to 15ms = frame improves to 15ms (GPU still idle, not improvement). Instead: reduce GPU to 7ms = frame = max(20, 7) = 20ms still (no improvement = CPU still bottleneck).

Profiling Tools: Unity Profiler (built-in = overview), RenderDoc/PIX (GPU detail), CPU Profiler (detailed call stacks), Memory Profiler (allocations tracked).

## Key Concepts

- **Frame Budget Breakdown**: 30 FPS = 33.33ms per frame. Allocation: Physics ~2ms, Animation ~2ms, Rendering ~15ms, Scripting ~12ms, other ~2.33ms. If rendering = 20ms (over budget), GPU-bound. Profiler shows actual breakdown = identify hogs.
- **CPU Bottleneck Indicators**: high Rendering.ProcessRenderRequests (draw call overhead), Physics (many bodies), Update (heavy logic). Solution: reduce draw calls (batching, LOD culling), simplify physics, optimize scripts (caching, reduce allocations).
- **GPU Bottleneck Indicators**: GPU Utilization >80% (Profiler GPU time high), shaders slow (measure shader time per call in RenderDoc), pixel overdraw (check overdraw visualization). Solution: simpler shaders, reduce resolution, LOD geometry, fewer particles.
- **Memory Bottleneck**: frame time spike every N frames (garbage collection = GC stall = CPU pause). Profiler shows GC spikes (Memory tab = GC Alloc = heap pressure). Solution: reduce allocations per frame (object pooling, static allocations, avoid LINQ/foreach = allocates temporarily).
- **Profiling Methodology**: baseline first (measure current FPS, frame time = write down), isolate variable (disable one system, measure delta = cost), iterate (each optimization = new baseline). Without iteration = flying blind (optimize CPU, if GPU-bound = no gain = waste).

## Best Practices

**Profiling Workflow**:
1. Measure baseline (current FPS, frame time = write down).
2. Identify bottleneck: Profiler > CPU/GPU tabs = see where time spent.
3. Hypothesis: "if I optimize X, frame time improves by Y".
4. Optimize: code change, profile again.
5. Verify: if result matches hypothesis = good optimization, else investigate.
6. Repeat until target met.

**CPU Bottleneck**:
- Check Profiler CPU tab (show Rendering category = draw call overhead visible).
- If Draw Calls > 500 = likely bottleneck (one call = ~10µs CPU = 500 * 10µs = 5ms overhead).
- Solution: batching (combine meshes = fewer calls), LOD culling (distant = not rendered).

**GPU Bottleneck**:
- Check Profiler GPU time (if CPU time < 15ms, GPU time > 20ms = GPU-bound).
- RenderDoc capture = inspect shaders (find expensive one = slow kernel).
- Solution: profile shader (count operations), simplify logic, reduce resolution (0.8x = 36% speedup typical).

**Memory Bottleneck**:
- Profiler Memory tab = GC Alloc per frame (if spiky = GC pressure). Count = bytes per frame should be <1MB (goal = 0 ideally).
- Solution: object pooling (reuse objects = no new allocations), avoid LINQ (allocates enumerator), avoid foreach with structs.

**Platform-Specific**:
- **PC**: measure at target resolution (1440p optimization = different than 1080p profile).
- **Console**: frame rate locked (30/60 FPS choice = measure at fixed rate = consistent).
- **Mobile**: profile on target device (device variance high = phone A might be CPU-bound, phone B GPU-bound = optimize for median).

## Common Pitfalls

**Wrong Bottleneck Identified**: profiler shows high CPU rendering time, developer optimizes CPU drawing. Actually GPU-bound = optimized CPU = no improvement. Symptom: optimization attempt = FPS same. Solution: profile thoroughly (check GPU time too = both CPU and GPU times tell truth).

**Profiler Inaccuracy**: Profiler overhead = 5-10% performance cost (profiling = measuring = overhead). Frame measured = 16ms in profiler, but actual gameplay = 15.2ms. Symptom: profiler shows bottleneck, but disabling doesn't improve gameplay. Solution: profile, then disable profiler and re-test = get true numbers.

**GC Stall Ignored**: frame time average = 20ms, but spikes to 50ms every 5 seconds (GC). Average profiling = hides spikes. Symptom: gameplay feels jerky (spikes visible), average metric good. Solution: check frame time histogram (not just average), or monitor GC Alloc.

## Tools & Workflow

**Profiler Bottleneck Check**:
1. Window > Analysis > Profiler
2. Record game session (play 10 seconds)
3. CPU tab: identify high time functions (Rendering, Physics, etc.)
4. GPU tab: check GPU time vs CPU time (if GPU >> CPU = GPU-bound)
5. Memory tab: check GC Alloc spikes

**Frame Time Measurement** (custom):
```csharp
private float[] frameTimes = new float[60];
private int frameIndex = 0;

void Update() {
    frameTimes[frameIndex] = Time.deltaTime * 1000f; // ms
    frameIndex = (frameIndex + 1) % 60;
    
    if (frameIndex == 0) {
        float avgTime = frameTimes.Average();
        float maxTime = frameTimes.Max();
        Debug.Log($"Avg: {avgTime:F2}ms, Max: {maxTime:F2}ms");
    }
}
```

## Related Topics

- [26.1 Visual Debugging](26-01-Visual-Debugging.md) - Frame Debugger
- [26.2 GPU Debugging](26-02-GPU-Debugging.md) - GPU profilers
- [26.4 Unity Debug Tools](26-04-Unity-Debug-Tools.md) - Profiler
- [04.1 PC Profiling Tools](04-01-PC-Profiling-Tools.md) - Profiling overview

---

[Previous: 26.2 GPU Debugging](26-02-GPU-Debugging.md) | [Next: 26.4 Unity Debug Tools →](26-04-Unity-Debug-Tools.md)
