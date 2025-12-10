# 13.3 Post-Processing

[← Back to Chapter 13](../chapters/13-Rendering-Techniques.md) | [Main Index](../README.md)

Post-processing applies screen-space effects after scene rendering: color grading, bloom, depth of field, motion blur, anti-aliasing, and tone mapping for cinematic presentation.

---

## Overview

Post-processing workflow: render scene to Render Texture (offscreen buffer), apply post-processing shaders (processes image pixel-by-pixel), output final image to screen. Each effect = full-screen pass (reads input texture, writes output texture). Multiple effects chain together (bloom reads rendered image, outputs bloomed image; color grading reads bloomed image, outputs graded image). Cost: fillrate (process every pixel per effect), bandwidth (read/write textures), and rendering passes (5-15 passes typical for full post-processing stack).

Common effects: Color Grading (adjust exposure, contrast, saturation, color curves, LUT-based grading), Bloom (bright areas glow, simulates camera lens effect), Depth of Field (blur background/foreground, focus on subject), Motion Blur (blur moving objects/camera, simulates shutter speed), Anti-Aliasing (smooth jagged edges: FXAA, TAA, SMAA), Ambient Occlusion (darken crevices, screen-space AO), and Tone Mapping (HDR to LDR conversion, filmic response curves).

Unity post-processing systems: Built-in (Post-Processing Stack v2, package-based), URP (Volume-based post-processing, integrated), HDRP (Volume system, high-end effects). Volumes define effect parameters (global volumes = entire scene, local volumes = region-specific like underwater tint). Performance: mobile uses minimal effects (FXAA, simple color grading), consoles use balanced stack (bloom, DOF, TAA, color grading), PC high-end uses full stack (all effects, high quality settings).

## Key Concepts

- **Color Grading**: Adjusts image colors post-render. Parameters: exposure (brightness), contrast, saturation, color curves (per-channel adjustments), white balance (temperature/tint). LUT-based (lookup texture, 3D color transform). Cinematic look (film-style color grading, mood).
- **Bloom**: Bright pixels glow/blur. Threshold (brightness cutoff, pixels above threshold bloom), intensity (glow strength), diffusion (blur size). Simulates camera lens glare (bright lights, sun, emissive materials). HDR-dependent (requires HDR rendering for proper bloom).
- **Depth of Field (DOF)**: Blurs out-of-focus areas. Focus distance (distance to focused plane), aperture (bokeh size, f-stop), focal length (lens characteristics). Gaussian DOF (simple blur), Bokeh DOF (physical lens simulation, hexagonal/circular bokeh shapes).
- **Motion Blur**: Blurs moving objects/camera. Camera motion blur (blur entire screen based on camera velocity), per-object motion blur (blur individual objects based on velocity). Simulates shutter speed (1/60s shutter = motion streaks). Expensive (requires velocity buffer, multiple samples).
- **Anti-Aliasing**: Smooths jagged edges. FXAA (fast, post-process filter, cheap but blurry), TAA (temporal AA, uses previous frames, sharp and effective), SMAA (subpixel morphological, quality balance). MSAA (hardware anti-aliasing, deferred rendering incompatible).

## Best Practices

**Effect Selection and Ordering:**
- Essential effects: Color Grading (always include, defines visual style), Bloom (common, cinematic feel), Anti-Aliasing (FXAA/TAA, smooth edges), Tone Mapping (HDR to LDR, required for HDR rendering). Nearly all games use these.
- Optional effects: DOF (focus attention, cinematic but motion sickness risk), Motion Blur (cinematic but polarizing, some players disable), Vignette (darkens screen edges, subtle framing), Film Grain (retro aesthetic, nostalgia effect).
- Mobile subset: Color Grading (LUT-based, cheap), FXAA (cheapest AA, single pass), optional bloom (low quality, small blur radius). Skip DOF, motion blur, expensive effects.
- Effect order: Unity handles automatically (Volume system orders effects optimally). Custom stacks: Tone Mapping first (HDR to LDR), then SSAO, bloom, DOF, motion blur, anti-aliasing, color grading last (final color adjustments).

