---
tags:
  - component
---
**Description**

- Data [[shared]] by a group of entities.
      
- Entities with different values live in **different [[Chunk|chunks]]**.

**Example**
```csharp
public struct RenderMaterial : ISharedComponentData
{
    public Material Material;
}
```

**Pros**

- Saves memory for repeated values.
      
- Useful for grouping and filtering.

**Cons**

- Frequent changes cause [[chunk rearrangement]] (expensive).
      
- Slower filtering than [[IComponentData]].

**Best use**

- Materials, meshes, constant AI parameters.

**Avoid if**

- Frequently changing values or unique per-entity parameters.