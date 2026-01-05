---
tags:
  - testing
  - performance
---
#### Description
- **Automated performance benchmarking** - measure system execution time using Unity's Performance Testing package to track performance across code changes

- **Critical for DOTS optimization** - helps compare multiple API approaches (EntityManager vs [[SystemAPI]]), determine when [[Job|jobs]] provide benefit, and track performance regressions in CI/CD

- **Reveals job overhead** - benchmarking shows parallelization isn't always faster - main thread can outperform jobs for small entity counts (<10,000)

- **Hardware-specific testing** - performance varies significantly between editor and target devices; always test on actual hardware

#### Example
```csharp
using NUnit.Framework;
using Unity.PerformanceTesting;
using Unity.Entities;
using Unity.Entities.Tests;  // ECSTestsFixture
using Unity.Collections;

// Custom performance test fixture extending ECSTestsFixture
public class PerformanceTestFixture : ECSTestsFixture
{
    // World and m_Manager inherited from base class
    // Base handles setup/teardown automatically

    protected void MeasureSystemUpdate<T>() where T : unmanaged, ISystem
    {
        ref var systemState = ref World.Unmanaged.GetExistingSystemState<T>();

        Measure.Method(() =>
        {
            systemState.Update(World.Unmanaged);
        })
        .WarmupCount(5)
        .MeasurementCount(10)
        .IterationsPerMeasurement(5)
        .SampleGroup("SystemUpdate")
        .Run();
    }
}

// Example: Comparing EntityManager vs SystemAPI
[TestFixture]
public class ComponentAccessPerformanceTests : PerformanceTestFixture
{
    private const int EntityCount = 1000;

    [Test, Performance]
    public void EntityManager_HasComponent_Performance()
    {
        // Arrange - create entities
        var archetype = m_Manager.CreateArchetype(typeof(Health));
        m_Manager.CreateEntity(archetype, EntityCount, Allocator.Temp);

        var entities = m_Manager.GetAllEntities(Allocator.Temp);

        // Act & Measure
        Measure.Method(() =>
        {
            foreach (var entity in entities)
            {
                bool hasHealth = m_Manager.HasComponent<Health>(entity);
            }
        })
        .WarmupCount(5)
        .MeasurementCount(10)
        .IterationsPerMeasurement(5)
        .SampleGroup("EntityManager.HasComponent")
        .Run();

        entities.Dispose();
    }

    [Test, Performance]
    public void SystemAPI_HasComponent_Performance()
    {
        // Arrange
        var archetype = m_Manager.CreateArchetype(typeof(Health));
        m_Manager.CreateEntity(archetype, EntityCount, Allocator.Temp);

        World.CreateSystem<TestSystem>();
        ref var system = ref World.Unmanaged.GetExistingSystemState<TestSystem>();

        // Act & Measure
        Measure.Method(() =>
        {
            system.Update(World.Unmanaged);
        })
        .WarmupCount(5)
        .MeasurementCount(10)
        .IterationsPerMeasurement(5)
        .SampleGroup("SystemAPI.HasComponent")
        .Run();
    }
}

// Example: Job scheduling comparison
[TestFixture]
public class JobSchedulingPerformanceTests : PerformanceTestFixture
{
    private const int EntityCount = 1000;

    [Test, Performance]
    public void MainThread_Execution()
    {
        CreateTestEntities(EntityCount);
        World.CreateSystem<MainThreadSystem>();
        MeasureSystemUpdate<MainThreadSystem>();
    }

    [Test, Performance]
    public void JobRun_Execution()
    {
        CreateTestEntities(EntityCount);
        World.CreateSystem<JobRunSystem>();
        MeasureSystemUpdate<JobRunSystem>();
    }

    [Test, Performance]
    public void JobSchedule_Execution()
    {
        CreateTestEntities(EntityCount);
        World.CreateSystem<JobScheduleSystem>();
        MeasureSystemUpdate<JobScheduleSystem>();
    }

    [Test, Performance]
    public void JobScheduleParallel_Execution()
    {
        CreateTestEntities(EntityCount);
        World.CreateSystem<JobScheduleParallelSystem>();
        MeasureSystemUpdate<JobScheduleParallelSystem>();
    }

    private void CreateTestEntities(int count)
    {
        var archetype = m_Manager.CreateArchetype(
            typeof(LocalTransform),
            typeof(Velocity));
        m_Manager.CreateEntity(archetype, count, Allocator.Temp);
    }
}
```

#### Why Performance Testing Matters

**API Clarity**
- Multiple Unity ECS APIs provide identical functionality (EntityManager vs [[SystemAPI]])
- Benchmarking reveals which approach is faster for your specific use case

