# 26.1 Visual Debugging & Frame Debugger

[← Back to Chapter 26](../chapters/26-Debugging-Techniques.md) | [Main Index](../README.md)

Visual debugging = graphical tools to see what GPU is rendering (frame by frame analysis, draw call breakdown, layer visualization). Frame Debugger = per-frame breakdown (all draw calls, shaders, render targets), helps identify visual bugs (missing objects, wrong colors, overdraw = expensive). Techniques: wireframe view (see mesh topology, detect modeling issues), overdraw visualization (see pixel depth complexity = optimization opportunity), layer masking (render only specific objects = isolate issues).

---

## Overview

Frame Debugging Process: enable Frame Debugger (Window > Analysis > Frame Debugger), play scene, pause, step through draw calls (each call = one object or batch rendered). Inspector shows: shader used, render target, blend mode, culling state. Helpful for: finding missing meshes (render call list shows all = check if expected call present), identifying slow shaders (measure per-call GPU time if supported), seeing render target overwrites (detect ping-pong patterns = expensive).

Visualization Modes: (1) solid (normal rendering), (2) wireframe (edges visible = mesh topology), (3) overdraw (color = depth count = red = many layers = expensive), (4) normals (check normal direction = lighting direction). Mobile/console = limited tools (Frame Debugger PC-only mostly), use screenshots instead.

## Key Concepts

- **Frame Debugger**: Window > Analysis > Frame Debugger, see event tree (all GPU commands per frame). Expand = reveal details (Camera.Render > RenderLoopJob > batched calls = 100+ calls listed). Select call = see: shader, material, mesh, bounds. GPU time = if profiler supports (depends on platform = PC usually supports, mobile doesn't). Benefit = find bottleneck draw calls (if one call = 5ms = slow shader suspected).
- **Wireframe Mode**: Scene window top-right, toggle Wireframe (show mesh edges without fill). Reveals: mesh quality (tessellation density), modeling issues (degenerate triangles = tiny faces), deformation (skinning issues = vertices moving wrong). Useful for: mesh debugging (is LOD working correctly = lower LOD visible), physics debugging (collision shape visible = check alignment with visuals).
- **Overdraw Visualization**: Show pixel depth complexity (how many layers rendered per pixel). Red = many (expensive = optimization needed), green = fewer (efficient). Scenes with transparency = high overdraw natural (particles, UI = expect red). Opaque = should be low (1 layer per pixel = green = good). Helps identify: particles too dense, UI overlapping, foliage overdraw.
- **Layer Masking**: select layers to render (Scene window top-right Layer dropdown). Isolate: render only "Enemies" layer = see if enemy render correct, omit particles/UI = faster iteration. Debugging: disable Particles layer = verify non-particle FPS increase = particle cost = measured.
- **Shader Replacement**: override all shaders (use debug shader instead = visualize normals, UV, depth). Example: "Show Normals" shader = all objects show normal direction (pink/purple/blue hues = direction). Helps: detect modeling issues (backwards normals = blue instead of pink), verify UV mapping.

## Best Practices

**Frame Debugging Workflow**:
- Play scene, spot visual issue (missing object, wrong color).
- Open Frame Debugger, find relevant draw call (search shader name or object name = narrow list).
- Check: shader correct? render target correct? blend mode correct?
- If wrong = trace back to code (why shader not correct = initialization issue?).

**Wireframe Analysis**:
- Check LOD transitions (switch LODs = wireframe updates = LOD swap visual = verify working).
- Mesh quality: overdense (triangle count high = performance risk), underdense (faceted = visual artifact = blocky appearance).
- Deformation: skinned mesh wireframe shows bone influence (if distorted = skinning weight issue).

**Overdraw Optimization**:
- Target: opaque = 1x (one layer per pixel = green), transparent UI = 2-3x acceptable (UI layers stack), particles = 5-10x = expensive but necessary.
- Reduce overdraw: limit particle emitters (fewer particles = less overdraw), merge UI (fewer layers = lower overdraw), optimize foliage (LOD distant = simpler meshes = lower triangle count = lower overdraw).

**Platform-Specific**:
- **PC**: full Frame Debugger support (inspect every call = powerful debugging).
- **Console**: RenderDoc/PIX frame capture (similar to Frame Debugger = detailed analysis).
- **Mobile**: limited tools (Screenshot comparison preferred = render frame, compare to reference = visual regression detected).

## Common Pitfalls

**Wrong Shader Showing**: Frame Debugger shows different shader than expected (material assigned shader A, Frame Debugger shows shader B). Symptom: visual bug doesn't match shader code = confusing. Solution: check shader keywords (may use different variant = _NORMALMAP on/off = different shader compiled), or asset corruption (reimport material).

**Overdraw Misleading**: developer sees high overdraw = assumes particles expensive. Disables particles = no FPS improvement (CPU-bound instead = GPU not bottleneck). Symptom: optimization attempt = no gain. Solution: profile first (check if GPU-bound or CPU-bound = Profiler GPU/CPU time = tells truth).

**Frame Debugger Stall**: Frame Debugger open = rendering stalls (GPU waiting for CPU readback = performance fake = not representative). Measured FPS in Debugger = lower than normal gameplay (artificial stall). Solution: profile in Play mode without Debugger open = accurate numbers.

## Tools & Workflow

**Frame Debugger Steps**:
1. Window > Analysis > Frame Debugger > Enable
2. Play scene
3. Pause when issue visible
4. Expand event tree (Camera.Render > ...)
5. Select call = Inspector shows details
6. Check: Shader, Material, Mesh, Bounds

**Wireframe in Scene View**:
- Top-right dropdown > Wireframe (toggles)
- Or Gizmo menu > select Wireframe checkbox
- Editor shortcut: Shift + W (not all versions)

**Shader Replacement Script**:
```csharp
Camera cam = GetComponent<Camera>();
cam.SetReplacementShader("Shaders/ShowNormals", null); // All shaders replaced
// Later: cam.ResetReplacementShader(); // Restore
```

## Related Topics

- [26.2 GPU Debugging](26-02-GPU-Debugging.md) - GPU profilers
- [26.3 Performance Debugging](26-03-Performance-Debugging.md) - Performance analysis
- [26.4 Unity Debug Tools](26-04-Unity-Debug-Tools.md) - Profiler
- [04.3 Unity Built-In Tools](04-03-Unity-Built-In-Tools.md) - Profiling overview

---

[Previous: Chapter 25](../chapters/25-Build-Pipeline-And-Workflow.md) | [Next: 26.2 GPU Debugging →](26-02-GPU-Debugging.md)
