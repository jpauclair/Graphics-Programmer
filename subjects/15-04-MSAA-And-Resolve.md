# 15.4 MSAA and Resolve

[← Back to Chapter 15](../chapters/15-Render-Targets-And-Framebuffers.md) | [Main Index](../README.md)

Multisample Anti-Aliasing (MSAA) reduces edge aliasing through multi-sampling geometry edges, requiring resolve operations to convert multi-sampled buffers to displayable textures.

---

## Overview

MSAA anti-aliases geometry edges: renders at multiple sub-pixel samples (2x, 4x, 8x = 2, 4, 8 samples per pixel), fragment shader runs once per pixel (not per sample, efficient), coverage mask determines which samples covered (edge pixels = partial coverage, interior = full). Resolve operation averages samples (2x = average 2 samples, 4x = 4 samples) to produce final pixel color. Result: smooth edges (stair-stepping reduced), interior pixels unchanged (no blur, shader runs once). Cost: memory (4x MSAA = 4x framebuffer size), bandwidth (resolve reads all samples, writes single pixel).

MSAA vs post-process AA: MSAA = hardware-based (GPU samples geometry edges, high quality, only edges affected), post-process (FXAA, TAA) = shader-based (analyzes final image, applies blur, affects entire image). MSAA advantages: no blur (only edges smoothed, interior sharp), temporal stability (no flicker/ghosting like TAA). MSAA disadvantages: expensive memory (2-8x framebuffer size), doesn't fix shader aliasing (texture sampling, specular highlights still alias), incompatible with deferred rendering (G-buffer + MSAA = excessive memory).

Resolve operation: converts MSAA render target to regular texture (multi-sample to single-sample). Automatic resolve (Unity resolves on present, or when sampling MSAA RT), manual resolve (explicit ResolveAntiAliasedSurface, control timing). Resolve averaging: simple average (sum samples / sample count), or custom resolve (shader-based, weighted average, specific sample selection). Unity default: automatic resolve (transparent to developer, happens on RT switch or final present).

## Key Concepts

- **Sample Count**: MSAA level (2x, 4x, 8x, 16x = samples per pixel). Higher = smoother edges + more memory/bandwidth. 2x MSAA = 2x framebuffer memory (double size), 4x = 4x (common standard), 8x = 8x (high-end), 16x = 16x (extreme, rare). Typical: 4x MSAA (good quality, acceptable cost), 2x for performance (moderate improvement, low cost), 8x for quality (diminishing returns, expensive).
- **Coverage Mask**: Per-sample bit indicating geometry coverage (1 = sample covered, 0 = not covered). Triangle rasterization sets mask (samples inside triangle = 1, outside = 0). Fragment shader runs once (generates single color), GPU writes color to covered samples only (based on mask). Edge pixels: partial coverage (some samples covered, some not = blended edge), interior pixels: full coverage (all samples covered = solid color).
- **Shader Invocation**: Fragment shader runs once per pixel (not per sample, cost savings). Centroid sampling (shader samples textures at covered sample centroid, avoids sampling outside geometry). SV_SampleIndex (optional, allows per-sample shading, expensive, forces shader to run per sample = negates MSAA efficiency). Standard MSAA: single shader invocation per pixel (efficient, typical usage).
- **Resolve Pass**: Averages MSAA samples to single color (4x MSAA = average 4 samples per pixel). Types: box filter (simple average, fast), custom filter (weighted average, edge-directed). Unity: automatic box filter (sufficient quality, fast). Manual: ResolveAntiAliasedSurface (explicit resolve, control source/dest). Cost: memory bandwidth (read N samples, write 1 pixel, bandwidth = N+1 memory accesses per pixel).
- **Depth/Stencil MSAA**: Depth buffer also multi-sampled (4x MSAA = depth buffer 4x size). Depth test per sample (occlusion tested at sub-pixel resolution, accurate edge occlusion). Stencil multi-sampled similarly (per-sample stencil values). Resolve: depth buffer typically not resolved (discarded after rendering), or min/max resolve (keep nearest/farthest sample).

## Best Practices

