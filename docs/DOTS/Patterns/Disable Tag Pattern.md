---
tags:
  - pattern
---
#### Description
- **System pause pattern** using [[Tag Component]] that systems include in `.WithNone<DisableTag>()` queries to temporarily stop processing entities

- Allows pausing systems by adding disable tag, avoiding expensive component removal/re-addition and preserving component data

- Unity's `Unity.Scenes` package uses `DisableSceneResolveAndLoad` tag to temporarily pause scene loading

- More granular than disabling entire systems - can disable specific entities while others continue processing

#### Example
```csharp
// Define disable tag
public struct DisableAI : IComponentData { }

// System respects disable tag
foreach (var (aiState, entity) in
    SystemAPI.Query<RefRW<AIState>>()
        .WithNone<DisableAI>()  // Skip disabled entities
        .WithEntityAccess())
{
    // Update AI logic...
}

// Toggle processing
entityManager.AddComponent<DisableAI>(entity);     // Pause
entityManager.RemoveComponent<DisableAI>(entity);  // Resume

// Multiple systems can respect same disable tag
[BurstCompile]
public partial struct AIUpdateSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var ai in SystemAPI.Query<RefRW<AIState>>()
            .WithNone<DisableAI>()) { /* ... */ }
    }
}

[BurstCompile]
public partial struct AITargetingSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var target in SystemAPI.Query<RefRW<Target>>()
            .WithNone<DisableAI>()) { /* ... */ }
    }
}

// Stun effect using disable tags
public struct StunTimer : IComponentData { public float Remaining; }

foreach (var (stunTimer, entity) in
    SystemAPI.Query<RefRW<StunTimer>>().WithEntityAccess())
{
    if (stunTimer.ValueRO.Remaining > 0)
    {
        // Apply disable tags
        ecb.AddComponent<DisableMovement>(entity);
        ecb.AddComponent<DisableCombat>(entity);
        stunTimer.ValueRW.Remaining -= deltaTime;
    }
    else
    {
        // Remove disable tags
        ecb.RemoveComponent<DisableMovement>(entity);
        ecb.RemoveComponent<DisableCombat>(entity);
    }
}
```

**Common disable tags:**
- `DisableAI` - pause all AI systems
- `DisableRendering` - hide entity
- `DisablePhysics` - pause physics simulation
- `DisableMovement` - prevent movement
- `DisableCombat` - prevent attacks

#### Pros
- **Preserves data** - disabling via tag keeps all component data intact, no need to cache and restore

- **Cheaper than removal** - adding/removing zero-size tag faster than removing/re-adding data components

- **Granular control** - can disable specific entities while others continue, unlike disabling entire systems

- **Reversible** - easily toggle on/off, perfect for temporary states (pause, stun, cutscenes)

- **Multi-system coordination** - multiple systems can respect same disable tag

#### Cons
- **Must update all queries** - every relevant system must include `.WithNone<DisableTag>()`

- **Easy to forget** - developers might create new systems and forget disable tag check

- **[[Structural changes]] still occur** - adding/removing tags causes structural changes

- **Not automatic** - requires manual coordination, unlike [[IEnableableComponent (toggleable components)|IEnableableComponent]]

- **Memory overhead** - tag component creates archetype variants (with/without disable tag)

#### Best use
- **Pause/resume systems** - temporarily pause scene loading, AI, rendering without losing state

- **Status effects** - stun, freeze, sleep effects that disable multiple systems

- **Cutscenes** - disable gameplay systems during cutscenes while keeping entity state

- **Debugging** - selectively disable entity processing to debug specific behaviors

- **Multi-system coordination** - when multiple systems need to pause together

#### Avoid if
- **Single system** - if only one system needs check, consider component flag instead

- **Frequent toggling** - if changes every frame, [[IEnableableComponent (toggleable components)|IEnableableComponent]] more efficient (no structural changes)

- **Complex conditions** - if logic is complex, consider state machine components

- **Need per-component disable** - disable tags affect entire entity, use IEnableableComponent for specific components

#### Extra tip
- **Naming conventions**: Prefix with `Disable` (DisableAI, DisableRendering) for clarity

- **Multiple disable tags**: Can use multiple for different system groups: `.WithNone<DisableMovement>().WithNone<DisableCombat>()`

- **IEnableableComponent alternative**: For frequent toggling, use IEnableableComponent to avoid structural changes:
  ```csharp
  public struct AIEnabled : IComponentData, IEnableableComponent { }
  SystemAPI.SetComponentEnabled<AIEnabled>(entity, false);  // No structural change
  ```

- **Global disable via singleton**: Can use singleton to disable all entities:
  ```csharp
  if (SystemAPI.HasSingleton<GlobalDisableAI>()) return;
  ```

- **Unity.Scenes usage**: `DisableSceneResolveAndLoad` is real example from Unity packages

- **Documentation critical**: Document which systems respect which disable tags - easy to forget

- **Archetype fragmentation**: Disable tags create variants - Enemy, Enemy+DisableAI, Enemy+DisableMovement, Enemy+Both

- **Batch operations**: When disabling many entities, batch tag additions for efficiency

- **Editor debugging**: Disable tags visible in Entity Inspector, easy to see why entity not processing

- **When to use IEnableableComponent instead**: Disable specific component, frequent toggling, avoid structural changes, per-component state

- **When to use disable tag**: Coordinate multiple systems, temporary infrequent disable, simple on/off, following Unity patterns
