---
tags:
  - job
---

> **💡 Modern Replacement (Unity Entities 6.5+)**
>
> `IJobEntity` is the **recommended replacement** for the deprecated `Entities.ForEach` and `Job.WithCode` patterns. If you're migrating from older DOTS code, use `IJobEntity` combined with [[SystemAPI.Query]] for equivalent functionality with better performance and clearer syntax.

#### Description
- **Simplified job type** for iterating over entities with [[Burst]] compilation, automatically generating queries based on `Execute` method parameters

- Defined as `partial struct` inside an [[ISystem]], with an `Execute` method that specifies which components to process

- **Most ergonomic job type** - no manual chunk iteration, automatic query generation, supports [[IAspect (component grouping)|Aspects]] and [[RefRO and RefRW|RefRO/RefRW]] parameters

- Scheduled with `.Schedule()` or `.ScheduleParallel()` for parallel execution across worker threads

#### Example
```csharp
[BurstCompile]
public partial struct ProjectileMoveSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Schedule the job to run in parallel
        state.Dependency = new ProjectileMoveJob()
        {
            DeltaTime = SystemAPI.Time.DeltaTime
        }.ScheduleParallel(state.Dependency);
    }

    // IJobEntity with aspect parameter
    public partial struct ProjectileMoveJob : IJobEntity
    {
        public float DeltaTime;

        // Execute runs per entity - parameters define the query
        private void Execute(ProjectileAspect projectile)
        {
            projectile.Move(DeltaTime);
        }
    }
}

// Example with multiple component parameters
public partial struct DamageJob : IJobEntity
{
    // RefRO = read-only, RefRW = read-write
    private void Execute(RefRW<Health> health, in DamageBuffer damageBuffer)
    {
        foreach (var damage in damageBuffer)
        {
            health.ValueRW.Current -= damage.Amount;
        }
    }
}

// Example with entity index for EntityCommandBuffer.ParallelWriter
public partial struct KillJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;

    // [ChunkIndexInQuery] provides sort key for parallel command buffer
    private void Execute(Entity entity, [ChunkIndexInQuery] int sortKey, in Health health)
    {
        if (health.Current <= 0)
        {
            Ecb.DestroyEntity(sortKey, entity);
        }
    }
}
```

#### Pros
- **Automatic query generation** - no need to manually create [[EntityQuery]], parameters define what entities to process

- **[[Burst]] compatible** - full performance benefits of compiled jobs

- **Clean syntax** - simpler than [[IJobChunk]], no manual chunk iteration code

- **Aspect support** - can use [[IAspect (component grouping)|IAspect]] parameters for grouped component access

- **Parallel execution** - `ScheduleParallel()` distributes work across multiple threads automatically

#### Cons
- **Less control than IJobChunk** - can't manually control chunk iteration or access chunk-level data easily

- **Hidden allocations** - query generation happens behind the scenes, less visibility into what's happening

- **Limited to entity iteration** - can't do custom iteration patterns like processing chunks in specific order

#### Best use
- **Simple entity iteration** - when you need to process entities with specific components in parallel

- **Per-entity logic** - damage calculation, movement, state updates, AI behavior

- **Aspect-based systems** - when using [[IAspect (component grouping)|Aspects]] to group related components and behavior

#### Avoid if
- **You need [[IJobChunk]]** - when you need chunk-level control, accessing ComponentTypeHandle, or custom iteration

- **Physics jobs are better suited** - for collision/trigger processing, use [[ITriggerEventsJob]] instead

- **The query is trivial** - for simple single-component updates, `SystemAPI.Query` in OnUpdate may be clearer and sufficient

#### Extra tip
- **Query filtering** - use attributes on the Execute method to filter queries: `[WithAll(typeof(TagComponent))]`, `[WithNone(typeof(Disabled))]`

- **EntityCommandBuffer.ParallelWriter** - use `[ChunkIndexInQuery] int sortKey` parameter to get the sort key for parallel command buffers, ensuring deterministic playback

- **in/ref/RefRO/RefRW** - parameter modifiers control access: `in` or `RefRO<T>` for read-only, `ref` or `RefRW<T>` for read-write

- **Entity parameter** - add `Entity entity` parameter if you need the entity ID (for command buffers, lookups, etc.)

- **DynamicBuffer parameters** - can use `DynamicBuffer<T>` parameters to access [[IBufferElementData (dynamic buffers)|buffer data]]

---

## Migration from Entities.ForEach (Deprecated)

Unity Entities 6.5+ deprecated `Entities.ForEach` and `Job.WithCode`. Here's how to migrate:

### Before: Entities.ForEach (Deprecated ❌)
```csharp
// OLD - Deprecated in Unity Entities 6.5+
public partial class OldHealthSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float deltaTime = Time.DeltaTime;

        Entities
            .WithAll<Active>()
            .WithNone<Dead>()
            .ForEach((ref Health health, in HealthRegen regen) =>
            {
                health.Current += regen.PointPerSec * deltaTime;
            })
            .ScheduleParallel();
    }
}
```

### After: IJobEntity (Recommended ✅)
```csharp
// NEW - Modern approach with IJobEntity
[BurstCompile]
public partial struct ModernHealthSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        new HealthRegenJob
        {
            DeltaTime = SystemAPI.Time.DeltaTime
        }.ScheduleParallel();
    }

    [BurstCompile]
    [WithAll(typeof(Active))]      // Query filtering via attributes
    [WithNone(typeof(Dead))]
    public partial struct HealthRegenJob : IJobEntity
    {
        public float DeltaTime;

        private void Execute(ref Health health, in HealthRegen regen)
        {
            health.Current += regen.PointPerSec * DeltaTime;
        }
    }
}
```

### Key Migration Points

1. **SystemBase → ISystem**
   - Change from `SystemBase` to `ISystem` (partial struct)
   - Add `[BurstCompile]` for performance

2. **ForEach → IJobEntity**
   - Extract lambda into `IJobEntity` struct with `Execute` method
   - Move query filters to attributes: `[WithAll]`, `[WithNone]`, `[WithAny]`

3. **Scheduling**
   - `ForEach(...).ScheduleParallel()` → `new MyJob().ScheduleParallel()`
   - `ForEach(...).Schedule()` → `new MyJob().Schedule(state.Dependency)`

4. **EntityCommandBuffer**
   - Before: `var ecb = GetSingleton<ECBSystem>().CreateCommandBuffer(this)`
   - After: `var ecbSingleton = SystemAPI.GetSingleton<ECBSystem.Singleton>(); var ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged)`

### Alternative: SystemAPI.Query for Simple Cases
For very simple iterations, you can also use [[SystemAPI.Query]]:
```csharp
[BurstCompile]
public partial struct SimpleHealthSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        // Direct iteration on main thread (no job)
        foreach (var (health, regen) in
            SystemAPI.Query<RefRW<Health>, RefRO<HealthRegen>>()
                .WithAll<Active>()
                .WithNone<Dead>())
        {
            health.ValueRW.Current += regen.ValueRO.PointPerSec * dt;
        }
    }
}
```

Use `IJobEntity` when you need parallel execution, `SystemAPI.Query` for simple main-thread iteration.
