---
tags:
  - pattern
---
#### Description
- **Cleanup detection pattern** using paired [[ICleanupComponentData]] components to detect when a regular [[Component]] is removed or its [[Entity]] is destroyed

- Works by pairing a regular component with a cleanup component - when the regular component is removed, the cleanup component remains temporarily

- Query pattern uses `.WithNone<RegularComponent>().WithAll<CleanupComponent>()` to find entities where the regular component was removed

- Unity's official packages use this extensively - `Parent` paired with `PreviousParent` to detect parent relationship changes

#### Example
```csharp
// Regular component
public struct Parent : IComponentData
{
    public Entity Value;
}

// Cleanup component - remains when Parent removed
public struct PreviousParent : ICleanupComponentData
{
    public Entity Value;
}

// Detect removal
foreach (var (previousParent, entity) in
    SystemAPI.Query<RefRO<PreviousParent>>()
        .WithNone<Parent>()  // Parent was removed
        .WithEntityAccess())
{
    // Access last known parent value
    Entity oldParent = previousParent.ValueRO.Value;
    // Perform cleanup logic...
    ecb.RemoveComponent<PreviousParent>(entity);
}

// Maintain pair - add cleanup when Parent added
foreach (var (parent, entity) in
    SystemAPI.Query<RefRO<Parent>>()
        .WithNone<PreviousParent>()
        .WithEntityAccess())
{
    ecb.AddComponent(entity, new PreviousParent { Value = parent.ValueRO.Value });
}
```

**Pattern flow:**
1. Entity gets `Parent` component
2. System auto-adds `PreviousParent` cleanup component
3. When `Parent` removed, `PreviousParent` remains one frame
4. Detection system finds `.WithNone<Parent>().WithAll<PreviousParent>()`
5. System performs cleanup, removes `PreviousParent`

#### Pros
- **Captures last state** - cleanup component preserves last known value for cleanup logic

- **Automatic entity destruction handling** - cleanup components automatically removed when entity destroyed

- **Reliable detection** - guaranteed to detect component removal regardless of how removed

- **Used in Unity packages** - battle-tested pattern in Unity.Transforms, Unity.Scenes

- **Decoupled cleanup** - separation between component and cleanup logic

#### Cons
- **Requires two components** - doubles component count, increases memory overhead

- **Manual maintenance** - must ensure cleanup component is added/updated alongside regular component

- **Processing delay** - cleanup detection happens in next frame, not immediately

- **Extra [[Archetype]] overhead** - entities cycle through multiple archetypes

#### Best use
- **Parent-child relationships** - detect when parent/child relationships broken (like Unity.Transforms)

- **Resource cleanup** - detect when entities with external resources destroyed

- **Relationship tracking** - detect when entity references removed to update bidirectional relationships

- **Death detection** - trigger effects when health/alive component removed

- **Network synchronization** - detect entity destruction to send removal messages

#### Avoid if
- **Simple removal logic** - if no special cleanup needed, use regular component destruction

- **Frequent add/remove** - overhead of maintaining cleanup pairs too high

- **Can use [[IEnableableComponent (toggleable components)|IEnableableComponent]]** - if just toggling on/off, enableable components lighter

- **Immediate cleanup needed** - cleanup pattern has one-frame delay

#### Extra tip
- **Unity.Transforms example**: `Parent` + `PreviousParent` is canonical example - check Unity.Transforms source

- **Automatic cleanup**: Cleanup components automatically removed when entity destroyed

- **Combining with change filters**: Use [[WithChangeFilter (reactive updates)|WithChangeFilter]] on regular component to detect changes, cleanup component to detect removal

- **Auto-add integration**: Combine with [[Auto-Add System Pattern]] to automatically add cleanup when regular component added

- **Query optimization**: `.WithNone<Regular>().WithAll<Cleanup>()` efficient - only matches transition state entities

- **Store meaningful state**: Cleanup component can store snapshot, not just tag

- **Always remove cleanup**: Remove cleanup component after processing to avoid re-detection next frame

- **Best practices**: Name cleanup components clearly (Previous*, Cleanup*), use ISystem with BurstCompile, document behavior
