# 17.4 LOD for Specific Assets

[← Back to Chapter 17](../chapters/17-Level-Of-Detail-Systems.md) | [Main Index](../README.md)

LOD strategies vary by asset type: characters require skeletal preservation, vegetation uses alpha cutout simplification, architecture focuses on silhouette, and particles use soft-particle LOD.

---

## Overview

Character LOD: Preserve skeleton (rig unchanged across LODs, animations work on all LODs), simplify mesh (remove facial details LOD1, blend shapes LOD2, basic capsule LOD3), reduce blend shapes (morph targets expensive, LOD1 = key expressions only, LOD2 = none). Typical: LOD0 = 15,000 tris (hero character, full detail, facial animation), LOD1 = 7,000 tris (simplified face, no inner mouth), LOD2 = 3,000 tris (basic body, simple head, no blend shapes), LOD3 = 500 tris (capsule with basic limbs, or impostor billboard). Skeleton constant (all LODs use same bone count/hierarchy, animator controller unchanged).

Vegetation LOD: Alpha cutout common (leaves use alpha textures, many overlapping quads). LOD strategy: reduce quad count (LOD0 = 200 leaf quads, LOD1 = 50 quads, LOD2 = 10 quads), merge to solid (LOD2 = solid mesh instead of alpha cutout, opaque rendering = faster), impostor billboard (LOD3 = single cross-quad or billboard). Typical tree: LOD0 = 5,000 tris (detailed trunk + leaves), LOD1 = 2,000 tris (simplified trunk, fewer leaf clusters), LOD2 = 500 tris (simple trunk, merged leaf geometry), LOD3 = 2 tris (billboard impostor). Wind animation: LOD0/1 = vertex animation (wind shader bends leaves), LOD2/3 = no wind (distant trees static, animation imperceptible).

Architecture LOD: Silhouette critical (buildings recognized by outline, not interior detail). LOD strategy: remove interior faces (LOD1 = no interior walls, only exterior shell), merge windows (LOD1 = texture instead of geometry), simplify roofs (LOD2 = flat surfaces, no detail tiles). Typical building: LOD0 = 20,000 tris (interior + exterior, full detail), LOD1 = 8,000 tris (exterior only, simplified windows), LOD2 = 2,000 tris (basic shape, merged textures), LOD3/cull = impostor or removed. Lightmapping important (buildings usually static, baked lighting, ensure LOD1/2 have valid UV1).

## Key Concepts

