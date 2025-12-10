# 22.2 HDR Display Output

[← Back to Chapter 22](../chapters/22-HDR-And-Color-Management.md) | [Main Index](../README.md)

HDR display output sends HDR data to monitor/TV (HDMI 2.1, DisplayPort 1.4), requires compatible display (HDR-capable TV/monitor). Standards: HDR10 (standard, 10-bit, tone mapping curve), Dolby Vision (premium, 12-bit, movie-standard), HLG (broadcast, 10-bit). Console support: PS5/Xbox Series X = native HDR output, PS4/One = no HDR (SDR only).

---

## Overview

HDR Standards: HDR10 (open, most common, 10-bit per channel = 1 billion colors + metadata specifying brightness range), Dolby Vision (premium, 12-bit per channel = 68 billion colors + dynamic per-frame metadata), HLG (Hybrid Log-Gamma, broadcast/streaming). PC: primarily HDR10 (industry standard, Windows 10/11 support, DirectX 12, Vulkan). Console: PS5 native HDR10, Xbox Series X native HDR10, older consoles (PS4/Xbox One) = SDR only, no native HDR (Windows can simulate via Auto HDR = takes SDR output, applies tone mapping = fake HDR, not true HDR).

Display Requirements: HDMI 2.1 (max 48 Gbps, supports 4K HDR 120 Hz, required for next-gen), HDMI 2.0 (18 Gbps, max 4K HDR 30 Hz, still common on older displays). PC monitor: DisplayPort 1.4 (48 Gbps), or HDMI 2.1 TV. Modern TV (last 5 years) = HDR capable, older monitors = SDR only. Detection: Windows Color Settings (Displays > HDR, toggles HDR on/off), device capabilities (query if HDR supported, don't force HDR on non-HDR displays).

Console Output: PS5 detects connected display (queries EDID = extended display identification data), reports HDR capability (supports HDR10 or SDR), game uses accordingly (renders HDR if supported, SDR fallback). Auto HDR: Windows feature (upconverts SDR game output to HDR via tone mapping + histogram, not true HDR but improves perception), available on Xbox Series X (can enable per game = applies even if game SDR).

## Key Concepts

- **HDR10**: Open standard (10-bit color, 1 billion colors possible, HDMI 2.1/DisplayPort 1.4), metadata (max brightness = 10000 nits, max frame-averaged brightness = 100-400 nits typical). Process: game renders HDR (tone mapping), outputs 10-bit per channel (tone mapped 0-1 range = 0-1023 10-bit values), sends metadata (peak brightness), display uses metadata to stretch tone mapping (applies its own tone mapping based on display's capability = bright display shows full range, dimmer display shows compressed range). Adoption: universal on modern displays, Windows 10+ support, PlayStation 5, Xbox Series X, Samsung/LG/TCL TVs.
- **Dolby Vision**: Premium standard (12-bit color = 68 billion colors, dynamic metadata per frame = each frame specifies its own tone mapping = better grading per-scene). Higher cost (licensing), adoption limited (Apple TV, Netflix, high-end displays = premium market). Gaming: rare (requires licensing, overhead, limited display support). Unlikely for indie games (cost-prohibitive).
- **HLG (Hybrid Log-Gamma)**: Broadcast/streaming standard (10-bit, image-based metadata = metadata describes image not scene, simpler than Dolby). Use case: TV broadcast, streaming (BBC, YouTube, Live sports). Gaming: not used (TV-centric).
- **Auto HDR (Windows/Xbox)**: Software feature (takes SDR game output, applies automatic tone mapping + histogram stretching), attempts to create HDR-like appearance. Implementation: monitor SDR output brightness histogram (where are bright pixels, where are dark), intelligently map to HDR range (stretches contrast = fake HDR effect). Effectiveness: improves perception (more contrast, brighter brights), but not true HDR (no real >1.0 values, just mapping 0-1 range cleverly). Pros: any SDR game instantly HDR on compatible display. Cons: not as good as native HDR (no real extended range, potential artifacts = halos, crushed blacks if Auto HDR fails).
- **Display Capabilities**: HDMI 2.0 (18 Gbps max = 4K 30Hz HDR, or 1440p 120Hz, or 1080p 240Hz), HDMI 2.1 (48 Gbps = 4K 120Hz HDR, future-proof). DisplayPort 1.4 (32 Gbps = 8K 30Hz HDR or 4K 120Hz), Thunderbolt 3 (40 Gbps, supports DisplayPort 1.4 bandwidth). Mobile: limited HDR (few devices support HDR display, Pixel 6+, some iPhones, high-end Samsung), most Android = SDR.
- **Console HDR Support**: PS5/Xbox Series X: native HDR rendering (game renders HDR, outputs 10-bit per HDMI 2.1), Auto HDR if supported (can apply to SDR game). PS4/Xbox One: SDR only (no native HDR support, Auto HDR available on Xbox One S/X via OS feature, not game responsibility). Compatibility: game checks hardware (if PS5/Series X, enable HDR, if PS4/One, SDR only), or always renders SDR (safe, works everywhere, skip HDR feature entirely).

## Best Practices

**HDR Output Configuration:**
- Console (PS5): automatic (detects display capability, renders HDR if supported), no game code needed (OS handles output). Verify: Settings > Video > HDR indicator shows "on" if HDR display connected.
- Console (Xbox Series X): similar automatic, game supports HDR via project settings (not manual output configuration).
- PC (Windows): enable HDR (Settings > Display > HDR > turn on HDR), app can query support (Windows.Graphics.Display.AdvancedColorInfo for HDR capability), explicitly opt-in to HDR rendering.
- Mobile: most devices SDR (render SDR by default, skip HDR feature), detect support (Android: Display.getHdrCapabilities(), iOS: no public HDR API), if rare HDR device detected, enable HDR mode.

**Platform-Specific HDR:**
- **PC Gaming**: Target HDR10 (universal standard), assume modern display with HDMI 2.1 or DisplayPort 1.4 (recommended hardware = RTX 2080+ / RX 5700+ = 2018+, most gaming PCs). Fallback SDR if HDR detection fails (graceful degradation).
- **Consoles**: PS5/Series X = native HDR assumed (all games should render HDR for these platforms), PS4/One = SDR only (older hardware, skip HDR feature).
- **Mobile**: SDR only (battery, display constraints, HDR rare). High-end devices (Galaxy S21+, Pixel 6+) = rare HDR support, opt-in if detected, otherwise SDR default.

**HDR Display Fallback:**
- Detect support: `SystemInfo.hdrDisplaySupportFlags` (Unity 2022+), or platform-specific queries (Windows: get HDR capability from settings). If no support, render SDR (tone mapping applied locally, SDR output).
- Graceful fallback: game always renders SDR if HDR unsupported, no error (user doesn't notice, just sees SDR quality).
- Recommendation: PC game = offer HDR toggle (Settings > Video > HDR enable/disable), console game = always enable if platform supports (transparent to user).

**TV Calibration (for HDR):**
- Brightness/Contrast: HDR displays auto-calibrate (EDID = metadata tells TV brightness range), game doesn't need manual calibration.
- HDMI color space: ensure HDMI > Color Space = "Auto" (TV detects), avoid forcing RGB 4:4:4 if not supported (causes artifacts).
- Game-specific: some AAA titles include calibration screen (adjust peak brightness for your TV), optional but recommended for best look (Peak Brightness = max display brightness in nits, game uses to scale tone mapping).

## Common Pitfalls

**HDR Output on Non-HDR Display**: Developer enables HDR rendering (renders to 16-bit float, tone mapping), outputs HDR data to non-HDR display (SDR monitor, older TV with HDMI 2.0 not reporting HDR). Display receives HDR metadata, can't interpret (doesn't support), falls back to SDR or shows artifacts (washed out, weird colors). Symptom: colors look off on some displays (test on HDR display, then SDR display = visible difference). Solution: detect display capability (query EDID/capabilities), render HDR if supported, SDR fallback if not.

**Auto HDR Confusion**: Developer ships SDR game on Xbox Series X (thinking Auto HDR = automatic HDR for free), marketing claims "HDR support" (misleading, Auto HDR is OS feature not game feature). Player connects non-HDR display, sees no HDR (Auto HDR only works with HDR display, fake HDR doesn't apply without HDR display hardware). Symptom: marketing claims not matching customer experience. Solution: clarify HDR support (if native HDR rendering, say so; if Auto HDR only, disclose as OS feature not game feature; if SDR only, don't claim HDR).

**Metadata Mismatch**: Developer renders HDR (bright sky = 10 nits average), outputs metadata (Peak = 1000 nits = assumes very bright scene), TV stretches display (applies aggressive tone mapping to show full 1000 nits range), sky becomes blown out white. Symptom: dynamic range too high (bright areas overexposed, loses detail). Solution: set metadata accurately (measure scene brightness = average of all pixels, set Peak to realistic value, not maximum possible).

## Tools & Workflow

**HDR Display Detection (PC)**: Use Windows APIs or Unity wrappers:
```csharp
if (SystemInfo.hdrDisplaySupportFlags != HdrDisplaySupportFlags.None) {
    // HDR display detected, enable HDR rendering
    enableHDR = true;
} else {
    // SDR only, render SDR
    enableHDR = false;
}
```

**PS5 HDR Setup**: Project Settings > Player > Resolution and Presentation > HDR = enabled (checkbox). PS5 SDK handles rest (detects display, outputs HDR if supported, SDR fallback if not).

**Xbox Series X HDR Setup**: Similar (project settings > enable HDR), Xbox SDK handles output.

**Windows Settings**: Settings > Display > Brightness and color > HDR > toggle on (enables HDR output to display, Windows reports support to applications).

**Profiler HDR Validation**: Profiler > GPU tab, Framebuffer format shows "ARGB Half" (16-bit = HDR), or render texture property in inspector (verify HDR format assigned).

## Related Topics

- [22.1 HDR Rendering Pipeline](22-01-HDR-Rendering-Pipeline.md) - Tone mapping, bloom, HDR rendering
- [22.3 Color Spaces](22-03-Color-Spaces.md) - Linear vs sRGB, color accuracy
- [13.3 Post-Processing](13-03-Post-Processing.md) - Exposure, color grading
- [18.3 High-Definition Render Pipeline](18-03-High-Definition-Render-Pipeline.md) - HDRP features

---

[Previous: 22.1 HDR Rendering Pipeline](22-01-HDR-Rendering-Pipeline.md) | [Next: 22.3 Color Spaces →](22-03-Color-Spaces.md)
