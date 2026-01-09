---
tags:
  - pattern
---
#### Description
- **Component bundling pattern** that provides utility functions to add groups of interdependent [[Component|components]] together in a single call, simplifying entity creation

- Uses `FixedList128Bytes<ComponentType>` to define component sets, then `ComponentTypeSet` for efficient batch addition via `EntityManager.AddComponent(entity, componentSet)`

- Prevents forgetting required supporting components and ensures consistent entity setup across codebase

- Unity's Entities Graphics package uses this pattern extensively - `RenderMeshUtility.AddComponents()` adds all rendering-related components in one call

#### Example
```csharp
// Example from Unity's Entities Graphics package (simplified)
public static class RenderMeshUtility
{
    // Helper method to add all components needed for rendering
    public static void AddComponents(
        Entity entity,
        EntityManager entityManager,
        RenderMeshDescription renderMeshDescription)
    {
        // Build component type list
        var componentTypes = new FixedList128Bytes<ComponentType>();

        // Core rendering components
        componentTypes.Add(ComponentType.ReadWrite<WorldRenderBounds>());
        componentTypes.Add(ComponentType.ReadWrite<LocalToWorld>());
        componentTypes.Add(ComponentType.ReadWrite<MaterialMeshInfo>());

        // Optional components based on description
        if (renderMeshDescription.FilterSettings.ShadowCastingMode != ShadowCastingMode.Off)
        {
            componentTypes.Add(ComponentType.ReadWrite<CastShadows>());
        }

        if (renderMeshDescription.LightProbeUsage != LightProbeUsage.Off)
        {
            componentTypes.Add(ComponentType.ReadWrite<LightProbes>());
        }

        // Convert to ComponentTypeSet and add all at once
        var componentSet = new ComponentTypeSet(componentTypes);
        entityManager.AddComponent(entity, componentSet);

        // Set initial values
        entityManager.SetComponentData(entity, new MaterialMeshInfo
        {
            Mesh = renderMeshDescription.Mesh,
            Material = renderMeshDescription.Material
        });
    }
}

// Usage
Entity renderEntity = entityManager.CreateEntity();
RenderMeshUtility.AddComponents(
    renderEntity,
    entityManager,
    new RenderMeshDescription
    {
        Mesh = myMesh,
        Material = myMaterial,
        ShadowCastingMode = ShadowCastingMode.On
    });

// Example: Character component helper
public static class CharacterUtility
{
    public static void AddCharacterComponents(
        Entity entity,
        EntityManager entityManager,
        bool isPlayer = false)
    {
        var componentTypes = new FixedList128Bytes<ComponentType>
        {
            ComponentType.ReadWrite<Health>(),
            ComponentType.ReadWrite<CharacterController>(),
            ComponentType.ReadWrite<MovementSpeed>(),
            ComponentType.ReadWrite<LocalTransform>(),
        };

        // Player-specific components
        if (isPlayer)
        {
            componentTypes.Add(ComponentType.ReadWrite<PlayerInput>());
            componentTypes.Add(ComponentType.ReadWrite<PlayerInventory>());
            componentTypes.Add(ComponentType.ReadWrite<CameraTarget>());
        }
        else
        {
            componentTypes.Add(ComponentType.ReadWrite<AIController>());
            componentTypes.Add(ComponentType.ReadWrite<AIState>());
        }

        var componentSet = new ComponentTypeSet(componentTypes);
        entityManager.AddComponent(entity, componentSet);

        // Set default values
        entityManager.SetComponentData(entity, new Health
        {
            Current = 100,
            Max = 100
        });

        entityManager.SetComponentData(entity, new MovementSpeed
        {
            Value = isPlayer ? 5f : 3f
        });
    }
}

// Example: Physics entity helper
public static class PhysicsEntityUtility
{
    public static void AddPhysicsComponents(
        Entity entity,
        EntityManager entityManager,
        bool isDynamic,
        bool needsCollisionEvents = false)
    {
        var componentTypes = new FixedList128Bytes<ComponentType>
        {
            ComponentType.ReadWrite<PhysicsCollider>(),
            ComponentType.ReadWrite<PhysicsWorldIndex>(),
        };

        if (isDynamic)
        {
            componentTypes.Add(ComponentType.ReadWrite<PhysicsVelocity>());
            componentTypes.Add(ComponentType.ReadWrite<PhysicsMass>());
            componentTypes.Add(ComponentType.ReadWrite<PhysicsDamping>());
        }

        if (needsCollisionEvents)
        {
            componentTypes.Add(ComponentType.ReadWrite<CollisionEventBuffer>());
        }

        var componentSet = new ComponentTypeSet(componentTypes);
        entityManager.AddComponent(entity, componentSet);
    }
}

// Example: Baker usage
public class CharacterAuthoring : MonoBehaviour
{
    public bool IsPlayer;
}

public class CharacterBaker : Baker<CharacterAuthoring>
{
    public override void Bake(CharacterAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);

        // Use helper to add all required components
        CharacterUtility.AddCharacterComponents(
            entity,
            EntityManager,
            authoring.IsPlayer);
    }
}

// Example: Networked entity helper
public static class NetworkEntityUtility
{
    public static void AddNetworkComponents(
        Entity entity,
        EntityManager entityManager,
        bool isServer)
    {
        var componentTypes = new FixedList128Bytes<ComponentType>
        {
            ComponentType.ReadWrite<NetworkId>(),
            ComponentType.ReadWrite<NetworkTransform>(),
            ComponentType.ReadWrite<ReplicatedEntity>(),
        };

        if (isServer)
        {
            componentTypes.Add(ComponentType.ReadWrite<ServerAuthority>());
            componentTypes.Add(ComponentType.ReadWrite<NetworkSpawnRequest>());
        }
        else
        {
            componentTypes.Add(ComponentType.ReadWrite<ClientPrediction>());
            componentTypes.Add(ComponentType.ReadWrite<ServerReconciliation>());
        }

        var componentSet = new ComponentTypeSet(componentTypes);
        entityManager.AddComponent(entity, componentSet);
    }
}

// Example: Weapon entity helper with configurable features
public enum WeaponFeatures
{
    None = 0,
    Recoil = 1 << 0,
    Reload = 1 << 1,
    Overheat = 1 << 2,
    Projectile = 1 << 3,
}

public static class WeaponUtility
{
    public static void AddWeaponComponents(
        Entity entity,
        EntityManager entityManager,
        WeaponFeatures features)
    {
        var componentTypes = new FixedList128Bytes<ComponentType>
        {
            ComponentType.ReadWrite<Weapon>(),
            ComponentType.ReadWrite<Damage>(),
            ComponentType.ReadWrite<FireRate>(),
        };

        if ((features & WeaponFeatures.Recoil) != 0)
        {
            componentTypes.Add(ComponentType.ReadWrite<RecoilData>());
        }

        if ((features & WeaponFeatures.Reload) != 0)
        {
            componentTypes.Add(ComponentType.ReadWrite<AmmoCount>());
            componentTypes.Add(ComponentType.ReadWrite<ReloadTime>());
        }

        if ((features & WeaponFeatures.Overheat) != 0)
        {
            componentTypes.Add(ComponentType.ReadWrite<HeatLevel>());
            componentTypes.Add(ComponentType.ReadWrite<CooldownRate>());
        }

        if ((features & WeaponFeatures.Projectile) != 0)
        {
            componentTypes.Add(ComponentType.ReadWrite<ProjectilePrefab>());
        }

        var componentSet = new ComponentTypeSet(componentTypes);
        entityManager.AddComponent(entity, componentSet);
    }
}
```

