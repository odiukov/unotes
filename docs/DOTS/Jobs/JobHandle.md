---
tags:
  - job
  - advanced
---

#### Description
- **Job tracking handle** returned by `.Schedule()` that represents a scheduled job and enables establishing execution order dependencies

- Used to **chain job execution** - passing a JobHandle as a dependency ensures the new job won't start until the dependency completes

- Calling `Complete()` on a JobHandle **completes all dependencies recursively**, waits for execution to finish, and removes the job from the queue

- **Only the main thread can complete jobs** - worker threads cannot call `Complete()` or schedule new jobs, only the main thread has this capability

#### Example
```csharp
[BurstCompile]
public struct GenerateDataJob : IJob
{
    public NativeArray<float> Output;

    public void Execute()
    {
        for (int i = 0; i < Output.Length; i++)
        {
            Output[i] = i * 2.5f;
        }
    }
}

[BurstCompile]
public struct ProcessDataJob : IJob
{
    [ReadOnly] public NativeArray<float> Input;
    public NativeArray<float> Output;

    public void Execute()
    {
        for (int i = 0; i < Input.Length; i++)
        {
            Output[i] = Input[i] * Input[i];  // Square the values
        }
    }
}

public void ChainJobs()
{
    var tempData = new NativeArray<float>(1000, Allocator.TempJob);
    var finalData = new NativeArray<float>(1000, Allocator.TempJob);

    // Schedule first job
    JobHandle generateHandle = new GenerateDataJob
    {
        Output = tempData
    }.Schedule();

    // Schedule second job with dependency - it waits for generateHandle
    JobHandle processHandle = new ProcessDataJob
    {
        Input = tempData,
        Output = finalData
    }.Schedule(generateHandle);  // Dependency passed here

    // Complete only the final job - automatically completes generateHandle first
    processHandle.Complete();

    // Now safe to access finalData on main thread

    tempData.Dispose();
    finalData.Dispose();
}

// Combining multiple dependencies
public void CombineDependencies()
{
    JobHandle job1 = new JobA { Data = data1 }.Schedule();
    JobHandle job2 = new JobB { Data = data2 }.Schedule();

    // Combine multiple handles - new job waits for BOTH to complete
    JobHandle combined = JobHandle.CombineDependencies(job1, job2);

    JobHandle finalJob = new JobC().Schedule(combined);

    finalJob.Complete();
}
```

#### Pros
- **Automatic dependency management** - worker threads respect dependencies without manual synchronization or locks

- **Prevents race conditions** - [[Job Safety System]] uses JobHandles to validate no two jobs access same data unsafely

- **Enables complex pipelines** - chain multiple jobs in specific execution order to build sophisticated data processing workflows

- **Minimal overhead** - dependency tracking is extremely lightweight, no significant performance cost

#### Cons
- **Main thread restriction** - only main thread can complete jobs, limiting flexibility in complex threading scenarios

- **Sync point overhead** - calling `Complete()` is a [[Sync points|sync point]] that blocks main thread until job finishes, potentially stalling execution

- **Manual dependency management** - developer must manually track and pass JobHandles, errors can lead to race conditions or deadlocks

- **No partial completion** - cannot complete just part of a job, must wait for entire execution to finish

#### Best use
- **Job dependency chains** - when job B needs results from job A, pass A's JobHandle to B's Schedule() method

- **Preventing race conditions** - when multiple jobs access same [[NativeCollections]], establish dependencies to ensure safe sequential access

- **Parallel pipelines** - run independent jobs concurrently, then combine their handles for a final processing job

#### Avoid if
- **Unity ECS handles it automatically** - in [[ISystem]], `SystemState.Dependency` automatically manages dependencies between systems based on component access

- **You need real-time blocking** - if you must immediately access job results, consider running on main thread instead of creating [[Sync points]]

- **Single job with no dependencies** - for simple cases, the default `default(JobHandle)` (no dependency) is sufficient

#### Extra tip
- **Complete() is recursive** - calling `processHandle.Complete()` automatically completes `generateHandle` first if it was a dependency. You only need to complete the final job in a chain

- **default(JobHandle) means no dependency** - passing `default` or omitting the dependency parameter schedules the job to run ASAP without waiting:
  ```csharp
  JobHandle immediate = job.Schedule();  // No dependency, runs ASAP
  JobHandle withDep = job.Schedule(previousJob);  // Waits for previousJob
  ```

- **SystemState.Dependency in ECS** - Unity ECS automatically manages JobHandles through `SystemState.Dependency`. Each system reads the dependency, schedules its jobs, and writes back the new handle:
  ```csharp
  public void OnUpdate(ref SystemState state)
  {
      // Read previous system's dependency
      JobHandle inputDeps = state.Dependency;

      // Schedule job with dependency
      JobHandle outputDeps = new MyJob().Schedule(inputDeps);

      // Write new dependency for next system
      state.Dependency = outputDeps;
  }
  ```

- **CombineDependencies for parallel work** - use `JobHandle.CombineDependencies()` to wait for multiple independent jobs:
  ```csharp
  JobHandle combined = JobHandle.CombineDependencies(jobA, jobB, jobC);
  JobHandle finalJob = aggregateJob.Schedule(combined);  // Waits for all 3
  ```

- **Complete() vs CompleteDependency()** - in ECS systems, use `state.CompleteDependency()` instead of `state.Dependency.Complete()` for clearer intent

- **Avoid premature Complete()** - each `Complete()` is a [[Sync points|sync point]] that forces the main thread to wait. Delay completing until you actually need the results

- **Read-only enables parallelism** - when multiple jobs need same input data, mark it `[ReadOnly]` so they can run in parallel instead of sequentially

- **Safety handle reservation** - JobHandles "reserve" safety handles on [[NativeCollections]] until `Complete()` is called, preventing conflicting access during execution

## See Also

- [[IJob]] - Single-threaded job
- [[IJobParallelFor]] - Parallel array processing
- [[IJobEntity]] - Entity iteration job
- [[Job Safety System]] - How dependencies prevent race conditions
- [[Sync points]] - When jobs are forced to complete
- [[System Dependencies]] - Automatic dependency management in ECS
