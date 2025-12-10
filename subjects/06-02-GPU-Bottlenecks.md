# GPU Bottlenecks

[← Back to Chapter 6](../chapters/06-Bottleneck-Identification.md) | [Main Index](../README.md)

GPU bottlenecks occur when the graphics processor limits frame rate despite CPU having spare capacity. Identifying which GPU stage is the bottleneck guides optimization strategy.

---

## Overview

Modern GPUs contain multiple specialized units: geometry processors (vertex shaders), rasterizers, fragment processors (pixel shaders), texture samplers, and memory controllers. Each unit has finite throughput—when one saturates, it becomes the bottleneck while other units idle. A scene with 50 million polygons might be vertex-bound (geometry unit saturated), while a scene with heavy post-processing could be fragment-bound (pixel shader unit saturated).

Distinguishing between vertex, fragment, texture, and memory bottlenecks requires testing: reduce polygon counts, lower resolution, disable textures, or simplify shaders. Whichever change improves performance reveals the bottleneck. Console profilers (PIX, Razor, NNGFX) provide hardware counters showing exact unit utilization, making diagnosis precise rather than guesswork.

GPU architecture varies significantly across platforms. NVIDIA and AMD PC GPUs prioritize raw compute and memory bandwidth. Xbox Series X uses RDNA 2 with dedicated ray tracing units. PlayStation 5's RDNA 2 GPU runs at higher clocks (2.23GHz) but has fewer compute units. Switch's Tegra X1 has low memory bandwidth (25.6 GB/s vs PS5's 448 GB/s), making it extremely memory-bound. Optimization strategies must account for these architectural differences.

## Key Concepts

- **Vertex-Bound**: Geometry processing (vertex shaders, tessellation) limits performance. High polygon counts or complex vertex shaders saturate geometry units.
- **Fragment-Bound**: Pixel shader execution limits performance. High resolution, complex shaders, or excessive overdraw saturates fragment processors.
- **Texture-Bound**: Texture sampling limits performance. Many texture fetches, large textures, or poor cache utilization saturates texture units or cache bandwidth.
- **Memory-Bound**: VRAM bandwidth or capacity limits performance. High-resolution textures, large framebuffers, or inefficient memory access patterns saturate memory controllers.
- **ROP-Bound**: Render output units (blend, depth test, MSAA resolve) limit performance. High-resolution framebuffers with alpha blending or heavy post-processing saturate ROPs.

## Best Practices

**Diagnosing GPU Bottlenecks:**
- Lower resolution from 4K to 1080p. If frame rate improves significantly (>30%), you're fragment-bound or memory-bound. No change indicates vertex-bound or CPU-bound.
- Reduce mesh LODs or disable distant objects. Performance gain suggests vertex-bound bottleneck.
- Replace complex shaders with unlit/simple shaders. Improvement confirms fragment-bound scenario.
- Disable texture sampling in shaders (return constant colors). Performance increase indicates texture-bound or memory-bound.

**Vertex-Bound Optimization:**
- Implement aggressive LOD systems with low-poly distant objects (50-500 triangles at 100m+ range).
- Use GPU occlusion culling to eliminate non-visible geometry before vertex processing. Unity's Hi-Z occlusion culling on consoles saves 30-50% vertex work.
- Simplify vertex shaders—move calculations to pixel shaders if amortized over fewer vertices (distant objects have large screen area but few vertices).
- On consoles, use mesh shaders (Xbox Series X/PS5) for dynamic LOD or procedural geometry, bypassing traditional vertex pipeline.

**Fragment-Bound Optimization:**
- Reduce shader instruction counts below 300 ALU ops for base pass (200 on Switch, 500+ acceptable on PS5/Series X).
- Enable dynamic resolution scaling (URP/HDRP DRS). Render at 1440p-1800p and upscale to 4K during heavy scenes, saving 40-50% fragment work.
- Minimize overdraw via depth prepass or sorting opaque objects front-to-back. PlayStation's depth prepass eliminates 70-90% of redundant fragment shading.
- Use variable rate shading (VRS) on Xbox Series X/S and NVIDIA RTX to render peripheral vision at lower resolution (30-40% fragment savings with minimal quality loss).

**Texture-Bound Optimization:**
- Pack textures into channels (R=metallic, G=occlusion, B=detail, A=smoothness) to reduce samples from 4 to 1.
- Use texture compression (BC7/ASTC) aggressively. Compressed textures reduce bandwidth by 4-6x, critical on Switch and Xbox Series S.
- Enable mipmapping always. Sampling high-resolution textures on distant objects thrashes texture cache. Mipmaps improve cache hit rates by 80-90%.
- Batch texture samples at shader start to hide latency with ALU work. GPUs pipeline texture fetches—interleaving samples and math maximizes throughput.

**Memory-Bound Optimization:**
- Reduce render target resolution or bit depth. Use R16G16B16A16_Float only when necessary—R11G11B10_Float saves 25% bandwidth with negligible quality loss.
- Minimize texture memory via compression and resolution caps (1024x1024 max for distant objects, 2048x2048 for hero assets).
- Enable Z-buffer compression (automatic on consoles). Hi-Z and delta compression reduce depth bandwidth by 50-80%.
- On Switch, use 16-bit color framebuffers (R5G6B5 or R4G4B4A4) for performance mode. 32-bit RGBA costs double bandwidth.

**Platform-Specific:**
- **PC**: Profile on NVIDIA (CUDA cores, high bandwidth) and AMD (compute units, infinity cache). Optimization for one may hurt the other.
- **Xbox Series X/S**: Series S has 10GB VRAM (4GB slower pool) and 20 compute units vs Series X's 52. Scale textures and resolution separately.
- **PlayStation 5**: High GPU clock (2.23GHz) makes it less memory-bound than Xbox, more compute-bound. Shader optimization yields better results.
- **Switch**: Memory bandwidth is critical bottleneck. Texture compression, low resolution (720p docked, 540p handheld), and simple shaders mandatory.

## Common Pitfalls

**Optimizing Wrong Stage**: Scene runs at 30fps with complex shaders, so you optimize pixel shader code aggressively. Frame rate improves 5%. Profiler reveals vertex-bound bottleneck from 10 million polygons—shader work was irrelevant. Always profile to identify bottleneck before optimizing.

**Ignoring Overdraw**: Transparent particle effects and UI render back-to-front, causing 5-10x overdraw in screen center. This looks GPU-bound (low frame rate, high GPU time), but it's specifically fragment-bound from redundant shading. Depth prepass doesn't help transparency—reduce particle counts and use simpler particle shaders instead.

**Texture Thrashing on Consoles**: Streaming 8K textures for open-world terrain saturates memory bandwidth, causing hitches and low frame rates. Consoles have limited VRAM (10-16GB) and slower streaming than PC SSDs. Use texture streaming systems (Unity's Mipmap Streaming) to load only visible high-res mips, keeping memory usage under 6-8GB for textures.

## Tools & Workflow

**PIX for Windows/Xbox**: GPU capture shows "GPU Work Duration" per draw call. High vertex shader time indicates vertex-bound; high pixel shader time suggests fragment-bound. "Memory" view shows bandwidth utilization—sustained >80% indicates memory bottleneck.

**RenderDoc**: Pixel history shows how many times each pixel was shaded (overdraw). Right-click pixel > "Debug Pixel" to see all draws affecting that pixel. High overdraw (4-8x) confirms fragment bottleneck.

**NVIDIA Nsight Graphics**: "GPU Trace" shows unit utilization (SM, Texture, ROP, Memory). If "SM Throughput" is 90%+ and others are low, you're compute/fragment-bound. High "L2 Cache Throughput" suggests memory-bound.

**AMD RGP**: "Wavefront Occupancy" shows fragment shader parallelism. Low occupancy (<50%) indicates register pressure or memory latency limiting fragment throughput. "Event Timing" highlights expensive draws—sort by duration to prioritize optimization.

**PlayStation Razor**: Hardware counters show precise bottleneck. "Shader Core Utilization" at 95%+ confirms fragment-bound. "Memory Bandwidth" nearing theoretical max (448 GB/s) indicates memory bottleneck.

**Resolution Scaling Test**: Create build with quality settings that scale resolution (Low=720p, Medium=1080p, High=1440p, Ultra=4K). If frame rate scales linearly with resolution, you're fragment or memory-bound. Flat scaling indicates vertex or CPU-bound.

## Related Topics

- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Shader and GPU optimization techniques
- [6.1 CPU Bottlenecks](06-01-CPU-Bottlenecks.md) - Distinguishing CPU vs GPU bottlenecks
- [13.1 Texture Formats and Compression](08-01-Texture-Compression-Formats.md) - Reducing texture memory bandwidth
- [17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) - Level of detail for vertex optimization

---

[← Previous: 6.1 CPU Bottlenecks](06-01-CPU-Bottlenecks.md) | [Next: 6.3 Memory Bottlenecks →](06-03-Memory-Bottlenecks.md)
