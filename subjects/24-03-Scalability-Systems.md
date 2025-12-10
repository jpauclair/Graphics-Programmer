# 24.3 Scalability Systems

[← Back to Chapter 24](../chapters/24-Mobile-And-Cross-Platform.md) | [Main Index](../README.md)

Scalability system auto-adjusts quality based on device/frame rate target (no manual settings required, optimal experience per hardware). Detect: device model, RAM, GPU name, frame rate achieved (measure FPS, if below target throttle settings), temperature. Set: resolution, draw call limits, shadow resolution, post-processing. Benefits: same game playable 2013 iPad to 2023 iPhone, no fragmentation (single APK/IPA, one codebase). Challenge: comprehensive device database (hundreds of models = maintain list or heuristic).

---

## Overview

Scalability Approach: detect device hardware (model name, system RAM, GPU capability), map to quality tier (Low/Medium/High), apply quality settings (resolution, LOD, shaders). Alternative: measure performance (run benchmark 60 seconds, measure FPS = too low = throttle settings and re-test, iterate until 60 FPS achieved = dynamic auto-scaling). Hybrid: initial detection for startup quality, runtime measurement for dynamic adjustment (if frame rate drops, reduce settings). Static approach simpler (database complete at ship time), dynamic more robust (adapts to OS updates, background processes).

Tier Structure: Low-end (720p, 30 FPS, minimal effects, simple shaders), Mid-range (1080p, 30-60 FPS dynamic, moderate effects, standard shaders), High-end (1440p, 60 FPS, volumetric effects, complex shaders). Per-tier settings: resolution scale (1.0 = native, 0.75 = 75% resolution = 75% pixel workload), draw call limit (low = 50-100, mid = 200-300, high = 500+), shadow cascade count (low = 1, mid = 2, high = 4), particle count multiplier (low = 0.5x, mid = 1.0x, high = 2.0x).

## Key Concepts

- **Device Detection**: Query hardware (SystemInfo class = graphicsDeviceName, systemMemorySize, processorCount, supportsRenderTextureFormat = capability check). Example: OnePlus 9 = 12GB RAM, Snapdragon 888 = High-end. iPhone 12 = 4GB RAM, A14 = Mid-range. Budget Android = 2GB RAM, Snapdragon 439 = Low-end. Database: hardcode tier per model (model name lookup table), or heuristic (RAM > 6GB AND GPU = high-end = High).
- **Quality Tiers**: Low (720p, 30 FPS, 50 draw calls, no shadows, no post-FX, basic particles), Mid (1080p, 30-60 FPS, 200 draw calls, baked shadows, SSAO, moderate particles), High (1440p, 60 FPS, 500 draw calls, realtime shadows, volumetric effects, many particles). Asset variants: low uses 512 textures, mid = 1024, high = 2048. Or single texture, runtime downscale (high memory = wasteful, prefer variants). Shader variants: low uses simple unlit, mid = standard PBR, high = PBR + detail maps + parallax.
- **Dynamic Frame Rate**: start at tier FPS target (low = 30, mid = 30-60, high = 60), measure actual FPS (Profiler or frame time counter), if below target 90% of frames = reduce settings (lower resolution 10%, next measure, iterate). If above target consistently = increase if possible (raise resolution, add particles). Hysteresis: require sustained low FPS (10+ frames below target) before downscale = avoid oscillation (bouncing up/down = distracting).
- **Resolution Scaling**: technique to reduce pixel workload (GPU-bound = common). Methods: render full resolution, downsample (blur + reduce), or render lower res + upscale (temporal upsampling + AI = DLSS/FSR, mobile = simple upscale = less quality but fast). Example: render 75% resolution = 0.75^2 = 56% pixels = 56% GPU work. Display: 1440p display but render 1080p = slightly blurry, acceptable trade-off for 2x GPU speedup.
- **LOD System**: detail levels (low = 10K tris, mid = 50K, high = 200K geometry). LOD switching: based on distance (far = low LOD) or global quality (low tier = all objects low LOD). Dynamic: measure triangle count, if exceeds tier limit = auto-reduce LOD (enable LOD1 over LOD0).

## Best Practices

**Device Database**:
- Comprehensive list: ship with 100+ device models mapped to tiers (maintain public database from app analytics = learn which devices play game).
- Heuristic fallback: if device not in database, use heuristic (RAM > 4GB && GPU supports feature X = High-end). Safer: unknown device = default to Mid (plays on anything, not optimal).
- Update: post-launch analysis (see which devices buy game, update database, push update = improved tier accuracy).

