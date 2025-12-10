# 20.4 Sky and Atmospheric Rendering

[← Back to Chapter 20](../chapters/20-Specialized-Rendering.md) | [Main Index](../README.md)

Sky rendering creates atmospheric effects: skybox (6-sided cubemap or procedural), physically-based atmosphere (Rayleigh/Mie scattering), volumetric clouds, and time-of-day systems.

---

## Overview

Skybox Methods: Cubemap skybox (6 textures forming cube, cheapest = <0.1ms, static), Procedural skybox (shader generates sky colors, sunrise/sunset gradients, dynamic but limited), Physically-Based Sky (HDRP feature, atmospheric scattering simulation, Rayleigh + Mie scattering, photorealistic = expensive). Unity default: context.DrawSkybox(camera) renders active skybox (set per scene in Lighting window, or per-camera override). IBL integration (skybox provides ambient lighting, Image-Based Lighting for PBR materials, reflection probes sample skybox).

Atmospheric Scattering: Sunlight passing through atmosphere (short wavelengths scattered more = blue sky, longer wavelengths penetrate = red sunset). Rayleigh scattering (gas molecules, causes blue sky), Mie scattering (particles/aerosols, causes haze/fog, sun halos). Implementation: complex (multiple scattering integrals, view/sun angle calculations, lookup tables), HDRP Physically-Based Sky handles automatically (artist-friendly parameters: planet radius, atmospheric density, ground tint). Custom: simplified models (approximate with gradients, analytical formulas, cheaper than full simulation).

Clouds: 2D skybox clouds (painted into cubemap, static, free), billboarded cloud planes (quads with cloud textures, layered at different heights, scrolling = animated), volumetric clouds (3D noise-based, raymarched, realistic but expensive = 5-10ms). Unity: VFX Graph for cloud particles (thousands of cloud sprites, GPU-driven = dense cloud layers), or HDRP volumetric clouds (built-in, high-quality, expensive). Mobile: static skybox only (no dynamic clouds, pre-baked, <0.1ms).

## Key Concepts

- **Cubemap Skybox**: Six textures (front/back/left/right/up/down) mapped to cube surrounding scene. Unity Material: Skybox/Cubemap shader (assign 6 textures or single cubemap asset). Rendering: drawn at infinite distance (depth Z = far plane, behind all geometry), spherical projection (samples cubemap based on view direction, seamless 360° environment). Creation: capture real-world (360° camera, HDR photos, convert to cubemap), or DCC render (Blender/Maya renders environment, 6 views or equirectangular, import as cubemap). Resolution: 1024x1024 per face typical (6MB compressed), 2048 = high quality (25MB), 512 = low quality (1.5MB mobile).
- **Procedural Skybox**: Shader generates sky colors (gradients, sun/moon, stars). Unity shaders: Skybox/Procedural (simple gradient, sun disc, customizable), or custom (write shader, procedural noise for clouds, dynamic colors). Parameters: Sky Tint (zenith color, blue), Horizon Color (horizon tint, orange at sunset), Sun Size/Convergence (sun disc appearance), Atmosphere Thickness (haze amount). Benefits: tiny memory (<1KB vs megabytes for cubemap), dynamic (change colors at runtime, time-of-day via parameters), infinite detail (no texture resolution limits). Drawbacks: limited realism (simple gradients, not photographic), no clouds (unless shader complex), harder to art-direct than painted cubemap.
- **Physically-Based Sky (HDRP)**: Simulates atmospheric scattering (physically accurate sky colors based on sun position). Parameters: Planet Radius (6378 km = Earth), Atmospheric Height (scales atmosphere thickness, affects scattering), Air/Aerosol Density (Rayleigh/Mie scattering strengths, controls blue sky intensity and haze), Ground Tint (terrain color reflected into atmosphere), Space Emission (stars/galaxy background multiplier). Lighting: Directional Light acts as sun (intensity, color, position drive sky appearance, automatic updates = sunrise/sunset transitions). Dynamic: fully real-time (rotate sun, sky transitions dawn → day → dusk → night, no baking). Cost: moderate (precomputed lookup tables, real-time evaluation = 0.5-2ms, acceptable for PC/console), not available in URP/Built-In (HDRP exclusive).
- **Volumetric Clouds**: 3D clouds (noise-based density field, raymarched to render). Implementation: 3D noise textures (Worley/Perlin, defines cloud density per voxel), raymarch shader (for each pixel, march ray through cloud volume, accumulate density, lighting from sun). HDRP: built-in volumetric clouds (Cloud Layer or Volumetric Clouds Volume override, artist-friendly parameters: altitude, density, coverage, scattering), expensive (5-10ms, multiple scattering passes, resolution options for quality/performance trade-off). Custom: simpler (single-scattering, cheaper raymarch, 2-5ms, or stylized clouds = no scattering, just density). Mobile: not viable (too expensive, use billboarded clouds or baked into skybox).
- **Time-of-Day System**: Animate sun/sky over time (rotates Directional Light, updates sky colors/fog, simulates day/night cycle). Script: rotates sun (Light.transform.Rotate), updates sky parameters (procedural skybox colors change, or HDRP sky intensity), adjusts ambient lighting (RenderSettings.ambientLight color, darker at night). Baked lighting: incompatible with dynamic TOD (lightmaps baked for single sun position, TOD changes lighting = breaks baked shadows). Realtime lighting required (all lights realtime, shadows realtime, or mixed lighting = limited TOD, shadowmask bakes shadows but allows sun rotation). Performance: realtime lighting expensive (dynamic shadows, GI updates, target 30 FPS for dynamic TOD, or 60 FPS with baked lighting + limited sun movement).
- **Stars and Celestial Bodies**: Stars (procedural shader dots, or star cubemap layer), Moon (textured quad or mesh sphere, positioned opposite sun, phases via UV scrolling or shader), Milky Way (cubemap texture layer, visible at night). Unity: layer multiple skyboxes (stars cubemap + cloud layer + procedural sky, blended based on time-of-day), or single shader (combines all elements, procedural stars + moon + sky gradient). Rotation: stars rotate slowly (simulate Earth rotation, skybox rotation parameter or camera rotation), moon/sun orbit (scripted movement, elliptical paths for seasons).

