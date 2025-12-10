# 29.2 Common Pitfalls

[← Back to Chapter 29](../chapters/29-Case-Studies-And-Best-Practices.md) | [Main Index](../README.md)

Production optimization learns through mistakes. Studios ship games, discover performance issues post-launch, and publish postmortems. This section catalogues common pitfalls: causes, symptoms, and solutions. Learning from others' mistakes accelerates optimization.

---

## Overview

Optimization pitfalls fall into categories: architectural (wrong choice early), measurement (optimizing wrong thing), and platform-specific (feature works on PC, breaks on console). Most pitfalls are discoverable via profiling. Some only appear at scale (network pressure, concurrent players).

Postmortems reveal patterns: subtitle rendering consuming 5% GPU (unexpected), physics stepping every frame instead of 60Hz (should be fixed), texture streaming predictive prefetch broken on spinning drives (SSD assumption).

## Key Concepts

- **Measurement Pitfall**: Optimizing average instead of percentile (p99 frame time). Average 20ms sounds good; p99 80ms = stutters.

- **Scaling Pitfall**: Feature works on dev PC (16GB RAM). Fails on console (10GB shared memory after OS). Must test on target hardware early.

- **Async Pitfall**: Streaming data async to avoid hitches. But GPU not ready for data (stall anyway). Poor pipelining.

- **Platform Assumption**: "It's fast on PC" ≠ "It's fast on console". CPU architecture differs (x86 vs custom), memory layout differs, GPU differs.

## Common Pitfalls (Detailed)

**Pitfall 1: Subtitle Rendering Performance**

*Symptom*: Frame rate drops 5-10% when dialogue heavy. Performance team baffled.

*Cause*: Subtitle text rendered via dynamic texture each frame. Text renderer generates texture, uploads to GPU, renders quad. If 10+ subtitles, 10+ texture uploads = bandwidth waste.

*Solution*: Pre-bake subtitle atlas at startup (1000 common phrases). Runtime: lookup glyph in atlas, render glyph quad (no texture upload). Impact: 0.2% GPU instead of 5%.

*Prevention*: Profile with subtitle-heavy scenes in development.

**Pitfall 2: Physics Ticking**

*Symptom*: Game feels responsive on developer PC (good frame rate), but console version feels sluggish (frame rate solid, but input lag).

*Cause*: Physics stepped every frame. If frame time varies (10ms to 30ms depending on scene), physics tick time varies. Results in jittery movement.

*Solution*: Fixed timestep physics (60Hz fixed, regardless of frame rate). Decouple rendering from simulation. Input samples at 1000Hz, but physics updates at 60Hz only. Feels responsive (input sampled fresh each frame) but stable (physics deterministic).

*Prevention*: Use engine defaults (Unity Physics set to 0.0166s fixed timestep = 60Hz).

**Pitfall 3: Memory Fragmentation**

*Symptom*: Game ships with 4GB VRAM budget. Level loads. Every allocation succeeds. But GPU stalls (memory fragmented). Texture can't allocate contiguous space even though total free > needed.

*Cause*: Long-lived allocations (lighting data, BVH trees) not freed between levels. Short-lived allocations (temp buffers) fragment heap.

*Solution*: Pooling. Pre-allocate pools for temp buffers, reuse instead of alloc/free each frame. Or buddy allocator (defragment at load screen, not runtime).

*Prevention*: Track allocation count and size histogram. Alert if heap fragmentation exceeds threshold.

**Pitfall 4: Streaming Prediction Cache Miss**

*Symptom*: Streaming works on PC SSD (NVMe 3GB/s). But production consoles sometimes stall (file request, wait 100ms for disk).

*Cause*: Prediction logic wrong. Game predicts "player moves right, load right area". Player turns around. Prediction miss. Must wait for "left area" to load.

*Solution*: Load nearest 3 areas (forward, left, right) not just forward. Increase prediction buffer (500ms lookahead instead of 200ms). Or use AI pathfinding: predict where player will go, not just direction facing.

*Prevention*: Telemetry: track prediction hits/misses. Monitor stall frequency per area.

**Pitfall 5: Dynamic Resolution Banding**

*Symptom*: Player sees visual pop: one frame 4K sharp, next frame 1080p soft.

*Cause*: Dynamic resolution changes every frame (trying to maintain frame rate). Eye perceives resolution jump.

*Solution*: Change resolution less frequently (every 2-3 frames, not every frame). Or use temporal jitter: output 4K on odd frames, 1080p on even frames. Temporal reconstruct = perceived quality boost. Or dither resolution (slightly worse quality, less pop).

*Prevention*: Test dynamic resolution with actual game content (busy scenes with UI). If banding visible, reduce change frequency.

**Pitfall 6: Mobile Battery Drain**

*Symptom*: 60fps game drains battery 2x faster than competitor's 30fps game.

*Cause*: GPU stays at max frequency entire game. Battery impact non-linear with frequency (power ∝ freq^2). 60fps = higher avg freq = battery death.

*Solution*: 60fps optional. Default 30fps. Or 60fps with aggressive power-save during idle (lower freq if no input 2+ seconds).

*Prevention*: Use platform profilers (Android Battery Historian, iOS Xcode Energy gauge). Set power budget per game feature.

**Pitfall 7: Unsynchronized CPU-GPU**

*Symptom*: GPU-side optimization (shader reduction) complete, but frame time unchanged. CPU stalling waiting for GPU readback (GPU fence).

*Cause*: Synchronous GPU readback. Code: `glGetData(readback_buffer)` blocks CPU until GPU done. Common in profiling code left in shipping build.

*Solution*: Async readback. Query GPU timestamp, read from previous frame (not current frame). Or use persistent mapping (no blocking needed).

*Prevention*: Audit all GPU readback. Wrap in `#if DEVELOPMENT_BUILD` and remove for shipping.

**Pitfall 8: Platform-Specific Shader Compilation**

*Symptom*: Shader compiles on PC (DX11). Fails on console (DXR required). Or compiles but 50% slower (missing pragma optimization hint).

*Cause*: Platform-agnostic shader code. PC compiler ignores feature, console compiler rejects.

*Solution*: Conditional compilation. Example: `#if SHADER_TARGET >= 65` for DXR features. Or use shader compiler directives: `#pragma optimize("off")` for console debug builds.

*Prevention*: Shader compilation tests per platform in CI. Fail build if shader fails to compile on any platform.

## Tools & Workflow

**Postmortem Template**: When issue ships, postmortem document: Symptom (what user saw), Cause (root reason), Impact (how many players, revenue loss), Prevention (what would catch this earlier). Share with team.

**Regression Testing**: Automate performance regression tests. Standard benchmark scene, measure frame time. If regression >5%, alert developers.

## Related Topics

- [29.1 AAA Game Examples](29-01-AAA-Game-Examples.md) — Case studies
- [04 Profiling Tools](../chapters/04-Profiling-And-Analysis-Tools.md) — Tools to catch pitfalls
- [06 Bottleneck Identification](../chapters/06-Bottleneck-Identification.md) — Diagnosis

---

[← Previous: 29.1 AAA Game Examples](29-01-AAA-Game-Examples.md) | [Next: 29.3 Workflow Optimization →](29-03-Workflow-Optimization.md)
