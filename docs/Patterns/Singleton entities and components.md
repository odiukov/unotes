---
tags:
  - pattern
---
#### Description
- **Single instance pattern** for [[Component|components]] or [[Entity|entities]] that should exist exactly once in the world, accessed globally across systems

- Accessed via `SystemAPI.GetSingleton<T>()` for read-only or `SystemAPI.GetSingletonRW<T>()` for read-write, throws error if zero or multiple instances exist

- **System requires singleton** via `state.RequireForUpdate<T>()` in OnCreate to ensure system only runs when singleton exists

- Common pattern for game state, configuration, physics world, [[EntityCommandBuffer]] systems, and global resources

#### Example
```csharp
// Define a singleton component
public struct GameConfig : IComponentData
{
    public int MaxEnemies;
    public float SpawnRate;
    public int StartingLives;
}

// Create singleton entity in system OnCreate or in SubScene
[BurstCompile]
public partial struct GameInitSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Create singleton entity with component
        Entity configEntity = state.EntityManager.CreateEntity();
        state.EntityManager.AddComponentData(configEntity, new GameConfig
        {
            MaxEnemies = 100,
            SpawnRate = 2.0f,
            StartingLives = 3
        });
    }
}

// Read singleton in other systems
[BurstCompile]
public partial struct EnemySpawnSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Require singleton - system won't run if GameConfig doesn't exist
        state.RequireForUpdate<GameConfig>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Read-only singleton access
        GameConfig config = SystemAPI.GetSingleton<GameConfig>();

        // Use singleton data
        if (GetEnemyCount() < config.MaxEnemies)
        {
            // Spawn enemy using config.SpawnRate
        }
    }
}

// Modify singleton (read-write access)
[BurstCompile]
public partial struct GameStateSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Read-write singleton access
        RefRW<GameConfig> config = SystemAPI.GetSingletonRW<GameConfig>();

        // Modify singleton data
        config.ValueRW.MaxEnemies = 150;
    }
}

// Physics singleton example (Unity-provided)
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
[BurstCompile]
public partial struct RaycastSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<PhysicsWorldSingleton>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Access Unity's physics world singleton
        PhysicsWorldSingleton physicsWorld = SystemAPI.GetSingleton<PhysicsWorldSingleton>();

        RaycastInput input = new RaycastInput
        {
            Start = float3.zero,
            End = new float3(0, -10, 0),
            Filter = CollisionFilter.Default
        };

        if (physicsWorld.CastRay(input, out RaycastHit hit))
        {
            // Process raycast hit
        }
    }
}

// EntityCommandBuffer singleton pattern
[BurstCompile]
public partial struct SpawnerSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<BeginSimulationEntityCommandBufferSystem.Singleton>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Get ECB singleton and create command buffer
        var ecbSingleton = SystemAPI.GetSingleton<BeginSimulationEntityCommandBufferSystem.Singleton>();
        EntityCommandBuffer ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

        // Use ECB...
        Entity newEntity = ecb.Instantiate(prefab);
    }
}

// Singleton entity with buffer
public struct LevelData : IComponentData
{
    public int CurrentLevel;
}

public struct HighScore : IBufferElementData
{
    public int Score;
    public FixedString64Bytes PlayerName;
}

[BurstCompile]
public partial struct ScoreSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Access singleton buffer
        DynamicBuffer<HighScore> highScores = SystemAPI.GetSingletonBuffer<HighScore>();

        // Modify buffer
        highScores.Add(new HighScore { Score = 1000, PlayerName = "Player1" });
    }
}
```

#### Pros
- **Global access** - any system can access singleton without passing references

- **Type-safe** - compile-time checking ensures component type exists

- **Simple API** - `GetSingleton<T>()` is cleaner than searching for entity with component

- **RequireForUpdate integration** - easy to make system dependent on singleton existence

- **[[Burst]] compatible** - works in Burst-compiled systems and jobs (read-only access)

#### Cons
- **Runtime error if missing** - `GetSingleton` throws if singleton doesn't exist or multiple exist

- **Hidden dependencies** - not obvious from system signature what singletons it uses

- **Testing complexity** - tests must ensure singleton exists before running system

- **Can't use in jobs easily** - `GetSingleton` is system-level, jobs need singleton data passed as field

#### Best use
- **Game configuration** - difficulty settings, game rules, global parameters

- **Physics/simulation singletons** - PhysicsWorldSingleton, SimulationSingleton (Unity-provided)

- **Game state** - player score, level info, game phase (menu, playing, paused)

- **ECB systems** - BeginSimulationEntityCommandBufferSystem.Singleton and similar

- **Global resources** - time management, random number generator state, debug settings

#### Avoid if
- **Multiple instances needed** - if you might have multiple instances, use regular components and queries

- **Frequently created/destroyed** - singletons are assumed to be stable, not created/destroyed often

- **Per-entity data** - use regular components on entities, not singletons

#### Extra tip
- **RequireForUpdate pattern** - always call `state.RequireForUpdate<SingletonType>()` in OnCreate to prevent errors

- **Singleton in jobs** - pass singleton data to job as field:
  ```csharp
  var config = SystemAPI.GetSingleton<GameConfig>();
  new MyJob { Config = config }.Schedule();
  ```

- **HasSingleton check** - use `SystemAPI.HasSingleton<T>()` to check existence without throwing

- **TryGetSingleton** - use `SystemAPI.TryGetSingleton<T>(out T singleton)` for safe access that returns bool

- **SetSingleton** - use `SystemAPI.SetSingleton<T>(value)` to update singleton (alternative to GetSingletonRW)

- **Singleton creation in SubScene** - create singleton entity in SubScene for persistent game config

- **Multiple singletons** - can have multiple different singleton component types, each with one instance

- **Singleton entity access** - use `SystemAPI.GetSingletonEntity<T>()` to get the Entity that has the singleton component

- **Buffer singletons** - use `GetSingletonBuffer<T>()` for [[IBufferElementData (dynamic buffers)|DynamicBuffer]] singletons

- **Unity-provided singletons:**
  - `PhysicsWorldSingleton` - physics simulation data
  - `SimulationSingleton` - physics simulation results
  - `BeginInitializationEntityCommandBufferSystem.Singleton` - ECB singleton
  - `BeginSimulationEntityCommandBufferSystem.Singleton` - ECB singleton
  - `EndSimulationEntityCommandBufferSystem.Singleton` - ECB singleton

- **Singleton entity creation patterns:**
  - In system OnCreate (for runtime-created singletons)
  - In SubScene (for config/persistent singletons)
  - In Baker (for baked singleton entities from scene)

- **Error handling** - GetSingleton throws `InvalidOperationException` if 0 or >1 instances exist, use Try variant for safety

- **Enableable singletons** - can use [[IEnableableComponent (toggleable components)|IEnableableComponent]] singletons and toggle with `SystemAPI.SetSingletonEnabled<T>(bool)`

- **GetSingleton in jobs** - can't call GetSingleton inside jobs, must pass data as job field from OnUpdate
