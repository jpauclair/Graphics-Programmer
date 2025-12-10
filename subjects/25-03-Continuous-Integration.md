# 25.3 Continuous Integration & Automated Testing

[← Back to Chapter 25](../chapters/25-Build-Pipeline-And-Workflow.md) | [Main Index](../README.md)

Continuous Integration (CI) = automated build/test on every commit (catches regressions early, prevents broken builds on main). Pipeline: (1) pull latest, (2) build project, (3) run tests (unit, graphics, performance), (4) report results. Benefits: team confidence (main = always playable), early bug detection (regression caught within hours), quality metrics (track build times, test pass rates). Graphics-specific tests: visual regression (screenshot compare = detect visual bugs), performance regression (frame time target = ensure <60ms on target platform).

---

## Overview

CI Server: Jenkins (self-hosted), GitLab CI (integrated), GitHub Actions (integrated), or cloud (Unity Cloud Build, Incredibuild). Process: developer commits, webhook triggers build server, build job runs (compile, test, report). Feedback: email/Slack notification = build success/failure. Dashboard = visible (team sees status = accountability). Graphics workflow: build for PC/mobile/console platforms (each needs different optimization, build times vary = 10-60 min per platform).

Test Types: (1) unit tests (C# logic, not visual), (2) integration tests (systems work together), (3) performance tests (frame time <target), (4) visual regression (screenshot diff = detect visual bugs).

## Key Concepts

- **Build Matrix**: test multiple platforms (PC, Android, iOS, console = 4 builds). Each takes 30-60 min = total 2-4 hours. Parallel execution (4 machines = concurrent builds = 1 hour total). CI server manages job distribution.
- **Visual Regression Testing**: render scene, capture screenshot (reference image stored), new build = render same scene, compare screenshots (pixel difference = detect visual bugs). Tools: PixelPlough (paid), custom (diff algorithm = flag changes >threshold). Benefit: catches rendering bugs (shader error, texture missing = visual change detected).
- **Performance Regression Testing**: measure frame time (target <33ms @ 30 FPS), if new build exceeds = fail (performance regressed). Store baseline (10 frames average), new builds = measure and compare. Threshold = allow 5% variance (measurement noise).
- **Build Notifications**: Slack integration ("Build #1234 failed - see log"), email, or dashboard. On failure = developer responsibility to fix immediately (don't let broken builds linger).
- **Caching**: artifact caching (compiled objects, cached shaders = stored, next build reuses = faster). Unity Cloud Build = managed caching. Self-hosted = configure cache server (speeds up incremental builds).

## Best Practices

**Pipeline Configuration**:
- Stages: build → test → report (each stage fails if thresholds exceeded). Build fails = project doesn't link, test fails = regressions detected, report = visibility.
- Notifications: fast feedback (within 30 min of commit). Developer still in context = quick fix.
- Scheduled builds: nightly full builds (long), commit builds fast subset (smoke tests).

**Performance Thresholds**:
- Target: 60 FPS PC = 16.67ms frame time. Fail if >20ms (allow 20% variance for measurement noise).
- Mobile: 30 FPS target = 33.33ms. Fail if >40ms.
- Screenshot compare: allow 1% pixel difference (measurement noise from anti-aliasing).

**Platform Coverage**:
- Minimum: PC (quick iteration), Android (mobile market), iOS (quality gate). If console team = add console builds.
- Parallel execution: if 4 machines available = 4 builds parallel = finish in 1 hour vs 4 hours serial.

**Test Strategy**:
- Fast smoke tests: every commit (critical path only = 10 min builds).
- Full tests: nightly (comprehensive = 2+ hours = too slow for every commit).
- Graphics tests: weekly (full scene complexity = expensive, not every build).

## Common Pitfalls

**Flaky Tests**: Test passes sometimes, fails sometimes (race condition, timing dependent = random). Symptom: builds marked failed randomly, no code change = frustrating. Solution: deterministic tests (no timing deps), retry failed tests (some flakiness acceptable if retry succeeds).

**Slow CI Pipeline**: CI slow (2+ hours per commit), developer commits waiting = iteration slow. Symptom: commits back up, developers frustrated. Solution: parallelize (4 machines = 1/4 time), or split into fast/slow (commit = fast smoke tests 10 min, nightly = full suite 2 hours).

**Missing Coverage**: CI doesn't test critical path (only builds, doesn't test rendering). Update breaks visuals = CI passes = shipped broken. Symptom: visual bugs reach players. Solution: add visual regression tests (screenshot compare).

## Tools & Workflow

**GitHub Actions Example** (simple CI):
```yaml
name: Build
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: game-ci/unity-builder@v2
        with:
          targetPlatform: StandaloneLinux64
      - uses: actions/upload-artifact@v2
        with:
          name: build
          path: build/
```

**Performance Test** (frame time check):
```csharp
public void TestFrameTime() {
    float totalTime = 0f;
    for (int i = 0; i < 60; i++) {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        renderer.Render();
        totalTime += sw.Elapsed.TotalMilliseconds;
    }
    float avgFrameTime = totalTime / 60;
    Assert.IsTrue(avgFrameTime < 16.67f, $"Frame time {avgFrameTime}ms exceeds 60 FPS target");
}
```

**Visual Regression Workflow**:
```csharp
if (isReferenceGeneration) {
    Texture2D ref = RenderScene();
    SaveScreenshot("reference.png", ref);
} else {
    Texture2D current = RenderScene();
    float diff = CompareScreenshots("reference.png", current);
    Assert.IsTrue(diff < 0.01f, $"Visual diff {diff} exceeds 1% threshold");
}
```

## Related Topics

- [25.1 Build Optimization](25-01-Build-Optimization.md) - Build pipeline
- [25.2 Version Control](25-02-Version-Control.md) - Commit workflow
- [25.4 Asset Pipeline](25-04-Asset-Pipeline.md) - Asset processing
- [04.3 Unity Built-In Tools](04-03-Unity-Built-In-Tools.md) - Profiling

---

[Previous: 25.2 Version Control](25-02-Version-Control.md) | [Next: 25.4 Asset Pipeline →](25-04-Asset-Pipeline.md)
