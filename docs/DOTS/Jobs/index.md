# Jobs

Jobs are the primary mechanism for parallel entity processing in Unity DOTS. They execute on worker threads, process entity data in a thread-safe manner, and compile with Burst for maximum performance.

## Job Types

### Core Job Types
- **[[IJobEntity]]** - Simplified entity iteration with automatic query generation (most common for ECS)
- **[[IJob]]** - Single-threaded job for sequential processing on worker thread
- **[[IJobParallelFor]]** - Parallel array processing distributed across multiple CPU cores

### Physics Job Types
- **[[ITriggerEventsJob]]** - Physics trigger event processing

### Advanced Job Types
- **IJobChunk** - Manual chunk iteration for advanced control over entity processing

## Key Concepts

Jobs enable **massive parallelization** of entity processing by distributing work across CPU cores. Unity's job system handles scheduling, dependencies, and thread safety automatically.

### Job Dependencies
- **[[JobHandle]]** - Returned by `.Schedule()`, represents a scheduled job and enables dependency chaining
- Jobs execute in order when dependencies are specified, preventing race conditions
- `JobHandle.Complete()` forces synchronization, creating a [[Sync points|sync point]]

### Data Containers
- **[[NativeCollections]]** - Unmanaged collections (`NativeArray`, `NativeList`, etc.) compatible with jobs and [[Burst]]
- Must use `Allocator.TempJob` or `Allocator.Persistent` when passing to jobs
- Manual disposal required with `.Dispose()` to prevent memory leaks

### Safety and Performance
All jobs support [[Burst]] compilation for 10-100x performance improvements and integrate with the [[EntityCommandBuffer]] system for deferred structural changes.

The [[Job Safety System]] validates thread safety at runtime, preventing race conditions by tracking read/write access to native containers.

## Best Practices

1. Use **[[IJobEntity]]** for most entity processing - it's the simplest and most ergonomic
2. Always add **[[BurstCompile]]** attribute for maximum performance
3. Use **[[ChunkIndexInQuery]]** when using EntityCommandBuffer.ParallelWriter
4. Mark read-only data with **[[ReadOnly and Optional|[ReadOnly]]]** to enable parallel execution
5. Update **[[ComponentLookup and BufferLookup|lookups]]** before scheduling jobs

## See Also

- [[BurstCompile]] - Burst compiler attribute
- [[EntityCommandBuffer]] - Deferred structural changes
- [[ComponentLookup and BufferLookup]] - Random entity access in jobs
