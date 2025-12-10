# 3.3 Nintendo Switch

[← Back to Chapter 3](../chapters/03-Platform-Specific-Graphics-APIs.md) | [Main Index](../README.md)

Nintendo Switch uses NVN (Nintendo's Vulkan-based API) with unique constraints from mobile hardware, dual display modes, and aggressive thermal management.

---

## Overview

NVN is Nintendo's proprietary graphics API built on Vulkan foundations, optimized for Switch's NVIDIA Tegra X1 SoC. Unlike Vulkan's verbosity, NVN simplifies common operations while maintaining low-level control. It exposes explicit memory management, command buffer recording, and synchronization similar to DirectX 12 or Vulkan, but tailored for Switch's power and thermal constraints.

Switch operates in two modes: docked (GPU 768 MHz, TV output up to 1080p) and handheld (GPU 307-460 MHz, 720p screen). Games must handle 60% GPU performance reduction in handheld, requiring dynamic resolution, quality scaling, or frame rate caps. Thermal throttling further reduces clocks during intensive workloads—prolonged heavy rendering drops GPU to 307 MHz even in docked mode.

Switch's 4GB shared memory (3GB usable after OS reservation) and 25.6 GB/s bandwidth (17x lower than PS5) create tight constraints. Optimization focuses on memory efficiency (aggressive texture compression, small render targets, 16-bit precision) and bandwidth conservation (minimal overdraw, simple shaders, reduced texture sampling). Unity's low-end optimizations (URP mobile renderer, GPU instancing, LOD) are essential for hitting 30-60fps targets.

## Key Concepts

- **NVN (Nintendo eXtension for Vulkan)**: Low-level graphics API based on Vulkan. Command buffer architecture, explicit sync, and manual resource management. Simpler than Vulkan but retains performance benefits.
- **Docked vs Handheld Mode**: Two operating modes with different GPU clocks. Docked: 768 MHz GPU, 1080p TV target. Handheld: 307-460 MHz GPU, 720p screen. Performance differs by 40-60%.
- **Thermal Throttling**: GPU/CPU clocks reduce under sustained load to manage heat. Docked mode throttles to 460 MHz, handheld to 307 MHz. Affects performance unpredictably during long play sessions.
- **Unified Memory (4GB)**: CPU/GPU share 4GB LPDDR4 (25.6 GB/s bandwidth). OS reserves ~1GB, leaving 3GB usable. Tight memory budget requires aggressive compression and streaming.
- **Tegra X1 Architecture**: NVIDIA Maxwell-based SoC (2015). 256 CUDA cores, 32 TMUs, 16 ROPs. Mobile architecture prioritizing power efficiency over raw performance.

## Best Practices

**NVN API Usage:**
- Use command buffers for all rendering. Record static scenes once, resubmit each frame. NVN's lightweight command buffers have lower overhead than OpenGL.
- Minimize API calls. Each NVN call costs CPU cycles; Switch's ARM CPU (4 cores at 1 GHz) is slower than console x86 CPUs. Batch operations and reduce state changes.
- Leverage NVN's texture tiling for optimal bandwidth. Tiled textures improve cache locality, reducing memory access by 20-40%. Unity handles this automatically.
- Use NVN's dedicated transfer queue for asynchronous texture uploads. Stream textures while rendering, hiding I/O latency.

**Handheld vs Docked Optimization:**
- Implement dynamic resolution scaling. Target 720p handheld, 900p-1080p docked. Use URP's Dynamic Resolution feature or custom scaling logic.
- Scale quality settings per mode: texture resolution (1K handheld, 2K docked), shadow resolution (512 handheld, 1024 docked), particle counts (50% reduction handheld).
- Cap frame rate appropriately. 720p/30fps handheld is acceptable; forcing 60fps drains battery and causes throttling. Use VSync or custom frame limiters.
- Test extensively in handheld mode. It's the performance floor—if it runs at 30fps handheld, docked will exceed 60fps easily. Optimize for handheld first.

**Thermal Management:**
- Design for throttled clocks (460 MHz docked, 307 MHz handheld). Games maintaining 30fps at minimum clocks won't drop frames during thermal throttling.
- Implement aggressive LOD. Distant objects (<50m) use <500 triangle meshes. Switch's limited parallelism favors fewer, larger triangles over many small ones.
- Use dynamic quality scaling. Detect frame time spikes (thermal throttling), reduce quality temporarily (resolution, effects, draw distance) to maintain frame rate.
- Avoid sustained maximum GPU load. 100% GPU utilization for 10+ minutes triggers throttling. Cap at 80-90% target allows thermal headroom.

**Memory and Bandwidth Optimization:**
- Compress textures aggressively: ASTC 6x6 or 8x8 for most textures (12:1-16:1 compression). Quality loss is acceptable on 720p screen.
- Use 16-bit render targets where possible: R11G11B10_Float for HDR, R16G16_SNorm for normals. Saves 25-50% bandwidth vs 32-bit.
- Minimize VRAM allocation to <2GB. Leave 1GB for CPU and system overhead. Exceeding causes memory pressure and hitches.
- Reduce texture resolution: 1K max for most textures, 2K for hero assets only. Switch's 720p screen doesn't benefit from 4K textures.
- Limit overdraw to <2x average. Switch's 25.6 GB/s bandwidth is precious—excessive overdraw stalls rendering. Use frame debugger to profile overdraw.

**Platform-Specific:**
- **Switch OLED**: Identical hardware to original Switch. OLED screen (720p) has better contrast but same performance. Don't optimize separately.
- **Switch Lite**: Handheld-only device. Always runs in handheld mode (307-460 MHz GPU). Test exclusively in handheld settings.
- **Future Switch Hardware**: Rumors of Switch Pro/2 with NVIDIA Ampere/Ada. Prepare scalable assets (higher-res textures, complex shaders) gated behind quality settings.

## Common Pitfalls

**Ignoring Handheld Performance**: Game runs 60fps docked (768 MHz GPU, 1080p), ships without handheld testing. In handheld (384 MHz GPU, 720p), frame rate drops to 20fps. Symptom: User complaints about poor handheld performance. Solution: Test in handheld mode early and often. Implement dynamic resolution or quality scaling to maintain target frame rate.

**Memory Over-Allocation**: Loading 2.5GB of textures seems fine (1GB for CPU, 0.5GB system overhead leaves 2.5GB). But OS memory usage fluctuates (suspend/resume, screenshots, video capture), causing unpredictable evictions and hitches. Solution: Cap VRAM at 1.8-2.0GB, implement aggressive texture streaming, monitor memory profiler for spikes.

**Sustained Heavy Rendering Causing Throttling**: Game renders complex scenes at 100% GPU for 15 minutes. GPU throttles from 768 MHz to 460 MHz, dropping frame rate from 60fps to 38fps. Symptom: Frame rate degrades over time. Solution: Target 80-90% GPU utilization, implement LOD/culling to reduce load, or use dynamic resolution to adapt to throttling.

## Tools & Workflow

**NNGFX Debugger**: Switch's graphics debugger. Shows draw calls, shader execution, and resource usage. Limited compared to PIX/Razor but essential for Switch optimization. Use to identify heavy draws and overdraw.

**Nintendo Profiler**: CPU/GPU profiler showing frame breakdown. "GPU" timeline reveals rendering phases; "CPU" shows game logic cost. Use to balance CPU/GPU workload and identify bottlenecks.

**Unity Profiler with Switch Module**: Install Nintendo Switch profiler module. Provides platform-specific GPU markers and memory tracking. "Memory Profiler" shows texture and mesh allocations—identify oversized assets.

**RenderDoc (PC Validation)**: While RenderDoc doesn't support Switch directly, use it on PC (OpenGL ES 3.1) to validate rendering correctness before Switch deployment. Similar API constraints (mobile GPU tier).

**Frame Debugger**: Unity's built-in Frame Debugger works on Switch dev kits. Shows draw call breakdown, shader assignments, and batch analysis. Essential for identifying inefficient rendering patterns.

## Related Topics

- [2.4 Console-Specific Architecture](02-04-Console-Specific-Architecture.md) - Switch Tegra X1 architecture details
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory management strategies
- [5.4 Bandwidth Optimization](05-04-Bandwidth-Optimization.md) - Reducing bandwidth usage
- [13.1 Texture Formats and Compression](08-01-Texture-Compression-Formats.md) - ASTC compression for Switch

---

[← Previous: 3.2 PlayStation](03-02-PlayStation.md) | [Next: 3.4 Cross-Platform APIs →](03-04-Cross-Platform-APIs.md)
