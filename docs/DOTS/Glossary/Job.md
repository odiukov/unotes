## Job

**Parallel work unit** that executes on worker threads, processing entity data in a [[Thread-safe]] and [[Burst]]-compilable manner for maximum performance.

Jobs are the primary way to parallelize entity processing in DOTS. Unity's job system schedules jobs across CPU cores, handles dependencies, and ensures thread safety.

**Key characteristics:**
- **Parallel execution** - runs on multiple worker threads simultaneously
- **[[Burst]] compatible** - compiles to highly optimized native code
- **Automatic scheduling** - Unity schedules jobs efficiently across available CPU cores
- **Dependency tracking** - job dependencies ensure correct execution order

**Common job types:**
- **[[IJobEntity]]** - simplified entity iteration (most common)
- **IJobChunk** - manual chunk iteration for advanced control
- **[[ITriggerEventsJob]]** - physics trigger event processing
- **IJobParallelFor** - general parallel for loops

**Example:** `ProjectileMoveJob : IJobEntity` processes all projectile entities in parallel

**See also:** [[IJobEntity]], [[BurstCompile]], [[ChunkIndexInQuery]]
