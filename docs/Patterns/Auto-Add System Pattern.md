---
tags:
  - pattern
---
#### Description
- **Automatic component addition pattern** where a system detects "entry point" [[Component]] and automatically adds required supporting components

- Query pattern uses `.WithAll<EntryComponent>().WithNone<SupportingComponent>()` to find entities missing required components

- Supporting components become difficult to manually remove - system re-adds them next frame, enforcing component dependencies

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
public struct CharacterVelocity : IComponentData { public float3 Value; }
public struct GroundedState : IComponentData { public bool IsGrounded; }

// Auto-add system
[UpdateInGroup(typeof(InitializationSystemGroup))]
[BurstCompile]
public partial struct AutoAddCharacterComponentsSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Find entities missing supporting components
        foreach (var (controller, entity) in
            SystemAPI.Query<RefRO<CharacterController>>()
                .WithNone<CharacterVelocity>()
                .WithEntityAccess())
        {
            // Auto-add required components
            ecb.AddComponent(entity, new CharacterVelocity { Value = float3.zero });
            ecb.AddComponent(entity, new GroundedState { IsGrounded = true });
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Auto-add with calculated values
public struct Enemy : IComponentData { public int Level; }
public struct EnemyStats : IComponentData
{
    public float Health;
    public float Damage;
    public float Speed;
}

foreach (var (enemy, entity) in
    SystemAPI.Query<RefRO<Enemy>>()
        .WithNone<EnemyStats>()
        .WithEntityAccess())
{
    // Calculate stats based on entry component data
    int level = enemy.ValueRO.Level;
    ecb.AddComponent(entity, new EnemyStats
    {
        Health = 100 + (level * 20),
        Damage = 10 + (level * 2),
        Speed = 3 + (level * 0.5f)
    });
}
```

**Common use cases:**
- Physics body → auto-add velocity + damping components
- Renderable → auto-add render bounds + material info
- Character → auto-add velocity + grounded state
- Parent → auto-add PreviousParent cleanup component

#### Pros
- **Enforces dependencies** - ensures required components always present, preventing bugs

- **Simplifies entity creation** - users only add entry component, supporting components added automatically

- **Self-healing** - if supporting components accidentally removed, re-added next frame

- **Decouples creation** - entity creation code doesn't need to know all required components

- **Centralized logic** - component dependencies defined in one place

#### Cons
- **Hidden behavior** - not obvious from entity inspector that components will be auto-added

- **Performance overhead** - system runs every frame checking for missing components

- **Can't manually remove** - supporting components resist removal, harder to test/debug

- **[[Structural changes]] cost** - adding components has overhead, though only happens once

- **Execution order dependency** - must run in InitializationSystemGroup or early

#### Best use
- **Required component pairs** - when component A always needs B and C (physics body + velocity + damping)

- **Complex features** - rendering, physics, AI requiring multiple components

- **Framework features** - reusable systems users interact with via single entry component

- **Preventing user errors** - when manual component management error-prone

- **Cleanup component pairs** - auto-add [[ICleanupComponentData]] cleanup components

#### Avoid if
- **Optional components** - if supporting components situational, don't auto-add

- **Performance critical** - if entities created very frequently, structural change overhead too high

- **Simple dependencies** - for 1-2 components, [[Helper Methods for Component Sets]] may be clearer

- **Need manual control** - if users need fine-grained control, auto-add too restrictive

#### Extra tip
- **Run early**: Use `[UpdateInGroup(typeof(InitializationSystemGroup))]` to ensure components added before other systems

- **RequireMatchingQueriesForUpdate**: Add this attribute to skip system when no matching entities:
  ```csharp
  [RequireMatchingQueriesForUpdate]
  public partial struct AutoAddSystem : ISystem { }
  ```

- **Multiple component sets**: Can use ComponentTypeSet to add multiple at once:
  ```csharp
  var components = new ComponentTypeSet(/* ... */);
  ecb.AddComponent(entity, components);
  ```

- **Cleanup component auto-add**: Common pattern for cleanup components:
  ```csharp
  // Auto-add PreviousParent when Parent added
  .WithAll<Parent>().WithNone<PreviousParent>()
  ```

- **Conditional auto-add**: Can check component values to decide what to add:
  ```csharp
  if (!config.ValueRO.IsPlayer)
      ecb.AddComponent<AIController>(entity);
  ```

- **Unity packages using this**: Unity.Transforms, Unity.Rendering, Unity.Physics all use variations

- **Documentation critical**: Clearly document which components are entry points and what gets auto-added

- **Performance optimization**: Cache EntityQuery in OnCreate and check `.IsEmpty`:
  ```csharp
  if (_missingComponentsQuery.IsEmpty) return;
  ```

- **Alternative - Baking**: Can add supporting components during baking instead of runtime:
  ```csharp
  public override void Bake(MyAuthoring authoring)
  {
      AddComponent<EntryComponent>(entity);
      AddComponent<SupportingComponent>(entity);
  }
  ```

- **Hybrid approach**: Use baking for SubScene entities, auto-add for runtime-spawned entities

- **Best practices**: Document entry points, run in InitializationSystemGroup, use RequireMatchingQueriesForUpdate, consider baking for static entities
