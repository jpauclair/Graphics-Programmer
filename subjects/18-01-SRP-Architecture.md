# 18.1 SRP Architecture

[← Back to Chapter 18](../chapters/18-Scriptable-Render-Pipeline.md) | [Main Index](../README.md)

Scriptable Render Pipeline (SRP) allows full control over rendering: C# scripts define render loop (replace Unity's built-in rendering), customizable features (lighting models, post-processing, culling), and platform-specific optimizations.

---

## Overview

SRP Core Concept: Unity traditionally used built-in render pipeline (fixed forward/deferred paths, hardcoded C++ rendering). SRP replaces this with C# scripts (RenderPipeline class defines rendering steps), giving developers full control (custom lighting, effects, optimizations). Unity provides: URP (Universal Render Pipeline, mobile/cross-platform optimized), HDRP (High-Definition Render Pipeline, high-end PC/console graphics), or custom SRP (write from scratch, complete control). SRP benefits: tailored rendering (optimize for specific game needs, remove unused features), platform targeting (mobile-specific pipeline, console-specific), easier debugging (C# code visible/editable, not black-box C++).

Render Loop: SRP `Render()` method called every frame (receives ScriptableRenderContext + Camera array). Typical steps: 1) Culling (ScriptableRenderContext.Cull, determines visible objects), 2) Rendering (CommandBuffer draws geometry, lighting, shadows), 3) Post-processing (effects applied to rendered image), 4) Present (submit CommandBuffer, GPU executes). Developer controls all steps (can reorder, add custom passes, skip steps entirely). Example: custom pipeline renders opaque objects, then transparent, then UI (order controlled explicitly in C# code).

RenderGraph API: Modern SRP (Unity 2022+) uses RenderGraph (declarative rendering, define passes and resources, RenderGraph automatically schedules/optimizes). Benefits: automatic resource management (transient textures allocated/released automatically), pass reordering (RenderGraph optimizes execution order), efficient memory (resources reused across passes, minimal allocations). Code: `using (var builder = renderGraph.AddRenderPass("Opaque", out var passData)) { builder.UseColorBuffer(colorTarget, 0); builder.SetRenderFunc((PassData data, RenderGraphContext ctx) => { /* render code */ }); }` RenderGraph emerging standard (URP/HDRP transitioning, custom SRPs should adopt).

## Key Concepts

- **RenderPipelineAsset**: ScriptableObject defining pipeline settings (quality levels, features, shader variants). Project Settings > Graphics > Scriptable Render Pipeline Settings = assign RenderPipelineAsset (e.g., UniversalRenderPipelineAsset, HDRenderPipelineAsset, or custom). Asset creates RenderPipeline instance (CreatePipeline method returns RenderPipeline subclass, instantiated at runtime). Pipeline settings: shadow quality, texture resolutions, post-processing enabled, reflection probe settings. Multiple assets: different quality levels (Low/Medium/High assets for different platforms, switch at runtime via `GraphicsSettings.renderPipelineAsset`).
- **RenderPipeline.Render()**: Core method called every frame by Unity rendering system (receives `ScriptableRenderContext` + `Camera[]`). Implementation: iterate cameras (foreach camera in cameras), setup camera (set view/projection matrices, render target), cull (ScriptableRenderContext.Cull), render passes (opaque, skybox, transparent), post-process, submit (ScriptableRenderContext.Submit, executes GPU commands). Developer defines all steps (complete control over rendering order, custom passes, platform-specific logic).
- **CommandBuffer**: Queue of GPU commands (draw calls, state changes, compute dispatches). Created in C#: `CommandBuffer cmd = new CommandBuffer { name = "MyPass" };` Commands: `cmd.DrawMesh()`, `cmd.SetRenderTarget()`, `cmd.Blit()`, `cmd.DispatchCompute()`. Executed: `context.ExecuteCommandBuffer(cmd); cmd.Clear();` (submits to ScriptableRenderContext, CommandBuffer reused). Efficient (batches GPU commands, reduces overhead), reusable (clear + reuse same CommandBuffer every frame).
- **Culling**: Determines visible objects (frustum culling, occlusion culling). SRP code: `ScriptableCullingParameters cullingParams; camera.TryGetCullingParameters(out cullingParams); CullingResults cullingResults = context.Cull(ref cullingParams);` Result: `cullingResults.visibleObjects` (list of renderers to draw), `cullingResults.visibleLights` (lights affecting scene). Custom culling: modify `cullingParams` (adjust far plane, layer masks, disable occlusion), or post-process cullingResults (filter objects based on custom criteria).
- **Drawing**: Render visible objects (opaque, then transparent). SRP code: `DrawingSettings drawSettings = new DrawingSettings(shaderTagId, sortingSettings); FilteringSettings filterSettings = new FilteringSettings(RenderQueueRange.opaque); context.DrawRenderers(cullingResults, ref drawSettings, ref filterSettings);` DrawingSettings: shader pass (e.g., "SRPDefaultUnlit"), sorting (front-to-back for opaque, back-to-front for transparent). FilteringSettings: render queue (opaque vs transparent), layer mask (which layers to draw). Multiple passes (opaque pass, transparent pass, custom passes for effects).
- **RenderGraph Passes**: Declarative rendering (define pass dependencies, RenderGraph schedules). Pass definition: `builder.UseColorBuffer(colorTarget, 0); builder.ReadTexture(depthTexture); builder.SetRenderFunc((data, ctx) => { /* rendering code */ });` Dependencies: UseColorBuffer (writes to texture), ReadTexture (reads texture), automatic scheduling (RenderGraph executes passes in optimal order). Benefits: transient resources (textures allocated only when needed, released immediately after last use), memory optimization (RenderGraph pools resources, minimal allocations).

## Best Practices

**Custom SRP Setup:**
- Create RenderPipelineAsset: ScriptableObject subclass (defines settings, e.g., shadow distance, MSAA quality), `CreatePipeline()` returns RenderPipeline instance. Code: `[CreateAssetMenu(menuName = "Rendering/MyRenderPipeline")] public class MyRenderPipelineAsset : RenderPipelineAsset { protected override RenderPipeline CreatePipeline() { return new MyRenderPipeline(this); } }`
- Create RenderPipeline: Subclass RenderPipeline, implement `Render()` method. Code: `public class MyRenderPipeline : RenderPipeline { protected override void Render(ScriptableRenderContext context, Camera[] cameras) { foreach (var camera in cameras) { RenderCamera(context, camera); } } }`
- Assign asset: Project Settings > Graphics > Scriptable Render Pipeline Settings (drag MyRenderPipelineAsset asset). Unity uses custom pipeline for all rendering.
- Iterate: Start simple (render opaque objects only), add features incrementally (shadows, post-processing, custom effects). Test frequently (validate rendering correct after each feature addition).

**Efficient CommandBuffer Usage:**
- Reuse CommandBuffers: Create once (cached as member variable), clear every frame (`cmd.Clear()`), reuse. Avoid: creating new CommandBuffer every frame (allocation overhead, GC pressure). Code: `CommandBuffer cmd; void Start() { cmd = new CommandBuffer { name = "MainPass" }; } void Render() { cmd.Clear(); /* add commands */ context.ExecuteCommandBuffer(cmd); }`
- Name buffers: Descriptive names (Frame Debugger shows CommandBuffer names, aids debugging). Example: `CommandBuffer shadowCmd = new CommandBuffer { name = "Shadows" };` (Frame Debugger displays "Shadows" pass, easy to identify).
- Minimize ExecuteCommandBuffer calls: Batch commands into single CommandBuffer (execute once), avoid execute per command (overhead per execution). Good: add 100 draw calls to buffer, execute once. Bad: 100 ExecuteCommandBuffer calls (100x overhead).
- Clear after use: `cmd.Clear()` releases temporary resources (render targets, buffers), prevents leaks. Call after execute: `context.ExecuteCommandBuffer(cmd); cmd.Clear();`

**Culling Optimization:**
- Frustum culling: Enabled by default (cullingParams from camera.TryGetCullingParameters), respects camera frustum (only objects in view frustum rendered). Validate: Scene view shows culling (objects outside frustum greyed out), Frame Debugger shows only visible objects.
- Occlusion culling: Enable in cullingParams (`cullingParams.cullingOptions |= CullingOptions.OcclusionCull`), requires baked occlusion data (Window > Rendering > Occlusion Culling, bake scene). Savings: 30-50% objects culled in dense scenes (objects behind buildings/terrain removed).
- Layer mask culling: Filter by layer (`cullingParams.cullingMask = camera.cullingMask`), allows per-camera layer control (UI camera renders UI layer only, main camera renders world). Custom: `cullingParams.cullingMask &= ~(1 << LayerMask.NameToLayer("IgnoredLayer"))`
- Custom distance culling: Modify `cullingParams.maximumVisibleLights` (limit lights processed), or post-cull filtering (iterate cullingResults.visibleObjects, remove distant objects). Example: mobile platforms cull objects beyond 100m (saves draw calls, reduces overdraw).

**RenderGraph Integration:**
- Adopt RenderGraph: Modern SRP best practice (automatic resource management, optimized scheduling). Setup: `RenderGraph renderGraph = new RenderGraph();` Render: `renderGraph.BeginFrame(); /* add passes */ renderGraph.Execute(context, cmd); renderGraph.EndFrame();`
- Define resources: `TextureHandle colorTarget = renderGraph.ImportBackbuffer(camera.targetTexture);` RenderGraph manages lifetime (allocates/releases automatically).
- Declare dependencies: `builder.UseColorBuffer(colorTarget, 0);` (writes color), `builder.ReadTexture(depthTexture);` (reads depth). RenderGraph schedules passes (ensures depth written before read, optimal ordering).
- Transient resources: `TextureDesc desc = new TextureDesc(width, height); TextureHandle tempRT = renderGraph.CreateTexture(desc);` Allocated only during pass execution (released immediately after, no persistent memory cost).

**Platform-Specific Pipelines:**
- Separate assets per platform: MyRenderPipeline_PC (high-quality, complex features), MyRenderPipeline_Mobile (simplified, optimized). Build settings: assign appropriate asset per platform (Editor > Build Settings > Player Settings > Graphics > SRP Asset).
- Feature flags: RenderPipelineAsset settings (bool enableShadows, bool enablePostProcessing), custom pipeline checks flags (if (asset.enableShadows) RenderShadows();). Mobile asset: shadows disabled, PC enabled.
- Shader variants: SRP shaders tagged (e.g., "RenderPipeline" = "MyPipeline"), shader features (multi_compile SHADOWS_ON, platform-specific keywords). Mobile shaders: simpler lighting (Lambert vs PBR), fewer texture samples.
- Runtime switching: `GraphicsSettings.renderPipelineAsset = mobileAsset;` (dynamically switch pipeline, e.g., quality settings menu). Requires: compatible shaders (shaders work with both pipelines, or reload materials after switch).

**Platform-Specific:**
- **PC**: Custom SRP viable (complex features, compute shaders, raytracing integration), URP/HDRP recommended (feature-rich, well-tested, automatic optimizations). Development: HDRP for AAA graphics (physically-based sky, volumetrics, high-quality post-processing), or custom SRP for specific needs (stylized rendering, non-photorealistic, custom lighting models).
- **Consoles**: URP recommended (optimized for console hardware, good performance), or custom SRP (platform-specific optimizations, leverage console APIs directly). HDRP viable (PS5/Xbox Series X support high-end features), but expensive (ensure performance targets met). Development: test on target hardware frequently (console-specific bottlenecks, memory constraints).
- **Switch**: URP essential (optimized for low-power hardware, mobile-class rendering), custom SRP if minimal features (ultra-simplified pipeline, removes all unnecessary overhead). HDRP not viable (too expensive, Switch lacks hardware features). Settings: URP Low quality preset (shadows low-res, post-processing minimal, MSAA disabled).
- **Mobile**: URP required (designed for mobile, tile-based rendering optimizations, ASTC support), custom SRP only if extremely simple (basic forward rendering, single light, no shadows). HDRP not viable (mobile unsupported, too expensive). Settings: URP Mobile quality preset (single directional light, no shadows, minimal post-processing, MSAA 2x max).

## Common Pitfalls

**Forgetting Context.Submit()**: Developer implements SRP Render() method (adds CommandBuffers, draws objects), but forgets `context.Submit()` at end of frame. Nothing renders (CommandBuffers queued but never executed, screen black). Symptom: Game view black, Scene view works (Scene view uses separate rendering), no errors (code runs, just doesn't submit). Solution: Always call `context.Submit()` at end of Render() method (after all rendering complete, submits all queued work to GPU).

**Shader Compatibility**: Developer switches from Built-In to URP (assigns URP RenderPipelineAsset), existing materials turn magenta (Built-In shaders incompatible with URP, different shader tags/lighting). Symptom: All materials pink/magenta (Unity fallback shader, "shader not compatible"), Game view broken. Solution: Upgrade materials (Edit > Render Pipeline > Universal > Upgrade Project Materials to URP, converts shaders), or use URP-compatible shaders (URP/Lit, URP/Unlit instead of Standard, Universal shader library instead of Built-In).

**Infinite Recursion in RenderGraph**: Developer creates RenderGraph pass (reads texture A, writes texture A), circular dependency (pass reads its own output). RenderGraph errors or hangs (can't resolve dependency, infinite loop in scheduling). Symptom: Console errors ("Circular dependency detected"), Unity freeze (RenderGraph stuck in scheduling loop), black screen. Solution: Separate read/write textures (use tempTexture as intermediate, read A → process → write tempTexture → read tempTexture → write A), or ping-pong buffers (alternate between two textures, read from one while writing to other).

**CommandBuffer Leaks**: Developer creates new CommandBuffer every frame (`cmd = new CommandBuffer()`), never disposes. Memory leak (CommandBuffers accumulate, managed memory grows indefinitely). Symptom: Profiler shows CommandBuffer memory increasing every frame (megabytes of CommandBuffer objects), eventual crash (out of memory). Solution: Reuse CommandBuffer (create once, clear + reuse every frame), or dispose (if creating temporary buffers, call `cmd.Dispose()` when done).

## Tools & Workflow

**Frame Debugger**: Window > Analysis > Frame Debugger (shows all rendering steps, CommandBuffer execution). SRP debugging: shows SRP passes (named CommandBuffers appear as hierarchy, "Shadows", "Opaque", "Transparent"), draw calls per pass (expand pass to see individual draws), state changes (render targets, shader changes). Usage: step through frame (click events, see what renders), identify bottlenecks (passes with many draw calls, expensive shaders).

**RenderDoc**: External tool (captures GPU frame, detailed analysis). SRP profiling: capture Unity frame (attach RenderDoc, F12 to capture), analyze passes (see exact GPU commands, timing per pass), debug shaders (inspect shader inputs/outputs, pixel history). Download: renderdoc.org, install, attach to Unity Editor or standalone build.

**URP Template Project**: Unity Hub > New Project > URP Template (pre-configured URP project, sample scenes, example shaders). Study: inspect UniversalRenderPipelineAsset (see settings structure), read URP package code (Packages/Universal RP/Runtime, C# source for URP rendering), learn from examples (URP shaders in Shader Graph, lighting/shadows implementation).

**Custom SRP Example**: Minimal SRP implementation (renders opaque objects only):
```csharp
public class MyRenderPipeline : RenderPipeline
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
            cmd.ClearRenderTarget(true, true, Color.black);
            context.ExecuteCommandBuffer(cmd);
            
            ShaderTagId shaderTag = new ShaderTagId("SRPDefaultUnlit");
            SortingSettings sortSettings = new SortingSettings(camera);
            DrawingSettings drawSettings = new DrawingSettings(shaderTag, sortSettings);
            FilteringSettings filterSettings = new FilteringSettings(RenderQueueRange.opaque);
            context.DrawRenderers(cullingResults, ref drawSettings, ref filterSettings);
            
            context.Submit();
        }
    }
}
```

**RenderGraph Debugging**: Console messages (RenderGraph logs errors, circular dependencies, resource issues). Profiler: Profiler > Rendering > RenderGraph (shows pass execution times, resource allocations). Frame Debugger: RenderGraph passes appear (automatic pass names, or custom names via builder.SetName("PassName")).

## Related Topics

- [18.2 Universal Render Pipeline](18-02-Universal-Render-Pipeline.md) - URP details
- [18.3 High-Definition Render Pipeline](18-03-High-Definition-Render-Pipeline.md) - HDRP features
- [18.5 Custom Render Pipelines](18-05-Custom-Render-Pipelines.md) - Building custom SRP
- [12.3 Shader Compilation and Variants](12-03-Shader-Compilation-Variants.md) - SRP shader compatibility

---

[← Previous: Chapter 17](../chapters/17-Level-Of-Detail-Systems.md) | [Next: 18.2 Universal Render Pipeline →](18-02-Universal-Render-Pipeline.md)
