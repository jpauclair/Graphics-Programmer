# 23.1 Unity Jobs System

[← Back to Chapter 23](../chapters/23-Multithreading-And-Jobs-System.md) | [Main Index](../README.md)

Unity Jobs System enables safe multithreading (physics, animation, pathfinding on worker threads). Jobs (small work units) scheduled on thread pool (automatically distributes across CPU cores). Safety: job burst compilation (native code generation), dependencies (jobs execute in order), NativeCollections (thread-safe data containers). Benefit: 2-4x speedup on multi-core CPUs (4-8 cores typical = 3-6x speedup theoretically, practice = 2-4x due to overhead).

---

## Overview

Job System Purpose: parallelize CPU-bound work (physics simulation, animation blending, pathfinding), distribute across available cores (4-8 on desktop, 2-4 on mobile), reduce main thread load (renders frame while jobs process in parallel). Requirements: Jobs must be thread-safe (no shared GameObject/transform access, use NativeCollections), small granularity (overhead = ~100µs per job, job must take >100µs to justify overhead).

Job Scheduling: Main thread schedules jobs (IJob interface, Execute() method), Job System queues on worker threads, returns handle (JobHandle), main thread waits if needed (jobHandle.Complete() blocks until job finishes). Dependencies: specify job order (job B depends on job A = B waits for A complete), enables pipelining (job C starts while job B waiting). No GPU work in jobs (GPU operations must happen on main thread = CommandBuffer).

## Key Concepts

