# 14.3 Light Probes and Reflection

[← Back to Chapter 14](../chapters/14-Global-Illumination-And-Lightmapping.md) | [Main Index](../README.md)

Light probes capture lighting at points in 3D space for dynamic objects, while reflection probes capture environment reflections for realistic metallic and glossy surfaces.

---

## Overview

Light probes enable dynamic objects to receive baked lighting: sample lighting at probe positions (not textures like lightmaps), interpolate between nearby probes (smooth lighting as object moves). Dynamic objects (characters, vehicles, physics props) can't use lightmaps (lightmaps require static UV mapping), so probes provide ambient lighting, indirect lighting, and light direction. Without probes, dynamic objects render flat (no GI, only direct realtime lights). With probes, dynamic objects match baked environment lighting (color bleeding, ambient occlusion, realistic integration).

Reflection probes capture surrounding environment as cubemap: 360° image from probe position, used for reflections on metallic/glossy surfaces. Real-time Screen Space Reflections (SSR) expensive or limited (only reflects on-screen objects), reflection probes provide full environment reflections (baked cubemaps, zero runtime rendering cost). Essential for PBR materials (metallic surfaces need accurate reflections). Types: Baked (precomputed, static environment), Realtime (rendered every frame, dynamic environment), Custom (manually assigned cubemap).

Light probe placement: volumetric coverage (place probes in 3D grid throughout scene), higher density in areas with lighting variation (near lights, color transitions, shadow boundaries), lower density in uniform areas (open sky, flat lighting). Typical spacing: 2-10 meters (dense in interiors, sparse in exteriors). Runtime samples 4 nearby probes (tetrahedral interpolation, smooth lighting). Reflection probes: key locations (room centers, intersection points), overlap regions (blend between probes for smooth transitions). Typical count: 10-50 per scene (depends on scene size, reflection importance).

## Key Concepts

- **Light Probe Group**: Component containing multiple probe positions (GameObject with LightProbeGroup component). Each probe stores: spherical harmonic coefficients (SH9 = 27 floats per probe, RGB lighting from all directions), indirect lighting, ambient occlusion, dominant light direction. Baked during lightmap baking (probes sample GI at their positions). Dynamic objects query nearest probes (interpolate SH coefficients, apply to object).
- **Spherical Harmonics (SH)**: Mathematical representation of lighting (captures lighting from all directions in compact form). SH9 (9 coefficients × 3 RGB = 27 floats) encodes low-frequency lighting (ambient, indirect, soft shadows). Fast to evaluate (dot product in shader), smooth lighting (no high-frequency details like sharp shadows). Unity uses SH for probe lighting (efficient, sufficient quality for ambient).
- **Reflection Probe**: Captures environment as cubemap (6 faces, square textures). Baked mode (precomputed, static reflections), Realtime mode (renders every frame or on-demand, dynamic reflections), Custom mode (manually assigned cubemap asset). Influences nearby renderers (within probe bounds), blended when overlapping (smooth transitions between probe regions). Used for: reflections on metallic PBR materials, skybox reflections, environment lighting.
- **Probe Blending**: Interpolates between multiple probes (light probes: tetrahedral interpolation of 4 nearest, reflection probes: box projection or sphere blending). Smooth transitions as objects move (no popping between probes). Blend weights based on distance (closer probes = higher weight). Light probe blending automatic (Unity finds 4 nearest), reflection blending configurable (Importance parameter for priority, blending regions for overlap).
- **Proxy Volume**: Reflection probe feature (box or sphere defining reflection parallax). Corrects reflections for interior spaces (without proxy, infinite distance reflections = incorrect for rooms). Proxy volume = room bounds (reflections projected onto proxy, accurate for walls/floor/ceiling). Essential for interiors (without proxy, wall reflections look like outdoor reflections).

## Best Practices

**Light Probe Placement:**
- Volumetric coverage: Place probes in 3D grid (x, y, z spacing). Cover entire playable area (if character can reach, needs probes). Y-axis important (vertical lighting variation: ground vs ceiling, different ambient). Typical grid: 5m x 3m x 5m (horizontal 5m, vertical 3m).
- Density at transitions: High density (1-2m spacing) at: light boundaries (shadow edges), color transitions (red room to blue room), vertical changes (stairs, ramps). Low density (10m+ spacing) in: uniform lighting (open outdoor, consistent indoor), areas without dynamic objects. Adaptive placement (dense where needed, sparse elsewhere) saves memory.
- Avoid probe clumping: Don't place many probes at same position (redundant data, wastes memory). Use Light Probe Group editing (Scene view gizmos, add/remove/move probes). Unity visualizes probes (yellow spheres in scene view), shows interpolation (select object, see which probes influence it).
- Probe geometry constraints: Place probes in open space (not inside walls, geometry). Probes inside geometry = incorrect lighting (sample wall interior, occluded lighting). Use collision detection script (ray cast from probe, if hits geometry = invalid placement, move probe). Automated tools: Light Probe Group > Edit > Select All > Distribute (automatic grid generation).

