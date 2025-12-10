# 31.3 Optimization Tools

[← Back to Chapter 31](../chapters/31-Third-Party-Tools.md) | [Main Index](../README.md)

Optimization tools = external profilers and analyzers (RenderDoc = GPU frame debugger, PIX = DirectX profiler, Nsight = NVIDIA GPU profiler). Beyond Unity Profiler: see GPU internals (waveform occupancy, register usage, cache hits), shader performance (instruction count, ALU/memory bound), draw call details (vertex count, overdraw, state changes). Essential for: console optimization (hit 60 FPS target), mobile (thermal/battery), AAA quality (squeeze every millisecond).

---

## Overview

RenderDoc: free open-source GPU debugger (Windows/Linux, supports D3D11/12, Vulkan, OpenGL). Capture frame: see all draw calls (geometry, textures, shaders), inspect resources (rendertargets, depth buffer), shader debugging (step through pixels, see interpolated values). Integration: capture Unity frame (Tools → RenderDoc → Capture), analyze in RenderDoc app. Use case: understand rendering (what's being drawn, why overdraw high, which shader expensive).

PIX (Performance Investigator for Xbox): Microsoft's DirectX profiler (free, Windows/Xbox). Features: GPU timeline (visualize draw calls, barriers, compute), shader profiler (hot spots, register pressure), memory bandwidth analysis (texture fetches, cache misses). Integration: attach to Unity process (PIX → Attach → select Unity), capture frame. Console-specific: official Xbox profiling tool (mandatory for certification = must hit performance targets).

## Key Concepts

- **RenderDoc**: open-source GPU debugger (https://renderdoc.org). Capture: hook into D3D/Vulkan API, record frame (all draw calls, resources, state). Features: Event Browser (list all draws), Pipeline State (vertex layout, shader bindings, blend modes), Texture Viewer (see rendertargets, mips, depth), Mesh Viewer (wireframe, vertex data), Shader Debugger (step through fragment, see registers). Unity integration: Window → Analysis → RenderDoc → Launch (captures build), or standalone inject (RenderDoc → Inject into Process → Unity).
- **PIX**: DirectX profiler (download: Microsoft PIX). Modes: GPU Capture (single frame analysis), Timing Capture (multi-frame timeline, find spikes). Features: GPU View (timeline of draw/compute, overlapping work), Shader Debugger (step through HLSL, see intermediate values), Pipeline Stats (vertex/fragment count, tessellation), Memory Stats (bandwidth, cache hit rate). Xbox: integrated profiler (required for certification, official performance targets).
- **NVIDIA Nsight Graphics**: NVIDIA GPU profiler (free, supports GeForce/Quadro). Features: Frame Debugger (similar to RenderDoc), GPU Trace (waveform occupancy, SM utilization), Shader Profiler (instruction latency, register usage, occupancy), Range Profiler (compare techniques = A/B testing). Unity: capture standalone build (Nsight → Launch → select .exe), or inject into editor. Use case: NVIDIA-specific optimization (Tensor cores, RT cores, DLSS integration).
- **Arm Mobile Studio**: profiling for ARM Mali GPUs (Android). Components: Streamline (system profiler = CPU/GPU/memory timeline), Graphics Analyzer (frame capture = draw calls, shaders), Mali Offline Compiler (shader analysis = cycle counts). Android: USB debug connection (ADB), capture running app. Use case: mobile optimization (thermal throttling, bandwidth reduction, tiled rendering analysis).
- **Unity Profiler Deep Dive**: built-in profiler enhancements. Module: Rendering (draw calls, batches, SetPass), GPU (GPU time per marker, memory), Memory (texture memory, mesh memory), Physics (collision time). Timeline View: see frame structure (which systems expensive, overlapping jobs). Deep Profile: record all function calls (expensive = editor only, find hotspots).
- **Third-Party Analyzers**: Graphy ($20, in-game stats overlay = FPS, RAM, advanced graphs), Simple Optimizations (free, automated checks = unused assets, texture compression), Build Report Tool (free, analyze build size = largest assets, shader variant counts). Use case: continuous monitoring (catch regressions), build optimization (reduce APK size).

## Best Practices

**Tool Selection by Platform**:
- **PC**: RenderDoc (D3D11/Vulkan), Nsight Graphics (NVIDIA GPUs), Intel GPA (Intel GPUs).
- **Console**: PIX (Xbox mandatory), Razor (PlayStation mandatory), Nintendo profiler (Switch).
- **Mobile**: Arm Mobile Studio (Mali GPUs), Snapdragon Profiler (Adreno), Xcode Instruments (iOS Metal).

**Profiling Workflow**:
1. Unity Profiler: identify bottleneck (CPU vs GPU, which system slow).
2. External profiler: deep dive GPU (RenderDoc = see draws, PIX = shader cost).
3. Shader analysis: identify expensive operations (too many texture fetches, complex math).
4. Iterate: optimize shader, re-profile, measure improvement.

**Frame Capture Tips**:
- Standalone build: capture standalone (editor overhead skews results).
- Release mode: profile release builds (debug symbols = slower).
- Worst case: capture heavy scene (many objects, effects enabled = stress test).

**Platform-Specific**:
- **Console**: use official tools (required for cert = PIX/Razor mandatory).
- **Mobile**: thermal state matters (cold start vs sustained = throttling changes performance).
- **VR**: stereo frame capture (two eyes = double draw calls, multiview optimization).

## Common Pitfalls

**Editor Profiling**: developer profiles in editor (play mode profiling = includes editor overhead). Symptom: performance looks worse than shipped game (editor UI, Scene view rendering = extra cost). Solution: always profile standalone builds (Build → Build and Run, then attach profiler).

**Too Many Shader Variants**: build includes 10,000+ shader variants (every material×keyword combination). Symptom: build time hours, APK 500MB+ (shader code bloat). Solution: Shader Variant Stripping (Edit → Project Settings → Graphics → Shader Stripping), analyze with Build Report Tool (find which shaders generating variants).

**Ignoring GPU Profiler**: developer only uses Unity Profiler CPU (misses GPU bottleneck). Symptom: frame time 30ms, CPU shows 10ms (remaining 20ms = GPU). Solution: enable GPU Profiler (Profiler → Add Profiler → GPU), or use RenderDoc (see actual GPU work).

## Tools & Workflow

**RenderDoc Frame Capture** (Unity):
```
1. Download RenderDoc: https://renderdoc.org
2. Unity: Window → Analysis → RenderDoc
3. Click "Launch" (captures standalone build automatically)
4. RenderDoc: File → Capture Frame (or F12 in game)
5. Analyze: Event Browser (list draws), Texture Viewer (see framebuffer)
```

**PIX Frame Capture** (DirectX):
```
1. Download PIX: https://devblogs.microsoft.com/pix/download/
2. PIX → GPU Capture → Launch Win32 (select Unity .exe)
3. In-game: Alt+F12 (capture frame)
4. PIX: Timeline View (see draw order), Shader tab (instruction count)
```

**NVIDIA Nsight Graphics**:
```
1. Download Nsight: https://developer.nvidia.com/nsight-graphics
2. Launch Nsight → Connect → Launch → select Unity.exe
3. In-game: Ctrl+Z (capture frame)
4. Analyze: Frame Debugger (draws), Range Profiler (compare techniques)
```

**Unity Profiler Deep Profile**:
```csharp
// Enable Deep Profile (editor only, expensive)
1. Profiler → Deep Profile (checkbox)
2. Play scene (records all function calls)
3. Select frame: see full callstack (which function called what)
4. Hierarchy: sort by Total Time (find expensive functions)
```

**Arm Mobile Studio (Android)**:
```
1. Download: https://developer.arm.com/Tools/Arm%20Mobile%20Studio
2. Connect device: USB debug enabled (ADB)
3. Streamline: Start Capture (select Unity app)
4. Graphics Analyzer: Capture Frame (inspect draws, shaders)
```

**Graphy Setup** (In-Game Stats):
```csharp
1. Import Graphy (Asset Store free, or $20 pro)
2. Prefab: drag Graphy Manager to scene
3. Configure: FPS graph (green/yellow/red thresholds), RAM graph, advanced GPU stats
4. Build: stats overlay in shipped game (toggle with hotkey)
```

## Related Topics

- [04.1 Unity Profiler](04-01-Unity-Profiler.md) - Built-in profiling
- [26.2 GPU Debugging Tools](26-02-GPU-Debugging-Tools.md) - RenderDoc/PIX details
- [26.3 Performance Debugging](26-03-Performance-Debugging.md) - Bottleneck identification
- [25.1 Build Optimization](25-01-Build-Optimization.md) - Shader variant stripping

---

[← Previous: 31.2 Rendering Extensions](31-02-Rendering-Extensions.md) | [Next: 31.4 Asset Creation Tools →](31-04-Asset-Creation-Tools.md)
