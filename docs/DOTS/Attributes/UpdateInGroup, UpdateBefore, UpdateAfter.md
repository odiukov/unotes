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
// Fixed timestep physics
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
[BurstCompile]
public partial struct SpawnerSystem : ISystem { }

// After transforms computed
[UpdateAfter(typeof(TransformSystemGroup))]
[BurstCompile]
public partial struct ProjectileMoveSystem : ISystem { }

// Before another system
[UpdateBefore(typeof(TowerPlacementSystem))]
[BurstCompile]
public partial struct InputSystem : ISystem { }

// After physics simulation
[UpdateInGroup(typeof(AfterPhysicsSystemGroup))]
[BurstCompile]
public partial struct EnemyPlayerCollisionSystem : ISystem { }

// Combining multiple ordering attributes
[UpdateInGroup(typeof(SimulationSystemGroup))]
[UpdateAfter(typeof(MovementSystem))]
[UpdateBefore(typeof(AnimationSystem))]
[BurstCompile]
public partial struct CombatSystem : ISystem { }
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

- **Nested groups** - SimulationSystemGroup structure:
  ```
  SimulationSystemGroup (every frame)
    ├─ FixedStepSimulationSystemGroup (fixed 60 Hz)
    ├─ VariableRateSimulationSystemGroup (variable ~15 FPS)
    └─ LateSimulationSystemGroup (after transforms)
  ```

- **OrderFirst/OrderLast** - `[UpdateInGroup(typeof(Group), OrderFirst = true)]` or `OrderLast = true` runs at start/end of group
  - **Never set both to true** - creates undefined behavior
  - Creates three priority brackets: OrderFirst → default → OrderLast
  - `[UpdateBefore]`/`[UpdateAfter]` only work within same priority bracket

- **Creation order** - within a group without explicit ordering, systems update in file/class creation order

- **Rate managers** - see [[Rate Managers]] for how system groups control update frequency and [[SystemAPI.Time]] for time perception

- **Debugging** - enable "DOTS > Preferences > Show System Execution Order" to visualize update sequence

- **Fixed timestep config** - `FixedStepSimulationSystemGroup.Timestep` controls fixed update rate (default 0.0166f = ~60 Hz)
