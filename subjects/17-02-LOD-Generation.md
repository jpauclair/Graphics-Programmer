# 17.2 LOD Generation

[← Back to Chapter 17](../chapters/17-Level-Of-Detail-Systems.md) | [Main Index](../README.md)

LOD generation creates lower-detail mesh variants through manual modeling, automatic decimation algorithms, or hybrid approaches to balance quality and authoring efficiency.

---

## Overview

LOD generation methods: Manual (artist creates each LOD by hand in DCC tools, highest quality), Automatic (algorithmic decimation reduces triangle count, fast but lower quality), Hybrid (LOD0 manual, LOD1/2 auto-generated, balances quality/speed). Manual best for: hero assets (characters, vehicles, important props), complex topology (organic shapes, architectural details). Automatic best for: background assets (rocks, trees, clutter), large asset libraries (hundreds of props, batch processing). Hybrid typical for production (artists focus on LOD0, tools generate distant LODs).

Decimation algorithms: Quadric error metrics (edge collapse, preserves surface approximation), vertex clustering (grid-based simplification, fast but lower quality), feature-preserving (maintains important edges/boundaries, slower but better silhouette). Unity tools: Mesh Simplifier (third-party, quadric-based), Simplygon (commercial, feature-rich), InstaLOD (commercial, batch processing). DCC tools: Blender Decimate modifier (multiple algorithms), Maya Reduce tool, 3ds Max ProOptimizer. Choose based on: asset type (organic vs hard-surface), quality requirements (hero vs background), workflow integration (Unity vs DCC pipeline).

LOD validation: Visual inspection (compare LOD0 vs LOD1/2, verify silhouette preservation), polygon count verification (LOD1 ~50% tris, LOD2 ~25%, cull if appropriate), lightmap UV preservation (ensure LOD1/2 have valid UV1 for baking), performance testing (profile triangle count reduction, frame rate improvement). Batch validation: scripts iterate assets (verify LOD groups exist, check polygon budgets, flag assets needing manual fixes).

## Key Concepts

- **Quadric Error Metrics**: Decimation algorithm (iteratively collapses edges, minimizes surface deviation). Quadric matrix per vertex (encodes surface curvature), edge collapse cost calculated (low cost = flat area, high cost = curved/important). Collapses lowest-cost edges first (flat surfaces simplified aggressively, curved preserved longer). Produces high-quality LODs (smooth surface approximation, maintains overall shape). Used by: Simplygon, Mesh Simplifier, Blender Decimate (Collapse mode).
- **Edge Collapse**: Merge two vertices (removes edge, combines adjacent faces). Reduces triangle count by 2 per collapse (edge + two adjacent tris removed). Algorithm selects edges (sorted by error cost, lowest error collapsed first). Iterates until target triangle count reached (e.g., collapse until 50% tris remaining = LOD1). Preserve boundaries (silhouette edges never collapsed, maintains outline), preserve UVs (minimize UV distortion during collapse).
- **Feature Preservation**: Maintains important edges (silhouette boundaries, sharp creases, UV seams). Assigns high collapse cost to features (algorithm avoids collapsing important edges). Improves silhouette quality (distant LODs recognizable, outline unchanged despite interior simplification). Parameters: feature angle threshold (edges with angle >60° preserved), boundary locking (mesh boundaries never modified), UV seam preservation (UV islands stay intact).
- **UV Preservation**: Decimation maintains UV coordinates (collapsed vertices blend UVs, maintains texture mapping). Important for: textured assets (LODs share textures, UVs must match), lightmapped assets (LOD1/2 need valid UV1 for baking). Without UV preservation: stretched textures (UVs distorted after decimation), broken lightmaps (UV1 invalid, baking fails). Algorithm constraints: edge collapse checks UV distortion (rejects collapse if UV stretch excessive).
- **Batch Processing**: Automate LOD generation for asset libraries (scripts process hundreds of meshes, generate LOD1/2 automatically). Unity Editor script: iterate prefabs (find MeshFilters, decimate meshes, assign to LODGroup), save assets (modified prefabs with LOD groups). DCC pipeline: export script generates LODs (Blender/Maya script decimates, exports _LOD0/1/2 FBX files), import to Unity (batch import, LODGroups configured automatically). Saves time (manual LODs = hours per asset, batch = seconds per asset).

