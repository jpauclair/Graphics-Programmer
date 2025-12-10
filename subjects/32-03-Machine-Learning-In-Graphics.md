# 32.3 Machine Learning in Graphics

[← Back to Chapter 32](../chapters/32-Future-Proofing.md) | [Main Index](../README.md)

Machine learning in graphics = AI techniques for rendering (DLSS = neural upscaling, denoising = ML-based temporal filtering, neural rendering = NeRFs/Gaussian Splatting). Benefits: performance (render at lower res, upscale to 4K = 2-4x faster), quality (ML denoising = fewer ray-tracing samples), new techniques (procedural content generation, real-time GI). Hardware: Tensor Cores (NVIDIA RTX = ML inference acceleration), Neural Engine (Apple A-series), AI accelerators (future GPUs = dedicated ML hardware).

---

## Overview

AI Upscaling: DLSS (NVIDIA Deep Learning Super Sampling), FSR 2.0/3.0 (AMD FidelityFX Super Resolution), XeSS (Intel Xe Super Sampling). Technique: render at lower resolution (e.g., 1080p), neural network upscales to native (4K = perceptually similar, 2-4x performance boost). DLSS = best quality (Tensor Cores required), FSR 2.0 = open-source (no special hardware = AMD/NVIDIA/Intel), XeSS = Intel GPUs optimized. Unity: DLSS plugin (HDRP/URP), FSR 2.0 integration (URP 14+).

Neural Rendering: NeRFs (Neural Radiance Fields = volumetric scene representation), Gaussian Splatting (point cloud rendering = real-time NeRF alternative), neural textures (compress 8K textures = ML decompress runtime). Use cases: photogrammetry (real-world capture → neural scene), virtual production (real-time rendering of captured environments), compression (neural codec = better quality per byte). Performance: inference expensive (NeRF = offline, Gaussian Splatting = 30-60 FPS), hardware acceleration required (Tensor Cores = 10x faster).

## Key Concepts

- **DLSS (Deep Learning Super Sampling)**: NVIDIA's AI upscaling (RTX 2000+ GPUs, Tensor Cores). Versions: DLSS 2 (temporal upscaling, motion vectors + jitter), DLSS 3 (frame generation = interpolate frames, RTX 4000+). Quality modes: Ultra Performance (3×3 = 1080p→4K), Quality (2×2 = 1440p→4K). Unity: DLSS plugin (HDRP 12+, URP 14+), enable in camera component. Performance: 2-4x FPS boost (render 1080p internal, output 4K = faster than native 4K). Quality: comparable to native (sometimes better = ML anti-aliasing).
- **FSR 2.0 / 3.0 (FidelityFX Super Resolution)**: AMD's open-source upscaling (works on all GPUs = NVIDIA/AMD/Intel). FSR 2.0: temporal upscaling (similar to DLSS, no ML = hand-tuned algorithm), FSR 3.0: frame generation (like DLSS 3). Unity: FSR 2.0 plugin (URP 14+), quality modes (Quality, Balanced, Performance). Performance: 1.5-3x FPS (slightly less than DLSS, but no hardware requirement). Advantage: cross-platform (consoles, older GPUs).
- **XeSS (Xe Super Sampling)**: Intel's upscaling (Arc GPUs optimized, fallback on other GPUs). Technique: ML-based (DP4a instructions = general compute), or XMX acceleration (Intel Arc Tensor units). Unity: XeSS plugin (HDRP/URP). Performance: between FSR 2.0 and DLSS (XMX = close to DLSS, DP4a = similar to FSR). Advantage: Intel GPU optimization (Arc A770 = best XeSS performance).
- **ML Denoising**: reduce ray-tracing noise (fewer samples needed = faster). Techniques: OptiX Denoiser (NVIDIA, AI-based = 1 sample/pixel sufficient), SVGF (spatiotemporal variance-guided filtering = hand-tuned), ReLAX (Intel, open-source). Unity: HDRP ray-tracing (built-in denoiser = temporal accumulation + filter). Performance: 5-10x fewer RT samples (1 vs 8 samples/pixel = 8x faster). Quality: ML denoiser preserves detail (SVGF blurs high-frequency = softer).
- **Neural Radiance Fields (NeRFs)**: volumetric scene representation (neural network = RGB + density per 3D point). Training: capture photos from multiple angles (100+ images), train network (hours on GPU), inference: ray-march through volume (query network per sample). Use cases: photogrammetry (real environment → NeRF), virtual production (film sets), view synthesis (novel camera angles). Performance: slow (inference = many network queries per pixel), offline or baked (lightfield textures). Real-time alternative: Gaussian Splatting (point cloud + anisotropic Gaussians = 30+ FPS).
- **Procedural Content with ML**: generate assets (textures, meshes, animations) using AI. Examples: neural texture synthesis (style transfer = generate tileable textures), mesh generation (text → 3D model), motion synthesis (ML-driven animation). Unity: integration via plugins (Runway ML, Leonardo AI = API calls), or local inference (ONNX Runtime = TensorFlow models in Unity). Use case: rapid prototyping (text prompt → asset), content variation (one model → many variants).

## Best Practices

**Upscaling Selection**:
- DLSS: NVIDIA RTX GPUs (best quality, Tensor Cores = fastest).
- FSR 2.0: cross-platform (all GPUs, consoles), or AMD GPUs (no Tensor Cores).
- XeSS: Intel Arc GPUs (XMX acceleration), or fallback (DP4a = slower).
- Native: ultra-high-end GPU (RTX 4090 = 4K native possible), or competitive (no upscaling artifacts).