**Reflection Probe Configuration:**
- Probe placement: Key locations (room centers, hallway intersections, outdoor gathering points). One probe per "reflection zone" (room, area with distinct environment). Probes overlap at boundaries (30-50% overlap, smooth blending). Avoid: too many probes (memory cost, blending overhead), too few (incorrect reflections, visible transitions).
- Baked vs realtime: Baked (static environment, zero runtime cost, best quality). Realtime (dynamic environment changes: moving objects, day/night transitions, costs per-frame rendering). Most scenes = baked (static buildings, environment). Realtime for: specific interactive objects (player can move/destroy environment), limited count (1-5 realtime probes max, expensive).
- Resolution settings: Reflection Probe > Resolution (128, 256, 512, 1024). Higher = sharper reflections + more memory (1024 cubemap = 6MB uncompressed). Typical: 128-256 for distant/unimportant probes, 512 for important (hero areas), 1024 for very close/highly visible. Total memory = probe count × resolution (50 probes × 512 = 75MB).
- Box projection: Enable for interiors (Reflection Probe > Box Projection, set Bounds to room size). Projects reflections onto proxy box (accurate for room geometry). Without projection, infinite distance reflections (wall looks like sky, incorrect). Essential for closed spaces (rooms, corridors), disable for exteriors (outdoor, open areas).

**Probe Baking Workflow:**
- Bake together: Light probes and reflection probes bake with lightmaps (Lighting window > Generate Lighting). Probes sample baked GI (consistent with lightmaps). Ensure all probes in scene (LightProbeGroup, ReflectionProbe components) before baking.
- Validate coverage: Select dynamic object (character, prop) > Scene view shows probe influences (yellow lines to 4 nearest light probes, reflection probe bounds). Verify: object always has 4 probes (green lines = good coverage), if no probes = add probes in area, if 1-3 probes = poor interpolation (add more probes for tetrahedral).
- Reflection probe preview: Select ReflectionProbe > Inspector shows cubemap preview. Verify: reflections capture correct environment (no black faces, no missing geometry), resolution sufficient (no pixelation in preview). Re-bake if incorrect (move probe, change resolution, regenerate).
- Ambient probe: Scene's overall ambient lighting (baked automatically, no GameObject). Used when: object outside all light probe groups (fallback ambient), skybox lighting (contributes to ambient probe). Lighting window > Environment > Ambient Source (Skybox or Color, affects ambient probe).

