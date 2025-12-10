# 16.2 Multiple Camera Rendering

[← Back to Chapter 16](../chapters/16-Camera-Systems.md) | [Main Index](../README.md)

Multiple cameras enable split-screen multiplayer, minimaps, picture-in-picture, and layered rendering through viewport configuration, render order, and camera stacking techniques.

---

## Overview

Multiple cameras render scene from different viewpoints or for different purposes: split-screen (two player cameras side-by-side), minimap (overhead camera for UI), reflections (mirror camera renders reflection), UI overlay (separate camera for UI layer). Each camera configured independently: viewport (screen region, 0-1 normalized rect), depth (render order), culling mask (which layers visible), clear flags (how buffer cleared). Unity renders cameras by depth order (lower depth first), composites to screen or RenderTextures.

Camera stacking: Base camera (renders main scene, depth = 0, clears framebuffer), Overlay cameras (render additional layers, depth > 0, preserve base camera output). Example: base camera renders world (Skybox clear, depth = 0), UI camera overlays UI (Depth Only clear, depth = 1 = renders after world, UI visible on top). URP Camera Stack: explicit overlay list (base camera lists overlay cameras, rendered sequentially), more control than depth-only ordering.

Viewport rendering: Camera.rect splits screen (normalized 0-1 coordinates, x/y = bottom-left, width/height = size). Split-screen: left camera rect = (0, 0, 0.5, 1) = left half, right = (0.5, 0, 0.5, 1) = right half. Picture-in-picture: main camera rect = (0, 0, 1, 1) = full screen, minimap = (0.75, 0.75, 0.25, 0.25) = top-right quarter. Cost: each camera renders separately (double cameras = ~double render cost, unless culling significantly different).

## Key Concepts

- **Camera Depth**: Render order priority (Camera.depth, integer -100 to 100). Lower renders first (depth = -1 renders before 0), higher overlays (depth = 1 renders after 0). Multiple cameras same depth = undefined order (platform-dependent, avoid). Use for: layering (background cameras < main < UI cameras), compositing (render base then overlay effects).
- **Viewport Rect**: Screen region (Camera.rect, Rect with x/y/width/height in 0-1 range). (0, 0, 1, 1) = full screen (default), (0, 0, 0.5, 1) = left half, (0.5, 0.5, 0.5, 0.5) = top-right quarter. Origin bottom-left (Unity screen space). Cameras render only to viewport (clipped to rect). Use for: split-screen, PIP, multi-monitor.
- **Clear Flags**: How framebuffer cleared before camera renders. Skybox/SolidColor = full clear (color+depth, first camera typically uses), Depth Only = clear depth keep color (overlay cameras, preserve underlying rendering), Don't Clear = no clear (accumulate, temporal effects). Choose: first camera = full clear, overlays = Depth Only (correct occlusion + preserve previous rendering).
- **Culling Mask**: Layer bitmask (Camera.cullingMask, which layers camera renders). Bitmask operations: include layer = `mask |= (1 << layer)`, exclude = `mask &= ~(1 << layer)`. UI camera only UI layer (cullingMask = 1 << LayerMask.NameToLayer("UI")), main camera excludes UI (cullingMask = ~(1 << LayerMask.NameToLayer("UI"))). Optimize: skip irrelevant layers per camera (minimap excludes particles/UI, saves rendering).
- **URP Camera Stack**: URP-specific feature (Base camera + Overlay camera list). Base camera (Universal Rendering Pipeline Data > Overlay Cameras list), overlay cameras render after base (sequential, explicit order). Better control than depth-only (clear ordering, explicit dependencies). Use for: complex multi-camera (UI + post-effects + debug overlays), URP projects (built-in support).

## Best Practices

