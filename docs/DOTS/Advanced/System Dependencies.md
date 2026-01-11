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
// System X reads components A and B
[BurstCompile]
public partial struct SystemX : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // BeforeUpdate: state.Dependency = default (no prior dependencies)

        state.Dependency = new ReadJob
        {
            ComponentA = SystemAPI.GetComponentTypeHandle<ComponentA>(true),
            ComponentB = SystemAPI.GetComponentTypeHandle<ComponentB>(true)
        }.ScheduleParallel(query, state.Dependency);

        // AfterUpdate: Registry updated: A -> SystemX.Dependency, B -> SystemX.Dependency
    }
}

// System Y writes component B - automatically waits for SystemX
[UpdateAfter(typeof(SystemX))]
[BurstCompile]
public partial struct SystemY : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // BeforeUpdate: state.Dependency = SystemX.Dependency
        // (Unity automatically chains dependency because B was accessed)

        state.Dependency = new WriteJob
        {
            ComponentB = SystemAPI.GetComponentTypeHandle<ComponentB>(false)
        }.ScheduleParallel(query, state.Dependency);

        // AfterUpdate: Registry updated: B -> SystemY.Dependency
    }
}

// System Z accesses different component - runs in parallel
[BurstCompile]
public partial struct SystemZ : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // No dependency on SystemX or SystemY - different component
        // Runs in PARALLEL with other systems!
        foreach (var c in SystemAPI.Query<RefRW<ComponentC>>())
            c.ValueRW.Value += 1;
    }
}
```

#### Component Registration

Components are registered when system calls:
- `SystemAPI.QueryBuilder().WithAll<Health>().Build()`
- `SystemAPI.GetComponentTypeHandle<Health>()`
- `SystemAPI.GetComponentLookup<Health>()`
- `SystemAPI.Query<RefRW<Health>>()`

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
- **Always use state.Dependency** - schedule jobs with it as input and assign result back:
  ```csharp
  state.Dependency = job.Schedule(state.Dependency);
  ```

- **Dependency combining** - if scheduling multiple independent jobs:
  ```csharp
  JobHandle jobA = jobA.Schedule(state.Dependency);
  JobHandle jobB = jobB.Schedule(state.Dependency);
  state.Dependency = JobHandle.CombineDependencies(jobA, jobB);
  ```

- **Read-only vs Read-write** - registry separates RO and RW dependencies, allowing multiple readers in parallel but exclusive writers

- **SystemAPI.Query auto-registers** - using `SystemAPI.Query<T>()` automatically registers component T

- **Per-component-type tracking** - systems accessing different component types have no dependency relationship

- **World-specific** - each World has its own registry, dependencies don't cross worlds

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

