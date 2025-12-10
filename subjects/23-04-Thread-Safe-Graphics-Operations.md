# 23.4 Thread-Safe Graphics Operations

[← Back to Chapter 23](../chapters/23-Multithreading-And-Jobs-System.md) | [Main Index](../README.md)

Graphics operations (Draw, SetRenderTarget, Dispatch) not thread-safe (GPU has single command queue = must execute serially from main thread). Jobs run worker threads (can't call Graphics.DrawMesh() = error). Solution: record GPU commands in jobs via CommandBuffer (defer execution to main thread), or pre-build command buffers. Async readback (non-blocking GPU data reads = don't stall frame).

---

## Overview

Graphics Thread Safety: GPU single command stream (not true parallelism like CPU). Rendering: main thread queues commands (Draw, Dispatch, Copy), GPU executes (asynchronously). Jobs: can't queue graphics commands (no thread-safe graphics API). Solution: CommandBuffer (record commands, execute on main thread), or deferred rendering (pre-built command buffers = no overhead, static scene only).

Common Pattern: Physics/Animation jobs run parallel (worker threads), produce new positions/transforms, main thread collects results (Complete() job), rebuilds render data (CommandBuffer or direct draw calls), executes graphics (GPU renders). Async operations: readback (copy GPU data to CPU, non-blocking = don't wait), queries (GPU performance counters, occlusion, primitive count = don't block).

## Key Concepts

- **CommandBuffer Recording in Jobs**: Jobs can't directly call Graphics.DrawMesh (not thread-safe = GPU command queue single, concurrent writes = corruption). Solution: CommandBuffer objects thread-safe (create per-job, record GPU commands, submit on main thread). Implementation: job struct has CommandBuffer field (mark with [NativeDisableParallelForRestriction] = override safety checks), job.Execute() records commands (buffer.DrawMesh, buffer.SetGlobalTexture), main thread submits (Graphics.ExecuteCommandBuffer). Limitation: only specific operations (Draw, SetGlobalMatrix, Copy, Compute Dispatch) thread-safe in buffer, most are. Avoid: GetNativeRenderingContext() = not thread-safe, use buffer only.
- **Command Buffer Submission**: Record on any thread (CommandBuffer = thread-safe data structure), submit on main thread only (Graphics.ExecuteCommandBuffer = queues to GPU = must be main). Pattern: job records buffer, main thread executes (ExecuteCommandBuffer in OnRenderObject or LateUpdate). Multiple buffers: record 10 jobs with buffers, main thread submits all 10 (3 ExecuteCommandBuffer calls or combine = Graphics.ExecuteCommandBuffer(combined)). Performance: ExecuteCommandBuffer = fast (C++ binding = direct GPU queue), no overhead beyond actual draw call cost.
- **Async GPU Readback**: Non-blocking GPU-to-CPU data copy (screenshot, compute result, physics data). Request: AsyncGPUReadback.Request(texture) = queues readback, returns immediately (doesn't wait), later (next frame or when done) .done = true, GetData() fetches result. Benefit: GPU writes data, CPU continues frame, GPU fence signals when ready = no stall. Drawback: latency (result ready next frame = delayed 1-2 frames, acceptable for non-critical). Use: compute output data, readback to CPU for analysis, physics based on GPU data.
- **DrawMesh from Jobs**: Can't call Graphics.DrawMesh directly (not thread-safe). Workaround: job populates matrix array (Position, Rotation data), main thread calls Graphics.DrawMeshInstanced(mesh, material, matrices, count) = single draw call = GPU renders many instances. Or: CommandBuffer.DrawMesh per instance = multiple draw calls = slower. Preference: batch all instances = single draw call (GPU efficient).

## Best Practices

**CommandBuffer in Jobs**: Create per-job or once + clear. Mark restriction: `[NativeDisableParallelForRestriction] CommandBuffer cmdBuffer;` = override safety check. Keep commands simple: Draw, SetGlobal are safe, GetNativeRenderingContext = not safe. Test: record 10 jobs with buffers, execute all, verify renders correctly.

**Async Readback Pattern**: Request early (don't immediately wait = defeats async). Later (next frame): check .done, fetch result if ready. Multiple readbacks: queue requests, track handles, fetch in order. Fallback: if synchronous needed, accept frame stall, or restructure (compute raycast GPU = async).

**Platform-Specific**: PC (full async support, CommandBuffer fully supported), Consoles (native APIs thread-safe, Unity abstracts = main thread standard), Mobile (CommandBuffer supported, async readback supported = test on target device).

## Common Pitfalls

**Graphics Call from Job**: Developer schedules job (calls Graphics.DrawMesh inside Execute() = compiler allows, runtime error = crash). Symptom: crashes with "not allowed from background thread" or graphics corruption. Solution: use CommandBuffer only (buffer.DrawMesh = thread-safe), no direct Graphics.* calls from jobs.

**Readback Stall**: Developer requests AsyncGPUReadback (doesn't wait), then immediately reads result (checks .done = false = not ready yet). Application uses stale data (delay = physics glitch). Solution: check .done = true before reading, or restructure (delay result usage 1 frame = async practical).

## Tools & Workflow

**Job with CommandBuffer**:
```csharp
struct RenderJob : IJob {
    [NativeDisableParallelForRestriction]
    public CommandBuffer buffer;
    public Mesh mesh;
    public Material material;
    
    public void Execute() {
        buffer.DrawMesh(mesh, Matrix4x4.identity, material, 0);
    }
}
```

**Async Readback**:
```csharp
AsyncGPUReadbackRequest request = AsyncGPUReadback.Request(renderTexture);
if (request.done) {
    NativeArray<Color32> data = request.GetData<Color32>();
    // Use data
}
```

## Related Topics

- [21.4 GPU Synchronization](21-04-GPU-Synchronization.md) - Command buffers, fences
- [23.1 Unity Jobs System](23-01-Unity-Jobs-System.md) - Job scheduling
- [11.1 Rendering Architecture](11-01-Material-System-Architecture.md) - Render pipeline
- [14.2 GPU Profiling](04-01-PC-Profiling-Tools.md) - Graphics profiling

---

[Previous: 23.3 ECS and DOTS](23-03-ECS-And-DOTS.md) | [Next: Chapter 24 →](../chapters/24-Mobile-And-Cross-Platform.md)
