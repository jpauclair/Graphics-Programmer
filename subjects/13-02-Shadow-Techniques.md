# 13.2 Shadow Techniques

[← Back to Chapter 13](../chapters/13-Rendering-Techniques.md) | [Main Index](../README.md)

Shadow techniques render contact shadows through shadow mapping, cascaded shadow maps (CSM), soft shadows via PCF filtering, and performance optimizations like shadow distance and resolution tiers.

---

## Overview

Shadow mapping renders scene from light's perspective (depth-only pass), stores depth in shadow map texture, then during main rendering compares pixel depth with shadow map (if pixel farther than shadow map = shadowed, else lit). Shadow maps are depth textures: directional lights use 2048x2048-4096x4096 (cover large areas), point lights use cubemaps 512x512-1024x1024 per face (6 faces = 6x memory), spot lights use 512x512-1024x1024 (cone projection). Cost: shadow rendering pass (renders all shadow casters), shadow sampling (texture fetch + comparison per pixel).

Cascaded Shadow Maps (CSM) solve directional light resolution problem: single shadow map for entire view (0-1000m) = low resolution (1 texel covers 1-5m, blocky shadows). CSM splits view into cascades (0-20m, 20-50m, 50-100m, 100m+), renders separate shadow map per cascade (each at appropriate resolution). Close shadows detailed (cascade 0 = 2048x2048 for 20m), distant shadows coarser (cascade 3 = 512x512 for 900m). Unity supports 1-4 cascades (Quality Settings > Shadow Cascades).

Soft shadows via PCF (Percentage Closer Filtering): sample shadow map multiple times (4-16 samples in kernel), average results (0-1 = shadow gradient). Creates soft shadow edges (penumbra), more realistic than hard shadows (0 or 1, no gradient). Cost: N texture samples per pixel (PCF 16 = 16x cost vs hard shadows). Unity Quality Settings > Shadow Quality (Hard Shadows, Soft Shadows). Mobile: hard shadows only (PCF too expensive).

## Key Concepts

- **Shadow Map**: Depth texture rendered from light's viewpoint. Stores distance from light to nearest surface. During main rendering, compare pixel depth with shadow map value (pixel depth > shadow map = shadowed, else lit). Resolution: 512x512 (low), 1024x1024 (medium), 2048x2048 (high), 4096x4096 (very high).
- **Cascaded Shadow Maps (CSM)**: Multiple shadow maps at different distances. Unity splits camera frustum into 2-4 cascades, renders shadow map per cascade. Close cascades high-resolution (detailed shadows), far cascades low-resolution (coarse shadows). Configured via Quality Settings > Shadow Cascades (Two/Four Cascades).
- **Percentage Closer Filtering (PCF)**: Soft shadow technique. Samples shadow map multiple times (4-16 samples), averages results. Creates soft shadow edges (penumbra). Unity Quality Settings > Shadow Quality = Soft Shadows enables PCF. Hard Shadows = single sample (sharp edges).
- **Shadow Distance**: Maximum distance for realtime shadows. Beyond distance, no shadows rendered (performance savings). Quality Settings > Shadow Distance = 50-150m typical. Combine with baked shadows (Shadowmask) for static shadows beyond distance.
- **Shadow Bias**: Offsets shadow comparison to avoid shadow acne (self-shadowing artifacts). Depth Bias (offset depth value), Normal Bias (offset sample position along normal). Light component > Shadow Type > Bias/Normal Bias. Too low = acne (surface shadows itself), too high = peter panning (shadow detaches from object).

## Best Practices

**Cascade Configuration:**
- Cascade count: 2 cascades for mobile/Switch (performance), 4 cascades for PC/console (quality). More cascades = better resolution distribution but more draw calls (render shadow casters 4 times instead of 2).
- Cascade splits: Control via Shadow Cascade Splits (Quality Settings). Default: [0.067, 0.2, 0.467, 1.0] (logarithmic distribution). Adjust based on needs: first cascade covers <10m (character shadows), second 10-30m (nearby objects), third 30-80m (environment), fourth 80m-distance.
- Cascade distances: Shadow Distance = 100m typical (outdoor), 50m for interiors (enclosed spaces, shorter sightlines). Longer distance = coarser far cascades (spread resolution over larger area). Shorter = better quality (concentrated resolution).
- Last cascade fadeout: Enable Last Cascade Fade (fades shadows at edge of shadow distance). Avoids hard cutoff (shadows pop out at distance). Smooth transition (shadows fade to ambient over 10-20m).

