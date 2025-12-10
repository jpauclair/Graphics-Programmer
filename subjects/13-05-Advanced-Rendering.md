# 13.5 Advanced Rendering

[← Back to Chapter 13](../chapters/13-Rendering-Techniques.md) | [Main Index](../README.md)

Advanced rendering techniques include deferred rendering, forward+ rendering, tile-based rendering, compute-based culling, and GPU-driven pipelines for high object counts and complex lighting.

---

## Overview

Rendering pipelines determine how scenes process lighting and geometry. Forward rendering calculates lighting per object (vertex/fragment shader evaluates all lights), simple but expensive for many lights (N objects * M lights = N*M lighting calculations). Deferred rendering separates geometry and lighting: render geometry to G-buffer (position, normal, albedo textures), then lighting pass reads G-buffer and calculates lighting per pixel (screen-space, decoupled from geometry). Benefits: constant lighting cost regardless of geometry (lights evaluated once per affected pixel), supports hundreds of lights efficiently. Limitations: transparent objects incompatible (can't write to G-buffer), bandwidth-heavy (read/write multiple textures), MSAA expensive (G-buffer = multiple render targets at MSAA resolution).

Forward+ (Forward Plus, Tiled Forward) hybridizes approaches: renders geometry in forward pass (like forward rendering), but uses compute shader to bin lights into screen tiles (16x16 or 32x32 pixel tiles). Shader accesses tile's light list (only lights affecting tile), evaluates lighting (reduced light iterations). Benefits: supports transparency (forward-compatible), less bandwidth than deferred (no G-buffer), hundreds of lights (culled per tile). Unity HDRP uses Forward+ for transparent objects, deferred for opaques.

GPU-driven rendering eliminates CPU overhead: GPU generates draw calls (indirect rendering), performs culling (frustum, occlusion via compute shaders), calculates LODs (distance-based LOD selection on GPU). CPU uploads static data once (mesh buffers, instance data), GPU autonomously renders (zero CPU per-frame overhead). Enables massive scenes (10,000-1,000,000 objects), eliminates CPU bottleneck (all culling/LOD on GPU). Requires modern GPU features (compute shaders, indirect rendering, shader model 5.0+).

## Key Concepts

- **Deferred Rendering**: Geometry pass writes G-buffer (albedo, normal, depth, metallic, roughness to multiple render targets), lighting pass reads G-buffer and calculates lighting (screen-space). Decouples geometry from lighting (light count independent of object count). Built-in Pipeline and HDRP support deferred.
- **Forward+ Rendering**: Tiles screen into grid (16x16 pixels), compute shader bins lights per tile (frustum culling per tile), fragment shader reads tile's light list and evaluates lighting. Supports transparency (forward rendering), efficient multi-light (culled per tile). HDRP uses for transparent objects.
- **G-Buffer**: Multiple render targets storing surface properties. Typical layout: RT0 = albedo (RGB) + occlusion (A), RT1 = normal (RGB) + smoothness (A), RT2 = emission (RGB), RT3 = depth. Lighting pass samples G-buffer, calculates lighting, outputs final color.
- **Clustered Rendering**: 3D version of Forward+ (tiles frustum in X, Y, Z). Divides view frustum into 3D clusters (16x16 pixels * depth slices), bins lights per cluster. More accurate culling than 2D tiles (depth-aware, rejects far lights). HDRP clustered for point/spot lights.
- **GPU-Driven Rendering**: GPU autonomously generates draw calls (DrawMeshInstancedIndirect), performs culling (compute shader frustum/occlusion culling), calculates LODs (distance-based on GPU). CPU uninvolved in rendering loop (uploads data once, GPU handles everything). Massive scalability (100,000+ objects).

## Best Practices

**Deferred Rendering Usage:**
- Enable for opaque objects: Deferred excels with many lights (100+ lights, constant cost per light). Unity Built-in Pipeline: Player Settings > Rendering Path = Deferred. HDRP: defaults to deferred for opaques.
- G-buffer format optimization: Reduce G-buffer textures (fewer render targets = less bandwidth). HDRP allows customizing G-buffer layout (omit unnecessary data, e.g., skip motion vectors if no motion blur).
- Transparent fallback: Deferred can't render transparent objects (requires forward pass). Unity automatically renders transparent in forward after deferred (hybrid approach). Minimize transparent objects (expensive, breaks deferred benefits).
- Light culling: Deferred calculates per-pixel lighting (all lights evaluated). Use light volumes (render light influence as geometry, stencil culling limits per-light pixels). HDRP does automatically (tiled/clustered deferred).

**Forward+ Implementation:**
- Tile size tuning: Larger tiles (32x32) = fewer tiles, less compute overhead, coarser culling. Smaller tiles (16x16) = more tiles, tighter culling, less light overdraw. HDRP default: 16x16 tiles (good balance).
- Light binning compute shader: Count lights per tile (atomic increment), store light indices per tile (structured buffer). Fragment shader reads tile's light list (index into global light buffer), iterates lights.
- Transparent objects: Forward+ supports transparency (forward rendering, just uses tile light lists). Unlike deferred (transparent incompatible), Forward+ handles glass, particles, alpha blending.
- Fallback for old hardware: Forward+ requires compute shaders (SM 5.0+). Fallback to forward rendering on old GPUs (check SystemInfo.supportsComputeShaders, switch rendering path dynamically).

**GPU-Driven Culling:**
- Frustum culling compute shader: Test object bounds against camera frustum planes (6 planes, dot products per corner). Objects outside frustum excluded from rendering (write visible flag to buffer, indirect args buffer).
- Occlusion culling: Render depth prepass (depth-only, coarse occluders), compute shader tests object bounds against depth buffer (hierarchical Z-buffer, mip chain). Occluded objects culled (not drawn).
- LOD selection on GPU: Compute distance from camera (GPU calculates per instance), select LOD level (distance thresholds), update indirect args buffer (different mesh per LOD). Zero CPU LOD overhead.
- Instance data management: Upload instance transforms/data once (large structured buffer, 100,000+ instances), GPU reads per-instance data (vertex shader accesses via instance ID), updates visible instances (compute shader culling updates counts).

**G-Buffer Optimization:**
- Minimize render targets: Each G-buffer texture = bandwidth (write during geometry pass, read during lighting). HDRP: 3-4 render targets typical (albedo+AO, normal+smoothness, emission, depth). Avoid unnecessary data (pack efficiently, use alpha channels).
- Use appropriate formats: Albedo = RGBA8 (sRGB), normals = RGB10A2 or BC5 (compressed), depth = D24S8 or D32. Higher precision (RGBA16F) only when necessary (HDR emission). Balance quality vs bandwidth.
- Depth prepass: Render depth-only pass before G-buffer (early Z rejection during G-buffer pass). Reduces overdraw (occluded pixels skipped, don't write G-buffer). Especially beneficial for dense scenes (urban environments, interiors).
- Lighting accumulation: Additive blend during lighting pass (each light adds to accumulation buffer). Or single full-screen pass (compute all lights, output final color). Additive = flexible (per-light volumes), full-screen = fewer passes.

**Platform-Specific:**
- **PC**: All rendering paths supported. Deferred (Built-in, HDRP default), Forward+ (HDRP for transparent), GPU-driven rendering (modern GPUs, DirectX 12/Vulkan). 100+ lights (deferred/Forward+), 100,000+ objects (GPU-driven).
- **Consoles**: Deferred or Forward+ (PS5/Xbox Series X). Clustered rendering (HDRP clustered deferred). GPU-driven rendering supported (async compute, optimized culling). 50-100 lights typical, 10,000-50,000 objects.
- **Switch**: Forward rendering only (deferred too expensive, limited bandwidth). No Forward+/GPU-driven (compute shader limitations). 1-4 lights max. <10,000 objects (CPU culling, traditional forward pipeline).
- **Mobile**: Forward rendering mandatory (deferred unsupported, bandwidth prohibitive). No Forward+/GPU-driven (older GPUs lack compute). 1-2 lights. <5,000 objects. Aggressive culling (CPU-based, conservative frustum culling).

## Common Pitfalls

**Deferred Rendering with Many Transparents**: Developer uses deferred pipeline, creates scene with 1000 transparent objects (glass windows, particles, alpha-blended UI). Transparent objects render in forward pass after deferred (expensive, no deferred benefits). Symptom: Low frame rate despite deferred, transparent objects GPU bottleneck. Solution: Minimize transparent objects (use opaque alternatives: reflective materials instead of glass, cutout instead of alpha blend). Or switch to Forward+ (handles transparency efficiently).

**No Light Culling in Forward+**: Developer implements Forward+ but bins all lights into all tiles (no culling, every tile gets all lights). No performance benefit over forward (still evaluates 100 lights per pixel). Symptom: Forward+ no faster than forward. Solution: Implement proper culling (frustum test light bounds vs tile frustum, store only affecting lights per tile). HDRP does automatically (use HDRP Forward+ or reference implementation).

**GPU-Driven Without Proper Sorting**: Developer implements GPU culling (writes visible instances to buffer) but doesn't sort (render order random). Transparent objects render incorrectly (depth sorting wrong), state changes thrash (material switches random order). Symptom: Transparent artifacts, low performance (excessive state changes). Solution: Sort visible instances on GPU (bitonic sort compute shader, sort by material then depth). Or use GPU radix sort (faster for large counts).

**G-Buffer Bandwidth Bottleneck**: Developer creates deferred renderer with 8 G-buffer render targets (position, normal, albedo, metallic, roughness, AO, emission, motion vectors = all separate textures). 8 RT writes = massive bandwidth (3840x2160 * 8 * 16 bytes = 1GB per frame at 4K). GPU bandwidth saturated. Symptom: GPU bound, Memory Profiler shows huge G-buffer, low frame rate. Solution: Pack G-buffer efficiently (4 render targets max: albedo+AO, normal+smoothness, emission, depth). Use packed formats (RGB10A2 for normals, RGBA8 for albedo).

## Tools & Workflow

**Rendering Path Selection**: Built-in Pipeline: Edit > Project Settings > Player > Rendering Path (Forward, Deferred, Legacy Vertex Lit). Per-camera override: Camera > Rendering Path. HDRP: Deferred default (opaque), Forward+ for transparent (automatic).

**HDRP Frame Settings**: HDRP Asset > Frame Settings > Lighting > Forward/Deferred rendering. Clustered settings (tile size, cluster count). Async compute (overlap culling with rendering, performance win on consoles).

**Frame Debugger**: View rendering pipeline. Shows G-buffer pass (multiple render targets), lighting pass (light volumes, full-screen pass), forward pass for transparents. Verify pipeline structure (correct passes, expected order).

**GPU Profiler**: Rendering pass costs. Geometry pass (G-buffer write time), lighting pass (per-light or full-screen), transparent forward pass. Identify bottlenecks (G-buffer bandwidth, lighting ALU, transparent overdraw).

**RenderDoc/PIX**: Inspect G-buffer contents (view render targets: albedo, normals, etc.), analyze tile light lists (Forward+ per-tile data), capture indirect draws (GPU-driven argument buffers). Debug advanced pipelines (verify culling, check light binning).

**Compute Shader Debugger**: Debug GPU-driven culling. Capture compute dispatch (frustum culling kernel), view input/output buffers (instance data, visibility flags, indirect args). Verify culling logic (correct frustum tests, proper counts).

**HDRP Rendering Debugger**: Window > Analysis > Rendering Debugger > Material > Material/Lighting. Visualize G-buffer contents (albedo, normals, smoothness), verify deferred data (correct packing). Debug lighting (isolate diffuse, specular, individual lights).

## Related Topics

- [13.1 Lighting Techniques](13-01-Lighting-Techniques.md) - Lighting setup
- [13.6 Culling Techniques](13-06-Culling-Techniques.md) - Culling methods
- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - CPU culling
- [12.4 Advanced Shader Techniques](12-04-Advanced-Shader-Techniques.md) - Compute shaders

---

[← Previous: 13.4 Transparency and Alpha](13-04-Transparency-And-Alpha.md) | [Next: 13.6 Culling Techniques →](13-06-Culling-Techniques.md)
