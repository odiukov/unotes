# ECS Design Decisions

A practical guide to architectural decisions when building games with ECS. Based on production experience and common design challenges.

## Core Design Questions

Every ECS architecture faces four fundamental decisions:

1. **Data distribution**: Which [[Component]] should hold specific data?
2. **Logic placement**: Should processing occur in [[ISystem|Systems]], scripts, or components?
3. **Integration**: How to combine ECS with existing OOP codebases?
4. **Refactoring**: When and how to restructure as understanding evolves?

---

## Component Design Anti-Patterns

### Monolithic Components

**Problem**: Combining multiple concerns into one component creates maintainability issues.

```csharp
// ❌ Bad: Everything bundled together
public struct CharacterComponent : IComponentData
{
    // Control data
    public float controlSpeed;
    public float controlAcceleration;

    // Platformer data
    public float platformGravity;
    public float platformFriction;

    // Jump data
    public float jumpForce;
    public int jumpMaxCount;
}
```

**Issues**:
- Variables with prefixes indicate mixed concerns
- Can't reuse jump logic without dragging control/platform data
- Systems querying this component get unnecessary data
- [[Archetype]] bloat when entities need only subset of features

**Solution**: Separate into focused components:

```csharp
// ✅ Good: Single responsibility
public struct ControlComponent : IComponentData
{
    public float Speed;
    public float Acceleration;
}

public struct PlatformerComponent : IComponentData
{
    public float Gravity;
    public float Friction;
}

public struct JumpComponent : IComponentData
{
    public float Force;
    public int MaxCount;
    public int CurrentCount;
}
```

**Benefits**:
- Entities selectively include relevant functionality
- Systems query only required components
- Better [[Cache-friendly]] data access
- Reduced [[Archetype]] fragmentation

---

## Component Design Patterns

### Tag Components Pattern

Empty components serve as inclusion/exclusion markers without data overhead.

```csharp
// Inclusion marker
public struct EnemyTag : IComponentData { }

// Exclusion marker
public struct DisabledTag : IComponentData { }
```

**Use cases**:
- **Filtering**: Systems use tags in queries to process specific entity subsets
- **State flags**: Add/remove tags instead of boolean fields (avoids [[Structural changes]] overhead when state changes frequently)
- **Debug markers**: Temporary tags for profiling or visualization

```csharp
// System processes only enabled enemies
partial struct EnemyAISystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, ai) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<AIComponent>>()
            .WithAll<EnemyTag>()
            .WithNone<DisabledTag>())
        {
            // Process enabled enemies only
        }
    }
}
```

**See also**: [[Tag-Based Behavior Selection]], [[Disable Tag Pattern]]

---

### Command Components Pattern

Single-frame components encapsulate one-time operations requiring system-level knowledge.

```csharp
public struct SpineSkinChangeCommand : IComponentData
{
    public FixedString64Bytes SkinName;
}
```

**When to use**:
- Operation needs access to system-level data structures
- Requires specific execution ordering
- Must coordinate with multiple systems
- Direct method calls unavailable due to burst/jobs context

**Implementation**:
```csharp
// Producer: Add command
ecb.AddComponent(entity, new SpineSkinChangeCommand { SkinName = "DamagedSkin" });

// Consumer: Process and remove
foreach (var (cmd, entity) in SystemAPI.Query<RefRO<SpineSkinChangeCommand>>()
    .WithEntityAccess())
{
    ProcessSkinChange(cmd.ValueRO.SkinName);
    ecb.RemoveComponent<SpineSkinChangeCommand>(entity);
}
```

**See also**: [[Request Component Pattern]], [[Cleanup Component Pattern]]

---

### Container Components Pattern

Use container structs instead of multiple component instances of same type.

```csharp
// ❌ Bad: Can't add multiple components of same type
// entity.AddComponent<AbilityComponent>(); // Want 3 abilities? Can't!

// ✅ Good: Container holds collection
public struct AbilitiesComponent : IComponentData
{
    public BlobAssetReference<BlobArray<Ability>> Abilities;
}

// Or with dynamic buffers:
public struct AbilityElement : IBufferElementData
{
    public int AbilityID;
    public float Cooldown;
}
```

**Benefits**:
- Query efficiency maintained (one component vs many)
- No special-case handling
- Natural iteration over collection
- Better memory locality