**Shadow Resolution Tuning:**
- Per-light resolution: Light > Shadow Type > Resolution (Low 512, Medium 1024, High 2048, Very High 4096). Main directional light (sun) = High/Very High (most visible). Point/spot lights = Medium/Low (localized, less visible).
- Quality tiers: Configure per Quality Setting (Low = 1024, Medium = 1024, High = 2048, Ultra = 4096). PC players choose quality (memory vs detail trade-off). Mobile: fixed Low (512-1024).
- Memory budget: Each shadow map = width * height * 4 bytes (depth texture). Directional 4K 4-cascade = 4096x4096*4*4 = 256MB. Point light cubemap 1K = 1024x1024*4*6 = 24MB. Budget memory (total shadow maps <200MB typical for consoles).
- Atlas packing: Unity packs multiple point/spot shadows into atlases (efficient memory use). Shadow Atlas Size (Quality Settings) = 2048 or 4096 (total atlas size for all non-directional shadows).

**Shadow Filtering:**
- Hard shadows: Single shadow map sample (fast, sharp edges). Use for: mobile (PCF too expensive), stylized games (hard shadows artistic choice), distant shadows (PCF artifacts at distance). Quality Settings > Shadow Quality = Hard Shadows.
- Soft shadows (PCF): 4-16 shadow map samples (smooth penumbra). Use for: PC/console (quality priority), realistic aesthetics, close-up shadows (character shadows on ground). Quality Settings > Shadow Quality = Soft Shadows. Cost: 5-10x hard shadows.
- PCF sample count: Unity PCF uses 4-tap (mobile), 9-tap (console), or 16-tap (PC high). Higher taps = smoother but slower. Configure via pipeline asset (URP: Shadow Sample Count, HDRP: Shadow Filtering Quality).
- Contact shadows: HDRP feature. Short-range shadows from screen-space depth (complements shadow maps). Captures small details (character fingers, facial features). Expensive (screen-space ray-march per pixel).

**Bias Tuning:**
- Depth bias: Offsets shadow map depth (reduces shadow acne). Start with 0.05, increase if acne visible (black specks on surfaces). Too high causes peter panning (shadows separate from objects, floating shadows).
- Normal bias: Offsets shadow sample along surface normal (reduces acne without peter panning). Start with 0.4, adjust per light. Better than depth bias (less prone to peter panning) but can cause missing shadows (thin objects).
- Per-material bias: Some materials need custom bias (double-sided foliage, thin geometry). Adjust via Light > Shadow Type > Normal Bias per scene requirements. Trial-and-error tuning (balance acne vs peter panning).
- Platform differences: Mobile GPUs have lower depth precision (16-bit vs 24-bit). Require higher bias (more acne prone). Test on target devices (tune bias for worst-case hardware).

