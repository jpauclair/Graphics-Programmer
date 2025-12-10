# 22.1 HDR Rendering Pipeline

[← Back to Chapter 22](../chapters/22-HDR-And-Color-Management.md) | [Main Index](../README.md)

HDR (High Dynamic Range) rendering extends brightness range (SDR = 0-1 normalized, HDR = 0-10+), captures bright lights/dark shadows in single image. Pipeline: render to HDR framebuffer (accumulates high-range colors), apply tone mapping (compresses HDR to SDR for display), optionally post-process. Benefit: realistic lighting (bright sky vs dark interior captured accurately), bloom/glow effects, better post-processing (color grading in HDR space = better grades).

---

## Overview

LDR vs HDR: SDR (Standard Dynamic Range, 0-1 range): limited (pure white = 1.0, anything brighter clamps to white, lost detail). HDR: extended (pure white = 1.0, bright light = 2-100+, captures full range). Perceptual: human eye sees ~14 stops of dynamic range (brighter than sun to shadow), SDR = 8 stops, HDR = 12+ stops = more realistic. Implementation: standard framebuffer RGBA 8-bit (0-255 integer, becomes 0-1 float) = LDR, HDR framebuffer = RGBA 16-bit float (or 32-bit) = unlimited range above 1.0.

Tone Mapping: compresses HDR to SDR for display (monitor can't display >1.0 brightness, needs curve to map HDR range to 0-1). Methods: simple (clamp >1.0 to 1.0 = loses bright details), linear (divide by max brightness = flattens image), curve (S-curve emphasizes midtones, darkens shadows, brightens highlights = photorealistic), filmic (based on film photography = natural look). HDRP: built-in tone mappers (Neutral, ACES = industry standard), developer picks one. Cheap: tone mapping negligible overhead (<0.1ms), worth the visual improvement.

Pipeline Flow: render opaque (HDR colors accumulate), render transparent (blends in HDR space), apply lighting (all lights accumulate, can exceed 1.0 = bright), apply post-processing (bloom = detect bright pixels, blur, add to image = glow effect), apply tone mapping (compress to SDR), output to display.

## Key Concepts

- **HDR Framebuffer Formats**: Standard LDR = RGBA 8-bit (R, G, B, A each 0-255 = 0.0-1.0 when converted to float), no values >1.0 stored (clamped). HDR alternatives: RGBA 16-bit half-float (R16G16B16A16_SFLOAT, -65504 to +65504 range, stores >1.0 values precisely), RGBA 32-bit float (R32G32B32A32_FLOAT, unlimited precision, overkill = 2x memory). Unity: RenderTextureFormat.ARGBHalf (16-bit = good compromise, used by HDRP), costs 2x VRAM vs LDR but enables HDR lighting. Mobile: not all devices support half-float (older Android = integer only), fallback to 8-bit (loses HDR benefits but compatible).
- **Tone Mapping**: Math: maps [0, inf) to [0, 1] range for display. Simple: out = in / (in + 1) (fast, everyone brightness levels to same output), Linear: out = in / maxBrightness (scales entire range proportionally), S-Curve: out = pow(in, 2.2) (gamma curve, emphasizes midtones), ACES (Academy Color Encoding System): industry standard (based on film photography, natural look, prevents crushed blacks/blown whites). Choose: ACES for cinematic/console games, Neutral (simple curve) for stylized, Linear if artistic control needed (apply own curve).
- **Bloom Effect**: bright pixels glow (halo effect, makes bright lights look bright). Implementation: render scene to HDR, copy bright pixels (threshold = 1.0, pixels >1.0 = bright), blur (Gaussian blur = soft glow), add back to image. Result: bright light has glow halo, enhances perception of brightness (actually >1.0 displayed as 1.0 + glow = eye perceives as brighter). HDRP: built-in Bloom (post-process volume, Threshold = 1.0, Intensity multiplier, Scatter = bloom spread). Performance: 2-5ms (multiple downsampled passes, blur, upsample blend), optional (can disable for performance).
- **Exposure Control**: brightness adjustment (artist controls overall brightness, independent of tone mapping). Manual: Exposure parameter (multiply all colors by 2^exposure, exposure = 0 = no change, +1 = 2x brighter, -1 = half). Auto-exposure: samples scene brightness, adjusts exposure to maintain consistent look (simulates eye adaptation, bright outdoor = reduce exposure, dark indoor = increase exposure). HDRP: Exposure component (fixed or auto-exposure, Speed = how fast adaptation occurs, target brightness = 0.5 typical).
- **Post-Processing in HDR**: color grading (adjust contrast, saturation, color balance) applied in HDR space before tone mapping = better results (more color information = grading has more data to work with, applied before compression = no banding artifacts). LDR grading (post tone mapping) = grading 0-1 range only = less expressive. HDRP: Color Lookup Table (LUT), Lift-Gamma-Gain sliders, all applied in HDR.

## Best Practices

**HDR Setup:**
- Create HDR render target: RenderTexture (Assets > Create > Render Texture), Depth Buffer = 24-bit (standard), Color Format = ARGBHalf or ARGBFloat (HDR), Enable Random Write = true (compute shaders). Assign to camera (Camera > Output Texture = render texture, renders to HDR). Or: HDRP (automatic, renders to HDR internally, no manual RT setup).
- Light intensity scaling: in HDR, lights can exceed 1.0 (bright lights = higher intensity = realistic). Directional light (Intensity = 100000 lux = direct sunlight = very bright, tone mapping compresses), Point light (Intensity = 1000-10000 = indoor light), requires HDR aware (0-1 intensity looks too dark in HDR, scale up 100x).
- Material albedo: keep 0-1 range (albedo = surface reflectance, never >1.0, white = 1.0, reflection = albedo * light = still ≤1.0 for non-metallic). Metallic/specular = ≥1.0 = specular multiplier (bright reflections allowed, ~1.5-5.0 for polished metal).

**Tone Mapping Selection:**
- ACES (industry standard): use for any shipping game (cinematic, neutral, proven). Advantages: avoids crushing blacks/blown whites, natural curves based on film, universally recognized.
- Neutral (simple curve): acceptable alternative (less processing, slightly different look, fine for indie).
- Custom: if artistic control needed (write own tone mapping curve), rare (most games use ACES or Neutral).

**Bloom Configuration:**
- Threshold: typically 1.0 (pixels >1.0 = bright, glow), or slightly lower (0.5-1.0) to glow dimmer lights. Higher = only very bright lights glow (more selective), lower = more pixels glow (more glow overall).
- Intensity: multiplier for bloom contribution (1.0 = add full bloom, 2.0 = double brightness from bloom, control glow strength).
- Scatter: spread of bloom blur (0 = tight around bright pixel, 1 = wide halo, artistic choice).
- Performance: disable on console/mobile if needed (optional feature, 2-5ms savings on low-end).

**Platform-Specific:**
- **PC**: Full HDR support (16-bit half-float standard, tone mapping + bloom expected). Exposure 100000 lux sunlight = realistic.
- **Consoles**: PS5/Xbox Series X = HDR capable (Auto HDR if not coded, or native HDR rendering), PS4/One = no native HDR (SDR only, skip HDR rendering).
- **Switch**: No HDR support (SDR only, tone mapping not used, render LDR).
- **Mobile**: Inconsistent (newer phones = half-float support, older = integers only). Detect: SystemInfo.SupportsRenderTextureFormat(RenderTextureFormat.ARGBHalf), if unsupported, render LDR fallback.

## Common Pitfalls

**Lights Too Bright in HDR**: Developer enables HDR rendering (switches to 16-bit framebuffer), renders same scene, entire image blown out white (exposure too high for HDR). Symptom: scene completely overexposed, barely visible. Cause: light intensity not adjusted for HDR (0-1 scale in SDR becomes too bright in HDR, requires 100-1000x increase). Solution: scale light intensity (Point light 1.0 SDR = 100+ HDR, Directional 50000-100000 lux), or reduce exposure (Exposure = -2 to -3, scales down overall brightness).

**Tone Mapping Loss of Detail**: Developer renders HDR, applies tone mapping, bright areas lose detail (sky blown white, no cloud detail visible). Symptom: bright regions look flat, no texture detail in highlights. Cause: tone mapping curve too aggressive (compresses bright range too much), or threshold for bloom too high (bright area not marked as bloom, not preserved). Solution: adjust tone mapping curve (ACES usually better), or enable bloom (marks bright pixels, preserves them visually), or adjust exposure (reduce to keep more detail in range).

**Bloom Halo Artifacts**: Bloom effect creates halos around bright pixels (unintended glow spreading beyond object). Symptom: bright light has thick halo, looks unnatural (not like real glow). Cause: bloom scatter too high (blur radius too large, spreads too far), or threshold too low (too many pixels marked bright, increases halo). Solution: reduce Scatter parameter (tighter bloom, less spread), increase Threshold (fewer pixels glow, reduces halo), or adjust Intensity (less bloom contribution = fainter halo).

## Tools & Workflow

**HDR Render Texture Setup**: Assets > Create > Render Texture, Inspector > Color Format = ARGBHalf (16-bit float), Depth Buffer = 24. Assign to camera (Canvas Camera > Output Texture = render texture). Or HDRP (automatic, no setup needed).

**Tone Mapper Selection**: HDRP Volume Profile > Add Override > Tonemapping > Mode = ACES or Neutral. Fine-tune Exposure (slider, multiplier on colors before tone mapping, simulates camera exposure), Post-Exposure (multiplier after tone mapping, affects output brightness).

**Bloom Setup**: HDRP Volume Profile > Add Override > Bloom > Threshold (1.0-2.0, pixels >threshold glow), Intensity (1.0 default), Scatter (0.1-1.0, blur spread). Test: in-scene bright light should have glow halo.

**Light Intensity for HDR**: Directional Light > Intensity = 100000 (direct sunlight, very bright). Point Light > Intensity = 500 (interior light, moderate). In Inspector, these values seem huge vs SDR (0-10 range), but correct for HDR. Verify: render with Exposure adjusted to maintain visible detail (not all white).

**Profiler HDR Rendering**: Profiler > GPU Usage tab, check Bloom time if enabled (should be 2-5ms, if >10ms reduce resolution or disable). Tonemapping negligible (<0.1ms).

## Related Topics

- [22.2 HDR Display Output](22-02-HDR-Display-Output.md) - HDMI 2.1, HDR standards, console output
- [22.3 Color Spaces](22-03-Color-Spaces.md) - Linear vs sRGB, gamma correction
- [22.4 Color Grading Workflows](22-04-Color-Grading-Workflows.md) - LUT, grade in HDR
- [13.3 Post-Processing](13-03-Post-Processing.md) - Bloom, exposure, tone mapping effects

---

[Previous: Chapter 21](../chapters/21-Frame-Management-And-Synchronization.md) | [Next: 22.2 HDR Display Output →](22-02-HDR-Display-Output.md)