**Performance Optimization:**
- Use LUT color grading: 3D LUT (32x32x32 or 64x64x64 texture) faster than per-pixel color math. Bake color curves/adjustments into LUT offline (DaVinci Resolve, Photoshop), apply via single texture sample (1 texture fetch vs 50+ math ops).
- Downsampled passes: Bloom, DOF render at half/quarter resolution (render bloom at 50% res, upscale to full res). 4x faster (1/4 pixels processed), minimal quality loss (blur hides low-res artifacts). Configure via effect quality settings.
- Skip unnecessary effects: Profile without post-processing, then add effects one-by-one. Measure GPU cost per effect (Profiler > GPU > Post-Processing). Disable expensive effects on low-end hardware (DOF, motion blur = 2-5ms each, skip on mobile).
- Uber shader: Unity combines multiple effects into single pass (reduces render target switches, bandwidth). Post-Processing Stack v2 and URP use uber shaders automatically (5-10 effects in 2-3 passes instead of 10 passes).

**Quality Settings per Platform:**
- PC Ultra: All effects high quality. Bloom (high iteration count, large blur), DOF (bokeh DOF, high sample count), TAA (high quality, 8-sample), full-res effects. 5-10ms GPU budget for post-processing.
- Console: Balanced quality. Bloom (medium iteration, moderate blur), DOF (Gaussian or low-quality bokeh), TAA (medium quality, 4-sample), some half-res effects. 3-5ms GPU budget.
- Switch: Minimal effects. Bloom (low quality, quarter-res), FXAA (no TAA, too expensive), color grading (LUT-based only), skip DOF/motion blur. <2ms GPU budget.
- Mobile: Essential only. Color grading (LUT), optional low-quality bloom (quarter-res, single iteration), FXAA, tone mapping. <1ms GPU budget. Aggressive optimization critical (60fps target).

**Volume Configuration:**
- Global volumes: Default settings for entire scene. Place at scene root (covers everything), configure base parameters (neutral color grading, subtle bloom). All areas use global volume unless overridden.
- Local volumes: Region-specific overrides. Examples: underwater volume (blue-green tint, increased contrast), cave volume (desaturated, vignette), sunset volume (warm temperature, high bloom). Box collider defines volume bounds (trigger = IsTriggering).
- Volume blending: Overlapping volumes blend smoothly (weight-based interpolation). Priority controls blend order (higher priority = stronger influence). Blend distance (transition zone between volumes, smooth color grading transitions).
- Profile assets: Shared volume settings (create Volume Profile asset, reference from multiple volumes). DRY principle (tune once, apply everywhere). Quality tiers: separate profiles per quality (Low/Medium/High profiles with different effect settings).

**Anti-Aliasing Selection:**
- FXAA: Fastest AA (single post-process pass, 0.2-0.5ms). Post-process filter (detects edges, smooths via smart blur). Blurs image slightly (less sharp than TAA/SMAA). Use for: mobile (only viable AA), low-end PC (performance priority), or stylized games (slight blur acceptable).
- TAA: Best quality (uses temporal information, previous frames). Sharp results, effective on thin edges (wires, foliage). Requires motion vectors (velocity buffer). Ghosting artifacts (moving objects trail). Use for: PC/console (quality priority), most modern games (industry standard).
- SMAA: Balanced option (better quality than FXAA, cheaper than TAA). Subpixel morphological anti-aliasing (edge detection + pattern matching). No temporal artifacts (no ghosting). Use for: games sensitive to ghosting (fast-paced, competitive), consoles (good quality-performance balance).
- MSAA: Hardware AA (not post-processing). Forward rendering only (incompatible with deferred). 2x/4x/8x samples per pixel (expensive, 2x/4x memory). Use for: VR (forward rendering required, MSAA effective), mobile (newer devices support MSAA efficiently), or simple forward renderers.

**Platform-Specific:**
- **PC**: Full post-processing stack. All effects enabled (bloom, DOF, motion blur, TAA, color grading, SSAO, tone mapping). Quality settings (Low/Med/High/Ultra) control effect parameters. Player choice (some disable motion blur, adjust bloom intensity).
- **Consoles**: Balanced stack. Bloom, TAA, color grading, tone mapping (essential). Optional DOF (cinematic scenes only, disable during gameplay for performance). Half-res SSAO (upscaled). Target 3-5ms GPU for post-processing.
- **Switch**: Minimal stack. Color grading (LUT), FXAA, tone mapping, optional low-quality bloom. No DOF, no motion blur, no TAA (too expensive). Target <2ms. Aggressive quality reduction (quarter-res bloom, 16x16x16 LUT).
- **Mobile**: Essential only. Color grading (LUT, 16x16x16 or 32x32x32), FXAA, tone mapping. Skip bloom on low-end devices (check SystemInfo.graphicsMemorySize, <2GB = no bloom). Target <1ms. Some games disable post-processing entirely (60fps priority).

