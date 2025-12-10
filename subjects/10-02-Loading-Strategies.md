# 10.2 Loading Strategies

[← Back to Chapter 10](../chapters/10-Asset-Bundling-And-Streaming.md) | [Main Index](../README.md)

AssetBundle loading strategies balance memory usage, loading speed, and gameplay responsiveness through synchronous/asynchronous loading, preloading, caching, and unloading patterns.

---

## Overview

Loading strategy determines when bundles load (startup, on-demand, preload during gameplay), how they load (synchronous blocking vs asynchronous non-blocking), where they load from (local files, downloaded from server, memory-cached), and when they unload (immediately after use, on level transition, on memory pressure). Poor strategy causes hitches (synchronous loads during gameplay), memory bloat (bundles never unloaded), or slow startup (loading everything upfront).

Synchronous loading (AssetBundle.LoadFromFile) blocks main thread until complete. Fast for local bundles (<50ms for 50MB LZ4 bundle), but freezes game during load (unacceptable during gameplay). Use for initial loading screens only. Asynchronous loading (AssetBundle.LoadFromFileAsync, LoadAssetAsync) loads on background thread, main thread continues. Requires coroutine or async/await pattern (await task completion, instantiate asset). Mandatory for gameplay loading (no frame drops).

Caching strategies optimize repeated loads: in-memory caching (keep loaded bundles/assets in dictionaries, instant access but consumes memory), disk caching (Unity Caching API stores downloaded bundles locally, avoids re-downloading), and preloading (load bundles before needed, hides loading times). Unloading strategies prevent memory leaks: AssetBundle.Unload(false) unloads bundle but keeps assets, Unload(true) unloads everything. Choose based on asset lifecycle (are assets still needed?).

## Key Concepts

- **Synchronous Loading**: LoadFromFile() blocks thread until bundle loaded. Fast (<50ms for local bundles), but causes frame drops if used during gameplay. Use only in loading screens or initialization.
- **Asynchronous Loading**: LoadFromFileAsync() loads on background thread. Returns AssetBundleRequest (awaitable/coroutine). Non-blocking, no frame drops. Essential for runtime loading. Adds complexity (callback handling).
- **Preloading**: Load bundles before needed (load Level2 bundle while playing Level1). Hides loading times (seamless transitions). Requires predicting what's needed next (level progression, spawn probabilities).
- **In-Memory Caching**: Store loaded bundles/assets in Dictionary<string, AssetBundle>. Instant access on subsequent requests (no disk I/O). Consumes memory (100-500MB per loaded bundle).
- **Disk Caching**: Unity Caching API (Caching.IsVersionCached, Caching.ClearCache). Downloaded bundles cached locally. Avoids re-downloading (faster load times, saves bandwidth). Managed by Unity automatically.

## Best Practices

**Asynchronous Loading Patterns:**
- Coroutine pattern: StartCoroutine(LoadBundleAsync()). yield return request (wait for completion), then access asset. Simple, Unity-native. Suitable for straightforward loading flows.
- async/await pattern: async Task<AssetBundle> LoadBundleAsync(). await request. Modern C#, cleaner code, easier error handling. Requires Unity 2021+ (earlier versions have limited async support).
- Callback pattern: LoadFromFileAsync() + completed callback (request.completed += callback). Event-driven. Use when integrating with existing callback-based systems.
- Hybrid approach: async/await for new code, coroutines for legacy code. Gradually migrate to async/await (better maintainability).

**Preloading Strategies:**
- Level-based preloading: While playing Level1, preload Level2 bundle in background. When player reaches Level1 exit, Level2 ready instantly (seamless transition). Requires level progression knowledge.
- Predictive preloading: Analyze player behavior (moving toward door? Preload next room). Spawn tables (enemy wave incoming? Preload enemy prefabs). Hides loading behind player actions.
- Progressive preloading: Load essential bundles first (environment, player character), then progressively load optional bundles (effects, audio, optional content). Game playable quickly, full content loads in background.
- Idle preloading: Preload during low-activity periods (menus, cutscenes, idle time in hub areas). Avoids loading during action-heavy moments (no frame drops).

**Caching Implementation:**
- In-memory cache: Implement AssetBundleManager with Dictionary<string, AssetBundle> cache. LoadBundle() checks cache first (return cached bundle), else load from disk and cache. Reduces repeated disk I/O.
- Cache eviction: Implement LRU (Least Recently Used) or size-based eviction. When cache exceeds threshold (e.g., 500MB), evict oldest bundles. Prevents unbounded memory growth.
- Reference counting: Track how many assets reference each bundle. Increment on LoadAsset(), decrement on asset destruction. Unload bundle when references reach zero. Prevents premature unloading.
- Unity Caching API: Use Caching.IsVersionCached(bundleName, hash) to check if bundle cached locally. UnityWebRequestAssetBundle.GetAssetBundle() automatically uses cache (no manual management).

**Unloading Strategies:**
- Unload(false) use case: Bundle loaded, assets instantiated in scene. Assets still in use, but bundle no longer needed. Unload(false) frees bundle memory (~5-20MB overhead), keeps assets (meshes, textures, materials).
- Unload(true) use case: Leaving level, destroying all assets from bundle. Unload(true) frees everything (bundle + assets). Typical on level transitions (unload old level bundles completely).
- Delayed unloading: Don't unload immediately after LoadAsset(). Assets may reference bundle data (materials reference textures still in bundle). Unload when all assets destroyed or on level transition.
- Profiling unloading: Use Memory Profiler to verify unloading actually frees memory. Unload(false) with assets still referenced shows "leaked" memory (expected). Unload(true) should free all memory.