**Quality Settings Asset**:
- Create per-tier (Low, Mid, High = separate quality profiles). Each: shadow distance, LOD bias, resolution target, particle emitter count. Assign in Settings > Quality > select tier on startup.
- Shader variants: use multi_compile (compile per tier, at build time select variants), not runtime branching (if QUALITY_LOW use simple, else detailed = shader complexity = slower).
- Texture variants: low = 512 atlas, high = 2048 atlas, or single 2048 + stream to 512 if low-end.

**Frame Rate Monitoring**:
- Measure every frame (deltaTime = frame time), accumulate (sum 60 frames = 1 sec), check if below target > 90% = throttle. Avoid: measuring single frame (transient spike = false positive).
- Hysteresis: only adjust if low FPS persists (10+ consecutive frames), prevents oscillation (rapid up/down = distracting visual changes).
- Smooth transitions: reduce resolution 10% at a time (jumps too jarring), or fade to lower quality = less obvious.

**Platform-Specific**:
- **High-end Android/iOS**: detect premium features (ray-tracing if GPU supports, volumetric if memory >8GB), auto-enable for flagship devices.
- **Mid-range**: balanced = 1080p 30-60 FPS dynamic, moderate effects.
- **Low-end**: playable = 720p 30 FPS, minimal effects, but still fun.

## Common Pitfalls

**Device Database Incomplete**: Developer ships game, player on new device model not in database. Defaults to Mid-tier, but device is low-end (specs below database minimum). Game runs 15 FPS = unplayable. Symptom: complaints from budget phone users (game too slow, unplayable). Solution: heuristic fallback (if unknown, query hardware = be conservative, default Low-end if specs low), or post-launch update (gather analytics, add device to database, push update).

**Oscillating Scalability**: Developer implements dynamic FPS monitoring (if FPS < target reduce quality, if FPS > target increase quality). Player sees constant flicker (settings alternating every frame = quality oscillating). Symptom: visual instability (resolution changing every frame = distracting, players notice). Solution: add hysteresis (require 10+ frames low FPS before throttle, 30+ frames high FPS before increase), smooth transitions (fade adjustments over time).

**Memory Bloat in High Tier**: High tier loads 3GB of textures/meshes (assumes flagship device high-end model has 8-12GB RAM, ignoring OS overhead). If device has 6GB (after OS = 4GB available), game crashes (out of memory = crash when trying second area). Symptom: crashes on some flagships (background apps = less available memory = crash). Solution: cap per-tier memory budget (high = 2GB max, mid = 1.5GB, low = 1GB), implement streaming if exceeds, or compress textures more.

## Tools & Workflow

**Quality Tiers Setup**: Edit > Project Settings > Quality, create multiple profiles (Low, Mid, High). Per profile: Shadow Distance, LOD Bias, Texture Streaming Pool Size, Physics Solver Iterations (low = 6, mid = 8, high = 10). At startup, detect tier and QualitySettings.SetQualityLevel(index).

**Device Detection Code**:
```csharp
int GetQualityTier() {
    int systemRam = SystemInfo.systemMemorySize; // MB
    string gpuName = SystemInfo.graphicsDeviceName;
    
    if (systemRam >= 6000 && gpuName.Contains("Adreno 660")) return 2; // High
    if (systemRam >= 4000) return 1; // Mid
    return 0; // Low
}
```

**Frame Rate Monitor**:
```csharp
int frameCount = 0;
float totalTime = 0;
float targetFPS = 60;

void Update() {
    totalTime += Time.deltaTime;
    frameCount++;
    if (frameCount >= 60) {
        float currentFPS = frameCount / totalTime;
        if (currentFPS < targetFPS * 0.9f) {
            ReduceQuality();
        }
        frameCount = 0;
        totalTime = 0;
    }
}
```

**Analytics Integration**: Log device model + tier chosen (Firebase Analytics, or custom), post-launch analyze (which devices buy game = update database).

## Related Topics

- [24.1 Mobile GPU Architecture](24-01-Mobile-GPU-Architecture.md) - Hardware differences
- [24.2 Mobile Optimization](24-02-Mobile-Optimization.md) - Optimization techniques
- [24.4 Mobile-Specific Features](24-04-Mobile-Specific-Features.md) - Platform features
- [14.1 Profiling Fundamentals](04-01-PC-Profiling-Tools.md) - Measurement techniques

---

[Previous: 24.2 Mobile Optimization](24-02-Mobile-Optimization.md) | [Next: 24.4 Mobile-Specific Features →](24-04-Mobile-Specific-Features.md)
