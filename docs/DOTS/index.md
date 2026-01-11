# Unity DOTS Cheat Sheet

Complete reference for Unity's Data-Oriented Technology Stack (DOTS) - covering Entity Component System (ECS), Jobs, Burst, and performance optimization.

## Quick Navigation

### 📦 **[[DOTS/Components/index|Components]]**
Data containers - what entities "have"
- [[IComponentData]] - General-purpose components
- [[IBufferElementData (dynamic buffers)]] - Dynamic arrays
- [[BlobAsset (immutable data)]] - Immutable shared data
- [[ISharedComponentData]] - Shared component values
- [[IEnableableComponent (toggleable components)]] - Toggle on/off
- [[UnityObjectRef (UnityEngine.Object references)]] - Unity asset references
- [[IAspect (component grouping)]] - Component grouping (deprecated 6.5+)

### ⚙️ **[[DOTS/Systems/index|Systems]]**
Logic processors - where behavior lives
- [[ISystem]] - Modern struct-based systems (recommended)
- [[SystemBase]] - Legacy class-based systems
- [[EntityCommandBuffer]] - Deferred structural changes
- [[ComponentLookup and BufferLookup]] - Random entity access
- [[RefRO and RefRW]] - Component access wrappers

### ⚡ **[[DOTS/Jobs/index|Jobs]]**
Parallel processing - maximize CPU utilization
- [[IJobEntity]] - Entity iteration jobs
- [[ITriggerEventsJob]] - Physics trigger events
- More job types in documentation

### 🔨 **[[DOTS/Baking/index|Baking]]**
Authoring workflow - GameObject to Entity conversion
- [[Baker (authoring conversion)]] - Conversion code
- [[TransformUsageFlags]] - Transform component control
- [[Baking Workflow and SubScenes]] - Complete baking guide

### 🏷️ **[[DOTS/Attributes/index|Attributes]]**
System control - ordering, compilation, access
- [[BurstCompile]] - Native code compilation
- [[UpdateInGroup, UpdateBefore, UpdateAfter]] - System ordering
- [[ChunkIndexInQuery]] - Parallel ECB determinism
- [[RequireMatchingQueriesForUpdate]] - Conditional updates
- [[ReadOnly and Optional]] - Access control

### 📐 **[[DOTS/Patterns/index|Patterns]]**
Common DOTS patterns and best practices
- [[SystemAPI.Query]] - Entity iteration pattern
- [[WithChangeFilter (reactive updates)]] - Reactive systems
- [[Singleton entities and components]] - Global data

### 🎓 **[[DOTS/Advanced/index|Advanced]]**
Deep dives - internals and optimization
- [[Job Safety System]] - Race condition prevention
- [[System Dependencies]] - Automatic dependency management
- [[Sync points]] - Performance killers
- [[API Gotchas]] - Non-obvious API behaviors

### 🧪 **[[DOTS/Testing/index|Testing]]**
Testing strategies - validation and performance
- [[Unit Testing]] - System logic validation with test worlds
- [[Performance Testing]] - Automated benchmarking and optimization

### 📚 **[[DOTS/Glossary/index|Glossary]]**
Core concepts and terminology
- [[Entity]] - Unique identifier
- [[Component]] - Data container
- [[Archetype]] - Component type combination
- [[Chunk]] - 16KB memory block
- [[Burst]] - High-performance compiler
- [[Job]] - Parallel work unit
- [[Baking]] - Authoring conversion process

## Unity Entities 6.5+ Updates

- ⚠️ [[IAspect (component grouping)|IAspect]] deprecated - use direct component access
- ⚠️ Entities.ForEach deprecated - use [[IJobEntity]]
- ✨ [[UnityObjectRef (UnityEngine.Object references)|UnityObjectRef]] - new blittable Unity object wrapper
- ✨ TryGetRefRW/TryGetRefRO - safer ComponentLookup access

