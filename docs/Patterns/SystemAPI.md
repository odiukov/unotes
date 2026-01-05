---
tags:
  - pattern
  - system
---

#### Description
- **Convenience API** with static methods covering functionality from [[World]], [[EntityManager]], and SystemState - provides unified interface for common ECS operations

- **Source-generated** - methods work in [[ISystem]] and [[IJobEntity]] (but not IJobChunk) through compile-time code generation for optimal performance

- **Preferred entry point** - check SystemAPI first before EntityManager or SystemState for most operations, ensuring consistent code between systems and jobs

- **Automatic dependency tracking** - methods called in systems automatically register component access for [[System Dependencies|dependency management]]

#### Example
```csharp
using Unity.Entities;
using Unity.Mathematics;

[BurstCompile]
public partial struct MovementSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Time access
        float deltaTime = SystemAPI.Time.DeltaTime;
        double elapsedTime = SystemAPI.Time.ElapsedTime;

        // Query iteration - most common pattern
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * deltaTime;
        }

        // Get singleton
        var config = SystemAPI.GetSingleton<GameConfig>();

        // Set singleton
        SystemAPI.SetSingleton(new GameConfig { Speed = 10f });

        // Check singleton existence
        if (SystemAPI.HasSingleton<PlayerInput>())
        {
            var input = SystemAPI.GetSingleton<PlayerInput>();
        }

        // Get entity from singleton
        Entity configEntity = SystemAPI.GetSingletonEntity<GameConfig>();

        // Component lookup
        var healthLookup = SystemAPI.GetComponentLookup<Health>(isReadOnly: true);
        if (healthLookup.TryGetComponent(someEntity, out Health health))
        {
            // Process health...
        }

        // Buffer lookup
        var waypointLookup = SystemAPI.GetBufferLookup<Waypoint>(isReadOnly: false);

        // Get entity query
        var query = SystemAPI.QueryBuilder()
            .WithAll<Health, Transform>()
            .WithNone<Dead>()
            .Build();
    }
}

// Works in IJobEntity too
[BurstCompile]
public partial struct DamageJob : IJobEntity
{
    public float DeltaTime;

    private void Execute(ref Health health, in DamageOverTime dot)
    {
        // SystemAPI works the same in jobs
        health.Current -= dot.DamagePerSecond * DeltaTime;
    }
}
```

#### Pros
- **Unified API** - same methods work in systems and jobs, enabling easy code copy-paste between contexts

- **Auto dependency tracking** - automatically registers component access for [[System Dependencies]], unlike direct EntityManager access

- **Cleaner code** - reduces boilerplate compared to manually accessing SystemState or EntityManager

- **Type-safe** - compile-time checking through source generation catches errors early

#### Cons
- **Limited contexts** - doesn't work in IJobChunk, must use ComponentTypeHandle and other lower-level APIs

- **Source generation requirement** - relies on compile-time code generation, can have IDE issues or confusion when generation fails

- **Less explicit** - hides underlying mechanisms compared to direct EntityManager/SystemState usage

- **Learning curve** - developers must learn SystemAPI patterns in addition to EntityManager patterns

#### Best use
- **System OnUpdate methods** - primary use case for iterating entities and accessing singletons

- **[[IJobEntity]] jobs** - enables consistent API between system main thread code and job code

- **Quick prototyping** - faster development with less boilerplate than manual EntityManager usage

#### Avoid if
- **IJobChunk jobs** - SystemAPI not available, must use ComponentTypeHandle, BufferTypeHandle, EntityTypeHandle directly

- **Need explicit control** - when you need fine-grained control over queries, type handles, or entity access patterns

- **Outside ECS systems** - SystemAPI methods require system context, won't work in regular C# classes

#### Extra tip
- **Priority of lookup:**
  1. Check **SystemAPI** first for common operations
  2. Check **SystemState** if SystemAPI doesn't have what you need
  3. Check **EntityManager** or **World** as last resort

- **Common SystemAPI patterns:**
  ```csharp
  // Query iteration (most common)
  foreach (var (health, transform) in SystemAPI.Query<RefRW<Health>, RefRO<Transform>>())
  {
      // Process...
  }

  // Singleton access
  var config = SystemAPI.GetSingleton<Config>();
  SystemAPI.SetSingleton(new Config { Value = 10 });
  bool exists = SystemAPI.HasSingleton<Config>();
  Entity e = SystemAPI.GetSingletonEntity<Config>();

  // Lookups for random access
  var lookup = SystemAPI.GetComponentLookup<Health>(isReadOnly: true);
  var bufferLookup = SystemAPI.GetBufferLookup<Waypoint>(isReadOnly: false);

  // Time access
  float dt = SystemAPI.Time.DeltaTime;
  double elapsed = SystemAPI.Time.ElapsedTime;

  // Query building
  var query = SystemAPI.QueryBuilder()
      .WithAll<A, B>()
      .WithAny<C, D>()
      .WithNone<E>()
      .Build();
  ```

- **Query with filters:**
  ```csharp
  foreach (var (health, transform) in
      SystemAPI.Query<RefRW<Health>, RefRO<Transform>>()
          .WithAll<Player>()
          .WithNone<Dead>())
  {
      // Only processes entities with Health, Transform, Player (not Dead)
  }
  ```

- **Aspect support:**
  ```csharp
  foreach (var characterAspect in SystemAPI.Query<CharacterAspect>())
  {
      characterAspect.Move(deltaTime);
  }
  ```

- **GetComponentLookup vs GetComponent:**
  - `GetComponentLookup<T>()` - returns lookup for random entity access by ID
  - `GetComponent<T>(entity)` - direct component access for specific entity

- **Source generation visibility** - generated code is in `Temp/GeneratedCode` folder, useful for debugging

- **RefRW vs RefRO** - use `RefRW<T>` for read-write access, `RefRO<T>` for read-only access in queries

## See Also

- [[SystemAPI.Query]] - Entity iteration patterns
- [[ISystem]] - Modern system implementation
- [[SystemState]] - System state and component access
- [[EntityManager]] - Direct entity manipulation
- [[IJobEntity]] - Job-based entity iteration
- [[ComponentLookup and BufferLookup]] - Random entity access