**Split-Screen Setup:**
- Two-player split: Create two cameras (Camera1 depth = 0 rect = (0, 0, 0.5, 1), Camera2 depth = 0 rect = (0.5, 0, 0.5, 1)). Both render simultaneously (same depth, different viewports = parallel rendering). Clear flags: both Skybox (independent clears, each camera full scene). Culling masks: identical (both see same layers), or customized (Camera1 excludes Camera2's UI layer, separate UI per player).
- Four-player split: Quadrants (Camera1 = (0, 0.5, 0.5, 0.5) top-left, Camera2 = (0.5, 0.5, 0.5, 0.5) top-right, Camera3 = (0, 0, 0.5, 0.5) bottom-left, Camera4 = (0.5, 0, 0.5, 0.5) bottom-right). All depth = 0 (parallel). Performance: 4x render cost (each camera full scene rendering), optimize via: reduced resolution (render each camera at 50% screen res, acceptable for split-screen), aggressive culling (each camera far plane shorter, cull distance per camera).
- Dynamic split-screen: Adjust viewport at runtime (script changes Camera.rect based on player count). Example: single player = full screen (1,1), two players = horizontal split (0.5,1 each), four = quadrants (0.5,0.5 each). Transition smoothly (lerp rect over time, animated split-screen transitions).
- Audio listener: Only one AudioListener active (multiple AudioListeners = audio conflicts, undefined behavior). Main player camera has AudioListener component, other split-screen cameras don't. Or: script manages (activates AudioListener for player 1 camera only).

**Minimap Camera Configuration:**
- Orthographic projection: Minimap uses orthographic (Camera.orthographic = true, size = world units visible). Top-down view (camera looks down Y-axis, rotation = (90, 0, 0)). Size determines zoom (size = 50 = 100 units visible vertically, smaller size = more zoomed). No perspective distortion (minimap flat, consistent object sizes).
- Render to texture: Camera.targetTexture = minimapRT (renders to RenderTexture, not screen). UI displays texture (RawImage component, texture = minimapRT). Resolution: 512x512 or 1024x1024 typical (minimap small on screen, high resolution unnecessary). Update rate: every frame (default), or every N frames (script toggles camera.enabled, reduces cost for static minimaps).
- Layering: Minimap camera cullingMask excludes (UI layer, particle effects, detailed props = not relevant to minimap). Include: terrain, major structures, characters/vehicles (minimap-relevant objects). Separate "MinimapVisible" layer (objects tagged for minimap, minimap camera only renders this layer). Optimize: minimal rendering (skip expensive shaders, use simple unlit materials for minimap objects).
- Iconographic minimap: Instead of 3D rendering, use icons (2D sprites for players/objectives). Camera renders terrain only (3D base), script overlays icons (UI sprites positioned based on world coordinates). More readable (clear icons vs tiny 3D models), cheaper (fewer objects rendered). Hybrid: 3D terrain + 2D icons for characters.

**Reflection Camera Setup:**
- Mirror reflection: Create reflection camera (mirror plane script calculates reflected position/rotation). Render to texture (targetTexture = reflectionRT), mirror material samples texture (shader reads reflectionRT). Oblique projection (near plane = mirror plane, clips geometry behind mirror = optimization). Culling mask: exclude player (player shouldn't see self in own reflection, typically).
- Planar reflections: Unity Water shader uses this technique (reflection camera for water surface). Script: calculate reflection matrix (reflect camera across water plane), set camera worldToCameraMatrix (custom view matrix), render to RT, apply to water material. Cost: double rendering (reflection camera renders full scene, expensive), optimize via: reduced resolution (reflection RT half-res), reduced far plane (reflection camera shorter draw distance), skippped layers (reflection excludes particles/small details).
- Reflection probe alternative: Reflection probes (Chapter 14.3) cheaper than realtime reflection cameras (baked cubemaps, zero runtime rendering). Use reflection camera only when: dynamic reflections required (moving objects in mirror, player character visible), planar surface (flat mirrors, water = reflection camera accurate). Use probes when: static environment (baked reflections sufficient), non-planar (curved surfaces, metallic objects = probes better).

**UI Camera Overlay:**
- Separate UI camera: Camera renders only UI layer (cullingMask = UI layer only), depth > main camera (depth = 1, renders after world). Orthographic projection (Camera.orthographic = true, fixed UI coordinates). Clear flags: Depth Only (preserves world rendering, clears depth for UI occlusion).
- Benefits: Independent control (UI camera resolution/post-effects separate from world), render order guaranteed (UI always on top via depth), culling optimization (main camera excludes UI layer, UI camera excludes world = each camera optimized). Cost: additional camera render (overhead ~0.5-2ms, acceptable for UI).
- Canvas Screen Space Camera: Alternative to separate UI camera (Canvas render mode = Screen Space - Camera, links to camera). Canvas rendered by main camera (at specified depth), simpler than separate camera (one camera handles world + UI). Use when: simple UI (overlay only, no special effects), prefer single camera (less overhead). Use separate camera when: complex UI (post-effects on UI only, 3D UI elements), strict layering (ensure UI always renders last).

**Performance Optimization:**
- Limit camera count: Each camera = separate render (CPU overhead + GPU rendering). Minimize: 2-4 cameras typical (main + UI + optional minimap/reflection), avoid 10+ cameras (excessive overhead, unless many disabled). Disable unused cameras (camera.enabled = false, skips rendering entirely = zero cost).
- Viewport rendering cost: Split-screen cameras render independently (2 cameras = 2x cost, 4 cameras = 4x). Optimize: reduce per-camera resolution (render each viewport at 75% res, upscale to screen), shorten far plane per camera (each camera shorter draw distance = cull more objects), LOD bias per camera (reduce lodBias for split-screen cameras, earlier LOD switches = lower poly counts).
- Culling mask optimization: Minimap camera excludes 50% layers (particles, effects, small props), reflection camera excludes 30% (player, UI, distant details). Massive draw call savings (Frame Debugger shows cameras rendering subset of objects). Profile per-camera: Profiler > Camera.Render shows time per camera (identify expensive cameras, optimize culling).
- Render texture resolution: Minimap/reflection RTs don't need full screen resolution. Minimap = 512x512 (small UI element, high resolution wasted), reflections = half-res (720p reflection for 1080p screen, imperceptible quality loss + 4x faster). Balance quality/performance (test different resolutions, find minimum acceptable).

**Platform-Specific:**
- **PC**: 5-10 cameras viable (main + UI + minimap + multiple reflections/portals). Split-screen 2-4 players (sufficient performance, 1080p-4K per viewport acceptable). High-resolution RTs (1024x1024 minimap, full-res reflections). Flexible configurations (complex multi-camera setups feasible).
- **Consoles**: 3-5 cameras typical (main + UI + minimap, limited reflections). Split-screen 2 players common (4 players possible at 900p-1080p, performance-bound). Moderate RT resolutions (512-1024 minimap, half-res reflections). Balance camera count/viewport resolution (fixed performance budget).
- **Switch**: 2-3 cameras maximum (main + UI, minimap optional). Split-screen 2 players only (4 players prohibitively expensive, <30fps). Low RT resolutions (256-512 minimap, quarter-res reflections). Minimize cameras (each camera significant cost, weak GPU).
- **Mobile**: 1-2 cameras (main only, or main + UI if necessary). No split-screen (performance insufficient). Very low RT resolutions (256x256 minimap maximum). Avoid multiple cameras (use single camera + UI overlay, separate camera too expensive).

## Common Pitfalls

**Multiple Cameras Same Depth**: Developer adds UI camera, forgets to set depth (both main and UI cameras depth = 0). Render order undefined (platform-dependent, UI may render first = hidden under world). Symptom: UI invisible or flickering (wrong render order). Solution: Explicitly set camera depths (main = 0, UI = 1, ensures UI renders last). Always specify depth for multi-camera setups.

**Wrong Clear Flags on Overlay**: Developer adds UI camera, leaves clear flags = Skybox. Camera clears screen (erases main camera rendering), only UI visible (world disappears). Symptom: Only UI visible, world rendering missing (cleared by UI camera). Solution: UI camera clear flags = Depth Only (preserves color from main camera, clears only depth for UI occlusion). Overlay cameras should never fully clear (use Depth Only).

**Too Many Active Cameras**: Developer creates 15 cameras (main + UI + minimap + 10 reflection cameras for mirrors/water). Each camera renders every frame (15x render cost, frame rate tanks). Symptom: Low frame rate (Profiler shows 15 Camera.Render entries, massive overhead), many cameras rendering redundantly. Solution: Disable unused cameras (camera.enabled = false), use reflection probes instead of cameras (baked reflections, zero runtime cost), limit realtime reflection cameras to 1-2 (key mirrors only, rest use probes).

**Split-Screen Full Resolution**: Developer implements 4-player split-screen, each camera renders at full screen resolution (thinking viewport controls resolution). Each camera renders full 1080p (4 cameras = 4x 1080p = 4K equivalent workload), frame rate drops to 15fps. Symptom: Extremely low frame rate in split-screen (GPU overloaded, rendering 4x pixels). Solution: Reduce render resolution per camera (use ScalableBufferManager or render to lower-res RT per camera, upscale to viewport), or accept lower overall resolution (split-screen at 900p instead of 1080p, each viewport effectively 450p).

## Tools & Workflow

**Camera Component Configuration**: Inspector > Camera component. Viewport Rect (x/y/width/height sliders, or manual input), Depth (integer, render order), Clear Flags (dropdown: Skybox/Solid Color/Depth Only/Don't Clear), Culling Mask (layer checkboxes). Preview (small window shows camera view).

**Camera Gizmo**: Scene view shows camera frustums (colored wireframes). Multiple cameras = multiple frustums visible (verify viewport positions, frustum overlaps, camera orientations). Select camera GameObject (highlight frustum, easier to identify which camera). Color: white = unselected, yellow = selected.

**Game View**: Game tab shows final composited output (all cameras rendered). Aspect ratio dropdown (simulates different resolutions/aspects, verify split-screen/viewport layouts). Stats button (shows draw calls, batches, tris = total across all cameras). Free Aspect (resizable window, test dynamic viewports).

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows cameras in render order (each camera = separate section, lists draw calls per camera). Verify: cameras rendering in correct order (depth values correct), appropriate objects per camera (culling masks working), no redundant rendering (cameras not overlapping unnecessarily).

**Unity Profiler**: Profiler > CPU > Camera.Render (shows time per camera, e.g., "Camera.Render (Main Camera)", "Camera.Render (UI Camera)"). Identify expensive cameras (high ms per camera = optimize culling/resolution). GPU Profiler (per-camera GPU time). Memory > RenderTextures (cameras rendering to RTs, shows RT sizes).

**URP Camera Stack**: URP camera Inspector > Rendering > Render Type (Base or Overlay). Base camera > Stack section (add Overlay cameras to list). Overlay cameras render after base (sequential order in list). More explicit control than depth-only (clear dependencies, easier debugging).

## Related Topics

- [16.1 Camera Fundamentals](16-01-Camera-Fundamentals.md) - Camera basics
- [15.1 RenderTexture Management](15-01-RenderTexture-Management.md) - Render to texture
- [16.3 Camera Effects](16-03-Camera-Effects.md) - Per-camera post-processing
- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - Multi-camera overhead

---

[← Previous: 16.1 Camera Fundamentals](16-01-Camera-Fundamentals.md) | [Next: 16.3 Camera Effects →](16-03-Camera-Effects.md)
