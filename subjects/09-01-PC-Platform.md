# 9.1 PC Platform Asset Guidelines

[← Back to Chapter 9](../chapters/09-Asset-Format-Guidelines.md) | [Main Index](../README.md)

PC platform asset guidelines optimize for wide hardware variance, providing quality settings from min-spec (GTX 1060, 6GB VRAM) to high-end (RTX 4090, 24GB VRAM) while supporting DirectX 11/12 and Vulkan.

---

## Overview

PC is the most diverse platform: CPUs from 4-core (min-spec 2025) to 32-core (enthusiast), GPUs from 6GB VRAM (min-spec) to 24GB (high-end), and APIs from DirectX 11 (compatibility) to DirectX 12/Vulkan (performance). Asset guidelines must provide scalability: Low settings for min-spec (1080p, 30-60fps), Medium for mainstream (1440p, 60fps), High for enthusiast (4K, 60fps), and Ultra for high-end (4K, 120fps with ray tracing).

Texture guidelines: BC7 compression for all color textures (universal DX11+ support), BC5 for normals, BC6H for HDR. Resolution tiers: Low (512-1K), Medium (1K-2K), High (2K-4K), Ultra (4K-8K for hero assets). Provide quality settings UI, auto-detect VRAM capacity, and default appropriately. Mesh complexity: Low (5-10K triangles), Medium (10-20K), High (20-50K), Ultra (50-100K with aggressive LOD).

Shader complexity scales with GPU tier: Low targets GTX 1060 level (50-100 ALU instructions per fragment), High targets RTX series (150-200 instructions, ray tracing). Audio uses Vorbis compression (128kbps music, 96kbps effects). AssetBundles use LZ4 compression (fast loading). Build for both DX11 (compatibility) and DX12 (performance), auto-select at runtime based on GPU capabilities.

## Key Concepts

- **Hardware Variance**: PC spans 10+ years of hardware. GTX 1060 (2016, 6GB VRAM) to RTX 4090 (2024, 24GB VRAM). 4x performance difference between generations. Must support range via quality settings.
- **DirectX Versions**: DX11 (universal compatibility, older GPUs), DX12 (modern low-level API, 30-50% better CPU performance on supported GPUs). Support both for maximum reach.
- **VRAM Tiers**: Low-end (6-8GB), Mid-range (8-12GB), High-end (12-24GB). Texture quality settings must fit within tiers. Auto-detect VRAM, set defaults automatically.
- **Display Resolutions**: 1080p (mainstream), 1440p (enthusiast), 4K (high-end), ultrawide variations. Render resolution scaling essential for performance (render at 70-85% native, upscale with TAA/DLSS).
- **Ray Tracing**: RTX 2000+ series support hardware ray tracing. Optional feature (off by default, enable in settings). Provides reflections, shadows, GI at 20-50% performance cost.

## Best Practices

**Texture Asset Guidelines:**
- Use BC7 for all RGBA textures: Universal support on DX11+ GPUs (2012+). High quality 8:1 compression. Unity Platform Override > PC > BC7.
- Use BC5 for normal maps: 2-channel compression optimal for normals. Reconstruct Z in shader. Higher quality than BC7 for normals.
- Use BC6H for HDR: Only compressed HDR format. Essential for emissive textures, HDRI skyboxes. 6:1 compression, 16-bit float RGB.
- Provide resolution tiers: Low = 512-1K max, Medium = 1K-2K, High = 2K-4K, Ultra = 4K-8K (hero assets only). Expose as Texture Quality setting in options menu.
- Enable mipmaps for all 3D textures: Improves cache efficiency 80-90%. Essential for performance. Disable only for UI textures.

**Mesh Asset Guidelines:**
- Target triangle budgets per quality: Low = 5-15K triangles (characters), Medium = 15-30K, High = 30-60K, Ultra = 60-120K with 4-5 LOD levels.
- Enable mesh compression High: 50% memory reduction, imperceptible quality loss. Import Settings > Mesh Compression = High.
- Implement 4-5 LOD levels: LOD0 (0-10m full detail), LOD1 (10-30m 50% triangles), LOD2 (30-80m 25%), LOD3 (80-150m 10%), LOD4 (150m+ 5% or billboard).
- Use vertex cache optimization: Unity 2021+ optimizes automatically. Improves vertex reuse 40-60% (30-50% performance gain).
- Disable Read/Write: Import Settings > Read/Write Enabled = false. Halves mesh memory (GPU only, no CPU copy).

**Shader and Material Guidelines:**
- Provide shader complexity tiers: Low = simplified shaders (<80 ALU), Medium = standard PBR (<120 ALU), High = advanced PBR + details (<180 ALU), Ultra = full features + ray tracing (<250 ALU).
- Use Shader Variant Collection: Strip unused variants, reduce shader memory 50-80%. Preload used variants to avoid runtime compilation hitches.
- Implement shader LOD: Distance-based shader simplification. Close objects use full shaders, distant use simplified (no normal maps, no detail textures). Unity shader LOD system or custom.
- Limit keywords to 6-8: Each keyword doubles variants. 10 keywords = 1,024 variants = gigabytes of shader memory. Use shader_feature instead of multi_compile where possible.

