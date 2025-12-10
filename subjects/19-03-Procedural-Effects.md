# 19.3 Procedural Effects

[← Back to Chapter 19](../chapters/19-Visual-Effects.md) | [Main Index](../README.md)

Procedural effects generate visuals algorithmically: compute shaders for simulation (fluids, cloth, destruction), shader effects (dissolve, distortion, holograms), and runtime mesh generation.

---

## Overview

Procedural vs Baked: Traditional VFX uses baked animations (particle systems, flipbook textures, pre-animated meshes), procedural generates at runtime (compute shaders simulate physics, fragment shaders generate patterns, vertex shaders deform geometry). Benefits: infinite variation (no repetition, each instance unique), responsive (reacts to gameplay, dynamic environment), memory-efficient (algorithms < texture data, single shader vs gigabytes of textures). Drawbacks: GPU cost (runtime computation expensive, balance quality vs performance), complexity (shader programming required, harder than placing particle emitters).

Compute Shader Effects: GPU simulation (fluids, cloth, particles, destruction). Compute dispatches (C# script dispatches compute shader, GPU processes data in parallel, writes results to buffers/textures). Example: fluid simulation (2D grid, compute shader updates pressure/velocity, fragment shader renders fluid surface), cloth (vertex positions in buffer, compute applies forces/constraints, mesh renderer displays result). Unity integration: ComputeShader asset (C# `ComputeShader.Dispatch()` executes), read/write buffers (RWStructuredBuffer, RWTexture3D, GPU modifies data).

Shader-Based Effects: Fragment shader effects (screen-space distortion, dissolve, holograms, shield impacts), vertex shader deformation (water waves, flag waving, pulsing). Runtime generation (no pre-baked data, pure math), parametric control (shader properties adjust effect: dissolve amount, wave frequency, distortion intensity). Examples: dissolve effect (noise texture + threshold, fragments discard when below threshold, object fades away), hologram (scanline pattern + color shift + transparency, sci-fi aesthetic), distortion (UV offset based on normal map, refractive glass/heat haze).

## Key Concepts

- **Compute Shaders**: HLSL programs for general GPU computation (not rendering, data processing). Structure: `#pragma kernel KernelName` (entry point), `[numthreads(X,Y,Z)]` (thread group size), kernel function (void KernelName(uint3 id : SV_DispatchThreadID)). Dispatch from C#: `computeShader.Dispatch(kernelIndex, groupsX, groupsY, groupsZ)` (launches threads, groupsX × numthreads.x = total threads in X). Buffers: `RWStructuredBuffer<float4> data` (read/write structured buffer, GPU modifies), `RWTexture2D<float4> output` (read/write texture, GPU writes pixels). Use cases: particle simulation, mesh deformation, fluid dynamics, procedural texture generation.
- **Structured Buffers**: GPU buffers with structured data (arrays of structs, arbitrary data types). C# side: `ComputeBuffer buffer = new ComputeBuffer(count, stride)` (count = element count, stride = bytes per element), `buffer.SetData(arrayData)` (upload CPU data to GPU), `computeShader.SetBuffer(kernelIndex, "BufferName", buffer)` (assign to compute shader). Shader side: `StructuredBuffer<MyStruct> inputBuffer` (read-only), `RWStructuredBuffer<MyStruct> outputBuffer` (read-write), access via index: `outputBuffer[id.x] = computeValue(inputBuffer[id.x]);` Dispose: `buffer.Dispose()` (release GPU memory, call when done).
- **Procedural Noise**: Algorithm-generated patterns (Perlin, Simplex, Voronoi, fractal noise). Shader functions: `float noise = PerlinNoise(uv * scale)` (generates 0-1 noise value, UV = position), fractal noise (sum multiple octaves, `noise += PerlinNoise(uv * scale * pow(2, octave)) * amplitude;`, creates detail). Use cases: dissolve masks (noise > threshold = discard pixel), terrain height (vertex displacement from noise), cloud generation (3D noise for volumetric clouds). Unity shader library: no built-in Perlin (implement custom, or use noise textures for performance).
- **Vertex Displacement**: Vertex shader modifies positions (waves, explosion, morphing). Input: mesh vertices (positions, normals), shader offsets positions (add noise-based offset, apply forces, deform along normal). Example: water wave (sine wave on Y, `vertex.y += amplitude * sin((vertex.x + _Time.y) * frequency);`), flag waving (sine wave on X/Z, vertex color controls influence = edges wave more). Performance: cheap (vertex processing fast, even 10,000 verts = <1ms), limited detail (tessellation required for smooth displacement on low-poly meshes).
- **Dissolve Effect**: Gradually removes object (edge glow + discard pixels). Implementation: noise texture (grayscale, used as dissolve mask), threshold (float property 0-1, controls dissolve amount), fragment shader (sample noise, `clip(noise - _DissolveAmount)` discards pixels below threshold), edge glow (if near threshold, add emission color = glowing edge as dissolve progresses). Variation: directional dissolve (noise + gradient, dissolve from bottom to top), dual-threshold (outer edge + inner edge, two-color glow).
- **Screen-Space Distortion**: UV offset to simulate refraction (heat haze, glass, magic portals). Implementation: distortion normal map (stores XY offset in RG channels), fragment shader (sample distortion map, offset UVs, sample scene color texture with offset UVs = distorted background). Intensity (multiply offset by intensity property, control distortion strength). Shader: `float2 distortion = tex2D(_DistortionMap, uv).rg * 2 - 1; float2 distortedUV = screenUV + distortion * _Intensity; float4 sceneColor = tex2D(_CameraOpaqueTexture, distortedUV);` URP: use Camera Opaque Texture (URP provides, scene color before transparent pass).

## Best Practices

**Compute Shader Setup:**
- Thread group sizing: numthreads(8,8,1) for 2D (64 threads per group, typical for image processing), numthreads(64,1,1) for 1D (particle systems, array processing), numthreads(8,8,8) for 3D (voxel processing, volumetrics). Total threads: dispatched groups × numthreads = total (1024 particles, numthreads(64,1,1), dispatch(16,1,1) = 16×64 = 1024 threads). GPU optimization: thread groups execute on wavefronts (NVIDIA = 32 threads, AMD = 64, align numthreads for efficiency).
- Buffer management: Create buffers in Start/Awake (one-time allocation, reuse every frame), SetData only when needed (CPU-to-GPU upload expensive, avoid per-frame if data unchanged), GetData sparingly (GPU-to-CPU readback extremely slow, synchronization stall, use only for debugging or end-of-frame results). Dispose on destroy (OnDestroy: buffer.Dispose(), release GPU memory).
- Dispatch optimization: Dispatch per frame for dynamic effects (particle simulation, fluid), or dispatch on-demand (static effects, bake once, reuse result), or async dispatch (dispatch, continue CPU work, GPU processes in parallel, check completion later). Profiler: GPU Profiler shows compute shader timing (validate <1-2ms for 60 FPS).

**Shader Effect Design:**
- Dissolve parameters: Noise texture (1024x1024 grayscale, tiling enabled, high-frequency noise for fine detail), Dissolve Amount (0-1 property, animated via script or Timeline), Edge Width (0.05-0.2, controls glow thickness), Edge Color (HDR color, intensity >1 for bloom = bright edge). Animation: `material.SetFloat("_DissolveAmount", Mathf.Lerp(0, 1, time))` (fade out over time), or reverse (1 → 0 = fade in).
- Distortion setup: Normal map (RG format, tangent-space normals unused, only XY offset stored), Intensity (0.01-0.1 typical, higher = extreme distortion), Tiling (distortion map tiling, 1x1 = once, 2x2 = repeated pattern). Animate: scroll UVs (`_Time.y * scrollSpeed` offset, animated distortion), or animate intensity (pulse distortion, `sin(_Time.y)` oscillation).
- Vertex displacement limits: Low-poly meshes (displacement blocky, tessellation required for smooth result, or use high-poly mesh), normal recalculation (displaced vertices need updated normals, compute in shader or recalculate in C#), bounds (displaced mesh exceeds original bounds, set mesh.bounds manually or objects culled incorrectly).
- Performance: Shader complexity (keep instructions low, mobile = <50 ALU instructions, PC = <200), texture samples (minimize, each sample = bandwidth, 2-4 samples typical, 10+ samples = expensive), branching (avoid dynamic branches in fragment shader, use static branches or branchless math: `lerp(A, B, condition)` instead of `if`).

**Fluid Simulation (2D):**
- Grid representation: 2D texture (each pixel = grid cell, R = velocity X, G = velocity Y, B = pressure), resolution (128x128 = fast, 512x512 = detailed, 1024x1024 = expensive). Compute shader: reads current state (velocity + pressure textures), applies Navier-Stokes (pressure solver, diffusion, advection), writes new state (updated textures).
- Simulation steps: Advection (move velocity field along itself, `newVel = sampleVel(uv - vel * dt)`), Diffusion (blur velocity, simulates viscosity), Pressure solve (iterative Jacobi, enforces incompressibility, 20-50 iterations typical), Projection (subtract pressure gradient, makes velocity divergence-free). Per-frame dispatch (multiple passes, each step = one compute dispatch).
- Rendering: Fragment shader samples velocity texture (visualize as color, `color.rg = velocity.xy`, blue/red = direction), or height field (velocity drives height, render as waves), or particles (particles advected by velocity field, GPU particles follow fluid flow).
- Boundaries: Solid boundaries (edges of texture = walls, compute shader clamps velocity at borders), obstacles (SDF or mask texture, velocity zeroed inside obstacles, fluid flows around).

**Cloth Simulation:**
- Vertex buffer: Store cloth vertices in ComputeBuffer (positions, velocities), initialize from mesh (mesh.vertices copied to buffer). Grid connectivity (cloth = grid of vertices, compute shader knows neighbors: left, right, up, down).
- Constraints: Distance constraints (maintain edge length, `delta = (currentDist - restDist) * stiffness`, move vertices closer/apart), bending constraints (resist folding, cross-diagonal springs), collision (sphere colliders, plane colliders, compute shader pushes vertices outside).
- Integration: Verlet integration (simple, no velocity storage, position-based), or semi-implicit Euler (velocity explicit, position implicit, stable for stiff springs). Forces: gravity (add `float3(0, -9.8, 0) * deltaTime` to velocity), wind (noise-based force, turbulence), damping (reduce velocity, simulate air resistance).
- Rendering: Update mesh vertices (buffer.GetData copies GPU buffer to CPU array, mesh.vertices = updated positions, mesh.RecalculateNormals), or GPU instancing (keep vertices on GPU, use DrawProcedural with vertex buffer, zero CPU copy). Performance: 1000-5000 vertices typical (more = expensive, limit for 60 FPS), simplify for distant cloth (LOD reduces vertex count).

**Mesh Generation:**
- Procedural meshes: Runtime generation (C# code creates vertices/triangles/UVs), assign to MeshFilter (mesh.vertices, mesh.triangles, mesh.uv, mesh.RecalculateNormals). Use cases: terrain (heightmap to mesh), voxel worlds (Minecraft-like, cubes generated on-demand), ribbons/trails (dynamic trail mesh behind moving object).
- Mesh data: vertices (Vector3 array, positions), triangles (int array, indices in CCW winding), UVs (Vector2 array, texture coordinates), normals (Vector3 array, or RecalculateNormals auto-calculates), tangents (Vector4 array, for normal mapping, or RecalculateTangents).
- Performance: Mesh.Clear (clear before rebuild, resets data), SetVertices/SetTriangles (faster than assigning arrays, avoids allocations), MarkDynamic (if mesh updated frequently, tells GPU to optimize for dynamic updates). Limit frequency (rebuild once per frame max, or on-demand only, constant rebuilding = slow).

**Platform-Specific:**
- **PC**: Full procedural effects (compute shaders standard, DX11+, complex simulations viable), high-resolution (fluid sim 512x512, cloth 5,000 verts, compute budget 5-10ms). Compute: DirectX 11/12 (fast compute, RTX GPUs = tensor/RT cores available for specialized effects), Vulkan (explicit control, advanced optimization possible).
- **Consoles**: Compute shaders supported (PS4/5, Xbox One/Series X, moderate complexity), medium-resolution (fluid 256x256, cloth 2,000 verts, compute budget 3-5ms). Platform-specific (direct API access, optimization via platform libraries, leverage async compute on PS5/Series X).
- **Switch**: Compute shaders available but slow (Tegra X1 compute limited, low-resolution only), minimal procedural (fluid 64x64, cloth 500 verts, <2ms budget), prefer baked (pre-baked animations often better, avoid complex compute). Shader effects (dissolve, distortion viable, vertex displacement okay, avoid compute-heavy effects).
- **Mobile**: Compute shaders limited (high-end only, many devices unsupported or slow), avoid procedural compute (use baked animations, flipbooks instead of simulation). Shader effects viable (dissolve, simple distortion, vertex displacement for water, keep shaders simple <50 instructions). Exceptions: GPU particles in URP (supported, but simpler than VFX Graph, acceptable performance).

## Common Pitfalls

**Compute Buffer Leaks**: Developer creates ComputeBuffer (allocates GPU memory), forgets to Dispose (memory leak, buffer remains allocated). After many allocations, GPU memory exhausted. Symptom: Memory Profiler shows ComputeBuffer accumulating (hundreds of MB), eventual crash (out of VRAM). Solution: Always Dispose (OnDestroy or OnDisable: `buffer?.Dispose()`), or use `using` statement (if temporary: `using (var buffer = new ComputeBuffer(...)) { /* use */ }`, auto-disposes).

**GetData Synchronization Stall**: Developer calls buffer.GetData every frame (thinking needs CPU data for logic), frame rate tanks (GetData forces GPU-to-CPU sync, GPU waits for compute completion, CPU waits for transfer = full pipeline stall). Symptom: Profiler shows WaitForGPU spike (milliseconds waiting), frame time increases (10ms added just for readback). Solution: Avoid GetData (keep data on GPU, render directly, or use GPU Events/callbacks), or async readback (AsyncGPUReadback.Request, non-blocking, result available next frame, eliminates stall).

**Thread Group Mismatch**: Developer dispatches compute (Dispatch(16, 16, 1), thinking 16x16 = 256 threads), but numthreads is (8,8,1), actual threads = 16×8 × 16×8 = 1024 threads (4x more than intended). Buffer overflow (writing beyond buffer size, corruption), or wrong results (threads accessing invalid indices). Symptom: Compute results wrong (artifacts, NaN values), occasional crashes (GPU fault). Solution: Calculate correctly (total threads = Dispatch(groupsX, groupsY, groupsZ) × numthreads(X,Y,Z), ensure groupsX = ceil(dataCount / numthreads.x)), or use defines (`#define THREADS 64`, `Dispatch(dataCount / THREADS, 1, 1)`), validate (`id.x < dataCount` check in shader, early return if out of bounds).

**Dissolve Seams**: Developer applies dissolve shader (noise-based clip), seams appear (hard edges between triangles, noise samples different per vertex, discontinuous clip). Symptom: Triangular gaps during dissolve (edges visible, mesh tears instead of smooth fade). Solution: Sample noise in fragment shader (not vertex, per-pixel sampling = smooth), increase noise resolution (low-res noise = blocky, 1024x1024+ = smooth), or world-space UVs (noise sampled in world space, consistent across mesh, `uv = worldPos.xy * scale`, no seams at mesh boundaries).

## Tools & Workflow

**Compute Shader Creation**: Assets > Create > Compute Shader (creates .compute file). Edit: Visual Studio/Rider (HLSL syntax highlighting), define kernel: `#pragma kernel MyKernel`, implement: `[numthreads(8,8,1)] void MyKernel(uint3 id : SV_DispatchThreadID) { /* logic */ }` Assign: C# script references ComputeShader asset (public ComputeShader compute), FindKernel: `int kernel = compute.FindKernel("MyKernel");`

**Compute Dispatch**: C# script: `compute.Dispatch(kernelIndex, groupsX, groupsY, groupsZ);` Calculate groups: `int groups = Mathf.CeilToInt(dataCount / (float)numthreads);` (ensures coverage, e.g., 1000 data, numthreads=64, groups=16). SetBuffer: `compute.SetBuffer(kernel, "data", buffer);` (assign ComputeBuffer to shader), SetTexture: `compute.SetTexture(kernel, "output", renderTexture);` (assign RenderTexture).

**Structured Buffer Setup**: C# create: `ComputeBuffer buffer = new ComputeBuffer(count, sizeof(float) * 4);` (4 floats per element = float4 struct), populate: `float4[] data = new float4[count]; buffer.SetData(data);` Shader declare: `RWStructuredBuffer<float4> data;` Access: `data[id.x] = float4(1,0,0,1);` Dispose: `void OnDestroy() { buffer?.Dispose(); }`

**Dissolve Shader Example**:
```hlsl
float _DissolveAmount;
float _EdgeWidth;
float4 _EdgeColor;
sampler2D _DissolveMap;

float4 frag(v2f i) : SV_Target
{
    float noise = tex2D(_DissolveMap, i.uv).r;
    clip(noise - _DissolveAmount); // Discard if below threshold
    
    float edge = smoothstep(_DissolveAmount, _DissolveAmount + _EdgeWidth, noise);
    float4 albedo = tex2D(_MainTex, i.uv);
    return lerp(_EdgeColor, albedo, edge); // Bright edge near dissolve boundary
}
```

**Distortion Shader Example**:
```hlsl
sampler2D _CameraOpaqueTexture;
sampler2D _DistortionMap;
float _Intensity;

float4 frag(v2f i) : SV_Target
{
    float2 distortion = tex2D(_DistortionMap, i.uv).rg * 2 - 1; // -1 to 1 range
    float2 screenUV = i.screenPos.xy / i.screenPos.w;
    float2 distortedUV = screenUV + distortion * _Intensity;
    return tex2D(_CameraOpaqueTexture, distortedUV);
}
```

**Profiling Compute**: Unity Profiler > GPU (shows Compute Shader passes, timing per dispatch). Frame Debugger (limited for compute, shows dispatch calls but not internal steps). External: RenderDoc (captures compute dispatches, shows buffer contents, shader code, per-thread debugging), Nsight (NVIDIA, advanced compute profiling, warp occupancy, memory throughput).

**Mesh Builder**: C# script: `Mesh mesh = new Mesh(); mesh.vertices = vertArray; mesh.triangles = triArray; mesh.uv = uvArray; mesh.RecalculateNormals(); meshFilter.mesh = mesh;` Performance: `mesh.MarkDynamic()` (if updating frequently), `mesh.SetVertices(vertList)` (faster than array assignment). Bounds: `mesh.RecalculateBounds()` (auto-calculates, or set manually: `mesh.bounds = new Bounds(center, size)`).

## Related Topics

- [19.2 VFX Graph](19-02-VFX-Graph.md) - GPU particle systems
- [12.2 Writing Custom Shaders](12-02-Shader-Types.md) - Shader programming
- [27.03 GPU-Driven Rendering](27-03-GPU-Driven-Rendering.md) - Advanced GPU techniques
- [5.2 GPU Optimization](05-02-GPU-Side-Optimization.md) - GPU performance

---

[← Previous: 19.2 VFX Graph](19-02-VFX-Graph.md) | [Next: Chapter 20 →](../chapters/20-Specialized-Rendering.md)
