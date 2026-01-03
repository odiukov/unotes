## Sync Points

**Main thread blocking operation** that forces completion of scheduled jobs by calling `JobHandle.Complete()`, causing the main thread to wait until jobs finish execution.

Sync points are **performance killers** - they prevent parallel execution by forcing sequential processing. Main thread sits idle waiting for worker threads instead of scheduling more work.

### What is a Sync Point?

```csharp
JobHandle.Complete();  // Main thread waits here until job finishes
```

Conceptually equivalent to:
```csharp
public void Complete()
{
    while (!JobHandle.IsComplete())
    {
        // Main thread loops, waiting for job to finish
        // (Actually main thread helps execute jobs, but still blocking)
    }
}
```

### Types of Sync Points

**1. Read-Only Sync Point**
- Completes only jobs with **ReadWrite** access to component
- Happens when: `EntityManager.GetComponentData<T>()`, `SystemAPI.GetComponent<T>()`
- Example: Reading `LocalTransform` completes jobs that write to `LocalTransform`, but not jobs that only read it

**2. Read-Write Sync Point**
- Completes **all jobs** (both read and write) accessing component
- Happens when: `EntityManager.SetComponentData<T>()`, `SystemAPI.SetComponent<T>()`
- Example: Writing `LocalTransform` completes all jobs accessing `LocalTransform`

**3. Structural Change Sync Point**
- Completes **ALL jobs** in the entire `Unity.Entities.World`
- Happens when: `EntityManager.AddComponent/RemoveComponent/CreateEntity/DestroyEntity/Instantiate`
- Most expensive sync point type - blocks everything

### Common Sync Point Causes

**EntityManager Operations:**
```csharp
// Read-only sync on Health component
Health health = state.EntityManager.GetComponentData<Health>(entity);

// Read-write sync on Health component
state.EntityManager.SetComponentData(entity, new Health { Current = 100 });

// STRUCTURAL CHANGE - syncs ALL jobs in world!
state.EntityManager.AddComponent<NewComponent>(entity);
state.EntityManager.DestroyEntity(entity);
Entity newEntity = state.EntityManager.CreateEntity();
```

**SystemAPI Operations:**
```csharp
// Sync point - reads component on main thread
Health health = SystemAPI.GetComponent<Health>(entity);

// Sync point - writes component on main thread
SystemAPI.SetComponent(entity, new Health { Current = 100 });

// Sync point - reads singleton buffer
DynamicBuffer<Element> buffer = SystemAPI.GetSingletonBuffer<Element>();

// Sync point - iterates on main thread
foreach (var health in SystemAPI.Query<RefRW<Health>>())
{
    // Main thread iteration = sync point for Health component
}
```

**EntityCommandBuffer Playback:**
```csharp
// ECB.Playback() causes structural change sync point
EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.Temp);
ecb.CreateEntity();
ecb.Playback(state.EntityManager);  // SYNC POINT - all jobs complete!
ecb.Dispose();

// Entity Command Buffer Systems also cause sync points during their update
// EndSimulationEntityCommandBufferSystem.OnUpdate -> Playback -> sync point
```

**EntityQuery Operations:**
```csharp
// Sync point if query has enableable components or change/order filtering
bool isEmpty = query.IsEmpty;
int count = query.CalculateEntityCount();

// No sync point - ignores filtering
bool isEmpty = query.IsEmptyIgnoreFilter;
int count = query.CalculateEntityCountWithoutFiltering();
```

### Operations That DON'T Cause Sync Points

```csharp
// Getting singleton - no sync (but throws if job is scheduled)
GameConfig config = SystemAPI.GetSingleton<GameConfig>();
RefRW<PlayerState> state = SystemAPI.GetSingletonRW<PlayerState>();

// Entity count without filtering - no sync
int count = query.CalculateEntityCountWithoutFiltering();
bool isEmpty = query.IsEmptyIgnoreFilter;

// Scheduling jobs - no sync, just queues work
state.Dependency = job.Schedule(state.Dependency);
```

### Why Sync Points Kill Performance

**Bad Performance Example:**
```csharp
// Frame execution timeline:
// [Main: Schedule JobA] → [Wait for JobA] → [Main: Schedule JobB] → [Wait for JobB]
//    Worker threads only used 50% of time, main thread wastes time waiting

SystemX.OnUpdate:
    Schedule JobA  // Worker thread starts JobA

SystemY.OnUpdate:
    EntityManager.GetComponent<T>()  // SYNC POINT! Main thread waits for JobA

SystemZ.OnUpdate:
    Schedule JobB  // Worker thread starts JobB

SomeSystem.OnUpdate:
    EntityManager.SetComponent<T>()  // SYNC POINT! Main thread waits for JobB
```

**Good Performance Example:**
```csharp
// Frame execution timeline:
// [Main: Schedule JobA, JobB, JobC...] → [Final sync point]
//    Worker threads utilized 100%, main thread schedules efficiently

SystemX.OnUpdate:
    Schedule JobA  // Worker starts JobA while main continues

SystemY.OnUpdate:
    Schedule JobB  // Worker starts JobB while main continues

SystemZ.OnUpdate:
    Schedule JobC  // Worker starts JobC while main continues

// Only sync at end of frame or when absolutely necessary
// By then, many jobs already finished - minimal waiting
```

### How to Avoid Sync Points

**1. Schedule Jobs Instead of Main Thread Access:**
```csharp
// BAD: Main thread access = sync point
foreach (var health in SystemAPI.Query<RefRW<Health>>())
    health.ValueRW.Current += 10;

// GOOD: Job access = no sync point
new HealJob().ScheduleParallel();
```

**2. Use EntityCommandBuffer Instead of Direct EntityManager:**
```csharp
// BAD: Immediate structural change = sync ALL jobs
state.EntityManager.DestroyEntity(entity);

// GOOD: Deferred via ECB = no sync (until ECB playback)
var ecb = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
    .CreateCommandBuffer(state.WorldUnmanaged);
ecb.DestroyEntity(entity);
```

**3. Organize System Order:**
```
Frame Structure:
├── Main Thread Systems (input, UI, managed code)
├── Job Scheduling Systems (schedule all jobs)
└── End of frame (jobs complete naturally)

No sync points between job scheduling systems!
```

**4. Use Singleton Read-Only Methods:**
```csharp
// GetSingleton doesn't sync (but throws if job scheduled with it)
GameConfig config = SystemAPI.GetSingleton<GameConfig>();

// GetSingletonBuffer syncs - avoid or use in job
// DynamicBuffer<T> buffer = SystemAPI.GetSingletonBuffer<T>();
```

### Best Practices

- **Minimize structural changes** - use [[IEnableableComponent (toggleable components)|IEnableableComponent]] instead of add/remove
- **Batch ECB usage** - use one ECB system per frame, not multiple
- **Schedule jobs for data access** - avoid main thread component iteration
- **Check Unity Profiler** - look for sync points in "Sync Point" markers
- **Front-load main thread work** - do main thread systems early, job systems late

**See also:** [[Job]], [[EntityCommandBuffer]], [[Job Safety System]], [[System Dependencies]]
