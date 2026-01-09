---
tags:
  - pattern
---
#### Description
- **[[Archetype]] caching pattern** defining reusable component combinations in struct, allowing multiple systems to reference same archetype definitions

- Implemented as static helper returning archetype collections, centralizing archetype definitions in one location

- Useful when multiple systems create entities with same component combinations or query similar archetypes

- Unity's `Unity.Scenes` uses this with `CreateResolveSceneSectionArchetypes()` to share definitions across scene loading systems

#### Example
```csharp
// Define reusable archetype group
public struct SceneArchetypes
{
    public EntityArchetype ResolvedSection;
    public EntityArchetype RequestLoaded;
    public EntityArchetype MetaData;

    public static SceneArchetypes Create(EntityManager em)
    {
        return new SceneArchetypes
        {
            ResolvedSection = em.CreateArchetype(
                ComponentType.ReadWrite<SceneSectionData>(),
                ComponentType.ReadWrite<LinkedEntityGroup>()),

            RequestLoaded = em.CreateArchetype(
                ComponentType.ReadWrite<RequestSceneLoaded>(),
                ComponentType.ReadWrite<SceneReference>()),

            MetaData = em.CreateArchetype(
                ComponentType.ReadWrite<SceneMetaData>())
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
        _archetypes = SceneArchetypes.Create(state.EntityManager);
    }

    public void OnUpdate(ref SystemState state)
    {
        // Reuse archetype for entity creation
        Entity scene = state.EntityManager.CreateEntity(_archetypes.RequestLoaded);
    }
}
```

**Pattern structure:**
1. Define struct with `EntityArchetype` fields
2. Static `Create()` method builds all archetypes
3. Systems call `Create()` in `OnCreate()`, store in field
4. Use stored archetypes for entity creation

#### Pros
- **No duplication** - archetype definitions centralized, avoiding copy-paste errors

- **Performance** - archetypes created once and reused, avoiding repeated lookup

- **Consistency** - guarantees all systems use same component combinations

- **Maintainability** - change archetype in one place, all systems automatically updated

- **Type safety** - named fields prevent using wrong archetype

#### Cons
- **Memory overhead** - each system stores archetype references

- **Initialization requirement** - must call `Create()` in OnCreate

- **Not dynamic** - archetype definitions static, can't vary at runtime

- **Overhead for simple cases** - overkill for 1-2 archetypes used once

#### Best use
- **Shared entity types** - multiple systems create same entity types (scenes, projectiles, effects)

- **Complex archetypes** - entities with many components where duplication error-prone

- **Framework systems** - packages providing reusable systems (like Unity.Scenes)

- **Large projects** - consistency important across many systems

#### Avoid if
- **Single use** - archetype only used in one system, no need to share

- **Simple archetypes** - 1-2 components easier to define inline

- **Dynamic archetypes** - if composition varies at runtime, pattern doesn't fit

- **Small projects** - overhead not justified for few systems

#### Extra tip
- **Unity.Scenes example**: Check Unity.Scenes package source for production implementation

- **ComponentTypeSet alternative**: Can return `ComponentTypeSet` instead of `EntityArchetype` for more flexibility

- **Combine with factories**: Use with [[Entity Factory with Archetype]] pattern for complete entity creation encapsulation

- **Partial application**: Can define some archetypes in struct, others inline in systems

- **Naming convention**: Suffix struct with `Archetypes` for clarity (SceneArchetypes, CharacterArchetypes)

- **Capacity hint**: When creating archetypes, can provide initial capacity hint for buffers

- **Burst compatibility**: Archetype references are Burst-compatible, entire pattern works in Burst

- **Testing**: In tests, can create archetypes struct to ensure test entities match production

- **Documentation**: Document each archetype field's purpose and when to use

- **Incremental adoption**: Can start with most commonly duplicated archetypes, add others gradually
