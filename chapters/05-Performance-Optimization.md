# Chapter 5: Performance Optimization

[← Back to Main Index](../README.md)

Performance optimization is at the heart of graphics programming. This chapter covers techniques for optimizing every aspect of rendering performance.

---

## Subjects in This Chapter

### [5.1 CPU-Side Optimization](../subjects/05-01-CPU-Side-Optimization.md)
Reducing CPU overhead through draw call reduction, batching strategies (static/dynamic), GPU instancing, SRP batcher, culling techniques, LOD systems, and multithreading.

### [5.2 GPU-Side Optimization](../subjects/05-02-GPU-Side-Optimization.md)
Optimizing shader performance including complexity reduction, overdraw minimization, texture sampling optimization, ALU vs texture balance, register pressure, and branch reduction.

### [5.3 Memory Optimization](../subjects/05-03-Memory-Optimization.md)
Managing memory efficiently including texture memory, mesh optimization, shader variant management, asset loading strategies, memory pools, and garbage collection mitigation.

### [5.4 Bandwidth Optimization](../subjects/05-04-Bandwidth-Optimization.md)
Reducing memory bandwidth usage through texture compression, vertex data compression, render target resolution scaling, mipmapping, and Z-buffer compression.

### [5.5 Draw Call Pipeline Optimization](../subjects/05-05-Draw-Call-Pipeline-Optimization.md)
Understanding and optimizing the draw call pipeline including SetPassCall analysis, material batching rules, Z-sorting strategies, render queue management, and transparent vs opaque ordering.

---

[← Previous: Chapter 4 - Profiling and Analysis Tools](04-Profiling-And-Analysis-Tools.md) | [Next: Chapter 6 - Bottleneck Identification →](06-Bottleneck-Identification.md)