#### Pros
- **Prevents forgotten components** - ensures all required components are added together, reducing bugs

- **Consistent entity setup** - centralizes entity creation logic, making codebase more maintainable

- **Conditional component addition** - easily add optional components based on configuration flags

- **Single [[Structural changes|structural change]]** - adding multiple components in one call is more efficient than multiple individual adds

- **Self-documenting** - helper methods clearly show what components are needed for specific entity types

#### Cons
- **Extra indirection** - adds abstraction layer between component usage and addition

- **Can be overused** - not every component combination needs a helper, can lead to helper explosion

- **Harder to discover** - developers need to know helpers exist, otherwise they might add components manually

- **Update maintenance** - when component requirements change, must update helper methods

#### Best use
- **Complex entity types** - rendering entities, physics entities, characters that need many interdependent components

- **Framework/package APIs** - when building reusable systems that other developers will use (like Unity's Entities Graphics)

- **[[Baking]] utilities** - simplify baker code by bundling common component additions

- **Entity archetypes** - when you have well-defined entity types (enemy, player, bullet) with specific component sets

- **Conditional features** - when entities need different component sets based on configuration (player vs AI, static vs dynamic)

#### Avoid if
- **Simple entities** - entities with 1-2 components don't need helpers, direct addition is clearer

- **Unique combinations** - if component set is only used once, helper adds unnecessary abstraction

- **Frequently changing requirements** - if component needs change often, helper becomes maintenance burden

- **Performance-critical paths** - in tight loops, direct component addition might be slightly faster (though difference is minimal)

#### Extra tip
- **FixedList128Bytes capacity** - can hold ~20-30 ComponentType entries depending on platform, use FixedList512Bytes for larger sets

- **ComponentTypeSet efficiency** - Unity optimizes batch component addition internally, significantly faster than individual adds

- **Combine with [[Archetype]] patterns** - can create EntityArchetype from ComponentTypeSet for even faster entity creation:
  ```csharp
  var archetype = entityManager.CreateArchetype(componentSet);
  Entity entity = entityManager.CreateEntity(archetype);
  ```

- **Extension methods pattern** - can make helpers more discoverable:
  ```csharp
  public static class EntityManagerExtensions
  {
      public static void AddCharacterComponents(this EntityManager em, Entity entity, bool isPlayer)
      {
          CharacterUtility.AddCharacterComponents(entity, em, isPlayer);
      }
  }
  // Usage: entityManager.AddCharacterComponents(entity, true);
  ```

- **Builder pattern alternative** - for complex entities, consider fluent builder:
  ```csharp
  new EntityBuilder(entityManager, entity)
      .WithHealth(100)
      .WithMovement(5f)
      .AsPlayer()
      .Build();
  ```

- **Documentation** - document helper methods with XML comments explaining what components are added and why

- **Unity's official examples:**
  - `RenderMeshUtility.AddComponents()` - Entities Graphics
  - `PhysicsUtility.AddComponents()` - Unity Physics (custom)
  - Check Unity's official packages for more examples

- **Static class organization** - group helpers by domain (PhysicsUtility, RenderUtility, CharacterUtility) for better code organization

- **Validation in helpers** - add runtime checks to validate component compatibility:
  ```csharp
  if (isDynamic && !hasPhysicsCollider)
  {
      throw new ArgumentException("Dynamic physics entities require PhysicsCollider");
  }
  ```

- **Default values** - set sensible defaults after adding components to ensure entities are immediately usable

- **Naming conventions:**
  - `Add[Type]Components` - for adding component sets
  - `Create[Type]Entity` - for creating entity with components
  - `Configure[Type]` - for setting up existing entity

- **Performance tip** - create ComponentTypeSet once and reuse if adding same components to many entities:
  ```csharp
  private static readonly ComponentTypeSet CharacterComponents = new ComponentTypeSet(
      new FixedList128Bytes<ComponentType> { /* ... */ });
  ```

- **Combining with Auto-Add System Pattern** - helpers are for initial entity creation, Auto-Add Systems ensure supporting components persist if manually removed

- **Debugging** - use Entity Debugger to verify all expected components were added by helper

- **Testing helpers** - write unit tests to verify helpers add expected components and set correct initial values
