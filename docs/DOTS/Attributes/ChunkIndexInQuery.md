---
tags:
  - attribute
---
#### Description
- **Job parameter attribute** that provides the current chunk's index in the query, used as a **sort key** for [[EntityCommandBuffer]].ParallelWriter

- Applied to an `int` parameter in [[IJobEntity]].Execute or IJobChunk jobs to receive the unique chunk index (0, 1, 2, ...)

- Critical for **deterministic parallel command buffer playback** - ensures commands from parallel jobs execute in consistent order

- Without this attribute, parallel [[EntityCommandBuffer]] commands may execute in random order, causing non-deterministic behavior

#### Example
```csharp
[BurstCompile]
public partial struct KillSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecbSingleton = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>();
        EntityCommandBuffer ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

        new KillJob
        {
            Ecb = ecb.AsParallelWriter()  // ParallelWriter for parallel jobs
        }.ScheduleParallel();
    }
}

[BurstCompile]
public partial struct KillJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;

    // [ChunkIndexInQuery] provides the sort key for deterministic ordering
    private void Execute(Entity entity, [ChunkIndexInQuery] int sortKey, in Health health)
    {
        if (health.Current <= 0)
        {
            // sortKey ensures commands execute in consistent chunk order
            Ecb.DestroyEntity(sortKey, entity);
        }
    }
}

// IJobChunk example
[BurstCompile]
public partial struct LimitedLifeTimeJob : IJobChunk
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    public EntityTypeHandle EntityType;
    public ComponentTypeHandle<LimitedLifeTime> LifeTimeType;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex, bool useEnabledMask, in v128 chunkEnabledMask)
    {
        // unfilteredChunkIndex is the equivalent of [ChunkIndexInQuery]
        NativeArray<Entity> entities = chunk.GetNativeArray(EntityType);
        NativeArray<LimitedLifeTime> lifeTimes = chunk.GetNativeArray(ref LifeTimeType);

        for (int i = 0; i < chunk.Count; i++)
        {
            if (lifeTimes[i].TimeLeft <= 0)
            {
                // Use unfilteredChunkIndex as sort key
                Ecb.DestroyEntity(unfilteredChunkIndex, entities[i]);
            }
        }
    }
}
```

#### Pros
- **Deterministic execution** - commands execute in same order every frame, preventing randomness bugs

- **Thread-safe parallel writes** - enables safe parallel command buffer writes from multiple threads

- **No performance cost** - chunk index is already known during iteration, no extra computation

- **Required for parallel ECB** - without it, ParallelWriter commands are non-deterministic

#### Cons
- **Easy to forget** - forgetting this attribute on parallel ECB jobs causes subtle non-determinism bugs

- **Must pass to ECB methods** - requires passing sortKey to every ParallelWriter method call

- **Not needed for single-threaded** - if using `.Schedule()` instead of `.ScheduleParallel()`, attribute is unnecessary

#### Best use
- **Parallel EntityCommandBuffer** - always use with ParallelWriter when scheduling jobs with `.ScheduleParallel()`

- **Deterministic spawning/destruction** - ensuring enemies spawn or die in consistent order

- **Multiplayer/replay systems** - critical for determinism in networked games or replay systems

#### Avoid if
- **Single-threaded jobs** - if using `.Schedule()` instead of `.ScheduleParallel()`, regular [[EntityCommandBuffer]] doesn't need sort keys

- **Main thread ECB** - when using ECB directly in system OnUpdate (not in a job), no sort key needed

#### Extra tip
- **ParallelWriter requirement** - `EntityCommandBuffer.ParallelWriter` methods require `int sortKey` as first parameter - this is what [ChunkIndexInQuery] provides

- **Unique per chunk** - sort key is unique per chunk being processed, not per entity - multiple entities in same chunk get same sort key

- **Playback order** - ECB plays back commands sorted by sort key first, then by order within that key

- **IJobEntity vs IJobChunk** - in IJobEntity use `[ChunkIndexInQuery] int sortKey` parameter, in IJobChunk use the `unfilteredChunkIndex` parameter (it's the same thing)

- **Multiple ECB fields** - if job has multiple ParallelWriters, same sort key is used for all of them

- **Name doesn't matter** - can name the parameter anything (`sortKey`, `chunkIndex`, `key`), attribute determines its value

- **Why determinism matters** - without deterministic ordering, entity creation/destruction order changes, IDs change, iteration order changes, causing different results each run

- **Testing determinism** - run game multiple times with same inputs - results should be identical if you use [ChunkIndexInQuery] correctly

- **Combine with entity ID** - for even finer sorting, can combine sort key with entity index: `(sortKey << 32) | entityIndexInQuery`
