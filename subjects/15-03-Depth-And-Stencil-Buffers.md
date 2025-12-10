# 15.3 Depth and Stencil Buffers

[← Back to Chapter 15](../chapters/15-Render-Targets-And-Framebuffers.md) | [Main Index](../README.md)

Depth buffers enable Z-testing for occlusion, while stencil buffers provide masking for effects like shadows, portals, and outlines through per-pixel integer tagging.

---

## Overview

Depth buffer (Z-buffer) stores per-pixel depth (distance from camera, typically 0-1 range), enables depth testing (compare incoming fragment depth vs buffer depth, discard if farther = occluded). Without depth buffer, objects render in submission order (wrong occlusion, back objects draw over front). With depth buffer, GPU automatically culls occluded pixels (correct visibility, front objects obscure back). Unity default: 24-bit depth buffer (16.7 million depth values, sufficient for most scenes), optionally 16-bit (lower precision, depth fighting possible) or 32-bit (extreme precision, large scenes).

Stencil buffer: per-pixel 8-bit integer (0-255 value per pixel), used for masking/tagging. GPU compares incoming fragment's stencil ref vs buffer value (conditional rendering based on stencil), updates buffer per operation (increment, decrement, replace). Use cases: stencil shadows (increment on front faces, decrement on back, non-zero = in shadow), portals (render portal contents only where stencil = portal mask), outlines (render object then outline where stencil != object), reflections (render reflection only in mirror region). Cost: minimal (8 bits per pixel, typically packed with 24-bit depth as D24S8 format, zero overhead).

Depth-stencil formats: D24S8 (24-bit depth + 8-bit stencil, standard), D32 (32-bit depth, no stencil), D16 (16-bit depth, mobile), D32_SFloat_S8 (32-bit float depth + 8-bit stencil, high precision). Unity default: D24S8 (balance precision/memory, stencil available). Mobile: D16 (lower memory, acceptable precision for typical scenes). High-precision: D32 for extreme depth ranges (astronomical scales, very large worlds).

## Key Concepts

- **Depth Testing**: GPU compares fragment depth vs depth buffer (ZTest operation). Operations: Less (default, render if closer), LEqual (render if closer or equal), Greater (render if farther, uncommon), Always (render regardless, disables depth test), Never (never render, unusual). After passing, fragment rendered and depth buffer updated (ZWrite On, writes depth), or not updated (ZWrite Off, read-only depth test).
- **Early-Z Optimization**: GPU performs depth test before fragment shader (culls occluded pixels early, skips expensive shading). Requires: opaque rendering (alpha clip/blending disables early-Z), ZWrite On (depth buffer writable), shader doesn't modify depth (no clip/discard in shader, or declared as early_fragment_tests). Massive savings: skip shading 50-90% occluded pixels (huge performance gain for complex scenes).
- **Depth Precision**: Non-linear depth distribution (more precision near camera, less far). 24-bit depth: near = 0.001 precision, far = 1.0 precision (less detailed). Depth fighting: insufficient precision causes Z-fighting (flickering between coplanar surfaces). Solutions: increase near plane (0.1 → 1.0, redistributes precision), use reverse-Z (1 at near, 0 at far, better precision distribution), 32-bit depth (more values, eliminates fighting).
- **Stencil Operations**: Stencil test compares stencil ref vs buffer (Comp function: Equal, NotEqual, Less, Greater, Always). On pass/fail: operation updates buffer (Keep, Zero, Replace, IncrSat, DecrSat, Invert, IncrWrap, DecrWrap). Example: increment on pass (Stencil { Ref 1 Comp Always Pass IncrSat }) = increments buffer value by 1 (clamped to 255).
- **Stencil Masking**: ReadMask and WriteMask (8-bit masks, control which stencil bits participate). ReadMask: ANDed with buffer before compare (Comp Equal with ReadMask 0x0F = compares only lower 4 bits). WriteMask: controls which bits writable (WriteMask 0x0F = only lower 4 bits update, upper 4 unchanged). Allows: multiple independent stencil channels (lower 4 bits = shadows, upper 4 bits = portals).

