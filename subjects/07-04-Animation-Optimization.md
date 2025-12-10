# 7.4 Animation Optimization

[← Back to Chapter 7](../chapters/07-Asset-Optimization.md) | [Main Index](../README.md)

Animation optimization reduces CPU overhead from skeletal animation, minimizes memory footprint of animation clips, and leverages GPU skinning for maximum performance.

---

## Overview

Skeletal animation is CPU-intensive: evaluating keyframes, blending animations, and computing bone matrices. A character with 50 bones running 3 blended animations evaluates 150 bone transforms per frame. In a scene with 100 characters, that's 15,000 bone evaluations—1-3ms CPU time. Animation clips also consume memory: 30-second clip with 50 bones at 60fps = 90,000 keyframes × 10 bytes = 900KB per clip. With 200 clips, that's 180MB.

Animation optimization targets three areas: CPU optimization (reduce bone counts, optimize blend tree complexity, use animation LOD), memory optimization (keyframe compression, clip simplification), and GPU offloading (GPU skinning moves vertex blending to GPU, freeing CPU). Additionally, culling off-screen animated objects and using animation instancing for crowds provides massive performance gains (10-50x for large crowds).

Optimization workflow: profile animation CPU cost (Profiler > MeshSkinning), identify high bone-count characters (>100 bones), implement animation LOD (reduce update rate for distant characters), compress animation clips (keyframe reduction), and enable GPU skinning in Player Settings. For crowds, implement animation instancing via GPU instancing or Entities/DOTS. Results: 50-70% CPU reduction, 30-50% memory savings, and scalability to hundreds/thousands of animated characters.

## Key Concepts

- **Skeletal Animation**: Mesh deformation driven by bone hierarchy. Each bone is transform (position, rotation, scale). Mesh vertices weighted to bones (skinning). CPU evaluates bone transforms, GPU blends vertices.
- **Animation Clip**: Keyframe data for bones over time. Each keyframe stores bone transforms. Clips at 60fps create dense keyframe data. Compression reduces keyframes via curve simplification (remove redundant keyframes).
- **Animation Blending**: Combining multiple clips (walk + aim). CPU interpolates between clips based on weights. 2-clip blend: evaluate both clips, lerp results. 4-clip blend tree: evaluate 4 clips, blend hierarchically. Cost scales with clip count.
- **GPU Skinning**: Vertex blending on GPU instead of CPU. Uploads bone matrices to GPU (64-128 bones × 16 bytes = 1-2KB per character), GPU computes final vertex positions. Frees CPU, essential for many characters.
- **Animation LOD**: Reducing animation update rate for distant objects. Close characters: 60fps updates. Medium distance: 30fps. Far distance: 15fps or static pose. CPU cost scales proportionally (30fps = 50% CPU reduction).

## Best Practices

**Bone Count Reduction:**
- Target bone budgets per platform: Hero characters (60-100 bones), NPCs (40-60 bones), background characters (20-40 bones), props (10-20 bones). More bones = higher CPU and memory cost.
- Remove unnecessary bones: Facial bones for characters never seen up close, finger bones for distant NPCs, detail bones (ribbons, cloth sim) for non-hero assets.
- Use bone LOD: Close-up version with full skeleton (100 bones), distant version with simplified skeleton (50 bones). Switch based on camera distance.
- Validate bone usage: Disable bones with zero-weight influence (not affecting any vertices). Unity's Optimize Game Objects feature removes these automatically.

**Animation Clip Compression:**
- Enable keyframe reduction: Import Settings > Animation > Keyframe Reduction. Unity removes redundant keyframes (curves flat or linear). Reduces clip size by 30-70%.
- Reduce animation sample rate: 30fps instead of 60fps for ambient animations (idle, breathing). Halves keyframe count with minimal visual difference.
- Use animation compression: Import Settings > Animation > Compression = Optimal. Unity quantizes keyframes to 16-bit, reduces clip size 40-60% with imperceptible quality loss.
- Remove unused curves: Animation Import > Clips > Enable "Mask" to disable unused bone tracks. If arm bones don't animate in clip, mask them out (saves memory and CPU).

**Blend Tree Optimization:**
- Minimize simultaneous blends: 2-clip blend = evaluate 2 clips. 4-clip blend = evaluate 4 clips (2x CPU). Prefer 1D/2D blend trees with minimal clip count (2-4 clips max).
- Use additive blending: Additive animations (aim overlay) compute delta from bind pose, apply on top of base. Cheaper than full blend (adds transform deltas instead of blending full transforms).
- Cache blend results: If blend weights constant across multiple characters (all in same state), evaluate once, reuse results. Unity's animation instancing does this automatically.
- Profile blend tree complexity: Profiler > Animator.Update shows per-state CPU time. Simplify expensive states (reduce clip count, simplify blend trees).

**GPU Skinning and Instancing:**
- Enable GPU skinning: Edit > Project Settings > Player > Other Settings > GPU Skinning (console platforms). Moves vertex blending to GPU, freeing 50-80% of animation CPU.
- Use animation instancing for crowds: Multiple characters share animation clip, GPU computes unique bone matrices per instance. Rendering 100 characters costs 1 animation evaluation + 100 draw calls (instanced). 100x CPU savings.
- Implement via Unity DOTS: Entities + Hybrid Renderer V2 + Animation package provides automatic animation instancing. Scales to 1,000+ animated characters at 60fps.
- Optimize bone matrix uploads: Modern APIs (DirectX 12, Vulkan) support efficient structured buffer uploads. Use StructuredBuffer for bone matrices instead of constant buffer.

