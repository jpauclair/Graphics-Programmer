# 3.1 DirectX (PC/Xbox)

[← Back to Chapter 3](../chapters/03-Platform-Specific-Graphics-APIs.md) | [Main Index](../README.md)

DirectX is Microsoft's graphics API for Windows PC and Xbox consoles. Understanding its evolution from DirectX 11 to DirectX 12, and Xbox-specific optimizations, is essential for high-performance rendering on these platforms.

---

## Overview

DirectX 11 (2009) provides a high-level abstraction over GPU hardware with automatic resource management and driver-controlled optimization. It's easier to use but limits control and adds CPU overhead—typical draw call costs 0.3-0.5ms CPU time. DirectX 12 (2015) exposes low-level GPU control similar to console APIs, reducing draw call overhead to 0.05-0.1ms but requiring explicit synchronization, memory management, and multi-threading from developers.

Xbox Series X/S use DirectX 12 Ultimate with console-specific extensions: explicit memory pool control, hardware-accelerated VRS, and mesh shaders. Xbox One uses DirectX 11-based API with some DirectX 12 features. Unity abstracts these differences via graphics API selection, but understanding underlying API capabilities helps optimize for platform strengths. HLSL (High-Level Shading Language) is the shader language for DirectX, compiling to platform-specific bytecode.

Modern PC/Xbox development targets DirectX 12 for maximum performance and feature access. However, DirectX 11 remains relevant for min-spec PC support (older GPUs, Windows 7) and simpler projects where explicit management isn't justified. Unity's SRP (Scriptable Render Pipeline) uses DirectX 12 on Xbox Series and modern PCs automatically.

## Key Concepts

- **DirectX 11**: Immediate-mode API with driver-managed resources. Simple to use, high CPU overhead. Used by Built-in Render Pipeline and older Unity versions. Still relevant for min-spec PC compatibility.
- **DirectX 12**: Explicit low-level API requiring manual resource management, synchronization, and multi-threading. Lower CPU overhead, more control. Used by URP/HDRP on modern platforms.
- **Command Lists**: DirectX 12's recording mechanism for GPU commands. Record once, submit repeatedly. Enables multi-threaded command recording (parallel scene submission).
- **Root Signature**: DirectX 12 structure defining shader resource bindings (CBVs, SRVs, UAVs). Must match shader expectations. Changing root signature flushes pipeline—minimize changes.
- **HLSL (High-Level Shading Language)**: C-like shader language for DirectX. Compiles to DXBC (DirectX 11) or DXIL (DirectX 12). Supports shader model 5.0 (DX11) to 6.6 (DX12 Ultimate).

## Best Practices

**DirectX 12 Optimization:**
- Use command lists for all rendering work. Record static scenes once at init, re-submit each frame. Reduces per-frame CPU cost by 80%.
- Implement multi-threaded command recording. Split scene into chunks (frustum sectors, layers), record per-chunk on worker threads. Scales to 8+ cores.
- Minimize root signature changes. Group draws by root signature, then by PSO (Pipeline State Object). Each change costs 0.1-0.3ms on consoles.
- Use ExecuteIndirect for GPU-driven rendering. Compute shaders generate draw args, GPU executes without CPU involvement. Eliminates CPU culling overhead.
- Leverage async compute. Schedule post-processing or particle simulation on compute queue during shadow/geometry rendering. Improves GPU utilization by 15-25%.

**HLSL Shader Authoring:**
- Use shader model 6.0+ for wave intrinsics (`WaveActiveSum`, `WaveReadLaneFirst`). Enables efficient inter-thread communication without shared memory.
- Leverage half-precision types (`min16float`, `min16int`) on Xbox and modern PCs. Doubles ALU throughput vs FP32 with negligible quality loss.
- Avoid dynamic indexing into arrays (`array[variable]`). Prevents compiler optimization and costs extra instructions. Use constants or unroll loops.
- Use `[branch]` and `[flatten]` attributes to control flow. `[branch]` generates actual branches (good if rarely taken), `[flatten]` executes both paths (good if frequently taken).
- Profile shader compilation times. Complex Shader Graph shaders can take 30-60 seconds to compile. Use `#pragma skip_optimizations` during development, enable for builds.

**Xbox Series X/S Features:**
- Enable Variable Rate Shading (VRS) tier 2 for 30-40% fragment shader savings. Reduce shading rate in peripheral vision (2x2 or 4x4 pixels per shader invocation).
- Use mesh shaders for LOD or procedural geometry. Bypasses vertex shader, provides more control over geometry. Requires DirectX 12 Ultimate and shader model 6.5+.
- Leverage sampler feedback for texture streaming. GPU reports accessed texture mips, streaming system loads only necessary data. Reduces memory by 40-60% in open-world games.
- Place render targets in optimal memory pool (10GB fast). Query `D3D12_FEATURE_DATA_ARCHITECTURE1` to detect memory pools, allocate via `ID3D12Heap::GetGPUVirtualAddress`.

