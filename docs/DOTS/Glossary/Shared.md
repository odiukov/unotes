## Shared

**Data referenced by multiple entities without duplication** - stored once in memory and accessed by many.

In Unity DOTS:
- **[[ISharedComponentData]]** - component value shared across entities
- **[[BlobAsset (immutable data)|BlobAsset]]** - immutable configuration shared via reference

**Benefits:**
- Memory efficiency - store once, use many times
- Cache-friendly - entities grouped by shared value in same [[Chunk]]

**Trade-offs:**
- [[ISharedComponentData]] changes cause [[Chunk]] rearrangement
- Must be [[Immutable]] or managed carefully for thread safety

**See also:** [[ISharedComponentData]], [[BlobAsset (immutable data)]], [[Immutable]]
