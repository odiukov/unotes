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

## Editor Baking Workflow

### Initial Editor Startup

```
1. User opens project (first time or after code change)
   ↓
2. Domain reload dirties all SubScenes
   ↓
3. Editor world attempts to load SubScenes with "Auto Load" enabled
   ↓
4. Since SubScene data invalidated, streaming system invokes SubSceneImporter
   ↓
5. SubSceneImporter runs in SEPARATE worker Unity Editor instance
   ├─ Creates baking world
   ├─ Runs all built-in and user Baker code
   ├─ Generates optimized entity data
   └─ Creates binary asset file
   ↓
6. Streaming system detects binary asset available
   ↓
7. Asynchronously loads binary asset
   ↓
8. Deserializes entities into Editor world
   ↓
9. Clone of baked SubScene appears in world (visible in Entity Debugger)
```

### Live-Baking Workflow

```
1. User checks "Open" checkbox on SubScene in Hierarchy
   ↓
2. Baking runs in MAIN Editor instance (not worker)
   ├─ Creates temporary baking world
   ├─ Runs all Bakers immediately
   └─ Injects baked entities into Editor world
   ↓
3. Entities appear BEFORE systems are created (injected early)
   ↓
4. User modifies GameObject in SubScene
   ↓
5. Incremental rebaking triggered (only affected entities)
   ↓
6. Editor world updated with new entity data instantly
```

**Live-Baking Limitations:**
- ❌ Behavior **cannot be reproduced in builds** without custom code
- ❌ Entities injected before systems created - may cause initialization issues
- ✅ Best for **authoring and quick iteration**
- ✅ Use for **visualizing baking results** while editing

### Play Mode Workflow (Pre-Baked SubScenes)

```
1. User enters Play Mode
   ↓
2. Default World starts update loop
   ↓
3. Streaming system scans loaded scenes for SubScene MonoBehaviours
   ↓
4. Locates existing SubScene binary assets
   ↓
5. Asynchronously loads binary assets (takes at least 1 frame)
   ↓
6. Deserializes entities into play mode world
   ↓
7. Clone of baked SubScene appears in world
```

**Important:** SubScene streaming is **asynchronous**, entities won't appear immediately on first frame

---

## Build Workflow

### Build Process

```
1. Build starts
   ↓
2. Unity scans all included GameObject scenes for SubScene MonoBehaviours
   ↓
3. Collects list of referenced SubScenes
   ↓
4. Runs IEntitySceneBuildAdditions implementations (custom additions)
   ↓
5. Bakes all SubScenes in list
   ├─ Parallel baking in worker Unity instances
   ├─ Each SubScene baked independently
   └─ Generates binary entity data files
   ↓
6. Scans baked SubScenes for weak references (EntitySceneReference, EntityPrefabReference)
   ↓
7. Adds weakly referenced SubScenes to baking list
   ↓
8. Repeats steps 5-7 until no new references found
   ↓
9. Generates content archive layout (asset bundling)
   ├─ Groups assets to minimize duplication
   ├─ Ensures SubScene only loads what it needs
   └─ Creates archive files
   ↓
10. Build complete
```

### Runtime Loading (Build)

```
1. Game starts
   ↓
2. Scene with SubScene MonoBehaviour loads
   ↓
3. Streaming system detects SubScene request
   ↓
4. Loads SubScene binary + required content archives
   ├─ Asynchronous loading (1+ frames)
   ├─ Loads shared content archives if not already loaded
   └─ Decompresses and deserializes entity data
   ↓
5. Entities appear in world
```

---

## Content Management and Archives

**Automatic Archive Generation:**
Unity automatically creates content archives during build, similar to AssetBundles but optimized for ECS.

**Archive Layout Example:**
```
SubScene A references: Mesh1, Mesh2, Texture1
SubScene B references: Mesh2, Texture1, Texture2

Generated Archives:
├─ Archive 1: [Mesh1] - only SubScene A
├─ Archive 2: [Mesh2, Texture1] - shared by A and B
└─ Archive 3: [Texture2] - only SubScene B

Benefits:
- No duplicate assets
- SubScenes only load what they need
- Shared assets loaded once
```

**Content Archive Rules:**
1. **SubScene only loads its dependencies** - won't load unused assets
2. **Assets grouped for efficiency** - shared assets in common archives
3. **Automatic deduplication** - same asset referenced multiple times stored once

**Weak References:**
```csharp
// EntitySceneReference - references another SubScene
public struct LevelData : IComponentData
{
    public EntitySceneReference NextLevel;  // Weak reference
}

// EntityPrefabReference - references prefab from another SubScene
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
    public Material CharacterMaterial;

    public class Baker : Baker<CharacterAuthoring>
    {
        public override void Bake(CharacterAuthoring authoring)
        {
            // Track texture dependency - rebake if texture changes
            DependsOn(authoring.CharacterTexture);

            // Track material dependency
            DependsOn(authoring.CharacterMaterial);

            // If CharacterTexture or CharacterMaterial is modified:
            // 1. AssetDatabase detects change
            // 2. SubScene marked dirty
            // 3. Rebaking triggered on next load/play
        }
    }
}

// Trigger rebake programmatically
EditorUtility.SetDirty(textureAsset);
AssetDatabase.SaveAssetIfDirty(textureAsset);
// Next SubScene load will trigger rebake
```

---

## SubScene Loading Patterns

### Auto-Load (Default)
```
SubScene in scene with "Auto Load" enabled
└─ Loads when scene loads
└─ Asynchronous (1+ frames)
```

### Manual Loading
```csharp
[BurstCompile]
public partial struct SubSceneLoaderSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Request SubScene load
        var sceneSystem = World.GetExistingSystemManaged<SceneSystem>();
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
// Unload SubScene and destroy its entities
sceneSystem.UnloadScene(sceneGuid);
```

---

## Best Practices

**For Authoring:**
1. **Use live-baking for iteration** - check "Open" on SubScene for instant feedback
2. **Test with streaming** - ensure game works with async SubScene loading
3. **Organize by usage** - group related content in SubScenes (Level1, Level2, SharedAssets)

**For Performance:**
1. **Avoid large SubScenes** - split into smaller SubScenes for streaming flexibility
2. **Share common assets** - weak references enable efficient asset sharing
3. **Profile baking time** - complex baking can slow iteration, optimize Bakers

**For Builds:**
1. **Implement IEntitySceneBuildAdditions** - to programmatically include SubScenes
2. **Verify weak references work** - test that referenced content loads correctly
3. **Check archive size** - use Build Report to verify content archives are efficient

**For Runtime:**
1. **Never assume immediate loading** - SubScenes take 1+ frames to load
2. **Use SceneSystem.IsSceneLoaded** - check before accessing SubScene entities
3. **Unload unused SubScenes** - free memory by unloading SubScenes not in use

## Common Pitfalls

❌ **Assuming synchronous loading**
```csharp
sceneSystem.LoadSceneAsync(guid);
// Entities NOT available yet!
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

## See Also

- [[Baker (authoring conversion)]] - Writing conversion code
- [[TransformUsageFlags]] - Transform component control
- [[BlobAsset (immutable data)]] - Efficient shared data in baking
