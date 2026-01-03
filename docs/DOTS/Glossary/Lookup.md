## Lookup

**Random access container** that provides dictionary-like access to [[Component|component]] or buffer data by [[Entity]] ID, enabling entity relationships and references.

Lookups involve **random memory access**, causing [[Cache miss|cache misses]] and poor performance compared to sequential iteration. Use only when necessary for entity relationships (targets, parents, references).

**Two types:**
- **[[ComponentLookup and BufferLookup|ComponentLookup]]<T>** - random access to [[IComponentData]]
- **[[ComponentLookup and BufferLookup|BufferLookup]]<T>** - random access to [[IBufferElementData (dynamic buffers)|DynamicBuffer]]

**Key characteristics:**
- **Must be updated** - call `lookup.Update(ref state)` before each use as entity storage can move
- **Read-only or read-write** - created with `GetComponentLookup<T>(isReadOnly)` parameter
- **Thread-safe with [ReadOnly]** - read-only lookups can be used in parallel jobs
- **HasComponent check** - use `lookup.HasComponent(entity)` before accessing to avoid errors

**Example:** Projectile system uses `ComponentLookup<LocalToWorld>` to get target entity's position

**See also:** [[ComponentLookup and BufferLookup]], [[ReadOnly and Optional]], [[ITriggerEventsJob]]
