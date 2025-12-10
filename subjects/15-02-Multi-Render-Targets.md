# 15.2 Multi-Render Targets (MRT)

[← Back to Chapter 15](../chapters/15-Render-Targets-And-Framebuffers.md) | [Main Index](../README.md)

Multiple Render Targets (MRT) enable shaders to output to multiple textures simultaneously in a single pass, essential for deferred rendering and advanced effects.

---

## Overview

MRT allows fragment shader to write to multiple RenderTextures in single draw call: one geometry pass outputs to 4-8 RTs simultaneously (G-buffer for deferred rendering), instead of rendering geometry 4-8 times (one per RT, expensive). Fragment shader declares multiple output targets (SV_Target0, SV_Target1, etc.), GPU writes to corresponding RTs (bound as MRT array). Performance: single geometry pass (one vertex processing, one rasterization), multiple outputs (write albedo, normal, roughness, emission to separate RTs simultaneously). Massive savings vs multi-pass (4 RTs via MRT = 1 pass, 4 RTs via separate passes = 4x geometry processing).

Deferred rendering primary use case: geometry pass renders G-buffer (4-6 RTs: albedo+AO, normal+smoothness, emission, depth), lighting pass samples G-buffer (full-screen quad, reconstructs lighting from textures). MRT critical: single geometry pass fills all G-buffer RTs (efficient, one draw call per object), without MRT = render geometry 4+ times (one per RT, prohibitively expensive). Unity HDRP/URP deferred use MRT (built-in MRT support for G-buffer output).

Platform support: PC (DirectX 11/12, Vulkan, OpenGL 4.0+, supports 4-8 MRTs), consoles (PlayStation, Xbox support 8 MRTs), Switch (4 MRTs typical), mobile (limited support, 4 MRTs on modern GPUs, older devices may not support). Bandwidth consideration: MRT writes to multiple RTs = multiple bandwidth (4 RTs × 8 bytes per pixel = 32 bytes/pixel, high bandwidth for 4K resolution = 250MB per frame at 60fps). Format/resolution optimization critical.

## Key Concepts

