# 15.1 RenderTexture Management

[← Back to Chapter 15](../chapters/15-Render-Targets-And-Framebuffers.md) | [Main Index](../README.md)

RenderTextures are GPU textures that capture camera output for effects like mirrors, minimaps, portals, post-processing, and deferred rendering, requiring careful memory and performance management.

---

## Overview

RenderTexture (RT) is a texture rendered to by camera: camera renders scene to RT instead of screen, RT used as texture input for materials, UI, or further rendering. Use cases: reflections (mirror renders second camera to RT, mirror material samples RT), minimaps (overhead camera renders to RT, UI displays RT), portals (second camera renders portal view to RT, portal quad samples RT), post-processing (render scene to RT, apply effects, output to screen). Cost: additional rendering pass (camera renders entire scene to RT = duplicate rendering work), memory overhead (RT texture = width × height × format, e.g., 1920×1080 RGBA32 = 8MB).

RenderTexture creation: temporary (RenderTexture.GetTemporary, pooled, automatic reuse) or permanent (new RenderTexture(), manual lifetime management). Temporary RTs efficient (Unity pools textures, minimal allocation overhead), use for: per-frame effects (post-processing, temporary buffers), short-lived operations (single-pass effects). Permanent RTs: explicit control (custom resolution, format, lifetime), use for: persistent textures (minimap camera always active), complex multi-frame effects (temporal anti-aliasing accumulation).

Memory management critical: RT memory significant (1080p RGBA32 = 8MB, 4K = 32MB, multiple RTs = hundreds of MB). Releasing RTs: RenderTexture.ReleaseTemporary (returns to pool, reusable), rt.Release() (frees GPU memory for permanent RTs). Not releasing = memory leak (GPU memory exhaustion, crashes on low-memory platforms). Profile RT usage: Unity Profiler > Memory > RenderTextures shows allocated RTs (resolution, format, memory usage).

## Key Concepts

- **RenderTexture Creation**: Temporary (RenderTexture.GetTemporary, pooled by Unity) or permanent (new RenderTexture(), manual management). Temporary: specify resolution, format, depth buffer (GetTemporary(width, height, depthBits, format)). Permanent: configure properties (width, height, format, depth, antiAliasing), call Create(). Temporary preferred (efficient pooling, automatic memory reuse).
- **Texture Formats**: ARGB32/RGBA32 (8-bit per channel, 32 bits total, standard color), ARGBHalf (16-bit float per channel, 64 bits, HDR), ARGBFloat (32-bit float per channel, 128 bits, very high precision), R8/RG16 (single/dual channel, memory efficient). Choose format: color rendering = ARGB32 (LDR) or ARGBHalf (HDR), depth-only = R16/R32 (single channel depth), normals = RG16 (two-channel normal encoding).
- **Depth Buffer**: RenderTexture includes depth buffer (depthBits parameter: 0, 16, 24, 32). 0 = no depth (alpha blending only, no depth testing), 16 = basic depth (sufficient for most cases), 24 = standard depth (typical default), 32 = high precision (for very large scenes, depth fighting prevention). Depth buffer memory: adds to RT size (1080p + 24-bit depth = 8MB color + 2MB depth = 10MB total).
- **Antialiasing**: RenderTexture.antiAliasing (MSAA samples: 1, 2, 4, 8). 1 = no AA, 2 = 2x MSAA, 4 = 4x MSAA (common), 8 = 8x MSAA (high quality). MSAA increases memory (4x MSAA = 4x memory for color/depth). Expensive: rendering cost + memory. Use for: high-quality RTs (minimap, character portraits), avoid for: temporary effects (post-processing, single-use RTs).
- **Camera Target**: Camera.targetTexture = RT. Camera renders to RT instead of screen. Multiple cameras can render to same RT (sequential rendering, overwrite or additive). Clear flags important: Skybox/Color = clears RT (first camera rendering to RT), Depth Only = doesn't clear color (overlays on existing RT content), Don't Clear = no clearing (accumulates across frames, temporal effects).

