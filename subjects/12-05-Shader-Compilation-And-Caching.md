# 12.5 Shader Compilation and Caching

[← Back to Chapter 12](../chapters/12-Shader-Programming.md) | [Main Index](../README.md)

Shader compilation transforms ShaderLab+HLSL source into platform-specific GPU bytecode through variant generation, cross-compilation, and caching systems to avoid runtime hitches.

---

## Overview

Shader compilation pipeline: Unity parses ShaderLab+HLSL (syntax validation), generates variants (keyword permutations: #pragma multi_compile creates 2^N variants), cross-compiles to target platforms (DirectX DXIL, Vulkan SPIR-V, Metal AIR, OpenGL GLSL), optimizes bytecode (driver-level optimization, instruction scheduling), and caches results (Library/ShaderCache for editor, per-platform shader cache for builds). Without caching, every shader recompiles on import (10-60 seconds per shader, unbearable for iteration).

Variant explosion problem: N keywords = 2^N variants. 10 keywords = 1,024 variants (long build times, runtime hitches when uncached variant first used). Example: Standard Shader compiles 10,000+ variants (fog, lightmaps, shadows, reflections, etc.). Management critical: strip unused variants (Shader Stripping in Graphics Settings), preload common variants (ShaderVariantCollection.WarmUp), and audit variant counts (check Editor.log for "Compiled X shader variants").

Runtime compilation causes hitches: first time material/shader appears, Unity compiles variant on-demand (100-500ms hitch, visible frame drop). Solutions: preload shaders during loading screen (ShaderVariantCollection.WarmUp), pre-warm shader variants (manually trigger compilation before gameplay), or build-time compilation (include all variants in build, no runtime compilation). Platform shader caches store compiled variants (avoid recompilation across runs).

## Key Concepts

- **Shader Variants**: Compiled permutations of shader based on keywords. Keywords (#pragma multi_compile FOG_ON FOG_OFF) create variants (one with fog, one without). Unity compiles needed variants only (shader feature stripping eliminates unused variants).
- **Shader Cache**: Local storage for compiled shaders. Editor cache: Library/ShaderCache (per-project, speeds up reimport). Build cache: platform-specific (PC: %LOCALAPPDATA%/Unity/cache, mobile: device-local). Persistent across runs (avoids recompilation).
- **Shader Stripping**: Removes unused variants at build time. Graphics Settings > Shader Stripping (Fog: strip if not used, Lightmaps: strip unused modes). Reduces build size (fewer variants), faster builds (less compilation), smaller runtime cache.
- **ShaderVariantCollection**: Asset listing shader variants for preloading. Create via Project > Create > Shader Variant Collection. Add variants manually (shader + keywords), or capture from editor/build ("Save to asset" in Graphics > Shader Compilation). Load and WarmUp() during loading screen (precompiles variants, avoids runtime hitches).
- **Async Compilation**: Unity 2021+ feature. Compiles shaders asynchronously (background thread), displays placeholder while compiling (cyan shader or previous frame's result). Avoids hitches but introduces visual popping (shader appears after compilation). Configure via Edit > Preferences > Shader Compilation.

## Best Practices

**Variant Management:**
- Minimize keywords: Limit to <8 keywords per shader (2^8 = 256 variants manageable). Excessive keywords = exponential explosion (2^10 = 1,024, 2^15 = 32,768 variants, unbuildable).
- Use shader_feature over multi_compile: #pragma shader_feature FOG_ON strips FOG_ON variant if not used by any material. #pragma multi_compile FOG_ON FOG_OFF always compiles both (even if unused). shader_feature for material-controlled features (normal maps, emission), multi_compile for runtime-toggled features (quality settings, platform switches).
- Audit variant counts: Build project, check Editor.log for "Compiled N shader variants" per shader. High counts (>1,000) indicate problems. Investigate keywords (which contribute most variants?), strip unused features.
- Shader Stripping configuration: Graphics Settings > Shader Stripping. Disable unused features globally (Fog: strip if no fog in scenes, Lightmaps: strip modes not used, Instancing: strip if not using instancing). Applies to all shaders (reduces variants universally).

**Shader Preloading:**
- ShaderVariantCollection creation: Create SVC asset, add used variants (shader + keyword combination). Automatic capture: Graphics > Shader Compilation > "Save to asset" after playing game (captures shaders used during play session).
- WarmUp during loading: Load SVC at startup (Resources.Load<ShaderVariantCollection>), call shaderCollection.WarmUp(). Compiles all variants synchronously (loading screen blocks, 1-5 seconds). Gameplay smooth (no hitches, all variants precompiled).
- Incremental SVC updates: Periodically capture new shaders during development (play game, "Save to asset", merge with existing SVC). Keeps SVC up-to-date as project evolves (new shaders, new materials).
- SVC per scene: Create separate SVCs per level (Level1SVC, Level2SVC). Load SVC for current level only (avoids preloading unused shaders). Reduces WarmUp time (1-2 seconds instead of 5 seconds for all shaders).

**Build-Time Compilation:**
- Include all variants in build: Disable shader stripping (Graphics Settings > Shader Stripping > Keep All). Compiles every variant at build time (long builds, 10-30 minutes), but zero runtime compilation (no hitches).
- Trade-off: Build time vs runtime hitches. Mobile: strip aggressively (builds faster, smaller APK/IPA), accept runtime compilation (async or preload). PC/console: include more variants (longer builds acceptable, runtime performance priority).
- Player Settings > Preload Shaders: Include shaders in preload list (always loaded at startup). Use for common shaders (UI, Standard, Skybox). Avoids first-use hitches.
- Addressables integration: Bundle shaders with content (level AssetBundle includes shaders). Shaders load with level (compiled when bundle loaded, not when material first appears).

**Async Compilation Configuration:**
- Enable async: Edit > Preferences > Shader Compilation > Asynchronous Shader Compilation. Compiles shaders on background thread (no frame drops), displays placeholder (cyan shader or previous frame).
- Placeholder options: Preferences > Shader Compilation > Show Placeholder > Cyan (bright cyan, obvious), Skip Rendering (invisible until compiled), Use Custom (specify fallback shader). Choose based on aesthetic (cyan obvious for debugging, skip rendering for shipped builds).
- Synchronous fallback: Some situations require synchronous compilation (shadows, depth-only passes). Unity falls back to blocking compilation (brief hitch). Minimize by preloading critical shaders.
- Mobile considerations: Async compilation less effective on mobile (single-core shader compiler, background compilation still impacts frame rate). Prefer preloading (ShaderVariantCollection.WarmUp).

**Cache Management:**
- Editor cache location: Library/ShaderCache. Safe to delete (regenerates on next import, slows down first import but fixes cache corruption). Automatic cleanup (Unity removes old entries).
- Build cache location: Platform-dependent. PC: %LOCALAPPDATA%/Unity/cache, Android: /data/data/com.company.game/cache, iOS: Library/Caches. Persists across app runs (faster subsequent launches).
- Clear build cache: Delete cache folder to force recompilation (fixes corrupted cache, shader bugs). Some platforms: clear via app settings (Android: clear app cache, iOS: delete app).
- Cache size monitoring: Editor cache can grow large (1-5GB for large projects). Periodically delete Library/ShaderCache to reclaim space. Build caches smaller (10-100MB typically).

**Platform-Specific:**
- **PC**: Fast shader compilation (DirectX shader compiler optimized). Editor iteration fast (1-5 seconds per shader). Build caches persistent (%LOCALAPPDATA%), shared across builds (first run compiles, subsequent runs cached).
- **Consoles**: Slower compilation (platform-specific compilers). Xbox/PlayStation compilers take 10-30 minutes for 10,000 variants. Strip aggressively (reduce variant counts). Build caches on dev kits (persistent).
- **Switch**: Slow compilation (NVNGFX compiler). Builds take 20-60 minutes (shader compilation bottleneck). Minimize variants (<500 per shader). Cache on dev kit (persistent, avoids recompilation).
- **Mobile**: Very slow compilation (driver-level, on-device). First launch compiles shaders (30-120 seconds, device warms up). Preload shaders (ShaderVariantCollection.WarmUp during splash screen). Cache on device (persistent, but users may clear cache).

## Common Pitfalls

**Variant Explosion**: Developer adds 12 multi_compile keywords. Shader generates 2^12 = 4,096 variants. Builds take 60+ minutes (compiling shaders), runtime hitches when new variants appear (100-500ms per variant). Symptom: Very long builds, "Compiling shader variants" in log, runtime hitches. Solution: Reduce keywords to <8. Use shader_feature instead of multi_compile (strips unused variants). Audit variant counts in Editor.log.

**No Shader Preloading**: Developer ships game without ShaderVariantCollection. Players experience hitches when new materials appear (first enemy spawn, first explosion, entering new area = 100-500ms hitch). Symptom: Frame drops on first occurrence (first enemy, first effect), smooth on subsequent occurrences. Solution: Create ShaderVariantCollection (capture variants during play testing), load and WarmUp() during loading screen or startup.

**Async Compilation Without Placeholder**: Developer enables async compilation, uses "Skip Rendering" placeholder. Objects invisible until shaders compile (characters/enemies appear 1-5 seconds after spawning, confusing players). Symptom: Objects pop in late, invisible during async compilation. Solution: Use "Cyan" placeholder (obvious, debugging) or preload shaders (ShaderVariantCollection.WarmUp, avoids async compilation entirely).

**Shader Cache Corruption**: Developer experiences pink shaders (missing material indicator), or shaders rendering incorrectly after Unity update. Shader cache corrupted (old cached shaders incompatible with new Unity version). Symptom: Pink shaders, rendering artifacts, shaders look wrong. Solution: Delete Library/ShaderCache (editor cache). Delete build cache (PC: %LOCALAPPDATA%/Unity/cache, mobile: clear app cache). Unity recompiles shaders from source.

## Tools & Workflow

**Shader Compilation Log**: Check Editor.log after builds. Search for "Compiled X shader variants" per shader. Identifies shaders with excessive variants (candidates for optimization). Log location: Edit > Preferences > External Tools > Reveal in Finder/Explorer (log file in parent directory).

**Graphics Settings**: Edit > Project Settings > Graphics > Shader Stripping. Configure global stripping (Fog, Lightmaps, Shadows, Instancing, LOD). Tier Settings (shader features per quality tier). Reduces variants universally.

**ShaderVariantCollection Asset**: Project > Create > Shader Variant Collection. Add variants manually (Add Variant button: select shader, choose keywords). Or capture automatically (Graphics > Shader Compilation > "Save to asset" after play session).

**Shader Compilation Preferences**: Edit > Preferences > Shader Compilation. Enable/disable async compilation, configure placeholder (Cyan/Skip Rendering/Custom). Adjust compilation timeout (slow machines need longer timeout).

**Frame Debugger**: Shows compiled shader variants per draw call. Inspect keywords (which variant used?), verify correct variant selected. Useful for debugging variant issues (wrong keywords enabled, expected variant not compiled).

**Shader Variant Collection Manager**: Asset Store tools for advanced SVC management (auto-capture during builds, merge SVCs, analyze variant usage). Simplifies SVC workflow (no manual "Save to asset" clicks).

**RenderDoc/PIX**: Capture compiled shader bytecode. View assembly (DXIL, SPIR-V, GCN ISA), instruction counts, register usage. Verify shaders compiled correctly (no driver bugs, optimizations applied).

## Related Topics

- [12.1 Unity Shader Languages](12-01-Unity-Shader-Languages.md) - HLSL and ShaderLab syntax
- [12.2 Shader Types](12-02-Shader-Types.md) - Surface, vertex/fragment shaders
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Shader performance optimization
- [4.1 PC Profiling Tools](04-01-PC-Profiling-Tools.md) - RenderDoc for shader debugging

---

[← Previous: 12.4 Advanced Shader Techniques](12-04-Advanced-Shader-Techniques.md) | [Next: Chapter 13 →](../chapters/13-Rendering-Techniques.md)
