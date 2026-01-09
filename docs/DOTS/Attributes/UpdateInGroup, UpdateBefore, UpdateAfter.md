---
tags:
  - attribute
---
#### Description
- **System ordering attributes** that control when systems execute relative to other systems in the frame update loop

- `[UpdateInGroup(typeof(GroupType))]` places a system inside a specific system group (InitializationSystemGroup, SimulationSystemGroup, PresentationSystemGroup, etc.)

- `[UpdateBefore(typeof(SystemType))]` and `[UpdateAfter(typeof(SystemType))]` create execution order constraints relative to other systems

- Essential for **deterministic execution order** - physics before movement, movement before rendering, spawning before AI processing, etc.

#### Example
```csharp
// Update in FixedStepSimulationSystemGroup (fixed timestep physics)
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
[BurstCompile]
public partial struct SpawnerSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs at fixed timestep (default 60 Hz)
    }
}

// Update after TransformSystemGroup to get final positions
[UpdateAfter(typeof(TransformSystemGroup))]
[BurstCompile]
public partial struct ProjectileMoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs after all transforms are computed
    }
}

// Update before TowerPlacementSystem
[UpdateBefore(typeof(TowerPlacementSystem))]
[BurstCompile]
public partial struct InputSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Input processed before placement logic
    }
}

// Physics trigger processing after physics simulation
[UpdateInGroup(typeof(AfterPhysicsSystemGroup))]
[BurstCompile]
public partial struct EnemyPlayerCollisionSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs after physics step completes
    }
}

// Combining multiple attributes
[UpdateInGroup(typeof(SimulationSystemGroup))]
[UpdateAfter(typeof(MovementSystem))]
[UpdateBefore(typeof(AnimationSystem))]
[BurstCompile]
public partial struct CombatSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs after movement, before animation, within simulation group
    }
}
```

#### Pros
- **Deterministic order** - guarantees consistent execution order across frames and builds

- **Explicit dependencies** - code clearly shows which systems depend on others

- **Frame structure control** - organize systems into logical phases (input → simulation → rendering)

- **Fixed timestep support** - `FixedStepSimulationSystemGroup` enables physics-style fixed update rate

#### Cons
- **Circular dependencies** - can create circular ordering constraints that cause errors

- **Maintenance overhead** - adding new systems requires thinking about ordering

- **Group hierarchy complexity** - nested system groups can be hard to reason about

- **Implicit dependencies** - easy to forget to add ordering attributes and get subtle bugs

#### Best use
- **Physics integration** - use `[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]` for physics, `[UpdateInGroup(typeof(AfterPhysicsSystemGroup))]` for collision response

- **Transform dependencies** - use `[UpdateAfter(typeof(TransformSystemGroup))]` when you need final world positions

- **Spawner/destroyer ordering** - spawners UpdateBefore systems that use spawned entities, destroyers UpdateAfter systems that reference entities

- **Input → Logic → Rendering** - input systems UpdateBefore gameplay logic, gameplay UpdateBefore presentation/rendering

#### Avoid if
- **No dependencies** - if execution order doesn't matter, don't add attributes (reduces coupling)

- **Default ordering is sufficient** - systems within a group run in creation order by default, sometimes that's enough

#### Extra tip
- **Common system groups:**
  - `InitializationSystemGroup` - runs first, initialization and setup
  - `SimulationSystemGroup` - main gameplay logic (default if no [UpdateInGroup])
  - `FixedStepSimulationSystemGroup` - fixed timestep updates (physics, deterministic simulation) - uses Fixed Rate Catch-Up Manager at 60 Hz
  - `VariableRateSimulationSystemGroup` - low-frequency updates using real time - uses Variable Rate Manager at 15 FPS (good for cleanup, non-critical background tasks)
  - `LateSimulationSystemGroup` - runs after [[TransformSystemGroup]], use when you need final world positions
  - `PresentationSystemGroup` - rendering and presentation, runs last
  - `AfterPhysicsSystemGroup` - runs after physics simulation (trigger/collision processing)
  - `TransformSystemGroup` - computes LocalToWorld from LocalTransform + Parent hierarchy
  - `BeginSimulationEntityCommandBufferSystem`, `EndSimulationEntityCommandBufferSystem` - [[EntityCommandBuffer]] playback points

- **Default group** - systems without `[UpdateInGroup]` default to `SimulationSystemGroup`

- **Nested groups** - system groups can contain other system groups, creating hierarchy. SimulationSystemGroup structure:
  ```
  SimulationSystemGroup (every frame)
    ├─ FixedStepSimulationSystemGroup (fixed 60 Hz)
    ├─ VariableRateSimulationSystemGroup (variable ~15 FPS)
    ├─ [Your systems without UpdateInGroup] (every frame, default)
    └─ LateSimulationSystemGroup (every frame, after transforms)
  ```

- **Custom group recommendation** - create your own system group for game-specific systems and update it before TransformSystemGroup:
  ```csharp
  [UpdateInGroup(typeof(SimulationSystemGroup))]
  [UpdateBefore(typeof(TransformSystemGroup))]
  public class GameplaySystemGroup : ComponentSystemGroup { }
  ```
  Then organize your systems inside this group for better structure.

- **Multiple constraints** - can combine `[UpdateBefore]` and `[UpdateAfter]` to sandwich a system between two others

- **OrderFirst/OrderLast** - use `[UpdateInGroup(typeof(Group), OrderFirst = true)]` or `OrderLast = true` to run at start/end of group
  - **Never set both to true** - setting both OrderFirst=true AND OrderLast=true creates undefined behavior
  - Creates three priority brackets: OrderFirst (earliest) → default (middle) → OrderLast (latest)
  - `[UpdateBefore]` and `[UpdateAfter]` only work within the same priority bracket

- **Creation order** - within a group without explicit ordering, systems update in the order they were created (file/class order)

- **CreateBefore/CreateAfter** - similar to UpdateBefore/UpdateAfter but controls creation order (rare use case):
  ```csharp
  [CreateBefore(typeof(OtherSystem))]
  public partial struct InitSystem : ISystem { }
  ```
  Only works if systems are in same group and same priority bracket

- **Rate managers and time** - see [[Rate Managers]] to understand how system groups control update frequency and [[SystemAPI.Time]] for time perception in systems

- **Debugging order** - enable "DOTS > Preferences > Show System Execution Order" in Unity to visualize system update sequence

- **Circular dependencies** - Unity will error if you create circular ordering constraints (A before B, B before C, C before A)

- **Component system groups** - can create custom system groups by inheriting from `ComponentSystemGroup` for organizing domain-specific systems

- **Fixed timestep config** - `FixedStepSimulationSystemGroup.Timestep` property controls fixed update rate (default 0.0166f = ~60 Hz)
