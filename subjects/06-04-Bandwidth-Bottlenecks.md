# 6.4 Bandwidth Bottlenecks

[← Back to Chapter 6](../chapters/06-Bottleneck-Identification.md) | [Main Index](../README.md)

Identifying bandwidth bottlenecks reveals when memory transfer rate between GPU and VRAM limits performance, requiring compression, resolution scaling, or algorithmic changes.

---

## Overview

Bandwidth bottlenecks occur when GPU stalls waiting for data from VRAM. Modern GPUs compute thousands of operations per cycle, but memory bandwidth is limited—Switch (25.6 GB/s), PS4 (176 GB/s), PS5 (448 GB/s), Xbox Series X (560 GB/s). Insufficient bandwidth causes GPU underutilization: compute units idle while waiting for texture data or framebuffer writes. This manifests as low frame rates despite low GPU utilization (40% GPU usage but 30fps indicates bandwidth bottleneck, not compute bottleneck).

Bandwidth bottlenecks have three primary sources: texture fetches (sampling large uncompressed textures), render target writes (4K MRT G-buffer writing gigabytes per frame), and inefficient memory access patterns (poor cache locality, excessive random access). Profiling reveals bandwidth issues: sustained >80% bandwidth utilization, high memory latency, or low cache hit rates (<80%).

Identifying bandwidth bottlenecks requires platform profilers (PIX, Razor, Nsight, RGP) showing memory subsystem metrics. Key indicators: Memory Bandwidth Utilization >80%, L2 Cache Hit Rate <80%, Memory Latency >300 cycles. These metrics indicate GPU is memory-bound, not compute-bound. Validation: Reducing texture resolution or render target resolution improves performance proportionally (50% resolution = 40-50% faster confirms bandwidth bottleneck).

## Key Concepts