**Job Overhead Analysis**
- [[Job]] parallelization has overhead - not always beneficial for small datasets
- Testing reveals breakeven point where jobs outperform main thread execution

**CI/CD Integration**
- Automated performance tests catch regressions before deployment
- Track performance trends across versions without manual profiling

**Unity Version Optimization**
- Minor Unity updates improve code generation and performance
- Automated tests quickly validate if your code benefits from updates

#### Configuration

**Recommended Settings**
```csharp
Measure.Method(() => { /* code */ })
    .WarmupCount(5)              // Warmup iterations
    .MeasurementCount(10)         // Number of samples
    .IterationsPerMeasurement(5)  // Iterations per sample
    .SampleGroup("GroupName")     // Category/label
    .Run();
```

**Package Requirements**
- Unity Performance Testing Package 3.0.3+
- Target device testing (not editor) for accurate results
- Custom test fixture for simplified system measurement

#### Pros
- **Automated regression detection** - CI/CD integration catches performance degradation before production

- **API comparison** - objectively compare EntityManager, [[SystemAPI]], [[ComponentLookup and BufferLookup|ComponentLookup]] performance for your use cases

- **Job overhead visibility** - reveals when [[Job]] parallelization helps vs hurts performance

- **Version upgrade confidence** - validate Unity updates improve (or don't regress) your specific systems

#### Cons
- **High variance** - [[Job]] system tests show significant standard deviation, making small differences hard to measure

- **Editor inaccuracy** - editor performance doesn't match target device; requires device builds for reliable data

- **Isolated measurement** - tests stripped-down systems, may not reflect real-world performance with full game context

- **Setup complexity** - requires understanding Performance Testing API, custom fixtures, and statistical interpretation

#### Best use
- **API decision making** - when choosing between equivalent APIs (EntityManager.HasComponent vs SystemAPI.HasComponent)

- **Job parallelization decisions** - determine entity count threshold where jobs become beneficial

- **CI/CD performance gates** - fail builds if systems regress beyond acceptable thresholds

- **Unity version validation** - test if upgrading Unity improves your specific systems

#### Avoid if
- **Profiling hot paths** - use Unity Profiler for detailed frame analysis and optimization of critical systems

- **Small sample size (<1000 entities)** - high variance makes results unreliable; use profiling instead

- **Integration testing** - performance tests measure isolated systems; not suitable for multi-system interaction testing

- **Initial development** - focus on correctness first, add performance tests after systems stabilize

#### Key Findings from Research

**EntityManager vs SystemAPI (HasComponent checks)**
- `SystemAPI.HasComponent<T>()` ~23% slower than `EntityManager.HasComponent<T>()`
- Both operations remain in microsecond range - rarely a bottleneck
- Choose based on code clarity, not performance

**Job Scheduling Comparison (1,000 entities)**
- Main thread: ~10.75 μs (fastest)
- Job.Run(): ~16.35 μs (52% slower)
- Job.Schedule(): ~17.70 μs (65% slower)
- Job.ScheduleParallel(): ~30.70 μs (185% slower)

**Parallelization Breakeven**
- Jobs show overhead for <10,000 entities
- Main thread often faster for small datasets
- Test your specific system to find breakeven point

#### Extra tip
- **Strip test systems** - remove unnecessary code from systems under test for accurate isolated measurement

- **Account for variance** - job testing shows high standard deviation; interpret results conservatively and look for large differences (>20%)

- **Test real scenarios** - use actual game systems or stripped versions matching production code, not synthetic benchmarks

- **Target device required** - editor performance misleading; always validate on actual hardware (mobile, console, PC)

- **Sample unit choice** - use microseconds for system-level tests; milliseconds for frame-time measurements

- **Comparative testing** - test multiple approaches side-by-side with identical entity counts and conditions

- **Statistical significance** - look for consistent differences across multiple runs; single measurements can be misleading

- **Profiler correlation** - use Unity Profiler to validate performance test findings in real game context

- **Entity count scaling** - test at multiple scales (100, 1K, 10K, 100K entities) to understand performance characteristics

- **Measure.Scope alternative** - for non-system code, use `Measure.Scope()` for simplified measurement blocks:
  ```csharp
  using (Measure.Scope("MyOperation"))
  {
      // Code to measure
  }
  ```

- **Burst compilation** - ensure [[BurstCompile|[BurstCompile]]] attribute present on systems to match production performance

- **Caching queries** - pre-create [[EntityQuery|EntityQuery]] objects in OnCreate to avoid query creation overhead in measurements
