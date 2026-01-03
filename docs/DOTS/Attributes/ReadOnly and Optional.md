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
// [ReadOnly] on lookups in jobs
[BurstCompile]
public partial struct ProjectileMoveSystem : ISystem
{
    private ComponentLookup<LocalToWorld> _positions;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Create read-only lookup (true parameter)
        _positions = SystemAPI.GetComponentLookup<LocalToWorld>(true);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        _positions.Update(ref state);

        new ProjectileMoveJob()
        {
            Positions = _positions
        }.ScheduleParallel();  // Parallel execution safe with [ReadOnly]
    }

    public partial struct ProjectileMoveJob : IJobEntity
    {
        // [ReadOnly] enables parallel execution without race conditions
        [ReadOnly] public ComponentLookup<LocalToWorld> Positions;

        private void Execute(ref LocalTransform transform, in Target target)
        {
            if (Positions.HasComponent(target.Entity))
            {
                // Read-only access to target position
                float3 targetPos = Positions[target.Entity].Position;
                // ... use target position
            }
        }
    }
}

// [Optional] in aspects
public readonly partial struct CharacterAspect : IAspect
{
    readonly RefRW<LocalTransform> _transform;  // Required
    readonly RefRW<Health> _health;             // Required

    // Optional component - entity can match query without Shield
    [Optional] readonly RefRW<Shield> _shield;

    public void TakeDamage(float damage)
    {
        // Check if optional component exists
        if (_shield.IsValid)
        {
            // Access with .ValueRW like normal
            _shield.ValueRW.Amount -= damage;
            if (_shield.ValueRO.Amount <= 0)
            {
                damage = -_shield.ValueRO.Amount; // Overflow damage
            }
            else
            {
                return; // Shield absorbed damage
            }
        }

        // Apply remaining damage to health
        _health.ValueRW.Current -= damage;
    }
}

// [ReadOnly] on DynamicBuffer in aspects
public readonly partial struct PathFollowerAspect : IAspect
{
    readonly RefRW<LocalTransform> _transform;
    readonly RefRW<NextPathIndex> _pathIndex;

    // [ReadOnly] makes buffer read-only in aspect
    [ReadOnly] readonly DynamicBuffer<Waypoints> _waypoints;

    public void MoveAlongPath(float deltaTime, float speed)
    {
        // Can read from buffer but not modify
        if (_pathIndex.ValueRO.Value >= _waypoints.Length)
            return;

        float3 targetPos = _waypoints[_pathIndex.ValueRO.Value].Position;
        // ... movement logic
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
- **[ReadOnly] and parallel jobs** - Unity's job system can run multiple jobs in parallel if they only read shared data, `[ReadOnly]` enables this optimization

- **ComponentLookup read-only creation** - `SystemAPI.GetComponentLookup<T>(true)` creates read-only lookup, `(false)` creates read-write

- **[Optional] with RefRO/RefRW** - can use on either `RefRO<T>` or `RefRW<T>`, check `.IsValid` property before accessing

- **IsValid pattern:**
  ```csharp
  [Optional] readonly RefRW<Shield> _shield;

  if (_shield.IsValid)  // Check before access
  {
      _shield.ValueRW.Amount -= 10;  // Safe to access
  }
  ```

- **[Optional] DynamicBuffer** - can mark DynamicBuffer as optional in aspects, check `.IsValid` before accessing

- **[Optional] nested aspects** - can mark nested aspects as optional, allowing hierarchical optional component groups

- **Performance consideration** - `[ReadOnly]` lookups are faster to create and update than read-write lookups

- **Job dependency tracking** - Unity's job system uses `[ReadOnly]` to automatically track dependencies and schedule jobs efficiently

- **Multiple [ReadOnly] fields** - can have unlimited `[ReadOnly]` lookups in a job, all can be accessed in parallel

- **Race condition prevention** - without `[ReadOnly]`, parallel writes to same data cause race conditions and undefined behavior, Unity will error instead

- **Attribute on native collections** - `[ReadOnly]` also works on NativeArray, NativeList, NativeHashMap when passed to jobs for read-only access
