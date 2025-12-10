# 13.4 Transparency and Alpha

[← Back to Chapter 13](../chapters/13-Rendering-Techniques.md) | [Main Index](../README.md)

Transparency rendering handles semi-transparent materials through alpha blending, alpha testing, depth sorting, and specialized techniques like dithering and order-independent transparency.

---

## Overview

Transparency types: Alpha Blending (smooth gradients, glass/water, blends source and destination colors), Alpha Testing/Cutout (hard edges, foliage/fences, discards pixels below threshold), and Additive/Multiplicative blending (particles, effects, special blend modes). Alpha blending formula: FinalColor = SourceColor * SourceAlpha + DestColor * (1 - SourceAlpha). Reads destination color (frame buffer), blends with source (transparent object), writes result. Requires depth sorting (render back-to-front, furthest objects first) because blending non-commutative (A over B ≠ B over A).

Depth sorting problems: transparent objects must render in correct order (back-to-front). Unity sorts by object center (rough approximation, breaks for large/intersecting objects). Intersecting transparent objects cause artifacts (sort order wrong, parts render incorrectly). Solutions: split geometry (smaller pieces sort better), manual render queue control (Shader render queue = 3000-3999 for transparent, higher numbers render later), or order-independent transparency (OIT, expensive advanced technique).

Alpha testing avoids sorting: Clip/discard pixels below threshold (alpha < 0.5 = discard, alpha ≥ 0.5 = opaque). No blending (binary transparency: fully visible or invisible). Writes depth (unlike alpha blending, which disables depth writes). Use for: foliage (leaves with hard edges), chain-link fences, decals with cutout. Cheaper than blending (no read-modify-write, no sorting), but hard edges (no smooth gradients).

## Key Concepts

- **Alpha Blending**: Smooth transparency. Blend source and destination colors based on alpha. Shader: Blend SrcAlpha OneMinusSrcAlpha (standard alpha blend). Requires: render back-to-front (depth sorting), disable depth writes (ZWrite Off), render queue = Transparent (3000+).
- **Alpha Testing/Cutout**: Binary transparency. Discard pixels below threshold (clip(alpha - threshold) in shader). No blending (opaque or invisible). Enables depth writes (ZWrite On), no sorting required (renders as opaque). Use for: foliage, fences, decals with hard edges.
- **Additive Blending**: Adds source to destination (Blend One One). Bright on bright = brighter (accumulates light). Use for: fire, explosions, bright particles, glowing effects. No depth sorting critical (additive somewhat commutative, order less important).
- **Premultiplied Alpha**: Multiplies RGB by alpha before blending (Blend One OneMinusSrcAlpha). Correct blending for particles (avoids dark edges). Unity particle systems use premultiplied alpha by default.
- **Depth Sorting**: Orders transparent objects back-to-front (furthest renders first, closest last). Unity sorts by object center (renderer bounds center). Sorting artifacts: intersecting/large objects render incorrectly (order ambiguous). Manual control: Shader render queue (3000 = early transparent, 3999 = late transparent).

## Best Practices

**Blend Mode Selection:**
- Standard alpha blending: Blend SrcAlpha OneMinusSrcAlpha, ZWrite Off. Use for: glass, water, transparent UI, semi-transparent materials. Smooth gradients (alpha 0.0-1.0), requires sorting.
- Alpha cutout: AlphaTest shader (clip(alpha - _Cutoff)), ZWrite On, render queue = AlphaTest (2450). Use for: foliage (trees, grass), chain-link fences, decals. Hard edges (binary transparency), no sorting needed.
- Additive: Blend One One, ZWrite Off. Use for: fire, explosions, magic effects, lens flares. Brightens background (pure additive), sorting less critical. Avoid overuse (too many additives = washed-out scene).
- Multiplicative: Blend DstColor Zero (multiply destination by source). Use for: shadows, dark effects, tinted glass. Darkens background. Rarely used (additive more common).

