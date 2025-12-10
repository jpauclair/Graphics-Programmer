# 11.2 PBR Material Properties

[← Back to Chapter 11](../chapters/11-Materials-And-Textures.md) | [Main Index](../README.md)

Physically-Based Rendering (PBR) material properties define surface appearance through albedo, metallic, smoothness, normal, and emission maps following energy conservation and real-world material behavior.

---

## Overview

PBR materials simulate realistic light interaction: dielectrics (non-metals: wood, plastic, skin) have colored diffuse reflection and white specular highlights, metals (iron, gold, copper) have no diffuse reflection and colored specular reflection. Energy conservation ensures light reflected + absorbed ≤ incident light (surfaces cannot emit more light than received). This physical accuracy produces consistent results under different lighting conditions (same material looks correct in daylight, indoor lighting, night scenes).

Core PBR properties: Albedo (base color, diffuse RGB values), Metallic (0.0 = dielectric, 1.0 = metal), Smoothness (0.0 = rough, 1.0 = mirror), Normal map (surface micro-geometry, XYZ perturbations), Ambient Occlusion (crevice darkening, contact shadows), Emission (self-illumination, glow effects). Unity Standard Shader and URP/HDRP Lit shaders implement PBR workflow. Material artists create texture maps (2K-4K resolution) in tools like Substance Painter, Photoshop, Quixel Mixer.

PBR workflow advantages: realistic appearance (matches real-world materials), consistent lighting (materials work in any lighting environment), artist-friendly (intuitive properties), and scalable quality (same material from mobile to high-end PC by adjusting texture resolution). Disadvantages: higher memory usage (5-7 textures per material vs 1-2 for unlit), increased shader complexity (PBR shaders = 100-300 ALU instructions), and longer authoring time (detailed texture maps required).

## Key Concepts

- **Albedo (Base Color)**: Diffuse surface color without lighting or shadows. RGB texture (2K-4K resolution). Pure color values (no lighting baked in). Metals have black albedo (no diffuse), dielectrics have colored albedo.
- **Metallic**: Defines material conductivity. 0.0 = dielectric (plastic, wood, skin), 1.0 = metal (iron, aluminum). Intermediate values rarely used (exceptions: oxidized metals, dust-covered surfaces). Grayscale texture or constant value.
- **Smoothness (Roughness)**: Surface micro-roughness. 0.0 = rough (diffuse reflection), 1.0 = smooth (sharp reflections). Stored in alpha channel of Metallic texture (memory efficiency). Roughness = 1.0 - Smoothness (some workflows use roughness directly).
- **Normal Map**: Tangent-space surface normals. RGB texture encoding XYZ perturbations. Simulates high-resolution geometry detail (scratches, dents, fabric weave) without adding polygons. Typically BC5 compressed (RG channels only).
- **Ambient Occlusion**: Global indirect lighting attenuation. Grayscale texture (0.0 = fully occluded, 1.0 = fully lit). Represents crevices, contact points where indirect light blocked. Often stored in albedo alpha or separate texture.

## Best Practices

**Albedo Authoring:**
- No lighting information: Albedo contains only base color. No shadows, highlights, or ambient occlusion baked in (those come from lighting and AO map). Flat, evenly-lit appearance in texture.
- Calibrated values: Dielectrics: 40-240 sRGB (too dark <40 = looks wrong, too bright >240 = unrealistic). Metals: 180-255 sRGB (high reflectivity). Use reference charts (Substance Painter material library).
- Variation: Add subtle color variation (wood grain, dirt, wear). Solid-color albedo looks artificial (real materials have imperfections). Use noise, procedural textures.
- sRGB color space: Albedo textures in sRGB space (gamma-corrected). Unity converts to linear for lighting calculations. Never store albedo in linear space (looks washed out).

**Metallic/Smoothness:**
- Binary metallic: Use 0.0 (dielectric) or 1.0 (metal), avoid intermediate values. Exception: transitions (rusted metal = 0.5-0.8 metallic in rust areas, 1.0 in clean metal areas).
- Smoothness variation: Add micro-roughness variation. Constant smoothness looks artificial (real surfaces have scratches, fingerprints, dust). Add noise (5-10% variation).
- Pack in alpha: Store smoothness in metallic texture alpha channel (saves memory, one texture instead of two). Unity Standard Shader expects this layout.
- Inverted storage: Some tools export roughness instead of smoothness. Unity expects smoothness (invert in import settings: Texture > Advanced > Invert).

