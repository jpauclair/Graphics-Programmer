# 16.3 Camera Effects

[← Back to Chapter 16](../chapters/16-Camera-Systems.md) | [Main Index](../README.md)

Camera effects enhance rendering through screen shake, depth of field, motion blur, and camera-specific post-processing to create cinematic visuals and feedback.

---

## Overview

Camera effects add visual polish: screen shake (impacts/explosions feedback), depth of field (focus on subject, blur background), motion blur (fast movement trails), vignette (darkens edges, focuses attention), chromatic aberration (lens distortion, color fringing). Applied per-camera: main camera gets full effects, UI camera skips effects (UI stays sharp). Post-processing stack (Unity Post Processing or URP/HDRP volumes) manages effects: configure per-camera, enable/disable dynamically, profile-based quality settings.

Screen shake: camera position/rotation animated (small random offsets, decaying over time). Triggered by events (explosions nearby, player damage, heavy impacts). Implementation: add offset to camera transform (Perlin noise or random, magnitude decreases over time), or camera rig (offset child camera, parent stays stable for raycasts). Intensity scales with proximity (close explosion = large shake, distant = subtle). Duration 0.1-1.0 seconds typical (brief shake, returns to stable).

Depth of field (DOF): blurs out-of-focus areas (background/foreground blurred, focal plane sharp). Cinematic effect (mimics real camera lens, focuses attention on subject). Parameters: focal distance (distance to focus plane, meters), aperture (blur amount, f-stop equivalent, lower = more blur), focal length (lens zoom, affects blur falloff). Performance: expensive (multiple blur passes, depth-based masking), optimize via lower resolution (half-res DOF, upscale), or simpler algorithms (bokeh vs simple Gaussian).

## Key Concepts

- **Screen Shake**: Camera position offset (small random displacement). Typical: 0.1-0.5 unit shake (subtle to moderate), exponential decay (trauma value decreases over time, shake magnitude = trauma²). Perlin noise (smooth random motion, more natural than pure random). Implementation: `shakeMagnitude = trauma * trauma; offset = PerlinNoise(time) * shakeMagnitude; camera.position += offset;` Resets to zero over 0.5-1 seconds.
- **Depth of Field (DOF)**: Post-process effect (blurs based on depth buffer). Focal plane (distance from camera, objects at this depth sharp), near/far blur (objects closer/farther blurred). Circle of confusion (CoC, blur size per pixel, calculated from depth vs focal distance). Bokeh (hexagonal/circular blur shapes, expensive but cinematic). Unity: Post-Processing DOF (depth-based blur, configurable focal distance/aperture).
- **Motion Blur**: Blurs fast-moving objects (camera or object motion). Camera motion blur (whole screen blurs when camera moves quickly, based on camera velocity), per-object motion blur (individual objects blur based on movement, requires motion vectors). Implementation: accumulate previous frames (weighted blend, trail effect), or velocity-based blur (sample motion vectors, directional blur). Expensive: multiple texture samples per pixel (5-20 samples typical).
- **Post-Processing Volume**: Unity URP/HDRP (Volume component, contains effect overrides). Global volume (affects all cameras, scene-wide settings), local volume (blends based on camera position, location-specific effects). Per-camera: camera can enable/disable volume layer mask (selective post-processing). Priority system (higher priority overrides lower, allows layered effects).
- **Camera-Specific Effects**: Different effects per camera (main camera = full post-processing, minimap camera = no effects, UI camera = no effects). Configure: disable post-processing layer on specific cameras (Camera.allowMSAA = false, Camera component > Rendering > Post Processing checkbox), or Volume layer masks (camera only sees specific volume layers). Optimize: expensive effects only on main camera (DOF/motion blur skip on minimap).

## Best Practices

**Screen Shake Implementation:**
- Trauma-based shake: Trauma value 0-1 (1 = max shake, 0 = stable), decays over time (trauma -= decayRate * deltaTime). Shake magnitude = trauma² or trauma³ (non-linear, sharp decrease as trauma reduces). Add trauma on events (explosion adds 0.8 trauma, gunshot adds 0.2 trauma). Example code: `trauma = Mathf.Max(0, trauma - decaySpeed * Time.deltaTime); float shake = trauma * trauma; offset = new Vector3(Perlin(time), Perlin(time+1), 0) * shake * maxShake;`
- Directional shake: Offset toward/away from impact (explosion direction influences shake direction). Calculate vector from impact to camera (normalize), add directional offset (offset += impactDir * shakeMagnitude). More realistic (shake moves away from explosion source, not random all directions).
- Camera rig: Parent camera object (empty GameObject parent, camera as child). Apply shake to parent (position/rotation offset), camera child inherits. Benefits: raycasts use parent position (stable, not shaking), camera shake doesn't affect gameplay (physics queries unaffected). Reset: parent returns to origin (0,0,0 local position when shake ends).
- Magnitude scaling: Scale by distance (close explosions = strong shake, far = weak). Formula: `shakeMagnitude = baseShake * (1 / (1 + distance²))` (inverse-square falloff). Clamp minimum distance (avoid division issues, min distance = 1m). Also scale by damage (small explosion = 0.3 trauma, large = 1.0 trauma full shake).

