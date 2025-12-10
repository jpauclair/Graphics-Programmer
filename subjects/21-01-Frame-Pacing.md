# 21.1 Frame Pacing

[← Back to Chapter 21](../chapters/21-Frame-Management-And-Synchronization.md) | [Main Index](../README.md)

Frame pacing controls render timing: V-Sync (synchronizes to monitor refresh, eliminates tearing, adds latency), frame rate limiting (caps FPS for power efficiency, thermal management), frame budget management (time-based profiling, maintain target frame time). Goal: consistent frame rates (no stutters, smooth gameplay) while managing power/heat.

---

## Overview

Frame Timing: Each frame must complete within target time (16.67ms for 60 FPS, 33.33ms for 30 FPS, 11.11ms for 90 FPS VR). Exceeding budget = frame drop (stutters, visible hiccup, 60 FPS drops to 30 = instant halving). Frame rate stability critical (steady 60 FPS better than 60-50-60-45 variation, players notice inconsistency). Profiler tracks frame time (Profiler > Timeline, shows CPU/GPU time per frame, identifies bottlenecks).

V-Sync (Vertical Synchronization): GPU waits for monitor vertical blanking interval (v-blank), renders at monitor refresh rate only (60Hz = 60 FPS, 120Hz = 120 FPS). Benefits: eliminates tearing (frame fragments on screen, v-sync prevents), smooth visual. Drawbacks: adds latency (waits for v-blank, input lag 16.67ms+ at 60 FPS), may waste GPU time (completes frame early, idles until v-blank), complicates timing. Implementation: QualitySettings.vSyncCount (0 = disabled, 1 = 60 FPS v-sync, 2 = 30 FPS v-sync = half refresh rate). Modern: adaptive v-sync (Nvidia G-Sync, AMD FreeSync) matches monitor to GPU output (no tearing, less latency).

Frame Rate Capping: Artificially limit FPS (throttle GPU/CPU execution), save power/heat. Use cases: mobile (limit 30 FPS = battery efficiency, 60 FPS = high power), console ports (target 30/60 FPS per platform), power-constrained devices (Switch = 30 FPS docked, 30 FPS handheld thermal). Implementation: Application.targetFrameRate (capped FPS, 60 default, set to 30 for mobile), or WaitForSeconds(1/targetFPS) in Update (frame delay loop). Disadvantage: may miss responsive input (capped 30 FPS = input checked 30x/sec, 120 FPS = 120x/sec, input-to-response feels laggy at 30).

Frame Budget: Time allocation per frame (CPU + GPU time <16.67ms for 60 FPS). Typical splits: Physics 2ms, Animation 1ms, Rendering 13.67ms = total 16.67ms. Profiling: Profiler.GetTotalMemoryUsed() for memory, Profiler.GetTotalAllocatedMemoryLong() for allocations, Profiler records frame time breakdown. Tools: Unity Profiler (CPU/GPU usage), external profilers (Frame Rate Counter on device, captures actual FPS over time). Optimization: identify worst frame (exceeds budget), break culprit (remove physics iteration, simplify animation, reduce draw calls), test again. Goal: 99th percentile frame time within budget (occasional spikes acceptable, consistent frame drops = problem).

## Key Concepts