**Animation LOD:**
- Implement distance-based update rates: Close (0-20m): 60fps animation. Medium (20-50m): 30fps. Far (50-100m): 15fps. Very far (100m+): static pose or culled.
- Use Unity Animation LOD components: Custom script or Asset Store solutions (Animation Baker, GPU Instancer). Set update intervals based on camera distance.
- Profile LOD effectiveness: Profiler > MeshSkinning.Update shows CPU time. Moving characters to distance should reduce time proportionally (30fps = 50% reduction).
- Combine with mesh LOD: Distant characters use low-poly mesh + low animation update rate. Synergistic savings (70-90% total cost reduction).

**Culling and Optimization:**
- Frustum cull animated objects: Disable Animator component for off-screen characters. Unity's culling does this automatically for skinned meshes, but verify in profiler.
- Implement occlusion culling: Characters behind walls don't need animation updates. Use Unity's occlusion culling or custom raycasts.
- Disable root motion for crowds: Root motion (animation drives position) requires Transform updates. Crowds can use baked root motion or procedural movement (cheaper).
- Pool animation clips: Preload common clips during level load. Avoid runtime instantiation (causes GC allocations). Use Addressables or AssetBundles for on-demand loading.

**Platform-Specific:**
- **PC**: Handle 50-100 skinned characters with full animation. Enable GPU skinning (freebies on modern GPUs). Animation LOD less critical, but still beneficial for min-spec.
- **Xbox Series X/S**: GPU skinning essential. Series X handles 100+ characters, Series S targets 50-70. Test on Series S (weaker CPU, fewer GPU compute units).
- **PlayStation 5/4**: PS5 similar to Xbox Series X. PS4 (Jaguar CPU) struggles with animation. GPU skinning mandatory, aggressive animation LOD (30fps updates for distant characters).
- **Switch**: CPU bottleneck (1 GHz ARM). Target 20-30 animated characters max. GPU skinning essential, animation LOD mandatory (15fps for distant, static poses beyond 50m), low bone counts (<40 bones).

## Common Pitfalls

**High Bone Counts on All Characters**: Artist rigs every character with 150 bones (full facial rig, finger articulation, detail bones). Background NPCs never seen close-up use same rig. 100 NPCs × 150 bones = 15,000 bone evaluations per frame = 5-10ms CPU. Symptom: High MeshSkinning.Update cost in Profiler. Solution: Create LOD rigs (hero 150 bones, NPC 60 bones, crowd 30 bones). Use appropriate rig per character importance.

**Disabled GPU Skinning**: Developer doesn't enable GPU Skinning in Player Settings (off by default on some platforms). All vertex blending runs on CPU. 50 characters = 8-15ms CPU just for skinning. Symptom: MeshSkinning.Update dominates CPU Profiler. Solution: Enable GPU Skinning in Player Settings > Other Settings. Verify in profiler that CPU time drops 70-90%.

**60fps Animation Updates for All Characters**: All characters update animation at 60fps regardless of distance. Distant character at 200m (occupying 10 pixels) receives same CPU budget as hero character. Wasteful. Symptom: Animation cost doesn't scale with visible detail. Solution: Implement animation LOD. Distant characters update at 15-30fps, very distant use static poses or culled entirely.

## Tools & Workflow

**Unity Profiler - Animator Module**: Shows per-Animator CPU time. MeshSkinning.Update is vertex blending cost. Animator.Update is state machine and blend tree evaluation. Profile hierarchy to identify expensive characters.

**Animation Import Settings**: Select animation clip > Inspector > Animation tab. Keyframe Reduction, Compression, Resample Curves settings. Essential for memory and CPU optimization.

**Optimize Game Objects**: Humanoid rigs: Avatar > Configure > Optimize Game Objects. Bakes transforms into mesh, removes unnecessary GameObjects. Reduces Transform overhead 50-70%.

**Frame Debugger**: Verify GPU skinning active. Skinned mesh draws should show "SkinnedMeshRenderer" in shader properties with bone matrices. If CPU skinning, draws show baked vertex buffers.

**Animation Window**: Window > Animation > Animation. Visualize keyframe density. Sparse keyframes (linear curves) compress well. Dense keyframes (complex motion) consume memory.

**DOTS Animation Package**: Unity.Animation for ECS-based animation. Provides animation instancing, jobs-based evaluation, and GPU skinning. Scales to 1,000+ characters. Requires DOTS knowledge.

**GPU Instancer**: Asset Store tool for crowd rendering with animation instancing. Handles GPU skinning, LOD, and frustum culling automatically. Good for non-DOTS projects.

## Related Topics

- [23.1 Multithreading and Jobs System](23-01-Multithreading-Jobs.md) - Jobs-based animation evaluation
- [5.1 CPU-Side Optimization](05-01-CPU-Side-Optimization.md) - CPU performance optimization
- [17.1 LOD Systems](17-01-LOD-Systems.md) - LOD strategies including animation LOD
- [6.1 CPU Bottlenecks](06-01-CPU-Bottlenecks.md) - Identifying animation bottlenecks

---

[← Previous: 7.3 Shader Optimization](07-03-Shader-Optimization.md) | [Next: 7.5 Audio Asset Optimization →](07-05-Audio-Asset-Optimization.md)
