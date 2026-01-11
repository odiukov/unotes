---
tags:
  - optimization
---
#### Description
- **Memory layout optimization** using `[InternalBufferCapacity]` attribute to control [[IBufferElementData (dynamic buffers)|buffer]] storage location in [[Chunk]] vs heap

- Buffers with capacity ≤ InternalBufferCapacity store elements inline in chunk memory ([[Cache-friendly]]), larger buffers allocate on heap

- Default capacity is 8 elements; tune based on actual usage patterns to maximize chunk storage and minimize heap allocations

- Strategic capacity sizing prevents both wasted chunk space (oversized capacity) and heap allocation overhead (undersized capacity)

#### Example
```csharp
// Default capacity (8 elements) - may be wasteful or insufficient
public struct Targets : IBufferElementData
{
    public Entity Value;  // 8 bytes
}
// Stores 8 Entities = 64 bytes in chunk by default
// If typical usage is 3 targets → wastes 40 bytes per entity
// If typical usage is 15 targets → allocates heap memory

// Optimized capacity based on profiling
[InternalBufferCapacity(3)]  // Typical enemy has 1-3 targets
public struct Targets : IBufferElementData
{
    public Entity Value;
}
// 3 Entities = 24 bytes in chunk
// Saves 40 bytes per entity vs default
// For 100 entities: saves 4KB chunk memory

// Boss state machine buffers
[InternalBufferCapacity(10)]  // ~2 conditions per state, 5 states
public struct BossStateConditionSetup : IBufferElementData
{
    public BossConditionType Type;
    public BossConditionEvaluationType EvalType;
    public float Parameter;
    public int StateIndex;
}
// 10 conditions * 16 bytes = 160 bytes inline

[InternalBufferCapacity(5)]  // 3-5 boss phases typical
public struct BossStateSetup : IBufferElementData
{
    public float Weight;
}
// 5 states * 4 bytes = 20 bytes inline

[InternalBufferCapacity(4)]  // 3-6 patrol waypoints common
public struct PatrolWaypoint : IBufferElementData
{
    public float3 Position;
}
// 4 waypoints * 12 bytes = 48 bytes inline

// Zero capacity for large/variable buffers
[InternalBufferCapacity(0)]  // Always heap-allocated
public struct LargeDataBuffer : IBufferElementData
{
    public float4x4 Matrix;  // 64 bytes per element - too large for chunk
}
// Chunk stores only pointer (8 bytes), data on heap

// Runtime capacity override
var buffer = ecb.SetBuffer<Targets>(enemy);
buffer.Capacity = 10;  // Override internal capacity
// If 10 > InternalBufferCapacity → allocates heap
// If 10 <= InternalBufferCapacity → stays inline
```

**Capacity sizing formula:**
- Element size: `sizeof(YourStruct)` bytes
- Typical usage: Profile actual buffer lengths in game
- Chunk budget: Balance against other components
- Optimal capacity: Cover 80th percentile usage

**Profiling workflow:**
1. Start with default (no attribute)
2. Profile: Log buffer lengths during gameplay
3. Analyze distribution: Find 80th percentile
4. Optimize: Set capacity to cover common case

#### Pros
- **[[Cache-friendly]]** - inline buffer storage keeps data in chunk, same cache line as other components

- **Zero heap allocations** - if buffer stays within capacity, no GC pressure or allocation overhead

- **Predictable performance** - chunk storage has consistent access patterns, no pointer chasing

- **Memory efficiency** - right-sized capacity avoids wasting chunk space on unused buffer elements

- **Query performance** - systems iterate chunks with better locality when buffers inline

#### Cons
- **Wasted chunk space** - oversized capacity reserves space even when buffer is small/empty

- **[[Archetype]] fragmentation** - changing capacity requires code change and archetype rebuild

- **Profiling required** - optimal capacity not obvious, requires measuring actual usage patterns

- **Chunk overflow** - entities with large components + buffers may not fit in 16KB [[Chunk]], forcing early chunk allocation

- **One size fits all** - single capacity value for all entities of that archetype, can't vary per entity

#### Best use
- **Known size ranges** - when profiling shows consistent buffer sizes (e.g., 90% of entities have 1-5 elements)

- **Small elements** - Entity (8 bytes), float (4 bytes), int (4 bytes) fit well in chunk with reasonable capacities

- **Frequent access** - buffers accessed every frame benefit most from chunk-inline storage

- **Performance-critical** - tight loops iterating thousands of entities gain measurable speedup from cache locality

- **Stable usage** - buffer sizes don't vary wildly between entities or over entity lifetime

#### Avoid if
- **Highly variable sizes** - if 50% of entities have 2 elements, 50% have 100 elements, no good capacity fits both

- **Large elements** - float4x4 (64 bytes), large structs waste chunk space even at capacity 1-2

- **Rare usage** - buffers accessed once per second don't benefit from chunk locality

- **Unknown distribution** - without profiling data, default capacity 8 is safer than guessing

#### Extra tip
- **Element size calculation**: Multiply struct size by capacity → `float3 (12 bytes) * capacity 5 = 60 bytes per entity`

- **Chunk budget awareness**: Chunk is 16KB shared by all components - large buffers reduce entities per chunk

- **Common capacity patterns**: Small lists (3), medium lists (5), large lists (10), heap-only (0)

- **Zero capacity pattern**: `[InternalBufferCapacity(0)]` for large/variable data - never store inline, always heap

- **Runtime capacity override**: Can set higher capacity at spawn with `buffer.Capacity = N` to avoid reallocations

- **Profiling in development**: Log buffer lengths to find distribution: 80% have 1-3 → use capacity 3

- **Performance impact**: Chunk-inline ~1ns access (L1 cache) vs heap ~10ns (pointer indirection)

- **Memory comparison**: Capacity 3 saves 40 bytes vs capacity 8 for Entity buffers - scales across thousands of entities

- **Combining with factories**: Set buffer capacity in [[Entity Factory with Archetype]] to avoid heap allocation

- **Testing tools**: Create custom inspector to visualize buffer usage: capacity, length, heap-allocated flag

- **Capacity evolution**: Start with default, profile in alpha, optimize in beta based on actual gameplay data

- **Best practices**: Profile first, optimize for 80th percentile, consider element size impact, document reasoning

#### See also
- [[Component Memory Alignment]] - Optimize memory layout within buffer elements for better cache performance
- [[ECS Design Decisions]] - Container component patterns and when to use buffers vs multiple components
