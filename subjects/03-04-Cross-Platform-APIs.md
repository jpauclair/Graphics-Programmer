# 3.4 Cross-Platform APIs

[← Back to Chapter 3](../chapters/03-Platform-Specific-Graphics-APIs.md) | [Main Index](../README.md)

Cross-platform graphics APIs enable rendering on multiple platforms with shared code. Vulkan, Metal, and legacy OpenGL provide varying levels of abstraction and platform support.

---

## Overview

Cross-platform APIs reduce development costs by sharing rendering code across Windows, Linux, Android, iOS, and sometimes consoles. Vulkan is the modern choice—low-level like DirectX 12, supporting PC (Windows/Linux), Android, and some consoles (Switch via NVN). Metal is Apple's API for iOS/macOS, offering similar low-level control with simpler design. OpenGL/OpenGL ES are legacy high-level APIs, still relevant for older hardware or simple projects but deprecated in favor of modern alternatives.

Unity abstracts platform APIs via SRP (Scriptable Render Pipeline), automatically selecting DirectX 12, Vulkan, Metal, or OpenGL based on platform. For most Unity developers, understanding API differences guides optimization (Vulkan's explicit barriers vs Metal's automatic resource tracking) and debugging (Vulkan validation layers vs Metal's frame capture). Custom native plugins or engine-level work requires direct API knowledge.

Vulkan's verbosity (500+ lines to create a triangle) is offset by explicit control and portability. Metal's cleaner API design reduces boilerplate but locks you to Apple platforms. OpenGL's simplicity is attractive but high driver overhead (0.5-1ms per draw call) makes it unsuitable for modern AAA rendering (2,000+ draws). Projects targeting PC/console should prioritize DirectX 12/Vulkan; mobile projects should use Metal (iOS) and Vulkan (Android).

## Key Concepts

- **Vulkan**: Low-level cross-platform API derived from AMD Mantle. Explicit memory management, command buffers, and synchronization. Supported on Windows, Linux, Android, and Switch (via NVN). Extremely verbose but maximum control.
- **Metal**: Apple's graphics API for iOS, macOS, and tvOS. Low-level control similar to Vulkan but cleaner design. Automatic resource tracking reduces boilerplate. Proprietary to Apple platforms.
- **OpenGL**: Legacy high-level API (1992-present). Driver-managed resources, implicit sync, and immediate-mode rendering. High overhead, deprecated by Vulkan. Still used for legacy support.
- **OpenGL ES**: OpenGL variant for mobile and embedded systems. Reduced feature set vs desktop OpenGL. Common on older Android devices and some embedded platforms. Being replaced by Vulkan.
- **Validation Layers**: Debug tools for Vulkan detecting API misuse (invalid states, sync errors). Essential for catching errors—Vulkan drivers assume correct usage and crash silently on errors.

## Best Practices

**Vulkan Optimization:**
- Use validation layers during development. Enable via `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` and layer settings. Catches sync errors, memory hazards, and API misuse before they cause GPU crashes.
- Minimize descriptor set updates. Vulkan descriptor sets bind resources to shaders; updates cost CPU time. Use descriptor indexing (dynamic indexing into large arrays) to reduce updates from hundreds per frame to <10.
- Leverage render passes for automatic layout transitions. Vulkan requires explicit image layout transitions (e.g., `COLOR_ATTACHMENT` to `SHADER_READ`). Render passes handle this automatically when declared.
- Batch command buffer submissions. Each `vkQueueSubmit` has overhead (0.1-0.5ms). Group draws into large command buffers and submit once per frame instead of per-draw.
- Use memory aliasing for transient resources. Vulkan allows multiple resources to share memory if usage doesn't overlap (e.g., G-buffer and post-process targets). Saves 30-50% VRAM in deferred renderers.

**Metal Optimization:**
- Use argument buffers for resource binding. Metal's equivalent to descriptor sets, reducing API calls for large resource arrays. Essential for bindless rendering or many material textures.
- Leverage tile shading on Apple GPUs. Mobile GPUs use tile-based rendering; Metal exposes tile memory (threadgroup memory for fragment shaders). Use for custom resolve operations or bandwidth savings.
- Minimize render encoder changes. Metal's `MTLRenderCommandEncoder` switching costs CPU time. Group draws by encoder when possible (same render target, similar state).
- Use Metal's memoryless render targets for intermediate buffers. Depth/stencil buffers used only during rendering don't need backing memory. Saves VRAM and bandwidth on mobile.
- Profile with Xcode Instruments. "Metal System Trace" shows GPU timeline, shader execution, and memory bandwidth. "Metal Debugger" provides frame capture and shader debugging.

**OpenGL Considerations:**
- Minimize state changes. OpenGL state machine changes (bind texture, enable blend, etc.) flush pipelines. Sort draws by state to reduce changes from thousands to hundreds.
- Use vertex array objects (VAOs) to cache vertex state. Binding VAO restores all vertex attributes instantly vs re-binding buffers/attributes per draw.
- Avoid immediate mode (`glBegin`/`glEnd`) entirely. Use vertex buffer objects (VBOs) for all geometry. Immediate mode is 10-100x slower.
- Prefer modern OpenGL (4.5+) over legacy (2.1). Modern OpenGL adds features like direct state access (DSA) reducing API overhead. Legacy OpenGL is maintenance mode only.
- On mobile, target OpenGL ES 3.1+ for compute shaders and advanced features. ES 2.0 is too limited for modern rendering (no MRT, limited texture formats).

**Cross-Platform Strategy:**
- Use Unity's abstraction when possible. SRP handles API differences (command buffer syntax, shader compilation). Custom shaders compile to HLSL, GLSL, or Metal Shading Language automatically.
- For native plugins, abstract APIs via interface: `IRenderDevice`, `ICommandBuffer`, etc. Implement per-API (Vulkan, DirectX 12, Metal). Isolates platform code, easing maintenance.
- Test on all target platforms early. Vulkan bugs on NVIDIA may not appear on AMD or mobile PowerVR. API differences cause subtle issues (GLSL precision qualifiers on mobile, Metal's buffer alignment requirements).

**Platform-Specific:**
- **Vulkan on PC**: Excellent on NVIDIA/AMD, improving on Intel. Use for Linux or multi-platform PC support. Driver quality varies—validate extensively.
- **Vulkan on Android**: Standard on modern devices (Android 7+). Performance varies wildly by GPU (Qualcomm Adreno, ARM Mali, PowerVR). Test on low-end devices (<$200 phones).
- **Metal on iOS**: Only API for iOS/macOS (OpenGL ES deprecated). Mandatory for App Store submission. Excellent tooling (Xcode debugger, Instruments) and driver quality.
- **Metal on macOS**: Replacing OpenGL (deprecated in macOS 10.14). Apple Silicon (M1/M2/M3) has unified memory and tile-based rendering. Optimize for these characteristics.

## Common Pitfalls

**Missing Validation Layers in Vulkan**: Developing without validation layers enabled. Vulkan assumes correct API usage—errors cause silent corruption or GPU crashes. Symptom: Works on development GPU, crashes on user hardware. Solution: Enable validation layers (`VK_LAYER_KHRONOS_validation`) during development. Fix all warnings before shipping.

**Over-Synchronization in Vulkan**: Inserting pipeline barriers after every draw call "to be safe." Each barrier stalls GPU, destroying parallelism. Performance drops 50-70%. Solution: Analyze resource dependencies, insert barriers only when necessary (render target write → shader read). Use render pass subpass dependencies for automatic sync.

**Ignoring Mobile GPU Differences**: Metal shader works on iOS (tile-based GPU) but has artifacts on macOS (immediate-mode GPU). Tile GPUs have different precision rules and memory bandwidth characteristics. Solution: Test on both architectures. Use explicit precision qualifiers in shaders (`lowp`, `mediump`, `highp` in Metal Shading Language).

## Tools & Workflow

**RenderDoc**: Excellent Vulkan and OpenGL debugger. Frame capture, shader debugging, and resource inspection. Supports PC (Windows/Linux) and Android. Essential for Vulkan development—visualizes resource states, barriers, and command buffers.

**Xcode GPU Debugger**: Metal's primary debugger. Frame capture, shader debugging, and pipeline state inspection. "Metal Debugger" shows bound resources and shader assembly. "Instruments" profiles GPU timeline and bandwidth.

**NVIDIA Nsight Graphics**: Supports Vulkan and DirectX 12. Deep GPU profiling (occupancy, memory bandwidth, unit utilization). Use for Vulkan performance analysis on NVIDIA hardware.

**Vulkan SDK**: Khronos Group's official SDK. Includes validation layers, shader compiler (glslangValidator), and documentation. LunarG provides enhanced SDK with additional tools (Vulkan Configurator, layer manager).

**Unity Shader Graph**: Abstracts shaders across APIs. Compiles to HLSL (DirectX), GLSL (Vulkan/OpenGL), or Metal Shading Language automatically. Ideal for cross-platform projects—avoids manual shader porting.

## Related Topics

- [3.1 DirectX (PC/Xbox)](03-01-DirectX.md) - DirectX API details
- [3.2 PlayStation](03-02-PlayStation.md) - PlayStation GNM/GNMX API
- [3.3 Nintendo Switch](03-03-Nintendo-Switch.md) - Switch NVN API (Vulkan-based)
- [12.1 Unity Shader Languages](12-01-Unity-Shader-Languages.md) - Cross-platform shader compilation

---

[← Previous: 3.3 Nintendo Switch](03-03-Nintendo-Switch.md) | [Next: Chapter 4 - Profiling and Analysis Tools →](../chapters/04-Profiling-And-Analysis-Tools.md)
