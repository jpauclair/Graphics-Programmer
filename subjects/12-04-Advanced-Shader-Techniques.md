# 12.4 Advanced Shader Techniques

[← Back to Chapter 12](../chapters/12-Shader-Programming.md) | [Main Index](../README.md)

Advanced shader techniques enable complex visual effects through parallax mapping, screen-space effects, vertex manipulation, instancing, and GPU-driven rendering optimizations.

---

## Overview

Advanced techniques solve specialized rendering challenges: parallax occlusion mapping (POM) simulates depth via height map ray-marching (fake 3D detail without geometry), screen-space effects process rendered pixels (SSAO, SSR, depth-based effects), vertex displacement creates dynamic geometry (waves, wind, terrain deformation), GPU instancing draws thousands of objects efficiently (one draw call for identical meshes with per-instance properties), and indirect rendering eliminates CPU overhead (GPU generates draw calls autonomously).

Parallax mapping family: parallax mapping (simple UV offset based on height), parallax occlusion mapping (ray-march height map, accurate depth), relief mapping (binary search for intersection). Trade-off: visual quality (POM = most realistic) vs performance (POM = 20-50 texture samples, parallax = 2-5 samples). Use for: close-up surfaces (brick walls, stone floors, cobblestone), hero assets (player-visible areas), not distant objects (wasted GPU on invisible detail).

