In DOTS, **[[chunk]] rearrangement** means Unity moves entities between [[chunks]] in memory when their **[[archetype]] changes** (i.e., the set of components they have changes).
#### Why rearrangement happens
If an entity **adds or removes a [[component]]**, it no longer fits in its current chunk’s [[archetype]].  
Unity must:
1. **Copy** that entity’s data from the old [[chunk]] to a [[chunk]] of the new [[archetype]].
    
2. **Remove** it from the old [[chunk]].
    
3. Possibly **reorder** the other entities in that [[chunk]] to keep it packed.
#### Example
Initial state:
```css
Chunk A: Archetype {Position, Velocity}
[Entity1] [Entity2] [Entity3]
```

You add `Health` to `Entity2` →

- Old archetype: `{Position, Velocity}`
    
- New archetype: `{Position, Velocity, Health}`

Result:
```css
Chunk A: Archetype {Position, Velocity}
[Entity1] [Entity2]

Chunk B: Archetype {Position, Velocity, Health} 
[Entity2]
```
#### Why it matters
- **Performance cost**: Moving entities between [[chunk|chunks]] is [[structural changes|structural change]] — relatively expensive because of memory copy + possible [[chunk]] allocations.
    
- **Cache impact**: Moving entities might break sequential processing order if your queries expect certain memory locality.
    
- **Job safety**: [[Structural changes]] can’t happen while jobs are reading/writing those [[Entity|entities]], so Unity adds [[sync points]] (stalls).
#### When you’ll see chunk rearrangement
- Adding/removing components.
    
- Instantiating or destroying entities.
    
- Converting from prefab archetype to runtime archetype.
#### 💡Tip
 Enabling/disabling [[IEnableableComponent (toggleable components)|IEnableableComponent]] changes the _enabled mask_ but **does not** cause rearrangement — faster!.