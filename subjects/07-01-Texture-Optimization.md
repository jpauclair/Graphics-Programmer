# 7.1 Texture Optimization

[← Back to Chapter 7](../chapters/07-Asset-Optimization.md) | [Main Index](../README.md)

Texture optimization reduces VRAM consumption, bandwidth usage, and loading times through compression, resolution management, and format selection.

---

## Overview

Textures consume 60-80% of VRAM in typical games. A single 4K RGBA32 texture occupies 64MB uncompressed, but compresses to 8-12MB with BC7/ASTC. Across 200 textures, that's 12.8GB uncompressed vs 1.6-2.4GB compressed—an 85% reduction. Texture optimization is the highest-impact graphics optimization technique, dramatically improving memory usage, bandwidth consumption, and loading performance.

Texture optimization has three primary goals: reduce VRAM footprint (compression, resolution limits), minimize bandwidth (compression, mipmaps, filtering), and accelerate loading (compression reduces file size, async loading prevents hitches). These goals are interconnected—compression simultaneously reduces memory, bandwidth, and loading time.

Optimization workflow: audit existing textures via Memory Profiler (identify largest consumers), apply platform-specific compression (BC7 for PC/Xbox, ASTC for mobile/Switch), set maximum resolutions per platform (2K current-gen, 1K Switch), enable mipmaps for 3D assets, and implement texture streaming for open-world games. Profiling validates improvements: Memory Profiler shows VRAM reduction, profilers show bandwidth improvements, and users experience faster loading.

## Key Concepts

- **Texture Compression**: Hardware-accelerated lossy compression (BC7, ASTC, BC5, BC6H). GPU decompresses transparently during sampling. Reduces memory by 75-90% with minimal quality loss.
- **Texture Resolution**: Pixel dimensions (512x512, 1024x1024, 2048x2048, 4096x4096). Memory usage scales quadratically: 2K = 16MB uncompressed, 4K = 64MB (4x memory for 2x resolution).
- **Mipmaps**: Precomputed lower-resolution versions of texture (1/4, 1/16, 1/64 size). Used for distant objects. Improves cache efficiency by 80-90%, essential for performance. Adds 33% memory cost.
- **Texture Format**: Data layout and bit-depth (RGBA32, RGB24, BC7, ASTC, R16, etc.). Determines memory consumption and quality. BC7 = 8:1 compression, ASTC 8x8 = 8:1 compression.
- **Texture Atlasing**: Combining multiple small textures into single large texture. Reduces draw calls (shared material), improves batching, and decreases texture bind overhead. Essential for UI and small props.

## Best Practices

**Compression Strategy:**
- Use BC7 for color textures on PC/Xbox: Highest quality block compression (8:1 ratio). Supports alpha. Unity import setting: Format Override > PC/Xbox > BC7.
- Use ASTC for mobile/Switch: Flexible compression ratios (4x4 = 8:1 high quality, 8x8 = 16:1 moderate quality). Format Override > Android/Switch > ASTC 6x6 (balance) or 8x8 (performance).
- Use BC5 for normal maps: 2-channel compression (4:1) optimized for normals. Reconstruct Z in shader (`z = sqrt(1 - x*x - y*y)`). Significantly better quality than BC7 for normals.
- Use BC6H for HDR textures: Only compressed HDR format. 6:1 compression. Essential for HDRI skyboxes and emissive textures (16-bit float RGB).
- Never use uncompressed formats (RGBA32, RGB24) except debugging. Even UI benefits from BC7 (imperceptible quality loss, 87.5% memory savings).

**Resolution Management:**
- Set maximum texture sizes per platform: Import Settings > Platform-Specific Overrides > Max Size. PC/Current-gen: 2048, Switch/Last-gen: 1024, Mobile Low-end: 512.
- Use 4K textures sparingly: Reserve for hero characters and close-up surfaces only. Most environment textures work fine at 1K-2K (tiling helps hide resolution).
- Implement texture quality settings: Low (512-1K max), Medium (1K-2K), High (2K), Ultra (4K for hero assets). Let users choose based on VRAM capacity.
- Downscale non-hero assets: Background characters (1K), small props (512), foliage (512), particles (256). Reserve resolution budget for important assets.
- Profile texture memory usage: Memory Profiler > Take Snapshot > Native Memory > Texture. Sort by size, identify oversized textures (2K texture on small 10cm prop).

**Mipmap Strategy:**
- Enable mipmaps for all 3D scene textures: Improves cache efficiency 80-90%. Distant objects sample small mipmaps (1/16 size) instead of full texture, fitting in cache.
- Disable mipmaps for UI: UI textures render at fixed size (pixel-perfect). Mipmaps waste 33% memory without benefit. Import Settings > Advanced > Generate Mip Maps = false.
- Use anisotropic filtering: 8-16x on PC/current-gen, 2-4x on Switch/mobile. Improves oblique angle quality without bandwidth cost (mipmaps already cached).
- Avoid negative LOD bias: `Texture.mipMapBias = -1` forces higher-res mips, increasing bandwidth 2-4x. Only use for hero assets if needed, never globally.

