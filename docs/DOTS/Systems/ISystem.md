---
tags:
  - system
---
#### Description
- **Modern system interface** for Unity ECS - lightweight, value-type system defined as `partial struct` instead of class

- **[[Burst]] compatible** - entire system can be Burst compiled for maximum performance, including OnCreate, OnUpdate, and OnDestroy

- **Recommended system type** for all new DOTS projects - replaces the older [[SystemBase]] (class-based systems)

- Requires manual state management via `ref SystemState` parameter but provides better performance and control

#### Example
```csharp
[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Initialize system - runs once when system created
        state.RequireForUpdate<HealthRegen>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Main system logic - runs every frame
        float deltaTime = SystemAPI.Time.DeltaTime;

        foreach (var (health, regen) in
            SystemAPI.Query<RefRW<Health>, RefRO<HealthRegen>>())
        {
            health.ValueRW.Current += regen.ValueRO.PointPerSec * deltaTime;
        }
    }
}
```

#### Pros
- **[[Burst]] compatible** - entire system compiles with Burst for maximum performance
- **Zero GC allocations** - struct-based, no managed allocations
- **Better performance** - no virtual calls, better cache locality
- **Modern API** - designed for [[SystemAPI.Query]] and [[IJobEntity]]

#### Cons
- **More boilerplate** - requires `[BurstCompile]` on methods, `ref SystemState` parameter
- **No managed access** - can't directly use managed types without `SystemAPI.ManagedAPI`

#### Best use
- **All new DOTS projects** - ISystem is the recommended system type
- **Performance-critical systems** - Burst compilation provides massive speedup
- **Job-based processing** - natural fit for [[IJobEntity]]

#### Extra tip
- **Always Burst compile** - add `[BurstCompile]` to system struct and all methods
- **SystemState.Dependency** - use for job dependencies: `state.Dependency = job.Schedule(state.Dependency)`
- **RequireForUpdate** - `state.RequireForUpdate<T>()` to conditionally run system
