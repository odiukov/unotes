---
tags:
  - component
---
**Description**
- Structs holding **data only** (value types, no references to objects).
- Stored in [[Chunk|chunks]], [[cache-friendly]].

**Example**
```csharp
public struct Health : IComponentData { public int Value; }
```

**Pros**
- Very high performance for bulk processing.
- Easy to serialize.
- Minimal memory overhead.
- Perfect for frequently accessed data.

**Cons**
- Cannot contain managed types (`string`, `class`, etc.).
- [[Structural changes]] are expensive (adding/removing component).

**Best use**
- Frequently updated game loop data: position, velocity, health.
- Can be used as [[Tag Component]]

**Avoid if**
- Large or rarely used data (wastes [[chunk]] memory).