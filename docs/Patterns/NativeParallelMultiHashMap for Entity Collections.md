---
tags:
  - pattern
---
#### Description
- **Collection storage pattern** using `NativeParallelMultiHashMap<TKey, TValue>` to store one-to-many relationships between entities, most commonly parent-to-children mappings

- Allows storing multiple values (children entities) per key (parent entity) with efficient parallel-safe access, perfect for hierarchies and groupings

- Standard iteration pattern uses `TryGetFirstValue()` followed by `do-while` loop with `TryGetNextValue()` to iterate all values for a given key

- [[Thread-safe]] parallel writes and reads make it ideal for jobs that need to build or query entity relationships

#### Example
```csharp
// Build parent-to-children mapping
[BurstCompile]
public partial struct BuildParentChildMapJob : IJobEntity
{
    public NativeParallelMultiHashMap<Entity, Entity>.ParallelWriter ChildMap;

    [BurstCompile]
    public void Execute(Entity entity, in Parent parent)
    {
        // Add child entity to parent's list
        ChildMap.Add(parent.Value, entity);
    }
}

[BurstCompile]
public partial struct ParentChildSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Create map with estimated capacity (parent count)
        var childMap = new NativeParallelMultiHashMap<Entity, Entity>(
            100,  // Estimated capacity
            Allocator.TempJob);

        // Build map in parallel
        new BuildParentChildMapJob
        {
            ChildMap = childMap.AsParallelWriter()
        }.ScheduleParallel();

        state.Dependency.Complete();

        // Iterate all children for each parent
        foreach (var (parentEntity, transform) in
            SystemAPI.Query<RefRO<LocalTransform>>()
                .WithAll<Parent>()
                .WithEntityAccess())
        {
            // Check if parent has children
            if (childMap.TryGetFirstValue(parentEntity, out Entity child, out var iterator))
            {
                // Process first child
                ProcessChild(ref state, child);

                // Iterate remaining children
                while (childMap.TryGetNextValue(out child, ref iterator))
                {
                    ProcessChild(ref state, child);
                }
            }
        }

        childMap.Dispose();
    }

    private static void ProcessChild(ref SystemState state, Entity child)
    {
        // Process child entity
    }
}

// Example: Group entities by type
public struct EnemyType : IComponentData
{
    public int TypeID;
}

[BurstCompile]
public partial struct GroupEnemiesByTypeSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Map from TypeID to list of enemy entities
        var enemiesByType = new NativeParallelMultiHashMap<int, Entity>(
            100,
            Allocator.TempJob);

        // Build map
        foreach (var (enemyType, entity) in
            SystemAPI.Query<RefRO<EnemyType>>()
                .WithEntityAccess())
        {
            enemiesByType.Add(enemyType.ValueRO.TypeID, entity);
        }

        // Process all enemies of type 1
        if (enemiesByType.TryGetFirstValue(1, out Entity enemy, out var iterator))
        {
            do
            {
                // Process enemy of type 1
            }
            while (enemiesByType.TryGetNextValue(out enemy, ref iterator));
        }

        enemiesByType.Dispose();
    }
}

// Example: Spatial hashing for collision detection
[BurstCompile]
public partial struct SpatialHashingSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Map from grid cell to entities in that cell
        var spatialHash = new NativeParallelMultiHashMap<int2, Entity>(
            1000,
            Allocator.TempJob);

        var spatialHashWriter = spatialHash.AsParallelWriter();

        // Build spatial hash in parallel
        new HashEntitiesJob
        {
            SpatialHash = spatialHashWriter
        }.ScheduleParallel();

        state.Dependency.Complete();

        // Query entities in specific cell
        int2 cellToQuery = new int2(5, 5);

        if (spatialHash.TryGetFirstValue(cellToQuery, out Entity entity, out var iterator))
        {
            do
            {
                // Process entities in same cell (potential collision)
            }
            while (spatialHash.TryGetNextValue(out entity, ref iterator));
        }

        spatialHash.Dispose();
    }
}

[BurstCompile]
public partial struct HashEntitiesJob : IJobEntity
{
    public NativeParallelMultiHashMap<int2, Entity>.ParallelWriter SpatialHash;

    [BurstCompile]
    public void Execute(Entity entity, in LocalTransform transform)
    {
        // Convert world position to grid cell
        int2 cell = new int2(
            (int)math.floor(transform.Position.x / 10f),
            (int)math.floor(transform.Position.z / 10f));

        SpatialHash.Add(cell, entity);
    }
}

// Example: Using statement for automatic disposal
[BurstCompile]
public partial struct UsingStatementExampleSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Using statement ensures disposal even if exception thrown
        using var childMap = new NativeParallelMultiHashMap<Entity, Entity>(
            100,
            Allocator.TempJob);

        // Build and use map...
        foreach (var (parent, entity) in
            SystemAPI.Query<RefRO<Parent>>()
                .WithEntityAccess())
        {
            childMap.Add(parent.ValueRO.Value, entity);
        }

        // Process map...
        // Automatically disposed at end of using block
    }
}
```

