# 7.3 Shader Optimization

[← Back to Chapter 7](../chapters/07-Asset-Optimization.md) | [Main Index](../README.md)

Shader optimization reduces GPU execution time, memory consumption, and variant count to improve frame rates and reduce build sizes.

---

## Overview

Shaders execute for every vertex (vertex shaders) and every pixel (fragment/pixel shaders). A 1080p frame with 1M triangles runs vertex shader 500K times (assuming 50% cache hits) and fragment shader 2M times (assuming 1 sample per pixel). Inefficient shaders multiply costs: A 100-cycle fragment shader on 2M pixels = 200M cycles per frame. Optimizing to 50 cycles = 100M cycles, doubling fragment performance.

Shader optimization targets three areas: reducing instruction count (mathematical simplifications, remove unused code), improving memory access patterns (texture sampling efficiency, coalesced reads), and minimizing shader variants (keyword management, feature stripping). Additionally, precision management (using half/float16 instead of float32) can double throughput on mobile and last-gen consoles.

Optimization workflow: profile with GPU profilers (PIX, Razor, Nsight) to identify expensive shaders, analyze shader assembly (count ALU/texture instructions), optimize hotspots (simplify math, reduce texture samples), and validate improvements. Modern tools like Shader Variant Collection help strip unused variants, reducing build sizes by 50-80% and preventing runtime compilation hitches.

## Key Concepts

