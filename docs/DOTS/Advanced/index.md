# Advanced DOTS Concepts

Deep dives into Unity DOTS internals, performance optimization, and advanced patterns. These topics assume familiarity with basic DOTS concepts.

## Core Advanced Topics

### [[Job Safety System]]
Understanding how Unity prevents race conditions in parallel code through atomic safety handles and dependency tracking.

**Key concepts:**
- AtomicSafetyHandle for read/write validation
- Job scheduling safety checks
- Container access safety during execution
- Common safety errors and solutions

### [[System Dependencies]]
How Unity ECS automatically manages job dependencies across systems to prevent race conditions while enabling parallelization.

**Key concepts:**
- Component dependency registry per World
- SystemState.Dependency lifecycle (BeforeUpdate → OnUpdate → AfterUpdate)
- Automatic dependency chaining based on component access
- Read-only vs read-write dependency separation

### [[Sync points]]
When and why jobs are forced to complete, killing parallel performance. Critical for optimization.

**Key concepts:**
- Three types: ReadOnly, ReadWrite, Structural Change
- Common sync point causes (EntityManager access, ECB playback, queries)
- Performance impact and how to avoid
- Organizing systems to minimize sync points

### [[API Gotchas]]
Non-obvious behaviors of EntityManager methods that can cause performance problems or bugs.

**Key gotchas:**
- `AddComponent(EntityQuery)` vs `AddComponent(Entity)` - chunk modification vs entity movement
- `Instantiate(Entity, NativeArray)` - doesn't instantiate LinkedEntityGroup
- When to use bulk operations vs individual operations

## Advanced Workflow Topics

### [[Baking Workflow and SubScenes]]
Complete guide to the baking process, SubScene streaming, and content management.

**Key concepts:**
- Editor vs Build baking workflows
- Live-baking for iteration
- SubScene streaming and async loading
- Content archives and dependency tracking

## Performance Optimization

**To maximize DOTS performance:**
1. **Minimize [[Sync points]]** - avoid EntityManager.Get/SetComponentData, batch structural changes
2. **Maximize parallelization** - understand [[System Dependencies]] to run jobs in parallel
3. **Use [[Job Safety System]]** knowledge - mark lookups `[ReadOnly]` for parallel access
4. **Avoid [[API Gotchas]]** - use correct overloads to prevent chunk fragmentation

## Best Practices Summary

### Job Scheduling
```csharp
// ✅ GOOD: Jobs chain dependencies, run in parallel
JobHandle jobA = new JobA().Schedule(state.Dependency);
JobHandle jobB = new JobB().Schedule(state.Dependency);  // Parallel with A
state.Dependency = JobHandle.CombineDependencies(jobA, jobB);
```

### Sync Point Avoidance
```csharp
// ❌ BAD: Main thread access = sync point
Health h = EntityManager.GetComponentData<Health>(entity);

// ✅ GOOD: Job access = no sync point
new ProcessHealthJob().ScheduleParallel();
```

### System Organization
```
Frame Structure:
├─ Input/UI Systems (main thread, unavoidable sync points)
├─ Job Scheduling Systems (all jobs scheduled here)
└─ Final sync at frame end (jobs complete naturally while rendering)
```

### Structural Changes
```csharp
// ❌ BAD: Immediate structural change = sync ALL jobs
EntityManager.AddComponent<Dead>(entity);

// ✅ GOOD: Deferred via ECB = no sync until playback
ecb.AddComponent<Dead>(entity);
```

## Common Optimization Mistakes

1. **Excessive sync points** - calling EntityManager.GetComponent every frame
2. **Poor system ordering** - systems with conflicting access not grouped efficiently
3. **Bulk API misuse** - using EntityQuery overloads for small entity counts
4. **Ignoring LinkedEntityGroup** - using NativeArray Instantiate on prefabs with children
5. **Live-baking assumptions** - assuming Editor behavior matches builds

## Debugging Tools

- **Unity Profiler** - Look for "Sync Point" markers
- **Entity Debugger** - Inspect chunk utilization (check for sparse chunks)
- **Burst Inspector** - Verify job compilation and optimizations
- **Build Report** - Check content archive sizes

## Advanced Reading

For more information, see:
- [Unity Job System Manual](https://docs.unity3d.com/Manual/JobSystem.html)
- [Unity.Entities Manual](https://docs.unity3d.com/Packages/com.unity.entities@latest)
- [Unity.Burst Manual](https://docs.unity3d.com/Packages/com.unity.burst@latest)

