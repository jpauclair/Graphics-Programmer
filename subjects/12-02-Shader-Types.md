# 12.2 Shader Types

[← Back to Chapter 12](../chapters/12-Shader-Programming.md) | [Main Index](../README.md)

Unity supports multiple shader types: Surface Shaders (high-level PBR abstraction), Vertex/Fragment Shaders (low-level control), Unlit Shaders (no lighting), and Compute Shaders (GPU compute tasks).

---

## Overview

Shader types determine abstraction level and use case. Surface Shaders (Built-in Pipeline only) abstract lighting calculations: write surface function (outputs albedo, metallic, smoothness), Unity generates lighting code automatically (supports multiple lights, shadows, GI). Vertex/Fragment Shaders provide full control: write vertex shader (transform positions, pass data to fragment), write fragment shader (calculate pixel color), manually implement lighting. Unlit Shaders skip lighting entirely: output color directly (UI, effects, stylized rendering). Compute Shaders aren't rendering shaders: general-purpose GPU programming (particles, physics, procedural generation).

Surface Shader benefits: less code (Unity generates lighting boilerplate), automatic light/shadow support (no manual light loops), and GI integration (lightmaps, probes work automatically). Limitations: Built-in Pipeline only (URP/HDRP don't support Surface Shaders), less control (can't customize lighting passes), and potential overhead (generated code sometimes suboptimal). Migration: URP/HDRP use Shader Graph or hand-written vertex/fragment shaders instead.

Vertex/Fragment Shader benefits: full control (optimize every instruction), cross-pipeline support (Built-in, URP, HDRP), and explicit behavior (no hidden code generation). Disadvantages: more code (manual lighting implementation), complex lighting (multi-light support requires loops, shadow sampling), and maintenance burden (more code to debug). Use for: custom effects (stylized rendering, special VFX), performance-critical shaders (mobile, optimized rendering), and SRP shaders (URP/HDRP Lit shaders).

## Key Concepts

- **Surface Shader**: High-level Built-in Pipeline shader. Write SurfaceOutput struct (albedo, metallic, smoothness, normal), Unity generates vertex/fragment code automatically. Supports lighting, shadows, GI. Syntax: #pragma surface surf Standard (surf = surface function, Standard = lighting model).
- **Vertex/Fragment Shader**: Low-level shader. Write vert function (vertex transformation), frag function (pixel color calculation). Full control over rendering pipeline. Syntax: #pragma vertex vert, #pragma fragment frag. Use for custom effects or SRP shaders.
- **Unlit Shader**: No lighting calculations. Outputs color/texture directly. Use for UI, particle effects, stylized rendering (toon shading, flat colors), or performance-critical mobile rendering. Minimal GPU cost (5-20 ALU instructions).
- **Compute Shader**: GPGPU shader for non-rendering tasks. Processes data in parallel (thread groups, dispatched via C#). Read/write buffers (RWStructuredBuffer), textures (RWTexture2D). Use cases: particle systems, physics simulation, procedural generation, image processing.
- **Geometry/Tessellation Shaders**: Advanced shader stages. Geometry Shader: generates/destroys primitives (per-triangle processing). Tessellation Shader: subdivides meshes (dynamic LOD, displacement mapping). Expensive, platform-limited (PC/consoles only, not mobile/Switch).

## Best Practices

**Surface Shader Usage (Built-in Pipeline):**
- Surface function implementation: Define SurfaceOutput (or SurfaceOutputStandard for PBR). Set albedo (o.Albedo = tex2D(_MainTex, IN.uv_MainTex).rgb), metallic (o.Metallic = _Metallic), smoothness (o.Smoothness = _Glossiness), normal (o.Normal = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap))).
- Lighting model selection: Standard (PBR, metallic workflow), StandardSpecular (PBR, specular workflow), Lambert (simple diffuse), BlinnPhong (diffuse + specular), Custom (implement custom lighting function).
- Pragma directives: #pragma surface surf Standard fullforwardshadows (fullforwardshadows = receive shadows from all lights). #pragma target 3.0 (Shader Model 3.0, mobile compatibility).
- Input struct: Define Input struct with required UVs (float2 uv_MainTex), world position (float3 worldPos), vertex color (float4 color). Unity passes data from vertex to surface function.

