# 19.1 Particle Systems

[← Back to Chapter 19](../chapters/19-Visual-Effects.md) | [Main Index](../README.md)

Particle systems = GPU-accelerated effects (fire, smoke, explosions, magic spells). Unity Particle System component: emission (spawn rate, bursts), shape (sphere, cone, mesh), lifetime (fade, size over time), collision (world interaction), rendering (billboard, mesh, trail). Performance: CPU particles (legacy, 10K-50K limit), GPU particles (VFX Graph, 100K-1M possible), instancing (batched rendering = one draw call). Optimization: limit active particles (pooling, distance culling), reduce overdraw (smaller particles, alpha cutout), texture atlas (combine textures = fewer material swaps).

---

## Overview

Unity Particle System: component-based (Inspector modules = Emission, Shape, Color, Size, Rotation, Collision). Emission: rate over time (100 particles/sec), bursts (spawn 500 instantly = explosion). Shape: spawn volume (sphere, cone, box, mesh surface). Lifetime modules: color gradient (fade alpha = dissolve), size curve (shrink over time), velocity (gravity, orbital). Rendering: billboard (always face camera = fire), mesh (3D particles = debris), stretched billboard (motion blur = sparks). Performance: 10K-20K particles typical (CPU limit = one system), batching via instancing (GPU Instancing = reduce draw calls).

VFX Graph: visual node editor for GPU particles (SRP only = URP/HDRP). GPU compute: spawn/update on GPU (1M particles possible = no CPU bottleneck), custom forces (turbulence, vector fields), collision (signed distance fields, depth buffer). Rendering: lit particles (receive shadows), distortion (refraction effect), decals (impact marks). Use case: massive effects (avalanche, swarms, weather), or complex simulation (fluid dynamics, flocking).

## Key Concepts

- **Particle System Component**: CPU-based particles (legacy, widely compatible). Modules: Main (duration, looping, start lifetime/speed/size/color), Emission (rate over time, bursts), Shape (spawn geometry = sphere/cone/box/mesh), Velocity Over Lifetime (acceleration, orbital), Color Over Lifetime (gradient), Size Over Lifetime (curve), Rotation (angular velocity), Collision (world planes, 3D collision), Sub Emitters (spawn particles on death = secondary effects), Renderer (billboard/mesh/trail). Performance: ~10K-50K particles max (CPU update = bottleneck), batching via GPU Instancing (same material = one draw call).
- **Emission Strategies**: Rate over Time (constant spawn = 100/sec for continuous fire), Bursts (instant spawn = 500 particles at t=0 for explosion), Distance-based (emit when moving = dust trail), Time-based triggers (spawn at specific times = timed effects). Optimization: reduce emission rate (fewer particles = less work), stop emission when off-screen (distance culling), use bursts sparingly (500 particle burst = spike).
- **Shape Module**: spawn volume determines particle origin. Sphere (radial = explosion), Cone (directional = fire, smoke plume), Box (volume fill = rain, snow), Mesh (surface emission = magic aura on character), Edge (spawn along edge = electric arc). Shape affects performance: mesh emission expensive (ray-cast to surface = CPU cost), simple shapes fast (sphere = analytic).
- **Overdraw Optimization**: particles overlap (alpha blending = expensive on mobile). Techniques: reduce particle size (smaller = less overdraw), alpha cutout (discard transparent pixels = no blend, but harsh edges), soft particles (fade at intersections = depth buffer read, moderate cost), reduce particle count (10K → 5K = half overdraw). Mobile: aggressive culling (max 2K-5K particles total), small textures (256×256 atlas).
- **VFX Graph (GPU Particles)**: node-based GPU compute (URP/HDRP, compute shaders = 1M particles possible). Workflow: Visual Effect Graph window (node editor), Contexts (Initialize, Update, Output), Blocks (operations = Set Velocity, Add Force), Operators (math = noise, multiply). Features: GPU spawning (compute shader = no CPU submission), GPU collision (SDF or depth buffer = fast), custom attributes (user data = flexibility). Performance: 100K-1M particles (GPU limited = fill-rate), batch friendly (indirect rendering = one draw call per system).
- **Texture Atlases**: combine particle textures (multiple sprites in one texture = reduce material swaps). Layout: 2×2 grid (4 variations), 4×4 grid (16 variations), or flipbook (animated sequence). Shader: UV offset/scale (sample correct sprite = atlas coordinates). Benefits: reduce SetPass calls (one material = all particle types), smaller builds (one 2K texture vs 16 × 512 textures). Unity: Texture Sheet Animation module (flipbook support = UV animation).

