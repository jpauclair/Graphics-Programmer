# 10.1 AssetBundle Fundamentals

[← Back to Chapter 10](../chapters/10-Asset-Bundling-And-Streaming.md) | [Main Index](../README.md)

AssetBundles provide runtime asset loading, modular content delivery, and platform-specific optimization by packaging assets into compressed archives separate from application binary.

---

## Overview

AssetBundles solve critical problems: reducing initial download size (ship game with core assets, download levels/DLC on-demand), enabling modular content updates (patch specific bundles without rebuilding entire game), and supporting platform-specific assets (different texture resolutions per platform in separate bundles). Without AssetBundles, all assets bake into application binary—a 50GB game requires 50GB download upfront. With AssetBundles, ship 5GB core game, stream remaining 45GB as needed (levels load on-demand, DLC downloads separately).

AssetBundles are archives containing assets: prefabs, textures, meshes, audio, materials, shaders. Building bundles via Unity's AssetBundle APIs or Addressables creates compressed files (.bundle extension) for distribution (download servers, local storage, streaming). Runtime loading uses AssetBundle.LoadFromFile (local bundles), LoadFromMemory (memory-resident bundles), or WWW/UnityWebRequest (downloaded bundles). Loaded bundles expose contained assets via LoadAsset<T>() API.

AssetBundle workflow: organize assets into logical bundles (per-level, per-feature, by asset type), configure bundle settings (compression: LZ4/LZMA/Uncompressed), build bundles per platform (PC/Xbox/PlayStation/Switch), distribute bundles (package with app or host on CDN), and load bundles at runtime (async loading, dependency management, memory management). Modern approach: use Addressables system (high-level wrapper simplifying AssetBundle complexity).

## Key Concepts