- **MRT Declaration**: Shader declares multiple outputs (struct with SV_Target0, SV_Target1, etc.). Example: struct MRTOutput { float4 rt0 : SV_Target0; float4 rt1 : SV_Target1; float4 rt2 : SV_Target2; }. Fragment shader returns MRTOutput (fills all RTs simultaneously). Unity binds RT array (SetRenderTarget with RT array, shader outputs to corresponding RTs).
- **G-Buffer Layout**: Deferred rendering typical G-buffer (4-6 RTs): RT0 = Albedo + AO (RGB albedo, A ambient occlusion), RT1 = Normal + Smoothness (RGB world-space normal, A smoothness), RT2 = Emission + unused (RGB emission, A free), RT3 = Depth + Stencil (depth buffer, stencil for masking). Compact encoding (pack multiple values per RT, minimize RT count).
- **Bandwidth Cost**: MRT writes to all RTs every pixel (bandwidth = sum of RT sizes). Example: 4 RTs × 8 bytes (RGBA16 half-float) × 1920×1080 pixels = 63MB per frame. At 60fps = 3.8GB/s bandwidth. High-resolution or many RTs = bandwidth bottleneck (GPU memory limited, reduces performance). Optimization: reduce RT count (pack data), use smaller formats (RG16 instead of RGBA32), lower resolution (render at 1080p, upscale to 4K).
- **MRT Limitations**: All RTs same resolution (can't mix 1080p and 540p in single MRT), same sample count (no mixing MSAA and non-MSAA), compatible formats (platform-specific restrictions, e.g., all float or all int). Typically: all RTs same dimensions, similar formats (RGBA8, RGBA16, or RG16), single MSAA setting (4x MSAA = all RTs 4x MSAA).
- **RenderBuffer vs RenderTexture**: Unity MRT uses RenderBuffer array (lightweight RT handles). SetRenderTarget(colorBuffers[], depthBuffer) binds MRT. RenderBuffer obtained from RenderTexture.colorBuffer (color surface), RenderTexture.depthBuffer (depth surface). Allows: mix RTs (composite from different RenderTexture sources), separate depth buffer (shared depth across MRTs).

## Best Practices

**MRT Setup in Unity:**
- Create RenderTexture array: RenderTexture[] rts = { rt0, rt1, rt2, rt3 } (array of RenderTextures for MRT). All RTs same resolution (e.g., 1920x1080), compatible formats (e.g., all ARGBHalf or ARGB32). Depth buffer: create separate depth RT (format = Depth, e.g., RFloat or D24S8), or use depth from one RT (rt0.depthBuffer).
- Set MRT: CommandBuffer.SetRenderTarget(colorBuffers, depthBuffer). colorBuffers = array of RenderBuffer (rt.colorBuffer for each RT), depthBuffer = depth RenderBuffer (depthRT.depthBuffer). Unity binds all RTs (shader outputs to SV_Target0, SV_Target1, etc., write to corresponding RTs).
- Shader output: Fragment shader returns struct with multiple SV_Targets. Example: `struct MRTOutput { float4 albedo : SV_Target0; float4 normal : SV_Target1; }; MRTOutput frag() { MRTOutput o; o.albedo = ...; o.normal = ...; return o; }`. Unity writes albedo to RT0, normal to RT1.
- Clear MRT: CommandBuffer.ClearRenderTarget (clears depth, color, or both) for MRT. Flags: clearDepth (clear depth buffer), clearColor (clear all color RTs), backgroundColor (clear color value). Ensure proper clear (don't accumulate garbage from previous frames).

**G-Buffer Optimization:**
- Minimize RT count: Fewer RTs = less bandwidth + memory. Typical: 3-4 RTs sufficient (Albedo+AO, Normal+Smoothness, Emission+SpecularColor, Depth). Advanced: 5-6 RTs for complex materials (additional RT for subsurface scattering, translucency, etc.). Avoid: 8+ RTs (excessive bandwidth, diminishing returns).
- Compact encoding: Pack multiple values per channel. Examples: Normal XY in RG (reconstruct Z = sqrt(1 - X² - Y²)), Metallic+Smoothness in single channel (e.g., Metallic in R, Smoothness in A of Albedo RT), Occlusion in alpha (AO in alpha channel of Albedo RT). Saves RTs (fewer textures, reduced bandwidth).
- Format selection: Use smallest format that preserves quality. Albedo = ARGB32 (8-bit sufficient for color), Normal = RG16 (16-bit XY normals, 8-bit too low precision), Emission = ARGBHalf (HDR emission, 16-bit float needed for >1 values). Avoid: ARGBFloat (32-bit float, excessive for most data, 4x memory vs ARGB32). Mobile: aggressive formats (RGB565 for albedo, R8 for single channels, minimize bandwidth).
- Resolution scaling: Render G-buffer at lower resolution for performance (e.g., 1080p G-buffer, upscale final image to 4K via TAA or DLSS). Reduces bandwidth 4x (1080p = 1/4 pixels of 4K). Quality impact minimal (deferred lighting low-frequency, upscaling effective). HDRP Dynamic Resolution uses this technique.

**Bandwidth Management:**
- Profile bandwidth: GPU Profiler or platform tools (RenderDoc, PIX, Nsight) show bandwidth usage. MRT passes typically bandwidth-bound (memory writes dominate). Symptom: GPU utilization <100%, memory bandwidth maxed out (bottleneck). Solution: reduce RT count, smaller formats, lower resolution.
- Tile-based GPUs: Mobile/Switch GPUs use tile-based rendering (render tiles in on-chip cache, write to memory once at end). MRT efficient on tile-based (write all RTs simultaneously from cache, minimal bandwidth impact vs separate passes). But: RT count limited (4 RTs typical, exceeding = fall back to memory, expensive). Optimize: 4 or fewer RTs for mobile (fits in tile cache).
- MSAA with MRT: MSAA increases memory/bandwidth (4x MSAA = 4x memory for each RT). MRT with 4x MSAA + 4 RTs = 16x memory vs single non-MSAA RT (prohibitive). Avoid MSAA with MRT (use post-process AA like TAA, FXAA instead). If MSAA required: 2x MSAA maximum, or reduce RT count (2 RTs instead of 4).
- Depth prepass: Render depth-only pass before G-buffer (writes only depth, early-Z culling reduces overdraw). Then render G-buffer (depth test only, skips occluded pixels, reduces MRT writes). Saves bandwidth (occluded pixels don't write to G-buffer RTs). Tradeoff: additional pass (depth prepass cost vs bandwidth savings), beneficial for complex scenes (high depth complexity, heavy overdraw).

**Shader Implementation:**
- Define MRT outputs: `struct MRTOutput { float4 rt0 : SV_Target0; float4 rt1 : SV_Target1; /* ... */ };`. Use SV_Target0 through SV_Target7 (max 8 MRTs on modern GPUs). Fragment shader returns MRTOutput (all fields filled).
- Conditional output: Some platforms support writing to subset of RTs (e.g., shader writes RT0 and RT2, skips RT1). Not universally supported (safer to write all RTs, use dummy data if unused). Unity: output to all bound RTs (avoids platform issues).
- Data packing: Encode efficiently (normal compression, HDR packing). Example: `output.normal = float4(worldNormal.xy * 0.5 + 0.5, 0, smoothness);` (pack XY normal + smoothness in alpha). Lighting pass decodes: `float3 normal = float3(gbuffer.normal.xy * 2 - 1, 0); normal.z = sqrt(1 - dot(normal.xy, normal.xy));` (reconstruct Z).
- Platform checks: Use `#if UNITY_FRAMEBUFFER_FETCH` or similar for platform-specific MRT handling. Mobile platforms may have restrictions (check SystemInfo.supportedRenderTargetCount for max MRTs). Fallback: render multiple passes if MRT unsupported (rare on modern platforms).

**Platform-Specific:**
- **PC**: 8 MRTs typical (DirectX 11/12, Vulkan). High bandwidth (100+ GB/s on modern GPUs, handles 4-6 RTs at 1080p-4K). MSAA viable (2x-4x MSAA with 4 RTs, sufficient bandwidth). Format flexibility (mix ARGB32, ARGBHalf, RG16 as needed).
- **Consoles**: 8 MRTs supported, 4-6 typical usage (balance bandwidth/memory). Moderate bandwidth (200-300 GB/s on PS5/Xbox Series X, sufficient for 1080p-1440p 4-6 RTs). Limited MSAA (2x MSAA max with MRT, prefer TAA). Fixed memory pool (MRTs compete with textures/meshes, optimize RT count/formats).
- **Switch**: 4 MRTs maximum (hardware limit). Low bandwidth (25 GB/s, very limited). 3-4 RTs typical (1080p too expensive, use 720p). No MSAA with MRT (bandwidth prohibitive). Aggressive formats (RGB565, RG16, minimize bandwidth).
- **Mobile**: 4 MRTs on modern GPUs (tile-based, efficient if <4 RTs). Very low bandwidth (10-20 GB/s, extremely limited). 2-3 RTs maximum (540p-720p resolution). No MSAA (bandwidth can't handle). Aggressive compression (RGB565, R8, minimize RT size). Some older devices: no MRT support (check SystemInfo, fallback to forward rendering).

## Common Pitfalls

**Mismatched RT Resolutions**: Developer creates MRT with different resolutions (RT0 = 1920x1080, RT1 = 960x540). SetRenderTarget fails or renders incorrectly (GPU requires all MRTs same resolution). Symptom: Black screen, rendering errors, or exception ("RenderTextures must be same size"). Solution: Ensure all RTs in MRT array same resolution (e.g., all 1920x1080). Validate before SetRenderTarget (assert all rts[i].width == rts[0].width).

**Excessive MRT Bandwidth**: Developer uses 6 RTs at 4K with ARGBFloat (32-bit float per channel). Bandwidth: 6 RTs × 16 bytes × 3840×2160 = 797MB per frame, 47GB/s at 60fps. GPU bandwidth saturated (memory bottleneck, low fps). Symptom: Low GPU utilization (waiting on memory), frame rate drops. Solution: Reduce RT count (4 RTs instead of 6), smaller formats (ARGBHalf instead of Float, 8 bytes vs 16), lower resolution (1080p instead of 4K, 4x less bandwidth). Profile bandwidth (RenderDoc/PIX shows memory traffic).

**Not Clearing MRT**: Developer sets MRT, doesn't clear (assumes default clear). Previous frame's data remains (garbage in RTs, incorrect rendering). Symptom: Visual artifacts (previous frame visible, flickering, incorrect colors). Solution: CommandBuffer.ClearRenderTarget(clearDepth: true, clearColor: true, backgroundColor: Color.black) after SetRenderTarget (before rendering). Always clear RTs (ensures clean slate).

**MSAA with MRT on Mobile**: Developer enables 4x MSAA with 4 MRTs on mobile (thinking improves quality). Memory explodes (4 RTs × 4x MSAA = 16x memory vs single non-MSAA RT), bandwidth saturated. Device crashes (out of memory) or frame rate <10fps. Symptom: Crash on mobile devices, extreme slowdown. Solution: Disable MSAA for MRT on mobile (antiAliasing = 1), use post-process AA (FXAA, TAA, cheap alternatives). Or: reduce RT count (2 RTs + 2x MSAA, less memory than 4 RTs no MSAA).

## Tools & Workflow

**CommandBuffer MRT Setup**: `var colorBuffers = new RenderBuffer[] { rt0.colorBuffer, rt1.colorBuffer, rt2.colorBuffer }; var depthBuffer = rt0.depthBuffer; cmd.SetRenderTarget(colorBuffers, depthBuffer); cmd.ClearRenderTarget(true, true, Color.black);` (sets 3 MRTs with shared depth, clears).

**Shader MRT Declaration**: Define output struct with SV_Target0-7. Example: `struct MRTOut { float4 albedo : SV_Target0; float4 normal : SV_Target1; float4 emission : SV_Target2; }; MRTOut frag() { MRTOut o; o.albedo = ...; o.normal = ...; o.emission = ...; return o; }`.

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows MRT passes (lists bound RTs for each pass), preview RTs (click draw call, see RT contents). Verify: all RTs rendering correctly (no black RTs, correct data per RT), RT resolution matches (consistent dimensions).

**Unity Profiler**: Profiler > GPU > MRT passes (shows rendering time per MRT pass). Memory > RenderTextures (shows MRT memory usage, resolution, format). Identify: bandwidth bottlenecks (MRT pass takes >5ms, likely bandwidth-bound), excessive memory (total MRT memory >500MB).

**RenderDoc/PIX**: External profilers show detailed MRT info (bandwidth per RT, writes per pixel, format info). RenderDoc: Texture Viewer shows all MRT outputs (side-by-side comparison, verify contents). PIX (Xbox/PC): shows memory bandwidth graph (identify MRT bottlenecks).

**SystemInfo Checks**: `SystemInfo.supportedRenderTargetCount` (max MRTs, typically 4-8). `SystemInfo.SupportsRenderTextureFormat(format)` (check format support). Use for: platform-specific fallbacks (if MRT unsupported, use multi-pass).

## Related Topics

- [15.1 RenderTexture Management](15-01-RenderTexture-Management.md) - RT creation and pooling
- [13.5 Advanced Rendering](13-05-Advanced-Rendering.md) - Deferred rendering with MRT
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Bandwidth optimization
- [15.3 Depth and Stencil Buffers](15-03-Depth-And-Stencil-Buffers.md) - Depth buffer with MRT

---

[← Previous: 15.1 RenderTexture Management](15-01-RenderTexture-Management.md) | [Next: 15.3 Depth and Stencil Buffers →](15-03-Depth-And-Stencil-Buffers.md)
