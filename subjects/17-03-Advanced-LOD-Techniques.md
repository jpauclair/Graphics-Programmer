# 17.3 Advanced LOD Techniques

[← Back to Chapter 17](../chapters/17-Level-Of-Detail-Systems.md) | [Main Index](../README.md)

Advanced LOD techniques include Hierarchical LOD (HLOD) for scene-wide optimization, impostor billboards for ultra-distant objects, and GPU-based LOD selection for massive object counts.

---

## Overview

Hierarchical LOD (HLOD): Combines multiple objects into single mesh at distance (100 trees become one merged mesh, one draw call instead of 100). Reduces draw call overhead (CPU bottleneck eliminated), maintains visual density (distant forest still visible, simplified representation). HLOD levels: detailed scene (near, individual objects), clusters (mid-distance, groups of 5-10 objects merged), mega-clusters (far, entire areas merged). Unity HLOD experimental (enable in project settings), Unreal built-in (automatic HLOD generation, widely used). Massive savings for open worlds (thousands of objects, distant areas use HLOD = dozens of draw calls instead of thousands).

Impostors: Replace 3D mesh with billboard sprite (quad textured with pre-rendered object views). Ultra-distant objects (tree at 500m = single quad, 2 tris vs 5,000 tri mesh). Multiple views: octahedral impostor (captures 8+ viewing angles, switches based on camera direction), or simple billboard (single view, rotates to face camera). Quality: imperceptible at distance (tiny screen size, billboard vs mesh indistinguishable), breaks at close range (billboard obvious, must transition to 3D mesh before noticeable). Typical: impostor beyond 200-500m (used as final LOD, before cull).

GPU LOD selection: CPU calculates LOD traditionally (iterate objects, compute screen size, set LOD level = expensive for 100,000+ objects). GPU LOD: compute shader calculates LODs (parallel, processes thousands of objects per frame), outputs LOD levels (buffer with per-object LOD index), rendering uses LOD (indirect draw calls select appropriate mesh). Unity DOTS/ECS supports (compute-based LOD, scales to millions of objects), or custom implementation (compute shader + DrawMeshInstancedIndirect). Enables massive worlds (100K+ objects, LOD selection <1ms on GPU).

## Key Concepts

- **HLOD Generation**: Combine nearby objects (spatial clustering, objects within bounds merged), merge meshes (combine geometry into single mesh, bake lighting/materials), create proxy (HLOD cluster replaces individual objects at distance). Example: 50 trees in area (LOD0/1 = individual trees, LOD2 = trees merged into single 5,000-tri mesh, one draw call). Unity: HLOD component groups objects (defines cluster bounds, generates merged mesh), LODGroup switches to HLOD (at threshold distance, disables individuals + enables merged proxy).
- **Impostor Billboards**: Quad mesh (2 triangles, always faces camera), textured with pre-rendered views (render object from multiple angles, store as texture atlas). Octahedral mapping: 8+ views (top, bottom, four cardinals, four diagonals), fragment shader selects view (based on camera-to-object angle, blends adjacent views). Simpler: single billboard (renders to face camera always, acceptable for round objects like trees). Transition: mesh LOD2 → impostor LOD3 (50-100m mesh, 100-500m impostor, beyond 500m cull).
- **GPU LOD Calculation**: Compute shader per object (calculates screen-space size, determines LOD level), outputs to buffer (StructuredBuffer with per-object LOD indices), indirect rendering reads buffer (DrawMeshInstancedIndirect uses LOD buffer, selects mesh variant). Parallel processing (GPU processes 10,000 objects simultaneously, vs CPU sequential = 100x faster). Requires: GPU instancing (all objects use instancing, LOD index per instance), indirect draw support (DX11+, consoles, modern mobile).
- **Crossfading HLOD**: Smooth transition between individual objects and HLOD proxy (dithered fade, blends single objects + merged cluster). Implementation: LODGroup crossfade (LOD2 individuals fade out, LOD3 HLOD fades in, 5-10m blend distance), shader dither pattern (LOD_FADE_CROSSFADE, same as regular LOD crossfade). Reduces pop-in (transition less noticeable, gradual blend vs instant switch). Cost: overdraw during transition (both individuals + HLOD render, 2x cost briefly).
- **Nanite-Style Virtualized Geometry**: Inspired by UE5 Nanite (not in Unity by default, custom implementation). Hierarchical mesh clusters (mesh subdivided into clusters, clusters organized in BVH), GPU-driven rendering (compute culling + LOD selection, only visible clusters rendered), automatic LOD (no manual LOD creation, algorithm selects detail level). Extremely advanced (requires significant engineering, research-level implementation), not typical for Unity projects (DOTS/ECS approaches more practical).

