# Testing

Testing in Unity DOTS ensures system correctness, prevents regressions, and tracks performance across code changes. The ECS architecture's deterministic nature makes it highly testable.

## Testing Types

### Unit Testing
- **[[Unit Testing]]** - Isolated system validation with test worlds, entity setup, and state assertions

### Performance Testing
- **[[Performance Testing]]** - Automated benchmarking to compare APIs, measure job overhead, and track performance regressions

## Quick Reference

| Test Type | Purpose | Tools | Best For |
|-----------|---------|-------|----------|
| [[Unit Testing]] | System logic validation | NUnit, ECSTestsFixture | Calculation systems, edge cases, refactoring |
| [[Performance Testing]] | Execution time measurement | Unity Performance Testing | API comparison, job overhead, CI/CD gates |

## Testing Workflow

### Unit Testing Workflow
1. **Setup** - Create test [[World]], [[EntityManager]], and entities
2. **Arrange** - Populate entities with test [[Component|components]]
3. **Act** - Execute [[ISystem|system]] update
4. **Assert** - Verify component state changes
5. **Teardown** - Dispose test world

### Performance Testing Workflow
1. **Setup** - Create test fixture with warmup/measurement configuration
2. **Arrange** - Create entities at target scale (100, 1K, 10K, 100K)
3. **Measure** - Run system with Performance Testing API
4. **Compare** - Analyze results across multiple approaches
5. **Validate** - Test on target hardware, not editor

## Best Practices

### Unit Testing
1. **Use custom test fixtures** - extend `ECSTestsFixture` with simplified APIs
2. **Test edge cases** - verify empty queries, zero values, boundary conditions
3. **Mark systems with [[BurstCompile|[BurstCompile]]]** - test production code paths
4. **Separate test assemblies** - use `UNITY_INCLUDE_TESTS` define constraint

### Performance Testing
1. **Test on target hardware** - editor performance misleading
2. **Account for variance** - [[Job]] tests show high standard deviation
3. **Test multiple scales** - small datasets favor main thread, large datasets favor jobs
4. **Isolate systems** - strip unnecessary code for accurate measurements

## Key Findings

### EntityManager vs SystemAPI
- `SystemAPI.HasComponent<T>()` ~23% slower than `EntityManager.HasComponent<T>()`
- Both remain microsecond-range operations
- Choose based on code clarity

### Job Scheduling (1,000 entities)
- Main thread: ~10.75 μs (fastest)
- Job.Run(): ~16.35 μs (52% slower)
- Job.Schedule(): ~17.70 μs (65% slower)
- Job.ScheduleParallel(): ~30.70 μs (185% slower)
- Jobs show overhead below ~10,000 entities

## Common Patterns

### Custom Test Fixture
```csharp
public class CustomEcsTestsFixture
{
    protected World World;
    protected EntityManager EntityManager;

    [SetUp]
    public void SetUp()
    {
        World = new World("Test World");
        EntityManager = World.EntityManager;
    }

    [TearDown]
    public void TearDown()
    {
        if (World != null && World.IsCreated)
            World.Dispose();
    }

    protected void UpdateSystem<T>() where T : unmanaged, ISystem
    {
        ref var system = ref World.GetOrCreateSystemManaged<T>();
        system.Update(World.Unmanaged);
    }
}
```

### Performance Measurement
```csharp
Measure.Method(() =>
{
    system.Update(World.Unmanaged);
})
.WarmupCount(5)
.MeasurementCount(10)
.IterationsPerMeasurement(5)
.Run();
```

## See Also

- [[ISystem]] - Systems to test
- [[World]] - Test world creation
- [[EntityManager]] - Entity manipulation in tests
- [[BurstCompile|[BurstCompile]]] - Ensure Burst compilation in tests
- [[Job]] - Job scheduling and parallelization
