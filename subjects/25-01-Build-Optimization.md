# 25.1 Build Optimization

[← Back to Chapter 25](../chapters/25-Build-Pipeline-And-Workflow.md) | [Main Index](../README.md)

Build optimization reduces time (30+ min full desktop build = unacceptable), binary size (1GB download = barrier to entry on mobile), and memory usage (build process = RAM intensive). Strategies: incremental compilation (recompile only changed files, cache intermediates), batch asset processing (compress textures, generate LOD offline), shader stripping (remove unused variants = smaller build), script stripping (remove unused C# code = smaller binary). Cloud builds = distributed compilation (expensive but parallel = fast).

---

## Overview

Build Pipeline: (1) asset processing (textures compressed, meshes optimized), (2) shader compilation (all variants compiled), (3) C# compilation (scripts compiled to IL), (4) IL2CPP conversion (IL -> C++ -> binary, console/mobile), (5) linking (resolve dependencies), (6) packaging (APK/IPA/EXE creation). Bottlenecks: shader compilation (takes 5-10 min for large projects), texture compression (takes 3-5 min), IL2CPP (takes 3-10 min for large projects). Total: 30-60 min full build typical AAA. Solution: incremental (only rebuild changed), cloud builds (parallel compilation across workers), or accept slow builds and optimize frequently-built targets (quick playable builds for testing, full builds for submission).

Memory: build process = asset import = threads consuming RAM, can peak 8-16 GB for large projects. Mitigation: disable heavy imports (enable only needed), profile build memory, upgrade CI machine RAM if needed.

## Key Concepts

- **Incremental Builds**: recompile only changed files (asset modified timestamp checked, if newer = re-import). Benefit: subsequent builds 1-2 min (vs 30+ min full). Challenge: dependency tracking (if texture changed, any shader using texture = rebuild, but shader cache miss = full recompile). Solution: dependency graphs (track all asset dependencies, invalidate entire chain if asset changes), or conservative = always full rebuild if any asset changed (simpler, slower).
- **Shader Stripping**: at build time, remove unused shader variants (multi_compile creates many = 10-100x variants per shader, mobile/low-end don't need all). Methods: (1) build report analysis (track which variants used in project), (2) manual stripping (white list used variants), (3) shader_feature (only compile if keyword defined in scene = automatic stripping). Benefit: project 10-50 unused variants per shader = remove = 30-50% binary size reduction, faster load times.
- **Script Stripping**: IL2CPP analyze project code, remove unreachable C# code (private methods not called = unused = strip). Benefit: 20-40% binary size reduction, faster startup (less code to load). Risk: reflection-based code broken (FindObjectsOfType via reflection might fail if stripped). Solution: link.xml = mark methods "don't strip", or use strip settings (Disabled = keep all, Aggressive = maximum stripping + risk).
- **Texture Atlasing Offline**: pre-build atlases (combine 100 individual textures = 1 atlas), avoid runtime performance cost (runtime = expensive). At build time = fast. Benefit: draw calls reduced (1 material 1 draw call vs 100 materials 100 calls), memory saved (no padding overhead). Tool: texture packer (Substance Alchemist, TexturePacker = external), or scripted (Unity C# script reading textures, generating atlas, reimporting).
- **Cloud Builds**: distributed compilation (build farm = 10-50 workers), each compiles subset of shaders/scripts in parallel. Benefit: 10x faster (1 hour -> 6 min with 10 workers). Cost: cloud service fees, setup complexity (need build service integration). Used by: AAA studios (reduce iteration time), cloud gaming platforms (Builds per day = time cost adds up).

## Best Practices

**Shader Compilation Optimization**:
- Use shader_feature (keyword active only if used) vs multi_compile (all variants compiled). shader_feature = local keywords, only compile variants used in current scenes.
- Batch shader compilation (compile in chunks, parallelize = faster than serial). Editor > Project Settings > Graphics > Shader Compilation = parallel on/off.
- Target shader validation (verify minimal variants in build = not all 1000 variants). Build Report analysis.

**Incremental Build Strategies**:
- Asset Database tracking enabled (Edit > Project Settings > Asset Pipeline > Accelerator enabled for speed).
- Only import changed assets (if texture modified = re-import, dependent shaders = rebuild).
- Fast playable build (editor test = quick iteration, separate from full build for submission).

**Binary Size Reduction**:
- Script stripping: Edit > Project Settings > Player > Optimization > Strip unused code = on (with link.xml for reflection cases).
- Shader stripping: remove unused variants (shader_feature, build report analysis).
- Texture compression: all textures ASTC mobile, BC compressed on desktop, minimize atlas size.
- Remove unused plugins (e.g., ad SDK not used = strip from build).

**Memory During Build**:
- Raise concurrent import workers limit (if RAM available), but monitor memory (peak should <80% total system RAM).
- Disable heavy sub-assets (Editor > Preferences > Asset Importers > disable unnecessary importers).
- Use 64-bit editor (more RAM addressable) vs 32-bit (capped 4GB).

## Common Pitfalls

**All Shader Variants Compiled**: Developer uses multi_compile without limiting. Project = 100 shaders * 50 variants = 5000 total. Build time = 60 min, binary = 2GB. Symptom: build slow, binary too large. Solution: convert to shader_feature (keywords only compile if used), or manually limit variants (disable unneeded platforms/features).

**Reflection Code Stripped**: Developer uses FindObjectsOfType or GetComponent via reflection, script stripping enabled. Build = code stripped = reflection fails at runtime = crash. Symptom: runtime crash (NullReferenceException on reflection call). Solution: disable stripping for affected assembly (link.xml), or refactor to avoid reflection (direct references preferred).

**Full Rebuild on Every Change**: Developer's build system doesn't track dependencies properly = every asset change triggers full rebuild. Iteration slow (30+ min per build). Solution: implement dependency tracking (asset database metadata), or use Addressables (incremental asset loading = faster iteration).

## Tools & Workflow

**Build Report Analysis**: Window > Analysis > Build Report, see binary breakdown (textures, meshes, shaders, code = sizes). Identify largest assets = optimize first.

**Shader Variant Analysis**: Build > Play > Profiler > Frame Debugger = shows which shader variants actually used in playtest. Compare to total compiled = identify strippable variants.

**Script Stripping Configuration** (link.xml):
```xml
<linker>
  <assembly fullname="Assembly-CSharp">
    <type fullname="MyClass" preserve="all"/>
  </assembly>
</linker>
```

**Incremental Build Scripting** (EditorBuildSettings):
```csharp
ublic class BuildOptimizer {
    public static void BuildIncremental() {
        BuildPlayerOptions opts = new BuildPlayerOptions {
            scenes = EditorBuildSettingsScene.GetActiveScenes(),
            target = BuildTarget.Android,
        };
        BuildPipeline.BuildPlayer(opts);
    }
}
```

## Related Topics

- [25.2 Version Control](25-02-Version-Control.md) - Project organization
- [25.3 Continuous Integration](25-03-Continuous-Integration.md) - Automated builds
- [25.4 Asset Pipeline](25-04-Asset-Pipeline.md) - Asset import processing
- [07.1 Texture Optimization](07-01-Texture-Optimization.md) - Texture compression

---

[Previous: Chapter 24](../chapters/24-Mobile-And-Cross-Platform.md) | [Next: 25.2 Version Control →](25-02-Version-Control.md)
