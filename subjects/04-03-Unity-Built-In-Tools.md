# 4.3 Unity Built-in Tools

[← Back to Chapter 4](../chapters/04-Profiling-And-Analysis-Tools.md) | [Main Index](../README.md)

Unity provides built-in profiling tools accessible without external software. Profiler, Frame Debugger, and Memory Profiler are essential for daily optimization work across all platforms.

---

## Overview

Unity's built-in tools offer immediate profiling capabilities for CPU, GPU, memory, and rendering analysis. Unlike platform-specific profilers requiring SDKs and devkits, Unity tools work in Editor and standalone builds on all platforms. They integrate deeply with Unity's systems—showing script execution, physics simulation, rendering batches, and memory allocations with Unity-specific context.

The Profiler Window (Window > Analysis > Profiler) is the primary tool, displaying frame-by-frame CPU and GPU metrics. Frame Debugger (Window > Analysis > Frame Debugger) shows step-by-step draw call execution with full state inspection. Memory Profiler package reveals texture/mesh allocations and memory leaks. Together, these tools handle 80% of optimization tasks before reaching for external profilers.

Unity tools have limitations: GPU profiling is high-level (draw call milliseconds, not wavefront occupancy), CPU profiling adds overhead (10-30% with Deep Profiling), and platform-specific features (VRS, ray tracing costs) are invisible. Use Unity tools for initial optimization, then graduate to PIX/Razor/Nsight for deep GPU analysis or platform-specific tuning.

## Key Concepts

- **Profiler Window**: Primary profiling interface showing CPU modules (scripts, rendering, physics), GPU timeline, memory usage, and audio/video metrics. Records 300-900 frames of data for analysis.
- **Frame Debugger**: Step-through debugger for draw calls. Pause rendering at any draw, inspect shader properties, textures, and render targets. Shows batching effectiveness and state changes.
- **Memory Profiler**: Package (com.unity.memoryprofiler) visualizing memory allocations. Shows texture/mesh memory, managed heap fragmentation, and native allocations. Essential for finding memory leaks.
- **Deep Profiling**: Records every C# function call (not just Unity API calls). Reveals expensive scripts but adds 30-50% overhead. Use sparingly on small test scenes.
- **Profiler Markers**: Custom markers inserted via `Profiler.BeginSample`/`EndSample` organizing profiler hierarchy. Name code sections for easier navigation ("AI.Pathfinding", "Rendering.PostProcess").

## Best Practices

**Profiler Window Usage:**
- Profile standalone builds, not Editor. Editor adds 20-50% overhead (Inspector updates, scene view rendering, Editor scripts). Use "Autoconnect Profiler" in Build Settings to profile releases.
- Enable "GPU Profiler" module (Profiler Window > Add Profiler > GPU). Shows millisecond timing per draw call. Identify heaviest 5-10 draws for optimization.
- Sort modules by "Self Time" not "Total Time". Total includes children (misleading for hierarchical code). Self reveals actual function cost.
- Use "Deep Profiling" only on isolated systems. Enable for AI code review or script optimization, not full game scenes (too much overhead). Disable for GPU profiling.
- Record multiple frames (300+) to capture variance. Single frame misleads—spikes from garbage collection or asset loading skew results. Look at median and 95th percentile times.

**Frame Debugger Workflow:**
- Open Frame Debugger (Window > Analysis > Frame Debugger) in Play Mode. Click "Enable" to pause rendering at each draw.
- Click through draws to see visual contribution. Identify expensive draws (complex shaders, high poly counts) by visual impact vs cost.
- Check "Why this draw call can't be batched" in right panel. Common breaks: different materials, unsupported scale, lightmap differences. Fix batching issues to reduce draw calls.
- Inspect shader properties and textures. Verify correct material assignments, texture resolutions, and property values. Catches artist errors (8K texture on small prop).
- Use "Render Target" preview to visualize intermediate buffers (G-buffer, shadow maps, post-process targets). Identify oversized or unnecessary render targets.

**Memory Profiler:**
- Install via Package Manager (Window > Package Manager > Memory Profiler). Captures memory snapshots for offline analysis.
- Take snapshot in problematic scenes ("Capture New Snapshot"). Compare "Before" and "After" scenes to track memory growth or leaks.
- Sort by "Size" in TreeMap view. Largest textures/meshes appear first. Identify 4K textures used on small objects or imported at incorrect settings.
- Check "Managed Heap" for C# allocations. Fragmentation causes garbage collection spikes (10-30ms). Use object pooling to reduce allocations.
- "Native Memory" shows Unity engine allocations (textures, meshes, particles). Exceeding platform limits (10GB on consoles) causes hitches or crashes.

