---
tags:
  - pattern
---
#### Description
- **Collection storage pattern** using `NativeParallelMultiHashMap<TKey, TValue>` to store one-to-many relationships between entities, most commonly parent-to-children mappings

- Allows multiple values (children) per key (parent) with efficient parallel-safe access, perfect for hierarchies and groupings

- Standard iteration uses `TryGetFirstValue()` followed by `do-while` loop with `TryGetNextValue()` to iterate all values for key

- [[Thread-safe]] parallel writes and reads make it ideal for jobs building or querying entity relationships

#### Example
```csharp
// Build parent-to-children mapping
[BurstCompile]
public partial struct BuildParentChildMapJob : IJobEntity
{
    public NativeParallelMultiHashMap<Entity, Entity>.ParallelWriter ChildMap;

    public void Execute(Entity entity, in Parent parent)
    {
        ChildMap.Add(parent.Value, entity);
    }
}

[BurstCompile]
public partial struct ParentChildSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Create map with estimated capacity
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
        foreach (var parentEntity in
            SystemAPI.QueryBuilder().WithAll<Parent>().Build().ToEntityArray(Allocator.Temp))
        {
            // Check if parent has children
            if (childMap.TryGetFirstValue(parentEntity, out Entity child, out var iterator))
            {
                ProcessChild(child);

                // Iterate remaining children
                while (childMap.TryGetNextValue(out child, ref iterator))
                {
                    ProcessChild(child);
                }
            }
        }

        childMap.Dispose();
    }
}
```

**Standard iteration pattern:**
```csharp
if (map.TryGetFirstValue(key, out TValue value, out var iterator))
{
    // Process first value
    DoSomething(value);

    // Process remaining values
    while (map.TryGetNextValue(out value, ref iterator))
    {
        DoSomething(value);
    }
}
```

**Common use cases:**
- Parent-to-children entity mappings
- Group entities by type/category
- Spatial partitioning (grid cell → entities)
- Tag-based collections (faction → members)

#### Pros
- **[[Thread-safe]]** - parallel reads and writes via `ParallelWriter`

- **Multiple values per key** - perfect for one-to-many relationships

- **[[Job]] compatible** - works in Burst-compiled jobs

- **Dynamic sizing** - grows automatically as values added

- **Fast lookups** - O(1) average case for key lookup

#### Cons
- **Unordered** - no guarantee on value iteration order

- **Memory overhead** - hash table structure adds memory cost

- **Must dispose** - manual memory management required

- **Duplicates allowed** - can add same value multiple times for same key

- **No value removal** - can only clear entire map, not individual values

#### Best use
- **Parent-child hierarchies** - building temporary parent→children maps

- **Entity grouping** - group entities by type, faction, or category

- **Spatial queries** - map grid cells to entities for proximity queries

- **Relationship tracking** - any one-to-many entity relationships

- **Parallel collection** - when multiple jobs need to build collection simultaneously

#### Avoid if
- **Ordered iteration needed** - use NativeList or DynamicBuffer instead

- **Single value per key** - use `NativeHashMap<TKey, TValue>` instead

- **Persistent storage** - multimap is temporary, use buffers for persistent relationships

- **Value removal needed** - can't remove individual values, only clear all

#### Extra tip
- **Capacity estimation**: Pre-allocate with estimated size to avoid reallocation: `new NativeParallelMultiHashMap<K, V>(estimatedSize, Allocator)`

- **Count values per key**: Use `CountValuesForKey(key)` to check how many values before iterating

- **Parallel writer**: Call `.AsParallelWriter()` for thread-safe adds in parallel jobs

- **Disposal**: Always dispose in same system that creates it, or use try-finally

- **Alternative - buffers**: For persistent relationships, use [[IBufferElementData (dynamic buffers)|DynamicBuffer]] on parent entity instead

- **Duplicate prevention**: Check `ContainsKey` before adding to avoid duplicates:
  ```csharp
  if (!map.ContainsKey(key))
      map.Add(key, value);
  ```

- **Empty check**: Use `IsEmpty` or `Count()` to check if map has any entries

- **Allocator choice**: Use `Allocator.TempJob` for jobs, `Allocator.Temp` for single-frame main thread

- **Performance tip**: Pre-size map capacity to avoid dynamic growth overhead

- **Combining patterns**: Use with [[ComponentLookup and BufferLookup]] to efficiently process grouped entities
