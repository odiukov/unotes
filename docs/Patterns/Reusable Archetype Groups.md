---
tags:
  - pattern
---
#### Description
- **[[Archetype]] caching pattern** that defines reusable component combinations in a struct, allowing multiple systems to reference the same archetype definitions without duplication

- Typically implemented as static helper methods that return archetype collections or `ComponentTypeSet`, centralizing archetype definitions in one location

- Useful when multiple systems need to create entities with the same component combinations or query similar archetypes

- Unity's `Unity.Scenes` package uses this pattern with `CreateResolveSceneSectionArchetypes()` to share archetype definitions across scene loading systems

#### Example
```csharp
// Example from Unity.Scenes (simplified concept)
public struct SceneArchetypes
{
    public EntityArchetype ResolvedSectionEntity;
    public EntityArchetype RequestSceneLoaded;
    public EntityArchetype SceneMetaData;

    public static SceneArchetypes Create(EntityManager entityManager)
    {
        return new SceneArchetypes
        {
            // Define archetype for resolved scene section
            ResolvedSectionEntity = entityManager.CreateArchetype(
                ComponentType.ReadWrite<SceneSectionData>(),
                ComponentType.ReadWrite<ResolvedSectionEntity>(),
                ComponentType.ReadWrite<LinkedEntityGroup>()
            ),

            // Define archetype for scene load request
            RequestSceneLoaded = entityManager.CreateArchetype(
                ComponentType.ReadWrite<RequestSceneLoaded>(),
                ComponentType.ReadWrite<SceneReference>()
            ),

            // Define archetype for scene metadata
            SceneMetaData = entityManager.CreateArchetype(
                ComponentType.ReadWrite<SceneMetaData>(),
                ComponentType.ReadWrite<ResolvedSectionEntity>()
            )
        };
    }
}

// Usage in systems
[BurstCompile]
public partial struct SceneLoadingSystem : ISystem
{
    private SceneArchetypes _archetypes;

    public void OnCreate(ref SystemState state)
    {
        // Create archetypes once
        _archetypes = SceneArchetypes.Create(state.EntityManager);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Reuse archetypes for entity creation
        Entity sceneEntity = state.EntityManager.CreateEntity(_archetypes.RequestSceneLoaded);
        // Set component data...
    }
}

// Example: Character archetypes
public struct CharacterArchetypes
{
    public EntityArchetype Player;
    public EntityArchetype Enemy;
    public EntityArchetype NPC;

    public static CharacterArchetypes Create(EntityManager em)
    {
        return new CharacterArchetypes
        {
            Player = em.CreateArchetype(
                ComponentType.ReadWrite<Health>(),
                ComponentType.ReadWrite<CharacterController>(),
                ComponentType.ReadWrite<PlayerInput>(),
                ComponentType.ReadWrite<Inventory>(),
                ComponentType.ReadWrite<LocalTransform>()
            ),

            Enemy = em.CreateArchetype(
                ComponentType.ReadWrite<Health>(),
                ComponentType.ReadWrite<CharacterController>(),
                ComponentType.ReadWrite<AIController>(),
                ComponentType.ReadWrite<LocalTransform>(),
                ComponentType.ReadWrite<Target>()
            ),

            NPC = em.CreateArchetype(
                ComponentType.ReadWrite<CharacterController>(),
                ComponentType.ReadWrite<DialogueData>(),
                ComponentType.ReadWrite<LocalTransform>()
            )
        };
    }
}

// Multiple systems using same archetypes
[BurstCompile]
public partial struct PlayerSpawnSystem : ISystem
{
    private CharacterArchetypes _archetypes;

    public void OnCreate(ref SystemState state)
    {
        _archetypes = CharacterArchetypes.Create(state.EntityManager);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Use player archetype
        Entity player = state.EntityManager.CreateEntity(_archetypes.Player);
        state.EntityManager.SetComponentData(player, new Health { Current = 100, Max = 100 });
        // ...
    }
}

[BurstCompile]
public partial struct EnemySpawnSystem : ISystem
{
    private CharacterArchetypes _archetypes;

    public void OnCreate(ref SystemState state)
    {
        _archetypes = CharacterArchetypes.Create(state.EntityManager);
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Use enemy archetype
        Entity enemy = state.EntityManager.CreateEntity(_archetypes.Enemy);
        state.EntityManager.SetComponentData(enemy, new Health { Current = 50, Max = 50 });
        // ...
    }
}

// Example: Projectile archetypes with variants
public struct ProjectileArchetypes
{
    public EntityArchetype SimpleBullet;
    public EntityArchetype Rocket;
    public EntityArchetype LaserBeam;

    public static ProjectileArchetypes Create(EntityManager em)
    {
        // Shared components
        var baseComponents = new FixedList128Bytes<ComponentType>
        {
            ComponentType.ReadWrite<LocalTransform>(),
            ComponentType.ReadWrite<Velocity>(),
            ComponentType.ReadWrite<Lifetime>(),
            ComponentType.ReadWrite<Damage>()
        };

        return new ProjectileArchetypes
        {
            SimpleBullet = em.CreateArchetype(baseComponents),

            Rocket = em.CreateArchetype(
                baseComponents,
                ComponentType.ReadWrite<HomingTarget>(),
                ComponentType.ReadWrite<ExplosionRadius>()
            ),

            LaserBeam = em.CreateArchetype(
                baseComponents,
                ComponentType.ReadWrite<BeamRenderer>(),
                ComponentType.ReadWrite<ContinuousDamage>()
            )
        };
    }
}

// Example: Pooling archetypes
public struct PooledArchetypes
{
    public EntityArchetype PooledEntity;
    public EntityArchetype ActiveEntity;

    public static PooledArchetypes Create(EntityManager em)
    {
        return new PooledArchetypes
        {
            // Pooled entities have Disabled tag
            PooledEntity = em.CreateArchetype(
                ComponentType.ReadWrite<Pooled>(),
                ComponentType.ReadWrite<Disabled>()
            ),

            // Active entities without Disabled
            ActiveEntity = em.CreateArchetype(
                ComponentType.ReadWrite<Pooled>()
            )
        };
    }
}

// Example: ComponentTypeSet approach
public static class ComponentSets
{
    // Define reusable component sets
    public static ComponentTypeSet PhysicsSet => new ComponentTypeSet(
        ComponentType.ReadWrite<PhysicsCollider>(),
        ComponentType.ReadWrite<PhysicsVelocity>(),
        ComponentType.ReadWrite<PhysicsMass>()
    );

    public static ComponentTypeSet RenderSet => new ComponentTypeSet(
        ComponentType.ReadWrite<RenderMesh>(),
        ComponentType.ReadWrite<WorldRenderBounds>(),
        ComponentType.ReadWrite<MaterialMeshInfo>()
    );

    public static ComponentTypeSet TransformSet => new ComponentTypeSet(
        ComponentType.ReadWrite<LocalTransform>(),
        ComponentType.ReadWrite<LocalToWorld>()
    );
}

// Usage with ComponentTypeSet
[BurstCompile]
public partial struct EntityFactorySystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        Entity entity = ecb.CreateEntity();
        ecb.AddComponent(entity, ComponentSets.PhysicsSet);
        ecb.AddComponent(entity, ComponentSets.RenderSet);
        ecb.AddComponent(entity, ComponentSets.TransformSet);

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: World-specific archetypes
public struct SimulationArchetypes
{
    public EntityArchetype Particle;
    public EntityArchetype Constraint;
    public EntityArchetype ForceField;

    public static SimulationArchetypes CreateForWorld(World world)
    {
        var em = world.EntityManager;

        return new SimulationArchetypes
        {
            Particle = em.CreateArchetype(
                ComponentType.ReadWrite<Position>(),
                ComponentType.ReadWrite<Velocity>(),
                ComponentType.ReadWrite<Mass>()
            ),

            Constraint = em.CreateArchetype(
                ComponentType.ReadWrite<ConstraintData>(),
                ComponentType.ReadWrite<ConnectedEntities>()
            ),

            ForceField = em.CreateArchetype(
                ComponentType.ReadWrite<ForceFieldData>(),
                ComponentType.ReadWrite<WorldRenderBounds>()
            )
        };
    }
}
```

