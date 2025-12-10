# 31.1 Shader Tools

[← Back to Chapter 31](../chapters/31-Third-Party-Tools.md) | [Main Index](../README.md)

Shader tools = visual editors for shader authoring (Shader Graph = Unity built-in node editor, Amplify Shader Editor = advanced Asset Store tool). Node-based workflow: connect operations visually (add, multiply, texture sample = drag-and-drop), generates HLSL automatically. Benefits: faster iteration (no compilation, live preview), accessible to artists (no coding required), visual debugging (see intermediate values).

---

## Overview

Shader Graph: Unity's node-based shader editor (URP/HDRP only, not Built-in). Create shaders visually (PBR, Unlit, custom lighting), connect nodes (math, texture, vertex manipulation), output to Master Stack (surface properties). Features: Sub Graphs (reusable node groups), Custom Function Nodes (embed HLSL), Keyword variants (shader permutations), live preview (real-time material ball).

Amplify Shader Editor: premium Asset Store tool ($60, supports all render pipelines). Advanced features: function library (pre-built effects), template system (customize shader structure), multi-pass shaders (complex rendering), better Built-in pipeline support. Popular choice: AAA studios, complex custom shaders, legacy projects.

## Key Concepts

- **Shader Graph (Unity)**: node-based shader editor, URP/HDRP. Window → Shader Graph → Create → PBR/Unlit/Custom. Master Stack = output nodes (Base Color, Normal, Metallic, Smoothness, Emission). Sub Graphs = reusable node groups (save as .shadersubgraph, reference in other graphs). Custom Function Node = embed HLSL code (custom calculations not available as nodes). Limitations: no Built-in RP support, URP/HDRP only.
- **Amplify Shader Editor**: third-party node editor, all render pipelines. Features: 300+ nodes (functions, effects, utilities), Templates (customize shader structure = surface, vertex, fragment), Function Library (pre-built effects like dissolve, triplanar, parallax), Shader Variants (keywords, multi_compile). Performance: generates optimized HLSL (no overhead vs hand-coded). Cost: $60 Asset Store, worth it for Built-in RP or complex shaders.
- **Node-Based Workflow**: visual programming (connect outputs to inputs, data flows left-to-right). Common nodes: Sample Texture 2D (UV input → color output), Multiply (A × B), Add (A + B), Lerp (interpolate), Split (separate RGBA), Combine (merge channels). Live preview: material ball updates real-time (see changes immediately). Debugging: preview individual nodes (right-click → preview, see intermediate values).
- **Sub Graphs / Macros**: reusable node groups (create once, use in multiple shaders). Example: custom lighting model (normal, light dir, attenuation → lit color), noise functions (UV → Perlin noise), triplanar mapping (world position → blended textures). Save as .shadersubgraph (Shader Graph) or .asset (Amplify). Benefits: consistency (same logic everywhere), maintainability (update once = all shaders updated).
- **Custom Function Nodes**: embed HLSL code in visual graph. Use cases: complex math (not available as nodes), hardware intrinsics (WaveReadLaneFirst), optimized code (avoid node overhead). Shader Graph: Custom Function Node (input/output declarations, HLSL file reference). Amplify: Custom Expression (inline HLSL, variable bindings).
- **Shader Variants**: multiple shader versions (keywords change code path). Example: #pragma multi_compile _ USE_NORMAL_MAP (with/without normal mapping). Shader Graph: Keywords (Boolean, Enum), enable/disable features. Amplify: Shader Variant node (multi_compile directives). Warning: too many variants = long build times (2^N combinations, exponential growth).

## Best Practices

**Tool Selection**:
- Shader Graph: URP/HDRP projects (built-in, free, official support).
- Amplify: Built-in RP, complex custom shaders, multi-pass effects, team familiar with tool.
- Hand-coded HLSL: maximum performance (node overhead eliminated), complex logic (node graphs hard to read).

**Performance Optimization**:
- Preview before bake: check instruction count (Inspector → Shader → Instructions).
- Avoid redundant nodes: same texture sampled multiple times (cache result).
- Use Sub Graphs: reduce complexity (break into logical pieces = easier optimization).
- Static branching: use keywords (compile-time = no runtime cost) vs dynamic branching (if statements = GPU divergence).

