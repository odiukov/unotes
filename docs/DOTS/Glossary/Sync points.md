## Sync Points

**Main thread blocking operation** that forces completion of scheduled jobs by calling `JobHandle.Complete()`, causing the main thread to wait until jobs finish execution.

Sync points are **performance killers** - they prevent parallel execution by forcing sequential processing.

### Types of Sync Points

**1. Read-Only Sync Point**
- Completes jobs with **ReadWrite** access to component
- Triggered by: `EntityManager.GetComponentData<T>()`, `SystemAPI.GetComponent<T>()`

**2. Read-Write Sync Point**
- Completes **all jobs** (both read and write) accessing component
- Triggered by: `EntityManager.SetComponentData<T>()`, `SystemAPI.SetComponent<T>()`

**3. Structural Change Sync Point**
- Completes **ALL jobs** in the entire [[World]]
- Triggered by: `EntityManager.AddComponent/RemoveComponent/CreateEntity/DestroyEntity`, `ECB.Playback()`
- Most expensive - blocks everything

### Common Causes

```csharp
// Main thread component access
Health h = state.EntityManager.GetComponentData<Health>(entity);
state.EntityManager.SetComponentData(entity, new Health { Current = 100 });

// Main thread iteration
foreach (var health in SystemAPI.Query<RefRW<Health>>())
    health.ValueRW.Current += 10;

// Structural changes
state.EntityManager.AddComponent<Tag>(entity);
state.EntityManager.DestroyEntity(entity);

// ECB playback
ecb.Playback(state.EntityManager);

// Query operations with filtering
int count = query.CalculateEntityCount();  // Syncs if has enableable/change filters
```

### Operations That DON'T Sync

```csharp
// Singleton access (but throws if job is scheduled with it)
GameConfig config = SystemAPI.GetSingleton<GameConfig>();

// Entity count without filtering
int count = query.CalculateEntityCountWithoutFiltering();
bool isEmpty = query.IsEmptyIgnoreFilter;

// Job scheduling
state.Dependency = job.Schedule(state.Dependency);
```

### How to Avoid

**1. Schedule jobs instead of main thread access:**
```csharp
// ❌ BAD: Main thread access = sync point
foreach (var health in SystemAPI.Query<RefRW<Health>>())
    health.ValueRW.Current += 10;

// ✅ GOOD: Job access = no sync point
new HealJob().ScheduleParallel();
```

**2. Use EntityCommandBuffer for structural changes:**
```csharp
// ❌ BAD: Immediate = sync ALL jobs
state.EntityManager.DestroyEntity(entity);

// ✅ GOOD: Deferred = no sync until playback
ecb.DestroyEntity(entity);
```

**3. Organize system order:**
```
Frame Structure:
├── Input/UI Systems (unavoidable sync points)
├── Job Scheduling Systems (no sync points!)
└── End of frame (natural completion)
```

### Best Practices

- **Minimize structural changes** - use [[IEnableableComponent (toggleable components)|IEnableableComponent]] instead of add/remove
- **Batch ECB usage** - use one ECB system per frame
- **Schedule jobs for data access** - avoid main thread component iteration
- **Check Unity Profiler** - look for "Sync Point" markers

**See also:** [[Job]], [[EntityCommandBuffer]], [[Job Safety System]], [[System Dependencies]]