**Depth Sorting Strategies:**
- Keep objects small: Small transparent objects sort better (object center more representative). Large transparent planes (100m water plane) have ambiguous sort order (center doesn't reflect pixel depth). Split large objects (10m tiles instead of 100m plane).
- Manual render queue: Shader render queue = 3000 + offset. Lower queue = renders earlier (background transparent objects), higher queue = renders later (foreground). Example: water = 3000, characters = 3100, UI = 3500. Manual layering.
- Disable intersecting transparency: Avoid intersecting transparent objects (two glass windows overlapping). Causes unsolvable sorting (neither entirely in front/behind). Redesign geometry (non-overlapping glass panes) or use opaque materials.
- Per-mesh sorting: Renderer.sortingOrder (manual per-object sort). Higher order = renders later (draws on top). Use for: 2D sprites (layered UI, parallax backgrounds), critical 3D transparent objects (force specific order).

**Alpha Testing Optimization:**
- Set appropriate cutoff: _Cutoff = 0.5 typical (alpha <0.5 discarded, ≥0.5 visible). Lower cutoff (0.3) = more visible pixels (thicker foliage), higher cutoff (0.7) = fewer pixels (thinner foliage). Balance coverage vs performance.
- Enable GPU instancing: Alpha cutout renders as opaque (ZWrite On, depth sorting not required). Compatible with GPU instancing (batch thousands of trees/grass). Massive performance win (1 draw call for 1000 trees).
- Mipmap LOD bias: Alpha cutout with mipmaps causes thin features to disappear at distance (mipmap blends alpha, drops below cutoff). Adjust Texture > Mipmap LOD Bias = -0.5 to -1.0 (sharper mipmaps, preserves alpha cutout features). Or use Alpha to Coverage (MSAA required).
- Alpha to Coverage: Hardware feature (MSAA-based). Converts alpha to MSAA sample coverage (smooth transitions at edges). Shader: AlphaToMask On. Requires MSAA enabled (Quality Settings). Better edges than cutout (less aliasing) but MSAA performance cost.

**Transparency Performance:**
- Minimize overdraw: Transparent objects don't write depth (later transparent objects render over earlier, all layers drawn). Overdraw = pixels shaded multiple times (expensive). Profile with Overdraw mode (Scene view > Shading Mode > Overdraw). Red = high overdraw (10+ layers, serious issue).
- Reduce fill rate: Shrink transparent geometry (tight bounds around visible areas). Large transparent quads (full-screen UI panel at alpha 0.1) waste fill rate (processing invisible pixels). Trim geometry (remove transparent areas, tighter mesh).
- Sort front-to-back when possible: For opaque with transparency (trees with cutout leaves), render opaque parts first (depth writes), then transparent parts (depth tested, early-Z rejection reduces overdraw). Submesh approach (tree trunk = opaque material, leaves = cutout material).
- LOD for transparency: Distant transparent objects simplified (fewer layers, lower resolution textures, or switch to opaque). Example: distant glass windows = opaque reflective material (skip transparency), close glass = proper alpha blending.

**Special Techniques:**
- Dithered transparency: Use dither pattern instead of blending (checkerboard alpha, temporal resolve via TAA). Looks like alpha blending after TAA accumulation. Benefits: writes depth (no sorting), cheap (no read-modify-write), works with MSAA/TAA. Use for: fading objects (LOD transitions, soft hiding), VR (alpha blending expensive).
- Order-Independent Transparency (OIT): Advanced technique (A-buffer, depth peeling, weighted blended OIT). Resolves sorting issues (renders correctly regardless of order). Expensive: multiple passes or heavy bandwidth (per-pixel linked lists). HDRP supports limited OIT (Transparent Depth Postpass for refractions).
- Refraction: Distorts background via UV offset (reads screen texture, samples with offset UVs). Simulates light bending (glass, water). Requires GrabPass (Built-in) or _CameraOpaqueTexture (URP/HDRP). Expensive (extra render pass, full-screen texture copy).
- Fresnel transparency: Transparency varies by view angle (perpendicular = opaque, glancing = transparent). Simulates glass (edge-on view more transparent). Common for water, force fields. Implementation: alpha = lerp(minAlpha, maxAlpha, fresnel) where fresnel = 1 - dot(normal, view).

**Platform-Specific:**
- **PC**: Full transparency support. Alpha blending (smooth gradients), alpha cutout (foliage), refraction (screen-space distortion), OIT possible (advanced techniques). Overdraw manageable (high fill rate GPUs). 10-20 layers acceptable.
- **Consoles**: Balanced transparency. Alpha blending and cutout standard. Limited refraction (expensive copy pass). Avoid excessive overdraw (target <10 layers). Dithered transparency for LOD fading (efficient, TAA-friendly).
- **Switch**: Simplified transparency. Alpha cutout preferred (no sorting, depth writes). Minimal alpha blending (expensive overdraw). No refraction (bandwidth limited). Aggressive LOD (distant transparency = opaque or culled). Target <5 layers overdraw.
- **Mobile**: Minimal transparency. Alpha cutout only for complex scenes (foliage = cutout leaves). Very limited alpha blending (UI = alpha blend, gameplay = avoid). No refraction (bandwidth critical). Target <3 layers overdraw. Some games use opaque materials entirely (no transparency).

## Common Pitfalls

**Transparent Objects Rendering Incorrectly**: Developer sees transparent objects render in wrong order (closer object behind farther object). Depth sorting failure (Unity sorts by bounds center, incorrect for large/intersecting objects). Symptom: Glass window behind character renders on top, transparent UI elements z-fighting. Solution: Use manual render queue (Shader render queue, closer objects = higher queue number). Split large transparent objects (smaller pieces sort better). Avoid intersecting transparency.

**Transparent Objects Too Dark**: Developer creates transparent material, renders black/too dark. Forgot to disable depth writes (ZWrite On blocks other transparent objects, renders only closest layer). Symptom: Transparent objects opaque-looking, only one layer visible. Solution: Set ZWrite Off in transparent shader. Render queue = Transparent (3000+). Unity Standard Shader > Rendering Mode > Transparent does this automatically.

**Foliage Aliasing**: Developer uses alpha blending for leaves (smooth alpha). Leaves alias badly at distance (shimmering, crawling edges, mipmap blends alpha = vanishing leaves). Symptom: Tree leaves flicker/disappear at distance, heavy aliasing. Solution: Use alpha cutout (AlphaTest, clip), enable Alpha to Coverage (AlphaToMask On, requires MSAA), adjust Texture mipmap LOD bias (-0.5 to preserve features). Cutout + Alpha to Coverage = best quality.

**Excessive Transparency Overdraw**: Developer uses large full-screen transparent UI panels (alpha 0.1-0.3, barely visible). GPU shades every pixel under panel (wasted processing, expensive overdraw). 5-10 overlapping panels = GPU bound. Symptom: Low frame rate with transparent UI, Profiler shows GPU bound, Overdraw view shows red. Solution: Trim transparent geometry (remove transparent areas, tighter bounds). Use opaque materials where possible (solid color panels = opaque, not transparent). Reduce panel count (combine overlapping panels into single texture).

## Tools & Workflow

**Shader Blend Modes**: Shader > Blend mode declaration. Blend SrcAlpha OneMinusSrcAlpha (standard), Blend One One (additive), Blend DstColor Zero (multiplicative). ZWrite On/Off (depth writes). Queue tags (Background, Geometry, AlphaTest, Transparent, Overlay).

**Render Queue Inspector**: Material > Render Queue dropdown (Background 1000, Geometry 2000, AlphaTest 2450, Transparent 3000, Overlay 4000). Custom queue value (manual control, 3000-3999 for transparent, lower = earlier).

**Overdraw Debug Mode**: Scene view > Shading Mode > Overdraw. Visualizes fill rate cost (blue = 1x overdraw, green = 2-3x, yellow = 5-8x, red = 10+). Identify transparency hotspots (red areas = too much transparency, optimize).

**Frame Debugger**: View transparent objects render order. Shows draw calls in sequence (verify back-to-front order), blend modes per draw call, render queue values. Debug sorting issues (identify mis-ordered objects).

**Alpha Cutout Preview**: Material Inspector > Rendering Mode > Cutout, Cutoff slider. Real-time preview (Scene/Game view shows cutoff result). Tune cutoff value (balance coverage vs pixel count). Too low = thin features visible, too high = features disappear.

**GPU Profiler**: Rendering > Transparent Pass shows transparent rendering cost. Identify expensive materials (complex shaders, high overdraw). Compare transparent vs opaque cost (transparent should be <30% total frame time, higher = optimization needed).

**RenderDoc/PIX**: Capture transparent rendering. View blend mode states (verify correct blend equation), inspect overdraw (pixel history shows how many times pixel shaded), analyze fill rate (bandwidth usage per transparent pass).

## Related Topics

- [13.3 Post-Processing](13-03-Post-Processing.md) - Effects on transparent objects
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Transparency performance
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Overdraw identification
- [12.2 Shader Types](12-02-Shader-Types.md) - Transparent shaders

---

[← Previous: 13.3 Post-Processing](13-03-Post-Processing.md) | [Next: 13.5 Advanced Rendering →](13-05-Advanced-Rendering.md)
