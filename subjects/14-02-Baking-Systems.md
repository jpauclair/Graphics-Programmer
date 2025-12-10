# 14.2 Baking Systems

[← Back to Chapter 14](../chapters/14-Global-Illumination-And-Lightmapping.md) | [Main Index](../README.md)

Unity provides two lightmapping systems: Progressive Lightmapper (modern, GPU-accelerated path tracing) and Enlighten (deprecated, CPU-based radiosity with runtime GI support).

---

## Overview

Progressive Lightmapper: Unity's default baking system (Unity 2020+), GPU-accelerated (DX12, Vulkan, Metal 2), path tracing for indirect lighting (physically accurate bounces), progressive refinement (see results immediately, improves over time). Fast iteration (seconds for preview, minutes for production), high quality (unbiased path tracing, accurate GI). Best for: modern projects (GPU acceleration available), static lighting (no runtime GI changes), fast iteration (real-time feedback during lighting design).

Enlighten: Legacy system (deprecated in Unity 2022+), CPU-based (multi-threaded radiosity), supports Precomputed Realtime GI (runtime light intensity/color changes). Slow bakes (hours for medium scenes), lower quality indirect lighting (biased radiosity, less accurate than path tracing). Used in: older projects (Unity 5-2019), games requiring runtime GI (dynamic time-of-day with baked lighting), backward compatibility (legacy content). New projects should avoid (deprecated, slower, lower quality).

Progressive workflow: Enable Auto Generate (automatic re-bakes on changes), adjust lights/geometry (see updates in real-time), iterate quickly (lighting preview in seconds), finalize with high-quality bake (increase samples, bounces). Progressive refinement shows coarse results immediately (noisy, low sample count), refines progressively (adds samples, reduces noise), converges to final quality (high sample count, clean result). Cancel anytime (partial results usable).

## Key Concepts