## Best Practices

**Depth Buffer Configuration:**
- Standard depth: 24-bit depth (RenderTexture depthBits = 24) for most scenes. Sufficient precision (0.1 near plane, 1000 far plane = acceptable Z-fighting avoidance). Unity default (Camera depth buffer = 24-bit). Consoles/PC standard.
- Low-precision depth: 16-bit (depthBits = 16) for mobile/performance-critical. Lower precision (Z-fighting more likely, avoid large depth ranges). Optimize: near plane = 1.0 (not 0.1, improves precision distribution), far plane = 100 (not 1000, tighter range = better precision). Mobile default (memory savings, acceptable quality).
- High-precision depth: 32-bit float (D32_SFloat) for extreme ranges (astronomical games, flight sims, huge open worlds). Eliminates Z-fighting (float precision sufficient for near = 0.01, far = 100,000). Cost: 2x memory vs 24-bit (minor, depth buffer small relative to textures). Use only when needed (Z-fighting unavoidable with 24-bit).
- Reverse-Z: Modern technique (1.0 at near plane, 0.0 at far). Better precision distribution (more values near camera where needed). Unity 2021+ supports (enable in Project Settings). Requires shader compatibility (some custom shaders need updates). Eliminates most Z-fighting (without 32-bit overhead).

**Stencil for Shadows (Stencil Shadows):**
- Shadow volume technique: Render shadow volume geometry (extruded from silhouette edges). Front faces increment stencil (Pass IncrSat), back faces decrement (Pass DecrSat). Final: stencil > 0 = in shadow (shadow accumulation), stencil = 0 = lit. Old technique (replaced by shadow maps, but useful for specific effects like hard shadows, portals).
- Setup: Clear stencil to 0 (ClearRenderTarget clears stencil), render shadow volume front faces (Comp Always, Pass IncrSat, ZWrite Off), render back faces (Comp Always, Pass DecrSat, ZWrite Off). Apply shadows: render fullscreen quad (Comp Greater, Ref 0 = renders where stencil > 0), darkens shadowed pixels.
- Advantages: pixel-perfect hard shadows (no filtering, crisp edges), works with any light (point, spot, directional). Disadvantages: expensive (shadow volume geometry + fillrate), only hard shadows (no soft shadows/penumbra), complex implementation (silhouette edge detection). Modern alternative: shadow maps (Chapter 13.2).

**Stencil for Outlines:**
- Two-pass outlines: Render object (writes stencil), render outline (reads stencil, renders where object NOT present). Pass 1: render object normally (Stencil { Ref 1 Comp Always Pass Replace }) = writes 1 to stencil where object rendered. Pass 2: render object slightly larger (scale 1.05x) with outline shader (Stencil { Ref 1 Comp NotEqual }) = renders only where stencil != 1 (halo around object).
- Setup: Pass 1 renders object (normal material, writes stencil = 1), Pass 2 renders enlarged object (outline color, Comp NotEqual = only outside original object, Cull Front = only back faces visible = outline). Result: outline drawn around object (common for selection highlights, cel-shading effects).
- Alternatives: post-process outlines (edge detection on normals/depth, cheaper but less control), geometry shader outlines (extrude edges in geometry shader, more precise). Stencil method: simple, artist-controllable (outline thickness = scale factor), works on any mesh.

