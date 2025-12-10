# 9.3 PlayStation Platform Asset Format Guidelines

[← Back to Chapter 9](../chapters/09-Asset-Format-Guidelines.md) | [Main Index](../README.md)

PlayStation guidelines leverage PS5's ultra-fast SSD, RDNA 2 GPU, and GNM/GNMX APIs, optimizing for unified memory architecture and Sony-specific tools.

---

## Overview

PlayStation 5 (10.3 TFLOPS, 36 CUs @ 2.23GHz, 16GB GDDR6, 448 GB/s bandwidth, 5.5 GB/s SSD) represents massive leap from PS4 (1.8 TFLOPS, 18 CUs @ 800MHz, 8GB GDDR5, 176 GB/s, HDD). This generational gap creates cross-gen complexity: titles supporting both consoles require separate asset pipelines (PS5 = high fidelity 4K 60fps, PS4 = optimized 1080p 30-60fps). PS5-exclusive titles leverage SSD for aggressive streaming, virtual texturing, and instant loading (eliminates traditional load screens).

PlayStation uses proprietary APIs: GNM (low-level graphics), GNMX (higher-level wrapper), and PSSL (PlayStation Shader Language based on HLSL). Asset formats follow industry standards (BCn textures, standard meshes) but benefit from Sony-specific optimizations: BC7 hardware decompression, Geometry Engine for geometry processing, Primitive Shaders for advanced rendering. Development requires PlayStation SDK (licensed through Sony dev portal) and PS5 dev kits for testing.

Unified memory architecture (UMA) means CPU and GPU share 16GB pool—no separate VRAM. Budget ~12-13GB for games (OS reserves 3-4GB). Memory management differs from PC: Cannot exceed budget (crashes), must optimize for both CPU and GPU usage (textures + meshes + code + audio + render targets), and streaming is critical for large games (aggressive asset streaming keeps baseline memory under budget).

## Key Concepts

- **PS5 vs PS4 Specs**: PS5 = 10.3 TFLOPS, 16GB RAM, 5.5 GB/s SSD, RDNA 2. PS4 = 1.8 TFLOPS, 8GB RAM, ~50 MB/s HDD, GCN. 5-6x GPU performance, 2x memory, 100x storage speed. Cross-gen titles must scale significantly.
- **GNM/GNMX**: PlayStation graphics APIs. GNM = low-level (manual memory management, explicit synchronization, maximum performance). GNMX = high-level (easier to use, handles details automatically). Unity uses GNMX wrapper.
- **PSSL**: PlayStation Shader Language. HLSL-like syntax with PlayStation extensions. Compiles to PlayStation GPU bytecode. Unity's shader compiler translates HLSL to PSSL automatically.
- **Unified Memory Architecture (UMA)**: CPU and GPU share single 16GB memory pool. No dedicated VRAM. Flexible allocation but requires careful budgeting (textures + meshes + CPU data < 12-13GB total).
- **Ultra-Fast SSD**: 5.5 GB/s raw speed (8-9 GB/s compressed with hardware decompression). Enables on-demand asset streaming, virtual texturing, and instant level transitions. Game design implications (no elevators/corridors as hidden load screens).

## Best Practices

**Texture Guidelines:**
- Use BC7 for color textures: Same as PC/Xbox. Hardware decompression on PS5 GPU. 8:1 compression, excellent quality. PS4 also supports BCn.
- Use BC5 for normal maps: 2-channel compression. Standard technique across all platforms. Reconstruct Z in PSSL shader.
- PS5 texture budgets: 2K textures standard, 4K for hero assets. Target ~8-10GB total texture memory. Leverage SSD for streaming (virtual texturing or mipmap streaming).
- PS4 texture budgets: 1K-1.5K textures, 2K for hero assets only. Target ~3-4GB total texture memory. HDD streaming slower—preload more, stream less.
- Enable mipmaps: Critical for cache efficiency on both PS4 (limited bandwidth 176 GB/s) and PS5 (high bandwidth 448 GB/s but needed for performance).

