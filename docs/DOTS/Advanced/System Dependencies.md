---
tags:
  - advanced
  - systems
---
#### Description
- **Automatic job dependency management** across systems - Unity ECS ensures jobs in different systems that access the same components are properly ordered to prevent race conditions

- Based on **component dependency registry** in `Unity.Entities.World` - tracks which jobs are reading/writing each component type

- Systems register component types they access (via [[EntityQuery]], `GetComponentTypeHandle`, `GetComponentLookup`) and Unity automatically chains job dependencies

- **SystemState.Dependency** is the key mechanism - before OnUpdate gets dependencies for accessed components, after OnUpdate writes back final dependency

#### How It Works

**Component Dependency Registry:**
- Each `World` maintains a registry mapping component types to `JobHandle`
- Initially all component dependencies are `default(JobHandle)`
- Registry tracks both read-only (RO) and read-write (RW) access separately

**System Update Lifecycle:**

**Before OnUpdate (BeforeUpdate):**
1. System has registered component types it accesses (gathered in OnCreate or OnUpdate)
2. Unity gets `JobHandle` for each component type from registry
3. Combines all handles into one `JobHandle`
4. Assigns combined handle to `SystemState.Dependency`

**During OnUpdate:**
1. User schedules jobs using `SystemState.Dependency` as input dependency
2. Jobs return `JobHandle` which gets assigned back to `SystemState.Dependency`
3. All jobs implicitly depend on all previous systems that accessed same components

**After OnUpdate (AfterUpdate):**
1. Final `SystemState.Dependency` is written back to component dependency registry
2. Now registry contains dependency on all jobs this system scheduled

#### Example

```csharp
// Example scenario: System X reads A and B, System Y writes B

// === Initial State ===
// Component Registry:
// A -> default(JobHandle)
// B -> default(JobHandle)

[BurstCompile]
public partial struct SystemX : ISystem
{
    private EntityQuery _query;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Registers that this system accesses components A and B (read-only)
        _query = SystemAPI.QueryBuilder()
            .WithAll<ComponentA, ComponentB>()
            .Build();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // BeforeUpdate: state.Dependency = default(JobHandle)
        // (both A and B have default dependency)

        // Schedule job that reads A and B
        state.Dependency = new ReadJob
        {
            ComponentA = SystemAPI.GetComponentTypeHandle<ComponentA>(true),
            ComponentB = SystemAPI.GetComponentTypeHandle<ComponentB>(true)
        }.ScheduleParallel(_query, state.Dependency);

        // AfterUpdate: Registry updated:
        // A -> SystemX.Dependency
        // B -> SystemX.Dependency
    }
}

// System Y updates after System X
[UpdateAfter(typeof(SystemX))]
[BurstCompile]
public partial struct SystemY : ISystem
{
    private EntityQuery _query;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Registers that this system writes to component B
        _query = SystemAPI.QueryBuilder()
            .WithAll<ComponentB>()
            .Build();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // BeforeUpdate: state.Dependency = SystemX.Dependency
        // (because component B was written to by SystemX)
        // This means any job scheduled by SystemY will wait for SystemX jobs!

        state.Dependency = new WriteJob
        {
            ComponentB = SystemAPI.GetComponentTypeHandle<ComponentB>(false)
        }.ScheduleParallel(_query, state.Dependency);

        // AfterUpdate: Registry updated:
        // A -> SystemX.Dependency (unchanged, SystemY doesn't access A)
        // B -> SystemY.Dependency (updated to SystemY's jobs)
    }
}

// System Z has no dependency conflicts
[UpdateAfter(typeof(SystemY))]
[BurstCompile]
public partial struct SystemZ : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Only accesses component C - no dependency on SystemX or SystemY
        // state.Dependency = default(JobHandle) for component C
        // SystemZ jobs run in PARALLEL with SystemX and SystemY!

        foreach (var c in SystemAPI.Query<RefRW<ComponentC>>())
        {
            c.ValueRW.Value += 1;
        }
    }
}
```

#### Component Registration Points

Components are registered when system calls:

1. **EntityQuery creation:**
   ```csharp
   var query = SystemAPI.QueryBuilder().WithAll<Health>().Build();
   ```

2. **GetComponentTypeHandle:**
   ```csharp
   var healthHandle = SystemAPI.GetComponentTypeHandle<Health>(true);
   ```