**Format Selection:**
- Single-channel textures (masks, heightmaps): Use R8 or R16 format. Saves 75% memory vs RGBA. Import Settings > Alpha Source = None, uncheck sRGB (if linear data).
- Two-channel textures (normal maps): Use BC5 (4:1 compression, 2 channels). Reconstruct Z in shader. Much higher quality than BC7 normals.
- HDR textures (emissive, skybox): Use BC6H (6:1 compression, 16-bit float RGB). Only option for compressed HDR on GPU. Falls back to uncompressed RGBA16F if unavailable.
- Alpha textures (cutouts, particles): Use BC7 or ASTC (both support alpha). BC1 (legacy) has terrible alpha quality (1-bit alpha only).

**Texture Atlasing:**
- Atlas UI elements: Combine buttons, icons, fonts into single texture. Reduces draw calls from 100+ to 10-20. Use Unity Sprite Atlas or TexturePacker tool.
- Atlas small props: Combine props sharing materials (crates, barrels, rocks). Single material enables static batching, instancing. Memory cost: Slight increase (padding/wasted space).
- Use Texture2DArray for tileables: Alternative to atlasing for similar-sized textures. GPU samples via array index. Better filtering than atlas (no bleeding between tiles).
- Avoid atlasing large or varied-size textures: Wastes space (padding), forces high atlas resolution (memory explosion), and complicates UV mapping.

**Platform-Specific:**
- **PC**: Use BC7 for color, BC5 for normals, BC6H for HDR. 2K-4K textures acceptable on high-end (12-16GB VRAM), provide quality settings for min-spec (4-6GB VRAM).
- **Xbox Series X/S**: Use BC7/BC5/BC6H. Series X: 2K standard. Series S: 1K-1.5K (lower VRAM budget). Test on Series S specifically—lowest common denominator.
- **PlayStation 5/4**: Same BC formats work. PS5: 2K textures. PS4: 1K-1.5K. Use Razor to monitor VRAM usage (3GB target for PS4, 10GB for PS5).
- **Nintendo Switch**: ASTC mandatory (no BC support). Use ASTC 6x6 for quality, 8x8 for performance. 1K max resolution for most textures, 512 for foliage/particles. Test handheld mode (thermal throttling).

## Common Pitfalls

**Importing 4K Textures Without Platform Overrides**: Artist imports 4K texture, assumes Unity will downscale per platform. Unity doesn't—imports at 4K for all platforms unless explicitly overridden. Switch now loads 64MB texture instead of 4MB (1K ASTC). Symptom: Memory usage identical across platforms. Solution: Always set Platform-Specific Overrides for target platforms (PC, Xbox, PlayStation, Switch).

**Disabling Mipmaps to Save Memory**: Developer sees mipmaps add 33% memory, disables globally. Now distant textures sample full resolution with cache thrashing, destroying bandwidth and performance. Frame rate drops from 60fps to 20fps in distant views. Symptom: Low FPS in open areas despite few objects. Solution: Always enable mipmaps for 3D scene textures. Memory cost (33%) is vastly outweighed by performance benefits (5-10x bandwidth reduction).

**Using RGBA for Single-Channel Data**: Mask texture or heightmap imported as RGBA32 (4 bytes per pixel). Actually uses only R channel, but occupies 4x memory. 2K mask = 16MB instead of 4MB. Symptom: Memory Profiler shows grayscale textures occupying full RGBA space. Solution: Import Settings > Alpha Source = None, Advanced > Single Channel = Red. Unity will use R8 format (1 byte per pixel).

## Tools & Workflow

**Unity Texture Import Settings**: Select texture > Inspector > Texture Type, Compression, Max Size. Essential: Platform-Specific Overrides (click "Default" platform dropdown, add PC, Xbox, Switch, etc.).

**Memory Profiler**: Window > Analysis > Memory Profiler > Capture. Native Memory > Textures shows per-texture memory consumption. Sort by size, identify optimization targets (4K textures on small objects).

**Texture Memory Report**: Build Settings > Player Settings > Other Settings > Generate Texture Memory Report. Produces CSV with per-texture memory usage in build. Import to spreadsheet, analyze largest consumers.

**RenderDoc/PIX Texture Viewer**: Inspect runtime textures, verify compression formats, check mipmap chains. RenderDoc: Texture Viewer tab. PIX: Resources > Textures > Select texture > Format column.

**Sprite Atlas**: Window > 2D > Sprite Atlas. Drag sprites to atlas. Unity automatically packs, generates texture atlas. Essential for UI and 2D games (reduces draw calls 90%).

**TexturePacker**: Third-party tool for advanced atlasing. Better packing algorithms than Unity's Sprite Atlas. Exports texture + metadata (UV coordinates). Download from https://www.codeandweb.com/texturepacker

**Unity Presets**: Create Texture Import Preset for each category (Characters, Environment, UI, Props). Apply preset on import for consistent settings. Right-click Import Settings > Save as Preset.

## Related Topics

- [13.1 Texture Formats and Compression](08-01-Texture-Compression-Formats.md) - Detailed compression techniques
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory reduction strategies
- [5.4 Bandwidth Optimization](05-04-Bandwidth-Optimization.md) - Bandwidth reduction through compression
- [11.1 Materials and Textures](11-01-Material-System-Architecture.md) - Material system overview

---

[← Previous: Chapter 6](../chapters/06-Bottleneck-Identification.md) | [Next: 7.2 Mesh Optimization →](07-02-Mesh-Optimization.md)
