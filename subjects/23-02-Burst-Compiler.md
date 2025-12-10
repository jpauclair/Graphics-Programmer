# 23.2 Burst Compiler

[← Back to Chapter 23](../chapters/23-Multithreading-And-Jobs-System.md) | [Main Index](../README.md)

Burst Compiler converts C# job code to optimized native machine code (LLVM IR = LLVM intermediate representation, compiled to x64/ARM64 backends). Benefits: near-C++ performance (5-20x vs managed C#), auto-vectorization (SIMD = 4-8x for data-parallel operations like float3 math), minimal overhead. Trade-off: burst-compatible restrictions (no allocations, reflection, managed objects = simpler code required). Essential for performance-critical jobs (physics, animation, pathfinding).

---

## Overview

Burst JIT Compilation: Job code submitted (IJob/IJobParallelFor), Burst analyzes (checks compatibility), generates LLVM IR (low-level intermediate representation = platform-independent), compiles to native (x64, ARM64, WebGL), links to game executable. Timing: first job compile = 1-2 seconds (slow), subsequent jobs faster (cached, link-only). Performance: native code = machine instructions directly = performance near hand-written C++, auto-vectorization = compiler spots data-parallel patterns (float3 addition = SIMD add instruction = 4x throughput).

Burst-Compatible Code: restrictions ensure safe compilation. Allowed: structs, blittable types (int, float, float3, fixed arrays), static methods, math operations. Forbidden: managed allocation (new, List.Add = unsafe = compiled code might allocate = unpredictable), reflection (GetType, MethodInfo = can't compile = too dynamic), virtual functions (object polymorphism = can't know target at compile time). Why: Burst generates optimized machine code = static types required (can't generate code for unknown types), no runtime allocation = predictable memory usage (real-time safe).

## Key Concepts

- **LLVM IR (Intermediate Representation)**: Burst target intermediate language (not compiled directly to native, intermediate step). LLVM = compiler framework (used by Clang, Swift, Julia = standard IR), Burst generates LLVM, LLVM toolchain compiles to native per platform. Benefit: platform-agnostic (same LLVM = compiles to x64, ARM64, WebGL without changes), future-proof (new platforms = new LLVM backend, code unchanged).
- **Auto-Vectorization**: SIMD optimization (Single Instruction Multiple Data = one CPU instruction processes 4 floats simultaneously). Burst detects patterns: `for(i=0; i<count; i++) result[i] = a[i] + b[i];` = vectorize = 1 SIMD add per 4 iterations = 4x speedup. Typical: float3/float4 operations (3-4 component math = natural SIMD targets), loop-based algorithms. Requirement: code patterns must be obvious (simple loops, no complex control flow breaking pattern detection). Drawback: not guaranteed (depends on code structure, Burst detection success = measure performance).
- **Compatibility Restrictions**: Burst-compatible code must compile to pure native (no runtime services). Forbidden: managed allocations (new, List.Add = runtime heap allocation = unpredictable cost), reflection (runtime type inspection = can't compile statically), virtual functions (dynamic dispatch = unknown target = can't inline), async/await (runtime state machines = can't compile), unsafe features partially supported (pointers allowed if blittable, delegates not supported). Allowed: value types (structs), static methods, blittable types (int, float, fixed arrays), deterministic logic.
- **Burst Compilation Targets**: x64 (Intel/AMD CPUs, Windows/Linux/Mac), ARM64 (mobile CPUs, iOS/Android), WebGL (browser, JavaScript backend = slower), Console native (PS5/Xbox Series X ARM64). Selective: developer chooses targets in Burst settings (smaller compile = faster, remove unused targets).
- **Debug vs Release**: Debug build = managed code (slower = easier debugging = breakpoints work), Release = Burst compiled = fast = no debugging. Profile on Release (Debug shows wrong performance).

## Best Practices

**Burst Compatibility**: Always enable Burst (Jobs > Burst > Enable compiler = required for shipping). Test compatibility: enable, compile, check for errors (orange warnings = might not compile, red errors = won't compile, address issues). Restrict to blittable types (structs, int, float, float3, fixed arrays, not List or string = compile error). If need dynamic data: use NativeCollections (NativeArray, NativeHashMap = compatible), pre-allocate sizes. Avoid allocations in job Execute() (any `new` = error, allocate before job schedule = inside NativeCollections or fixed buffers).

**Performance Optimization**: Enable profiling (build Release with Burst = actual performance), profile Jobs tab (compare with managed = measure speedup, typically 5-20x). Vectorization tuning: structure code for SIMD (tight loops over float3 arrays = vectorize well, scattered memory access = poor). Measure: profile same job on/off Burst = speedup = 4-8x for data-parallel jobs. Target selectively (mobile = ARM64 only, PC = x64 + ARM64 if targeting, console = ARM64, remove WebGL if not used = faster Burst compilation).

**Platform-Specific**: PC (x64 Burst compilation standard, near-C++ performance, Release build mandatory), Consoles (ARM64 compilation, full Burst support, optimized per console), Mobile (ARM64 compilation, Burst essential = managed C# too slow), WebGL (JavaScript backend = slower, use if necessary, plan for lower performance).

## Common Pitfalls

**Allocation in Job**: Developer writes job (allocates list inside Execute() = new List<float>()), Burst compilation fails (errors = won't compile, manages to work = behaves wrong = random crashes/memory corruption). Solution: allocate before job (NativeArray = pre-allocated), pass to job, no allocations in Execute().

**Debug Build Performance**: Developer profiles game (Debug mode, without Burst = managed C#), sees slow Jobs, assumes Jobs slow. Actually: Burst compiles only in Release = Debug is intentionally slow for debugging. Solution: profile Release build (with Burst enabled), measure actual performance (typically 10-20x faster than Debug).

## Tools & Workflow

**Burst Enable**: Window > Burst > Burst Menu > Enable Compiler (checkbox, enables Burst), Disable Compilation (checkbox for debug = runs managed instead of native, ONLY for debugging). Compile: "Compile" button = compile now (check errors/warnings). Target Selection: "Burst Settings" > list of targets (x64, ARM64, WebGL, console), uncheck unused (faster compilation).

**Burst Inspector**: Window > Burst > Burst Inspector (shows assembly code for selected job). Advanced feature (review generated code = verify optimization applied).

## Related Topics

- [23.1 Unity Jobs System](23-01-Unity-Jobs-System.md) - Job scheduling, dependencies
- [23.3 ECS and DOTS](23-03-ECS-And-DOTS.md) - Data-oriented with Burst
- [14.2 GPU Profiling](14-02-GPU-Profiling.md) - GPU vs CPU profiling
- [06.1 CPU Architecture](06-01-CPU-Architecture.md) - SIMD, multi-core concepts

---

[Previous: 23.1 Unity Jobs System](23-01-Unity-Jobs-System.md) | [Next: 23.3 ECS and DOTS →](23-03-ECS-And-DOTS.md)
