# 24.2 Mobile Optimization

[← Back to Chapter 24](../chapters/24-Mobile-And-Cross-Platform.md) | [Main Index](../README.md)

Mobile optimization addresses unique constraints: battery (5-20 Wh capacity, 2-4 hour play session budget), thermal (no active cooling, throttle if >85°C), memory (4-8 GB RAM, shared with OS, 1-2 GB available). Frame time tight (30 FPS = 33ms budget, mobile CPU/GPU slower than console = aggressive optimization mandatory). Battery drain = CPU + GPU + connectivity (WiFi drains less than LTE), screen brightness dominant (50%+ battery).

---

## Overview

Mobile Battery Budget: 2-4 hour session = typical gaming target (players expect 2+ hours), power budget = 5-20 Wh / 2 hours = 2.5-10 Watts sustained. Breakdown: screen 50%, CPU/GPU 30%, other (connectivity, sensors) 20%. Optimization: reduce CPU/GPU load (lower draw calls, simpler shaders, reduced logic), balance frame rate (30 FPS 2W vs 60 FPS 4W = 2x battery drain), or dynamic frame rate (60 FPS when plugged, 30 FPS on battery).

Thermal Management: mobile devices = thin, poor heat dissipation. GPU sustained high load = temperature rises, throttle activates (clock speed drops, performance degrades), player feels lag. Solution: profile thermal load (run 60 minutes, measure peak temp), keep under 80°C sustained (aim <75°C for safety), dynamic throttle (if temp detected, reduce settings). Switch handheld mode = thermal limit critical (no ventilation = temp rises fast).

