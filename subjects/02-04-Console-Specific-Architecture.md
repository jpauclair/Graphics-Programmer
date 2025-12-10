# 2.4 Console-Specific Architecture

[← Back to Chapter 2](../chapters/02-GPU-Hardware-Architecture.md) | [Main Index](../README.md)

Console GPUs have unique architectural characteristics optimized for fixed hardware targets. Understanding platform-specific features enables maximum performance extraction from each console.

---

## Overview

Console GPUs differ fundamentally from PC GPUs despite shared lineage (Xbox uses AMD RDNA, Switch uses NVIDIA Tegra). Consoles have unified memory architectures (GPU and CPU share RAM), fixed clock speeds, and platform-specific APIs exposing hardware features unavailable on PC. Developers can optimize aggressively for known hardware specs—no need to scale across GTX 1060 to RTX 4090. This enables techniques impossible on PC: explicit memory management, hardware-specific shader paths, and low-level GPU control.

Current-gen consoles (PS5, Xbox Series X/S) use RDNA 2 architecture with ray tracing hardware, variable rate shading, and mesh shaders. Last-gen (PS4, Xbox One) used GCN architecture with different performance characteristics and limitations. Cross-gen games must handle both—GCN's lower bandwidth (176 GB/s PS4 vs 448 GB/s PS5) and compute throughput (1.84 TFLOPS PS4 vs 10.3 TFLOPS PS5) require separate optimization strategies.

Switch stands apart with mobile architecture (NVIDIA Maxwell-based Tegra X1, 2015 tech). It operates in two modes: docked (768 MHz GPU, 1080p target) and handheld (307-460 MHz GPU, 720p screen). Thermal throttling reduces clocks during intensive workloads, making predictable performance challenging. Switch optimization focuses on minimizing power draw and maintaining stable frame rates across modes.

## Key Concepts

- **Unified Memory Architecture (UMA)**: CPU and GPU share same RAM pool. Xbox Series X has 16GB total; CPU/GPU allocate dynamically. Eliminates CPU→GPU data copies but requires careful memory budget management.
- **RDNA 2 Architecture**: AMD's current GPU architecture (PS5, Xbox Series X/S). Features hardware ray tracing, Infinity Cache, improved compute, and variable rate shading. More efficient than GCN per-clock.
- **GCN Architecture**: AMD's previous GPU architecture (PS4, Xbox One). Optimized for compute but lower geometry throughput than RDNA. Requires explicit async compute management for peak performance.
- **Tegra X1 Architecture**: NVIDIA's mobile SoC (Switch). Maxwell-based (2015), 256 CUDA cores, 32-bit memory bus (limited bandwidth). Emphasis on power efficiency over raw performance.
- **Memory Pools**: Xbox Series X divides 16GB into 10GB "optimal" (560 GB/s) and 6GB "standard" (336 GB/s). Render targets and frequently accessed textures should use optimal pool for best performance.

## Best Practices

**Xbox Series X/S Optimization:**
- Place render targets and dynamic resources in 10GB fast memory pool. Static textures can use slower 6GB pool. Query available pools via DirectX 12 APIs.
- Leverage DirectX 12 Ultimate features: mesh shaders (better LOD control), sampler feedback (texture streaming), and VRS tier 2 (image-based shading rate).
- Profile for both Series X and Series S separately. Series S has 10GB total (8GB usable), 20 CUs vs Series X's 52 CUs. Target 1440p on Series S vs 4K on Series X.
- Use async compute extensively. RDNA 2's compute queues allow post-processing during shadow rendering, improving GPU utilization by 15-20%.
- Leverage PIX for Xbox's hardware counters: CU utilization, memory pool allocation, VRS effectiveness. Xbox PIX provides insights unavailable in PC PIX.

