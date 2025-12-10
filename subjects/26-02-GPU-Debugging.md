# 26.2 GPU-Level Debugging & Profiling Tools

[← Back to Chapter 26](../chapters/26-Debugging-Techniques.md) | [Main Index](../README.md)

GPU debugging = capture GPU commands and inspect them (render state, shader inputs, memory access patterns). Tools: RenderDoc (free, multi-API), PIX (Windows/Xbox native), Nsight (NVIDIA GPUs), GPUVendor Profilers (AMD Radeon Profiler, Intel GPA). Workflow: capture frame, replay in debugger, inspect each draw call (see shaders compiled, inputs, outputs, texture contents). Helps identify: shader bugs (unexpected color output), state errors (blending wrong), synchronization issues (GPU stalling on CPU).

---

## Overview

GPU Capture Process: (1) launch game with debugger attached, (2) press capture hotkey (Ctrl+G typical), (3) debugger captures all GPU commands for that frame, (4) save to file (.rdc RenderDoc, etc.), (5) analyze in debugger UI (timeline, draw calls, state, resources). Analysis: per-draw-call inspection (what shader, what inputs, what outputs), resource view (see texture contents, buffer data, render targets), timeline (GPU utilization per command). Bottleneck identification: if frame = 16.67ms budget, see which calls take longest = GPU hotspots.

Validation Layers: GPU driver layer (debug mode) = verify correctness (API usage errors caught = error messages logged). Enable: Vulkan validation layers, DX12 debug layer = performance overhead (use in debug builds only).

## Key Concepts

- **RenderDoc**: free, open-source, multi-platform (PC, console, mobile). Captures entire frame (all GPU commands, all resources used = self-contained replay). Interface: timeline (frame timeline = bars = draw calls), resource view (textures, buffers, render targets = inspect contents), state (graphics state at each call = rasterization, blending, depth settings). Benefit: deterministic replay (run same frame many times = consistent debugging, easier to reproduce bugs).
- **PIX** (Performance In eXecution): Windows + Xbox native, integrated performance profiling. Timeline view = GPU utilization per event (frame budget = bars showing time spent). GPU Events = custom markers (developer inserts PIXSetMarker = creates labeled regions = hierarchical profiling). Real-time capture (can run game, see performance live = not just offline analysis).
- **Nsight**: NVIDIA GPUs specific, deep profiling (shader metrics = instruction counts, register pressure = very detailed). Counter profiling = measure SM utilization, memory throughput, instruction cache hits. Helps: NVIDIA-specific optimization (warp utilization = achieve >80% for efficiency).
- **Shader Debugging**: most tools support step-through (pause GPU execution, step through shader code, inspect variables at each instruction). Challenge: shader optimization (compiler = may not preserve source order = debugging complex). Solution: compile shader in debug mode (no optimization = easier to debug, slower execution).
- **Memory Debugging**: GPU memory tracking (texture/buffer allocations tracked = see what's resident on GPU = memory budget validation). Helps: identify VRAM leaks (allocate but never free = memory grows over time).

## Best Practices

**RenderDoc Workflow**:
- Launch game (standalone or editor), attach RenderDoc (File > Launch or Attach to Running Process).
- Play scene until visual issue visible.
- Press Ctrl+G = capture frame.
- Open .rdc in RenderDoc UI (inspect draw calls, select problematic call).
- Check: texture inputs (right-click = inspect), shader code (bottom panel), resource state.

**PIX Workflow**:
- Launch game with PIX attached (Debug > Performance Profiler or built-in instrumentation).
- Play game (PIX records GPU commands in real-time).
- Can set GPU Budget ("maintain 60 FPS = 16.67ms target" = colored bars show if exceeded).
- View Performance > GPU tab = timeline visualization.

**Validation Layers**:
- Enable in debug builds (Edit > Project Settings > Graphics > Debug Level = Full/Medium/Low). Full = all checks = slowest, Medium = common checks = good balance.
- VK_LAYER_KHRONOS_validation enabled in Vulkan = logs API misuse (e.g., format mismatch = error message with suggestion).
- DX12 debug layer: Device.CreateDebugInterface enabled in D3D12.

**Custom Markers**:
```csharp
Profiler.BeginSample("Render Sky");
Graphics.Blit(skyTexture, skyRenderTarget);
Profiler.EndSample();
```

## Common Pitfalls

**Capture Not Working**: enable capture in game settings (some platforms disable by default), or device permission (Android = needs permission to profile). Symptom: capture hotkey doesn't work = frame not captured. Solution: check debugger output (error message = missing permission or capture disabled).

**GPU Measurements Misleading**: captured frame = GPU not under normal load (may be faster/slower than gameplay). Symptom: frame captures show fast (16ms), but gameplay stutters (30ms+). Solution: profile real gameplay, or average multiple captures (cherry-picked frame = outlier).

**Shader Step-Through Complex**: shader optimized = code order scrambled (difficult to step through). Symptom: debug experience poor (step doesn't match source = confusing). Solution: compile in debug mode (no optimization), or use printf debugging (output variables to texture = inspect).

## Tools & Workflow

**RenderDoc Capture + Analysis**:
1. Download RenderDoc (renderdoc.org)
2. Launch app, attach debugger
3. Ctrl+G = capture frame
4. Open .rdc file in RenderDoc UI
5. Select draw call = inspect

**Performance Markers** (cross-platform):
```csharp
#if UNITY_EDITOR
Unity.Profiling.ProfilerMarker marker = new("CustomPass");
using (marker.Auto()) {
    CustomPass();
}
#endif
```

## Related Topics

- [26.1 Visual Debugging](26-01-Visual-Debugging.md) - Frame Debugger
- [26.3 Performance Debugging](26-03-Performance-Debugging.md) - Performance analysis
- [26.4 Unity Debug Tools](26-04-Unity-Debug-Tools.md) - Profiler
- [04.4 Third-Party Profiling Tools](04-04-Third-Party-Profiling-Tools.md) - Tools overview

---

[Previous: 26.1 Visual Debugging](26-01-Visual-Debugging.md) | [Next: 26.3 Performance Debugging →](26-03-Performance-Debugging.md)
