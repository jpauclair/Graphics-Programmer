# 21.2 Console Frame Pacing

[← Back to Chapter 21](../chapters/21-Frame-Management-And-Synchronization.md) | [Main Index](../README.md)

Console frame pacing targets platform-specific frame rates: PS4/One = 30/60 FPS choices, PS5/Series X = 60/120 FPS capability, Switch = 30 FPS standard. Hardware constraints (thermal limits, power budgets, target HDMI standard = 30/60 Hz) drive timing. Goal: consistent FPS on fixed hardware, no framerate variation.

---

## Overview

Console Targets: PS4/Xbox One (4-7 year-old hardware, 2013-2016): 1080p/30 FPS or 900p/60 FPS (quality vs speed trade-off), PS5/Xbox Series X (2020 hardware, powerful): 1440p/60 FPS or 4K/30 FPS (cinematic modes) or 1080p/120 FPS (performance modes, newer displays), Switch (2017 portable): 1280x720 docked / 1280x720 handheld at 30 FPS standard (thermal limits, no high FPS viable). Frame rate selection impacts quality (higher FPS = lower settings, more aggressive optimization), development cost (supporting multiple modes = multiple content targets).

Thermal Management: Console hardware temperatures (GPU/CPU throttle if >80°C to prevent damage). Sustained high FPS = sustained heat output (PS5 running 120 FPS all cores = significant heat), thermal limit drops clock speed (performance degrades, frame rate drops if temp exceeds threshold). Solution: dynamic frame rate (target 60 FPS, if thermal throttle detected, cap to 45 FPS until cooling recovers, or switch to 30 FPS mode), or sustained optimization (profile thermal load, optimize to keep below 75°C limit, allow sustained 60 FPS without throttle).

Frame Rate Modes: Performance (60/120 FPS, lower quality settings, no ray-tracing or aggressive simplification), Quality (30 FPS, high settings, ray-tracing, complex effects), Balanced (40-50 FPS compromise, subset of quality features). Player choice (game offers modes, player picks preference = latency vs visuals). Implementation: Application.targetFrameRate (set per mode), Quality Settings (LOD, shadow res, post-processing, swap per mode), shader variants (different shaders per quality = different costs).

## Key Concepts

