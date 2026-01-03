---
tags:
  - system
---
#### Description
- **Deferred command recording system** that queues [[Structural changes]] (create entity, add/remove components, destroy entity) for later playback on the main thread

- Necessary because [[Structural changes]] cannot happen during job execution - ECB records commands during parallel jobs, plays back on main thread at specific [[Sync points]]

- Multiple **ECB system types** available (BeginInitialization, BeginSimulation, EndSimulation, etc.) that control when commands are played back

- Supports **ParallelWriter** for thread-safe command recording across parallel jobs with deterministic playback ordering

#### Example
```csharp
[BurstCompile]
public partial struct SpawnerSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Get ECB singleton - commands play back at beginning of simulation
        var ecbSingleton = SystemAPI.GetSingleton<BeginSimulationEntityCommandBufferSystem.Singleton>();
        EntityCommandBuffer ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

        foreach (var spawner in SystemAPI.Query<RefRW<SpawnerData>>())
        {
            spawner.ValueRW.Timer -= SystemAPI.Time.DeltaTime;

            if (spawner.ValueRO.Timer <= 0)
            {
                // Queue entity creation - executes later
                Entity newEntity = ecb.Instantiate(spawner.ValueRO.Prefab);
                ecb.SetComponent(newEntity, LocalTransform.FromPosition(spawner.ValueRO.SpawnPosition));

                spawner.ValueRW.Timer = spawner.ValueRO.SpawnRate;
            }
        }
    }
}

// ParallelWriter example for jobs
public partial struct KillSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecbSingleton = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>();
        EntityCommandBuffer ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

        new KillJob
        {
            Ecb = ecb.AsParallelWriter()
        }.ScheduleParallel();
    }
}

public partial struct KillJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;

    // [ChunkIndexInQuery] provides sort key for deterministic playback
    private void Execute(Entity entity, [ChunkIndexInQuery] int sortKey, in Health health)
    {
        if (health.Current <= 0)
        {
            // Sort key ensures deterministic command order
            Ecb.DestroyEntity(sortKey, entity);
        }
    }
}

// Manual ECB creation (less common)
public void OnUpdate(ref SystemState state)
{
    var ecb = new EntityCommandBuffer(Allocator.Temp);

    // Record commands
    Entity e = ecb.CreateEntity();
    ecb.AddComponent<MyComponent>(e);

    // Manual playback - happens immediately
    ecb.Playback(state.EntityManager);
    ecb.Dispose();
}
```

#### Pros
- **Enables parallel [[Structural changes]]** - jobs can queue changes that would otherwise require main thread

- **Deferred execution** - commands recorded during iteration don't invalidate iterators

- **Deterministic playback** - ParallelWriter with sort keys ensures same command order every run

- **[[Burst]] compatible** - can be used in Burst-compiled jobs

#### Cons
- **Memory overhead** - commands are stored in memory until playback, large batches can allocate significantly

- **Delayed effect** - changes don't apply until playback, can't query new entities immediately

- **Requires sort keys** - ParallelWriter needs `[ChunkIndexInQuery]` for deterministic ordering, easy to forget

- **Manual disposal** - manually created ECBs must be disposed, forgetting causes memory leaks

#### Best use
- **Parallel structural changes** - spawning, destroying, or modifying entities from within parallel jobs

- **Job-based entity creation** - spawning enemies, projectiles, effects from IJobEntity

- **Collision response** - destroying entities or adding components based on physics events in [[ITriggerEventsJob]]

#### Avoid if
- **Main thread with immediate effect** - if on main thread and need immediate changes, use `EntityManager` directly instead of ECB

- **Simple queries** - for reading/writing components without structural changes, use [[SystemAPI.Query]] or [[IJobEntity]] without ECB

- **Singleton data updates** - for updating singleton components, direct access is simpler than ECB

#### Extra tip
- **ECB system types and playback timing:**
  - `BeginInitializationEntityCommandBufferSystem` - plays back before transform system, good for creating entities
  - `BeginSimulationEntityCommandBufferSystem` - plays back early in simulation frame, good for destroying entities
  - `EndSimulationEntityCommandBufferSystem` - plays back at end of simulation, after all gameplay systems
  - `BeginPresentationEntityCommandBufferSystem` - plays back before rendering

- **Singleton pattern** - `SystemAPI.GetSingleton<ECBSystem.Singleton>().CreateCommandBuffer(state.WorldUnmanaged)` is the preferred way to get ECB

- **PlaybackPolicy** - use `EntityCommandBuffer.PlaybackPolicy.SinglePlayback` for manual ECBs that should only play once

- **Multiple ECBs strategy** - common pattern: use BeginInitializationECB for creation/adding components, BeginSimulationECB for destruction to ensure proper execution order

- **Sorting determinism** - `[ChunkIndexInQuery] int sortKey` parameter in jobs provides the sort index - pass to ParallelWriter methods as first parameter

- **Parallel vs regular** - use `.AsParallelWriter()` only for jobs scheduled with `ScheduleParallel()`, not for `Schedule()` or main thread execution
