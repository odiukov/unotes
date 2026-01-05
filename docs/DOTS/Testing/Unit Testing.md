---
tags:
  - testing
---
#### Description
- **Isolated system validation** - create test [[World|worlds]], populate with entities and components, execute systems, and assert expected state changes

- **Two main fixture approaches** - reference Unity's built-in test fixtures via `testables` in manifest.json, or create custom copies for isolated control

- **Recommended pattern** - extend `ECSTestsFixture` (or custom version) with simplified APIs like `UpdateSystem<T>()`, `CreateEntity()`, and `CreateEntities()` to reduce boilerplate

- Tests follow **Arrange-Act-Assert** pattern - set up entities (arrange), execute system (act), verify component state (assert)

#### Example
```csharp
using NUnit.Framework;
using Unity.Entities;
using Unity.Entities.Tests;  // ECSTestsFixture location
using Unity.Collections;

// Custom test fixture extending Unity's ECSTestsFixture
// Inherit from ECSTestsFixture to get World, EntityManager, and proper setup/teardown
public class CustomEcsTestsFixture : ECSTestsFixture
{
    // World and EntityManager already available from base class
    // Base class handles setup/teardown automatically

    // Add simplified helper methods
    protected void UpdateSystem<T>() where T : unmanaged, ISystem
    {
        ref var system = ref World.GetOrCreateSystemManaged<T>();
        system.Update(World.Unmanaged);
    }

    protected Entity CreateEntity(params ComponentType[] types)
    {
        return m_Manager.CreateEntity(types);  // m_Manager from base class
    }

    protected void CreateEntities(ComponentType[] types, int count)
    {
        var archetype = m_Manager.CreateArchetype(types);
        m_Manager.CreateEntity(archetype, count, Allocator.Temp);
    }
}

// Example test - testing movement system
public class MovementSystemTests : CustomEcsTestsFixture
{
    [Test]
    public void MovementSystem_AppliesVelocity_WhenNoCollision()
    {
        // Arrange - create entity with velocity
        var entity = CreateEntity(
            ComponentType.ReadWrite<LocalTransform>(),
            ComponentType.ReadWrite<Velocity>());

        m_Manager.SetComponentData(entity, new LocalTransform { Position = float3.zero });
        m_Manager.SetComponentData(entity, new Velocity { Value = new float3(1, 0, 0) });

        // Act - run system
        UpdateSystem<MovementSystem>();

        // Assert - verify position changed
        var transform = m_Manager.GetComponentData<LocalTransform>(entity);
        Assert.Greater(transform.Position.x, 0, "Entity should have moved");
    }

    [Test]
    public void MovementSystem_StopsMovement_WhenCollisionInDirection()
    {
        // Arrange - create entity with collision component
        var entity = CreateEntity(
            ComponentType.ReadWrite<LocalTransform>(),
            ComponentType.ReadWrite<Velocity>(),
            ComponentType.ReadWrite<CollisionData>());

        m_Manager.SetComponentData(entity, new Velocity { Value = new float3(1, 0, 0) });
        m_Manager.SetComponentData(entity, new CollisionData { Direction = new float3(1, 0, 0) });

        // Act
        UpdateSystem<MovementSystem>();

        // Assert - velocity should be reset
        var velocity = m_Manager.GetComponentData<Velocity>(entity);
        Assert.AreEqual(float3.zero, velocity.Value, "Movement should stop on collision");
    }
}
```

#### Setup Options

**How to Access ECSTestsFixture**

`ECSTestsFixture` is Unity's built-in base class located in `Unity.Entities.Tests` namespace. Two approaches to access it:

**Option 1: Reference Testables (Recommended)**

Add to `manifest.json` in your Packages folder:
```json
{
  "dependencies": {
    "com.unity.entities": "1.3.0"
  },
  "testables": ["com.unity.entities"]
}
```

Then inherit from it:
```csharp
using Unity.Entities.Tests;  // Accessible via testables

public class CustomEcsTestsFixture : ECSTestsFixture
{
    // Your helper methods here
}
```

