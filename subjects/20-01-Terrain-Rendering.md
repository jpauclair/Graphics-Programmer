# 20.1 Terrain Rendering

[← Back to Chapter 20](../chapters/20-Specialized-Rendering.md) | [Main Index](../README.md)

Terrain rendering handles large landscapes: heightmap-based geometry, texture splatting (multiple layers blended), LOD tessellation (adaptive detail), and GPU-optimized rendering.

---

## Overview

Terrain System: Unity Terrain component (heightmap-based, sculpt in Editor, paint textures, place trees/grass). Heightmap: 2D grayscale texture (Y values per XZ position, defines elevation), resolution (513x513 typical, 1025x1025 high detail, 2049x2049 extreme). Texture splatting: multiple terrain layers (grass, rock, dirt, snow), blend based on blend map (splatmap texture, RGBA = 4 layers per texture, multiple splatmaps for more layers). Trees/Details: instanced vegetation (Unity places trees/grass, GPU instancing renders thousands efficiently).

Terrain LOD: Adaptive tessellation (near terrain = high poly, far = low poly), quad-tree subdivision (terrain divided into patches, patches subdivide based on distance, seamless transitions). Unity automatic (Terrain component handles LOD, pixel error setting controls quality vs performance). Custom terrain: manual LOD (multiple mesh resolutions, switch based on distance), or GPU tessellation (hardware tessellation shader, DX11+, dynamic subdivision).

Performance: Terrain expensive (large meshes, multiple texture samples for splatting, vegetation instancing overhead). Optimization: reduce heightmap resolution (513 = fast, 2049 = slow, balance detail vs performance), limit texture layers (4-8 layers max, more = expensive fragment shader), detail distance (grass/rocks render only near camera, cull beyond 50-100m), tree LOD (billboard imposters for distant trees, 3D meshes near camera).

## Key Concepts

- **Heightmap Terrain**: 2D array of height values (Y coordinate per XZ position). Unity stores as 16-bit RAW (0-65535 range, high precision), or texture (grayscale, imported as heightmap). Resolution: 513x513 = 512m×512m terrain at 1m per sample (typical), 1025 = 2x detail, 2049 = 4x detail (expensive). Heightmap editing: Terrain Inspector > Paint Terrain > Raise/Lower (sculpt in Editor, modifies height values), or import (RAW file from WorldMachine, Gaia, external tools). Vertex count: resolution² vertices (513² = 263K verts, 2049² = 4.2M verts, high poly).
- **Texture Splatting**: Blend multiple textures based on blend weights (grass where weight.r, rock where weight.g, etc.). Splatmap: RGBA texture (4 channels = 4 layers per splatmap, Unity supports multiple splatmaps = 8-16 layers total). Shader: samples all layer textures (albedo, normal per layer), blends via splatmap weights: `albedo = tex2D(grass) * splat.r + tex2D(rock) * splat.g + ...`, expensive (N texture samples per layer, 4 layers = 8+ samples for albedo + normals). Painting: Terrain Inspector > Paint Texture (brush paints splatmap, assigns layer weights, visual blending in Editor).
- **Terrain LOD**: Patches (terrain divided into grid of patches, e.g., 16x16 patches), patch LOD (each patch rendered at different detail, near patches = high poly, far = low poly). Unity: Pixel Error setting (Terrain Settings > Pixel Error, 1 = highest quality/slowest, 50 = lowest quality/fastest, controls tessellation threshold). Automatic transitions (Unity blends LOD seams, no visible popping, gradual detail reduction). Custom LOD: not exposed (Unity internal, custom terrain requires manual implementation or third-party assets).
- **Detail Objects**: Grass, rocks, flowers (Unity instancing, thousands of instances). Detail resolution: per-patch resolution (e.g., 128x128 detail samples per patch, density controlled), distance limit (render details only within N meters, Detail Distance setting = 50-150m typical). Billboard grass: mesh grass (3D quads, expensive), billboard grass (2D sprites, cheaper, crossfade to mesh at close range). Performance: GPU instancing (one draw call per detail mesh type, thousands of instances), density limits (reduce density with distance, fewer instances far away).
- **Tree Instancing**: Unity places trees (Paint Trees tool, positions stored), LOD system (LOD Group on tree prefab, LOD0/1/2 meshes + billboard LOD3). Rendering: GPU instancing (single draw call per LOD level, thousands of trees batched), or traditional rendering (individual draw calls, slow). Billboard transition: Tree settings (Billboard Start distance, trees beyond become billboards, 2 tris vs 5,000 tri mesh). Max trees: thousands viable (10,000 trees with instancing = fast, without instancing = slideshow).
- **Basemap**: Low-resolution composite (pre-rendered terrain texture, used at distance). Unity generates basemap (combines all splatmap layers, resolution 512x512-1024x1024), rendered when terrain distance exceeds Basemap Distance (Terrain Settings, e.g., 500m, beyond this distance terrain uses basemap instead of splatmap = single texture sample vs N layers). Performance gain: far terrain cheap (one sample vs 4-8 layer samples), seamless transition (crossfade between splatmap and basemap).

