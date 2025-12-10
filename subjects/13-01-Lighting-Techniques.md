# 13.1 Lighting Techniques

[← Back to Chapter 13](../chapters/13-Rendering-Techniques.md) | [Main Index](../README.md)

Lighting techniques balance real-time dynamic lights, baked lightmaps, light probes, and mixed lighting to achieve visual quality while maintaining performance across platforms.

---

## Overview

Unity supports three lighting modes: Realtime (dynamic lights computed every frame, fully interactive but expensive), Baked (pre-computed lighting stored in lightmaps, fast runtime but static-only), and Mixed (combines realtime and baked, dynamic objects receive baked indirect light). Realtime lighting calculates lighting per frame: for each pixel, loop through lights (directional, point, spot), calculate diffuse/specular contribution, accumulate result. Cost: N lights * M pixels * lighting calculations (expensive, limits light count: 4-8 per-pixel lights typical).

Baked lighting offloads computation to offline lightmap baking: Unity calculates lighting for static geometry (global illumination, area lights, bounced light), stores results in lightmaps (2D textures applied to static meshes). Runtime: sample lightmap (single texture fetch), no lighting calculations (fast, supports hundreds of lights). Limitations: static only (baked objects can't move), requires unique UVs (UV1 channel, non-overlapping), storage overhead (lightmaps = 5-50MB per scene).

Mixed lighting bridges gap: static objects use baked lighting (lightmaps), dynamic objects use Light Probes (baked indirect light sampled at object position) + realtime direct light (calculated per frame). Result: consistent lighting between static/dynamic objects (both receive bounced light from environment), performance balance (dynamic objects only calculate direct lighting, not expensive GI). Common configuration: one directional light (sun, Mixed mode) + few realtime point/spot lights (interactive sources: explosions, muzzle flashes).

## Key Concepts

- **Realtime Lighting**: Dynamic lights calculated per frame. Supports moving lights, moving objects, animated lighting. Expensive: 4-8 per-pixel lights max (forward rendering), or unlimited (deferred rendering but higher base cost). Configure via Light > Mode > Realtime.
- **Baked Lighting**: Pre-computed lighting stored in lightmaps. Static geometry only (objects marked Static). Fast runtime (single lightmap sample), supports unlimited lights, global illumination (bounced light). Configure via Light > Mode > Baked, Lighting window > Generate Lighting.
- **Mixed Lighting**: Combines baked and realtime. Static objects use baked lighting, dynamic objects use Light Probes (baked indirect) + realtime direct light. Configure via Light > Mode > Mixed. Modes: Baked Indirect (default), Subtractive (mobile-friendly), Shadowmask (high-quality shadows).
- **Light Probes**: Sample points for baked indirect lighting. Place probes in scene (volumes, grids), Unity bakes indirect light at probe positions, dynamic objects sample nearest probes (interpolates lighting). Enables dynamic objects to receive GI (bounce lighting, ambient color).
- **Reflection Probes**: Cubemaps capturing environment reflections. Baked or realtime. Objects sample probe for specular reflections (PBR materials). Provides realistic reflections without screen-space techniques (SSR). Place probes in reflective areas (water, mirrors, shiny floors).

## Best Practices

**Lighting Mode Selection:**
- Realtime only: Fast iteration (no baking wait), fully dynamic (moving lights, interactive). Use for: prototyping (quick testing), VR teleportation (lighting needs to update instantly), small enclosed spaces (few lights, low pixel count). Expensive for outdoor scenes (sun affects entire screen).
- Baked only: Best performance (single lightmap sample per pixel). Use for: mobile (limited GPU), static scenes (architectural visualization, museums), VR (consistent 90fps critical). No dynamic shadows/lighting (use fake dynamic elements: animated textures, particle effects).
- Mixed (Baked Indirect): Balanced approach. Static objects baked (GI, indirect), dynamic objects realtime (direct light) + probes (indirect). Use for: most games (characters move, environments static), outdoor scenes (baked GI for terrain, realtime for characters), console/PC targets.
- Mixed (Shadowmask): High-quality shadows for static casters. Bakes static shadows into shadowmask texture, realtime shadows for dynamic casters. Use for: high-end PC/console (detailed static shadows), when shadow distance limited (static shadows beyond realtime range).

**Light Probe Placement:**
- Volume coverage: Place probes in volumes where dynamic objects move (player nav mesh, enemy spawn zones, physics areas). Probes outside these volumes wasted (never sampled).
- Lighting transitions: Dense probes at lighting boundaries (shadow edges, doorways, room transitions). Coarse probes in uniform areas (middle of lit room, open field). Captures lighting gradients accurately.
- Height variation: Place probes at multiple heights (ground level, chest height, ceiling height). Dynamic objects at different Y positions sample correct lighting (character on ground vs flying enemy).
- Avoid geometry: Don't place probes inside geometry (walls, floors, props). Unity samples lighting at probe position (inside geometry = incorrect lighting, black/wrong colors). Use automatic probe placement (Light Probe Group > Edit > Auto Generate).

**Reflection Probe Configuration:**
- Baked vs Realtime: Baked probes = offline rendering (slow bake, no runtime cost). Realtime probes = refresh every frame or on-demand (runtime cost, dynamic reflections). Use baked for static scenes (most common), realtime for mirrors/water (reflections update).
- Resolution and HDR: Higher resolution = sharper reflections but more memory (128x128 = 1MB, 512x512 = 16MB per probe). HDR = supports bloom/emissive reflections (realistic) but 2x memory. Balance quality vs memory (256x256 typical, 128x128 mobile).
- Box vs Sphere: Box projection = accurate for interiors (room-shaped bounds, parallax-corrected reflections). Sphere projection = simple, outdoor scenes. Use box for architectural interiors, sphere for outdoor environments.
- Blend distance: Overlapping probes blend (smooth transition between probe influences). Set Blend Distance = 10-20% of probe bounds. Avoids visible seams (probe boundaries).

**Lightmap Resolution:**
- Texels per unit: Lightmapping Settings > Lightmap Resolution (default 40 texels/unit). Higher = more detail but larger lightmaps (80 texels/unit = 4x memory). Lower = faster baking, smaller lightmaps but blurry (20 texels/unit for backgrounds).
- Per-object Scale: Mesh Renderer > Lightmap Scale in Lightmaps. Multiply global resolution (important objects = 2x scale, backgrounds = 0.5x). Prioritizes detail budget (hero areas detailed, distant areas coarse).
- Lightmap packing: Unity packs multiple objects into lightmap atlases (2048x2048 or 4096x4096 textures). Efficient packing reduces lightmap count (fewer textures = less memory). Enable Progressive GPU Lightmapper (faster baking, better packing).
- Multiple lightmap sets: Separate lightmaps per quality tier (Low = 1K lightmaps, High = 4K lightmaps). Load appropriate set at runtime (Quality Settings > Lightmap Resolution Multiplier).

**Lighting Optimization:**
- Reduce realtime light count: Target 1 directional (sun) + 2-4 point/spot lights (important sources). Additional lights = baked or Light Probes. Each realtime light adds GPU cost (shadow rendering, lighting calculations).
- Shadow distance: Quality Settings > Shadow Distance = 50-100m. Shadows beyond distance not rendered (performance savings). Combine with Shadowmask (static shadows beyond realtime range).
- Light culling: Unity culls per-object lights (only lights affecting object). Reduce light ranges (shorter range = fewer objects affected = less GPU). Point lights expensive (affect sphere, many objects). Spot lights cheaper (cone, fewer objects).
- Lightmap compression: Texture Import > Lightmap Compression (Normal Quality, High Quality, or Low Quality). Compresses lightmaps (BC6H on PC, ASTC on mobile). Reduces memory (4x-8x compression) with minimal quality loss.

**Platform-Specific:**
- **PC**: Full lighting support. Mixed mode with Shadowmask (detailed static shadows), 4-8 realtime lights (forward rendering), or unlimited (deferred rendering). 4K lightmaps acceptable (2048-4096px), BC6H compression.
- **Consoles**: Mixed mode (Baked Indirect or Shadowmask). 4-6 realtime lights typical. 2K lightmaps (1024-2048px). Use clustered lighting (HDRP) for many lights efficiently. BC6H compression.
- **Switch**: Baked or Mixed (Subtractive) mode. 1-2 realtime lights max. 1K lightmaps (512-1024px), ASTC compression. Disable realtime GI (too expensive). Favor baked lighting (performance critical).
- **Mobile**: Baked mode strongly preferred. Mixed (Subtractive) acceptable (cheapest mixed mode). 0-1 realtime lights (main directional light only). 512-1K lightmaps, ASTC 8x8 compression. No realtime shadows (baked shadows only).

## Common Pitfalls

**Too Many Realtime Lights**: Developer adds 20 realtime point lights (decorative lamps, torches). Forward rendering limits to 4-8 per-pixel lights (additional lights = per-vertex, low quality). GPU bound, low frame rate. Symptom: Profiler shows expensive lighting, many lights in scene. Solution: Bake decorative lights (Light > Mode > Baked). Reserve realtime for interactive lights (player flashlight, muzzle flashes, explosions). Use Baked + Emissive materials for static glowing objects.

**No Light Probes for Dynamic Objects**: Developer uses Mixed lighting but doesn't place Light Probes. Dynamic objects (characters, props) don't receive GI (look flat, wrong colors, too dark in shaded areas). Symptom: Characters darker than surroundings, don't match static environment lighting. Solution: Add Light Probe Group (GameObject > Light > Light Probe Group). Place probes throughout playable area (automatic placement or manual grid). Bake lighting (Lighting window > Generate Lighting).

**Overlapping Lightmaps**: Developer marks all objects Static without proper UVs. Objects use overlapping UV0 (tiling textures), Unity generates bad lightmap UVs (overlaps, incorrect lighting). Symptom: Black splotches, incorrect lighting, artifacts on meshes. Solution: Use UV1 for lightmaps. Mesh Import > Generate Lightmap UVs (creates non-overlapping UV1 automatically). Or manually unwrap UV1 in DCC tool (Blender, Maya).

**Excessive Lightmap Resolution**: Developer sets Lightmap Resolution = 100 texels/unit (thinks more = better). Lightmaps explode in size (50-100MB per scene), baking takes hours, texture memory exhausted. Symptom: Long bake times, huge lightmap textures (Memory Profiler), out of memory on builds. Solution: Reduce to 20-40 texels/unit (sufficient for most games). Use per-object Scale (important objects = higher, backgrounds = lower). Compress lightmaps (BC6H, ASTC).

## Tools & Workflow

**Lighting Window**: Window > Rendering > Lighting. Configure lightmapping settings (resolution, bounces, denoising), bake lighting (Generate Lighting button), manage lightmaps (generated textures, memory usage). Auto Generate for realtime baking (slow, disable for large scenes).

**Light Explorer**: Window > Rendering > Light Explorer. Lists all lights in scene (type, mode, intensity, shadows). Bulk edit lights (select multiple, change settings). Identify realtime lights (candidates for baking), excessive shadow distances.

**Progressive Lightmapper**: Default lightmapper (Unity 2020+). GPU-accelerated baking (uses GPU ray tracing), progressive preview (see results during baking), denoising (clean results with fewer samples). Configure in Lighting window > Lightmapping Settings.

**Light Probe Visualization**: Scene view > Gizmos > Light Probes (show probe positions, influence zones). Edit mode: Light Probe Group > Edit Light Probes (move probes interactively). Runtime: Rendering Debugger > Lighting > Light Probe visualization (see interpolated values on objects).

**Reflection Probe Debugger**: Rendering Debugger (URP/HDRP) > Lighting > Reflections. Shows active probes, preview cubemaps, blend weights. Verify probes covering scene (no missing reflections), check resolution (sufficient detail).

**Memory Profiler**: Captures lightmap memory usage. Native Memory > Texture2D shows lightmaps (size, compression format). Identify large lightmaps (candidates for resolution reduction, aggressive compression).

**Frame Debugger**: View lighting per draw call. Shows lights affecting object (realtime lights list), lightmap sampling (UV coordinates, lightmap texture). Debug lighting issues (wrong lightmap, incorrect UV channel).

## Related Topics

- [13.2 Shadow Techniques](13-02-Shadow-Techniques.md) - Shadow rendering
- [14.1 Lightmapping Fundamentals](../chapters/14-GI-And-Lightmapping.md) - Baked lighting details
- [14.3 Light Probes](../chapters/14-GI-And-Lightmapping.md) - Dynamic object lighting
- [12.3 Lighting Models](12-03-Lighting-Models.md) - Shader lighting calculations

---

[← Previous: Chapter 12](../chapters/12-Shader-Programming.md) | [Next: 13.2 Shadow Techniques →](13-02-Shadow-Techniques.md)
