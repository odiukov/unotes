---
tags:
  - job
---

#### Description
- **Parallel job** that processes array indices across multiple worker threads, defined as a struct implementing `IJobParallelFor` with an `Execute(int index)` method

- Scheduled with `.Schedule(count, batchSize)` where **count** is the number of indices to process and **batchSize** controls how many indices each worker thread processes at a time

- Work is automatically **distributed across CPU cores** - batches execute concurrently on separate threads for massive performance gains

- **Batch size tuning** - larger batches reduce scheduling overhead but decrease parallelism granularity, requiring empirical testing to optimize

#### Example
```csharp
[BurstCompile]
public struct FindClosestTargetJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<float3> SeekerPositions;
    [ReadOnly] public NativeArray<float3> TargetPositions;

    // NativeDisableParallelForRestriction allows writing to any index
    [NativeDisableParallelForRestriction]
    public NativeArray<int> ClosestTargetIndices;

    public void Execute(int seekerIndex)
    {
        float3 seekerPos = SeekerPositions[seekerIndex];
        float closestDistSq = float.MaxValue;
        int closestTarget = -1;

        // Search all targets for closest one
        for (int i = 0; i < TargetPositions.Length; i++)
        {
            float distSq = math.distancesq(seekerPos, TargetPositions[i]);
            if (distSq < closestDistSq)
            {
                closestDistSq = distSq;
                closestTarget = i;
            }
        }

        ClosestTargetIndices[seekerIndex] = closestTarget;
    }
}

// Scheduling the parallel job
public void FindClosestTargets()
{
    int seekerCount = 10000;
    var seekers = new NativeArray<float3>(seekerCount, Allocator.TempJob);
    var targets = new NativeArray<float3>(1000, Allocator.TempJob);
    var results = new NativeArray<int>(seekerCount, Allocator.TempJob);

    // Populate arrays...

    var job = new FindClosestTargetJob
    {
        SeekerPositions = seekers,
        TargetPositions = targets,
        ClosestTargetIndices = results
    };

    // Process in batches of 64 - tune this value!
    JobHandle handle = job.Schedule(seekerCount, batchSize: 64);

    handle.Complete();

    // Process results...

    seekers.Dispose();
    targets.Dispose();
    results.Dispose();
}
```

#### Pros
- **Massive parallelization** - automatically distributes work across all CPU cores for near-linear performance scaling

- **Simple parallel programming** - no manual thread management, locks, or synchronization needed - Unity handles it

- **[[Burst]] compatible** - combines parallel execution with highly optimized native code for maximum throughput

- **[[Cache-friendly]]** - batch processing enables better CPU cache utilization as each thread processes contiguous indices

#### Cons
- **Index-only parallelism** - can only parallelize array-based operations, not arbitrary work patterns

- **Batch size tuning required** - optimal batch size varies by workload and must be determined through profiling and testing

- **Limited write access** - by default, each thread can only write to its assigned batch indices, requiring `[NativeDisableParallelForRestriction]` for arbitrary writes

- **No guaranteed order** - batches execute in unpredictable order, making deterministic execution challenging without careful design

#### Best use
- **Array transformations** - processing positions, velocities, damage calculations across large arrays

- **Brute-force searches** - finding nearest neighbors, collision checks, spatial queries on large datasets

- **Independent calculations** - any per-element computation where each result doesn't depend on others (embarrassingly parallel problems)

#### Avoid if
- **You're processing entities** - use [[IJobEntity]] instead, which provides better integration with ECS and automatic query generation

- **Heavy inter-element dependencies** - if processing index N requires results from index N-1, parallel execution becomes difficult

- **Tiny arrays** - for arrays with < 100 elements, parallel overhead may exceed benefits, use [[IJob]] or main thread instead

#### Extra tip
- **Performance progression** (from Unity Job System 101 tutorial with 1000 seekers × 1000 targets):
  - Main thread: ~330ms
  - Single-threaded [[IJob]] without Burst: ~30ms
  - Single-threaded [[IJob]] with [[Burst]]: ~1.5ms
  - `IJobParallelFor` with 16 cores: ~17ms elapsed for 10,000 entities

- **Batch size guidelines** - start with 32-64 and profile. Larger batches (128+) reduce overhead for heavy computations, smaller batches (16-32) increase parallelism for lightweight operations

- **[NativeDisableParallelForRestriction]** - disables safety checks that restrict write access to assigned batch indices. Use carefully - you're responsible for preventing race conditions:
  ```csharp
  [NativeDisableParallelForRestriction]
  public NativeArray<int> SharedOutput;  // Can write to any index
  ```

- **Read-only data sharing** - multiple threads can safely read from same [[NativeCollections]] if marked `[ReadOnly]`, enabling data sharing without copies

- **Dependency chaining** - combine with other jobs using [[JobHandle]] dependencies to build complex parallel pipelines:
  ```csharp
  JobHandle sortJob = sortJob.Schedule(count, 64);
  JobHandle searchJob = searchJob.Schedule(count, 64, sortJob);  // Waits for sort
  ```

- **For entity processing, prefer [[IJobEntity]]** - it automatically handles parallel scheduling, provides cleaner syntax, and integrates better with the ECS query system

## See Also

- [[IJob]] - Single-threaded job
- [[IJobEntity]] - Simplified parallel entity iteration
- [[JobHandle]] - Job dependencies and scheduling
- [[Burst]] - Burst compiler for optimization
- [[NativeCollections]] - Thread-safe containers for jobs
- [[Job Safety System]] - Parallel access validation
