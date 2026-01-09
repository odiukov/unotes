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
Updates **every frame** but reports a **fixed delta time**, unaffected by `Time.timeScale`.

```csharp
// Create custom group with fixed rate simple
[UpdateInGroup(typeof(SimulationSystemGroup))]
public class FixedRateSimpleGroup : ComponentSystemGroup
{
    public FixedRateSimpleGroup()
    {
        // Always reports 0.02s (50 FPS) regardless of actual framerate
        SetRateManagerCreateAllocator(new RateUtils.FixedRateSimpleManager(50f));
    }
}

[UpdateInGroup(typeof(FixedRateSimpleGroup))]
public partial struct OldSchoolGameSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // SystemAPI.Time.DeltaTime is always 0.02 (50 FPS)
        // Runs every frame like old games that assumed fixed framerate
    }
}
```

**Characteristics:**
- Runs every frame (actual framerate doesn't matter)
- Reports constant delta time (e.g., 0.02s for 50 FPS setting)
- Ignores `Time.timeScale` - always reports same fixed delta
- Good for legacy code porting or simple fixed-step logic

##### Fixed Rate Catch-Up Manager
Works like Unity's `FixedUpdate` - runs at **fixed timestep**, may update **0, 1, or multiple times per frame** to maintain target rate.

```csharp
// FixedStepSimulationSystemGroup uses this by default (60 Hz)
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
public partial struct PhysicsSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs at fixed 60 Hz (default)
        // At 120 FPS: runs every other frame
        // At 30 FPS: runs twice per frame
        // At 15 FPS with timeScale=2: runs 4x per frame (capped by MaximumDeltaTime)
        float fixedDelta = SystemAPI.Time.DeltaTime; // Always 0.0166f (1/60)
    }
}

// Configure fixed rate for custom group
public class CustomPhysicsGroup : ComponentSystemGroup
{
    public CustomPhysicsGroup()
    {
        // 50 FPS fixed rate
        var fixedRate = new RateUtils.FixedRateCatchUpManager(50f);
        SetRateManagerCreateAllocator(fixedRate);
    }
}
```

**Characteristics:**
- Runs 0-N times per frame to maintain fixed rate
- Reports fixed delta time
- Respects `Time.timeScale` (2x speed = 2x updates per frame)
- Has `MaximumDeltaTime` cap to prevent runaway spiral (if update takes >1 frame)
- **Known Unity bug**: doesn't respect timeScale for MaximumDeltaTime calculation (can copy/fix if needed)

**How it works:**
- Accumulates time debt each frame
- If debt >= fixed timestep, update and subtract timestep from debt
- Keeps updating until debt < fixed timestep
- Prevents spiral with max updates cap

##### Variable Rate Manager
Updates at **target framerate using real time** (unscaled), independent of `Time.timeScale`. Good for **low-frequency operations**.

```csharp
// VariableRateSimulationSystemGroup uses this (15 FPS default)
[UpdateInGroup(typeof(VariableRateSimulationSystemGroup))]
public partial struct CleanupDeadEnemiesSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs ~15 times per second (using real stopwatch time)
        // Unaffected by Time.timeScale
        // At 60 FPS: runs ~every 4th frame
        // DeltaTime = real time since last update

        // Clean up off-screen dead enemies
        // Doesn't need to happen every frame
    }
}

// Custom variable rate group
public class SlowUpdateGroup : ComponentSystemGroup
{
    public SlowUpdateGroup()
    {
        // Run at 10 FPS using real time
        var variableRate = new RateUtils.VariableRateManager(10f);
        SetRateManagerCreateAllocator(variableRate);
    }
}
```

**Characteristics:**
- Uses stopwatch to measure real elapsed time
- Compares real time to target interval, runs if exceeded
- Reports **unscaled time** - ignores `Time.timeScale`
- Good for non-gameplay systems (cleanup, analytics, background tasks)

#### Internal Mechanism

How rate managers work inside system group update:

```csharp
// Pseudo-code of what happens in ComponentSystemGroup.OnUpdate
void OnUpdate()
{
    // 1. Check if should update this frame
    if (!RateManager.ShouldGroupUpdate(out float deltaTime))
        return; // Skip this frame

    // 2. Push time values to SystemAPI.Time
    PushTime(deltaTime, elapsedTime);

    // 3. Set group allocators (World Update Allocator)
    PushAllocators();

    // 4. Update all child systems
    UpdateAllSystems();

    // 5. Check if should update again (catch-up scenario)
    while (RateManager.ShouldGroupUpdate(out deltaTime))
    {
        // Can run multiple times per frame
        PushTime(deltaTime, elapsedTime);
        UpdateAllSystems();
    }

    // 6. Pop allocators
    PopAllocators();
    PopTime();
}
```

**Key points:**
- `ShouldGroupUpdate()` returns true/false based on rate manager logic
- Time is "pushed" - child systems see rate manager's time via `SystemAPI.Time`
- [[World]] Update Allocator is set per-group (automatic deallocation)
- Can loop multiple times (Fixed Rate Catch-Up Manager)
- Can return false on first call (skip frame entirely)

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
- **World Update Allocator** - rate managers automatically manage group allocators. Use `WorldUpdateAllocator` for temporary allocations that auto-dispose at group update end:
  ```csharp
  var tempArray = CollectionHelper.CreateNativeArray<int>(100,
      state.WorldUpdateAllocator); // No manual Dispose needed!
  ```

- **Configure FixedStepSimulationSystemGroup rate:**
  ```csharp
  var fixedGroup = world.GetExistingSystemManaged<FixedStepSimulationSystemGroup>();
  fixedGroup.Timestep = 1f / 50f; // Change to 50 Hz
  ```

- **Priority brackets** - within a system group:
  - `[UpdateInGroup(typeof(Group), OrderFirst = true)]` - earliest bracket
  - Default - middle bracket
  - `[UpdateInGroup(typeof(Group), OrderLast = true)]` - latest bracket
  - Never set both OrderFirst=true AND OrderLast=true (creates undefined behavior)

- **Custom rate manager** - can implement `IRateManager` interface for custom update logic (e.g., every 5 frames, on specific events)

- **LateSimulationSystemGroup** - runs after [[TransformSystemGroup]], use when you need final world positions for rendering/effects

- **Default placement** - systems without `[UpdateInGroup]` go to SimulationSystemGroup, after FixedStepSimulationSystemGroup but before LateSimulationSystemGroup

- **Create custom group before TransformSystemGroup** - recommended for organizing game-specific systems:
  ```csharp
  [UpdateInGroup(typeof(SimulationSystemGroup))]
  [UpdateBefore(typeof(TransformSystemGroup))]
  public class GameplaySystemGroup : ComponentSystemGroup { }
  ```

- **Debugging time** - add debug logs showing `SystemAPI.Time.DeltaTime` vs `UnityEngine.Time.deltaTime` to understand rate manager behavior
