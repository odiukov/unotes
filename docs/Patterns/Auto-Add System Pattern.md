---
tags:
  - pattern
---
#### Description
- **Automatic component addition pattern** where a system detects an "entry point" [[Component]] and automatically adds required supporting components, ensuring dependencies are always satisfied

- Query pattern uses `.WithAll<EntryComponent>().WithNone<SupportingComponent>()` to find entities missing required components

- Supporting components become difficult to manually remove - system will re-add them next frame, enforcing component dependencies

- Useful for maintaining invariants where certain components always require other components to function correctly

#### Example
```csharp
// Entry point component - user adds this
public struct CharacterController : IComponentData
{
    public float Speed;
    public float JumpForce;
}

// Supporting components - auto-added by system
public struct CharacterVelocity : IComponentData
{
    public float3 Value;
}

public struct GroundedState : IComponentData
{
    public bool IsGrounded;
}

// Auto-add system
[UpdateInGroup(typeof(InitializationSystemGroup))]
[BurstCompile]
public partial struct AutoAddCharacterComponentsSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Find entities with CharacterController but missing supporting components
        foreach (var (controller, entity) in
            SystemAPI.Query<RefRO<CharacterController>>()
                .WithNone<CharacterVelocity>()  // Missing this component
                .WithEntityAccess())
        {
            // Automatically add required components
            ecb.AddComponent(entity, new CharacterVelocity { Value = float3.zero });
            ecb.AddComponent(entity, new GroundedState { IsGrounded = true });
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Physics auto-add
public struct PhysicsBody : IComponentData
{
    public float Mass;
}

// Supporting physics components
public struct PhysicsVelocity : IComponentData
{
    public float3 Linear;
    public float3 Angular;
}

public struct PhysicsDamping : IComponentData
{
    public float Linear;
    public float Angular;
}

[UpdateInGroup(typeof(InitializationSystemGroup))]
[BurstCompile]
public partial struct AutoAddPhysicsComponentsSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Auto-add velocity
        foreach (var entity in
            SystemAPI.QueryBuilder()
                .WithAll<PhysicsBody>()
                .WithNone<PhysicsVelocity>()
                .Build()
                .ToEntityArray(Allocator.Temp))
        {
            ecb.AddComponent(entity, new PhysicsVelocity
            {
                Linear = float3.zero,
                Angular = float3.zero
            });
        }

        // Auto-add damping
        foreach (var entity in
            SystemAPI.QueryBuilder()
                .WithAll<PhysicsBody>()
                .WithNone<PhysicsDamping>()
                .Build()
                .ToEntityArray(Allocator.Temp))
        {
            ecb.AddComponent(entity, new PhysicsDamping
            {
                Linear = 0.01f,
                Angular = 0.05f
            });
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Multiple entry points
public struct Renderable : IComponentData { }
public struct WorldRenderBounds : IComponentData { }
public struct MaterialMeshInfo : IComponentData { }

[UpdateInGroup(typeof(InitializationSystemGroup))]
[UpdateAfter(typeof(TransformSystemGroup))]
[BurstCompile]
public partial struct AutoAddRenderComponentsSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Auto-add multiple components at once
        var componentsToAdd = new FixedList128Bytes<ComponentType>
        {
            ComponentType.ReadWrite<WorldRenderBounds>(),
            ComponentType.ReadWrite<MaterialMeshInfo>(),
            ComponentType.ReadWrite<LocalToWorld>(),
        };

        foreach (var entity in
            SystemAPI.QueryBuilder()
                .WithAll<Renderable>()
                .WithNone<WorldRenderBounds>()
                .Build()
                .ToEntityArray(Allocator.Temp))
        {
            // Add all supporting components together
            ecb.AddComponent(entity, new ComponentTypeSet(componentsToAdd));
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Conditional auto-add
public struct Health : IComponentData
{
    public float Current;
    public float Max;
}

public struct Regeneration : IComponentData
{
    public float RatePerSecond;
}

public struct HealthBar : IComponentData
{
    public Entity UIElement;
}

[UpdateInGroup(typeof(InitializationSystemGroup))]
[BurstCompile]
public partial struct AutoAddHealthComponentsSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Auto-add only if health has regeneration
        foreach (var (health, regen, entity) in
            SystemAPI.Query<RefRO<Health>, RefRO<Regeneration>>()
                .WithNone<HealthBar>()
                .WithEntityAccess())
        {
            // Only add health bar if entity has regeneration
            ecb.AddComponent(entity, new HealthBar { UIElement = Entity.Null });
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Auto-add with initial values based on entry component
public struct Enemy : IComponentData
{
    public int Level;
}

public struct EnemyStats : IComponentData
{
    public float Health;
    public float Damage;
    public float Speed;
}

[UpdateInGroup(typeof(InitializationSystemGroup))]
[BurstCompile]
public partial struct AutoAddEnemyStatsSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (enemy, entity) in
            SystemAPI.Query<RefRO<Enemy>>()
                .WithNone<EnemyStats>()
                .WithEntityAccess())
        {
            // Calculate stats based on level
            int level = enemy.ValueRO.Level;
            ecb.AddComponent(entity, new EnemyStats
            {
                Health = 100 + (level * 20),
                Damage = 10 + (level * 2),
                Speed = 3 + (level * 0.5f)
            });
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```