**Custom Profiler Markers:**
- Instrument expensive code with `Profiler.BeginSample("MySystem.Update")` / `Profiler.EndSample()`. Organizes profiler hierarchy, isolating systems for analysis.
- Use hierarchical names: "AI.Pathfinding", "AI.Perception", "AI.DecisionMaking". Groups related systems under parent category.
- Remove markers in release builds via conditional compilation (`#if ENABLE_PROFILER`). Marker overhead is small (microseconds) but eliminates it entirely in shipping builds.
- Markers propagate to platform profilers (PIX, Razor). Insert markers for GPU work via CommandBuffers to organize console profiler timelines.

**Platform-Specific:**
- **PC**: Profiler works in Editor or standalone. Use "Development Build" checkbox for profiler access. GPU timings accurate on DirectX 11/12 and Vulkan.
- **Console**: Install platform profiler packages (Xbox, PlayStation, Switch modules). Connect to devkit via "Profiler" window > "Attach to Player" dropdown. Provides Unity view of console profiling.
- **Mobile**: Use "Autoconnect Profiler" for remote profiling (Android via USB, iOS via WiFi). GPU metrics limited on mobile vs console—times are estimates, not hardware counters.

## Common Pitfalls

**Profiling in Editor**: Developer profiles in Unity Editor, sees 10ms "Render.OpaqueGeometry" cost. Builds standalone, actual cost is 6ms. Editor overhead (Inspector, Scene view, Editor coroutines) adds 20-50% fake cost. Optimization based on Editor profiling wastes effort. Solution: Always profile standalone Development builds. Editor profiling is acceptable only for CPU script optimization (physics, AI).

**Ignoring "Gfx.WaitForPresent"**: Profiler shows 8ms "Gfx.WaitForPresent" on CPU timeline. Developer focuses on optimizing CPU scripts (reducing 0.5ms here, 1ms there). WaitForPresent means CPU waits for GPU—GPU is the bottleneck, not CPU. Solution: Check GPU module. If GPU time > CPU time, optimize GPU (shaders, overdraw, resolution) not CPU.

**Deep Profiling Everything**: Enabling Deep Profiling for full game scene. Frame rate drops from 60fps to 20fps due to profiling overhead. Data is useless—overhead distorts results completely. Solution: Use Deep Profiling only on isolated test scenes (single AI character, small level section). Profile specific systems, not entire game.

## Tools & Workflow

**Profiler Window** (Window > Analysis > Profiler): Primary tool. "CPU Usage" module shows script and rendering cost. "Rendering" module displays batches, SetPass calls, triangles. "GPU" module (enable manually) shows draw call timings. Use "Hierarchy" view for drill-down, "Timeline" for visualizing frame work.

**Frame Debugger** (Window > Analysis > Frame Debugger): Step through draw calls. Left panel lists all draws with thumbnails. Right panel shows shader properties, textures, render targets. Bottom preview shows rendered output after selected draw. Use to validate batching and identify heavy draws.

**Memory Profiler** Package (install via Package Manager): Window > Analysis > Memory Profiler. "Capture Snapshot" saves memory state for analysis. "TreeMap" view shows allocations by size. "Table" view lists all objects with sizes. "Compare" mode finds memory leaks (allocations in Snapshot B but not A).

**Physics Debugger** (Window > Analysis > Physics Debugger): Visualizes colliders, contact points, and forces in Scene view. Use to debug why physics costs are high (too many active rigidbodies, complex colliders, excessive contacts).

**Audio Profiler** (Window > Analysis > Audio): Shows playing audio sources, DSP load, and audio memory. Use to identify expensive audio effects or excessive voice counts (>32 voices on consoles).

## Related Topics

- [4.1 PC Profiling Tools](04-01-PC-Profiling-Tools.md) - External profilers (PIX, RenderDoc, Nsight)
- [4.2 Console Profiling Tools](04-02-Console-Profiling-Tools.md) - Platform profilers (Razor, NNGFX)
- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - Using profiler data to optimize CPU
- [6.1 CPU Bottlenecks](06-01-CPU-Bottlenecks.md) - Interpreting profiler data for bottlenecks

---

[← Previous: 4.2 Console Profiling Tools](04-02-Console-Profiling-Tools.md) | [Next: 4.4 Third-Party Profiling Tools →](04-04-Third-Party-Profiling-Tools.md)