## Best Practices

**Cubemap Skybox Setup:**
- Texture source: HDR images preferred (HDRI Haven, Poly Haven, HDR captures, .exr or .hdr format), SDR acceptable (LDR .jpg, lower quality reflections, acceptable for stylized). Import: Unity detects panoramic images (equirectangular), Inspector > Texture Shape = Cube, Convolution Type = Specular/Diffuse (for IBL), Apply. Or: import six individual textures (Cubemap > Create > Legacy Cubemap, assign 6 faces manually, outdated method).
- Material: Create material (Assets > Create > Material), Shader = Skybox/Cubemap, assign cubemap texture (Cubemap property). Settings: Exposure (HDR brightness, 1.0 default, increase for brighter sky), Rotation (rotates skybox, animate for moving clouds on static cubemap), Tint (colorize skybox, white = no tint).
- Assignment: Lighting window (Window > Rendering > Lighting > Environment > Skybox Material = your skybox material, applies scene-wide), or per-camera (Camera component > Environment > Skybox = override per camera, useful for reflection cameras, UI cameras). Ambient lighting: Lighting > Environment Lighting > Source = Skybox (sky provides ambient, PBR materials receive skybox lighting).

**Procedural Skybox:**
- Shader choice: Skybox/Procedural (Unity built-in, simple gradient + sun), or custom (write shader, advanced effects: stars, clouds, auroras). Material: create material, assign shader, configure parameters (Sky Tint, Horizon Color, Sun parameters).
- Time-of-day integration: Script animates parameters (lerp Sky Tint from blue (day) to orange (sunset) to black (night), Sun Size changes (larger at horizon = atmospheric effect), Atmosphere Thickness increases at sunset = red/orange glow). Code: `skyboxMaterial.SetColor("_SkyTint", Color.Lerp(dayColor, sunsetColor, sunsetAmount));`
- Directional Light: synchronize sun position with Light rotation (Light.transform.eulerAngles = new Vector3(sunAngle, 0, 0), sunAngle = 0 sunrise, 90 noon, 180 sunset). Skybox sun direction (Material property _SunDir = -Light.forward, skybox shader renders sun disc at light direction).