**Normal Map Usage:**
- Tangent-space normals: Use tangent-space normal maps (RGB = XYZ perturbations in surface tangent space). More flexible than object-space (works with deforming meshes, animated characters).
- BC5 compression: Use BC5 format for normal maps (PC/Xbox/PlayStation). Stores only RG channels (XY), reconstructs Z in shader (Z = sqrt(1 - X*X - Y*Y)). 2x memory savings vs BC7 (no quality loss).
- Normal map strength: Avoid excessive strength (Inspector > Normal Map > Bump Scale). Too high = artifacts (inverted normals, lighting discontinuities). Typical range: 0.5-2.0.
- Mipmap filtering: Use default mipmap filtering for normal maps. Blurry normals at distance acceptable (distant objects don't need micro-detail). Avoids aliasing (sharp normals at distance cause sparkles).

**Ambient Occlusion:**
- Bake in DCC: Generate AO in modeling tools (Blender, Maya, 3ds Max) or texture tools (Substance Painter, Marmoset Toolbag). Captures surface crevices, contact points.
- Albedo alpha or separate: Store AO in albedo texture alpha (saves memory), or separate texture (more flexibility, adjust AO strength independently).
- Multiply with lighting: AO multiplied with indirect lighting only (not direct lights). Darkens crevices in ambient light, doesn't affect sunlight/spotlights.
- Screen-space AO complement: Texture AO captures static geometry occlusion. SSAO (Screen-Space Ambient Occlusion) adds dynamic occlusion (moving objects, runtime-placed props). Use both for best results.

**Emission:**
- HDR emission: Use HDR color values (intensity >1.0) for bloom effects. Emission = Color.white * 5.0 creates bright glow (visible in bloom post-process).
- Emission maps: Grayscale or RGB texture (emissive areas white, non-emissive black). Multiply by emission color (tint). Example: neon sign = emission map (text pattern) * color (red/blue neon).
- Performance cost: Emission adds shader ALU (5-10 instructions). Use sparingly (emissive objects only). Don't enable emission on all materials (wastes GPU).
- Global Illumination: Emissive materials contribute to baked GI (lightmaps). Increase emission intensity for GI impact (Emission GI multiplier in Lighting window).

**Platform-Specific:**
- **PC**: Full PBR support. Use 2K-4K textures (albedo, metallic-smoothness, normal, AO, emission). BC7 (albedo), BC5 (normal), BC4 (metallic-smoothness). Quality tiers: Low = 512-1K, High = 2K-4K.
- **Consoles**: Full PBR support. 2K textures typical (4K for hero assets). Use BC7/BC5 compression. Optimize shader variants (reduce PBR features on distant LODs).
- **Switch**: Simplified PBR. 1K-2K textures (ASTC 6x6 or 8x8). Reduce map count (combine AO into albedo alpha, skip emission unless necessary). Mobile/Diffuse shader for background objects.
- **Mobile**: Reduced PBR. 512-1K textures (ASTC 8x8). Fewer maps (albedo + normal only, skip metallic/AO). Use Shader Graph Mobile/Lit shader (optimized instruction count).

## Common Pitfalls

**Lighting in Albedo**: Artist bakes shadows/highlights into albedo texture. Material looks wrong under different lighting (shadows in wrong places, double-lighting). Symptom: Material has dark spots (baked shadows) that don't match scene lighting. Solution: Remove lighting from albedo (flatten shadows in Photoshop, use "Diffuse" map from texture baking tool, not "Shaded").

**Incorrect Metallic Values**: Artist uses intermediate metallic (0.5) for all materials. Materials look neither metal nor non-metal (incorrect lighting response). Symptom: Weird specular highlights, energy conservation violations, materials look "plasticky". Solution: Use binary metallic (0.0 or 1.0). Exceptions: oxidized metals (0.7-0.9), dust-covered metals (0.8-1.0).

**Constant Smoothness**: Artist uses uniform smoothness (0.5 across entire material). Surface looks artificial (too perfect, CG-like). Symptom: Unrealistic, plasticky appearance. Solution: Add micro-variation. Use noise (5-10% variation), paint wear/scratches in high-traffic areas (edges, handles), vary smoothness per material region (clean metal = 0.9, rusted area = 0.3).

**Wrong Normal Map Format**: Artist imports normal map as default texture (sRGB, compressed as BC7). Normals stored incorrectly (sRGB gamma mangling), compression artifacts. Symptom: Lighting discontinuities, inverted normals, "solarization" effect. Solution: Set Texture Type = Normal Map in import settings. Unity converts to linear, uses BC5 compression automatically.

## Tools & Workflow

**Substance Painter**: Industry-standard PBR texture authoring. Paint albedo, metallic, roughness, normal, AO in layers. Export Unity-compatible maps (albedo, metallic-smoothness, normal). www.substance3d.com.

**Quixel Mixer**: Free PBR material creation (formerly Megascans). Blend materials (wood + dirt + moss), export Unity maps. Bridge integration (drag materials into Unity). quixel.com/mixer.

**Photoshop/GIMP**: Manual texture editing. Adjust albedo colors, paint smoothness variation, combine maps (pack smoothness into metallic alpha). Use color calibration (dielectric albedo = 40-240 sRGB).

**RenderDoc/PIX**: Inspect material properties at runtime. Capture frame, view textures bound to material (albedo, metallic, normal). Verify correct formats (BC7, BC5), identify missing maps.

**Unity Frame Debugger**: View material properties per draw call. Shows shader properties (albedo color, metallic value, smoothness), bound textures. Verify materials assigned correctly.

**Material Validator (Asset Store)**: Analyzes materials for PBR correctness. Checks albedo brightness (warns if <40 or >240 sRGB), metallic values (warns if intermediate values used), normal map format. Identifies authoring mistakes.

**Shader Graph**: Visual PBR shader creation. Exposes PBR properties (albedo, metallic, smoothness, normal, emission) as graph inputs. Customize PBR workflow (add detail maps, parallax, custom lighting).

## Related Topics

- [11.1 Material System Architecture](11-01-Material-System-Architecture.md) - Material management
- [11.3 Texture Mapping Techniques](11-03-Texture-Mapping-Techniques.md) - UV mapping
- [7.1 Texture Optimization](07-01-Texture-Optimization.md) - Texture compression
- [13.1 Lighting Models](../chapters/13-Rendering-Techniques.md) - PBR lighting

---

[← Previous: 11.1 Material System Architecture](11-01-Material-System-Architecture.md) | [Next: 11.3 Texture Mapping Techniques →](11-03-Texture-Mapping-Techniques.md)