**Audio Asset Guidelines:**
- Use Vorbis compression: 128kbps for music (near CD quality), 96kbps for ambient, 64kbps for effects. Platform Override > PC > Vorbis.
- Stream music: Load Type = Streaming. Reduces memory 10-50x (loads in chunks vs full PCM). Essential for 3-5 minute tracks.
- Decompress On Load for effects: Short sounds (<5 seconds) decompress at load time, store PCM in memory. Fast playback.
- Use 44.1kHz for music, 22kHz for effects: Halves memory for effects with no perceptible quality loss. Sample Rate Setting = Override > 22050Hz for effects.
- Force mono for 3D sounds: Stereo disrupts 3D spatialization. Use mono sources, Unity AudioSource provides HRTF spatialization. Import Settings > Force To Mono.

**AssetBundle and Loading:**
- Use LZ4 compression: Fast decompression (<5ms per bundle). 2-3x compression. AssetBundle Compression = LZ4.
- Implement Addressables: Automatic dependency management, reference counting, async loading. Replaces manual AssetBundle management.
- Preload critical assets: Player character, weapons, UI during level load. Stream non-critical assets (distant objects, optional content) asynchronously.
- Chunk content into small bundles: 10-50MB per bundle (fast loading, granular streaming). Avoid monolithic 500MB bundles (slow loading, large memory spikes).

**Platform-Specific Considerations:**
- **DirectX 11 vs 12**: Build both. DX12 on Windows 10+ with DX12-capable GPU (2014+). DX11 fallback for older hardware/OS. Auto-detect at runtime.
- **Vulkan Support**: Optional for performance on AMD hardware or Linux. Requires additional testing. Unity supports Vulkan on PC.
- **Ray Tracing**: RTX 2000+ series only. DXR 1.0/1.1 API. Implement as optional setting (off by default, enable in graphics menu). Test performance impact (20-50% cost).
- **DLSS/FSR**: NVIDIA DLSS (RTX series, AI upscaling) or AMD FSR (universal, spatial upscaling). Render at 60-70% native resolution, upscale to target. 40-70% performance gain.
- **HDR Output**: Enable HDR for compatible monitors. HDR10 standard on Windows. Proper tonemapping and color space management required.

## Common Pitfalls

**No Quality Settings**: Developer ships single quality tier targeting high-end PC (4K textures, complex shaders, no LOD). Min-spec users (GTX 1060, 6GB VRAM) cannot run game (VRAM exceeded, low frame rates). Symptom: Negative reviews about performance on "recommended spec" hardware. Solution: Implement Low/Medium/High/Ultra settings, test on min-spec hardware (GTX 1060 or equivalent).

**DX12 Only**: Developer builds DX12-only to leverage modern features. Users with DX11-only GPUs or Windows 7/8 cannot launch game. Symptom: "DX12 not supported" errors, refunds. Solution: Support both DX11 (compatibility) and DX12 (performance). Auto-detect and select appropriate API at launch.

**Fixed Resolution Rendering**: Game renders at native display resolution (4K = 3840x2160). High-end GPUs struggle with 4K at 60fps (fragment bound). Symptom: Low frame rates on 4K displays, even with high-end GPUs. Solution: Implement dynamic resolution scaling (render at 70-85% native, upscale with TAA) or DLSS/FSR support.

## Tools & Workflow

**Quality Settings**: Edit > Project Settings > Quality. Create Low, Medium, High, Ultra presets. Configure texture quality, shadow resolution, LOD bias, anti-aliasing per preset. Expose in options menu.

**Unity Graphics Settings**: Edit > Project Settings > Graphics > Tier Settings. Configure shader quality per platform tier. Low tier = simplified shaders, High tier = full features.

**Platform Build Settings**: Build Settings > PC, Mac & Linux Standalone > Target Platform = Windows. Graphics API: Auto (DX11 + DX12), or explicit (DX12, DX11, Vulkan). Test all configured APIs.

**NVIDIA Nsight Graphics**: GPU debugger for DX12/Vulkan. Capture frames, analyze draw calls, shaders, GPU performance. Essential for optimizing PC builds. Download from NVIDIA developer site.

**RenderDoc**: Open-source graphics debugger. Supports DX11/12, Vulkan, OpenGL. Inspect textures, shaders, pipeline state. Frame capture and analysis. Download from https://renderdoc.org/

**AMD Radeon GPU Profiler**: GPU profiler for AMD hardware. Memory bandwidth, shader performance, bottleneck identification. Download from AMD developer site.

**Steam Hardware Survey**: Valve's survey of Steam users' hardware. Shows GPU distribution, VRAM capacity, display resolutions. Use to set min-spec targets. View at store.steampowered.com/hwsurvey

## Related Topics

- [3.1 DirectX](03-01-DirectX.md) - DirectX API details
- [8.1 Texture Compression Formats](08-01-Texture-Compression-Formats.md) - BC7/BC5/BC6H compression
- [24.1 Mobile and Cross-Platform](09-01-PC-Platform.md) - Cross-platform considerations
- [4.1 PC Profiling Tools](04-01-PC-Profiling-Tools.md) - PIX, RenderDoc, Nsight

---

[← Previous: Chapter 8](../chapters/08-Compression-Techniques.md) | [Next: 9.2 Xbox Platform →](09-02-Xbox-Platform.md)
