---
tags:
  - job
  - advanced
---

#### Description
- **Chunk-by-chunk entity processing** job type that iterates [[Chunk|chunks]] of entities with direct array access - lower-level alternative to [[IJobEntity]] with more control over data access

- **ComponentTypeHandle access** - uses type handles to get NativeArray slices of component data from chunks, allowing batch operations on contiguous memory

- **Full chunk control** - Execute method receives entire [[ArchetypeChunk]], enabling enableable component filtering, change detection, and custom iteration patterns

- **No [[SystemAPI]] support** - unlike IJobEntity, cannot use SystemAPI methods; must manually create and update type handles

#### Example
```csharp
// Basic IJobChunk example
[BurstCompile]
public partial struct MovementJob : IJobChunk
{
    public float DeltaTime;

    // Type handles for component access (must be updated each frame)
    public ComponentTypeHandle<LocalTransform> TransformHandle;
    [ReadOnly] public ComponentTypeHandle<Velocity> VelocityHandle;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                       bool useEnabledMask, in v128 chunkEnabledMask)
    {
        // Get component arrays from chunk
        NativeArray<LocalTransform> transforms = chunk.GetNativeArray(ref TransformHandle);
        NativeArray<Velocity> velocities = chunk.GetNativeArray(ref VelocityHandle);

        // Iterate entities in chunk
        for (int i = 0; i < chunk.Count; i++)
        {
            LocalTransform transform = transforms[i];
            Velocity velocity = velocities[i];

            transform.Position += velocity.Value * DeltaTime;

            transforms[i] = transform;
        }
    }
}

// System scheduling IJobChunk
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    private EntityQuery _query;
    private ComponentTypeHandle<LocalTransform> _transformHandle;
    private ComponentTypeHandle<Velocity> _velocityHandle;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Create query
        _query = state.GetEntityQuery(typeof(LocalTransform), typeof(Velocity));

        // Get type handles (must update each frame)
        _transformHandle = state.GetComponentTypeHandle<LocalTransform>(isReadOnly: false);
        _velocityHandle = state.GetComponentTypeHandle<Velocity>(isReadOnly: true);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Update type handles (REQUIRED each frame)
        _transformHandle.Update(ref state);
        _velocityHandle.Update(ref state);

        // Schedule job
        var job = new MovementJob
        {
            DeltaTime = SystemAPI.Time.DeltaTime,
            TransformHandle = _transformHandle,
            VelocityHandle = _velocityHandle
        };

        state.Dependency = job.ScheduleParallel(_query, state.Dependency);
    }
}

// IJobChunk with enableable components
[BurstCompile]
public partial struct EnableableComponentJob : IJobChunk
{
    public ComponentTypeHandle<Health> HealthHandle;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                       bool useEnabledMask, in v128 chunkEnabledMask)
    {
        NativeArray<Health> healths = chunk.GetNativeArray(ref HealthHandle);

        // Iterate only enabled entities using mask
        var enumerator = new ChunkEntityEnumerator(useEnabledMask, chunkEnabledMask, chunk.Count);
        while (enumerator.NextEntityIndex(out int i))
        {
            // Only processes entities with enabled Health component
            Health health = healths[i];
            health.Current += 1f;
            healths[i] = health;
        }
    }
}
```

#### Pros
- **Maximum performance** - direct array access with minimal overhead, ideal for performance-critical systems

- **Chunk-level operations** - can process entire chunks atomically, useful for spatial partitioning or batch operations

- **Enableable component support** - full control over enabled/disabled entity iteration with ChunkEntityEnumerator

- **Change filtering** - chunk.DidChange() enables efficient "only process modified data" patterns

#### Cons
- **More boilerplate** - requires creating, storing, and updating ComponentTypeHandle for each component accessed

- **No SystemAPI** - cannot use SystemAPI.Query, SystemAPI.Time, or other convenience methods inside job

- **Manual handle updates** - forgetting to Update() type handles each frame causes stale data bugs

- **Steeper learning curve** - more complex API compared to [[IJobEntity]] with foreach loops

#### Best use
- **Performance-critical systems** - when you've profiled and identified IJobEntity as bottleneck, IJobChunk may be faster

- **Enableable component filtering** - when you need precise control over which enabled entities to process

- **Change detection** - systems that only react to component changes, avoiding redundant processing

- **Chunk-wide operations** - spatial hashing, bounding box computation, or other chunk-level batch operations

