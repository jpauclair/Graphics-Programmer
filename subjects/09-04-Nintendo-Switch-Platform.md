# 9.4 Nintendo Switch Platform Asset Format Guidelines

[← Back to Chapter 9](../chapters/09-Asset-Format-Guidelines.md) | [Main Index](../README.md)

Nintendo Switch guidelines address extreme resource constraints—weak CPU, limited memory, and thermal throttling—requiring aggressive optimization and ASTC texture compression.

---

## Overview

Nintendo Switch (NVIDIA Tegra X1, 256 CUDA cores, 4 CPU cores @ 1 GHz, 4GB RAM with 3GB usable) is the most challenging modern platform. GPU performance is 0.4-0.5 TFLOPS (docked 768MHz, handheld 307-460MHz variable due to thermal throttling). This is 20x weaker than PS5, 25x weaker than Xbox Series X, 8x weaker than Xbox One. Memory is severely limited: 3GB usable (OS reserves 1GB) split between CPU and GPU. Bandwidth is constrained: 25.6 GB/s (7-22x less than other consoles). Every optimization technique matters on Switch.

Switch-specific challenges: docked vs handheld modes (2.5x GPU performance difference, thermal throttling in handheld), ASTC texture compression mandatory (no BCn support), limited CPU (1 GHz ARM Cortex-A57 cores struggle with draw calls and physics), and tiny memory budget (3GB total for everything). Asset strategy: target 720p handheld (lower GPU load), use ASTC 8x8 compression aggressively, implement 4-5 LOD levels with aggressive transitions, reduce draw calls below 500-800, and keep total memory under 2.5GB.

Development workflow: develop on PC, test on Switch dev kit constantly (performance characteristics drastically different), profile with Nintendo NNGFX Debugger and LLGD, optimize for handheld mode first (worst case), validate battery life (2-3 hours minimum, 4-6 hours ideal), and test thermal throttling (extended play sessions). Cross-platform titles face massive downscaling: PS5/PC assets at 4K/2K textures reduce to 1K/512 on Switch.

## Key Concepts

- **Docked vs Handheld**: Docked = 768MHz GPU, 1080p output, better cooling. Handheld = 307-460MHz GPU (variable thermal throttling), 720p screen, battery-powered. Target handheld mode for optimization (worst case).
- **ASTC Compression**: Adaptive Scalable Texture Compression. Mandatory on Switch (no BCn support). ASTC 6x6 = 5.3:1 high quality, ASTC 8x8 = 8:1 moderate quality, ASTC 12x12 = 18:1 low quality. Use 8x8 as standard.
- **Memory Budget**: 4GB total RAM, 3GB usable (1GB OS reserve). Unified memory for CPU/GPU (like PS5 but 5x smaller). Target <2.5GB for safety margin. Typical split: 1.5GB VRAM, 0.8GB CPU, 0.2GB buffer.
- **Thermal Throttling**: Extended play heats device, GPU downclocks to 307MHz minimum (60% performance loss vs 768MHz docked). Game must maintain acceptable performance at minimum clocks.
- **NVN API**: Nintendo Vulkan-based API (NVN = Nintendo Vulkan Next). Low-level graphics API similar to Vulkan/DX12. Unity wraps NVN, exposes via platform-specific settings.

## Best Practices

**Texture Guidelines:**
- Use ASTC 8x8 for most textures: 8:1 compression, acceptable quality. Import Settings > Switch Platform > Format = ASTC 8x8. Balances quality and memory.
- Use ASTC 6x6 for hero assets only: 5.3:1 compression, better quality. Reserve for player character, key objects. Wastes memory on background assets.
- Use ASTC 12x12 for distant objects: 18:1 compression, lower quality but acceptable for background, foliage, particles (viewed at distance).
- Maximum texture resolution: 1K (1024x1024) for most textures, 2K only for hero character/objects, 512 for background/foliage, 256 for particles. 4K textures impossible (memory).
- Implement texture streaming: Mipmap Streaming essential. Load only visible mips. Reduces VRAM from 2GB to 0.8-1.2GB in open-world games.

**Mesh Guidelines:**
- Extremely low poly: Characters 5-8K triangles, environment props 2-5K, background objects <1K. GPU cannot handle high poly counts (0.4 TFLOPS).
- Implement 4-5 LOD levels: LOD0 (0-5m), LOD1 (5-20m), LOD2 (20-50m), LOD3 (50-100m), LOD4 (100m+ or culled). Aggressive transitions to shed triangles quickly.
- Enable mesh compression High: 50-60% size reduction. Critical for meeting 3GB memory budget.
- Vertex complexity matters: Reduce vertex attributes. Disable tangents if possible (calculate in shader), use compressed normals, limit UV sets to 1-2.

**Shader Guidelines:**
- Target <80 ALU instructions: Fragment shaders must be simple. Switch GPU (Maxwell-based) has low instruction throughput. <80 instructions for 720p 30fps, <50 for 60fps.
- Use half precision aggressively: Maxwell GPU has FP16 support. `half4 color`, `half3 normal`. Doubles ALU throughput. Essential for performance.
- Minimize texture samples: Each sample costs 30-50 cycles (bandwidth-limited). Target <4 texture samples per fragment shader. Combine textures (pack channels).
- Avoid complex lighting: Forward rendering with 1-2 lights max. Deferred rendering too expensive (bandwidth). Light baking essential (lightmaps for ambient, real-time for key lights only).
- Profile with NNGFX shader analysis: Shows instruction count, register usage, estimated cycles. Optimize expensive shaders (>100 instructions).

