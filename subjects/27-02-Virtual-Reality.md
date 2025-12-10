# 27.2 Virtual Reality

[← Back to Chapter 27](../chapters/27-Advanced-Topics.md) | [Main Index](../README.md)

Virtual reality demands extreme performance constraints: 90 fps (11.1ms frame budget) on Meta Quest 3, 90-144 fps on PC VR. Two cameras render simultaneously (3.6ms per eye), leaving minimal compute headroom. Motion-to-photons latency must be <20ms to prevent nausea. This section covers VR rendering optimization, latency reduction, stereo rendering techniques, foveated rendering, and platform-specific approaches (Meta Quest 3, SteamVR, PSVR2).

---

## Overview

VR rendering differs fundamentally: two camera viewpoints (6cm stereo baseline), extreme latency sensitivity (motion-to-photons <20ms), and high refresh rates (90+ fps). Frame budget at 90 fps: 11.1ms total (GPU 8-9ms, CPU 2-3ms). This forces radical optimization: render at 72 fps using TimeWarp, or foveated rendering (render 4x fewer pixels peripherally), or reduce geometry 50% vs 2D graphics.

Latency is critical: motion-to-photons latency (time from head movement to display update) >20ms causes motion sickness. Quest 3 achieves 15-18ms (excellent). Async TimeWarp (OS compositor applies head motion post-render) reduces perceived latency by 5-7ms, enabling smooth 72 fps feeling like 90+ fps. Stereo rendering: instanced stereo (single draw call, both eyes) costs 1.3x vs mono, traditional stereo costs 2x.

Performance requirements force architectural changes: no forward lighting (too expensive), use compute-based light culling instead. Geometry LOD aggressive (100k-200k triangles vs 1M-2M for 2D). Texture budget strict: 256MB max (all ASTC compressed). No dynamic allocations during frame (GC stalls cause frame drops = nausea).

## Key Concepts

- **Frame Budget**: 11.1ms (90 fps) unbreakable limit. Quest 3: GPU 8-9ms, CPU 2-3ms. Exceed = frame drop + motion sickness spike. 120 fps (8.33ms) even tighter. No scaling; must sustain fixed framerate.

- **Motion-to-Photons Latency**: Time from head rotation to image pixel update. Target <20ms (ideal 15ms). Breakdown: tracking 5-8ms, GPU render 4-8ms, display 8-12ms. Latency >25ms = judder, >35ms = nausea. VR unplayable at 30 fps (33ms latency alone).

- **Async TimeWarp**: OS compositor applies head pose correction post-render. Renders at 72 fps, warps to 90 fps using latest head tracking. Reduces perceived latency 5-7ms. Requires accurate pose prediction and fast tracking.

- **Instanced Stereo**: Single draw call outputs to both eyes via SV_ViewportIndex (left/right). Cost: 1.3x vs mono (VS overhead). Bandwidth savings: shared depth buffer, shared G-buffer queries. Eliminates 2x geometry bottleneck.

- **Foveated Rendering**: Render high-quality center, lower quality periphery. Eye-tracking determines fovea. Saves 50-70% pixel shading. Imperceptible to user (human peripheral vision is low-acuity). Quest 3: fixed fovea (center). PC VR: variable-rate shading (VRS).

- **IPD (Interpupillary Distance)**: Distance between eyes (~63mm average, 58-68mm range). Wrong IPD = eye strain, discomfort. Rendering: offset cameras by IPD/2 left/right. Incorrect IPD causes vertical misalignment (headaches).

- **Reprojection**: If GPU misses deadline, OS reproj approximates missing frame via depth warping. Preserves appearance if geometry static, fails on disocclusions. Better: hit deadline.

## Best Practices