- **Memory Bandwidth**: Data transfer rate between GPU and VRAM (GB/s). Hardware-limited: Switch 25.6 GB/s, PS4 176 GB/s, PS5 448 GB/s, Xbox Series X 560 GB/s. Exceeding capacity stalls GPU.
- **Cache Hit Rate**: Percentage of memory requests served by L1/L2 cache vs VRAM. High (>90%) minimizes bandwidth consumption. Low (<80%) indicates poor cache locality or excessive data volume.
- **Texture Fetch Bandwidth**: Memory traffic from sampling textures. Uncompressed 4K texture (64MB) sampled 4 times per pixel at 1080p = 500MB per frame. At 60fps = 30 GB/s (exceeds Switch's total bandwidth).
- **Render Target Bandwidth**: Memory traffic writing pixels to framebuffers. 4K RGBA16 framebuffer (32MB) written once per frame = 1.9 GB/s. Deferred G-buffer with 4 MRTs = 7.6 GB/s.
- **Bandwidth Bottleneck Symptoms**: Low GPU utilization (<50%) despite low frame rate, performance scales linearly with resolution reduction, profiler shows >80% bandwidth usage, high memory latency.

## Best Practices

**Identifying Bandwidth Bottlenecks:**
- Profile with platform tools: PIX (Xbox), Razor (PlayStation), Nsight Graphics (NVIDIA PC), RGP (AMD PC). Look for "Memory Bandwidth" metric >80% sustained.
- Check GPU utilization vs frame time: 40% GPU utilization but 33ms frame time (30fps) indicates bandwidth bottleneck. Compute-bound would show 95-100% GPU utilization.
- Test resolution scaling: Render at 50% resolution (720p instead of 1440p). If frame rate doubles, bandwidth bottleneck confirmed (4x less data = 2x faster due to cache effects).
- Analyze cache hit rates: L2 hit rate <80% indicates poor cache utilization. Investigate texture access patterns (mipmaps disabled? excessive random sampling?).
- Monitor memory latency: >300 cycles average latency indicates memory stalls. Compare against baseline (<200 cycles on modern GPUs with good cache hit rates).

**Texture Bandwidth Reduction:**
- Compress all textures: BC7 (PC/Xbox), ASTC (mobile/Switch), BC5 (normals). Reduces bandwidth by 4-6x. GPU decompresses in cache (no additional cost).
- Enable mipmaps: Improves cache hit rate by 80-90%. Distant objects sample small mips (64x64 = 16KB) instead of base 4K texture (64MB). Massive bandwidth savings.
- Reduce texture resolution per platform: 2K on current-gen, 1K on Switch/last-gen. 50% resolution = 75% bandwidth reduction (smaller textures + better cache fit).
- Use anisotropic filtering efficiently: 16x AF on PC/current-gen, 4x on Switch/last-gen. Higher AF improves quality without bandwidth cost (better mip selection).
- Avoid negative LOD bias: Forces high-res mips, increasing bandwidth 2-4x. Only use for hero assets, not globally.

**Render Target Bandwidth Optimization:**
- Use dynamic resolution scaling: Render at 1440p-1800p, upscale to 4K with TAA/FSR. Reduces bandwidth by 30-50% vs native 4K.
- Lower bit-depth when possible: R11G11B10_Float for HDR (12 bytes) vs RGBA16_Float (16 bytes). 25% bandwidth savings with minimal quality loss.
- Reduce MRT count in deferred rendering: Combine channels (normal.w = metallic, albedo.a = smoothness). 4 MRTs instead of 5 = 20% bandwidth savings.
- Use single-channel targets when possible: Depth prepass uses R32_Float (4 bytes) vs full G-buffer (48-64 bytes). AO uses R8_UNorm (1 byte).
- Enable render target compression: Fast-clear (clear to 0.0 or 1.0 enables DCC/FMask compression on consoles). Automatic 50-70% bandwidth reduction.

**Memory Access Pattern Optimization:**
- Improve texture cache locality: Use mipmaps, enable filtering (bilinear/trilinear). Sequential UV access patterns cache better than random access.
- Minimize random texture reads in shaders: Each `tex2D(randomUV)` likely misses cache. Use texture arrays or atlases for coherent access (adjacent pixels sample adjacent texels).
- Reduce buffer reads in compute shaders: Coalesce reads (thread 0 reads element 0, thread 1 reads element 1). Divergent reads (thread 0 reads element 17, thread 1 reads element 3) miss cache.
- Use shared memory (groupshared) in compute: Load data once to shared memory, entire thread group accesses from fast shared memory instead of slow VRAM.

**Algorithmic Changes:**
- Switch deferred → forward+ for bandwidth-limited platforms: Deferred writes 4-8 MRTs (128-256 bytes/pixel). Forward+ writes 1 framebuffer (8 bytes/pixel). 16-32x bandwidth savings.
- Use compute shaders for post-processing: Better memory access patterns (coalesced reads/writes) vs pixel shaders (incoherent random access). 20-40% faster on bandwidth-bound effects.
- Implement temporal techniques: Render effect at half rate (every other frame), reuse previous result. Saves 50% bandwidth. Works for AO, reflections, shadows.
- Use screen-space approximations: SSAO (screen-space AO) vs raytraced AO (60% bandwidth), SSR (screen-space reflections) vs cubemap reflections (40% bandwidth).

**Platform-Specific:**
- **Switch**: 25.6 GB/s bandwidth (lowest of all platforms). Bandwidth is primary bottleneck. ASTC 8x8 compression mandatory, 720p rendering typical, aggressive LOD.
- **PlayStation 4**: 176 GB/s bandwidth. Sufficient for 1080p deferred rendering. Bandwidth issues emerge at 1440p+ or with many MRTs. Use checkerboard rendering for 4K.
- **Xbox Series S**: 224 GB/s bandwidth for fast memory pool (vs 560 GB/s Series X). More bandwidth-constrained than Series X. Target 1440p, reduce MRTs, compress aggressively.
- **High-end PC**: 500-1,000 GB/s bandwidth (NVIDIA 4090: 1,008 GB/s). Rarely bandwidth-bound at 1440p. Bandwidth issues emerge at 4K with heavy RT or many MRTs.

## Common Pitfalls

**Assuming GPU Utilization Equals Performance**: Developer sees 45% GPU utilization and concludes "GPU is underused, not the bottleneck". Actually bandwidth-limited: GPU is idle waiting for memory. Symptom: Low GPU%, low frame rate, profiler shows >85% bandwidth usage. Solution: Profile memory bandwidth explicitly with platform tools. Low GPU% + high bandwidth% = bandwidth bottleneck.

**Disabled Mipmaps to Save Memory**: Developer disables mipmaps on large textures to save 33% memory. Now distant objects sample full 4K textures (64MB per texture) with terrible cache hit rate. Bandwidth consumption increases 5-10x, destroying performance. Symptom: Frame rate drops in distant views despite fewer objects. Solution: Always enable mipmaps except UI. Memory savings (33%) is vastly outweighed by bandwidth cost (5-10x increase).

**Excessive MRTs in Deferred Rendering**: G-buffer uses 8 MRTs for comprehensive material data: albedo, normal, metallic, smoothness, AO, emission, subsurface, detail maps. 8 MRTs × 16 bytes = 128 bytes per pixel. At 1440p = 354MB per frame. At 60fps = 21.2 GB/s (exceeds PS4's bandwidth budget). Solution: Combine channels (normal.w = metallic, albedo.a = smoothness), reduce to 4-5 MRTs. Saves 40-60% bandwidth.

## Tools & Workflow

**NVIDIA Nsight Graphics** (PC): "Memory Workload Analysis" shows bandwidth utilization, L2 cache hit rate, memory latency. "Texture" tab shows per-texture bandwidth consumption. Identifies hotspots. Download from NVIDIA developer site.

**AMD RGP (Radeon GPU Profiler)** (PC/consoles): "Memory Chart" displays bandwidth usage over frame. "Wavefront Occupancy" view combined with "Memory Latency" reveals memory-bound shaders (high latency, low occupancy). Works on PS4/PS5 via Sony tools.

**PIX for Xbox**: "Memory" section shows bandwidth usage vs capacity (560 GB/s Series X, 224 GB/s Series S fast pool). "Texture" view shows texture fetch bandwidth. "Render Target" shows framebuffer write bandwidth. Essential for Xbox optimization.

**PlayStation Razor**: "Memory Bandwidth" metric in performance overlay. Shows real-time utilization percentage. Sustained >80% indicates bottleneck. Detailed breakdown available in Razor profiler (texture vs framebuffer bandwidth).

**RenderDoc**: "Texture Viewer" > "Resource Usage" shows texture size and estimated bandwidth consumption. Overdraw view indicates framebuffer write bandwidth (high overdraw = high bandwidth).

**Unity Profiler**: Less precise for bandwidth but "GPU" module > "Render Target Switches" and "Texture Bindings" give indirect indicators. Many switches = poor cache locality.

## Related Topics

- [5.4 Bandwidth Optimization](05-04-Bandwidth-Optimization.md) - Bandwidth reduction techniques
- [6.3 Memory Bottlenecks](06-03-Memory-Bottlenecks.md) - Memory capacity issues
- [2.1 GPU Memory Hierarchy](02-01-GPU-Memory-Hierarchy.md) - Understanding memory systems
- [13.1 Texture Formats and Compression](08-01-Texture-Compression-Formats.md) - Texture compression methods

---

[← Previous: 6.3 Memory Bottlenecks](06-03-Memory-Bottlenecks.md) | [Next: Chapter 7 - Culling and LOD →](../chapters/07-Culling-And-LOD.md)
