---
name: cocos-scene-json-component-binding
description: Use when binding code-driven TypeScript components in Cocos Creator scenes.
  triggers:
    - scene json
    - component bind
    - cocos scene
    - ts component

---

# Cocos Scene JSON Component Binding

正確將 code-driven TypeScript Component 綁定到 Scene JSON 的方法，唔開 Editor 都做到。

## When to use

- User 唔想/唔可以操作 Cocos Creator Scene Editor GUI
- Scene JSON 需要掛載 code-driven Component（如 GameBootstrapper）
- Build 出嚟場景加載成功但 Component 冇執行（冇 console.log 輸出）
- Error 3817「Canvas component can only have one instance」出現

## Scene JSON Structure

Cocos Creator 3.x 嘅 Scene JSON 係一個 array of objects，用 `__id__` 做 cross-reference：

```json
[
  { "__id__": 0 },  // SceneAsset (entry point)
  { "__id__": 1 },  // Scene root node
  { "__id__": 2 },  // Child node (e.g. Canvas / Bootstrapper)
  { "__id__": 3 },  // Component on node 2
  ...
]
```

### Key reference pattern:
- **SceneAsset** (index 0): `"scene": { "__id__": 1 }` → points to Scene node
- **Scene node** (index 1): `"_children": [{ "__id__": 2 }]` → lists child nodes
- **Child node** (index N): `"_parent": { "__id__": 1 }`, `"_components": [{ "__id__": M }]`
- **Component** (index M): `"node": { "__id__": N }` → points back to its node

## Step-by-step: Adding a Component to Scene JSON

### 1. Find the component's UUID
Check the `.meta` file of your TypeScript file:
```bash
cat assets/scripts/control/GameBootstrapper.ts.meta
# => { "uuid": "df04b42a-58f5-41c9-9980-bc4605129b94", ... }
```

### 2. Add the component __id__ reference to the target node
In the node's `_components` array, add:
```json
"_components": [
  { "__id__": 5 },    // existing component (e.g. Canvas)
  { "__id__": 15 }    // new component reference — pick unused __id__
]
```

### 3. Append the component object to the array
At the end of the Scene JSON array:
```json
{
  "__type__": "df04b42a-58f5-41c9-9980-bc4605129b94.GameBootstrapper",
  "_name": "GameBootstrapper",
  "_objFlags": 0,
  "node": { "__id__": 2 },
  "_enabled": true,
  "__prefab": null,
  "_id": "gameBootstrapper"
}
```

### 4. Rebuild and test
```bash
npm run build
# Then open in DevTools / simulator
```

## ⚠️ Critical Pitfalls

### Error 3817: Canvas component can only have one instance
**Cause**: WeChat Mini Game's Cocos engine **automatically creates a Canvas** during initialization. Adding a `Canvas` component manually in Scene JSON or via `addComponent(Canvas)` triggers this error.

**Fix**: 
1. Do NOT put `Canvas` node in Scene JSON
2. Do NOT call `this.node.addComponent(Canvas)` in code
3. Let the Bootstrapper Component live on a plain `Node` (not Canvas)
4. All UI nodes are dynamically created children of the Bootstrapper node

### Scene MUST have an entry Component
If the Scene JSON has no component that runs code, the scene loads but nothing executes. Console shows `LoadScene ...: 20ms` but no app logs.

**Fix**: Always ensure at least one node in Scene JSON has a Component that runs `onLoad()`/`start()`.

### __id__ collision
If you use an `__id__` already used by another object, Cocos will either crash or silently fail.

**Best practice**: Use high numbers (e.g., 15, 20) for manually-added components to avoid collision with auto-generated IDs (usually 0-14 for simple scenes).

### Camera position for auto-created Camera
If your code dynamically creates a Camera (child of the Bootstrapper node), set `z` position to 1000:
```typescript
camNode.setPosition(v3(0, 0, 1000));
```
Otherwise the camera might be behind or inside the scene geometry.

## Verification checklist
After binding a component to Scene JSON:
1. [ ] `npm run build` succeeds
2. [ ] DevTools console shows component's `onLoad` log
3. [ ] No Error 3817 in console
4. [ ] `LoadScene` message appears with scene path
5. [ ] UI elements render in simulator

## Related
- `cocos-gray-screen-camera-canvas-fix` — gray/blue screen debugging flow
- `cocos-creator-scene-less-test-bootstrap` — fully code-driven test bootstrap
- `cocos-code-driven-ui-common-pitfalls` — UITransform, font, layer issues
