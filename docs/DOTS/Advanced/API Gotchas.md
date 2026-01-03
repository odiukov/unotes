---
tags:
  - advanced
  - pitfalls
---
#### Description
- **Non-obvious API behaviors** and common pitfalls in Unity DOTS that can cause performance issues, memory problems, or unexpected behavior

- Understanding these gotchas **prevents subtle bugs** and helps choose the right API overload for your use case

- Focus on **EntityManager** methods with multiple overloads that behave very differently

#### EntityManager.AddComponent(EntityQuery) and RemoveComponent(EntityQuery)

**The Problem:**
The `EntityQuery` overload modifies chunks themselves to match target [[Archetype]], rather than moving entities between chunks. This is **extremely efficient** for large entity counts but creates **empty chunks** when used frequently on small entity counts.

**How It Works:**
```csharp
// EntityQuery overload: Modifies source chunk archetype
EntityManager.AddComponent<NewComponent>(entityQuery);
// Result: Original chunk now has NewComponent archetype
// Entities stay in same chunk, chunk archetype changed

// Entity overload: Moves entity to different chunk
EntityManager.AddComponent<NewComponent>(singleEntity);
// Result: Entity moves to chunk with NewComponent archetype
// Original chunk unchanged
```

**The Gotcha:**
```csharp
// Scenario: Adding component to 1 entity at a time using EntityQuery
var query = SystemAPI.QueryBuilder().WithAll<Enemy>().Build();

// Frame 1: 1 enemy entity matches query in Chunk A
EntityManager.AddComponent<Dead>(query);
// Chunk A archetype changed to [Enemy, Dead]
// Chunk A now has 1 entity

// Frame 2: New enemy spawns in Chunk B (archetype [Enemy])
// Again, 1 entity matches query
EntityManager.AddComponent<Dead>(query);
// Chunk B archetype changed to [Enemy, Dead]
// Chunk B now has 1 entity

// Problem: Now have 2 chunks with 1 entity each!
// Normal chunk capacity is 128 entities
// Memory: 1.5% utilized (2/256)
// Iteration performance: Terrible (jumping between chunks)

// After 100 frames: 100 chunks with 1 entity each = disaster
```

**Appropriate Use Cases:**

✅ **Good: One-time bulk operations**
```csharp
// Post-processing a SubScene after loading
var allEntities = SystemAPI.QueryBuilder().WithAll<FromSubScene>().Build();
EntityManager.AddComponent<Initialized>(allEntities);
// Efficient: Processes thousands of entities, modifies chunks in bulk
```

✅ **Good: Large batch operations**
```csharp
// Disabling all enemies at once
var allEnemies = SystemAPI.QueryBuilder().WithAll<Enemy>().Build();
if (allEnemies.CalculateEntityCountWithoutFiltering() > 100)
{
    EntityManager.AddComponent<Disabled>(allEnemies);
    // Efficient: Many entities affected, chunk modification is faster
}
```

❌ **Bad: Incremental processing**
```csharp
// Processing entities one at a time over many frames
var deadEnemies = SystemAPI.QueryBuilder()
    .WithAll<Enemy, Health>()
    .WithNone<Dead>()
    .Build();

// Every frame, a few enemies die
EntityManager.AddComponent<Dead>(deadEnemies);
// Creates empty chunks! Use EntityCommandBuffer instead
```

❌ **Bad: Instantiated entity post-processing**
```csharp
// Post-processing spawned entities
var newlySpawned = SystemAPI.QueryBuilder()
    .WithAll<Spawned>()
    .WithNone<Initialized>()
    .Build();

EntityManager.AddComponent<Initialized>(newlySpawned);
// Bad: Spawned entities trickle in, creates empty chunks
```

**Better Alternatives:**

**Use EntityCommandBuffer for incremental changes:**
```csharp
var ecb = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
    .CreateCommandBuffer(state.WorldUnmanaged);

foreach (var (health, entity) in
    SystemAPI.Query<RefRO<Health>>().WithEntityAccess())
{
    if (health.ValueRO.Current <= 0)
    {
        ecb.AddComponent<Dead>(entity);  // Moves entity properly
    }
}
```

**Use Entity overload for single entities:**
```csharp
EntityManager.AddComponent<Dead>(singleEntity);
// Properly moves entity to correct chunk
```

---

#### EntityManager.Instantiate(Entity, NativeArray<Entity>)

**The Problem:**
The `NativeArray` overload **does NOT instantiate** the `LinkedEntityGroup` (child entities), while the normal `Entity` overload does. This breaks prefabs with hierarchies.

**How It Works:**
```csharp
// Normal overload: Instantiates entity AND all linked entities
Entity instance = EntityManager.Instantiate(prefabEntity);
// Result: If prefab has children, all are instantiated

// NativeArray overload: Only instantiates PRIMARY entity
var instances = new NativeArray<Entity>(10, Allocator.Temp);
EntityManager.Instantiate(prefabEntity, instances);
// Result: Children NOT instantiated! Only root entity cloned 10 times
```