## Common Pitfalls

**Excessive Bloom**: Developer sets Bloom Intensity = 5.0, Threshold = 0.1 (thinks "more bloom = prettier"). Entire screen glows (white haze, blown-out image, lost details). Symptom: Washed-out visuals, excessive glow, hard to see details. Solution: Reduce intensity (0.3-1.0 typical), increase threshold (1.0-2.0, only bright areas bloom). Subtle bloom better than obvious (enhance bright sources, don't overwhelm image).

**Motion Blur Causing Nausea**: Developer enables motion blur (cinematic effect) without option to disable. Players experience motion sickness (fast camera movement = heavy blur, nausea-inducing). Symptom: Player complaints about nausea, motion sickness, "why is game blurry?". Solution: Make motion blur optional (Settings > Motion Blur On/Off). Or disable entirely (polarizing effect, many players hate motion blur). Use sparingly (cutscenes only, not gameplay).

**Not Using LUT Color Grading**: Developer implements color grading via per-pixel math (curves, saturation, exposure, all in shader). 50-100 ALU instructions per pixel (expensive, 2-5ms GPU cost). Symptom: Profiler shows expensive color grading, GPU bound. Solution: Bake color grading into 3D LUT (32x32x32 texture). Apply LUT via single texture sample (<1ms). Tool workflow: adjust curves in DaVinci Resolve/Photoshop, export LUT, import to Unity.

**DOF Too Strong**: Developer sets DOF Aperture = f/1.4 (wide open, extreme blur). Background completely blurred (can't see environment, disorienting). Players miss important visual information (enemies, hazards behind blur). Symptom: Player complaints about not seeing environment, "too blurry". Solution: Reduce aperture (f/5.6-f/11, subtle blur). Use DOF sparingly (cutscenes, photo mode, not combat). Gameplay readability priority over cinematic effect.

## Tools & Workflow

**Post-Processing Stack v2**: Unity package (Built-in Pipeline). Window > Package Manager > Post Processing. Add Post-Process Volume (GameObject > Rendering > Post-process Volume), create Profile (new Post-process Profile asset), enable effects (Bloom, Color Grading, etc.).

**URP Post-Processing**: Integrated into URP. Create Volume (GameObject > Volume > Global Volume), add overrides (Inspector > Add Override > Post-processing), configure effects. Volume Profile asset stores settings.

**HDRP Volume System**: Similar to URP. Global/Local Volumes, Volume Profiles, effect overrides. Additional effects (lens distortion, film grain, panini projection, etc.). Higher quality implementations (better bloom, bokeh DOF).

**Volume Profile Editor**: Inspector shows Volume Profile (list of effects, parameters). Enable/disable effects (checkbox per effect), adjust parameters (sliders, colors, curves). Real-time preview (changes visible immediately in Scene/Game view).

**LUT Generator**: DaVinci Resolve (professional color grading), Adobe Photoshop (Camera Raw filter + LUT export), Unity-provided LUT generator (Post-Process > Utilities > Export LUT). Workflow: adjust color grading visually, export 3D LUT, import to Unity Post-Process Volume.

**Frame Debugger**: View post-processing passes. Shows render target switches (each effect = render pass), shader used per pass, texture inputs/outputs. Verify effect order (correct sequence), identify expensive passes (high resolution, complex shaders).

**Unity Profiler**: GPU Profiler > Rendering > Post-Processing shows per-effect cost. Identify expensive effects (DOF = 2-5ms, bloom = 1-3ms, color grading = 0.5ms). Optimize or disable bottleneck effects (balance quality vs performance).

**Rendering Debugger**: URP/HDRP: Window > Analysis > Rendering Debugger > Rendering > Post-processing. Disable effects temporarily (debug which effect causes issue), visualize intermediate textures (bloom mip chain, DOF COC map).

## Related Topics

- [13.1 Lighting Techniques](13-01-Lighting-Techniques.md) - Lighting for post-processing
- [12.4 Advanced Shader Techniques](12-04-Advanced-Shader-Techniques.md) - Screen-space effects
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Post-processing performance
- [1.5 Output Merger](01-05-Output-Merger.md) - Final composition

---

[← Previous: 13.2 Shadow Techniques](13-02-Shadow-Techniques.md) | [Next: 13.4 Transparency and Alpha →](13-04-Transparency-And-Alpha.md)
