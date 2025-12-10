# 5.3 Memory Optimization

[← Back to Chapter 5](../chapters/05-Performance-Optimization.md) | [Main Index](../README.md)

Memory optimization reduces VRAM and RAM usage, preventing crashes, hitches, and performance degradation on memory-constrained platforms like Switch and last-gen consoles.

---

## Overview

Consoles have fixed memory budgets: Switch (3GB usable), Xbox Series S (8GB), PlayStation 4 (7GB for games). Exceeding budgets causes OS intervention—texture evictions, mesh unloading, or crashes. Even within budget, poor memory management creates hitches from garbage collection (10-30ms spikes) or streaming stalls (20-100ms loading textures mid-frame).

Memory optimization targets three areas: VRAM (textures, meshes, render targets), managed heap (C# objects), and native memory (Unity engine allocations). Textures dominate VRAM—a typical game uses 60-80% of VRAM for textures. Compressing textures, reducing resolutions, and streaming via Mipmap Streaming recover gigabytes. Mesh memory is smaller (<10% typically) but still benefits from compression and LOD.

Shader variants inflate builds and runtime memory. A Shader Graph with 10 keywords generates 1,024 variants (2¹⁰), consuming 100-500MB of shader memory. Unused variants waste memory—strip them via build preprocessors or manual variant control. Garbage collection spikes are avoided via object pooling (pre-allocate, reuse objects instead of allocating/destroying per frame).

## Key Concepts

- **VRAM Budget**: GPU memory for textures, meshes, and render targets. Consoles have 7-16GB total (shared with CPU). Target <8GB VRAM on current-gen, <4GB on Switch, <3GB on last-gen for safety margins.
- **Managed Heap**: C# object memory managed by garbage collector. Excessive allocations fragment heap, causing GC spikes (10-30ms). Use object pooling and avoid per-frame allocations (new, closures, boxing).
- **Native Memory**: Unity engine allocations (textures, meshes, particles, audio). Not garbage collected. Leaks cause memory growth over time until crash. Profile with Memory Profiler.
- **Shader Variants**: Compiled shader permutations from keywords. Each variant costs memory (10KB-1MB). Thousands of variants consume gigabytes. Strip unused variants aggressively.
- **Texture Streaming**: Loading/unloading texture mipmaps based on visibility. Reduces VRAM by 50-70% in open-world games. Unity Mipmap Streaming or custom streaming systems.

## Best Practices

**Texture Memory Reduction:**
- Compress all textures: BC7 (PC), ASTC (mobile/Switch), BC6H (HDR). Reduces size by 4-6x vs uncompressed. Quality loss is minimal on final hardware.
- Set max texture size per platform: 2K for current-gen, 1K for Switch/last-gen, 512-1K for distant objects. Override in import settings per texture.
- Use Mipmap Streaming (Edit > Project Settings > Quality > Texture Streaming). Loads only visible mips, reducing VRAM by 50-70%. Set budget to 70% of platform VRAM.
- Reduce texture resolution for non-hero assets: UI (512x512), small props (512x512), distant foliage (256x256). Reserve 2K-4K for hero characters and key environment pieces.
- Audit textures via Memory Profiler. Sort by size, identify 4K textures on small objects. Artists often import at max resolution—catches oversights.

**Mesh Optimization:**
- Enable mesh compression in import settings ("Mesh Compression: High"). Reduces size by 50-70% with minimal quality loss (quantized positions/normals).
- Use LOD aggressively. LOD0 (close): full detail. LOD1 (10-50m): 50% triangles. LOD2 (50-100m): 25% triangles. LOD3 (100m+): 10% triangles or billboards.
- Disable Read/Write Enabled in mesh import settings unless CPU access needed (procedural meshes, collision). Doubles mesh memory if enabled unnecessarily.
- Combine static meshes via static batching or mesh combining tools. Reduces mesh count, improving cache efficiency and draw call overhead.

**Shader Variant Management:**
- Use `#pragma multi_compile` sparingly. 10 keywords = 1,024 variants. Use `#pragma shader_feature` for features disabled by default (variants stripped if unused).
- Implement shader variant collection prebaking. Use Edit > Project Settings > Graphics > Shader Stripping to limit platforms/tiers.
- Strip unused variants in builds via `IPreprocessShaders` callback. Analyze used materials, strip variants not referenced anywhere. Saves 50-80% shader memory.
- Avoid global keywords (`#pragma multi_compile FEATURE`). Use local keywords (`#pragma multi_compile _ FEATURE`) to limit impact to single shader instead of all shaders.

**Asset Loading and Streaming:**
- Use AssetBundles for large assets (textures, meshes, audio). Load on-demand, unload when off-screen or distant. Reduces baseline memory by 30-50% in open-world games.
- Implement texture streaming via Mipmap Streaming or custom system. Stream high-res mips only when player approaches. Distant objects use low-res mips (1/16 memory).
- Preload critical assets (player character, UI, common props) during level load. Stream non-critical assets (distant buildings, optional content) asynchronously.
- Unload unused assets via `Resources.UnloadUnusedAssets()` after scene transitions. Reclaims memory from previous scene. Call during load screens (hides 0.5-2s cost).

**Memory Pooling:**
- Pool frequently created/destroyed objects: bullets, particles, UI elements, temporary buffers. Pre-allocate 100-1,000 instances, reuse instead of new/destroy.
- Use Unity's `ObjectPool<T>` (Unity 2021+) or implement custom pooling. Eliminates per-frame allocations, preventing GC spikes.
- Pool managed arrays and lists. Clearing list (`list.Clear()`) reuses backing array. Creating new list allocates heap memory—0.5-5KB per allocation.
- Profile with Profiler > Memory module. "GC Alloc" shows per-frame allocations. Target <50KB per frame. Higher values cause frequent GC (5-10ms spikes).

**Platform-Specific:**
- **Switch**: 3GB usable memory total. Aim for <1.5GB VRAM, <1GB managed heap, <0.5GB native. Aggressive compression (ASTC 8x8), low resolutions (1K textures max).
- **Xbox Series S**: 8GB total memory. Target <5GB VRAM, <2GB CPU. Reduce texture resolution vs Series X (1K instead of 2K, 1440p render targets instead of 4K).
- **PlayStation 4**: 7GB for games (1GB OS reserve). Target <3GB VRAM, <2GB CPU. Use texture streaming aggressively—baseline memory fits in 5GB, streaming adds detail.
- **Mobile**: 2-4GB total memory. Target <1GB VRAM on low-end, <2GB on high-end. ASTC compression mandatory, 512-1K textures typical.

## Common Pitfalls

**Read/Write Enabled on All Meshes**: Import settings default to "Read/Write Enabled: false". Developer enables globally for one procedural mesh script, forgetting to disable per-mesh. Now every mesh occupies double memory (GPU VRAM + CPU RAM copy). Symptom: Memory usage doubles unexpectedly. Solution: Disable Read/Write except for specific procedural or physics meshes needing CPU access.

**Shader Variant Explosion**: Shader Graph with 12 keywords (fog, shadows, reflections, AO, etc.) generates 4,096 variants. With 10 Shader Graphs, that's 40,960 variants consuming 500MB-2GB shader memory. Symptom: Build size bloats, memory profiler shows huge shader memory. Solution: Reduce keywords to 6-8 max (64-256 variants), use shader feature keywords, or implement manual variant stripping.

**Ignoring GC Allocations**: Per-frame allocations from `FindObjectsOfType`, `GetComponent`, string concatenation, or LINQ queries create 1-10KB garbage per frame. At 60fps, that's 60-600KB/sec. GC triggers every 2-5 seconds, causing 10-20ms spike. Symptom: Frame rate hitches every few seconds. Solution: Cache references, use object pooling, avoid per-frame allocations. Profile with "GC Alloc" in Unity Profiler.

## Tools & Workflow

**Unity Memory Profiler** (Package Manager > Memory Profiler): Capture snapshots, visualize memory allocations. TreeMap view shows largest consumers. "Managed Heap" reveals C# fragmentation. "Native Memory" shows engine allocations. Essential for memory optimization.

**Texture Memory Report**: Build Settings > Player Settings > Other Settings > "Generate Texture Memory Report". Produces CSV with per-texture memory usage. Import to spreadsheet, sort by size, identify optimization targets.

**Frame Debugger**: Shows active render targets and their memory usage. Identify oversized shadow maps (2K instead of 1K) or render targets (4K G-buffer when 1080p is sufficient).

**Profiler > Memory Module**: Real-time memory usage graph. "Total Allocated" shows overall memory. "Texture Memory", "Mesh Memory" break down VRAM. "GC Allocated" shows managed heap allocations per frame.

**Shader Variant Collection**: Edit > Project Settings > Graphics > Preloaded Shaders. Add used shaders manually to prevent runtime compilation (hitches). Use `ShaderVariantCollection` to track used variants, strip unused.

## Related Topics

- [5.4 Bandwidth Optimization](05-04-Bandwidth-Optimization.md) - Reducing memory bandwidth usage
- [13.1 Texture Formats and Compression](08-01-Texture-Compression-Formats.md) - Texture compression techniques
- [7.2 Mesh Optimization](07-02-Mesh-Optimization.md) - Mesh memory reduction
- [2.1 GPU Memory Hierarchy](02-01-GPU-Memory-Hierarchy.md) - Understanding GPU memory systems

---

[← Previous: 5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) | [Next: 5.4 Bandwidth Optimization →](05-04-Bandwidth-Optimization.md)
