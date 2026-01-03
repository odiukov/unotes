## Contiguous

**Data stored in consecutive memory addresses** without gaps - elements placed next to each other in a continuous block.

Contiguous memory is fundamental to [[Cache-friendly]] code because CPU cache prefetcher can load upcoming data automatically.

**Unity ECS contiguous storage:**
- Components in [[Chunk]] stored contiguously by type ([[SoA layout]])
- Entities with same [[Archetype]] grouped together
- Iteration accesses memory sequentially

**Example:**
```
Contiguous array: [1][2][3][4][5] - cache-friendly
Fragmented data: [1]--[2]----[3]- - cache-unfriendly
```

**See also:** [[Cache-friendly]], [[SoA layout]], [[Chunk]]
