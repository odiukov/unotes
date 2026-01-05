---
tags:
  - system
---

#### Description
- **System instance state parameter** passed to OnUpdate(), OnCreate(), and OnDestroy() methods of [[ISystem]] - provides access to system properties and ECS operations

- **Central access point** for [[EntityManager]], [[World]], job dependencies, and queries within a system context

- **Tracks component access** - operations through SystemState register which components the system uses, essential for automatic [[System Dependencies|dependency management]]

- **Replaces direct EntityManager access** - prefer SystemState methods over direct EntityManager to ensure proper dependency tracking

#### Example
```csharp
using Unity.Entities;
using Unity.Burst;

[BurstCompile]
public partial struct ExampleSystem : ISystem
{
    // Store query as system field
    private EntityQuery _query;
    private ComponentLookup<Health> _healthLookup;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Access World
        Debug.Log($"System created in world: {state.WorldUnmanaged.Name}");

        // Create query through SystemState (registers component access)
        _query = state.GetEntityQuery(typeof(Health), typeof(Transform));

        // Get component lookup through SystemState
        _healthLookup = state.GetComponentLookup<Health>(isReadOnly: true);

        // Require specific components for system to update
        state.RequireForUpdate<GameConfig>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Access time
        float deltaTime = state.WorldUnmanaged.Time.DeltaTime;

        // Update lookups before use
        _healthLookup.Update(ref state);

        // Access EntityManager
        Entity entity = state.EntityManager.CreateEntity(typeof(Health));

        // Job dependency management
        JobHandle inputDeps = state.Dependency;

        JobHandle outputDeps = new MyJob
        {
            DeltaTime = deltaTime
        }.Schedule(_query, inputDeps);

        // Write back new dependency for next system
        state.Dependency = outputDeps;

        // Enable/disable system
        if (someCondition)
        {
            state.Enabled = false;  // Skip updates until re-enabled
        }
    }

    [BurstCompile]
    public void OnDestroy(ref SystemState state)
    {
        // Cleanup if needed
        Debug.Log("System destroyed");
    }
}
```

#### Pros
- **Automatic dependency tracking** - registers component access for [[System Dependencies]], unlike direct EntityManager usage

- **[[Burst]] compatible** - operations through SystemState work in Burst-compiled systems

- **Centralized state** - single parameter provides access to all system needs (world, entities, queries, dependencies)

- **Type-safe** - compile-time checking ensures correct usage patterns

#### Cons
- **Must be passed by ref** - always `ref SystemState state`, forgetting `ref` causes compilation error

- **Not available outside systems** - only works within [[ISystem]] lifecycle methods, not in regular classes

- **Requires understanding** - developers must learn which operations should go through SystemState vs EntityManager

- **Boilerplate** - every system method needs the SystemState parameter even if not used

#### Best use
- **All [[ISystem]] operations** - always use SystemState in OnCreate/OnUpdate/OnDestroy for proper dependency tracking

- **Query creation** - use `state.GetEntityQuery()` or `state.GetComponentLookup()` to register component access

- **Job scheduling** - read `state.Dependency` before scheduling, write new JobHandle back after scheduling

#### Avoid if
- **Outside system context** - SystemState only available in [[ISystem]], use [[EntityManager]] or [[World]] directly elsewhere

- **Static methods** - cannot use SystemState in static methods, must be instance methods of [[ISystem]]

#### Extra tip
- **Key SystemState properties and methods:**
  ```csharp
  // World access
  World world = state.World;  // Managed world
  WorldUnmanaged worldUnmanaged = state.WorldUnmanaged;  // Unmanaged (Burst-compatible)

  // EntityManager access
  EntityManager em = state.EntityManager;

  // Job dependency management
  JobHandle deps = state.Dependency;  // Read before scheduling
  state.Dependency = newJobHandle;    // Write after scheduling

  // Enable/disable system
  state.Enabled = false;  // Disable system updates

  // Query creation (registers component access)
  EntityQuery query = state.GetEntityQuery(typeof(Health));

  // Component lookup creation (registers component access)
  ComponentLookup<Health> lookup = state.GetComponentLookup<Health>(isReadOnly: true);

  // Component type handle (for IJobChunk)
  ComponentTypeHandle<Health> handle = state.GetComponentTypeHandle<Health>(isReadOnly: true);

  // Require components for update
  state.RequireForUpdate<GameConfig>();  // System won't update without this component

  // Last system version (for change detection)
  uint lastVersion = state.LastSystemVersion;

  // Global system version (current frame)
  uint globalVersion = state.GlobalSystemVersion;
  ```

- **Dependency property usage pattern:**
  ```csharp
  public void OnUpdate(ref SystemState state)
  {
      // 1. Read input dependencies
      JobHandle inputDeps = state.Dependency;

      // 2. Schedule your jobs
      JobHandle job1 = new Job1().Schedule(inputDeps);
      JobHandle job2 = new Job2().Schedule(job1);

      // 3. Write output dependencies
      state.Dependency = job2;
  }
  ```

- **RequireForUpdate patterns:**
  ```csharp
  // Require singleton component
  state.RequireForUpdate<GameConfig>();

  // Require query to have matching entities
  EntityQuery query = state.GetEntityQuery(typeof(Player));
  state.RequireForUpdate(query);

  // System won't update if requirement not met
  ```

- **SystemState vs EntityManager:**
  - **Use SystemState** when in [[ISystem]] methods (OnCreate/OnUpdate/OnDestroy)
  - **Use EntityManager** when outside system context or need operations not in SystemState
  - SystemState operations register component access for dependency tracking

- **WorldUnmanaged vs World:**
  - `state.World` - managed World reference (not Burst-compatible)
  - `state.WorldUnmanaged` - unmanaged WorldUnmanaged (Burst-compatible)
  - Use WorldUnmanaged in Burst-compiled code

- **Complete dependency manually:**
  ```csharp
  // Force wait for all dependencies (creates sync point)
  state.Dependency.Complete();  // or
  state.CompleteDependency();   // Clearer intent
  ```

## See Also

- [[ISystem]] - Modern system implementation
- [[SystemAPI]] - Convenience API for systems
- [[EntityManager]] - Direct entity manipulation
- [[System Dependencies]] - Automatic dependency management
- [[JobHandle]] - Job dependency tracking
- [[EntityQuery]] - Entity filtering
