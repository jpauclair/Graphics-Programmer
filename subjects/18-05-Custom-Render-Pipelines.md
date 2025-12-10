# 18.5 Custom Render Pipelines

[← Back to Chapter 18](../chapters/18-Scriptable-Render-Pipeline.md) | [Main Index](../README.md)

Custom SRP enables tailored rendering: stylized graphics (toon shading, non-photorealistic), platform-specific optimization (mobile/VR-optimized pipelines), or removing unnecessary features (minimal pipeline for performance).

---

## Overview

Why Custom SRP: URP/HDRP cover most use cases (URP = performance-first, HDRP = high-end graphics), but limitations exist (fixed feature set, can't remove unwanted overhead, stylized rendering requires workarounds). Custom SRP: full control (implement only needed features, custom lighting models, novel rendering techniques), optimization (remove all unnecessary code, target specific hardware, achieve maximum performance), unique art styles (toon shading, pixel art upscaling, retro aesthetics, photographic effects).

Development Cost: High (requires graphics programming expertise, understanding of render pipelines, GPU architectures), time-intensive (weeks to months for production-ready pipeline, URP/HDRP provide years of engineering out-of-box). Maintenance (ongoing bug fixes, Unity version updates, platform-specific issues). Recommendation: Use URP/HDRP unless (team has graphics expert, project needs truly unavailable in URP/HDRP, or extreme performance requirements justify custom implementation).

Approaches: Minimal pipeline (basic forward rendering, single light, no shadows = ultra-lightweight for mobile/VR), feature-specific (add one advanced feature to minimal base: custom GI, volumetric fog, stylized lighting), or full-featured (URP-like complexity, custom implementation for learning/control). Start minimal (render opaque objects only), add features incrementally (shadows, then lights, then post-processing), test frequently (validate rendering correctness after each addition).

## Key Concepts

- **RenderPipeline Subclass**: Core custom pipeline class (inherits from `UnityEngine.Rendering.RenderPipeline`). Implement: `Render(ScriptableRenderContext context, Camera[] cameras)` method (called every frame by Unity, receives rendering context + cameras to render). Minimal: `protected override void Render(ScriptableRenderContext context, Camera[] cameras) { foreach (var camera in cameras) { context.SetupCameraProperties(camera); context.Submit(); } }` Renders nothing but valid pipeline (black screen, no errors).
- **RenderPipelineAsset Subclass**: Settings asset (ScriptableObject, stores pipeline configuration). Implement: `CreatePipeline()` method (returns RenderPipeline instance), expose settings (public fields for shadows, quality, features, visible in Inspector). Code: `[CreateAssetMenu(menuName = "Rendering/MyPipeline")] public class MyPipelineAsset : RenderPipelineAsset { public bool enableShadows = true; protected override RenderPipeline CreatePipeline() { return new MyPipeline(this); } }` Assign in Project Settings > Graphics.
- **CommandBuffer Management**: Queue GPU commands (draw calls, state changes, compute dispatches). Best practice: reuse CommandBuffer (create once as member variable, clear every frame, avoid allocations). Code: `CommandBuffer cmd = new CommandBuffer { name = "MainPass" }; void RenderCamera(...) { cmd.Clear(); cmd.ClearRenderTarget(true, true, Color.black); context.ExecuteCommandBuffer(cmd); cmd.Clear(); }` Named buffers (Frame Debugger shows names, essential for debugging).
- **Culling Implementation**: Determine visible objects (frustum culling, occlusion culling). Code: `ScriptableCullingParameters cullingParams; if (!camera.TryGetCullingParameters(out cullingParams)) return; CullingResults cullingResults = context.Cull(ref cullingParams);` Returns: `cullingResults.visibleObjects` (renderers to draw), `cullingResults.visibleLights` (lights affecting scene). Custom culling (modify cullingParams before Cull: adjust culling mask, far plane, disable occlusion, or filter cullingResults after Cull: remove distant objects, limit by layer).
- **Drawing Renderers**: Render visible objects (opaque pass, transparent pass, skybox). Code: `ShaderTagId shaderTag = new ShaderTagId("MyPipelineTag"); SortingSettings sortSettings = new SortingSettings(camera) { criteria = SortingCriteria.CommonOpaque }; DrawingSettings drawSettings = new DrawingSettings(shaderTag, sortSettings); FilteringSettings filterSettings = new FilteringSettings(RenderQueueRange.opaque); context.DrawRenderers(cullingResults, ref drawSettings, ref filterSettings);` ShaderTagId (matches shader tags: `Tags { "LightMode" = "MyPipelineTag" }`, only shaders with tag render), sorting (CommonOpaque = front-to-back, CommonTransparent = back-to-front), filtering (RenderQueueRange.opaque = render queue 0-2500, .transparent = 2501-5000).
- **Custom Lighting**: Implement lighting in shaders (custom SRP has no automatic lighting, shaders must calculate). Per-object data: `_WorldSpaceLightPos0` (main light direction/position), `_LightColor0` (main light color), pass via Material PropertyBlock or shader globals. Multi-light (iterate cullingResults.visibleLights, pass light data to shader via StructuredBuffer or per-draw properties, shader loops lights and accumulates). Baked lighting (support lightmaps: `cmd.EnableShaderKeyword("LIGHTMAP_ON")`, shaders sample `unity_Lightmap` texture, UVs from mesh UV1 channel).

## Best Practices

**Minimal Pipeline Implementation:**
- Start with bare minimum: Render(ScriptableRenderContext, Camera[]) clears screen (black), submit context (context.Submit()), nothing else. Validate (Unity runs without errors, Game view shows black screen, Frame Debugger shows Clear pass).
- Add camera setup: `context.SetupCameraProperties(camera)` (sets view/projection matrices, render target, viewport), draws in correct space (world-space objects appear correctly relative to camera).
- Add culling: `camera.TryGetCullingParameters()` + `context.Cull()` (determines visible objects), store CullingResults (used for drawing).
- Add opaque rendering: `context.DrawRenderers()` with opaque filter (render queue 0-2500, front-to-back sorting), requires shaders with matching tag (create custom unlit shader: `Tags { "LightMode" = "MyPipelineOpaque" }`).
- Add skybox: `context.DrawSkybox(camera)` (renders skybox between opaque and transparent), uses camera's skybox setting.
- Add transparent rendering: `context.DrawRenderers()` with transparent filter (render queue 2501+, back-to-front sorting).
- Result: Basic forward pipeline (opaque, skybox, transparent, no lighting/shadows, ~100 lines C#).

**Shader Integration:**
- Define shader tags: Custom pipeline uses custom tag (e.g., "MyPipeline" instead of "UniversalPipeline" or "ForwardBase"), shaders must match. Shader: `Tags { "RenderPipeline" = "MyPipeline" "LightMode" = "MyPipelineOpaque" }` C# code: `ShaderTagId shaderTag = new ShaderTagId("MyPipelineOpaque");`
- Provide shader library: Create .hlsl include files (lighting functions, transforms, utilities), package in `Packages/com.yourcompany.mypipeline/ShaderLibrary/`, shaders include: `#include "Packages/com.yourcompany.mypipeline/ShaderLibrary/Common.hlsl"`
- Support Shader Graph: Create custom Shader Graph targets (define Master Stack nodes for custom pipeline), advanced but enables visual shader authoring. Alternatively: document manual shader requirements (developers write HLSL shaders, provide templates/examples).
- Fallback shaders: Define fallback (if shader incompatible, Unity uses fallback instead of magenta). Shader: `Fallback "MyPipeline/Unlit"` (custom unlit shader always works, pink = broken).

**Lighting Implementation:**
- Single light approach (minimal): Support one directional light only (main light), pass to shader globally: `cmd.SetGlobalVector("_MainLightPosition", -light.light.transform.forward); cmd.SetGlobalColor("_MainLightColor", light.finalColor);` Shader reads globals, calculates Lambert/Blinn-Phong.
- Multi-light approach (advanced): Pass light data via StructuredBuffer (array of light structs: position, color, range, direction), shader loops lights: `for (int i = 0; i < _LightCount; i++) { lighting += CalculateLight(_Lights[i], worldPos, normal); }` More complex (CPU packing lights, GPU looping, per-light attenuation), supports many lights efficiently.
- Baked lighting support: Enable lightmap keyword: `cmd.EnableShaderKeyword("LIGHTMAP_ON")` (when lightmaps present), shader samples: `half3 bakedGI = SampleLightmap(lightmapUV);` (lightmapUV from mesh UV1), add to final lighting. Light Probes: pass probe data via SphericalHarmonicsL2, shader evaluates SH (complex, Unity provides functions in shader library).
- Shadows (advanced): Render shadow maps (additional pass renders depth from light's perspective, stores in RenderTexture), shader samples shadow map (compares fragment depth to shadow map, applies shadow attenuation). Cascaded shadows (split frustum into cascades, render shadow map per cascade, shader selects appropriate cascade). Significant complexity (50-200 lines for basic shadows, hundreds for production quality).

**Performance Optimization:**
- SRP Batcher: Enable (GraphicsSettings.useScriptableRenderPipelineBatching = true), structure shaders correctly (all material properties in single cbuffer, object data separate), batching automatic (Unity batches draw calls with same shader variant, major performance gain). Validate: Frame Debugger shows "SRP Batch" (multiple objects in single batch), Profiler shows reduced SetPass calls.
- GPU Instancing: Enable in shaders (`#pragma multi_compile_instancing`, UNITY_SETUP_INSTANCE_ID in vertex shader), render with DrawMeshInstanced or indirect drawing. Pass instanced data (StructuredBuffer with per-instance transforms, materials), shader reads instance ID: `UNITY_ACCESS_INSTANCED_PROP(Props, _Color)`. Massive savings (hundreds of objects = single draw call, zero CPU overhead).
- Minimize state changes: Batch by material (sort draw calls by material, render all objects with Material A, then Material B, reduces SetPass calls), batch by shader (even better, all objects with Shader X, regardless of material properties if SRP Batcher active). Profiler target: <100 SetPass calls (more = CPU overhead, batching insufficient).
- Frustum culling: Always enabled (TryGetCullingParameters respects camera frustum, only in-view objects returned by Cull). Occlusion culling: `cullingParams.cullingOptions |= CullingOptions.OcclusionCull` (requires baked occlusion data, optional but saves 30-50% in dense scenes).

**Debugging Support:**
- CommandBuffer names: Name all buffers (`CommandBuffer cmd = new CommandBuffer { name = "DescriptiveName" };`), Frame Debugger shows names (hierarchy of passes, easy to identify which code created which draw calls).
- Debug modes: Implement debug overlays (Rendering Debugger window, show albedo-only, normals-only, lighting-only, overdraw). Volume override: `[Serializable] public class MyDebugSettings : VolumeComponent { public ClampedIntParameter debugMode = new ClampedIntParameter(0, 0, 5); }` Pipeline reads setting, shader uses keyword to render debug mode.
- Validation: Shader validation (check shader compilation errors, log missing shaders), asset validation (ensure all materials have pipeline-compatible shaders, report incompatible), performance warnings (log if draw call count exceeds threshold, light count excessive).

**Platform-Specific:**
- **PC**: Full feature custom pipeline viable (complex effects, compute shaders, advanced techniques), development target (prototype on PC, optimize later for other platforms). Shader complexity: no limits (complex lighting, many texture samples, expensive post-processing). Use case: unique rendering (toon shading for stylized game, custom GI for archviz, novel effects unavailable in URP/HDRP).
- **Consoles**: Custom pipeline viable (similar capability to PC, platform-specific optimizations possible). Leverage console APIs (direct access to platform graphics APIs, bypass Unity overhead, extreme optimization). Use case: AAA studios (have graphics experts, justify custom pipeline for last 10% performance, or unique visual style). Development: requires console devkits (test on hardware frequently, validate performance, memory, thermals).
- **Switch**: Custom pipeline viable if minimal (ultra-lightweight forward renderer, single light, no shadows = better than URP Low), or use URP (already optimized for Switch, custom unlikely to beat unless extremely specific). Use case: extreme optimization (custom pipeline removes all overhead, 5-10% performance gain over URP, justify only if critical for 30 FPS). Complexity: keep simple (complex custom pipeline slower than optimized URP, diminishing returns).
- **Mobile**: Custom pipeline viable if ultra-minimal (unlit rendering, single texture, no lighting = AR/UI-only apps), or use URP (URP already mobile-optimized, custom rarely better). Tile-based GPUs (unique optimizations: render to tile memory, avoid framebuffer loads/stores, URP does this automatically, custom must implement manually). Use case: specialized apps (minimal rendering, URP overhead unjustified, custom 2D pipeline for sprite-only game).

## Common Pitfalls

**Forgetting SRP Batcher Compatibility**: Developer implements custom pipeline (draws objects with DrawRenderers), performance poor (many SetPass calls, each object changes state). SRP Batcher not active (shaders incompatible, material properties not in cbuffer). Symptom: Profiler shows high SetPass call count (1000+ for 1000 objects, should be <10 with batching), Frame Debugger shows no batches (each object separate draw). Solution: Structure shaders correctly (all material properties in `CBUFFER_START(UnityPerMaterial)`, object data in UnityPerDraw cbuffer), enable batching (GraphicsSettings.useScriptableRenderPipelineBatching = true), validate (Frame Debugger shows "SRP Batch" entries, SetPass calls reduced 100x).

**Shader Lighting Missing**: Developer creates custom pipeline (renders geometry), objects appear flat/black (no lighting calculated, shaders expect Built-In lighting which custom pipeline doesn't provide). Symptom: All objects same color (ambient only, or black if no ambient), no shading (no diffuse/specular, flat look), shaders work in Built-In but not custom pipeline. Solution: Implement lighting in shaders (custom pipeline has no automatic lighting, shader must calculate), pass light data (SetGlobalVector for main light position/color), shader calculates (Lambert: `dot(normal, lightDir) * lightColor`, Blinn-Phong: add specular term), or use unlit shaders (if lighting not needed, unlit shaders render albedo only, no lighting calculations).

**Transparent Sorting Issues**: Developer renders transparent objects (context.DrawRenderers with transparent filter), sorting incorrect (objects render in wrong order, near transparent in front of far transparent). Sorting not configured (SortingSettings.criteria not set, or set to CommonOpaque instead of CommonTransparent). Symptom: Transparency popping (wrong depth order, objects flicker as camera moves), Z-fighting (co-planar transparents fight). Solution: Sort back-to-front (SortingSettings.criteria = SortingCriteria.CommonTransparent, Unity sorts by distance from camera, far to near), disable depth write (transparent shaders: `ZWrite Off`, allows proper alpha blending), or reduce transparency (minimize alpha-blended objects, prefer alpha cutout where possible).

**Excessive CommandBuffer Allocation**: Developer creates new CommandBuffer every frame (`CommandBuffer cmd = new CommandBuffer()` inside Render loop), memory leaks (CommandBuffer not disposed, managed memory grows). Garbage collection spikes (GC every few seconds, frame hitches). Symptom: Profiler shows GC Alloc increasing (megabytes per frame), CommandBuffer objects accumulating, frame stutters (GC pauses). Solution: Reuse CommandBuffer (create once as member variable, clear every frame: `cmd.Clear()`, no allocations), or dispose temporary buffers (if creating temporary: `using (CommandBuffer cmd = new CommandBuffer()) { /* use cmd */ }`, auto-disposes, but prefer reuse).

## Tools & Workflow

**Custom Pipeline Template**: Create C# scripts (MyPipeline.cs, MyPipelineAsset.cs), define RenderPipeline + RenderPipelineAsset subclasses, implement Render() method (minimal: clear + submit). Create asset (Assets > Create > Rendering > My Render Pipeline, creates MyPipelineAsset instance). Assign: Project Settings > Graphics > Scriptable Render Pipeline Settings = MyPipelineAsset. Validate: Game view shows rendering (black screen if minimal, or rendered scene if implemented drawing).

**Example Minimal Pipeline**:
```csharp
public class MinimalPipeline : RenderPipeline
{
    CommandBuffer cmd = new CommandBuffer { name = "Render" };
    
    protected override void Render(ScriptableRenderContext context, Camera[] cameras)
    {
        foreach (var camera in cameras)
        {
            context.SetupCameraProperties(camera);
            
            ScriptableCullingParameters cullingParams;
            if (!camera.TryGetCullingParameters(out cullingParams)) continue;
            CullingResults cullingResults = context.Cull(ref cullingParams);
            
            cmd.Clear();
            cmd.ClearRenderTarget(true, true, camera.backgroundColor);
            context.ExecuteCommandBuffer(cmd);
            
            // Opaque
            var sortSettings = new SortingSettings(camera) { criteria = SortingCriteria.CommonOpaque };
            var drawSettings = new DrawingSettings(new ShaderTagId("SRPDefaultUnlit"), sortSettings);
            var filterSettings = new FilteringSettings(RenderQueueRange.opaque);
            context.DrawRenderers(cullingResults, ref drawSettings, ref filterSettings);
            
            context.DrawSkybox(camera);
            
            // Transparent
            sortSettings.criteria = SortingCriteria.CommonTransparent;
            drawSettings.sortingSettings = sortSettings;
            filterSettings.renderQueueRange = RenderQueueRange.transparent;
            context.DrawRenderers(cullingResults, ref drawSettings, ref filterSettings);
            
            context.Submit();
        }
    }
}
```

**Custom Shader Template**:
```hlsl
Shader "MyPipeline/Lit"
{
    Properties
    {
        _BaseColor("Color", Color) = (1,1,1,1)
        _BaseMap("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderPipeline" = "MyPipeline" "RenderType"="Opaque" }
        Pass
        {
            Tags { "LightMode" = "MyPipelineOpaque" }
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            
            CBUFFER_START(UnityPerMaterial)
            float4 _BaseColor;
            float4 _BaseMap_ST;
            CBUFFER_END
            
            sampler2D _BaseMap;
            float3 _MainLightPosition;
            float4 _MainLightColor;
            
            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float2 uv : TEXCOORD0;
            };
            
            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float3 normalWS : TEXCOORD0;
                float2 uv : TEXCOORD1;
            };
            
            Varyings vert(Attributes input)
            {
                Varyings output;
                output.positionCS = UnityObjectToClipPos(input.positionOS);
                output.normalWS = UnityObjectToWorldNormal(input.normalOS);
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                return output;
            }
            
            float4 frag(Varyings input) : SV_Target
            {
                float4 baseMap = tex2D(_BaseMap, input.uv);
                float3 albedo = baseMap.rgb * _BaseColor.rgb;
                
                float3 normal = normalize(input.normalWS);
                float ndotl = max(0, dot(normal, _MainLightPosition));
                float3 lighting = ndotl * _MainLightColor.rgb;
                
                return float4(albedo * lighting, 1);
            }
            ENDHLSL
        }
    }
}
```

**Frame Debugger**: Window > Analysis > Frame Debugger (inspect custom pipeline rendering). Shows: named CommandBuffers ("Render", "Shadows", etc., from cmd.name), draw calls per pass (expand to see individual objects), shader passes ("MyPipelineOpaque", "MyPipelineTransparent"). Usage: validate rendering order (opaque before transparent, shadows before lighting), identify missing draws (objects not rendering = wrong shader tag or filtering), debug state changes (render target switches, shader changes).

**Profiler Integration**: Custom pipeline shows in Profiler (Profiler > Rendering). Markers: wrap code in profiler samples: `using (new ProfilingScope(cmd, new ProfilingSampler("My Pass"))) { /* rendering code */ }` Profiler shows "My Pass" marker (timing, hierarchy). GPU Profiler: shows GPU timing per CommandBuffer (validate GPU cost of custom passes, compare to URP/HDRP for performance validation).

**Rendering Debugger**: Create custom debug modes (Window > Analysis > Rendering Debugger, add custom panels). Implement: `DebugUI.Panel` with custom controls (dropdown for debug modes, checkboxes for features), read in pipeline: `var debugMode = myDebugSettings.debugMode.value;` Apply in shader (via keyword or property, shader renders albedo-only/normals-only/etc.). Usage: validate rendering (albedo mode shows textures correct, normals mode shows normal maps correct), performance profiling (disable features, measure impact).

## Related Topics

- [18.1 SRP Architecture](18-01-SRP-Architecture.md) - SRP fundamentals
- [18.2 Universal Render Pipeline](18-02-Universal-Render-Pipeline.md) - URP reference
- [12.2 Writing Custom Shaders](12-02-Shader-Types.md) - Custom shader development
- [5.1 CPU Optimization](05-01-CPU-Side-Optimization.md) - Performance optimization

---

[← Previous: 18.4 Built-In Render Pipeline](18-04-Built-In-Render-Pipeline.md) | [Next: Chapter 19 →](../chapters/19-Visual-Effects.md)