**Vertex/Fragment Shader Implementation:**
- Vertex shader: Transform object-space positions to clip space (output.pos = UnityObjectToClipPos(input.vertex)). Pass data to fragment shader (UVs, world position, normals). Calculate per-vertex values (lighting, fog).
- Fragment shader: Return pixel color (return fixed4(color.rgb, 1.0)). Sample textures (tex2D(_MainTex, input.uv)), calculate lighting (dot products, BRDF), apply fog (UNITY_APPLY_FOG).
- Appdata/v2f structs: Define input (Appdata: vertex position, UVs, normals from mesh) and output (v2f: interpolated data from vertex to fragment). Use semantics (POSITION, TEXCOORD0, COLOR).
- Lighting implementation: Manual light loops (for(int i=0; i<4; i++)), shadow sampling (UNITY_LIGHT_ATTENUATION), GI (SampleSH, sample lightmaps). Requires significant code (50-100 lines for PBR).

**Unlit Shader Patterns:**
- Simple texture output: Sample texture, output color (return tex2D(_MainTex, input.uv) * _Color). No lighting calculations. Use for UI images, sprites, particle effects.
- Vertex color multiplication: Multiply texture by vertex color (return tex2D(_MainTex, input.uv) * input.color). Allows per-vertex tinting (particle systems, procedural coloring).
- Alpha blending: Set Blend mode (Blend SrcAlpha OneMinusSrcAlpha for alpha blending, Blend One One for additive). Use for transparent effects (glass, holograms, particle effects).
- Mobile optimization: Unlit shaders fastest on mobile (no lighting = minimal ALU). Use for background objects, distant LODs, stylized games (flat-shaded aesthetics).

**Compute Shader Development:**
- Kernel declaration: #pragma kernel CSMain (CSMain = kernel function name). Multiple kernels per file (different entry points).
- Thread groups: [numthreads(8,8,1)] void CSMain(uint3 id : SV_DispatchThreadID). Dispatch via ComputeShader.Dispatch(kernelIndex, groupsX, groupsY, groupsZ). Total threads = groups * numthreads.
- Buffer access: RWStructuredBuffer<float4> Result (read-write). StructuredBuffer<float4> Input (read-only). Set via ComputeShader.SetBuffer(kernelIndex, "BufferName", buffer).
- Common use cases: Particle systems (update positions/velocities on GPU), procedural generation (noise, fractals), image processing (blur, edge detection), physics simulation (cloth, fluids).

**Shader Type Selection:**
- Surface Shader: Built-in Pipeline + PBR rendering + standard lighting. Quick prototyping, standard materials (characters, environments with realistic lighting).
- Vertex/Fragment: Custom lighting, stylized rendering, SRP projects (URP/HDRP). Performance-critical shaders (mobile), special effects (distortion, dissolve).
- Unlit: No lighting needed (UI, flat-shaded games, particle effects, skyboxes). Mobile performance priority (simplest shaders).
- Compute: Non-rendering GPU tasks (particle systems, procedural generation, physics, image processing). Data-parallel problems (millions of independent calculations).

**Platform-Specific:**
- **PC**: All shader types supported. Geometry/Tessellation shaders available (DirectX 11+, rarely used for performance reasons). Compute shaders fast (modern GPUs optimized for compute).
- **Consoles**: Full support except geometry shaders (rarely used, performance cost). Compute shaders optimized (async compute on GCN/RDNA). Surface Shaders only in Built-in Pipeline.
- **Switch**: Vertex/Fragment and Unlit shaders. No geometry/tessellation (Maxwell GPU limitations). Compute shaders supported but slower (not optimized for compute). Avoid Surface Shaders (Built-in Pipeline overhead).
- **Mobile**: Vertex/Fragment and Unlit shaders. No geometry/tessellation (unsupported). Compute shaders on newer devices only (Android 7+, iOS 13+). Prefer Unlit for performance (lighting expensive on mobile GPUs).

