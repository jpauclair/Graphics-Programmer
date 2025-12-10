# 11.1 Material System Architecture

[← Back to Chapter 11](../chapters/11-Materials-And-Textures.md) | [Main Index](../README.md)

Unity's material system manages shader properties, texture sampling, render state, and GPU-side rendering pipeline configuration through Material assets and shader variants.

---

## Overview

Materials define surface appearance: colors (albedo, emission), textures (diffuse, normal, metallic maps), shader selection (Standard, URP Lit, custom shaders), and render settings (blend mode, culling, depth testing). Each Material asset references Shader asset, exposes shader properties (floats, colors, textures), and generates GPU state blocks (blend modes, stencil operations, depth functions). Without materials, meshes render with default pink shader (missing material indicator).

Material system workflow: create Material asset (Project > Create > Material), assign Shader (dropdown selector or drag-drop shader asset), configure properties (Inspector exposes shader properties as UI fields), assign to Renderer (MeshRenderer.material or .sharedMaterial). Runtime: Unity batches draw calls by material (objects with same material batch together), uploads material properties to GPU (constant buffers, texture bindings), and switches materials when needed (state changes, breaking batches).

Shader variants complicate material system: single shader compiles to hundreds of variants (platform combinations, keyword permutations, quality settings). Material selects variant at runtime (enabled keywords, platform, quality level). Variant explosion causes long build times (compile all variants) and runtime hiccups (shader compilation on first use). Management critical: strip unused variants (Shader Variant Collection), preload common variants (warmup), and audit variant counts (Graphics Settings > Shader Stripping).

## Key Concepts

