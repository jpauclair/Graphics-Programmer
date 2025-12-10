# 8.1 Texture Compression Formats

[← Back to Chapter 8](../chapters/08-Compression-Techniques.md) | [Main Index](../README.md)

Understanding platform-specific texture compression formats enables optimal quality-to-size ratios, reducing VRAM by 75-90% while maintaining visual fidelity.

---

## Overview

Texture compression is hardware-accelerated lossy compression. GPU decompresses transparently during sampling—no runtime CPU cost. Uncompressed 2K RGBA texture occupies 16MB (2048×2048×4 bytes), but BC7 compresses to 2MB (8:1 ratio) with minimal quality loss. Across 200 textures, that's 3.2GB uncompressed vs 400MB compressed—87.5% reduction. Texture compression is the highest-impact single optimization, essential for all platforms.

Compression formats vary by platform: PC/Xbox use BCn (Block Compression), mobile/Switch use ASTC (Adaptive Scalable Texture Compression), and legacy platforms use DXT/ETC. Format selection depends on content: BC7/ASTC for color textures (8:1 compression), BC5 for normal maps (4:1, 2-channel optimized), BC6H for HDR (6:1, 16-bit float), and BC4 for single-channel masks (4:1).

Optimization workflow: audit textures (identify uncompressed), select format per texture type and platform (PC/Xbox = BC7, Switch/mobile = ASTC), set compression quality (balance size vs quality), and validate visually (compare compressed vs original). Modern formats (BC7, ASTC) provide superior quality vs legacy formats (DXT1/5, ETC1/2) at equivalent bitrates—always prefer modern formats.

## Key Concepts

- **Block Compression (BCn)**: Fixed-rate compression storing 4x4 pixel blocks. BC7 = 8:1 RGBA, BC5 = 4:1 2-channel, BC6H = 6:1 HDR, BC4 = 4:1 1-channel. Used on PC, Xbox, PlayStation. Hardware support universal on modern GPUs.
- **ASTC**: Adaptive compression with configurable block sizes (4x4 to 12x12). Flexible quality/size tradeoff: 4x4 = 8:1 high quality, 8x8 = 16:1 moderate quality. Supports LDR and HDR. Used on mobile, Switch, modern APIs.
- **Compression Ratio**: Size reduction vs uncompressed. RGBA32 = baseline. BC7/ASTC 6x6 = 8:1. BC5 = 4:1. BC6H = 6:1. Higher ratio = smaller textures, lower quality (blocky artifacts).
- **Alpha Handling**: BC7 and ASTC support full alpha. BC1 has 1-bit alpha (on/off only). BC3 has 8-bit alpha. BC5 has no alpha (2 channels only). Choose based on alpha needs.
- **Perceptual Quality**: Compression artifacts vary by content. Photographs compress well (natural detail hides artifacts). Flat colors/gradients compress poorly (banding visible). Test compressed textures visually.

## Best Practices

**PC and Xbox Compression:**
- Use BC7 for color textures: Highest quality RGBA compression (8:1). Supports full alpha. Replaces legacy BC1/BC3. Unity Import Settings > PC/Xbox platform override > Format = BC7.
- Use BC5 for normal maps: 2-channel compression (4:1) optimized for tangent-space normals (RG channels). Reconstruct Z in shader: `normal.z = sqrt(1 - dot(normal.xy, normal.xy))`. Much higher quality than BC7 for normals.
- Use BC6H for HDR textures: Only compressed HDR format. 6:1 compression, 16-bit float RGB. Essential for HDRI skyboxes, emissive textures. Format = BC6H.
- Use BC4 for single-channel masks: Grayscale textures (heightmaps, masks). 4:1 compression, 1 channel. Format = BC4 (grayscale).
- Legacy BC1/BC3 for backward compatibility: BC1 (DXT1) = 6:1 RGB or 8:1 RGBA (1-bit alpha). BC3 (DXT5) = 8:1 RGBA (8-bit alpha). Older hardware support only.

**Mobile and Switch Compression:**
- Use ASTC for all textures: Universal format on modern mobile GPUs and Switch. Replaces ETC/PVRTC. Platform override > Format = ASTC.
- ASTC 6x6 for high quality: 5.3:1 compression. Best balance of quality and size for hero assets, UI, close-up textures. Block size = 6x6.
- ASTC 8x8 for performance: 8:1 compression. Moderate quality sufficient for environment textures, foliage, particles. Block size = 8x8.
- ASTC 4x4 for maximum quality: 8:1 compression (same as 8x8 ratio but different algorithm). Slightly better quality than 6x6 for complex textures. Block size = 4x4.
- ASTC supports HDR: ASTC HDR mode for emissive and skybox textures. No need for separate format (unlike BC6H on PC). Enable HDR checkbox in import settings.

**PlayStation Compression:**
- Use BC7/BC5 on PlayStation: GNM API supports BCn formats. Same as PC/Xbox. BC7 for color, BC5 for normals, BC6H for HDR.
- PS4/PS5 hardware decodes BCn: No performance difference vs uncompressed (decompression in texture units). Always compress.