#### Pros
- **One-to-many relationships** - natural fit for parent-children, groups, spatial hashing, and other 1:N relationships

- **[[Thread-safe]] parallel access** - `ParallelWriter` allows multiple jobs to write simultaneously without conflicts

- **Efficient iteration** - fast sequential access to all values for a given key using iterator pattern

- **Flexible key types** - works with any unmanaged type (Entity, int, int2, float3, custom structs)

- **Dynamic sizing** - automatically grows capacity as needed, though initial capacity helps performance

#### Cons
- **Manual disposal required** - must call `.Dispose()` or use `using` statement to prevent memory leaks

- **Unordered values** - values for a key are not guaranteed to be in insertion order

- **No duplicate prevention** - same key-value pair can be added multiple times if not careful

- **Memory overhead** - hash table overhead compared to simple arrays, especially for small collections

- **Not Burst-compatible in some contexts** - creation/disposal must happen in main thread context in some Unity versions

#### Best use
- **Parent-child hierarchies** - build temporary maps of parent to children for hierarchy traversal

- **Entity grouping** - group entities by type, team, faction, state for batch processing

- **Spatial hashing** - map grid cells to entities for efficient spatial queries and collision detection

- **Dependency tracking** - map entities to their dependents for propagating changes

- **Temporary lookups** - build lookup tables for single-frame processing then dispose

#### Avoid if
- **Need persistent storage** - use [[IBufferElementData (dynamic buffers)|DynamicBuffer]] on entities for persistent one-to-many relationships

- **Simple 1:1 mapping** - use `NativeHashMap` (not multi) for one-to-one relationships

- **Need ordered values** - hash map doesn't preserve order, use [[IBufferElementData (dynamic buffers)|DynamicBuffer]] if order matters

- **Very small collections** - for <10 items, linear search in array may be faster than hash lookups

#### Extra tip
- **Standard iteration pattern** - always use `TryGetFirstValue` + `do-while` + `TryGetNextValue`:
  ```csharp
  if (map.TryGetFirstValue(key, out var value, out var iterator))
  {
      do
      {
          // Process value
      }
      while (map.TryGetNextValue(out value, ref iterator));
  }
  ```

- **Using statement for cleanup** - wrap in `using` for automatic disposal:
  ```csharp
  using var map = new NativeParallelMultiHashMap<Entity, Entity>(100, Allocator.TempJob);
  // Automatically disposed at end of scope
  ```

- **Capacity estimation** - set initial capacity to approximate expected size to avoid costly resizes:
  ```csharp
  int estimatedParents = query.CalculateEntityCount();
  var map = new NativeParallelMultiHashMap<Entity, Entity>(estimatedParents, Allocator.TempJob);
  ```

- **Parallel writer usage** - always use `.AsParallelWriter()` in parallel jobs:
  ```csharp
  new MyJob { Map = childMap.AsParallelWriter() }.ScheduleParallel();
  ```

- **ContainsKey check** - use `ContainsKey(key)` to check if key exists before iteration

- **Count values per key** - use `CountValuesForKey(key)` to get number of values for a key without iterating

- **Remove operations** - can remove specific key-value pairs with `Remove(key, value)` or all values for key with `Remove(key)`

- **Clear vs Dispose** - `Clear()` removes all items but keeps capacity, `Dispose()` frees memory

- **Allocator choice:**
  - `Allocator.Temp` - single frame, main thread only (fastest)
  - `Allocator.TempJob` - up to 4 frames, job-safe (most common)
  - `Allocator.Persistent` - long-lived, must manually dispose

- **Thread safety** - reads and parallel writes are safe, but sequential writes from different jobs need synchronization

- **Key uniqueness** - can add same key multiple times with different values (that's the "Multi" in MultiHashMap)

- **Performance tips:**
  - Pre-allocate with estimated capacity
  - Use Allocator.TempJob for job usage
  - Dispose immediately after use to free memory
  - Avoid frequent allocations, reuse if possible

- **Debugging** - check `Count()` and `Capacity` properties to verify expected size and catch leaks

- **Alternative collections:**
  - `NativeHashMap<K,V>` - one value per key
  - `NativeList<T>` - simple dynamic array
  - `DynamicBuffer<T>` - per-entity dynamic arrays
  - `NativeParallelHashSet<T>` - set of unique values

- **Unity packages using this** - Unity.Transforms and Unity.Physics use this pattern extensively for parent-child and collision queries