- **AssetBundle**: Compressed archive containing Unity assets. Built offline, loaded at runtime. Reduces application size, enables modular content, supports platform variants. Typical sizes: 10-500MB per bundle.
- **Bundle Dependencies**: Bundles reference assets in other bundles. Loading dependent bundle requires loading dependency bundles first. Unity tracks dependencies automatically (AssetBundleManifest provides dependency graph).
- **Compression Modes**: LZ4 (fast decompression <5ms, ~2x compression), LZMA (slow decompression 100-500ms, ~5x compression), Uncompressed (instant loading, no compression). Choose based on use case.
- **Bundle Variants**: Multiple versions of same bundle for different platforms/quality levels. Example: TexturesHigh.bundle (2K textures), TexturesLow.bundle (1K textures). Load appropriate variant at runtime.
- **Caching**: Downloaded bundles cached locally (Application.persistentDataPath or Unity's Caching API). Avoids re-downloading. Cache invalidation via bundle version hashes (CRC/hash comparison).

## Best Practices

**Bundle Organization Strategies:**
- Per-Level Bundles: Each level/scene in separate bundle (Level1.bundle, Level2.bundle). Loads only active level. Unloads when leaving level. Ideal for linear games.
- Per-Feature Bundles: Group by feature (CharacterCustomization.bundle, WeaponsBundle, VehiclesBundle). Loads features on-demand. Ideal for games with optional content.
- By Asset Type: Separate bundles for textures, meshes, audio (Textures.bundle, Meshes.bundle, Audio.bundle). Enables granular loading but increases bundle count and complexity.
- Hybrid Approach: Combine strategies. Core bundles (always loaded), level bundles (load per level), optional bundles (DLC, cosmetics). Most flexible.

**Compression Selection:**
- LZ4 for runtime loading: Fast decompression (<5ms per 50MB bundle). 2-3x compression. Use for levels, characters, anything loaded during gameplay. BuildAssetBundleOptions.ChunkBasedCompression = LZ4.
- LZMA for downloadable content: High compression (5-10x). Slow decompression (200-500ms per 50MB). Use for initial downloads, DLC, updates. Decompress during load screens. BuildAssetBundleOptions.None = LZMA (default).
- Uncompressed for streaming assets: Texture streaming, audio streaming. Avoids double-decompression (LZ4 + BC7/Vorbis). Larger downloads but instant loading. BuildAssetBundleOptions.UncompressedAssetBundle.

**Dependency Management:**
- Minimize dependencies: Bundles with many dependencies require loading multiple bundles (memory overhead, loading complexity). Group tightly-coupled assets in same bundle.
- Shared bundles: Extract commonly-referenced assets (shared materials, shaders) into SharedAssets.bundle. All other bundles depend on SharedAssets. Reduces duplication.
- Analyze dependencies: Use AssetBundle Browser (Package Manager) to visualize dependencies. Identify circular dependencies (breaks loading), excessive dependencies (performance hit).
- Load dependencies first: Before loading bundle, load all dependent bundles. Unity AssetBundleManifest provides GetAllDependencies() to automate.

**Platform-Specific Bundles:**
- Build per platform: Assets differ per platform (PC = BC7, Switch = ASTC). Build separate bundles per platform (PC/Windows, Xbox, PlayStation, Switch).
- Use BuildTarget parameter: AssetBundleBuild APIs accept BuildTarget enum. Build same logical bundle for all platforms, Unity handles format conversions automatically.
- Bundle naming conventions: Level1_PC.bundle, Level1_PS5.bundle, Level1_Switch.bundle. Runtime selects appropriate bundle based on platform detection.
- Addressables auto-handles: Addressables system builds platform-specific bundles automatically, abstracts loading logic. Recommended over manual AssetBundle APIs.

**Memory Management:**
- Unload bundles when unused: AssetBundle.Unload(false) keeps loaded assets, unloads bundle metadata. Unload(true) unloads everything (assets + bundle). Use Unload(false) for assets still in use, Unload(true) when leaving level.
- Track bundle references: Implement reference counting. Increment when loading asset from bundle, decrement when destroying asset. Unload bundle when references reach zero.
- Avoid asset duplication: If asset exists in multiple bundles, Unity loads multiple copies (wastes memory). Use dependency bundles to share assets.
- Profile with Memory Profiler: Capture snapshots before/after bundle loading. Verify memory usage matches expectations, identify leaks (bundles not unloaded).

**Bundle Size Guidelines:**
- Target 10-100MB per bundle: Too small (<5MB) = excessive bundle count (overhead), too large (>500MB) = long download/load times, memory spikes.
- Split large levels: Level bundle >200MB split into Level1_Environment, Level1_Characters, Level1_Effects. Load progressively (environment first, characters during spawn).
- Consider download limits: Mobile users on cellular (avoid >100MB downloads without WiFi warning). Console certification limits (vary by platform, typically 1-5GB initial download).

**Platform-Specific:**
- **PC**: Large bundles acceptable (50-200MB). Fast SSDs + high bandwidth. Use LZ4 compression for local bundles, LZMA for downloadable content.
- **Consoles**: Bundle sizes 50-100MB typical. Fast internal SSDs (PS5/Series X). Use LZ4 compression. Respect platform download limits (certification requirements).
- **Switch**: Smaller bundles preferred (20-50MB). Limited storage (32GB internal, often full). Use LZMA for downloads (smaller files), LZ4 for cartridge-based bundles.
- **Mobile**: Small bundles critical (<50MB). Cellular data limits, slow networks. Use LZMA compression, implement WiFi-only download option. Google Play/App Store have <150MB limit for initial APK/IPA.

## Common Pitfalls

**Asset Duplication Across Bundles**: Developer includes same material in multiple bundles (CharactersBundle, EnvironmentBundle). Unity loads material twice (wastes memory). Symptom: Memory usage 2x expected, duplicate assets in Memory Profiler. Solution: Create SharedAssetsBundle, reference from other bundles. Material loads once, shared across bundles.

**LZMA for Runtime Loading**: Developer uses LZMA compression for all bundles (smallest file size). Runtime loading takes 500ms-2s per bundle (slow decompression). Causes hitches when spawning objects or loading mid-game. Symptom: Profiler shows long LoadAssetBundle times (100-500ms). Solution: Use LZ4 for runtime bundles (fast decompression <5ms). Reserve LZMA for downloads.

**Unloading Bundles Too Early**: Developer calls AssetBundle.Unload(true) immediately after LoadAsset(). Loaded asset references unloaded data (broken materials, missing textures). Symptom: Pink/missing textures, broken references. Solution: Keep bundle loaded while assets in use. Use AssetBundle.Unload(false) when done with bundle but assets still needed, or implement reference counting.

## Tools & Workflow

**AssetBundle Browser**: Unity package (com.unity.assetbundlebrowser). Visualizes bundle contents, dependencies, sizes. Configure bundles (assign assets to bundles), build bundles, analyze results. Window > AssetBundle Browser.

**Addressables System**: Modern AssetBundle abstraction (com.unity.addressables). Automates bundle building, dependency management, loading, unloading. Highly recommended over raw AssetBundle APIs. Window > Asset Management > Addressables.

**AssetBundle Build Pipeline**: Unity APIs for building bundles. BuildPipeline.BuildAssetBundles() creates bundles. AssetBundleManifest tracks dependencies. Scriptable Build Pipeline (SBP) provides advanced control (custom processing, incremental builds).

**Memory Profiler**: Captures memory snapshots. Native Memory > AssetBundle shows loaded bundles and memory usage. Identify bundles not unloaded (memory leaks), asset duplication (same asset in multiple bundles).

**Bundle Analyzer**: Third-party tools (Asset Store) for analyzing bundle contents, finding optimization opportunities (duplicate assets, large textures, unused assets).

**CDN Integration**: Host bundles on Content Delivery Network (AWS CloudFront, Azure CDN, Google Cloud CDN). Users download from geographically-close servers (faster downloads). Use UnityWebRequest with CDN URLs.

**Version Management**: Use bundle hashes (CRC32, Hash128) for versioning. UnityWebRequestAssetBundle.GetAssetBundle(url, hash) validates downloaded bundle matches expected hash (prevents corrupted downloads).

## Related Topics

- [10.2 Loading Strategies](10-02-Loading-Strategies.md) - AssetBundle loading techniques
- [10.3 Content Delivery](10-03-Content-Delivery.md) - Distribution and downloading
- [8.4 Data Compression](08-04-Data-Compression.md) - LZ4 vs LZMA compression
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory management

---

[← Previous: Chapter 9](../chapters/09-Asset-Format-Guidelines.md) | [Next: 10.2 Loading Strategies →](10-02-Loading-Strategies.md)