- **PS4/Xbox One Frame Rate Modes**: 30 FPS or 60 FPS choice (not both simultaneously). 30 FPS = cinematic, maximum quality (detailed geometry, high shadow resolution, complex shaders, ray-tracing if supported), 1440p/1600p resolution typical. 60 FPS = performance (simpler geometry, lower shadow res, no ray-tracing, aggressive LOD), 1080p/900p resolution. Development: ship both modes (requires two quality targets = double optimization effort), or single mode (simplifies development, limits player choice). Frame time budget: 30 FPS = 33.33ms (more time to render complex scene), 60 FPS = 16.67ms (half time, must simplify).
- **PS5/Xbox Series X Capability**: Native 60 FPS standard (faster hardware = 60 FPS achievable with reasonable quality). 120 FPS mode: 1440p/120 or 1080p/120 (requires aggressive optimization), or backwards compatible PS4/Xbox One titles at 120 FPS (upscaled, simplified). Quality modes: 4K/30 FPS (cinematic, maximum visual fidelity), 1440p/60 FPS (balanced), 1080p/120 FPS (performance, extreme optimization). Supported by newer displays (HDMI 2.1, 120 Hz support, PlayStation 5 Pro, Xbox Series X).
- **Switch Thermal Constraints**: Fixed 30 FPS (thermal limit prevents higher, sustained CPU/GPU load = heat buildup in portable form factor, 3.5" device = poor heat dissipation). Docked (external cooling = better sustained thermal, can maintain 30 FPS longer), handheld (internal temperature rises faster, thermal throttle activates sooner = handheld sessions shorter before throttle). Optimization: target 30 FPS without thermal throttle (profile on handheld mode, test sustained play 30+ minutes, verify no throttle), or accept throttle (30 FPS → 20 FPS if overheating, plan for graceful degradation).
- **Frame Rate Mode Selection**: Quality (player preference: fidelity or responsiveness). UX: pause menu or settings > Video > Frame Rate Mode (30/60/120), or auto-detect (system recommends based on display capability). Implementation: store preference (PlayerPrefs), on level load set Application.targetFrameRate, Quality Settings (LOD bias, shadow distance, post-process scale). Shaders: compile shader variants per quality (Quality=0: high-res shadows, ray-tracing, Quality=1: medium shadows, no RT, Quality=2: low shadows, simple), reduce compile time by supporting only shipped variants (disable unused variants in project settings).
- **Dynamic Frame Rate**: Auto-adjust FPS based on load (no static mode, frame rate varies 30-60 or 40-120 based on scene complexity). Method: monitor frame time every frame, if exceeding budget (e.g., target 60 FPS, frame = 18ms = within budget = maintain 60, frame = 20ms = exceed = throttle to 45 FPS), adjust LOD/shadow resolution dynamically to maintain target. Benefit: never drops below target FPS (smooth experience, no stutters), maintains player choice (prefer responsive = 60, prefer fidelity = 30). Drawback: requires sophisticated profiling (know which setting impacts which system), risk of oscillation (rapidly switching modes = distracting visual change).

## Best Practices

**Frame Rate Mode Support:**
- Minimum: support two modes (30 FPS quality, 60 FPS performance) for current-gen consoles (PS5/Series X). Entry: Quality mode mandatory (ships with game), Performance mode optional but recommended (players expect choice).
- Implementation: Quality Settings asset (Assets > Create > Quality Settings), create multiple profiles (Quality_30fps: shadow distance 100, post-process resolution 100%, LOD bias 1; Performance_60fps: shadow distance 50, post-process resolution 75%, LOD bias 2). On startup or settings change, QualitySettings.SetQualityLevel(index) switches profile.
- Shader variants: use shader keywords (multi_compile, shader_feature) to compile variants per quality, or single shader with conditionals (if QUALITY_MODE_30 use high-res shadows, else use low-res). Avoid excessive variants (10+ variants = slow shader compilation, test on lowest-end hardware = slow compile times).

**Thermal Management:**
- Profile thermal load: run game 30+ minutes on target device (console or dev kit), monitor temperature (built-in dev tools, Profiler if available), peak thermal (should stay <75°C for sustained operation, <85°C peak acceptable). If exceeds, optimize (reduce draw calls, simplify shaders, lower shadow resolution) until within thermal budget.
- Graceful degradation: if thermal throttle detected (GPU clock drops, frame time increases), lower target frame rate (60 → 45 FPS) or reduce quality settings dynamically (disable post-processing, reduce shadow distance, lower geometry LOD). Recovery: once cooled, restore settings. Communicate to player (subtle HUD message: "Running hot, reducing quality to prevent throttle"), avoid abrupt visual change.
- Switch handheld: test in handheld mode specifically (docked runs cooler = different thermal profile), verify 30 FPS sustainable 30+ minutes without throttle. Accept minor dips (brief 28 FPS acceptable, sustained <25 FPS = problem).

**Development Strategy:**
- Single frame rate (30 FPS) simplifies development (one target, one quality level, no mode switching code), ship time reduced, risk lower. Acceptable for narrative/cinematic games (action games demand responsive input = benefit from 60 FPS more). Trade-off: players perceive 30 FPS as slower/less responsive (even cinematic games benefit from 60 FPS perception).
- Multiple modes: adds development cost (two quality levels, shader variants, separate optimization passes), but increases appeal (players choose, game looks next-gen at 60 FPS). Invest if competitive (shooters, action = demand 60 FPS), or niche title (story-driven = 30 FPS acceptable).
- Backwards compatibility: older console (PS4) runs on newer (PS5) at 60 FPS (console handles upscale/boost clock automatically), no additional work required. Can enhance with PS5 patch (better shaders, higher res, still 60 FPS).

**Platform-Specific:**
- **PS4/Xbox One**: target 30 or 60, not both simultaneously (architectural limits, no way to dynamically switch on these consoles as smoothly as PS5). Choose one (30 = cinematic, 60 = competitive). Frame budget: 30 FPS = 33.33ms (7-10ms overhead = ~23ms rendering available), 60 FPS = 16.67ms (5ms overhead = ~11.67ms rendering available).
- **PS5/Xbox Series X**: support 60 FPS standard (hardware capable), optional 120 FPS for high-end displays (requires aggressive optimization). Quality mode 30 FPS acceptable (backwards compatibility, enhanced visuals). Thermal headroom larger than previous gen (better cooling), sustained 60 FPS viable without throttle.
- **Switch**: fixed 30 FPS docked and handheld (no higher viable). Thermal management critical (handheld throttles sooner, accept graceful degradation). Memory/storage tight (cartridge = smaller than disc, all optimization applies).

## Common Pitfalls

**Thermal Throttle Ignored**: Developer ships 60 FPS game on Switch (aggressive optimization passes PC/console, assumes Switch can handle), on-device thermal testing skipped. In the wild, handheld overheats after 30 minutes (throttles to 15 FPS, unplayable). Symptom: user complaints (game unplayable after brief play, overheating). Solution: profile on handheld (sustained play test, 60+ minutes, monitor thermal), ensure <75°C, or reduce target to 30 FPS stable.

**Frame Rate Mode Optimization Incomplete**: Developer ships 60 FPS mode (but same Quality Settings as 30 FPS mode, just frame rate increased), 60 FPS mode runs out of frame budget (lags at 16.67ms = dips to 45 FPS regularly). Symptom: performance mode performs worse (unstable frame rate, defeats purpose). Solution: adjust Quality Settings per mode (Performance_60fps uses lower LOD bias, shorter shadow distance, simpler shaders), test frame time in Profiler per mode.

**Shader Variant Explosion**: Developer compiles shader variants per quality and per platform (quality × platform = 8 variants), shader permutations explode (multi_compile with 3+ keywords = 2^3 = 8 variants, add more keywords = exponential growth). Build time = 1 hour+ (slow iteration). Symptom: slow builds, shader compilation hangs. Solution: use shader_feature instead of multi_compile (unused variants stripped at build, only shipped variants compiled), reduce keywords (combine related features into single keyword), or accept fewer variants (support only essential quality differences).

## Tools & Workflow

**Frame Rate Mode Setup**: Edit > Project Settings > Quality, duplicate Standard profile (create Quality_30fps and Performance_60fps). Quality_30fps: Shadows > Distance = 100, MSAA = 4x, LOD Bias = 1.0, Particle Raycast Hits = 32, Async Upload = enabled. Performance_60fps: Shadows > Distance = 50, MSAA = 2x, LOD Bias = 2.0, Particle Raycast Hits = 8, Async Upload = disabled. In code: set via QualitySettings.SetQualityLevel(index) based on player preference.

**Thermal Profiling**: Dev kit has built-in thermal monitoring (PlayStation DevKit Utilities, Xbox DevKit). Run game sustained (30+ minutes), monitor temperature (dashboard or console UI), peak thermal (note coolest point = docked mode, warmest = handheld sustained play). If >75°C, optimize (reduce shader complexity, lower draw calls, disable expensive features per mode).

**Shader Variant Compilation**: Use shader_feature for optional quality features:
```glsl
#pragma shader_feature QUALITY_MODE_30
#pragma shader_feature ENABLE_RT

void frag() {
    #if defined(QUALITY_MODE_30)
        // High-quality path
        float shadow = PCFShadow(shadowCoord);
    #else
        // Performance path
        float shadow = SimpleShadow(shadowCoord);
    #endif
}
```
Compile only used variants (strip unused via project settings: Graphics > Shader Stripping).

**Frame Rate Mode Detection**: Detect console capability:
```csharp
bool supports120Hz = SystemInfo.graphicsDeviceVersion.Contains("DX12") && 
    SystemInfo.graphicsMemorySize > 10000; // PS5/Series X only
if (supports120Hz) {
    // Offer 30/60/120 modes
} else {
    // Offer 30/60 modes only
}
```

## Related Topics

- [21.1 Frame Pacing](21-01-Frame-Pacing.md) - General frame timing
- [06.2 GPU Architecture](02-03-GPU-Components.md) - Console GPU specs
- [14.1 Profiling Fundamentals](04-01-PC-Profiling-Tools.md) - Thermal profiling
- [25.1 Build Pipeline](25-01-Build-Optimization.md) - Console builds

---

[Previous: 21.1 Frame Pacing](21-01-Frame-Pacing.md) | [Next: 21.4 GPU Synchronization →](21-04-GPU-Synchronization.md)
