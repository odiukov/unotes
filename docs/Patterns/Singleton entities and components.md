---
tags:
  - pattern
---
#### Description
- **Single instance pattern** for [[Component|components]] or [[Entity|entities]] existing exactly once in world, accessed globally across systems

- Accessed via `SystemAPI.GetSingleton<T>()` (read-only) or `SystemAPI.GetSingletonRW<T>()` (read-write), throws error if zero or multiple instances

- Systems use `state.RequireForUpdate<T>()` in OnCreate to ensure system only runs when singleton exists

- Common for game state, configuration, physics world, [[EntityCommandBuffer]] systems, and global resources

#### Example
```csharp
// Define singleton component
public struct GameConfig : IComponentData
{
    public int MaxEnemies;
    public float SpawnRate;
}

// Create singleton entity
[BurstCompile]
public partial struct GameInitSystem : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        Entity configEntity = state.EntityManager.CreateEntity();
        state.EntityManager.AddComponentData(configEntity, new GameConfig
        {
            MaxEnemies = 100,
            SpawnRate = 2.0f
        });
    }
}

// Read singleton
[BurstCompile]
public partial struct EnemySpawnSystem : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<GameConfig>();  // System won't run without singleton
    }

    public void OnUpdate(ref SystemState state)
    {
        GameConfig config = SystemAPI.GetSingleton<GameConfig>();  // Read-only
        // Use config.MaxEnemies...
    }
}

// Modify singleton
[BurstCompile]
public partial struct GameStateSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        RefRW<GameConfig> config = SystemAPI.GetSingletonRW<GameConfig>();  // Read-write
        config.ValueRW.MaxEnemies = 150;
    }
}

// EntityCommandBuffer singleton pattern
var ecb = SystemAPI.GetSingleton<BeginSimulationEntityCommandBufferSystem.Singleton>()
    .CreateCommandBuffer(state.WorldUnmanaged);
```

**Common singletons:**
- Game state (score, level, time)
- Configuration (difficulty, settings)
- Physics world (`PhysicsWorldSingleton`)
- ECB systems (`BeginSimulationEntityCommandBufferSystem.Singleton`)
- Input state
- Random seed

#### Pros
- **Global access** - any system can access singleton data without passing references

- **Type-safe** - compile-time error if component type wrong

- **Performance** - direct component access, no lookup overhead

- **Clear intent** - `GetSingleton` clearly communicates single instance requirement

- **Automatic validation** - runtime error if multiple instances exist

#### Cons
- **Runtime error** - throws if zero or multiple instances, no compile-time check

- **Hidden dependencies** - systems using singletons have implicit dependencies

- **Testing complexity** - tests must create singleton entities

- **Shared mutable state** - multiple systems modifying singleton can cause race conditions

- **Not [[Job]] compatible** - can't access singletons directly in jobs, must capture in job struct

#### Best use
- **Game configuration** - settings, difficulty, constants that rarely change

- **Game state** - score, level, time, global counters

- **Physics/input** - Unity-provided singletons for physics world, input

- **ECB systems** - accessing EntityCommandBuffer system singletons

- **Random seed** - single random seed for deterministic gameplay

#### Avoid if
- **Multiple instances** - if might need multiple, use regular components

- **Per-entity data** - if data belongs to specific entity, not global

- **Frequent creation/destruction** - singleton should exist for entire session

- **Job parallelism** - if need to access in parallel jobs, use ComponentLookup instead

#### Extra tip
- **RequireForUpdate**: Always use to prevent null reference when singleton doesn't exist yet

- **Singleton entity**: Use `EntityQuery.GetSingletonEntity()` to get entity holding singleton component

- **ECB pattern**: Access ECB via singleton for consistent timing:
  ```csharp
  var ecb = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
      .CreateCommandBuffer(state.WorldUnmanaged);
  ```

- **HasSingleton**: Check existence with `SystemAPI.HasSingleton<T>()` before accessing

- **Multiple worlds**: Each World has separate singletons - useful for client/server separation

- **Burst compatibility**: `GetSingleton<T>()` works in Burst, but ECB singleton requires `.CreateCommandBuffer()`

- **SubScene creation**: Can create singleton in SubScene for designer-friendly configuration

- **Naming convention**: Suffix singleton components with `Config`, `State`, or `Singleton` for clarity

- **Testing**: Create singleton in test setup:
  ```csharp
  [SetUp]
  public void Setup()
  {
      var entity = entityManager.CreateEntity();
      entityManager.AddComponentData(entity, new GameConfig { /* ... */ });
  }
  ```

- **Alternative - static**: For truly global data, static variables simpler than singletons