**Depth of Field Configuration:**
- Auto-focus: Raycast from camera center (hit distance = focal distance). Dynamic focus (focus follows center object, e.g., player character always in focus, background blurs). Update per frame (or every few frames for performance). Code: `RaycastHit hit; if (Physics.Raycast(camera.position, camera.forward, out hit, maxDistance)) { dofFocalDistance = hit.distance; }`
- Manual focus: Artist-controlled focal distance (scripted or animated focal plane). Use for: cinematic sequences (focus shifts to specific objects, dramatic effect), dialogue scenes (focus on speaker), gameplay emphasis (highlight specific object by focusing on it, blur surroundings). Lerp focal distance (smooth transitions, DOF.focalDistance = Mathf.Lerp(current, target, time)).
- Aperture tuning: Lower f-stop = more blur (f/1.4 = very blurry background, f/8 = moderate, f/16 = minimal blur). Typical: f/2.8-f/5.6 (noticeable blur, not excessive). Mobile: disable DOF (too expensive), or use simplified (single-pass blur, no bokeh). PC/console: full bokeh DOF (hexagonal/circular shapes, cinematic quality).
- Performance: DOF expensive (5-10ms on mid-range GPUs). Optimize: half-resolution DOF (render DOF at 50% resolution, upscale, 4x faster + imperceptible quality loss), disable on low settings (quality tier = Low disables DOF, Medium+ enables), limit blur radius (fewer samples, less expensive). Profile: GPU Profiler shows DOF pass time (target <5ms).

