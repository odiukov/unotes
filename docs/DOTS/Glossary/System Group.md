## System Group

**Container for organizing systems** - groups systems into logical phases (initialization, simulation, rendering) and controls their execution order and update frequency.

**Key characteristics:**
- **System groups are systems** - they implement the same ISystem/SystemBase interface
- **Can be nested** - system groups can contain other system groups, creating hierarchy
- **Control update order** - systems run in group order: earlier groups before later groups
- **Manage rate** - each group can have a [[Rate Managers|rate manager]] to control update frequency and time perception

**Structure:**
```
InitializationSystemGroup
  └─ [initialization systems]

SimulationSystemGroup
  ├─ FixedStepSimulationSystemGroup (60 Hz fixed)
  │    └─ [physics systems]
  ├─ VariableRateSimulationSystemGroup (15 FPS variable)
  │    └─ [low-frequency tasks]
  ├─ [default gameplay systems]
  └─ LateSimulationSystemGroup
       └─ [post-transform systems]

PresentationSystemGroup
  └─ [rendering systems]
```

**Usage:**
```csharp
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
public partial struct PhysicsSystem : ISystem { }

// Custom system group
[UpdateInGroup(typeof(SimulationSystemGroup))]
[UpdateBefore(typeof(TransformSystemGroup))]
public class GameplaySystemGroup : ComponentSystemGroup { }

[UpdateInGroup(typeof(GameplaySystemGroup))]
public partial struct PlayerMovementSystem : ISystem { }
```

**Default groups:**
- **InitializationSystemGroup** - runs first every frame (setup, initialization)
- **SimulationSystemGroup** - main gameplay logic (default if no [UpdateInGroup] specified)
- **PresentationSystemGroup** - rendering and presentation, runs last

**Important:**
- Systems without `[UpdateInGroup]` automatically go to SimulationSystemGroup
- Within a group, systems run in creation order unless explicit ordering is specified
- System groups push their own time context via [[SystemAPI.Time]]
- Each group can have different update frequency via [[Rate Managers]]

**See also:** [[UpdateInGroup, UpdateBefore, UpdateAfter]], [[Rate Managers]], [[ISystem]], [[SystemAPI.Time]]