**PlayStation 5 Optimization:**
- Utilize 16GB unified GDDR6 at 448 GB/s. Allocate ~10-12GB for GPU, leaving 4-6GB for CPU/OS. Exceed this and OS may force evictions (hitches).
- Leverage Geometry Engine for primitive culling. PS5 has dedicated hardware culling invisible primitives before rasterization (10-30% performance gain in dense scenes).
- Use explicit async compute via GNM/GNMX. Schedule compute work during fragment-light rendering phases (shadows, depth prepass). Requires careful synchronization.
- Take advantage of high GPU clock (2.23 GHz vs Xbox's 1.825 GHz). PS5 favors shader complexity over parallel work—fewer CUs but faster execution.
- Profile with Razor and Tuner. Razor shows detailed wavefront occupancy and memory bandwidth. Tuner provides CPU/GPU timeline for synchronization analysis.

**Nintendo Switch Optimization:**
- Target 720p handheld, 900p-1080p docked. Use dynamic resolution to maintain frame rate as GPU clock throttles under thermal load.
- Minimize memory usage: 4GB shared between CPU/GPU/OS (3GB usable for games). Compress textures aggressively (ASTC 6x6 or 8x8), use 16-bit render targets.
- Limit shader complexity: <150 ALU instructions for fragment shaders. Switch's 256 CUDA cores at 768 MHz have limited throughput. Favor simpler shading models.
- Reduce bandwidth consumption: 25.6 GB/s is 17x lower than PS5. Minimize overdraw (<2x average), use small render targets, avoid blending.
- Test in handheld mode extensively. GPU clocks down to 307 MHz minimum—performance drops 60% from docked. Use frame rate governors to prevent drops below 30fps.

**Unified Memory Management:**
- Profile memory allocation carefully on consoles. GPU over-allocation causes CPU starvation (scripting, physics suffer). Aim for 60/40 GPU/CPU split on 16GB consoles.
- Avoid excessive CPU↔GPU transfers. Unified memory eliminates copy overhead but cache coherency costs remain. Use persistent mapped buffers for frequent updates.
- Leverage fast GPU-accessible CPU memory for dynamic data (particle buffers, indirect args, skinning outputs). No PCIe transfer latency.

**Platform-Specific:**
- **Xbox One X**: 12GB GDDR5 (326 GB/s), 40 CUs. Target 4K checkerboard or dynamic 1800p-4K. GCN architecture—async compute critical for peak performance.
- **PS4 Pro**: 8GB GDDR5 (218 GB/s), 36 CUs. Target 1440p-4K checkerboard. Geometry Engine helps cull efficiently. Use ID buffer for checkerboard reconstruction.
- **Switch OLED**: Same hardware as original Switch. OLED screen (720p) has better colors but identical performance. Don't optimize separately.

## Common Pitfalls

**Ignoring Series S Limitations**: Developing for Series X (52 CUs, 10GB fast RAM) and scaling down to Series S (20 CUs, 8GB total) by lowering resolution isn't sufficient. Series S needs aggressive texture resolution caps (1K instead of 2K), reduced draw distances, and simplified shaders. Symptom: Series S drops to 25fps while Series X maintains 60fps. Solution: Separate quality presets with meaningful cuts beyond resolution.

**Over-Allocating PS5 Memory**: Using 14GB for GPU assets seems fine (2GB left for CPU). But OS reserves fluctuate (suspend/resume, notifications, video recording), causing unpredictable evictions. Frame hitches appear randomly. Solution: Cap GPU allocation at 10-11GB, leaving 5-6GB headroom. Profile with Razor's memory tracking to catch peaks.

**Thermal Throttling on Switch**: Game runs 60fps in docked mode during testing (GPU at 768 MHz). After 30 minutes, GPU throttles to 460 MHz due to heat, dropping frame rate to 35fps. Symptom: Initially good performance degrading over time. Solution: Test for extended periods (1+ hours), implement dynamic resolution/quality scaling to maintain frame rate during throttling.

## Tools & Workflow

**PIX for Xbox**: Essential for Xbox development. Hardware counters expose CU utilization, memory pool allocation, and async compute effectiveness. Timeline view shows Series X vs Series S performance side-by-side. Use "Frame Analysis" for detailed component breakdown.

**PlayStation Razor**: Primary PS5/PS4 profiler. "Wavefront Occupancy" view shows CU utilization per shader. "Memory" tab displays VRAM usage and bandwidth consumption. "Command Buffer" timeline reveals GPU work scheduling and sync points.

**PlayStation Tuner**: CPU/GPU profiler showing both timelines simultaneously. Identifies CPU-GPU sync issues and async compute opportunities. "Trace" mode captures detailed hardware events for deep analysis.

**Nintendo NNGFX Debugger**: Switch's graphics debugger. Shows frame breakdown, draw calls, and shader execution. Limited compared to Xbox/PlayStation tools but essential for Switch optimization. Use for identifying heavy draws and overdraw.

**Unity Profiler with Platform Modules**: Install Xbox, PlayStation, or Switch profiler modules. Provides platform-specific GPU markers and memory tracking. Deep Profile shows native API calls, revealing platform-specific overhead.

## Related Topics

- [2.1 GPU Memory Hierarchy](02-01-GPU-Memory-Hierarchy.md) - Console memory systems and unified architectures
- [3.1 DirectX (PC/Xbox)](03-01-DirectX.md) - Xbox DirectX 12 API specifics
- [3.2 PlayStation](03-02-PlayStation.md) - PlayStation GNM/GNMX API details
- [3.3 Nintendo Switch](03-03-Nintendo-Switch.md) - Switch NVN API and optimization

---

[← Previous: 2.3 GPU Components](02-03-GPU-Components.md) | [Next: Chapter 3 - Platform-Specific Graphics APIs →](../chapters/03-Platform-Specific-Graphics-APIs.md)
