# 29.3 Workflow Optimization

[← Back to Chapter 29](../chapters/29-Case-Studies-And-Best-Practices.md) | [Main Index](../README.md)

Studio workflow determines optimization iteration speed. Fast iteration accelerates optimization: build changes, profile, identify bottleneck, optimize, re-profile, repeat. Slow workflow (5min builds, manual profiling) kills productivity. This section covers studio setup for performance optimization efficiency.

---

## Overview

Optimization ROI = (performance gain) / (time spent). Slow workflows kill ROI. Example: shader recompile 10 seconds + profiling 5 seconds + iteration = 1 hour for 2% optimization. Quick iteration (1 second recompile + 30 second profile = 30 minutes) doubles ROI.

Workflow optimization: reduced iteration time, parallel profiling (all developers profile during development), shared telemetry (all builds log performance), automated regression detection.

## Key Concepts

- **Iteration Loop**: Code change → Build → Profile → Identify bottleneck → Code change. Loop time < 5 minutes ideal.

- **Distributed Profiling**: Instead of one person profiling late in project, all developers profile during feature work. Catches performance regressions early.

- **Automated Testing**: Regression tests run on every build. If performance drops >5%, build fails. Forces performance-aware coding.

- **Platform Parity**: Optimizations target lead platform first (usually console). Scaling to other platforms quick and cheap.

## Best Practices

**Build Optimization:**

Target build time <5 minutes (incremental rebuild). Strategies:
- Shader compilation: offline (build-time not runtime). Use shader cache (DXC, FXC cache compiled bytecode).
- Parallel compilation: Use all CPU cores (`/MP` flag MSVC).
- Linking: Unity engine use incremental linking (avoid relinking entire binary if one .obj changes).
- Scriptable Render Pipeline: Skip shader recompilation if shader unchanged (Shader Variant Collection).

*Implementation (Unity):*
```csharp
// In ShaderVariantCollectionGenerator.cs
using (var variantCollection = new ShaderVariantCollection())
{
    var shader = Shader.Find("MyShader/Opaque");
    variantCollection.Add(new ShaderVariantCollection.ShaderVariant(shader, 
        PassType.Forward, "FEATURE_X"));
    variantCollection.Save("Assets/ShaderVariants.shadervariants");
}
// This precompiles variants at build-time, not runtime. Play mode startup: 0ms (vs 500ms)
```

**Local Profiling (Rapid Iteration):**

Set up profiling station: console dev kit + PC profiler running. Developer make code change, press Play in Unity, profiler captures frame. Result in 30 seconds.

Tools:
- PC: Nsight Graphics (NVIDIA) or RenderDoc (all vendors).
- Console: Built-in profilers (PIX for Xbox, Razor for PS5).
- Remote profiling: Setup so developer doesn't need physical console (remote debugging).

**Continuous Performance Integration:**

Setup CI pipeline:

```yaml
name: Performance Regression
on: [push, pull_request]
jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Game
        run: unity --batchmode -buildTarget Windows64 -executeMethod BuildScript.Build
      - name: Run Benchmark
        run: ./Builds/Game.exe -benchmark -resultsFile results.json
      - name: Check Regression
        run: python3 check_regression.py results.json
        # Fail if p95_frame_time > 17ms (60fps target, 5ms headroom)
```

**check_regression.py:**
```python
import json

with open('results.json') as f:
    data = json.load(f)
    
p95_frame_time = sorted(data['frame_times'])[int(len(data['frame_times']) * 0.95)]
p50_frame_time = sorted(data['frame_times'])[int(len(data['frame_times']) * 0.50)]

if p95_frame_time > 17.0:  # 60fps, 5ms headroom
    print(f"FAIL: p95 frame time {p95_frame_time}ms > 17ms")
    exit(1)
else:
    print(f"PASS: p95 {p95_frame_time}ms, p50 {p50_frame_time}ms")
    exit(0)
```

**QA Performance Testing:**

Dedicated QA role: performance QA (not general gameplay QA). Runs every build through profiler, documents regressions, assigns to developer.

Setup:
- Minimum-spec hardware test rig (oldest target platform).
- Telemetry dashboard: QA checks daily. If crash rate spike, alert engineering.
- Load tests: simulate 100+ players on multiplayer server. Monitor CPU/frame time under load.

**Version Control & Performance:**

- Every commit: include performance impact comment. Example: "Optimized cloth physics, p95 frame time -2ms".
- Perf branch: experimental optimizations tested before merge to main.
- Revert policy: if perf regression > 10%, auto-revert after 24 hours (unless fix committed).

**Team Communication:**

- Weekly perf sync: 15min meeting. Review telemetry. Discuss regressions. Allocate optimization budget.
- Perf budget spreadsheet: "8ms GPU rendering" per feature. When new feature proposed, estimate GPU cost. If exceeds budget, redesign or remove lower-priority feature.

## Platform-Specific Workflow

**Console Development:**
- Dev kit in office, remote-accessed by team. Enable profiling on remote hardware.
- Certification requirements: Studio must pass performance gate before submission (e.g., "no frame drops >2ms over 5-minute gameplay").

**PC Development:**
- Scalability testing: test on high-end (RTX 4090) and low-end (GTX 960) hardware. Scales 3-5x between hardware tiers.
- DirectX debug layer: always enable during development (catches GPU errors, performance warnings).

**Mobile Development:**
- Device farm: various phones tested per build (iPhone 13, 14, 15; Galaxy S20, S21, S22). Automated tests (frame time capture) on each.
- Battery monitoring: measure battery drain on standardized test (3 minutes gameplay). Alert if battery drain increases.

## Common Pitfalls

**1. Late Performance Focus**
- *Symptom*: Game feature-complete 6 months before release. Optimization crunch last 2 months.
- *Cause*: Performance treated as post-feature work, not concurrent.
- *Solution*: Performance budget from day 1. Features must meet perf budget or be redesigned.

**2. Optimization Without Measurement**
- *Symptom*: Developer "optimizes" shader. Frame time unchanged (or worse).
- *Cause*: Optimized wrong bottleneck (CPU instead of GPU, or wrong GPU bottleneck).
- *Solution*: Always profile before optimizing. "Measure, don't guess."

**3. Single-Platform Optimization**
- *Symptom*: PC version 120fps. Console version 45fps (optimization didn't scale down).
- *Cause*: Optimized for high-end only. Forgot to test on target platform.
- *Solution*: Test on all platforms during development, not end.

## Tools & Workflow

**Performance Dashboard**: Grafana/Kibana pulling telemetry data. Team checks daily. Alerts if regression detected.

**Optimization Tracking**: GitHub issues for performance work. Link to commits, profiling traces. Track time-to-fix.

## Related Topics

- [29.1 AAA Game Examples](29-01-AAA-Game-Examples.md) — Studio case studies
- [28 Instrumentation](../chapters/28-Instrumentation-And-Telemetry.md) — Measurement tools
- [04 Profiling Tools](../chapters/04-Profiling-And-Analysis-Tools.md) — Tools referenced

---

[← Previous: 29.2 Common Pitfalls](29-02-Common-Pitfalls.md) | [Next: 29.4 Platform Certification Requirements →](29-04-Platform-Certification-Requirements.md)