#### Pros
- **Eliminates duplication** - archetype definitions shared across systems, reducing copy-paste errors

- **Centralized definitions** - all archetypes for a feature/domain defined in one place, easier to maintain

- **Type safety** - struct members provide compile-time checking of archetype names

- **Performance** - archetype creation happens once in OnCreate, not every frame

- **Consistency** - ensures all systems use identical component combinations for same entity types

#### Cons
- **Extra boilerplate** - requires defining struct and Create method, more code than inline archetype creation

- **Less flexible** - harder to create one-off archetypes, encourages creating more reusable types

- **Memory overhead** - struct instance stored per system, though archetypes themselves are shared

- **Can be overused** - not every archetype needs to be in shared struct, simple one-off archetypes are fine inline

#### Best use
- **Multiple systems creating same entities** - when 3+ systems create entities with identical component sets

- **Complex features** - scene loading, physics, networking where many archetypes are related

- **Framework/package development** - when building reusable systems for other developers

- **Entity variants** - when you have multiple variants of similar entities (player/enemy/NPC, different projectile types)

- **Large projects** - projects with dozens of entity types benefit from centralized archetype management

#### Avoid if
- **Simple projects** - small projects with few entity types don't need this organization

- **One-off archetypes** - archetypes used in single system are fine to define inline

