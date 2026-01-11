---
tags:
  - component
  - advanced
---

#### Description
- **Toggleable components** implementing both [[IComponentData]] and IEnableableComponent - can be enabled/disabled at runtime without [[Structural changes]]

- **Per-entity bitmask** stored in [[Chunk]] tracks enabled/disabled state - component memory always allocated but logically "turned off"

- **Query filtering** supports special filters (WithAll, WithNone, WithDisabled, WithPresent) to process enabled/disabled entities differently

- **Zero-size optimization** - empty enableable components are ideal for boolean flags without memory overhead beyond the enable bit

#### Example
```csharp
// Define enableable component (usually zero-size for flags)
public struct Stunned : IComponentData, IEnableableComponent {}

public struct Shield : IComponentData, IEnableableComponent
{
    public float Strength;  // Can have data like regular IComponentData
}

// Toggling enabled state
foreach (var entity in SystemAPI.Query<RefRO<Entity>>().WithEntityAccess())
{
    // Check and toggle enabled state
    if (SystemAPI.IsComponentEnabled<Stunned>(entity))
    {
        SystemAPI.SetComponentEnabled<Stunned>(entity, false);
    }
}

// Query filtering by enabled/disabled state
// Process only entities with ENABLED Stunned component
foreach (var transform in
    SystemAPI.Query<RefRW<LocalTransform>>()
        .WithAll<Stunned>())  // Only enabled Stunned
{
    // These entities are stunned - don't move
}

// Process entities WITHOUT enabled Stunned component
foreach (var (transform, velocity) in
    SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>()
        .WithNone<Stunned>())  // Excludes enabled Stunned
{
    transform.ValueRW.Position += velocity.ValueRO.Value * deltaTime;
}

// Process entities with DISABLED Stunned component
foreach (var entity in
    SystemAPI.Query<RefRO<Entity>>()
        .WithDisabled<Stunned>())  // Only disabled
{
    // Entity was stunned but recovered
}

// ComponentLookup with enableable components
var stunnedLookup = SystemAPI.GetComponentLookup<Stunned>(isReadOnly: false);
if (stunnedLookup.IsComponentEnabled(targetEntity))
{
    stunnedLookup.SetComponentEnabled(targetEntity, false);
}

// EntityCommandBuffer support
ecb.SetComponentEnabled<Stunned>(entity, true);
```

#### Pros
- **No structural changes** - toggling doesn't move entity between chunks, avoiding expensive archetype changes

- **Query filtering** - efficiently filter enabled vs disabled entities with WithAll, WithNone, WithDisabled, WithPresent

- **Zero-size flags** - empty enableable components use minimal memory (just enable bit), perfect for boolean state

- **Component data preserved** - disabling keeps data intact, re-enabling doesn't require setting data again

#### Cons
- **Memory always allocated** - disabled components still consume chunk memory, not removed until component removed

- **Mixed-state overhead** - chunks with both enabled and disabled entities slower to filter than homogeneous chunks

- **Bitmask storage** - additional bitmask per enableable component type reduces available entity capacity

- **Not a replacement for tags** - if component never has data, consider zero-size IComponentData instead

#### Best use
- **Temporary state flags** - stunned, frozen, invulnerable, invisible states that toggle frequently

- **Conditional processing** - enable/disable to include/exclude entities from systems without removing component

- **Activity toggles** - AI active/inactive, physics enabled/disabled, rendering visible/invisible

- **State machines** - current state enabled, other states disabled on same entity

#### Avoid if
- **Permanent states** - if state doesn't toggle often, use regular components with add/remove

- **Large data components** - memory stays allocated when disabled, wasteful for big components

- **All-or-nothing per chunk** - if all entities in chunk always have same enabled state, regular components work better

- **Simple boolean flag** - for rare checks, storing bool in regular component may be simpler

#### Extra tip
- **API methods:**
  ```csharp
  // EntityManager
  bool isEnabled = em.IsComponentEnabled<T>(entity);
  em.SetComponentEnabled<T>(entity, true);

  // ComponentLookup
  var lookup = SystemAPI.GetComponentLookup<T>(isReadOnly: false);
  lookup.SetComponentEnabled(entity, true);

  // EntityCommandBuffer
  ecb.SetComponentEnabled<T>(entity, true);
  ```

- **Query filter options:**
  - `WithAll<T>()` - only ENABLED components (default)
  - `WithNone<T>()` - excludes ENABLED components
  - `WithDisabled<T>()` - only DISABLED components
  - `WithPresent<T>()` - both enabled AND disabled

- **Default enabled state** - adding component starts ENABLED, set disabled after adding if needed

- **Performance characteristics** - all enabled or all disabled in chunk is fast, mixed state slower

- **Zero-size pattern (recommended):**
  ```csharp
  public struct Stunned : IComponentData, IEnableableComponent {}
  // Only uses 1 bit per entity, no data memory
  ```

- **With data:**
  ```csharp
  public struct Shield : IComponentData, IEnableableComponent
  {
      public float Strength;
  }
  // When disabled: data memory still allocated but ignored
  ```

- **State machine pattern:**
  ```csharp
  public struct IdleState : IComponentData, IEnableableComponent {}
  public struct WalkState : IComponentData, IEnableableComponent {}
  // Entity has all states, only one enabled at a time
  ```

- **IJobChunk support** - use `useEnabledMask` and `chunkEnabledMask` parameters with `ChunkEntityEnumerator` to iterate only enabled entities

