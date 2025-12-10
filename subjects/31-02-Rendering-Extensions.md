# 31.2 Rendering Extensions

[← Back to Chapter 31](../chapters/31-Third-Party-Tools.md) | [Main Index](../README.md)

Rendering extensions = Asset Store packages extending Unity graphics (post-processing stacks, volumetric effects, advanced lighting). Post Processing Stack = full-screen effects (bloom, color grading, depth of field), Aura 2 = volumetric lighting (god rays, fog), SEGI = global illumination (dynamic GI), Beautify = visual enhancement (sharpen, vibrance, vignette). Benefits: production-quality effects (AAA visuals), time savings (weeks of development = $50 package).

---

## Overview

Post-Processing: Unity Post Processing Stack v2 (built-in free, URP/HDRP integrated). Effects: Bloom (bright glow), Ambient Occlusion (SSAO = contact shadows), Motion Blur (camera/object motion), Depth of Field (focus blur), Color Grading (LUT-based color correction), Vignette (screen edge darkening). Integration: Post Process Volume component (global/local effects), camera Post Process Layer (applies effects).

Volumetric Effects: Aura 2 ($45, volumetric lighting/fog), Enviro (weather system with volumetrics), Atmospheric Height Fog. Features: light shafts (god rays from directional lights), volumetric fog (density-based scattering), temporal reprojection (reduce noise = smooth result). Performance: expensive (ray-marching through volume = many samples), use lower resolution (quarter-res then upscale).

## Key Concepts

- **Post Processing Stack v2**: Unity's free post-processing (Built-in RP, deprecated in URP/HDRP favor integrated). Package Manager → Post Processing. Setup: camera add Post Process Layer (volume layer mask), scene add Post Process Volume (global/local trigger). Effects: Bloom (threshold, intensity), AO (quality, radius), Chromatic Aberration (lens distortion), Grain (film noise). Performance: ~2-5ms console (1080p), ~5-10ms mobile (720p).
- **URP Volume Framework**: integrated post-processing (URP 10+). Volume component: Global (affects all cameras) or Local (trigger volume, blend weights). Effects profile: Inspector → Add Override → select effect (Bloom, Tonemapping, Color Adjustments). Layer-based: camera Render Post Processing enabled, Volume on specific layer. Performance: optimized (uber shader = combined passes), mobile-friendly (scalable quality).
- **Aura 2**: volumetric lighting/fog ($45 Asset Store). Features: directional light volumetrics (sun shafts), point/spot light volumes (light shafts from any light), volumetric fog (density gradients), temporal filtering (reduce noise). Integration: camera add Aura component, lights enable Aura (checkbox). Performance: quarter-res rendering (upscale = 4x faster), temporal reprojection (blend frames = smooth). Cost: ~3-8ms (depends on light count, volume resolution).
- **SEGI (Sonic Ether GI)**: real-time global illumination ($40, voxel-based). Features: dynamic GI (light bounces update real-time), infinite bounces (multi-bounce diffuse), emissive GI (glowing objects illuminate scene). Technique: voxelize scene (3D grid), trace cones (gather indirect light). Performance: expensive (~10-20ms), requires high-end PC/console. Alternative: Light Probe-based GI (Unity built-in, faster).
- **Beautify**: visual enhancement package ($30). Effects: sharpen (edge enhancement, crisper image), dither (reduce banding), LUT-based color grading (fast lookup table), vignette (screen edge fade), bloom + lens dirt (cinematic glow). Performance: lightweight (~1-2ms), single pass (uber shader). Use case: quick visual polish (add to existing project = instant quality boost).
- **Enviro**: dynamic weather system ($75, includes volumetrics). Features: day/night cycle (sun/moon rotation, skybox color), weather (rain, snow, fog density), volumetric clouds (ray-marched), lightning. Integration: prefab drop-in (auto-configure lights, post-processing), timeline support (scripted weather). Performance: volumetric clouds expensive (~5-10ms), fallback skybox mode (mobile-friendly).

## Best Practices

**Post-Processing Optimization**:
- URP: enable only needed effects (each effect = shader passes, disable unused = faster).
- Mobile: disable expensive effects (motion blur, depth of field = 5ms+ mobile), use LUT color grading only (fast lookup).
- Console: full effects supported (SSAO, bloom, motion blur = ~3-5ms total).

