# 5.4 Bandwidth Optimization

[← Back to Chapter 5](../chapters/05-Performance-Optimization.md) | [Main Index](../README.md)

Bandwidth optimization reduces memory traffic between GPU and VRAM, critical for performance on bandwidth-limited platforms like Switch (25.6 GB/s) and last-gen consoles.

---

## Overview

Memory bandwidth is often the primary bottleneck on consoles and mobile. Switch's 25.6 GB/s bandwidth is 17x lower than PS5 (448 GB/s). Even high-end GPUs saturate bandwidth with 4K rendering, heavy texturing, or excessive overdraw. Reducing bandwidth consumption improves frame rates proportionally on bandwidth-bound scenarios—cutting bandwidth 50% can improve performance 40-50%.

Bandwidth optimization targets reads (texture sampling, buffer fetches) and writes (render target updates, framebuffer blending). Texture compression reduces reads by 4-6x. Vertex compression reduces vertex fetch bandwidth by 40-50%. Lower render target resolutions reduce write bandwidth by 2-4x (1080p → 4K is 4x bandwidth). Z-buffer compression (automatic on modern GPUs) reduces depth bandwidth by 50-80% with minimal effort.

Monitoring bandwidth is critical: profilers (Razor, PIX, Nsight) show bandwidth utilization percentage. Sustained >80% usage indicates bandwidth bottleneck. Solutions include compression, resolution scaling, or algorithmic changes (deferred → forward+ to reduce G-buffer bandwidth, screen-space effects → compute shaders to optimize access patterns).

## Key Concepts

- **Memory Bandwidth**: Data transfer rate between GPU and VRAM (GB/s). Limited by hardware: Switch 25.6 GB/s, PS4 176 GB/s, PS5 448 GB/s, Xbox Series X 560 GB/s. Exceeding capacity stalls rendering.
- **Texture Compression**: Storing textures in compressed formats (BC7, ASTC) in VRAM. GPU decompresses on-the-fly during sampling. Reduces bandwidth by 4-6x vs uncompressed RGBA.
- **Cache Efficiency**: Percentage of memory requests served by cache vs VRAM. High cache hit rate (>90%) minimizes bandwidth. Mipmapping improves texture cache hits by 80-90%.
- **Render Target Bandwidth**: Memory traffic writing pixels to framebuffers. 4K RGBA framebuffer writes 33MB per frame (at 60fps = 2 GB/s). Multiply by MRTs (4-8) and bandwidth explodes.
- **Z-Buffer Compression**: Hardware-accelerated depth buffer compression. Reduces depth bandwidth by 50-80% automatically. Fast-clear values (0.0 or 1.0) enable better compression.

## Best Practices

**Texture Compression:**
- Compress all textures via platform-appropriate formats: BC7/BC5 (PC/Xbox), ASTC (mobile/Switch), BCn (legacy). Import settings > Texture Compression > Enable.
- ASTC offers best quality/compression ratio: ASTC 6x6 (5.3:1) for high quality, ASTC 8x8 (8:1) for moderate quality. Switch benefits hugely—use 8x8 for most textures.
- BC7 (RGBA) and BC5 (normals) on PC/Xbox compress to 8:1 and 4:1. BC6H for HDR textures (6:1 compression). No runtime cost—GPU decompresses transparently.
- Never use uncompressed textures (RGBA32) except for UI requiring pixel-perfect accuracy. Even UI typically uses BC7 without visible artifacts.

**Vertex Data Compression:**
- Enable mesh compression in import settings: Mesh Compression = High. Quantizes positions, normals, UVs to 16-bit. Reduces vertex size by 40-60%.
- Use 16-bit vertex formats: R16G16_SNorm for normals, R16G16_UNorm for UVs. Pack attributes into fewer channels (normal.w = metallic).
- Align vertex stride to 16 bytes (cache line). 48-byte stride (pos 12B + normal 12B + UV 8B + tangent 16B) aligns perfectly. Misalignment wastes bandwidth.

**Render Target Resolution Scaling:**
- Use dynamic resolution scaling: render at 1440p-1800p, upscale to 4K via TAA or FSR. Saves 30-50% bandwidth vs native 4K.
- Reduce render target resolution for bandwidth-heavy effects: reflections (half-res), AO (quarter-res), fog (half-res). Upscale with bilateral filter.
- Lower bit-depth when possible: R11G11B10_Float for HDR (12 bytes vs 16 for RGBA16_Float), R16G16_SNorm for normals (4 bytes vs 8 for RGBA16_SNorm). Saves 25-50% bandwidth.
- Use single-channel or dual-channel targets instead of RGBA when possible. Depth-only prepass uses R32_Float (4 bytes) instead of RGBA32 (16 bytes)—4x bandwidth savings.

