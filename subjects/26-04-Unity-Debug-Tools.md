# 26.4 Unity Built-In Debug Tools & Profiler

[← Back to Chapter 26](../chapters/26-Debugging-Techniques.md) | [Main Index](../README.md)

Unity Profiler = built-in performance measurement (CPU, GPU, memory, rendering metrics). Panels: CPU (where time spent), GPU (GPU timeline), Memory (allocations), Rendering (draw calls, geometry). Other tools: Memory Profiler (deep memory analysis), Build Report (binary breakdown), Frame Debugger (GPU commands per frame), Console (error messages = quick debugging). Integration: all accessible from Editor menu or code (programmatic access via APIs = custom profiling).

---

## Overview

Profiler Architecture: profiler samples runtime = every frame records metrics (CPU time per category, GPU time, memory used). Data stored = can replay offline (record session, analyze later). Overhead: active profiling = 5-10% performance cost (sampling + data collection = CPU work). Best practice: profile, then disable profiler = get true game performance (profiler off = accurate frame times).

Typical Workflow: identify slow frame (Profiler shows spike), drill down (CPU tab = show which function), optimize (change code), re-profile (verify improvement).

## Key Concepts

- **CPU Profiler Tab**: hierarchical call stack (Deep Profile = track every function call = expensive, Shallow Profile = sample = fast). Time spent: self time (function itself) vs total time (function + callees). Categories: Rendering (GPU commands prep), Physics (collision, simulation), Scripting (C# code), Other (engine systems). Per-sample breakdown: see how frame divided between systems.
- **GPU Profiler Tab**: GPU timeline (each draw call = time spent). Organized by: Camera (eye0, eye1 for stereo), render pass (G-buffer, lighting, post-fx = organized), objects. GPU Utilization = bar height = time spent. Identify expensive calls (tall bars = hot spot).
- **Memory Profiler**: Window > Analysis > Memory Profiler (separate window). Show: allocated memory by category (Textures, Meshes, Audio, Scripts), tracking (what allocated = call stack, when = timeline). Memory Leak Detection: allocate every frame without freeing = graph shows growth = leak indicated.
- **Rendering Statistics**: bottom of Profiler = quick stats (Batches = draw calls, Vertices = total polygons, Texture Mem = VRAM used). At a glance = see rendering load (if Batches > 500 = CPU bottleneck likely).
- **Build Report**: File > Build Settings > Build (generates report showing binary breakdown = Textures/Meshes/Code sizes). Identify biggest assets = optimize first (texture 2GB, code 100MB = focus on textures).

## Best Practices

**Profiler Setup**:
- Enable in Development builds (Edit > Project Settings > Player > Development Build = on = profiler included).
- Play scene (Profiler records live = see real gameplay metrics).
- Record session (UI button = capture 10-60 sec of play).

**CPU Bottleneck Analysis**:
- CPU tab > Rendering category (if high = draw call overhead = batching needed).
- CPU tab > Physics category (if high = many bodies = optimize collision, or use Burst + DOTS).
- CPU tab > Scripting category (if high = custom code slow = profile deeper = see function names).

**GPU Bottleneck Analysis**:
- GPU tab = see timeline (tall bars = expensive calls).
- Select call = see shader name (if unknown shader = problem = verify material assigned correctly).
- Count expensive calls (if same shader many times = loop rendering = combine into single call = batching).

**Memory Analysis**:
- Memory Profiler = see what allocated. "Textures 1.5 GB" = too much = compress or use lower resolution.
- Timeline = allocations over time (if growing every frame = leak = investigate).
- Categories: Texture Mem (runtime VRAM), Managed (C# objects), Native (engine objects).

**Development vs Release**:
- Development = slow (profiler = overhead, Burst disabled usually = managed code = slower). Absolute numbers misleading.
- Release = fast (IL2CPP = compiled, Burst = optimized). But relative profiling still valid (if function A > function B in dev = A still slower in release, proportions maintained).

## Common Pitfalls

**Profiler Overhead**: developer profiles game = sees 30 FPS. Disables profiler = runs 60 FPS (profiler = 50% overhead). Symptom: profiler makes game look slow. Solution: profile to identify bottleneck, then optimize based on findings (don't obsess with profiler absolute numbers = focus on relative comparison).

**Memory Leak Missed**: profiler shows memory slowly growing (100 MB -> 200 MB over 30 min play). Developer doesn't notice (gradual = not obvious). Symptom: late-session crashes (memory full = OOM killer). Solution: review Memory Profiler timeline (growth indicates leak = investigate allocations).

**Deep Profile Always On**: developer enables Deep Profile (sample every function). Profiler = 50-100% overhead (too slow). Iteration slow (can't test realistically). Solution: use Shallow Profile for general overview, Deep Profile only for specific function investigation (enable/disable as needed).

## Tools & Workflow

**Profiler Quick Start**:
1. Window > Analysis > Profiler (opens Profiler)
2. Play scene (game runs, profiler records)
3. Record button = capture 10 sec
4. Pause (analyze captured data)
5. Select frame = inspect CPU/GPU breakdown

**Memory Leak Detection**:
```csharp
// Bad: allocate every frame, no cleanup
void Update() {
    var temp = new List<Vector3>(); // NEW every frame = leak
    temp.Add(transform.position);
}

// Good: allocate once, reuse
private List<Vector3> temp = new List<Vector3>();
void Update() {
    temp.Clear(); // Reuse
    temp.Add(transform.position);
}
```

**Profiler Breakdown** (estimate frame budget):
- 60 FPS = 16.67ms per frame
- Example: CPU 12ms (Rendering 5ms, Scripting 4ms, Physics 2ms, Other 1ms), GPU 14ms
- Bottleneck = GPU 14ms > CPU 12ms = GPU-bound = optimize shaders

**Custom Profiler Markers**:
```csharp
using Unity.Profiling;

private ProfilerMarker markerUpdateAI = new("UpdateAI");

void UpdateAI() {
    using (markerUpdateAI.Auto()) {
        // Custom profiler time measured
    }
}
```

## Related Topics

- [26.1 Visual Debugging](26-01-Visual-Debugging.md) - Frame Debugger, wireframe
- [26.2 GPU Debugging](26-02-GPU-Debugging.md) - RenderDoc, PIX
- [26.3 Performance Debugging](26-03-Performance-Debugging.md) - Bottleneck identification
- [04.3 Unity Built-In Tools](04-03-Unity-Built-In-Tools.md) - Tools overview

---

[Previous: 26.3 Performance Debugging](26-03-Performance-Debugging.md) | [Next: Chapter 27 →](../chapters/27-Advanced-Topics.md)
