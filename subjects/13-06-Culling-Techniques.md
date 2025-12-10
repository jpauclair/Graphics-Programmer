# 13.6 Culling Techniques

[← Back to Chapter 13](../chapters/13-Rendering-Techniques.md) | [Main Index](../README.md)

Culling techniques eliminate invisible objects from rendering through frustum culling, occlusion culling, distance culling, and layer-based culling to reduce draw calls and improve performance.

---

## Overview

Culling removes objects before rendering: frustum culling (objects outside camera view), occlusion culling (objects hidden behind other objects), distance culling (objects beyond threshold), and layer culling (excluded layers). Without culling, GPU renders everything (including off-screen/hidden objects), wastes draw calls and GPU time (processing invisible geometry). Culling reduces rendering workload: 10,000 objects in scene, frustum culling reduces to 2,000 visible, occlusion culling reduces to 500 (rendering 5% of scene, 20x performance improvement).

Frustum culling tests object bounds (AABB or bounding sphere) against camera frustum planes (6 planes: near, far, left, right, top, bottom). Objects entirely outside any plane culled (not rendered). Unity performs automatic frustum culling (Camera.cullingMask for layer-based exclusion). Cost: CPU-based (test each object, 1-5µs per object), efficient for most scenes (<10,000 objects). Conservative culling possible (expand frustum slightly, cull aggressively for distant objects).

Occlusion culling identifies objects blocked by occluders (buildings, terrain, large props). Preprocessing bakes occlusion data (PVS: Potentially Visible Set per camera position), runtime queries PVS (loads visible objects for current position). Unity Occlusion Culling system: manual setup (mark occluders/occludees), bake occlusion (generates visibility data), runtime culls (automatic, uses baked data). Massive savings in dense environments (urban scenes, interiors: 50-90% objects culled). Cost: memory overhead (occlusion data = 5-50MB), baking time (minutes to hours for large scenes).

## Key Concepts

- **Frustum Culling**: Tests object bounds against camera frustum (6 planes). Objects outside frustum not rendered. Unity automatic (happens every frame, per camera). Culling mask (Camera.cullingMask) excludes layers. Fast (CPU-based plane tests), effective (culls off-screen objects).
- **Occlusion Culling**: Culls objects hidden behind occluders. Baked system (Unity Occlusion Culling): precomputes visibility data (PVS), runtime queries visibility. Effective in dense scenes (cities, interiors), less effective in open areas (outdoor, few occluders). Requires manual setup (mark occluders/occludees, bake).
- **Distance Culling**: Culls objects beyond distance threshold. Per-layer distances (Camera.layerCullDistances), or per-object (custom scripts). Use for: small objects invisible at distance (debris, props), optimization tiers (Low = cull at 50m, High = cull at 200m).
- **Level of Detail (LOD)**: Swaps meshes based on distance (not culling, but related). LOD0 = close (high poly), LOD1 = medium distance (mid poly), LOD2 = far (low poly), Cull = removed entirely. Reduces vertex processing (distant objects use fewer triangles). Unity LOD Group component.
- **Portal-Based Culling**: Manual visibility system for interconnected rooms. Portals (doorways, windows) define room connections, camera in room renders connected rooms only (through visible portals). Pre-occlusion culling (before GPU submission). Effective for interiors (buildings with rooms, dungeons).

## Best Practices

**Frustum Culling Configuration:**
- Use layer culling masks: Camera.cullingMask excludes layers from rendering (UI camera = only UI layer, main camera = everything except UI). Separate cameras for different content (main scene, UI, minimap).
- Conservative bounds: Renderer bounds should tightly fit geometry (oversized bounds = culled late, undersized = culled early causing popping). Unity calculates automatically (MeshRenderer bounds from mesh). Custom bounds: SkinnedMeshRenderer.localBounds for animated characters.
- Batch-friendly culling: Static batched objects cull as single unit (all-or-nothing, if batch bounds visible = render entire batch). Avoid batching objects with distant spatial separation (batch spans large area = less culling benefit). Group nearby objects for batching.
- Culling distance per layer: Camera.layerCullDistances = float[32] (distance per layer). Example: layer "SmallProps" = cull at 30m, layer "Environment" = cull at 200m. Per-layer control (small objects cull aggressively, important objects cull conservatively).