3. **GetComponentLookup:**
   ```csharp
   var healthLookup = SystemAPI.GetComponentLookup<Health>(false);
   ```

4. **SystemAPI.Query in OnUpdate:**
   ```csharp
   foreach (var health in SystemAPI.Query<RefRW<Health>>())
   ```

#### EntityCommandBuffer System Dependencies

```csharp
[BurstCompile]
public partial struct SpawnerSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // GetSingleton registers dependency on ECB singleton component
        state.RequireForUpdate<BeginSimulationEntityCommandBufferSystem.Singleton>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Gets ECB singleton - registers RW dependency
        var ecbSingleton = SystemAPI.GetSingleton<BeginSimulationEntityCommandBufferSystem.Singleton>();
        EntityCommandBuffer ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

        // Schedule jobs that write to ECB
        new SpawnJob { Ecb = ecb.AsParallelWriter() }.ScheduleParallel();
        // state.Dependency now contains dependency on spawn job

        // ECB System will Complete() the RW dependency on its singleton
        // This ensures all jobs writing to ECB complete before playback
    }
}
```

#### Pros
- **Automatic safety** - no manual dependency tracking across systems, Unity handles it

- **Prevents race conditions** - jobs accessing same components are automatically ordered correctly

- **Enables parallelization** - systems accessing different components run jobs in parallel

- **Simple API** - just use `SystemState.Dependency`, Unity does the rest

#### Cons
- **Hidden dependencies** - not obvious from code what dependencies exist

- **Can be confusing** - requires understanding of how component registry works

- **Over-conservative sometimes** - may create dependencies that aren't strictly necessary

#### Best use
- **All [[ISystem]] implementations** - dependency system is fundamental to ECS, always active

- **Cross-system job coordination** - ensures jobs in different systems don't conflict

- **Performance optimization** - enables parallel execution of independent systems

#### Avoid if
- N/A - system dependencies are built into Unity ECS, cannot be avoided

#### Extra tip
- **Read-only vs Read-write separation** - registry separates RO and RW dependencies, allowing multiple readers in parallel but exclusive writers

- **state.Dependency is the key** - always schedule jobs with it as input and assign result back:
  ```csharp
  state.Dependency = job.Schedule(state.Dependency);
  ```

- **Dependency combining** - if scheduling multiple independent jobs:
  ```csharp
  JobHandle jobA = jobA.Schedule(state.Dependency);
  JobHandle jobB = jobB.Schedule(state.Dependency);
  state.Dependency = JobHandle.CombineDependencies(jobA, jobB);
  ```

- **System ordering affects efficiency** - systems with overlapping component access should be ordered carefully to minimize sync points

- **EntityQuery caching** - queries created in OnCreate register components once, queries created in OnUpdate register every frame

- **SystemAPI.Query auto-registers** - using `SystemAPI.Query<T>()` automatically registers component T

- **Manual dependency override** - can manually set `state.Dependency` to break automatic dependency chain (advanced, usually not recommended)

- **Dependency tracking is per-component-type** - two systems accessing different component types have no dependency relationship

- **World-specific** - each World has its own component dependency registry, dependencies don't cross worlds

## Practical Example: Parallel Systems

```csharp
// These three systems can run jobs in PARALLEL
// because they access different components

[BurstCompile]
public partial struct HealthRegenSystem : ISystem  // Accesses: Health
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var health in SystemAPI.Query<RefRW<Health>>())
            health.ValueRW.Current += 1;
    }
}

[BurstCompile]
public partial struct MovementSystem : ISystem  // Accesses: Position, Velocity
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (pos, vel) in SystemAPI.Query<RefRW<Position>, RefRO<Velocity>>())
            pos.ValueRW.Value += vel.ValueRO.Value;
    }
}

[BurstCompile]
public partial struct DamageSystem : ISystem  // Accesses: Damage
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var damage in SystemAPI.Query<RefRW<Damage>>())
            damage.ValueRW.Amount = 0;
    }
}

// All three run in parallel - maximum CPU utilization!
```

## See Also

- [[Job Safety System]] - How job safety works
- [[Sync points]] - When jobs are forced to complete
- [[UpdateInGroup, UpdateBefore, UpdateAfter]] - System ordering
- [[ISystem]] - System interface