**Ray-Tracing + ML Denoising**:
- Sample count: 1 sample/pixel with denoiser (vs 4-8 without = 4-8x faster).
- Temporal stability: denoiser requires motion vectors (jitter + reprojection = smooth result).
- Fallback: screen-space techniques (SSR, SSAO = no RT, fast).

**Neural Rendering Feasibility**:
- NeRFs: offline rendering (too slow for real-time), or baked lightfields (precompute views).
- Gaussian Splatting: real-time (30-60 FPS on RTX 3080), good for static scenes (photogrammetry).
- Use case: virtual production (real environment capture = realistic backgrounds), not gameplay (too slow for dynamic).

**Platform-Specific**:
- **PC**: DLSS/FSR/XeSS all supported (player chooses based on GPU).
- **Console**: FSR 2.0 (PS5/Xbox = no Tensor Cores), or native rendering (some games skip upscaling).
- **Mobile**: no ML upscaling yet (future: Snapdragon/Apple Neural Engine = mobile DLSS).

## Common Pitfalls

**DLSS Ghosting**: camera moves fast (motion vectors incorrect = temporal reprojection fails). Symptom: trailing artifacts (previous frames blended incorrectly). Solution: reset DLSS history on teleport (discontinuous motion = break temporal), or increase jitter (reduce temporal reliance).

**FSR 2.0 Over-Sharpening**: developer uses Performance mode (3×3 upscale = aggressive sharpening). Symptom: ringing artifacts (edges over-enhanced = halos). Solution: Quality mode (2×2 = less aggressive), or adjust sharpening slider (reduce overshoot).

**NeRF Real-Time Assumption**: developer tries to render NeRF at 60 FPS (inference = 500ms per frame). Symptom: 2 FPS (network queries too slow). Solution: use Gaussian Splatting (real-time alternative = rasterize Gaussians instead of network queries), or bake NeRF to lightfield (precompute views = lookup texture).

## Tools & Workflow

**DLSS Integration** (Unity HDRP):
```csharp
1. Package Manager → NVIDIA DLSS (requires HDRP 12+)
2. Camera → Add Override → DLSS
   - Quality Mode: Quality (1440p→4K), Balanced, Performance (1080p→4K)
3. Build: Windows DX12, NVIDIA RTX GPU required
4. Runtime: DLSS.SetQualityMode(DLSSQuality.Quality);
```

**FSR 2.0 Setup** (Unity URP):
```csharp
1. Import FSR 2.0 plugin (AMD GitHub or Unity Asset Store)
2. Camera → Rendering → Upscaling Filter: FSR 2.0
   - Quality Mode: Quality / Balanced / Performance
3. Build: cross-platform (all GPUs supported)
```

**XeSS Integration**:
```csharp
1. Download XeSS plugin (Intel GitHub)
2. Camera → Add Component → XeSS Upscaler
   - Quality: Quality (best), Balanced, Performance, Ultra Performance
3. Build: DX12 or Vulkan, Intel Arc = XMX acceleration, others = DP4a fallback
```

**HDRP Ray-Tracing with Denoising**:
```csharp
1. HDRP Asset → Ray Tracing: Enabled
2. Volume → Ray-Traced Reflections
   - Sample Count: 1 (denoiser compensates)
   - Denoise: Temporal (ML-based or hand-tuned)
3. Result: 1 sample/pixel = 8x faster than 8 samples, comparable quality
```

**Gaussian Splatting (Third-Party)**:
```csharp
// Example: Import Gaussian Splatting dataset
1. Capture scene: photos from multiple angles (COLMAP or NeRF Studio)
2. Train Gaussian Splatting: 3DGS framework (30min on RTX 3080)
3. Export: .ply point cloud (positions, colors, covariances)
4. Unity: import plugin (render Gaussians as billboards = compute shader)
5. Performance: 30-60 FPS (millions of Gaussians, RTX 3060+)
```

**ONNX Runtime for ML Inference** (Unity):
```csharp
using Unity.Barracuda; // Or ONNX Runtime

// Load trained model
Model model = ModelLoader.Load(modelAsset);
IWorker worker = WorkerFactory.CreateWorker(WorkerFactory.Type.ComputePrecompiled, model);

// Run inference
Tensor input = new Tensor(1, height, width, 3); // Input texture
worker.Execute(input);
Tensor output = worker.PeekOutput(); // Upscaled result

// Apply to texture
RenderTexture result = output.ToRenderTexture();
```

**Neural Texture Synthesis** (API Call):
```csharp
// Example: Leonardo AI API (text → texture)
using UnityEngine.Networking;

IEnumerator GenerateTexture(string prompt)
{
    string url = "https://api.leonardo.ai/v1/generations";
    string json = $"{{\"prompt\":\"{prompt}\", \"num_images\":1}}";
    
    UnityWebRequest request = UnityWebRequest.Post(url, json);
    yield return request.SendWebRequest();
    
    Texture2D texture = /* parse response, download image */;
    material.mainTexture = texture;
}
```

## Related Topics

- [27.1 Ray-Tracing](27-01-Ray-Tracing.md) - RT denoising
- [15.3 Temporal Anti-Aliasing](13-04-Transparency-And-Alpha.md) - TAA + upscaling
- [32.1 Next Generation Consoles](32-01-Next-Generation-Consoles.md) - Console upscaling (FSR)
- [32.4 Cloud And Streaming Graphics](32-04-Cloud-And-Streaming-Graphics.md) - Remote ML inference

---

[← Previous: 32.2 Graphics API Evolution](32-02-Graphics-API-Evolution.md) | [Next: 32.4 Cloud And Streaming Graphics →](32-04-Cloud-And-Streaming-Graphics.md)
