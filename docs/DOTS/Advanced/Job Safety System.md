---
tags:
  - advanced
  - safety
---
#### Description
- **Compile-time race condition prevention** system that validates job scheduling to ensure no two jobs can access the same data unsafely

- Implemented through **AtomicSafetyHandle** tracking on native containers (NativeArray, NativeList, etc.) - every container has a safety handle that tracks read/write access

- **Prevents two types of race conditions:** one job writes while another reads the same container, or both jobs write to the same container

- Safety checks happen both **during scheduling** (prevents unsafe job combinations) and **during execution** (prevents unsafe container access on current thread)

#### How It Works

**During Job Scheduling:**
1. Job is scanned recursively for all fields that are native containers
2. Each container's safety handle is "reserved" under this job
3. When another job schedules, Unity checks if its containers conflict with already-scheduled jobs
4. If unsafe behavior detected (write-read or write-write conflict), Unity throws exception and prevents scheduling

**Reserved State Release:**
- Safety handles remain reserved until `JobHandle.Complete()` is called (a [[Sync points|sync point]])
- This ensures jobs complete before conflicting access is allowed

**During Container Access:**
- Every read/write to native container calls `AtomicSafetyHandle.CheckReadAndThrow()` or `CheckWriteAndThrow()`
- Ensures container access is allowed on current thread
- Throws if job is scheduled but you try to access container on main thread
- Throws if trying to write to read-only container in job

#### Example

```csharp
[BurstCompile]
public partial struct SafetyExampleSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var positions = new NativeArray<float3>(100, Allocator.TempJob);

        // Schedule write job
        JobHandle writeJob = new WriteJob { Positions = positions }.Schedule(state.Dependency);

        // ❌ ERROR: Cannot schedule - write-read conflict
        JobHandle readJob = new ReadJob { Positions = positions }.Schedule(state.Dependency);

        // ✅ CORRECT: Chain dependency
        JobHandle readJob = new ReadJob { Positions = positions }.Schedule(writeJob);

        state.Dependency = readJob;
        positions.Dispose(state.Dependency);
    }
}

// ❌ ERROR: Main thread access during job
var array = new NativeArray<int>(10, Allocator.TempJob);
JobHandle job = new WriteJob { Array = array }.Schedule();
int value = array[0];  // Throws AtomicSafetyHandle exception

// ✅ CORRECT: Complete job first
job.Complete();  // Sync point
int value = array[0];  // Now safe
```

#### Common Safety Errors

**Error: Write-Read Conflict**
```
InvalidOperationException: The previously scheduled job `WriteJob` writes to
the NativeArray `Positions`. You must call JobHandle.Complete() on the job
`WriteJob`, before you can read from the NativeArray safely.
```
**Solution:** Use job dependencies - pass write job's JobHandle as dependency to read job

**Error: Write-Write Conflict**
```
InvalidOperationException: The previously scheduled job `JobA` writes to the
NativeArray `Data`. The job `JobB` also writes to the same NativeArray. You
cannot schedule two jobs that write to the same data simultaneously.
```
**Solution:** Chain jobs with dependencies or use different containers

**Error: Main Thread Access During Job**
```
InvalidOperationException: The NativeArray Positions is being written to by a
job. You are trying to read from it on the main thread. This is not allowed.
```
**Solution:** Call `jobHandle.Complete()` before accessing on main thread

#### Pros
- **Prevents race conditions** - compile-time and runtime checking catches threading bugs

- **Deterministic validation** - safety errors are reproducible and caught early in Editor

- **Thread safety guarantees** - if code compiles and runs without errors, it's thread-safe

- **Clear error messages** - exceptions explain exactly what conflict occurred

#### Cons
- **Performance overhead in Editor** - safety checks add overhead (disabled in builds for performance)

- **Requires understanding** - developers must learn job dependencies and [[ReadOnly and Optional|[ReadOnly]]] attributes

- **Can be restrictive** - prevents some valid patterns that are technically safe but complex to validate

#### Best use
- **All job scheduling** - safety system is always active, ensuring all jobs are scheduled correctly

- **Debugging race conditions** - safety errors pinpoint exact source of threading issues

- **Learning DOTS** - safety system teaches correct job dependency patterns

#### Avoid if
- N/A - safety system is fundamental to Unity jobs, cannot be avoided

#### Extra tip
- **[ReadOnly] attribute** - only fields marked `[ReadOnly]` are considered read-only for safety checks

- **Job dependencies solve conflicts** - chain dependencies to prevent race conditions:
  ```csharp
  JobHandle jobB = jobB.Schedule(jobA);  // jobB waits for jobA
  ```

- **Unity ECS auto-manages dependencies** - `SystemState.Dependency` automatically tracks component access across systems

- **Parallel reads** - multiple jobs can read same container if all marked `[ReadOnly]`, but only one can write

- **Safety in builds** - safety checks are Editor-only by default, disabled in builds (enable with `ENABLE_UNITY_COLLECTIONS_CHECKS`)

- **Complete() is a [[Sync points|sync point]]** - avoid calling `JobHandle.Complete()` when possible

## See Also

- [[Job]] - Overview of jobs in DOTS
- [[Sync points]] - When jobs are forced to complete
- [[IJobEntity]] - Simplified job type
- [[ReadOnly and Optional]] - Access control attributes