## Best Practices

**Heightmap Optimization:**
- Resolution selection: 513x513 for typical terrain (1km×1km, acceptable detail, fast), 1025 for detailed terrain (close-up views, visible sculpting), 2049 for hero terrain (small area, extreme detail, expensive = use sparingly). Mobile/Switch: 257x257 or 513x513 (lower = fewer verts, better performance). Profiler target: <50K tris on screen (Profiler > Rendering > Triangles, reduce resolution if exceeded).
- Sculpting: Smooth transitions (avoid sharp cliffs = triangulation issues, smooth sculpting better), consistent detail (don't mix extreme high/low detail, uniform sculpting), optimize for gameplay (paths/roads flat = better character movement, steep hills culled at distance).
- External tools: WorldMachine (procedural terrain generation, export heightmap RAW), Gaia (Unity Editor terrain stamping, biome painting), Terrain Composer (node-based terrain, Unity asset). Import: Terrain Settings > Import Raw, assign heightmap file, configure depth (16-bit) and resolution (match terrain size).

**Texture Splatting Optimization:**
- Layer count: 4 layers ideal (one splatmap, cheap), 8 layers acceptable (two splatmaps, moderate cost), 16+ layers expensive (avoid, each splatmap = additional texture samples, fragment shader cost scales linearly). Profiler: GPU Profiler shows terrain fragment cost (high = too many layers, reduce).
- Texture resolution: 1024x1024 per layer (albedo + normal = 2 textures per layer, 4 layers = 8 textures), compress (BC7 for albedo, BC5 for normals, reduce memory). Tiling: textures repeat (layer tile size 15x15m typical, grass small tiles, rock large tiles), balance detail vs repetition (too small = obvious tiling, too large = blurry).
- Triplanar mapping: Sample textures from XYZ axes (blends based on surface normal, eliminates stretching on steep slopes). Implementation: `float3 blendWeights = abs(normal); blendWeights /= dot(blendWeights, 1); float3 xColor = tex2D(layer, worldPos.zy); float3 yColor = tex2D(layer, worldPos.xz); float3 zColor = tex2D(layer, worldPos.xy); color = xColor * blendWeights.x + yColor * blendWeights.y + zColor * blendWeights.z;` Cost: 3x texture samples (expensive, use only on steep terrain or as optional quality setting).
- Layer painting: Use appropriate layers (grass on flat, rock on steep, snow on high elevation), blend smoothly (soft brush, gradual transitions, avoid hard edges), test at distance (terrain viewed from far = blending visible, validate from player perspective).

**LOD Configuration:**
- Pixel Error: 1-5 for high quality (near-perfect tessellation, minimal LOD, expensive), 10-20 for balanced (good quality, reasonable performance), 30-50 for low quality (aggressive LOD, mobile/Switch). Test: adjust in Editor (Terrain Settings > Pixel Error, Scene view shows tessellation changes), profile (Profiler > Rendering > Triangles, lower pixel error = more tris).
- Basemap Distance: 500-1000m typical (terrain beyond uses basemap, closer = splatmaps), shorter for performance (basemap at 200-300m, distant terrain cheap), longer for quality (basemap at 1500m+, splatmap detail visible far away). Mobile: 100-200m (aggressive basemap, distant terrain very cheap).
- Detail Distance: 50-100m for grass/rocks (render details near camera only, cull beyond), 30m for mobile (very short, minimal detail rendering), 150m for PC (extended range, higher quality). Density: near camera 100% density, reduce with distance (50% at half distance, 10% at max distance, automatic LOD).
- Tree Distance: Max Tree Distance setting (trees beyond culled entirely, 500-2000m typical), Billboard Start distance (trees beyond become billboards, 50-150m typical, 3D meshes near, billboards far). Mobile: aggressive culling (Max 300m, Billboard 30m, reduce tree count).

**Detail and Tree Setup:**
- Detail meshes: Low-poly grass (10-20 tris per blade, crossfade to billboard at distance), rock clumps (50-100 tris, instanced), flowers (20-30 tris). GPU Instancing: enable in detail settings (Terrain > Paint Details > Detail Mesh, enable GPU Instancing, mandatory for performance). Render mode: Grass shader (vertex color wind animation, alpha cutout), or VertexLit (simple lighting, cheap).
- Tree prefabs: LOD Group required (LOD0 = 5,000 tris, LOD1 = 2,000, LOD2 = 500, LOD3 = billboard 2 tris), SpeedTree recommended (Asset Store, optimized tree LODs, wind animation). Placement: Paint Trees tool (density, brush size, random rotation/scale), or procedural (script places trees based on terrain height/slope, noise patterns for natural distribution).
- Wind animation: Detail shaders (vertex shader offsets vertices, sine wave based on position + time, simulates wind), Tree shaders (SpeedTree wind = sophisticated, branch/leaf motion, or simple sine wave). Performance: cheap (vertex shader, no physics, pure math).
- Density limits: Details (100-200 instances per m² near camera, 10-20 far away, automatic LOD), Trees (avoid >10,000 visible trees, even with instancing GPU limited, cull aggressively or reduce placement density).

**Platform-Specific:**
- **PC**: High-quality terrain (1025-2049 heightmap, 8-16 layers, Pixel Error 5-10, Detail Distance 100-150m, thousands of trees), basemap 1000m+ (splatmap detail far visible), GPU Instancing (essential, enables high tree/detail counts). Target: 60 FPS with large terrains (profile, adjust settings per scene complexity).
- **Consoles**: Medium-quality terrain (1025 heightmap, 6-8 layers, Pixel Error 15-25, Detail Distance 80-120m, moderate trees), basemap 500m (balance quality/performance), instancing (supported, use for all details/trees). Memory: watch texture memory (splatmaps + layer textures = tens of MB, compress textures, reduce resolution if needed).
- **Switch**: Low-quality terrain (513 heightmap, 4-6 layers max, Pixel Error 30-50, Detail Distance 50m, minimal trees), basemap 200m (aggressive, distant terrain very cheap), instancing (supported but limit counts, <5,000 trees total). Simplify: fewer layers (4 layers = one splatmap only, avoid multiple), aggressive culling (terrain beyond 500m culled or ultra-low LOD).
- **Mobile**: Very low-quality terrain (257-513 heightmap, 4 layers only, Pixel Error 50+, Detail Distance 30m, very few trees <1,000), basemap 100m (near basemap transition, splatmap minimal), instancing (supported on high-end, but limit instances <1,000 details). Consider alternatives: static mesh terrain (baked, no dynamic LOD overhead, single mesh for small area), or no terrain (pre-modeled environments, avoid Terrain component entirely).

## Common Pitfalls

**Too Many Texture Layers**: Developer adds 12 terrain layers (thinking variety needed), terrain shader samples 12 textures (albedo + normal = 24 samples), frame rate drops (fragment shader expensive, texture bandwidth saturated). Symptom: GPU bottleneck (Profiler shows terrain rendering expensive, fragment-bound), overdraw on terrain (Frame Debugger shows many texture samples). Solution: Reduce layers (4-6 layers sufficient, combine similar materials, e.g., DirtLight + DirtDark = one Dirt layer with color variation), or layer distance culling (far layers disabled, only near terrain uses all layers, basemap beyond).

**High Heightmap Resolution**: Developer creates 2049x2049 terrain (thinking detail needed), vertex count = 4.2M vertices, terrain rendering slow (GPU vertex processing, tessellation expensive). Symptom: High triangle count (Profiler shows millions of tris, terrain dominates), low frame rate even with no other objects. Solution: Reduce resolution (1025 = 1M verts, acceptable detail, 4x faster than 2049), or split terrain (multiple smaller terrains, e.g., four 1025 terrains instead of one 2049, each culled independently, better performance).

**Detail Overdraw**: Developer places dense grass details (1,000 grass instances per m², alpha cutout textures), overdraw extreme (thousands of overlapping transparent quads, GPU processes all layers). Frame rate collapses. Symptom: GPU fragment-bound (Profiler shows terrain details expensive), Scene view overdraw mode = bright red (extreme overdraw). Solution: Reduce density (100-200 instances per m² max, still looks dense), increase Detail Distance fade (grass fades out gradually at distance, reduces instance count), or simpler geometry (fewer tris per grass blade, 5-10 tris instead of 20).

**Missing Instancing**: Developer disables GPU Instancing on detail meshes (thinking unnecessary or not understanding setting), Unity renders each detail individually (1,000 grass instances = 1,000 draw calls, CPU bottleneck). Frame rate tanks. Symptom: High SetPass/Draw calls (Profiler shows thousands of draw calls, CPU bound), details rendering slowly. Solution: Enable GPU Instancing (Terrain > Paint Details > Edit Details > Enable GPU Instancing, mandatory for performance, one draw call per detail type instead of thousands).

## Tools & Workflow

**Terrain Creation**: GameObject > 3D Object > Terrain (creates Terrain GameObject with Terrain component). Settings: Terrain Settings (Inspector > gear icon), configure Width/Length (1000m typical, large open world), Height (600m typical, vertical range), Heightmap Resolution (513/1025/2049, vertex count). Tools: Terrain Inspector (Paint Terrain dropdown, Raise/Lower, Set Height, Smooth, Paint Texture, Place Trees, Paint Details).

**Heightmap Sculpting**: Terrain Inspector > Paint Terrain > Raise/Lower Terrain (sculpt tool). Brush: size (10-100, smaller = detailed sculpting, larger = broad shaping), opacity (strength, 1-100, lower = gradual changes), shape (circle, square, custom brush textures). Shortcuts: Shift+LMB = lower terrain (inverse of raise), Smooth tool (Paint Terrain > Smooth Height, blends height values, removes sharp edges). Flatten: Set Height tool (Paint Terrain > Set Height, sets specific height value, useful for roads/platforms).

**Texture Layer Setup**: Terrain Inspector > Paint Texture > Create Layer (adds terrain layer asset). Layer settings: Diffuse (albedo texture), Normal Map (tangent-space normals), Mask Map (optional, metallic/AO/height/smoothness packed), Tile Size (15x15m typical, texture repeat size). Painting: select layer (click layer thumbnail), brush (size/opacity), paint on terrain (LMB paints, weights in splatmap updated).

**Detail Painting**: Terrain Inspector > Paint Details > Edit Details (add detail mesh or texture). Detail Mesh: assign prefab (grass prefab, rock prefab), configure (Render Mode = Grass/Vertex Lit, Density = 1-10, enable GPU Instancing). Painting: Target Strength (density, 1-100), brush (size, terrain turns green with grass instances). Erase: Shift+LMB (removes details). Batch: Paint Details > Details > Remove (removes all details, or select type to remove specific).

**Tree Painting**: Terrain Inspector > Paint Trees > Edit Trees (add tree prefab). Tree settings: prefab (LOD Group tree prefab, SpeedTree or custom), Bend Factor (wind bending amount). Painting: brush (size, density, random rotation/scale toggles), LMB places trees (multiple per click based on density). Mass Place: Paint Trees > Settings > Mass Place Trees (places trees across entire terrain, density-based, random distribution). Remove: Shift+LMB (removes trees), or Remove Trees button (clears all).

**Terrain Settings**: Terrain component > Terrain Settings (gear icon). Rendering: Pixel Error (1-50, LOD quality), Basemap Distance (200-2000m, basemap transition), Cast Shadows (on/off, terrain casts shadows = expensive, off = faster). Details: Detail Distance (30-150m, grass/rock render distance), Detail Density (0-1, scales detail instance counts). Trees: Max Mesh Trees (tree count limit, beyond = billboard only), Billboard Start (50-150m, 3D to billboard distance). Preserve: Preserve Tree Prototype Layers (trees respect layer settings, or inherit terrain layer).

**Profiling Terrain**: Unity Profiler > Rendering: Triangles (terrain vertex count, reduce if millions), SetPass Calls (draw calls, instancing should keep low), Batches (batching efficiency). GPU Profiler: shows terrain rendering cost (vertex + fragment passes, splatmap sampling expensive). Frame Debugger: expand terrain draws (shows LOD patches, texture bindings, detail/tree instancing draws). Optimize: adjust Pixel Error (higher = fewer tris), reduce layers (fewer splatmaps = cheaper fragment), cull details/trees (shorter distances = fewer instances).

## Related Topics

- [17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) - LOD systems
- [11.1 Material System Architecture](11-01-Material-System-Architecture.md) - Material/texture basics
- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Mesh performance
- [20.2 Vegetation Rendering](20-02-Vegetation-Rendering.md) - Tree/grass rendering

---

[← Previous: Chapter 19](../chapters/19-Visual-Effects.md) | [Next: 20.2 Vegetation Rendering →](20-02-Vegetation-Rendering.md)
