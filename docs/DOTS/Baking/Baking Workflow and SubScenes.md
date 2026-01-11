---
tags:
  - baking
  - workflow
---
#### Description
- **Editor-time conversion process** that transforms GameObject scenes and prefabs into optimized binary entity data files

- Runs in **separate worker Unity instances** for builds, or in main Editor instance for **live-baking** (instant preview)

- **Incremental and automatic** - only rebakes when changes detected to scene, prefabs, code assemblies, or dependent assets

- SubScene is the primary way to organize baked content - binary asset loaded at runtime via streaming system

#### How Baking Works

**Key Facts:**
- **Editor-only** - baking happens during authoring, not at runtime
- **Worker instances** - builds use separate Unity processes for parallel baking
- **Live-baking** - opening SubScene checkbox in Hierarchy triggers instant preview in Editor
- **Binary output** - creates optimized entity data file (not human-readable)
- **Streaming-based loading** - SubScenes load asynchronously at runtime (requires at least 1 frame)

**What Triggers Rebaking:**
1. **SubScene content changes** - modifying GameObjects, components, or hierarchy
2. **Code changes** - modifying [[Baker (authoring conversion)|Baker]] code or component definitions
3. **Assembly changes** - recompiling assemblies with baking or component code
4. **Asset dependencies** - changes to assets referenced via `DependsOn(UnityEngine.Object)`

**What DOESN'T Trigger Rebaking:**
- Runtime changes to entities (entities are runtime copies, not linked to baked data)
- Changes to unrelated assets
- Domain reloads without code changes affecting baking

---

## Workflow Diagrams

### Editor Startup Workflow
```
1. User opens project
   ↓
2. Domain reload dirties all SubScenes
   ↓
3. SubSceneImporter runs in SEPARATE worker Unity Editor instance
   ├─ Creates baking world
   ├─ Runs all Bakers
   └─ Generates binary asset file
   ↓
4. Streaming system asynchronously loads binary asset
   ↓
5. Deserializes entities into Editor world
```

### Live-Baking Workflow
```
1. User checks "Open" on SubScene in Hierarchy
   ↓
2. Baking runs in MAIN Editor instance (not worker)
   ├─ Creates temporary baking world
   ├─ Runs all Bakers immediately
   └─ Injects baked entities into Editor world
   ↓
3. User modifies GameObject → Incremental rebaking instantly updates entities
```

**Live-Baking Limitations:**
- ❌ Behavior **cannot be reproduced in builds** without custom code
- ❌ Entities injected before systems created - may cause initialization issues
- ✅ Best for **authoring and quick iteration**

### Play Mode Workflow
```
1. User enters Play Mode
   ↓
2. Streaming system scans scenes for SubScene MonoBehaviours
   ↓
3. Locates existing SubScene binary assets
   ↓
4. Asynchronously loads binary (takes 1+ frames)
   ↓
5. Deserializes entities into play mode world
```

**Important:** SubScene streaming is **asynchronous**, entities won't appear immediately

---

## Build Workflow

### Build Process
```
1. Build starts
   ↓
2. Unity scans scenes for SubScene MonoBehaviours
   ↓
3. Collects list of referenced SubScenes
   ↓
4. Bakes all SubScenes in parallel (worker instances)
   ↓
5. Scans for weak references (EntitySceneReference, EntityPrefabReference)
   ↓
6. Repeats until no new references
   ↓
7. Generates content archives (asset bundling)
   ├─ Groups assets to minimize duplication
   └─ Ensures SubScenes only load what they need
```

### Runtime Loading (Build)
```
1. Game starts → Scene with SubScene loads
   ↓
2. Streaming system detects SubScene request
   ↓
3. Loads SubScene binary + required content archives (async, 1+ frames)
   ↓
4. Entities appear in world
```

---

## Content Archives

**Automatic Archive Generation:**
Unity creates content archives during build, optimized for ECS.

**Archive Layout Example:**
```
SubScene A: Mesh1, Mesh2, Texture1
SubScene B: Mesh2, Texture1, Texture2

Generated Archives:
├─ Archive 1: [Mesh1] - only A
├─ Archive 2: [Mesh2, Texture1] - shared by A and B
└─ Archive 3: [Texture2] - only B

Benefits: No duplicates, SubScenes only load dependencies, shared assets loaded once
```

**Weak References:**
```csharp
public struct LevelData : IComponentData
{
    public EntitySceneReference NextLevel;  // Weak reference
}

public struct Spawner : IComponentData
{
    public EntityPrefabReference EnemyPrefab;  // Weak reference
}
```
Build system scans for these and includes referenced content automatically.

---

## Baker Dependency Tracking

```csharp
public class CharacterAuthoring : MonoBehaviour
{
    public Texture2D CharacterTexture;

    public class Baker : Baker<CharacterAuthoring>
    {
        public override void Bake(CharacterAuthoring authoring)
        {
            // Track dependency - rebake if texture changes
            DependsOn(authoring.CharacterTexture);

            // If CharacterTexture modified:
            // 1. AssetDatabase detects change
            // 2. SubScene marked dirty
            // 3. Rebaking triggered on next load/play
        }
    }
}
```

---

## SubScene Loading Patterns

### Auto-Load (Default)
```
SubScene with "Auto Load" enabled
└─ Loads when scene loads (async, 1+ frames)
```

### Manual Loading
```csharp
[BurstCompile]
public partial struct SubSceneLoaderSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var sceneSystem = World.GetExistingSystemManaged<SceneSystem>();

        // Request load
        sceneSystem.LoadSceneAsync(sceneGuid);

        // Check if loaded
        if (sceneSystem.IsSceneLoaded(sceneGuid))
        {
            // SubScene entities now in world
        }
    }
}
```

### Unloading
```csharp
sceneSystem.UnloadScene(sceneGuid);  // Destroy entities
```

---

## Best Practices

**Authoring:**
- Use live-baking for iteration - check "Open" on SubScene for instant feedback
- Test with streaming - ensure game works with async SubScene loading
- Organize by usage - group related content (Level1, Level2, SharedAssets)

**Performance:**
- Avoid large SubScenes - split into smaller SubScenes for streaming flexibility
- Share common assets - weak references enable efficient asset sharing
- Profile baking time - complex baking can slow iteration, optimize Bakers

**Builds:**
- Implement IEntitySceneBuildAdditions - programmatically include SubScenes
- Verify weak references work - test referenced content loads correctly
- Check archive size - use Build Report to verify content archives are efficient

**Runtime:**
- Never assume immediate loading - SubScenes take 1+ frames to load
- Use SceneSystem.IsSceneLoaded - check before accessing SubScene entities
- Unload unused SubScenes - free memory by unloading SubScenes not in use

---

## Common Pitfalls

❌ **Assuming synchronous loading**
```csharp
sceneSystem.LoadSceneAsync(guid);
var entities = query.ToEntityArray(Allocator.Temp);  // Empty!
```

✅ **Correct async pattern**
```csharp
if (!sceneSystem.IsSceneLoaded(guid))
{
    sceneSystem.LoadSceneAsync(guid);
    return;  // Wait for next frame
}
// Now entities are available
```

❌ **Live-baking in builds**
```csharp
// This ONLY works in Editor with "Open" SubScene
// Will NOT work in builds!
```

✅ **Use SubScene streaming for builds**
```csharp
// SubScene with "Auto Load" or manual SceneSystem.LoadSceneAsync
// Works in both Editor and builds
```

