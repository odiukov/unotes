# Components

Components are pure data containers attached to [[Entity|entities]] - they define what an entity "has" or "is" without containing behavior logic.

## Component Types

### Core Component Types
- **[[IComponentData]]** - General-purpose component (blittable struct)
- **[[IBufferElementData (dynamic buffers)]]** - Dynamic array of elements
- **[[BlobAsset (immutable data)]]** - Immutable shared configuration data
- **[[ISharedComponentData]]** - Value shared across multiple entities
- **[[IEnableableComponent (toggleable components)]]** - Component that can be enabled/disabled
- **[[ICleanupComponentData]]** - Persists after entity destroyed for cleanup
- **[[Tag Component]]** - Empty struct for filtering (no data)

### Hybrid/Advanced Types
- **[[UnityObjectRef (UnityEngine.Object references)]]** - Blittable wrapper for Unity objects (Unity Entities 6.5+)
- **[[Managed IComponentData (class components)]]** - Class-based components (not recommended)
- **[[IAspect (component grouping)]]** - Component grouping (deprecated in Unity Entities 6.5+)

### Component Features
- **[[InternalBufferCapacity (IBC)]]** - Optimize buffer memory allocation

## Quick Reference

| Component Type | Blittable | Burst | Best For |
|----------------|-----------|-------|----------|
| [[IComponentData]] | ✅ | ✅ | General gameplay data |
| [[IBufferElementData (dynamic buffers)\|IBufferElementData]] | ✅ | ✅ | Dynamic arrays |
| [[BlobAsset (immutable data)\|BlobAsset]] | ✅ | ✅ | Shared config data |
| [[ISharedComponentData]] | ✅ | ❌ | Shared values (LOD, teams) |
| [[IEnableableComponent (toggleable components)\|IEnableableComponent]] | ✅ | ✅ | Toggle without structural change |
| [[UnityObjectRef (UnityEngine.Object references)\|UnityObjectRef]] | ✅ | ⚠️ | Unity asset references |
| [[Managed IComponentData (class components)\|Managed IComponentData]] | ❌ | ❌ | GameObject companions |

## Best Practices

1. **Prefer IComponentData** - use blittable structs whenever possible
2. **Avoid Managed Components** - they break cache performance
3. **Use IEnableableComponent** - for toggles instead of add/remove
4. **Share with BlobAssets** - for read-only configuration data
5. **Tag Components** - for cheap filtering (zero memory per entity)

