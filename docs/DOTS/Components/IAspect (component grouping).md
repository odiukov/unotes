---
tags:
  - component
  - deprecated
---

> **⚠️ DEPRECATION WARNING (Unity Entities 6.5+)**
>
> `IAspect` is **deprecated** as of Unity Entities 6.5. Unity recommends using direct `Component` and `EntityQuery` APIs instead for clearer, more maintainable code patterns.
>
> **Alternative approaches:**
> - Use [[SystemAPI.Query]] with multiple component parameters
> - Create helper methods that accept component parameters instead of aspects
> - Use [[IJobEntity]] with component parameters
>
> Existing aspect code will continue to work but is not recommended for new projects.

#### Description
- **Wrapper for a set of components** that together provide specific behavior, allowing code organization and reusability across systems and jobs

- Defined as `readonly partial struct` implementing `IAspect`, containing [[RefRO and RefRW|RefRO/RefRW]] fields for component access and methods that operate on those components

- **Circumvents query limitations** - useful when a single query would exceed the maximum number of component types

- Automatically generates query code - when you query for an aspect, Unity generates the underlying [[Entity|entity]] query for all required components

#### Example
```csharp
// Define an aspect grouping projectile-related components
public readonly partial struct ProjectileAspect : IAspect
{
    // Read-only access to components
    readonly RefRO<Guided> _guided;
    readonly RefRO<Speed> _speed;
    readonly RefRO<Target> _target;

    // Read-write access to transform
    readonly RefRW<LocalTransform> _transform;

    // Methods operating on the grouped components
    public void Move(float time, ComponentLookup<LocalToWorld> positions)
    {
        // Access read-only data with .ValueRO
        if (_guided.ValueRO.Enabled && positions.HasComponent(_target.ValueRO.Value))
        {
            // Access read-write data with .ValueRW
            _transform.ValueRW.Rotation = TransformHelpers.LookAtRotation(
                _transform.ValueRW.Position,
                positions[_target.ValueRO.Value].Position,
                _transform.ValueRW.Up());
        }

        _transform.ValueRW.Position +=
            _speed.ValueRO.value * time * _transform.ValueRW.Forward();
    }
}

// Use in IJobEntity
public partial struct ProjectileMoveJob : IJobEntity
{
    public float DeltaTime;
    [ReadOnly] public ComponentLookup<LocalToWorld> Positions;

    // Aspect as parameter - Unity generates query automatically
    private void Execute(ProjectileAspect projectile)
    {
        projectile.Move(DeltaTime, Positions);
    }
}
```

#### Pros
- **Code reusability** - behavior defined once in aspect, used across multiple systems and jobs

- **Better organization** - groups related components together with their behavior logic

- **Cleaner queries** - `SystemAPI.Query<ProjectileAspect>()` instead of multi-component tuples

- **Bypasses query limits** - aspects don't count toward the maximum component types per query

- **Compile-time safety** - ensures all required components are present in queries

#### Cons
- **Additional abstraction layer** - adds complexity compared to direct component access

- **Cannot include [[Tag Component|tag components]]** - only works with data-containing components

- **Learning curve** - requires understanding RefRO/RefRW access patterns

#### Best use
- **Complex entity behaviors** - when an entity's behavior requires 4+ components working together (movement, combat, AI, etc.)

- **Reusable logic** - when the same component combination and behavior is used in multiple systems

- **Large queries** - when direct component queries would exceed Unity's per-query component limit

#### Avoid if
- **Simple single-component operations** - use direct [[IComponentData]] access with `SystemAPI.Query<MyComponent>()`

- **Tag components are involved** - aspects don't support tags, use regular queries with `WithAll<TagComponent>()`

- **One-off behavior** - if the logic is only used in one place, direct component access is simpler

#### Extra tip
- **Nested aspects** - aspects can contain other aspects as fields for hierarchical component grouping

- **Optional components** - use `[Optional]` attribute on RefRO/RefRW fields for components that may not be present

- **DynamicBuffer support** - aspects can contain `DynamicBuffer<T>` fields (use `[ReadOnly]` attribute for read-only buffers)

- **Access patterns** - use `.ValueRO` for read-only access, `.ValueRW` for read-write access (analogous to `ref readonly` and `ref`)

- **IEnableableComponent support** - use `EnabledRefRO` and `EnabledRefRW` fields to access enabled state of [[IEnableableComponent (toggleable components)|enableable components]]

---

## Modern Alternatives (Unity Entities 6.5+)

Since `IAspect` is deprecated, here are recommended alternatives:

### Alternative 1: Helper Methods with Component Parameters
```csharp
// Instead of aspect, use static helper methods
public static class ProjectileHelpers
{
    public static void Move(
        ref LocalTransform transform,
        in Speed speed,
        in Guided guided,
        in Target target,
        float deltaTime,
        ComponentLookup<LocalToWorld> positions)
    {
        if (guided.Enabled && positions.HasComponent(target.Value))
        {
            transform.Rotation = TransformHelpers.LookAtRotation(
                transform.Position,
                positions[target.Value].Position,
                transform.Up());
        }
        transform.Position += speed.value * deltaTime * transform.Forward();
    }
}

// Use in IJobEntity
public partial struct ProjectileMoveJob : IJobEntity
{
    public float DeltaTime;
    [ReadOnly] public ComponentLookup<LocalToWorld> Positions;

    private void Execute(
        ref LocalTransform transform,
        in Speed speed,
        in Guided guided,
        in Target target)
    {
        ProjectileHelpers.Move(ref transform, speed, guided, target, DeltaTime, Positions);
    }
}
```

### Alternative 2: Direct Component Access in SystemAPI.Query
```csharp
[BurstCompile]
public partial struct ProjectileMoveSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var positions = SystemAPI.GetComponentLookup<LocalToWorld>(true);
        float dt = SystemAPI.Time.DeltaTime;

        // Direct multi-component query instead of aspect
        foreach (var (transform, speed, guided, target) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Speed>, RefRO<Guided>, RefRO<Target>>())
        {
            if (guided.ValueRO.Enabled && positions.HasComponent(target.ValueRO.Value))
            {
                transform.ValueRW.Rotation = TransformHelpers.LookAtRotation(
                    transform.ValueRO.Position,
                    positions[target.ValueRO.Value].Position,
                    transform.ValueRO.Up());
            }
            transform.ValueRW.Position += speed.ValueRO.value * dt * transform.ValueRO.Forward();
        }
    }
}
```

### Alternative 3: Component-Based Methods
```csharp
// Extension methods on components
public static class LocalTransformExtensions
{
    public static void MoveTowards(
        this ref LocalTransform transform,
        float3 target,
        float speed,
        float deltaTime)
    {
        float3 direction = math.normalize(target - transform.Position);
        transform.Position += direction * speed * deltaTime;
        transform.Rotation = quaternion.LookRotation(direction, math.up());
    }
}

// Usage in job
private void Execute(
    ref LocalTransform transform,
    in Speed speed,
    in Target target,
    ComponentLookup<LocalToWorld> positions)
{
    if (positions.TryGetComponent(target.Value, out LocalToWorld targetPos))
    {
        transform.MoveTowards(targetPos.Position, speed.value, deltaTime);
    }
}
```