**MSAA Configuration:**
- Enable MSAA: QualitySettings.antiAliasing = 2/4/8 (global setting, applies to screen), or RenderTexture.antiAliasing = 2/4/8 (per-RT MSAA). Unity applies MSAA to main framebuffer (screen rendering) and RTs (if specified). Shader support automatic (no shader changes needed for basic MSAA).
- Choose sample count: 4x MSAA standard (good quality, ~2-3ms cost on PC/consoles), 2x MSAA for performance (moderate quality, ~1ms), 8x for quality (high-end only, ~5-8ms, diminishing returns). Test on target platform (consoles/mobile = 2x-4x maximum, PC = 4x-8x viable). Higher sample counts: <10% visual improvement over 4x (not worth 2x cost).
- Deferred rendering incompatibility: Deferred + MSAA = G-buffer multi-sampled (4x MSAA = 4x G-buffer memory, prohibitive). Unity URP deferred: no MSAA support (use post-process AA like FXAA/TAA instead). Forward rendering: MSAA works well (efficient, typical use case). Solution: forward pipeline with MSAA, or deferred with TAA.
- Transparency handling: MSAA doesn't fix alpha-blended transparency (shader aliasing, not geometry edge aliasing). Transparent objects: render after MSAA resolve (to resolved buffer, no MSAA benefit), or use alpha-to-coverage (A2C, converts alpha to coverage mask, approximates MSAA for alpha clip). A2C: Shader keyword "alphatest" with A2C enabled (good for foliage, fences with alpha cutout).

**Resolve Operations:**
- Automatic resolve: Unity resolves MSAA RT when: sampling RT as texture (shader reads RT, triggers resolve), switching render targets (rendering to different RT, resolves previous), final present (resolving to screen). Developer does nothing (Unity handles automatically). Use for: most cases (simple, efficient, no manual management).
- Manual resolve: Graphics.ResolveAntiAliasedSurface(msaaRT, resolvedRT) explicitly resolves. Use when: need control over timing (resolve at specific point, avoid automatic resolves), custom resolve destination (resolve to specific RT), platform-specific optimizations (e.g., resolve to different format).
- Resolve destination: Can resolve to: same RT (MSAA RT resolves to internal single-sample surface, typical), different RT (manual resolve to separate RT, custom format/resolution). Unity default: in-place resolve (MSAA RT internally resolves, transparent to user). Custom: resolve to smaller RT (e.g., MSAA 1080p resolves to 1080p non-MSAA), or different format.
- Discard vs resolve: Some platforms support discard (MSAA data discarded, no resolve, saves bandwidth). Use for: depth-only MSAA (depth buffer not needed after rendering, discard samples), intermediate buffers (MSAA RT used temporarily, discarded before final). Unity: automatic (discards MSAA when possible, e.g., depth buffer after opaque rendering).

**MSAA with RenderTextures:**
- Create MSAA RT: RenderTexture.GetTemporary(w, h, depthBits, format, RenderTextureReadWrite.Default, antiAliasing: 4) for 4x MSAA. Or permanent: new RenderTexture { width = w, height = h, antiAliasing = 4, ... }. Specify antiAliasing parameter (2, 4, 8), Unity creates multi-sampled RT.
- Rendering to MSAA RT: Camera.targetTexture = msaaRT (camera renders with MSAA to RT), or CommandBuffer.SetRenderTarget(msaaRT) (manual rendering). Unity handles MSAA automatically (fragment shaders run once per pixel, hardware manages samples).
- Sampling MSAA RT: Shader samples MSAA RT as texture (tex2D(_MainTex, uv)). Unity automatically resolves MSAA before sampling (converts multi-sample to single-sample texture, transparent to shader). Resolve happens first time RT sampled (cached, reused for subsequent samples until RT rendered to again).
- MSAA + post-processing: Post-processing requires resolved RT (effects work on single-sample textures). Unity resolves MSAA automatically before post-processing (MSAA RT -> resolve -> post-process passes). Or: render to MSAA RT, manual resolve, apply post-processing to resolved RT.

