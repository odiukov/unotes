---
tags:
  - pattern
---
#### Description
- **Data-driven state machine** pattern combining [[IBufferElementData (dynamic buffers)|buffers]], managed components, and registry pattern for complex AI behavior without enum-based state machines

- Separates state transition logic (condition evaluation) from state action execution (multi-frame operations) into distinct systems for clean separation of concerns

- Uses **Condition Evaluators** (registry pattern) for pluggable condition types and **Action Executors** (managed components) for multi-frame actions

- Supports multiple transition modes: Sequential (cycle through states), ConditionalRandom (weighted selection from valid states)

#### Example
```csharp
// State components
public struct BossCurrentState : IComponentData
{
    public int Value;  // Current state index
}

public struct BossConfig : IComponentData
{
    public BossTransitionMode TransitionMode;  // Sequential or ConditionalRandom
}

// State definitions in buffers
[InternalBufferCapacity(5)]
public struct BossStateSetup : IBufferElementData
{
    public float Weight;  // For weighted random selection
}

// Condition definitions
[InternalBufferCapacity(10)]
public struct BossStateConditionSetup : IBufferElementData
{
    public BossConditionType Type;      // DistanceToPlayer, BossHealth
    public BossConditionEvaluationType EvalType;  // Greater, Less, Equal
    public float Parameter;
    public int StateIndex;
}

// Flow control between systems
public struct BossStateChangeRequest : IComponentData, IEnableableComponent { }

// Condition evaluator registry
public interface IBossConditionEvaluator
{
    bool Evaluate(ref BossSensorData sensorData,
        BossConditionEvaluationType evalType, float parameter);
}

public static class BossConditionRegistry
{
    private static readonly IBossConditionEvaluator[] _evaluators = new IBossConditionEvaluator[3];

    static BossConditionRegistry()
    {
        _evaluators[(int)BossConditionType.DistanceToPlayer] = new PlayerDistanceEvaluator();
        _evaluators[(int)BossConditionType.BossHealth] = new HealthConditionEvaluator();
    }
}

// Action executor for multi-frame operations
[System.Serializable]
public abstract class IBossActionExecutor
{
    public bool Completed { get; set; }
    public abstract void Execute(Entity bossEntity, float deltaTime, ref BossSensorData sensorData);
}

// Example: Delay action takes multiple frames
public class DelayActionExecutor : IBossActionExecutor
{
    private float _duration;
    private float _timeElapsed;

    public override void Execute(Entity bossEntity, float deltaTime, ref BossSensorData sensorData)
    {
        _timeElapsed += deltaTime;
        if (_timeElapsed >= _duration)
            Completed = true;
    }
}

// Managed component stores executors
public class BossStateExecutors : IComponentData
{
    public List<List<IBossActionExecutor>> States = new();
}
```

**System Flow:**
1. **Sensor systems** collect data (player distance, health) into `BossSensorData`
2. **Action system** executes current state actions until complete, then enables `BossStateChangeRequest`
3. **Transition system** evaluates conditions, selects next state, disables `BossStateChangeRequest`

#### Pros
- **Data-driven flexibility** - states, conditions, and transitions configured via [[IBufferElementData (dynamic buffers)|buffers]], not hardcoded

- **Extensible architecture** - add new condition types by implementing interface and registering in registry, no system changes needed

- **Multi-frame actions** - managed component executors can span multiple frames (delays, animations, complex sequences)

- **Clean separation** - transition logic isolated from action execution, each system has single responsibility

- **Weighted randomness** - ConditionalRandom mode supports weighted state selection for varied AI behavior

#### Cons
- **Managed components** - BossStateExecutors uses managed List, breaks [[Burst Compile|Burst]] compatibility and adds GC pressure

- **Complexity** - requires understanding of registry pattern, managed components, and multi-system coordination

- **[[Structural changes]]** - adding conditions via buffers can trigger structural changes if buffer exceeds [[InternalBufferCapacity]]

- **Debugging difficulty** - state machine flow spread across multiple systems, conditions, and executors harder to trace

- **Memory overhead** - buffers, managed components, and evaluator registry consume more memory than simple enum state machine

#### Best use
- **Complex boss AI** - multi-phase bosses with conditional state transitions based on health, distance, or other metrics

- **Dynamic behavior trees** - configurable AI behavior that changes based on runtime conditions without code changes

- **Data-driven game design** - designers can configure states and conditions in editor without programmer intervention

- **Multi-frame sequences** - actions that span multiple frames (play animation, wait, spawn enemies, repeat)

- **Pluggable systems** - registry pattern allows adding new condition/action types without modifying existing systems

#### Avoid if
- **Simple state machines** - 2-3 states with straightforward transitions better served by enum + switch statement

- **Burst-critical code** - managed components prevent Burst compilation, use unmanaged alternatives for performance-critical AI

- **Memory-constrained** - buffers and managed components add significant overhead vs simple state enum

- **Single-frame actions** - if all actions complete in one frame, complexity not justified (use [[Tag-Based Behavior Selection]] instead)

#### Extra tip
- **Sensor data pattern**: Collect all required data (player position, health) in separate sensor systems before transition system runs

- **[[IEnableableComponent (toggleable components)|IEnableableComponent]] for flow control**: Use BossStateChangeRequest to coordinate between transition and action systems without structural changes

- **Buffer capacity hints**: Set `[InternalBufferCapacity(5)]` for states, `[InternalBufferCapacity(10)]` for conditions to avoid heap allocation

- **Registry pattern performance**: Static array lookup by enum index is O(1), faster than Dictionary

- **Managed component alternatives**: For Burst compatibility, use NativeList or blob assets instead of managed List

- **Transition modes**: Sequential cycles through states in order, ConditionalRandom picks from valid states based on weights

- **Condition logic**: Current implementation uses AND (all conditions must be met), can change to OR (any condition met)

- **State cooldowns**: Prevent rapid state changes by adding cooldown component after transition

- **Debugging**: Add logging to track state transitions: `Debug.Log($"Boss: {currentState} -> {nextState}")`

- **Authoring workflow**: Create MonoBehaviour authoring with ScriptableObject configs, bake into buffers

- **Burst alternatives**: For performance-critical AI, use [[Tag-Based Behavior Selection]] with IEnableableComponent states instead of managed components

- **Combining patterns**: Can integrate with [[Entity Factory with Archetype]] to spawn entities from action executors

- **Visualization**: Create Unity Editor custom inspector to visualize state graph with conditions for designers
