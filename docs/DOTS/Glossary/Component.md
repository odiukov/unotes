## Component

**Data container attached to an [[Entity]]** - pure data struct (no logic) that defines what an entity "has" or "is".

Components are stored in memory using [[SoA layout]] (Structure of Arrays) within [[Chunk|chunks]], grouped by [[Archetype]] for optimal [[Cache-friendly]] iteration.

**Component types:**
- **[[IComponentData]]** - General-purpose component (struct, blittable)
- **[[ISharedComponentData]]** - Shared across entities for memory efficiency
- **[[IBufferElementData (dynamic buffers)|IBufferElementData]]** - Dynamic array attached to entity
- **[[BlobAsset (immutable data)|BlobAsset]]** - Immutable shared configuration data
- **[[IEnableableComponent (toggleable components)|IEnableableComponent]]** - Component that can be toggled on/off
- **[[ICleanupComponentData]]** - Persists after entity destroyed for cleanup logic
- **[[Tag Component]]** - Empty struct for filtering (no data)
- **[[Managed IComponentData (class components)|Managed IComponentData]]** - Class-based component (not recommended)

**Key characteristics:**
- **Data only** - no methods or behavior (logic lives in systems)
- **Blittable structs** - must be copyable byte-by-byte for Burst
- **Cache-optimized storage** - stored contiguously in chunks by type

**Example:**
```csharp
public struct Health : IComponentData
{
    public float Current;
    public float Max;
}
```

**See also:** [[Entity]], [[Archetype]], [[ISystem]]
