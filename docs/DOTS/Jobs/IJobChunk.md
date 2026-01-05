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
using Unity.Entities;
using Unity.Burst;
using Unity.Collections;
using Unity.Mathematics;

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

        // Check if chunk has any enabled Health components
        if (!chunk.Has(ref HealthHandle))
            return;

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

// IJobChunk with EntityTypeHandle
[BurstCompile]
public partial struct EntityAccessJob : IJobChunk
{
    [ReadOnly] public EntityTypeHandle EntityHandle;
    public ComponentTypeHandle<Health> HealthHandle;

    public NativeQueue<Entity>.ParallelWriter DeadEntities;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                       bool useEnabledMask, in v128 chunkEnabledMask)
    {
        NativeArray<Entity> entities = chunk.GetNativeArray(EntityHandle);
        NativeArray<Health> healths = chunk.GetNativeArray(ref HealthHandle);

        for (int i = 0; i < chunk.Count; i++)
        {
            if (healths[i].Current <= 0)
            {
                // Queue dead entity for processing
                DeadEntities.Enqueue(entities[i]);
            }
        }
    }
}

// IJobChunk with DynamicBuffer
[BurstCompile]
public partial struct BufferJob : IJobChunk
{
    public BufferTypeHandle<Waypoint> WaypointHandle;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                       bool useEnabledMask, in v128 chunkEnabledMask)
    {
        // Get buffer accessor for chunk
        BufferAccessor<Waypoint> waypointAccessor = chunk.GetBufferAccessor(ref WaypointHandle);

        for (int i = 0; i < chunk.Count; i++)
        {
            // Get buffer for this entity
            DynamicBuffer<Waypoint> waypoints = waypointAccessor[i];

            // Process buffer elements
            for (int j = 0; j < waypoints.Length; j++)
            {
                Waypoint wp = waypoints[j];
                // Process waypoint...
            }
        }
    }
}

// Change filtering in IJobChunk
[BurstCompile]
public partial struct ChangeFilterJob : IJobChunk
{
    [ReadOnly] public ComponentTypeHandle<LocalTransform> TransformHandle;
    public uint LastSystemVersion;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                       bool useEnabledMask, in v128 chunkEnabledMask)
    {
        // Check if any transforms in chunk changed since last update
        if (!chunk.DidChange(ref TransformHandle, LastSystemVersion))
            return;  // Skip chunk if no changes

        NativeArray<LocalTransform> transforms = chunk.GetNativeArray(ref TransformHandle);

        // Process only changed chunk
        for (int i = 0; i < chunk.Count; i++)
        {
            // React to transform changes...
        }
    }
}

// System with change filtering
[BurstCompile]
public partial struct ChangeFilterSystem : ISystem
{
    private EntityQuery _query;
    private ComponentTypeHandle<LocalTransform> _transformHandle;

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        _transformHandle.Update(ref state);

        var job = new ChangeFilterJob
        {
            TransformHandle = _transformHandle,
            LastSystemVersion = state.LastSystemVersion
        };

        state.Dependency = job.ScheduleParallel(_query, state.Dependency);
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
      private ComponentTypeHandle<LocalTransform> _transformHandle;

      public void OnCreate(ref SystemState state)
      {
          // Create handles in OnCreate
          _healthHandle = state.GetComponentTypeHandle<Health>(isReadOnly: false);
          _transformHandle = state.GetComponentTypeHandle<LocalTransform>(isReadOnly: true);
      }

      public void OnUpdate(ref SystemState state)
      {
          // CRITICAL: Update handles every frame
          _healthHandle.Update(ref state);
          _transformHandle.Update(ref state);

          // Use in job
          new MyJob { HealthHandle = _healthHandle }.ScheduleParallel(query, state.Dependency);
      }
  }
  ```

- **Type handle types:**
  ```csharp
  // Component access
  ComponentTypeHandle<T>  // Read-write or read-only component

  // Buffer access
  BufferTypeHandle<T>  // DynamicBuffer<T> access

  // Entity access
  EntityTypeHandle  // Get Entity ID from chunk

  // Shared component access
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
  public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                     bool useEnabledMask, in v128 chunkEnabledMask)
  {
      // Check if should use enabled mask
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
  }
  ```

- **Change detection:**
  ```csharp
  public void Execute(in ArchetypeChunk chunk, ...)
  {
      // Did component change since last system version?
      if (chunk.DidChange(ref transformHandle, LastSystemVersion))
      {
          // Process changed chunk
      }

      // Check multiple components
      bool healthChanged = chunk.DidChange(ref healthHandle, LastSystemVersion);
      bool transformChanged = chunk.DidChange(ref transformHandle, LastSystemVersion);

      if (healthChanged || transformChanged)
      {
          // React to changes
      }
  }
  ```

- **Scheduling methods:**
  ```csharp
  // Single-threaded execution
  JobHandle handle = job.Schedule(query, inputDeps);

  // Parallel execution (recommended)
  JobHandle handle = job.ScheduleParallel(query, inputDeps);

  // Parallel with custom batch count
  JobHandle handle = job.ScheduleParallel(query, batchesPerChunk: 1, inputDeps);
  ```

- **unfilteredChunkIndex parameter:**
  ```csharp
  public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex, ...)
  {
      // Use unfilteredChunkIndex for thread-safe writes to parallel arrays
      // Each chunk gets unique index, safe for NativeArray.ParallelWriter

      results[unfilteredChunkIndex] = computedValue;

      // Or for NativeQueue.ParallelWriter sorting key
      queue.Enqueue(entity, unfilteredChunkIndex);
  }
  ```

- **Checking if chunk has component:**
  ```csharp
  public void Execute(in ArchetypeChunk chunk, ...)
  {
      // Check if chunk archetype includes component
      if (chunk.Has(ref healthHandle))
      {
          NativeArray<Health> healths = chunk.GetNativeArray(ref healthHandle);
          // Process...
      }

      // Useful for optional components in query
  }
  ```

- **Read-only vs read-write handles:**
  ```csharp
  // OnCreate
  var readOnlyHandle = state.GetComponentTypeHandle<Health>(isReadOnly: true);
  var readWriteHandle = state.GetComponentTypeHandle<Health>(isReadOnly: false);

  // Read-only allows multiple systems to read in parallel
  // Read-write creates dependency, only one system can write at a time
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

  // WRONG: Modifying chunk.Count
  for (int i = 0; i < chunk.Count; i++)  // chunk.Count is read-only, don't cache incorrectly

  // WRONG: Using chunk data outside Execute
  NativeArray<Health> healths;  // Field in job struct
  public void Execute(in ArchetypeChunk chunk, ...)
  {
      healths = chunk.GetNativeArray(ref healthHandle);  // Don't store! Only valid in Execute
  }
  ```

## See Also

- [[IJobEntity]] - Higher-level entity iteration job (easier to use)
- [[IJobParallelFor]] - Parallel array processing
- [[EntityQuery]] - Entity filtering for jobs
- [[ComponentLookup and BufferLookup]] - Random entity access
- [[IEnableableComponent (toggleable components)|IEnableableComponent]] - Toggleable components
- [[Chunk]] - 16KiB memory blocks storing entities