**Physically-Based Sky (HDRP):**
- Setup: Volume component (GameObject > Volume > Global Volume), Volume Profile > Add Override > Sky > Physically Based Sky. Parameters: default values (Earth-like, 6378 km radius, standard atmospheric density), Ground Tint (sample terrain color, brownish for desert, greenish for grass), Space Emission Multiplier (0 = no stars, 1 = visible stars at night).
- Directional Light: configure as sun (Intensity = 100,000-130,000 lux for daylight, Color = white/yellowish, Temperature mode = 6500K daylight). Light affects sky (rotation changes sky color, intensity changes brightness, automatic atmospheric scattering updates). Multiple lights (only one light affects sky, set as "Sun Source" in HDRP settings, others act as additional lights).
- Time-of-day: animate Directional Light rotation (0° = sunrise, 90° = noon, 180° = sunset, 270° = night), sky updates automatically (atmospheric scattering recalculates, dawn/dusk colors appear naturally). Ambient: sky provides GI (baked lightmaps sample sky, dynamic objects use sky for ambient via Light Probes).

**Volumetric Clouds (HDRP):**
- Simple clouds: Cloud Layer (HDRP Volume override, 2D cloud layer at fixed altitude, fast = 1-2ms). Parameters: Layer Altitude (1000-5000m, height above ground), Density (opacity, 0-1), Opacity (multiplier for transparency). Texture: cloud map (grayscale texture, white = clouds, black = sky, scrolls over time for animation).
- Advanced clouds: Volumetric Clouds (HDRP Volume override, 3D raymarched clouds, expensive = 5-10ms). Parameters: Bottom/Top Altitude (cloud layer height range, 2000-8000m typical), Density Multiplier (overall cloud thickness), Wind Speed (cloud scrolling), Lighting (scattering multiplier, sun penetration). Quality: resolution (quarter-res = fast, half-res = balanced, full-res = high quality + expensive), accumulation (temporal reprojection, smooths clouds over frames, reduces noise).
- Performance: lower resolution (volumetric clouds at quarter-res = 4x faster, acceptable quality), reduce march steps (fewer samples = faster + noisier, 64-128 steps typical, reduce to 32 on console), or disable on low settings (Cloud Layer only for Medium settings, no clouds on Low).

**Time-of-Day Script:**
- Sun rotation: Script updates Directional Light rotation (Light.transform.Rotate(sunSpeed * Time.deltaTime, 0, 0, Space.World), or set directly: Light.transform.eulerAngles = new Vector3(timeOfDay * 360 / 24, 0, 0)). Time variable (float timeOfDay = 0-24, increments over time, wraps at 24, 0 = midnight, 12 = noon).
- Sky updates: Procedural skybox (lerp colors based on sun angle, sunrise/sunset = orange/red, noon = blue, night = black), HDRP (automatic, sky reacts to sun position, no manual updates needed). Fog: lerp fog color (RenderSettings.fogColor, daytime = light blue, sunset = orange, night = dark blue/black), density (thicker at dawn/dusk, lighter at noon).
- Ambient lighting: RenderSettings.ambientLight (color changes with TOD, bright at noon, dark at night), or RenderSettings.ambientIntensity (multiplier, 1.0 day, 0.1 night). Baked lightmaps (incompatible, lightmaps static for single sun position, dynamic TOD requires realtime lighting or accept mismatch).
- Performance: realtime shadows expensive (sun shadow cascades recalculate, 3-5ms, acceptable for 30 FPS TOD), disable shadows at night (sun below horizon, shadows off = saves 3ms), or baked + shadowmask (baked static shadows, realtime dynamic objects only, compromise).

