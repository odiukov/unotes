---
tags:
  - component
---
#### Description
- **Large, [[Immutable|immutable]], and [[Shared|shared]]** data structures in [[Contiguous|contiguous]] native memory

- Read-only at runtime — built once (usually at initialization) and shared across many entities **without duplicating memory**

- Stored outside of [[Chunk|chunks]] — referenced by components as `BlobAssetReference<T>`

#### Example
```csharp
public struct WeaponData
{
    public BlobString Name;
    public int Damage;
    public BlobArray<float> AttackPattern;
}

// Building a blob asset
var builder = new BlobBuilder(Allocator.Persistent);
ref var root = ref builder.ConstructRoot<WeaponData>();

builder.AllocateString(ref root.Name, "Laser Gun");
root.Damage = 42;

var array = builder.Allocate(ref root.AttackPattern, 3);
array[0] = 0.1f;
array[1] = 0.3f;
array[2] = 0.6f;

// This ref will be persistent, you need to clear it manually
var blobRef = builder.CreateBlobAssetReference<WeaponData>(Allocator.Persistent);

// Don't forget to dispose!
builder.Dispose();
```

#### Pros
- Saves memory by avoiding per-entity duplication of large static data

- Highly [[Cache-friendly|cache-friendly]] for read-only lookups

- Thread-safe access without locks

- No structural changes needed to modify "configuration" — just build new blob and swap references

#### Cons
- Read-only — if you need to change data, must build new blob asset

- Building blobs more verbose than creating normal structs

- Stored outside chunk memory → requires one pointer indirection to access

#### Best use
- Large, shared, immutable configuration data:
    - Level layouts, weapon stats, animation curves, dialogue scripts
- Precomputed lookup tables

#### Avoid if
- Data changes frequently (better use [[IComponentData]] or [[IBufferElementData (dynamic buffers)|IBufferElementData]])

- Data is small and unique per entity (just use normal component)

#### Extra tip
- `BlobAsset` often paired with [[ICleanupComponentData]] to free it at right time, especially if built dynamically at runtime
