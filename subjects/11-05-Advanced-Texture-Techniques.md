# 11.5 Advanced Texture Techniques

[← Back to Chapter 11](../chapters/11-Materials-And-Textures.md) | [Main Index](../README.md)

Advanced texture techniques enhance visual quality through texture arrays, virtual texturing, runtime procedural generation, texture blending, and GPU-accelerated texture manipulation.

---

## Overview

Advanced techniques solve specialized rendering problems: texture arrays eliminate material switching (array of textures indexed by ID, one bind for 128 textures), virtual texturing handles massive textures (stream 16K-32K textures from disk, load visible tiles only), procedural textures generate content at runtime (noise, fractals, patterns without disk storage), and texture blending creates material transitions (terrain blending: grass fades to rock, snow fades to dirt).

Texture arrays store multiple textures in GPU array (Texture2DArray in Unity). Single bind, index via shader parameter (texture2DArray.Sample(sampler, float3(uv, index))). Benefits: eliminates material switches (major performance win), reduces draw calls (all terrain chunks use same terrain array material), and simplifies management (one material for 128 variations). Limitation: all textures must be same resolution and format (2048x2048 BC7).

Virtual texturing (Megatexture, SVT) streams large textures (16K-128K) from disk. GPU renders only visible tiles (128x128 pixel tiles), streaming system loads tiles on-demand (LRU cache, asynchronous loading). Use case: open-world games (unique texturing per surface without repetition), satellite imagery (Google Earth-style rendering), massive terrains (no tiling artifacts). Unity supports virtual texturing via Texture Streaming (automatic mipmap streaming, simpler than full SVT) or third-party plugins (Amplify Imposters, RTP3).

## Key Concepts

- **Texture Arrays**: GPU array of identically-sized textures. Texture2DArray in Unity (Texture2D[] uploaded to GPU array). Shader indexes via float3 UV (uv.xy = texture coordinates, uv.z = array index). All textures same resolution/format.
- **Virtual Texturing**: Streaming massive textures (16K-128K) by loading visible tiles on-demand. GPU renders from tile cache (128-512MB), streaming system loads missing tiles from disk. Avoids loading entire texture into VRAM.
- **Procedural Textures**: Runtime texture generation via shaders or C# code. Noise functions (Perlin, Simplex, Worley), fractals (Mandelbrot), patterns (stripes, checkerboards). No disk storage, infinite resolution (calculated per pixel).
- **Texture Blending**: Mixing multiple textures based on weights. Terrain blending (grass + dirt + rock, blend via splat map), detail blending (base + detail texture), or decal blending (bullet holes over wall texture).
- **Render Textures**: GPU-writable textures. Rendered to by cameras (render-to-texture), compute shaders (procedural generation), or Graphics.Blit (post-processing). Dynamic content (mirrors, security cameras, minimap).

## Best Practices

