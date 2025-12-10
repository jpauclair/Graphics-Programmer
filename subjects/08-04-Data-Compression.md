# 8.4 Data Compression

[← Back to Chapter 8](../chapters/08-Compression-Techniques.md) | [Main Index](../README.md)

Data compression reduces build sizes, loading times, and runtime memory for AssetBundles, save files, and network traffic using general-purpose algorithms like LZ4, LZMA, and Brotli.

---

## Overview

General-purpose compression applies to non-media data: AssetBundles, save files, configuration files, network packets, and serialized data. Unlike domain-specific compression (texture, mesh, audio), general-purpose algorithms work on arbitrary byte streams. Compression ratios vary by content: text files (3-10x), binary data (2-5x), already-compressed data (minimal gains). Tradeoffs exist: high compression (LZMA) produces small files but slow decompression, fast compression (LZ4) produces larger files but instant decompression.

AssetBundle compression is primary use case: Unity offers LZ4 (fast decompression, ~2x compression) and LZMA (slow decompression, ~5x compression). LZ4 preferred for runtime loading (decompression <5ms per bundle). LZMA preferred for downloadable content (smaller downloads, decompress during install screens). Save file compression uses GZip or custom schemes. Network compression uses LZ4 or Brotli for real-time multiplayer (low latency requirement).

