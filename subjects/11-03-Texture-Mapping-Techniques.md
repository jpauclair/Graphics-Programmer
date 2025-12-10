# 11.3 Texture Mapping Techniques

[← Back to Chapter 11](../chapters/11-Materials-And-Textures.md) | [Main Index](../README.md)

Texture mapping techniques project 2D textures onto 3D surfaces through UV coordinates, tiling, sampling modes, and advanced techniques like triplanar and parallax mapping.

---

## Overview

Texture mapping assigns texture pixels (texels) to mesh vertices via UV coordinates. UV coordinates (U = horizontal, V = vertical) range 0.0-1.0, mapping to texture space (0,0 = bottom-left, 1,1 = top-right). Each mesh vertex stores UV coordinate (Vector2), GPU interpolates UVs across triangles (rasterization), fragment shader samples texture at interpolated UV (texture2D(tex, uv) in GLSL). Result: texture projected onto 3D surface.

UV mapping strategies: non-overlapping UVs (unique texture space per triangle, used for lightmaps), overlapping UVs (multiple triangles share texture space, tiling textures), and UV channels (Unity supports 4 UV sets: UV0-UV3, used for different textures on same mesh). Tiling and offsetting: material properties control texture repeat (Tiling = 2,2 repeats texture 4x) and offset (Offset = 0.5,0 shifts texture half-width right). Sampling modes affect texture filtering: Point (nearest neighbor, blocky), Bilinear (lerp between 4 texels, smooth), Trilinear (bilinear + mipmap blending, smoothest).

Advanced techniques solve specific problems: triplanar mapping (automatic UVs from world position, no UV seams), parallax mapping (fakes depth via height map, more detail than normal maps), detail mapping (high-frequency detail overlaid on base texture, improves close-up appearance), and texture arrays (array of textures sampled via index, avoids bind changes). Platform considerations: mobile prefers simpler techniques (UV mapping + tiling), consoles support all techniques, PC handles complex shaders (parallax, triplanar).

## Key Concepts

- **UV Coordinates**: 2D texture coordinates per vertex. U = horizontal (0.0 left, 1.0 right), V = vertical (0.0 bottom, 1.0 top). Stored in mesh vertex buffer (Vector2 per vertex). GPU interpolates across triangles.
- **Tiling and Offset**: Material properties controlling texture repetition and shift. Tiling (2,2) repeats texture 4 times (2x horizontal, 2x vertical). Offset (0.5,0) shifts texture half-width right. Controls via material._MainTex_ST (scale/translation).
- **Sampling Modes**: Texture filtering methods. Point (nearest texel, sharp), Bilinear (interpolate 4 texels, smooth), Trilinear (bilinear + mipmap blending, smoothest, avoids LOD popping). Set in Texture Import Settings > Filter Mode.
- **Wrap Modes**: Behavior at UV edges. Repeat (tiles texture, default), Clamp (extends edge texels, no tiling), Mirror (mirrors texture, seamless tiling). Set per texture axis (U/V independently).
- **UV Channels**: Multiple UV sets per mesh. UV0 (main textures), UV1 (lightmaps), UV2 (detail maps), UV3 (custom). Shader samples different channels (TEXCOORD0, TEXCOORD1, TEXCOORD2, TEXCOORD3).

## Best Practices

**UV Layout Optimization:**
- Minimize seams: UV seams (where UVs disconnect) create texture discontinuities. Visible as lines on surface. Place seams in hidden areas (back of objects, under overlaps). Use UV islands sparingly (more islands = more seams).
- Maximize space: Fill UV space (0-1 range) efficiently. Wasted space = lower texture resolution per surface area. Use packing algorithms (DCC tools, RizomUV, Unfold3D) to tighten island packing.
- Texel density: Maintain consistent texel density (texels per world-space unit). Character face = high density (important detail), character back = low density (less visible). Inconsistent density = blurry areas next to sharp areas.
- Non-overlapping for lightmaps: Lightmap UVs (UV1) must be non-overlapping (each triangle gets unique lightmap space). Overlaps cause lighting artifacts (multiple triangles write to same lightmap texels). Unity generates UV1 automatically (Mesh Import > Generate Lightmap UVs).

**Tiling Texture Techniques:**
- Seamless textures: Tiling textures (bricks, wood planks, concrete) must be seamless (tile edges match). Use tools (Photoshop Offset filter, Substance Designer tiling nodes) to create seamless tiles.
- Tiling values: Material tiling controls repetition. Tiling = (10,10) repeats texture 100 times (10x10 grid). Use for large surfaces (floors, walls). Adjust tiling to avoid obvious repetition (vary tiling per material instance).
- Detail tiling: Overlay detail texture (high-frequency noise, scratches) with different tiling (DetailTiling = 50,50 while MainTiling = 1,1). Breaks up repetition, adds close-up detail.
- Avoid obvious patterns: Tiled textures often show visible grid (human eye detects patterns). Break with detail maps, vertex color variation, decals, or triplanar mapping.

**Sampling Mode Selection:**
- Point filtering: Use for pixel art (retro aesthetic), UI (crisp text), stylized games (voxel art). Avoids blur but causes aliasing (jagged edges at distance).
- Bilinear filtering: Default for most textures. Smooth interpolation between texels. Cheap (2-4 texture fetches). Use for albedo, metallic, emission.
- Trilinear filtering: Use for textures with mipmaps (all 3D textures). Blends between mipmap levels (avoids LOD popping). Slightly more expensive (4-8 fetches) but essential for quality.
- Anisotropic filtering: Improves texture sharpness at oblique angles (floor textures viewed at shallow angle). Performance cost (8-16 fetches), but modern GPUs handle well. Enable globally (Quality Settings > Anisotropic Textures = Per Texture or Forced On).

