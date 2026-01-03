---
tags:
  - pattern
---
#### Description
- **Change detection pattern** that filters queries to only include entities where specific [[Component|components]] have changed since the last query execution

- Applied via `.WithChangeFilter<T>()` on [[SystemAPI.Query]] or `EntityQuery.SetChangedVersionFilter()` to create **reactive systems** that only process modified data

- Unity tracks **change versions** per [[Chunk]] - when component data is written to (via `.ValueRW`, `RefRW<T>`, or `EntityManager.SetComponentData`), the chunk's version increments

- Massive **performance optimization** for UI, synchronization, or expensive operations that only need to happen when data changes

#### Example
```csharp
// Health bar UI system - only update when health changes
public partial struct DisplayLifeSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // SystemAPI.ManagedAPI for managed component access
        // WithChangeFilter ensures this only runs for entities with changed Health
        foreach (var (health, healthBar) in
            SystemAPI.Query<RefRO<Health>, HealthBarController>()
                .WithChangeFilter<Health>())  // Only entities where Health changed
        {
            // Update UI - only when health actually changed
            healthBar.UpdateHealthBar(health.ValueRO.Current, health.ValueRO.Max);
        }
    }
}

// Multiple change filters - triggers if ANY specified component changed
[BurstCompile]
public partial struct SyncTransformSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, previousPos) in
            SystemAPI.Query<RefRO<LocalTransform>, RefRW<PreviousPosition>>()
                .WithChangeFilter<LocalTransform>())  // Only when transform changed
        {
            // Store previous position for motion vectors
            previousPos.ValueRW.Value = transform.ValueRO.Position;
        }
    }
}

// Manual EntityQuery with change filter
[BurstCompile]
public partial struct EnemySpawnSystem : ISystem
{
    private EntityQuery _spawnPointQuery;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Create query with change filter
        _spawnPointQuery = SystemAPI.QueryBuilder()
            .WithAll<SpawnPoint, Active>()
            .Build();

        // Set change filter on SpawnPoint component
        _spawnPointQuery.SetChangedVersionFilter(typeof(SpawnPoint));
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Only processes entities with changed SpawnPoint
        foreach (var spawnPoint in _spawnPointQuery.ToComponentDataArray<SpawnPoint>(Allocator.Temp))
        {
            // Expensive spawning logic only runs when spawn points change
        }
    }
}

// Combining filters
[BurstCompile]
public partial struct ComplexSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (health, shield) in
            SystemAPI.Query<RefRO<Health>, RefRO<Shield>>()
                .WithChangeFilter<Health>()    // Trigger on health change
                .WithAll<Active>()             // Only active entities
                .WithNone<Dead>())             // Not dead
        {
            // Process entities where health changed and entity is active and not dead
        }
    }
}
```

#### Pros
- **Massive performance gains** - skip processing entities with unchanged data, can be 10-100x faster for reactive systems

- **Reactive programming** - natural pattern for UI updates, synchronization, or event-driven logic

- **Automatic change tracking** - Unity handles version tracking, no manual dirty flags needed

- **[[Chunk]]-level optimization** - entire chunks with no changes are skipped in iteration

#### Cons
- **Chunk-level granularity** - if ANY entity in chunk changed, whole chunk is processed (not per-entity filtering)

- **Write-triggers-change** - accessing `.ValueRW` marks as changed even if you don't actually modify data (use `.ValueRO` for read-only)

- **False positives** - [[Structural changes]] (adding/removing components) can mark chunks as changed

- **State tracking** - change filters are per-query, different queries track changes independently

#### Best use
- **UI synchronization** - health bars, name plates, score displays that update when game data changes

- **Expensive post-processing** - particle effects, audio, visual effects triggered by data changes

- **Network synchronization** - only send entity updates when data actually changed

- **Debug/logging systems** - only log when values change, not every frame

#### Avoid if
- **Need to process all entities** - if you must process every entity every frame, change filter is counterproductive

- **Frequent changes expected** - if component changes every frame, change filter adds overhead without benefit

- **Need per-entity tracking** - change filters are chunk-level, not per-entity

#### Extra tip
- **How change detection works:**
  - Each chunk has a "change version" counter per component type
  - When `.ValueRW` is accessed or `SetComponentData` is called, counter increments
  - Query tracks "last seen version" per component type
  - On next query, only chunks with higher version than last seen are processed

- **Read vs Write access:**
  - `.ValueRO` on `RefRW<T>` - read-only, doesn't mark as changed
  - `.ValueRW` on `RefRW<T>` - marks as changed, even if you don't modify
  - `RefRO<T>` - always read-only, never marks as changed

- **Structural changes reset** - adding/removing components, destroying entities can reset change versions, causing false positives

- **Multiple component filters** - `.WithChangeFilter<T1>().WithChangeFilter<T2>()` triggers if T1 OR T2 changed (OR logic)

- **ResetChangeFilter** - call `query.ResetVersionFilter()` to force processing all entities next update

- **SetChangedVersionFilter with ComponentType** - for manual EntityQuery, use `query.SetChangedVersionFilter(ComponentType.ReadWrite<T>())`

- **Chunk iteration optimization** - Unity skips entire chunks if no changes, making iteration very fast for sparse changes

- **Combining with other filters** - change filters work with `.WithAll`, `.WithNone`, `.WithAny` for complex reactive queries

- **Best practices:**
  - Use `RefRO<T>` when only reading to avoid false changes
  - Group reactive components together in same chunk for better filtering
  - Combine change filters with other filters for precise reactive behavior

- **Debugging changes** - use Unity Profiler with "Record" to see when components change and triggers happen

- **Version overflow** - change versions use `uint`, can theoretically overflow after ~4 billion changes (not a practical concern)