## Best Practices

**HLOD Setup:**
- Cluster definition: Group nearby objects (spatial proximity, trees in 50m×50m area = one cluster), similar objects (same material = can batch, different materials = separate clusters), static only (HLOD for static geometry, dynamic objects use regular LOD). Tool places cluster volumes (box volumes, objects inside merged), or automatic (algorithm groups based on density/distance).
- Mesh merging: Combine geometry (all objects in cluster → single mesh, vertices/triangles appended), bake materials (combine textures into atlas, or use single material), bake lighting (optional: bake lightmaps for HLOD, or use unlit/vertex lighting). Unity HLOD tools (experimental, editor scripts generate HLOD proxies), or manual (export objects, merge in DCC tool, import as HLOD mesh).
- Transition distances: Individuals LOD0-2 (0-100m), HLOD LOD3 (100-300m, merged cluster visible), cull beyond 300m (entire cluster removed). Crossfade: blend individuals → HLOD (90-110m transition, dithered fade), avoid pop (gradual transition, imperceptible switch). Tune per cluster size (large clusters = earlier transition, small = later).
- Performance validation: Profile draw calls (with HLOD: dozens of calls for distant area, without: thousands). CPU time (LODGroup updates reduced, fewer objects to manage). GPU time (vertex processing similar, but draw call overhead eliminated = faster). Target: <100 draw calls for distant background (vs 1,000+ without HLOD).

**Impostor Implementation:**
- Capture views: Render object orthographically (8+ angles, store as texture atlas), resolution per view (256x256 typical, impostor small on screen). Octahedral mapping (8 views: +X/-X/+Y/-Y/+Z/-Z/+45°/-45° diagonals), or hemispheric (16 views, better quality + more memory). Unity: custom render script (position camera, render to RT, save as texture), or tools (Amplify Impostors, asset store).
- Billboard shader: Quad mesh (2 tris, vertices positioned to face camera), shader samples texture atlas (calculate view angle, select appropriate atlas region), blend adjacent views (smooth transition between angles). Code: `float3 viewDir = normalize(cameraPos - objectPos); float angle = atan2(viewDir.y, viewDir.x); int viewIndex = (int)((angle / TWO_PI) * numViews); float2 uv = atlasUV[viewIndex] + localUV;`
- Transition to mesh: LODGroup levels (LOD0/1/2 = mesh, LOD3 = impostor billboard, cull beyond). Crossfade: mesh LOD2 fades out, impostor LOD3 fades in (dithered blend, smooth transition). Distance: impostor starts 100-200m (beyond this distance, billboard acceptable), culls 500m+ (tiny, removal imperceptible).
- Dynamic impostors: Update captures periodically (animated objects, re-render impostor views every N frames), or static (baked impostors, never update = zero cost). Trees/rocks: static impostors (never change, permanent billboards). Characters far away: dynamic (update every 10-30 frames, captures animation, expensive but acceptable for few characters).

