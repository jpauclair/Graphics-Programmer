# 1.3 Primitive Assembly and Rasterization

[← Back to Chapter 1](../chapters/01-Graphics-Pipeline-Fundamentals.md) | [Main Index](../README.md)

Primitive assembly groups vertices into primitives (triangles, lines, points), clips them against view frustum, and rasterizes them into fragments for shading. This fixed-function stage operates at billions of pixels per second.

---

## Overview

After vertex shaders output clip-space positions, the GPU's fixed-function hardware assembles vertices into triangles based on topology (triangle list, strip, fan). Primitives outside the view frustum are clipped, and back-facing triangles are culled to save fragment processing. Surviving triangles undergo viewport transformation (clip space → screen space) and rasterization—determining which pixels each triangle covers.

Rasterization converts mathematical triangles into discrete pixel fragments. The rasterizer generates one fragment per covered pixel, interpolates vertex attributes (UVs, normals, colors) across the triangle, and sends fragments to pixel shaders. Modern GPUs rasterize at incredible rates—Xbox Series X can rasterize ~200 billion pixels/sec. However, small triangles (<10 pixels) waste efficiency due to quad overshading, and degenerate triangles (zero area) cause pipeline stalls.

Understanding rasterization rules (top-left fill convention, subpixel precision) and culling modes (front-face, back-face) is essential for debugging rendering artifacts like missing triangles, Z-fighting, or incorrect transparency sorting. Platform-specific optimizations like hierarchical Z-buffering and conservative rasterization unlock advanced techniques (voxelization, collision detection on GPU).

## Key Concepts

- **Primitive Assembly**: Grouping vertices into geometric primitives based on topology. Triangle list uses 3 vertices per triangle, triangle strip shares edges (1 new vertex per triangle after the first 3).
- **Frustum Clipping**: Removing primitives partially or fully outside the view frustum (near/far planes, left/right/top/bottom). Clipped triangles are split into new triangles that fit entirely within frustum.
- **Back-Face Culling**: Discarding triangles facing away from camera. Determined by vertex winding order (clockwise vs counterclockwise). Saves ~50% fragment shading for closed meshes.
- **Rasterization**: Converting triangles from continuous geometry to discrete pixel fragments. Determines coverage (which pixels overlap triangle), interpolates vertex attributes using barycentric coordinates.
- **Viewport Transformation**: Mapping clip-space coordinates (-1 to 1) to screen-space pixels (0 to screen width/height). Applies depth range transformation for Z-buffer mapping.

## Best Practices

**Primitive Topology Selection:**
- Use triangle lists for general meshes—simpler and more GPU-friendly than strips. Modern index compression and caching make strips obsolete.
- Triangle strips save indices (50% reduction: 5 triangles = 7 verts vs 15) but break on hard edges, requiring degenerate triangles. Cache efficiency gains are minimal on modern GPUs.
- Use indexed primitives always. Non-indexed draws duplicate vertices (cube needs 36 verts instead of 8+36 indices), wasting vertex processing and memory.

**Culling Configuration:**
- Enable back-face culling for all opaque closed meshes (characters, props, buildings). Saves 40-50% fragment shader cost by eliminating invisible back faces.
- Use front-face culling for mirrors or portal rendering (camera inside geometry, rendering outward). Double-sided rendering (no culling) doubles fragment cost—use only when necessary (foliage, cloth).
- Verify winding order correctness—flipped normals cause front faces to be culled. Unity uses counterclockwise front faces; validate in RenderDoc if triangles disappear.
- Disable culling for shadow map rendering if using two-sided geometry or alpha-tested foliage. Culling shadows incorrectly creates light leaks.

**Clipping Optimization:**
- Set appropriate near/far planes. Tight bounds (0.3 near, 1000 far) improve depth precision and reduce clipping work vs extreme ranges (0.01 near, 50000 far).
- Guard band clipping (hardware extension) allows triangles slightly outside viewport without expensive software clipping. Modern GPUs support this automatically.
- For UI or screen-space effects, use orthographic projection with tight bounds to minimize clipping checks.

