---
tags:
  - pattern
---
#### Description
- **Component bundling pattern** providing utility functions to add groups of interdependent [[Component|components]] in single call

- Uses `FixedList128Bytes<ComponentType>` to define component sets, then `ComponentTypeSet` for efficient batch addition via `EntityManager.AddComponent(entity, componentSet)`

- Prevents forgetting required supporting components and ensures consistent entity setup across codebase

- Unity's Entities Graphics uses this - `RenderMeshUtility.AddComponents()` adds all rendering-related components in one call

#### Example
```csharp
// Helper class with utility methods
public static class CharacterUtility
{
    public static void AddCharacterComponents(
        Entity entity,
        EntityManager em,
        bool isPlayer = false)
    {
        // Build component list
        var components = new FixedList128Bytes<ComponentType>
        {
            ComponentType.ReadWrite<Health>(),
            ComponentType.ReadWrite<CharacterController>(),
            ComponentType.ReadWrite<MovementSpeed>(),
            ComponentType.ReadWrite<LocalTransform>(),
        };

        // Conditional components
        if (isPlayer)
        {
            components.Add(ComponentType.ReadWrite<PlayerInput>());
            components.Add(ComponentType.ReadWrite<Inventory>());
        }
        else
        {
            components.Add(ComponentType.ReadWrite<AIController>());
        }

        // Add all at once
        var componentSet = new ComponentTypeSet(components);
        em.AddComponent(entity, componentSet);

        // Set initial values
        em.SetComponentData(entity, new Health { Current = 100, Max = 100 });
        em.SetComponentData(entity, new MovementSpeed { Value = 5f });
    }
}

// Usage
Entity player = entityManager.CreateEntity();
CharacterUtility.AddCharacterComponents(player, entityManager, isPlayer: true);

// Unity.Entities.Graphics example
public static class RenderMeshUtility
{
    public static void AddComponents(
        Entity entity,
        EntityManager em,
        RenderMeshDescription desc)
    {
        var components = new FixedList128Bytes<ComponentType>
        {
            ComponentType.ReadWrite<WorldRenderBounds>(),
            ComponentType.ReadWrite<LocalToWorld>(),
            ComponentType.ReadWrite<MaterialMeshInfo>(),
        };

        // Conditional rendering features
        if (desc.ShadowCastingMode != ShadowCastingMode.Off)
            components.Add(ComponentType.ReadWrite<CastShadows>());

        if (desc.LightProbeUsage != LightProbeUsage.Off)
            components.Add(ComponentType.ReadWrite<LightProbes>());

        em.AddComponent(entity, new ComponentTypeSet(components));
    }
}
```

**Pattern structure:**
1. Static utility class with helper methods
2. Method takes entity + parameters
3. Build `FixedList128Bytes<ComponentType>`
4. Convert to `ComponentTypeSet`
5. Call `AddComponent(entity, componentSet)`
6. Set initial component values

#### Pros
- **Prevents errors** - guarantees all required components added together

- **Consistent setup** - all entities created with same helper get same components

- **Centralized logic** - change component requirements in one place

- **Performance** - batch add faster than multiple individual AddComponent calls

- **Conditional logic** - can add different components based on parameters

#### Cons
- **Not type-safe** - using `ComponentType.ReadWrite<T>()` loses compile-time type checking

- **Initialization separate** - must set initial values separately from adding components

- **Runtime overhead** - building FixedList has small cost vs compile-time archetype

- **Limited flexibility** - harder to vary component combinations than with archetypes

#### Best use
- **Complex entity types** - entities requiring many interdependent components (rendering, physics, AI)

- **Conditional composition** - when component set varies based on parameters (player vs enemy)

- **Framework APIs** - packages providing entity creation utilities (Unity.Entities.Graphics)

- **Consistency critical** - when forgetting a component causes bugs

#### Avoid if
- **Simple entities** - 1-2 components easier to add directly

- **Fixed composition** - if components never vary, use [[Reusable Archetype Groups]] instead

- **Performance critical** - creating FixedList adds overhead vs direct archetype creation

- **Type safety required** - pattern loses compile-time type checking

#### Extra tip
- **Unity.Entities.Graphics example**: `RenderMeshUtility.AddComponents()` is production example

- **Combine with archetypes**: Can use helper method to add components to entity created from archetype

- **FixedList capacity**: `FixedList128Bytes` holds ~16 ComponentTypes, use `FixedList512Bytes` for more

- **ECB support**: Can use ComponentTypeSet with [[EntityCommandBuffer]]:
  ```csharp
  var components = new ComponentTypeSet(/* ... */);
  ecb.AddComponent(entity, components);
  ```

- **Naming convention**: Suffix utility class with `Utility` (CharacterUtility, RenderUtility)

- **Initialization pattern**: Set component values immediately after adding:
  ```csharp
  em.AddComponent(entity, componentSet);
  em.SetComponentData(entity, new Health { Max = 100 });
  ```

- **Alternative - extension methods**: Can implement as extension methods on EntityManager:
  ```csharp
  public static void AddCharacterComponents(this EntityManager em, Entity entity) { }
  ```

- **Combining patterns**: Use with [[Auto-Add System Pattern]] - helper adds base components, system auto-adds supporting ones

- **Testing**: Helper methods make tests cleaner by encapsulating entity creation

- **Best practices**: Document which components are added and why, include parameter validation, set sensible defaults