## Common Pitfalls

**Using Surface Shaders in URP/HDRP**: Developer migrates Built-in Pipeline project to URP, keeps Surface Shaders. Shaders break (pink materials), console errors "Surface Shaders not supported in SRP". Symptom: Pink materials, compilation errors. Solution: Rewrite as vertex/fragment shaders or use Shader Graph. URP/HDRP don't support Surface Shaders (architectural limitation).

**Complex Vertex/Fragment Lighting**: Developer writes vertex/fragment shader with full PBR lighting (multi-light support, shadows, GI). 200+ lines of code, bugs in lighting calculations (incorrect BRDF, shadow artifacts). Symptom: Incorrect lighting, shadows not working, long development time. Solution: Use Shader Graph (generates lighting code automatically) or reference URP Lit shader source (Packages/com.unity.render-pipelines.universal/Shaders/LitInput.hlsl) as template.

**Unlit Shader for Lit Objects**: Developer uses Unlit shader for character (thinks it's faster). Character doesn't respond to lighting (always bright, no shadows, looks flat). Symptom: Character ignores scene lighting, no shadows cast/received. Solution: Use lit shader (Vertex/Fragment with lighting, or Shader Graph Lit). Unlit only for objects that shouldn't respond to lights (UI, emissive objects, stylized flat rendering).

**Compute Shader Thread Mismanagement**: Developer dispatches Dispatch(1024,1,1) with numthreads(1024,1,1). Launches 1024*1024 = 1,048,576 threads (intended 1,024). GPU hangs or crashes. Symptom: GPU timeout (TDR), Unity freezes, device resets. Solution: Understand dispatch math. Total threads = Dispatch(groupsX, groupsY, groupsZ) * numthreads(X,Y,Z). For 1024 threads: Dispatch(128,1,1) with numthreads(8,1,1) = 128*8 = 1,024.

## Tools & Workflow

**Shader Creation Wizards**: Assets > Create > Shader > Surface Shader/Unlit Shader/Image Effect Shader/Compute Shader. Generates template code (boilerplate structure). Customize template for specific needs.

**Shader Graph**: Visual shader creation (URP/HDRP). Supports Unlit, Lit (vertex/fragment equivalent with PBR lighting), Custom Lit (custom lighting models). No Surface Shader equivalent (SRP design choice).

**URP/HDRP Lit Shader Source**: Packages/com.unity.render-pipelines.universal/Shaders (URP), Packages/com.unity.render-pipelines.high-definition/Runtime/Material/Lit (HDRP). Reference implementation for vertex/fragment PBR shaders. Copy as starting point for custom shaders.

**Compute Shader Debugger**: RenderDoc/PIX capture compute dispatches. View thread group execution, buffer contents (input/output data), instruction counts. Debug compute kernel issues (race conditions, incorrect indexing).

**Unity Profiler**: GPU Profiler shows shader costs per type (vertex shaders, fragment shaders, compute kernels). Identify expensive shaders (>1ms per frame). Deep Profile > Scripts shows compute shader dispatches (ComputeShader.Dispatch calls).

**Shader Include Browser**: Visual Studio/Rider with HLSL Tools. Go-to-definition for Unity includes (UnityCG.cginc, Core.hlsl). Browse Unity shader library source (transformations, lighting helpers, common functions).

**Frame Debugger**: Window > Analysis > Frame Debugger. Shows shader used per draw call (Surface Shader, Vertex/Fragment, Unlit). Verify correct shader type selected, inspect shader properties.

## Related Topics

- [12.1 Unity Shader Languages](12-01-Unity-Shader-Languages.md) - HLSL and ShaderLab syntax
- [12.3 Lighting Models](12-03-Lighting-Models.md) - Implementing lighting in shaders
- [12.4 Advanced Shader Techniques](12-04-Advanced-Shader-Techniques.md) - Complex shader effects
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Shader optimization

---

[← Previous: 12.1 Unity Shader Languages](12-01-Unity-Shader-Languages.md) | [Next: 12.3 Lighting Models →](12-03-Lighting-Models.md)
