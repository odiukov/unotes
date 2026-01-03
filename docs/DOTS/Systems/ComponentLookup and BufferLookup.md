---
tags:
  - system
---
#### Description
- **Random access containers** that provide access to component/buffer data of arbitrary entities, similar to a dictionary lookup by [[Entity]] ID

- `ComponentLookup<T>` provides random access to [[IComponentData]], `BufferLookup<T>` provides random access to [[IBufferElementData (dynamic buffers)|DynamicBuffer]]

- Created in `OnCreate` with `SystemAPI.GetComponentLookup<T>(isReadOnly)` and **must be updated** with `lookup.Update(ref state)` before each use

- Involves **random memory access** so less performant than sequential iteration with [[SystemAPI.Query]] or [[IJobEntity]], but necessary for entity references and relationships

#### Example
```csharp
[BurstCompile]
public partial struct ProjectileMoveSystem : ISystem
{
    // Store lookup as system field
    private ComponentLookup<LocalToWorld> _positions;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Create read-only lookup (true = read-only)
        _positions = SystemAPI.GetComponentLookup<LocalToWorld>(true);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // CRITICAL: Update before use - entity storage may have moved
        _positions.Update(ref state);

        new ProjectileMoveJob()
        {
            DeltaTime = SystemAPI.Time.DeltaTime,
            Positions = _positions
        }.ScheduleParallel();
    }

    public partial struct ProjectileMoveJob : IJobEntity
    {
        public float DeltaTime;
        [ReadOnly] public ComponentLookup<LocalToWorld> Positions;

        private void Execute(ref LocalTransform transform, in Target target)
        {
            // Check if target entity still has the component
            if (Positions.HasComponent(target.Entity))
            {
                // Random access to target's position
                float3 targetPos = Positions[target.Entity].Position;
                transform.Position = math.lerp(transform.Position, targetPos, DeltaTime);
            }
        }
    }
}

// BufferLookup example
[BurstCompile]
public partial struct EnemyCollisionSystem : ISystem
{
    BufferLookup<Waypoints> _waypointLookup;
    ComponentLookup<Player> _playerLookup;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Read-only buffer lookup
        _waypointLookup = SystemAPI.GetBufferLookup<Waypoints>(true);
        // Read-write component lookup (false = read-write)
        _playerLookup = SystemAPI.GetComponentLookup<Player>(false);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        _waypointLookup.Update(ref state);
        _playerLookup.Update(ref state);

        foreach (var (triggerEvent) in SystemAPI.Query<TriggerEvent>())
        {
            // Check which entity has which component
            if (_playerLookup.HasComponent(triggerEvent.EntityA) &&
                _waypointLookup.HasBuffer(triggerEvent.EntityB))
            {
                // Read-write access to player data
                Player player = _playerLookup[triggerEvent.EntityA];
                player.LifeCount -= 1;
                _playerLookup[triggerEvent.EntityA] = player;

                // Read-only access to waypoints
                DynamicBuffer<Waypoints> waypoints = _waypointLookup[triggerEvent.EntityB];
                // ... use waypoints data
            }
        }
    }
}
```

#### Pros
- **Random entity access** - can access any entity's component data by [[Entity]] reference

- **[[Burst]] compatible** - works in Burst-compiled jobs unlike managed lookups

- **Entity relationships** - enables parent-child, target, or other entity reference patterns

- **Thread-safe with [ReadOnly]** - read-only lookups can be used safely in parallel jobs

#### Cons
- **Poor [[Cache-friendly|cache performance]]** - random access causes [[Cache miss]], much slower than sequential iteration

- **Must update before use** - forgetting `lookup.Update(ref state)` causes stale data or crashes

- **Less efficient than queries** - if you can use [[SystemAPI.Query]] or [[IJobEntity]] instead, do so for better performance

- **No safety for write** - read-write lookups can't be used in parallel jobs without careful synchronization

#### Best use
- **Entity references** - following target entity, parent-child relationships, linked entities

- **Physics jobs** - [[ITriggerEventsJob]] provides entity pairs, must use lookups to access their components

- **Cross-archetype relationships** - when entities in different [[Archetype|archetypes]] need to communicate

#### Avoid if
- **Sequential iteration** - if processing all entities with component T, use [[SystemAPI.Query]] or [[IJobEntity]] instead for better performance

- **Same-chunk access** - if accessing components on the same entity being processed, use direct component access instead

- **No entity references** - if you don't have [[Entity]] IDs to look up, you don't need lookups

#### Extra tip
- **Always update** - call `lookup.Update(ref state)` at the start of every `OnUpdate` before scheduling jobs - entity data locations can change between frames

- **Read-only flag** - pass `true` for read-only access (`GetComponentLookup<T>(true)`), `false` for read-write (`GetComponentLookup<T>(false)`)

- **[ReadOnly] attribute** - mark read-only lookups in jobs with `[ReadOnly]` attribute for parallel job safety and documentation

- **HasComponent/HasBuffer checks** - always check `lookup.HasComponent(entity)` before accessing to avoid errors when entity doesn't have the component

- **TryGetComponent pattern** - use `lookup.TryGetComponent(entity, out T component)` for safe access that returns bool instead of throwing

- **TryGetRefRW / TryGetRefRO (Unity Entities 6.5+)** - safer alternative to deprecated `GetRefRWOptional`/`GetRefROOptional`:
  ```csharp
  // Old (deprecated):
  var healthRef = lookup.GetRefRWOptional(entity);
  if (healthRef.IsValid)
      healthRef.ValueRW.Current = 100;

  // New (recommended):
  if (lookup.TryGetRefRW(entity, out RefRW<Health> healthRef))
      healthRef.ValueRW.Current = 100;
  ```
  Combines existence check and retrieval in one call, improving safety and clarity

- **IsComponentEnabled** - for [[IEnableableComponent (toggleable components)|enableable components]], check `lookup.IsComponentEnabled(entity)` to see if component is enabled

- **SetComponentEnabled** - for enableable components, use `lookup.SetComponentEnabled(entity, true/false)` to toggle without [[Structural changes]]

- **Performance consideration** - lookups are ~10-100x slower than sequential iteration due to random memory access, use sparingly