**See also**: [[IBufferElementData (dynamic buffers)]], [[BlobAsset (immutable data)]]

---

## System Architecture Patterns

### Focused System Queries

**Problem**: Broad system queries create processing inefficiencies.

```csharp
// ❌ Bad: Requires all components
partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Only processes entities with ALL components
        foreach (var (transform, velocity, gravity, friction) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>,
                           RefRO<Gravity>, RefRO<Friction>>())
        {
            // Entities without Gravity or Friction are excluded entirely
        }
    }
}
```

**Solution**: Decompose into focused systems:

```csharp
// ✅ Good: Minimal component requirements
partial struct VelocitySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}

partial struct GravitySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Separate system for gravity - runs on entities with Velocity + Gravity
        foreach (var (velocity, gravity) in
            SystemAPI.Query<RefRW<Velocity>, RefRO<Gravity>>())
        {
            velocity.ValueRW.Value += gravity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}
```

**Benefits**:
- Selective processing (entities get only relevant updates)
- Better parallelization opportunities
- Clearer system dependencies
- Reduced false-positive entity inclusions

**Principle**: "Better to have more systems running small chunks of logic than fewer systems handling mixed concerns"

---

### System Ordering Dependencies

Incorrect ordering causes perception delays and bugs.

```csharp
// ❌ Problem: Heavy objects don't initialize until frame 2
// Frame 1: SpawnSystem creates entity
// Frame 2: InitializationSystem processes entity (too late!)
```

**Solutions**:

#### 1. Explicit System Ordering
```csharp
[UpdateInGroup(typeof(InitializationSystemGroup))]
[UpdateAfter(typeof(SpawnSystem))]
public partial struct InitializationSystem : ISystem { }
```

#### 2. Entity Creation Callbacks
```csharp
var entity = ecb.CreateEntity(archetype);
// Immediate initialization
ecb.AddComponent(entity, new InitializedComponent { Value = ComputeHeavyValue() });
```

#### 3. Lazy Initialization
```csharp
public struct HeavyDataComponent : IComponentData
{
    public bool IsInitialized;
    public float ExpensiveValue;
}

// Defer until needed
if (!data.IsInitialized)
{
    data.ExpensiveValue = ComputeHeavyValue();
    data.IsInitialized = true;
}
```

#### 4. Multiple Update Phases
```csharp
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
public partial struct PhysicsSystem : ISystem { }

[UpdateInGroup(typeof(SimulationSystemGroup))]
public partial struct GameplaySystem : ISystem { }

[UpdateInGroup(typeof(PresentationSystemGroup))]
public partial struct RenderSystem : ISystem { }
```

**See also**: [[UpdateInGroup, UpdateBefore, UpdateAfter]], [[System Group]], [[System Dependencies]]

---

### One-Frame Delays vs Immediate Execution

**Guideline**: Not all logic belongs in systems.

```csharp
// ✅ Good: Immediate state transitions
public void OnTriggerEntered(Entity other)
{
    // Change state immediately
    currentState = State.Attacking;

    // Store event for system processing
    ecb.AddComponent(entity, new StateChangedEvent
    {
        OldState = State.Idle,
        NewState = State.Attacking
    });
}

// System processes events later for callbacks/reactions
partial struct StateEventSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (evt, entity) in SystemAPI.Query<RefRO<StateChangedEvent>>()
            .WithEntityAccess())
        {
            NotifyScriptsOfStateChange(evt.ValueRO);
            ecb.RemoveComponent<StateChangedEvent>(entity);
        }
    }
}
```

**Principle**: Simple operations (vector addition, state transitions) can execute instantly. Defer to systems only when coordination, ordering, or system-level knowledge required.

---

## Integration with OOP Codebases

### Gradual Migration Pattern

Don't refactor everything at once. Embed existing classes initially:

```csharp
// ✅ Good: Incremental transition
public struct FormationComponent : IComponentData
{
    public BlobAssetReference<FormationData> Formation; // Converted to blob
}

public struct BehaviorComponent : IComponentData
{
    public ManagedBehavior Behavior; // Still OOP class (managed component)
}
```

**Benefits**:
- Reuse existing logic without rewrites
- Identify ECS boundaries incrementally
- Transition step-by-step as systems stabilize
- Validate architecture before committing

**Note**: Managed components break [[Burst]] compilation and reduce performance. Migrate to unmanaged data over time.