Optimization workflow: analyze data compressibility (text compresses well, encrypted data doesn't), select algorithm per use case (AssetBundles = LZ4, downloads = LZMA, saves = GZip), profile decompression cost vs size savings, and implement chunked decompression for large files (avoid hitches). Results: 50-80% build size reduction, 30-50% faster downloads, minimal runtime performance impact with proper algorithm selection.

## Key Concepts

- **LZ4**: Fast compression/decompression. 2-3x compression ratio. Decompression 500-2,000 MB/s (essentially free on modern CPUs). Ideal for runtime loading (AssetBundles, streaming). Unity's default AssetBundle compression.
- **LZMA**: High compression. 3-10x ratio (depends on content). Slow decompression 20-50 MB/s (20-50ms per MB). Ideal for downloads or offline content. Unity supports for AssetBundles (legacy mode).
- **GZip**: Balanced compression (3-5x ratio). Moderate decompression speed (100-200 MB/s). Standard for web content, save files. Unity supports via .NET System.IO.Compression.
- **Brotli**: Modern algorithm (better compression than GZip at equivalent speed). 4-6x ratio. Used for web content, network traffic. Requires Unity 2021+ or external library.
- **Compression Ratio**: Size reduction (uncompressed size / compressed size). 2x = 50% smaller. 10x = 90% smaller. Higher ratio = smaller files, but often slower decompression.

## Best Practices

**AssetBundle Compression:**
- Use LZ4 for runtime-loaded bundles: Build Settings > AssetBundle Compression = LZ4. Fast decompression (<5ms per bundle, 10-100MB). 2-3x compression sufficient for most content.
- Use LZMA for downloadable content: Initial game download, DLC, patches. Smaller downloads (5-10x compression). Decompress during install/load screens (hide 50-200ms cost).
- Uncompressed for streaming: AssetBundle Compression = Uncompressed. Used for texture streaming (avoid double-decompression: LZ4 + BC7). Slightly larger downloads, instant loading.
- Chunked loading for large bundles: Decompress in background thread, load progressively. Avoid main-thread stalls from decompressing 500MB LZMA bundle (5-10 seconds).
- Profile bundle loading: Profiler > Loading.LoadAssetBundle shows decompression time. LZ4 should be <10ms. LZMA can be 100-1,000ms (acceptable during load screens, not runtime).

**Save File Compression:**
- Use GZip for save files: System.IO.Compression.GZipStream. 3-5x compression for text-heavy saves (JSON, XML). Decompression fast enough for save/load (10-50ms).
- Serialize to binary first: Convert save data to compact binary (BinaryFormatter, MessagePack, Protobuf). Then compress. Binary + GZip compresses better than compressing text directly.
- Implement automatic compression: Save pipeline auto-compresses files >10KB. Small saves (<10KB) skip compression (overhead not worth it).
- Validate compressed size: Test with worst-case save data (large inventories, max progress). Ensure compressed size acceptable for cloud saves (limits are 1-10MB per platform).

**Network Compression:**
- Use LZ4 for real-time multiplayer: Low latency requirement (<5ms). LZ4 decompression fits in frame budget. Compress large packets (>1KB), skip small packets (<500 bytes, overhead exceeds savings).
- Use Brotli for HTTP downloads: Web requests for asset downloads, leaderboards, catalog data. Brotli provides better compression than GZip at equivalent speed. Unity WebRequest supports.
- Selective compression: Compress text/JSON (high compressibility), skip binary/encrypted data (minimal gains). Profile per-packet savings vs CPU cost.
- Implement compression threshold: Only compress packets >500 bytes. Smaller packets have overhead (compressed size + header) exceeding uncompressed size.

**Build Pipeline Compression:**
- Enable build compression: Build Settings > Compression Method = LZ4. Compresses build executable and data files. Reduces install size 30-50%.
- LZ4 vs LZMA for builds: LZ4 = faster build times (5-10 minutes), larger builds (+20-30%). LZMA = slower builds (10-30 minutes), smaller builds (-30-50%). Choose based on priority (development = LZ4, distribution = LZMA).
- Exclude pre-compressed assets: AssetBundles already LZ4/LZMA compressed. Build compression doesn't help (minimal gains). Use build report to identify uncompressible content.

**Custom Data Compression:**
- Compress procedural data: Generated meshes, texture data, simulation state. Store compressed, decompress when needed. Saves memory for rarely-accessed data.
- Delta compression: Store deltas vs base state (animation curves, time-series data). Compress deltas (high redundancy). Provides 5-20x compression for incremental data.
- Dictionary-based compression: Repeated strings (IDs, tags, names) replaced with indices into dictionary. Custom scheme for heavily-repeated data. 3-10x compression.

**Platform-Specific:**
- **PC**: LZ4 for AssetBundles. Fast CPUs handle decompression easily. LZMA for downloads (users expect smaller downloads on PC).
- **Consoles**: LZ4 for AssetBundles. Xbox/PlayStation SSDs fast enough that decompression not bottleneck. Avoid LZMA at runtime (slow CPUs, long decompression times).
- **Switch**: LZ4 mandatory (weak CPU). LZMA decompression too slow (200-500ms per MB). Prefer smaller AssetBundles (<50MB) over large compressed bundles.
- **Mobile**: LZ4 for AssetBundles. LZMA for app downloads (reduces cellular data usage). Battery consideration (decompression power consumption vs download power consumption).

## Common Pitfalls

**LZMA Compression for Runtime AssetBundles**: Developer uses LZMA for all AssetBundles to minimize build size. Runtime loading now takes 500ms-2s per bundle (LZMA decompression is slow). Causes hitches when spawning objects or loading scenes. Symptom: Profiler shows Loading.LoadAssetBundle taking 500+ms. Solution: Use LZ4 for runtime bundles (fast decompression). Reserve LZMA for downloadable content (decompress during install screens).

**Compressing Already-Compressed Data**: AssetBundles already contain BC7 textures, ADPCM audio, compressed meshes. Developer enables LZMA build compression. LZMA tries compressing already-compressed data—minimal gains (5-10%) for significant decompression cost. Symptom: Build compression saves <10% size, decompression takes long time. Solution: Use Uncompressed AssetBundles if contents already compressed. Or use LZ4 (minimal overhead on pre-compressed data).

**Synchronous Decompression on Main Thread**: Developer decompresses 100MB LZMA AssetBundle on main thread. Decompression takes 2-5 seconds (LZMA slow). Game freezes during decompression. Symptom: Profiler shows multi-second spike in Loading.LoadAssetBundle. Solution: Decompress asynchronously (background thread), show progress bar. Or use LZ4 instead (decompression <50ms, acceptable on main thread).

## Tools & Workflow

**AssetBundle Build Settings**: Build Settings > AssetBundle Compression dropdown (Uncompressed, LZ4, LZMA). Select per use case. LZ4 default for runtime, LZMA for downloads.

**Build Report**: After build, Build Report shows compressed sizes. Analyze per-bundle compression ratios. Identify bundles with poor compression (already-compressed content).

**Unity Profiler**: Profiler > Loading module shows LoadAssetBundle time. LZ4 should be <10ms per bundle. LZMA can be 100-1,000ms. Optimize accordingly.

**Custom Compression Script**: Implement compression utilities for save files. Example:
```csharp
using System.IO;
using System.IO.Compression;

public static byte[] Compress(byte[] data) {
    using (var output = new MemoryStream()) {
        using (var gzip = new GZipStream(output, CompressionMode.Compress)) {
            gzip.Write(data, 0, data.Length);
        }
        return output.ToArray();
    }
}

public static byte[] Decompress(byte[] data) {
    using (var input = new MemoryStream(data))
    using (var gzip = new GZipStream(input, CompressionMode.Decompress))
    using (var output = new MemoryStream()) {
        gzip.CopyTo(output);
        return output.ToArray();
    }
}
```

**7-Zip Benchmark**: Standalone tool for testing compression algorithms/ratios. Compare LZ4, LZMA, Brotli on sample data. Validates compression ratio vs speed tradeoffs. Download from https://www.7-zip.org/

**LZ4 Unity Package**: Unity Package Manager > Add package from git URL: com.unity.nuget.lz4. Provides direct LZ4 access for custom compression needs.

**MessagePack/Protobuf**: Binary serialization libraries. Produces compact binary (30-50% smaller than JSON) before compression. Faster serialization too. Available via NuGet or Unity Package Manager.

## Related Topics

- [10.1 Asset Bundling and Streaming](10-01-AssetBundle-Fundamentals.md) - AssetBundle compression strategies
- [25.1 Build Pipeline and Workflow](25-01-Build-Optimization.md) - Build compression
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory reduction
- [21.1 Frame Management](21-01-Frame-Pacing.md) - Avoiding loading hitches

---

[← Previous: 8.3 Audio Compression](08-03-Audio-Compression.md) | [Next: Chapter 9 - Asset Format Guidelines →](../chapters/09-Asset-Format-Guidelines.md)
