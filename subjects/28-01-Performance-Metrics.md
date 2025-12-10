# 28.1 Performance Metrics

[← Back to Chapter 28](../chapters/28-Instrumentation-And-Telemetry.md) | [Main Index](../README.md)

Performance metrics are quantitative measurements of rendering, gameplay, and system performance. Critical metrics include frame time (total, CPU, GPU split), draw calls, triangle count, VRAM usage, battery drain, and network latency. This section covers metric collection architecture, histogram tracking (frame-time distributions, not averages), per-frame profiling vs. aggregate statistics, and platform-specific considerations (PC/console/mobile).

---

## Overview

Performance optimization is data-driven: measure, identify bottlenecks, optimize, measure again. Metrics guide decisions: if frame time 16.67ms at 60 fps, GPU 10ms, CPU 6.67ms, optimize CPU (game logic, physics) before GPU. If GPU 16ms, optimize rendering (draw calls, geometry, shading). Metrics must be accurate and granular: measuring "frame time" isn't enough; need CPU/GPU split, per-draw-call time, memory allocation rates, cache miss rates.

Key principle: focus on percentiles, not averages. Average frame time 16ms sounds good; 99th percentile 33ms (frame drop every 1% of frames) means stutters, unplayable. Distributions matter: histogram of frame times reveals patterns (spikes every 1000 frames = GC, not optimization).

Metrics collection overhead: profiling adds cost (GPU stalls for timer queries, CPU overhead for logging). Production builds: minimize overhead (10-bit frame-time histogram instead of per-frame log).

## Key Concepts

- **Frame Time**: Total time from frame start to display. Target 16.67ms (60 fps). Breakdown: CPU (game logic, rendering API), GPU (graphics pipeline), sync (wait for GPU to finish before CPU submits next frame).

- **CPU Frame Time**: Time spent in CPU code (gameplay, physics, rendering API calls). Measured via system clock (QueryPerformanceCounter on Windows). Granular timing: per-system (game logic, physics, rendering), per-draw-call (API overhead). Typical budget: 8-12ms at 60 fps (rest is GPU).

- **GPU Frame Time**: Time spent in graphics pipeline (VS, PS, compute, memory transfers). Measured via GPU timestamps (query events in command buffer, read results 1-2 frames later). Cost: small GPU overhead (timestamp query ~10 clocks). Typical budget: 12-16ms at 60 fps (leaves 0.67-4.67ms for sync/CPU logic).

- **Draw Call Count**: Number of rendering API calls (DrawIndexed, Dispatch, etc.). Lower is better. 50 draw calls = low overhead ("GPU-bound"), 500 = moderate overhead (5-10ms CPU), 5000 = high overhead (50-100ms CPU, API overhead dominates).

- **Triangle Count**: Visible geometry. 100k triangles = efficient, 1M = typical 1440p, 10M = massive (Nanite scale). Measure on-screen triangles (after culling), not total.

- **VRAM Usage**: Memory resident in GPU. Monitor per-frame (often not constant due to streaming, GC). Typical budgets: PC 4-8GB (GPU VRAM), Console 8-16GB (shared system RAM), Mobile 2-4GB (shared). Exceeding = GC stutters, reduced quality.

- **Frame-Time Histogram**: Bucketed distribution of frame times. Buckets: 0-8ms, 8-10ms, 10-12ms, 12-14ms, 14-16ms, 16-33ms (dropped frames), >33ms (very bad). Production metric: log histogram per 10 seconds, identify spikes.

## Best Practices

**CPU Frame Time Measurement:**
```csharp
private double m_CPUFrameStart;
private List<double> m_CPUFrameTimes = new List<double>(300);  // 5 seconds at 60 fps

void Update()
{
    m_CPUFrameStart = GetHighResolutionTime();
    
    // Game logic, physics, rendering API calls
    GameLogic();
    Physics();
    RenderScene();
    
    double cpuFrameTime = GetHighResolutionTime() - m_CPUFrameStart;
    m_CPUFrameTimes.Add(cpuFrameTime);
    
    // Keep rolling window of 5 seconds
    if (m_CPUFrameTimes.Count > 300)
        m_CPUFrameTimes.RemoveAt(0);
}

double GetHighResolutionTime()
{
    return System.Diagnostics.Stopwatch.GetTimestamp() / (double)System.Diagnostics.Stopwatch.Frequency;
}
```

**GPU Frame Time Measurement (DX12):**
```hlsl
// Shader code: insert timestamps
beginQuery(TIMESTAMP_QUERY_HEAP, 0);  // Frame start
RenderGeometry();
endQuery(TIMESTAMP_QUERY_HEAP, 1);    // Frame end

// CPU code: read timestamps (1-2 frame latency)
if (previousFrame.timestampQuery.Ready())
{
    uint64_t startTicks = GetQueryResult(timestampHeap, 0);
    uint64_t endTicks = GetQueryResult(timestampHeap, 1);
    float gpuFrameMS = (endTicks - startTicks) / (float)gpuFrequency * 1000.0f;
}
```

**Frame-Time Histogram (Production):**
```csharp
public class FrameTimeHistogram
{
    private int[] m_Buckets = new int[10];  // 0-2ms, 2-4ms, ..., 18-20ms, 20+ms
    private int m_TotalFrames = 0;
    
    public void RecordFrameTime(float frameTimeMS)
    {
        int bucket = Mathf.Clamp((int)(frameTimeMS / 2.0f), 0, 9);
        m_Buckets[bucket]++;
        m_TotalFrames++;
    }
    
    public void LogStats()
    {
        Debug.Log("Frame Time Histogram:");
        for (int i = 0; i < 10; i++)
        {
            float percent = (m_Buckets[i] / (float)m_TotalFrames) * 100.0f;
            Debug.Log($"{i*2}-{(i+1)*2}ms: {percent:F1}%");
        }
    }
}
```