**Format Selection by Content Type:**
- Color textures (albedo, emission): BC7 (PC/Xbox/PlayStation), ASTC 6x6 (mobile/Switch). Full RGBA support, excellent quality.
- Normal maps: BC5 (PC/Xbox/PlayStation), ASTC 6x6 or BC5 (mobile/Switch). 2-channel optimized, reconstruct Z in shader.
- Masks and grayscale: BC4 (PC/Xbox/PlayStation), ASTC 8x8 (mobile/Switch). Single channel, smallest size.
- HDR textures: BC6H (PC/Xbox/PlayStation), ASTC HDR (mobile/Switch). 16-bit float, only compressed HDR options.
- UI and text: BC7 or ASTC 4x4 for maximum quality. Avoid compression if pixel-perfect required (editor tools).

**Quality Settings:**
- Unity Quality Slider (0-100): Higher = less aggressive compression, larger files, better quality. 50 = balanced. 75 = high quality. 100 = maximum quality (minimal artifacts).
- Test compressed textures visually: Compare in-game vs source. Check for banding (gradients), blockiness (flat colors), or detail loss (text, fine patterns).
- Per-texture quality: Hero assets at 75-100 quality, environment at 50-75, background at 25-50. Balance visual importance with memory budget.
- Use Crunch compression cautiously: Crunch adds CPU-side compression on top of BC/ASTC. Smaller builds, slower load times (decompression cost). Good for downloadable content, bad for streaming.

**Platform-Specific:**
- **PC**: BC7/BC5/BC6H/BC4. Supported on all DX11+ GPUs (GTX 600+, AMD HD 7000+). Fallback to BC1/BC3 for ancient hardware (not recommended in 2025).
- **Xbox Series X/S**: BC7/BC5/BC6H. Hardware support universal. No fallback needed. Use highest quality settings (ample VRAM on Series X).
- **PlayStation 5/4**: BC7/BC5/BC6H. Full hardware support. PS5 recommends BC7 over BC1/BC3 (better quality at same memory).
- **Switch**: ASTC mandatory (no BCn support). Use 8x8 for most textures (memory constrained), 6x6 for hero assets. Test on device (mobile GPU differences).
- **Mobile (iOS/Android)**: ASTC on modern devices (2016+). Fallback ETC2 for older Android, PVRTC for old iOS. Prefer ASTC universally (Unity handles fallbacks).

## Common Pitfalls

**Using BC1/BC3 Instead of BC7**: Developer uses legacy BC1 (DXT1) for color textures or BC3 (DXT5) for alpha textures. BC7 provides significantly better quality at same memory cost. BC1 has severe color banding, BC3 has poor alpha gradients. Symptom: Visible compression artifacts, banding in gradients. Solution: Use BC7 for all RGBA textures on PC/console. BC1/BC3 are obsolete (pre-2011 hardware).

**Normal Maps Compressed as BC7**: Developer compresses normal maps with BC7 (RGBA compression). BC7 treats each channel equally, wasting bits on Z component (reconstructable). BC5 (2-channel) provides 2x better quality for RG channels. Symptom: Normal map artifacts, loss of detail. Solution: Always use BC5 for normal maps. Reconstruct Z in shader: `normal.z = sqrt(max(0, 1 - normal.x * normal.x - normal.y * normal.y))`.

**Uncompressed Textures on Mobile**: Developer leaves mobile textures uncompressed (RGBA32) or uses RGBA16. Switch with 3GB memory loads 10 uncompressed 2K textures = 160MB vs 20MB ASTC 8x8 (8x memory waste). Crashes or texture thrashing. Symptom: Memory errors, crashes on Switch/mobile. Solution: Always compress mobile textures with ASTC. No exceptions except debugging.

## Tools & Workflow

**Unity Texture Import Settings**: Select texture > Inspector > Platform Overrides (Default, PC, Xbox, PlayStation, Android, iOS, Switch). Set Format per platform. Quality slider controls compression quality.

**Texture Compression Wizard**: Custom editor script to batch-process textures. Applies settings based on naming conventions ("_N" suffix = BC5 normal, "_Mask" = BC4, default = BC7). Automates format selection.

**Intel Texture Works Plugin**: Unity plugin for advanced BCn compression. Provides visual preview of compression artifacts. Better quality than default compressor. Download from Intel.

**ARM ASTC Encoder**: Standalone tool for evaluating ASTC compression. Compare 4x4, 6x6, 8x8 block sizes. Produces PNG comparison images. Download from ARM developer site.

**RenderDoc Texture Viewer**: Capture frame, inspect textures in Texture Viewer tab. "Format" column shows runtime format (verify BC7/ASTC applied). "Memory" column shows GPU memory consumption.

**PIX Texture Viewer**: Similar to RenderDoc for Xbox. Inspect texture formats and memory usage. Essential for verifying platform-specific compression.

**Crunch Compression**: Unity option (checkbox in import settings). Applies additional CPU compression on top of BC/ASTC. Reduces build size 20-40%, adds loading cost (0.5-2ms per texture). Use for downloadable content, not streaming textures.

## Related Topics

- [7.1 Texture Optimization](07-01-Texture-Optimization.md) - Texture optimization strategies
- [13.1 Texture Formats and Compression](13-01-Texture-Formats-And-Compression.md) - Advanced compression techniques
- [5.3 Memory Optimization](05-03-Memory-Optimization.md) - Memory reduction
- [5.4 Bandwidth Optimization](05-04-Bandwidth-Optimization.md) - Bandwidth savings from compression

---

[← Previous: Chapter 7](../chapters/07-Asset-Optimization.md) | [Next: 8.2 Mesh Compression →](08-02-Mesh-Compression.md)