**Memory Management:**
- Total budget: 3GB usable. Target <2.5GB for safety. Typical allocation: 1.5GB textures, 0.4GB meshes, 0.3GB audio, 0.3GB code/gameplay.
- Implement aggressive streaming: Load/unload assets constantly. Player enters new area = unload previous area, load new area. Cannot keep everything in memory.
- Compress everything: ASTC 8x8 textures (8:1), mesh compression High (50%), Vorbis audio 64kbps (15:1). Every byte matters.
- Monitor memory with LLGD: Nintendo's profiler shows real-time memory usage. Validate game stays under 2.5GB in all scenarios (worst-case area with max assets loaded).

**Performance Targets:**
- Handheld mode: Target 720p 30fps minimum. 60fps ideal but difficult to achieve (most AAA games are 30fps on Switch). Test at 307MHz GPU clock (thermal throttling worst case).
- Docked mode: Target 1080p 30-60fps. 2.5x GPU performance vs handheld allows higher resolution or frame rate. Don't rely on docked mode—handheld is primary use case.
- Draw calls: <500-800 max. CPU at 1 GHz cannot handle 1,000+ draw calls (5-10ms CPU time). Use static batching, GPU instancing, SRP Batcher aggressively.
- Bandwidth: 25.6 GB/s (lowest of all platforms). Overdraw must be minimal (<1.5x average). Texture sampling must be efficient (mipmaps, compression).

**Thermal and Battery Considerations:**
- Thermal throttling: GPU downclocks during extended play (30-60 minutes). Test for 2+ hours continuously. Game must maintain acceptable performance at 307MHz minimum clock.
- Battery life: Target 3-6 hours gameplay. Handheld mode battery = 4,310 mAh. Intensive GPU/CPU usage drains battery quickly (2-3 hours). Moderate usage extends to 4-6 hours.
- Optimize for power efficiency: Lower GPU load (reduce resolution, simplify shaders), lower CPU load (reduce draw calls, simplify physics), implement frame rate limiter (30fps uses less power than 60fps).

**Platform-Specific Optimizations:**
- **Handheld Mode (Primary Target)**: 720p 30fps, ASTC 8x8 textures, 1K max resolution, <500 draw calls, aggressive LOD, simple shaders (<80 instructions), memory <2.5GB.
- **Docked Mode (Bonus)**: 900p-1080p 30-60fps (if performance allows), can increase LOD distances slightly, maybe ASTC 6x6 for hero assets, draw call budget can increase to 800.
- **Cross-Platform Ports**: Expect 5-10x asset reduction vs PS5/Xbox. 4K textures → 1K. 30K triangle characters → 5K. Deferred rendering → forward. Ray tracing → baked lighting. Heavy post-processing → minimal.

## Common Pitfalls

**Using BCn Textures**: Developer imports textures with PC/console settings (BC7). Switch doesn't support BCn formats (NVIDIA Tegra has no BC hardware). Game fails to build or textures appear broken. Symptom: Build errors or corrupted textures. Solution: Always use ASTC for Switch. Import Settings > Switch Platform > Format = ASTC 8x8.

**Docked Mode Optimization**: Developer tests only docked mode (1080p, 768MHz GPU). Looks great at 60fps. Switch to handheld (720p, 307-460MHz GPU) = 15-25fps unplayable. Thermal throttling after 30 minutes makes it worse. Symptom: Handheld performance terrible. Solution: Optimize for handheld mode first (worst case). Docked performance will exceed it.

**Exceeding Memory Budget**: Cross-platform game allocates 8GB VRAM on PC/consoles. Switch port tries loading same assets = crashes (exceeds 3GB total memory). Symptom: Out of memory crashes, texture thrashing. Solution: Create Switch-specific asset bundles. 1K textures, low-poly meshes, aggressive streaming. Target <2.5GB total memory (includes OS overhead).

## Tools & Workflow

**Nintendo NNGFX Debugger**: Primary GPU profiler. Shows draw calls, GPU timing, shader performance, texture usage. Capture frame, analyze bottlenecks. Download from Nintendo Developer Portal (NDI account required).

**LLGD (Low-Level GPU Debugger)**: Nintendo's graphics debugger. Captures GPU commands, inspects memory, validates API usage. Similar to RenderDoc but Switch-specific. Essential for low-level optimization.

**Nintendo SDK (NintendoSDK)**: Required for Switch development. Includes NVN API, compilers, profiling tools, certification tools. Requires Nintendo developer license (apply via NDI portal).

**Unity Switch Platform Settings**: Edit > Project Settings > Player > Switch. Configure texture format (ASTC), mesh compression, shader settings. Build Switch-specific builds.

**Profiler Module**: Unity Profiler connects to Switch dev kit. Shows CPU profiling, memory usage, rendering stats. Essential for identifying CPU bottlenecks (draw calls, scripts, physics).

**Switch Dev Kit**: Required for testing. Cannot ship game without testing on actual hardware. Dev kits ~$500 USD. Available through Nintendo Developer Program (NDI).

**Certification Lotcheck**: Nintendo's certification process. Validates technical requirements (performance, memory, stability), content guidelines, and user experience (boot time, suspend/resume). Automated tools + manual testing by Nintendo QA.

## Related Topics

- [3.3 Nintendo Switch APIs](03-03-Nintendo-Switch.md) - NVN API details
- [8.1 Texture Compression Formats](08-01-Texture-Compression-Formats.md) - ASTC compression
- [4.2 Console Profiling Tools](04-02-Console-Profiling-Tools.md) - NNGFX Debugger, LLGD
- [2.4 Console-Specific Architecture](02-04-Console-Specific-Architecture.md) - Switch GPU architecture

---

[← Previous: 9.3 PlayStation Platform](09-03-PlayStation-Platform.md) | [Next: Chapter 10 - Asset Bundling and Streaming →](../chapters/10-Asset-Bundling-And-Streaming.md)