#### Pros
- **Enforces dependencies** - ensures required components are always present, preventing bugs from missing components

- **Simplifies entity creation** - users only need to add entry component, supporting components added automatically

- **Self-healing** - if supporting components are accidentally removed, they're re-added next frame

- **Decouples creation** - entity creation code doesn't need to know all required components

- **Centralized logic** - component dependencies defined in one place (auto-add system) rather than scattered across codebase

#### Cons
- **Hidden behavior** - not obvious from entity inspector or code that components will be auto-added

- **Performance overhead** - system runs every frame checking for missing components (though query is usually empty)

- **Can't manually remove** - supporting components resist removal, making testing and debugging harder

- **[[Structural changes]] cost** - adding components has structural change overhead, though only happens once per entity

- **Execution order dependency** - must run in InitializationSystemGroup or early to avoid one-frame delays

#### Best use
- **Required component pairs** - when component A always needs components B and C to function (physics body + velocity + damping)

- **Complex features** - rendering, physics, AI that require multiple components to work correctly

- **Framework features** - when building reusable systems that users interact with via single entry component

- **Preventing user errors** - when manual component management is error-prone (forgetting required components)

- **Cleanup component pairs** - auto-add [[ICleanupComponentData]] cleanup components when regular components are added

#### Avoid if
- **Optional components** - if supporting components are optional or situational, don't auto-add them

- **Performance critical** - if entities are created very frequently, structural change overhead may be too high

- **Simple dependencies** - for 1-2 components, [[Helper Methods for Component Sets]] pattern may be clearer

- **Need manual control** - if users need fine-grained control over component presence, auto-add is too restrictive

#### Extra tip
- **Run early** - use `[UpdateInGroup(typeof(InitializationSystemGroup))]` to ensure components added before other systems run

- **Combine with helper methods** - use [[Helper Methods for Component Sets]] pattern to add multiple components efficiently:
  ```csharp
  var componentsToAdd = new ComponentTypeSet(new FixedList128Bytes<ComponentType> { /* ... */ });
  ecb.AddComponent(entity, componentsToAdd);
  ```

- **[[RequireMatchingQueriesForUpdate]] optimization** - add this attribute to skip system when no matching entities exist:
  ```csharp
  [RequireMatchingQueriesForUpdate]
  public partial struct AutoAddSystem : ISystem { }
  ```

- **Multiple queries** - can have multiple queries in same system for different entry/supporting component pairs

- **Cleanup component auto-add** - common pattern is auto-adding cleanup components:
  ```csharp
  // Auto-add PreviousParent when Parent is added
  foreach (var entity in
      SystemAPI.QueryBuilder()
          .WithAll<Parent>()
          .WithNone<PreviousParent>()
          .Build()
          .ToEntityArray(Allocator.Temp))
  {
      ecb.AddComponent<PreviousParent>(entity);
  }
  ```

- **Conditional auto-add** - can check component values to decide what to add:
  ```csharp
  foreach (var (config, entity) in
      SystemAPI.Query<RefRO<CharacterConfig>>()
          .WithNone<AIController>()
          .WithEntityAccess())
  {
      if (!config.ValueRO.IsPlayer)
      {
          ecb.AddComponent<AIController>(entity);
      }
  }
  ```

- **Documentation is critical** - clearly document which components are entry points and what gets auto-added

- **Unity packages using this** - Unity.Transforms, Unity.Rendering, Unity.Physics all use variations of this pattern

- **Testing considerations** - tests need to run auto-add systems or manually add all required components

- **Debugging auto-adds** - add debug logs in editor to show when components are auto-added:
  ```csharp
  #if UNITY_EDITOR
  Debug.Log($"Auto-added supporting components to entity {entity}");
  #endif
  ```

- **Remove resistance** - supporting components can be removed with `EntityManager.RemoveComponent` but will be re-added next frame unless entry component also removed

- **Performance optimization** - cache EntityQuery in OnCreate and use `.IsEmpty` check:
  ```csharp
  private EntityQuery _missingComponentsQuery;

  public void OnCreate(ref SystemState state)
  {
      _missingComponentsQuery = SystemAPI.QueryBuilder()
          .WithAll<EntryComponent>()
          .WithNone<SupportingComponent>()
          .Build();
  }

  public void OnUpdate(ref SystemState state)
  {
      if (_missingComponentsQuery.IsEmpty) return;
      // Process additions...
  }
  ```

- **Alternative: Prefabs** - for static entity types, consider using prefabs with all components pre-added instead of auto-add systems

- **Alternative: [[Baking]]** - can add supporting components during baking process instead of runtime auto-add:
  ```csharp
  public override void Bake(MyAuthoring authoring)
  {
      var entity = GetEntity(TransformUsageFlags.Dynamic);
      AddComponent<EntryComponent>(entity);
      AddComponent<SupportingComponent1>(entity);
      AddComponent<SupportingComponent2>(entity);
  }
  ```

- **Hybrid approach** - use baking for SubScene entities, auto-add for runtime-spawned entities

- **Error checking** - can add validation to ensure component values are valid when auto-adding
