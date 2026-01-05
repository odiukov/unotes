---
tags:
  - pattern
  - advanced
  - initialization
---

#### Description
- **Custom world initialization** interface that replaces Unity's default world creation - implement Initialize() method to control which [[World|worlds]] are created and which [[ISystem|systems]] are added

- **Overrides default behavior** - when ICustomBootstrap implementation exists, Unity calls it instead of creating default world with all systems automatically

- **Full control over worlds** - create multiple worlds (client/server, simulation/presentation), choose specific systems, customize system groups and ordering

- **Must return true** - Initialize() must return true to indicate custom bootstrap succeeded and prevent default world creation (return false for default behavior)

#### Example
```csharp
using Unity.Entities;
using UnityEngine;

// Basic custom bootstrap - create custom world manually
public class MyCustomBootstrap : ICustomBootstrap
{
    public bool Initialize(string defaultWorldName)
    {
        // Create custom world
        var world = new World("My Custom World");

        // Add systems manually
        var simulationGroup = world.GetOrCreateSystemManaged<SimulationSystemGroup>();
        var movementSystem = world.GetOrCreateSystemManaged<MovementSystem>();
        var renderingSystem = world.GetOrCreateSystemManaged<RenderingSystem>();

        // Add systems to system group
        simulationGroup.AddSystemToUpdateList(movementSystem);
        simulationGroup.AddSystemToUpdateList(renderingSystem);

        // Sort system update order
        simulationGroup.SortSystems();

        // Set as default world (used by SceneSystem, etc.)
        World.DefaultGameObjectInjectionWorld = world;

        // Return true to indicate success (prevents default world creation)
        return true;
    }
}

// Advanced: Creating multiple worlds (client/server)
public class ClientServerBootstrap : ICustomBootstrap
{
    public bool Initialize(string defaultWorldName)
    {
        // Create server world
        var serverWorld = new World("Server World", WorldFlags.GameServer);

        // Add server-specific systems
        var serverSim = serverWorld.GetOrCreateSystemManaged<SimulationSystemGroup>();
        serverSim.AddSystemToUpdateList(serverWorld.GetOrCreateSystemManaged<ServerGameLogicSystem>());
        serverSim.AddSystemToUpdateList(serverWorld.GetOrCreateSystemManaged<ServerNetworkSystem>());
        serverSim.SortSystems();

        // Create client world
        var clientWorld = new World("Client World", WorldFlags.GameClient);

        // Add client-specific systems
        var clientSim = clientWorld.GetOrCreateSystemManaged<SimulationSystemGroup>();
        clientSim.AddSystemToUpdateList(clientWorld.GetOrCreateSystemManaged<ClientInputSystem>());
        clientSim.AddSystemToUpdateList(clientWorld.GetOrCreateSystemManaged<ClientRenderingSystem>());
        clientSim.SortSystemsAndUpdateList();

        // Set client as default world
        World.DefaultGameObjectInjectionWorld = clientWorld;

        return true;
    }
}

// Using DefaultWorldInitialization helpers (recommended approach)
public class RecommendedBootstrap : ICustomBootstrap
{
    public bool Initialize(string defaultWorldName)
    {
        // Create world with default system groups (Initialization, Simulation, Presentation)
        var world = new World(defaultWorldName);

        // Get all system types from assemblies
        var systems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.Default);

        // Filter systems (example: exclude specific systems)
        var filteredSystems = new List<Type>();
        foreach (var system in systems)
        {
            // Skip unwanted systems
            if (system == typeof(UnwantedSystem))
                continue;

            filteredSystems.Add(system);
        }

        // Add filtered systems to world
        DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(world, filteredSystems);

        // Set as default world
        World.DefaultGameObjectInjectionWorld = world;

        return true;
    }
}

// Conditional worlds based on defines or configuration
public class ConditionalBootstrap : ICustomBootstrap
{
    public bool Initialize(string defaultWorldName)
    {
#if UNITY_SERVER
        // Server build: create server world only
        var world = new World("Server World", WorldFlags.GameServer);

        var systems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.ServerSimulation);
        DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(world, systems);
#else
        // Client build: create client world only
        var world = new World("Client World", WorldFlags.GameClient);

        var systems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.ClientSimulation);
        DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(world, systems);
#endif

        World.DefaultGameObjectInjectionWorld = world;
        return true;
    }
}

// Custom bootstrap with logging and error handling
public class RobustBootstrap : ICustomBootstrap
{
    public bool Initialize(string defaultWorldName)
    {
        try
        {
            Debug.Log($"Custom bootstrap: Creating world '{defaultWorldName}'");

            var world = new World(defaultWorldName);

            // Get systems
            var systems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.Default);
            Debug.Log($"Found {systems.Count} systems to initialize");

            // Add systems
            DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(world, systems);

            // Set default world
            World.DefaultGameObjectInjectionWorld = world;

            Debug.Log("Custom bootstrap: Success");
            return true;
        }
        catch (System.Exception e)
        {
            Debug.LogError($"Custom bootstrap failed: {e.Message}");

            // Return false to fall back to default world creation
            return false;
        }
    }
}

// Editor-only custom bootstrap (different behavior in editor vs builds)
public class EditorAwareBootstrap : ICustomBootstrap
{
    public bool Initialize(string defaultWorldName)
    {
#if UNITY_EDITOR
        // Editor: Create world with editor-specific systems
        var world = new World(defaultWorldName);

        var systems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.Default | WorldSystemFilterFlags.Editor);
        DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(world, systems);

        Debug.Log("[Editor] Custom bootstrap with editor systems");
#else
        // Build: Create optimized world without editor systems
        var world = new World(defaultWorldName);

        var systems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.Default);
        DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(world, systems);
#endif

        World.DefaultGameObjectInjectionWorld = world;
        return true;
    }
}
```

