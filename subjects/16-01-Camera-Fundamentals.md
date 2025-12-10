# 16.1 Camera Fundamentals

[← Back to Chapter 16](../chapters/16-Camera-Systems.md) | [Main Index](../README.md)

Cameras define the viewpoint for rendering through projection matrices, view frustum, culling parameters, and rendering order to transform 3D scenes to 2D images.

---

## Overview

Unity Camera converts 3D world to 2D image: defines view position/rotation (where camera looks from), projection (perspective or orthographic), field of view (how much visible), clipping planes (near/far rendering distances). Camera renders scene to screen or RenderTexture: calculates view-projection matrix (transforms world space to clip space), culls objects (frustum culling eliminates off-screen objects), submits draw calls (renders visible geometry). Multiple cameras composite: UI camera overlays UI (depth = 1), main camera renders world (depth = 0), skybox camera renders background (depth = -1).

Projection types: Perspective (realistic depth, objects farther = smaller, FOV defines viewing angle), Orthographic (no perspective, parallel projection, objects same size regardless of distance, FOV defines view width/height). Perspective for: 3D games (first-person, third-person, realistic depth cues), orthographic for: 2D games (side-scrollers, top-down), UI (fixed-size elements), technical visualization (CAD, isometric views). Projection matrix converts camera space to clip space (GPU performs clipping, viewport transformation to screen coordinates).

Camera parameters: Field of View (vertical angle in degrees, 60° typical, wider = more visible + distortion), aspect ratio (width/height, typically screen resolution ratio), near plane (closest rendering distance, 0.1-1.0 typical), far plane (farthest distance, 100-10,000 typical). Near/far balance: tight range improves depth precision (reduces Z-fighting), wide range = more visible + depth fighting. Culling mask (layer bitmask, controls which layers rendered), depth (render order, lower renders first), clear flags (how buffer cleared before rendering).

## Key Concepts

- **View-Projection Matrix**: Combines view matrix (world to camera space, camera position/rotation) and projection matrix (camera to clip space, FOV/aspect/near/far). VP matrix = Projection × View. GPU transforms vertices: clip space = VP × world position. Unity calculates automatically (Camera component provides VP matrix to shaders via UNITY_MATRIX_VP).
- **Frustum**: View volume (truncated pyramid for perspective, box for orthographic). Six planes define frustum: near, far, left, right, top, bottom. Objects outside frustum culled (not rendered, frustum culling optimization). Unity calculates frustum from camera parameters (FOV, aspect, near/far planes), performs frustum culling automatically (GameObject.activeInHierarchy + Renderer.bounds intersects frustum).
- **Field of View (FOV)**: Vertical viewing angle (perspective projection). Typical: 60° (neutral, matches human peripheral vision), 90°+ (wide, fisheye distortion), 30° (narrow, telephoto, zoomed). Horizontal FOV calculated from vertical FOV + aspect ratio (HFOV = 2 × atan(tan(VFOV/2) × aspect)). Affects: how much visible (wider FOV = more visible + edge distortion), perceived speed (wider FOV = faster apparent motion).
- **Clipping Planes**: Near plane (minimum render distance, closer objects clipped), far plane (maximum distance, farther culled). Near typical: 0.1-1.0 (0.1 for first-person, 1.0 for better depth precision), far: 100-10,000 (100 for interiors, 1,000+ for open worlds). Depth precision: logarithmic distribution (more precision near, less far), tighter range = better precision (reduces Z-fighting). Reverse-Z improves (Unity 2021+, better near-plane precision).
- **Render Order (Depth)**: Camera.depth determines render order (lower values render first, higher overlay). Main camera depth = 0 (standard), UI camera = 1 (renders after, overlays), background camera = -1 (renders before, underlying layer). Use for: layer composition (background, world, UI, post-effects), multiple viewports (split-screen, Picture-in-Picture), rendering to textures then compositing.

## Best Practices

