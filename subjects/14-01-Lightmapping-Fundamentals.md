# 14.1 Lightmapping Fundamentals

[← Back to Chapter 14](../chapters/14-Global-Illumination-And-Lightmapping.md) | [Main Index](../README.md)

Lightmapping bakes lighting into textures (lightmaps) for static geometry, capturing direct lighting, indirect lighting (bounces), and shadows to enable realistic lighting at minimal runtime cost.

---

## Overview

Lightmapping precomputes lighting for static objects: light bounces (global illumination), shadows, and ambient occlusion baked into textures (lightmaps). Runtime renders lit geometry without real-time lighting calculations (samples lightmap texture instead, zero lighting cost). Enables complex lighting (multiple bounces, soft shadows, color bleeding) impossible in real-time. Cost: offline baking time (minutes to hours), memory overhead (lightmap textures = 10-500MB), static-only (baked objects can't move).

Lightmaps encode lighting in textures: each static mesh has lightmap UV (separate from albedo UV), lightmap pixels store lighting (RGB = irradiance, per texel lighting information). Runtime samples lightmap (UVs map mesh to lightmap texture), multiplies albedo by lightmap (final lit appearance). Multiple resolutions possible: high-resolution for important geometry (4096x4096 lightmaps = 64 texels/unit), low-resolution for large surfaces (256x256 = 4 texels/unit).

Unity supports two lightmappers: Progressive Lightmapper (default, path tracing, fast iteration), and Enlighten (deprecated, real-time GI). Progressive Lightmapper: GPU-accelerated, fast bakes (minutes for medium scenes), high-quality indirect lighting (path tracing for bounces). Enlighten: CPU-based, slow (hours), supports runtime GI updates (light intensity/color changes at runtime). Modern projects use Progressive (faster, higher quality).

## Key Concepts

- **Lightmap Texture**: Stores baked lighting (RGB irradiance per texel). Static meshes sample lightmap via lightmap UVs (second UV channel). Resolution determines quality: high resolution = smooth lighting (64 texels/unit), low = blocky (4 texels/unit). Unity packs multiple objects into single lightmap atlas (efficient memory usage, fewer textures).
- **Lightmap UV**: Secondary UV channel (UV1) for lightmapping (separate from albedo UV0). Must be non-overlapping (each triangle unique UV space, no overlap), proper padding (gaps between UV islands prevent bleeding), 0-1 range (normalized coordinates). Unity generates automatically (Mesh Import Settings > Generate Lightmap UVs) or manually created (DCC tools like Blender, Maya).
- **Lightmap Resolution**: Texels per unit (Lightmap Resolution parameter). Higher = more detailed lighting (64 texels/unit = sharp shadows, smooth GI), lower = less memory (4 texels/unit = blocky shadows). Per-object resolution scale (MeshRenderer > Lightmap Settings > Scale in Lightmap = multiplier, 0.5 = half resolution, 2.0 = double).
- **Direct vs Indirect Lighting**: Direct = light from source to surface (no bounces, one light ray). Indirect = light bounces off surfaces (GI: light hits red wall, bounces to white wall, white wall tinted red). Lightmaps capture both (realistic lighting with color bleeding). Real-time lighting only direct (no bounces, unrealistic).
- **Baking Modes**: Baked (fully baked, no runtime cost, static only), Mixed (baked indirect + realtime direct, dynamic objects receive baked GI), Realtime (deprecated Enlighten feature). Mixed mode common (static environment = baked, dynamic objects = realtime shadows + baked GI).

## Best Practices

**Lightmap UV Setup:**
- Auto-generate UVs: Mesh Import Settings > Generate Lightmap UVs (enabled). Unity creates UV1 automatically (non-overlapping, padded). Works for most assets (simple geometry). Parameters: Hard Angle (UV seam threshold, 88° default), Pack Margin (padding between islands, 4 pixels default), Angle Error/Area Error (quality vs island count, lower = higher quality).
- Manual UVs for complex meshes: DCC tools (Blender, Maya) create custom UV1. Ensures optimal layout (minimize seams, maximize space usage). Required for: organic shapes (characters, vehicles), tiling textures (architectural surfaces), optimized padding (large objects need less padding).
- UV padding: Pack Margin prevents light bleeding (adjacent UV islands bleed lighting across edges). 4 pixels default (works for most), increase for high-resolution lightmaps (8-16 pixels for 4096x4096), decrease for memory savings (2 pixels for 1024x1024). Insufficient padding = visible seams in lighting.
- Non-overlapping requirement: Overlapped UVs = multiple surfaces share lightmap space (incorrect lighting). Unity's auto-generate ensures no overlap. Manual UVs must avoid overlap (each triangle unique space). Validate: UV Editor shows overlaps (red highlighting in DCC tools).

**Lightmap Resolution Tuning:**
- Scene-wide resolution: Lighting window > Lightmapping Settings > Lightmap Resolution (texels per unit). Default = 40 texels/unit (balanced quality). High quality = 64-128 (detailed shadows, smooth GI), low quality = 4-16 (blocky, low memory).
- Per-object scaling: MeshRenderer > Scale in Lightmap. Important objects (hero props, player-visible areas) = 2.0-4.0 (higher resolution), background (distant walls, occluded areas) = 0.5-0.25 (lower resolution). Optimizes memory (high resolution where visible, low where not).
- Resolution validation: Lightmap preview shows texel density (Lighting > Debug Settings > Lightmap Resolution). Color overlay: green = good resolution, blue = low (blocky), red = excessive (wasted memory). Adjust per-object scale to balance quality/memory.
- Atlas packing: Unity packs objects into lightmap atlases (4096x4096 max texture size). Many small objects = efficient packing (minimal waste). Few large objects = poor packing (large empty areas). Group nearby objects (static batching) for better packing.

**Lighting Setup for Baking:**
- Mark static geometry: Select objects > Inspector > Static checkbox (or Contribute GI). Only static objects baked (moving objects can't use lightmaps). Mark: environment (terrain, buildings, props), large static objects (furniture, rocks). Don't mark: dynamic objects (characters, vehicles), small movable props.
- Light mode settings: Directional light = Mixed or Baked (sun provides baked shadows + indirect lighting). Point/spot lights = Baked (indoor lights fully baked). Real-time lights on dynamic objects only (character flashlight, car headlights).
- Baked Global Illumination: Lighting window > Scene > Baked Global Illumination (enabled). Enables indirect lighting (light bounces). Bounces parameter (default = 2, more = slower bakes + softer lighting). Ambient occlusion (darkens crevices, enhances depth).
- Lightmap compression: Project Settings > Player > Lightmap Encoding. Normal Quality = BC6H (HDR, 6:1 compression), Low Quality = RGBM (LDR, 4:1 compression). PC/consoles = Normal Quality (high quality HDR), mobile = Low Quality (memory savings).

**Baking Workflow:**
- Progressive baking: Lighting window > Auto Generate (enabled). Automatic re-bakes on changes (light moved, geometry changed). Fast iteration (real-time feedback). Disable for final bakes (manual control, higher quality settings).
- Bake settings: Lighting window > Lightmapping Settings. Lightmapper = Progressive GPU (fastest, requires DX12/Vulkan), Lightmap Resolution (texels/unit), Indirect Resolution (GI detail, lower = faster), Bounces (GI bounces, more = softer + slower). Prioritize Speed (low settings) for iteration, Prioritize Quality for final bakes.
- Preview vs final bakes: Low settings during iteration (Indirect Resolution = 0.5, Bounces = 1, fast feedback). High settings for final (Indirect Resolution = 2, Bounces = 3-4, Samples = 1000+). Iteration: seconds per bake, final: minutes to hours.
- Baked data storage: Unity generates lightmap textures (Assets > Scenes > <SceneName> folder). Lightmap-0.exr, Lightmap-1.exr (HDR textures), Lightmap-ShadowMask (optional mixed lighting data). Include in version control (shared bakes across team).

**Platform-Specific:**
- **PC**: High-resolution lightmaps (64-128 texels/unit). BC6H compression (HDR, high quality). Large atlases (4096x4096). Multiple bounces (3-4 for realistic GI). Progressive GPU lightmapper (fast bakes on powerful GPUs).
- **Consoles**: Moderate resolution (40-64 texels/unit). BC6H compression. 4096x4096 atlases (memory available). 2-3 bounces (balanced quality/bake time). Offline bakes (console devkits for final bakes).
- **Switch**: Low resolution (16-32 texels/unit). ASTC compression (5-6:1, smaller file sizes). 2048x2048 atlases (memory limited). 1-2 bounces (minimal GI). Aggressive compression (prioritize memory over quality).
- **Mobile**: Very low resolution (4-16 texels/unit). ASTC 8x8 (aggressive compression). 1024x1024 atlases (extremely memory limited). 1 bounce (minimal GI, baking speed). RGBM encoding (LDR, smaller size than HDR).

## Common Pitfalls

**Overlapping Lightmap UVs**: Developer imports mesh without proper UV1 (overlapping UVs). Baking produces incorrect lighting (multiple surfaces share lightmap space, one surface's lighting bleeds onto another). Symptom: Lighting looks wrong (dark patches, incorrect shadows, visible artifacts). Solution: Enable Generate Lightmap UVs in mesh import settings (Unity creates non-overlapping UV1 automatically). Or manually create UV1 in DCC tool (unwrap without overlaps).

**Insufficient Lightmap Resolution**: Developer uses default 40 texels/unit for all objects (small props get blocky shadows, large walls waste resolution). Lighting quality inconsistent. Symptom: Small objects have pixelated lighting (blocky shadows, stair-stepping), low visual quality. Solution: Increase per-object Scale in Lightmap (important objects = 2.0-4.0x scale). Optimize: large background objects = 0.5x scale (saves memory), hero props = 4.0x (high quality). Lightmap resolution preview validates settings.

**Excessive Lightmap Memory**: Developer sets very high resolution (128 texels/unit, all objects 4.0x scale). Lightmaps balloon to 2GB (hundreds of 4096x4096 textures). Loading times increase (disk I/O bottleneck), memory exhausted (crashes on consoles/mobile). Symptom: Long level load times (15+ seconds), out-of-memory crashes, huge build sizes. Solution: Reduce resolution (40-64 texels/unit), scale down unimportant objects (background = 0.5x), reduce scene complexity (merge meshes, fewer unique objects). Monitor lightmap memory (Lighting window shows total size).

**No UV Padding**: Developer generates UVs with 0 padding (Pack Margin = 0). Adjacent UV islands bleed lighting (edges of objects show lighting from neighbors). Symptom: Visible seams in lighting (dark or bright lines along mesh edges, lighting inconsistencies). Solution: Set Pack Margin to 4+ pixels in mesh import settings (Generate Lightmap UVs > Pack Margin = 4). Higher resolution lightmaps need more padding (8-16 pixels for 4096x4096).

## Tools & Workflow

**Lighting Window**: Window > Rendering > Lighting. Scene tab (Baked Global Illumination, Bounces, Ambient settings), Lightmapping Settings (Lightmapper, Resolution, Indirect Resolution), Auto Generate toggle (automatic baking), Generate Lighting button (manual bake).

**Lightmap Preview**: Scene view > Shaded > Baked Lightmap. Shows baked lighting in scene (without albedo, lighting-only view). Identifies issues (dark spots, low resolution, seams). Lighting > Debug Settings > Lightmap Resolution (color overlay: green = good, blue = low resolution, red = excessive).

**Mesh Import Settings**: Select mesh > Inspector > Generate Lightmap UVs (enabled). Hard Angle (88° default, UV seam threshold), Pack Margin (4 pixels default, padding), Angle/Area Error (quality settings). Preview UVs (bottom of inspector, shows UV0 and UV1 layouts).

**Progressive Lightmapper**: Lighting window > Lightmapper > Progressive GPU (fastest, DX12/Vulkan required) or Progressive CPU (slower, universal). Prioritize View toggle (bakes visible area first, fast preview), Lightmap Parameters (Custom asset for advanced control: AO distance, backface tolerance, sampling).

**Lightmap Parameters Asset**: Create > Rendering > Lightmap Parameters. Advanced settings: Backface Tolerance (prevents light leaking through thin walls), AO Distance (ambient occlusion radius), Indirect Resolution (GI detail), Anti-aliasing Samples (edge quality). Assign to objects (MeshRenderer > Lightmap Parameters).

**Frame Debugger**: Verify lightmap usage. Shows objects rendering with lightmaps (texture preview), identifies missing lightmaps (objects rendering black), checks lightmap atlas packing (how many atlases, packing efficiency).

## Related Topics

- [14.2 Baking Systems](14-02-Baking-Systems.md) - Progressive vs Enlighten
- [14.3 Light Probes and Reflection](14-03-Light-Probes-And-Reflection.md) - Dynamic object lighting
- [13.1 Lighting Techniques](13-01-Lighting-Techniques.md) - Realtime vs baked
- [11.3 Texture Mapping](11-03-Texture-Mapping-Techniques.md) - UV layout

---

[← Previous: 13.6 Culling Techniques](../subjects/13-06-Culling-Techniques.md) | [Next: 14.2 Baking Systems →](14-02-Baking-Systems.md)
