---
tags:
  - pattern
---
#### Description
- **System pause pattern** using a [[Tag Component]] that systems include in `.WithNone<DisableTag>()` queries to temporarily stop processing entities without removing their other components

- Allows pausing systems by adding a disable tag component to entities, avoiding expensive component removal/re-addition and preserving component data

- Unity's `Unity.Scenes` package uses `DisableSceneResolveAndLoad` tag to temporarily pause scene loading operations

- More granular than disabling entire systems - can disable specific entities while others continue processing

#### Example
```csharp
// Define disable tag component
public struct DisableSceneResolveAndLoad : IComponentData { }

// System respects disable tag
[BurstCompile]
public partial struct SceneLoadingSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // WithNone excludes entities with disable tag
        foreach (var (sceneRef, entity) in
            SystemAPI.Query<RefRO<RequestSceneLoaded>>()
                .WithNone<DisableSceneResolveAndLoad>()  // Skip disabled entities
                .WithEntityAccess())
        {
            // Process scene loading...
        }
    }
}

// Toggle system processing by adding/removing tag
public static class SceneLoadingControl
{
    public static void PauseSceneLoading(EntityManager em, Entity sceneEntity)
    {
        em.AddComponent<DisableSceneResolveAndLoad>(sceneEntity);
    }

    public static void ResumeSceneLoading(EntityManager em, Entity sceneEntity)
    {
        em.RemoveComponent<DisableSceneResolveAndLoad>(sceneEntity);
    }
}

// Example: AI disable tag
public struct DisableAI : IComponentData { }

[BurstCompile]
public partial struct AIUpdateSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (aiState, entity) in
            SystemAPI.Query<RefRW<AIState>>()
                .WithNone<DisableAI>()  // Skip AI-disabled entities
                .WithEntityAccess())
        {
            // Update AI logic...
        }
    }
}

[BurstCompile]
public partial struct AITargetingSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (target, entity) in
            SystemAPI.Query<RefRW<Target>>()
                .WithNone<DisableAI>()  // All AI systems check same disable tag
                .WithEntityAccess())
        {
            // Update targeting...
        }
    }
}

// Example: Rendering disable
public struct DisableRendering : IComponentData { }

[BurstCompile]
public partial struct RenderingSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (renderData, transform) in
            SystemAPI.Query<RefRO<RenderMesh>, RefRO<LocalTransform>>()
                .WithNone<DisableRendering>())
        {
            // Render entity...
        }
    }
}

// Example: Physics disable
public struct DisablePhysics : IComponentData { }

[BurstCompile]
public partial struct PhysicsSimulationSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (velocity, mass) in
            SystemAPI.Query<RefRW<PhysicsVelocity>, RefRO<PhysicsMass>>()
                .WithNone<DisablePhysics>())
        {
            // Simulate physics...
        }
    }
}

// Example: Conditional disable based on game state
public struct GamePaused : IComponentData { }  // Singleton

[BurstCompile]
public partial struct PauseControlSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<GamePaused>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        bool isPaused = SystemAPI.GetSingleton<GamePaused>().Value;
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        if (isPaused)
        {
            // Add disable tag to all gameplay entities
            foreach (var entity in
                SystemAPI.QueryBuilder()
                    .WithAll<Gameplay>()
                    .WithNone<DisableGameplay>()
                    .Build()
                    .ToEntityArray(Allocator.Temp))
            {
                ecb.AddComponent<DisableGameplay>(entity);
            }
        }
        else
        {
            // Remove disable tag to resume
            foreach (var entity in
                SystemAPI.QueryBuilder()
                    .WithAll<DisableGameplay>()
                    .Build()
                    .ToEntityArray(Allocator.Temp))
            {
                ecb.RemoveComponent<DisableGameplay>(entity);
            }
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Multiple disable tags for different systems
public struct DisableMovement : IComponentData { }
public struct DisableCombat : IComponentData { }
public struct DisableInteraction : IComponentData { }

[BurstCompile]
public partial struct MovementSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (velocity, entity) in
            SystemAPI.Query<RefRW<Velocity>>()
                .WithNone<DisableMovement>()
                .WithEntityAccess())
        {
            // Process movement...
        }
    }
}

[BurstCompile]
public partial struct CombatSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (weapon, entity) in
            SystemAPI.Query<RefRW<Weapon>>()
                .WithNone<DisableCombat>()
                .WithEntityAccess())
        {
            // Process combat...
        }
    }
}

// Example: Stun effect using disable tags
[BurstCompile]
public partial struct StunSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);
        float deltaTime = SystemAPI.Time.DeltaTime;

        foreach (var (stunTimer, entity) in
            SystemAPI.Query<RefRW<StunTimer>>()
                .WithEntityAccess())
        {
            if (stunTimer.ValueRO.Remaining > 0)
            {
                // Keep entity stunned - ensure disable tags present
                if (!SystemAPI.HasComponent<DisableMovement>(entity))
                {
                    ecb.AddComponent<DisableMovement>(entity);
                }
                if (!SystemAPI.HasComponent<DisableCombat>(entity))
                {
                    ecb.AddComponent<DisableCombat>(entity);
                }

                // Decrease timer
                stunTimer.ValueRW.Remaining -= deltaTime;
            }
            else
            {
                // Stun expired - remove disable tags
                ecb.RemoveComponent<DisableMovement>(entity);
                ecb.RemoveComponent<DisableCombat>(entity);
                ecb.RemoveComponent<StunTimer>(entity);
            }
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```