**Motion Blur Setup:**
- Camera motion blur only: Whole-screen blur based on camera movement (rotation/position velocity). Cheaper than per-object (single pass, screen-space effect). Unity Post-Processing: Motion Blur effect (shutter angle controls intensity, 0-360°, 180° typical = half-frame blur). Use for: fast camera movement (running, flying, vehicle chase), first-person games (camera motion dominant).
- Per-object motion blur: Requires motion vectors (velocity buffer, records per-pixel velocity). Expensive: additional render pass (motion vector generation), complex blur (direction varies per pixel). Unity: enable motion vectors (Camera > Allow Dynamic Resolution, motion vectors auto-generated for moving objects). Use for: third-person games (character animations blur, camera relatively stable), slow-motion effects (Matrix-style, moving objects blur while camera doesn't).
- Intensity control: Shutter angle (0-360°, higher = more blur). Typical: 90-180° (moderate blur, not excessive), 270-360° (extreme blur, stylistic). Adjust per scenario (normal gameplay = 90°, chase sequence = 180°, impact moment = 270° brief spike). Sample count (blur samples, 4-16 typical, more = smoother + slower).
- Disable for UI/minimap: Motion blur only on main camera (UI camera no blur, text stays readable). Camera post-processing layer (main camera layer = 0, UI camera layer = 1, motion blur volume only on layer 0). Critical: UI with motion blur = unreadable (blurred text, nausea-inducing for menus).

**Post-Processing Volume Management:**
- Global volume: Single volume (affects entire scene, default settings). Place in scene root (Volume component, Profile = global post-processing asset, Is Global = true). Configure: color grading (global tone mapping, saturation, contrast), bloom (global glow), vignette (global edge darkening). Always active (no position-based blending).
- Local volumes: Position-based blending (enter trigger area, post-processing changes). Example: indoor volume (darkens, increases contrast, reduces saturation), outdoor volume (brightens, vibrant colors, sky-influenced color grading). Volume component: Is Global = false, Blend Distance (meters to blend over, 5-10m typical), Priority (higher overrides lower when overlapping).
- Per-camera volumes: Camera only sees specific volume layers (Camera component > Volume Layer Mask). Main camera = all layers (sees all volumes), minimap camera = no volumes (layer mask = None, no post-processing). Allows selective effects (DOF only on main camera, minimap stays sharp).
- Quality scaling: Different profiles per quality tier (Low quality = minimal post-processing, High = full effects). QualitySettings script: assign appropriate profile to global volume based on current quality level. Example: Low profile (no DOF/motion blur, basic color grading), High profile (all effects enabled, max quality settings).

**Platform-Specific:**
- **PC**: Full post-processing (DOF, motion blur, bloom, color grading, ambient occlusion). High quality settings (16-sample DOF, bokeh enabled, full-res motion blur). Per-camera customization (main camera full effects, secondary cameras selective). Performance acceptable (5-15ms post-processing, within budget).
- **Consoles**: Standard post-processing (DOF, motion blur, bloom, color grading). Moderate quality (8-sample DOF, simplified bokeh, half-res motion blur). Fixed performance budget (target 5-10ms post-processing, balance with other rendering). Quality profiles (Performance mode = reduced effects, Quality mode = enhanced).
- **Switch**: Minimal post-processing (bloom, basic color grading, no DOF/motion blur). Very low quality (4-sample blur, no bokeh, simplified algorithms). Performance critical (target <3ms post-processing, or disable entirely). Screen shake only (cheap feedback, no expensive blur effects).
- **Mobile**: Extremely minimal post-processing (color grading only, no DOF/motion blur/AO). Cheapest algorithms (LUT color grading, single-pass bloom). Performance prohibitive (DOF = 10-20ms on mobile GPU, unacceptable). Screen shake acceptable (CPU-based, minimal cost). Avoid complex effects entirely (prefer baked lighting, simple shaders over post-processing).

## Common Pitfalls

**Screen Shake on Raycasts**: Developer applies shake directly to camera position (Camera.transform.position += shake). Raycasts from camera affected (crosshair raycasts shake with camera, hit wrong objects, weapon aim wobbles). Symptom: Raycasts unreliable during shake (player shooting misses, interaction prompts flicker). Solution: Camera rig (parent GameObject stable for raycasts, child camera shakes), or store original position (raycasts use unshaken position, rendering uses shaken).

**DOF on All Cameras**: Developer enables DOF globally (all cameras get DOF post-processing). Minimap blurred (DOF applied to overhead camera, minimap unreadable), UI blurred (DOF on UI camera, text illegible). Symptom: Minimap/UI blurry, players complain about readability. Solution: Disable post-processing on non-main cameras (minimap/UI camera > Rendering > Post Processing = off), or use volume layer masks (DOF volume only on main camera layer).

**Excessive Motion Blur**: Developer sets motion blur shutter angle = 360° (thinking more blur = better). Every movement extremely blurred (entire screen trails, nausea-inducing, gameplay unplayable). Symptom: Player complaints (motion sickness, can't see anything, disorienting). Solution: Moderate settings (shutter angle 90-180°, noticeable but not overwhelming), add quality option (allow players to disable motion blur in settings, some players hate motion blur). Test with playtesters (motion blur tolerance varies widely).

**Not Clearing Trauma**: Developer adds trauma on every hit (trauma += 0.5), never decays (trauma accumulates infinitely). Camera shakes constantly (trauma reaches 10+, permanent extreme shake). Symptom: Camera never stops shaking (unplayable after few hits, trauma accumulated without decay). Solution: Decay trauma over time (trauma -= decayRate * Time.deltaTime), clamp maximum (trauma = Mathf.Clamp01(trauma), prevents accumulation >1.0). Always decay (ensure shake eventually stops).

## Tools & Workflow

**Post-Processing Stack**: URP/HDRP: Volume component (GameObject > Volume > Global Volume or Box Volume). Inspector: Profile (post-processing asset), Priority (override order), Is Global (global vs local), Blend Distance (local volumes only). Profile editor: Add Overrides (DOF, Motion Blur, Bloom, etc.), configure parameters per effect.

**Legacy Post-Processing**: Unity Post Processing Stack v2 (older projects, pre-URP/HDRP). Package Manager > Post Processing (install), Layer for post-processing (create layer "Post Processing"), Camera > Post Process Layer component (layer = Post Processing), Post-process Volume (GameObject with volume, profile assigned). Similar workflow to URP volumes.

**DOF Debugging**: Scene view > Depth visualization (verify depth buffer, DOF uses depth). Frame Debugger > DOF passes (shows CoC calculation, blur passes, composite). Adjust focal distance (real-time feedback in game view, see focus plane shift). Preview bokeh (Frame Debugger shows bokeh texture, verify shape).

**Motion Blur Debugging**: Frame Debugger > Motion Blur pass (shows motion vectors, blur direction per pixel). Scene view motion vectors (enable motion vector visualization, color-coded directions). Verify: moving objects have motion vectors (colored pixels), static objects don't (black pixels = no motion). Adjust sample count (Frame Debugger shows samples per pixel, reduce if too slow).

**Screen Shake Script**: Create C# script (CameraShake.cs), attach to camera rig parent. Public methods: AddTrauma(float amount) (called from gameplay events), Update() (decays trauma, applies shake). Inspector: Max Shake magnitude (0.1-0.5), Decay Rate (1-3), Noise Speed (10-30 for Perlin). Expose to gameplay (explosion script calls CameraShake.AddTrauma(0.8)).

**Volume Profile Assets**: Create > Rendering > Volume Profile (creates profile asset). Add to global volume (Volume component > Profile = asset). Duplicate profiles (Low/Medium/High quality variants), swap at runtime (volume.profile = highQualityProfile). Version control: commit profiles (shared settings across team), per-artist profiles (individual look-dev, merge to master profile).

## Related Topics

- [16.1 Camera Fundamentals](16-01-Camera-Fundamentals.md) - Camera basics
- [13.3 Post-Processing](13-03-Post-Processing.md) - Post-processing effects
- [16.2 Multiple Camera Rendering](16-02-Multiple-Camera-Rendering.md) - Per-camera configuration
- [5.2 GPU-Side Optimization](05-02-GPU-Side-Optimization.md) - Post-processing performance

---

[← Previous: 16.2 Multiple Camera Rendering](16-02-Multiple-Camera-Rendering.md) | [Next: Chapter 17 →](../chapters/17-Level-Of-Detail-Systems.md)
