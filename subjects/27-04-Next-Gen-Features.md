# 27.4 Next-Gen Features

[← Back to Chapter 27](../chapters/27-Advanced-Topics.md) | [Main Index](../README.md)

Next-generation graphics features leverage specialized hardware from PS5, Xbox Series X, and modern PC GPUs. Features include hardware-accelerated ray tracing, SSD streaming (PS5's proprietary tech), mesh shaders (NVIDIA, reducing draw-call overhead), sampler feedback streaming (variable-resolution texture streaming), and hardware-level acceleration structures. This section covers next-gen rendering patterns, hardware capabilities per platform, and practical implementation strategies.

---

## Overview

Next-gen hardware (PS5 2020, Xbox Series X 2020, RTX 30 series 2020+) introduced specialized features beyond traditional rasterization: dedicated ray-tracing cores (RT cores), ultra-fast storage (PS5 SSD 5.5 GB/s custom), mesh shaders reducing CPU-GPU sync points, and variable-rate shading. These features enable new rendering paradigms: Unreal Engine 5's Nanite (GPU-driven, automatic LOD), DirectStorage API (Windows, ultra-fast I/O for streaming), and console-exclusive optimizations (PS5 custom SSD architecture, Xbox Smart Delivery).

Key next-gen benefits: ray tracing becomes standard (not premium feature), geometry LOD handled automatically (Nanite), storage streaming enables massive worlds (Starfield), and variable-rate shading scales quality per hardware tier. Trade-off: features not universally supported (mesh shaders NVIDIA/AMD only, DirectStorage Windows only). Game engines (UE5, modern Unity) abstract differences, but graphics programmers must understand capabilities per platform.

## Key Concepts

- **Mesh Shaders**: NVIDIA GeForce (Ampere+) and AMD RDNA2+. Replace vertex/geometry/tessellation shaders with single mesh shader stage. Output primitives directly (no intermediate stages). Cost: eliminates VS->GS->PS overhead (10-20% GPU time savings). Typical: 32-256 threads per group, outputs 64-256 triangles. Enables complex geometry patterns (procedural LOD, skeletal deformation) without vertex buffer indirection.

- **Sampler Feedback Streaming (SFS)**: NVIDIA RTX, AMD RDNA. GPU reports which texels were sampled each frame. Driver streams high-resolution texels only (coarse mip levels until needed). 50-80% texture VRAM savings vs streaming all mips. VRAM budget: 256MB traditional, 512-1024MB with SFS (streams on-demand).

- **Hardware Acceleration Structures**: PS5/Xbox Series X/RTX: dedicated BVH acceleration for ray tracing. Hardware builds structures orders-of-magnitude faster than compute shaders. Enables dynamic ray tracing (rebuild per frame for moving geometry).

- **Variable-Rate Shading (VRS)**: Render different regions at different quality. Center at 1x, edges at 0.5x (4x fewer pixels shaded). Typical: 40-50% fillrate savings. Imperceptible loss (human peripheral vision is low-acuity). Tile-based VRS (coarse tiles, e.g., 16x16 pixels) or per-sample VRS (fine, pixel-level). NVIDIA/AMD supported; mobile limited.

- **DirectStorage API (Windows)**: GPU decompression of textures/meshes during load. CPU doesn't decompress; GPU does directly from GPU memory. Ultra-fast: decompresses megabytes/ms. PC equivalent of PS5 SSD benefits. Standard for modern PC games (Microsoft Flight Simulator, Starfield).

- **Mesh Shaders vs Traditional Pipeline**: Traditional: CPU->VB/IB->VS->GS->Rasterizer->PS. Mesh: CPU->Task Shader (optional)-> Mesh Shader (outputs primitives) -> Rasterizer->PS. Benefits: no vertex expansion (VS->GS output growth), better cache coherence (triangles output locally), procedural geometry (output different triangles per invocation).

## Best Practices

**Mesh Shaders (NVIDIA RTX/AMD RDNA):**
- Output 64-256 triangles per group (maximize throughput). Smaller groups = underutilized, larger = occupancy drops.
- Shared memory: use LDS to store intermediate vertex data, reduce global memory pressure.
- Procedural LOD: compute mesh quality per group (distance to camera). Output coarse LOD for distant, fine LOD for near.
- Example:
  ```hlsl
  [numthreads(128, 1, 1)]  // 128 threads per group
  void MeshShader(uint groupIdx : SV_GroupIndex, uint3 groupId : SV_GroupID)
  {
      uint meshletIdx = groupId.x;
      Meshlet meshlet = g_Meshlets[meshletIdx];
      
      // Compute distance LOD
      float3 meshletCenter = meshlet.center;
      float distToCamera = distance(meshletCenter, g_CameraPos);
      int triangleCount = distToCamera < 50 ? meshlet.triangleCount : max(meshlet.triangleCount/4, 4);
      
      SetMeshOutputCounts(meshlet.vertexCount, triangleCount);
      
      // Output vertices and triangles (per-thread)
      if (groupIdx < meshlet.vertexCount)
      {
          uint vertexIdx = meshlet.vertexStart + groupIdx;
          gs_Vertices[groupIdx] = g_Vertices[vertexIdx];
      }
      
      if (groupIdx < triangleCount)
      {
          uint triIdx = meshlet.triangleStart + groupIdx;
          gs_Triangles[groupIdx] = g_Indices[triIdx];
      }
  }
  ```

**Sampler Feedback Streaming (RTX/RDNA):**
- Enable feedback reporting in driver. GPU reports sampled texel coordinates.
- Streaming system: analyze feedback each frame, stream texels accessed next frame at high-res.
- VRAM budget: coarse LOD (1-2 mips always resident), feedback determines which finer mips to stream.
- Cost: 1-2% GPU for feedback tracking, 2-3ms CPU for streaming logic. VRAM savings 50-80%.
- Practical: 256MB resident (coarse mips), stream 256MB high-res texels on-demand. Total logical 512MB+ textures.

**Variable-Rate Shading (VRS):**
- Tile-based VRS (16x16 pixel tiles, 4 rate options: 1x, 1x2, 2x1, 2x2). Specify per-tile via attachment.
  ```hlsl
  // Write VRS image per tile
  float4 sampleVariableRateShadingImage()
  {
      uint2 tileCoord = (uint2)input.pixelPos / 16;
      uint2 screenCenter = (uint2)(g_ScreenSize / 2);
      float dist = distance(tileCoord, screenCenter / 16);
      
      // Foveate: center 1x, edges 2x2
      return dist < 10 ? 0 : float4(0, 0, 0, 2);  // 2 = 2x2 rate
  }
  ```
- Typical setup: center 50% of screen 1x, outer 50% at 2x (4x fewer pixels shaded). 25-40% fillrate savings.

**DirectStorage (Windows):**
- Queue GPU operations instead of CPU reads. Texture/mesh compressed on disk, GPU decompresses directly.
- API: CreateFile -> DirectStorageQueue -> EnqueueRequest(file, GPU buffer) -> Process(). GPU handles decompression asynchronously.
- Cost: <1ms to queue 10MB load (vs 50-100ms CPU decompression).
- Typical: game loads 1GB mesh/texture data per level in <100ms (DirectStorage + GPU decompression vs 500ms+ traditional).

## Platform-Specific Guidance

**PC (DX12, RTX 30+):**
- Mesh shaders: fully supported (Ampere+). Use for complex geometry (procedural LOD, skeletal deformation).
- SFS: supported on RTX 20+. Enables massive texture budgets (512MB+ with SFS vs 256MB traditional).
- VRS: supported. Tile-based or per-sample (fine VRS). Implement foveated rendering for VR.
- DirectStorage: Windows 11+. Ultra-fast I/O for streaming. Standard for modern AAA.

**PlayStation 5:**
- Custom SSD: 5.5 GB/s raw bandwidth (100x faster than PS4 HDD). Enables streaming assets that were previously impossible (massive worlds, asset instancing).
- Ray tracing: dedicated ray engines, 5.5 TFLOPS ray compute. Ray-traced shadows/reflections standard.
- Mesh shaders not standard (RDNA-based, but PS5 doesn't expose). Use compute workarounds or traditional pipeline.
- Custom audio: 3D Tempest audio (3D positional audio hardware). Music scheduling critical for CPU.

**Xbox Series X:**
- Similar SSD speed to PS5 (2.36 GB/s common, 4.8 GB/s optimal). DirectStorage support (native).
- Ray tracing: 6 TFLOPS ray compute (higher than PS5). Efficient RDNA architecture.
- Smart Delivery: deliver game versions per hardware tier (Series S vs Series X). Same game, different LOD/resolution automatically.

**Mobile (None):**
- Mesh shaders: not supported. Use traditional VS+GS workarounds or skip.
- SFS: not supported. Fixed texture streaming (traditional mipmap approach).
- VRS: limited support (some Snapdragon 8 Gen 2 variants). Standard mobile = no VRS.

## Common Pitfalls

**1. Mesh Shader Occupancy Too Low**
- *Symptom*: Mesh shader code takes 30-40ms (should be <5ms), GPU utilization low (<60%).
- *Cause*: Group size too small (16 threads with 256 thread limit per group = 1 active group, massive waste). Or not enough meshlets to dispatch.
- *Solution*: Increase group size to 64-128 threads. Dispatch more meshlets (subdivide large meshes). Aim for 10+ active groups per SM.

**2. SFS Thrashing (Excessive Streaming)**
- *Symptom*: Frame time spikes periodically (every 100ms), disk I/O maxed despite game not loading.
- *Cause*: Camera motion causes excessive texture feedback (streaming too many high-res texels). SFS system can't keep up.
- *Solution*: Limit high-res texel budget per frame (stream only 50MB texels/frame max). Use coarser feedback tiers. Prioritize by screen coverage.

**3. VRS Artifacts (Banding/Aliasing)**
- *Symptom*: Visible banding patterns in shaded regions, especially on surfaces with fine gradients (specular highlights).
- *Cause*: VRS tile boundaries too coarse (16x16 pixels = 8 tiles across 128 pixel screen). Low-rate edges visible.
- *Solution*: Use per-sample VRS (finer granularity), not tile-based. Or blend VRS rate softly at boundaries (scale shading quality vs hard transitions).

**4. DirectStorage Decompression CPU Impact**
- *Symptom*: GPU decompression causes unexpected CPU spikes, thread stalls.
- *Cause*: Not fully GPU-driven. CPU submitting commands while GPU decompresses, causing sync points. Or using CPU decompression fallback.
- *Solution*: Queue all DirectStorage operations asynchronously. Don't read results until next frame. Use 100% GPU decompression path (no CPU fallback).

## Tools & Workflow

**Mesh Shader Debugging:**
- NVIDIA Nsight: inspect mesh shader occupancy per SM, thread group utilization, primitive output rates.
- PIX (Xbox): mesh shader timeline, output verification (correct triangle counts).
- Debug output: log meshlet dispatch count, triangles output per group. Should be 64-256 triangles per group.

**SFS Monitoring:**
- GPU memory profiler: track SFS residency, streaming bandwidth. Typical: 50-100MB/s streaming, 1-2 frame latency for feedback->stream.
- Feedback visualization: overlay showing which texels were sampled (hot vs cold). Helps identify streaming bottlenecks.

**DirectStorage Profiling:**
- Windows 11 Storage QoS: monitor I/O bandwidth. DirectStorage should saturate link (PCIe Gen 4: 7 GB/s max).
- GPU decompression timers: measure GPU decompression overhead (should be <5% GPU time for typical loads).

## Related Topics

- [27.3 GPU-Driven Rendering](27-03-GPU-Driven-Rendering.md) — Mesh shaders reduce draw-call overhead, complement GPU culling
- [27.1 Ray Tracing](27-01-Ray-Tracing.md) — Hardware ray tracing on next-gen platforms
- [07 Asset Optimization](../chapters/07-Asset-Optimization.md) — Compression techniques for DirectStorage
- [10 Asset Bundling & Streaming](../chapters/10-Asset-Bundling-And-Streaming.md) — DirectStorage and next-gen streaming patterns

---

[← Previous: 27.3 GPU-Driven Rendering](27-03-GPU-Driven-Rendering.md) | [Next: 27.5 Nanite-Like Techniques →](27-05-Nanite-Like-Techniques.md)
