---
tags:
  - component
  - advanced
---

#### Description
- **Toggleable components** implementing both [[IComponentData]] and IEnableableComponent - can be enabled/disabled at runtime without [[Structural changes]] (no archetype change)

- **Per-entity bitmask** stored in [[Chunk]] tracks enabled/disabled state for each entity - component memory always allocated but can be logically "turned off"

- **Query filtering** supports special filters (WithAll, WithNone, WithDisabled, WithPresent) to process enabled/disabled entities differently

- **Zero-size optimization** - empty enableable components (no fields) are ideal for boolean flags without memory overhead beyond the enable bit

#### Example
```csharp
using Unity.Entities;
using Unity.Mathematics;

// Define enableable component (usually zero-size for flags)
public struct Stunned : IComponentData, IEnableableComponent {}

public struct Frozen : IComponentData, IEnableableComponent
{
    public float Duration;  // Can have data like regular IComponentData
}

// Toggling enabled state
[BurstCompile]
public partial struct StunSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        EntityManager em = state.EntityManager;

        foreach (var (stunned, entity) in
            SystemAPI.Query<RefRO<Stunned>>()
                .WithEntityAccess())
        {
            // Check if component is enabled
            if (em.IsComponentEnabled<Stunned>(entity))
            {
                // Disable after 3 seconds
                if (SystemAPI.Time.ElapsedTime > 3.0)
                {
                    em.SetComponentEnabled<Stunned>(entity, false);
                }
            }
        }
    }
}

// Query filtering by enabled/disabled state
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // Process only entities with ENABLED Stunned component
        foreach (var (transform, entity) in
            SystemAPI.Query<RefRW<LocalTransform>>()
                .WithAll<Stunned>()  // Only processes enabled Stunned
                .WithEntityAccess())
        {
            // These entities are stunned - don't move
            // WithAll<Stunned> filters for enabled components by default
        }

        // Process entities WITHOUT enabled Stunned component
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>()
                .WithNone<Stunned>())  // Excludes enabled Stunned
        {
            // These entities can move normally
            transform.ValueRW.Position += velocity.ValueRO.Value * deltaTime;
        }
    }
}

// Using WithDisabled and WithPresent
[BurstCompile]
public partial struct RecoverySystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Process entities with DISABLED Stunned component
        foreach (var entity in
            SystemAPI.Query<RefRO<Entity>>()
                .WithDisabled<Stunned>())  // Only disabled Stunned
        {
            // Entity was stunned but recovered
            // Could remove component here
        }

        // Process entities with Stunned component (enabled OR disabled)
        foreach (var (entity, stunned) in
            SystemAPI.Query<RefRO<Entity>, RefRO<Stunned>>()
                .WithPresent<Stunned>())  // Both enabled and disabled
        {
            // Process all entities that have Stunned component
            // regardless of enabled state
        }
    }
}

// EntityCommandBuffer support
[BurstCompile]
public partial struct ApplyStunJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;

    private void Execute(Entity entity, [ChunkIndexInQuery] int sortKey, in Health health)
    {
        if (health.Current < 10)
        {
            // Add enableable component (starts enabled by default)
            Ecb.AddComponent<Stunned>(sortKey, entity);

            // Or add as disabled
            Ecb.AddComponent(sortKey, entity, new Stunned());
            Ecb.SetComponentEnabled<Stunned>(sortKey, entity, false);
        }
    }
}

// ComponentLookup with enableable components
[BurstCompile]
public partial struct LookupSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var stunnedLookup = SystemAPI.GetComponentLookup<Stunned>(isReadOnly: false);

        foreach (var (attacker, target) in
            SystemAPI.Query<RefRO<Attacker>, RefRO<Target>>())
        {
            Entity targetEntity = target.ValueRO.Value;

            // Check if target has enabled Stunned component
            if (stunnedLookup.IsComponentEnabled(targetEntity))
            {
                // Target is stunned - bonus damage
            }

            // Enable Stunned on target
            if (stunnedLookup.HasComponent(targetEntity))
            {
                stunnedLookup.SetComponentEnabled(targetEntity, true);
            }
        }
    }
}

// IJobChunk with enableable components
[BurstCompile]
public partial struct ProcessEnabledJob : IJobChunk
{
    public ComponentTypeHandle<Stunned> StunnedHandle;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                       bool useEnabledMask, in v128 chunkEnabledMask)
    {
        // Check if we need to filter by enabled state
        if (!useEnabledMask)
        {
            // All components in chunk are enabled, process normally
            for (int i = 0; i < chunk.Count; i++)
            {
                // Process all entities
            }
        }
        else
        {
            // Some components disabled, use enumerator
            var enumerator = new ChunkEntityEnumerator(useEnabledMask, chunkEnabledMask, chunk.Count);
            while (enumerator.NextEntityIndex(out int i))
            {
                // Only processes entities with enabled Stunned
            }
        }
    }
}
```

