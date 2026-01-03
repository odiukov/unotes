# Attributes

Attributes control system behavior, compilation, execution order, and access patterns in Unity DOTS. They're essential for performance optimization and correctness.

## Compilation & Performance

- **[[BurstCompile]]** - Compiles systems and jobs to highly optimized native code (10-100x speedup)

## System Ordering

- **[[UpdateInGroup, UpdateBefore, UpdateAfter]]** - Control when systems execute in the frame
  - `[UpdateInGroup(typeof(Group))]` - Place system in specific system group
  - `[UpdateBefore(typeof(System))]` - Run before another system
  - `[UpdateAfter(typeof(System))]` - Run after another system

## System Optimization

- **[[RequireMatchingQueriesForUpdate]]** - Skip system execution when no entities match queries
- **[[ChunkIndexInQuery]]** - Provides sort key for deterministic EntityCommandBuffer.ParallelWriter

## Access Control

- **[[ReadOnly and Optional]]** - Control component and lookup access patterns
  - `[ReadOnly]` - Mark lookups/buffers as read-only for parallel job safety
  - `[Optional]` - Mark aspect components as optional in queries

## Usage Patterns

### System with Burst and Ordering
```csharp
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
[UpdateAfter(typeof(PhysicsSystemGroup))]
[BurstCompile]
public partial struct MySystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state) { }
}
```

### Job with Parallel EntityCommandBuffer
```csharp
public partial struct MyJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;

    private void Execute(Entity entity, [ChunkIndexInQuery] int sortKey)
    {
        Ecb.DestroyEntity(sortKey, entity);
    }
}
```

### Aspect with Optional Component
```csharp
public readonly partial struct MyAspect : IAspect
{
    readonly RefRW<Health> _health;
    [Optional] readonly RefRW<Shield> _shield;
}
```

## Best Practices

1. **Always use [[BurstCompile]]** on systems and jobs for performance
2. **Use system ordering attributes** to ensure correct execution order
3. **Mark read-only lookups** with `[ReadOnly]` to enable parallelization
4. **Use [[ChunkIndexInQuery]]** with ParallelWriter for determinism
5. **Add [[RequireMatchingQueriesForUpdate]]** for systems that may have no work

## See Also

- [[ISystem]] - System interface
- [[IJobEntity]] - Job type
- [[IAspect (component grouping)]] - Aspect interface
- [[EntityCommandBuffer]] - Deferred structural changes
