# 21.4 GPU Synchronization

[← Back to Chapter 21](../chapters/21-Frame-Management-And-Synchronization.md) | [Main Index](../README.md)

GPU synchronization coordinates CPU-GPU execution: command buffering (CPU queues GPU work ahead), GPU fences (signal frame completion), pipeline stalls (CPU waits for GPU = frame rate loss). Goal: keep GPU busy while avoiding stalls (maintain high utilization, eliminate bubbles = idle frames).

---

## Overview

CPU-GPU Pipeline: CPU writes commands (Draw, Compute, Copy), GPU executes asynchronously (CPU continues to next frame while GPU renders previous frame). Typical frame (CPU updates scene, writes draw commands to command buffer, submits to GPU, CPU idle or doing physics/animation, GPU executes draw commands, frame later displays). Latency: 1-3 frames behind (CPU frame 100 submitting, GPU frame 98-99 executing, display frame 97-98 visible). Optimization: maximize GPU workload ahead of CPU (queue multiple frames), avoid stalls (CPU waiting on GPU results = zero throughput).

Command Buffers: CPU records GPU commands (Draw, Compute, Dispatch, Copy operations) into buffer. Immediate: commands execute as queued (traditional, immediate-mode = CPU sends one command, GPU executes), deferred: commands recorded then submitted batch (modern = record then replay). Unity: CommandBuffer API (record custom commands), or automatic (Scene batching records commands). Optimization: reuse buffers (record once, execute many times), or regenerate if scene changes (dynamic objects = regenerate every frame).

GPU Fences: Synchronization primitives (mark frame completion = CPU knows GPU finished). Implementation: GPU.InsertFence() marks GPU timeline, GPU.WaitOnFence() CPU blocks until fence reached. Use case: read GPU data (readback texture = must wait for GPU to finish rendering first). Alternative: CommandBuffer.RequestAsyncReadback() (non-blocking readback, CPU continues, GPU signals when ready). Avoiding: expensive fences (every frame = stalls CPU, defeats async execution), use sparingly (end-of-frame physics updates that need GPU data).

## Key Concepts

- **Command Buffers**: Records GPU operations (Draw, Dispatch, SetRenderTarget, Copy). Process: CPU creates buffer, records commands (buffer.DrawRenderer(renderer, material), buffer.SetGlobalTexture), submits buffer (Graphics.ExecuteCommandBuffer), GPU processes sequentially. Reuse: if commands identical each frame (static scene), reuse same buffer (recreate only when scene changes), saves CPU time (no buffer generation overhead). Dynamic: if commands change (animated objects, LOD changes), record new buffer each frame or use dynamic batching (automatic). Overhead: CommandBuffer generation = CPU cost, minimize by batching commands (fewer submissions = fewer function calls).
- **Render Texture Readback**: CPU requests GPU data (render result, compute buffer), GPU must finish writing before CPU reads (synchronization required). Methods: GPU.WaitOnFence() (CPU blocks until GPU fence, simple but stalls), CommandBuffer.RequestAsyncReadback() (CPU continues, GPU signals when ready, non-blocking). Async preferred (prevents frame stalls, queues readback for later frame). Use case: screenshot, compute result copy-back, physics based on render target data.
- **GPU Pipeline Stalls**: CPU waits for GPU (GPU.Flush waits for GPU queue empty, CPU blocks, zero throughput that frame). Causes: fence blocking, readback synchronous, SetPass exceeds buffer size (GPU command queue overflows = auto-stall). Symptom: frame rate drops (sudden spikes if stall every N frames). Avoidance: async readback (avoid blocking), fence usage sparingly (end-of-frame only if necessary), profile GPU activity (avoid queue overflow).
- **Frame Latency**: Frames in flight (CPU ahead of GPU). Default: 1-3 frames latency (CPU frame N submitted, GPU frame N-1 executing, display frame N-2 visible). Increasing latency (more frames ahead = higher GPU utilization but more input lag, acceptable for offline games, bad for multiplayer/VR). Decreasing latency (fewer frames = lower lag but lower GPU utilization, VR/competitive require minimal lag). Configuration: Application.targetFrameRate + GPU.maxFrameLatency (cap max frames in flight, newer APIs support dynamic adjustment).
- **GPU Utilization**: Percentage of GPU time spent doing useful work (opposite = stalls/idle). Target: >90% utilization (GPU consistently busy, no idle bubbles). Measurement: Profiler GPU Usage tab (shows GPU time per pass), if low = either too little work or stalls occurring. Optimization: increase workload (higher resolution, more effects, more objects), or reduce CPU latency (fewer frames in flight = tighter synchronization, GPU less starved).

## Best Practices

**Command Buffer Optimization:**
- Reuse buffers: if static scene (fixed geometry, no LOD changes), record CommandBuffer once at startup, execute every frame (Graphics.ExecuteCommandBuffer). Saves CPU time (no buffer generation). Update: only regenerate if scene changes (object added/removed, LOD triggered).
- Batching: group related commands (set material, set properties, draw calls) into single buffer, execute once instead of multiple submissions (fewer function calls, lower CPU overhead). Example: setup pass (render depth only = single buffer), then main pass (render final colors = another buffer), not interleaved.
- Avoid Graphics.ExecuteCommandBuffer inside loops (call once per frame, not N times/frame if N objects). If per-object commands needed (different properties), consider compute shader or indirect rendering (GPU decides workload, CPU just submits one command).

