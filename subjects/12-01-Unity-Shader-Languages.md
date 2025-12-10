# 12.1 Unity Shader Languages

[← Back to Chapter 12](../chapters/12-Shader-Programming.md) | [Main Index](../README.md)

Unity supports multiple shader authoring approaches: HLSL (code-based shaders), ShaderLab (Unity wrapper syntax), Shader Graph (visual node-based), and Compute Shaders for GPU computation.

---

## Overview

Shader languages define how GPUs process vertices and pixels. Unity uses HLSL (High-Level Shading Language, Microsoft's shader language) for shader code, wrapped in ShaderLab syntax (Unity-specific metadata format). Developers write vertex/fragment shader functions in HLSL, declare properties/passes in ShaderLab. Alternative: Shader Graph (visual scripting, node-based), no code required. Compute Shaders use HLSL for general-purpose GPU programming (particle systems, procedural generation, image processing).

ShaderLab structure: Properties block (exposed material parameters), SubShader block (shader implementations, one per graphics API/platform), Pass block (rendering pass: vertex+fragment shaders), and Tags (render queue, blend mode, culling). HLSL code inside CGPROGRAM/ENDCG (legacy) or HLSLPROGRAM/ENDHLSL (modern SRP). Unity compiles ShaderLab+HLSL to platform-specific bytecode (DirectX DXIL, Vulkan SPIR-V, Metal AIR, OpenGL GLSL).

Shader Graph provides visual alternative: drag-drop nodes (textures, math, lighting), connect nodes (data flow), compile to HLSL automatically. Benefits: artist-friendly (no coding), fast iteration (real-time preview), error prevention (type-safe connections). Limitations: less control than code (some features unavailable), performance overhead (generated code sometimes suboptimal), SRP-only (URP/HDRP, not Built-in Render Pipeline).

## Key Concepts

- **HLSL (High-Level Shading Language)**: Microsoft's shader programming language. C-like syntax (functions, structs, intrinsics). Unity's HLSL variant includes Unity-specific functions (UnityObjectToClipPos, UnityWorldSpaceLightDir). Cross-compiles to DirectX, Vulkan, Metal, OpenGL.
- **ShaderLab**: Unity's shader wrapper syntax. Defines properties (material UI), SubShaders (platform variants), Passes (render stages), and Tags (rendering settings). Contains HLSL code blocks (CGPROGRAM or HLSLPROGRAM). .shader file format.
- **Shader Graph**: Visual shader editor (node-based). Create shaders via graphs (nodes = operations, edges = data flow). Compiles to HLSL automatically. Supports URP/HDRP only (not Built-in Pipeline). .shadergraph file format.
- **Compute Shaders**: HLSL-based GPU compute (not rendering). General-purpose GPU programming (GPGPU). Use cases: particle systems, physics simulation, procedural generation, image processing. .compute file format.
- **Cg (C for Graphics)**: Deprecated NVIDIA shader language. Unity supported Cg historically (CGPROGRAM blocks), now uses HLSL. Legacy shaders still use Cg syntax (compatible with HLSL). Modern shaders use HLSLPROGRAM.

## Best Practices

**HLSL Shader Development:**
- Use HLSLPROGRAM over CGPROGRAM: Modern Unity shaders use HLSLPROGRAM/ENDHLSL (SRP compatibility, better cross-platform support). CGPROGRAM deprecated (legacy Built-in Pipeline only). Migrate existing shaders to HLSLPROGRAM.
- Include Unity helpers: #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl" (URP) or "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl" (Core). Provides Unity functions (transformations, lighting, etc.).
- Shader variants: Use #pragma multi_compile or #pragma shader_feature for conditional code (keywords enable/disable features). Example: #pragma multi_compile _ _NORMALMAP (compiles with/without normal mapping). Limit variants (<8 keywords per shader to avoid explosion).
- Precision qualifiers: Use half for low-precision (colors, normals), float for high-precision (world positions, complex math). Mobile GPUs benefit from half (2x throughput). PC/console: float and half equivalent (no benefit).

**ShaderLab Structure:**
- Properties for material UI: Declare properties in Properties block (_MainTex("Albedo", 2D), _Color("Tint", Color), _Metallic("Metallic", Range(0,1))). Exposes parameters in Material Inspector. Use descriptive names and labels.
- SubShader organization: Multiple SubShaders for platform fallbacks (SubShader LOD 300 = high quality, LOD 200 = medium, Fallback = low). Unity selects first compatible SubShader based on platform/hardware. Typical: one SubShader per pipeline (URP, HDRP, Built-in).
- Pass Tags: UsePass "TagName" for pass types ("LightMode" = "UniversalForward" for URP, "LightMode" = "ForwardBase" for Built-in). Blend/ZWrite/Cull inside Pass (per-pass render state).
- Fallback shaders: Fallback "Diffuse" at end of shader. Used when no SubShader compatible (old hardware, unsupported features). Prevents pink shader (missing material indicator).

**Shader Graph Workflow:**
- Start with templates: Create via Assets > Create > Shader Graph > URP > Lit or Unlit. Templates provide base graph structure (vertex/fragment outputs). Customize from there.
- Use SubGraphs: Extract reusable logic into SubGraphs (node groups as separate asset). Example: DetailMappingSubGraph (detail texture + blending logic). Reference SubGraph in multiple Shader Graphs (DRY principle).
- Custom Function Nodes: Embed HLSL code in graph via Custom Function Node. Write HLSL snippets for features unavailable in nodes (noise functions, custom lighting). Bridges gap between visual and code.
- Optimize generated code: Shader Graph generates HLSL automatically (sometimes inefficient). Profile generated shader (RenderDoc/PIX), compare to hand-written. Optimize hot paths manually if needed.

**Compute Shader Usage:**
- Thread group sizes: Declare [numthreads(8,8,1)] (thread group dimensions). GPU dispatches thread groups (ComputeShader.Dispatch(kernelIndex, groupsX, groupsY, groupsZ)). Total threads = groups * numthreads (Dispatch(64,64,1) with numthreads(8,8,1) = 512x512 threads).
- Read/write buffers: Use RWStructuredBuffer<T> for read-write data (particle positions), RWTexture2D<T> for images (procedural textures). Bind via ComputeShader.SetBuffer/SetTexture.
- Avoid thread divergence: GPU executes threads in warps/wavefronts (32-64 threads). Divergent branches (if statements) serialize execution (warp waits for all branches). Minimize branching in compute kernels (use branchless math).
- Synchronization: groupshared memory for thread group communication (shared LDS). Use GroupMemoryBarrierWithGroupSync() to synchronize threads within group (wait for all threads to reach barrier).

**Platform-Specific:**
- **PC**: Full HLSL support (Shader Model 5.0-6.0). DirectX 11/12 (native HLSL), Vulkan (SPIR-V cross-compile), OpenGL (GLSL cross-compile). All shader features supported (geometry shaders, tessellation, compute).
- **Consoles**: Shader Model 5.0-6.0. Xbox uses DirectX (native HLSL), PlayStation uses custom compiler (PSSL, Unity cross-compiles from HLSL). Compute shaders fully supported (optimized for GCN/RDNA).
- **Switch**: Shader Model 4.0 (Maxwell GPU). Simplified HLSL (no geometry shaders, no tessellation). Compute shaders supported but slower (Maxwell not optimized for compute). Use NVNGFX shader compiler.
- **Mobile**: Shader Model 3.0-4.0. OpenGL ES (GLSL cross-compile), Metal (AIR cross-compile), Vulkan (SPIR-V). Limited compute shader support (newer devices only, Android 7+, iOS 13+). Precision qualifiers critical (half vs float).

## Common Pitfalls

**Using Cg Instead of HLSL**: Developer writes CGPROGRAM blocks (legacy syntax). Shaders incompatible with SRP (URP/HDRP), only work in Built-in Pipeline. Unity console warns "CGPROGRAM is deprecated". Symptom: Pink shaders in URP/HDRP projects, warnings in console. Solution: Migrate to HLSLPROGRAM/ENDHLSL. Replace Unity CG includes with SRP includes (UnityCG.cginc → Core.hlsl).

**Shader Variant Explosion**: Developer adds 10 keywords (#pragma multi_compile). Shader compiles 2^10 = 1,024 variants (30-minute build times, runtime hitches). Symptom: Long builds, "Compiling shader variants" messages, runtime hitches when material appears. Solution: Reduce keywords to <8. Use #pragma shader_feature for optional features (strips unused variants). Audit variants via Shader Variant Collection.

**Not Using Precision Qualifiers**: Developer uses float for everything (colors, normals, UVs). Mobile shaders run at half speed (ALUs process half precision 2x faster). Wasted performance. Symptom: Low frame rate on mobile, GPU bound. Solution: Use half for colors, normals, UVs (low precision acceptable). Use float for positions, complex math (high precision required). Declare: half3 normal, float3 worldPos.

**Shader Graph Performance**: Developer builds complex shader in Shader Graph (100+ nodes). Generated HLSL suboptimal (redundant calculations, excessive texture samples). Shader costs 5ms per frame (hand-written equivalent = 2ms). Symptom: GPU bound, Profiler shows expensive shader. Solution: Profile generated HLSL (RenderDoc), identify inefficiencies. Optimize by simplifying graph or rewriting in HLSL manually.

## Tools & Workflow

**Visual Studio/Rider**: HLSL syntax highlighting, intellisense. Write shader code in IDE (better than Unity text editor). HLSL Tools extension (Visual Studio) adds HLSL-specific features (go-to-definition, refactoring).

**Shader Graph**: Window > Shader Graph or Assets > Create > Shader Graph. Visual shader editor (node-based). Real-time preview (see shader result as you build). Supports URP/HDRP (not Built-in Pipeline).

**RenderDoc/PIX**: Capture frame, inspect compiled shaders. View HLSL source (generated by Unity), assembly (DXIL, SPIR-V), instruction counts. Debug pixel shaders (step through HLSL code, inspect variables).

**Unity Shader Compiler**: Invoked automatically during builds. Compiles .shader/.compute files to platform-specific bytecode. Editor > Preferences > External Tools > Shader Compiler Timeout (increase for complex shaders).

**Shader Variant Collection**: Project > Create > Shader Variant Collection. Preload shader variants (avoids runtime compilation hitches). Add variants manually or capture from build (Graphics > Shader Compilation > "Save to asset").

**SRP Debug Tools**: URP/HDRP include debug views (Rendering Debugger window). Visualize shader outputs (albedo, normals, metallic), identify shader issues (incorrect normals, missing textures).

**Amplify Shader Editor**: Third-party visual shader editor (Asset Store). More features than Shader Graph (supports Built-in Pipeline, more node types). Alternative to Shader Graph for complex shaders.

## Related Topics

- [12.2 Shader Types](12-02-Shader-Types.md) - Surface, vertex/fragment, compute shaders
- [12.3 Lighting Models](12-03-Lighting-Models.md) - PBR and custom lighting
- [12.5 Shader Compilation and Caching](12-05-Shader-Compilation-And-Caching.md) - Compilation process
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Shader optimization

---

[← Previous: Chapter 11](../chapters/11-Materials-And-Textures.md) | [Next: 12.2 Shader Types →](12-02-Shader-Types.md)
