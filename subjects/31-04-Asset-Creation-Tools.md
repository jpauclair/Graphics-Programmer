# 31.4 Asset Creation Tools

[← Back to Chapter 31](../chapters/31-Third-Party-Tools.md) | [Main Index](../README.md)

Asset creation tools = DCC (Digital Content Creation) software integration (Maya, Blender, 3ds Max → Unity), texture tools (Substance Designer/Painter = procedural textures), procedural generation (Houdini, World Creator = terrain/vegetation), batch processing (automate exports, optimize assets). Pipeline: artist creates in DCC (modeling, texturing), exports FBX/OBJ (geometry + materials), Unity imports automatically (with textures, animations, prefabs).

---

## Overview

DCC Integration: Unity FBX Exporter (built-in, bidirectional Maya/Blender sync), direct imports (drop .blend/.ma/.max file → auto-converts to FBX). Workflow: artist saves in DCC, Unity detects change (auto-reimport), maintains prefab overrides (scene instances preserved). Materials: automatic material creation (Standard shader by default), or custom import pipeline (ScriptedImporter = parse DCC materials → URP/HDRP).

Substance Tools: Substance 3D Designer (node-based texture creation = procedural, resolution-independent), Substance 3D Painter (3D painting = textures directly on mesh). Integration: Unity Substance plugin (import .sbsar = runtime procedural textures), or bake to PNGs (static textures, faster runtime). Use case: material authoring (AAA textures, consistent look), asset variations (change parameters = new materials).

## Key Concepts

