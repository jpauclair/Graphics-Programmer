# 14.4 Real-Time Global Illumination

[← Back to Chapter 14](../chapters/14-Global-Illumination-And-Lightmapping.md) | [Main Index](../README.md)

Real-time global illumination (RTGI) enables dynamic lighting changes at runtime through Enlighten's Precomputed Realtime GI or modern ray-traced techniques, with significant performance tradeoffs.

---

## Overview

Real-time GI allows runtime lighting changes: light intensity/color adjustments (sun at different angles, day/night transitions), emissive material changes (glowing surfaces turn on/off), dynamic light sources (moving lights affect indirect lighting). Unlike baked lightmaps (static, unchangeable), RTGI recalculates indirect lighting every frame or periodically. Cost: CPU/GPU overhead (GI updates expensive), memory overhead (GI data structures larger than baked lightmaps), quality tradeoffs (realtime GI lower quality than baked for equivalent performance).

Enlighten Precomputed Realtime GI: Unity's legacy RTGI system (deprecated Unity 2022+), precomputes light transport (how light bounces between surfaces), runtime modulates intensity/color (applies current light parameters to precomputed transport). CPU-based updates (milliseconds per frame), supports: intensity changes (light brightness), color changes (light color/temperature), emissive changes (material emission on/off). Limitations: slow baking (hours for complex scenes), high memory (larger than baked lightmaps), static geometry only (moving objects don't affect GI).

Modern RTGI techniques: Hardware ray-traced GI (DXR/RTX, Unity HDRP only), screen-space GI (SSGI, approximates bounces from visible surfaces), signed distance field GI (Lumen-style, UE5 technique). Ray-traced GI: highest quality (accurate bounces, dynamic geometry), most expensive (requires RTX GPU, heavy performance cost). SSGI: moderate quality (only visible surfaces contribute), moderate cost (screen-space effects, limited range). SDF GI: balanced quality/performance (good approximations, requires preprocessing). Unity support: ray-traced GI (HDRP with DXR), basic SSGI (post-processing stack), no SDF GI (custom implementation required).

## Key Concepts

- **Enlighten Precomputed Realtime GI**: Legacy system (Unity 5-2021). Precomputes light transport (radiosity, how light bounces), stores as clusters/surface data. Runtime: CPU updates GI based on light intensity/color (applies to precomputed transport, generates indirect lighting). Updates: per frame (expensive) or periodic (every N frames, amortized cost). Deprecated (removed Unity 2022+, modern approach = bake multiple lighting scenarios).
- **Light Transport**: Mathematical representation of how light propagates (surface to surface bounces). Precomputed offline (baking process, independent of light intensity/color), modulated at runtime (multiply by current light parameters). Allows: changing light brightness (10x brighter = 10x brighter indirect), changing color (red light = red indirect bounces). Does NOT allow: moving geometry (transport assumes static surfaces).
- **Ray-Traced GI**: Hardware ray tracing (DXR, RTX cores). Shoots rays from surfaces (samples environment, accumulates lighting), bounces through scene (accurate indirect lighting). Fully dynamic (geometry can move, lights can change), highest quality (physically accurate). Requires: RTX GPU (Nvidia 20 series+, AMD RDNA2+), HDRP pipeline (not URP), significant performance cost (30-50% frame time). Unity HDRP: Ray-Traced Global Illumination setting (experimental, high-end only).
- **Screen-Space Global Illumination (SSGI)**: Approximates GI using depth buffer and screen colors. Samples nearby pixels (ray marches in screen space, accumulates indirect lighting from visible surfaces). Limitations: only screen-visible surfaces contribute (off-screen = no GI), limited range (short ray distances), temporal noise (requires denoising). Moderate cost (post-process pass, several ms). Unity: custom SSGI in post-processing (not built-in, asset store solutions available).
- **Update Frequency**: RTGI updates per frame (full quality, expensive) or amortized (every N frames, cheaper + temporal lag). Enlighten: configurable update rate (1 = every frame, 4 = every 4 frames), lower rate = better performance + visible lag (lighting changes slowly). Ray-traced GI: per-frame or temporally accumulated (multiple frames blend, reduces noise + increases latency). Balance: responsiveness (immediate updates) vs performance (amortized updates).