**Performance Optimization:**
- Memory cost: MSAA multiplies framebuffer memory (4x MSAA = 4x color + 4x depth = 8x total memory vs non-MSAA). 1080p ARGB32 + D24S8 = 8MB color + 2MB depth = 10MB. 4x MSAA = 32MB color + 8MB depth = 40MB (4x increase). Monitor: Unity Profiler > Memory > RenderTextures (shows MSAA RT memory usage).
- Bandwidth cost: MSAA increases memory traffic (rendering writes 4x samples, resolve reads 4x + writes 1x = 5x bandwidth per pixel vs non-MSAA). High-resolution + MSAA = bandwidth bottleneck (4K 4x MSAA = 200MB per frame at 60fps = 12GB/s). Reduce: lower resolution (1080p instead of 4K, 4x less bandwidth), lower MSAA (2x instead of 4x, 2x less).
- Tile-based GPUs (mobile/Switch): MSAA efficient (multi-sampled tiles stay in on-chip cache, resolved on write to memory, minimal bandwidth overhead vs traditional GPU). 4x MSAA on mobile: acceptable cost (resolve happens in cache, bandwidth impact small). But: memory still 4x (framebuffer size, can be limiting on low-memory devices).
- MSAA alternatives: Post-process AA (FXAA = cheap, slight blur, no memory cost; TAA = high quality, temporal stability, no MSAA memory), SMAA (morphological AA, better than FXAA, more expensive), DLSS/FSR (upscaling + AA, better than MSAA for performance). Choose MSAA when: forward rendering (compatible), memory available (2-4GB+ VRAM), targeting PC/consoles (sufficient performance). Choose post-process when: deferred rendering (MSAA incompatible), memory limited (mobile, Switch), or prefer temporal stability (TAA for motion).

**Platform-Specific:**
- **PC**: 4x-8x MSAA viable (6-8GB VRAM typical, sufficient memory). Bandwidth high (100+ GB/s, handles MSAA traffic). MSAA preferred for forward rendering (high quality, no blur). Deferred rendering: use TAA instead (MSAA incompatible, G-buffer too large with MSAA). 8x MSAA: high-end GPUs only (RTX 3070+, diminishing returns over 4x).
- **Consoles**: 2x-4x MSAA typical (8-12GB memory, MSAA competes with textures/meshes). Bandwidth moderate (200-300 GB/s, sufficient for 4x at 1080p-1440p). 4x MSAA common (acceptable quality/cost balance). Deferred: use TAA (MSAA memory prohibitive). Some games: dynamic MSAA (2x in heavy scenes, 4x in light scenes, maintain frame rate).
- **Switch**: 2x MSAA maximum (memory limited, 3GB usable). Bandwidth low (25 GB/s, MSAA expensive despite tile-based GPU). Often: no MSAA (use FXAA instead, cheaper). If MSAA: 2x only (4x too much memory/bandwidth). Tile-based helps (resolve in cache), but memory constraint dominant.
- **Mobile**: 2x-4x MSAA on high-end (tile-based GPUs, efficient MSAA implementation). Memory limited (2-4GB total, MSAA 4x doubles framebuffer). Bandwidth low (10-20 GB/s, but tile-based mitigates). Modern devices: 4x MSAA viable (efficient on-chip resolve). Low-end: no MSAA (use FXAA, zero memory overhead). iOS Metal, Android Vulkan: optimized MSAA paths (efficient tile-based resolve).

## Common Pitfalls

**MSAA with Deferred Rendering**: Developer enables 4x MSAA in deferred rendering project (HDRP/URP deferred). G-buffer memory explodes (4 RTs × 4x MSAA = 16x memory vs non-MSAA). Frame rate tanks, memory warnings. Symptom: Extreme memory usage (Profiler shows gigabytes of RTs), very low frame rate (<20fps), or crashes (out of VRAM). Solution: Disable MSAA (QualitySettings.antiAliasing = 0), use post-process AA (TAA in HDRP/URP, high quality alternative). Deferred + MSAA = incompatible (memory prohibitive).

**Not Resolving MSAA RT**: Developer creates MSAA RenderTexture, renders to it, tries to sample in shader without resolve (assumes resolved automatically in all cases). Shader samples multi-sampled data incorrectly (platform-dependent behavior, may sample first sample only, or error). Symptom: Rendering looks wrong (only partial MSAA benefit, incorrect colors), platform-specific issues (works on PC, broken on console). Solution: Ensure resolve happens (Unity usually automatic, but verify), or manually resolve (Graphics.ResolveAntiAliasedSurface) before sampling.