**Mesh Guidelines:**
- PS5 handles complex meshes: Characters 15-25K triangles, environment 50-100K. RDNA 2 + Geometry Engine provides excellent geometry processing.
- PS4 requires optimization: Characters 8-12K triangles, environment 20-40K. GCN architecture slower at geometry. Implement LOD aggressively.
- Enable mesh compression: 50% size reduction. Beneficial on both consoles. PS5 SSD loads compressed meshes near-instantly (decompression hardware-accelerated).
- Use Primitive Shaders on PS5: Advanced feature (PS5-specific). Better performance than traditional vertex shaders for certain workloads (procedural geometry, tessellation). Requires low-level GNM programming (Unity doesn't expose yet).

**Shader Guidelines:**
- Write shaders in HLSL: Unity compiles HLSL to PSSL automatically. Avoid platform-specific #ifdefs unless necessary (use shader variants for PS4 vs PS5 differences).
- PS5 shader complexity: Target <150 instructions per fragment shader for 4K 60fps. RDNA 2 architecture handles complex shaders well.
- PS4 shader complexity: Target <100 instructions per fragment shader for 1080p 60fps. GCN architecture slower—simpler shaders required.
- Optimize for wave64: PS5 (RDNA 2) and PS4 (GCN) use 64-wide wavefronts (vs 32 on NVIDIA). Reduce register pressure (<32 VGPRs) for maximum occupancy. Use Razor shader profiler.
- Ray tracing on PS5: DXR-equivalent support. Light ray tracing acceptable (reflections, shadows, ambient occlusion). Heavy ray tracing (global illumination) too slow for 60fps—use for 30fps cinematic modes or limit scope.

**Memory Management:**
- PS5: 16GB total, ~12-13GB available for games. Budget: ~8-10GB GPU assets (textures, meshes, render targets), ~2-3GB CPU data (code, audio, gameplay state).
- PS4: 8GB total, ~5-7GB available for games (OS reserves vary by firmware). Budget: ~3-4GB GPU assets, ~2GB CPU data. Extremely tight—aggressive optimization required.
- Unified memory = flexible but dangerous: Easy to exceed budget (crash). Use Razor memory view to monitor allocation. Implement memory tracking (log current usage, warn at 90% capacity).
- Leverage SSD streaming on PS5: Keep baseline memory under 10GB, stream additional assets on-demand. Open-world games can maintain 8GB baseline, stream 10-20GB of assets as player explores.

**Render Target Guidelines:**
- PS5: 4K render targets (3840x2160) for native 4K or reconstructed (checkerboard, temporal). G-buffer with 4-5 MRTs acceptable (RDNA 2 handles well).
- PS4: 1080p render targets (1920x1080) standard. Native 1080p or checkerboard 1440p/1800p. Limit MRTs to 3-4 (GCN bandwidth-limited).
- Use dynamic resolution on PS5: Adjust 1800p-2160p to maintain 60fps. Rare drops below 1800p in heavy scenes. Keeps frame rate stable.
- Use checkerboard rendering on PS4: Render 1080p checkerboard, reconstruct to 1440p or 1800p. 50% pixel cost vs native. Standard technique for PS4 Pro 4K modes.

**SSD Optimization (PS5):**
- Implement aggressive streaming: Load assets on-demand. 5.5 GB/s SSD loads 1GB in ~200ms. Eliminate traditional load screens.
- Use virtual texturing: Stream texture tiles as needed. Baseline memory stays low, high-res textures stream instantly (SSD fast enough for real-time streaming).
- Prioritize I/O requests: PS5 I/O system supports priority queues. Mark critical assets (player character, immediate environment) high priority. Background assets (distant LODs) low priority.
- Test streaming on target hardware: SSD performance varies with drive usage (fragmentation, wear). Test on consumer PS5 units, not just dev kits.

**Platform-Specific Optimizations:**
- **PlayStation 5**: Target 4K 60fps or 1440p 120fps. 2K textures, high mesh detail, 2K shadows, light ray tracing (reflections). VRAM budget 8-10GB. Leverage SSD streaming.
- **PlayStation 4**: Target 1080p 30-60fps (30fps for demanding games, 60fps for optimized titles). 1K textures, medium mesh detail, 1K shadows, no ray tracing. VRAM budget 3-4GB. Preload most assets (HDD too slow for aggressive streaming).
- **PS4 Pro**: Intermediate spec (4.2 TFLOPS, 8GB RAM). Target 1440p 60fps or checkerboard 4K 30fps. 1.5K textures, higher LOD distances than base PS4. Memory budget same as PS4 (8GB total).

## Common Pitfalls

**Exceeding Unified Memory Budget**: Developer allocates 10GB textures + 3GB meshes + 2GB audio = 15GB total. PS5 has 12-13GB available. Game crashes (OS kills process). Symptom: Memory allocation failures, crashes in specific areas. Solution: Monitor memory with Razor. Implement streaming to keep baseline <10GB. Unload unused assets aggressively.

**PS4 Optimization Ignored**: Developer optimizes for PS5, assumes PS4 Pro will handle it (4.2 TFLOPS vs 10.3). PS4 Pro runs <30fps, base PS4 crashes (exceeds 5-7GB memory). Certification fails. Solution: Optimize for base PS4 first (lowest common denominator). Separate asset bundles for PS4 vs PS5 (different texture resolutions, mesh detail).

**SSD Streaming Assumptions**: Developer assumes SSD can stream 20GB/sec ("5.5 GB/s raw × 4 compression"). Reality: Compressed throughput is 8-9 GB/s peak, sustained is 4-6 GB/s (decompression overhead, I/O queuing). Over-aggressive streaming causes hitches. Solution: Profile I/O with Razor. Limit concurrent streaming to 2-4 GB/s sustained. Preload critical assets.

## Tools & Workflow

**PlayStation Razor**: Primary profiling tool. CPU profiler, GPU profiler, memory view, I/O analysis. Shows unified memory allocation, bandwidth usage, shader performance. Download from PlayStation dev portal (requires Sony partner access).

**PlayStation Tuner**: Real-time debugging and profiling. Connects to PS5/PS4 dev kit, shows live metrics (FPS, memory, CPU/GPU usage). Essential for optimization iteration.

**PlayStation SDK**: Required for PS5/PS4 development. Includes GNM/GNMX APIs, PSSL compiler, certification tools, debug libraries. Requires Sony developer license (ID@PlayStation or publisher partnership).

**Unity PlayStation Platform Settings**: Edit > Project Settings > Player > PlayStation. Configure texture quality, mesh compression, shader variants per console (PS5 vs PS4). Build separate builds.

**PSSL Shader Compiler**: Compiles PSSL shaders to PlayStation GPU bytecode. Unity invokes automatically during build. Manual compilation available via SDK tools for debugging.

**Certification Tools**: Sony provides automated tests for PlayStation certification (TRC = Technical Requirements Checklist). Validates performance, memory, compatibility, accessibility. Run before submission.

**Dev Kits**: PS5 and PS4 dev kits required for testing. Cannot ship game without testing on target hardware. Dev kits expensive (~$2,500 PS5 dev kit) but mandatory for professional development. Available through Sony dev portal.

## Related Topics

- [3.2 PlayStation APIs](03-02-PlayStation.md) - GNM, GNMX, PSSL details
- [8.1 Texture Compression Formats](08-01-Texture-Compression-Formats.md) - BC7/BC5 compression
- [4.2 Console Profiling Tools](04-02-Console-Profiling-Tools.md) - Razor, Tuner
- [2.4 Console-Specific Architecture](02-04-Console-Specific-Architecture.md) - PS5 GPU architecture

---

[← Previous: 9.2 Xbox Platform](09-02-Xbox-Platform.md) | [Next: 9.4 Nintendo Switch Platform →](09-04-Nintendo-Switch-Platform.md)