- **V-Sync (Vertical Synchronization)**: GPU synchronizes output to monitor refresh cycle (waits for v-blank = vertical blanking interval, short period between monitor scans). Process: GPU renders frame, displays in frame buffer, waits until v-blank (monitor finishes displaying previous frame), GPU signals display to show new frame, process repeats. Timing: at 60 Hz monitor (16.67ms per refresh), v-sync occurs 60 times/second, GPU can only present frame during v-blank. Effect: no tearing (image fragments not visible, v-sync ensures atomic frame swap), smooth visual (aligned with display), but input latency (GPU may complete frame early, waits for next v-blank = delays input response).
- **Adaptive V-Sync**: Modern GPU/monitor sync technology (Nvidia G-Sync, AMD FreeSync, VESA Adaptive-Sync). Monitor refresh rate variable (instead of fixed 60 Hz, varies 30-144 Hz based on GPU output). GPU outputs frame at any time (not waiting for v-blank), monitor refreshes exactly when frame ready (no tearing, no v-sync latency). Benefits: tear-free + low latency + power efficient (GPU not idling), adoption increasing (high-end displays, modern consoles). Drawback: requires compatible monitor/GPU (not universal).
- **Frame Rate Limiting**: Artificially cap GPU/CPU frame rate (below max possible). Methods: Application.targetFrameRate = 30 (OS enforces 30 FPS target, throttles rendering), or WaitForSeconds(sleepTime) per frame (active frame delay, sleeps CPU until next frame time). Use case: mobile battery (limit 30 FPS = 50% power vs 60 FPS), console optimization (target 60 or 30 per spec), thermal management (sustained high FPS = heat, throttle to stay cool). Trade-off: latency vs efficiency (30 FPS = lower latency + less power, 120 FPS = higher latency + more power).
- **Frame Time Budget**: Time allocation across frame systems (total per-frame time cap). Typical: 16.67ms at 60 FPS divides into CPU time (script updates, physics, animation, rendering prep = 8-10ms), GPU time (actual rendering = 8-10ms), reserve (1-2ms headroom). Measurement: Profiler tracks CPU time (script, physics, rendering prepare), GPU time (rendering execute), records per-frame breakdown. Optimization: profile (find bottleneck = which system exceeds budget), reduce (fewer objects, simpler physics, fewer draw calls), retest. Rule of thumb: 99th percentile frame time (worst 1% of frames) determines perceived smoothness (occasional 20ms frame acceptable if 99% are 16.67ms).
- **Frame Drops**: Frame exceeds time budget (takes 25ms instead of 16.67ms, causes visible pause/stutter). Single drop (one 25ms frame among 60fps) = noticeable hiccup (momentary stutter), persistent drops (frame rate halves 60→30) = game feels sluggish. Causes: garbage collection spike (memory cleanup), physics recalculation, AI update, terrain loading, draw call peak. Detection: Profiler shows frame spike (tall bar in timeline), or visible pause during gameplay. Fix: identify cause (GC profiler tab, Physics profiler, Rendering profiler), reduce workload that frame (defer tasks, use object pooling for GC, disable colliders, simplify visuals).
- **Frame Rate Variance**: Inconsistent FPS (60, 58, 62, 55, 60 FPS over seconds) feels jerky (eye perceives variance, steady 55 FPS feels smoother than 60-50-60-50). Causes: variable workload (some frames complex, others simple), thermal throttling (GPU heats up, slows down, cools, speeds up, cycle repeats), background system activity (OS task, interrupts). Measurement: frame time variance (standard deviation of frame times), delta time (Time.deltaTime, changes frame to frame). Optimization: smooth workload (level out frame complexity, defer heavy tasks), target steady state (50 FPS steady better than 60 peak = 30 drop).

## Best Practices

**V-Sync Configuration:**
- Desktop: QualitySettings.vSyncCount = 1 (60 FPS v-sync, standard, eliminates tearing), or 0 if adaptive sync (G-Sync/FreeSync, handles sync), or 2 if high FPS cap needed (30 FPS v-sync fallback for old displays). Note: v-sync adds 1-frame latency (waits for refresh), acceptable for most games, critical for competitive/VR (lower latency = better).
- Console: platform-specific (PS5 defaults 120Hz capable, Xbox Series X defaults 120Hz capable, set per game spec = 30/60/120 FPS).
- Mobile: adaptive sync rare (older phones no support), disable v-sync (vSyncCount = 0) + frame rate cap (targetFrameRate = 30-60) gives better control, or enable v-sync if 60 Hz display (vSyncCount = 1, 60 FPS, stable).

**Frame Rate Target Selection:**
- 60 FPS: standard for action games (responsive input, smooth motion), mainstream target (16.67ms budget, most devices capable).
- 30 FPS: cinematic, power-efficient (33.33ms budget, half GPU load of 60), acceptable for slower games (strategy, simulation), mobile battery savings.
- 90 FPS: VR standard (11.11ms budget, reduced motion sickness, input latency matters), high-end mobile (OnePlus, Samsung flagship 120Hz screens), PC enthusiasts.
- 120 FPS: competitive shooters (8.33ms budget, 120Hz screens, lower latency), console capability (PS5/Series X, newer displays), rare mobile (battery intensive).
- Mixed targets: 60 FPS default, 30 FPS when CPU/GPU load high (cap down), 90 FPS in VR, adaptive (DLSS/FSR scale, maintain target framerate with quality adjustment).

**Frame Budget Management:**
- Profiler setup: Window > Analysis > Profiler (CPU tab = script time, Physics, Rendering prepare), GPU Usage tab (rendering execute), Memory tab (allocations, GC). Profile at target device (profile PC but ship mobile = different bottlenecks, always profile target hardware).
- Frame time targets: Calculate per-frame time (1000ms / FPS = 16.67ms at 60 FPS, 33.33ms at 30 FPS, 11.11ms at 90 FPS), allocate budget (Physics 20%, Animation 20%, Rendering 40%, Scripts 20% = breakdown), assign per-system (Physics.defaultSolverIterations = reduce iterations saves time, draw calls target = optimize rendering).
- Workload leveling: defer heavy tasks (load assets next frame, not this frame), use object pooling (eliminates GC spikes, pre-allocate objects), spread work (particle spawns = 5 per frame instead of 50 once/sec).

**Frame Rate Consistency:**
- Monitor frame time delta (Time.deltaTime, log to understand variance), target low variance (std dev <5% of target FPS, 60 FPS ±3ms variance = good, ±10ms = poor).
- Reduce garbage collection (use pooling, avoid List.Add() in Update, cache component references), avoid frame spikes (defer loading, disable physics if not needed, cull distant objects = draw call reduction).
- Test on worst-case device (oldest target phone, base PS4, etc), cap frame rate if necessary (targetFrameRate = 30 on old mobile, steady 30 better than variable 30-50).

