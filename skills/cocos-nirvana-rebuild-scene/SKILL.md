---
name: cocos-nirvana-rebuild-scene
description: Use when Cocos Creator scene files are corrupted with Error 1222/1223.
  triggers:
    - scene corrupt
    - rebuild scene
    - cocos nirvana

---

# Cocos 3.x 涅槃重生 — 重建損壞 Scene

## 何時使用
- Cocos Creator 3.x Build fail with `resource.activate is not a function`
- Error 1222: "Failed to initialize render pipeline"  
- Error 1223: "Custom pipeline and legacy pipeline are all culled"
- `Open scene failed: UUID` error in Editor
- `Build task is break!` with deserialization errors
- bundle.js 得 3,414 bytes（正常 200KB+）

## 前置條件
1. 確認 correct project path（唔係 Desktop backup，係 Hermes workspace 內）
2. 確認 Cocos Creator 3.x（目前用 3.8.8）
3. GameBootstrapper.ts 存在（或你嘅 boot script）

## 步驟

### Step 1: 檢查 MainScene.scene JSON 結構
```bash
cat assets/scenes/MainScene.scene | python3 -m json.tool 2>/dev/null | head -5
```
如果 print 到 JSON 即係結構 intact。如果 fail 即係極端損壞。

### Step 2: 讀取 MainScene.scene 做格式參考
MainScene.scene 係 Cocos 3.x JSON 格式，用 `__id__` reference system：
- 每個 `__id__` 值 = array index
- `__id__` numbering 必須連續無 gap
- Sub-object（Vec3, Quat, Color 等）有 `__id__: null`，唔佔 numbering

```
array[0] = cc.SceneAsset（scene asset entry point）
array[1] = cc.Scene（`_children`, `_globals` reference）
array[2] = cc.Node（Canvas 或 root node）
array[3] = cc.Node（Main Camera）
array[4] = cc.SceneGlobals（`ambient`, `fog`, `shadows`, `skybox`, `skins`）
array[5] = cc.UITransform（Canvas 嘅 transform）
array[6] = cc.Canvas（Canvas component）
...
```

### Step 3: 建立全新 ZenScene.scene
寫入 JSON 陣列，結構：
```json
[
  // [0] cc.SceneAsset
  {"__type__": "cc.SceneAsset", "_name": "", "_objFlags": 0, "native": "",
   "scene": {"__id__": 1}},
  
  // [1] cc.Scene
  {"__type__": "cc.Scene", "_name": "ZenScene", "_objFlags": 0,
   "autoReleaseAssets": false,
   "_children": [{"__id__": 2}, {"__id__": 3}],
   "_globals": {"__id__": 4},
   "_inited": true},
  
  // [2] Canvas node
  {"__type__": "cc.Node", "_name": "Canvas", "_objFlags": 0, ...},
  
  // [3] Main Camera node
  
  // [4] SceneGlobals
  
  // [5] UITransform
  
  // [6] Canvas component
  
  // [7] GameBootstrapper custom script component
  {"__type__": "cc.Script", "_name": "", "_objFlags": 0,
   "_serializedDependencies": [],
   "_scriptAsset": {"__uuid__": "<GameBootstrapper UUID>"},
   "_enabled": true}
]
```

### Step 4: Custom Script Component 正確格式
```json
{"__type__": "cc.Script", "_name": "", "_objFlags": 0,
 "_serializedDependencies": [],
 "_scriptAsset": {"__uuid__": "df04b42a-58f5-41c9-9980-bc4605129b94"},
 "_enabled": true}
```

**關鍵 rule**: `__id__` = array index，必須連續由 0 開始無 gap。

### Step 5: 更新所有設定指向 ZenScene
更新:
- `profiles/v2/packages/builder.json` → startScene + scene list
- `profiles/v2/packages/wechatgame.json` → scene list
- `settings/v2/packages/builder.json` → startScene
- `settings/v2/packages/project.json` → scene list
- `settings/packages/project.json` → defaultScene

### Step 6: 清理舊 scenes
```bash
rm assets/scenes/MainScene.scene
rm assets/scenes/MainScene.scene.meta
```

### Step 7: Build
```bash
rm -rf build temp library
npm run build
```

### Step 8: 驗證
```bash
ls -la build/wechatgame/src/chunks/bundle.js
# 應該 > 200KB
```

## 常見錯誤
1. **`Cannot read properties of undefined (reading '__type__')`** → 某個 `__id__` reference 指向咗唔存在嘅 array index。
2. **`resource.activate is not a function`** → SceneGlobals shadows/skybox reference 錯誤。
3. **bundle.js 仍得 3.4KB** → 嘗試 GUI Editor Build（行為可能不同）。
4. **`glCheckFramebufferStatus() - FRAMEBUFFER_INCOMPLETE_MISSING_ATTACHMENT`** → OpenGL warning，可忽略。

## 參考
- MainScene UUID: 1f597e16-a4fc-43e1-bfb0-d8c96e240617
- GameBootstrapper UUID: df04b42a-58f5-41c9-9980-bc4605129b94
- Project path: `<your_project_dir>/BuddhaHeart_Game`
