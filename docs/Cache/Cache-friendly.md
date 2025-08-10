Data is laid out in memory so that when the CPU loads one piece of data, it’s likely to also load other nearby data into its **CPU cache** — reducing the number of slow memory fetches from RAM.
### Why it matters here
Modern CPUs don’t read memory one byte at a time — they fetch it in **cache lines** (usually 64 bytes). 
If your data is [[Contiguous|contiguous]] in memory and you process it in order, the CPU can pre-load the next data chunk while you’re still working on the current one.

**ECS [[chunk]] layout** and [[BlobAsset (immutable data)|BlobAssets]] are designed for this:

- [[IComponentData]] stores many entities’ components of the same type **back-to-back in a [[chunk|chunks]]** → processing them in a loop hits data that’s already in [[Levels of cache|L1/L2]] cache.
    
- [[BlobAsset (immutable data)|BlobAssets]] put all their fields, arrays, and strings **in a single [[contiguous]] memory block** → reading one part brings nearby parts into cache automatically.

--- 
### Example
Imagine you have 1000 enemies and you want to update their health:

- **Cache-friendly**: All `Health` values are stored consecutively in a chunk. CPU fetches a cache line and gets several `Health` structs at once → minimal RAM trips.
    
- **Cache-unfriendly**: Each `Health` is scattered in memory (e.g., as part of different GameObjects in MonoBehaviours). Every access may trigger a separate RAM fetch → slower.

---
**TL;DR in DOTS terms**:  
Cache-friendly = [[contiguous]] data layout → fewer CPU [[Cache miss|cache misses]]→ much faster iteration over large sets of entities.