#### Pros
- **Full initialization control** - decide exactly which worlds exist and which systems run in each

- **Multiple world support** - easily create client/server, simulation/rendering, or other world separation patterns

- **System filtering** - exclude specific systems from initialization (testing, optimization, feature flags)

- **Custom execution order** - manually control system update order beyond [UpdateBefore]/[UpdateAfter] attributes

#### Cons
- **All or nothing** - disables default world creation, must manually create all needed worlds and systems

- **Easy to forget systems** - manually adding systems risks missing critical Unity systems (rendering, transforms, etc.)

- **Maintenance burden** - custom bootstrap code needs updates when adding new systems or Unity versions change

- **Complexity** - harder to debug issues with world initialization compared to default setup

#### Best use
- **Client/server architecture** - separate worlds for client simulation and server simulation

- **Multiple simulation contexts** - gameplay world, UI world, background processing world

- **System filtering** - exclude specific systems based on platform, configuration, or feature flags

- **Custom world lifetime** - worlds that need non-standard creation/destruction timing

#### Avoid if
- **Default setup sufficient** - most projects work fine with Unity's default world creation

- **Simple system ordering** - use [UpdateInGroup], [UpdateBefore], [UpdateAfter] attributes instead

- **Prototyping** - custom bootstrap adds complexity during early development

- **Single world projects** - unnecessary overhead if only one world needed with all systems

#### Extra tip
- **ICustomBootstrap discovery:**
  ```csharp
  // Unity automatically finds ICustomBootstrap implementations
  // Place in any assembly - no registration needed
  // First implementation found is used (only one allowed)

  public class MyBootstrap : ICustomBootstrap
  {
      public bool Initialize(string defaultWorldName)
      {
          // Called automatically at startup
          return true;
      }
  }
  ```

- **DefaultWorldInitialization helpers (recommended):**
  ```csharp
  // Get all system types from assemblies
  List<Type> systems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.Default);

  // Filter flags
  WorldSystemFilterFlags.Default           // Normal gameplay systems
  WorldSystemFilterFlags.Editor            // Editor-only systems
  WorldSystemFilterFlags.Streaming         // Scene streaming systems
  WorldSystemFilterFlags.ClientSimulation  // Client-side systems
  WorldSystemFilterFlags.ServerSimulation  // Server-side systems
  WorldSystemFilterFlags.ThinClientSimulation  // Thin client systems

  // Add systems to world with default groups
  DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(world, systems);

  // Or create default world entirely
  World world = DefaultWorldInitialization.Initialize(defaultWorldName);
  ```

- **World.DefaultGameObjectInjectionWorld:**
  ```csharp
  // Set which world GameObject conversion uses
  World.DefaultGameObjectInjectionWorld = myWorld;

  // Many Unity systems use this world:
  // - SceneSystem loads SubScenes into this world
  // - Baking adds entities to this world
  // - ConvertToEntity uses this world
  ```

- **WorldFlags options:**
  ```csharp
  // Game worlds
  new World("Client", WorldFlags.GameClient)
  new World("Server", WorldFlags.GameServer)
  new World("Thin Client", WorldFlags.GameThinClient)

  // Editor worlds
  new World("Editor", WorldFlags.Editor)

  // Streaming
  new World("Streaming", WorldFlags.Streaming)

  // Live conversion
  new World("Shadow", WorldFlags.Shadow)

  // No flags (custom world)
  new World("Custom")
  ```