**The Gotcha:**
```csharp
// Prefab setup: Character entity with child entities
// Prefab Entity has LinkedEntityGroup:
//   [0] Character (root)
//   [1] Weapon (child)
//   [2] Shield (child)
//   [3] HealthBar (child)

// Using NativeArray overload
var characters = new NativeArray<Entity>(5, Allocator.Temp);
EntityManager.Instantiate(characterPrefab, characters);

// Result: 5 character root entities, NO weapons, shields, or health bars!
// LinkedEntityGroup was ignored
```

**Appropriate Use Cases:**

✅ **Good: Simple entities without children**
```csharp
// Spawning bullets (no children)
var bullets = new NativeArray<Entity>(100, Allocator.Temp);
EntityManager.Instantiate(bulletPrefab, bullets);
// Safe: Bullet archetype has no LinkedEntityGroup

// How to verify safety:
if (!EntityManager.HasComponent<LinkedEntityGroup>(prefabEntity))
{
    // Safe to use NativeArray overload
    EntityManager.Instantiate(prefabEntity, outputArray);
}
```

✅ **Good: Performance-critical spawning of known-simple entities**
```csharp
// Particle system spawning 1000 particles (verified to have no children)
var particles = new NativeArray<Entity>(1000, Allocator.TempJob);
EntityManager.Instantiate(particlePrefab, particles);
```

❌ **Bad: Prefabs from baking (unknown structure)**
```csharp
// Character prefab baked from GameObject
var instances = new NativeArray<Entity>(10, Allocator.Temp);
EntityManager.Instantiate(bakedCharacterPrefab, instances);
// DANGER: Baked prefabs often have child entities!
// Weapons, equipment, visual elements will be missing
```

❌ **Bad: Any prefab with hierarchy**
```csharp
// Enemy prefab with weapon attachments
var enemies = new NativeArray<Entity>(50, Allocator.Temp);
EntityManager.Instantiate(enemyPrefab, enemies);
// Enemies spawn without weapons!
```

**Better Alternatives:**

**Use normal overload for prefabs with children:**
```csharp
for (int i = 0; i < count; i++)
{
    Entity instance = EntityManager.Instantiate(prefabEntity);
    // Correctly instantiates all linked entities
}
```

**Use EntityCommandBuffer for deferred instantiation:**
```csharp
var ecb = SystemAPI.GetSingleton<BeginSimulationEntityCommandBufferSystem.Singleton>()
    .CreateCommandBuffer(state.WorldUnmanaged);

for (int i = 0; i < count; i++)
{
    Entity instance = ecb.Instantiate(prefabEntity);
    // Correctly handles LinkedEntityGroup
}
```

**Check for LinkedEntityGroup before using NativeArray overload:**
```csharp
if (EntityManager.HasComponent<LinkedEntityGroup>(prefabEntity))
{
    // Use normal overload
    for (int i = 0; i < count; i++)
        instances[i] = EntityManager.Instantiate(prefabEntity);
}
else
{
    // Safe to use bulk overload
    EntityManager.Instantiate(prefabEntity, instances);
}
```

---

## Summary Table

| API | Appropriate Use | Inappropriate Use | Alternative |
|-----|----------------|-------------------|-------------|
| `AddComponent(EntityQuery)` | Bulk one-time operations on 100+ entities | Incremental per-frame additions | `ECB.AddComponent(entity)` |
| `RemoveComponent(EntityQuery)` | Bulk scene post-processing | Processing spawned entities | `ECB.RemoveComponent(entity)` |
| `Instantiate(Entity, NativeArray)` | Simple entities without children | Baked prefabs or hierarchies | `Instantiate(entity)` in loop |

---

## Best Practices

1. **Default to Entity overloads** - use `AddComponent(entity)`, `Instantiate(entity)` unless you have verified reason to use bulk overloads

2. **Check LinkedEntityGroup** - before using `Instantiate(entity, array)`, verify prefab has no children:
   ```csharp
   if (!EntityManager.HasComponent<LinkedEntityGroup>(prefab))
       // Safe to use bulk instantiate
   ```

3. **Use EntityCommandBuffer for incremental changes** - avoids chunk fragmentation from EntityQuery overloads

4. **Profile chunk utilization** - use Entity Debugger to check for sparse chunks (many chunks with few entities)

5. **Bulk operations = manual entities only** - EntityQuery overloads are safe for manually created entities, risky for baked prefabs

## See Also

- [[EntityCommandBuffer]] - Deferred structural changes
- [[Baking]] - Understanding baked entity structure
- [[Chunk]] - Chunk-based memory layout
- [[Structural changes]] - Operations that modify archetypes
