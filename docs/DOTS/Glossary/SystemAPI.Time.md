## SystemAPI.Time

**Time context for ECS systems** - provides delta time and elapsed time values that are controlled by [[Rate Managers]] and system groups.

**Key properties:**
- **DeltaTime** - time since last update (framerate-dependent or fixed)
- **ElapsedTime** - total time elapsed since game start
- Values depend on system group's rate manager configuration

**How it works:**
- Each system group "pushes" its own time context
- Systems see time values from their parent group's rate manager
- Different groups can have different time perception simultaneously

**Example:**
```csharp
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
public partial struct PhysicsSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Always 0.0166f (60 Hz fixed) regardless of actual framerate
        float fixedDelta = SystemAPI.Time.DeltaTime;

        // Move with fixed timestep for deterministic physics
        velocity.y -= 9.81f * fixedDelta;
    }
}

[UpdateInGroup(typeof(SimulationSystemGroup))]
public partial struct AnimationSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Actual frame delta (varies with framerate)
        float frameDelta = SystemAPI.Time.DeltaTime;

        // Smooth animation that matches framerate
        alpha += speed * frameDelta;
    }
}
```

**Comparison:**

| System Group | DeltaTime Behavior | Use Case |
|--------------|-------------------|----------|
| SimulationSystemGroup | Frame delta (varies) | Smooth visual updates |
| FixedStepSimulationSystemGroup | Fixed (0.0166f default) | Deterministic physics |
| VariableRateSimulationSystemGroup | Real time (unscaled) | Background tasks |

**Important:**
- `SystemAPI.Time` ≠ `UnityEngine.Time` - they can have different values
- Each rate manager controls what time values systems perceive
- Fixed rate systems see fixed DeltaTime even when framerate varies
- Variable rate systems see real unscaled time

**See also:** [[Rate Managers]], [[ISystem]], [[UpdateInGroup, UpdateBefore, UpdateAfter]]
