# 23.3 ECS and DOTS

[← Back to Chapter 23](../chapters/23-Multithreading-And-Jobs-System.md) | [Main Index](../README.md)

ECS (Entity Component System) + DOTS (Data-Oriented Tech Stack) = data-oriented architecture (organize data for cache efficiency + parallelism). Traditional: GameObjects with MonoBehaviour scripts (object-oriented, scattered data = poor cache locality). DOTS: entities (pure data containers), components (data blobs), systems (logic over data). Benefits: massive parallelism (systems operate on component arrays = perfect for IJobParallelFor), cache-efficient (data grouped = fewer cache misses = 10-100x speedup on large datasets).

---

## Overview

Entity Component System: Entities = unique IDs (no data), Components = data (transform, velocity, health), Systems = logic (iterate entities with specific components, update data). Example: PhysicsSystem iterates all (Position, Velocity) components, updates positions (cache-friendly = all positions contiguous = L3 cache hit rate high). Contrast: GameObject iteration (each has independent script = scattered pointers = cache misses = slow).

DOTS: Data-Oriented Tech Stack = ECS framework (Unity.Entities package), Burst compiler (native code generation from ECS systems), Collections (NativeArray-backed data structures). Adoption: optional (traditional MonoBehaviour still works), recommended for performance-critical (physics, animation, large entity counts). Learning curve: mental model shift (data-first vs object-first = harder initially, much faster long-term).

## Key Concepts

- **Entity Component System (ECS)**: Entities = unique 64-bit IDs (just identifiers, no data), Components = data structs (Position struct = x, y, z), Systems = IComponentData implementations (iterate component arrays, update data). Entity creation: `var entity = EntityManager.CreateEntity(typeof(Position), typeof(Velocity));`, data access: `EntityManager.SetComponentData(entity, new Position { Value = new float3(0, 0, 0) });`. Iteration: SystemBase (MonoBehaviour-like, but iterates component arrays) or Entities.ForEach (LINQ-like iteration). Benefit: entity-level flexibility (add/remove components at runtime = change entity properties), data-level efficiency (components grouped = cache-friendly).
- **Data-Oriented Design**: Organize data by access pattern (cache-friendly = fewer misses = fast). Traditional: class-based (each GameObject = pointer to scattered heap objects = cache misses). DOTS: array-of-structs (all positions in array = contiguous = cache line = fast). Example: 10000 entities, update position = traditional 10000 cache misses (each GameObject pointer different cache line), DOTS 10000/64 cache misses (64 positions per cache line = massive speedup). Trade-off: less object-oriented (no inheritance, methods scattered as systems = different mental model).
- **SystemBase**: ECS system (iterates components, executes logic). Structure: class inheriting SystemBase, OnUpdate() executes per frame. Entities.ForEach: LINQ-like (query components, run job per matching entity). Example: `Entities.WithoutBurst().ForEach((in Translation trans, in Velocity vel) => { /* update */ }).ScheduleParallel();`. Automatic burst compilation (Burst compiles ForEach lambda = native code). Advantage: concise, auto-parallelization (ForEach = automatically scheduled on worker threads).
- **Chunk Architecture**: ECS internally groups entities with same component set into chunks (16KB blocks). Query optimization: query components (Position, Velocity) = finds chunks with both = iterates chunk arrays only (not all entities = faster). Archetype = component set = chunk template. Chunk iteration = SystemBase.Entities iteration = fast because contiguous.

## Best Practices

**ECS Architecture**: Entity design (entities = lightweight ID only, components = data, systems = behavior), avoid entities with methods/components with logic. Component granularity: small components (Position, Velocity, Health = separate), benefit = systems iterate only relevant data. System organization: group related systems, control order (dependencies), batch updates (all physics, then all animation = predictable).

**Performance Optimization**: Profile cache behavior (Profiler > Entities tab = measure improvement). Burst-compatible systems (Entities.ForEach = Burst by default = fast). Avoid: managed allocations in system, LINQ (creates intermediate collection = slow), GameObject access (not parallel-safe). Migration: identify bottlenecks (profile = find slow systems), port incrementally (physics first = biggest win), keep UI/logic as MonoBehaviour.

**Platform-Specific**: PC (full ECS support, Burst x64 = multithreading full 8+ cores), Consoles (full support, ARM64), Mobile (ECS beneficial = limited cores, cache optimization critical, Burst mandatory).

## Common Pitfalls

**ECS Overkill**: Developer converts entire game to ECS (UI, menus = not performance-sensitive). Overly complex: ECS benefits large-scale data (thousands of entities), small counts = no benefit, traditional clearer. Solution: use ECS only for performance-critical, keep UI/dialogue as traditional GameObjects.

**Burst Incompatibility in ECS**: Developer writes ECS system (allocates List inside = managed allocation), Burst fails. System runs but slow (fallback to managed = defeats purpose). Solution: use NativeCollections only, no managed allocations.

## Tools & Workflow

**SystemBase Example**:
```csharp
public partial class PhysicsSystem : SystemBase {
    protected override void OnUpdate() {
        float dt = SystemAPI.Time.DeltaTime;
        Entities.ForEach((ref Translation trans, in Velocity vel) => {
            trans.Value += vel.Value * dt;
        }).ScheduleParallel();
    }
}
```

**Profiler Entities Tab**: Window > Analysis > Profiler > Entities tab (shows chunk count, archetype distribution = verify cache-efficient grouping).

## Related Topics

- [23.1 Unity Jobs System](23-01-Unity-Jobs-System.md) - Job scheduling, Burst
- [23.2 Burst Compiler](23-02-Burst-Compiler.md) - ECS compilation, optimization
- [23.4 Thread-Safe Graphics Operations](23-04-Thread-Safe-Graphics-Operations.md) - Graphics + ECS
- [06.1 CPU Architecture](06-01-CPU-Bottlenecks.md) - Cache locality, performance

---

[Previous: 23.2 Burst Compiler](23-02-Burst-Compiler.md) | [Next: 23.4 Thread-Safe Graphics Operations →](23-04-Thread-Safe-Graphics-Operations.md)