- **Material Asset**: Unity asset storing shader reference and property values. Serialized as .mat file (YAML format). Contains shader GUID, property overrides (floats, colors, textures), render queue, and keywords.
- **Shader Properties**: Exposed shader variables (uniform inputs). Types: float/int (metallic, smoothness), Color (tint, emission), Texture2D (albedo, normal map), Vector (UV offsets, tiling). Set via Material.SetFloat/SetColor/SetTexture.
- **Material Instancing**: Each object can have unique material (material property modifications). Unity creates material copy per instance (expensive). Use MaterialPropertyBlock for per-instance properties without duplication.
- **Shared Material**: Multiple objects reference same Material asset (MeshRenderer.sharedMaterial). Changes affect all objects. Efficient (no duplication), but no per-object customization. Use for static objects (environments, props).
- **Shader Variants**: Compiled shader permutations. Keywords (#pragma multi_compile, #pragma shader_feature) generate variants. Example: NORMALMAP keyword creates 2 variants (with/without normal map). N keywords = 2^N variants (exponential).

## Best Practices

**Material Organization Strategies:**
- Per-material approach: Each object type has dedicated material (WoodMaterial, StoneMaterial, MetalMaterial). Textures unique per material. Simple, but requires many materials (100-500 in large games).
- Material instances: Base materials with property variations (BaseMetal: steel = blue tint, copper = orange tint). Use sharedMaterial for base, SetColor() for tint. Reduces material count, enables batching.
- Atlased materials: Pack multiple textures into atlas (all wood textures in WoodAtlas 4096x4096). Single material references atlas, UV offsets select sub-texture. Maximum batching (all objects use same material).
- Material libraries: Organize by category (Materials/Environment/Stone, Materials/Characters/Skin). Use prefixes (ENV_Stone, CHAR_Skin). Simplifies searching, avoids naming collisions.

**Shader Property Access:**
- Use Shader.PropertyToID: Convert property name strings to int IDs (Shader.PropertyToID("_MainTex")). Store ID in static variable. 10x faster than string lookup (avoid string hashing per frame).
- MaterialPropertyBlock: Per-renderer property overrides without material duplication. Use for per-instance colors, UVs, floats. Example: renderer.SetPropertyBlock(block). Efficient instancing (GPU batching maintained).
- Avoid material.property: Accessing material.color or material.mainTexture creates material copy (breaks batching). Use sharedMaterial for read-only access, MaterialPropertyBlock for per-instance writes.
- Batch property updates: Call SetFloat/SetColor once per material update, not per property. Material.SetPass() uploads all properties to GPU (minimize SetPass calls).

**Shader Variant Management:**
- Identify variants: Build project, check Editor.log for shader compilation stats ("Compiled 1,234 shader variants"). High counts (>10,000) indicate variant explosion.
- Strip unused variants: Graphics Settings > Shader Stripping. Disable unused features (fog, lightmaps, instancing if not used). Reduces build times (faster compilation), runtime memory (fewer variants loaded).
- Shader Variant Collection: Create collection asset (Project > Create > Shader Variant Collection). Add used variants manually or via "Save to asset" in Graphics > Shader Compilation. Preload collection at startup (ShaderVariantCollection.WarmUp()).
- Keyword audit: Count keywords per shader (each #pragma multi_compile adds variant). Limit to <8 keywords per shader (2^8 = 256 variants). Use #pragma shader_feature for optional features (strips unused variants).

**Runtime Material Management:**
- Preload materials: Instantiate materials during loading screen (Resources.LoadAll<Material>("Materials")). Avoids runtime allocation hitches (material creation = disk I/O + shader variant compilation).
- Pool material instances: Create material pool for dynamic objects (particle systems, decals). Reuse materials instead of Instantiate(material) per object. Reduces GC pressure (material destruction = garbage).
- Monitor material count: Use Memory Profiler to track material instances. Duplicate materials waste memory (1MB per material with 2K textures). Identify duplicates, use sharedMaterial.
- Material LOD: Swap materials based on distance. Close objects = high-quality material (PBR, normal maps, detail textures), distant objects = simple material (unlit, albedo only). Reduces texture sampling, shader ALU.

**Platform-Specific:**
- **PC**: Support all shader variants (high-end GPUs handle complexity). Use shader LOD (Shader.maximumLOD) to disable expensive variants on low-end GPUs (GTX 1060).
- **Consoles**: Optimize variant counts (long build times on console compilers). Xbox/PlayStation compilers slower than PC (10-30 minutes for 10,000 variants). Strip aggressively.
- **Switch**: Minimize variants (limited shader cache, 3GB memory). Strip fog, lightmap variants if unused. Favor simple shaders (Unlit, Mobile/Diffuse). Complex variants cause hitches (shader compilation stalls).
- **Mobile**: Use Mobile shaders (Shader Graph > Mobile category). Fewer variants, optimized instructions. Strip all unused variants (build size critical, <150MB limit).

## Common Pitfalls

**Accessing material.property**: Developer changes color via renderer.material.color = Color.red. Unity creates material copy per object (breaks batching). 100 objects = 100 material instances (wastes memory, kills performance). Symptom: Profiler shows 100 SetPass calls (one per material), Frame Debugger shows no batching. Solution: Use MaterialPropertyBlock. block.SetColor("_Color", Color.red); renderer.SetPropertyBlock(block). Maintains batching (objects share material).

**Shader Variant Explosion**: Developer adds 10 multi_compile keywords. Shader generates 2^10 = 1,024 variants. Build time explodes (30 minutes compiling shaders). First-time variant use hitches game (200-500ms compilation). Symptom: Long builds, runtime hitches when new material appears, "Compiling shader variant" messages in log. Solution: Reduce keywords to <8. Use shader_feature instead of multi_compile (strips unused variants). Preload variants via ShaderVariantCollection.

**Not Using Shader.PropertyToID**: Developer calls material.SetFloat("_Metallic", value) every frame. String hashing costs 5-10µs per call (1000 calls = 5-10ms CPU). Symptom: Profiler shows Material.SetFloat taking milliseconds in CPU time. Solution: Cache property ID. static int metallicID = Shader.PropertyToID("_Metallic"); material.SetFloat(metallicID, value). 10x faster (int lookup vs string hash).

## Tools & Workflow

**Material Inspector**: Default Unity Inspector for Materials. Shows shader dropdown, exposed properties (floats, colors, textures), render queue, keyword toggles. Use for manual material editing.

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows draw calls, material assignments, batching status. Identify material-related issues (excessive SetPass calls, broken batches, material duplication).

**Memory Profiler**: Captures material instances. Native Objects > Material shows all loaded materials, memory usage, reference counts. Identify duplicate materials (same shader + properties = should share).

**Shader Variant Collection**: Asset for preloading variants. Create via Project > Create > Shader Variant Collection. Add variants manually or capture from Editor/build. Load at startup (Resour ces.Load, call WarmUp()) to precompile variants.

**Graphics Settings**: Edit > Project Settings > Graphics. Shader stripping settings (Fog, Lightmaps, Instancing, LOD). Tier settings (per-quality shader features). Use to reduce variant counts globally.

**Material Analyzer (Asset Store)**: Third-party tools for analyzing materials. Find unused materials, duplicate materials, expensive shaders. Optimize material organization, reduce asset bloat.

**Shader Graph**: Visual shader editor (Window > Shader Graph). Create shaders without code (node-based). Generates optimized shader variants automatically. Recommended for artists (no shader code required).

## Related Topics

- [11.2 PBR Material Properties](11-02-PBR-Material-Properties.md) - Material property details
- [11.3 Texture Mapping Techniques](11-03-Texture-Mapping-Techniques.md) - UV mapping and sampling
- [12.1 Unity Shader Languages](../chapters/12-Shader-Programming.md) - Shader programming
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - GPU optimization techniques

---

[← Previous: Chapter 10](../chapters/10-Asset-Bundling-And-Streaming.md) | [Next: 11.2 PBR Material Properties →](11-02-PBR-Material-Properties.md)