**Loading Priority:**
- Critical bundles first: Load player character, core gameplay bundles immediately (blocking acceptable during startup loading screen). Gameplay unplayable without these.
- Progressive loading: Load environment bundles (visible areas first), then background assets (distant LODs, optional effects), finally optional content (DLC, cosmetics). Player sees gameplay quickly.
- Thread priority: Use Application.backgroundLoadingPriority (High, Normal, Low). High = load faster (more CPU), Low = slower loading but less frame rate impact. Adjust based on situation (loading screen = High, mid-gameplay = Low).
- Time slicing: Load assets incrementally (1-2 assets per frame) instead of all at once. Avoids frame spikes. Use AssetBundle.LoadAssetAsync() + isDone polling, spread over multiple frames.

**Platform-Specific:**
- **PC**: Fast SSDs (1-5GB/s). Synchronous loading acceptable for local bundles (<50ms load time). Asynchronous loading for large bundles (>100MB). Preload aggressively (ample memory, 8-32GB typical).
- **Consoles**: Very fast internal SSDs (PS5: 5.5GB/s, Xbox Series X: 2.4GB/s). Asynchronous loading still recommended (avoid certification issues with hitches). Preload during load screens (4-5 second typical load time).
- **Switch**: Slower storage (cartridge: 100-300MB/s, eMMC: 50-100MB/s). Mandatory asynchronous loading (slow I/O causes long hitches). Smaller bundles (20-50MB) to reduce load times. Limited preloading (3GB usable memory).
- **Mobile**: Slow storage (100-200MB/s). Asynchronous loading mandatory. Aggressive time slicing (1 asset per frame max) to maintain 60fps. Minimal preloading (1-3GB memory budget).

## Common Pitfalls

**Synchronous Loading During Gameplay**: Developer uses AssetBundle.LoadFromFile() when spawning enemy (instant spawn, no async complexity). Causes 50-200ms hitch (loading bundle from disk blocks frame). Symptom: Frame drops when enemies spawn, Profiler shows LoadFromFile spike. Solution: Preload enemy bundles during level load, or use LoadFromFileAsync() with coroutine (spawn when loading complete).

**Not Unloading Bundles**: Developer loads bundles but never calls Unload(). Memory usage grows continuously (100MB per level, 1GB after 10 levels). Eventually causes crashes (out of memory). Symptom: Memory Profiler shows increasing AssetBundle count, allocated memory grows linearly. Solution: Call AssetBundle.Unload(false) when done with bundle but assets in use, Unload(true) on level transitions. Implement reference counting.

**Unloading Too Early**: Developer calls AssetBundle.Unload(true) immediately after LoadAsset(). Destroys loaded asset (prefab loses references, materials become pink). Symptom: Missing textures, broken prefabs, "Destroyed Object" errors in console. Solution: Keep bundle loaded while assets in use. Unload only on level transition or when all assets destroyed. Use Unload(false) if unsure.

**No Caching Implementation**: Developer loads same bundle repeatedly (load CharactersBundle every time spawning character). Each load hits disk (50-100ms), wastes I/O bandwidth. Symptom: Profiler shows repeated LoadFromFile calls for same bundle. Solution: Implement in-memory cache (Dictionary<string, AssetBundle>). Check cache first, load only if missing.

## Tools & Workflow

**AssetBundle Loading APIs**: AssetBundle.LoadFromFile (synchronous, local bundles), LoadFromFileAsync (asynchronous, local bundles), LoadFromMemory (synchronous, memory-resident bundles), LoadFromMemoryAsync (asynchronous, memory-resident). Choose based on use case.

**UnityWebRequestAssetBundle**: Download bundles from server. UnityWebRequestAssetBundle.GetAssetBundle(url, hash). Integrates with Unity Caching API (automatic disk caching). Use for CDN-hosted bundles, DLC, patches.

**Addressables LoadAssetAsync**: High-level API wrapping AssetBundle loading. Addressables.LoadAssetAsync<T>("AssetAddress"). Handles dependency loading, caching, unloading automatically. Recommended over manual AssetBundle APIs (simpler, less error-prone).

**Profiler Sampling**: Add Profiler.BeginSample("Load Bundle") / Profiler.EndSample() around loading code. Shows loading time in Profiler timeline. Identify slow bundles (LZMA bundles, large bundles, network-downloaded bundles).

**Memory Profiler**: Capture snapshots after loading bundles. AssetBundle section shows loaded bundles, memory usage, referenced assets. Verify bundles unload correctly (compare snapshot before/after Unload call).

**Resource Manager (Addressables)**: Addressables > Event Viewer shows real-time loading events (bundle loads, asset loads, reference counts). Debug loading issues (why bundle loaded twice? Why asset not unloading?).

**Custom Asset Manager**: Implement centralized AssetBundleManager class. Manages loading, caching, unloading, reference counting. Provides single API for all bundle operations (LoadBundle, LoadAsset, Unload). Easier to maintain than scattered AssetBundle calls.

## Related Topics

- [10.1 AssetBundle Fundamentals](10-01-AssetBundle-Fundamentals.md) - Bundle creation and structure
- [10.3 Content Delivery](10-03-Content-Delivery.md) - Downloading bundles from servers
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory management strategies
- [6.3 Memory Bottlenecks](06-03-Memory-Bottlenecks.md) - Identifying memory issues

---

[← Previous: 10.1 AssetBundle Fundamentals](10-01-AssetBundle-Fundamentals.md) | [Next: 10.3 Content Delivery →](10-03-Content-Delivery.md)
