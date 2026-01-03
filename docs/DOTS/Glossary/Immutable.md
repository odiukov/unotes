## Immutable

**Data that cannot be modified after creation** - read-only and unchanging throughout its lifetime.

In Unity DOTS, [[BlobAsset (immutable data)|BlobAssets]] are immutable shared data structures that multiple entities can reference without copying.

**Benefits:**
- **Thread-safe** - multiple threads can read safely
- **Memory efficient** - shared across entities
- **Performance** - no need for synchronization

**Example:** Configuration data, lookup tables, shared mesh data

**See also:** [[BlobAsset (immutable data)]], [[Shared]]