- **IJob Interface**: Single-threaded job (runs once per schedule). Structure: struct implementing IJob, Execute() method (job work), NativeCollections for data. Example: `struct ClearData : IJob { public NativeArray<float> data; public void Execute() { for(int i=0; i<data.Length; i++) data[i] = 0; } }`. Schedule: `var job = new ClearData { data = nativeArray }; JobHandle handle = job.Schedule();`. Benefit: simplicity (no thread management, Job System handles), safety (burst compiled = native code, no GC).
- **IJobParallelFor**: Parallel job (runs Execute() multiple times, once per element). Structure: struct implementing IJobParallelFor, NativeArray to iterate, Execute(int index) processes element at index. Job System automatically distributes iterations across cores (4 cores = 4 iterations in parallel). Example: `struct UpdatePositions : IJobParallelFor { public NativeArray<float3> positions; public float deltaTime; public void Execute(int i) { positions[i] += velocity[i] * deltaTime; } }`. Schedule: `job.ScheduleParallel(arrayLength, batchSize: 64, dependsOn)` (64 = iterations per thread before switching, tune for performance). Speedup: nearly linear with cores (4 cores = 3.5x typical, 8 cores = 6-7x typical, due to cache/scheduling overhead).
- **NativeCollections**: Thread-safe data containers (NativeArray, NativeList, NativeHashMap). Features: no garbage collection (pre-allocated, fixed size), owned by native memory (persistent across frames if needed), Dispose() required (manual cleanup). Thread-safe: multiple threads can read-only access simultaneously, single writer + multiple readers supported (via read-only attributes). Example: `NativeArray<float> data = new NativeArray<float>(1000, Allocator.Persistent); /* ... job processes ... */ data.Dispose();`. Allocators: Temporary (frame-scoped = auto freed), Persistent (manual Dispose), TempJob (job-scoped).
- **Job Dependencies**: JobHandle system (jobHandle = job.Schedule(deps); job depends on deps completing first). Chaining: `jobA.Schedule()` returns handleA, `jobB.Schedule(handleA)` makes B depend on A. Benefits: pipelining (multiple jobs queued, schedule as dependencies ready), no deadlocks (declarative = system ensures correct order), GPU/CPU sync (return CPU fence before GPU work = implicit sync). Example: jobPhysics.Schedule(), then jobAnimation.Schedule(jobPhysics), then jobRender.Schedule(jobAnimation) = serialized pipeline.
- **Burst Compiler**: JIT (Just-In-Time) compilation of job code to native machine code. Process: job struct submitted, Burst analyzes, generates optimized LLVM IR, compiles to native (x64, ARM64, WebGL), links into game. Benefits: near-C++ performance (same machine code as hand-written C), auto-vectorization (SIMD optimization = 4x+ speedup on operations like float3 math), no allocation (IL2CPP compiles managed C#, Burst compiles job only). Requirement: job code must be burst-compatible (no dynamic allocation, reflection, virtual functions = use blittable structs, static math functions). Drawback: compilation overhead (first job = slow, 1-2 sec, caching across sessions).

## Best Practices

**Job Structure Design:**
- Keep jobs small (coarse-grain = few jobs large each is slower, fine-grain = many jobs small each = overhead, balance ≈ 64-256 iterations per job).
- NativeCollections only (no managed objects, no GameObject references = not thread-safe, causes crashes if accessed).
- Read-only inputs (mark with [ReadOnly], multiple jobs can read simultaneously), write output (single writer, safe = Job System enforces).
- Burst-compatible (structs, blittable types, no allocations, no reflection). Test: Enable Burst (Jobs > Burst > Enable), compile check (Jobs > Burst > Compile).

**Dependency Management:**
- Chain jobs (jobA complete before jobB starts), not DAG (Job System doesn't support complex graphs, declare linear chains). Reorder if needed (jobA -> jobB -> jobC serialized, or run independent jobs simultaneously = new jobA_alt in parallel with jobA).
- Complete strategically (jobHandle.Complete() blocks main thread = do when necessary = before render only, not every frame = defeats parallelism). Schedule all jobs first (full parallel work submitted), Complete at end (GPU rendering waits, jobs finish in background).

**Performance Optimization:**
- Profile: Profiler > Jobs tab (shows job times, dependencies, scheduling overhead). Target: total job time < frame budget (e.g., 16ms at 60 FPS, jobs should complete within budget).
- Batch size tuning: IJobParallelFor batchSize parameter (64-256 typical, measure performance = smaller batch = more overhead, larger = poor load balancing). Tune per job type (physics = 128, animation = 64).
- Reduce job count (schedule 3 large jobs beats 300 tiny jobs = overhead dominates).

**Platform-Specific:**
- **PC**: 4-16 cores typical, full job parallelism. Burst native compilation. Target: all jobs complete <10ms (multithreading budget).
- **Consoles**: 8 cores (PS5/Series X), fixed hardware. Burst compiled. Target: jobs complete <5ms (frame budget 16.67ms at 60 FPS).
- **Mobile**: 2-4 cores (lower-end), 6-8 cores (high-end). Burst compiled. Target: jobs <2ms (thermal budget tight, limited cores).

## Common Pitfalls

**Job Accessing GameObject**: Developer writes job (accesses `transform.position = newPos`), compiles with Burst, crashes at runtime (GameObject access not thread-safe = race condition = corruption). Symptom: random crashes, undefined behavior, sometimes crashes sometimes doesn't. Solution: use NativeArray only (schedule job with native array of positions, job writes to array, main thread applies to GameObjects after Complete()).

**No Job Completion**: Developer schedules job (forgets Complete()), moves on, data not ready when needed (job still executing = stale data). Symptom: physics/animation updates delayed (job queue builds up = increasing latency). Solution: Complete() before using data, or structure code to not need data immediately (schedule job early, wait until actually needed).

**Job Too Large**: Developer puts entire level physics into single job (1000 bodies, 100ms to simulate), Burst can't optimize (too complex), no speedup. Symptom: job takes longer than sequential code = defeats purpose of multithreading. Solution: split job (schedule multiple smaller jobs in parallel = 8 jobs 12.5ms each on 8 cores = 12.5ms total vs 100ms sequential).

## Tools & Workflow

**Job Definition**: 
```csharp
struct PhysicsUpdateJob : IJobParallelFor {
    public NativeArray<float3> positions;
    public NativeArray<float3> velocities;
    public float deltaTime;
    
    public void Execute(int index) {
        positions[index] += velocities[index] * deltaTime;
    }
}

// Schedule
var job = new PhysicsUpdateJob {
    positions = posArray,
    velocities = velArray,
    deltaTime = Time.deltaTime
};
JobHandle handle = job.ScheduleParallel(posArray.Length, 64, JobHandle.CombineDependencies(dep1, dep2));
handle.Complete(); // Wait if needed
```

**Profiler Jobs Tab**: Window > Analysis > Profiler > Jobs tab (shows scheduled jobs, dependencies, execution time, wait time). Identify long jobs (complete early), or scheduling overhead (many small jobs = high overhead).

**Burst Compiler**: Jobs > Burst > Enable (enable compilation), Compile (compile now to check errors), Open Compiler Log (see issues). Target: all jobs burst-compatible (no errors = green checkmark).

**Dependency Chains**: Visualize with JobHandle.CombineDependencies(handle1, handle2) (merges multiple dependencies into single handle = common pattern for multiple input jobs).

## Related Topics

- [23.2 Burst Compiler](23-02-Burst-Compiler.md) - Job compilation, optimization
- [23.3 ECS and DOTS](23-03-ECS-And-DOTS.md) - Data-oriented architecture
- [23.4 Thread-Safe Graphics Operations](23-04-Thread-Safe-Graphics-Operations.md) - Graphics + threading
- [14.1 Profiling Fundamentals](14-01-Profiling-Fundamentals.md) - Job profiling

---

[Previous: Chapter 22](../chapters/22-HDR-And-Color-Management.md) | [Next: 23.2 Burst Compiler →](23-02-Burst-Compiler.md)