## Best Practices

**Performance Targets**:
- **PC/Console**: 50K-100K particles (CPU particles = 50K limit, GPU particles = 100K+).
- **Mobile**: 5K-10K particles (fill-rate limited = reduce overdraw), GPU particles (VFX Graph = better than CPU on high-end mobile).
- **VR**: 10K-20K particles (stereo = double fill-rate cost), avoid near camera (periphery less noticeable).

**Particle Culling**:
- Distance culling: disable systems beyond 50m (ParticleSystem.Stop() = save CPU/GPU).
- Frustum culling: automatic (Unity culls off-screen systems), verify in Profiler (Culling.CullParticles).
- LOD: reduce emission rate at distance (LOD Group = switch to low-particle version), or disable entirely (very far = not visible).

**Overdraw Reduction**:
- Soft particles: enable (Frame Debugger → check Soft Particles = depth fade, moderate cost).
- Particle size: reduce by 30-50% (smaller = less overdraw, often imperceptible).
- Alpha cutout: use for opaque-ish particles (smoke core = cutout, edges = alpha blend).

**Platform-Specific**:
- **Mobile**: CPU particles only (VFX Graph requires compute shaders = not all mobile GPUs), texture compression (ETC2/ASTC = reduce memory).
- **Console**: VFX Graph supported (PS5/Xbox = compute shaders fast), massive particle counts (100K+ feasible).
- **WebGL**: CPU particles (no compute shaders = VFX Graph unsupported), limit to 5K particles (WebGL overhead high).

## Common Pitfalls

**Too Many Active Systems**: developer spawns 100 particle systems (one per enemy projectile = each system has overhead). Symptom: CPU spike (Particle.Update = 10ms, many small systems). Solution: pooling (reuse ParticleSystem instances = Stop/Clear/Play), or single system with Sub Emitters (one system = multiple effects).

**Alpha Blending Overdraw**: fire effect uses 10K large particles (each 512×512 billboard = massive screen coverage). Symptom: GPU bottleneck (fill-rate = 30 FPS mobile), Frame Debugger shows 50x overdraw. Solution: reduce particle count (10K → 3K), smaller particles (512 → 256 pixels), alpha cutout (opaque core = no blend).

**VFX Graph on Mobile**: developer uses VFX Graph (compute shaders required). Symptom: pink particles on mobile (compute shader unsupported = Mali-G76 or older). Solution: CPU Particle System (legacy = widely compatible), or require high-end mobile (Adreno 650+, Mali-G77+ = compute shaders).

## Tools & Workflow

**Particle System Creation**:
```csharp
// Create via code
GameObject go = new GameObject("Fire");
ParticleSystem ps = go.AddComponent<ParticleSystem>();

// Configure Main module
var main = ps.main;
main.startLifetime = 2.0f;
main.startSpeed = 5.0f;
main.startSize = 0.5f;
main.startColor = new Color(1, 0.5f, 0, 1); // Orange

// Configure Emission
var emission = ps.emission;
emission.rateOverTime = 50; // 50 particles/sec

// Configure Shape
var shape = ps.shape;
shape.shapeType = ParticleSystemShapeType.Cone;
shape.angle = 25; // Cone angle

// Configure Color Over Lifetime
var col = ps.colorOverLifetime;
col.enabled = true;
Gradient grad = new Gradient();
grad.SetKeys(
    new GradientColorKey[] { new GradientColorKey(Color.white, 0.0f), new GradientColorKey(Color.red, 1.0f) },
    new GradientAlphaKey[] { new GradientAlphaKey(1.0f, 0.0f), new GradientAlphaKey(0.0f, 1.0f) }
);
col.color = grad;
```

