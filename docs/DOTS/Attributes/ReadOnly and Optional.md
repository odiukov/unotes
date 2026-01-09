---
tags:
  - attribute
---
#### Description
- **Access control attributes** used in jobs and [[IAspect (component grouping)|aspects]] to specify component access patterns

- `[ReadOnly]` marks [[ComponentLookup and BufferLookup|ComponentLookup/BufferLookup]], `DynamicBuffer`, or native collections as read-only, enabling **parallel job execution** without race conditions

- `[Optional]` marks [[IAspect (component grouping)|aspect]] component fields as optional - query matches entities even if optional component is missing

- Both attributes improve job scheduling efficiency and provide flexibility in component queries

#### Example
```csharp
// [ReadOnly] on lookups - enables parallel job execution
[BurstCompile]
public partial struct ProjectileMoveJob : IJobEntity
{
    [ReadOnly] public ComponentLookup<LocalToWorld> Positions;  // Multiple threads can read

    private void Execute(ref LocalTransform transform, in Target target)
    {
        if (Positions.HasComponent(target.Entity))
        {
            float3 targetPos = Positions[target.Entity].Position;
            // ... use target position
        }
    }
}

// [Optional] in aspects - component not required for query match
public readonly partial struct CharacterAspect : IAspect
{
    readonly RefRW<LocalTransform> _transform;  // Required
    readonly RefRW<Health> _health;             // Required
    [Optional] readonly RefRW<Shield> _shield;  // Optional

    public void TakeDamage(float damage)
    {
        if (_shield.IsValid)  // Check before accessing optional component
        {
            _shield.ValueRW.Amount -= damage;
            if (_shield.ValueRO.Amount > 0)
                return;  // Shield absorbed damage
            damage = -_shield.ValueRO.Amount;  // Overflow damage
        }
        _health.ValueRW.Current -= damage;
    }
}
```

#### Pros
- **[ReadOnly] enables parallelization** - multiple threads can safely read same data, improving performance

- **[ReadOnly] prevents bugs** - compile-time error if you try to write to read-only data

- **[Optional] increases query flexibility** - one aspect/query can match entities with or without optional components

- **[Optional] reduces code duplication** - don't need separate systems/aspects for entities with/without optional components

#### Cons
- **[ReadOnly] prevents writes** - can't modify data marked as read-only, need separate read-write access if needed

- **[Optional] requires null checks** - must check `.IsValid` before accessing optional fields, easy to forget

- **[Optional] limited to aspects** - can't use on regular query parameters, only in IAspect fields

#### Best use
- **[ReadOnly] on lookups** - always mark read-only lookups with `[ReadOnly]` for parallel job safety

- **[ReadOnly] on shared data** - when multiple jobs need to read same reference data (configuration, target positions, etc.)

- **[Optional] for optional gameplay features** - shields, buffs, special abilities that not all characters have

- **[Optional] for progressive upgrades** - components added over time (armor, weapons, perks)

#### Avoid if
- **Need to write data** - don't use `[ReadOnly]` if you need to modify the data (use read-write access)

- **Component is required** - don't use `[Optional]` if component must always be present (defeats type safety)

#### Extra tip
- **ComponentLookup creation** - `SystemAPI.GetComponentLookup<T>(true)` creates read-only lookup, `(false)` creates read-write

- **[Optional] with RefRO/RefRW** - can use on either `RefRO<T>` or `RefRW<T>`, check `.IsValid` property before accessing

- **[Optional] DynamicBuffer** - can mark DynamicBuffer as optional in aspects, check `.IsValid` before accessing

- **Job dependency tracking** - Unity's job system uses `[ReadOnly]` to automatically track dependencies and schedule jobs efficiently

- **Multiple [ReadOnly] fields** - can have unlimited `[ReadOnly]` lookups in a job, all can be accessed in parallel safely

- **Native collections** - `[ReadOnly]` also works on NativeArray, NativeList, NativeHashMap when passed to jobs