- **FBX Exporter**: Unity package for DCC integration (Package Manager → FBX Exporter). Features: Export to FBX (Unity objects → FBX file for DCC), Import from DCC (bidirectional sync, maintain hierarchy), Round-tripping (Unity → Maya → Unity = preserve data). Workflow: artist models in Maya, exports FBX, Unity imports (meshes, materials, animations). Updates: artist changes model, saves, Unity auto-reimports (prefab instances in scene updated automatically).
- **Blender Integration**: direct .blend file import (Unity reads Blender format, no manual export). Setup: install Blender (Unity detects installation), drag .blend to project (auto-converts to FBX on import). Materials: use FBX export materials (Blender → Export FBX with materials), or Unity import settings (generate Standard materials). Animation: Blender actions → Unity AnimationClips (multiple animations per file). Advantages: faster workflow (no manual export), version control (track .blend files directly).
- **Substance 3D Integration**: procedural texture authoring (Designer = node-based, Painter = 3D painting). Unity plugin: Substance in Unity (Package Manager or Asset Store). Import .sbsar: procedural material (tweak parameters = color, roughness, height), generates textures runtime or bake. Performance: runtime generation (slow, only if parameters change), baked PNGs (normal workflow = designer exports textures, Unity imports). Use case: material variations (one .sbsar → many materials by changing params), consistency (procedural rules = similar look across assets).
- **Houdini Engine**: procedural generation in Unity (Houdini Digital Assets = HDA). Features: node-based workflows (scatter trees, generate roads, terrain erosion), parameters in Unity (inspector controls HDA), live updates (change param → regenerate geometry). Integration: Houdini Engine for Unity (free plugin, requires Houdini license $200/mo). Use case: procedural environments (cities, forests, terrain details), tool creation (level design tools for artists).
- **World Creator**: terrain generation ($99-$199, standalone tool). Features: realistic terrain (erosion, sediment, vegetation), biome system (automatic texture splatting), export to Unity (heightmap, splat maps, object placement). Workflow: design terrain in World Creator (sculpt, erode, paint), export (heightmap .raw + textures), Unity Terrain import (Import Raw, apply textures). Alternative: Gaia Pro ($120 Unity Asset Store = similar features, Unity-native).
- **Batch Processing**: automate asset pipelines (AssetPostprocessor = C# scripts on import). Examples: texture compression (auto-detect diffuse = sRGB, normal = linear), mesh optimization (weld vertices, calculate tangents), prefab creation (FBX import → auto-create prefab). Code: `OnPreprocessTexture`, `OnPostprocessModel`, `OnPostprocessAllAssets`. Use case: enforce standards (all textures compressed, all meshes optimized), save artist time (automatic processing = no manual steps).

## Best Practices

**DCC Workflow**:
- FBX export: bake all transforms (frozen transforms = identity matrices), triangulate meshes (Unity does this anyway, avoid inconsistencies).
- Naming: consistent conventions (lowercase, underscores, no spaces = SM_Rock_01.fbx).
- Pivot points: set correctly in DCC (Unity imports as-is, wrong pivot = rotation issues).
- Scale: work in consistent units (1 Unity unit = 1 meter, set DCC to match).

**Texture Optimization**:
- Substance: bake to PNGs for shipping (runtime generation = slow, only for procedural variations).
- Compression: BC7 (high quality, PC), ASTC (mobile best quality/size), BC3 (normal maps).
- Mip maps: enable for most textures (except UI = no mips needed).

**Procedural Generation**:
- Houdini: bake results for shipping (runtime generation = expensive, only for tools).
- World Creator: export at target resolution (4k heightmap = 4097×4097, matches Unity Terrain resolution).
- Version control: commit generated assets (don't rely on procedural tools at runtime = build consistency).

**Platform-Specific**:
- **PC/Console**: high-res textures (2K-4K diffuse, 1K normals), complex meshes (100K+ tris OK).
- **Mobile**: low-res textures (512-1K), simple meshes (10K tris), aggressive LODs.
- **VR**: optimized assets (reduce poly count = maintain 90 FPS), texture streaming (large worlds).

## Common Pitfalls

**Non-Uniform Scale in DCC**: artist scales object (X=1, Y=2, Z=1) without freezing transforms. Symptom: Unity imports with scale, normals wrong (lighting broken), physics incorrect. Solution: DCC freeze transforms before export (bake scale into vertices, export with uniform scale).

**Missing Textures on Import**: FBX references textures by absolute path (C:\Users\Artist\...). Symptom: Unity imports FBX but textures missing (materials pink). Solution: embed media in FBX (export setting), or place textures in same folder (Unity searches relative path).

**Substance Runtime Generation**: developer uses .sbsar with runtime generation (generates textures every startup). Symptom: load time 10+ seconds (texture generation expensive). Solution: bake textures (Substance → Export textures as PNG), import PNGs into Unity (faster load, same quality).

## Tools & Workflow

**FBX Exporter Setup** (Unity ↔ Maya):
```csharp
1. Package Manager → FBX Exporter (install)
2. Select GameObject → File → Export → Export FBX (save .fbx)
3. Maya: Import FBX (File → Import), edit model
4. Maya: Export FBX (embed media = textures included)
5. Unity: drag FBX to project (auto-imports, creates prefab + materials)
```

**Blender Direct Import**:
```
1. Blender: create model (save .blend file)
2. Unity: drag .blend to Assets folder (auto-converts to FBX internally)
3. Blender: edit model, save (Ctrl+S)
4. Unity: detects change, auto-reimports (prefab instances updated)
```

**Substance 3D Workflow**:
```
1. Substance Designer: create material (node graph, export .sbsar)
2. Unity: Package Manager → Substance 3D for Unity (install plugin)
3. Import .sbsar: drag to project (creates Substance material)
4. Inspector: tweak parameters (base color, roughness, height)
5. Bake: right-click .sbsar → Substance → Export Bitmaps (PNGs for shipping)
```

**Houdini Engine Integration**:
```
1. Install Houdini (Apprentice free, Indie $269/year, full $2000+)
2. Unity: Download Houdini Engine (free plugin)
3. Houdini: create HDA (node graph, expose parameters)
4. Unity: import .hda (creates GameObject with HDA component)
5. Inspector: change parameters (regenerate geometry live)
```

**AssetPostprocessor Automation** (C#):
```csharp
using UnityEditor;
using UnityEngine;

class CustomAssetProcessor : AssetPostprocessor
{
    // Auto-configure texture import
    void OnPreprocessTexture()
    {
        TextureImporter importer = (TextureImporter)assetImporter;
        
        // Detect normal maps (filename contains "_Normal")
        if (assetPath.Contains("_Normal"))
        {
            importer.textureType = TextureImporterType.NormalMap;
            importer.sRGBTexture = false; // Linear
        }
        // Diffuse/Albedo
        else if (assetPath.Contains("_Diffuse") || assetPath.Contains("_Albedo"))
        {
            importer.sRGBTexture = true; // sRGB
            importer.mipmapEnabled = true;
        }
    }
    
    // Auto-optimize mesh import
    void OnPreprocessModel()
    {
        ModelImporter importer = (ModelImporter)assetImporter;
        importer.importNormals = ModelImporterNormals.Calculate; // Recalculate normals
        importer.importTangents = ModelImporterTangents.CalculateMikk; // Tangents for normal mapping
        importer.optimizeMeshPolygons = true; // Merge vertices
    }
}
```

**World Creator Export** (Terrain):
```
1. World Creator: design terrain (sculpt, erode, add vegetation)
2. Export: File → Export → Heightmap (Raw 16-bit), Textures (splatmaps, color map)
3. Unity: Terrain → Import Raw (select heightmap .raw file, set resolution)
4. Terrain Layers: create layers (grass, rock, dirt), assign splatmaps (weight textures)
```

## Related Topics

- [09.1 Texture Import Settings](10-01-AssetBundle-Fundamentals.md) - Texture compression, sRGB
- [09.2 Mesh Import Settings](10-01-AssetBundle-Fundamentals.md) - FBX import configuration
- [25.4 Asset Pipeline Optimization](25-04-Asset-Pipeline.md) - AssetPostprocessor
- [31.1 Shader Tools](31-01-Shader-Tools.md) - Material creation

---

[← Previous: 31.3 Optimization Tools](31-03-Optimization-Tools.md) | [Next: Chapter 32 - Future Proofing →](../chapters/32-Future-Proofing.md)