**GPU LOD Selection:**
- Compute shader LOD: Input buffer (object positions, bounds), compute per object (screen-space size = bounds / distance / tan(FOV/2)), output LOD level (0-3, based on size thresholds), write to buffer (StructuredBuffer<int> lodLevels). GPU processes in parallel (10,000 objects = one dispatch, milliseconds).
- Indirect rendering: DrawMeshInstancedIndirect per LOD level (separate calls for LOD0/1/2/3), args buffer (instance count per LOD, LOD compute shader fills counts), GPU renders (only objects matching LOD level, automatic culling). Example: LOD buffer has 5,000 LOD0, 3,000 LOD1, 2,000 LOD2, 0 cull → DrawIndirect calls render 5K+3K+2K instances.
- Instance data: StructuredBuffer with per-instance data (position, rotation, scale, LOD level), shader reads buffer (vertex shader applies transforms, fragment shader uses data), updated by compute (LOD compute writes instance data, rendering reads). All GPU (zero CPU overhead for LOD calculation, CPU only dispatches compute + indirect draws).
- Platform requirements: DirectX 11+ (compute shaders, indirect rendering), Vulkan/Metal (equivalent features), consoles (PS4/Xbox One+, fully supported). Mobile: limited (high-end only, many devices don't support compute), Switch: supported (Tegra X1 compute capable, but performance-limited). Best for: PC/console open worlds (100K+ objects, CPU LOD bottleneck eliminated).

**Platform-Specific:**
- **PC**: Full HLOD (multi-level clusters, automatic generation), impostors (high-resolution captures, 16-view octahedral), GPU LOD viable (compute shaders standard, indirect rendering efficient). Open worlds benefit (massive scenes, 100K+ objects with HLOD+GPU LOD). Tools: Unity HLOD experimental, or custom compute-based LOD.
- **Consoles**: Standard HLOD (2-3 cluster levels, simpler than PC), impostors (moderate-resolution captures, 8-view), GPU LOD viable (compute supported, indirect rendering efficient). Performance-critical (fixed budgets, HLOD essential for large scenes). Memory consideration (HLOD meshes compete with textures/lighting, balance cluster count/memory).
- **Switch**: Limited HLOD (single cluster level, simpler merging), low-res impostors (128x128 captures, 4-view), GPU LOD possible but expensive (compute available, but slow, CPU LOD may be faster). Aggressive optimization (HLOD critical for performance, reduce draw calls drastically). Memory constrained (HLOD meshes minimal, prefer culling over distant LOD).
- **Mobile**: No HLOD (too complex, avoid), basic impostors (single-view billboards, 64x64 captures), no GPU LOD (compute unsupported on most devices, CPU LOD only). Minimal LOD (2 levels max, aggressive culling instead of complex LOD systems). Simplicity priority (complex LOD overhead > benefit, keep systems simple).

## Common Pitfalls

**HLOD Lighting Mismatch**: Individual objects have baked lightmaps (lighting correct up close), HLOD proxy has no lightmaps (unlit or vertex-lit, looks different). Transition visible (near objects lit properly, HLOD cluster flat/dark, obvious pop when switching). Symptom: Distant clusters look wrong (too dark, no shadows, lighting inconsistent). Solution: Bake lightmaps for HLOD proxies (Unity lightmap baking includes HLOD meshes, ensure Contribute GI enabled), or use vertex lighting (approximate lighting, acceptable for distant HLOD). Match lighting system (if scene baked, HLOD must bake too).

**Impostor Update Cost**: Developer implements dynamic impostors (updates every frame, thinking necessary). Capturing impostor = render object 8+ times (orthographic views, to texture atlas), every frame = 8x object rendering cost. Worse than regular mesh (impostor intended to save cost, dynamic updates negate benefit). Symptom: Performance worse with impostors (CPU/GPU rendering 8 views per frame per object). Solution: Static impostors (bake once, never update, zero cost), or very infrequent updates (every 30-60 frames for animated objects, or on-demand when object changes significantly).

**GPU LOD Without Instancing**: Developer implements GPU LOD compute shader (calculates LOD levels), but renders with individual DrawMeshNow calls (CPU iterates objects, draws each separately). GPU LOD benefit lost (compute shader calculates LODs fast, but CPU draw calls slow = bottleneck unchanged). Symptom: Performance no better than CPU LOD (draw call overhead remains, compute shader overhead added). Solution: Use GPU instancing + indirect rendering (DrawMeshInstancedIndirect, GPU handles both LOD selection + rendering, zero CPU per-object cost).

**HLOD Memory Explosion**: Developer generates HLOD for entire scene (thousands of objects, merged into dozens of huge HLOD clusters). Each HLOD cluster = megabytes of geometry (merged mesh vertices/indices), total HLOD meshes = gigabytes. Memory exhausted. Symptom: Long load times (loading huge HLOD meshes), out-of-memory crashes (HLOD meshes exceed available memory), high memory usage (Profiler shows gigabytes of mesh data). Solution: Aggressive HLOD culling (cull HLOD clusters beyond distance, don't keep all loaded), limit cluster sizes (max 10,000 tris per HLOD cluster, split large areas into multiple clusters), or reduce HLOD LOD count (fewer HLOD levels, balance memory vs draw calls).

## Tools & Workflow

**Unity HLOD (Experimental)**: Enable: Project Settings > Graphics > HLOD (experimental feature). Create HLOD Volume: GameObject > Rendering > HLOD Volume (defines cluster bounds). Configure: HLOD Volume component (objects inside merged, LOD threshold, mesh simplification settings). Generate: HLOD Volume Inspector > Generate button (merges objects, creates proxy mesh). Result: HLOD proxy asset (merged mesh, assigned to HLOD Volume).

**Amplify Impostors**: Asset Store tool (billboard LOD generation). Select mesh: right-click asset > Create > Amplify Impostor (opens baker). Configure: views (8/16/32 views, resolution per view), atlas size (texture resolution), baking settings. Bake: generates impostor texture (atlas with views), impostor prefab (quad mesh with impostor shader). Use: assign impostor as final LOD (LODGroup LOD3 = impostor prefab, beyond mesh LODs).

**Custom Compute LOD**: Create compute shader (LODCalculation.compute): `#pragma kernel CalculateLODs; StructuredBuffer<float3> positions; RWStructuredBuffer<int> lodLevels; [numthreads(64,1,1)] void CalculateLODs(uint3 id : SV_DispatchThreadID) { float distance = length(cameraPos - positions[id.x]); float screenSize = (objectSize / distance) / tan(fov * 0.5); lodLevels[id.x] = screenSize > 0.6 ? 0 : screenSize > 0.25 ? 1 : screenSize > 0.1 ? 2 : 3; }` Dispatch from C#: `compute.Dispatch(kernel, objectCount/64, 1, 1);`

**DrawMeshInstancedIndirect**: Setup: `ComputeBuffer argsBuffer = new ComputeBuffer(1, 5 * sizeof(uint), ComputeBufferType.IndirectArguments); argsBuffer.SetData(new uint[] { meshLOD0.GetIndexCount(0), instanceCount, 0, 0, 0 });` Render: `Graphics.DrawMeshInstancedIndirect(meshLOD0, 0, material, bounds, argsBuffer, 0, materialPropertyBlock);` Repeat for LOD1/2/3 (separate DrawIndirect calls per LOD level, args buffer updated by compute shader).

**HLOD Profiling**: Frame Debugger: shows HLOD proxy draws (verify single draw call for cluster vs dozens for individuals), triangle counts (HLOD mesh tris vs sum of individual tris). Unity Profiler > Rendering: draw calls reduced (with HLOD: <100, without: 1,000+), batches (HLOD increases batching, fewer state changes). GPU Profiler: vertex processing time (should reduce with HLOD, fewer draw calls = less overhead).

## Related Topics

- [17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) - LOD basics
- [17.2 LOD Generation](17-02-LOD-Generation.md) - Creating LOD meshes
- [13.6 Culling Techniques](13-06-Culling-Techniques.md) - Culling with LOD
- [27.03 GPU-Driven Rendering](27-03-GPU-Driven-Rendering.md) - GPU LOD selection

---

[← Previous: 17.2 LOD Generation](17-02-LOD-Generation.md) | [Next: 17.4 LOD for Specific Assets →](17-04-LOD-For-Specific-Assets.md)