**Platform-Specific:**
- **PC**: Full sky features (HDR cubemap skybox, HDRP Physically-Based Sky, volumetric clouds, dynamic TOD with realtime shadows). Quality: high-resolution skybox (2048 cubemap, HDR), volumetric clouds half-res (acceptable performance 5-10ms). IBL: high-quality reflection probes (512-1024 resolution, multiple probes for varied environments).
- **Consoles**: Moderate sky (HDR cubemap, or HDRP Physically-Based Sky on PS5/Series X, Cloud Layer instead of volumetric), dynamic TOD viable (realtime shadows acceptable at 30 FPS). Quality: 1024 cubemap (compressed, 5-10MB), Cloud Layer only (volumetric clouds too expensive on PS4/Xbox One, viable on PS5/Series X at quarter-res).
- **Switch**: Simple sky (LDR cubemap 512-1024, compressed, or procedural skybox), static (no TOD or limited = rotate sun but baked lighting, dynamic lighting too expensive). No volumetric (Cloud Layer too slow, clouds baked into skybox or billboarded sprites). IBL: low-res reflection probes (128-256, minimal memory).
- **Mobile**: Static skybox only (LDR cubemap 512, heavily compressed <1MB, or single texture gradient), no procedural (shader cost unjustified, static cheaper). No clouds (baked into skybox, or none), no dynamic TOD (static lighting, sun fixed position). IBL: single ambient probe (no per-object reflections, global ambient only).

## Common Pitfalls