GPU instancing renders identical meshes with per-instance variation (position, color, scale) in single draw call. Requires: identical mesh+material, instance data in buffer (StructuredBuffer or MaterialPropertyBlock), shader support (#pragma instancing_options). Benefits: 1000 trees = 1 draw call instead of 1000 (massive CPU savings), GPU processes instances in parallel (efficient rendering). Indirect rendering extends instancing: GPU generates draw calls (via compute shader), CPU doesn't touch rendering (zero CPU overhead for dynamic objects like particles, foliage).

## Key Concepts

- **Parallax Occlusion Mapping (POM)**: Ray-marches height map to find intersection. Samples height map 10-50 times per pixel (finds depth where ray intersects surface). Offsets UVs based on intersection (creates 3D appearance). Expensive but realistic depth simulation.
- **Screen-Space Effects**: Post-processing operating on rendered image. Examples: SSAO (ambient occlusion from depth), SSR (reflections from color buffer), depth-based fog. Requires render-to-texture (render scene, then process image with screen-space shader).
- **Vertex Displacement**: Modify vertex positions in vertex shader. Read displacement map (height/noise texture), offset vertices along normal. Use for: terrain deformation, water waves, cloth simulation, procedural animation (wind, grass).
- **GPU Instancing**: Render multiple instances in single draw call. Unity built-in instancing (MaterialPropertyBlock, Graphics.DrawMeshInstanced), or manual via StructuredBuffer. Shader accesses instance ID (unity_InstanceID), reads per-instance data from buffer.
- **Indirect Rendering**: GPU generates draw calls. CPU fills argument buffer (DrawMeshInstancedIndirect: instance count, start index), GPU dispatches draws. Combine with compute shader (GPU culls instances, updates argument buffer). Zero CPU overhead (fully GPU-driven).

## Best Practices

**Parallax Occlusion Mapping:**
- Implementation: Ray-march from view direction through height map. Start at UV, step along view vector (10-50 steps), sample height map each step, find intersection (height < surface). Offset final UV by intersection point.
- Step count tuning: More steps = accurate depth but expensive (50 steps = 50 height map samples). Fewer steps = faster but quality loss (10 steps = visible banding). Typical: 20-30 steps for quality, 10-15 for performance.
- Height map authoring: Grayscale texture (white = high, black = low). 8-bit precision sufficient (256 height levels). Store in metallic/smoothness alpha (save texture slot), or dedicated height map (better quality).
- Silhouette limitation: POM doesn't affect silhouette (mesh outline unchanged). Edges look flat (2D mesh with 3D interior). Use for surfaces viewed head-on (floors, walls), not edges (avoid viewing at glancing angles).

**Screen-Space Ambient Occlusion (SSAO):**
- Depth-based sampling: Sample depth buffer around pixel (kernel of 16-64 samples). Count samples above surface (occluded) vs below (not occluded). Ratio determines occlusion (0 = fully occluded dark, 1 = no occlusion bright).
- Noise texture: Random rotation per pixel (avoids banding). Tile small noise texture (4x4 or 8x8) across screen. Rotate sample kernel per pixel (varied patterns).
- Blur pass: SSAO generates noisy result (high-frequency noise from random sampling). Apply bilateral blur (blur while preserving edges via depth). Smooth AO without blurring across geometry boundaries.
- Performance: SSAO expensive (64 samples * depth fetches = 100-200 texture reads per pixel). Use half-resolution (render SSAO at 50% res, upscale). Mobile: skip SSAO entirely (use baked AO only).

**Vertex Displacement:**
- Read displacement map: Sample texture in vertex shader (tex2Dlod for explicit mip level, avoids mip derivatives unavailable in vertex shader). Offset vertex position along normal (vertex.xyz += normal * height).
- Coordinate spaces: Displace in object space (simple, works for rigid objects) or world space (consistent displacement for deforming meshes). Transform normal to appropriate space (mul((float3x3)unity_ObjectToWorld, normal)).
- UV considerations: Ensure UVs tile seamlessly (displacement map wraps without seams). Use world-space UVs for large surfaces (terrain, avoids stretching). Object-space UVs for props (baked displacement per asset).
- Mesh density: Vertex displacement requires sufficient vertices (low-poly mesh = blocky displacement). Tessellation shader can subdivide mesh dynamically (adds vertices before displacement). PC/console only (tessellation unsupported on mobile/Switch).

**GPU Instancing Setup:**
- Enable in shader: #pragma multi_compile_instancing in shader. Adds unity_InstanceID semantic (per-instance index). Access via UNITY_SETUP_INSTANCE_ID macro (in vertex shader).
- Per-instance properties: Use MaterialPropertyBlock (renderer.SetPropertyBlock(block)), or StructuredBuffer (manual buffer binding). Properties: _Color, _Position, _Scale (vary per instance).
- Graphics.DrawMeshInstanced: C# API for instanced rendering. DrawMeshInstanced(mesh, 0, material, matrices, count). matrices = Matrix4x4[] (transformation per instance). Up to 1023 instances per call.
- Batching compatibility: GPU instancing incompatible with static/dynamic batching (disable batching when using instancing). SRP Batcher compatible with instancing (combine both for maximum efficiency).

**Indirect Rendering:**
- Argument buffer: ComputeBuffer with indirect args (uint[5]: instance count, instance start, base vertex, base instance, padding). GPU reads args, issues draw calls automatically.
- Graphics.DrawMeshInstancedIndirect: C# API. DrawMeshInstancedIndirect(mesh, 0, material, bounds, argsBuffer). Draws instance count from buffer (CPU doesn't know count).
- Compute shader integration: Compute shader updates argsBuffer (culls instances, counts visible). Example: frustum culling (GPU tests each instance, writes visible count to argsBuffer[0]). Zero CPU culling overhead.
- Procedural instancing: Generate instance data in compute shader (positions, rotations, scales). Store in StructuredBuffer, access in vertex shader via unity_InstanceID. Fully GPU-driven (particles, foliage, debris).

**Platform-Specific:**
- **PC**: All advanced techniques supported. POM with 30-50 steps, SSAO at full-res, vertex displacement + tessellation, GPU instancing (10,000+ instances), indirect rendering. High-end GPUs handle complexity well.
- **Consoles**: Full support except tessellation (rarely used for performance). POM with 20-30 steps, SSAO at half-res (upscaled), vertex displacement (no tessellation), GPU instancing common (foliage, crowds), indirect rendering for particles/VFX.
- **Switch**: Limited techniques. Simple parallax (5-10 steps, not POM), no SSAO (baked AO only), vertex displacement (limited vertex budget), GPU instancing (500-1000 instances), no indirect rendering (limited compute performance).
- **Mobile**: Basic techniques only. No parallax/POM (ALU cost too high), no SSAO (render-to-texture expensive), minimal vertex displacement (simple waves acceptable), GPU instancing limited (100-500 instances, driver support varies), no indirect rendering.

## Common Pitfalls

**POM on Distant Objects**: Developer applies POM to all brick walls (including distant). Distant walls (100m away, occupying 10 pixels) ray-march 30 times per pixel (wasted GPU, detail invisible). Symptom: GPU bound, Profiler shows expensive fragment shaders. Solution: Use material LOD (POM for close objects <20m, simple parallax for mid-range 20-50m, normal map only for distant >50m). Or disable POM via shader LOD (Shader.maximumLOD).

**Incorrect Instancing Setup**: Developer enables instancing (#pragma multi_compile_instancing) but doesn't call UNITY_SETUP_INSTANCE_ID in vertex shader. Instance ID not propagated, per-instance properties don't work (all instances same color). Symptom: Instancing renders but all instances identical. Solution: Call UNITY_SETUP_INSTANCE_ID(input) at start of vertex shader, UNITY_TRANSFER_INSTANCE_ID(input, output) to pass to fragment.

**SSAO Without Blur**: Developer implements SSAO (depth sampling + occlusion calculation) but skips blur pass. SSAO output extremely noisy (high-frequency grain, distracting artifacts). Symptom: Grainy ambient occlusion, flickering noise. Solution: Add bilateral blur pass (blur AO while preserving edges via depth comparison). Typical: 4x4 or 8x8 kernel blur.

**Indirect Rendering Without Bounds**: Developer uses DrawMeshInstancedIndirect with tight bounds (assumes instances in small area). Instances outside bounds culled by GPU frustum culling (invisible despite being in view). Symptom: Instances pop out of view, disappear when camera moves. Solution: Provide loose bounds (encapsulate all possible instance positions) or disable culling (infinite bounds via Bounds.Encapsulate(Vector3.positiveInfinity)).

## Tools & Workflow

**Parallax Occlusion Mapping Shader**: Amplify Shader Editor or Shader Graph custom function node. Implement POM ray-marching in HLSL (embed in Custom Function Node). Or use Asset Store POM shaders (pre-built implementations).

**Post-Processing Stack**: Unity Post-Processing (Built-in), URP Post-Processing (Volume system), HDRP Volume system. Includes SSAO, SSR, depth-based effects. Configure via Volume profiles (quality settings, effect parameters).

**GPU Instancing Profiler**: Profiler > Rendering > Batches shows instanced batches. "Instanced" indicator confirms instancing active. Frame Debugger shows instance count per draw call (verify instances batched correctly).

**Compute Shader Debugger**: RenderDoc/PIX captures compute dispatches. View argument buffer contents (instance counts), instance buffers (positions, colors). Debug indirect rendering (verify GPU writes correct counts, buffers populated correctly).

**Mesh Baker**: Asset Store tool for baking vertex displacement. Apply displacement map to mesh offline (generates high-poly displaced mesh). Use when vertex displacement too expensive at runtime (bake once, use low-cost static mesh).

**SSAO Debugger**: Rendering Debugger (URP/HDRP) > Lighting > Ambient Occlusion > Show AO only. Visualizes SSAO output (black = occluded, white = lit). Verify AO quality, identify artifacts (halos, banding).

**Shader Graph Custom Function**: Implement advanced techniques as Custom Function Nodes (HLSL code embedded in graph). Example: POM node (inputs: height map, view direction, steps; output: offset UV). Reusable across Shader Graphs.

## Related Topics

- [12.3 Lighting Models](12-03-Lighting-Models.md) - PBR and custom lighting
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Shader performance
- [13.3 Post-Processing](../chapters/13-Rendering-Techniques.md) - Screen-space effects
- [11.5 Advanced Texture Techniques](11-05-Advanced-Texture-Techniques.md) - Procedural textures

---

[← Previous: 12.3 Lighting Models](12-03-Lighting-Models.md) | [Next: 12.5 Shader Compilation and Caching →](12-05-Shader-Compilation-And-Caching.md)
