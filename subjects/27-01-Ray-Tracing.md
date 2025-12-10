# 27.1 Ray Tracing

[← Back to Chapter 27](../chapters/27-Advanced-Topics.md) | [Main Index](../README.md)

Real-time ray tracing brings film-quality lighting, reflections, and shadows to games by simulating light ray bounces through scenes. Hardware-accelerated ray tracing on RTX/RDNA GPUs and recent consoles enables progressive quality improvements: primary visibility with full reflections/shadows at lower sample rates, denoising for interactive quality, or hybrid rendering (rasterization + ray-traced effects). This section covers ray tracing fundamentals, hardware acceleration (DXR, OptiX, RDNA), shader pipeline, denoising, and cross-platform trade-offs.

---

## Overview

Ray tracing has transitioned from offline rendering to real-time through hardware acceleration and algorithmic improvements. NVIDIA RTX (2018+), AMD RDNA (2020+), and current-gen consoles (PS5, Xbox Series X) provide dedicated ray-tracing cores accelerating BVH traversal and ray-triangle intersection. This hardware support reduces ray-tracing overhead from 50-100x rasterization cost to 2-10x, enabling production implementations.

Real-time ray tracing operates in two modes: full ray tracing (replace rasterization entirely, 30-60 fps at 1440p with heavy denoising) or hybrid (ray-traced shadows/reflections over rasterized base, maintains 60+ fps). Most production games use hybrid: rasterization for G-buffer, ray tracing for shadows (1 ray per pixel) and reflections (1-4 rays), with aggressive denoising (NVIDIA OptiX, AMD FSR 3, temporal filtering). PC (RTX 2080 Ti: 16 RT cores/SM, 4352 rays/frame at 1080p 60fps) and consoles (PS5: 5.5 TFLOPS ray-traced, Xbox Series X: 6 TFLOPS) push ray-traced effects into mainstream graphics pipelines.

Key challenges: ray tracing is memory-intensive (BVH traversal, material buffers, textures), shader-complex (closest-hit, miss, any-hit shaders), and noise-prone (stochastic sampling requires heavy filtering). Denoising is critical to production quality: temporal history accumulates samples over frames (2-4 samples/pixel/frame with reprojection), spatial denoisers remove fireflies, or ML denoisers (OptiX, DirectML) achieve high quality at 1-2 samples/pixel.

## Key Concepts

- **Hardware Ray Tracing**: Dedicated GPU cores (RT cores, RDNA ray engines) accelerate BVH traversal and ray-triangle tests. NVIDIA RTX: 128 rays/RT core/clock. Enables 10-30x speedup vs compute shaders. AMD RDNA3: 512 rays/engine/clock. Consoles: PS5 100 rays/clock sustained, Xbox Series X 120+ rays/clock.

- **DXR API**: Microsoft's ray-tracing API for Windows/Xbox. Integrates with D3D12. Defines HLSL shader stages: ray generation, closest-hit, miss, any-hit shaders. Acceleration structures built on GPU (2-10ms for complex scenes).

- **Acceleration Structures (BVH)**: Spatial hierarchies enabling fast ray intersection. Bottom-level AS (BLAS): per-mesh BVH. Top-level AS (TLAS): instance hierarchy. Rebuild cost varies by geometry complexity.

- **Denoising**: Essential for quality. Temporal accumulation (8-16 frames, motion-vector reprojection), spatial denoise (bilateral filter), or ML denoisers (OptiX, FSR 3). Critical: 1 ray/pixel + OptiX achieves 16+ ray quality.

- **Shadow Rays**: One ray per pixel toward light. Cost: 1-2ms per 1080p. Replaces shadow maps with perfect accuracy, handles transparency correctly.

- **Reflection Rays**: 1-4 rays/pixel for reflections. Cost: 5-15ms at 1440p depending on sample rate. Requires temporal + spatial denoising.

- **Performance Budgets**: 60 fps = 16.67ms frame. Ray-tracing shadows 2-4ms (12-24% budget), reflections 3-5ms (18-30%), denoising 2-3ms. Total 7-12ms, leaving headroom for rasterization.

## Best Practices

**Ray Tracing Setup:**
- Build acceleration structures asynchronously. Static geometry: build once at load. Dynamic: rebuild TLAS per frame (0.1-0.5ms for 100 instances).
- Compact BLAS/TLAS post-build: reduces memory 30-50%, improves cache and traversal speed 2-5%.
- Use single main-light shadow rays only. Multi-light shadow rays prohibitive (20ms+). Use shadow maps/deferred for secondary lights.

**Shadow Rays (Cost-Effective):**
- One ray per pixel toward main light. Cost: 1-2ms at 1440p. Any-hit shaders skip transparent pixels for correct self-shadowing.
- Soft shadows: 16 rays/pixel for quality, or temporal jitter (rotate pattern per frame, 4-frame accumulation = 16-sample equivalent).
- Console optimization: PS5/Xbox Series X have efficient ray hardware; ray-traced shadows are faster than PCSS.

**Reflection Rays (Denoising Critical):**
- Start with 1-2 rays/pixel. Temporal reprojection via motion vectors accumulates over 8 frames (equivalent to 32+ rays).
- Importance-sample by roughness: mirror surfaces need more rays, diffuse surfaces converge faster.
- ML denoiser (OptiX, FSR 3): 1 ray/pixel + OptiX = 4-8 ray quality, total 8-12ms (vs 20ms for many-sample).

