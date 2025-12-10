# 19.2 VFX Graph

[← Back to Chapter 19](../chapters/19-Visual-Effects.md) | [Main Index](../README.md)

VFX Graph is Unity's node-based GPU particle system: millions of particles via compute shaders, complex behaviors (forces, collisions, events), and high-quality visual effects for PC/console.

---

## Overview

VFX Graph vs Particle System: Traditional Particle System = CPU-based (C# simulation, limited to thousands of particles, Unity 3.5 era), VFX Graph = GPU-based (compute shader simulation, millions of particles, Unity 2019+ SRP only). Performance: VFX Graph 10-100x more particles (CPU system = 5,000 particles max before slowdown, VFX Graph = 500,000+ particles at 60 FPS). Flexibility: node-based visual scripting (spawn, update, render contexts, custom forces/attributes), GPU Events (particles trigger other particles, chain reactions).

Requirements: SRP only (URP or HDRP, not Built-In pipeline), compute shader support (DirectX 11+, Vulkan, Metal, PS4+, Xbox One+), not for mobile (compute shaders expensive on mobile GPUs, limited support, use Particle System instead). Best for: PC/console VFX (explosions, magic spells, weather, destruction debris), high particle counts (rain = 100,000 droplets, snow = 500,000 flakes, fire embers = 50,000 particles).

Workflow: Create VFX Graph asset (Assets > Create > Visual Effects > Visual Effect Graph), open editor (double-click asset, VFX Graph window), build graph (add contexts: Spawn, Initialize, Update, Output, connect nodes), assign to scene (VisualEffect component on GameObject, assign VFX Graph asset). Contexts define stages: Spawn (emission rate/bursts), Initialize (starting values: position, velocity, color), Update (per-frame simulation: forces, collisions, age), Output (rendering: quad particles, mesh particles, lines).

## Key Concepts

- **Contexts and Blocks**: VFX Graph organized in Contexts (stages: Spawn Context, Initialize Context, Update Context, Output Context), Blocks within contexts (operations: Set Velocity, Add Force, Set Color). Execution flow: Spawn → Initialize (sets initial attributes), Update (runs every frame per particle, applies forces/logic), Output (renders particles). Add blocks: right-click context > Create Block, or drag from Blackboard. Example: Spawn Context (Rate: 100 particles/sec), Initialize Context (Set Position: Sphere, Set Velocity: Random), Update Context (Gravity: -9.8 Y), Output Context (Quad, Blend Mode: Additive).
- **Attributes**: Per-particle data (position, velocity, color, size, lifetime, custom). Built-in: position, velocity, age, lifetime, size, color, alpha. Custom: add via Blackboard (e.g., "Temperature" float, "SpinSpeed" vector3), use in blocks (operators read/write attributes). GPU memory: attributes stored in buffers (each particle = N bytes for attributes, 1M particles × 100 bytes = 100MB, keep attribute count low). Data types: float, vector2/3/4, color, uint, int, bool.
- **Operators and Expressions**: Compute values (math, noise, sampling). Operators: Add, Multiply, Sine, Perlin Noise, Sample Texture, etc. Connections: drag output pin to input pin (data flows, visual programming). Example: Perlin Noise (coordinate = particle position) → Multiply (scale noise) → Add Force (noise-driven turbulence). Inline operators: click property > Convert to Operator (exposes as node, allows complex expressions).
- **GPU Events**: Particles trigger events (spawn new particles, trigger logic). Workflow: Output Context > GPU Event (triggers event on particle death/collision), Event Context (receives event, spawns particles). Example: firework rocket (Initialize: upward velocity, Update: gravity, Output: mesh particle, GPU Event on death → Spawn 1000 spark particles). Chain reactions: spark particles trigger more sparks (recursive events, massive particle counts). Rate limit: max events per frame (prevent infinite recursion, configurable in event settings).
- **Signed Distance Fields (SDF)**: 3D volume for collisions (baked mesh collision data, particles collide with complex geometry). Bake SDF: Mesh → SDF Baker (Unity tool converts mesh to 3D texture, stores distance to surface). VFX Graph: Collision SDF block (reads SDF texture, particles bounce off surfaces). Expensive: SDF = 3D texture (128³ = 8MB per SDF, memory cost), but accurate collisions (thousands of particles collide with complex mesh, no per-particle raycasts). Alternative: simple colliders (sphere, plane, faster but less accurate).
- **Output Contexts**: Rendering modes (how particles appear). Quad Output (billboards, default, camera-facing quads), Mesh Output (3D mesh per particle, rocks/debris), Line Output (trails, lasers, connects particles), Lit Output (particles affected by scene lighting, shadows, expensive but realistic). Blend modes: Additive (bright, fire/sparks), Alpha Blend (transparent, smoke), Opaque (solid particles, rare). Sorting: automatic (GPU depth sort, back-to-front for transparency), or no sorting (faster, acceptable for additive blending).

## Best Practices

**Graph Organization:**
- Contexts left-to-right: Spawn (leftmost) → Initialize → Update → Output (rightmost), visual flow (clear execution order, easy to read). Multiple systems: separate subgraphs (firework system = rocket graph + explosion graph, organize hierarchically).
- Systems: VFX Graph supports multiple systems in one asset (System A = smoke, System B = embers, System C = sparks, all in one .vfx file). Activate/deactivate systems (VisualEffect component: `visualEffect.SetBool("SystemA_Enabled", true)`), or separate assets (one .vfx per effect, easier to manage but more files).
- Blackboard: Group properties (Spawn category: emission rate/burst count, Appearance category: color/size, Forces category: gravity/wind), expose to Inspector (VisualEffect component shows exposed properties, designers tune without editing graph). Default values: set in Blackboard (property default = 10, Inspector shows 10, user can override).
- Comments: Add sticky notes (VFX Graph > Create Sticky Note, document complex sections, "Turbulence applied here", "GPU Event triggers explosion"), frame groups (select nodes, right-click > Group, visual organization).

**Performance Optimization:**
- Particle count limits: Target <100K particles for 60 FPS (PC/console, 100K = ~2ms simulation + render), <50K for 30 FPS (console), <10K for VR (low latency critical). Profile: Profiler > GPU > VFX Graph passes (shows simulation + rendering cost), Frame Debugger (shows particle draw calls).
- Attribute minimization: Use only needed attributes (every attribute = memory + bandwidth, 10 attributes × 1M particles = 40-80MB). Remove unused (default graph adds many attributes, delete unnecessary blocks, attributes auto-removed if unused). Custom attributes sparingly (prefer reusing built-in attributes, e.g., reuse "size" for temperature instead of adding custom attribute).
- Operator optimization: Expensive operators (Perlin Noise, Sample Texture = texture reads, slow), cheap operators (Add, Multiply, Sine = ALU, fast). Reduce noise samples (one Perlin Noise per particle = expensive, sample once in Spawn/Initialize, store result, reuse in Update). Texture sampling (minimize, cache samples, use low-resolution textures for noise/gradients).
- Update frequency: Not all logic needs per-frame update (turbulence can update every 2-3 frames, use Particle Strip for trails instead of per-frame line rendering). Fixed Delta Time: VFX Graph Update Context > Fixed Delta Time (updates at fixed rate, e.g., 30 FPS update for 60 FPS render = half cost, acceptable for distant effects).
- Output optimization: Quad particles (cheapest, 2 tris per particle), Mesh particles (expensive, avoid for thousands of particles, use for dozens of large debris), Lit particles (lighting cost, use sparingly for hero particles, bulk particles unlit). Additive blending (no depth sorting needed, faster than alpha blend with sorting).

**Collision Setup:**
- Simple colliders: Collide with Sphere/Plane/Cylinder blocks (analytic collision, very cheap, GPU evaluates formula). Use for: ground plane (infinite plane, particles bounce off floor), spherical shields (sphere collider, particles deflect). Multiple colliders (add multiple Collide blocks, particles check all, reasonable for 5-10 colliders).
- SDF collisions: Bake mesh to SDF (select mesh, Add Component > VFX > SDF Bake, configure resolution: 32 = low detail fast, 128 = high detail expensive, bake), assign SDF (VFX Graph > Collide with Signed Distance Field block, SDF Texture = baked SDF). Use for: complex environment collision (particles bounce off terrain, buildings, props), accurate contact (particles settle into crevices, realistic debris). Memory: 64³ SDF = 1MB, 128³ = 8MB (limit SDF count, share SDFs where possible).
- Collision responses: Bounce (particles reflect off surface, velocity reversed), Stick (particles stop at surface, velocity zero), Slide (particles slide along surface, tangent velocity preserved). Restitution (bounciness: 0 = stick, 1 = perfect elastic bounce, 0.5 = typical). Friction (reduces tangent velocity, simulates rough surfaces).

**Shader Integration:**
- Output Shader Graph: VFX Graph Output Context can use Shader Graph (custom particle appearance, e.g., distortion particles, holographic effects). Create: Shader Graph > VFX target (create SG with VFX output), assign in Output Context (Output > Shader Graph = your shader). Particle attributes accessible (Shader Graph reads particle color, size, velocity, custom attributes via VFX nodes).
- Lit particles: Output Context > Lit Output (particles receive scene lighting, cast/receive shadows). Normal maps (assign normal map texture, particles have 3D lighting), expensive (lighting per particle, acceptable for <10K particles, avoid for 100K+). Use cases: magical runes (glowing symbols on surfaces, lit by scene), embers (fire particles with lighting, realistic illumination).
- Custom rendering: Advanced (write custom HLSL, particle shaders with special effects), requires graphics programming (not node-based, manual code). Example: Shader Graph with custom HLSL function (distortion effect, refraction, heat haze).

**Platform-Specific:**
- **PC**: Full VFX Graph (millions of particles, complex graphs, GPU Events, SDF collisions), unlimited (RTX GPUs handle 1M+ particles, high-end effects). Target: 100K-500K particles (60 FPS at 1080p, typical for PC VFX). Compute: DirectX 11/12, Vulkan (all support compute shaders, VFX Graph fully functional).
- **Consoles**: VFX Graph supported (PS4/5, Xbox One/Series X, compute shaders available), moderate particle counts (PS4/Xbox One = 50K-100K particles 30 FPS, PS5/Series X = 100K-200K particles 60 FPS). Memory: watch VRAM (VFX Graph particle buffers + SDFs, consoles have unified memory, balance with textures/meshes). Optimize: reduce attributes, limit SDF use, profile on devkit.
- **Switch**: VFX Graph possible but expensive (Tegra X1 has compute, but slow, <10K particles viable), use Particle System instead (CPU particles more efficient on Switch, VFX Graph overhead unjustified). Exception: simple VFX (few thousand particles, no SDF, minimal operators, test performance on hardware).
- **Mobile**: VFX Graph not recommended (compute shaders expensive on tile-based GPUs, many devices lack support or perform poorly), use Particle System (CPU particles better for mobile, or GPU particles via Particle System GPU mode in URP). Avoid: VFX Graph on mobile = poor performance, crashes on unsupported devices.

## Common Pitfalls

**Unbounded Particle Count**: Developer sets Spawn Context Rate = 10,000 particles/sec (thinking more = better), lifetime = 10 seconds, particle count explodes (10K/sec × 10 sec = 100,000 particles alive, continues growing). Frame rate collapses (GPU simulation + rendering 100K particles expensive, >10ms). Symptom: FPS drops over time (first few seconds okay, then drops to 10 FPS as particles accumulate), Profiler shows VFX Graph Update + Render expensive. Solution: Reduce spawn rate (1,000 particles/sec typical, 10K only for short bursts), shorter lifetime (2-5 seconds, particles die quickly, limit alive count), or capacity limit (Spawn Context > Capacity = max alive particles 50K, oldest die when cap reached).

**Excessive Attributes**: Developer adds many custom attributes (Attribute: Temperature, WindForce, SpinRate, RotationAxis, RandomSeed = 10+ custom attributes per particle, thinking enables complex behavior). Memory explodes (1M particles × 20 attributes × 4 bytes = 80MB per effect, multiple effects = gigabytes). Performance poor (GPU memory bandwidth saturated, reading/writing attributes dominates cost). Symptom: High memory usage (Profiler shows VFX Graph buffers = hundreds of MB), slow simulation (Profiler shows VFX Update expensive, memory-bound). Solution: Minimize attributes (use only essential, typical = 5-8 attributes total including built-ins), reuse built-in attributes (repurpose "size" for temperature, avoid custom), or compute on-the-fly (calculate derived values in operators instead of storing, e.g., compute spin from particle ID + time, don't store SpinRate attribute).

**SDF Memory Leak**: Developer bakes many SDFs (50 props in scene, each baked to 128³ SDF, thinking collision accuracy critical). Memory usage gigabytes (50 × 8MB = 400MB for SDFs alone). Load times long (loading hundreds of MB of SDF data). Symptom: High texture memory (Profiler shows 3D textures dominating memory), long scene load (loading bar stuck on SDFs). Solution: Reduce SDF resolution (64³ instead of 128³, 8x less memory, acceptable quality for distant collision), limit SDF count (only hero props need SDF, use simple colliders for others, 5-10 SDFs max), or disable collisions (particles don't always need collision, disable for effects where intersection invisible).

**GPU Event Infinite Loop**: Developer creates GPU Event (particle death triggers spawn of 10 particles, each dies and triggers 10 more, exponential growth). Particle count explodes (1 → 10 → 100 → 1000 → infinite, frame crashes or freezes). Symptom: Particle count grows exponentially (first frame 10, next 100, next 1000, then freeze), Unity hangs (GPU locked processing infinite particles), crash (out of memory). Solution: Event rate limiting (GPU Event > Max Events Per Frame = 1000, caps event triggers, prevents runaway), decrease spawn count (each event spawns 1-2 particles, not 10, controlled growth), or lifetime limits (spawned particles have short lifetime, die before triggering more events, breaks recursion).

## Tools & Workflow

**VFX Graph Editor**: Window > Visual Effects > Visual Effect Graph (opens VFX Graph editor window). Create asset: Assets > Create > Visual Effects > Visual Effect Graph (.vfx file). Edit: double-click .vfx asset (opens in VFX Graph window). Interface: left = Blackboard (properties, attributes), center = graph canvas (contexts + blocks), right = Inspector (selected node settings). Navigation: middle-mouse drag (pan canvas), scroll wheel (zoom), F key (frame selected nodes).

**Context Creation**: Right-click canvas > Create Node > Context (Spawn/Initialize/Update/Output Context). Auto-connections: contexts auto-connect (Initialize connects to Update, Update to Output, visual flow). Multiple contexts: add multiple Update contexts (separate logic stages, e.g., Update Context 1 = forces, Update Context 2 = collisions), or multiple Output contexts (render same particles as quads + trails).

**Block Library**: Right-click context > Create Block (shows block library: Set Position, Add Force, Collide with Sphere, etc.). Search: type in search box ("gravity", "color", "noise"), filters blocks. Categories: Attribute (set/get attributes), Force (physics forces), Collision (collider blocks), Event (GPU Events), Custom (user scripts). Add: click block (adds to context), or drag from library.

**Blackboard Properties**: Blackboard > + button (add property: float, vector, color, texture, mesh). Expose: property checkbox "Exposed" (appears in VisualEffect component Inspector, designers tune). Default value: set in Blackboard (property = 100, Inspector starts at 100). Use in graph: drag property from Blackboard to canvas (creates Get Property node, reads value).

**Profiling VFX**: Unity Profiler > GPU > VFX (shows VFX Graph passes: Update Compute, Render Particles). Timing: Update = simulation cost (compute shader), Render = draw cost (fragment shader, overdraw). Breakdown: expand VFX entry (shows per-system cost, identify expensive effects). Frame Debugger: shows VFX draws (VFX Graph renders as "VFX" passes, see particle quads/meshes, validate blend modes/sorting).

**SDF Baker**: Select mesh > Add Component > Visual Effects > SDF Bake (adds SDF Bake component). Configure: Resolution (32/64/128/256, higher = more detail + memory), Bake button (generates SDF, saves as 3D texture asset in Assets folder). Assign: VFX Graph > Collide with Signed Distance Field block > SDF Texture = baked SDF asset. Preview: SDF Bake component shows gizmo (visualizes SDF volume, red = inside mesh, green = outside).

**VFX Component**: GameObject > Effects > Visual Effect (adds VisualEffect component). Assign: VisualEffect > Asset Template = .vfx asset (links component to VFX Graph). Properties: exposed Blackboard properties appear (adjust values, real-time updates in Scene view). Control: `visualEffect.Play()`, `visualEffect.Stop()`, `visualEffect.SetFloat("PropertyName", value)` (script control). Events: `visualEffect.SendEvent("OnHit")` (trigger named events, VFX Graph Event Context receives).

## Related Topics

- [19.1 Particle Systems](19-01-Particle-Systems.md) - CPU-based particles
- [19.3 Procedural Effects](19-03-Procedural-Effects.md) - Compute shader effects
- [12.4 Shader Graph](12-01-Unity-Shader-Languages.md) - Visual shader authoring
- [18.2 Universal Render Pipeline](18-02-Universal-Render-Pipeline.md) - URP integration

---

[← Previous: 19.1 Particle Systems](19-01-Particle-Systems.md) | [Next: 19.3 Procedural Effects →](19-03-Procedural-Effects.md)