## Best Practices

**Temporary RenderTexture Pooling:**
- Use GetTemporary: RenderTexture.GetTemporary(width, height, depthBits, format) for short-lived RTs (single frame, single pass). Unity pools textures (reuses existing RTs matching parameters, minimal allocation). Always Release: RenderTexture.ReleaseTemporary(rt) when done (returns to pool for reuse). Pattern: Get at start of function, use, Release at end (or finally block).
- Matching parameters: Unity pools by exact match (width, height, depth, format must match). Similar RTs reuse (GetTemporary(1920, 1080, 24, ARGB32) twice = reuses same texture). Mismatch = new allocation (GetTemporary(1920, 1080, 24, ARGB32) then (1920, 1080, 16, ARGB32) = two separate textures). Standardize parameters (consistent resolutions/formats = better pooling).
- Scope management: Get/Release in same scope (avoid leaks). Bad: GetTemporary in function, forget ReleaseTemporary (RT never returned to pool, accumulates). Good: try-finally block (ensures Release even if exception). Or: using custom wrapper (IDisposable RT wrapper, automatic Release on dispose).
- Avoid permanent for temporary: Don't use new RenderTexture() for per-frame effects (no pooling, manual memory management, slower). Permanent RTs for: persistent textures (minimap always active, character portrait cached), long-lived buffers (temporal AA accumulation, multi-frame effects). Temporary for: post-processing passes, single-use effects, intermediate buffers.

**Resolution and Scaling:**
- Match screen resolution: Camera.pixelWidth/pixelHeight for full-screen effects (post-processing). Ensures 1:1 pixel mapping (no scaling, sharp result). Example: RT for post-processing = Screen.width × Screen.height (matches display resolution).
- Downsampling for performance: Half-resolution RTs (Screen.width / 2, Screen.height / 2) for expensive effects (bloom, depth of field, expensive post-processing). 1/4 cost (half width × half height = 1/4 pixels), slight quality loss (acceptable for blurry effects like bloom). Blur/glow effects benefit (already blurry, lower resolution imperceptible).
- Fixed resolution for specific uses: Minimap = fixed 512x512 or 1024x1024 (independent of screen resolution, UI scales texture). Character portraits = 512x512 (consistent quality, UI displays at specific size). Avoid: screen-dependent resolution for fixed-size UI elements (unnecessary memory for large screens).
- Dynamic resolution: Adjust RT resolution based on performance (Quality Settings, dynamic resolution scale). Example: target 60fps, if <60fps = reduce RT resolution (90% scale), if >60fps = increase (100% scale). Unity Dynamic Resolution (HDRP/URP feature, automatic RT scaling for frame rate).

**Format Selection:**
- Standard color: RenderTextureFormat.ARGB32 or DefaultHDR (DefaultHDR = platform-specific HDR format, typically ARGBHalf on modern platforms). ARGB32 for LDR (0-1 color range, typical rendering), DefaultHDR for HDR (>1 color values, bloom/tonemapping). Mobile: ARGB32 or RGB565 (16-bit, lower memory, slight banding).
- Depth-only: R16 or Depth (16-bit depth, single channel). Use for: shadow maps (depth-only rendering, no color), depth-based effects (SSAO, depth of field requiring separate depth). Saves memory (1 channel vs 4 channels, 1/4 memory).
- Normal buffers: RG16 (two-channel 16-bit, encodes XY normals, reconstruct Z). Use for: deferred rendering G-buffer, normal-based effects (edge detection, SSAO). Saves memory vs ARGB32 (2 channels vs 4, half memory).
- High precision: ARGBFloat (32-bit float per channel) for: scientific visualization (high dynamic range data), precision-critical (accumulation buffers, precision loss unacceptable). Expensive: 4x memory vs ARGB32 (128 bits vs 32), slower (float operations).