**Implementation Example (C#):**
```csharp
private void InitializeRayTracing()
{
    var buildFlags = RayTracingAccelerationStructure.ManagementMode.Manual;
    m_RayTracingAS = new RayTracingAccelerationStructure(buildFlags, RayTracingAccelerationStructure.RayTracingInstanceFlag.None);
    
    foreach (var meshCollider in FindObjectsOfType<MeshCollider>())
    {
        var mesh = meshCollider.GetComponent<MeshFilter>().sharedMesh;
        m_RayTracingAS.AddInstance(mesh, Matrix4x4.identity);
    }
    m_RayTracingAS.Build();
}

private void TraceShadows(CommandBuffer cmd, RenderTexture shadowResult)
{
    var shader = Resources.Load<RayTracingShader>("Shaders/ShadowTrace");
    cmd.SetRayTracingShaderPass(shader, "ShadowPass");
    cmd.SetRayTracingAccelerationStructure(shader, "g_AccelStruct", m_RayTracingAS);
    cmd.SetRayTracingTextureParam(shader, "g_Output", shadowResult);
    cmd.DispatchRays(shader, "ShadowGeneration", (uint)Screen.width, (uint)Screen.height, 1);
}
```

**Platform Specifics:**

*PC (RTX/RDNA):*
- RTX 2080 Ti+: 1 shadow + 2 reflection rays at 1440p.
- Detect GPU, scale ray counts dynamically.
- High-end denoising: OptiX + frame interpolation.

*Console (PS5/Xbox Series X):*
- Ray-traced shadows: 1-2 rays/pixel, 2-3ms (efficient hardware).
- Reflections: 1-2 rays on PS5, 2-3 on Xbox Series X. Total 4-6ms with denoising.
- VRAM budget: BVH 500MB-1.5GB (3% of 16GB). Mandatory structure compaction.

*Mobile (None):*
- No ray-tracing hardware. Use ray-cast (CPU) for shadows, screen-space reflections or cubemaps for reflections.

## Common Pitfalls

**1. Unbounded Shadow Ray Cost**
- *Symptom*: Shadow rendering 15-20ms, frame drops to 30-40 fps.
- *Cause*: Ray-tracing shadows for multiple lights (directional + 20 point lights).
- *Solution*: Ray-trace main light only (2-3ms). Use shadow maps for secondary lights.

**2. Memory Pressure from Uncompacted BVH**
- *Symptom*: Frame stutters, VRAM >95% despite small scene.
- *Cause*: Uncompacted acceleration structures (50-100MB uncompacted vs 20-40MB compacted).
- *Solution*: CompactRaytracingAccelerationStructure post-build. 40-60% memory savings, 2-5% traversal speedup.

**3. Noisy Ray-Traced Reflections**
- *Symptom*: Reflections flicker, sparkle on rough surfaces even with temporal accumulation.
- *Cause*: Insufficient denoising (1 ray/pixel + cheap bilateral filter = very noisy).
- *Solution*: Proper temporal accumulation (8-16 frame history), spatial denoise post-temporal, consider ML denoise (OptiX).

**4. Ghosting on Dynamic Objects**
- *Symptom*: Reflections lag character movement, ghosting visible.
- *Cause*: Temporal reprojection assumes static geometry.
- *Solution*: Disable temporal accumulation for dynamic objects. Increases sample rate needed, eliminates ghosting.

## Tools & Workflow

**HLSL Ray Tracing Shaders:**
```hlsl
[shader("raygeneration")]
void ShadowGeneration()
{
    uint2 launchIdx = DispatchRaysIndex().xy;
    float2 uv = (launchIdx + 0.5) / DispatchRaysDimensions().xy;
    float3 pos = g_Position[launchIdx].xyz;
    float3 normal = g_Normal[launchIdx].xyz;
    
    RayDesc ray;
    ray.Origin = pos + normal * 0.01;
    ray.Direction = normalize(g_LightDirection);
    ray.TMin = 0.0;
    ray.TMax = 1000.0;
    
    TraceRay(g_AccelStruct, RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH, 0xFF, 0, 1, 0, ray, payload);
    g_Output[launchIdx] = float4(payload.hitT > 0 ? 0.0 : 1.0, 0, 0, 1.0);
}

[shader("closesthit")]
void ClosestHit(inout Payload payload, BuiltInTriangleIntersectionAttributes attr)
{
    payload.hitT = RayTCurrent();
    payload.instanceIdx = InstanceIndex();
}

[shader("miss")]
void Miss(inout Payload payload)
{
    payload.hitT = -1.0;
}
```

**Profiling:**
- NVIDIA Nsight: ray-tracing occupancy, BVH traversal efficiency, shader time.
- PIX (Xbox): ray timeline, ray count per frame, denoiser latency.
- AMD GPU Profiler: RDNA ray-engine metrics, memory bandwidth.

## Related Topics

- [27.2 Virtual Reality](27-02-Virtual-Reality.md) — VR latency constraints for ray tracing
- [27.3 GPU-Driven Rendering](27-03-GPU-Driven-Rendering.md) — Indirect dispatch and adaptive ray counts
- [13 Rendering Techniques](../chapters/13-Rendering-Techniques.md) — Hybrid rasterization + ray tracing workflows
- [04 Profiling Tools](../chapters/04-Profiling-And-Analysis-Tools.md) — Ray-tracing performance profiling

---

[← Previous: 26.4 Unity Debug Tools](26-04-Unity-Debug-Tools.md) | [Next: 27.2 Virtual Reality →](27-02-Virtual-Reality.md)