**Occlusion Culling Setup:**
- Mark occluders: Static large objects (buildings, terrain, walls) = Occluder Static. Small objects (props, debris) = don't mark (too small to occlude). Occluder count impacts bake time (fewer = faster bakes).
- Mark occludees: All static objects that should be culled = Occludee Static. Dynamic objects culled via baked data (if static scene occludes dynamic object). Non-static objects don't contribute to occlusion data (can't occlude others).
- Bake occlusion: Window > Rendering > Occlusion Culling > Bake. Smallest Occluder (minimum occluder size, larger = faster bake), Smallest Hole (minimum gap to see through, larger = more aggressive culling). Balance bake time vs accuracy.
- Occlusion areas: Define regions for occlusion (Occlusion Area component). Useful for optimizing specific areas (dense city = high detail occlusion, open field = skip occlusion). Reduces bake time (only bake important areas).

**Distance Culling Implementation:**
- Per-layer distances: Set Camera.layerCullDistances in C#. Example: layerCullDistances[LayerMask.NameToLayer("SmallProps")] = 30f. Different distances per layer (small props = 30m, characters = 100m, environment = 300m).
- Quality tier distances: Adjust per quality setting (Low quality = aggressive culling 50m, High = conservative 200m). Script reads QualitySettings.GetQualityLevel(), sets distances accordingly.
- LOD + culling combination: LOD Group final level = Cull (removes object entirely). Example: LOD0 (0-20m), LOD1 (20-50m), LOD2 (50-100m), Cull (100m+). Gradual quality reduction then removal.
- Script-based culling: Custom scripts for complex logic (cull based on importance, player line-of-sight, gameplay state). SetActive(false) to cull, SetActive(true) to restore. More control than built-in systems.

**Portal Culling (Manual System):**
- Portal placement: Place portals at doorways, windows, archways (connections between rooms). Portal mesh with trigger collider (detects which room camera occupies). Script manages visibility (enable room A + connected rooms, disable disconnected rooms).
- Room management: Group objects per room (parent under Room GameObject). Room script controls children (SetActive for entire room). Camera enters room = activate room + connected rooms via portals.
- Visibility determination: Ray cast through portals (is room visible through portal?). Or simple frustum test (is portal in frustum?). More sophisticated: portal occlusion (portal partially blocked = partial visibility).
- Performance: Portal culling = CPU-based (script logic per frame). Efficient for moderate room counts (<100 rooms). Large buildings: hierarchical portals (rooms group into sectors, sectors connect via major portals).

**Culling Performance:**
- Profile culling overhead: Unity Profiler > CPU > Rendering > Culling (time spent culling objects). High cost (>2ms) = too many objects or complex bounds. Reduce object count (merge meshes, batching) or simplify bounds (fewer colliders).
- GPU-driven culling: Move culling to GPU (compute shader frustum/occlusion tests). CPU uploads object data once, GPU culls per frame (zero CPU overhead). Requires indirect rendering (DrawMeshInstancedIndirect). Advanced technique (modern GPUs, complex setup).
- Spatial partitioning: Organize scene hierarchically (quadtree, octree, BVH). Culling tests nodes (cull entire branch if node outside frustum). Faster than per-object tests (O(log N) instead of O(N)). Unity uses internally (scene spatial hashing).
- Async culling: Spread culling across frames (cull 1/4 objects per frame, 4-frame update cycle). Reduces per-frame spikes (smoother frame times), acceptable for distant objects (delayed culling invisible to player). Custom implementation (not built-in Unity feature).

**Platform-Specific:**
- **PC**: Moderate culling (10,000-50,000 objects handled). Frustum + occlusion culling (both enabled). Conservative distances (cull at 200-500m). CPU handles culling overhead easily (multi-core, high clock speed).
- **Consoles**: Balanced culling. Frustum + occlusion (both enabled). Moderate distances (100-300m). Occlusion critical for dense games (urban environments, interiors). Bake occlusion offline (runtime performance priority).
- **Switch**: Aggressive culling. Frustum only (occlusion too expensive, memory limited). Short distances (50-100m). Reduce object counts (merge meshes, aggressive LOD). CPU-bound (culling overhead significant, optimize object count).
- **Mobile**: Very aggressive culling. Frustum only (no occlusion, memory/performance prohibitive). Very short distances (30-50m). Minimal objects (<5,000 total). CPU weak (culling overhead critical, reduce object count drastically).