**See also**: [[Managed IComponentData (class components)]]

---

### Entity Abstractions for Complex Concepts

Abstract concepts (formations, squads, teams) benefit from becoming entities.

```csharp
// ❌ Old: Formation as data in members
public struct UnitComponent : IComponentData
{
    public int FormationID; // Weak reference
}

// ✅ Better: Formation as entity
public struct FormationTag : IComponentData { }
public struct FormationMemberComponent : IComponentData
{
    public Entity FormationEntity; // Strong entity reference
}

// Query all units in a formation
foreach (var member in SystemAPI.Query<RefRO<FormationMemberComponent>>())
{
    if (member.ValueRO.FormationEntity == targetFormation)
    {
        // Process unit
    }
}

// Formation entity can have its own components
ecb.AddComponent(formationEntity, new DebugVisualizationComponent());
ecb.AddComponent(formationEntity, new FormationConfigComponent { Shape = Shape.Wedge });
```

**Benefits**:
- Composition flexibility (add debug/visual components to formations)
- Natural identity management (formation auto-destruction cascades to members)
- Proper query semantics
- Fits ECS paradigm naturally

**See also**: [[Entity Factory with Archetype]], [[Helper Methods for Component Sets]]

---

## Decision Framework: Script vs System

When uncertain about logic placement, use this workflow:

### 1. Start with Scripts
Write unproven logic in MonoBehaviour/scripts first.

```csharp
// Initial prototype
public class UnitController : MonoBehaviour
{
    void Update()
    {
        // Test gameplay logic here
    }
}
```

**Why**: Avoids premature architecture decisions, faster iteration.

### 2. Extract Data to Components
When multiple scripts/systems need access, move data to components.

```csharp
public struct UnitStatsComponent : IComponentData
{
    public float Speed;
    public float Health;
}
```

### 3. Move Logic to Systems
When reusability becomes evident, migrate processing to systems.

```csharp
partial struct UnitMovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, stats) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<UnitStatsComponent>>())
        {
            // Shared movement logic
        }
    }
}
```

### 4. Remove Unused Code
Delete obsolete scripts/systems throughout development.

**Principle**: Treat initial architecture as provisional. Optimize for refactorability, not perfect first decisions.

---

## Refactoring Code Smells

ECS exhibits similar refactoring needs as OOP but requires different solutions:

| Code Smell | Signal | ECS Solution |
|------------|--------|--------------|
| **Variable prefixes** | `controlSpeed`, `platformGravity` in same component | Separate into focused components |
| **Monolithic systems** | Single system handling many responsibilities | Extract logic into separate systems |
| **Inflexible filtering** | Boolean flags to disable features | [[Tag Component]]s or [[IEnableableComponent (toggleable components)]] |
| **Broad queries** | System queries many components but uses few | Split into focused systems with minimal queries |
| **Shared mutable state** | Multiple components modifying same data | Entity-based abstraction or singleton pattern |

---

## Best Practices Summary

### Component Design
- ✅ Minimize component dependencies per system query
- ✅ Separate data concerns instead of boolean flags
- ✅ Use component presence as primary control mechanism
- ✅ Prefer containers over multiple instances of same type
- ✅ Allow helper methods in components for readonly operations

### System Architecture
- ✅ Make system order explicit and testable
- ✅ Query minimum number of components needed
- ✅ Decompose into focused systems for parallelization
- ✅ Defer only when coordination/ordering required
- ✅ Use multiple update phases (Fixed/Simulation/Presentation)

### Integration & Migration
- ✅ Gradual migration preserves existing logic
- ✅ Entity abstractions for complex concepts
- ✅ Start with scripts, extract to systems when proven
- ✅ Remove unused code continuously

---

## Related Patterns

- [[Request Component Pattern]] - Command component variation
- [[Cleanup Component Pattern]] - Lifecycle management
- [[Tag-Based Behavior Selection]] - Tag component usage
- [[Disable Tag Pattern]] - Exclusion filtering
- [[State Machine Architecture]] - State management in ECS
- [[Auto-Add System Pattern]] - Component initialization
- [[Entity Factory with Archetype]] - Entity creation optimization

---

## Further Reading

**Source**: [Design Decisions When Building Games Using ECS](https://arielcoppes.dev/2023/07/13/design-decisions-when-building-games-using-ecs.html) by Ariel Coppes
