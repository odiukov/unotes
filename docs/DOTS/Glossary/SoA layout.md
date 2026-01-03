## SoA Layout (Structure of Arrays)

**Memory layout pattern** where component data of the same type is stored contiguously in arrays, as opposed to AoS (Array of Structures).

In Unity ECS, each [[Component]] type is stored in its own array within a [[Chunk]], allowing efficient [[Cache-friendly]] iteration over components of the same type.

**SoA (Unity ECS):**
```
Chunk memory layout:
[Health1, Health2, Health3, ...] (all Health components together)
[Transform1, Transform2, Transform3, ...] (all Transform components together)
```

**AoS (traditional OOP):**
```
[Health1+Transform1, Health2+Transform2, Health3+Transform3, ...]
```

**Benefits:**
- **[[Cache-friendly]]** - CPU prefetcher loads next component automatically
- **SIMD vectorization** - process multiple components in parallel
- **Skip unused data** - only load component types needed by system

**See also:** [[Chunk]], [[Cache-friendly]], [[Archetype]]