Memory Constraints: shared RAM with OS (8GB total, 2GB OS = 6GB game, but foreground apps add overhead = 2-3GB available). Asset budget = 1-2GB (everything: textures, meshes, scripts, UI), streaming = load on-demand (divide levels = 500MB each, stream chunks). Cache management = aggressive (don't cache assets not immediately needed).

## Key Concepts

- **Battery Power Modeling**: Power (Watts) = CPU load + GPU load + fixed idle power. CPU: ~0.1W per core active, 4 cores idle + 1 active = 0.1W typical, 4 active = 0.4W. GPU: light (simple shaders) = 0.5W, heavy (complex shaders, high polygon) = 2-3W. Fixed: screen 2-3W (backlight), modem 0.5W, other 0.5W = total 3-5W baseline, add CPU/GPU = 3-8W total. Battery: 3600 mAh 3.7V = 13.3 Wh, 60 FPS 60W = 13.3Wh/60W = 8 minutes. 30 FPS 4W = 13.3Wh/4W = 3.3 hours = reasonable.
- **Thermal Throttling**: Temperature sensor monitors GPU/CPU, OS throttles if exceeds threshold (typically 85°C critical, 80°C warning). Throttle effect: clock speed reduces (e.g., 800MHz -> 600MHz = 75% throughput), frame time increases (30 FPS -> 22.5 FPS = noticeable lag), player perceives as game slowdown (not expected lag). Prevention: keep sustained <75°C (5°C margin before throttle), or accept throttle and handle gracefully (UI message, reduce quality automatically).
- **Memory Streaming**: divide level into chunks (main area 500MB, adjacent areas 300MB preloaded, distant 300MB unloaded). Load ahead of time (player approaching area = preload before visible). Unload behind (leave area = unload after 1 minute). Streaming cost: 2-5 seconds per 300MB chunk (depends on storage speed = SSD faster, HDD slower). Bandwidth: phone storage 200-600 MB/sec (newer phones SSD-like, older USB 2.0 = 60 MB/sec = slow).
- **Frame Rate Modes**: 30 FPS standard (battery efficient, sustained thermal), 60 FPS high-end only (flagship phones, plugged in or short play). Dynamic: start 60, if battery low switch to 30, if thermal detected switch to 30. Mixed: UI @ 60 FPS (responsive), gameplay @ 30 FPS (battery saving). Settings: Application.targetFrameRate dynamic switch.
- **Scalability Systems**: detect device capability (model name, RAM, GPU name), set quality tier automatically. High-end (flagship 2020+ = full quality), mid (2-3 year old = medium quality, 1080p, medium draw calls), low-end (budget phone = 720p, low draw calls, simple shaders). Per-device mapping (hardcoded list, or heuristic = if RAM > 6GB and GPU = Adreno 660, = high-end).

## Best Practices

**Battery Optimization**:
- Target 30 FPS (half power of 60, acceptable visual experience for most games), or 60 FPS for action games (trade battery for responsiveness).
- Dynamic frame rate: cap 30 FPS on battery, 60 FPS plugged. Code: `Application.targetFrameRate = IsBatteryLow() ? 30 : 60;`
- Reduce draw calls (each = CPU work = power), simplify shaders (complex shaders = GPU power), LOD aggressive (distance culling = save GPU).
- Disable features on low battery: particles, post-processing, shadows = optional cosmetics, disable = battery savings.

**Thermal Management**:
- Profile on device (run 60+ minutes, log temperature periodically, peak = thermal limit understanding). If >80°C sustained = problematic.
- Reduce GPU load (lower resolution, simplify shaders, fewer particles, disable expensive effects), or active cooling mitigation (if temp detected > 75°C, reduce quality settings dynamically).
- For mobile games: Thanksgiving stress test (intensive game for extended play), ensure <80°C sustained = safe for release.

**Memory Optimization**:
- Asset budget: 1-2 GB total (textures, meshes, audio = precious), compress aggressively (ASTC textures, meshes simplified, audio OGG/OPUS).
- Streaming: divide levels (main area + adjacent preload + far unload), asynchronous loading (LoadScene with allowSceneActivation = false, load in background, activate when ready).
- Cache: Resources folder deprecated (expensive to load), use addressables or AssetBundles (explicit memory management, unload when done).

**Platform-Specific**:
- **High-end** (2020+ flagship, 6+ GB RAM, Snapdragon 888+, A15): 60 FPS, 1440p, high draw calls (500+), complex shaders, volumetric fog.
- **Mid-range** (2018-2020, 4-6 GB, Snapdragon 730-855, A12-A13): 30-60 FPS dynamic, 1080p, medium draw calls (200-300), simple shadows, minimal effects.
- **Low-end** (budget, <4 GB, Snapdragon 600-700, older A-series): 30 FPS, 720p, low draw calls (<100), no shadows, minimal physics.

## Common Pitfalls

**Ignoring Thermal**: Developer ships game (doesn't profile thermal load), players report phone gets hot, throttles. Symptom: players complain "game lags after 30 min", support issues = thermal. Solution: profile on handheld mode sustained play 60+ minutes, ensure <80°C, or implement thermal throttle handling (reduce quality if temp exceeds 75°C).

**60 FPS Target Everywhere**: Developer targets 60 FPS (good on desktop), ships mobile, battery drains quickly (60 FPS = 4-6W, 30 FPS = 2-3W = 2x battery drain). Players dissatisfied (play 1-2 hours, battery dead). Symptom: battery complaints, low App Store rating ("drains battery"). Solution: default 30 FPS for battery games, offer 60 FPS toggle, or dynamic (60 FPS if plugged).

**Memory Bloat**: Developer loads entire level (3GB) into memory, mobile only has 2GB available. Game crashes (out of memory). Symptom: random crashes early in play (initial loading succeeds, crashes when second area loads). Solution: implement streaming (divide level into chunks, load/unload dynamically), or compress assets more (target 1.5GB level max).

## Tools & Workflow

**Thermal Profiling**: Run game 60+ minutes on device, log `SystemInfo.deviceModel` and temperature (via thermal API if available = limited on mobile, or monitor via device dashboard = Settings > Battery).

**Memory Profiler**: Window > Analysis > Memory Profiler (see what's loaded = textures, meshes, audio), target <2 GB active.

**Battery Simulation**: connect device to power meter (USB ammeter = measures current), multiply by voltage (3.7V) = Watts. Or device BatteryStats (adb shell dumpsys batterystats, view power consumption per component).

**Frame Rate Dynamic Code**:
```csharp
void Update() {
    if (SystemInfo.batteryLevel < 0.3f) { // <30% battery
        Application.targetFrameRate = 30;
    } else {
        Application.targetFrameRate = 60;
    }
}
```

**Scalability Detection**:
```csharp
Quality tier = "High";
if (SystemInfo.graphicsMemorySize < 2000) tier = "Low";
else if (SystemInfo.systemMemorySize < 4000) tier = "Mid";
QualitySettings.SetQualityLevel(tier switch {
    "Low" => 0,
    "Mid" => 1,
    "High" => 2,
});
```

## Related Topics

- [24.1 Mobile GPU Architecture](24-01-Mobile-GPU-Architecture.md) - TBDR, tile-based rendering
- [24.3 Scalability Systems](24-03-Scalability-Systems.md) - Dynamic quality adjustment
- [05.2 Memory Optimization](05-02-Memory-Optimization.md) - Asset memory management
- [14.1 Profiling Fundamentals](14-01-Profiling-Fundamentals.md) - Profiling on device

---

[Previous: 24.1 Mobile GPU Architecture](24-01-Mobile-GPU-Architecture.md) | [Next: 24.3 Scalability Systems →](24-03-Scalability-Systems.md)