**Stencil for Portals:**
- Render portal contents only in portal region: Render portal surface (writes stencil), render portal view (reads stencil, renders only where stencil = portal mask). Pass 1: render portal quad (Stencil { Ref 2 Comp Always Pass Replace }, ColorMask 0 = no color output, only stencil write). Pass 2: render portal camera view (Stencil { Ref 2 Comp Equal }) = only renders where stencil = 2 (inside portal bounds).
- Setup: Portal quad writes stencil = 2 (identifies portal pixels), portal camera renders scene (with stencil test Ref 2 Comp Equal = only where portal), final composite (portal view visible only in portal region, rest of screen shows main scene). Use case: mirrors (reflect camera view in mirror region), windows (render different scene through window), actual portals (see through to connected room).
- Optimization: Render portal contents to RenderTexture (separate RT for portal view, sample RT in portal quad shader), cheaper than stencil (single portal render, reused across frames if static). Stencil method useful for: real-time portals (dynamic portal view per frame), multiple portals (each with unique stencil ref).

**Depth Prepass:**
- Technique: Render depth-only pass before color (fills depth buffer, no color output), then render color (depth test ONLY mode, fragments already culled by depth, skip occluded pixels). Saves: fragment shader cost for occluded pixels (complex shaders don't run on hidden surfaces). Cost: additional depth-only pass (vertex processing + rasterization).
- Implementation: Pass 1 depth-only (ColorMask 0, ZWrite On, ZTest Less, simple vertex shader, no fragment shader), Pass 2 color (ZWrite Off, ZTest Equal = render only where depth matches, full shaders). Unity: ShaderLab Pass with ColorMask 0 for depth prepass.
- When beneficial: Complex fragment shaders (PBR with many textures, expensive lighting), high depth complexity (many overlapping objects, heavy overdraw), large scenes (lots of occlusion). Not beneficial: simple shaders (depth prepass overhead > savings), forward rendering with low overdraw (most pixels visible).
- Early-Z alternative: Modern GPUs have early-Z (automatic depth test before fragment shader, no manual prepass needed). Early-Z requires: render front-to-back (occludes back objects early), opaque rendering (no alpha blend/clip), ZWrite On. Early-Z often sufficient (manual depth prepass less common on modern hardware).

**Platform-Specific:**
- **PC**: 24-bit or 32-bit depth (D24S8 standard, D32 for high precision). Reverse-Z supported (Unity 2021+, improved precision). Stencil always available (8-bit stencil in D24S8). Early-Z efficient (hardware optimized, renders front-to-back for best performance).
- **Consoles**: 24-bit depth standard (D24S8). Reverse-Z supported (PS5/Xbox Series X). Stencil available (8-bit). Early-Z critical (fixed hardware, maximize early culling via front-to-back rendering). Depth prepass optional (early-Z usually sufficient).
- **Switch**: 16-bit or 24-bit depth (D16 for memory savings, D24S8 if precision needed). Stencil available (but minimize usage, fillrate limited). Early-Z important (weak GPU, avoid overdraw). Depth prepass rarely beneficial (overhead > savings on low-end hardware).
- **Mobile**: 16-bit depth typical (D16, memory constrained). Stencil available but expensive (tile-based rendering, stencil operations flush tiles = bandwidth cost). Early-Z critical (very weak GPUs, render front-to-back mandatory). Avoid stencil (use alternatives like multi-pass without stencil), minimize depth complexity.

## Common Pitfalls

**No Depth Buffer**: Developer creates RenderTexture with depthBits = 0 (no depth buffer), renders 3D scene. Depth testing disabled (objects render in submission order, wrong occlusion). Symptom: Back objects visible through front objects (incorrect Z-order, looks broken). Solution: Specify depth buffer (RenderTexture.GetTemporary(w, h, 24, format), 24-bit depth), or permanent RT (new RenderTexture { depthBits = 24 }). Only use depthBits = 0 for: 2D rendering (sprites, UI), post-processing (fullscreen effects, no depth needed).

**ZWrite Off with Opaque**: Developer sets ZWrite Off on opaque material (thinking saves performance). Objects don't write depth (depth buffer not updated), subsequent objects render incorrectly (assume nothing there, render through). Symptom: Occlusion broken (objects render through each other, flickering as camera moves). Solution: ZWrite On for opaque objects (standard, updates depth buffer correctly). ZWrite Off only for: transparent objects (alpha blending, render after opaques, read depth but don't write), post-process effects (fullscreen quads, no depth).

