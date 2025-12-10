# 6.3 Memory Bottlenecks

[← Back to Chapter 6](../chapters/06-Bottleneck-Identification.md) | [Main Index](../README.md)

Identifying and resolving memory bottlenecks prevents crashes, hitches, and performance degradation from exhausted VRAM or RAM budgets.

---

## Overview

Memory bottlenecks manifest as crashes (out-of-memory), hitches (garbage collection spikes, texture streaming stalls), or degraded performance (texture thrashing, paging to system memory). Consoles have fixed memory budgets—Switch (3GB usable), PS4 (7GB), Xbox Series S (8GB)—and exceeding them causes OS intervention or termination. Even within budget, poor memory management creates frame time spikes: 10-30ms GC collections, 50-200ms asset loading stalls, or constant texture eviction/reloading.

Memory bottlenecks have three primary causes: VRAM exhaustion (textures, render targets, meshes), managed heap issues (C# garbage collection, fragmentation), and asset loading problems (synchronous loads blocking rendering, streaming failures). Profiling reveals symptoms: Memory Profiler shows allocations exceeding budgets, Profiler shows GC spikes, and platform tools (PIX, Razor) show texture thrashing or memory warnings.

Resolving memory bottlenecks requires measurement first. Establish baseline memory usage per platform (target 70-80% of max to leave headroom), profile peak usage (loading screens, dense areas), and identify culprits (which textures/meshes consume most memory). Then apply targeted solutions: compression, streaming, pooling, or algorithmic changes (reduce render target resolution, lower texture quality settings).

## Key Concepts

- **VRAM Exhaustion**: GPU memory (textures, meshes, render targets) exceeds capacity. Causes texture evictions (reloading from disk, 50-200ms hitches) or driver fallbacks (paging to system RAM, 10-50x slower access).
- **Garbage Collection Spikes**: Automatic memory reclamation in managed languages (C#). Triggers when heap fills, pausing all threads for 10-100ms to compact memory. Causes frame rate hitches.
- **Memory Fragmentation**: Free memory split into small non-contiguous blocks. Prevents large allocations despite sufficient total free space. Causes allocation failures or heap expansion (triggering GC).
- **Texture Thrashing**: Repeatedly loading/unloading textures due to insufficient VRAM. GPU requests texture, loads from disk (50ms), renders, evicts due to lack of space, then requests again next frame.
- **Synchronous Loading**: Blocking main thread while loading assets from disk. Freezes rendering for 50-500ms. Visible as hitches when entering new areas or spawning objects.

## Best Practices

**VRAM Budget Management:**
- Establish per-platform VRAM budgets: Switch <1.5GB, PS4/Xbox One <3GB, PS5/Xbox Series X <10GB, PC varies (6-16GB typical). Target 70-80% of max for safety margin.
- Profile baseline VRAM with Memory Profiler: Capture snapshot, analyze "Native Memory > Texture" and "Mesh". Sort by size, identify largest consumers.
- Compress textures aggressively: BC7/ASTC reduce size by 4-6x vs uncompressed. Every uncompressed 2K texture wastes 16MB (compressed = 2MB).
- Implement texture streaming (Mipmap Streaming or custom). Loads only visible mips, reducing VRAM by 50-70%. Essential for open-world games.
- Lower texture quality on lower-spec platforms: 2K on current-gen, 1K on Switch/last-gen. Override in Quality Settings > Texture Quality.

**Managed Heap Optimization:**
- Minimize per-frame allocations: Avoid `new`, string concatenation, LINQ, boxing, closures in Update/FixedUpdate. Target <50KB allocations per frame.
- Use object pooling for frequent allocations: bullets, particles, temporary buffers. Pre-allocate 100-1,000 instances, reuse instead of new/destroy.
- Cache component references: `GetComponent<T>()` allocates GC memory. Call once in Start/Awake, cache result. Never call per-frame.
- Implement manual memory management for critical systems: Use unsafe code with fixed buffers, NativeArray, or Unity Collections for zero-GC paths.
- Profile with Profiler > Memory module: "GC Alloc" column shows per-frame allocations. Drill down to find offending methods.

**Garbage Collection Mitigation:**
- Increase GC incremental time slicing: Edit > Project Settings > Player > Scripting Settings > "Use Incremental GC". Spreads GC across multiple frames (5ms/frame vs 30ms spike).
- Force GC during safe moments: Call `GC.Collect()` during loading screens or pauses. Prevents mid-gameplay spikes.
- Reduce heap size growth: Avoid large allocations (>1MB). Large allocations trigger immediate GC. Split into smaller chunks or stream data.
- Use scripting backend optimizations: IL2CPP has lower GC overhead than Mono. Xbox/PlayStation require IL2CPP (faster, smaller heaps).

**Asset Loading Optimization:**
- Use async loading for all runtime assets: `Resources.LoadAsync`, `AssetBundle.LoadAssetAsync`, `Addressables.LoadAssetAsync`. Never block main thread.
- Preload critical assets during level load: Player character, weapons, UI, common particles. Everything else streams asynchronously.
- Implement asset streaming system: Load assets as player approaches areas. Unload when distant (>100m). Maintains memory budget while providing detail.
- Use Addressables for automatic memory management: Reference counting, dependency tracking, automatic unloading when unused. Eliminates manual tracking.
- Compress AssetBundles with LZ4: Fast decompression (<5ms) vs LZMA (50-200ms). Trade file size (30% larger) for loading performance.

**Platform-Specific:**
- **Switch**: 3GB total memory (1.5GB VRAM target, 1GB managed heap target). Aggressive texture streaming mandatory. Compress everything ASTC 8x8. Preload cautiously.
- **PlayStation 4**: 7GB for games. Target <3GB VRAM, <2GB heap. Use Razor to monitor memory warnings. Sony OS kills apps exceeding 7GB (no grace period).
- **Xbox Series S**: 8GB total (vs 16GB Series X). Reduce texture resolution 30-50% vs Series X. Lower render target resolution (1440p vs 4K). Critical memory optimization target.
- **PC**: Varies widely (6-32GB). Provide texture quality settings (Low/Medium/High/Ultra). Low = 1K textures, Ultra = 4K. Essential for min-spec support.

## Common Pitfalls

**Texture Quality Settings Ignored**: Import textures at 4K, assume Quality Settings will reduce resolution. Unity's Texture Quality setting only affects mipmaps (shifts mip level), not base resolution. 4K texture still occupies 64MB even on "Low" quality. Symptom: VRAM usage identical across quality settings. Solution: Create platform-specific texture overrides (AssetImporter API) or use Addressables variants per platform.

**Untracked Native Allocations**: Developer monitors managed heap (C# objects) but ignores native memory (textures, meshes, particles, audio). Managed heap stable at 500MB, but native memory grows from 2GB to 8GB over 30 minutes, causing crash. Symptom: No GC issues, but memory steadily increases until OOM crash. Solution: Profile native memory with Memory Profiler, track allocations per category (Texture, Mesh, Audio), identify leaks.

**Synchronous Asset Loading in Update**: Developer calls `Resources.Load()` in Update when player picks up item, causing 50-200ms hitch. Happens every pickup, creating stuttery experience. Symptom: Profiler shows "Loading.ReadObject" in main thread, blocking rendering. Solution: Preload all possible pickup items during level load, or use async loading with placeholder objects until asset loads.

## Tools & Workflow

**Unity Memory Profiler** (Package Manager): Capture snapshots, compare over time. TreeMap view visualizes largest allocations. "Managed Heap" shows fragmentation. "Native Memory" shows Unity engine allocations. Compare "Before" vs "After" scene load to identify leaks.

**Profiler > Memory Module**: Real-time memory graph. "Total Allocated", "Texture Memory", "Mesh Memory" categories. "GC Allocated" shows per-frame managed allocations. "GC Alloc" column in CPU Profiler identifies allocation sources.

**Platform Profilers**: PIX (Xbox) shows memory pools (Standard vs Optimal ESRAM). Razor (PlayStation) displays memory warnings and predictions. Nintendo NNGFX shows CPU/GPU memory split. Use for platform-specific issues.

**Frame Debugger**: Displays active render targets and sizes. Identify oversized shadow maps (2K when 1K sufficient) or post-processing buffers (4K when 1080p sufficient). Right-click render target > "Show Info" for memory usage.

**Addressables Event Viewer** (Window > Asset Management > Addressables > Event Viewer): Tracks asset loading/unloading. Shows reference counts, memory usage per bundle, and load times. Debug asset streaming issues.

**Custom Memory Tracking**: Implement periodic memory logging (`Profiler.GetTotalAllocatedMemoryLong()`, `Profiler.GetMonoUsedSizeLong()`). Log every 5 seconds with timestamp, location, and active scene. Analyze logs to identify memory growth trends.

## Related Topics

- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory reduction techniques
- [6.4 Bandwidth Bottlenecks](06-04-Bandwidth-Bottlenecks.md) - Memory bandwidth issues
- [13.4 Texture Streaming](10-02-Loading-Strategies.md) - Streaming implementation details
- [2.1 GPU Memory Hierarchy](02-01-GPU-Memory-Hierarchy.md) - Understanding GPU memory

---

[← Previous: 6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) | [Next: 6.4 Bandwidth Bottlenecks →](06-04-Bandwidth-Bottlenecks.md)
