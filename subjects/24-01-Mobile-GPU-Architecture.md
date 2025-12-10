# 24.1 Mobile GPU Architecture

[← Back to Chapter 24](../chapters/24-Mobile-And-Cross-Platform.md) | [Main Index](../README.md)

Mobile GPU architecture fundamentally different from desktop (tile-based deferred rendering vs immediate-mode forward). ARM Mali (Samsung, OnePlus), Qualcomm Adreno (most Android), Apple A-series (iPhone). Key: tile-based = screen divided into tiles (32x32 or 16x16), rendering per-tile (load-action = local memory = fast, save-action = write to main memory = slow). Understanding architecture = optimize for bandwidth, reduce frame buffer writes, avoid render target ping-ponging.

---

## Overview

Mobile GPU Tiling: screen divided into 32x32 (16KB fit in L3 cache) or 16x16 (4KB) tiles, renderer processes per-tile (all triangles affecting tile queued, rasterized, shading), tile result written to main memory. Benefit: tile-size = L3 cache = no bandwidth to main RAM (vs desktop = full framebuffer = hundreds MB/sec bandwidth needed). Drawback: render target switches = problem (tile-deferred = must flush tile when render target changes = expensive).

Bandwidth Pressure: mobile = limited memory bandwidth (iPad Pro 2021 = 60 GB/sec, iPhone 13 = 40 GB/sec vs RTX 3090 = 600 GB/sec = 10x worse). Optimization: reduce render target writes (combine passes, use subpasses = avoid intermediate writes), MSAA via fragment shader (not supersampling = 4x render = 4x bandwidth), reduce framebuffer format (RGBA8 = 4 bytes/pixel, max bandwidth = bandwidth / 4).

## Key Concepts

- **Tile-Based Deferred Rendering (TBDR)**: Screen split into independent tiles (32x32 or 16x16 pixels), per-tile rendering (geometry list compiled, lighting applied, shaded). Process: (1) geometry pass = all triangles affecting tile, (2) lighting = apply lights to tile (local to 512 pixels vs desktop = 2 million pixels = parallelism higher), (3) write tile = transfer L3 result to main memory. Benefit: L3 cache = 512-1024KB per tile, lighting/shading = cache-friendly (all pixel data in cache = fast), no bandwidth to main RAM during shading. Drawback: render target switch = flush tile to RAM, restart new render target = stall.
- **ARM Mali Architecture**: Common mobile GPU (Samsung Galaxy, OnePlus, Google Pixel), TBDR (tile-based), OpenGL ES / Vulkan support. Features: 4-16 cores (4 typical mobile, 16 iPad Pro = high-end), tile size 16x16 or 32x32 (configurable), fragment rate shading = variable resolution per tile (important for foveated rendering VR). Optimization: tile-friendly (use subpasses, avoid ping-ponging).
- **Qualcomm Adreno**: Snapdragon GPUs (most Android phones), TBDR, OpenGL ES / Vulkan. Features: 2-6 cores (most phones 2-4), binning pass (initial geometry pass, creates triangle lists per tile), rendering pass (actual pixel shading), deferred compression = lossless tile data compression (save bandwidth). Optimization: wide pipelines = high parallelism = lots of threads needed (avoid stalling any core).
- **Apple A-series**: iPhone/iPad GPUs (unified architecture = CPU/GPU on same chip), TBDR with immediate mode rendering hybrid, tile size 32x32 (1KB shared memory per tile). Features: tile memory = on-chip = ultra-fast, Metal API = explicit control (loadAction, storeAction per render target), deferred rendering = multiple inputs accepted (binning efficient). Optimization: leverage Metal explicitly (set load/store actions = avoid unnecessary writes).
- **Bandwidth Limits**: Mobile bandwidth (40-100 GB/sec) = precious, framebuffer writes expensive. Calculation: 1440p 30 FPS RGBA8 = 1440 * 2560 * 4 * 30 = 442 GB/sec (impossible = exceeds bandwidth). Solution: (1) lower resolution (1080p = 248 GB/sec, still over, needs optimization), (2) reduce framebuffer (RGB565 = 3 bytes/pixel), (3) combine passes (use subpasses = avoid intermediate writes), (4) variable resolution (render 75% resolution in some areas = save bandwidth).
- **Subpass Rendering**: Multiple passes within single render pass (avoid render target switch = no flush). Example: render G-buffer (positions, normals, albedo), then lighting pass input = G-buffer, output = lit color = subpass = no intermediate write = faster. Metal: renderPassDescriptor with multiple color attachments + fragment shader inputs = implicit subpass. Vulkan: VK_KHR_multiview + input attachments. Benefit: tile-resident = no main memory write/read = cache-friendly.

## Best Practices

