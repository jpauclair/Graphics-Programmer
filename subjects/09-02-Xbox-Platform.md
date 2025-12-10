# 9.2 Xbox Platform Asset Format Guidelines

[← Back to Chapter 9](../chapters/09-Asset-Format-Guidelines.md) | [Main Index](../README.md)

Xbox platform guidelines optimize for Series X/S hardware variance, leveraging BCn textures, DirectX 12, and XDK/GDK tools for console-specific performance.

---

## Overview

Xbox Series X and Series S represent 3x performance variance: Series X (12 TFLOPS, 52 CUs, 16GB GDDR6, 560 GB/s bandwidth) vs Series S (4 TFLOPS, 20 CUs, 10GB GDDR6, 224 GB/s for fast pool). This creates optimization challenge: assets must scale between 1440p Series S and 4K Series X while maintaining 60fps. Series S is the critical target—lowest common denominator for current-gen Xbox development.

Xbox uses DirectX 12 with custom optimizations (hardware-accelerated BC7 decompression, sampler feedback streaming, DirectStorage). Asset formats leverage PC pipeline (BCn textures, HLSL shaders) but require console-specific tuning: Series S needs aggressive optimization (1440p render targets, 1K-1.5K textures, reduced shadow resolution), Series X allows higher fidelity (4K render targets, 2K textures, higher shadow resolution). Certification requires both consoles hit performance targets (stable 60fps minimum).

Development workflow: develop on PC, test early/often on dev kits (Series S mandatory for optimization validation), use PIX for Xbox to profile (GPU, CPU, memory, bandwidth), optimize for Series S first (if Series S hits 60fps, Series X will exceed it), and validate certification requirements (memory limits, frame pacing). Xbox backwards compatibility with Xbox One means legacy considerations for cross-gen titles.

## Key Concepts

- **Series X vs Series S**: Series X = flagship (4K 60fps target, 16GB total memory, 13.5GB available for games). Series S = budget (1440p 60fps target, 10GB total memory, 7.5GB available for games). 2x memory difference drives asset strategy.
- **BCn Texture Formats**: Xbox uses BCn compression (BC7 color, BC5 normals, BC6H HDR, BC4 masks). Hardware-accelerated decompression (free performance). Same as PC formats—share asset pipeline.
- **DirectX 12 Ultimate**: Xbox Series X/S support DXR (ray tracing), VRS (variable rate shading), mesh shaders, sampler feedback. Series S supports all features but lower performance (prioritize rasterization over ray tracing).
- **Sampler Feedback Streaming**: Hardware feature for efficient texture streaming. GPU reports which mips accessed, stream only needed data. Reduces VRAM by 30-50% for open-world games. Requires DX12 API integration.
- **Smart Delivery**: Xbox platform feature. Ships separate asset bundles per console (Series X = high quality, Series S = optimized). Users download appropriate version automatically. Reduces install size for Series S users.

## Best Practices

**Texture Guidelines:**
- Use BC7 for color textures: Same as PC. Hardware decompression on Xbox GPU. 8:1 compression, excellent quality. Essential for meeting memory budgets.
- Use BC5 for normal maps: 2-channel compression. Better quality than BC7 for normals. Reconstruct Z in shader (standard technique).
- Series X: 2K textures standard, 4K for hero assets. VRAM budget ~10GB for textures. Series S: 1K-1.5K textures, 2K for hero assets only. VRAM budget ~5-6GB for textures.
- Implement Sampler Feedback Streaming: Reduces texture memory 30-50%. Essential for open-world games on Series S. Use DirectX 12 Sampler Feedback APIs. Unity support via custom packages.
- Enable mipmaps: Critical for cache efficiency. Series S benefits greatly (limited bandwidth 224 GB/s fast pool). Anisotropic filtering 8-16x supported.

**Mesh Guidelines:**
- Series X handles complex meshes: Characters 15-25K triangles, environment 50-80K. RDNA 2 architecture excellent at geometry processing.
- Series S requires optimization: Characters 10-15K triangles, environment 30-50K. Fewer CUs (20 vs 52) = lower geometry throughput. Implement LOD aggressively.
- Enable mesh compression: 50% size reduction. Beneficial on both consoles (saves memory and bandwidth).
- Use mesh shaders on Series X: Advanced DX12 feature. Better performance than traditional vertex shaders for complex scenarios (procedural geometry, LOD, culling). Series S supports but slower (prioritize for Series X-exclusive features).

**Shader Guidelines:**
- Target Shader Model 6.x: DX12 shader model. Supports advanced features (wave intrinsics, dynamic resources, ray tracing). Compile with DXC compiler.
- Optimize for RDNA 2 architecture: Wave64 execution (64 threads per wave vs 32 on NVIDIA). Reduce register pressure (<32 VGPRs for max occupancy). Use PIX shader profiler.
- Series S shader complexity: Target <120 instructions per fragment shader for 1440p 60fps. Series X: <180 instructions for 4K 60fps.
- Ray tracing shaders (DXR): Series X handles light RT (reflections, shadows). Series S supports but too slow for 60fps RT (disable RT features on Series S, use screen-space alternatives).