**Synchronization Points:**
- Minimize fences: avoid GPU.WaitOnFence() in game loop (blocks frame, stalls), use only for critical data (physics readback end-of-frame, screenshot capture). Async preferred: CommandBuffer.RequestAsyncReadback() queues operation, continues without blocking.
- Frame latency tuning: GPU.maxFrameLatency = 1 (minimize lag, single frame in flight) for VR/competitive, = 3 (maximize GPU utilization) for offline games. Balance: responsive input vs GPU utilization.
- Async operations: compute results (readback), texture reads (wait for GPU write complete before CPU reads), or rendering dependencies (render target written by GPU, read by next draw pass = GPU internally synchronizes).

**GPU Utilization Optimization:**
- Target utilization: Profiler GPU Usage tab (aim >90% busy, <10% stalls), if low, profile to find cause (insufficient work = increase settings, or stalls = reduce latency/fences).
- Work distribution: spread workload evenly across frame (don't front-load all heavy work early = GPU idle late), or load-balance across frames (dynamic objects render N per frame, spread across frames = smoother GPU load).
- Multiple rendering passes: if one pass cheap (shadow pass), another expensive (forward rendering), interleave on GPU (shadow then forward = balanced GPU load, not spike-spike pattern).

**Platform-Specific:**
- **PC**: Vulkan/DirectX control GPU latency (queueing behavior, flush behavior), tune per API (DirectX = traditional, Vulkan = explicit command buffering, more control).
- **Consoles**: Fixed GPU (PS5/Xbox Series X optimized for specific workload patterns), tune latency per console (PS5 accepts 2-frame latency, Xbox one adapts).
- **Mobile**: GPU queues small (Mali = limited command buffer, Adreno = larger), avoid overflow (submit commands frequently if large batches, test on low-end device for queue overflow).

## Common Pitfalls

**Stall on ReadBack**: Developer reads GPU data (`Texture2D.ReadPixels()` after rendering), CPU blocks until GPU finishes current frame (stalls). Symptom: frame rate drops periodically (every N frames = screenshot taken, frame drops to 1ms that frame). Solution: async readback (`CommandBuffer.RequestAsyncReadback()` queues request, fetches next frame without stall), or read previous frame data (delay one frame = no synchronization).

**CommandBuffer Memory Bloat**: Developer appends to CommandBuffer repeatedly (calls buffer.DrawRenderer in loop, once per frame = buffer grows unbounded). Memory: buffer never cleared = allocations accumulate, RAM consumption = megabytes per minute. Symptom: GC spike (garbage collection of old buffers), memory pressure. Solution: reuse buffer (Clear() method, resets buffer capacity without deallocation), or dispose and recreate (buffer.Release() deallocates if not needed).

**GPU Queue Overflow**: Developer submits too many commands in batch (mobile GPU = limited queue, overflows), GPU stalls waiting for queue space. Symptom: unexpected frame spikes (Profiler shows GPU stall but no obvious culprit). Solution: profile on low-end device (older mobile = smaller queue, test reveals limit), split submissions (submit commands in batches, CPU sleeps briefly between, GPU queue drains), or reduce draw calls.

## Tools & Workflow

**CommandBuffer Profiling**: Profiler GPU Usage tab (shows per-pass timing), if specific pass slow, profile with Frame Debugger (Window > Analysis > Frame Debugger, identify bottleneck = which commands expensive). Optimize: reduce draw calls (batching), simplify shaders, or cache buffer (reuse instead of regenerate).

**Async ReadBack Example**:
```csharp
AsyncGPUReadbackRequest request = AsyncGPUReadback.Request(targetTexture);
// CPU continues, no stall

if (request.done) {
    if (request.hasError) Debug.Log("ReadBack failed");
    else {
        NativeArray<Color32> data = request.GetData<Color32>();
        // Process data
        request.Dispose();
    }
}
```

**GPU Utilization Check**: Profiler > GPU Usage tab, monitor GPU Time (should be consistent frame-to-frame = good utilization, spikes = stalls or work variance). Target: 90%+ utilization (GPU not idle), <10% overhead (sync/stall).

**Frame Latency Control**: `GPU.maxFrameLatency = targetFrames;` (1 = tight sync, 3 = maximum throughput). For VR: set to 1 (minimize lag, single frame in flight). For offline: set to 3 (GPU remains busy).

## Related Topics

- [21.1 Frame Pacing](21-01-Frame-Pacing.md) - Frame timing, frame budgets
- [06.2 GPU Architecture](06-02-GPU-Architecture.md) - GPU pipeline, queues
- [14.2 GPU Profiling](14-02-GPU-Profiling.md) - GPU bottleneck identification
- [11.1 Rendering Architecture](11-01-Rendering-Architecture.md) - Rendering pipeline

---

[Previous: 21.2 Console Frame Pacing](21-02-Console-Frame-Pacing.md) | [Next: Chapter 22 →](../chapters/22-HDR-And-Color-Management.md)
