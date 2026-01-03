---
tags:
  - pattern
---
#### Description
- **Main iteration pattern** for accessing [[Entity|entities]] and [[Component|components]] in [[ISystem]] OnUpdate methods

- Provides foreach-compatible query results with [[RefRO and RefRW|RefRO/RefRW]] component access, entity access, and query builder methods

- **Type-safe and ergonomic** - components specified as generic parameters, automatically generates underlying [[EntityQuery]]

- Supports filtering with `.WithAll<T>()`, `.WithNone<T>()`, `.WithAny<T>()`, and entity access with `.WithEntityAccess()`

#### Example
```csharp
[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // Basic query - RefRW for read-write, RefRO for read-only
        foreach (var (health, regen) in
            SystemAPI.Query<RefRW<Health>, RefRO<HealthRegen>>())
        {
            health.ValueRW.Current += regen.ValueRO.PointPerSec * deltaTime;
        }
    }
}

// Query with filters
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>()
                .WithAll<Active>()      // Only entities with Active tag
                .WithNone<Frozen>())    // Exclude entities with Frozen tag
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}

// Query with entity access
[BurstCompile]
public partial struct DestroySystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (health, entity) in
            SystemAPI.Query<RefRO<Health>>()
                .WithEntityAccess())  // Adds Entity to tuple
        {
            if (health.ValueRO.Current <= 0)
            {
                ecb.DestroyEntity(entity);
            }
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Query with aspect
[BurstCompile]
public partial struct ProjectileSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        foreach (var projectile in SystemAPI.Query<ProjectileAspect>())
        {
            projectile.Move(deltaTime);
        }
    }
}

// Query with DynamicBuffer
[BurstCompile]
public partial struct PathFollowSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, waypoints, pathIndex) in
            SystemAPI.Query<RefRW<LocalTransform>, DynamicBuffer<Waypoints>, RefRW<NextPathIndex>>())
        {
            if (pathIndex.ValueRO.Value >= waypoints.Length)
                continue;

            float3 targetPos = waypoints[pathIndex.ValueRO.Value].Position;
            // ... movement logic
        }
    }
}
```

#### Pros
- **Ergonomic syntax** - clean foreach loop instead of manual query creation and chunk iteration

- **Type safety** - components checked at compile time, errors if component types don't exist

- **Automatic query optimization** - Unity optimizes generated queries for performance

- **[[Burst]] compatible** - fully compatible with Burst compilation for maximum performance

- **Flexible filtering** - WithAll/WithNone/WithAny enable complex query logic

#### Cons
- **Less control than EntityQuery** - can't easily store query, get entity count, or use advanced query features

- **Hidden allocations** - query object created every frame (though usually optimized away)

- **Limited to OnUpdate** - works best in OnUpdate, not suitable for queries that need to be stored or reused

#### Best use
- **Simple entity iteration** - when you need to process all entities with specific components

- **Main gameplay systems** - movement, damage, AI, spawning - most common use case

- **Systems with one primary query** - when system does one main iteration task

#### Avoid if
- **Need to store query** - use manual [[EntityQuery]] field in system if you need to reuse query or check entity count

- **Complex multi-query logic** - if system needs multiple independent queries with different logic, manual queries may be clearer

- **Need query metadata** - if you need `CalculateEntityCount()`, `IsEmpty`, or other query properties, use EntityQuery

#### Extra tip
- **Query builder methods:**
  - `.WithAll<T>()` - only entities that have component T (can pass multiple types)
  - `.WithNone<T>()` - exclude entities that have component T
  - `.WithAny<T1, T2>()` - entities with at least one of the specified components
  - `.WithEntityAccess()` - adds Entity to the query tuple
  - `.WithChangeFilter<T>()` - only entities where T changed since last query

- **Component access patterns:**
  - `RefRW<T>` - read-write access (use `.ValueRW`)
  - `RefRO<T>` - read-only access (use `.ValueRO`)
  - `DynamicBuffer<T>` - buffer access for [[IBufferElementData (dynamic buffers)|IBufferElementData]]
  - `AspectType` - aspect access for [[IAspect (component grouping)|IAspect]]
  - `Entity` - entity ID (requires `.WithEntityAccess()`)

- **Alternative ref/in syntax** - in [[IJobEntity]], you can use `ref T` and `in T` instead of RefRW/RefRO, but SystemAPI.Query requires RefRW/RefRO

- **Multiple queries** - can have multiple `SystemAPI.Query` calls in same OnUpdate - each is independent

- **Performance tip** - query generation is optimized by compiler, minimal overhead compared to manual EntityQuery

- **EnabledRefRW/EnabledRefRO** - for [[IEnableableComponent (toggleable components)|enableable components]], query for enabled state instead of component data

- **Query caching** - Unity caches queries internally, repeated identical queries reuse same query object

- **Deconstruction** - use tuple deconstruction in foreach: `var (a, b, c)` for clean variable naming

- **WithOptions** - advanced filtering: `.WithOptions(EntityQueryOptions.IncludePrefab)` to include prefab entities in query

- **Shared component filtering** - can filter by shared component value (less common, check Unity docs for syntax)
