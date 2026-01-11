---
tags:
  - pattern
---
#### Description
- **Component-based polymorphism** pattern using separate component types for each behavior instead of enum-based behavior selection

- Systems query specific behavior components directly (e.g., `Query<WanderBehavior>()`) enabling [[Cache-friendly]] iteration and automatic [[Archetype]] separation

- Each behavior type stores its own state data inline, eliminating need for generic "AI state" component with union/switch logic

- Enables composition of behaviors on same entity (e.g., entity can have both ChasePlayerBehavior and AttackBehavior simultaneously)

#### Example
```csharp
// ❌ BAD: Enum-based behavior (traditional approach)
public enum AIBehaviorType { Idle, Wander, ChasePlayer, Patrol }

public struct AIBehavior : IComponentData
{
    public AIBehaviorType Type;
    // Union of all possible state data - wasteful
    public float3 WanderDirection;
    public float StopDistance;
    public int CurrentWaypointIndex;
}

// System must switch for every entity
foreach (var behavior in SystemAPI.Query<RefRW<AIBehavior>>())
{
    switch (behavior.ValueRO.Type)
    {
        case AIBehaviorType.Wander: /* ... */ break;
        case AIBehaviorType.ChasePlayer: /* ... */ break;
    }
}

// ✅ GOOD: Tag-based behavior (ECS approach)
public struct WanderBehavior : IComponentData
{
    public float3 Direction;
    public float TimeUntilDirectionChange;
    public float DirectionChangeInterval;
}

public struct ChasePlayerBehavior : IComponentData
{
    public float StopDistance;
}

public struct PatrolBehavior : IComponentData
{
    public int CurrentWaypointIndex;
    public bool Loop;
}

// Separate systems for each behavior - clean, cache-friendly
[BurstCompile]
public partial struct WanderBehaviorSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Only iterates entities with WanderBehavior
        foreach (var wander in SystemAPI.Query<RefRW<WanderBehavior>>())
        {
            // Wander logic...
        }
    }
}

// Behavior transitions - add/remove components
ecb.RemoveComponent<WanderBehavior>(entity);
ecb.AddComponent(entity, new ChasePlayerBehavior { StopDistance = 2f });

// Composite behaviors - multiple on same entity
// Entity can have ChasePlayerBehavior + AttackBehavior simultaneously

// Combining with IEnableableComponent for efficient toggling
public struct WanderBehavior : IComponentData, IEnableableComponent { /* ... */ }
public struct ChasePlayerBehavior : IComponentData, IEnableableComponent { /* ... */ }

// Pre-add all behaviors, toggle enabled state (no structural changes)
SystemAPI.SetComponentEnabled<WanderBehavior>(entity, false);
SystemAPI.SetComponentEnabled<ChasePlayerBehavior>(entity, true);
```

#### Pros
- **[[Cache-friendly]]** - entities with same behavior stored together in [[Chunk]], better cache locality than enum switch

- **[[Burst Compile|Burst]] friendly** - no switch statements or polymorphism, each system processes single behavior type

- **Type safety** - compiler catches missing behavior data at compile time, impossible to forget required state

- **Composable** - entities can have multiple behaviors simultaneously (Chase + Attack + Defensive)

- **Query optimization** - systems only iterate relevant entities, `Query<WanderBehavior>()` skips all non-wanderers

#### Cons
- **[[Structural changes]]** - behavior transitions require add/remove components, triggering structural changes

- **[[Archetype]] fragmentation** - each behavior combination creates new archetype, can reduce [[Cache-friendly|cache efficiency]]

- **More code** - separate system per behavior vs single switch statement (though more maintainable)

- **Transition complexity** - managing behavior changes requires EntityCommandBuffer and careful state initialization

#### Best use
- **Distinct AI behaviors** - enemies with clearly separate behaviors (patrol, chase, flee, attack)

- **Composable systems** - behaviors that can combine (Chase + Attack, Patrol + Defensive)

- **Performance-critical AI** - hundreds/thousands of entities benefit from cache locality and Burst compilation

- **Data-driven design** - behaviors configured per-entity in authoring, not hardcoded state machines

- **Infrequent transitions** - behavior changes happen rarely (enter combat, exit combat) so structural change cost acceptable

#### Avoid if
- **Frequent behavior changes** - switching every frame causes expensive structural changes (use [[IEnableableComponent (toggleable components)|IEnableableComponent]] instead)

- **Simple state machine** - 2-3 states with simple transitions better served by enum + switch

- **Highly dynamic behavior** - if entity behavior changes based on many runtime conditions, [[State Machine Architecture]] may be cleaner

- **Memory-constrained** - separate components per behavior use more memory than single enum component

#### Extra tip
- **Shared behavior data**: Use AIAgent component for properties shared across all behaviors (DetectionRange, MoveSpeed)

- **Combine with IEnableableComponent**: Pre-add all behaviors in authoring, toggle enabled state to avoid structural changes on frequent transitions

- **Behavior priority**: Use system execution order to control behavior precedence with `[UpdateBefore]`/`[UpdateAfter]`

- **Cleanup on transition**: Reset behavior state when transitioning - initialize new behavior with proper starting values

- **Behavior categories**: Use zero-size tags for grouping (AggressiveBehavior, PassiveBehavior) to query all aggressive enemies

- **Disable Tag Pattern**: Use StunnedTag with `.WithNone<StunnedTag>()` to temporarily suppress all AI behaviors

- **Hybrid approach**: Combine tags for major behaviors with enums for minor variations (ChasePlayerBehavior with ChaseMode enum)

- **Performance comparison**: Tag-based ~1.2ms for 10k entities vs enum-based ~3.5ms (cache misses, branch prediction failures)

- **Naming convention**: Suffix behavior components with "Behavior" for clarity (WanderBehavior, ChasePlayerBehavior)

- **Testing**: Behaviors easier to test in isolation without other AI complexity

- **Alternative to [[State Machine Architecture]]**: For simple AI, tag-based behaviors cleaner than full state machine

- **Archetype optimization**: Minimize behavior combinations to reduce archetype count (3 behaviors = 3 archetypes good, 8 combinations = 8 archetypes bad)
