# 22.4 Color Grading Workflows

[← Back to Chapter 22](../chapters/22-HDR-And-Color-Management.md) | [Main Index](../README.md)

Color grading adjusts colors/contrast after rendering (tone mapping, global color adjustments, look-dev). Methods: LUT (Look-Up Table, 3D color mapping, fast, limited precision), Lift-Gamma-Gain (intuitive sliders), color curves (precise control), or shader (custom per-effect). Grading in HDR (before tone mapping = more color info) vs SDR (after tone mapping = limited range). Industry: DaVinci Resolve, Nuke for film; Photoshop, Lightroom for photos; game engines (Unity, Unreal) with in-engine grading.

---

## Overview

Color Grading Purpose: mood (dark blue = moody, warm orange = cozy), consistency (white balance corrections = skin tones look natural across scenes), stylization (cinematic LUT = film look = desaturated + warm). Applied: post-rendering (doesn't affect rendering, only appearance), before output (affects displayed image), global (entire screen) or localized (per-region). Workflow: render scene (neutral colors), apply grade (artist adjusts via LUT or sliders), output displayed image (graded colors).

LUT (Look-Up Table): 3D color mapping (input RGB → lookup table → output RGB), implemented as texture (64x64x64 voxels = 256KB, or 32x32x32 = smaller, lower quality). Fast: single texture read = negligible overhead (<0.01ms, GPU cache-friendly). Created: outside engine (DaVinci Resolve, Nuke exports .cube format = text file defining RGB mappings, imported to game engine = 3D texture). Standards: .cube format (portable, industry standard), Unreal UE4 format (proprietary).

Grading Spaces: SDR (grade 0-1 range post tone mapping, limited color information), HDR (grade extended range pre tone mapping, more colors available = better grades). HDRP: grade in HDR space (ACES + grade LUT + tone mapping = best quality). Disadvantage of SDR: banding risk (grading compressed 0-1 range, rounding errors = visible bands in gradients), limited headroom (can't enhance bright areas, already at 1.0), banding more visible in shadows (compression of limited SDR range).

## Key Concepts

- **LUT (Look-Up Table)**: 3D mapping (RGB input → RGB output via texture lookup). Structure: cubic texture (64 cubes per side = 64x64x64 = 256K voxels, or 32x32x32 = smaller), each voxel stores output color for input. Lookup: GPU samples texture at normalized input position (R, G, B coordinates 0-1 = position in cube, bilinear interpolation = smooth results). Speed: single texture read = <1 instruction on GPU, negligible time (<0.01ms per screen, cache-friendly). Quality: resolution trade-off (64 = accurate, 32 = coarser, 16 = very coarse/pixelated in color space). File format: .cube text file (lists output RGB for each input RGB in grid), or engine-native (Unity = 3D texture asset).
- **Lift-Gamma-Gain (LGG)**: Intuitive sliders for color grading (similar to photography editing). Lift: brighten shadows (add brightness to dark areas = lifts blacks), Gamma: adjust midtones (S-curve for contrast = brighten midtones or darken), Gain: brighten highlights (add brightness to bright areas = exposes). Alternative names: Shadows-Midtones-Highlights (Photoshop term), or Input/Output sliders. HDRP: built-in LGG (Volume component, sliders for each channel RGB or luminance).
- **Color Curves**: Precise per-channel control (X axis = input 0-1, Y axis = output 0-1, curve defines mapping). S-curve (shadow lift + highlight crush = contrast boost), linear (no change), inverted (negative), arbitrary (custom tone mapping). Tools: shader (custom curve per effect), or artist tools (DaVinci Resolve = complex curves, Photoshop = simple curves). Unity: support via shader (write curve equation or lookup texture), not built-in UI widget for curves (custom solution needed).
- **Grading in HDR vs SDR**: HDR grading (pre tone mapping = full color range available = shadows have colors, highlights detailed, grades preserve more information = less banding risk). SDR grading (post tone mapping = 0-1 range compressed = shadows darker, highlights limited, banding risk from compression). Workflow: HDR (render, grade, tone map, output), SDR (render, tone map, grade, output = not recommended). HDRP: grades in HDR by default (applies LUT before tone mapping), better quality.
- **Color Grading Volume**: HDRP component (scene object with override for grading parameters). Properties: Color Grading Mode (Graded = apply LUT, Ungraded = no LUT), LUT texture (3D texture asset), post-processing (Lift-Gamma-Gain), tone mapping mode. Blending: multiple volumes in scene (camera finds volumes at its position, blends between them = smooth transitions). Use case: interior (cool blue grade), exterior (warm orange grade), as camera moves between areas, grade interpolates = cinematic transitions.

## Best Practices

**LUT Grading Workflow:**
- External grading: render neutral scene (no grade, white balance neutral, slight desaturation), export renders (frame sequences), import to DaVinci Resolve (color correction tool, grade for mood/style), export LUT (.cube format).
- Game engine: import .cube into Unity (TextureImporter > texture shape = 3D, format = RGBA half or float for HDR), create Material with Color Grading shader (sample LUT texture using RGB as texture coordinates, apply to rendered image). Or HDRP: Volume > Color Grading > LUT (assign 3D texture, game applies automatically).
- Testing: render scene in-engine, apply LUT, verify colors match DaVinci preview (colors may shift due to grading space differences, iterate if needed).

**Lift-Gamma-Gain Adjustment:**
- Shadows (Lift): add brightness to dark areas (fix dark skin tones, prevent crushed blacks). Amount: small (0.05-0.15) typical, too much = looks fake.
- Midtones (Gamma): adjust contrast/saturation. Amount: ±0.1 typical (negative = darken midtones = lower contrast, positive = brighten = higher contrast).
- Highlights (Gain): brighten bright areas (expose details in sky). Amount: ±0.1 typical, too much = blown-out highlights.
- Per-channel: grade R, G, B separately (fix color casts = too much red in shadows, reduce red Lift), or luminance (brightness only).

**Grading Spaces (HDR Priority):**
- Always grade in HDR (pre tone mapping, before compressing to 0-1 range) = better color information = less banding = more control. HDRP: default (grades before tone mapping), correct approach.
- SDR grading acceptable for stylized games (banding less noticeable), but not recommended for photo-realistic (banding visible in gradients).

**Platform-Specific:**
- **PC**: Full LUT capability (64x64x64 = good quality, supporting full grading precision).
- **Consoles**: Same as PC (full support), 64x64x64 LUT standard.
- **Switch**: LUT viable (memory-efficient = 256KB for 64 cube), apply if needed (no performance impact = single texture read).
- **Mobile**: LUT viable (efficient), but consider file size (64x64x64 = 256KB, compress if possible). Alternative: simple LGG sliders (cheaper than LUT, less artistic control).

## Common Pitfalls

**LUT Color Mismatch**: Developer grades in DaVinci Resolve (linear color space), exports LUT, imports to game (SDR color space), colors look wrong (saturation different, tone mapping interaction unexpected). Symptom: game colors don't match DaVinci preview. Cause: grade space mismatch (DaVinci assumes linear, game might SDR = different color interpretation). Solution: ensure grading space matches (DaVinci linear, game linear before tone mapping = correct), or grade in-engine (HDRP Color Grading Volume, see in-game immediately).

**LUT Resolution Too Low**: Developer uses 16x16x16 LUT (small file size), banding visible in subtle color gradients (sky gradient shows bands = quality too low). Symptom: banding in color transitions (especially visible in sky, gradients look stepped). Solution: increase resolution (32x32x32 = acceptable quality, 64x64x64 = excellent, 16x16x16 = only acceptable for extremely stylized).

**Grading Applied After Compression**: Developer grades in SDR space (after tone mapping), limited color range available (can't enhance shadows = already compressed, highlights clipped at 1.0). Symptom: grade looks weak (can't adjust shadows/highlights much, limited control). Solution: grade in HDR (HDRP = before tone mapping), more expressive grades = better look.

## Tools & Workflow

**DaVinci Resolve LUT Export**: Open rendered frame sequence (File > Import > select frames), grade colors (Color tab, Lift/Gamma/Gain, saturation, contrast sliders), export LUT (File > Export > LUT > .cube format). Cube file text format (readable, lists output RGB for input grid).

**Unity LUT Import**: Drag .cube file to Assets folder, Unity imports (TextureImporter), set Texture Shape = 3D (Inspector), Format = RGBA Half (HDR quality) or RGBA (SDR). Apply to shader:
```glsl
texture3D _GradingLUT;
float4 Frag(v2f i) : SV_Target {
    float4 color = tex2D(_MainTex, i.uv);
    float4 graded = tex3D(_GradingLUT, color.rgb); // Sample 3D LUT
    return graded;
}
```

**HDRP Color Grading**: Volume component (GameObject > Volume > create Volume, Volume Profile > Add Override > Color Grading). Settings: Mode = Graded, Color Grading LUT = assign 3D texture. Additional: Lift-Gamma-Gain sliders (per-channel), Post-Exposure (global brightness), Hue Shift, Saturation. Blending: if multiple volumes, properties blend based on camera position.

**Photoshop Reference**: Grade reference image in Photoshop (Adjustment > Color Balance / Curves / Hue-Saturation), export, compare to game screenshot. Use Photoshop as inspiration (game shader replicates look).

**In-Engine Grading**: HDRP Volume + LUT (fastest iteration), no external tools. Or shader with parameters (Lift/Gamma/Gain sliders exposed to Inspector, adjust in real-time = immediate feedback).

## Related Topics

- [22.1 HDR Rendering Pipeline](22-01-HDR-Rendering-Pipeline.md) - Tone mapping, HDR rendering
- [22.2 HDR Display Output](22-02-HDR-Display-Output.md) - Output standards
- [22.3 Color Spaces](22-03-Color-Spaces.md) - Linear vs sRGB, gamma
- [13.3 Post-Processing](13-03-Post-Processing.md) - Post-process effects, color effects

---

[Previous: 22.3 Color Spaces](22-03-Color-Spaces.md) | [Next: Chapter 23 →](../chapters/23-Multithreading-And-Jobs-System.md)
