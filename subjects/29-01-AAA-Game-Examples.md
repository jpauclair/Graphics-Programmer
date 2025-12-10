# 29.1 AAA Game Examples

[← Back to Chapter 29](../chapters/29-Case-Studies-And-Best-Practices.md) | [Main Index](../README.md)

AAA game studios optimize graphics through years of iteration, learning from shipped titles. This section shares real-world case studies from successful games, their optimization priorities by platform, performance targets, and specific techniques that drove results. These examples demonstrate how theory translates to production.

---

## Overview

Optimization is game-specific. A physics-heavy simulation prioritizes CPU budgeting differently than a graphics showcase. AAA examples: Elden Ring (complex geometry, outdoor streaming), Fortnite (mobile reach, consistent 60fps), God of War Ragnarök (PS5 visual fidelity), Call of Duty (60fps competitive requirement). Each faced unique bottlenecks.

Target metrics vary by genre: AAA single-player allows 60fps or 30fps depending on scope. Competitive multiplayer demands 120fps+ (esports requirement). Mobile expects 60fps with <2s load time. VR demands 90fps rock-solid. Understanding studio priorities reveals optimization strategy.

## Key Concepts

- **Genre-Specific Performance Targets**: Single-player action: 60fps target but 30fps acceptable if visual quality justifies. Competitive multiplayer: 120fps+ or lose esports viability. Battle royale: 60fps on console, flexible mobile.

- **Platform Strategy**: AAA titles optimize for lead platform first (e.g., PS5), then scale down (PC high-end, console mid-tier, mobile low-end). Not equal effort per platform.

- **Performance Budgets**: Frame time split by feature. Example: Elden Ring allocates 8ms CPU (game logic), 10ms GPU (rendering), 2ms overhead at 60fps (16.67ms total).

- **Visual Target**: Resolution and fidelity matter for marketing. 4K at 60fps on console = marketable. 1080p 60fps on mobile = competitive edge.

- **Load Time Optimization**: AAA games target <5s boot, <10s level load (excluding streaming). SSD speeds (PS5 5.5GB/s, Xbox Series X 2.4GB/s) enable instant load.

## Best Practices: Case Studies

**Case 1: Elden Ring (FromSoftware) - Outdoor Open World**

*Challenge*: Vast outdoor maps with complex architecture, 1000+ NPCs, dynamic lighting, frame rate consistency.

*Strategy*:
- Culling: Strict frustum, occlusion, per-object distance LOD. Outdoor scenes cull 90% of geometry per view.
- Streaming: Level chunks load/unload based on player position (2-chunk lookahead).
- Lighting: Prebaked lightmaps for static geometry. Dynamic lighting: shadows only for player and 2-3 NPCs.
- Animation: Skeletal LOD for distant NPCs (1 bone per 10m distance).

*Result*: 60fps on PS5 with complex visuals. Trade-off: lower resolution (dynamic 4K down to 1080p in heavy scenes).

*Performance Budget (60fps):
- CPU: 5ms game logic (AI, physics, animation) + 2ms rendering setup = 7ms
- GPU: 8ms geometry + 2ms lighting + 2ms effects = 12ms
- Headroom: 1ms for OS overhead

**Case 2: Fortnite (Epic) - Mobile Reach**

*Challenge*: Ship on 200+ Android devices (2GB RAM, Adreno 505 GPU) and iOS. Consistent 60fps critical for competitive play.

*Strategy*:
- Scalable rendering: High-end (PS5) = full detail. Mid-tier (PS4) = 1440p, reduced draw distance. Low-end (mobile) = 1080p, LOD0 only, no dynamic shadows.
- Draw calls: Aggressive batching, instancing for props (1000 trees = 1 draw call via hardware instancing).
- Memory: Streaming textures at 512x512 (vs 2K on console). Mesh simplification: mobile models 50% triangle count of console.
- Battery: Mobile target 30fps, not 60fps (battery impact 2x worse). Users opt-in to 60fps mode.

*Result*: Playable on $200 Android phones. Competitive esports on high-end (120fps).

*Performance Budget (Mobile 30fps at 1920x1080):
- CPU: 20ms game logic (lower tickrate AI)
- GPU: 15ms geometry + 1ms lighting (baked) = 16ms
- Total: 33ms per frame (one frame every other vsync)

**Case 3: God of War Ragnarök (Sony Santa Monica) - PS5 Visual Showcase**

