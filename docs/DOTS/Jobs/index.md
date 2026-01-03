# Jobs

Jobs are the primary mechanism for parallel entity processing in Unity DOTS. They execute on worker threads, process entity data in a thread-safe manner, and compile with Burst for maximum performance.

## Job Types

### Core Job Types
- **[[IJobEntity]]** - Simplified entity iteration with automatic query generation (most common)
- **[[ITriggerEventsJob]]** - Physics trigger event processing

### Advanced Job Types
- **IJobChunk** - Manual chunk iteration for advanced control
- **IJobParallelFor** - General parallel for loops

## Key Concepts

Jobs enable **massive parallelization** of entity processing by distributing work across CPU cores. Unity's job system handles scheduling, dependencies, and thread safety automatically.

All jobs support [[Burst]] compilation for 10-100x performance improvements and integrate with the [[EntityCommandBuffer]] system for deferred structural changes.

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
