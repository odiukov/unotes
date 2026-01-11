---
tags:
  - system
  - legacy
---

> **⚠️ LEGACY SYSTEM TYPE**
>
> `SystemBase` is the **legacy class-based system** type. For all new projects, use [[ISystem]] instead. SystemBase remains supported but [[ISystem]] offers better performance through [[Burst]] compilation and zero GC allocations.
>
> **Modern alternative:** [[ISystem]] (partial struct with Burst support)

#### Description
- **Legacy class-based system** type for Unity ECS - defined as `class` instead of struct

- **Cannot be [[Burst]] compiled** - OnUpdate and other methods run as managed C# code, missing major performance benefits

- **Easier managed API access** - can directly use managed types, LINQ, and C# features not available in Burst

- **Entities.ForEach** - originally designed for now-deprecated `Entities.ForEach` pattern (replaced by [[IJobEntity]])

#### Example
```csharp
// Legacy SystemBase approach
public partial class HealthRegenSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float deltaTime = Time.DeltaTime;

        // Old Entities.ForEach (deprecated in Unity Entities 6.5+)
        Entities
            .ForEach((ref Health health, in HealthRegen regen) =>
            {
                health.Current += regen.PointPerSec * deltaTime;
            })
            .ScheduleParallel();
    }
}

// Modern ISystem equivalent (RECOMMENDED)
[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        new RegenJob
        {
            DeltaTime = SystemAPI.Time.DeltaTime
        }.ScheduleParallel();
    }

    [BurstCompile]
    public partial struct RegenJob : IJobEntity
    {
        public float DeltaTime;
        private void Execute(ref Health health, in HealthRegen regen)
        {
            health.Current += regen.PointPerSec * DeltaTime;
        }
    }
}
```

#### Pros
- **Managed C# access** - can use LINQ, strings, managed collections, exceptions

- **Easier for beginners** - familiar class-based OOP pattern

- **Direct managed component access** - no need for `SystemAPI.ManagedAPI`

#### Cons
- **No [[Burst]] compilation** - misses 10-100x performance improvements from Burst

- **GC allocations** - class-based systems create garbage, causing GC pauses

- **Slower** - virtual method calls, poor cache locality compared to [[ISystem]]

- **Deprecated patterns** - designed for `Entities.ForEach` which is now deprecated

#### Best use
- **Legacy codebase migration** - temporary use while migrating to [[ISystem]]

- **Heavy managed API usage** - if system must use lots of managed Unity APIs (rare)

#### Avoid if
- **New project** - always use [[ISystem]] for new DOTS projects

- **Performance matters** - SystemBase is significantly slower than [[ISystem]]

- **Can use [[ISystem]]** - in 99% of cases, ISystem is the better choice

#### Extra tip
- **Migration path** - when migrating to [[ISystem]]:
  1. Change `class` to `partial struct`
  2. Add `[BurstCompile]` attribute
  3. Add `ref SystemState state` parameter to OnUpdate
  4. Replace `Entities.ForEach` with [[IJobEntity]]
  5. Replace direct property access with `state.Property`

- **Property differences:**
  ```csharp
  // SystemBase
  EntityManager em = EntityManager;
  float dt = Time.DeltaTime;
  JobHandle dep = Dependency;

  // ISystem
  EntityManager em = state.EntityManager;
  float dt = SystemAPI.Time.DeltaTime;
  JobHandle dep = state.Dependency;
  ```

- **Still supported** - SystemBase will continue to work, but is not recommended for new code