*Challenge*: Push PS5 graphics (real-time ray tracing, 4K, 60fps). Demonstrate next-gen capability.

*Strategy*:
- Ray tracing: 1 bounce reflections on water/metal, ray-traced shadows (boss fights only, performance spikes OK). Denoising: temporal for quality.
- Dynamic resolution: Lock at 60fps, vary resolution (1800p to 2160p). Better than frame rate drops.
- Streaming: Seamless world with no loading screens (SSD enables preload next area in background).
- AI: Complex combat behaviors, multiple enemies. GPU-driven rendering for culling (100k objects, 50k visible).

*Result*: Visual benchmark for PS5. 60fps with ray tracing (achievement at time).

*Performance Budget (60fps):
- CPU: 6ms gameplay + 2ms streaming
- GPU: 8ms opaque pass + 2ms ray tracing + 2ms effects = 12ms
- Headroom: 2ms

**Case 4: Call of Duty (Infinity Ward) - Competitive Esports**

*Challenge*: 120fps on console (esports requirement), consistent frame time <8.3ms (no variance > 1 frame).

*Strategy*:
- CPU: Fixed update rate (60 ticks/second), deterministic AI (no variance per frame).
- GPU: Aggressive LOD (lower polys at distance), dynamic resolution (lock 120fps, vary res).
- Draw calls: 1000-2000 draw calls per frame, heavy batching.
- Network: Tickrate 60Hz, tick time = 16.67ms. Consistency matters more than quality.

*Result*: 120fps locked on Xbox Series X/PS5. No frame variance (esports-critical).

*Performance Budget (120fps = 8.33ms):
- CPU: 4ms game logic (fixed 60Hz update) + 1ms rendering = 5ms
- GPU: 6.5ms rendering
- Headroom: 0.8ms (tight budget)

## Platform-Specific Guidance

**PC**: High variance in hardware. Scalability essential (settings slider: Low/Medium/High/Ultra). Typical: 3 versions of shaders, 2-3 mesh LOD tiers. Allow uncapped frame rate (esports players want 240fps).

**Console**: Fixed hardware. One optimization target (PS5 or Xbox Series X). Optimize for that, then scale down to last-gen (PS4 30fps OK). Ray tracing: use judiciously (costs 20-40% frame time).

**Mobile**: Target 60fps on flagship, 30fps on low-end. Or tiered releases (high-end game vs lite version). Memory: aim for <2GB (leave room for OS/other apps).

**VR**: 90fps or nothing (motion sickness threshold). Mobile VR: 72fps acceptable. No dynamic resolution (VR sickness risk). Foveated rendering saves 40-50%.

## Common Pitfalls

**1. Optimizing Wrong Platform**
- *Symptom*: PC version locked at 60fps, but console version can't hit 60fps.
- *Cause*: Lead platform choice wrong (optimized for PC, scales down poorly to console).
- *Solution*: Optimize for most constrained platform first (usually console), then scale up PC.

**2. Visual Regression in Scaled Versions**
- *Symptom*: Mobile version so ugly, players refuse to play. Console version 40% less detailed, perception of "cheap port".
- *Cause*: Over-aggressive LOD, removing features instead of scaling intelligently.
- *Solution*: Maintain visual coherence. Scale via resolution, draw distance, not feature removal.

**3. Inconsistent Frame Time**
- *Symptom*: Average 60fps, but frame variance 30-80ms. Feels stuttery despite high average.
- *Cause*: GC stalls, streaming hitches, uneven work distribution per frame.
- *Solution*: Profile frame-time histogram, not average. Lock frame rate (dynamic resolution OK).

## Tools & Workflow

**Performance Benchmarking**: PIX/Nsight to profile. Setup regression testing: every build measures frame time on standard scene. Alert if p95 > target.

**QA**: Dedicated performance QA. Test on minimum-spec hardware. Use telemetry to catch real-world regressions post-ship.

## Related Topics

- [29.2 Common Pitfalls](29-02-Common-Pitfalls.md) — Production issues and solutions
- [04 Profiling Tools](../chapters/04-Profiling-And-Analysis-Tools.md) — Tools used in these case studies
- [05 Performance Optimization](../chapters/05-Performance-Optimization.md) — Techniques applied

---

[← Previous: 28.4 Analytics & Telemetry](28-04-Analytics-And-Telemetry.md) | [Next: 29.2 Common Pitfalls →](29-02-Common-Pitfalls.md)