**Memory Tracking:**
- Allocations per frame: log all new/malloc calls, track count and size. Typical: <5 allocations/frame (should be 0 in optimized code). >100 = GC pressure.
- VRAM usage: sample every frame via `SystemInfo.graphicsMemorySize` (approximate) or detailed profilers (NVIDIA Nsight, PIX).
- Monitor per-texture (editor: Profiler -> Memory -> Textures). Identify bloat (2K textures when 1K sufficient, etc.).

**Platform-Specific Metrics:**

*PC (Windows/Linux):*
- CPU: QueryPerformanceCounter (microsecond precision).
- GPU: DX12 timestamps (GPU clock domain, may drift vs CPU). NVIDIA/AMD provide additional metrics (register utilization, memory bandwidth utilization).
- Profilers: NVIDIA Nsight (RTX, ray tracing), AMD GPU Profiler (RDNA), PIX (Windows).

*Console (PS5/Xbox Series X):*
- PS5: Gnmx profiler, custom frame-capture tools. GPU timeline accurate (hardware timestamps).
- Xbox Series X: PIX (native Windows tool, used for Xbox profiling too). Per-shader timing, memory access patterns.
- Both expose specialized metrics (ray-tracing occupancy, SSD I/O, network latency).

*Mobile (Android/iOS):*
- CPU: gettimeofday (millisecond precision, sufficient for mobile). High-res timers available but expensive.
- GPU: Android GPU Inspector (AGI), Xcode Metal profiler (iOS). Timestamps supported but may have overhead (3-5% GPU time).
- Battery: measure power draw (mA) via device APIs. Frame time vs power: 60 fps might use 2W, 30 fps 1W (optimize for power, not just frame time).

## Common Pitfalls

**1. Measuring Average Frame Time Only**
- *Symptom*: Average frame time 16.2ms (looks good), but game feels stuttery. 1% of frames take 40ms.
- *Cause*: Average hides outliers. One 40ms frame per 100 = 0.2% increase in average, but very noticeable to player.
- *Solution*: Always track percentiles. 50th, 95th, 99th percentile frame times. Report worst-case (99th), not average.

**2. GPU Timestamp Drift (Inconsistent Measurements)**
- *Symptom*: GPU frame time shows 8ms one frame, 15ms next frame, despite same workload.
- *Cause*: GPU timestamp clock drifts vs CPU clock. Or reading stale results (timestamp query results delayed 2-3 frames, CPU cache effects).
- *Solution*: Average GPU timestamps over 10+ frames. Or use stable clock domain (CPU for consistency, accept slight inaccuracy). Add disclaimer (GPU times are approximate).

**3. Profiling Overhead Dominating Results**
- *Symptom*: Game runs 60 fps normally, but with metrics collection drops to 30 fps.
- *Cause*: Excessive logging (string formatting, allocations, file I/O). GPU timestamp queries stalling pipeline.
- *Solution*: Use low-overhead metrics (integer histogram, no strings). Reduce query frequency (every 10th frame, or aggregate). Never enable detailed profiling in production.

**4. VRAM Budget Exceeded Intermittently**
- *Symptom*: Game occasionally stutters (33ms+ frames), VRAM spikes to 110% capacity, then recovers.
- *Cause*: Dynamic streaming causes temporary VRAM overages. Streaming new asset while old still resident.
- *Solution*: Monitor VRAM trend (smoothed moving average). Preemptive: stream assets before needed, not on-demand. Or increase VRAM budget estimate (Plan for 110%, not 100%).

## Tools & Workflow

**Real-Time Metrics Dashboard:**
```csharp
public class MetricsDisplay : MonoBehaviour
{
    private float m_CPUTime, m_GPUTime, m_FrameTime;
    private int m_DrawCalls, m_Triangles;
    
    void Update()
    {
        m_CPUTime = Time.deltaTime * 1000.0f;
        m_GPUTime = GetGPUFrameTime();  // From GPU query results
        m_FrameTime = m_CPUTime + m_GPUTime;
        m_DrawCalls = GetDrawCallCount();  // Via graphics profiler
        m_Triangles = GetVisibleTriangleCount();
        
        // Display on-screen or log
        string metrics = $"CPU: {m_CPUTime:F2}ms GPU: {m_GPUTime:F2}ms Draws: {m_DrawCalls} Tris: {m_Triangles}";
        Debug.Log(metrics);
    }
}
```

## Related Topics

- [28.2 In-Game Debug UI](28-02-In-Game-Debug-UI.md) — Visualizing metrics in real-time
- [28.3 Remote Profiling](28-03-Remote-Profiling.md) — Streaming metrics to development machine
- [04 Profiling Tools](../chapters/04-Profiling-And-Analysis-Tools.md) — Industry profiling tools (NVIDIA Nsight, PIX)
- [06 Bottleneck Identification](../chapters/06-Bottleneck-Identification.md) — Interpreting metrics to find bottlenecks

---

[← Previous: 27.5 Nanite-Like Techniques](27-05-Nanite-Like-Techniques.md) | [Next: 28.2 In-Game Debug UI →](28-02-In-Game-Debug-UI.md)