**Camera Configuration:**
- Set targetTexture: camera.targetTexture = rt (render to RT), camera.targetTexture = null (render to screen). Change at runtime (script controls camera output). Multiple cameras: different cameras render to different RTs (minimap camera = minimap RT, main camera = screen), or same RT (multiple views composited).
- Clear flags: Skybox/SolidColor = clears RT (full overwrite, use for first camera rendering to RT). Depth Only = clears depth, keeps color (overlay on existing RT, e.g., second camera adds to first camera's output). Don't Clear = no clear (temporal accumulation, motion vectors, multi-frame effects). Choose based on use case (single render = clear, additive = depth only, accumulation = don't clear).
- Culling mask: Camera.cullingMask controls layers rendered to RT. Minimap example: minimap camera culls only "MinimapVisible" layer (excludes UI, particles, effects not relevant to minimap). Reflection camera: culls "Player" layer (player not visible in own reflection). Optimizes rendering (skip unnecessary objects, faster RT rendering).
- Render depth: camera.depth controls render order (lower depth renders first). Main camera depth = 0 (renders last, to screen), minimap camera depth = -1 (renders before main, to RT, RT ready for UI display). Order important (RT must render before material samples it, or visual lag/artifacts).

**Memory Profiling:**
- Unity Profiler: Profiler > Memory > RenderTextures (lists all RTs, shows resolution, format, memory usage). Identify: large RTs (4K textures = 32MB+, candidates for downsampling), unused RTs (allocated but not released, memory leaks), excessive count (100+ RTs = too many, reduce or pool better).
- Memory warnings: Monitor Profiler > Memory > Total Reserved (GPU memory usage, includes RTs). High usage (approaching VRAM limit) = reduce RT count/resolution. Consoles: fixed memory budgets (e.g., 8GB shared, RTs compete with textures/meshes), mobile: very limited (2-4GB total, aggressive RT reduction needed).
- Release verification: Verify ReleaseTemporary called (Profiler shows RT count decreasing after Release). Memory leak symptom: RT count grows every frame (hundreds allocated, never released). Fix: ensure all GetTemporary matched with ReleaseTemporary (use try-finally or IDisposable wrapper).

**Platform-Specific:**
- **PC**: Large RTs viable (4K RTs acceptable, 6-8GB VRAM typical). HDR formats (ARGBHalf common, high quality). Multiple RTs (10-20 active RTs for complex effects, sufficient VRAM). MSAA (4x common, 8x for high-end).
- **Consoles**: Moderate RTs (1080p-1440p typical, 2-4GB available for RTs). HDR formats supported (ARGBHalf). Limit RT count (5-10 active, memory shared with textures/meshes). MSAA (2x-4x, balance quality/memory).
- **Switch**: Small RTs (720p max, 1GB available for RTs). LDR formats (ARGB32, avoid HDR = too much memory). Minimal RT count (1-3 active, very memory constrained). No MSAA (memory prohibitive, use FXAA post-process instead).
- **Mobile**: Very small RTs (540p-720p, <500MB for RTs). LDR formats (ARGB32 or RGB565 for extreme memory saving). Single RT typical (minimap or post-processing, not both). No MSAA (use FXAA, MSAA doubles memory cost).

## Common Pitfalls

**Not Releasing Temporary RenderTextures**: Developer calls RenderTexture.GetTemporary every frame (post-processing effect), forgets ReleaseTemporary. RTs accumulate (pool never returns textures, new allocations every frame). Memory usage grows (100 frames = 100 RTs × 8MB = 800MB leaked). Symptom: Profiler shows hundreds of RTs, memory usage increasing every frame, eventual crash (out of VRAM). Solution: Always pair GetTemporary with ReleaseTemporary (try-finally pattern, or using statement with IDisposable wrapper). Verify Profiler shows stable RT count (same count every frame, indicates proper pooling).

**Using Permanent RenderTextures for Temporary Effects**: Developer creates new RenderTexture() for post-processing (new RT every frame, applies effect, discards). No pooling (Unity doesn't reuse permanent RTs automatically), memory allocated/deallocated constantly (GC pressure, fragmentation). Symptom: Performance stutters (allocation spikes), Profiler shows RT allocations every frame. Solution: Use RenderTexture.GetTemporary (pooled, efficient reuse), or create permanent RT once (reuse across frames). GetTemporary preferred for temporary effects (automatic pooling, zero allocation after first frame).

**Excessive RenderTexture Resolution**: Developer creates full-resolution RTs for all effects (bloom RT = 4K, DOF RT = 4K, every effect = screen resolution). Memory explodes (10 effects × 32MB = 320MB for RTs alone). Symptom: High VRAM usage, slow performance (large RTs = more GPU work), crashes on low-end (out of VRAM). Solution: Downsample RTs (bloom = half-resolution 1080p, DOF = half-resolution, only final composite full-resolution). Blur/glow effects already blurry (lower resolution imperceptible). Reduces memory 4x (half width × half height).

**Forgetting Depth Buffer**: Developer creates RenderTexture without depth buffer (depthBits = 0), renders 3D scene. Depth testing disabled (no Z-buffer), objects render in submission order (wrong occlusion, back objects drawn over front). Symptom: Rendering looks wrong (objects in wrong order, no depth sorting, transparency artifacts). Solution: Specify depth buffer (GetTemporary(w, h, 24, format), 24-bit depth typical). Only use depthBits = 0 for: 2D rendering (UI, sprites, no depth testing needed), alpha-blended effects (depth testing not required).

## Tools & Workflow

**RenderTexture API**: Create temporary (RenderTexture.GetTemporary(w, h, depth, format)), release (RenderTexture.ReleaseTemporary(rt)). Create permanent (var rt = new RenderTexture(w, h, depth)), configure (rt.format, rt.antiAliasing), initialize (rt.Create()), cleanup (rt.Release(), Destroy(rt)).

**Camera Targeting**: Camera component > Target Texture slot (assign RT asset), or script (camera.targetTexture = rt). Null targetTexture = render to screen (default). Change at runtime (script toggles camera output between screen and RT).

**Unity Profiler**: Profiler > Memory > RenderTextures (shows all allocated RTs, resolution, format, memory). GPU Profiler (Profiler > GPU, or Frame Debugger) shows RT usage per pass (which passes render to which RTs). Monitor: total RT memory, RT count (should be stable, not growing), large RTs (candidates for downsampling).

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows rendering steps (which camera renders to which RT, draw calls per RT). Verify: RT cleared properly (clear color/depth visible), RT rendered before sampling (no temporal lag), RT resolution correct (matches expected size).

**Graphics Settings**: Project Settings > Graphics > Tier Settings (per-platform RT format defaults). Edit > Project Settings > Quality > Render Textures (global RT quality settings, e.g., resolution scale per quality tier). Configure: Low quality = half-resolution RTs, High quality = full-resolution.

**Shader Sampling**: Sample RT in shader (_MainTex = RT assigned to material). Use tex2D(_MainTex, uv) in fragment shader. For effects: Blit (Graphics.Blit(source, destination, material), renders source RT through material to destination RT), common for post-processing chains.

## Related Topics

- [15.2 Multi-Render Targets](15-02-Multi-Render-Targets.md) - MRT for deferred rendering
- [13.3 Post-Processing](13-03-Post-Processing.md) - RT for post effects
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - RT memory management
- [16.2 Multiple Camera Rendering](16-02-Multiple-Camera-Rendering.md) - Cameras with RTs

---

[← Previous: 14.4 Real-Time Global Illumination](../subjects/14-04-Real-Time-Global-Illumination.md) | [Next: 15.2 Multi-Render Targets →](15-02-Multi-Render-Targets.md)
