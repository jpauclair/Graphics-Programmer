# 12.3 Lighting Models

[← Back to Chapter 12](../chapters/12-Shader-Programming.md) | [Main Index](../README.md)

Lighting models calculate surface illumination through mathematical formulations: Lambert diffuse, Blinn-Phong specular, Physically-Based Rendering (PBR), and custom stylized models like toon/cel-shading.

---

## Overview

Lighting models define how surfaces reflect light. Basic models: Lambert (diffuse reflection, N·L, rough surfaces), Blinn-Phong (diffuse + specular, adds shininess highlights), and Phong (reflection-based specular, computationally expensive). Modern games use PBR (Physically-Based Rendering): Cook-Torrance BRDF (microfacet theory), GGX specular distribution (realistic highlights), Fresnel effect (grazing angle reflections), and energy conservation (reflected + absorbed ≤ incident light).

PBR advantages: realistic appearance (matches real-world materials), consistent results (works in any lighting environment), artist-friendly workflow (intuitive parameters: metallic, roughness), and scalable (same material from mobile to high-end PC). Unity Standard Shader implements PBR (metallic workflow: albedo, metallic, smoothness). URP/HDRP use PBR by default (Lit shader = Cook-Torrance BRDF + GGX).

Custom lighting models: stylized rendering (toon shading with hard shadow boundaries, cel-shading with discrete lighting bands), artistic direction (non-photorealistic rendering), or performance optimization (simplified Lambert for mobile, no specular calculations). Implementation: Surface Shader with custom lighting function (LightingCustom), or vertex/fragment shader with manual lighting code. Trade-offs: artistic control vs realistic appearance, performance vs visual quality.

## Key Concepts

- **Lambert Diffuse**: Basic diffuse lighting. Light intensity = max(0, N·L) (dot product of normal and light direction). No view dependence (looks same from all angles). Fast (1 dot product), suitable for rough surfaces (cloth, matte materials).
- **Blinn-Phong**: Diffuse + specular highlights. Diffuse = Lambert, Specular = max(0, N·H)^shininess (H = half vector between light and view). Shininess controls highlight size (high = small sharp highlight, low = large soft highlight). Classic model (pre-PBR games).
- **Cook-Torrance BRDF**: PBR microfacet model. BRDF = D(h) * F(v,h) * G(l,v,h) / (4 * (n·l) * (n·v)). D = specular distribution (GGX), F = Fresnel (Schlick approximation), G = geometry/shadowing term. Physically accurate, energy conserving.
- **GGX Distribution**: Specular highlight shape in PBR. Realistic tail (soft falloff), wider highlights than Blinn-Phong. Used in Unity Standard Shader, URP/HDRP Lit, Unreal Engine. Formula: D(h) = α² / (π * ((n·h)² * (α²-1) + 1)²), α = roughness.
- **Fresnel Effect**: Reflectivity increases at grazing angles. Surfaces more reflective when viewed edge-on (water reflects more at shallow angles). Schlick approximation: F(v,h) = F0 + (1-F0) * (1 - v·h)^5. F0 = base reflectivity (0.04 for dielectrics, 0.5-1.0 for metals).

## Best Practices