- **Manually adding systems to groups:**
  ```csharp
  var world = new World("My World");

  // Get or create system groups
  var initGroup = world.GetOrCreateSystemManaged<InitializationSystemGroup>();
  var simGroup = world.GetOrCreateSystemManaged<SimulationSystemGroup>();
  var presGroup = world.GetOrCreateSystemManaged<PresentationSystemGroup>();

  // Create and add systems
  var mySystem = world.GetOrCreateSystemManaged<MySystem>();
  simGroup.AddSystemToUpdateList(mySystem);

  // Sort systems (respects [UpdateBefore]/[UpdateAfter] attributes)
  simGroup.SortSystems();

  // Or sort all systems in world
  world.GetOrCreateSystemManaged<SimulationSystemGroup>().SortSystems();
  ```

- **Return value semantics:**
  ```csharp
  public bool Initialize(string defaultWorldName)
  {
      // Return true: Custom bootstrap succeeded, don't create default world
      return true;

      // Return false: Custom bootstrap failed, create default world as fallback
      return false;
  }
  ```

- **Disposing worlds:**
  ```csharp
  // Worlds don't automatically dispose
  // Manual cleanup needed:

  void OnApplicationQuit()
  {
      // Dispose all worlds
      foreach (var world in World.All)
      {
          world.Dispose();
      }
  }

  // Or dispose specific world
  myWorld.Dispose();
  ```

- **Accessing all worlds:**
  ```csharp
  // Iterate all created worlds
  foreach (var world in World.All)
  {
      Debug.Log($"World: {world.Name}, Flags: {world.Flags}");
  }

  // Find specific world by name
  foreach (var world in World.All)
  {
      if (world.Name == "Server World")
      {
          // Found server world
      }
  }
  ```

- **System filtering example:**
  ```csharp
  public bool Initialize(string defaultWorldName)
  {
      var world = new World(defaultWorldName);

      // Get all systems
      var allSystems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.Default);

      // Filter out specific systems
      var filteredSystems = new List<Type>();
      foreach (var systemType in allSystems)
      {
          // Skip systems by type
          if (systemType == typeof(UnwantedSystem))
              continue;

          // Skip systems by namespace
          if (systemType.Namespace?.StartsWith("UnityEngine.Rendering") == true)
              continue;

          // Skip systems by attribute
          if (systemType.GetCustomAttribute<DebugOnlyAttribute>() != null)
              continue;

          filteredSystems.Add(systemType);
      }

      DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(world, filteredSystems);
      World.DefaultGameObjectInjectionWorld = world;

      return true;
  }
  ```

- **Script execution order:**
  ```csharp
  // ICustomBootstrap.Initialize() called very early:
  // 1. Assemblies loaded
  // 2. ICustomBootstrap.Initialize() called
  // 3. Worlds created (either custom or default)
  // 4. Systems initialized
  // 5. MonoBehaviour Awake() called
  // 6. First system updates

  // Can access worlds in MonoBehaviour.Awake():
  void Awake()
  {
      var world = World.DefaultGameObjectInjectionWorld;
      // World already exists
  }
  ```

- **Common gotchas:**
  ```csharp
  // WRONG: Forgetting to set DefaultGameObjectInjectionWorld
  public bool Initialize(string defaultWorldName)
  {
      var world = new World(defaultWorldName);
      // ... add systems ...
      return true;  // Missing: World.DefaultGameObjectInjectionWorld = world;
  }
  // Result: SubScene loading, baking, etc. won't work

  // WRONG: Forgetting to sort systems
  simGroup.AddSystemToUpdateList(mySystem);
  // Missing: simGroup.SortSystems();
  // Result: Systems run in random order, [UpdateBefore] ignored

  // WRONG: Multiple ICustomBootstrap implementations
  // Only one ICustomBootstrap allowed per project
  // Unity uses first one found (undefined which one)
  ```

- **Testing with custom bootstrap:**
  ```csharp
  // Option 1: Disable in tests
  #if !UNITY_TESTS
  public class MyBootstrap : ICustomBootstrap { ... }
  #endif

  // Option 2: Conditional return
  public bool Initialize(string defaultWorldName)
  {
      #if UNITY_TESTS
      return false;  // Use default in tests
      #else
      // Custom initialization
      return true;
      #endif
  }

  // Option 3: Manual world setup in tests
  [Test]
  public void MyTest()
  {
      var world = new World("Test World");
      // Add only systems needed for test
      // Dispose after test
  }
  ```

## See Also

- [[World]] - Isolated entity collection container
- [[ISystem]] - Modern system implementation
- [[SystemBase]] - Legacy managed system base class
- [[EntityManager]] - Entity and component operations
- [[SimulationSystemGroup]] - Main simulation update group
