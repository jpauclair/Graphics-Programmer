# 27.3 GPU-Driven Rendering

[← Back to Chapter 27](../chapters/27-Advanced-Topics.md) | [Main Index](../README.md)

GPU-driven rendering shifts visibility determination (frustum culling, occlusion testing) and draw-call generation from CPU to GPU via compute shaders and indirect buffers. Instead of CPU collecting visible objects (expensive, serial), GPU processes all objects in parallel, culls invisibles, generates indirect draw parameters directly into GPU buffers. This eliminates CPU-GPU sync, enables rendering 1M objects with 50-100 draw calls (vs traditional 10k objects, 500+ draws), and scales to massive worlds. This section covers culling algorithms, indirect drawing, multi-draw indirection, and architectural patterns.

---

## Overview

Traditional rendering: CPU frustum-culls objects each frame (100-500µs per 10k objects), builds drawcall list, submits to GPU (synchronous, blocking). GPU-driven: compute shader processes all objects in parallel (10k objects in <100µs on GPU). Scales linearly with object count: CPU O(n) serial, GPU O(n/1024) parallel (1024 = compute group size).

Historically, indirect drawing required GPU->CPU readback (slow, 1-2 frame latency) or predefined counts. Modern indirection: compute shader iterates objects, writes visible indices to append buffer, counts instances. GPU then issues variable draw count via ExecuteIndirect. Result: perfect culling efficiency, zero CPU overhead (one ExecuteIndirect call per frame).

Real-world impact: Unreal Engine 5's Nanite renders 10M micro-polygons at 60 fps via GPU culling. Demon's Souls remake (PS5) renders 100k+ objects simultaneously. Starfield uses GPU culling for massive open worlds without CPU bottleneck.

## Key Concepts

- **Frustum Culling (GPU)**: Compute shader tests each object's bounding box against 6 frustum planes. If outside any plane, cull. Cost: 1 compute invocation per object (parallel). 10k objects: <1ms on modern GPU (vs 100-500µs CPU serial). GPU wins at 10k+ objects.

- **Occlusion Culling (GPU)**: Test object AABB against Hi-Z buffer (hierarchical depth pyramid). If behind previous depth, occluded. Cost: 5-10 clocks per test. 10M objects: 50-100ms compute. Trade-off: Hi-Z may have false negatives/positives vs perfect occlusion, but extreme scale.

- **Append Buffer**: GPU buffer with atomic counter. Compute shader appends visible object indices via AppendStructuredBuffer.Append(). Eliminates CPU knowing visible count. Typical: 100-1000 visible from 10M tests (0.01-0.1% visibility).

- **Indirect Draw Buffer**: GPU buffer with draw parameters (vertex count, instance count, offsets, 16 bytes per draw). ExecuteIndirect reads parameters, issues draw call. GPU writes directly, CPU submits one ExecuteIndirectCountArguments (variable count).

- **Persistent Mapping**: Map GPU buffer to CPU address space, read visibility results without stall. GPU writes frame N, CPU reads frame N-1 (double-buffer). Zero-latency readback.

- **Draw Call Reduction**: Traditional 10k objects = 10k draws = 300-500ms CPU overhead (prohibitive). GPU-driven: 100 draws (per material type) regardless of object count = 3-5ms CPU overhead.

- **Instance Data Indirection**: Object data (matrix, material) not in vertex buffer. GPU reads object index, looks up data from buffer. Enables millions of objects with single VB + IB.

## Best Practices

**Compute-Based Culling:**
- Structure data cache-efficiently: bounding spheres in tightly-packed array (16B each). Compute iterates sequentially, high cache coherence.
- Launch 64-256 objects per compute group (maximize occupancy). Group size 256 = high occupancy.
- Fast frustum test: sphere-plane distance test (6 dot products = ~30 clocks per object). Sphere-frustum test formula:
  ```hlsl
  float dist = dot(plane.xyz, boundsCenter) + plane.w;
  if (dist < -boundsExtents.x) outside = true;
  ```