**Cross-Platform DX11/DX12:**
- Unity abstracts DX11/DX12 via GraphicsDeviceType. Use `SystemInfo.graphicsDeviceType` to detect API at runtime and adjust quality (disable expensive features on DX11).
- Test on both APIs during development. DX12 bugs may not appear in DX11 (resource state mismatches, synchronization issues). RenderDoc and PIX catch API violations.
- For custom native plugins, use `IUnityGraphicsD3D12v5` interface to access DirectX 12 device. Coordinate resource states with Unity's command buffers.

**Platform-Specific:**
- **PC DirectX 12**: Target shader model 6.0 min (Windows 10 1809+). Use Agility SDK for latest features (enhanced barriers, work graphs) on older Windows 10 versions.
- **Xbox Series X**: Full DirectX 12 Ultimate feature set. Prioritize 4K, ray tracing optional (costs 4-8ms for reflections/GI). Use fast memory pool religiously.
- **Xbox Series S**: Same API as Series X but lower specs (20 CUs vs 52, 8GB vs 16GB). Target 1440p, disable ray tracing or use minimal bounce counts (1-2 bounces max).
- **Xbox One**: DirectX 11-based with some DX12 features. No mesh shaders, VRS, or ray tracing. Async compute available but GCN architecture requires careful scheduling.

## Common Pitfalls

**Resource State Mismatches in DirectX 12**: Rendering to texture as `RENDER_TARGET`, then immediately sampling as `PIXEL_SHADER_RESOURCE` without transition causes corruption or GPU crash. DirectX 12 requires explicit state transitions via `ResourceBarrier`. Unity handles this internally, but custom native plugins must track states manually. Symptom: Black textures, flickering, or device removed errors.

**Excessive Root Signature Changes**: Binding 50 different material CBVs (constant buffer views) via root descriptors costs 0.2ms × 50 = 10ms per frame. Root signature changes flush pipeline state, stalling GPU. Solution: Use descriptor tables (bind array of descriptors once) or SRP Batcher (Unity uploads material properties to persistent GPU buffer).

**Ignoring Xbox Memory Pools**: Allocating render targets in standard memory pool (336 GB/s) instead of optimal pool (560 GB/s) causes 40% slower rendering for bandwidth-intensive workloads. Series X splits 16GB into pools, but default allocation uses standard. Solution: Query memory architecture, explicitly allocate performance-critical resources in fast pool.

## Tools & Workflow

**PIX for Windows**: Essential DirectX 12 debugger. GPU captures show command list structure, resource states, and shader execution. "Timing Data" reveals CPU/GPU work breakdown. "Memory" view shows allocations and leaks. Use for DirectX 12 validation and optimization.

**RenderDoc**: Alternative DirectX 11/12 debugger. Great for API state inspection and shader debugging. Texture viewer shows resource formats and contents. Works on PC only (not Xbox).

**Visual Studio Graphics Debugger**: Built into VS, supports DirectX 11/12 debugging. Frame capture, shader debugging, and pixel history. Simpler than PIX but less powerful. Good for quick checks during development.

**HLSL Compiler (DXC)**: Offline shader compiler for HLSL. Produces DXIL bytecode (DirectX 12) or DXBC (DirectX 11). Unity invokes automatically, but standalone compilation helps debug complex shaders. Use `-Od` for debug symbols, `-O3` for max optimization.

**Xbox PIX**: Console-specific version with hardware performance counters. Shows CU utilization, memory pool allocation, and VRS effectiveness. Essential for Xbox optimization—exposes details unavailable in PC PIX.

## Related Topics

- [2.4 Console-Specific Architecture](02-04-Console-Specific-Architecture.md) - Xbox hardware architecture details
- [4.1 PC Profiling Tools](04-01-PC-Profiling-Tools.md) - PIX and DirectX profiling
- [12.1 Unity Shader Languages](12-01-Unity-Shader-Languages.md) - HLSL in Unity shaders
- [24.2 Async Compute](05-02-GPU-Side-Optimization.md) - DirectX 12 multi-queue rendering

---

[← Previous: Chapter 3 Index](../chapters/03-Platform-Specific-Graphics-APIs.md) | [Next: 3.2 PlayStation →](03-02-PlayStation.md)
