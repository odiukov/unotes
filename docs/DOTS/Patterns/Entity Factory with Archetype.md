---
tags:
  - pattern
---
#### Description
- **Pre-configured entity spawning** pattern using `EntityArchetype` to define component composition upfront for fast, cache-friendly entity creation

- Factory struct stores archetype and provides Burst-compiled spawn methods, eliminating repeated component type lookups

- Enables type-safe entity creation with guaranteed component layout, preventing missing or incorrect components

- Particularly powerful when combined with [[EntityCommandBuffer]] for deferred spawning and conditional component setup

#### Example
```csharp
// Factory stores pre-created archetype
[BurstCompile]
public readonly struct EffectFactory
{
    private readonly EntityArchetype _effectArchetype;

    public EffectFactory(EntityManager entityManager)
    {
        // Define archetype once at construction
        _effectArchetype = entityManager.CreateArchetype(
            ComponentType.ReadWrite<Effect>(),
            ComponentType.ReadWrite<DestroyEntityFlag>(),
            ComponentType.ReadWrite<Processed>(),
            ComponentType.ReadWrite<EffectValue>(),
            ComponentType.ReadWrite<Producer>(),
            ComponentType.ReadWrite<Target>());
    }

    [BurstCompile]
    public void CreateEffect(EntityCommandBuffer ecb, EffectSetup setup,
        Entity producer, Entity target)
    {
        // Fast creation using pre-cached archetype
        var effect = ecb.CreateEntity(_effectArchetype);

        // Initialize component values
        ecb.SetComponent(effect, new EffectValue { Value = setup.Value });
        ecb.SetComponentEnabled<Processed>(effect, false);

        // Conditionally add type-specific components
        switch (setup.Type)
        {
            case EffectTypeId.Damage:
                ecb.AddComponent<DamageEffect>(effect);
                break;
            case EffectTypeId.Heal:
                ecb.AddComponent<HealEffect>(effect);
                break;
        }
    }
}

// Usage in system
[BurstCompile]
public partial struct SpawnSystem : ISystem
{
    private EffectFactory _factory;

    public void OnCreate(ref SystemState state)
    {
        _factory = new EffectFactory(state.EntityManager);
    }

    public void OnUpdate(ref SystemState state)
    {
        // Use factory to spawn
        _factory.CreateEffect(ecb, setup, producer, target);
    }
}
```

#### Pros
- **Performance** - archetype lookup cached at construction, entity creation is 2-3x faster than repeated `CreateEntity(params ComponentType[])`

- **[[Burst Compile|Burst]] compatible** - readonly struct with no managed references can be Burst compiled for maximum performance

- **Type safety** - factory method signatures enforce required parameters, preventing missing components at spawn time

- **[[Cache-friendly]]** - entities created with same archetype are stored in same [[Chunk]], improving cache locality

- **Centralized logic** - all entity creation code in one place, easier to maintain and modify

#### Cons
- **Archetype fragmentation** - conditional `AddComponent` in factory creates multiple [[Archetype|archetypes]], potentially reducing [[Cache-friendly|cache efficiency]]

- **Initialization overhead** - must construct factory in `OnCreate`, adds setup complexity

- **Limited flexibility** - archetype defined upfront, can't easily vary component composition per spawn

- **Memory footprint** - storing archetype in system state adds small memory cost per factory instance

#### Best use
- **Frequent spawning** - projectiles, effects, enemies that spawn hundreds per second benefit from cached archetype

- **Complex entity setup** - entities with 5+ components where manual creation is error-prone

- **Type variations** - multiple entity types (basic/homing/explosive projectile) that share common components

- **[[EntityCommandBuffer]] workflows** - factory methods take ECB, perfect for deferred spawning in jobs

- **Guaranteed composition** - when entity must have exact component set, factory enforces this at compile time

#### Avoid if
- **Rare spawning** - entities created once per level don't benefit from archetype caching overhead

- **Highly variable composition** - if every entity needs different components, archetype caching provides no benefit

- **Simple entities** - 1-2 component entities are simpler with direct `CreateEntity` call

- **Authoring-driven** - entities created from prefabs in editor should use [[Baker|baking]] workflow, not factories

#### Extra tip
- **Factory lifetime**: Store factory in system state, create once in OnCreate for maximum efficiency

- **Minimize fragmentation**: Include optional components in base archetype, use [[IEnableableComponent (toggleable components)|IEnableableComponent]] instead of AddComponent to avoid creating multiple archetypes

- **Buffer pre-allocation**: Set buffer capacity in factory to avoid heap allocation: `targets.Capacity = 10`

- **Parallel jobs**: Factories work great with parallel job ECB - capture factory for job, use `ecb.AsParallelWriter()`

- **Naming for debugging**: Use `ecb.SetName(effect, "DamageEffect")` for better entity inspector visibility

- **Multiple archetypes**: Create specialized factories for different entity categories (ProjectileFactory, EffectFactory, EnemyFactory)

- **Factory parameters**: Pass setup data via struct parameter for clean API instead of many method parameters

- **Performance comparison**: Factory archetype creation ~0.5ms for 10k entities vs manual ~2ms (4x speedup)

- **Readonly struct**: Mark factory readonly for better Burst optimization: `public readonly struct Factory`

- **Testing benefit**: Factories make unit tests cleaner by centralizing entity creation logic
