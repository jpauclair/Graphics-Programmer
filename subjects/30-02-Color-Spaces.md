# 30.2 Color Space Mathematics

[← Back to Chapter 30](../chapters/30-Mathematics-For-Graphics.md) | [Main Index](../README.md)

Color spaces = mathematical representations of color (RGB additive, CMYK subtractive, HSV perceptual). Graphics uses RGB (red,green,blue = monitor pixels), but artists prefer HSV/HSL (hue,saturation,value = intuitive color picking). Conversions = mathematical transformations (RGB ↔ HSV ↔ Lab), gamma correction (sRGB nonlinear = human perception), and color operations (blend, lerp, multiply = combine colors).

---

## Overview

RGB Color Model: additive color (red + green + blue = white), each component 0-1 (or 0-255 8-bit). Linear RGB = physics-correct (double value = double brightness), sRGB = gamma-corrected (0.5 sRGB ≈ 0.22 linear = matches human eye perception). Textures stored sRGB, GPU converts to linear for rendering, converts back to sRGB for display.

Color Operations: lerp colors (linear interpolation = RGB lerp), multiply colors (material color × light color = modulation), add colors (light accumulation = additive blending). Color space affects results (RGB lerp through gray = desaturated, HSV lerp preserves saturation = more vivid).

## Key Concepts

- **RGB (Red, Green, Blue)**: additive color model, each component [0,1]. (1,0,0) = red, (0,1,0) = green, (0,0,1) = blue, (1,1,1) = white, (0,0,0) = black. Blending: (r1,g1,b1) + (r2,g2,b2) = component-wise add. Multiply: (r1,g1,b1) × (r2,g2,b2) = modulation (darken). Used by: GPU rasterization, framebuffers, textures.
- **HSV (Hue, Saturation, Value)**: cylindrical color space, perceptual. Hue [0°,360°] = color wheel (0°=red, 120°=green, 240°=blue), Saturation [0,1] = color intensity (0=gray, 1=vivid), Value [0,1] = brightness (0=black, 1=full). Conversion: RGB → HSV (max=V, delta=max-min, H from ratios), HSV → RGB (sector-based piecewise). Used by: color pickers, artist tools, procedural generation.
- **Linear vs sRGB**: Linear RGB = physics (doubling value doubles brightness), sRGB = gamma 2.2 (nonlinear, matches human perception). Conversion: Linear → sRGB = pow(c, 1/2.2), sRGB → Linear = pow(c, 2.2). Rendering: compute in linear (correct light accumulation), display in sRGB (match monitor). Texture import: diffuse/albedo = sRGB, normal/roughness = linear.
- **Color Interpolation**: RGB lerp = lerp(c1, c2, t) = (1-t)×c1 + t×c2 (component-wise). HSV lerp = convert to HSV, lerp H/S/V separately, convert back = preserves hue (RGB lerp desaturates through gray). Lab lerp = perceptually uniform (equal distance = equal perceived difference). Used by: gradients, color grading, animation.
- **Luminance**: perceived brightness Y = 0.299×R + 0.587×G + 0.114×B (weighted sum, green contributes most to brightness). Used by: grayscale conversion, tone mapping, contrast calculations. Alternative: Rec.709 = 0.2126×R + 0.7152×G + 0.0722×B (HDTV standard).
- **Alpha Blending**: combine colors with transparency α. Premultiplied alpha: Cout = Csrc + (1-αsrc)×Cdst (source already multiplied by alpha). Straight alpha: Cout = αsrc×Csrc + (1-αsrc)×Cdst (multiply during blend). Premultiplied = faster (one multiply vs two), industry standard.

## Best Practices

**Color Space Selection**:
- Rendering: always linear RGB (correct light accumulation, additive blending works correctly).
- Display: convert to sRGB on output (match monitor gamma = correct perceived brightness).
- Textures: diffuse/albedo = sRGB import, normal/metallic/roughness = linear (data not color).
- Color picking: HSV for artists (intuitive = adjust hue/saturation independently).

**Gamma Correction**:
- Project setting: Color Space = Linear (Unity default = correct rendering).
- Framebuffer: use sRGB format (automatic conversion on write = GPU converts linear → sRGB).
- Avoid: manual pow(c, 2.2) in shader (slow, handled by framebuffer).

**Color Interpolation**:
- RGB lerp: fast, simple, but desaturates (gray midpoint between colors).
- HSV lerp: preserves saturation, better for UI gradients (convert to HSV, lerp, convert back).
- Lab lerp: perceptually uniform (expensive, rarely needed in real-time).

**Platform-Specific**:
- **PC/Console**: full sRGB support (framebuffer automatic conversion).
- **Mobile**: ensure sRGB framebuffer enabled (some older GPUs require explicit enable).
- **VR**: color space critical (incorrect gamma = eye strain), verify linear workflow.

## Common Pitfalls

**Blending in Wrong Space**: developer blends sRGB colors directly (lerp in sRGB = too dark midpoint). Symptom: gradients too dark (blending 0.5 sRGB ≈ 0.22 linear = appears dim). Solution: convert to linear, blend, convert back (or use linear framebuffer = automatic).

**Texture Import Wrong**: normal map imported as sRGB (values treated as colors, incorrect math). Symptom: lighting broken (normals wrong = weird specular). Solution: texture importer = disable sRGB for data textures (normals, masks, heightmaps).

**RGB Lerp Desaturation**: UI gradient from red to blue (RGB lerp goes through gray = ugly desaturated midpoint). Symptom: gradient muddy. Solution: HSV lerp (preserves saturation = vivid midpoint), or use color curve (artist-defined control points).

## Tools & Workflow

**RGB ↔ HSV Conversion** (C#):
```csharp
Color.RGBToHSV(rgbColor, out float H, out float S, out float V);
Color hsvColor = Color.HSVToRGB(H, S, V);
```

**Linear ↔ sRGB Conversion**:
```csharp
Color linear = color.gamma; // sRGB → Linear (Unity property)
Color srgb = color.linear;  // Linear → sRGB
// Manual: Mathf.Pow(c, 2.2f); // sRGB → Linear
```

**Color Interpolation**:
```csharp
// RGB lerp (fast, desaturates)
Color result = Color.Lerp(color1, color2, t);

// HSV lerp (preserves saturation)
Color.RGBToHSV(color1, out float h1, out float s1, out float v1);
Color.RGBToHSV(color2, out float h2, out float s2, out float v2);
float h = Mathf.Lerp(h1, h2, t);
float s = Mathf.Lerp(s1, s2, t);
float v = Mathf.Lerp(v1, v2, t);
Color result = Color.HSVToRGB(h, s, v);
```

**Luminance Calculation**:
```csharp
float luminance = 0.299f * color.r + 0.587f * color.g + 0.114f * color.b;
Color grayscale = new Color(luminance, luminance, luminance);
```

## Related Topics

- [22.3 Color Spaces](22-03-Color-Spaces.md) - Linear vs sRGB in rendering
- [22.4 Color Grading Workflows](22-04-Color-Grading-Workflows.md) - Color manipulation
- [30.1 Linear Algebra](30-01-Linear-Algebra.md) - Mathematical foundations
- [11.2 PBR Material Properties](11-02-PBR-Material-Properties.md) - Material colors

---

[Previous: 30.1 Linear Algebra](30-01-Linear-Algebra.md) | [Next: 30.3 Coordinate Systems →](30-03-Coordinate-Systems.md)
