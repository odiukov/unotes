## Allocators

**Memory allocation strategies** in Unity for native containers - determines lifetime and performance characteristics of allocated memory.

**Common allocators:**
- **Allocator.Temp** - single frame, fastest, disposed automatically
- **Allocator.TempJob** - up to 4 frames, for jobs
- **Allocator.Persistent** - permanent until manual disposal
- **Allocator.Domain** - lives until domain unload (rare use)

**Usage:**
```csharp
var array = new NativeArray<int>(100, Allocator.TempJob);
// Use in job
array.Dispose(state.Dependency); // Schedule disposal after job
```

**Rules:**
- `Allocator.Temp` - use for single-frame temporary data
- `Allocator.TempJob` - use for data passed to jobs (dispose after)
- `Allocator.Persistent` - use for data that lives across frames

**See also:** [[Job]], NativeArray, NativeList