**Volumetric Effects**:
- Resolution: half-res or quarter-res volumetrics (upscale = acceptable quality, 4x-16x faster).
- Temporal filtering: enable reprojection (blend frames = smoother, reduce sample count).
- Light count: limit volumetric lights (each light = additional samples, expensive).

**Package Integration**:
- Version compatibility: check Unity version support (some packages lag behind latest Unity).
- Render pipeline: verify RP support (Built-in/URP/HDRP = different implementations).
- Documentation: read package docs (integration steps, performance tips, known issues).

**Platform-Specific**:
- **PC/Console**: full effects (volumetrics, complex post-processing, high-res).
- **Mobile**: minimal post-processing (bloom + color grading only), disable volumetrics (or very low-res).
- **VR**: careful with post-processing (motion blur = nausea, vignette OK), optimize for 90 FPS.

## Common Pitfalls

**Too Many Post Effects**: developer enables all post-processing effects (bloom, AO, motion blur, DOF, CA, grain = 10+ effects). Symptom: frame time spike (15ms+ just post-processing). Solution: profile each effect (disable one at a time, measure frame time), keep only essential (bloom + color grading usually sufficient).

**Wrong Volume Priority**: multiple Post Process Volumes overlap (conflicting settings). Symptom: effects flickering (blend between volumes = interpolation glitches). Solution: set Volume priority (higher value = overrides lower), or use single global volume (simpler = one source of truth).

**Aura Overdraw**: many overlapping volumetric lights (10+ point lights with Aura enabled). Symptom: GPU bottleneck (fragment shader expensive = volumetric sampling). Solution: limit volumetric lights (3-5 maximum), use baked lighting for static sources (Light Probe GI cheaper).

## Tools & Workflow

**Post Processing Stack v2 Setup** (Built-in RP):
```csharp
1. Package Manager → Post Processing (install)
2. Camera → Add Component → Post Process Layer
   - Volume Layer: PostProcessing (create layer)
3. GameObject → 3D Object → Post Process Volume
   - Is Global: checked
   - Profile: New (creates profile asset)
4. Profile → Add effect → Bloom (configure intensity, threshold)
5. Add more effects: AO, Color Grading, Vignette
```

**URP Volume Framework**:
```csharp
1. Camera → Rendering → Post Processing: enabled
2. GameObject → Volume → Global Volume
   - Profile: New (or existing)
3. Profile Inspector → Add Override → Bloom
   - Threshold: 1.0, Intensity: 0.5
4. Add more: Tonemapping (ACES), Color Adjustments (saturation)
```

**Aura 2 Integration**:
```csharp
1. Import Aura 2 package (Asset Store)
2. Camera → Add Component → Aura Camera
   - Frustum Settings: Volume Resolution: Medium (128³)
   - Temporal Filtering: Enabled (reduce noise)
3. Directional Light → Aura Light (checkbox enable)
4. Point/Spot Lights → Aura Light (enable for volumetric shafts)
```

**Beautify Quick Setup**:
```csharp
1. Import Beautify package
2. Camera → Add Component → Beautify
3. Inspector: Sharpen: 2.0, Bloom: enabled, Vignette: 0.3
4. Profile: save settings (reuse across cameras)
```

**SEGI Configuration**:
```csharp
1. Import SEGI package
2. GameObject → SEGI → SEGI (creates manager)
3. SEGI Inspector:
   - Voxel Resolution: 128 (higher = better quality, slower)
   - Bounces: 1 (multi-bounce = expensive)
4. Lights: enable SEGI contribution (checkbox on light component)
```

**Enviro Weather System**:
```csharp
1. Import Enviro - Sky and Weather package
2. GameObject → Enviro → Lite/Standard (drag prefab to scene)
3. Enviro Manager: auto-configures (skybox, fog, post-processing)
4. Timeline: add Enviro Weather track (script weather changes)
```

## Related Topics

- [15.1 Post-Processing Pipeline](15-01-Post-Processing-Pipeline.md) - Post-processing architecture
- [15.2 Common Post Effects](15-02-Common-Post-Effects.md) - Bloom, AO, DOF details
- [31.1 Shader Tools](31-01-Shader-Tools.md) - Custom shader creation
- [31.3 Optimization Tools](31-03-Optimization-Tools.md) - Profiling extensions

---

[← Previous: 31.1 Shader Tools](31-01-Shader-Tools.md) | [Next: 31.3 Optimization Tools →](31-03-Optimization-Tools.md)
