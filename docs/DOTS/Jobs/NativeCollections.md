---
tags:
  - job
  - collections
---

#### Description
- **Unmanaged collections** from `Unity.Collections` namespace (`NativeArray`, `NativeList`, etc.) designed for use with jobs and [[Burst]] compilation

- **Compatible with job system** - include thread-safety checks via the [[Job Safety System]] to prevent race conditions during parallel execution

- **Manual memory management** - must call `Dispose()` to free memory, avoiding garbage collection overhead but requiring careful lifetime management

- Allocated with one of three allocator types: `Allocator.Persistent` (long-lived), `Allocator.Temp` (frame-scoped), or `Allocator.TempJob` (thread-safe for jobs)

#### Example
```csharp
using Unity.Collections;
using Unity.Jobs;
using Unity.Burst;

[BurstCompile]
public partial struct ProcessDataSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Allocator.TempJob - thread-safe, manual disposal required
        var positions = new NativeArray<float3>(1000, Allocator.TempJob);
        var velocities = new NativeArray<float3>(1000, Allocator.TempJob);

        // Populate arrays...
        for (int i = 0; i < positions.Length; i++)
        {
            positions[i] = new float3(i, 0, 0);
            velocities[i] = new float3(1, 0, 0);
        }

        // Schedule job using the native arrays
        var job = new MoveJob
        {
            Positions = positions,
            Velocities = velocities,
            DeltaTime = SystemAPI.Time.DeltaTime
        };

        JobHandle handle = job.Schedule(state.Dependency);
        handle.Complete();

        // CRITICAL: Always dispose to prevent memory leaks
        positions.Dispose();
        velocities.Dispose();
    }
}

[BurstCompile]
public struct MoveJob : IJob
{
    public NativeArray<float3> Positions;
    [ReadOnly] public NativeArray<float3> Velocities;  // Read-only optimization
    public float DeltaTime;

    public void Execute()
    {
        for (int i = 0; i < Positions.Length; i++)
        {
            Positions[i] += Velocities[i] * DeltaTime;
        }
    }
}

// Example: Allocator types
public void AllocatorExamples()
{
    // Allocator.Persistent - for long-lived data (must dispose manually)
    var persistent = new NativeArray<int>(100, Allocator.Persistent);
    // ... use across multiple frames ...
    persistent.Dispose();

    // Allocator.Temp - fastest, auto-disposed at end of frame
    // CANNOT be passed to jobs
    var temp = new NativeArray<int>(100, Allocator.Temp);
    // ... use within single frame on main thread ...
    // Auto-disposed, but can call Dispose() explicitly

    // Allocator.TempJob - thread-safe for jobs, manual disposal required
    var tempJob = new NativeArray<int>(100, Allocator.TempJob);
    var job = new MyJob { Data = tempJob }.Schedule();
    job.Complete();
    tempJob.Dispose();  // Must dispose manually
}
```

#### Pros
- **Burst compatible** - enables 10-100x performance gains through native code compilation

- **No garbage collection** - unmanaged memory eliminates GC pauses and overhead during gameplay

- **Thread-safety validation** - [[Job Safety System]] automatically detects race conditions and throws exceptions before bugs occur

- **[[Cache-friendly]]** - contiguous memory layout enables efficient CPU cache utilization for better performance

#### Cons
- **Manual disposal required** - forgetting to call `Dispose()` causes memory leaks, requiring disciplined memory management

- **Allocator restrictions** - `Allocator.Temp` cannot be passed to jobs, must use thread-safe `Allocator.TempJob` instead

- **Limited API** - native collections have fewer features than managed collections (no LINQ, limited resizing, etc.)

- **Complexity overhead** - thread-safety attributes and disposal tracking add cognitive load compared to managed collections

#### Best use
- **Job data passing** - primary mechanism for passing data to [[IJob]], [[IJobParallelFor]], and [[IJobEntity]] jobs

- **Burst-compiled code** - required data structures when using [[Burst]] compiler for maximum performance

- **High-performance arrays** - when you need contiguous, [[Cache-friendly]] memory without GC overhead for large datasets

#### Avoid if
- **Data only used on main thread** - managed arrays are simpler if you never pass data to jobs

- **Temporary small allocations** - for tiny, short-lived data (< 10 elements), stack-allocated arrays may be sufficient

- **Complex collections needed** - if you need dictionaries, hash sets, or advanced LINQ operations, managed collections are more appropriate

#### Extra tip
- **Allocator.Persistent vs TempJob lifetime**:
  - `Persistent`: indefinite lifetime, use for data lasting multiple frames or entire game session
  - `TempJob`: up to 4 frames, use for data passed to jobs that complete within a few frames
  - `Temp`: single frame only, fastest but CANNOT be used in jobs

- **Thread-safe allocators mandatory for jobs** - only `Allocator.TempJob` and `Allocator.Persistent` are thread-safe. Never pass `Allocator.Temp` collections to jobs

- **[ReadOnly] attribute enables parallelism** - mark fields `[ReadOnly]` when jobs only read data, allowing multiple jobs to access same collection concurrently:
  ```csharp
  [ReadOnly] public NativeArray<float> SharedInputData;
  ```

- **Dispose with job dependencies** - use `Dispose(JobHandle)` to schedule disposal after job completes, avoiding manual completion:
  ```csharp
  var array = new NativeArray<int>(100, Allocator.TempJob);
  JobHandle job = new MyJob { Data = array }.Schedule();
  array.Dispose(job);  // Schedules disposal after job completes
  ```

- **Safety checks are Editor-only by default** - native collection safety checks add overhead in Editor but are disabled in builds for performance

- **Common native collection types**:
  - `NativeArray<T>`: Fixed-size array (most common)
  - `NativeList<T>`: Resizable list (like `List<T>`)
  - `NativeQueue<T>`: FIFO queue for producer-consumer patterns
  - `NativeHashMap<K,V>`: Dictionary for key-value lookups
  - `NativeHashSet<T>`: Set for unique value storage

- **Memory leak detection** - Unity Editor warns about undisposed native collections in console. Enable "Jobs > Leak Detection" in Preferences for detailed tracking

- **[NativeDisableContainerSafetyRestriction]** - disables safety checks on a specific field when you guarantee thread-safety manually. Dangerous - only use when profiling proves safety checks are a bottleneck

## See Also

- [[IJob]] - Single-threaded job using native collections
- [[IJobParallelFor]] - Parallel job for array processing
- [[JobHandle]] - Job dependencies and completion
- [[Job Safety System]] - Thread safety validation
- [[Burst]] - Burst compiler for performance
- [[Allocator]] - Memory allocation strategies