#### Pros
- **Preserves data** - disabling via tag keeps all component data intact, no need to cache and restore values

- **Cheaper than removal** - adding/removing zero-size tag component is faster than removing/re-adding data components

- **Granular control** - can disable specific entities while others continue processing, unlike disabling entire systems

- **Reversible** - easily toggle on/off by adding/removing tag, perfect for temporary states (pause, stun, cutscenes)

- **No [[Sync points]]** - unlike removing components with data, tag removal doesn't force immediate structural changes

#### Cons
- **Must update all queries** - every system that should respect disable tag must include `.WithNone<DisableTag>()`

- **Easy to forget** - developers might create new systems and forget to add disable tag check

- **[[Structural changes]] still occur** - adding/removing tags causes structural changes, though cheaper than data components

- **Not automatic** - requires manual coordination across systems, unlike [[IEnableableComponent (toggleable components)|IEnableableComponent]] which is built-in

- **Memory overhead** - tag component adds to [[Archetype]], creating archetype variants (with/without disable tag)

#### Best use
- **Pause/resume systems** - temporarily pause scene loading, AI processing, rendering without losing state

- **Status effects** - stun, freeze, sleep effects that disable multiple systems temporarily

- **Cutscenes** - disable gameplay systems during cutscenes while keeping entity state

- **Debugging** - selectively disable entity processing to debug specific behaviors

- **Multi-system coordination** - when multiple systems need to be paused together for specific entities

#### Avoid if
- **Single system** - if only one system needs to check disabled state, consider using regular component flag instead

- **Frequent toggling** - if disabled state changes every frame, [[IEnableableComponent (toggleable components)|IEnableableComponent]] is more efficient (no structural changes)

- **Complex conditions** - if enable/disable logic is complex, consider using state machine components instead

- **Need per-component disable** - disable tags affect entire entity, use [[IEnableableComponent (toggleable components)|IEnableableComponent]] to disable specific components

#### Extra tip
- **Naming conventions** - prefix with `Disable` (DisableAI, DisableRendering) or suffix with domain (SceneLoadingDisabled)

- **Multiple disable tags** - can use multiple disable tags for different system groups:
  ```csharp
  .WithNone<DisableMovement>()
  .WithNone<DisableCombat>()
  ```

- **Combine with [[IEnableableComponent (toggleable components)|IEnableableComponent]]** - for frequent toggling, prefer IEnableableComponent (no structural changes):
  ```csharp
  public struct AIEnabled : IComponentData, IEnableableComponent { }

  // Toggle without structural changes
  SystemAPI.SetComponentEnabled<AIEnabled>(entity, false);
  ```

- **Global disable via singleton** - can use singleton component to disable all entities:
  ```csharp
  bool globallyDisabled = SystemAPI.HasSingleton<GlobalDisableAI>();
  if (globallyDisabled) return;
  ```

- **Unity.Scenes usage** - `DisableSceneResolveAndLoad` is real example from Unity packages

- **Documentation critical** - document which systems respect which disable tags, easy to forget

- **Testing consideration** - tests should verify systems correctly respect disable tags

- **Combining disable tags** - can create compound disable behavior:
  ```csharp
  // Disable all AI
  ecb.AddComponent<DisableAI>(entity);

  // More specific disable
  ecb.AddComponent<DisableTargeting>(entity);
  ```

- **Cleanup pattern** - pair with cleanup components to detect when disable tag removed:
  ```csharp
  // Detect when DisableAI is removed to trigger re-initialization
  foreach (var entity in
      SystemAPI.QueryBuilder()
          .WithAll<PreviousDisableAI>()
          .WithNone<DisableAI>()
          .Build()
          .ToEntityArray(Allocator.Temp))
  {
      // Re-initialize AI
  }
  ```

- **Performance consideration** - queries with `.WithNone` are slightly slower than queries without, but difference is negligible

- **Alternative: System disable** - to disable entire system group, use `World.GetOrCreateSystemManaged<T>().Enabled = false`

- **Archetype fragmentation** - disable tags create archetype variants, consider impact on chunk utilization:
  - Enemy archetype
  - Enemy + DisableAI archetype
  - Enemy + DisableMovement archetype
  - Enemy + DisableAI + DisableMovement archetype

- **Batch operations** - when disabling many entities, batch tag additions:
  ```csharp
  var entitiesToDisable = query.ToEntityArray(Allocator.Temp);
  ecb.AddComponent<DisableTag>(entitiesToDisable);
  ```

- **Editor debugging** - disable tags visible in Entity Inspector, easy to see why entity not processing

- **Best practices:**
  - Document disable tag behavior in component comments
  - Use consistent naming (Disable prefix)
  - Add disable check to all relevant systems
  - Consider IEnableableComponent for frequent toggling
  - Test that systems properly respect disable tags

- **When to use IEnableableComponent instead:**
  - Need to disable specific component, not entire entity
  - Toggle happens frequently (every frame or multiple times per frame)
  - Want to avoid structural changes
  - Need per-component enable/disable state

- **When to use disable tag:**
  - Need to coordinate multiple systems
  - Disable state is temporary and infrequent
  - Want simple on/off toggle for entity processing
  - Following established patterns (like Unity.Scenes)