**Performance Budgeting (Quest 3, 90 fps):**
- Frame budget: 11.1ms GPU, 2-3ms CPU. Non-negotiable. Use GPU profilers every frame.
- Render target: 1832x1920 per eye. Reduce geometry LOD 30-40% vs 2D. Max 100k-200k visible triangles.
- Batch limit: <50 draw calls (vs 1000+ for 2D deferred). Geometry instancing, merged meshes mandatory.
- Texture budget: 256MB max (ASTC compressed). No uncompressed textures.

**Stereo Rendering (Instanced):**
- VS outputs SV_ViewportIndex (0=left eye, 1=right eye). Both eyes in same draw call. Cost: 5-10% VS overhead.
- Dual-view matrices: VS reads eye offset per viewport:
  ```hlsl
  if (viewportIndex == 0) {
      viewPos.x -= g_IPD / 2.0;  // Left eye
      outPos = mul(float4(viewPos, 1), g_LeftViewProj);
  } else {
      viewPos.x += g_IPD / 2.0;  // Right eye
      outPos = mul(float4(viewPos, 1), g_RightViewProj);
  }
  ```
- Shared depth buffer (small stereo baseline works). Saves VRAM, bandwidth.
- Avoid geometry shaders (poor mobile performance). Use instanced VS exclusively.

**Latency Minimization:**
- Maintain frame deadline religiously. Miss one frame = 11.1ms latency spike + nausea. Profile CPU + GPU separately.
- Predict head pose: predict 11.1ms into future. Reduces latency 2-3ms (net latency ~1-2ms reduction).
- Rely on TimeWarp as safety net (not guarantee). Publish poses correctly to compositor.
- Input latency: process input late (close to display time). Minimize input-to-visual lag (ideal <10ms).

**Practical C# Setup (Unity VR):**
```csharp
public class VROptimizedRenderer : MonoBehaviour
{
    void Start()
    {
        XRSettings.useOcclusionMesh = true;  // 20-30% fillrate savings
        XRSettings.renderViewportScale = 1.0f;
        Application.targetFrameRate = 90;
        QualitySettings.vSyncCount = 0;
    }
    
    void OnPreRender()
    {
        var headPose = InputTracking.GetLocalPosition(XRNode.Head);
        var headRot = InputTracking.GetLocalRotation(XRNode.Head);
        
        // Predict pose 11.1ms forward
        PredictHeadPose(ref headPose, ref headRot, 0.0111f);
        UpdateCameras(headPose, headRot);
        
        // Monitor frame time, reduce LOD if needed
        if (GetGPUFrameTime() > 9.0f)
            QualitySettings.lodBias = 0.7f;  // Cut geometry ~30%
    }
    
    void UpdateCameras(Vector3 headPos, Quaternion headRot)
    {
        float ipd = 0.064f;  // 64mm
        var leftCam = GetComponent<Camera>();
        leftCam.stereoTargetEye = StereoTargetEyeMask.Left;
        leftCam.transform.localPosition = headPos + headRot * new Vector3(-ipd/2, 0, 0);
        leftCam.transform.localRotation = headRot;
    }
}
```

**Foveated Rendering (High-End PC VR):**
- Variable-rate shading (VRS): mark peripheral tiles for lower shading rate. 40-50% fillrate savings.
- Fixed fovea: center 50% resolution, outer 50% half-resolution. Imperceptible, 25% fillrate saved.
- Eye-tracking fovea (future): 4° cone high-quality, rest low-quality. 60-70% fillrate savings.

**Comfort & Motion Sickness:**
- Maintain 90 fps floor (never <60 fps). Single frame drop = 16.6ms latency spike = nausea.
- Smooth camera motion: no sudden acceleration. Linear motion OK, angular accel bad.
- Vignette effect during motion: reduces vestibulo-ocular reflex mismatch, prevents nausea in 40% susceptible users.
- Snap-turn toggle: 45° snap < smooth rotation. Allow toggle for comfort.

**Platform Specifics:**

