---
tags:
  - pattern
---
#### Description
- **Cleanup detection pattern** that uses paired [[ICleanupComponentData]] components to detect when a regular [[Component]] is removed or its [[Entity]] is destroyed

- Works by pairing a regular component with a cleanup component - when the regular component is removed, the cleanup component remains temporarily, allowing systems to detect and react to the removal

- Query pattern uses `.WithNone<RegularComponent>().WithAll<CleanupComponent>()` to find entities where the regular component was removed but cleanup hasn't happened yet

- Unity's official packages use this extensively - for example, `Parent` component is paired with `PreviousParent` cleanup component to detect parent relationship changes

#### Example
```csharp
// Regular component
public struct Parent : IComponentData
{
    public Entity Value;
}

// Cleanup component - automatically removed when entity destroyed
public struct PreviousParent : ICleanupComponentData
{
    public Entity Value;
}

// System that detects when Parent component is removed
[BurstCompile]
public partial struct DetectParentRemovalSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Find entities where Parent was removed but PreviousParent remains
        foreach (var (previousParent, entity) in
            SystemAPI.Query<RefRO<PreviousParent>>()
                .WithNone<Parent>()  // Parent was removed
                .WithEntityAccess())
        {
            // Access last known parent value
            Entity oldParent = previousParent.ValueRO.Value;

            // Perform cleanup logic
            // e.g., remove entity from old parent's child list

            // Remove cleanup component after processing
            ecb.RemoveComponent<PreviousParent>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// System that maintains Parent-PreviousParent pair
[BurstCompile]
public partial struct MaintainParentCleanupSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Add cleanup component when Parent is added
        foreach (var (parent, entity) in
            SystemAPI.Query<RefRO<Parent>>()
                .WithNone<PreviousParent>()
                .WithEntityAccess())
        {
            ecb.AddComponent(entity, new PreviousParent { Value = parent.ValueRO.Value });
        }

        // Update cleanup component when Parent changes
        foreach (var (parent, previousParent) in
            SystemAPI.Query<RefRO<Parent>, RefRW<PreviousParent>>()
                .WithChangeFilter<Parent>())
        {
            previousParent.ValueRW.Value = parent.ValueRO.Value;
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Health component with cleanup detection
public struct Health : IComponentData
{
    public float Current;
    public float Max;
}

public struct PreviousHealth : ICleanupComponentData
{
    public float LastKnownHealth;
}

// Detect when entity dies (Health component removed)
[BurstCompile]
public partial struct DeathDetectionSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (prevHealth, entity) in
            SystemAPI.Query<RefRO<PreviousHealth>>()
                .WithNone<Health>()  // Health was removed (entity died)
                .WithEntityAccess())
        {
            // Spawn death effects, update statistics, etc.
            // Can access last known health value
            float lastHealth = prevHealth.ValueRO.LastKnownHealth;

            // Clean up
            ecb.RemoveComponent<PreviousHealth>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```

#### Pros
- **Captures last state** - cleanup component preserves the last known value of the removed component for cleanup logic

- **Automatic entity destruction handling** - cleanup components are automatically removed when entity is destroyed, triggering detection

- **Reliable detection** - guaranteed to detect component removal regardless of how it was removed (manual, entity destroyed, [[Structural changes]])

- **Used in Unity packages** - battle-tested pattern used in Unity.Transforms, Unity.Scenes, and other official packages

- **Decoupled cleanup** - separation between regular component and cleanup logic improves code organization

#### Cons
- **Requires two components** - doubles component count for tracked data, increases memory overhead

- **Manual maintenance** - must ensure cleanup component is added/updated alongside regular component (can use Auto-Add System Pattern)

- **Processing delay** - cleanup detection happens in next frame, not immediately when component removed

- **Extra [[Archetype]] overhead** - entities cycle through multiple archetypes (with regular, with both, with cleanup only)

#### Best use
- **Parent-child relationships** - detect when parent/child relationships are broken (like Unity.Transforms does)

- **Resource cleanup** - detect when entities with external resources (file handles, native allocations) are destroyed

- **Relationship tracking** - detect when entity references are removed to update bidirectional relationships

- **Death detection** - trigger death effects, statistics, achievements when health/alive component removed

- **Network synchronization** - detect entity destruction to send removal messages to clients

#### Avoid if
- **Simple removal logic** - if you just need to destroy entity without special cleanup, use regular component destruction

- **Frequent add/remove** - if components are added/removed frequently, overhead of maintaining cleanup pairs may be too high

- **Can use [[IEnableableComponent (toggleable components)|IEnableableComponent]]** - if you just need to toggle behavior on/off, enableable components are lighter weight

- **Immediate cleanup needed** - cleanup pattern has one-frame delay, if you need instant cleanup use direct component removal with immediate processing

#### Extra tip
- **Unity.Transforms example** - `Parent` + `PreviousParent` is the canonical example, check Unity.Transforms package source code for implementation details

- **Automatic cleanup** - cleanup components are automatically removed when entity is destroyed, you don't need to manually remove them in that case

- **Combining with change filters** - can use [[WithChangeFilter (reactive updates)|WithChangeFilter]] on regular component to detect changes, then cleanup component to detect removal

- **Auto-add pattern integration** - combine with Auto-Add System Pattern to automatically add cleanup component when regular component is added

- **Query optimization** - queries with `.WithNone<Regular>().WithAll<Cleanup>()` are efficient because they only match entities in transition state

- **Cleanup component data** - cleanup component can store snapshot of regular component data, not just tag, useful for accessing last known state

- **Multiple cleanup components** - can have multiple cleanup components for same entity to track different removal events

- **[[EntityCommandBuffer]] usage** - always remove cleanup component in same system that detects it, otherwise it will be detected again next frame

- **Best practices:**
  - Always remove cleanup component after processing to avoid re-detection
  - Store meaningful state in cleanup component (last value, timestamp, etc.)
  - Consider using [[ISystem]] with [[BurstCompile]] for performance
  - Name cleanup components clearly (Previous*, Cleanup*, etc.)

- **Debugging** - use Entity Debugger window to inspect entities with cleanup components to verify they're being removed properly

- **Performance consideration** - cleanup pattern adds [[Structural changes]] overhead (component add/remove), use sparingly on hot paths