#### Avoid if
- **Simple iteration** - [[IJobEntity]] is cleaner and easier for straightforward entity processing

- **Prototyping** - IJobEntity has less boilerplate, better for rapid development

- **No performance issues** - premature optimization; start with IJobEntity, switch to IJobChunk only if profiling shows need

#### Extra tip
- **ComponentTypeHandle pattern:**
  ```csharp
  public partial struct MySystem : ISystem
  {
      // Store handles as fields
      private ComponentTypeHandle<Health> _healthHandle;

      public void OnCreate(ref SystemState state)
      {
          // Create handles in OnCreate
          _healthHandle = state.GetComponentTypeHandle<Health>(isReadOnly: false);
      }

      public void OnUpdate(ref SystemState state)
      {
          // CRITICAL: Update handles every frame
          _healthHandle.Update(ref state);

          // Use in job
          new MyJob { HealthHandle = _healthHandle }.ScheduleParallel(query, state.Dependency);
      }
  }
  ```

- **Type handle types:**
  ```csharp
  ComponentTypeHandle<T>  // Read-write or read-only component
  BufferTypeHandle<T>  // DynamicBuffer<T> access
  EntityTypeHandle  // Get Entity ID from chunk
  SharedComponentTypeHandle<T>  // Access shared component value
  ```

- **Getting data from chunk:**
  ```csharp
  public void Execute(in ArchetypeChunk chunk, ...)
  {
      // Component arrays
      NativeArray<Health> healths = chunk.GetNativeArray(ref healthHandle);

      // Buffer accessor
      BufferAccessor<Waypoint> waypoints = chunk.GetBufferAccessor(ref waypointHandle);

      // Entity array
      NativeArray<Entity> entities = chunk.GetNativeArray(entityHandle);

      // Shared component value (single value for entire chunk)
      SharedComponent shared = chunk.GetSharedComponent(sharedHandle);

      // Chunk count (entity count in this chunk)
      int count = chunk.Count;
  }
  ```

- **Enableable component iteration:**
  ```csharp
  if (!useEnabledMask)
  {
      // All entities enabled, iterate normally
      for (int i = 0; i < chunk.Count; i++) { /* ... */ }
  }
  else
  {
      // Some entities disabled, use enumerator
      var enumerator = new ChunkEntityEnumerator(useEnabledMask, chunkEnabledMask, chunk.Count);
      while (enumerator.NextEntityIndex(out int i))
      {
          // Only processes enabled entities
      }
  }
  ```

- **Change detection:**
  ```csharp
  // Did component change since last system version?
  if (chunk.DidChange(ref transformHandle, LastSystemVersion))
  {
      // Process changed chunk
  }
  ```

- **unfilteredChunkIndex parameter** - use for thread-safe writes to parallel arrays:
  ```csharp
  results[unfilteredChunkIndex] = computedValue;
  ```

- **Scheduling methods:**
  ```csharp
  JobHandle handle = job.Schedule(query, inputDeps);  // Single-threaded
  JobHandle handle = job.ScheduleParallel(query, inputDeps);  // Parallel (recommended)
  ```

- **Common mistakes:**
  ```csharp
  // WRONG: Forgot to Update() handles
  public void OnUpdate(ref SystemState state)
  {
      // _transformHandle is stale! Will read old data
      new MyJob { Handle = _transformHandle }.ScheduleParallel(query, state.Dependency);
  }

  // CORRECT: Update handles every frame
  public void OnUpdate(ref SystemState state)
  {
      _transformHandle.Update(ref state);  // Update first!
      new MyJob { Handle = _transformHandle }.ScheduleParallel(query, state.Dependency);
  }
  ```

- **IJobChunk vs IJobEntity:**
  ```csharp
  // Use IJobEntity for:
  - Simple per-entity processing
  - Prototype/development speed
  - SystemAPI convenience methods
  - Cleaner, more maintainable code

  // Use IJobChunk for:
  - Maximum performance (profiled bottleneck)
  - Enableable component filtering
  - Change detection optimization
  - Chunk-wide batch operations
  - Custom iteration patterns
  ```

## See Also

- [[IJobEntity]] - Higher-level entity iteration job (easier to use)
- [[IJobParallelFor]] - Parallel array processing
- [[EntityQuery]] - Entity filtering for jobs
- [[ComponentLookup and BufferLookup]] - Random entity access
- [[IEnableableComponent (toggleable components)|IEnableableComponent]] - Toggleable components
- [[Chunk]] - 16KiB memory blocks storing entities