*Meta Quest 3 (90 fps):*
- GPU 8-9ms, CPU 2-3ms. Render 1832x1920 per eye. Use Vulkan (15-20% faster than GLES).
- Forward+ (grid-based light culling, forward rendering). Deferred too expensive (3-4ms G-buffer).
- 20-40 lights, 50-100k triangles visible. Occlusion mesh: 20-30% fillrate savings.

*PC VR / SteamVR (90-144 fps):*
- RTX 3070+ for 90 fps, RTX 4080+ for 144 fps. Render 2016x2240 per eye. Use OpenXR.
- Variable-rate shading: 20-30% savings. Forward or deferred acceptable. 200k-500k triangles, 50-100 lights.

*PlayStation VR2 (90 fps):*
- GPU-bound heavily (10 TFLOPS optimized for 4K, not stereo). Budget: 6-7ms GPU, 3-4ms CPU.
- Render 2064x2208 per eye. Eye-tracking foveation: render 20% high-res fovea, 80% low-res. 40-50% performance gain.

## Common Pitfalls

**1. Frame Drops (Latency Spikes)**
- *Symptom*: Occasional stutters, motion sickness after 5-10 min, world lag.
- *Cause*: GPU frame >11.1ms, drops to 45 fps, 25ms latency spike.
- *Solution*: Profile every frame. If >2% frames exceed deadline, reduce physics rate (30Hz), cut LOD, lower draw calls. Dynamic resolution scaling hard-cap at 10ms.

**2. Stereo Misalignment (IPD Wrong)**
- *Symptom*: Eye strain, headaches after 30 min, vertical ghosting, divergence discomfort.
- *Cause*: Incorrect IPD. Renders eyes too far apart or too close.
- *Solution*: Detect/store IPD. Quest 3 provides it. Verify: look at vertical edge, should align perfectly, no double image.

**3. Jitter / Ghosting on Fast Motion**
- *Symptom*: Image trails behind head motion, world oscillates during head rotation.
- *Cause*: Pose prediction wrong. Or TimeWarp misalignment (pose not updated correctly).
- *Solution*: Verify pose prediction <2mm error. Check TimeWarp callback. Disable temporarily to isolate cause.

**4. Extreme Motion Sickness**
- *Symptom*: User nauseous within 5 min, dizziness persists 30 min post-VR.
- *Cause*: Latency >25ms, frame drops, jerky camera motion, missing comfort settings.
- *Solution*: Fix frame times. Check camera smoothing (no sudden accel). Add vignette. Offer snap-turn, comfort modes default-on.

## Tools & Workflow

**VR Profiling:**
- Xcode/Android Studio GPU Profiler (Quest): GPU time per shader, total <9ms target.
- Frame Debugger (Unity): Inspect draw calls, verify instanced stereo (VP indices 0, 1 in same draw).
- OVRMetricsDisplay: In-game overlay showing 90 fps locked, latency <20ms.

**Debug Visualization:**
```csharp
public class VRDebugDisplay : MonoBehaviour
{
    void Update()
    {
        float fps = 1.0f / Time.deltaTime;
        float gpuTime = GetGPUTime();
        Debug.Log($"FPS: {fps:F1}, GPU: {gpuTime:F1}ms, Latency: {gpuTime + 12:F1}ms");
    }
}
```

## Related Topics

- [27.1 Ray Tracing](27-01-Ray-Tracing.md) — Ray tracing limited on VR (budget constraints)
- [27.3 GPU-Driven Rendering](27-03-GPU-Driven-Rendering.md) — Reducing draw calls for VR CPU budgets
- [23 Multithreading & Jobs System](../chapters/23-Multithreading-And-Jobs-System.md) — Efficient threading for VR latency
- [05 Performance Optimization](../chapters/05-Performance-Optimization.md) — Frame budgeting fundamentals

---

[← Previous: 27.1 Ray Tracing](27-01-Ray-Tracing.md) | [Next: 27.3 GPU-Driven Rendering →](27-03-GPU-Driven-Rendering.md)
