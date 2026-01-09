---
tags:
  - job
---

#### Description
- **Single-threaded job** that executes on a worker thread, defined as a struct implementing the `IJob` interface with an `Execute()` method

- Scheduled from the main thread using `.Schedule()`, returning a [[JobHandle]] that tracks the job's completion and enables dependency chaining

- Cannot access managed objects, perform I/O, or access non-readonly static fields - only works with [[NativeCollections]] and unmanaged data types

- Scheduled jobs create a **private copy of the struct**, but external data accessed through pointers or native containers remains visible outside the job

#### Example
```csharp
[BurstCompile]
public struct CalculateSumJob : IJob
{
    [ReadOnly] public NativeArray<int> Input;
    public NativeArray<int> Result;  // Single element to store sum

    public void Execute()
    {
        int sum = 0;
        for (int i = 0; i < Input.Length; i++)
        {
            sum += Input[i];
        }
        Result[0] = sum;
    }
}

// Scheduling the job
public void ProcessData()
{
    var input = new NativeArray<int>(1000, Allocator.TempJob);
    var result = new NativeArray<int>(1, Allocator.TempJob);

    // Populate input...

    // Schedule the job
    var job = new CalculateSumJob
    {
        Input = input,
        Result = result
    };

    JobHandle handle = job.Schedule();

    // Complete the job before accessing results
    handle.Complete();  // This is a sync point

    int sum = result[0];

    input.Dispose();
    result.Dispose();
}
```

#### Pros
- **Simple threading model** - single Execute() method runs on one worker thread, no need to think about parallel synchronization

- **[[Burst]] compatible** - compiles to highly optimized native code for 10-100x performance gains over managed C#

- **Composable with dependencies** - [[JobHandle]] enables chaining jobs in specific execution order without explicit synchronization

- **No garbage collection** - works with unmanaged data only, eliminating GC overhead during execution

#### Cons
- **Single-threaded execution** - doesn't leverage multiple CPU cores, use [[IJobParallelFor]] or [[IJobEntity]] for parallel processing

- **Main thread scheduling only** - jobs must be scheduled from the main thread, cannot schedule jobs from within other jobs

- **Sync point required** - must call `JobHandle.Complete()` before accessing job results on main thread, potentially stalling execution

- **Limited data access** - cannot use managed collections, strings, or classes - restricted to [[NativeCollections]] and primitives

#### Best use
- **Expensive single-threaded calculations** - physics queries, pathfinding, AI decision making that doesn't parallelize well

- **Data aggregation** - summing values, finding min/max, calculating averages from entity data processed by other jobs

- **Serialization preparation** - converting entity data to serializable format before saving or network transmission

#### Avoid if
- **You're processing arrays or entities** - use [[IJobParallelFor]] for array processing or [[IJobEntity]] for entity iteration instead

- **Simple calculations on main thread** - if the work is trivial (< 0.1ms), job scheduling overhead may exceed the benefit

- **You need structural changes** - jobs cannot add/remove entities or components, use [[EntityCommandBuffer]] for deferred structural changes

#### Extra tip
- **Always add [[BurstCompile]]** attribute for maximum performance - without Burst, jobs are only ~30% faster than main thread

- **Job scheduling overhead** - the Unity Job System 101 tutorial showed ~30ms main thread execution vs ~30ms single-threaded job (without Burst), but only ~1.5ms with Burst enabled

- **Dependencies prevent race conditions** - when scheduling multiple jobs that access same data, pass previous [[JobHandle]] as dependency:
  ```csharp
  JobHandle jobA = jobA.Schedule();
  JobHandle jobB = jobB.Schedule(jobA);  // jobB waits for jobA to complete
  ```

- **Complete() is a [[Sync points|sync point]]** - calling `handle.Complete()` forces the main thread to wait for the job, blocking execution until completion

- **Use TempJob allocator** - when passing [[NativeCollections]] to jobs, use `Allocator.TempJob` which is thread-safe and designed for job usage

- **Read-only optimization** - mark input containers with `[ReadOnly]` attribute to help [[Job Safety System]] validate thread safety

## See Also

- [[IJobParallelFor]] - Parallel version for array processing
- [[IJobEntity]] - Simplified entity iteration job
- [[JobHandle]] - Job dependencies and scheduling
- [[Burst]] - Burst compiler for job optimization
- [[NativeCollections]] - Native containers for job data
- [[Job Safety System]] - Thread safety validation
