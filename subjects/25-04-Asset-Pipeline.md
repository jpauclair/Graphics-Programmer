# 25.4 Asset Pipeline & Processing

[← Back to Chapter 25](../chapters/25-Build-Pipeline-And-Workflow.md) | [Main Index](../README.md)

Asset pipeline = automated workflow from artist-created source (PSD, FBX, WAV) to game-ready format (DDS, mesh data, OGG). Steps: (1) import source (Photoshop PSD), (2) validate (check resolution, format), (3) process (compress texture, optimize mesh), (4) cache (store processed version), (5) package (include in build). Benefits: one-click = all assets processed consistently, no manual conversion, faster iteration (edit source = auto-reprocess). Custom importers = specialized handling (procedural mesh generation, texture atlas creation).

---

## Overview

Asset Workflow: artist saves PSD in project Assets folder, Unity detects (file system monitor), importer runs, processes, caches result. Next import = use cache (fast). If artist edits = re-import (detects change timestamp). Pipeline = declarative (settings specify behavior = no coding needed), or scriptable (C# post-processor = custom logic).

Performance: thousands of assets = import time substantial (5-10 min initial import on new machine = populate cache). Subsequent builds = cached = fast. Parallel import threads (configurable = balance CPU vs quality).

## Key Concepts

- **Importers**: built-in (Texture, Mesh, Audio importers = standard settings), or custom (ModelImporter extension = specialized processing). Each asset type = dedicated importer (FBX = mesh + animation importer, PNG = texture importer). Settings: compression (ASTC, BC), resolution (1024x1024 target), filtering (trilinear), MipMap (on/off).
- **Post-Processors**: C# scripts (IPreprocessor/IPostprocessor) = run after import (modify result before caching). Example: auto-scale high-res textures to target resolution, or generate LOD meshes automatically. Benefit: consistency (no manual tweaks = all textures same quality).
- **Import Presets**: save importer settings as preset (can apply to multiple assets = batch settings). Example: "Mobile-Texture" preset = compression ASTC, max 512x512 resolution. Apply to all mobile textures = consistency.
- **Addressables**: asset reference system (replaces Resources folder = deprecated). Assets tagged with addresses ("Character/Hero"), loaded by address (not path). Benefit: asset movement safe (reference by name not path = refactoring easy), streaming support, memory management explicit.
- **Version Control Integration**: import settings stored in .meta files (Unity metadata = source control friendly). If artist and programmer both edit asset = .meta merge conflict possible (rare if organized properly).

## Best Practices

**Importer Settings**:
- Texture: compression = platform-specific (ASTC mobile, BC desktop, ETC2 older Android). Max resolution = 2048 typical (higher = memory risk). MipMaps = on (avoid aliasing at distance).
- Mesh: optimization = on (compress vertex data, reduce bone influences), collision = off (generate separately if needed), smoothing groups = on (preserves normal direction).
- Audio: compression = OGG/OPUS (smaller than WAV), mono if possible (stereo = 2x size, not always needed).

**Post-Processor Strategy**:
- Normalize: all textures go through processor (validate format, rescale if needed = consistency).
- Auto-LOD: mesh importer processor generates LOD automatically (LOD0 source, LOD1/2/3 generated = no manual work).
- Atlasing: group textures into atlases (processor detects groups = combine automatically = fewer draw calls).

**Presets for Consistency**:
- Create tier presets: "High-Quality", "Mid-Quality", "Low-Quality" = texture resolution/compression differs. Assign to project assets based on target platform.
- Per-platform: override default preset on Android = force ASTC, iOS = force PVRTC = platform-specific optimization without per-asset tweaking.

**Addressables Workflow**:
- Tag assets (mark as Addressable), assign address ("UI/Button", "Models/Enemy"), group (streaming group = load together).
- Reference by address ("UI/Button" loaded = automatic, path not needed).
- Memory explicit (add/remove references = load/unload = control).

## Common Pitfalls

**Importer Settings Wrong**: texture importer set to "Uncompressed" (debug mode, forgot to change). All textures = 4 bytes per pixel = VRAM bloat. Symptom: game runs low memory warning. Solution: audit imports (batch change back to compressed).

**Post-Processor Infinite Loop**: processor modifies asset = triggers re-import = processor runs again = infinite loop. Build hangs. Solution: guard conditions (processor checks if already processed = skip).

**Asset Address Changed**: asset addressed "UI/Button", later renamed to "UI/UIButton". Code still loads "UI/Button" = not found = fail. Symptom: addressable not found runtime error. Solution: deprecation warning (code checks old address = warning logged), or refactoring tool (rename all references).

## Tools & Workflow

**Texture Import Settings** (common presets):
```csharp
// Mobile preset: ASTC compression, 512x512 max
var importer = (TextureImporter)AssetImporter.GetAtPath(path);
importer.textureCompression = TextureImporterCompression.CompressedHQ;
importer.crunchedCompression = true;
importer.maxTextureSize = 512;
importer.SaveAndReimport();
```

**Mesh LOD Auto-Generation**:
```csharp
public class MeshPostprocessor : AssetPostprocessor {
    void OnPostprocessModel(GameObject g) {
        var importer = (ModelImporter)assetImporter;
        importer.importNormals = ModelImporterNormals.ComputeIfMissing;
        importer.optimizeMesh = true;
    }
}
```

**Addressable Setup**:
```csharp
// Load addressable asset
var handle = Addressables.LoadAssetAsync<GameObject>("UI/Button");
var go = handle.WaitForCompletion();

// Or async
Addressables.LoadAssetAsync<GameObject>("UI/Button").Completed += (op) => {
    var obj = op.Result;
};
```

## Related Topics

- [25.1 Build Optimization](25-01-Build-Optimization.md) - Build pipeline
- [25.2 Version Control](25-02-Version-Control.md) - Asset management
- [25.3 Continuous Integration](25-03-Continuous-Integration.md) - Automated pipeline
- [10.1 AssetBundle Fundamentals](10-01-AssetBundle-Fundamentals.md) - Asset bundling

---

[Previous: 25.3 Continuous Integration](25-03-Continuous-Integration.md) | [Next: Chapter 26 →](../chapters/26-Debugging-Techniques.md)