- Code example:
  ```hlsl
  [numthreads(256, 1, 1)]
  void FrustumCull(uint3 id : SV_DispatchThreadID)
  {
      uint objIdx = id.x;
      if (objIdx >= g_ObjectCount) return;
      
      float3 boundsCenter = g_ObjectBounds[objIdx].center;
      float3 boundsExtents = g_ObjectBounds[objIdx].extents;
      
      bool visible = true;
      for (int i = 0; i < 6; i++)
      {
          float4 plane = g_FrustumPlanes[i];
          float dist = dot(plane.xyz, boundsCenter) + plane.w;
          if (dist < -boundsExtents.x)
          {
              visible = false;
              break;
          }
      }
      
      if (visible)
          g_VisibleIndices.Append(objIdx);
  }
  ```

**Indirect Draw Setup:**
- Create indirect args buffer: [vertexCount, instanceCount, startVertex, startInstance] per draw type (16B each). GPU writes, CPU reads via ExecuteIndirectCountArguments.
- Compute writes: one entry per material type. Example structure:
  ```hlsl
  struct IndirectArgs {
      uint vertexCount;
      uint instanceCount;
      uint startVertexLocation;
      uint startInstanceLocation;
  };
  ```

**Occlusion with Hi-Z:**
- Build Hi-Z pyramid after depth prepass: mip 0 = full resolution, mip N = max(2x2 quad of mip N-1). Cost: 33% of mip 0 bandwidth total.
- Mip selection: object screen size determines mip level. Formula: `mipLevel = ceil(log2(max(pixelWidth, pixelHeight)))`.
- Conservative test: max depth in mip > object near-plane = potentially visible. Safer for logic.
- Performance: 1-2 clocks per test once cached. 10M objects, 1-2 samples = 10-20ms compute.

**Instance Data Indirection:**
- Single large vertex buffer + instance data buffer. Avoid per-object vertex buffers.
  ```hlsl
  StructuredBuffer<uint> g_ObjectIndices;
  StructuredBuffer<float4x4> g_ObjectMatrices;
  StructuredBuffer<MaterialData> g_Materials;
  
  PS_Input VS_Main(uint vertexID : SV_VertexID, uint instanceID : SV_InstanceID)
  {
      uint objIdx = g_ObjectIndices[vertexID];
      float4x4 worldMat = g_ObjectMatrices[objIdx];
      MaterialData mat = g_Materials[objIdx];
      // ... rest of VS
  }
  ```
- Typical: 10k objects x 64B per object = 640KB (fits L2 cache).

**Persistent Mapping:**
```csharp
private AsyncGPUReadbackRequest m_ReadbackRequest;
private ComputeBuffer m_IndirectArgsBuffer;

void ReadbackResults()
{
    if (m_ReadbackRequest.hasData)
    {
        var data = m_ReadbackRequest.GetData<uint>();
        int visibleCount = (int)data[0];
        ProcessVisibleObjects(visibleCount);
    }
    m_ReadbackRequest = AsyncGPUReadback.Request(m_IndirectArgsBuffer);
}
```

## Platform-Specific Guidance

**PC (DX12/Vulkan):**
- ExecuteIndirectCountArguments (DX12): GPU controls draw count, no CPU stall.
- Compute groups: 1024 threads for RTX/RDNA optimal occupancy.
- Persistent mapping via async readback (frame-latency acceptable).

**Console (PS5/Xbox Series X):**
- PS5 Gnmx: hardware command processor generates commands from GPU. Indirect dispatch fully supported.
- Xbox Series X RDNA: similar, generates indirect draws at GPU speed (1-2ms for 1M objects).
- Both enable Nanite-like tech. Culling efficiency critical on fixed architecture.

**Mobile (Android/iOS):**
- Compute available (Snapdragon 8 Gen 2 RDNA). Atomic operations slower (3-5x vs desktop).
- Frustum culling practical (<1ms). Occlusion culling cost-prohibitive.
- Indirect draws supported (EXT_multi_draw_indirect), but API overhead higher. Use GPU culling to reduce 100 draws -> 10 (acceptable).

## Common Pitfalls

