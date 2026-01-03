# Patterns

Common design patterns and best practices for Unity DOTS development.

## Query Patterns

- **[[SystemAPI.Query]]** - Main entity iteration pattern with foreach syntax
- **[[WithChangeFilter (reactive updates)]]** - Reactive systems that only process changed entities
- **[[Singleton entities and components]]** - Global single-instance components and entities

## Query Pattern Overview

### SystemAPI.Query
The primary way to iterate entities in OnUpdate:
```csharp
foreach (var (health, damage) in
    SystemAPI.Query<RefRW<Health>, RefRO<Damage>>()
        .WithAll<Active>()
        .WithNone<Dead>())
{
    health.ValueRW.Current -= damage.ValueRO.Amount;
}
```

### WithChangeFilter
Reactive pattern for UI, synchronization, or expensive operations:
```csharp
foreach (var (health, healthBar) in
    SystemAPI.Query<RefRO<Health>, HealthBarUI>()
        .WithChangeFilter<Health>())
{
    healthBar.UpdateUI(health.ValueRO.Current);
}
```

### Singleton Pattern
Global access to single-instance components:
```csharp
GameConfig config = SystemAPI.GetSingleton<GameConfig>();
RefRW<PlayerState> state = SystemAPI.GetSingletonRW<PlayerState>();
```

## Additional Patterns

### Entity Relationships
- **Parent-Child** - Using Parent component and [[ComponentLookup and BufferLookup|ComponentLookup]]
- **Entity References** - Components storing Entity IDs for targets, owners, etc.

### Deferred Changes
- **[[EntityCommandBuffer]]** - Queue structural changes for later playback
- **Parallel ECB** - Use [[ChunkIndexInQuery]] for deterministic parallel writes

### Performance Patterns
- **[[Burst]]** - Always use [[BurstCompile]] for maximum performance
- **Job Parallelization** - Use [[IJobEntity]].ScheduleParallel() for multi-threading
- **Change Filters** - Skip unchanged entities with [[WithChangeFilter (reactive updates)]]

## Best Practices

1. **Use SystemAPI.Query** for most entity iteration (simplest and most ergonomic)
2. **Add WithChangeFilter** for reactive systems (UI, logging, expensive operations)
3. **Use singletons** for global state and configuration
4. **Prefer read-only access** (RefRO, [ReadOnly]) when possible for parallelization
5. **Use EntityCommandBuffer** for structural changes in jobs

## See Also

- [[SystemAPI.Query]] - Entity iteration
- [[IJobEntity]] - Parallel job execution
- [[EntityCommandBuffer]] - Deferred structural changes
- [[ComponentLookup and BufferLookup]] - Random entity access