**Memory Management:**
- Series X: 16GB total, ~13.5GB available for games. Target <10GB VRAM, <3GB system RAM. Leaves headroom for OS/system reserves.
- Series S: 10GB total (7.5GB available). Memory split: 8GB fast pool (560 GB/s), 2GB standard (336 GB/s). Target <5-6GB VRAM fast pool, <1.5GB standard pool.
- Use PIX memory view: Shows memory allocation per pool (optimal vs standard). Ensure textures/render targets in fast pool, CPU data in standard pool.
- Implement texture streaming: Essential for Series S. Mipmap streaming or Sampler Feedback Streaming. Keeps baseline memory under budget while loading high-res mips on demand.

**Render Target Guidelines:**
- Series X: 4K render targets (3840x2160). Native 4K or reconstructed (temporal upscaling). G-buffer with 4-5 MRTs acceptable.
- Series S: 1440p render targets (2560x1440) standard. Upscale to 4K output via hardware scaler or TAA. Native 1440p = 44% pixels vs 4K (2.25x faster rendering).
- Use dynamic resolution: Adjust render resolution dynamically to maintain 60fps. Series X: 1800p-2160p. Series S: 1080p-1440p. Keeps frame rate stable.
- Lower shadow resolution on Series S: 1K shadows vs 2K on Series X. Users won't notice difference (playing on smaller screens typically).

**Platform-Specific Optimizations:**
- **Xbox Series X**: Target 4K 60fps or 1440p 120fps. 2K textures, high mesh detail, 2K shadows, raytracing optional (reflections, ambient occlusion). VRAM budget 10-12GB.
- **Xbox Series S**: Target 1440p 60fps or 1080p 120fps. 1K-1.5K textures, medium mesh detail, 1K shadows, no raytracing (use screen-space alternatives). VRAM budget 5-6GB.
- **Cross-gen (Xbox One compatibility)**: Support Xbox One/One X requires separate asset bundles. One X = 6 TFLOPS, 12GB memory. One = 1.3 TFLOPS, 8GB memory. Significantly lower spec—consider Series S/X only for new titles.

## Common Pitfalls

**Series S Optimization Ignored**: Developer optimizes for Series X (16GB memory, 4K assets), assumes Series S will scale automatically. Series S crashes (exceeds 7.5GB memory) or runs <40fps. Certification fails. Symptom: Series S performance unacceptable. Solution: Optimize for Series S first (lowest common denominator). Implement Smart Delivery with Series S-specific asset bundles (1K textures, lower mesh detail).

**Memory Pool Confusion**: Developer allocates textures to standard pool (336 GB/s) instead of fast pool (560 GB/s). Bandwidth bottleneck—textures fetch at 60% speed. Series S frame rate drops 30-40%. Symptom: PIX shows textures in standard pool, high memory latency. Solution: Use PIX memory view, ensure render targets and frequently-accessed textures in fast pool. Unity handles automatically but validate.

**No Dynamic Resolution**: Game targets native 4K on Series X, native 1440p on Series S. Heavy scenes drop to 45fps on both consoles (GPU-bound). Certification requires stable 60fps. Symptom: Frame rate unstable in complex scenes. Solution: Implement dynamic resolution scaling (1800p-2160p on Series X, 1080p-1440p on Series S). Maintains 60fps target by adjusting resolution dynamically.

## Tools & Workflow

**PIX for Xbox**: Primary profiling tool. Capture GPU trace, analyze CPU/GPU timings, inspect memory allocation (fast vs standard pools), shader profiling. Download from Microsoft Game Development Kit (GDK).

**Xbox Development Kit (GDK)**: Required for Xbox Series X/S development. Includes PIX, compilers (DXC for shaders), certification tools, debug builds. Requires Xbox partner access.

**Smart Delivery**: Xbox platform feature. Ship separate app packages per console. Series X = high-quality assets, Series S = optimized assets. Users download appropriate version. Configure via Microsoft Partner Center.

**Unity Xbox Platform Settings**: Edit > Project Settings > Player > Xbox Series X|S. Configure texture quality, mesh LOD distances, shader variants per console. Build separate builds for Series X and Series S.

**DirectX Shader Compiler (DXC)**: Modern HLSL compiler for Shader Model 6+. Replaces FXC. Produces better optimized bytecode, supports new features (wave intrinsics, mesh shaders, ray tracing). Unity 2021+ uses DXC.

**Certification Tools**: Microsoft provides automated tests for Xbox certification (performance, memory, compatibility). Run before submission. Identifies issues (frame rate drops, crashes, excessive memory usage).

**Series S Dev Kit**: Mandatory for optimization. Series S is bottleneck console—test extensively. Cannot rely on Series X performance scaling. Dev kits available through ID@Xbox or Xbox partner program.

## Related Topics

- [3.1 DirectX](03-01-DirectX.md) - DirectX 12 API details
- [8.1 Texture Compression Formats](08-01-Texture-Compression-Formats.md) - BC7/BC5/BC6H compression
- [4.2 Console Profiling Tools](04-02-Console-Profiling-Tools.md) - PIX for Xbox
- [2.4 Console-Specific Architecture](02-04-Console-Specific-Architecture.md) - Xbox Series X/S GPU architecture

---

[← Previous: 9.1 PC Platform](09-01-PC-Platform.md) | [Next: 9.3 PlayStation Platform →](09-03-PlayStation-Platform.md)