**Rasterization Quality:**
- Small triangles (<8 pixels) suffer from quad overshading—GPUs process fragments in 2x2 quads for derivative calculations (mipmapping). A 3-pixel triangle activates 4-pixel quad, wasting 25% work.
- Use LOD systems to merge small distant triangles into larger primitives. 1,000 tiny triangles (5 pixels each) render slower than 200 medium triangles (25 pixels each) covering same area.
- Conservative rasterization (optional hardware feature) marks pixels as covered if triangle touches them at all—useful for voxelization or tight collision bounds. Enable via DirectX 12/Vulkan rasterizer state.

**Platform-Specific:**
- **PC DirectX 12**: Control rasterizer state via `D3D12_RASTERIZER_DESC` (FillMode, CullMode, DepthBias). Use `ConservativeRaster` for voxel-based GI or shadow algorithms.
- **Xbox Series X/S**: Hardware supports fast hierarchical Z-culling (Hi-Z). Render opaque front-to-back to maximize early rejection before fragment shading.
- **PlayStation 5**: RDNA 2 rasterizer has excellent small-triangle performance vs older GCN. Still, favor 10-50 pixel triangles for peak efficiency.
- **Switch**: Rasterizer is efficient but limited by bandwidth. Minimize overdraw and prioritize LOD—rasterizing 10M triangles/frame is feasible, but fragment shading bandwidth is constrained.

## Common Pitfalls

**Incorrect Culling Modes**: Setting `CullMode.Front` instead of `CullMode.Back` makes all geometry disappear. Symptoms: scene renders empty, frame debugger shows 0 pixels shaded despite draws executing. Always verify culling in material inspector and test with culling disabled temporarily to isolate issue.

**Degenerate Triangles**: Zero-area triangles (colinear vertices) from bad mesh data or animation blending cause GPU stalls. Rasterizer detects zero area but still processes setup cost. Symptoms: unexplained GPU spikes on specific meshes. Solution: validate meshes during import, reject degenerate faces in asset pipeline.

**Over-Tessellation**: Subdividing surfaces into millions of tiny triangles for "smooth" curves wastes performance. A sphere with 10,000 triangles looks identical to 500 triangles beyond 10m distance. Use LOD and normal maps instead of geometric detail. Modern hardware prefers fewer large triangles over many small ones (better quad utilization, fewer setup costs).

## Tools & Workflow

**RenderDoc**: Mesh Output view shows triangle coverage on screen. Green pixels = covered by triangle, red = fragment shaded. Use to detect overdraw or small triangle inefficiency. Pipeline State > Rasterizer shows cull mode, fill mode, depth bias.

**PIX**: Geometry View renders wireframe overlay on scene. Color-codes triangles by size (green = large, red = small). Use to identify problematic meshes with excessive tiny triangles. "Rasterizer Stats" shows primitives culled vs rasterized.

**Unity Frame Debugger**: Shows draw call mesh and material. Click draw to see mesh in wireframe. If mesh disappears, check culling mode in material. "Render Target" preview shows pixel coverage—black areas indicate no fragments generated (culled or clipped).

**NVIDIA Nsight Graphics**: Range Profiler > Rasterizer shows utilization percentage. <50% suggests vertex-bound or setup-bound (small triangles). >90% indicates rasterizer bottleneck (massive overdraw or extremely dense triangles).

**Platform Profilers**: PlayStation Razor exposes primitive culling counters (backface, frustum, Hi-Z). Xbox PIX shows clip/cull statistics. Use to validate culling effectiveness—if cull rate <30%, front-to-back sorting or Hi-Z isn't working.

## Related Topics

- [1.4 Fragment Processing](01-04-Fragment-Processing.md) - Next pipeline stage after rasterization
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Identifying rasterizer bottlenecks
- [17.1 LOD Fundamentals](17-01-LOD-Fundamentals.md) - Reducing triangle counts with LOD
- [26.2 GPU Debugging](26-02-GPU-Debugging.md) - Debugging rasterization issues

---

[← Previous: 1.2 Vertex Processing](01-02-Vertex-Processing.md) | [Next: 1.4 Fragment Processing →](01-04-Fragment-Processing.md)