## Best Practices

**Manual LOD Creation (DCC Tools):**
- Topology reduction: Remove edge loops (select loop, dissolve = removes edges without affecting shape), merge coplanar faces (select flat surfaces, merge = single large face), collapse small details (remove screws, hinges, interior geometry). Maintain silhouette (preserve boundary edges, never merge silhouette loops). LOD1: remove 50% edges (from 10,000 tris to 5,000), LOD2: remove 75% (to 2,500 tris).
- LOD0 as base: Create LOD0 first (full detail, artist-modeled). Duplicate for LOD1 (start from LOD0 copy, apply manual reduction). Duplicate LOD1 for LOD2 (further reduce from LOD1). Workflow: LOD0 (hero model) → LOD1 (remove interior detail, simplify) → LOD2 (simplify silhouette, remove small features). Each LOD based on previous (iterative simplification).
- Preserve silhouette edges: Identify outline (render orthographic views, trace outline edges in wireframe), lock edges (mark as unremovable, never dissolve). Simplify interior (remove internal subdivisions, invisible from distance), keep boundary (silhouette unchanged, recognizable shape). Test: render LOD from distance (verify looks like LOD0, no major shape changes).
- Export conventions: Separate FBX per LOD (Tree_LOD0.fbx, Tree_LOD1.fbx, Tree_LOD2.fbx), or single FBX with multiple meshes (Tree.fbx containing Tree_LOD0, Tree_LOD1, Tree_LOD2 objects). Unity import: assign meshes to LODGroup (drag LOD0/1/2 meshes into LOD slots). Version control: track LOD files (commit all LODs, reviewers verify LOD quality).

