# 11.4 Material LOD

[← Back to Chapter 11](../chapters/11-Materials-And-Textures.md) | [Main Index](../README.md)

Material Level-of-Detail (LOD) optimizes rendering performance by swapping materials based on distance, reducing texture resolution, shader complexity, and map count for distant objects.

---

## Overview

Material LOD reduces rendering cost for distant objects: close objects (0-20m) use full PBR materials (2K-4K textures, albedo+metallic+normal+AO+emission), mid-range objects (20-50m) use simplified materials (1K-2K textures, albedo+normal only), distant objects (50m+) use minimal materials (512-1K textures, albedo only or unlit shaders). Without material LOD, distant objects waste GPU bandwidth (sampling 4K textures for objects occupying 10 pixels) and shader ALU (complex PBR calculations for barely-visible surfaces).

Implementation approaches: Unity LOD Group component (swaps entire meshes + materials per LOD level), manual material swapping (C# script changes renderer.sharedMaterial based on distance), Shader LOD (Shader.maximumLOD disables expensive shader features globally), and mipmap bias (increase bias for distant objects, forces lower mipmap levels = lower resolution textures). Texture streaming (Unity 2020.2+) automatically reduces texture resolution based on screen size (automatic material LOD without manual work).

Benefits: reduced VRAM usage (distant objects use lower-res textures = 75% memory savings), lower bandwidth (fewer texture fetches, smaller textures), decreased shader ALU (simpler shaders for distant objects), and improved frame rate (5-15ms savings on GPU time in large scenes). Trade-off: increased complexity (managing multiple material variants, LOD transitions), potential popping (visible material swap unless blended), and authoring time (creating simplified materials per LOD level).

## Key Concepts

- **LOD Levels**: Discrete distance ranges with corresponding materials. LOD0 = 0-20m (full quality), LOD1 = 20-50m (medium quality), LOD2 = 50-100m (low quality), LOD3 = 100m+ (minimal quality). Each level swaps mesh and material.
- **Shader LOD**: Global shader complexity setting. Shader.maximumLOD = 200 disables shader features above LOD 200 (e.g., parallax mapping = LOD 500, disabled). Quality Settings > Shader LOD sets per quality tier.
- **Texture Streaming**: Automatic texture resolution reduction based on screen coverage. Unity loads high-res mipmaps for close objects, low-res mipmaps for distant objects. Enabled per texture (Texture Import > Streaming Mipmaps).
- **Mipmap Bias**: Offset applied to mipmap level selection. Positive bias (+1.0) uses lower-res mipmaps (blurrier but faster), negative bias (-1.0) uses higher-res mipmaps (sharper but slower). Quality Settings > Texture Quality.
- **Material Swapping**: Runtime material replacement. Measure distance to camera (Vector3.Distance(transform.position, camera.position)), swap renderer.sharedMaterial when crossing thresholds. Requires multiple material variants (FullMaterial, SimplifiedMaterial).

## Best Practices

**LOD Group Configuration:**
- Distance thresholds: LOD0 = 0-15m (close, player sees details), LOD1 = 15-40m (mid-range, less detail visible), LOD2 = 40-80m (far, simplified rendering), LOD3 = 80m+ (very far, minimal rendering or culling). Adjust based on object size (large buildings = longer distances, small props = shorter distances).
- Material swapping per LOD: Assign different materials per LOD level. LOD0 = full PBR material (5 textures, complex shader), LOD1 = simplified material (2-3 textures, standard shader), LOD2 = basic material (albedo only, unlit/mobile shader). Unity LOD Group allows per-LOD renderer assignment.
- Transition zone: LOD Group > Fade Transition Width creates blend zone between LOD levels (cross-fades meshes/materials over distance range). Reduces popping (gradual transition). Cost: rendering both LODs simultaneously during transition (2x draw calls in fade zone).
- Culling LOD: LOD Group > Cull flag removes object entirely beyond threshold (e.g., Cull at 100m). Saves draw calls (object not rendered). Use for small objects (debris, props) that become invisible at distance.

**Texture Resolution Scaling:**
- LOD0: Full resolution (2K-4K textures). Close objects require high detail (player inspects surfaces, screenshots taken at close range). Use BC7 (albedo), BC5 (normal), BC4 (metallic-smoothness).
- LOD1: Half resolution (1K-2K textures). Mid-range objects (20-50m). Player still sees objects but fine details not visible. Downsample textures (Texture Import > Max Size = 1024), use same compression.
- LOD2: Quarter resolution (512-1K textures). Far objects (50-100m). Occupy small screen area (10-50 pixels). Aggressive downsampling, reduce map count (albedo + normal only, skip metallic/AO/emission).
- LOD3: Minimal resolution (256-512 textures). Very far objects (100m+). Nearly invisible or removed entirely (culling). Albedo-only or single-color tint (no textures, just vertex color).

**Shader Complexity Reduction:**
- LOD0 shader: Full PBR (Standard Shader, URP Lit, HDRP Lit). All features enabled (normal maps, metallic, smoothness, AO, emission, reflections). 100-300 ALU instructions.
- LOD1 shader: Simplified PBR. Disable expensive features (emission, detail maps, parallax). Use shader keywords (#pragma shader_feature) to strip unused features. 50-100 ALU instructions.
- LOD2 shader: Basic lighting. Mobile/Diffuse shader or custom lightweight shader. Albedo + simple diffuse lighting (Lambert), no normal maps. 20-40 ALU instructions.
- LOD3 shader: Unlit or vertex color. No lighting calculations (just albedo color or vertex colors). 5-10 ALU instructions. Or cull entirely (no rendering).

**Texture Streaming Integration:**
- Enable streaming: Texture Import Settings > Streaming Mipmaps = checked. Unity loads/unloads mipmaps based on screen coverage (automatic LOD). Requires Unity 2020.2+.
- Memory budget: Quality Settings > Texture Streaming > Memory Budget. Allocates memory pool for streamed textures (e.g., 512MB). Unity prioritizes visible textures (loads high-res for close objects, low-res for distant).
- Override per texture: Texture Priority (0-255) controls streaming priority. High priority = loads high-res mipmaps faster (hero characters = 255, background props = 50).
- Works with material LOD: Streaming automatically handles texture resolution per distance. Material LOD focuses on map count and shader complexity. Combine both for maximum optimization.

**Cross-Fading and Blending:**
- Dithered LOD: Use dithering (checkerboard pattern) to blend between LOD levels. Render both LODs with dither alpha (50% pixels per LOD). Temporal AA resolves dither (smooth blend). Implemented in HDRP (LOD Group > Fade Mode = Cross Fade).
- Transition buffer zones: Add distance buffer (e.g., LOD0 = 0-15m, transition = 15-17m, LOD1 = 17-40m). During transition, both LODs visible (cross-fade alpha). Softens popping.
- Avoid mid-frame swaps: Swap materials during low-intensity moments (idle time, slow movement). Avoid swapping during action (camera whip-pans, combat). Minimize visual disruption.

**Platform-Specific:**
- **PC**: Aggressive LOD distances (player sees far). LOD0 = 0-20m, LOD1 = 20-60m, LOD2 = 60-150m. High-end GPUs handle more draw calls (less aggressive culling). Quality Settings > LOD Bias = 1.0-2.0 (extends LOD ranges on high-quality settings).
- **Consoles**: Balanced LOD distances. LOD0 = 0-15m, LOD1 = 15-40m, LOD2 = 40-80m. Fixed hardware (optimize for target frame rate). Use texture streaming (saves VRAM, 10-16GB unified memory).
- **Switch**: Aggressive LOD (limited GPU, 3GB memory). LOD0 = 0-10m, LOD1 = 10-25m, LOD2 = 25-50m, Cull = 50m+. Simplified shaders (Mobile/Diffuse for LOD1+). Texture streaming critical (limited VRAM).
- **Mobile**: Very aggressive LOD. LOD0 = 0-5m, LOD1 = 5-15m, LOD2 = 15-30m, Cull = 30m+. Unlit shaders for LOD1+. Texture streaming mandatory (1-3GB memory budget, shared with OS/apps).

## Common Pitfalls

**No Material LOD**: Developer uses same full-quality material for all distances. Distant objects (100m away, occupying 5 pixels) sample 4K textures (wasteful). GPU bandwidth saturated (reading GB/s for invisible details). Symptom: Low frame rate despite low poly count, Memory Profiler shows 4K textures loaded for distant objects. Solution: Implement material LOD (LOD0 = 2K-4K, LOD1 = 1K, LOD2 = 512). Use LOD Group or manual distance-based swapping.

**Visible Popping**: Developer swaps materials instantly at threshold distance. Player sees sudden material change (popping, jarring). Especially noticeable on moving objects (car driving away, character walking). Symptom: Material "pops" at specific distance, player reports visual glitches. Solution: Enable LOD cross-fade (LOD Group > Fade Transition Width = 0.1). Or use dithered LOD (HDRP Cross Fade mode). Gradual blend reduces popping.

**Inconsistent LOD Distances**: Developer sets different LOD distances per object (Tree LOD0 = 0-10m, Rock LOD0 = 0-30m). Scene has inconsistent quality (some objects detailed, others simple at same distance). Symptom: Visual inconsistency, some objects pop earlier than others. Solution: Standardize LOD distances globally (all environment objects use same thresholds: 0-15m, 15-40m, 40-80m). Adjust per object class (vehicles = longer distances, small props = shorter).

**Forgetting Shader LOD**: Developer creates material LOD variants with reduced texture resolution but same complex shader. Distant objects still run expensive PBR shader (100+ ALU) despite low-res textures. Symptom: Material LOD implemented, but no GPU performance improvement. Solution: Reduce shader complexity per LOD. LOD0 = Standard Shader, LOD1 = Simplified Shader, LOD2 = Mobile/Diffuse, LOD3 = Unlit.

## Tools & Workflow

**LOD Group Component**: Unity component for automatic LOD management. Add to GameObject (Add Component > LOD Group), configure distance percentages (LOD0 = 100%-60%, LOD1 = 60%-30%, LOD2 = 30%-10%, Cull = <10%), assign renderers per LOD level (drag mesh renderers into LOD slots). Scene view shows LOD preview (colored bars indicate LOD ranges).

**Shader LOD System**: Quality Settings > Shader LOD sets global maximum. Shaders define LOD levels (LOD 200 = simple, LOD 400 = standard, LOD 600 = complex). Unity disables features above threshold. Standard Shader: LOD 200 = no bump, LOD 300 = bump + emission, LOD 400 = full features.

**Texture Streaming**: Quality Settings > Texture Streaming > Enable Texture Streaming. Memory Budget (512MB-2GB), Load Behavior (All Mipmaps Initially vs On Demand). Texture Import > Streaming Mipmaps enables per texture. Profiler > Memory > Texture Streaming shows loaded mipmaps.

**RenderDoc/PIX**: Capture frame, inspect materials per draw call. View texture resolutions (verify distant objects use low-res mipmaps), shader complexity (instruction count per LOD). Validate LOD implementation (LOD0 objects use full textures, LOD2 objects use downsampled textures).

**Unity Profiler**: CPU Profiler > Rendering shows material swaps (Material.SetPass calls). Too many material swaps = performance cost. Memory Profiler shows texture memory per material (verify LOD materials use less memory).

**LOD Analyzer (Asset Store)**: Visualizes LOD coverage. Scene view overlay shows LOD levels per object (color-coded: green = LOD0, yellow = LOD1, red = LOD2). Identify objects missing LOD, optimize LOD distances.

**Simplygon/InstaLOD**: Automatic mesh + material LOD generation. Input high-poly mesh with full material, output simplified LODs with reduced textures. Commercial tools (Simplygon, InstaLOD) or Unity ProBuilder for basic LOD creation.

## Related Topics

- [11.3 Texture Mapping Techniques](11-03-Texture-Mapping-Techniques.md) - Texture sampling
- [7.1 Texture Optimization](07-01-Texture-Optimization.md) - Texture resolution and compression
- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Mesh LOD techniques
- [13.6 Culling Techniques](../chapters/13-Rendering-Techniques.md) - Object culling

---

[← Previous: 11.3 Texture Mapping Techniques](11-03-Texture-Mapping-Techniques.md) | [Next: 11.5 Advanced Texture Techniques →](11-05-Advanced-Texture-Techniques.md)