## Best Practices

**Enlighten Setup (Legacy):**
- Enable Realtime GI: Lighting window > Scene > Realtime Global Illumination (checkbox, Unity 2019-2021 only). Precomputed Realtime GI bakes with lightmaps (generates transport data). Bake time: hours for large scenes (complex radiosity computation). Memory: 2-5x larger than baked lightmaps (transport data + clusters).
- Realtime resolution: Lighting window > Realtime Resolution (texels per unit, lower than baked). Typical: 1-2 (low resolution GI, fast updates), higher = 4-8 (detailed, slow updates). Low resolution acceptable (indirect lighting low-frequency, blurry by nature). Balance: detail (higher resolution) vs performance (update cost scales with resolution).
- Light settings: Directional light > Mode = Realtime or Mixed. Realtime = fully dynamic GI (expensive, updates every frame). Mixed = baked indirect + realtime direct (cheaper, indirect precomputed). Point/spot lights: Mode = Realtime (if need dynamic GI), Baked (if static, no runtime GI). Minimize realtime GI lights (each adds update cost).
- Update rate: Lighting window > Realtime Global Illumination > Realtime GI CPU Budget (milliseconds per frame). Unity spreads updates (if exceeds budget, updates over multiple frames). Increase budget = faster updates + higher cost, decrease = slower updates + amortized. Typical: 1-2ms (acceptable for 60fps, ~3% frame time).