**Advanced Mapping Techniques:**
- Triplanar mapping: Project texture from 3 axes (X, Y, Z), blend based on surface normal. Avoids UV seams, automatic UVs. Use for terrain, rocks, organic shapes. Cost: 3x texture samples (one per axis). Shader implementation: sample texture 3 times with world positions, blend by abs(normal).
- Parallax mapping: Offset UVs based on height map and view angle. Fakes depth (cracks, bricks, stones). Parallax Occlusion Mapping (POM) = advanced version (ray-marches height map). Cost: 10-30 texture samples (POM). Use sparingly (hero assets, close-up surfaces).
- Detail mapping: Overlay high-frequency detail texture (scratches, noise) on base texture. Detail texture tiles at higher frequency (DetailTiling = 20 vs MainTiling = 1). Multiplies or overlays onto albedo. Cheap (2 texture samples), effective for close-up detail.
- Decals: Project textures onto surfaces (bullet holes, graffiti, dirt). Use Render Texture projection or Decal Projector (URP/HDRP). Avoids baking details into base textures (flexibility, runtime placement).

**Platform-Specific:**
- **PC**: All techniques supported. Trilinear + anisotropic filtering standard. Parallax mapping acceptable (RTX GPUs handle 30-50 samples easily). 4K textures common (2048-4096px).
- **Consoles**: Full feature support. Use trilinear filtering, anisotropic 4x-8x. Parallax mapping for hero assets only (distant objects = simple UV mapping). 2K textures typical (1024-2048px).
- **Switch**: Simplified techniques. Bilinear filtering (trilinear expensive on Maxwell GPU). No parallax mapping (ALU cost too high). Detail mapping acceptable. 1K textures (512-1024px), ASTC compression.
- **Mobile**: Basic UV mapping + tiling. Point or bilinear filtering only (trilinear expensive). No advanced techniques (parallax, triplanar = frame rate killer). 512-1K textures, ASTC 8x8 compression.

## Common Pitfalls

**UV Seams Visible**: Artist creates UV layout with seams on visible surfaces (character face seam down center). Textures show line at seam (pixel mismatch). Symptom: Visible line on model, texture discontinuity. Solution: Place seams in hidden areas (back of head, under arms, inside mouth). Use UV welding (stitch seams in DCC tool), paint over seams in texture (manual fix).

**Overlapping Lightmap UVs**: Developer uses UV0 for lightmaps (has overlapping UVs from tiling textures). Multiple triangles write to same lightmap area (lighting artifacts, black spots). Symptom: Black splotches on model, incorrect lighting. Solution: Use UV1 for lightmaps. Enable "Generate Lightmap UVs" in Mesh Import Settings (creates non-overlapping UV1 automatically).

**Excessive Tiling**: Developer sets Tiling = (100,100) on floor texture. Texture tiles 10,000 times (obvious repetition, grid pattern visible). Looks artificial ("video game floor"). Symptom: Repetitive texture pattern, player notices tiling. Solution: Reduce tiling (10-20 typical), add detail maps (break repetition), use decals (dirt, cracks), or triplanar mapping (automatic variation).

**Point Filtering on 3D Textures**: Developer sets Filter Mode = Point for albedo textures (thinks it saves performance). Textures look blocky (no interpolation), aliasing at distance (shimmering pixels). Symptom: Pixelated, retro appearance (unintended), texture aliasing/shimmering. Solution: Use Bilinear or Trilinear filtering. Point filtering only for pixel art (intentional aesthetic).

## Tools & Workflow

**UV Editing (DCC Tools)**: Blender (UV Editing workspace), Maya (UV Editor), 3ds Max (Unwrap UVW modifier). Manually lay out UVs, place seams, optimize packing. Export FBX with UVs to Unity.

**Automatic UV Generation**: Unity Mesh Import > Generate Lightmap UVs. Creates UV1 channel automatically (non-overlapping, lightmap-ready). Parameters: Hard Angle (seam threshold), Pack Margin (island padding), Angle Error/Area Error (quality).

**RizomUV**: Dedicated UV unwrapping tool. Fast auto-unwrap (ZenUnwrap algorithm), optimized packing. Industry standard for UV optimization. www.rizom-lab.com.

**Substance Painter**: Texture painting with automatic UV preview. Paint directly on 3D model, textures map to UVs automatically. Identifies UV seams, allows seamless painting (no visible seams).

**Shader Graph**: Visual shader editor with texture mapping nodes. Main Texture (UV input, Tiling/Offset properties), Triplanar node (automatic triplanar mapping), Parallax Occlusion Mapping node (POM implementation). Build custom mapping techniques visually.

**Unity Frame Debugger**: Inspect texture sampling. View bound textures, see UV coordinates per vertex (Mesh Preview), verify tiling/offset values (Material Properties). Debug mapping issues (wrong UV channel, incorrect tiling).

**GPU Profilers (RenderDoc/PIX)**: Capture frame, inspect texture samples. View UV coordinates passed to pixel shader, see texture fetches. Identify excessive sampling (parallax mapping = 30+ samples), optimize accordingly.

## Related Topics

- [11.2 PBR Material Properties](11-02-PBR-Material-Properties.md) - Texture maps for PBR
- [11.4 Material LOD](11-04-Material-LOD.md) - Texture LOD techniques
- [7.1 Texture Optimization](07-01-Texture-Optimization.md) - Texture compression and resolution
- [12.2 Shader Types](../chapters/12-Shader-Programming.md) - Shader texture sampling

---

[← Previous: 11.2 PBR Material Properties](11-02-PBR-Material-Properties.md) | [Next: 11.4 Material LOD →](11-04-Material-LOD.md)