**PBR Implementation:**
- Use Unity Standard Shader: Built-in Pipeline PBR shader. Metallic workflow (albedo, metallic, smoothness, normal, AO, emission). Supports all PBR features (GGX, Fresnel, energy conservation). Well-tested, optimized, cross-platform.
- URP/HDRP Lit Shader: SRP PBR shaders. URP Lit (mobile-friendly PBR), HDRP Lit (high-end PBR with advanced features: anisotropy, coat, subsurface scattering). Use Shader Graph to customize (add custom PBR variations).
- Material properties: Albedo = base color (no lighting), Metallic = 0 (dielectric) or 1 (metal), Smoothness = 0 (rough) to 1 (mirror), Normal map = surface detail. Follow PBR authoring guidelines (albedo values 40-240 sRGB for dielectrics, 180-255 for metals).
- Energy conservation: Diffuse + specular ≤ 1.0. PBR shaders handle automatically (metallic = 1 disables diffuse, surfaces can't reflect more light than received). Don't override energy conservation (breaks physical accuracy).

**Custom Lighting Functions (Surface Shader):**
- Define LightingCustom: half4 LightingCustom(SurfaceOutput s, half3 lightDir, half atten). Implement custom lighting math (Lambert, Blinn-Phong, stylized). Return final color (diffuse + specular).
- Surface Shader pragma: #pragma surface surf Custom (surf = surface function, Custom = LightingCustom function). Unity calls LightingCustom for each light (supports multi-light automatically).
- Access light properties: lightDir (light direction), atten (attenuation/shadows), _LightColor0 (light color/intensity). Unity provides per-light data (no manual light loops).
- Example toon shading: Quantize diffuse (NdotL > 0.5 ? 1.0 : 0.0). Creates hard shadow boundary (cel-shaded look). Add specular threshold for stylized highlights.

**Vertex/Fragment Lighting:**
- Manual light loops: Built-in Pipeline supports 4 per-pixel lights + 4 per-vertex lights. Loop through unity_LightColor[i], unity_LightPosition[i]. Accumulate lighting (diffuse + specular per light). Expensive (4-8 lights = 50-100 ALU per pixel).
- URP lighting: URP provides helper functions (GetMainLight, GetAdditionalLight(i)). Main light (directional sun) + additional lights (point/spot, limited count). Use in fragment shader (calculate lighting per pixel).
- HDRP lighting: HDRP uses clustered rendering (lights grouped by screen tiles). GetLightCount(), GetLightData(i). Supports hundreds of lights efficiently (culled per tile).
- Shadow sampling: SHADOW_COORDS, TRANSFER_SHADOW (vertex), SHADOW_ATTENUATION (fragment). Samples shadow maps, returns attenuation (0 = shadowed, 1 = lit). Multiply diffuse by attenuation (applies shadows).

**Stylized/Non-Photorealistic Lighting:**
- Toon/Cel-shading: Quantize lighting (diffuse = step(NdotL, threshold)). Creates hard edges (cartoon aesthetic). Multi-band: use smoothstep or multiple thresholds (shadow, mid-tone, highlight). Outline rendering via inverted hull or edge detection.
- Rim lighting: Highlight edges (fresnel-like effect). rimLight = pow(1.0 - NdotV, rimPower). Adds glow around silhouette (sci-fi, stylized characters). Multiply by rim color, add to final output.
- Wrap lighting: Extend diffuse range (NdotL remapped). diffuse = max(0, (NdotL + wrap) / (1 + wrap)). Simulates subsurface scattering (light wraps around edges). Use for skin, foliage, translucent materials.
- Custom BRDF: Implement artistic BRDF (non-physical, stylized highlights). Example: cloth BRDF (velvet-like sheen), skin BRDF (subsurface scattering approximation). Requires shader code (Surface Shader custom function or vertex/fragment).

**Performance Optimization:**
- Half precision: Use half for lighting calculations on mobile (half3 lightDir, half3 viewDir, half NdotL). Mobile GPUs process half 2x faster (16-bit ALUs). PC/console: half and float equivalent (no benefit).
- Simplified models: Mobile uses simplified PBR (single light, no specular, or Lambert-only). Distant LODs use unlit or vertex lighting (no per-pixel calculations). Balance quality vs frame rate.
- Precomputed lighting: Lightmaps (baked diffuse), light probes (dynamic objects), reflection probes (specular reflections). Offload lighting to offline baking (runtime only samples precomputed data). Massive performance win (5-10ms GPU savings).
- Shader LOD: Disable expensive lighting features on distant objects. LOD0 = full PBR, LOD1 = simplified PBR (no specular), LOD2 = Lambert diffuse, LOD3 = unlit. Use Shader.maximumLOD or material swapping.

**Platform-Specific:**
- **PC**: Full PBR with all features. GGX specular, Fresnel, multiple lights (4-8 per-pixel), shadows, GI. High shader complexity acceptable (100-300 ALU instructions). Quality Settings > High/Ultra.
- **Consoles**: Full PBR. Optimized for fixed hardware (4-6 per-pixel lights typical). Use clustered lighting (HDRP) or forward+ (efficient multi-light rendering). Quality Settings > High.
- **Switch**: Simplified PBR. Single main light + ambient. Reduce specular quality (Blinn-Phong instead of GGX acceptable). Mobile shaders recommended (Mobile/Diffuse for background). Quality Settings > Medium/Low.
- **Mobile**: Very simplified lighting. Lambert diffuse or simplified PBR (single light, no specular). Unlit shaders for backgrounds. Baked lighting preferred (lightmaps + probes, no runtime lights). Quality Settings > Low.

## Common Pitfalls

**Non-Physical Albedo**: Artist bakes lighting into albedo (highlights, shadows in texture). PBR lighting applies on top (double-lighting, incorrect appearance). Symptom: Too-bright highlights, baked shadows in wrong places, material doesn't respond correctly to lights. Solution: Remove lighting from albedo (flatten in Photoshop, use "Diffuse" map not "Shaded" map). Albedo should be flat, evenly-lit base color.

**Incorrect Normal Map Space**: Developer uses object-space normal map with tangent-space shader. Normals incorrect (lighting artifacts, inverted normals). Symptom: Lighting discontinuities, surface appears inside-out. Solution: Use tangent-space normal maps (standard). Unity expects tangent-space (RGB encoded XYZ perturbations in surface tangent basis).

**Missing Energy Conservation**: Developer implements custom lighting without energy conservation. Diffuse + specular > 1.0 (surfaces emit more light than received, physically impossible). Symptom: Over-bright materials, blown-out highlights. Solution: Ensure diffuse + specular ≤ 1.0. In PBR: lerp(diffuse, 0, metallic) (metals have no diffuse). Normalize BRDF output.

**Per-Vertex Lighting Artifacts**: Developer uses vertex lighting for large triangles (terrain, walls). Lighting interpolated linearly across triangle (smooth gradient, misses details). Specular highlights swim across surface (vertex movement causes highlight drift). Symptom: Lighting looks blurry, highlights move incorrectly, low-poly look. Solution: Use per-pixel lighting (fragment shader). Vertex lighting only for low-poly objects (distant LODs, stylized games).

## Tools & Workflow

**Unity Standard Shader**: Built-in PBR shader. Material Inspector shows PBR properties (Albedo, Metallic, Smoothness, Normal, etc.). Use as reference for custom PBR implementations (source in built-in shaders package).

**URP/HDRP Lit Shader**: SRP PBR shaders. URP: Packages/com.unity.render-pipelines.universal/Shaders/Lit*.hlsl. HDRP: Packages/com.unity.render-pipelines.high-definition/Runtime/Material/Lit. Well-commented, reference implementations.

**Shader Graph Lighting**: Shader Graph > Master Stack > Fragment > Lighting. Choose Built-in Lit (PBR lighting), Custom Lit (write custom lighting in Custom Lighting sub-graph). Visual PBR implementation (no code).

**RenderDoc/PIX**: Inspect lighting calculations. Pixel debugger shows intermediate values (NdotL, specular, Fresnel). Step through lighting code (HLSL debugger). Identify incorrect calculations (wrong normals, inverted values).

**Lighting Debugger**: URP/HDRP Rendering Debugger (Window > Analysis > Rendering Debugger). Material > Lighting shows lighting components (diffuse only, specular only, GI only). Isolate lighting issues (is diffuse correct? Is specular wrong?).

**Frame Debugger**: View lighting per draw call. Shows light count, light types (directional, point, spot), shadow maps. Verify lighting setup (are lights affecting object? Are shadows casting?).

**Substance Painter**: PBR material authoring. Real-time PBR preview (matches Unity PBR). Verify materials look correct in PBR renderer before importing to Unity.

## Related Topics

- [12.2 Shader Types](12-02-Shader-Types.md) - Surface vs Vertex/Fragment shaders
- [11.2 PBR Material Properties](11-02-PBR-Material-Properties.md) - PBR material authoring
- [13.1 Lighting Techniques](../chapters/13-Rendering-Techniques.md) - Scene lighting setup
- [14.1 Lightmapping Fundamentals](../chapters/14-GI-And-Lightmapping.md) - Baked lighting

---

[← Previous: 12.2 Shader Types](12-02-Shader-Types.md) | [Next: 12.4 Advanced Shader Techniques →](12-04-Advanced-Shader-Techniques.md)