**Automatic Decimation (Unity):**
- Mesh Simplifier package: Unity Asset Store (open-source, UnityMeshSimplifier). API: `var simplifier = new UnityMeshSimplifier.MeshSimplifier(mesh); simplifier.SimplifyMesh(targetQuality); Mesh lodMesh = simplifier.ToMesh();` targetQuality = 0.5 for LOD1 (50% tris), 0.25 for LOD2 (25%). Preserves UVs/normals (maintains texture mapping, lightmap UVs intact).
- Batch script: Editor script iterates assets (find all prefabs with MeshFilters), generates LODs (simplify mesh, create LODGroup, assign LODs), saves (EditorUtility.SetDirty, AssetDatabase.SaveAssets). Example: `foreach (var prefab in prefabs) { var mesh = prefab.GetComponent<MeshFilter>().sharedMesh; var lod1 = SimplifyMesh(mesh, 0.5f); var lod2 = SimplifyMesh(mesh, 0.25f); CreateLODGroup(prefab, mesh, lod1, lod2); }` Processes hundreds of assets (run overnight, batch generation).
- Quality thresholds: Adjust target quality per asset type (large assets = 0.4 LOD1, 0.15 LOD2 for aggressive reduction; small props = 0.6 LOD1, 0.3 LOD2 for quality preservation). Too aggressive: broken silhouettes (LOD2 unrecognizable, needs manual fix), too conservative: insufficient reduction (LOD2 still expensive, doesn't help performance). Test range (0.3-0.6 for LOD1, 0.1-0.3 for LOD2), find balance per project.
- Fallback to manual: Automatic decimation fails on: complex organic shapes (character faces, hands = detail loss unacceptable), mechanical parts (gears, joints = topology breaks), transparent meshes (leaves, hair = alpha cutout issues). Flag failures (script detects broken LODs, logs for manual review), manual fixup (artist reviews flagged assets, creates manual LODs or tweaks auto-generated).

**Hybrid Workflow:**
- LOD0 manual: Artists create hero LOD0 (full detail, hand-modeled, highest quality). Export LOD0 only (single FBX, no LOD1/2 yet). Standard art pipeline (modeling, texturing, rigging for LOD0).
- LOD1/2 automatic: Import script triggers (detects new LOD0 assets, no LODGroup), runs decimation (generates LOD1 at 50%, LOD2 at 25% automatically), creates LODGroup (assigns LOD0/1/2, sets thresholds), flags for review (marks assets for artist validation). Artists verify (check auto-generated LODs, approve or request manual LOD if quality insufficient).
- Manual overrides: For hero assets (characters, vehicles, key props), artists create manual LOD1/2 (export as _LOD1.fbx, _LOD2.fbx), script detects manual LODs (skips auto-generation if manual files present), uses manual (higher quality for important assets). Hybrid: most assets auto-generated (90% of props, background), critical assets manual (10% heroes, receives artist attention).
- Quality gates: Automated validation (script checks LOD triangle counts, verifies <threshold), visual validation (screenshot renders of LOD1/2, displayed in Unity Editor for artist review), performance validation (profile scene with LODs, verify frame rate improvement). Fail gates (flag assets: polygon count too high, silhouette changed significantly, UVs broken) for manual fixup.

**DCC Pipeline Integration:**
- Blender Decimate: Decimate modifier (Collapse mode for quadric decimation, Ratio = 0.5 for LOD1, 0.25 for LOD2), apply modifier (generates decimated mesh), export separate FBX (LOD1.fbx, LOD2.fbx). Script automates: Python script iterates objects (apply Decimate, export each LOD), batch processes entire project (run from command-line, overnight processing).
- Maya Reduce: Mesh > Reduce (opens reduction tool, set target percentage = 50% for LOD1), preserve features (enable Preserve Quads, Keep Boundary Edges), apply (generates reduced mesh). Export FBX per LOD. Script: MEL/Python automates (select mesh, run reduce command, export, iterate).
- 3ds Max ProOptimizer: Select mesh > ProOptimizer modifier (set Vertex %, Maintain UVs), apply, export LOD. Batch script: MaxScript iterates scene objects (apply ProOptimizer, export each LOD). Max supports batch mode (command-line MaxScript execution, CI integration).
- CI integration: Automated asset pipeline (git commit triggers CI, runs LOD generation scripts in DCC tools, exports FBX files, imports to Unity, creates LODGroups). Nightly builds (entire asset library processed, LODs regenerated, Unity project updated). Ensures LODs always up-to-date (artists modify LOD0, CI regenerates LOD1/2 automatically).

**Platform-Specific:**
- **PC**: Moderate reduction (LOD1 = 50-60% tris, LOD2 = 25-30%, prioritize quality). Hero assets manual (character LODs hand-crafted, highest quality), props auto (background assets automated decimation). Validation: visual inspection (verify LODs at distance, ensure acceptable quality).
- **Consoles**: Standard reduction (LOD1 = 40-50%, LOD2 = 20-25%, balance quality/memory). Most assets auto (efficient workflow, artist review for heroes only). Fixed budgets (polygon limits per scene, LODs critical for meeting budgets). Validation: performance profiling (verify LOD reduces triangle count, improves frame rate).
- **Switch**: Aggressive reduction (LOD1 = 30-40%, LOD2 = 10-20%, performance priority). All assets auto (manual LODs too expensive, time-constrained). Very low budgets (strict polygon limits, aggressive decimation necessary). Validation: performance critical (target frame rate, LODs must significantly reduce load).
- **Mobile**: Very aggressive reduction (LOD1 = 25-30%, LOD2 = 10-15%, minimal detail). Automatic only (no manual LODs, time/budget prohibitive). Extreme budgets (thousands of tris per scene, LODs essential). Validation: device testing (test on low-end devices, ensure playable performance).

## Common Pitfalls

**Broken Silhouettes**: Decimation algorithm collapses boundary edges (silhouette changes, LOD2 shape completely different from LOD0). Character's head becomes pointed (forehead edge collapsed, head shape broken), vehicle corners rounded (sharp edges lost). Symptom: LODs don't resemble LOD0 (players notice object "morphs" as distance changes, immersion broken). Solution: Lock boundary edges (decimation algorithm preserves silhouette, feature angle threshold >60° locks sharp edges), or manual LODs for critical assets (artist maintains silhouette manually).

**UV Seam Issues**: Decimation collapses vertices across UV seams (UVs stretch, textures distorted). Texture bleeding (adjacent UV islands bleed into each other), lightmap errors (UV1 seams split, baking artifacts). Symptom: LODs have texture artifacts (stretched textures, visible UV seams, lightmap seams visible). Solution: UV-aware decimation (algorithm respects UV seams, doesn't collapse across boundaries), or regenerate UVs for LODs (Unity auto-generates lightmap UVs for LOD meshes, ensures valid UV1).

**Over-Decimation**: Script generates LOD2 at 5% tris (thinking more reduction = better performance). LOD2 completely broken (unrecognizable shape, collapsed to few vertices, looks like blob). Symptom: Distant objects look wrong (shapeless blobs, players complain about "low-quality graphics"). Solution: Conservative minimums (LOD2 minimum 20% tris for most assets, 10% only for simple shapes like boxes), visual validation (screenshot LOD2, verify recognizable), manual override for complex assets (characters LOD2 = 30% minimum, preserve key features).

**Missing Cull LOD**: Auto-generation creates LOD0/1/2 (no cull level, objects render at all distances). Distant objects still rendered (LOD2 low-poly but still processed, draw calls wasted). Symptom: High draw call count (Frame Debugger shows hundreds of tiny distant objects), performance doesn't scale with distance. Solution: Add cull LOD to generation script (LODGroup final level = 0 renderers, cull distance based on object size), typical cull = 100-200m (objects beyond this invisible, not rendered).

## Tools & Workflow

**Unity Mesh Simplifier**: Asset Store package (UnityMeshSimplifier, free/open-source). API: `var simplifier = new MeshSimplifier(mesh); simplifier.SimplifyMesh(0.5f); Mesh lod = simplifier.ToMesh();` Inspector: mesh asset > right-click > Simplify Mesh (if tool integration available). Editor window: custom tool window (select meshes, set quality, generate LODs button, batch processing).

**Simplygon**: Commercial tool (licensing required, Unity integration plugin). Window > Simplygon > Process (select assets, configure LOD settings, generate). Advanced features: LOD chain generation (automatic LOD0/1/2/3 creation), material merging (combines materials for LODs, reduces draw calls), impostor generation (billboard LODs for distant objects). Best for: large studios (budget for licensing), complex pipelines (advanced LOD requirements).

**Blender Decimate Modifier**: Select mesh > Modifiers > Add > Decimate (Collapse mode). Ratio slider (0.5 = 50% tris, 0.25 = 25%). Apply modifier (makes permanent, mesh reduced). Export: File > Export > FBX (save as _LOD1.fbx or _LOD2.fbx). Batch script: Python script automates (iterate objects, apply Decimate, export each LOD).

**LOD Group Inspector**: Select GameObject with LODGroup > Inspector shows LOD levels. Renderers section per LOD (drag mesh renderers into slots, empty = cull). Triangle count shown (per-LOD vertex/triangle counts, verify reduction). Preview slider (simulates camera distance, shows active LOD in scene view).

**Batch Generation Script**: Create Editor script (Assets/Editor/LODGenerator.cs). Menu item: MenuItem("Tools/Generate LODs"), window opens (select prefabs, configure quality thresholds, Generate button). Iterates selected prefabs: get MeshFilter, simplify mesh (UnityMeshSimplifier or custom), create LODGroup, assign LOD meshes, save prefab. Progress bar (EditorUtility.DisplayProgressBar, shows generation progress).

**Visual Validation Tool**: Editor window displays LOD screenshots (render LOD0/1/2 from distance, side-by-side comparison). Quality metrics: triangle count reduction (%), silhouette similarity score (compare outlines), UV validation (check for stretching). Approve/Reject buttons (approved LODs committed, rejected flagged for manual review). Integrated with version control (approved LODs auto-commit, reviewers see before/after).

## Related Topics

- [17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) - LOD basics
- [17.3 Advanced LOD Techniques](17-03-Advanced-LOD-Techniques.md) - HLOD and impostors
- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Mesh decimation techniques
- [25.4 Asset Pipeline](25-04-Asset-Pipeline.md) - Asset import automation

---

[← Previous: 17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) | [Next: 17.3 Advanced LOD Techniques →](17-03-Advanced-LOD-Techniques.md)