**Version Control**:
- Shader Graph: .shadergraph files (text-based, merge-friendly YAML).
- Amplify: .asset files (binary, merge conflicts difficult).
- Solution: Amplify text serialization mode (Preferences → Amplify → Force Text Serialization).

**Platform-Specific**:
- **Mobile**: use Shader Graph Simplified nodes (mobile-optimized), avoid complex math (pow, sin, cos).
- **Console**: full feature set supported (all nodes, variants, custom functions).
- **VR**: instancing support (Shader Graph built-in, Amplify configure templates).

## Common Pitfalls

**Too Many Variants**: developer adds 5 keywords (2^5 = 32 variants). Symptom: build time hours (Unity compiles all permutations), APK bloated (megabytes of shader code). Solution: use shader_feature instead of multi_compile (only compiled if used), limit keywords to 3-4.

**Missing Sub Graph References**: Sub Graph renamed/moved (graph breaks, pink shader). Symptom: material pink after update. Solution: fix references in broken graph (reconnect Sub Graph node), or use GUID-based references (don't rename in file explorer).

**Node Overhead**: complex graph (100+ nodes, deeply nested). Symptom: performance worse than hand-coded HLSL (node abstraction overhead). Solution: Custom Function Node for complex sections (optimize critical path), or hand-code entire shader (if performance critical).

## Tools & Workflow

**Shader Graph Creation** (Unity):
```
1. Project → Create → Shader Graph → URP → Lit Shader Graph
2. Open graph: double-click .shadergraph file
3. Add nodes: Right-click → Create Node → search (e.g., "Sample Texture 2D")
4. Connect nodes: drag output → input (color → Base Color)
5. Save: Ctrl+S (generates HLSL automatically)
6. Use: Material → Shader dropdown → Shader Graphs → YourShader
```

**Amplify Shader Editor Workflow**:
```
1. Asset Store → Download Amplify Shader Editor
2. Create shader: Project → Create → Amplify Shader → Surface Shader
3. Open editor: double-click .shader file
4. Node library: right-click canvas → search (e.g., "Triplanar")
5. Configure: Master Node (Albedo, Normal, Emission inputs)
6. Generate: Live Mode (auto-compile), or Manual (click Apply)
```

**Custom Function Node** (Shader Graph):
```hlsl
// CustomNoise.hlsl
void CustomNoise_float(float2 UV, out float Noise)
{
    Noise = frac(sin(dot(UV, float2(12.9898, 78.233))) * 43758.5453);
}
```
```
// Shader Graph
1. Create → Custom Function Node
2. Type: File, File: CustomNoise.hlsl
3. Inputs: UV (Vector2), Outputs: Noise (Float)
4. Connect UV → node → use Noise output
```

**Sub Graph Example** (Shader Graph):
```
1. Create → Sub Graph
2. Add nodes: Input (UV), Sample Texture, Multiply (tint color)
3. Outputs: expose Color output
4. Save as "TintedTexture.shadersubgraph"
5. Use: Main graph → Add → Sub Graph → select TintedTexture
```

**Shader Variant Management**:
```hlsl
// Amplify: Shader Variant node
#pragma multi_compile _ USE_NORMAL_MAP
#pragma multi_compile _ USE_PARALLAX

// Shader Graph: Keyword node
Keyword: USE_NORMAL_MAP (Boolean)
Branch: connect to Normal input (enabled/disabled path)
```

## Related Topics

- [11.1 Shader Fundamentals](12-01-Unity-Shader-Languages.md) - HLSL basics
- [11.3 Custom Shader Writing](12-02-Shader-Types.md) - Hand-coded shaders
- [12.3 URP Shader Graph](12-01-Unity-Shader-Languages.md) - URP integration
- [31.2 Rendering Extensions](31-02-Rendering-Extensions.md) - Post-processing tools

---

[← Previous: Chapter 30 Mathematics](../chapters/30-Mathematics-For-Graphics.md) | [Next: 31.2 Rendering Extensions →](31-02-Rendering-Extensions.md)