**MSAA on Transparent Objects**: Developer enables MSAA, expects transparency aliasing to disappear (alpha-blended particles, glass). Transparency still aliased (MSAA only fixes geometry edges, not shader aliasing). Symptom: Edges smooth (MSAA works), but transparent objects still jaggy (shader aliasing unchanged). Solution: MSAA doesn't fix transparency (shader-based aliasing, not edge aliasing). Use: alpha-to-coverage (A2C for alpha cutout, helps foliage), or post-process AA (TAA smooths shader aliasing), or super-sampling (render at higher resolution, expensive).

**Excessive MSAA Sample Count**: Developer sets 8x MSAA (thinking more = better quality). Memory usage doubles vs 4x MSAA (8x samples instead of 4x), frame rate drops significantly (bandwidth bottleneck). Visual quality barely improves (diminishing returns beyond 4x, <10% improvement). Symptom: High memory usage (Profiler shows large MSAA RTs), low frame rate (GPU bandwidth maxed), negligible quality gain vs 4x. Solution: Use 4x MSAA (best balance, 8x not worth cost). Compare 4x vs 8x side-by-side (usually imperceptible difference, 2x cost). Save 8x for: photo mode (high-quality screenshots, not real-time gameplay).

## Tools & Workflow

**Quality Settings**: Edit > Project Settings > Quality > Anti Aliasing (dropdown: Disabled, 2x, 4x, 8x). Sets QualitySettings.antiAliasing (global MSAA level for screen rendering). Per-quality tier (Low = Disabled, Medium = 2x, High = 4x, Ultra = 8x). Runtime: QualitySettings.antiAliasing = 4 (change dynamically, e.g., graphics options menu).

**RenderTexture MSAA**: Create MSAA RT (new RenderTexture { antiAliasing = 4 }), or GetTemporary (RenderTexture.GetTemporary(w, h, depth, format, RenderTextureReadWrite.Default, antiAliasing: 4)). Inspector shows MSAA setting (RenderTexture asset > Anti-aliasing dropdown). Verify: Profiler > Memory > RenderTextures (memory = width × height × format × antiAliasing).

**Manual Resolve**: Graphics.ResolveAntiAliasedSurface(sourceRT, destRT) copies and resolves MSAA RT to non-MSAA RT. Or: CommandBuffer.ResolveAntiAliasedSurface(src, dst). Use for: explicit control (resolve at specific point), custom destination (different format/resolution). Example: `Graphics.ResolveAntiAliasedSurface(msaaRT, resolvedRT);` (resolves msaaRT to resolvedRT, both must match resolution).

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows MSAA state per pass (which RTs use MSAA, sample count). Preview: multi-sampled RTs show resolved image (automatic resolve for preview). Verify: MSAA enabled (check RT properties in frame debugger), resolve operations visible (explicit resolve passes listed).

**Unity Profiler**: Profiler > GPU > MSAA resolve passes (shows resolve timing, typically <1ms). Memory > RenderTextures (MSAA RT memory = NxN samples, e.g., 4x MSAA = 4x memory). Monitor: total MSAA memory (should fit in VRAM budget), resolve cost (should be <5% of frame time).

**Graphics Debuggers**: RenderDoc, PIX, Nsight. Show MSAA details (sample count per RT, coverage masks, per-sample values). Visualize: multi-sampled data (inspect individual samples), resolve operation (how samples averaged). Debug: MSAA not working (check if multi-sampled, verify sample count), quality issues (inspect sample distribution).

## Related Topics

- [15.1 RenderTexture Management](15-01-RenderTexture-Management.md) - MSAA RenderTextures
- [15.3 Depth and Stencil Buffers](15-03-Depth-And-Stencil-Buffers.md) - Depth MSAA
- [13.3 Post-Processing](13-03-Post-Processing.md) - Post-process AA alternatives
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - MSAA performance cost

---

[← Previous: 15.3 Depth and Stencil Buffers](15-03-Depth-And-Stencil-Buffers.md) | [Next: Chapter 16 →](../chapters/16-Camera-Systems.md)
