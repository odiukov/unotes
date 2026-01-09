---
tags:
  - pattern
  - system
---
#### Description
- **Convenience API** with static methods covering functionality from [[World]], [[EntityManager]], and SystemState - unified interface for common ECS operations

- **Source-generated** - methods work in [[ISystem]] and [[IJobEntity]] through compile-time code generation for optimal performance

- **Preferred entry point** - check SystemAPI first before EntityManager or SystemState for most operations

- **Automatic dependency tracking** - methods automatically register component access for [[System Dependencies|dependency management]]

#### Example
```csharp
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Time access
        float deltaTime = SystemAPI.Time.DeltaTime;

        // Query iteration
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * deltaTime;
        }

        // Singleton access
        var config = SystemAPI.GetSingleton<GameConfig>();
        SystemAPI.SetSingleton(new GameConfig { Speed = 10f });

        if (SystemAPI.HasSingleton<PlayerInput>())
        {
            Entity entity = SystemAPI.GetSingletonEntity<GameConfig>();
        }

        // Component lookup
        var healthLookup = SystemAPI.GetComponentLookup<Health>(isReadOnly: true);

        // Buffer lookup
        var waypointLookup = SystemAPI.GetBufferLookup<Waypoint>(isReadOnly: false);

        // Query builder
        var query = SystemAPI.QueryBuilder()
            .WithAll<Health>()
            .WithNone<Dead>()
            .Build();
    }
}
```

**Common methods:**
- `SystemAPI.Time.DeltaTime` - frame delta time
- `SystemAPI.Query<T>()` - iterate entities
- `SystemAPI.GetSingleton<T>()` - read singleton
- `SystemAPI.GetSingletonRW<T>()` - write singleton
- `SystemAPI.GetComponentLookup<T>()` - random access
- `SystemAPI.QueryBuilder()` - build custom queries

#### Pros
- **Unified API** - same methods work in systems and jobs, easy code sharing

- **Auto dependency tracking** - automatically registers component access

- **Cleaner code** - reduces boilerplate vs manual SystemState/EntityManager

- **Type-safe** - compile-time checking through source generation

- **Consistent** - uniform API across different contexts

#### Cons
- **Limited contexts** - doesn't work in IJobChunk, must use ComponentTypeHandle

- **Source generation requirement** - relies on compile-time codegen, IDE issues possible

- **Less explicit** - hides underlying mechanisms vs direct EntityManager

- **Learning curve** - need to learn which methods available

#### Best use
- **ISystem implementations** - primary API for OnUpdate methods

- **IJobEntity jobs** - consistent API between systems and jobs

- **Quick prototyping** - fast iteration without boilerplate

- **Standard operations** - queries, singletons, lookups

#### Avoid if
- **IJobChunk** - SystemAPI not available, use ComponentTypeHandle

- **Need explicit control** - if need fine-grained EntityManager control

- **Source gen issues** - if IDE/build having code generation problems

#### Extra tip
- **Works in jobs**: SystemAPI methods work identically in IJobEntity jobs

- **Dependency tracking**: SystemAPI automatically adds dependencies, EntityManager doesn't

- **Fallback to state**: If SystemAPI doesn't have method, use `ref SystemState state` parameter

- **Burst compatible**: All SystemAPI methods Burst-compatible

- **Query caching**: SystemAPI.Query creates new query each call, cache manually for hot paths

- **Time access**: `SystemAPI.Time` provides DeltaTime, ElapsedTime, frameCount

- **Best practices**: Prefer SystemAPI over EntityManager for standard operations, use EntityManager for advanced scenarios