#### Pros
- **No structural changes** - toggling enabled/disabled doesn't move entity between chunks, avoiding expensive archetype changes

- **Query filtering** - can efficiently filter enabled vs disabled entities with WithAll, WithNone, WithDisabled, WithPresent

- **Zero-size flags** - empty enableable components use minimal memory (just enable bit), perfect for boolean state

- **Component data preserved** - disabling component keeps data intact, re-enabling doesn't require setting data again

#### Cons
- **Memory always allocated** - disabled components still consume chunk memory, not removed until component removed

- **Mixed-state overhead** - chunks with both enabled and disabled entities are slower to filter than homogeneous chunks

- **Bitmask storage** - additional bitmask per enableable component type in chunk reduces available entity capacity

- **Not a replacement for tags** - if component never has data, consider zero-size IComponentData instead of enableable component

#### Best use
- **Temporary state flags** - stunned, frozen, invulnerable, invisible states that toggle frequently

- **Conditional processing** - enable/disable to include/exclude entities from specific systems without removing component

- **Activity toggles** - AI active/inactive, physics enabled/disabled, rendering visible/invisible

- **State machines** - current state enabled, other states disabled on same entity

#### Avoid if
- **Permanent states** - if state doesn't toggle often, use regular components with add/remove instead

- **Large data components** - memory stays allocated when disabled, wasteful for big components

- **All-or-nothing per chunk** - if all entities in chunk always have same enabled state, regular components with query filters work better

- **Simple boolean flag** - for rare checks, storing bool in regular component may be simpler than enableable component

#### Extra tip
- **API methods:**
  ```csharp
  // EntityManager methods
  bool isEnabled = entityManager.IsComponentEnabled<T>(entity);
  entityManager.SetComponentEnabled<T>(entity, true);  // Enable
  entityManager.SetComponentEnabled<T>(entity, false); // Disable

  // ComponentLookup methods
  var lookup = SystemAPI.GetComponentLookup<T>(isReadOnly: false);
  bool isEnabled = lookup.IsComponentEnabled(entity);
  lookup.SetComponentEnabled(entity, true);

  // EntityCommandBuffer methods
  ecb.SetComponentEnabled<T>(entity, true);
  ecb.SetComponentEnabled<T>(sortKey, entity, false);  // ParallelWriter
  ```

- **Query filter options:**
  ```csharp
  // WithAll<T> - only ENABLED components (default behavior)
  SystemAPI.Query<RefRW<T>>().WithAll<Stunned>()

  // WithNone<T> - excludes ENABLED components
  SystemAPI.Query<RefRW<T>>().WithNone<Stunned>()

  // WithDisabled<T> - only DISABLED components
  SystemAPI.Query<RefRW<T>>().WithDisabled<Stunned>()

  // WithPresent<T> - both enabled AND disabled (any presence)
  SystemAPI.Query<RefRW<T>>().WithPresent<Stunned>()
  ```

- **Default enabled state:**
  ```csharp
  // Adding component via EntityManager - starts ENABLED
  em.AddComponent<Stunned>(entity);
  bool isEnabled = em.IsComponentEnabled<Stunned>(entity);  // true

  // Adding component via ECB - starts ENABLED
  ecb.AddComponent<Stunned>(entity);

  // To add as disabled, set enabled state after adding
  em.AddComponent<Stunned>(entity);
  em.SetComponentEnabled<Stunned>(entity, false);
  ```

- **Performance characteristics:**
  ```csharp
  // Fast: All enabled or all disabled in chunk
  Chunk: [E E E E E E E E]  // All enabled - fast filtering
  Chunk: [D D D D D D D D]  // All disabled - fast filtering

  // Slower: Mixed enabled/disabled in chunk
  Chunk: [E D E E D E D E]  // Mixed - must check each entity's bit

  // Avoid frequent toggling that creates mixed-state chunks
  ```