**Skybox Seams Visible**: Developer creates cubemap skybox (6 individual textures), seams appear at cube edges (textures don't align, visible lines where faces meet). Symptom: visible lines in sky (horizontal/vertical seams, especially noticeable with clouds or sun crossing edges). Solution: Use seamless cubemap (import equirectangular panorama, Unity converts to seamless cube), or fix seams in DCC tool (Blender cubemap export with edge padding, Photoshop clone stamp across edges), or increase texture resolution (seams less noticeable at higher res, 2048 vs 1024), Anisotropic filtering (enable on skybox texture, smooths edges).

**HDR Skybox Too Bright**: Developer imports HDR skybox (HDRI with 10-100+ lum intensity, thinking realistic), entire scene overexposed (skybox =  sun intensity, over-brightens ambient lighting, materials washed out). Symptom: Scene too bright (everything overlit, loss of contrast, bloom overwhelms screen). Solution: Reduce skybox Exposure (Material > Exposure = 0.5-0.7, scales HDR brightness down), or adjust camera Exposure (Camera > Post-Processing > Exposure = compensate for bright skybox), or use Tonemapping (ACES/Neutral tonemapping, compresses HDR range to SDR, prevents overexposure).

**Physically-Based Sky at Night Stays Bright**: Developer implements TOD (rotates sun 0-360°), sky never gets dark (sun at 270° = midnight, but sky still blue, no night appearance). HDRP Physically-Based Sky simulates atmosphere (even at night, faint scattering remains, not pure black). Symptom: Night sky too bright (should be black/dark blue, remains light blue, stars invisible). Solution: Disable sky at night (when sun below horizon, switch to black cubemap or disable sky rendering), or blend skyboxes (HDRP Volume blending, interpolate Physically-Based Sky weight to 0 at night, interpolate black skybox weight to 1), or adjust Space Emission (increase Space Emission Multiplier at night, makes stars brighter, compensates for residual sky brightness).

**Volumetric Clouds Performance Collapse**: Developer enables HDRP Volumetric Clouds (full-res, thinking quality needed), frame rate drops to 15 FPS (clouds = 30ms per frame, raymarching expensive, full-res unplayable). Symptom: GPU bottleneck (Profiler shows Volumetric Clouds pass dominating GPU time, 20-50ms), clouds beautiful but game unplayable. Solution: Lower resolution (quarter-res clouds = 4-8x faster, acceptable visual quality, clouds far away = resolution less noticeable), reduce march distance (Max Cloud Distance = 20km instead of 50km, fewer raymarch steps), disable on Low quality (use Cloud Layer instead, or static skybox clouds), or remove clouds entirely (many games have clear skies, clouds optional feature, not mandatory).

## Tools & Workflow

**Skybox Material Creation**: Assets > Create > Material (create material), Inspector > Shader dropdown > Skybox > Cubemap (or Procedural, Panoramic, 6-sided). Cubemap: assign texture (Cubemap slot, drag cubemap asset), adjust Exposure (1.0 default, <1.0 darker, >1.0 brighter), Rotation (0-360°, rotates skybox, animate for moving sky). Procedural: configure Sky Tint (zenith color), Horizon Color, Atmosphere Thickness, Sun Size (disc size), Sun Convergence (disc sharpness).

**Lighting Window Assignment**: Window > Rendering > Lighting > Environment tab. Skybox Material (drag skybox material, applies to entire scene), Sun Source (Directional Light acting as sun, affects procedural skybox sun position and Physically-Based Sky), Environment Lighting > Source = Skybox (sky provides ambient), Ambient Intensity (multiplier, 1.0 = full sky contribution, 0 = black ambient). Environment Reflections > Source = Skybox (materials sample skybox for reflections), Resolution (128/256/512/1024, reflection probe resolution, higher = sharper reflections in materials).

**HDRP Sky Setup**: Global Volume (GameObject > Volume > Global Volume if not exists), Volume Profile > Add Override > Sky > Sky Type = Physically Based Sky (or HDRI Sky for cubemap in HDRP, or Gradient Sky for simple). Physically Based Sky parameters: Air Maximum Altitude (default 42km, atmosphere height), Aerosol Maximum Altitude (8km, haze layer), Ground Tint (brownish, controls terrain reflection into sky). Fog: Add Override > Fog > enable Volumetric Fog (optional, atmospheric fog, integrates with sky).

**Volumetric Clouds**: HDRP Volume Profile > Add Override > Sky > Cloud Layer (simple 2D clouds, fast), or Add Override > Sky > Volumetric Clouds (3D raymarched, expensive). Cloud Layer: Cloud Map (grayscale texture, cloud density), Opacity Multiplier (0-1, transparency), Layer Altitude (fixed height, 3000m typical), Wind Speed (scrolls texture). Volumetric Clouds: Bottom/Top Altitude (2000-8000m, cloud height range), Density Multiplier (thickness), Resolution (Quarter/Half/Full, performance vs quality), Lighting > Scattering Tint (sun color penetration).

**Time-of-Day Script Example**:
```csharp
public Light sun;
public float dayDurationSeconds = 120; // 2 minutes = 24 hours
float timeOfDay = 6; // Start at 6 AM

void Update()
{
    timeOfDay += (24f / dayDurationSeconds) * Time.deltaTime;
    if (timeOfDay >= 24) timeOfDay -= 24;
    
    float sunAngle = (timeOfDay / 24f) * 360f - 90f; // 0 = sunrise, 90 = noon
    sun.transform.rotation = Quaternion.Euler(sunAngle, 0, 0);
    
    // Update ambient
    if (sunAngle > 0 && sunAngle < 180) // Daytime
        RenderSettings.ambientIntensity = Mathf.Lerp(0.3f, 1.0f, Mathf.Sin(sunAngle * Mathf.Deg2Rad));
    else // Night
        RenderSettings.ambientIntensity = 0.1f;
}
```

**Cubemap Import**: Import HDR image (drag .hdr or .exr into Assets), Inspector > Texture Shape = Cube (converts to cubemap), Mapping = Latitude-Longitude Layout (for equirectangular), Convolution Type = Specular (for reflections) or None (for skybox only), Max Size (1024/2048, resolution), Compression (None for HDR quality, or compressed for smaller size). Generate: Apply button (Unity processes cubemap, generates mip levels). Use: assign to Skybox Material, or Reflection Probe > Cubemap.

**Profiling Sky**: Unity Profiler > GPU: shows sky rendering cost (DrawSkybox = typically <1ms for cubemap, 0.5-2ms for Physically-Based Sky, 5-10ms for volumetric clouds). Frame Debugger: shows sky pass (rendered after opaque, before transparent, uses camera far plane depth = infinite distance). Optimize: reduce volumetric cloud resolution (quarter-res), simplify procedural skybox shader (remove expensive operations, analytical formulas), or use static cubemap (cheapest, <0.1ms).

## Related Topics

- [13.1 Lighting Fundamentals](13-01-Lighting-Techniques.md) - Sky as ambient lighting
- [18.3 High-Definition Render Pipeline](18-03-High-Definition-Render-Pipeline.md) - HDRP sky features
- [20.3 Water and Fluid Rendering](20-03-Water-And-Fluid-Rendering.md) - Sky reflections in water
- [13.3 Post-Processing](13-03-Post-Processing.md) - Atmosphere post-processing

---

[← Previous: 20.3 Water and Fluid Rendering](20-03-Water-And-Fluid-Rendering.md) | [Next: 20.5 UI Rendering →](20-05-UI-Rendering.md)