**Mipmapping:**
- Enable mipmaps for all textures except UI and specific effects (scrolling textures, animated UVs). Texture Import Settings > Generate Mip Maps = Enabled.
- Mipmaps improve cache efficiency by 80-90%. Sampling 4K texture for distant object (10 pixels on screen) without mips thrashes cache. With mips, GPU samples 16x16 mip level—perfect cache fit.
- Use anisotropic filtering (8x-16x on PC/current-gen consoles, 2x-4x on Switch). Improves mip selection for oblique angles, reducing aliasing without bandwidth cost (mips are cached).
- Avoid negative LOD bias (`sampler.LODBias = -1`). Forces higher-res mips, increasing bandwidth by 2-4x. Use only for specific visual effects, not globally.

**Overdraw Reduction:**
- Sort opaque objects front-to-back. Enables early-Z rejection, preventing fragment shader execution and framebuffer reads/writes for occluded pixels.
- Use depth prepass for complex scenes with expensive shaders. Costs extra draw calls but eliminates 70-90% of redundant shading in dense environments.
- Minimize transparent overdraw. Particles and UI cause 5-20x overdraw in screen center. Reduce particle counts, simplify particle shaders, or use additive blending (no framebuffer read).
- Profile overdraw with RenderDoc or PIX "Overdraw" view. Target <2x average for opaque, <5x for full scene with transparency.

**Platform-Specific:**
- **Switch**: Bandwidth is critical (25.6 GB/s). Use ASTC 8x8 compression, 720p render targets, 16-bit precision. Minimize overdraw (<1.5x ideal). Switch performance scales almost linearly with bandwidth savings.
- **Xbox Series S**: 224 GB/s bandwidth (vs Series X's 560 GB/s for fast pool). Target 1440p, use aggressive compression, limit MRT count to 4 max. S is more bandwidth-constrained than Series X.
- **PlayStation 4**: 176 GB/s bandwidth. Use checkerboard rendering for 4K (half bandwidth vs native). Leverage depth prepass and aggressive LOD.
- **Mobile**: 10-50 GB/s bandwidth on phones. ASTC compression mandatory, half/quarter-res effects, tile-based rendering optimizations (minimize framebuffer loads/stores).

## Common Pitfalls

**Uncompressed Textures**: Importing textures without compression (RGBA32 or RGBA16) consumes 4-8x bandwidth vs compressed equivalents. A 2K RGBA32 texture (16MB) becomes 2MB BC7—8x savings. On Switch, 10 uncompressed 2K textures = 160MB per frame at 60fps = 9.6 GB/s (38% of total bandwidth). Solution: Compress all textures via import settings (Platform > Texture Compression).

**High-Resolution Render Targets**: Rendering G-buffer at native 4K (8.3M pixels) with 4 MRTs (albedo, normal, depth, properties) writes 531MB per frame. At 60fps = 31.9 GB/sec bandwidth. Xbox One (68 GB/s total) cannot sustain this. Solution: Use 1440p or 1800p render targets, upscale with TAA. Saves 40-60% bandwidth.

**Disabled Mipmaps**: Disabling mipmaps to "save memory" increases bandwidth consumption by 4-10x. Without mips, distant textures sample high-res mips, causing cache thrashing. A 4K texture sampled at 10 pixels on screen should use 32x32 mip (1KB), not 4K base (64MB). Solution: Always enable mipmaps except UI or specific technical reasons.

## Tools & Workflow

**RenderDoc Overdraw View**: Enable "Overlay" > "Quad Overdraw (Pass/Fragment)". Color-codes pixels by overdraw count: green (1x), yellow (2-5x), red (>5x). Identifies problematic areas for optimization.

**PIX Bandwidth Analysis**: "Memory" view shows bandwidth consumption per draw. "Texture" shows texture fetch bandwidth. "Render Target" shows framebuffer write bandwidth. Filter by bandwidth cost, optimize heaviest draws.

**NVIDIA Nsight Graphics**: "Memory Workload Analysis" shows detailed bandwidth breakdown. L2 cache hit rate, VRAM bandwidth utilization, and texture cache efficiency. Target >90% cache hit rate.

**AMD RGP (Radeon GPU Profiler)**: "Memory Chart" displays bandwidth over frame execution. Spikes indicate heavy draws. "Wavefront Occupancy" + "Memory Latency" reveals memory-bound shaders (high latency, low occupancy).

**Unity Profiler > Rendering Module**: "Batches" and "Triangles" give proxy for bandwidth (more triangles = more vertex fetch). "SetPass Calls" indicates state changes potentially flushing caches.

**Platform Profilers**: PlayStation Razor shows precise bandwidth usage vs theoretical max (448 GB/s PS5, 176 GB/s PS4). Xbox PIX exposes memory pool bandwidth separately (optimal vs standard). Essential for console bandwidth optimization.

## Related Topics

- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Reducing memory consumption
- [13.1 Texture Formats and Compression](08-01-Texture-Compression-Formats.md) - Texture compression details
- [2.1 GPU Memory Hierarchy](02-01-GPU-Memory-Hierarchy.md) - Understanding memory systems
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Identifying bandwidth bottlenecks

---

[← Previous: 5.3 Memory Optimization](05-03-Memory-Optimization.md) | [Next: 5.5 Draw Call Pipeline Optimization →](05-05-Draw-Call-Pipeline-Optimization.md)
