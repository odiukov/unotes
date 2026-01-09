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
// Basic query - RefRW for read-write, RefRO for read-only
[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (health, regen) in
            SystemAPI.Query<RefRW<Health>, RefRO<HealthRegen>>())
        {
            health.ValueRW.Current += regen.ValueRO.PointPerSec * dt;
        }
    }
}

// Query with filters
foreach (var (transform, velocity) in
    SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>()
        .WithAll<Active>()      // Only entities with Active
        .WithNone<Frozen>())    // Exclude Frozen
{
    transform.ValueRW.Position += velocity.ValueRO.Value * dt;
}

// Query with entity access
foreach (var (health, entity) in
    SystemAPI.Query<RefRO<Health>>()
        .WithEntityAccess())  // Adds Entity to tuple
{
    if (health.ValueRO.Current <= 0)
        ecb.DestroyEntity(entity);
}

// Query with aspect
foreach (var projectile in SystemAPI.Query<ProjectileAspect>())
{
    projectile.Move(deltaTime);
}

// Query with DynamicBuffer
foreach (var (transform, waypoints, pathIndex) in
    SystemAPI.Query<RefRW<LocalTransform>, DynamicBuffer<Waypoint>, RefRW<PathIndex>>())
{
    // Access waypoints buffer...
}

// Multiple filters
foreach (var (health, damage) in
    SystemAPI.Query<RefRW<Health>, RefRO<Damage>>()
        .WithAll<Enemy>()           // Has Enemy
        .WithNone<Dead, Invincible>()  // Not Dead or Invincible
        .WithAny<Burning, Poisoned>()) // Has Burning or Poisoned
{
    // Process...
}

// Change filter
foreach (var (health, healthBar) in
    SystemAPI.Query<RefRO<Health>, HealthBarUI>()
        .WithChangeFilter<Health>())  // Only when Health changed
{
    healthBar.UpdateBar(health.ValueRO.Current);
}
```

**Query syntax:**
- `RefRW<T>` - read-write component access
- `RefRO<T>` - read-only component access
- `DynamicBuffer<T>` - buffer access
- `IAspect` - aspect access
- `.WithEntityAccess()` - adds Entity to tuple
- `.WithAll<T>()` - require component
- `.WithNone<T>()` - exclude component
- `.WithAny<T>()` - require any of components
- `.WithChangeFilter<T>()` - only changed

#### Pros
- **Ergonomic** - clean foreach syntax, no boilerplate

- **Type-safe** - components as generic parameters, compile-time checked

- **Fast** - generates efficient EntityQuery, [[Burst Compile|Burst]] compatible

- **Automatic dependencies** - registers component access for [[System Dependencies]]

- **Flexible** - supports components, buffers, aspects, entity access

#### Cons
- **Creates query each call** - not cached, overhead for hot paths

- **Limited to foreach** - can't use LINQ or other patterns

- **No manual batching** - less control than IJobChunk

- **Tuple limit** - max ~7-8 components in query (compiler limit)

#### Best use
- **Standard iteration** - most common system pattern

- **Prototype/development** - quick iteration without boilerplate

- **Simple systems** - straightforward component processing

- **Component updates** - reading/writing component data

#### Avoid if
- **Hot paths** - cache EntityQuery manually for performance-critical loops

- **Complex iteration** - if need manual batching, use IJobChunk

- **Many components** - if querying 8+ components, restructure or use aspects

#### Extra tip
- **Cache for performance**: For hot paths, cache query manually:
  ```csharp
  private EntityQuery _query;
  public void OnCreate(ref SystemState state)
  {
      _query = SystemAPI.QueryBuilder().WithAll<Health>().Build();
  }
  ```

- **Tuple deconstruction**: Can use `var (a, b)` or explicit types `(RefRW<Health> health, Entity entity)`

- **Managed components**: Use managed component types directly (not RefRW/RefRO):
  ```csharp
  SystemAPI.Query<Transform, ManagedData>()
  ```

- **Options parameter**: Advanced options like `EntityQueryOptions.FilterWriteGroup`

- **Combining filters**: Multiple `.WithAll()` calls are AND logic

- **WithAny vs WithAll**: `WithAny<A, B>()` = has A OR B, `WithAll<A, B>()` = has A AND B

- **Order matters**: Filter order doesn't affect results, but entity access must be last in tuple

- **Best practices**: Use RefRO for read-only, RefRW for write; cache queries for hot paths; use aspects for related components