- **Progressive Lightmapper**: GPU path tracing (Unity 2020+ default). Progressive CPU (fallback, no GPU acceleration required) or Progressive GPU (fastest, requires DX12/Vulkan/Metal 2). Path tracing: shoots rays from camera, bounces through scene, accumulates lighting (unbiased, physically accurate). Fast previews (low samples = noisy), high-quality finals (high samples = clean).
- **Enlighten**: Legacy CPU radiosity system. Precomputed Realtime GI (bakes light transport, allows runtime intensity/color changes). Baked GI (fully static, no runtime changes). Slow (hours for complex scenes), lower quality (biased radiosity, less accurate bounces). Deprecated in Unity 2022+ (removed from new projects, legacy support only).
- **Lightmap Parameters**: Control baking quality/speed. Samples (Direct Samples, Indirect Samples, Environment Samples = ray count, higher = less noise + slower). Bounces (indirect light bounces, more = softer lighting + longer bakes). Resolution (Lightmap Resolution = texels/unit, Indirect Resolution = GI detail). AO (Ambient Occlusion Max Distance, AO Intensity).
- **Prioritize View**: Progressive feature (bakes visible area first). Scene view camera determines priority (objects in view bake first, preview lighting immediately). Background areas bake progressively (lower priority). Useful for large scenes (focus on current area, don't wait for entire scene).
- **Auto Generate vs Manual**: Auto Generate = automatic re-bakes on changes (real-time feedback, convenient for iteration). Manual = Generate Lighting button (explicit control, disable for final bakes with high settings). Workflow: Auto during design (fast feedback), Manual for finals (high quality, no interruptions).

## Best Practices

**Progressive Lightmapper Configuration:**
- Choose lightmapper: Lighting window > Lightmapper > Progressive GPU (fastest, requires DX12/Vulkan), Progressive CPU (slower, universal compatibility). GPU requires: Windows DX12, Linux Vulkan, or macOS Metal 2. Older systems or incompatible GPUs fall back to CPU.
- Sampling settings: Direct Samples (1-500, shadows quality), Indirect Samples (100-2000, GI quality), Environment Samples (100-1000, skybox lighting). Preview = low (Direct 1, Indirect 100, fast noisy results). Production = high (Direct 100, Indirect 1000+, clean results). Higher = less noise + longer bakes.
- Bounces configuration: Bounces = indirect light iterations (default = 2). More bounces = softer lighting (light reaches more areas), longer bakes (each bounce doubles rays). 1 bounce = direct GI only (one surface reflection), 2 = typical (two reflections, balanced), 3-4 = very soft (extensive bounces, long bakes). Diminishing returns beyond 4 (minimal visual difference).
- Filtering settings: Filtering = reduces noise in lightmaps (smooths lighting). Direct/Indirect/AO filtering (None, Auto, Advanced). Auto = Unity determines filter radius (works for most scenes). Advanced = manual control (Gaussian filter radius, large = smooth + blurry, small = noisy + sharp). Production uses Auto or small Advanced filter.

**Quality vs Speed Tradeoffs:**
- Preview baking (iteration): Low samples (Direct 1, Indirect 100), low bounces (1-2), low resolution (Indirect Resolution 0.5), Prioritize View enabled. Bake time: seconds to minutes (fast feedback). Quality: noisy, approximate (sufficient for design iteration). Use during: lighting layout, object placement, material tweaking.
- Production baking (finals): High samples (Direct 32-100, Indirect 500-2000), high bounces (2-4), high resolution (Indirect Resolution 1-2), Prioritize View disabled (entire scene). Bake time: minutes to hours (depends on scene complexity). Quality: clean, accurate (final release quality). Use for: final builds, screenshots, trailers.
- Incremental baking: Bake iteratively (start low, increase settings gradually). Preview (1min bake, noisy) → intermediate (10min, cleaner) → final (1hr+, production). Avoids wasting time (don't do final bake if lighting design wrong). Cancel anytime (progressive results preserved, partial quality usable).
- Scene complexity management: Reduce bake time via: lower geometry density (merge meshes, LOD), fewer lights (consolidate light sources), smaller lightmaps (reduce resolution, per-object scale), disable unimportant objects (temporary disable background during iteration). Large scenes (10,000+ objects) = hours even with fast settings.

**Enlighten Baking (Legacy):**
- Enable Enlighten: Lighting window > Lightmapper > Enlighten (if available, removed in Unity 2022+). Precomputed Realtime GI checkbox (runtime GI, intensity/color changes). Baked GI (static, no runtime changes). Both can coexist (Realtime + Baked).
- Realtime resolution: Realtime Resolution (texels/unit for realtime GI system, lower than baked lightmap resolution). Low resolution (0.5-2) = faster computation + less memory, higher (4-8) = detailed GI. Realtime GI memory intensive (GI data larger than baked lightmaps).
- Clustering: Enlighten groups nearby surfaces (clusters for radiosity computation). Clustering Resolution (surfaces per cluster, higher = more clusters = slower + more accurate). Default = 0.5-1 (balanced). High = 2+ (detailed GI + very slow bakes).
- Runtime performance: Realtime GI has runtime cost (CPU updates GI on light changes, milliseconds per frame). Limit to: key lights only (directional sun, primary point lights), avoid many realtime GI lights (each adds cost). Modern approach: use Progressive for static lighting (no runtime GI, bake time-of-day variations separately).

**Lightmap Parameters Assets:**
- Custom parameters: Create > Rendering > Lightmap Parameters (asset with advanced settings). Assign to objects (MeshRenderer > Lightmap Parameters) or scene (Lighting > Scene > Lightmap Parameters). Controls per-object baking behavior.
- Key parameters: Backface Tolerance (0-1, prevents light leaking through walls, higher = less leaking), AO Max Distance (ambient occlusion radius, meters), Indirect Resolution (multiplier for GI detail, 2.0 = double resolution), Samples multiplier (increases samples for this object only).
- Use cases: Thin walls (increase Backface Tolerance, prevents light bleeding through). Detailed props (increase Indirect Resolution, more GI detail). Dark interiors (increase AO Distance, stronger shadows in corners). Hero objects (increase Samples, reduce noise).
- Default vs custom: Unity Default-Medium parameters (balanced, ships with Unity). Custom for: specific problems (light leaking, noise), artistic control (enhance AO, sharper/softer GI), platform optimization (low-spec = lower samples, high-spec = higher).

**Platform-Specific:**
- **PC**: Progressive GPU (fastest, DX12/Vulkan available). High samples (Indirect 1000-2000), 3-4 bounces. BC6H compression (HDR lightmaps). Bake on high-end GPU (RTX/AMD RDNA, minutes for large scenes). Final builds = maximum quality.
- **Consoles**: Progressive GPU (consoles support DX12/equivalent). Moderate samples (Indirect 500-1000), 2-3 bounces. BC6H compression. Bake on devkit or PC (transfer lightmap data to console). Balance quality/memory (consoles have fixed memory budgets).
- **Switch**: Progressive CPU (GPU acceleration unavailable on Switch). Low samples (Indirect 200-500), 1-2 bounces. ASTC compression (smaller file size). Bake on PC (Switch underpowered for baking). Aggressive memory optimization (small lightmaps, low resolution).
- **Mobile**: Progressive CPU. Very low samples (Indirect 100-200), 1 bounce. ASTC 8x8 compression (minimal memory). Bake on PC (mobile devices can't bake). Minimal GI (single bounce, low quality acceptable for mobile visuals).

## Common Pitfalls

**Using Enlighten for New Projects**: Developer chooses Enlighten for new Unity 2022+ project (wants runtime GI for day/night cycle). Enlighten deprecated (not available in new Unity versions), or very slow (hours to bake). Symptom: No Enlighten option (Unity 2022+), or extremely long bake times (6+ hours for medium scene). Solution: Use Progressive Lightmapper (bake multiple lighting scenarios: day/night/dusk as separate scenes or LightingSettings assets). Runtime GI = performance cost (not worth for most games, baked variations better).

**Too Many Samples (Wasted Time)**: Developer sets Indirect Samples = 10,000 (thinking more = better). Baking takes 24 hours for medium scene. Diminishing returns beyond 2000 samples (minimal noise reduction, massive time increase). Symptom: Extremely long bake times (overnight bakes), negligible quality improvement over lower settings. Solution: Use progressive refinement (start low, increase until noise acceptable). Typically 500-1000 samples sufficient (1000-2000 for hero areas). Compare 500 vs 5000 samples (usually imperceptible difference, 10x bake time).

**Ignoring Prioritize View**: Developer starts bake with Prioritize View disabled (entire scene bakes uniformly). Waits 30+ minutes to see preview (can't iterate lighting quickly). Symptom: No feedback during baking (scene view stays dark), long waits between iterations. Solution: Enable Prioritize View (Lighting window checkbox). Scene view camera area bakes first (preview in seconds), background bakes progressively. Move camera to area of interest (prioritizes that region).

**Mixed Enlighten and Progressive**: Developer enables both Enlighten Realtime GI and Progressive Baked GI (thinking both work together). Baking extremely slow (both systems running), huge memory usage (two sets of GI data). Symptom: Bakes take hours, GI data >500MB (both systems generating data). Solution: Choose one system. Modern projects = Progressive only (Baked GI checkbox, disable Realtime GI). Enlighten only if: legacy project requiring runtime GI (and willing to accept slow bakes).

## Tools & Workflow

**Lighting Window**: Window > Rendering > Lighting. Lightmapper dropdown (Progressive GPU/CPU, Enlighten if available), Prioritize View checkbox, Auto Generate toggle, Generate Lighting button (manual bake). Lightmapping Settings (Samples, Bounces, Resolution). Progress bar (shows bake progress, cancel button).

**Lightmap Parameters Asset**: Create > Rendering > Lightmap Parameters. Edit: Backface Tolerance, AO Max Distance, Indirect Resolution, Samples multiplier. Assign: Lighting window > Scene > Lightmap Parameters (scene-wide), or MeshRenderer > Lightmap Parameters (per-object).

**Progressive Lightmapper Settings**: Lighting > Lightmapping Settings. Direct/Indirect/Environment Samples (ray counts), Bounces (GI iterations), Filtering (noise reduction), Lightmap Resolution (texels/unit), Indirect Resolution (GI detail), Lightmap Padding (texels), Lightmap Size (atlas resolution, 1024-4096).

**Bake Visualization**: Scene view > Shaded > Baked Lightmap (lighting only, no albedo). Shows progressive refinement (starts noisy, becomes cleaner). Lighting > Debug Settings > Lightmap Indices (color-coded lightmap atlases), Lightmap Resolution (texel density overlay).

**Profiling Bake Times**: Unity Editor.log (shows bake statistics: time per stage, lightmap count, memory usage). Linux/Mac: ~/.config/unity3d/Editor.log, Windows: %APPDATA%\..\Local\Unity\Editor\Editor.log. Search for "Bake" (detailed timing information). Identify bottlenecks (geometry processing, sampling, compositing).

**Lightmap Compression**: Project Settings > Player > Lightmap Encoding. Normal Quality (BC6H HDR, PC/consoles), Low Quality (RGBM LDR, mobile). Affects file size (BC6H = 6:1 compression, RGBM = 4:1 + LDR conversion). Baked lightmaps stored as .exr (HDR), converted to platform format at build time.

## Related Topics

- [14.1 Lightmapping Fundamentals](14-01-Lightmapping-Fundamentals.md) - Lightmap UVs and basics
- [14.4 Real-Time Global Illumination](14-04-Real-Time-Global-Illumination.md) - Enlighten realtime GI
- [13.1 Lighting Techniques](13-01-Lighting-Techniques.md) - Baked vs realtime lighting
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Lightmap sampling performance

---

[← Previous: 14.1 Lightmapping Fundamentals](14-01-Lightmapping-Fundamentals.md) | [Next: 14.3 Light Probes and Reflection →](14-03-Light-Probes-And-Reflection.md)