**Optimizing Probe Count:**
- Light probe memory: 27 floats per probe (SH9 RGB) = 108 bytes/probe. 1000 probes = 108KB (negligible). Cost: CPU interpolation (4 probes per object per frame, fast). Typical scenes: 500-2000 probes (acceptable, minimal overhead). Reduce only if: 10,000+ probes (excessive, re-evaluate placement), targeting very low-end mobile (reduce to 200-500).
- Reflection probe memory: Resolution-dependent (128 = 0.5MB, 256 = 2MB, 512 = 8MB, 1024 = 32MB per probe, BC6H compression = 1/6th). 50 probes × 512 = 400MB uncompressed, 70MB compressed. Reduce: lower resolution (256 vs 512), fewer probes (10-20 key probes), compress aggressively (BC6H for PC, ASTC for mobile).
- Probe importance: Reflection Probe > Importance (0-1, higher priority in blending). Key probes = 1 (hero areas, player-visible), background = 0.5 (less important). Blending uses importance (high-importance probes dominate when overlapping). Optimize: fewer high-importance probes (expensive, dominant), many low-importance (cheap, fill gaps).
- Runtime updates: Realtime reflection probes expensive (renders scene 6 times per probe per frame, massive cost). Minimize: 0-1 realtime probes (player-specific, character's immediate area), update infrequently (every 5-10 frames, not every frame), lower resolution (128-256, fast rendering). Most games = 100% baked probes (static environment, zero runtime cost).

**Platform-Specific:**
- **PC**: Many probes (1000+ light probes, 50-100 reflection probes). High-resolution reflections (512-1024 cubemaps). BC6H compression (HDR, high quality). Realtime probes possible (1-5, GPU handles rendering). Blending quality high (smooth transitions, importance-weighted).
- **Consoles**: Moderate probe count (500-1000 light probes, 20-50 reflection probes). Medium-resolution reflections (256-512). BC6H compression. Limited realtime probes (0-2, performance budget tight). Balance quality/memory (fixed memory pool, shared with lightmaps/textures).
- **Switch**: Low probe count (200-500 light probes, 10-20 reflection probes). Low-resolution reflections (128-256). ASTC compression (smaller than BC6H). No realtime probes (performance insufficient). Aggressive optimization (minimal probes, low resolution, prioritize performance over reflection quality).
- **Mobile**: Minimal probes (100-300 light probes, 5-10 reflection probes). Very low-resolution reflections (128 only). ASTC 8x8 compression (aggressive, small file size). No realtime probes. Avoid complex blending (limit overlapping probes, simplified interpolation). Memory critical (probes compete with textures/lightmaps for budget).

## Common Pitfalls

**No Light Probe Coverage**: Developer places dynamic characters in scene without light probes. Characters render with default lighting (flat grey, no GI). Looks incorrect (static environment lit beautifully, characters look pasted-in). Symptom: Dynamic objects appear flat, no color bleeding from environment, incorrect brightness. Solution: Add LightProbeGroup component, place probes throughout playable area (5m grid), bake lighting. Verify coverage (select character, see yellow probe lines in scene view).

**Reflection Probes Without Box Projection**: Developer places reflection probe in room, leaves Box Projection disabled. Reflections on metallic surfaces show infinite distance (wall reflects outdoor sky, not room interior). Looks wrong (room walls should reflect other walls, floor, ceiling). Symptom: Reflections don't match environment (outdoor sky visible in indoor reflections, incorrect perspective). Solution: Enable Box Projection (Reflection Probe component), set Bounds to room size (box enclosing room). Reflections now project onto room geometry (correct interior reflections).

**Excessive Reflection Probe Count**: Developer places 200 reflection probes (one per room, hallway, area) at 1024 resolution. Memory usage explodes (200 × 32MB = 6.4GB uncompressed, 1GB compressed). Loading times increase drastically, memory errors on consoles. Symptom: Long level loads (30+ seconds), out-of-memory crashes, huge build size. Solution: Reduce probe count (20-50 key probes, focus on hero areas), lower resolution (256-512), consolidate probes (one probe per large area, not per small room). Monitor memory (Profiler > Rendering > Reflection Probes).

**Light Probes Inside Geometry**: Developer auto-generates probe grid, many probes placed inside walls, floors, ceilings (geometry not considered). Probes sample interior of walls (dark, incorrect lighting), dynamic objects receive wrong lighting (too dark or incorrect color). Symptom: Dynamic objects too dark in specific areas, lighting pops (moving from good probe to bad probe). Solution: Manually adjust probe positions (Edit Bounding Box tool in scene view, move probes out of geometry), or script validation (ray cast from probe, if inside geometry = move probe). Unity doesn't auto-detect geometry (manual placement required).

## Tools & Workflow

**Light Probe Group Component**: GameObject > Light > Light Probe Group. Scene view shows probes (yellow spheres), edit mode (add/remove/move probes with mouse). Edit Bounding Box (generates grid of probes within bounds, spacing adjustable). Select All/Delete (bulk operations). Inspector shows probe count, SH coefficients (after baking).

**Reflection Probe Component**: GameObject > Light > Reflection Probe. Type (Baked/Realtime/Custom), Resolution (128-1024), HDR (enables HDR reflections), Box Projection (parallax correction), Importance (blending priority). Gizmo shows: yellow box (probe bounds, influences renderers inside), white box (proxy volume, if Box Projection enabled). Inspector shows cubemap preview (after baking).

**Light Probe Visualization**: Select dynamic object (with MeshRenderer) > Scene view shows probe influences (yellow lines to 4 nearest light probes). Verify coverage (4 probes = good, fewer = poor interpolation, none = no GI). Lighting > Debug Settings > Light Probes (color overlay, shows probe density).

**Reflection Probe Debug**: Scene view > Shading Mode > Reflection Probes (shows which probe influences each surface, color-coded). Verify: correct probe affecting each object (not wrong probe from different room), blend regions smooth (no hard transitions). Frame Debugger shows reflection probe cubemaps (texture preview, verify content).

**Baking Together**: Lighting window > Generate Lighting (bakes lightmaps + light probes + reflection probes simultaneously). Ensures consistency (probes sample same GI as lightmaps). Progress bar shows: Lightmap baking → Light probe baking → Reflection probe baking (sequential stages).

**Unity Profiler**: Profiler > Rendering > Reflection Probes (memory usage, count, resolution breakdown), Lighting (light probe interpolation time, typically <0.1ms). Memory Profiler > Reflection Probe Textures (per-probe memory, cubemap sizes). Identify excessive probes (too many, too high resolution).

## Related Topics

- [14.1 Lightmapping Fundamentals](14-01-Lightmapping-Fundamentals.md) - Static lighting
- [11.2 PBR Properties](11-02-PBR-Material-Properties.md) - Metallic reflections
- [13.1 Lighting Techniques](13-01-Lighting-Techniques.md) - Baked vs realtime
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Probe memory management

---

[← Previous: 14.2 Baking Systems](14-02-Baking-Systems.md) | [Next: 14.4 Real-Time Global Illumination →](14-04-Real-Time-Global-Illumination.md)
