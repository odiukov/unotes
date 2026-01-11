---
tags:
  - pattern
---
#### Description
- **Zero-cost state management** pattern using [[IEnableableComponent (toggleable components)|IEnableableComponent]] to control system execution flow without [[Structural changes]]

- Enables/disables components to gate systems and orchestrate multi-system workflows, eliminating need for bool flags

- Unlike traditional component add/remove (which triggers structural changes), enable/disable is a fast, [[Cache-friendly]] bit flag operation

- Common use cases: cooldown timers, permission gates (CanShoot), readiness flags (ReadyToCollectTargets), request flow control (BossStateChangeRequest)

#### Example
```csharp
// Permission gate - control when entity can perform action
public struct CanShoot : IComponentData, IEnableableComponent { }

// Cooldown with data
public struct Cooldown : IComponentData, IEnableableComponent
{
    public float TimeToNextShoot;
    public float ShootRate;
}

// Readiness flag - coordinate multi-system workflows
public struct ReadyToCollectTargets : IComponentData, IEnableableComponent { }

// Request flow control
public struct BossStateChangeRequest : IComponentData, IEnableableComponent { }

// Query only enabled components
SystemAPI.Query<RefRO<Weapon>>()
    .WithAll<CanShoot>()  // Only entities with CanShoot enabled

// Query only disabled components
SystemAPI.Query<RefRW<Cooldown>>()
    .WithDisabled<CanShoot>()

// Toggle without structural changes
SystemAPI.SetComponentEnabled<CanShoot>(entity, false);
ecb.SetComponentEnabled<ReadyToCollectTargets>(entity, true);
```

#### Pros
- **No [[Structural changes]]** - enable/disable is instant bit flag operation, doesn't trigger archetype changes or [[Sync points]]

- **[[Cache-friendly]] queries** - systems only iterate enabled components, skipping disabled entities improves cache utilization

- **Clean flow control** - enables declarative system gating (WithAll/WithDisabled) vs imperative bool checks in loops

- **Query optimization** - [[RequireMatchingQueriesForUpdate]] with enableable components prevents system execution when no work needed

- **Composition friendly** - can combine multiple enableable components for complex flow control (e.g., CanShoot + ReadyToFire)

#### Cons
- **Limited to existing entities** - can't use to dynamically add/remove components, entity must already have the component in its [[Archetype]]

- **Authoring overhead** - must pre-add all potentially needed IEnableableComponents during entity creation, even if initially disabled

- **Memory always allocated** - disabled components still occupy memory in [[Chunk]], unlike removed components

- **Debugging visibility** - enabled/disabled state not as visible in Entity Inspector compared to component presence/absence

#### Best use
- **Cooldown systems** - disable CanShoot during cooldown, re-enable when ready (avoids structural changes every shot)

- **Multi-stage workflows** - enable ReadyToCollect flag to signal between interval system and collection system

- **State machine control** - enable StateChangeRequest to coordinate transition evaluation and action execution systems

- **Processed tracking** - disable Processed flag for new entities, enable after processing (cleaner than destroy-on-process)

- **Permission gates** - CanMove, CanShoot, CanJump tags that systems check with WithAll<CanMove>()

#### Avoid if
- **Need to change [[Archetype]]** - if component truly needs to be added/removed (not just toggled), use structural changes with [[EntityCommandBuffer]]

- **One-time initialization** - if component is set once and never toggled, regular IComponentData is simpler

- **Very large components** - since disabled components still occupy chunk memory, large data structures waste space when disabled (use [[Request Component Pattern]] instead)

- **Entity lifetime management** - for entities that should be destroyed after processing, use DestroyEntity instead of Processed flag

#### Extra tip
- **Query syntax**: Use `.WithAll<CanShoot>()` for enabled, `.WithDisabled<CanShoot>()` for disabled, `.WithPresent<CanShoot>()` for both

- **Query optimization**: Use `[RequireMatchingQueriesForUpdate]` to skip system when no enabled components exist

- **Default state**: Components are enabled by default, disable explicitly if needed: `entityManager.SetComponentEnabled<Cooldown>(entity, false)`

- **ECB support**: Can enable/disable via ECB for deferred execution: `ecb.SetComponentEnabled<CanShoot>(entity, false)`

- **Authoring pattern**: Pre-add enableable components in Baker even if initially disabled, then control enabled state

- **Performance**: IEnableableComponent toggle ~100x faster than add/remove component (no structural change)

- **State machines**: Combine multiple enableable flags for exclusive states - enable only one at a time

- **Alternative to [[Request Component Pattern]]**: For frequent toggling operations, use IEnableableComponent instead of add/remove

- **Common pattern**: Interval system sets ReadyToCollect flag, collection system processes and clears flag - clean coordination without structural changes

- **Combining patterns**: Works well with [[Cleanup Component Pattern]] to detect when enableable component changes state
