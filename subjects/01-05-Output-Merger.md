# 1.5 Output Merger

[← Back to Chapter 1](../chapters/01-Graphics-Pipeline-Fundamentals.md) | [Main Index](../README.md)

The output merger stage performs final per-pixel operations: depth testing, stencil operations, and color blending. These fixed-function operations determine which fragments update the framebuffer and how.

---

## Overview

After fragment shaders compute pixel colors, the output merger tests each fragment against depth and stencil buffers before writing to render targets. Depth testing rejects occluded fragments (cheaper than early-Z which happens before shading), while stencil testing enables masking effects (reflections, shadows, portals). Color blending combines fragment output with existing framebuffer values for transparency, additive effects, or compositing.

Output merger operations occur at incredible rates—Xbox Series X has 112 ROPs (Render Output units) processing billions of pixels per second. However, bandwidth-intensive operations like blending or reading render targets can bottleneck. Modern deferred rendering relies heavily on MRTs (Multi-Render Targets), writing geometry to 4-6 buffers simultaneously (albedo, normal, depth, metallic/smoothness, emission). Understanding ROP throughput and bandwidth costs is critical for high-resolution or high-framerate targets.

Depth testing alone is simple, but combined with stencil operations and blending, it enables complex rendering techniques: shadow volumes (stencil counting), order-independent transparency (depth peeling), screen-space reflections (stencil masking), and deferred decals (stencil marking). Mastering output merger configuration unlocks advanced graphics programming techniques essential for AAA titles.

## Key Concepts

- **Depth Testing**: Comparing fragment depth against Z-buffer value. Common tests: `Less` (fragment closer than buffer = visible), `LessEqual`, `Greater`, `Always`. Failed fragments are discarded without updating framebuffer.
- **Stencil Buffer**: 8-bit integer buffer accompanying depth buffer. Enables per-pixel masking via compare operations (Equal, NotEqual, etc.) and update operations (Keep, Replace, Increment). Used for shadows, portals, decals, and outlining.
- **Blending**: Combining fragment shader output with existing framebuffer color via blend equation. Common modes: Alpha (transparency), Additive (particles/lights), Multiplicative (lightmaps). Costs bandwidth reading framebuffer before writing.
- **Render Output Unit (ROP)**: Fixed-function hardware performing depth/stencil/blend operations. Modern GPUs have 64-128 ROPs. Bottleneck when doing expensive blending or writing high-resolution MRTs.
- **Multi-Render Target (MRT)**: Writing fragment shader outputs to multiple render targets simultaneously. Deferred rendering uses MRT to populate G-buffer (albedo, normal, depth, properties) in one geometry pass.

## Best Practices

**Depth Testing Configuration:**
- Use `DepthTest Less` for opaque forward rendering—standard Z-buffering where closest fragment wins. Pair with `DepthWrite On` to update Z-buffer for subsequent draws.
- Use `DepthTest Equal` with `DepthWrite Off` for additional passes on same geometry (decals, additional lighting). Ensures fragments only shade where geometry already rendered.
- Disable depth testing (`DepthTest Always`) only for fullscreen effects (post-processing, UI) where depth is irrelevant. Saves depth read bandwidth slightly.
- Use reverse-Z (`DepthTest Greater`) on modern platforms for better precision. Maps far plane to 0, near plane to 1, improving floating-point precision distribution.

**Stencil Operations:**
- Reserve stencil bits for different features: bits 0-3 for shadows, bit 4 for reflections, bits 5-7 for portals. Use bit masking (`Stencil ReadMask`, `WriteMask`) to isolate operations.
- Increment stencil for shadow volume front faces, decrement for back faces. Final stencil value = number of shadow volumes enclosing pixel. Non-zero = shadowed.
- Use `Stencil Comp Equal [RefValue]` to mask rendering to specific regions (e.g., `RefValue = 1` only renders where stencil is 1). Common for mirrors, portals, or screen-space effects.
- Clear stencil only when changing techniques. Don't clear every frame if reusing stencil data—clearing costs bandwidth.

**Blending Strategies:**
- Minimize blended draws—each blend operation reads and writes framebuffer, doubling bandwidth. 1080p RGBA32 blend = 16MB read + 16MB write per fullscreen quad.
- Sort transparents back-to-front as required for alpha blending correctness (`Blend SrcAlpha OneMinusSrcAlpha`). But minimize transparent objects—they disable early-Z and cost overdraw.
- Use additive blending (`Blend One One`) for particles and lights in forward rendering. Doesn't require sorting (commutative) and often looks better than alpha.
- Premultiplied alpha (`Blend One OneMinusSrcAlpha`) avoids dark fringing on transparent edges. Multiply color by alpha in shader before output.

