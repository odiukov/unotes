---
tags:
  - system
  - advanced
---
#### Description
- **Control how often a system group updates** - rate managers determine if a system group should run this frame, how many times, and what time values it reports

- **Manage time perception** - each rate manager pushes its own delta time and elapsed time to systems via `SystemAPI.Time`, independent of actual framerate

- **Enable fixed timestep logic** - essential for deterministic physics simulation and framerate-independent gameplay logic

- System groups can run **0, 1, or multiple times per frame** depending on their rate manager configuration

#### Rate Manager Types

##### No Rate Manager (Default)
System group updates every frame with parent group's time.

```csharp
// Default behavior - runs every frame
[UpdateInGroup(typeof(SimulationSystemGroup))]
public partial struct MyGameplaySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs once per frame
        // DeltaTime = parent group's delta time
    }
}
```

##### Fixed Rate Simple Manager
Updates **every frame** but reports **fixed delta time**, unaffected by `Time.timeScale`.

```csharp
[UpdateInGroup(typeof(SimulationSystemGroup))]
public class FixedRateSimpleGroup : ComponentSystemGroup
{
    public FixedRateSimpleGroup()
    {
        SetRateManagerCreateAllocator(new RateUtils.FixedRateSimpleManager(50f));
    }
}

[UpdateInGroup(typeof(FixedRateSimpleGroup))]
public partial struct OldSchoolGameSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // SystemAPI.Time.DeltaTime always 0.02 (50 FPS), runs every frame
    }
}
```

**Characteristics:** Runs every frame, reports constant delta time, ignores timeScale, good for legacy porting

##### Fixed Rate Catch-Up Manager
Like Unity's `FixedUpdate` - runs at **fixed timestep**, may update **0, 1, or N times per frame**.

```csharp
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]  // Default 60 Hz
public partial struct PhysicsSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // At 120 FPS: runs every other frame
        // At 30 FPS: runs twice per frame
        float fixedDelta = SystemAPI.Time.DeltaTime; // Always 0.0166f (1/60)
    }
}
```

**Characteristics:** Runs 0-N times per frame, fixed delta time, respects timeScale, has MaximumDeltaTime cap

##### Variable Rate Manager
Updates at **target framerate using real time** (unscaled), good for **low-frequency operations**.

```csharp
[UpdateInGroup(typeof(VariableRateSimulationSystemGroup))]  // Default 15 FPS
public partial struct CleanupSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs ~15x per second using real time (unaffected by timeScale)
        // At 60 FPS: runs ~every 4th frame
    }
}
```

**Characteristics:** Uses real time stopwatch, ignores timeScale, good for cleanup/background tasks

#### Internal Mechanism

Rate managers control `ComponentSystemGroup.OnUpdate`:
- `ShouldGroupUpdate()` returns true/false based on rate logic
- Time is "pushed" - child systems see rate manager's time via `SystemAPI.Time`
- [[World]] Update Allocator is set per-group (automatic deallocation)
- Can loop multiple times (catch-up) or skip frames entirely

#### Default System Groups & Rate Managers

Unity's built-in system groups and their rate managers:

| System Group | Rate Manager | Frequency | Purpose |
|--------------|--------------|-----------|---------|
| **InitializationSystemGroup** | None | Every frame | Runs first - initialization, setup |
| **SimulationSystemGroup** | None | Every frame | Main gameplay logic (default group) |
| **FixedStepSimulationSystemGroup** | Fixed Rate Catch-Up (60 Hz) | Fixed 60 Hz | Physics, deterministic simulation |
| **VariableRateSimulationSystemGroup** | Variable Rate (15 FPS) | ~15 FPS real time | Low-frequency tasks, cleanup |
| **LateSimulationSystemGroup** | None | Every frame | After [[TransformSystemGroup]] |
| **PresentationSystemGroup** | None | Every frame | Rendering, presentation - runs last |

**Nested structure:**
```
InitializationSystemGroup (every frame)
SimulationSystemGroup (every frame)
  ├─ FixedStepSimulationSystemGroup (fixed 60 Hz)
  ├─ VariableRateSimulationSystemGroup (variable ~15 FPS)
  ├─ [Your systems here] (every frame, default location)
  └─ LateSimulationSystemGroup (every frame, after transforms)
PresentationSystemGroup (every frame)
```

#### Pros
- **Deterministic simulation** - fixed timestep ensures consistent physics regardless of framerate

- **Performance optimization** - variable rate allows expensive systems to run less frequently

- **Framerate independence** - gameplay logic decoupled from rendering framerate

- **Time control** - each system group has independent time perception and scaling

- **Automatic allocator management** - [[World]] Update Allocator lifetime tied to group update

#### Cons
- **Complexity** - multiple time contexts can be confusing (system sees different DeltaTime than game framerate)

- **Catch-up spiral** - if fixed rate system takes >1 frame, can spiral (mitigated by MaximumDeltaTime cap)

- **Debugging difficulty** - systems running 0 or multiple times per frame harder to reason about

- **Unity bug** - Fixed Rate Catch-Up doesn't respect timeScale for MaximumDeltaTime (can copy/fix code)

#### Best use
- **Physics simulation** - use Fixed Rate Catch-Up Manager (FixedStepSimulationSystemGroup) for deterministic physics

- **Low-priority tasks** - use Variable Rate Manager for cleanup, analytics, non-critical background work

- **Legacy porting** - use Fixed Rate Simple Manager when porting old code that assumed fixed framerate

- **Custom update schedules** - create custom system groups with specific rate managers for domain-specific timing needs

#### Avoid if
- **Default timing is sufficient** - don't add rate managers unnecessarily, most gameplay systems work fine in SimulationSystemGroup

- **Need frame-synchronized logic** - if system must run exactly once per rendered frame, avoid rate managers

- **Simple prototypes** - rate managers add complexity, use default groups for rapid prototyping

#### Extra tip
- **World Update Allocator** - use `state.WorldUpdateAllocator` for temporary allocations that auto-dispose:
  ```csharp
  var tempArray = CollectionHelper.CreateNativeArray<int>(100, state.WorldUpdateAllocator);
  ```

- **Configure fixed rate:**
  ```csharp
  var fixedGroup = world.GetExistingSystemManaged<FixedStepSimulationSystemGroup>();
  fixedGroup.Timestep = 1f / 50f; // Change to 50 Hz
  ```

- **Custom rate manager** - implement `IRateManager` interface for custom update logic

- **Default placement** - systems without `[UpdateInGroup]` go to SimulationSystemGroup