- **Shader Instructions**: GPU operations (ALU = arithmetic, TEX = texture sampling, memory load/store). Each instruction costs cycles. Fragment shaders with 200+ instructions are expensive (target <100 for mobile, <150 for console).
- **Shader Variants**: Compiled permutations from keywords (multi_compile). 10 keywords = 1,024 variants (2¹⁰). Each variant costs memory (10KB-1MB). Thousands of variants = gigabytes of shader memory and slow compilation.
- **Precision**: Data type bit-depth (float32 = full, float16/half = half, fixed/min16float = minimum). Half precision doubles ALU throughput on mobile/Switch. Use aggressively for colors, UVs, non-critical calculations.
- **Register Pressure**: Shader register usage (temps, interpolators). High pressure reduces occupancy (GPU can't run multiple warps simultaneously), lowering performance. Target <32 registers per shader on console.
- **Dependent Texture Reads**: Texture sample whose UV depends on previous texture sample (e.g., normal map determines UV for detail map). Prevents texture fetch parallelism, 2-5x slower than independent samples.

## Best Practices

**Instruction Count Reduction:**
- Simplify math operations: Replace `pow(x, 2)` with `x * x` (10x faster). Replace `1.0 / x` with `rcp(x)` intrinsic (2x faster). Replace `normalize()` in fragment shader with interpolated vertex normal when possible.
- Remove redundant calculations: Multiply by 1.0, add 0.0, operations on compile-time constants. Shader compilers optimize some, but manual cleanup helps.
- Move calculations to vertex shader: Per-vertex calculations run 100-1,000x less than per-pixel. Move lighting calculations, texture coordinate transforms, color adjustments to vertex shader when visual quality permits.
- Use lookup textures for complex functions: Replace `sin()`, `cos()`, `exp()` with texture lookup when domain is limited. 1D/2D LUT texture samples are 5-10x faster than complex math.
- Profile instruction counts: Shader Inspector shows instruction count estimates. Target <50 ALU instructions for mobile fragments, <100 for console, <150 for PC. Texture instructions: <5 per fragment shader.

**Texture Sampling Optimization:**
- Minimize texture samples: Each sample costs 20-100 cycles (bandwidth dependent). Combine textures into channels (albedo.a = smoothness, normal.a = AO) to reduce sample count.
- Use texture atlases: Reduces material count, enables batching. Single sample from atlas vs multiple samples from separate textures.
- Avoid dependent texture reads: First sample determines UV for second sample. Stalls texture unit, preventing parallelism. Example: Sampling normal map, then using normal to calculate parallax offset for albedo sample. 3-5x slower than independent samples.
- Use appropriate filtering: Point filtering (0 cost) for exact pixel lookups (LUTs, data textures). Bilinear for standard sampling. Trilinear adds 1 extra sample (2x cost). Anisotropic adds 2-4 samples (2-4x cost).
- Clamp texture coordinates: Use `SamplerState` with Clamp mode instead of saturate(uv) in shader. Hardware clamp is free, shader saturate costs ALU instruction.

**Shader Variant Management:**
- Minimize keywords: Each `#pragma multi_compile` keyword doubles variant count. 10 keywords = 1,024 variants consuming 100MB-1GB memory. Limit to 6-8 keywords max (64-256 variants).
- Use shader_feature instead of multi_compile: `#pragma shader_feature FEATURE` strips unused variants at build time. If no material uses FEATURE, variants not compiled. multi_compile always compiles all variants.
- Implement shader variant stripping: IPreprocessShaders callback removes unused platform/tier combinations. Saves 50-80% shader memory in builds.
- Create specialized shaders: Instead of one shader with 15 keywords, create 3 shaders with 5 keywords each (specialized for characters, environment, props). Fewer variants per shader.
- Use Shader Variant Collection: Track used variants in play mode, strip unused. Edit > Project Settings > Graphics > Preloaded Shaders.

**Precision Management:**
- Use half precision aggressively on mobile/Switch: `half4 color`, `half3 normal`, `half2 uv`. Doubles ALU throughput (2 half operations per cycle vs 1 float operation). Minimal quality loss for colors, UVs, lighting.
- Keep full precision for positions and critical math: `float3 worldPos`, `float depth`. Half precision (10-bit mantissa) insufficient for world-space coordinates (causes jitter/artifacts).
- Annotate precision explicitly: `half` or `float` keywords. Shader compiler respects annotations on mobile (uses FP16 hardware). On PC/console, half may compile to float (no hardware FP16 support).
- Profile precision impact: Mobile gains 40-100% fragment performance using half. Consoles: Small gains on last-gen (GCN has limited FP16), no gain on current-gen (RDNA 2 primarily FP32).

**Branching and Flow Control:**
- Avoid dynamic branching in fragment shaders: `if (condition)` causes thread divergence (some threads execute branch A, others branch B). Both branches execute, results masked. 2x cost.
- Use static branching for keywords: `#ifdef FEATURE` compiles separate variants, no runtime branch cost. Preferred over `if (feature_enabled)`.
- Branch on uniform conditions when possible: `if (globalFlag)` where all pixels in draw have same value. GPU can skip inactive branch. Still slower than no branch—avoid when possible.
- Prefer multiplication by 0/1 instead of branching: `result = baseColor * enabled + altColor * (1 - enabled)` avoids branch. 2 ALU instructions vs 50+ cycle branch penalty.

**Platform-Specific:**
- **Mobile**: Half precision mandatory for performance. Target <50 ALU instructions, <4 texture samples per fragment. Avoid complex lighting (defer to vertex shader or lookup textures).
- **Switch**: Use half precision (Maxwell GPU has FP16 support). Target <80 ALU instructions, <6 texture samples. Shader complexity is primary bottleneck on Switch.
- **PlayStation 4**: GCN architecture benefits from wave64 occupancy. Reduce register pressure (<32 VGPRs) to maximize occupancy. Use shader profiler in Razor to check occupancy.
- **PC**: Modern GPUs handle complex shaders well. Still optimize hotspots (fragment shaders running millions of times per frame). Target <150 ALU instructions, <8 texture samples.

## Common Pitfalls

**Shader Keyword Explosion**: Developer adds keywords for every feature: fog, shadows, reflections, AO, lightmaps, directional lights, point lights, spot lights, etc. 12 keywords = 4,096 variants. Build size balloons to 2GB, shader memory consumes 500MB-1GB. Symptom: Long build times, huge build size, memory issues. Solution: Limit keywords to 6-8 max, use shader_feature instead of multi_compile, implement uber-shader with feature flags in constant buffer.

**Half Precision for World Positions**: Developer uses `half3 worldPos` to optimize. Half precision has 10-bit mantissa (~1/1000 precision). At 1,000 units from origin, precision is 1.0 unit. Mesh vertices jitter, Z-fighting, artifacts. Symptom: Objects far from origin have visual glitches. Solution: Always use full float precision for positions, depths, world-space coordinates.

**Excessive Texture Samples**: Fragment shader samples 10+ textures (albedo, normal, metallic, roughness, AO, emission, detail albedo, detail normal, lightmap, reflection). Each sample costs 20-50 cycles. Total: 200-500 cycles just for texture fetches. Symptom: GPU bandwidth-bound or texture-unit-bound. Solution: Combine textures (pack channels), use texture arrays, or reduce texture detail (not all surfaces need 10 textures).

## Tools & Workflow

**Shader Inspector**: Select shader > Inspector > "Compile and show code". Shows generated shader assembly (HLSL, GLSL, or platform-specific). Count ALU instructions (add, mul, mad), texture instructions (sample, load), registers.

**Frame Debugger**: Window > Analysis > Frame Debugger > Select draw > "Shader" shows active shader variant and properties. "Mesh" shows vertex count. Identify expensive draws.

**PIX Shader Profiler**: Capture frame > Select draw > Shader Model > View disassembly. Shows instruction count, register usage, estimated cycles. "Shader" tab shows per-shader performance metrics.

**Razor (PlayStation)**: "Shader Profiler" shows VGPR (vector register) usage, SGPR (scalar register) usage, occupancy, and wave statistics. Optimize for <32 VGPRs, high occupancy (>50%).

**Nsight Graphics (NVIDIA)**: "Shader Profiler" shows instruction throughput, memory throughput, occupancy. "Source" view correlates HLSL code with assembly instructions.

**Shader Variant Collection**: Window > Rendering > Shader Variant Collection. Track variants used during play sessions, strip unused. Essential for build size reduction.

**RenderDoc Shader Debugging**: Capture frame > Pipeline State > VS/PS tabs show active shaders. "Edit" opens shader debugger, step through shader execution per-pixel.

## Related Topics

- [8.1 Shader Optimization Basics](08-01-Shader-Optimization-Basics.md) - Advanced shader optimization techniques
- [12.1 Shader Programming](12-01-Shader-Programming.md) - Shader development fundamentals
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - GPU performance optimization
- [6.2 GPU Bottlenecks](06-02-GPU-Bottlenecks.md) - Fragment-bound identification

---

[← Previous: 7.2 Mesh Optimization](07-02-Mesh-Optimization.md) | [Next: 7.4 Animation Optimization →](07-04-Animation-Optimization.md)