## Common Pitfalls

**Not Using Occlusion Culling**: Developer creates dense city scene (1000 buildings, 10,000 objects), doesn't enable occlusion culling. GPU renders everything (including buildings behind camera, fully occluded). Frame rate tanks (10-20fps). Symptom: Low frame rate in dense areas, GPU rendering 10,000 objects (only 500 visible). Solution: Enable occlusion culling (Window > Occlusion Culling), mark occluders (buildings = Occluder Static), mark occludees (all static objects), bake. Frame rate improves to 60fps (renders only 500 visible objects).

**Oversized Bounding Boxes**: Developer imports mesh with incorrect bounds (bounds 10x larger than actual mesh). Object culled late (bounds still in frustum despite mesh off-screen), or not culled at all (bounds always in frustum). Symptom: Objects render when off-screen (visible in Frame Debugger), culling ineffective. Solution: Recalculate bounds (Mesh.RecalculateBounds in script), or manually adjust (SkinnedMeshRenderer.localBounds, MeshRenderer bounds via custom mesh import).

**Occlusion Baking with Small Occluders**: Developer marks all objects as occluders (trees, rocks, small props). Occlusion bake takes 24+ hours (too many occluders), resulting data huge (500MB). Runtime loads slow (long level load times). Symptom: Very long bake times, huge occlusion data files. Solution: Mark only large occluders (buildings, terrain, walls). Small objects can't meaningfully occlude (too small, gaps too large). Reduces bake time to minutes, data to 10-20MB.

**No Distance Culling for Small Objects**: Developer renders all debris/props regardless of distance. Player sees 5,000 small objects at 200m (each <1 pixel on screen, invisible but still rendered). Wasted draw calls. Symptom: High draw call count, many tiny objects in Frame Debugger. Solution: Implement distance culling (Camera.layerCullDistances for "SmallProps" layer = 50m). Culls invisible distant objects (saves 3,000+ draw calls).

## Tools & Workflow

**Occlusion Culling Window**: Window > Rendering > Occlusion Culling. Visualization tab (see occlusion in scene view, color-coded: blue = occluded, green = visible), Bake tab (Smallest Occluder, Smallest Hole, baking settings), Bake button (generates occlusion data).

**Camera Culling Mask**: Camera Inspector > Culling Mask (layer checkboxes). Exclude layers from rendering (UI camera shows only UI layer, main camera excludes UI). Per-camera control (minimap camera, reflection probe cameras).

**LOD Group Component**: Add Component > LOD Group. Configure LOD levels (LOD0/1/2, distances), assign renderers per LOD (drag mesh renderers into LOD slots), set Cull distance. Scene view shows LOD preview (colored bars = LOD ranges).

**Frame Debugger**: View culled vs rendered objects. Shows draw calls (rendered objects), identifies culling effectiveness (how many objects culled?). Scene view shows frustum culling visualization (objects outside frustum highlighted).

**Unity Profiler**: CPU Profiler > Rendering > Culling shows culling time. High cost = too many objects or expensive bounds tests. Memory Profiler shows occlusion data size (MB of occlusion culling data loaded).

**Statistics Window**: Game view > Stats button. Shows Batches (draw calls), Tris (triangles rendered), Verts (vertices). Compare with/without culling (verify culling reduces counts). Also shows culled objects (FPS, culling effectiveness).

**Occlusion Visualization**: Scene view (Occlusion Culling window open) > Visualization tab > Occlusion. Color-coded objects (blue = culled by occlusion, green = visible). Move scene camera to see how occlusion changes (different positions = different visibility).

## Related Topics

- [13.5 Advanced Rendering](13-05-Advanced-Rendering.md) - GPU-driven culling
- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - Culling performance
- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - LOD systems
- [6.1 CPU Bottlenecks](06-01-CPU-Bottlenecks.md) - Culling overhead

---

[← Previous: 13.5 Advanced Rendering](13-05-Advanced-Rendering.md) | [Next: Chapter 14 →](../chapters/14-Global-Illumination-And-Lightmapping.md)
