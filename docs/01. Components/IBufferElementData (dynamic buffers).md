---
tags:
  - component
---
**Description**
- A flexible array of one element type per entity.
- Can be stored **inside the [[Chunk|chunk]]**(in-place) or **outside** in heap memory if too large.
- Uses [[InternalBufferCapacity (IBC)]] to determine how many elements fit in-chunk before spilling to heap.

**Example**
```csharp
[InternalBufferCapacity(4)] // optional attribute
public struct InventoryItem : IBufferElementData
{
    public Entity Item;
}
```

**Pros**
- Variable length.
- Great for collections (inventory, waypoints, per-frame events).

**Cons**
- Changing length is slower than [[IComponentData]].
- Can reduce cache locality if buffers are large.
- Spill to heap if exceeding [[InternalBufferCapacity (IBC)|IBC]] → slower + extra memory ops.
- Large [[InternalBufferCapacity (IBC)|IBC]] wastes memory if most buffers stay small.

**Best use**
- Arrays of points, inventory, event history.

**Avoid if**
- Small fixed-size data sets (better store in [[IComponentData]] fields).
- Buffers that frequently exceed [[InternalBufferCapacity (IBC)|IBC]] unless absolutely necessary.