**Modern RTGI Alternatives:**
- Baked lighting scenarios: Precompute multiple lighting setups (daytime, night, dusk = separate lightmap sets), swap at runtime (load different LightingSettings asset). Zero runtime GI cost (all lighting baked), supports any number of scenarios (limited by disk/memory). Swap time: loading delay (1-5 seconds to load new lightmaps), or seamless (crossfade between scenarios via script).
- Ray-traced GI (HDRP): HDRP project > Volume > Add Override > Ray-Traced Global Illumination. Requires: DXR-capable GPU, HDRP 10.0+, Ray Tracing enabled (Pipeline Asset > Rendering > Ray Tracing Supported). Settings: Ray Length (GI ray distance, shorter = faster), Sample Count (rays per pixel, more = higher quality + slower), Denoising (temporal accumulation, reduces noise). Cost: 10-30ms per frame (high-end GPUs only, 30-40fps with RTGI enabled).
- Hybrid approach: Baked GI for static environment (buildings, terrain, static props), realtime GI for key dynamic elements (emissive materials, movable lights in specific areas). Example: static world baked, player's flashlight uses localized RTGI (small area, limited performance impact). Minimizes cost (most GI baked, only critical elements dynamic).
- Temporal accumulation: Spread GI updates across frames (update 1/4 scene per frame, 4-frame cycle). Reduces per-frame cost (1/4 update cost), introduces latency (lighting changes lag 4 frames). Acceptable for: slow changes (day/night over minutes), background lighting (player doesn't notice lag). Avoid for: fast changes (lights flashing), player-controlled lights (flashlight feels unresponsive).

**Performance Optimization:**
- Limit realtime GI lights: Enlighten cost scales with light count (each realtime GI light = separate GI update). Limit to: 1-3 key lights (directional sun, primary indoor light), mark others Baked (no runtime GI). Overhead: 0.5-2ms per realtime GI light (depends on resolution, scene complexity). More than 5 lights = 5-10ms overhead (unacceptable for 60fps).
- Lower realtime resolution: Reduce Realtime Resolution (2 → 1 → 0.5). Lower = blockier GI + much faster updates. GI is low-frequency (blurry bounces), resolution 0.5-1 often acceptable (player doesn't notice blockiness in indirect lighting). Compare: resolution 2 = 5ms update, resolution 0.5 = 0.5ms (10x faster).
- Emissive materials optimization: Unity's Realtime GI samples emissive materials (glowing surfaces contribute to GI). Reduce emissive count (fewer emissive materials = less GI work). Use Emission GI flag (Material > Emission > Global Illumination > Realtime, Baked, or None). Realtime emissive expensive (updates GI on emission changes), Baked cheaper (emission precomputed), None = no GI contribution (emission visible but doesn't light environment).
- CPU budget tuning: Lighting window > Realtime GI CPU Budget (milliseconds per frame). Unity spreads work (if exceeds budget, updates over multiple frames). Low budget (1ms) = slower GI updates (multi-frame delay), high budget (5ms) = fast updates + frame drops. Balance: acceptable latency (2-3 frame delay okay) vs frame rate (maintain 60fps, limit to 1-2ms).

**Ray-Traced GI Configuration (HDRP):**
- Enable ray tracing: HDRP Asset > Rendering > Ray Tracing Supported (checkbox), Render Pipeline Resources (assign HDRP's default ray tracing resources). Project Settings > Player > Other Settings > Scripting Backend = IL2CPP (required for DXR). Verify: GPU supports DXR (Nvidia RTX, AMD RDNA2+).
- RTGI volume settings: Volume profile > Add Override > Ray-Traced Global Illumination. Enable (checkbox), Ray Length (10-50m, longer = more distant bounces + slower), Sample Count (1-4, more = less noise + slower), Denoising (enabled, temporal filter reduces noise). Start low (Ray Length 10, Samples 1), increase until quality acceptable.
- Mixed with baked: Use RTGI selectively (specific materials/objects marked for ray tracing, rest use baked). Material > Ray Tracing Mode > Enabled (this material uses RTGI), Disabled (uses baked lightmaps). Hybrid: hero objects RTGI (high quality, dynamic), background baked (static, cheap). Reduces cost (only fraction of scene ray traced).
- Performance targets: RTGI expensive (10-30ms, depends on resolution/settings). Target: high-end PC only (RTX 3070+, 1440p 60fps possible), not consoles (limited ray tracing support, performance insufficient). Alternative: lower resolution (50-75% render scale), or 30fps target (acceptable for cinematic games). RTGI not viable for: competitive games (60fps+ required), broad PC audience (many lack RTX), consoles (insufficient performance).

**Platform-Specific:**
- **PC**: Enlighten Realtime GI viable (2-5ms cost, acceptable on mid-range CPUs). Ray-traced GI on high-end (RTX 3070+, HDRP). Baked scenarios best for broad compatibility (works on all PCs, zero runtime cost). Hybrid: baked main lighting + limited RTGI for hero elements.
- **Consoles**: Enlighten expensive (CPU-limited, 5-10ms on console CPUs). Ray tracing limited (PS5/Xbox Series X support, but costly = 20-30ms). Recommendation: baked lighting scenarios (multiple lightmap sets, runtime swap). Realtime GI not worth cost (frame time budget tight, baked alternatives sufficient).
- **Switch**: No realtime GI (CPU too weak, GPU lacks ray tracing). Baked lighting only (static lightmaps, zero runtime cost). Multiple scenarios (day/night/dusk baked separately, swap as needed). Enlighten not viable (bake times excessive, runtime cost prohibitive).
- **Mobile**: No realtime GI (CPU/GPU too weak). Baked lighting only. Single lighting scenario typical (multiple scenarios = memory cost, mobile storage limited). Avoid: any runtime GI (performance unacceptable), stick to fully baked static lighting.

## Common Pitfalls

**Using Enlighten for Performance-Critical Games**: Developer enables Enlighten Realtime GI for 60fps action game. GI updates cost 5-10ms per frame (reduces frame rate to 40-50fps). Symptom: Frame rate drops significantly (was 60fps with baked lighting, now 40fps with realtime GI), CPU-bound (Profiler shows GI update time dominating). Solution: Disable Realtime GI (use baked lightmaps), or bake multiple lighting scenarios (day/night as separate lightmap sets, swap at load time). Realtime GI not worth cost (visual benefit small, performance cost massive).

**Too Many Realtime GI Lights**: Developer marks all lights as Realtime for GI (10+ directional/point/spot lights). GI update cost explodes (10 lights × 2ms = 20ms per frame). Frame rate tanks. Symptom: Profiler shows huge GI update time, frame rate <30fps. Solution: Mark only key lights as Realtime (1-2 lights: sun + primary indoor light), rest Baked. Each Realtime GI light adds cost (limit to essential lights only).

**Ray-Traced GI Without Denoising**: Developer enables HDRP ray-traced GI with 1 sample per pixel, no denoising (wants maximum performance). Result: extremely noisy lighting (speckled, flickering pixels, unusable). Symptom: GI looks like TV static, flickering every frame. Solution: Enable denoising (Ray-Traced GI volume > Denoising = On, Temporal Filter = enabled), increase samples (2-4 samples per pixel). Ray tracing requires denoising (low sample counts noisy, temporal accumulation smooths over frames). Acceptable: 1-2 samples + denoising (5-10ms, clean result).

**Enlighten with Dynamic Geometry**: Developer uses Enlighten Realtime GI, moves static geometry at runtime (thinking GI updates dynamically). GI doesn't update (light transport precomputed for static positions, moving objects = incorrect bounces). Symptom: Lighting looks wrong (bounces from old positions, doesn't match moved geometry). Solution: Don't move Contribute GI objects (mark dynamic objects as non-static, exclude from GI). Enlighten requires static geometry (transport precomputed, can't handle moving surfaces). For dynamic geometry: use ray-traced GI (HDRP, expensive) or light probes (approximate GI for dynamic objects).