**Performance Optimization:**
- Shadow distance reduction: Reduce Shadow Distance (Quality Settings). 50m instead of 150m = 3x less area = 3x better cascade resolution. Significant GPU savings (fewer pixels shadowed, smaller shadow maps).
- Cull shadow casters: Disable Renderer > Cast Shadows for objects that don't need shadows (small props, distant background objects, skybox). Reduces shadow render pass cost (fewer draw calls, less geometry).
- Layer culling distance: Light > Culling Mask culls layers from shadows. Example: exclude "Effects" layer from shadows (particle systems don't cast shadows, saves geometry processing). Per-light control (main light shadows all, point lights shadow only nearby layers).
- Update frequency: Realtime shadows update every frame (expensive). Static shadows bake once (free runtime). Mixed mode (Shadowmask): static shadows baked, dynamic shadows realtime. Best balance (most shadows free, only dynamic objects cast realtime shadows).

**Platform-Specific:**
- **PC**: High-quality shadows. 4 cascades, 2K-4K resolution (main light), PCF soft shadows (16-tap). Shadow Distance = 100-200m (long view distances). Multiple realtime shadow-casting lights (4-8 lights).
- **Consoles**: Balanced shadows. 4 cascades, 1K-2K resolution (main light), PCF soft shadows (9-tap). Shadow Distance = 80-150m. 2-4 realtime shadow lights. Use Shadowmask for static shadows (performance critical).
- **Switch**: Performance shadows. 2 cascades, 512-1K resolution, hard shadows (no PCF). Shadow Distance = 30-60m (short draw distance). 1 realtime shadow light (main directional). Heavy reliance on baked shadows.
- **Mobile**: Minimal shadows. 1-2 cascades, 512 resolution, hard shadows only (no PCF). Shadow Distance = 20-50m. 0-1 realtime shadow light (often disabled, baked shadows only). Aggressive culling (small shadow distance, few casters).

## Common Pitfalls

**Shadow Acne**: Developer sees black speckles on surfaces (shadow acne, surface self-shadowing due to depth precision). Increase Depth Bias or Normal Bias (Light component > Shadow Bias settings). Symptom: Black dots/specks on lit surfaces, flickering shadows. Solution: Start Normal Bias = 0.4, increase to 0.8-1.2 until acne gone. Avoid excessive bias (causes peter panning).

**Peter Panning**: Developer increases bias too much. Shadows detach from objects (floating shadows, gap between object and shadow). Unrealistic appearance. Symptom: Shadows don't touch objects, visible gap. Solution: Reduce bias (Normal Bias to 0.2-0.4, Depth Bias to 0.01-0.05). Balance between acne and peter panning (slight acne acceptable, peter panning more noticeable).

**Insufficient Cascade Resolution**: Developer uses 2 cascades with Shadow Distance = 200m. Far cascade covers 100-200m = 1024x1024 for 100m area = 10cm per texel (very blocky shadows at distance). Symptom: Blocky shadows far from camera, visible pixelation. Solution: Increase cascade count (4 cascades), reduce Shadow Distance (100m instead of 200m), or increase shadow resolution (2048x2048 main light).

**Too Many Shadow-Casting Lights**: Developer enables Cast Shadows on 10 point lights (decorative lamps). Each light renders shadow map (cubemap = 6 faces), 10 lights = 60 render passes. GPU bottleneck. Symptom: Low frame rate, Profiler shows expensive shadow rendering. Solution: Disable Cast Shadows on unimportant lights (decorative lamps = no shadows). Reserve realtime shadows for 1-2 important lights (main sun, player flashlight). Use baked shadows for static lights.

## Tools & Workflow

**Quality Settings**: Edit > Project Settings > Quality. Configure Shadow Distance, Shadow Cascades (None/Two/Four), Shadow Resolution, Shadow Quality (Hard/Soft). Create tiers (Low/Medium/High/Ultra) for player choice.

**Light Component**: Inspector > Light > Shadow Type (No Shadows, Hard Shadows, Soft Shadows). Shadow Resolution (Low 512, Medium 1024, High 2048, Very High 4096). Bias settings (Depth Bias, Normal Bias). Per-light control.

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows shadow map rendering (draw calls per cascade), shadow map textures (resolution, contents), shadow sampling (per draw call). Identify expensive shadow passes (too many casters, high resolution).

**Rendering Debugger**: URP/HDRP: Window > Analysis > Rendering Debugger > Lighting > Shadows. Visualize shadow cascades (color-coded: red = cascade 0, green = cascade 1, blue = cascade 2, yellow = cascade 3). Verify cascade coverage (appropriate splits).

**RenderDoc/PIX**: Capture shadow map rendering. View shadow map textures (depth values), inspect shadow sampling (compare depth, PCF samples). Debug shadow artifacts (acne, peter panning, incorrect bias).

**Unity Profiler**: GPU Profiler > Rendering shows shadow rendering time (ShadowMap rendering). CPU Profiler > Rendering shows culling (shadow caster culling). Identify shadow performance cost (which lights expensive? Which cascades?).

**Shadow Visualizer**: Scene view > Shading Mode > Shadow Cascades (visualizes cascades in scene view, color-coded). Edit mode: adjust Shadow Distance, cascade splits, see results immediately. Helps tune cascade configuration.

## Related Topics

- [13.1 Lighting Techniques](13-01-Lighting-Techniques.md) - Lighting modes and setup
- [14.1 Lightmapping Fundamentals](../chapters/14-GI-And-Lightmapping.md) - Baked shadows
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Shadow performance optimization
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Identifying shadow bottlenecks

---

[← Previous: 13.1 Lighting Techniques](13-01-Lighting-Techniques.md) | [Next: 13.3 Post-Processing →](13-03-Post-Processing.md)
