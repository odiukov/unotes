---
tags:
  - system
---
#### Description
- **Type-safe reference wrappers** for accessing [[Component|component]] data with explicit read-only (`RefRO<T>`) or read-write (`RefRW<T>`) semantics

- Used in [[SystemAPI.Query]], [[IJobEntity]], and [[IAspect (component grouping)|IAspect]] to access component data with compile-time access control

- `RefRO<T>` provides read-only access via `.ValueRO` property (analogous to `ref readonly T` or `in T`)

- `RefRW<T>` provides read-write access via `.ValueRW` property (analogous to `ref T`)

#### Example
```csharp
[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // RefRW for read-write, RefRO for read-only
        foreach (var (health, regen) in
            SystemAPI.Query<RefRW<Health>, RefRO<HealthRegen>>())
        {
            // Read-only access with .ValueRO
            float regenAmount = regen.ValueRO.PointPerSec * deltaTime;

            // Read-write access with .ValueRW
            health.ValueRW.Current = math.min(
                health.ValueRW.Current + regenAmount,
                health.ValueRO.Max  // Can use .ValueRO even on RefRW
            );
        }
    }
}

// IAspect example
public readonly partial struct ProjectileAspect : IAspect
{
    // Readonly fields in aspect
    readonly RefRO<Guided> _guided;
    readonly RefRO<Speed> _speed;

    // Read-write field
    readonly RefRW<LocalTransform> _transform;

    public void Move(float deltaTime)
    {
        // Read with .ValueRO
        if (_guided.ValueRO.Enabled)
        {
            // Write with .ValueRW
            _transform.ValueRW.Position += _speed.ValueRO.Value * deltaTime;
        }
    }
}

// IJobEntity with ref/in alternative syntax
public partial struct DamageJob : IJobEntity
{
    // ref = read-write (equivalent to RefRW)
    // in = read-only (equivalent to RefRO)
    private void Execute(ref Health health, in Damage damage)
    {
        // ref gives direct access, no .ValueRW needed
        health.Current -= damage.Amount;
    }
}
```

#### Pros
- **Type safety** - compile-time enforcement of read-only vs read-write access intent

- **Performance hints** - read-only access enables better job scheduling and parallelization

- **Clear intent** - code clearly shows which data is modified vs just read

- **[[Burst]] compatible** - works in all Burst-compiled contexts (systems, jobs, aspects)

- **Flexible access** - `RefRW<T>.ValueRO` allows reading from read-write reference

#### Cons
- **Extra syntax** - requires `.ValueRO` or `.ValueRW` property access instead of direct field access

- **Learning curve** - new developers must understand when to use RefRO vs RefRW vs ref vs in

- **Verbosity** - adds more characters compared to plain `ref` / `in` parameters

#### Best use
- **SystemAPI.Query** - use `RefRW<T>` and `RefRO<T>` for explicit access control in queries

- **IAspect fields** - use as field types in aspects to define component access patterns

- **Read-mostly data** - use `RefRO` when you only need to read, helps job system optimize parallel execution

#### Avoid if
- **IJobEntity Execute parameters** - prefer `ref T` and `in T` syntax for cleaner code in job Execute methods

- **Simple read access** - if you're only reading one field once, `in T` may be clearer than `RefRO<T>`

#### Extra tip
- **ValueRO on RefRW** - you can use `.ValueRO` on `RefRW<T>` for read-only access without triggering change tracking or preventing parallelization

- **Equivalent syntax:**
  - `RefRO<T>` ≈ `in T` ≈ `ref readonly T`
  - `RefRW<T>` ≈ `ref T`

- **Change detection optimization** - Unity can detect when you only use `.ValueRO` on `RefRW` and optimize accordingly (e.g., allowing more parallelization)

- **SystemAPI.Query requires Ref types** - can't use `ref` / `in` in foreach patterns, must use `RefRW` / `RefRO`

- **IJobEntity flexibility** - in IJobEntity.Execute, you can mix: `RefRW<Health> health`, `ref Health health`, and `in Health health` - all valid, choose based on preference

- **EnabledRefRO / EnabledRefRW** - similar concept for [[IEnableableComponent (toggleable components)|enableable components]], used to access enabled state instead of component data

- **Performance consideration** - `RefRO` allows Unity to run multiple jobs in parallel that read same component, `RefRW` requires exclusive access
