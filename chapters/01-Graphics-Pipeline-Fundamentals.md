# Chapter 1: Graphics Pipeline Fundamentals

[← Back to Main Index](../README.md)

Understanding the complete rendering pipeline from CPU to GPU is essential for graphics programming. This chapter covers the entire journey of rendering data through the graphics pipeline.

---

## Subjects in This Chapter

### [1.1 Rendering Pipeline Overview](../subjects/01-01-Rendering-Pipeline-Overview.md)
The high-level structure of the graphics pipeline, including CPU-GPU communication, command buffer architecture, frame timing and synchronization, and double/triple buffering strategies.

### [1.2 Vertex Processing](../subjects/01-02-Vertex-Processing.md)
How vertices are processed by the GPU, including vertex shaders, vertex formats and layouts, compression techniques, and skinning/animation processing.

### [1.3 Primitive Assembly and Rasterization](../subjects/01-03-Primitive-Assembly-Rasterization.md)
The conversion of vertices into primitives and pixels, covering triangle setup, clipping and culling, viewport transformation, and conservative rasterization.

### [1.4 Fragment Processing](../subjects/01-04-Fragment-Processing.md)
Per-pixel operations including fragment shaders, early-Z optimization, Z-prepass techniques, pixel quad execution, and helper invocations.

### [1.5 Output Merger](../subjects/01-05-Output-Merger.md)
The final stage of the pipeline covering depth testing, stencil operations, blending modes, and multi-render target (MRT) configurations.

### [1.6 Compute Pipeline](../subjects/01-06-Compute-Pipeline.md)
GPU compute capabilities including compute shaders, thread groups and dispatching, shared memory usage, and common GPU compute patterns.

---

[← Previous: Main Index](../README.md) | [Next: Chapter 2 - GPU Hardware Architecture →](02-GPU-Hardware-Architecture.md)
