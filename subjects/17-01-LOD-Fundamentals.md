# 17.1 LOD Fundamentals

[← Back to Chapter 17](../chapters/17-Level-Of-Detail-Systems.md) | [Main Index](../README.md)

Level of Detail (LOD) systems reduce rendering cost by swapping meshes based on distance, using lower-poly models for distant objects and culling objects beyond thresholds.

---

## Overview

LOD swaps mesh based on camera distance: LOD0 (highest detail, close range 0-20m), LOD1 (medium detail, mid range 20-50m), LOD2 (low detail, far range 50-100m), Cull (removed entirely, beyond 100m). Unity LODGroup component manages transitions: calculates screen-space size (object's bounds projected to screen, percentage of screen height), switches LOD based on thresholds (LOD0 = 60-100%, LOD1 = 25-60%, LOD2 = 10-25%, Cull <10%). Reduces vertex processing (distant objects use fewer triangles, less GPU work), reduces overdraw (culled objects not rendered at all).

LOD benefits: massive performance savings (10,000 objects, LOD0 avg 5,000 tris = 50M tris, with LOD2 at distance avg 500 tris = 5M tris visible = 10x reduction), maintains visual quality (distant objects low-poly acceptable, player can't see detail). Cost: memory overhead (storing multiple LOD meshes, 3 LODs = 3x mesh memory), pop-in artifacts (visible LOD transitions, mitigated by crossfading), authoring time (creating multiple LOD meshes, either manual or automated decimation).

LOD metrics: Screen-relative size (Unity default, percentage of screen height), Distance-based (fixed distance thresholds, simpler but less adaptive), Custom (script-controlled, based on importance/gameplay). Screen-relative better: adapts to screen resolution (same visual threshold on 1080p and 4K), adapts to FOV (wide FOV = objects smaller = LOD switches earlier). Distance-based simpler: fixed thresholds (LOD0 0-20m, LOD1 20-50m), predictable, easier to debug. Unity default = screen-relative (best for most cases).

## Key Concepts

- **LODGroup Component**: Manages LOD levels for GameObject. Contains array of LOD levels (LOD0, LOD1, LOD2, etc.), each with: screen-relative threshold (0-1, percentage of screen height), Renderers array (meshes active at this LOD). Unity calculates screen size (bounds projected to screen), activates appropriate LOD (enables Renderers for matching threshold range, disables others). Automatic per frame (component updates LOD based on camera distance/FOV).
- **Screen-Relative Size**: LOD metric based on object's screen-space height. Formula: screenSize = (boundingSize / distance) / tan(FOV/2). Unity compares screenSize to LOD thresholds (LOD0 = 0.6-1.0, LOD1 = 0.25-0.6, etc.). Adaptive: same object switches LODs at different distances based on FOV/resolution (zoomed camera = switches later, wide FOV = switches earlier). Balances visual quality and performance (visible detail matches screen space).
- **LOD Bias**: Global multiplier for LOD distances (QualitySettings.lodBias, default = 1.0). Values: <1.0 = aggressive LODs (switches earlier, performance priority, 0.5 = switches at half distance), >1.0 = conservative LODs (switches later, quality priority, 2.0 = switches at double distance). Use for: quality tiers (Low = lodBias 0.5, High = 2.0), dynamic performance scaling (reduce bias when frame rate drops).
- **Crossfade**: Smooth transition between LODs (dithered alpha fade, blends LOD meshes). LODGroup.fadeMode = LODFadeMode.CrossFade (enables crossfading), fade transition time adjustable. Shader support required (LOD_FADE_CROSSFADE keyword, shader uses dither pattern based on unity_LODFade). Reduces pop-in artifacts (gradual transition, less noticeable LOD switch). Cost: overdraw (both LODs render during transition, 2x cost during fade).
- **Cull LOD**: Final LOD level (object removed entirely, no rendering). LODGroup.SetLODs includes cull level (last LOD, threshold <0.1 or specified, Renderers empty = cull). Culls distant objects (beyond useful visibility, skip rendering = massive savings). Essential for: large scenes (thousands of objects, cull distant ones), open worlds (horizon objects culled, only nearby rendered). Unity: final LOD with 0 renderers = cull (object deactivated).

## Best Practices

**LOD Level Design:**
- Three LOD levels typical: LOD0 (close, 100% detail, 0-30m), LOD1 (medium, 50% tris, 30-80m), LOD2 (far, 25% tris, 80-150m), Cull (beyond 150m). More LODs diminishing returns (LOD management overhead, memory cost, authoring time), fewer LODs = larger jumps (more visible popping). Three levels balance quality/performance/memory.
- Triangle budgets: LOD0 = hero detail (5,000-20,000 tris for important objects, 500-2,000 for props), LOD1 = half tris (2,500-10,000 / 250-1,000), LOD2 = quarter tris (1,250-5,000 / 125-500). Reductions: remove small details (interior faces, small protrusions), simplify silhouette (fewer edge loops), merge coplanar faces (flat surfaces collapsed).
- Silhouette preservation: Maintain overall shape (distant objects recognized by outline, not detail). LOD1/2: keep silhouette edges (character head/shoulders, vehicle profile), remove interior detail (facial features, wheel spokes). Collapse interior geometry (reduce subdivisions), preserve boundary edges (visible shape unchanged). Player notices changed silhouette (breaks immersion), doesn't notice simplified interior.
- Screen thresholds: LOD0 = 60-100% screen height (object occupies significant screen space, close to camera), LOD1 = 25-60% (mid-distance, visible but smaller), LOD2 = 10-25% (far, small on screen), Cull <10% (tiny, imperceptible). Adjust based on object importance (hero objects = tighter thresholds = longer LOD0 range, background = aggressive = early LOD2/cull).

**LOD Generation Workflow:**
- Manual LOD creation: DCC tools (Blender, Maya) create multiple mesh versions (artist-controlled decimation, optimized by hand). Highest quality (artist decisions preserve important features), time-consuming (manual work per asset). Best for: hero objects (characters, vehicles, key props), assets with complex details (architecture, organic shapes). Export separate FBX per LOD (Character_LOD0.fbx, Character_LOD1.fbx, Character_LOD2.fbx), assign to LODGroup in Unity.
- Automatic decimation: Unity or DCC tools auto-generate LODs (algorithmic mesh simplification, e.g., quadric edge collapse). Fast (batch process hundreds of assets), lower quality (algorithm may remove important features). Best for: background assets (rocks, trees, clutter), large quantities (procedural generation, asset libraries). Unity: Mesh.Optimize (simplifies mesh, not automatic LOD generation), or third-party tools (Simplygon, InstaLOD).
- Hybrid approach: LOD0 = original artist asset (full quality), LOD1/2 = auto-generated (decimation from LOD0). Balances quality and speed (hero LOD0 hand-crafted, distant LODs automated). Typical workflow: import LOD0, script generates LOD1 (50% tris) and LOD2 (25% tris), assign to LODGroup. Unity Asset Store tools (e.g., Unity Mesh Simplifier, Auto LOD) automate.
- Baked lighting consideration: Lightmap UVs per LOD (each LOD mesh needs UV1 for lightmapping). Ensure: LOD0/1/2 all have Generate Lightmap UVs enabled (mesh import settings), or manually created UV1 (DCC tool). Without UV1: LOD doesn't bake correctly (missing lighting, errors). See Chapter 14.1 for lightmap UV requirements.

**LOD Transition Optimization:**
- Crossfade settings: LODGroup.fadeMode = CrossFade (dithered blend), fadeTransitionWidth (0.01-0.1, blend duration as fraction of LOD range). Narrow width (0.01) = fast transition (1% of LOD range, less overdraw), wide (0.1) = slow transition (10% of range, smoother but more overdraw). Balance: visibility of pop-in vs cost (slow fade = expensive, fast fade = noticeable).
- Shader crossfade support: Shader needs LOD_FADE_CROSSFADE (built-in shaders support, custom shaders add manually). Shader code: `#pragma multi_compile _ LOD_FADE_CROSSFADE`, use `unity_LODFade` (fade value, clip based on dither pattern). Without support: crossfade doesn't work (hard LOD switch). Unity Standard Shader supports (automatic).
- Disable crossfade for performance: LODGroup.fadeMode = None (no crossfade, instant LOD switch). Saves overdraw (no blending = single LOD rendered). Use when: performance critical (mobile, Switch, tight budgets), pop-in acceptable (fast-paced games, player doesn't notice), objects small (LOD transitions imperceptible). Enable when: quality critical (cinematic games, slow movement), large objects (buildings, vehicles, noticeable switches).
- Animate LOD bias: Script adjusts QualitySettings.lodBias dynamically (reduce bias when FPS drops, increase when stable). Example: if (fps < 30) lodBias = 0.5; else if (fps > 55) lodBias = 1.0; (dynamic quality scaling). Maintains frame rate (sacrifices LOD quality when needed). HDRP/URP Dynamic Resolution does this automatically.

**Performance Monitoring:**
- Profile LOD impact: Unity Profiler > CPU > Rendering.UpdateLODs (time spent calculating LOD switches, typically <0.5ms). GPU Profiler > triangle counts (verify LOD reducing tris, e.g., LOD0 = 50M tris, with LOD = 10M tris). Frame Debugger > verify correct LODs rendering (distant objects use LOD2, close use LOD0).
- Memory profiling: Profiler > Memory > Meshes (shows all LOD meshes, memory per LOD). 3 LODs = 3x mesh memory (LOD0 + LOD1 + LOD2 all in memory). Optimize: aggressive LOD2 decimation (LOD2 = 10% tris of LOD0, saves memory), or unload distant LODs (AssetBundles, load/unload LOD meshes dynamically, advanced).
- Overdraw analysis: Scene view > Overdraw shading mode (red = high overdraw). LOD should reduce overdraw (distant objects use simpler meshes = fewer overlapping triangles). Crossfade increases overdraw temporarily (both LODs rendered during fade = 2x overdraw). Verify: overdraw lower with LODs (Frame Debugger shows pixel overdraw counts).
- Distance validation: Scene view select LODGroup > Inspector shows LOD ranges (colored bars = LOD active ranges). Scene view camera distance shown (helps verify transitions). Move camera to LOD boundaries (verify switches at expected distances, no premature culling, appropriate LOD active per distance).

**Platform-Specific:**
- **PC**: Three LOD levels + cull (LOD0 high-poly, LOD1 mid, LOD2 low, cull >200m). Conservative lodBias (1.0-2.0, prioritizes quality). Crossfade enabled (smooth transitions, overdraw acceptable). High triangle budgets (LOD0 = 10,000+ tris, LOD1 = 5,000, LOD2 = 2,000).
- **Consoles**: Three LOD levels + cull. Standard lodBias (1.0). Crossfade optional (enabled for hero objects, disabled for props). Moderate budgets (LOD0 = 5,000 tris, LOD1 = 2,500, LOD2 = 1,000). Balance quality/memory (fixed memory pool, LOD meshes compete with textures).
- **Switch**: Two-three LOD levels + aggressive cull (LOD0/1 only, or LOD0/1/2 with early cull). Aggressive lodBias (0.5-0.7, early LOD switches). No crossfade (overdraw prohibitive). Low budgets (LOD0 = 2,000 tris, LOD1 = 1,000, LOD2 = 500). Cull at 50-100m (aggressive culling for performance).
- **Mobile**: Two LOD levels + very aggressive cull (LOD0/1 only). Very aggressive lodBias (0.3-0.5). No crossfade. Very low budgets (LOD0 = 1,000 tris, LOD1 = 500). Cull at 30-50m (extremely aggressive, most objects culled). Minimize LOD count (memory extremely limited, two LODs sufficient).

## Common Pitfalls

**No LOD on Major Objects**: Developer creates scene with 1,000 high-poly trees (5,000 tris each, no LOD). GPU renders 5M triangles (many trees distant, full detail wasted). Frame rate tanks. Symptom: Low frame rate (GPU vertex-bound, Profiler shows millions of tris), distant objects unnecessarily detailed. Solution: Add LODGroup (3 LOD levels: LOD0 5,000 tris, LOD1 1,500, LOD2 500, cull >100m). Frame rate improves 5-10x (distant trees use LOD2 500 tris or culled).

**LOD Transitions Too Visible**: Developer sets LOD thresholds too aggressive (LOD1 at 80% screen size, large objects pop suddenly). Player notices LOD switches (immersion broken, objects visibly change detail mid-game). Symptom: Visible popping (LOD switches obvious, especially during camera movement). Solution: Widen LOD ranges (LOD0 = 60-100%, LOD1 = 25-60%, smoother transitions), enable crossfade (LODGroup.fadeMode = CrossFade, dithered blend = less noticeable), tune per object importance (hero objects = wider LOD0 range).

**Excessive LOD Levels**: Developer creates 5-6 LOD levels (thinking more = better). Memory usage doubles (5 LOD meshes vs 3), LOD management overhead increases (Unity updates more LODs per frame). Minimal visual benefit (LOD3-5 imperceptible difference, wasted memory). Symptom: High mesh memory (Profiler shows excessive LOD meshes), negligible performance gain (LOD2 already low-poly, further LODs redundant). Solution: Use 3 LOD levels + cull (sufficient for 99% cases). More LODs only for: extremely large objects (buildings visible at extreme distances, 5+ LODs justified), or very high-end targets (PC ultra settings).

**No Culling LOD**: Developer adds LOD0/1/2, no cull level (objects render at all distances, even tiny on screen). Distant objects still rendered (LOD2 low-poly but still processed, draw calls + vertex processing wasted). Symptom: High draw call count (Frame Debugger shows hundreds of tiny objects rendered), performance doesn't scale with distance. Solution: Add cull LOD (LODGroup final level = 0 renderers, objects removed beyond threshold). Cull distance based on object size (small props = 50m, large buildings = 200m). Massive savings (hundreds of draw calls eliminated).

## Tools & Workflow

**LODGroup Component**: GameObject > 3D Object > LOD Group (creates GameObject with LODGroup component). Inspector shows LOD levels (colored bars: LOD0 green, LOD1 yellow, LOD2 red, Cull gray), drag Renderers into LOD slots (assign meshes per LOD), adjust thresholds (drag LOD boundaries, changes screen-relative percentage).

**LOD Preview**: Select LODGroup > Inspector shows LOD bar. Drag camera slider (simulates camera distance, previews which LOD active). Scene view shows active LOD (LOD0/1/2 label, Renderers for active LOD enabled, others disabled). Useful for: verifying LOD transitions (check switches at expected distances), visual quality (preview LOD2 appearance).

**Quality Settings**: Edit > Project Settings > Quality > LOD Bias (global multiplier, 0.5-2.0). Maximum LOD Level (disables LODs beyond level, e.g., Maximum = 1 = only LOD0/1 used, LOD2 ignored). Per-quality tier (Low = lodBias 0.5, Medium = 1.0, High = 1.5). Runtime: QualitySettings.lodBias = value (change dynamically).

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows LOD per object (draw call lists LOD level, e.g., "MeshRenderer (LOD1)"). Verify: distant objects using LOD2 (correct LOD active), close objects LOD0, culled objects absent (not in draw call list). Triangle counts per LOD (draw call stats show vertex count).

**Scene View Overdraw**: Scene view > Shading Mode > Overdraw (red = high overdraw, green = low). Verify: LODs reduce overdraw (LOD2 less red than LOD0, simpler meshes = less overlap). Crossfade: temporarily increases overdraw (both LODs rendered, double red during transition).

**Unity Profiler**: Profiler > CPU > Rendering.UpdateLODs (LOD calculation time, <0.5ms typical). Memory > Meshes (LOD mesh memory, LOD0/1/2 listed separately). GPU > Triangles (total tris rendered, verify LODs reducing count). Monitor: LOD overhead (CPU time acceptable), memory usage (LOD meshes fit in budget), triangle reduction (LODs effective).

## Related Topics

- [17.2 LOD Generation](17-02-LOD-Generation.md) - Creating LOD meshes
- [17.3 Advanced LOD Techniques](17-03-Advanced-LOD-Techniques.md) - HLOD and imposters
- [13.6 Culling Techniques](13-06-Culling-Techniques.md) - Culling and LOD
- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Mesh decimation

---

[← Previous: 16.3 Camera Effects](../subjects/16-03-Camera-Effects.md) | [Next: 17.2 LOD Generation →](17-02-LOD-Generation.md)