**1. Atomic Counter Contention**
- *Symptom*: Compute shader takes 20-50ms (should be <2ms), low occupancy (<50%).
- *Cause*: 256+ threads writing single atomic counter. Hardware serializes, stalls.
- *Solution*: Use local sharing. Each group accumulates in LDS/shared memory, one atomic per group. 256x speedup.

**2. GPU->CPU Readback Stall**
- *Symptom*: Frame spikes to 33ms every few frames, GPU stalls despite <5ms work.
- *Cause*: Synchronous readback of indirect args. GPU must complete, transfer (PCIe latency).
- *Solution*: Async readback with frame-latency buffering. Read frame N-1 while GPU does frame N. No stall.

**3. Over-Culling (False Negatives)**
- *Symptom*: Objects pop in/out during camera motion, especially screen edges.
- *Cause*: Sphere-plane test uses object radius, but geometry extends beyond (skeletal mesh). Gets culled wrongly.
- *Solution*: Use conservative radius (1.2x). Or axis-aligned box test (7 planes, slower, more accurate). Debug: render culled in red, visible in green.

**4. Indirect Args Corruption**
- *Symptom*: Rendering corrupts memory, crashes unpredictably, garbage geometry.
- *Cause*: Indirect args buffer overflow. Compute writes past bounds. Indices reference invalid ranges.
- *Solution*: Validate sizes. Assert objectCount < bufferSize. PIX/Nsight: inspect args GPU wrote, check sane values.

## Tools & Workflow

**Pipeline Setup (C#):**
```csharp
public class GPUDrivenRenderer : MonoBehaviour
{
    private ComputeShader m_FrustumCullShader;
    private ComputeBuffer m_ObjectBoundsBuffer, m_IndirectArgsBuffer, m_VisibleIndicesBuffer;
    private int m_FrustumCullKernel, m_ObjectCount;
    
    void Initialize()
    {
        m_ObjectCount = 100000;
        m_ObjectBoundsBuffer = new ComputeBuffer(m_ObjectCount, sizeof(float) * 8);
        m_IndirectArgsBuffer = new ComputeBuffer(100, sizeof(uint) * 4);
        m_VisibleIndicesBuffer = new ComputeBuffer(m_ObjectCount, sizeof(uint));
        m_FrustumCullKernel = m_FrustumCullShader.FindKernel("FrustumCull");
    }
    
    void CullAndRender(CommandBuffer cmd, Camera camera)
    {
        var planes = GeometryUtility.CalculateFrustumPlanes(camera);
        Vector4[] planeMat = new Vector4[6];
        for (int i = 0; i < 6; i++)
            planeMat[i] = new Vector4(planes[i].normal.x, planes[i].normal.y, planes[i].normal.z, planes[i].distance);
        
        m_FrustumCullShader.SetVectorArray("g_FrustumPlanes", planeMat);
        m_FrustumCullShader.SetBuffer(m_FrustumCullKernel, "g_ObjectBounds", m_ObjectBoundsBuffer);
        m_FrustumCullShader.SetBuffer(m_FrustumCullKernel, "g_VisibleIndices", m_VisibleIndicesBuffer);
        m_FrustumCullShader.SetInt("g_ObjectCount", m_ObjectCount);
        
        uint threadGroupsX = (uint)(m_ObjectCount + 255) / 256;
        cmd.DispatchCompute(m_FrustumCullShader, m_FrustumCullKernel, (int)threadGroupsX, 1, 1);
        cmd.ExecuteIndirectCountArguments(m_IndirectArgsBuffer, 0, m_IndirectArgsBuffer, 4);
    }
}
```

## Related Topics

- [27.1 Ray Tracing](27-01-Ray-Tracing.md) — Culling/traversal similar to ray-tracing BVH
- [05 Performance Optimization](../chapters/05-Performance-Optimization.md) — GPU culling as extreme optimization
- [06 Bottleneck Identification](../chapters/06-Bottleneck-Identification.md) — Profiling GPU-driven rendering
- [13 Rendering Techniques](../chapters/13-Rendering-Techniques.md) — Indirect rendering patterns

---

[← Previous: 27.2 Virtual Reality](27-02-Virtual-Reality.md) | [Next: 27.4 Next-Gen Features →](27-04-Next-Gen-Features.md)
