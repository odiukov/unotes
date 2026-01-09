---
tags:
  - pattern
---
#### Description
- **Change detection pattern** filtering queries to only include entities where specific [[Component|components]] changed since last query execution

- Applied via `.WithChangeFilter<T>()` on [[SystemAPI.Query]] or `EntityQuery.SetChangedVersionFilter()` for **reactive systems** processing only modified data

- Unity tracks **change versions** per [[Chunk]] - when component data written via `.ValueRW`, `RefRW<T>`, or `EntityManager.SetComponentData`, chunk version increments

- Massive **performance optimization** for UI, synchronization, or expensive operations needing to run only on data changes

#### Example
```csharp
// Health bar UI - only update when health changes
public partial struct DisplayLifeSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // WithChangeFilter ensures this only runs for changed Health
        foreach (var (health, healthBar) in
            SystemAPI.Query<RefRO<Health>, HealthBarController>()
                .WithChangeFilter<Health>())  // Only when Health changed
        {
            healthBar.UpdateHealthBar(health.ValueRO.Current, health.ValueRO.Max);
        }
    }
}

// Multiple change filters - triggers if ANY component changed
[BurstCompile]
public partial struct SyncTransformSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, prevPos) in
            SystemAPI.Query<RefRO<LocalTransform>, RefRW<PreviousPosition>>()
                .WithChangeFilter<LocalTransform>())
        {
            prevPos.ValueRW.Value = transform.ValueRO.Position;
        }
    }
}

// Manual EntityQuery with change filter
[BurstCompile]
public partial struct SpawnSystem : ISystem
{
    private EntityQuery _query;

    public void OnCreate(ref SystemState state)
    {
        _query = SystemAPI.QueryBuilder()
            .WithAll<SpawnPoint, Active>()
            .Build();
        _query.SetChangedVersionFilter(typeof(SpawnPoint));
    }

    public void OnUpdate(ref SystemState state)
    {
        // Only processes entities with changed SpawnPoint
        foreach (var spawnPoint in _query.ToComponentDataArray<SpawnPoint>(Allocator.Temp))
        {
            // Expensive spawning logic only when spawn points change
        }
    }
}
```

**How it works:**
- Each chunk stores change version number
- Writing to component increments chunk's version
- Change filter compares chunk version to last query execution
- Only chunks with newer version are processed

#### Pros
- **Massive performance gains** - skip processing unchanged data, can be 10-100x faster for reactive systems

- **Simple API** - single `.WithChangeFilter<T>()` call enables optimization

- **[[Chunk]]-level tracking** - efficient granularity, not per-entity overhead

- **Multiple filters** - can combine multiple change filters, triggers if ANY changed

- **Automatic** - no manual tracking required, Unity handles version management

#### Cons
- **Chunk-level granularity** - if one entity in chunk changes, entire chunk reprocessed

- **Write-only detection** - reading component doesn't mark as changed, only writes

- **Not immediate** - changes tracked per-chunk, not per-entity

- **Version overflow** - after ~4 billion changes, versions wrap (extremely rare in practice)

- **Combining filters** - multiple filters use OR logic, can't specify AND requirement

#### Best use
- **UI updates** - health bars, nameplates, minimaps only update when values change

- **Synchronization** - network sync, physics sync only when transforms/velocities change

- **Expensive operations** - pathfinding recalculation, LOD updates only when needed

- **Event systems** - damage events, state changes that trigger reactions

- **Optimization** - any system where most entities don't change most frames

#### Avoid if
- **Always process all** - if system must run on all entities every frame, change filter adds overhead

- **Frequent changes** - if component changes every frame, filter provides no benefit

- **Need per-entity detection** - change filter is chunk-level, not entity-level

- **Complex change logic** - if need to detect specific field changes, manual tracking clearer

#### Extra tip
- **Read vs write**: Only writing marks changed - reading via `RefRO` doesn't increment version

- **Multiple filters OR logic**: `.WithChangeFilter<A>().WithChangeFilter<B>()` triggers if A **or** B changed

- **Combining with other filters**: Can combine with `.WithAll`, `.WithNone`, etc

- **Manual version comparison**: Can check versions manually with `EntityQuery.CalculateChunkCount()` and chunk iteration

- **Testing**: Create entities, modify component, verify system processes them

- **Performance measurement**: Profile system with/without change filter to quantify benefit

- **False positives**: If any entity in chunk changes, all entities in that chunk reprocessed

- **Structural changes reset**: Adding/removing components resets change versions

- **Burst compatible**: Change filters work in Burst-compiled systems

- **Best practices**: Use for UI, sync, expensive operations; don't use for every-frame processing