**Stencil Not Cleared**: Developer uses stencil operations, doesn't clear stencil buffer (assumes 0 default). Previous frame's stencil data remains (incorrect stencil values, effects broken). Symptom: Stencil effects work intermittently (first frame wrong, subsequent frames correct), or artifacts (random stencil masking). Solution: Clear stencil (CommandBuffer.ClearRenderTarget(clearDepth: true, clearColor: false, backgroundColor, clearStencil: true), or Camera clear flags = Solid Color/Skybox clears stencil). Always clear stencil before stencil-based effects.

**Depth Fighting (Z-Fighting)**: Developer places coplanar surfaces (floor + decal at same position), uses default near plane (0.1). Insufficient depth precision causes Z-fighting (surfaces flicker, alternates which renders on top). Symptom: Flickering between overlapping surfaces (decals on walls, coplanar geometry). Solution: Offset geometry slightly (decal 0.01 units above surface), increase near plane (0.3 instead of 0.1, better precision), use 32-bit depth (eliminates fighting), or polygon offset (shader property OffsetFactor/OffsetUnits, biases depth slightly).

## Tools & Workflow

**Depth Buffer Setup**: RenderTexture creation (GetTemporary(width, height, depthBits, format), depthBits = 0/16/24/32). Camera (Camera.depthTextureMode = DepthTextureMode.Depth, enables depth texture for shaders to sample). Shader access: UNITY_DECLARE_DEPTH_TEXTURE(_CameraDepthTexture), sample with tex2D or SAMPLE_DEPTH_TEXTURE.

**Stencil Shader Syntax**: ShaderLab Stencil block. Example: `Stencil { Ref 1 Comp Equal Pass IncrSat Fail Keep ZFail Keep ReadMask 255 WriteMask 255 }`. Ref = reference value (0-255), Comp = comparison (Equal/NotEqual/Less/Greater/Always/Never), Pass/Fail/ZFail = operations on test result (Keep/Zero/Replace/IncrSat/DecrSat/Invert/IncrWrap/DecrWrap).

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows depth/stencil state per draw call (ZTest, ZWrite, stencil operations). Preview depth buffer (click draw call, select Depth in preview dropdown, grayscale depth visualization). Preview stencil (Stencil dropdown, color-coded stencil values).

**Scene View Depth Visualization**: Scene view > Shading Mode > Depth (visualizes depth buffer, dark = near, bright = far). Useful for: verifying depth range (all white = far plane too close, all black = near plane too far), identifying depth precision issues (banding = insufficient precision).

**Graphics Debuggers**: RenderDoc, PIX, Nsight Graphics. Show depth/stencil buffers per pass (depth visualization, stencil heatmap). Verify: depth buffer filling correctly (occluded objects culled), stencil operations working (correct stencil values per pixel). Debug Z-fighting (depth buffer visualization shows coplanar surfaces).

**Depth Texture Mode**: Camera.depthTextureMode (enables depth texture for shader sampling). Modes: None (no depth texture), Depth (depth only), DepthNormals (depth + normals encoded). Unity generates texture (accessible as _CameraDepthTexture in shaders). Use for: post-processing (SSAO, depth of field, fog), effects requiring depth (edge detection, outlines).

## Related Topics

- [15.1 RenderTexture Management](15-01-RenderTexture-Management.md) - Depth buffer in RenderTextures
- [15.4 MSAA and Resolve](15-04-MSAA-And-Resolve.md) - Depth with MSAA
- [13.2 Shadow Techniques](13-02-Shadow-Techniques.md) - Depth-based shadows
- [13.6 Culling Techniques](13-06-Culling-Techniques.md) - Early-Z and occlusion

---

[← Previous: 15.2 Multi-Render Targets](15-02-Multi-Render-Targets.md) | [Next: 15.4 MSAA and Resolve →](15-04-MSAA-And-Resolve.md)