**VFX Graph Setup** (GPU Particles):
```csharp
1. Package Manager → Visual Effect Graph (URP/HDRP)
2. Project → Create → Visual Effects → Visual Effect Graph
3. Open graph: node editor (Contexts = Initialize, Update, Output)
4. Initialize: Set Lifetime Random (1-3 sec), Set Velocity Random
5. Update: Add Gravity (world space), Turbulence (noise field)
6. Output: Quad (billboard), Color Over Life (gradient)
7. GameObject → Effects → Visual Effect (assign graph)
```

**Particle Pooling**:
```csharp
// Pool of particle systems (reuse instances)
Queue<ParticleSystem> particlePool = new Queue<ParticleSystem>();

ParticleSystem GetParticle()
{
    if (particlePool.Count > 0)
    {
        ParticleSystem ps = particlePool.Dequeue();
        ps.Clear(); // Clear old particles
        ps.Play();  // Restart
        return ps;
    }
    else
    {
        // Create new if pool empty
        return Instantiate(particlePrefab);
    }
}

void ReturnParticle(ParticleSystem ps)
{
    ps.Stop();
    ps.Clear();
    particlePool.Enqueue(ps);
}
```

**Texture Sheet Animation** (Flipbook):
```csharp
// Particle System → Texture Sheet Animation module
var texSheet = ps.textureSheetAnimation;
texSheet.enabled = true;
texSheet.mode = ParticleSystemAnimationMode.Grid;
texSheet.numTilesX = 4; // 4×4 grid
texSheet.numTilesY = 4;
texSheet.animation = ParticleSystemAnimationType.WholeSheet; // Animate through all tiles
texSheet.frameOverTime = AnimationCurve.Linear(0, 0, 1, 16); // 16 frames over lifetime
```

**Soft Particles** (Depth Fade):
```csharp
// Material: Particles/Standard Unlit
// Enable Soft Particles (Inspector checkbox)
// Camera must have depth texture enabled (automatic in URP/HDRP)

// Custom shader (HLSL)
// Sample depth buffer
float sceneZ = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, uv));
float fragZ = i.projPos.z;
float fade = saturate((sceneZ - fragZ) / _SoftParticleDistance);
color.a *= fade; // Fade alpha near geometry
```

**Distance Culling**:
```csharp
void Update()
{
    float distance = Vector3.Distance(transform.position, Camera.main.transform.position);
    
    if (distance > cullDistance)
    {
        if (particleSystem.isPlaying)
            particleSystem.Stop(true, ParticleSystemStopBehavior.StopEmittingAndClear);
    }
    else
    {
        if (!particleSystem.isPlaying)
            particleSystem.Play();
    }
}
```

**VFX Graph Custom Force** (Turbulence):
```
// VFX Graph: Update context
1. Add Block → Force → Turbulence
   - Frequency: 1.0 (noise scale)
   - Intensity: 2.0 (force strength)
   - Drag: 0.1 (resistance)
2. Result: particles follow noise field (organic motion)
```

## Related Topics

- [19.2 Trail Renderers](19-02-VFX-Graph.md) - Particle trails
- [19.3 Billboard Rendering](19-03-Procedural-Effects.md) - Particle billboards
- [06.3 GPU Instancing](05-02-GPU-Side-Optimization.md) - Batch particles
- [09.1 Texture Import Settings](10-01-AssetBundle-Fundamentals.md) - Particle texture atlases

---

[← Previous: Chapter 18 Animation](../chapters/18-Animation-Systems.md) | [Next: 19.2 Trail Renderers →](19-02-VFX-Graph.md)