- **Zero-size enableable components (recommended pattern):**
  ```csharp
  // Empty struct - only uses enable bit, no data memory
  public struct Stunned : IComponentData, IEnableableComponent {}
  public struct Frozen : IComponentData, IEnableableComponent {}
  public struct Invisible : IComponentData, IEnableableComponent {}

  // Size overhead: just 1 bit per entity in bitmask
  // No component data memory allocated
  ```

- **Enableable components with data:**
  ```csharp
  // Can have fields like regular IComponentData
  public struct Shield : IComponentData, IEnableableComponent
  {
      public float Strength;
      public float RechargeRate;
  }

  // When disabled: data memory still allocated but logically ignored
  // When enabled: data accessible and processed

  // Use case: temporary buffs with expiration
  em.SetComponentEnabled<Shield>(entity, true);
  em.SetComponentData(entity, new Shield { Strength = 100, RechargeRate = 5 });

  // Later: disable instead of removing (keeps data for re-enable)
  em.SetComponentEnabled<Shield>(entity, false);
  ```

- **Combining filters:**
  ```csharp
  // Entities with enabled Health but disabled Shield
  foreach (var entity in
      SystemAPI.Query<RefRO<Entity>>()
          .WithAll<Health>()      // Health enabled
          .WithDisabled<Shield>())  // Shield disabled
  {
      // Process...
  }

  // Entities with Health (any state) and disabled Shield
  foreach (var entity in
      SystemAPI.Query<RefRO<Entity>>()
          .WithPresent<Health>()   // Health enabled or disabled
          .WithDisabled<Shield>())  // Shield disabled
  {
      // Process...
  }
  ```

- **EntityQuery creation with filters:**
  ```csharp
  public void OnCreate(ref SystemState state)
  {
      // Query for enabled components
      var enabledQuery = state.GetEntityQuery(
          ComponentType.ReadOnly<Stunned>()  // Enabled by default
      );

      // Query for disabled components
      var disabledQuery = new EntityQueryBuilder(Allocator.Temp)
          .WithDisabled<Stunned>()
          .Build(ref state);

      // Query for present components (enabled or disabled)
      var presentQuery = new EntityQueryBuilder(Allocator.Temp)
          .WithPresent<Stunned>()
          .Build(ref state);
  }
  ```

- **Use cases by pattern:**
  ```csharp
  // State machine states (only one enabled at a time)
  public struct IdleState : IComponentData, IEnableableComponent {}
  public struct WalkState : IComponentData, IEnableableComponent {}
  public struct RunState : IComponentData, IEnableableComponent {}

  // Entity has all three components, only one enabled
  em.SetComponentEnabled<IdleState>(entity, true);
  em.SetComponentEnabled<WalkState>(entity, false);
  em.SetComponentEnabled<RunState>(entity, false);

  // Temporary effects with duration
  public struct Burning : IComponentData, IEnableableComponent
  {
      public float RemainingTime;
  }

  // Enable when applied, disable when expired (instead of removing)
  ```

- **IJobChunk filtering:**
  ```csharp
  // Execute receives useEnabledMask and chunkEnabledMask
  public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                     bool useEnabledMask, in v128 chunkEnabledMask)
  {
      // If useEnabledMask is false, all components in chunk are enabled
      // If true, use ChunkEntityEnumerator to skip disabled entities

      if (useEnabledMask)
      {
          var enumerator = new ChunkEntityEnumerator(
              useEnabledMask, chunkEnabledMask, chunk.Count);

          while (enumerator.NextEntityIndex(out int i))
          {
              // Only processes enabled entities
          }
      }
      else
      {
          for (int i = 0; i < chunk.Count; i++)
          {
              // All enabled, process normally
          }
      }
  }
  ```

## See Also

- [[IComponentData]] - Base component interface
- [[Chunk]] - 16KiB memory blocks with enable bitmasks
- [[EntityQuery]] - Entity filtering with enabled/disabled support
- [[EntityCommandBuffer]] - Deferred component enable/disable
- [[ComponentLookup and BufferLookup]] - Random access with enabled state checks