# 22.3 Color Spaces

[← Back to Chapter 22](../chapters/22-HDR-And-Color-Management.md) | [Main Index](../README.md)

Color spaces define how colors are encoded/decoded. Linear: mathematical color space (physically accurate for rendering, light accumulates linearly), sRGB: perceptual color space (gamma-corrected for human eye, standard for display, textures, UI). All rendering happens in linear space (physics-correct), output converted to sRGB for display (monitor expects sRGB). Texture import: mark textures sRGB or linear (sRGB textures auto-converted to linear on read, linear textures used as-is).

---

## Overview

Linear Space: mathematical (light values proportional to physical brightness = double value = double brightness). Rendering: additive (lights combine linearly, multiple lights sum accurately), materials (multiply with light = correct blending). Physics-based: physical values match real-world (photons, energy conservation). Problem: human perception nonlinear (eye perceives gamma curve = darker shadows, brighter highlights, S-shaped perception curve).

sRGB Space: perceptual (gamma-corrected for human perception), standard on monitors/web (accounts for eye sensitivity), values non-linear (0.5 sRGB ≠ half brightness = 0.5 linear, sRGB 0.5 ≈ linear 0.25). UI/textures typically sRGB (artists paint in sRGB, natural to human eye). Problem: can't do math in sRGB (adding colors = incorrect results), must convert to linear, do math, convert back to sRGB.

Workflow: sRGB textures (diffuse maps, colors) imported with Gamma = 2.2 (automatic sRGB-to-linear conversion when sampled), linear textures (normal maps, masks, roughness) imported with Gamma = 1.0 (no conversion = raw linear values). Rendering: all calculations linear space (light accumulation, material blending), output: tone mapping (compress HDR to 0-1), convert to sRGB (inverse gamma = linear^(1/2.2) = compressed for display), write to framebuffer.

## Key Concepts

