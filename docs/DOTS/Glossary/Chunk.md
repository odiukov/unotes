## Chunk

**16KB memory block** that stores entities with the same [[Archetype]] in [[SoA layout]] (Structure of Arrays).

Chunks are the fundamental storage unit in Unity ECS - components of the same type are stored [[Contiguous|contiguously]] within a chunk for [[Cache-friendly]] iteration.

**Key facts:**
- **Fixed size:** 16KB per chunk
- **Same archetype:** all entities in chunk have identical component set
- **Capacity:** varies by archetype (typically 32-128 entities)
- **SoA storage:** each component type has its own array

**Memory layout:**
```
Chunk (16KB):
├─ [Health, Health, Health, ...]     (all Health components)
├─ [Transform, Transform, ...]        (all Transform components)
└─ [Velocity, Velocity, ...]          (all Velocity components)
```

**Performance:**
- Good: full chunks = efficient memory use
- Bad: sparse chunks (few entities) = wasted memory, poor iteration

**See also:** [[Archetype]], [[SoA layout]], [[Cache-friendly]], [[Chunk rearrangement]]
