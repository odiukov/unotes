## Burst

**High-performance compiler** that translates C# code to highly optimized native CPU code using LLVM, providing 10-100x performance improvements over standard C#.

Burst uses SIMD (Single Instruction Multiple Data) vectorization, aggressive optimizations, and platform-specific code generation to achieve near-C++ performance from C# code.

**Key characteristics:**
- **SIMD vectorization** - automatically uses CPU vector instructions (SSE, AVX, NEON)
- **Zero garbage collection** - no managed allocations or GC pauses
- **Platform-optimized** - generates optimal code for each target platform
- **Restricted C# subset** - no managed objects, exceptions, or virtual calls

**What Burst allows:**
- Blittable types (primitives, structs)
- Unity.Mathematics (math library)
- Native collections (NativeArray, NativeList)
- [[IComponentData]], [[IBufferElementData (dynamic buffers)|DynamicBuffer]], [[BlobAsset (immutable data)|BlobAsset]]

**What Burst forbids:**
- Managed objects (classes, strings)
- Garbage collection allocations
- Exceptions (try/catch/throw)
- Virtual method calls
- `Debug.Log`

**Example:** `[BurstCompile]` attribute on [[ISystem]] or [[IJobEntity]] enables Burst compilation

**See also:** [[BurstCompile]], [[IJobEntity]], [[ISystem]]
