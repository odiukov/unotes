---
tags:
  - job
  - physics
---
#### Description
- **Physics-specific job** for processing trigger collision events from Unity Physics, executing once per trigger pair with [[Burst]] compilation

- Scheduled with `.Schedule(SimulationSingleton, state.Dependency)` after physics simulation completes

- Provides access to `TriggerEvent` struct containing the pair of entities involved in the collision and collision details

- Typically runs in `[UpdateInGroup(typeof(AfterPhysicsSystemGroup))]` to process results after physics step

#### Example
```csharp
[UpdateInGroup(typeof(AfterPhysicsSystemGroup))]
[BurstCompile]
public partial struct EnemyPlayerCollisionSystem : ISystem
{
    BufferLookup<Waypoints> _enemyLookup;
    ComponentLookup<Player> _playerLookup;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<SimulationSingleton>();

        // Create lookups for component access
        _enemyLookup = SystemAPI.GetBufferLookup<Waypoints>(true);
        _playerLookup = SystemAPI.GetComponentLookup<Player>(false);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Get physics simulation results
        SimulationSingleton simulation = SystemAPI.GetSingleton<SimulationSingleton>();

        // Update lookups before use
        _enemyLookup.Update(ref state);
        _playerLookup.Update(ref state);

        // Schedule trigger events job
        state.Dependency = new EnemyPlayerCollisionJob()
        {
            Enemies = _enemyLookup,
            Players = _playerLookup,
            Ecb = /* EntityCommandBuffer */
        }.Schedule(simulation, state.Dependency);
    }

    [BurstCompile]
    private struct EnemyPlayerCollisionJob : ITriggerEventsJob
    {
        [ReadOnly] public BufferLookup<Waypoints> Enemies;
        public ComponentLookup<Player> Players;
        public EntityCommandBuffer Ecb;

        // Executes once per trigger event
        public void Execute(TriggerEvent triggerEvent)
        {
            // Identify which entity is which
            Entity playerEntity = Entity.Null;
            Entity enemyEntity = Entity.Null;

            if (Players.HasComponent(triggerEvent.EntityA))
                playerEntity = triggerEvent.EntityA;
            if (Players.HasComponent(triggerEvent.EntityB))
                playerEntity = triggerEvent.EntityB;
            if (Enemies.HasBuffer(triggerEvent.EntityA))
                enemyEntity = triggerEvent.EntityA;
            if (Enemies.HasBuffer(triggerEvent.EntityB))
                enemyEntity = triggerEvent.EntityB;

            if (playerEntity == Entity.Null || enemyEntity == Entity.Null)
                return;

            // Process collision - damage player, destroy enemy
            Player player = Players[playerEntity];
            player.LifeCount -= 1;
            Players[playerEntity] = player;

            Ecb.DestroyEntity(enemyEntity);
        }
    }
}
```

#### Pros
- **[[Burst]] compatible** - full performance benefits for physics event processing

- **Automatic event iteration** - Unity calls Execute for each trigger pair, no manual iteration needed

- **Collision filtering** - only processes actual trigger events based on CollisionFilter setup

- **Entity pair access** - provides both entities involved in collision via `TriggerEvent.EntityA` and `EntityB`

#### Cons
- **Requires [[ComponentLookup and BufferLookup|ComponentLookup/BufferLookup]]** - can't use direct component access, must use lookups for random access

- **Entity identification required** - must determine which entity is which from the pair using lookups

- **Physics-only** - only works with Unity Physics trigger colliders, not useful for other collision detection

- **No chunk iteration** - processes one event at a time, can't batch process events by chunk

#### Best use
- **Trigger-based gameplay** - projectile hits, player-enemy collisions, pickup collection

- **Event-driven damage** - apply damage when entities collide based on physics triggers

- **Area detection** - detecting when entities enter/exit zones or trigger volumes

#### Avoid if
- **You need collision contacts** - for detailed collision contact info, use `ICollisionEventsJob` instead

- **Non-physics collision detection** - for spatial queries without physics simulation, use [[PhysicsWorldSingleton]] raycasts or overlap queries

- **Simple trigger checks** - if you only need to check if entities are nearby, spatial queries in a regular system may be simpler

#### Extra tip
- **Update lookups** - always call `lookup.Update(ref state)` before scheduling the job, as entity storage may have been reorganized

- **Entity identification pattern** - use `HasComponent()`/`HasBuffer()` on both EntityA and EntityB to determine which entity has which component type

- **Multiple EntityCommandBuffers** - common pattern: use BeginInitializationECB for adding components, BeginSimulationECB for destroying entities to ensure proper execution order

- **CollisionFilter setup** - configure `CollisionFilter.BelongsTo` and `CollidesWith` during [[Baker<T>|baking]] to control which entities can trigger with each other

- **SystemGroup ordering** - use `[UpdateInGroup(typeof(AfterPhysicsSystemGroup))]` to ensure physics simulation has completed before processing events
