# Unity DOTS Cheat Sheet

Complete reference for Unity's Data-Oriented Technology Stack (DOTS) - covering Entity Component System (ECS), Jobs, Burst, and performance optimization.

## Quick Navigation

### 📦 **[[Components/index|Components]]**
Data containers - what entities "have"
- [[IComponentData]] - General-purpose components
- [[IBufferElementData (dynamic buffers)]] - Dynamic arrays
- [[BlobAsset (immutable data)]] - Immutable shared data
- [[ISharedComponentData]] - Shared component values
- [[IEnableableComponent (toggleable components)]] - Toggle on/off
- [[UnityObjectRef (UnityEngine.Object references)]] - Unity asset references
- [[IAspect (component grouping)]] - Component grouping (deprecated 6.5+)

### ⚙️ **[[Systems/index|Systems]]**
Logic processors - where behavior lives
- [[ISystem]] - Modern struct-based systems (recommended)
- [[SystemBase]] - Legacy class-based systems
- [[EntityCommandBuffer]] - Deferred structural changes
- [[ComponentLookup and BufferLookup]] - Random entity access
- [[RefRO and RefRW]] - Component access wrappers

### ⚡ **[[Jobs/index|Jobs]]**
Parallel processing - maximize CPU utilization
- [[IJobEntity]] - Entity iteration jobs
- [[ITriggerEventsJob]] - Physics trigger events
- More job types in documentation

### 🔨 **[[Baking/index|Baking]]**
Authoring workflow - GameObject to Entity conversion
- [[Baker (authoring conversion)]] - Conversion code
- [[TransformUsageFlags]] - Transform component control
- [[Baking Workflow and SubScenes]] - Complete baking guide

### 🏷️ **[[Attributes/index|Attributes]]**
System control - ordering, compilation, access
- [[BurstCompile]] - Native code compilation
- [[UpdateInGroup, UpdateBefore, UpdateAfter]] - System ordering
- [[ChunkIndexInQuery]] - Parallel ECB determinism
- [[RequireMatchingQueriesForUpdate]] - Conditional updates
- [[ReadOnly and Optional]] - Access control

### 📐 **[[../Patterns/index|Patterns]]**
Common DOTS patterns and best practices
- [[SystemAPI.Query]] - Entity iteration pattern
- [[WithChangeFilter (reactive updates)]] - Reactive systems
- [[Singleton entities and components]] - Global data

### 🎓 **[[Advanced/index|Advanced]]**
Deep dives - internals and optimization
- [[Job Safety System]] - Race condition prevention
- [[System Dependencies]] - Automatic dependency management
- [[Sync points]] - Performance killers
- [[API Gotchas]] - Non-obvious API behaviors

### 🧪 **[[Testing/index|Testing]]**
Testing strategies - validation and performance
- [[Unit Testing]] - System logic validation with test worlds
- [[Performance Testing]] - Automated benchmarking and optimization

### 📚 **[[Glossary/index|Glossary]]**
Core concepts and terminology
- [[Entity]] - Unique identifier
- [[Component]] - Data container
- [[Archetype]] - Component type combination
- [[Chunk]] - 16KB memory block
- [[Burst]] - High-performance compiler
- [[Job]] - Parallel work unit
- [[Baking]] - Authoring conversion process

## Getting Started

1. **New to DOTS?** Start with [[Entity]], [[Component]], and [[Archetype]] in the Glossary
2. **Writing systems?** Learn [[ISystem]] and [[SystemAPI.Query]]
3. **Need performance?** Read [[Burst]], [[Job]], and [[Sync points]]
4. **Building content?** Check [[Baking/index|Baking]] section
5. **Testing code?** Explore [[Testing/index|Testing]] for unit and performance testing
6. **Optimization?** Study [[Advanced/index|Advanced]] topics

## Unity Entities 6.5+ Updates

- ⚠️ [[IAspect (component grouping)|IAspect]] deprecated - use direct component access
- ⚠️ Entities.ForEach deprecated - use [[IJobEntity]]
- ✨ [[UnityObjectRef (UnityEngine.Object references)|UnityObjectRef]] - new blittable Unity object wrapper
- ✨ TryGetRefRW/TryGetRefRO - safer ComponentLookup access

## See Also

- [Unity Entities Manual](https://docs.unity3d.com/Packages/com.unity.entities@latest)
- [Unity Burst Manual](https://docs.unity3d.com/Packages/com.unity.burst@latest)
- [Unity Job System Manual](https://docs.unity3d.com/Manual/JobSystem.html)