- **Frequently changing** - if entity structure changes often, maintaining centralized definitions is overhead

- **Baking entities** - entities created during [[Baking]] don't benefit from runtime archetype caching

#### Extra tip
- **Naming conventions** - suffix struct with `Archetypes` or prefix with domain (`SceneArchetypes`, `CharacterArchetypes`)

- **Static Create method** - use static factory method pattern for clean initialization:
  ```csharp
  _archetypes = CharacterArchetypes.Create(state.EntityManager);
  ```

- **Cache in system OnCreate** - create archetypes once in `OnCreate`, store in system field:
  ```csharp
  private CharacterArchetypes _archetypes;
  public void OnCreate(ref SystemState state)
  {
      _archetypes = CharacterArchetypes.Create(state.EntityManager);
  }
  ```

- **Per-World archetypes** - archetypes are World-specific, can't share across Worlds:
  ```csharp
  public static SimulationArchetypes CreateForWorld(World world)
  ```

- **Shared components** - can use `FixedList` to build base component sets:
  ```csharp
  var baseComponents = new FixedList128Bytes<ComponentType> { /* ... */ };
  var archetype = em.CreateArchetype(baseComponents);
  ```

- **Archetype extensions** - can extend archetypes by adding more components:
  ```csharp
  // Base archetype
  var baseArchetype = em.CreateArchetype(baseComponents);

  // Extended archetype with additional components
  var extendedArchetype = em.CreateArchetype(
      baseComponents,
      ComponentType.ReadWrite<ExtraComponent>()
  );
  ```

- **Unity.Scenes example** - check `Unity.Scenes` package source for `CreateResolveSceneSectionArchetypes()` implementation

- **Documentation** - document what each archetype is for and what systems use it

- **ComponentTypeSet alternative** - for adding to existing entities, ComponentTypeSet may be more appropriate than archetypes:
  ```csharp
  public static ComponentTypeSet PhysicsComponents => new ComponentTypeSet(/* ... */);
  ```

- **Prefab alternative** - for static entity types, consider using entity prefabs instead of archetypes:
  ```csharp
  Entity instance = em.Instantiate(prefabEntity);
  ```

- **Combining patterns** - combine with [[Helper Methods for Component Sets]] for flexible entity creation:
  ```csharp
  // Archetype for base structure
  Entity entity = em.CreateEntity(_archetypes.Character);

  // Helper for conditional components
  CharacterUtility.AddOptionalComponents(entity, em, isPlayer: true);
  ```

- **System groups** - archetypes used by systems in same group can be defined in group-level struct

- **Editor workflow** - can expose archetype struct in inspector for debugging/validation

- **Testing** - tests can create archetype struct to ensure consistent entity creation

- **Migration friendly** - when refactoring entity structure, only need to update archetype definition, not all usage sites

- **Performance consideration** - archetype creation is fast but not free, create in OnCreate not OnUpdate

- **Burst compatibility** - archetype structs work in Burst-compiled code when created in OnCreate

- **Best practices:**
  - One archetype struct per feature/domain
  - Name archetypes descriptively
  - Document component purpose
  - Keep Create method focused on archetype creation only
  - Store in system field, not as singleton component

- **Alternative pattern - Singleton archetype storage** - can store archetypes in singleton component for cross-system access:
  ```csharp
  public struct CharacterArchetypesSingleton : IComponentData
  {
      public EntityArchetype Player;
      public EntityArchetype Enemy;
  }
  ```