**Texture Array Implementation:**
- Create Texture2DArray: Code-based creation (new Texture2DArray(width, height, count, format, mips)). Load individual Textures, copy to array (Graphics.CopyTexture(src, 0, 0, array, index, 0)). Set filter/wrap modes.
- Shader sampling: Declare sampler2DArray in shader. Sample via UNITY_SAMPLE_TEX2DARRAY(array, float3(uv, index)). Index passed via vertex color, material property, or instance ID (GPU instancing).
- Indexing strategies: Vertex color alpha (store index 0-255 in vertex alpha), per-instance property (MaterialPropertyBlock.SetFloat("_TextureIndex", index)), or texture index map (R8 texture storing indices).
- Limitations: All textures same resolution (can't mix 2K and 4K), same format (all BC7 or all ASTC), same mipmap count. Arrays up to 2048 textures (platform-dependent: check SystemInfo.maxTextureArraySlices).

**Virtual Texturing Setup:**
- Unity Texture Streaming: Enable Quality Settings > Texture Streaming. Import textures with Streaming Mipmaps = true. Unity loads/unloads mipmaps automatically based on screen coverage. Simpler than full virtual texturing.
- Custom virtual texturing: Implement tile-based system. Divide texture into tiles (128x128 pixels), create tile cache (Render Texture array), stream tiles from disk (AssetBundle or file I/O), update cache LRU (evict least-recently-used tiles).
- Indirection texture: Low-res texture (e.g., 512x512 for 16K texture) mapping UVs to tile cache. Pixel shader samples indirection texture (gets tile coordinates), then samples tile cache. Two texture fetches per pixel (overhead).
- Disk I/O optimization: Compress tiles (BC7/ASTC), store in AssetBundle or custom file format. Load asynchronously (background thread). Target <16ms load time per tile (avoid hitches).

**Procedural Texture Generation:**
- Noise functions: Use Shader Graph noise nodes (Simple Noise, Gradient Noise, Voronoi) or custom implementations (Perlin noise, Simplex noise). Generate textures at runtime (no disk storage, infinite resolution).
- Compute shader generation: Use Compute Shader for complex procedural generation (fractals, cellular automata). Write to RWTexture2D, dispatch threads per pixel (1024x1024 texture = 1024x1024 threads).
- C# generation: Generate Texture2D in C# (nested loop per pixel, SetPixel, Apply). Slow (CPU-bound), use for initialization only (not per-frame). Compute shaders 100-1000x faster.
- Caching: Generate once, cache result (Render Texture or Texture2D asset). Avoid regenerating per frame (expensive). Update only when parameters change.

**Texture Blending Techniques:**
- Splat map blending: Terrain blending via RGBA splat map (R = grass weight, G = dirt weight, B = rock weight, A = snow weight). Sample 4 textures, blend via weights (grass * R + dirt * G + rock * B + snow * A).
- Height-based blending: Blend based on height map (snow on peaks, grass in valleys). Sample height map, compare to threshold, blend textures. Adds verticality to blending (more realistic than splat maps alone).
- Normal-based blending: Blend based on surface angle (grass on flat surfaces, rock on steep slopes). Use surface normal (dot with up vector), blend textures. Avoids manual splat map painting.
- Detail blending: Overlay detail texture on base (multiply, overlay, or linear dodge blend modes). Detail adds high-frequency noise (scratches, dirt). Blend via detail intensity slider (0 = no detail, 1 = full detail).

**Render Texture Usage:**
- Camera render-to-texture: Assign Render Texture to Camera.targetTexture. Camera renders to texture instead of screen. Use for mirrors (second camera renders reverse view), security cameras (minimap cameras), portals (recursive rendering).
- Compute shader output: Declare RWTexture2D in Compute Shader. Write pixels via imageStore (HLSL) or result[id.xy] = color (Unity). Use for procedural generation, image processing, GPU particles.
- Graphics.Blit: Copy/process textures via material. Graphics.Blit(source, dest, material). Material shader processes each pixel (post-processing, blur, color grading). Fast (GPU-accelerated).
- Memory management: Render Textures allocate VRAM (1024x1024 RGBA = 4MB). Release when unused (RenderTexture.Release()). Use RenderTexture pools (reuse textures) to avoid allocation churn.

**Platform-Specific:**
- **PC**: Full support for all techniques. Texture arrays up to 2048 slices, virtual texturing via custom implementation or Texture Streaming. Compute shaders for procedural generation (fast on modern GPUs).
- **Consoles**: Full support. Texture arrays common for terrain (256-512 textures). Virtual texturing via Texture Streaming (fast SSDs enable streaming). Compute shaders efficient (GCN/RDNA on PS, Turing/Ampere on Xbox).
- **Switch**: Limited texture arrays (512 slices max). No virtual texturing (slow storage, limited memory). Simple procedural textures acceptable (noise, gradients). Avoid complex compute shaders (Maxwell GPU slower).
- **Mobile**: Basic texture arrays (256 slices, check SystemInfo.maxTextureArraySlices). No virtual texturing (slow storage, limited memory). Simple procedural textures only (noise). Compute shader support limited (newer devices only).

## Common Pitfalls

**Texture Array Resolution Mismatch**: Developer creates Texture2DArray with mixed resolutions (some 2K, some 1K). Unity throws error ("All textures must be same resolution"). Array creation fails. Symptom: Exception during Texture2DArray creation, array not created. Solution: Resize all textures to same resolution (2048x2048) before copying to array. Use Texture2D.Resize or Graphics.CopyTexture with scaled source.

**Virtual Texturing Without Fallback**: Developer implements virtual texturing, but doesn't handle missing tiles (tile not loaded yet). Shader samples invalid tile cache entry (black texture, artifacts). Symptom: Black spots on terrain, flickering textures during camera movement. Solution: Implement fallback (sample low-res base texture when tile missing). Indirection texture stores "tile loaded" flag, shader checks before sampling cache.

**Per-Frame Procedural Generation**: Developer generates procedural texture every frame (Perlin noise in Update()). Texture generation costs 5-20ms (CPU-bound), kills frame rate. Symptom: Profiler shows SetPixels/Apply taking milliseconds, low frame rate. Solution: Generate once, cache result. Or use Compute Shader (GPU-accelerated, <1ms). Update only when parameters change, not every frame.

**Excessive Render Textures**: Developer creates Render Texture per object (100 mirrors = 100 Render Textures at 1024x1024 = 400MB VRAM). Out-of-memory on consoles/mobile. Symptom: Memory Profiler shows 400MB+ in Render Textures, crashes on memory-limited platforms. Solution: Reduce Render Texture count (shared mirrors = single Render Texture rendered multiple times). Lower resolution (mirrors = 512x512 instead of 1024). Release unused Render Textures.

## Tools & Workflow

**Texture2DArray Creation**: C# script using Texture2DArray constructor, Graphics.CopyTexture to populate. Unity has no built-in Texture2DArray asset (code-only). Third-party tools (Texture Array Creator on Asset Store) simplify creation.

**Shader Graph Texture Array**: Sample Texture2D Array node in Shader Graph. Drag Texture2DArray into node, connect UV + Index inputs. Visual workflow for texture array shaders (no code required).

**Compute Shader Editor**: Visual Studio or Rider for Compute Shader code (HLSL). Unity compiles .compute files automatically. Dispatch via ComputeShader.Dispatch(kernel, threadGroupsX, Y, Z). Use for procedural generation, image processing.

**RenderDoc/PIX**: Capture frame, inspect texture arrays (view individual slices), Render Textures (see rendered content), procedural textures (verify generation). Debug sampling issues (wrong index, invalid UV).

**Unity Profiler**: Memory Profiler > Texture shows Texture2DArray memory, Render Texture allocations. CPU Profiler > Rendering shows Compute Shader dispatch times, texture generation costs.

**Amplify Shader Editor**: Visual shader editor (Asset Store) with advanced texture nodes (texture arrays, triplanar, parallax, blending). More features than Shader Graph (pre-built complex techniques).

**Substance Designer**: Procedural texture authoring. Create graph-based procedural textures (exported to Unity as Texture2D or runtime-generated). Alternative to runtime procedural generation (baked procedural textures).

## Related Topics

- [11.3 Texture Mapping Techniques](11-03-Texture-Mapping-Techniques.md) - Basic texture sampling
- [11.4 Material LOD](11-04-Material-LOD.md) - Texture streaming and LOD
- [7.1 Texture Optimization](07-01-Texture-Optimization.md) - Texture compression
- [12.4 Advanced Shader Techniques](../chapters/12-Shader-Programming.md) - Shader implementation

---

[← Previous: 11.4 Material LOD](11-04-Material-LOD.md) | [Next: Chapter 12 →](../chapters/12-Shader-Programming.md)