**Platform-Specific:**
- **PC**: 60 FPS standard (targetFrameRate = 60, vSyncCount = 1 for display, 0 for G-Sync), high-end = 120+ FPS (2080 Ti, 3090, newer displays), ultra-settings scale (quality + frame rate = trade-off).
- **Consoles**: PS5/Series X = 60 FPS default (some games 30 FPS mode for maximum quality), 120 FPS for backwards compatible titles, older consoles (PS4/One) = 30 FPS standard (targetFrameRate = 30, build optimized for 30 FPS frame budget).
- **Switch**: 30 FPS handheld and docked (targetFrameRate = 30, thermal constraints), aggressive optimization (smallest content, physics LOD, draw call limits).
- **Mobile**: 30 FPS default (battery efficiency, thermal), 60 FPS for high-end devices (OnePlus 9+, Samsung Galaxy S20+), detect device tier (device model check), cap accordingly.

## Common Pitfalls

**V-Sync Latency Misunderstood**: Developer enables V-Sync (thinking eliminates all tearing, improves quality), doesn't realize latency cost (mouse click delayed by 16.67ms at 60 FPS, feels unresponsive in competitive games). Symptom: input lag (click response delayed, targeting feels sluggish, especially noticed in FPS games). Solution: use adaptive sync if available (G-Sync/FreeSync, latency eliminated), or disable v-sync for competitive (accept minimal tearing, gain latency), or run uncapped + frame pacing (target framerate = 120 FPS uncapped, most frames render sub-16.67ms, occasional frame over budget, acceptable trade-off).

**Frame Budget Ignored**: Developer ships game, doesn't profile (assumes 60 FPS works), on-device frame rate is 30 FPS (drop due to thermal throttling, GC, unoptimized physics). Symptom: poor device performance (game stutters, slower than expected), user reviews negative (laggy, unplayable). Solution: profile on target device early (not after shipping), set frame budget (16.67ms for 60 FPS), identify bottleneck (CPU or GPU limited), optimize (reduce draw calls for GPU, physics iterations for CPU, GC impact).

**Inconsistent Delta Time**: Developer assumes Time.deltaTime = constant (writes physics as distance = speed * 0.016), Time.deltaTime actually varies (0.010-0.020 due to frame time variation), physics drifts (position calculation errors accumulate, desync occurs). Symptom: physics simulation inconsistent (objects move different distances per second, cumulative error over time), network desync (server simulates 60 FPS, client frame-time variable, position mismatches). Solution: use fixed timestep (Physics.fixedDeltaTime = 0.016 constant, decoupled from frame rate), or normalize Time.deltaTime in calculation (distance = speed * Time.deltaTime maintains consistency regardless of frame rate variance).

## Tools & Workflow

**Unity Profiler Frame Time Analysis**: Window > Analysis > Profiler. Timeline tab (shows frame-by-frame breakdown, bars colored by system: yellow = script, green = render, purple = physics), click bar to inspect breakdown (which script/system exceeded budget). Memory tab (GC allocation spikes = memory churn). GPU Usage (rendering time breakdown). Workflow: capture frame (record button), play game, look for spikes (tall bars = frame exceeds budget), hover to identify culprit (script? Physics? Rendering?), optimize.

**Frame Rate Counter**: built-in display (Canvas with FPS text, updates every frame). Code: 
```csharp
public Text fpsText;
float deltaTime = 0;
void Update() {
    deltaTime += Time.deltaTime;
    if (deltaTime >= 1) {
        fpsText.text = (1f / Time.deltaTime).ToString("F1");
        deltaTime -= 1;
    }
}
```

**Application.targetFrameRate**: Set in code or Editor Preferences (Edit > Project Settings > Time > Fixed Timestep). Code: `Application.targetFrameRate = 60;` (caps rendering to 60 FPS, OS throttles execution).

**Physics Profiling**: Profiler > Physics tab (shows Simulate time, narrow phase, broad phase), reduce if high (Physics.defaultSolverIterations = 6 default, reduce to 4 saves time, may reduce stability, balance quality/performance). Alternatively, disable colliders (Collider.enabled = false) on distant objects (saves broad phase time).

## Related Topics

- [06.2 GPU Architecture](06-02-GPU-Architecture.md) - GPU scheduling, pipeline depth
- [21.2 Console Frame Pacing](21-02-Console-Frame-Pacing.md) - Platform-specific timing
- [21.4 GPU Synchronization](21-04-GPU-Synchronization.md) - GPU command buffering
- [14.1 Profiling Fundamentals](14-01-Profiling-Fundamentals.md) - Frame time measurement

---

[Previous: Chapter 20](../chapters/20-Specialized-Rendering.md) | [Next: 21.2 Console Frame Pacing →](21-02-Console-Frame-Pacing.md)
