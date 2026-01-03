## Structural changes

**Operations that modify [[Entity]] [[Archetype|archetypes]]** by adding/removing components, creating/destroying entities, or changing entity structure.

Structural changes cause **[[Sync points]]** - all scheduled jobs must complete before the change executes, killing parallelization and performance.

**Common structural changes:**
- `EntityManager.AddComponent/RemoveComponent`
- `EntityManager.CreateEntity/DestroyEntity`
- `EntityManager.Instantiate`
- `EntityCommandBuffer.Playback`

**Alternatives to avoid sync points:**
- Use [[IEnableableComponent (toggleable components)|IEnableableComponent]] - toggle components on/off without structural change
- Use [[EntityCommandBuffer]] - defer changes to specific [[Sync points|sync point]]
- Batch operations - do many changes at once instead of incrementally

**See also:** [[Sync points]], [[Archetype]], [[IEnableableComponent (toggleable components)|IEnableableComponent]]