**Camera Configuration:**
- Perspective settings: FOV 60-70° (standard, comfortable viewing), aspect ratio match screen (Camera.aspect = Screen.width / Screen.height, or leave default for automatic). Near plane 0.3-1.0 (0.3 for tight spaces, 1.0 for open areas + better depth precision), far plane based on scene size (100 for rooms, 1,000 for cities, 5,000+ for landscapes). Avoid: near < 0.1 (depth fighting increases), far > 10,000 (precision suffers, skybox better for distant background).
- Orthographic settings: Size = half-height in world units (size 5 = 10 units tall view). Aspect ratio determines width (width = size × aspect × 2). Near/far still relevant (depth range for 2D layers, e.g., near = -100, far = 100 for 2D game layers). Use for: 2D games (consistent pixel size), UI (fixed layout), isometric views (technical visualization).
- Culling mask: Camera.cullingMask excludes layers (bitwise mask, 1 << layer = include layer). Main camera excludes UI (cullingMask = ~(1 << LayerMask.NameToLayer("UI"))), UI camera only UI (cullingMask = 1 << LayerMask.NameToLayer("UI")). Optimize rendering (skip irrelevant layers, reduce draw calls). Example: minimap camera excludes particles/UI (performance optimization, minimap doesn't need effects).
- Clear flags: Skybox (clears with skybox, typical for main camera), Solid Color (clears to flat color, typical for UI camera overlays), Depth Only (clears depth not color, overlays on previous camera), Don't Clear (no clear, accumulates, temporal effects). Choose based on: first camera = Skybox/Color (full clear), overlay cameras = Depth Only (preserve underlying rendering).

**Projection Matrix Customization:**
- Oblique frustum: Modify near plane (custom Camera.projectionMatrix, shifts near plane for effects like water reflections, portals). Use GeometryUtility.CalculateObliqueMatrix (Unity utility, creates oblique projection). Example: water reflection camera (oblique near plane = water surface, clips geometry below water).
- Off-center projection: Shift frustum center (asymmetric frustum, used for VR, multi-display setups, perspective-correct projections). Custom projection matrix (Camera.projectionMatrix = Matrix4x4.Frustum, manual control). Use for: VR eye rendering (each eye offset frustum), projection mapping (camera matches projector FOV).
- Custom FOV per axis: Standard FOV = vertical (horizontal calculated from aspect). Custom: set horizontal FOV directly (modify projection matrix, manual calculation). Rare use case (anamorphic widescreen effects, non-square pixels, specialized hardware).
- Reset projection: Camera.ResetProjectionMatrix() restores default projection (reverts custom matrices). Use after: temporary custom projection (reset to normal after effect), debugging (ensure standard projection for troubleshooting).

**Multi-Camera Setup:**
- Camera stacking: Order cameras by depth (background depth -1, main = 0, UI = 1). Lower depth renders first (composites bottom-to-top). Clear flags: first camera = Skybox (full clear), overlays = Depth Only (preserve color, clear depth for correct occlusion). Use for: layered rendering (background layer, world layer, UI layer), effects (render to texture then composite).
- Split-screen: Two cameras, different viewports (Camera.rect defines viewport, 0-1 normalized coordinates). Example: left camera rect = (0, 0, 0.5, 1) = left half, right camera = (0.5, 0, 0.5, 1) = right half. Both depth = 0 (render simultaneously, to different screen regions). Use for: local multiplayer, picture-in-picture minimaps.
- Render to texture: Camera.targetTexture = RT (renders to RenderTexture instead of screen). Use for: minimap (overhead camera renders to RT, UI displays texture), mirrors (reflection camera renders to RT, mirror material samples texture), portals (portal view rendered to RT, portal quad shows texture). See Chapter 15.1 for RenderTexture management.
- Camera enable/disable: Active cameras render every frame (enabled = true). Disable cameras not needed (camera.enabled = false, skips rendering = performance savings). Example: minimap camera disabled when minimap UI hidden (no wasted rendering). Re-enable when needed (toggle based on UI state).

**Performance Optimization:**
- Frustum culling validation: Unity performs automatic frustum culling (Renderer bounds vs camera frustum). Verify: Frame Debugger shows only visible objects rendered (off-screen objects absent). Optimize: use layers + culling mask (exclude entire layers from expensive cameras, e.g., reflection camera excludes particles/UI layers).
- Near/far plane tuning: Tight near/far range improves depth precision + culling efficiency. Near = closest object visible (not smaller, wastes precision), far = farthest object visible (not larger, wastes precision + culls efficiently). Example: indoor scene near = 0.3, far = 50 (room size), outdoor near = 1.0, far = 500 (visible distance). Adjust per scene type (tight range = better).
- Layered distance culling: Camera.layerCullDistances[layer] = distance (culls layer beyond distance). Example: "SmallProps" layer cull at 30m (small objects invisible far away, skip rendering). Per-layer control (environment = 300m, details = 30m). Significant savings (hundreds of draw calls culled for distant small objects).
- Occlusion culling: Unity Occlusion Culling bakes visibility (see Chapter 13.6). Camera uses baked data (automatically culls occluded objects, complements frustum culling). Enable: Window > Rendering > Occlusion Culling > Bake (generates visibility data). Runtime: automatic (camera queries visibility, culls hidden objects). Massive savings in dense scenes (cities, interiors, 50-90% objects culled).

**Platform-Specific:**
- **PC**: Wide FOV viable (60-90°, large monitors comfortable with wider FOV). Far plane large (1,000-10,000, open worlds visible to horizon). Multiple cameras acceptable (5-10 active cameras, minimap + reflections + portals, sufficient performance). High-resolution rendering (4K cameras, detailed rendering).
- **Consoles**: Standard FOV (60-75°, TV viewing distance). Moderate far plane (500-2,000, balance visibility/precision). Limited cameras (2-5 active, main + UI + 1-2 effects, fixed performance budget). 1080p-4K rendering (balance resolution/frame rate).
- **Switch**: Narrow FOV (55-65°, small screen). Short far plane (100-500, aggressive culling for performance). Minimal cameras (1-2 active, main + UI only, avoid extra render passes). 720p rendering (1080p docked, handheld 720p).
- **Mobile**: Narrow FOV (50-60°, small screens). Very short far plane (50-200, aggressive culling). Single camera typical (main only, avoid UI camera if possible = render UI in main camera, save render pass). Low resolution (540p-1080p, depends on device tier).

## Common Pitfalls

**Near Plane Too Close**: Developer sets near plane = 0.01 (wants to see very close objects). Depth precision suffers (Z-fighting on distant objects, flickering coplanar surfaces). Symptom: Distant objects flicker (depth fighting, insufficient precision at far plane). Solution: Increase near plane (0.3-1.0, redistributes depth precision, reduces fighting). Only use near <0.1 for: extreme close-ups (rare), or 32-bit depth buffer (eliminates precision issues).

**Far Plane Too Large**: Developer sets far plane = 100,000 (wants entire world visible). Depth precision terrible (Z-fighting everywhere, even close objects flicker). Skybox renders at infinity (far plane irrelevant for skybox, skybox always visible). Symptom: Z-fighting on all geometry (insufficient precision, massive depth range). Solution: Reduce far plane (100-1,000, based on actual visible distance), use skybox for distant background (skybox renders at infinity, doesn't use depth buffer).

**Multiple Cameras No Depth Order**: Developer adds UI camera, forgets to set depth (both cameras depth = 0). Render order undefined (platform-dependent, UI may render before world = invisible). Symptom: UI not visible, or flickering (render order wrong, z-fighting between cameras). Solution: Set camera depth explicitly (main camera depth = 0, UI camera = 1, ensures UI renders last = visible on top). Always specify depth for multi-camera setups.

**Wrong Clear Flags**: Developer adds overlay camera (UI), leaves clear flags = Skybox. Camera clears screen (erases previous rendering), only UI visible (world rendering cleared). Symptom: Only last camera's output visible (previous cameras erased by clear). Solution: Overlay cameras use Depth Only clear flags (preserves color from previous cameras, clears only depth for correct occlusion). First camera = Skybox/Color (full clear), subsequent cameras = Depth Only (overlay).

## Tools & Workflow

**Camera Component**: GameObject > Camera (adds Camera component). Inspector: Projection (Perspective/Orthographic), FOV (60° default), Clipping Planes (Near/Far), Clear Flags (Skybox/Color/Depth/None), Culling Mask (layer checkboxes), Depth (render order), Target Texture (RenderTexture asset or None for screen).

**Scene View Camera**: Scene view mimics game camera (WASD to move, right-click drag to look). GameObject > Align View to Selected (Ctrl+Shift+F, moves scene camera to match selected camera GameObject). Useful for: previewing camera view (see exactly what camera renders), adjusting camera placement (visualize camera frustum gizmo).

**Camera Gizmo**: Scene view shows camera frustum (wire frame truncated pyramid for perspective, box for orthographic). Color indicates: white = unselected, yellow = selected. Frustum shows: near plane (small rectangle), far plane (large rectangle), FOV cone. Adjust: move/rotate camera GameObject (frustum updates in real-time).

**Camera Preview**: Inspector > Camera component (selected) shows preview (small game view window at bottom). Real-time rendering (shows camera output while editing). Useful for: multi-camera setup (preview each camera's view), render-to-texture (preview RT output), adjusting camera parameters (immediate visual feedback).

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows camera passes (each camera = separate render section), draw calls per camera (what each camera renders), camera parameters (VP matrix, clear flags, culling). Verify: cameras rendering in correct order (depth values correct), appropriate objects per camera (culling mask working).

**Profiler**: Unity Profiler > CPU > Camera.Render (time per camera). GPU Profiler (per-camera rendering time). Identify: expensive cameras (high render time, optimize culling/resolution), redundant cameras (disabled when not needed). Optimize: reduce camera count (combine cameras if possible), lower resolution (render-to-texture cameras at half-res).

## Related Topics

- [16.2 Multiple Camera Rendering](16-02-Multiple-Camera-Rendering.md) - Camera stacking and composition
- [13.6 Culling Techniques](13-06-Culling-Techniques.md) - Frustum and occlusion culling
- [15.1 RenderTexture Management](15-01-RenderTexture-Management.md) - Rendering cameras to textures
- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - Camera rendering performance

---

[← Previous: 15.4 MSAA and Resolve](../subjects/15-04-MSAA-And-Resolve.md) | [Next: 16.2 Multiple Camera Rendering →](16-02-Multiple-Camera-Rendering.md)