**TBDR-Friendly Rendering**:
- Minimize render target switches: if multi-pass rendering (G-buffer, lighting, post-fx), use subpasses if supported (Metal explicit, Vulkan VK_KHR_multiview, Unity batching minimizes switches).
- Set load/store actions (Metal): loadAction = Clear (clear tile, don't load = saves bandwidth), storeAction = Store (write result, necessary = do it), or DontCare (discard = save bandwidth if not needed). Example: G-buffer loadAction = Clear (don't need previous), storeAction = Store (need for lighting).
- Avoid ping-ponging: alternating render targets (write RT1, read RT1, write RT2, read RT2) = tile flush between each = stalls. Instead: blit-free (compute shader, or single pass if possible), or batch switches (all writes to RT1, then all to RT2).

**Bandwidth Optimization**:
- Resolution selection: target device tiers (low = 720p, mid = 1080p, high = 1440p), measure FPS on each = scaling sweet spot (60 FPS 720p = good, 1080p might be 45 FPS = scale down for 60).
- Framebuffer format: RGBA8 standard (4 bytes), RGB565 if possible (3 bytes = 25% bandwidth save, quality loss = acceptable for some games), compressed (ASTC = variable size, 1-8 bits/pixel).
- Subpass rendering: use where possible (combine G-buffer + lighting = skip intermediate write).
- Indirect rendering: compute shader populates indirect buffers (GPU-CPU no sync = parallel work), main thread renders based on computed data = avoid stalls.

**Platform-Specific Optimization**:
- **Mali**: TBDR standard (tile-size = 32x32 typical), Mali GPU Counter = profiler (check tile shader time vs waiting, identify bottleneck).
- **Adreno**: Binning pass = cost (profile, may dominate on transparent heavy games), binning bandwidth = relevant (compress geometry).
- **Apple A-series**: Metal explicit load/store actions = optimize (research per scenario, set precisely).

## Common Pitfalls

**Render Target Ping-Pong**: Developer renders to RT1, reads RT1, renders to RT2, reads RT2 (alternating read/write). Per-tile renderer flushes tile between each = massive stall (tile bandwidth = <1ms/tile, switch time = 10-50ms = 10-50x slower than needed). Symptom: GPU stall (Profiler shows GPU wait time high, actual pixel shading low). Solution: batch operations (render all to RT1, then all to RT2), or use compute shader (avoid ping-pong pattern).

**Framebuffer Bandwidth Mismatch**: Developer renders 1440p @ 60 FPS (1440 * 2560 * 4 * 60 = 1.76 Gbps = exceeds bandwidth), expects 60 FPS, gets 30. Symptom: FPS capped (GPU-bound, no hope of improvement, bandwidth limited). Solution: lower resolution (1080p = 700 Mbps = within budget), or post-process downscale (render 1440p, downscale to 1080p = save bandwidth on final output), or variable resolution (critical areas 1440p, background 720p = weighted average).

**Subpass Not Used**: Developer renders G-buffer, writes to RT, reads RT, applies lighting (intermediate write to main memory = stall). With subpass (single pass, input attachments): no intermediate write = 2-3x faster. Symptom: deferred rendering slow on mobile, forward rendering same speed (indication subpass not used = intermediate writes = problem). Solution: convert to subpass rendering (Metal straightforward, Vulkan VK_KHR_multiview, or compute-based).

## Tools & Workflow

**Mobile GPU Profiling**: Snapdragon Profiler (Adreno devices), Mali Graphics Debugger (Mali devices), Xcode Instruments (Apple). Measure: tile time (shading per tile), binning time (geometry compilation), memory bandwidth (MB/sec current vs max), stalls (GPU idle).

**Metal Load/Store Actions** (iOS):
```swift
renderPassDescriptor.colorAttachments[0].loadAction = .clear // Clear tile, don't load
renderPassDescriptor.colorAttachments[0].storeAction = .store // Write result
renderPassDescriptor.depthAttachment.loadAction = .clear
renderPassDescriptor.depthAttachment.storeAction = .dontCare // Discard depth
```

**Bandwidth Calculation**: Resolution * 4 (RGBA8) * FPS * bit depth. Example: 1440p 30 FPS = 1440 * 2560 * 4 * 30 / (1024^3) = 0.42 Gbps. Mobile limit ≈ 40-100 Gbps depending on device.

## Related Topics

- [06.2 GPU Architecture](02-03-GPU-Components.md) - Desktop GPU comparison
- [24.2 Mobile Optimization](24-02-Mobile-Optimization.md) - Mobile-specific optimization
- [05.3 Bandwidth Optimization](05-04-Bandwidth-Optimization.md) - Memory bandwidth concepts
- [14.2 GPU Profiling](04-01-PC-Profiling-Tools.md) - Profiling techniques

---

[Previous: Chapter 23](../chapters/23-Multithreading-And-Jobs-System.md) | [Next: 24.2 Mobile Optimization →](24-02-Mobile-Optimization.md)