## Tools & Workflow

**Lighting Window**: Window > Rendering > Lighting. Scene tab > Realtime Global Illumination (checkbox, enables Enlighten). Realtime Resolution (texels/unit), Realtime GI CPU Budget (milliseconds per frame). Auto Generate (automatic GI updates), Generate Lighting (manual bake). Progress bar shows precomputation progress (hours for complex scenes).

**HDRP Ray-Traced GI**: Volume profile > Add Override > Ray-Traced Global Illumination. Enable (checkbox), Ray Length (meters), Sample Count (1-4), Denoising (enabled/disabled), Temporal Accumulation (frames to blend). Frame Debugger shows ray-traced passes (G-buffer, ray tracing, denoising). Profiler > GPU > Ray-Traced GI (milliseconds per frame).

**Lightmap Scenarios**: Create multiple Lighting Settings assets (Assets > Create > Rendering > Lighting Settings). Bake scene with different assets (Lighting window > Scene > Lighting Settings = asset). Runtime: LightmapSettings.lightmaps = load different lightmap array (C# script loads textures, swaps lightmaps). Example: daytime.asset, night.asset (separate bakes, runtime swap).

**Enlighten Debugging**: Scene view > Shading > Realtime Global Illumination (visualizes realtime GI, shows clusters). Lighting > Debug Settings > Realtime GI Resolution (color overlay, shows realtime GI texel density). Profiler > CPU > Rendering > Enlighten (shows GI update time, per-light breakdown).

**Material GI Settings**: Material Inspector > Emission > Global Illumination dropdown (None/Realtime/Baked). None = emission doesn't affect GI (visible but no lighting), Realtime = emissive updates GI at runtime (expensive), Baked = emissive baked into lightmaps (static). Optimize: most emissive = Baked (one-time cost), only dynamic emissive = Realtime (e.g., lights turning on/off).

**Performance Profiling**: Unity Profiler > CPU > Rendering > Enlighten.Runtime (Enlighten update time), GPU Profiler > Ray-Traced GI (HDRP ray tracing cost). Frame Debugger > Ray-Traced passes (sample count, resolution). Monitor: GI update time (should be <2ms for 60fps), frame time impact (GI should not dominate).

## Related Topics

- [14.2 Baking Systems](14-02-Baking-Systems.md) - Enlighten vs Progressive
- [14.3 Light Probes and Reflection](14-03-Light-Probes-And-Reflection.md) - Dynamic object GI
- [13.1 Lighting Techniques](13-01-Lighting-Techniques.md) - Realtime vs baked
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Ray tracing performance

---

[← Previous: 14.3 Light Probes and Reflection](14-03-Light-Probes-And-Reflection.md) | [Next: Chapter 15 →](../chapters/15-Render-Targets-And-Buffers.md)
