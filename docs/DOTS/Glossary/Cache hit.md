## Cache hit

**Successful retrieval of data from CPU cache** - requested memory is already in cache (L1/L2/L3), avoiding slow RAM access.

Cache hits are **10-100x faster** than cache misses. Unity DOTS achieves high cache hit rates through:
- [[SoA layout]] - components stored [[Contiguous|contiguously]]
- [[Chunk]]-based memory - entities grouped by [[Archetype]]
- Sequential iteration - systems process chunks linearly

**Cache hit rate** = (cache hits) / (total memory accesses) - higher is better.

**See also:** [[Cache miss]], [[Cache-friendly]], [[Levels of cache]]
