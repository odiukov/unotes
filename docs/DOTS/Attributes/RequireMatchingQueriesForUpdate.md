---
tags:
  - attribute
---
#### Description
- **System optimization attribute** that prevents system `OnUpdate` from running unless the system's queries match at least one [[Entity]]

- Applied to [[ISystem]] to automatically disable the system when no entities exist matching its queries

- Replaces manual `RequireForUpdate<ComponentType>()` calls for simple cases - Unity analyzes all queries in OnUpdate and only runs system if matches exist

- Improves **performance by skipping systems** that have no work to do, avoiding unnecessary overhead

#### Example
```csharp
// Without RequireMatchingQueriesForUpdate - runs every frame even if no entities
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // This runs every frame, even if no entities have Velocity
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
        // If no entities match query, loop body never executes but system still ran
    }
}

// With RequireMatchingQueriesForUpdate - skips OnUpdate if no matching entities
[RequireMatchingQueriesForUpdate]
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // This only runs if at least one entity has Velocity + LocalTransform
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}

// Example with initialization query - system with companion GameObjects
[RequireMatchingQueriesForUpdate]
[BurstCompile]
public partial struct PresentationGoInitialisationSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Only runs when new entities with PresentationGo + no PresentationGoCleanup exist
        foreach (var (presentationGo, entity) in
            SystemAPI.Query<PresentationGo>()
            .WithNone<PresentationGoCleanup>()
            .WithEntityAccess())
        {
            // Instantiate GameObject companion
            GameObject instance = Object.Instantiate(presentationGo.Prefab);
            state.EntityManager.AddComponentObject(entity, new PresentationGoCleanup
            {
                Instance = instance
            });
        }
    }
}
```

#### Pros
- **Automatic optimization** - no manual `RequireForUpdate` calls needed for simple cases

- **Performance improvement** - systems skip entirely when no matching entities, saving CPU time

- **Clean code** - one attribute instead of multiple `RequireForUpdate` calls in OnCreate

- **Query-based** - automatically analyzes all queries in OnUpdate, no need to manually specify components

#### Cons
- **All queries must match** - if you have multiple queries, system only runs if ALL have matches (AND condition, not OR)

- **Doesn't work with manual queries** - only works with queries created in OnUpdate, not queries stored as system fields

- **Less flexible** - for complex conditions (OR logic, singleton checks), manual `RequireForUpdate` is better

#### Best use
- **Simple entity processing systems** - when system has one or two queries and should skip if no entities match

- **Initialization systems** - systems that only run when new entities with specific components appear

- **Performance optimization** - when you have many systems that may have no work some frames (enemy AI when no enemies, projectile systems when no projectiles)

#### Avoid if
- **Complex update conditions** - if system should run based on singleton existence, OR conditions, or multiple independent queries

- **Manual EntityQuery fields** - if storing queries as system fields and using them across multiple methods

- **Needs to run every frame** - if system must always run (input processing, time tracking, etc.) don't use this attribute

#### Extra tip
- **Combines with manual RequireForUpdate** - you can use both `[RequireMatchingQueriesForUpdate]` AND manual `state.RequireForUpdate<T>()` calls - system runs only if both conditions are satisfied

- **Singleton requirements** - common pattern: `[RequireMatchingQueriesForUpdate]` for entity queries + manual `state.RequireForUpdate<SingletonComponent>()` for singleton checks in OnCreate

- **SystemAPI.Query only** - attribute analyzes `SystemAPI.Query<T>()` calls in OnUpdate, doesn't analyze manual EntityQuery fields or queries created in OnCreate

- **Multiple queries = AND logic** - if OnUpdate has multiple different queries, system runs only if ALL queries have at least one match

- **Performance measurement** - use Unity Profiler to verify systems are being skipped - look for systems not appearing in frame when no entities match

- **WithNone/WithAll patterns** - attribute respects query filters like `.WithNone<T>()` and `.WithAll<T>()` when determining if queries match

- **ISystemStartStop alternative** - if you need more complex enable/disable logic, implement `ISystemStartStop` interface (OnStartRunning/OnStopRunning callbacks)

- **Default behavior** - without this attribute, systems run every frame regardless of whether queries match, only the query loop body is skipped