- **Skeletal Consistency**: Characters require same skeleton across LODs (bone count, hierarchy, bind poses identical). Allows: single Animator (one controller drives all LODs), seamless transitions (LOD switch doesn't break animations). Implementation: LOD0 mesh (full detail, rigged to skeleton), LOD1 mesh (simplified, same skeleton, skin weights updated for simplified mesh), LOD2 mesh (basic, same skeleton). Exporter ensures (DCC tool exports all LODs with identical armature).
- **Blend Shape Reduction**: Morph targets expensive (CPU blending, memory overhead per shape). LOD0 = 50+ blend shapes (full facial rig, lip sync, expressions), LOD1 = 10 shapes (key expressions only: happy, sad, angry), LOD2 = 0 shapes (no morphing, static mesh). Memory: 50 shapes × 15K verts × 12 bytes = 9MB per character, LOD1 = 10 × 7K × 12 = 840KB, LOD2 = 0 (huge savings). Distant characters don't need facial detail (blend shapes invisible beyond 10m).
- **Alpha Cutout Optimization**: Vegetation uses alpha textures (leaves with transparency, GPU alpha testing). Expensive: overdraw (many overlapping transparent quads, GPU processes all layers), alpha testing (discard pixels, breaks early-Z). LOD strategy: reduce alpha quads (fewer leaf cards), merge to opaque geometry (LOD2 = solid leaf clumps, no alpha = cheaper), or alpha-to-coverage (A2C, converts alpha to MSAA coverage, better than alpha test).
- **Impostor Billboards for Characters**: Distant characters (beyond 100m) use billboard (single quad with pre-rendered character sprite). Captures: orthographic renders of character (8-16 angles, multiple animations = idle/walk/run), sprite sheet (atlas with all views/animations). Runtime: billboard shader samples atlas (selects view based on camera angle, animates via UV offset). Massive savings (character at 200m = 2 tris vs 3,000 tri LOD2).
- **Architectural Simplification**: Buildings simplified by: removing interior (LOD1 = exterior shell only, no indoor geometry = 50% tris saved), merging windows (LOD1 = windows baked to texture, not modeled geometry), flattening details (LOD2 = no roof tiles, no trims, basic boxes). Maintains silhouette (building outline unchanged, recognizable from distance). Detail visible up close (LOD0 full interior), simplified at distance (LOD1/2 exteriors only).

## Best Practices

**Character LOD Setup:**
- Skeleton preservation: DCC tool (Blender, Maya) exports LOD meshes (LOD0/1/2 all reference same armature), Unity import (all meshes share skeleton, Animator controller works on all LODs). Verify: same bone count (Avatar shows identical hierarchy for LOD0/1/2), same bone names (scripting references bones, must match across LODs). Test: switch LODs during animation (seamless transitions, no animation breakage).
- Facial detail reduction: LOD0 = full face (eyes, teeth, tongue, inner mouth, wrinkles), LOD1 = simplified face (closed mouth, no inner geometry, basic eyes), LOD2 = low-poly head (no facial features modeled, details in texture only). Distance: LOD0 = 0-10m (face visible, needs detail), LOD1 = 10-30m (face small, simplified acceptable), LOD2 = 30-100m (face tiny, basic shape sufficient).
- Blend shape culling: LOD0 = all blend shapes (facial rig, lip sync), LOD1 = key shapes only (import mesh, keep 5-10 important shapes, delete others in Unity), LOD2 = no shapes (import mesh without blend shapes, or delete all). Memory savings significant (10MB per character with shapes → <1MB without). Script toggles (disable SkinnedMeshRenderer blend shapes at distance, zero CPU blending cost).
- Clothing/accessories: LOD0 = all accessories (hats, belts, pouches, detailed clothing folds), LOD1 = merged accessories (combine small items into main mesh, bake to texture), LOD2 = no accessories (only main body mesh, accessories invisible at distance). Separate GameObjects (accessories as children, disable per LOD, LODGroup controls enables).

**Vegetation LOD Strategy:**
- Alpha quad reduction: LOD0 = 200 leaf quads (alpha cutout, detailed canopy), LOD1 = 50 quads (key leaf clusters, outer canopy only), LOD2 = 10 quads (simplified crown, basic shape). Remove interior leaves (LOD1 culls leaves inside canopy, only outer visible), merge clusters (combine nearby quads into single cards). Reduce overdraw (fewer alpha quads = less GPU overdraw, 4x faster at LOD1 vs LOD0).
- Alpha to opaque: LOD2 converts alpha cutout to opaque mesh (model leaf clumps as solid geometry, no transparency). Benefits: no overdraw (opaque rendering, early-Z culling works), faster (GPU skips alpha testing, standard z-buffer rendering). Looks acceptable at distance (distant trees = opaque silhouette sufficient, alpha detail imperceptible). Transition: LOD1 = alpha cutout (50 quads), LOD2 = opaque mesh (500 tris, no alpha).
- Impostor billboards: LOD3 = cross-quad (two perpendicular quads forming X, tree visible from all angles), or billboard (single quad rotating to face camera). Pre-rendered texture (render tree orthographically, capture as texture, apply to billboard). Update: static (baked texture, never changes), or dynamic wind (animate UV offset, simulate swaying). Distance: LOD2 = 50-100m (3D mesh), LOD3 = 100-300m (impostor), cull >300m.
- Wind animation LOD: LOD0/1 = vertex shader wind (detailed leaf swaying, branch bending, expensive), LOD2 = simple wind (whole-tree sway only, no per-leaf), LOD3/impostor = no wind (static billboard, or simple rotation). Shader keywords (WIND_QUALITY_HIGH for LOD0, WIND_QUALITY_LOW for LOD2, no keyword for LOD3). Performance: LOD0 wind = 0.2ms per tree, LOD2 = 0.02ms, LOD3 = 0ms (100 trees = 20ms vs 2ms vs 0ms).

**Architecture LOD Guidelines:**
- Interior removal: LOD0 = full building (interior rooms, furniture, walls), LOD1 = exterior shell only (delete interior faces, keep outer walls/roof/windows). Culling: player inside = LOD0 (interior visible), player outside >20m = LOD1 (only exterior matters, interior culled). Savings: 50% tris (interior = ~half of building geometry). Tools: DCC script selects interior faces (by normal direction, faces pointing inward), deletes for LOD1 export.
- Window simplification: LOD0 = modeled windows (frames, glass, interior visible through windows), LOD1 = texture windows (bake windows to diffuse texture, flat geometry), LOD2 = no windows (merged texture, single building color). Distance: LOD0 = 0-30m (window details visible), LOD1 = 30-100m (windows small, texture acceptable), LOD2 = 100-200m (building tiny, no window detail needed).
- Roof/detail flattening: LOD0 = detailed roof (tiles, chimneys, vents, trim), LOD1 = simplified roof (flat planes, no tiles, major features only), LOD2 = basic box (flat roof, no details). Merge coplanar faces (LOD1 roof = single plane, no subdivisions). Texture detail (bake roof tiles to normal map, LOD1 geometry simple but texture detailed).
- Lightmap LOD: Ensure LOD1/2 have lightmap UVs (UV1 generated, valid for baking), share lightmaps (LOD0/1/2 pack into same lightmap atlas, consistent lighting across LODs), or separate lightmaps (LOD0 high-res, LOD1 low-res, saves memory). Without lightmaps (LOD1/2 appear unlit or incorrectly lit, lighting mismatch during LOD transition).

**Particle System LOD:**
- Particle count reduction: LOD0 = 1,000 particles (smoke plume, dense), LOD1 = 500 particles (simplified plume, half density), LOD2 = 100 particles (sparse, basic shape), LOD3/cull = disabled (particles invisible beyond distance). Particle System component > LOD Bias (multiplier for emission rate, LOD0 = 1.0, LOD1 = 0.5, LOD2 = 0.1). Script controls (based on camera distance, adjusts emission dynamically).
- Soft particles LOD: LOD0 = soft particles enabled (depth-based fade, smooth intersection with geometry, expensive depth texture reads), LOD1/2 = soft particles disabled (hard edges, acceptable at distance, faster). Material LOD: separate materials per LOD (LOD0 material = complex shader with soft particles, LOD1 = simple shader without), LODGroup switches materials (assign per-LOD material, shader complexity varies).
- Simulation quality: LOD0 = high simulation (collision, forces, accurate physics), LOD1 = medium (basic collision only), LOD2 = no collision (particles don't interact, just emit/fade). Collision detection expensive (GPU raycasts per particle, LOD0 only). Distant particles (beyond 50m) don't need collision (player can't see detail, simplified simulation acceptable).

**Platform-Specific:**
- **PC**: Detailed LODs (characters LOD0 = 20K tris, LOD1 = 10K, LOD2 = 5K), blend shapes (LOD0 = full facial rig, 50+ shapes), alpha cutout (vegetation LOD0 = 200 quads, acceptable overdraw). Conservative transitions (LOD0 extended range, quality priority). Impostors optional (use for extreme distances >200m, or skip and cull earlier).
- **Consoles**: Moderate LODs (characters LOD0 = 10K tris, LOD1 = 5K, LOD2 = 2K), reduced blend shapes (LOD0 = 20 shapes, LOD1 = 5, LOD2 = none), moderate alpha (vegetation LOD0 = 100 quads). Balanced transitions (LOD1 at ~30m, LOD2 at ~80m). Impostors common (for vegetation, characters beyond 100m).
- **Switch**: Aggressive LODs (characters LOD0 = 5K tris, LOD1 = 2K, LOD2 = 500), minimal blend shapes (LOD0 = 10 shapes max, LOD1 = none), low alpha (vegetation LOD0 = 50 quads, convert to opaque early). Early transitions (LOD1 at 15m, LOD2 at 50m, cull at 100m). Impostors critical (reduce 3D rendering, billboards for distant objects).
- **Mobile**: Very aggressive LODs (characters LOD0 = 3K tris, LOD1 = 1K, cull at 50m), no blend shapes (too expensive, static meshes only), minimal alpha (vegetation LOD0 = 20 quads, opaque mesh preferred). Very early transitions (LOD1 at 10m, cull at 30m). No impostors (too complex, prefer aggressive culling instead).

## Common Pitfalls

**Mismatched Skeletons**: Artist exports character LODs with different skeletons (LOD1 missing bones, LOD2 simplified rig). Animations break (LOD switch causes errors, animator can't find bones, character T-poses). Symptom: Character pops to T-pose when LOD switches (animations stop working, console errors about missing bones). Solution: Export all LODs with identical skeleton (same bone count, names, hierarchy), verify in Unity (Avatar configuration shows same bones for all LODs), test (play animations while switching LODs manually, ensure no breakage).

**Blend Shapes Memory Leak**: Developer keeps all 50 blend shapes on all LODs (LOD1/2 still have full facial rig, thinking doesn't hurt). Memory usage stays high (LOD1 character 7K verts × 50 shapes × 12 bytes = 4MB per character, 100 characters = 400MB for shapes alone). Symptom: High mesh memory (Profiler shows gigabytes of SkinnedMesh data, blend shapes dominate), no memory savings from LODs. Solution: Remove blend shapes from LOD1+ (import mesh, delete blend shapes, or DCC export without shapes), keep shapes only on LOD0 (close-up characters only).

**Vegetation Overdraw**: Developer creates vegetation LOD0 with 200 alpha cutout quads (highly detailed leaves), no LOD1/2 reduction (LOD1 also 200 quads, LOD2 also 200 quads, thinking mesh simple enough). Overdraw extreme (alpha quads stack, GPU processes all layers, 10-20x overdraw per tree). Frame rate tanks with many trees. Symptom: GPU fragment-bound (Profiler shows high pixel overdraw, vegetation expensive), Scene view overdraw mode = bright red (extreme overdraw). Solution: Aggressive alpha reduction for LOD1/2 (LOD1 = 50 quads, LOD2 = opaque mesh or 10 quads), transition to impostor (LOD3 = billboard), cull aggressively (trees beyond 100m removed).

**Architecture Interior Not Culled**: Developer creates building LOD1 (simplifies exterior), but keeps interior geometry (all interior walls/furniture still present in LOD1, just slightly simplified). LOD1 still expensive (interior = 50% of tris, wasted on invisible geometry). Symptom: LOD1 not much faster than LOD0 (Profiler shows LOD1 still has thousands of tris, minimal performance gain). Solution: Delete interior entirely for LOD1 (player outside = interior never visible, pure waste), keep exterior shell only (walls, roof, windows), validate (Frame Debugger shows only exterior rendering for LOD1 buildings).

## Tools & Workflow

**Character LOD Export**: DCC tool (Blender) > select armature + all LOD meshes (LOD0_mesh, LOD1_mesh, LOD2_mesh as children of armature), File > Export > FBX (check "Armature" and "Mesh", export as Character.fbx). Unity import > Avatar Configuration (verify all meshes show same humanoid rig, same bone count). LODGroup: add component, assign LOD0 mesh to LOD0, LOD1 mesh to LOD1, etc.

**Blend Shape Removal**: Unity mesh asset > Inspector > Import Settings > Model tab > Blend Shapes section (uncheck specific blend shapes, or "Import Blend Shapes" = off to remove all). Reimport (apply changes, blend shapes removed from mesh). Memory: Profiler > Memory > SkinnedMesh Renderers (shows memory with/without blend shapes, verify reduction).

**Vegetation LOD Tools**: SpeedTree (procedural tree generator, automatic LOD generation, exports LOD0/1/2/billboard), Unity import (SpeedTree assets have LODGroup pre-configured). Manual: DCC tool reduces leaf quads (delete interior leaves, keep outer canopy), export per LOD (Tree_LOD0.fbx = 200 quads, Tree_LOD1.fbx = 50 quads, Tree_LOD2.fbx = 10 quads opaque mesh).

**Architecture Interior Culling**: DCC tool (Blender) > select building mesh, Edit mode > select interior faces (Box Select interior, or Select > Select All by Trait > Facing direction = interior normals), Delete faces (X > Faces, removes interior geometry). Export LOD1 (File > Export > FBX > Building_LOD1.fbx, exterior shell only). Verify: import to Unity, Frame Debugger shows only exterior rendering.

**Particle LOD Script**: C# script adjusts ParticleSystem based on distance: `float distance = Vector3.Distance(Camera.main.transform.position, transform.position); var emission = particleSystem.emission; emission.rateOverTimeMultiplier = distance < 30 ? 1.0f : distance < 80 ? 0.5f : 0.1f;` Attach to particle GameObject, automatically reduces emission at distance. Or: LODGroup controls particle enable (LOD0 = particles enabled, LOD1 = lower emission, LOD2 = disabled).

**Impostor Tools**: Amplify Impostors (Asset Store, billboard baker), configure views (8-16 angles), bake (generates billboard texture + prefab), assign to LODGroup (LOD3 = impostor prefab). For characters: bake multiple animations (idle, walk, run sprites), shader animates UV (cycles through animation frames based on time).

## Related Topics

- [17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) - LOD basics
- [17.2 LOD Generation](17-02-LOD-Generation.md) - LOD creation methods
- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Mesh reduction techniques
- [7.5 Animation and Rigging](07-04-Animation-Optimization.md) - Skeletal optimization

---

[← Previous: 17.3 Advanced LOD Techniques](17-03-Advanced-LOD-Techniques.md) | [Next: Chapter 18 →](../chapters/18-Scriptable-Render-Pipeline.md)