- **Linear Color Space**: Mathematical encoding (value = physical brightness proportional). Example: linear 0.5 = half brightness, linear 1.0 = full brightness. Operations: adding colors = correct (light 0.5 + light 0.5 = 1.0 = expected), multiplying = correct (albedo 0.5 * light 1.0 = 0.5 = reflects half). Used in: physics engine, renderer (all math), material calculations. Problem: monitor can't display linear (human eye uses gamma curve), so linear values appear too dark on display (linear 0.5 looks too dim).
- **sRGB Color Space**: Perceptual encoding (gamma-corrected for human eye, non-linear). Math: sRGB value = linear^(1/2.2) (approximately, exact curve slightly different). Example: sRGB 0.5 = linear 0.25 (sRGB 0.5 looks 50% bright to eye, but mathematically 25% value). Used in: textures (artists paint in sRGB, intuitive), display output (monitor expects sRGB = applies inverse gamma = 2.2 to display = converts sRGB 0.5 to perceived 0.5 brightness). Equation: linear_value = pow(sRGB_value, 2.2), sRGB_value = pow(linear_value, 1/2.2).
- **Gamma Correction**: Gamma = 2.2 (standard for sRGB), represents monitor's gamma curve (older CRTs had inherent 2.2 gamma, standard perpetuated). Forward gamma: applies gamma encoding (linear → sRGB = pow(x, 1/2.2) = dims bright values more than dark values = S-curve perception). Inverse gamma: applies gamma decoding (sRGB → linear = pow(x, 2.2) = amplifies dark values, compresses bright). Workflow: shader samples sRGB texture (GPU applies inverse gamma = converts to linear automatically if texture marked sRGB), does math in linear, outputs to sRGB framebuffer (GPU applies forward gamma = converts to sRGB for display).
- **Texture Import sRGB Flag**: sRGB enabled (checked): texture treated as sRGB (GPU applies inverse gamma when sampling = linear values in shader). Use for: diffuse colors (painted by artist in sRGB), normal maps (NO = don't use sRGB, normal maps are linear), metallic/roughness maps (NO = linear), height maps (NO = linear), masks (NO = linear). Enabled textures: diffuse, specular color, emission, any painted texture. Disabled textures: derived data (normal maps, masks), height/depth data, any data meant for computation not display.
- **sRGB Framebuffer**: Output framebuffer marked as sRGB (GPU applies automatic gamma encoding when writing). Process: shader outputs linear color (result of all calculations), GPU applies pow(color, 1/2.2) = forward gamma (converts linear to sRGB), writes to framebuffer (monitor receives sRGB = applies 2.2 gamma decoding = displays correctly). Without sRGB framebuffer: shader outputs linear, GPU writes directly (no gamma), monitor applies 2.2 gamma (makes image too bright = linear values interpreted as sRGB = appears over-brightened, washed out).

## Best Practices

**Texture Import Setup:**
- Diffuse/Albedo maps: Srgb = checked (artist painted in sRGB, import as sRGB). Alpha channel: treated as linear (separate from sRGB, not gamma-corrected).
- Normal maps: sRGB = unchecked (normals are linear vectors, not colors, gamma corrupts normal directions). Compression: DXT5/BC5 (preserves normal precision).
- Metallic/Roughness maps: sRGB = unchecked (linear values, not colors). Grayscale textures: generally unchecked (data not color).
- Height maps: sRGB = unchecked (linear height values).
- Emission maps: sRGB = checked (emitted light color perceived by eye).
- Rule: if texture is painted color or perceived brightness = sRGB. If texture is data (normals, masks, heights) = linear.

**Rendering Setup:**
- Linear color space: Edit > Project Settings > Player > Color Space = Linear (ensures all rendering in linear, not Gamma = deprecated). Linear requires more processing (shader math in linear = more operations), but correct physics-based rendering.
- Framebuffer: automatically sRGB (GPU handles conversion), verify in Profiler (Render Texture format shows sRGB = correct).
- Output: Camera renders to sRGB framebuffer, automatically gamma-corrected on write.

**Material Creation:**
- Diffuse texture: sRGB checked (drag diffuse map into Albedo slot, sRGB encoding handled).
- Normal map: sRGB unchecked (drag normal map into Normal slot, texture import preview shows correct normals).
- Other maps: unchecked (data maps).
- Verify: sample texture in shader, linear values in calculations produce correct results (no manual gamma correction needed, GPU handles automatically).

**Platform-Specific:**
- **PC**: Linear color space standard (all modern engines use, players expect correct colors). Fallback: Gamma color space for very old systems (rare).
- **Consoles**: PS5/Xbox Series X = linear, PS4/Xbox One = linear (all modern).
- **Mobile**: varies (older Android = Gamma, newer = linear). Detect: SystemInfo.operatingSystemFamily (if old Android, might need Gamma fallback, test on old device).

## Common Pitfalls

**Diffuse Map Not sRGB**: Developer imports diffuse texture, forgets sRGB checkbox (leaves unchecked), image appears too dark in game (linear texture interpreted as linear, shader multiplies by light = results too dim, compared to expected sRGB brightness). Symptom: materials look dark, albedo lost details. Solution: Texture import > Srgb = checked, verify preview in inspector shows correct brightness (not too dark).

**Normal Map sRGB Enabled**: Developer imports normal map, accidentally enables sRGB (gamma corrupts normal vectors), lighting looks wrong (normals distorted by gamma = specular highlights shift, surfaces look incorrectly lit). Symptom: normal maps don't look right (specular highlights in wrong place, surfaces look flat or bizarrely lit). Solution: Normal map texture > sRGB = unchecked, verify preview shows correct normal visualization (blue-ish colors if normal is +Z facing camera).

**Gamma Mismatch Between Platforms**: Developer ships game PC (linear color space), player connects on older Android device that reports Gamma color space (game doesn't handle fallback, rendering incorrect). Scene appears wrong (colors inverted or too bright/dark, depending on how gamma is handled). Solution: detect color space at runtime (SystemInfo.colorSpace), if mismatch apply correction (might apply manual gamma, or recommend player upgrade).

## Tools & Workflow

**Texture Import sRGB Setting**: Import texture (drag into Assets), Inspector > Texture tab > Srgb = checkbox (checked for painted colors, unchecked for data). Preview: shows how texture appears in-game (sRGB checked = brighter, unchecked = darker). Verify preview matches intended use.

**Linear Color Space Project Setting**: Edit > Project Settings > Player > Color Space = Linear. Restart editor (changes apply). Verify: Profiler > GPU shows sRGB framebuffer (correct).

**Framebuffer Format Check**: Profiler > GPU tab, Framebuffer format shows "sRGB" (correct), or "Linear" (wrong = rendering in linear but output not gamma-corrected = image too bright).

**Material Texture Assignment**: Create material, Shader = Standard, drag textures into slots (Albedo = sRGB diffuse, Normal = unchecked normal map, Metallic/Roughness = unchecked masks). Inspector validates (shows texture settings, warns if sRGB wrong = red X on texture slot).

## Related Topics

- [22.1 HDR Rendering Pipeline](22-01-HDR-Rendering-Pipeline.md) - Tone mapping, color accumulation
- [22.2 HDR Display Output](22-02-HDR-Display-Output.md) - Display standards, HDR output
- [22.4 Color Grading Workflows](22-04-Color-Grading-Workflows.md) - Grading in linear/sRGB
- [09.3 Asset Optimization Strategies](09-03-Asset-Optimization-Strategies.md) - Texture compression, formats

---

[Previous: 22.2 HDR Display Output](22-02-HDR-Display-Output.md) | [Next: 22.4 Color Grading Workflows →](22-04-Color-Grading-Workflows.md)