**MRT Usage:**
- Limit MRT count based on platform: PC/PS5/Xbox handle 8 MRTs well, Switch struggles past 4. Each additional RT multiplies bandwidth linearly.
- Use compact formats: R11G11B10_Float for HDR albedo (12 bytes) instead of RGBA16_Float (8 bytes). R8G8_SNorm for normals (2 bytes) with octahedral encoding.
- Write only necessary data. Don't output to MRT slot if unused. Shader `SV_Target1` output is wasted if RT1 isn't bound.
- Prefer deferred rendering for many lights (100+ point lights). Forward rendering costs per-light per-object; deferred costs per-light per-pixel—huge savings in light-heavy scenes.

**Platform-Specific:**
- **PC DirectX 12**: Use `OMSetRenderTargets` to configure up to 8 render targets. Check hardware capabilities—older GPUs may have lower ROP counts (32-64 vs 128).
- **Xbox Series X/S**: 112 ROPs on Series X, 80 on Series S. Optimize for Series S by reducing MRT count or resolution (1440p instead of 4K).
- **PlayStation 5**: RDNA 2 ROPs are highly efficient. Use fast clear values (depth = 1.0 or 0.0) to leverage compressed clear optimizations.
- **Switch**: Limited ROP throughput and bandwidth. Use 4 MRTs max, prefer R8G8B8A8 or compressed formats. Consider forward+ instead of full deferred rendering.

## Common Pitfalls

**Incorrect Depth Test Mode**: Setting `DepthTest Greater` on opaque objects causes reversed depth logic—objects render back-to-front, maximizing overdraw. Symptom: scene renders correctly but GPU time doubles. Verify depth test in material properties matches shader expectations (standard Z or reverse Z).

**Forgetting Depth Writes**: `DepthTest Less` with `DepthWrite Off` on opaque objects causes incorrect occlusion—first-drawn object wins regardless of depth. Subsequent objects test against empty/stale Z-buffer. Symptom: rendering order dependency, flickering as objects overlap. Always enable depth writes for opaque geometry.

**Blending Bandwidth Explosion**: 4K resolution with 10x overdraw from particles = 830 million blended pixels per frame. At RGBA16 = 6.6GB/sec bandwidth consumed just by blending. Exceeds Xbox One's 68 GB/sec memory budget significantly. Solution: Reduce particle resolution (render at 1080p, upscale), use simpler blending (additive without read), or limit particle density.

## Tools & Workflow

**RenderDoc**: Pipeline State > Output Merger shows depth/stencil/blend configuration for selected draw. "Texture Viewer" displays depth/stencil buffers visually—identify stencil masks or depth discontinuities. Pixel debugger shows exact depth test results per fragment.

**PIX**: "Render Target" view displays all bound render targets, including depth/stencil. "Pixel History" shows every fragment affecting a pixel: depth values, test results (passed/failed), blend operations. Essential for debugging overdraw or incorrect transparency.

**Unity Frame Debugger**: Click draw call to see render target configuration. "Stencil" and "Depth" previews show buffer contents. "Shader Properties" displays blend mode, depth test, and stencil operations configured in material.

**NVIDIA Nsight Graphics**: "ROP Throughput" counter shows render output utilization. >90% suggests ROP bottleneck (high-resolution MRTs, excessive blending). "Memory" view reveals framebuffer read/write bandwidth—identify blend-heavy draws.

**Platform Profilers**: PlayStation Razor shows ROP cache efficiency and fast-clear optimization status. Xbox PIX exposes color/depth compression ratios—low compression (10:1) suggests poor cache efficiency or random write patterns.

## Related Topics

- [1.4 Fragment Processing](01-04-Fragment-Processing.md) - Previous pipeline stage before output merger
- [8.1 Deferred Rendering](13-05-Advanced-Rendering.md) - MRT usage for G-buffer rendering
- [19.1 Shadows Fundamentals](13-02-Shadow-Techniques.md) - Stencil shadow volumes
- [20.1 Transparency Techniques](13-04-Transparency-And-Alpha.md) - Alpha blending and OIT

---

[← Previous: 1.4 Fragment Processing](01-04-Fragment-Processing.md) | [Next: 1.6 Compute Pipeline →](01-06-Compute-Pipeline.md)