**Pros:**
- Automatic updates when Unity Entities package updates
- Official Unity implementation with proper setup/teardown
- Access to `World`, `m_Manager` (EntityManager), and other test utilities

**Cons:**
- Unity's internal tests appear in Test Runner (can be filtered)
- Depends on Unity package internals

**Option 2: Custom Copy**

Copy `ECSTestsFixture.cs` from Unity Entities package into your test assembly:
1. Find file at: `Library/PackageCache/com.unity.entities@X.X.X/Unity.Entities.Tests/ECSTestsFixture.cs`
2. Copy to your test assembly folder
3. Create inheriting class:

```csharp
public class CustomEcsTestsFixture : ECSTestsFixture
{
    // Your helper methods here
}
```

**Pros:**
- Clean Test Runner (no Unity internal tests)
- Full control over fixture implementation
- Can customize base setup/teardown logic

**Cons:**
- Manual maintenance when upgrading Unity Entities
- Duplicate code from Unity package

**ECSTestsFixture Provides:**
- `World` - Test world instance
- `m_Manager` - EntityManager for the test world
- `EmptySystem` - Empty system for testing
- Automatic setup/teardown of test world
- Helper methods for entity/system creation

#### Pros
- **Fast feedback** - isolated tests run in milliseconds, catching bugs before runtime

- **Regression prevention** - automated tests prevent breaking existing functionality during refactors

- **[[Burst]] validation** - mark systems with `[BurstCompile]` to test production code paths and compilation errors

- **Component state verification** - easily verify [[Component|component]] data changes using [[EntityManager]] queries

#### Cons
- **Boilerplate setup** - each test needs [[World]] creation, [[Entity|entity]] setup, and teardown (mitigated by custom fixture)

- **Limited to logic** - can't test rendering, physics, or other Unity engine integrations in unit tests

- **Manual maintenance** - custom fixtures need updates when Unity ECS APIs change

#### Best use
- **System logic validation** - test calculation systems (damage, health regen, movement) with predictable inputs/outputs

- **Edge case testing** - verify systems handle empty queries, zero values, negative numbers, boundary conditions

- **Refactoring safety** - comprehensive test suite enables confident refactoring of system implementations

#### Avoid if
- **Testing Unity engine integration** - use PlayMode tests for rendering, physics, animation systems

- **Testing performance** - use [[Performance Testing]] with benchmarking tools for measuring system execution time

- **Complex multi-system interactions** - integration tests better suited for testing multiple systems working together

#### Extra tip
- **ECSTestsFixture base class members** - when inheriting from `ECSTestsFixture`, you get access to:
  - `World` - the test world
  - `m_Manager` - the EntityManager (use this instead of `EntityManager`)
  - `EmptySystem` - empty system for dependency testing
  - Override `[SetUp]` and `[TearDown]` carefully - call `base.SetUp()` and `base.TearDown()` if you add custom logic

- **Use `[BurstCompile]` consistently** - annotate test systems same as production to ensure Burst compilation succeeds

- **ComponentLookup for read-only access** - in jobs, use `ComponentLookup<T>` for efficient random component access without write conflicts

- **Define test assemblies properly** - add `"UNITY_INCLUDE_TESTS"` constraint to test assembly definition to prevent build failures:
  ```json
  {
    "name": "MyGame.Tests",
    "references": ["MyGame", "Unity.Entities"],
    "optionalUnityReferences": ["TestAssemblies"],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": true,
    "defineConstraints": ["UNITY_INCLUDE_TESTS"]
  }
  ```

- **Maintain infrastructure assemblies** - separate reusable test utilities into dedicated assembly for use across EditMode and PlayMode tests

- **Test multiple scenarios** - for each system, test: nominal case, edge cases (empty, zero, max values), error conditions

- **Entity archetype testing** - verify systems correctly query entities with specific [[Archetype|archetype]] combinations using `.WithAll<T>()`, `.WithNone<T>()`

- **Shared test data** - create helper methods for common entity setups to reduce duplication across test classes
