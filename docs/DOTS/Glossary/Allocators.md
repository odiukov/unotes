## Allocators

**Memory allocation strategies** in Unity for native containers - determines lifetime and performance characteristics of allocated memory.

**Common allocators:**
- **Allocator.Temp** - single frame, fastest, disposed automatically
- **Allocator.TempJob** - up to 4 frames, for jobs
- **Allocator.Persistent** - permanent until manual disposal
- **Allocator.Domain** - lives until domain unload (rare use)
- **WorldUpdateAllocator** - lives until system group update completes, **auto-disposes** (recommended for temp data in systems)

**Usage:**
```csharp
// TempJob for jobs
var array = new NativeArray<int>(100, Allocator.TempJob);
// Use in job
array.Dispose(state.Dependency); // Schedule disposal after job

// WorldUpdateAllocator - auto-disposes, no manual cleanup needed!
var tempData = CollectionHelper.CreateNativeArray<int>(100,
    state.WorldUpdateAllocator); // Automatically disposed when system group update completes
// Use in system, no Dispose() needed
```

**Rules:**
- `Allocator.Temp` - use for single-frame temporary data
- `Allocator.TempJob` - use for data passed to jobs (dispose after)
- `Allocator.Persistent` - use for data that lives across frames
- `WorldUpdateAllocator` - **preferred for temp allocations in systems** - automatic cleanup, no manual Dispose needed

**See also:** [[Job]], [[Rate Managers]], NativeArray, NativeList
