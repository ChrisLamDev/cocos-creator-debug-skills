---
name: cocos-creator-scene-less-test-bootstrap
category: software-development
description: Use when creating a runtime test without a .scene file in Cocos Creator 3.x.
  triggers:
    - scene less
    - no scene
    - test without scene

---

# Cocos Creator 3.x Scene-less Test Bootstrap

Use when: you need to test Cocos Creator components at runtime but cannot (or should not) modify .scene files manually — because Creator's `.scene` JSON format does NOT support hand-written `cc.Script` components.

## Problem

Writing a `.scene` JSON file by hand and including a `"__type__": "cc.Script"` component with a `"file"` UUID reference **will not work**. Creator 3.8.x scene importer processes the scene JSON and may report `imported: true` in the `.meta`, but the Script component is silently dropped. This results in a Canvas node with no scripts attached — no `start()` is called, no UI appears in Preview.

## Solution: Manual Script Attachment via Creator Editor

**Module-level auto-injection does NOT work in Creator 3.8.8 Preview.** The module-level code runs in the Editor's `Scene` process, NOT in the Preview web view runtime. `game.on(EVENT_GAME_INITED)` and `director.on('scene-loaded')` hooks execute in the wrong context — they cannot inject components into the Preview runtime scene.

**The ONLY reliable method** is to manually attach the script component via the Creator Editor, because only Creator itself can generate the correct internal component reference:

1. **Create your Bootstrap script** as a standard `@ccclass` Component in `assets/scripts/test/`
2. **Prepare a clean 2D scene** (Canvas + orthographic Camera only — see scene file format below)
3. **Open Creator** and double-click the scene to open it
4. **Manually add a child node** to Canvas: right-click Canvas → Create Empty → rename (e.g. "TestRunner")
5. **Select the new node**, then in the Inspector panel: Add Component → search your script class name
6. **Press Preview ▶️** — Creator will correctly resolve the script reference

### Step-by-Step

### 1. Create the Bootstrap Component

Write to `assets/scripts/test/YourTestBootstrap.ts`:

```typescript
import { _decorator, Component, Node, Label, Button, UITransform, Color } from 'cc';
import { MeritEvolutionController } from '../control/MeritEvolutionController';
const { ccclass } = _decorator;

@ccclass('YourTest')
export class YourTest extends Component {
    start() {
        console.log('[YourTest] start() ✅');
        
        let parent = this.node.parent;
        const root = parent!;
        
        // IMPORTANT: Set layer on ALL new nodes to match Canvas layer
        const uiLayer = root.layer; // UI_2D = 33554432
        
        const ctrl: any = root.getComponent(MeritEvolutionController);
        if (!ctrl) root.addComponent(MeritEvolutionController);
        const controller: any = root.getComponent(MeritEvolutionController)!;
        
        // Create labeled nodes with correct layer
        const n = new Node();
        n.parent = root;
        n.layer = uiLayer;   // ← CRITICAL or camera won't see it!
        n.setPosition(x, y, 0);
        const l = n.addComponent(Label);
        l.string = text;
        // ...
        
        // Buttons also need node.layer set
        const btn = new Node();
        btn.parent = root;
        btn.layer = uiLayer;
        btn.addComponent(Button);
        // ...
        
        n.on(Node.EventType.TOUCH_END, cb, this);
    }
}
```

### 2. Create a Clean 2D Scene File

Use the official `scene-2d.scene` template (found in `CocosCreator.app/Contents/Resources/resources/3d/engine/editor/assets/default_file_content/scene/scene-2d.scene`).

Copy its structure exactly. The key elements are:

| `__id__` | Type | Notes |
|----------|------|-------|
| 1 | `cc.Scene` | Root scene |
| 2 | `cc.Node` ("Canvas") | `_layer: 33554432` (UI_2D) |
| 3 | `cc.Node` ("Camera") | Child of Canvas node 2 |
| 4 | `cc.Camera` | `_projection: 0` (orthographic), `_visibility: 33554432` |
| 5 | `cc.UITransform` | Canvas node's transform, size = design resolution |
| 6 | `cc.Canvas` | `_cameraComponent` references camera (id 4) |
| 7 | `cc.Widget` | Aligns Canvas to screen |
| 8-14 | SceneGlobals + children | Ambient, shadows, skybox, fog, octree, skin |

**DO NOT** add any `"__type__": "cc.Script"` entries — they will be silently dropped.

Use a unique `_id` for the Scene node (e.g., the same as the file's UUID). Give the scene file a `.meta` with a UUID and `"imported": false` — Creator will re-import it as `true` on load.

### 3. Configure Project Settings

Write to `settings/v2/packages/project.json`:

```json
{
  "__version__": "1.0.6",
  "general": {
    "designResolution": { "width": 960, "height": 640 },
    "startScene": "<your-scene-uuid>",
    "defaultScene": "<your-scene-uuid>"
  },
  "script": { "preserveSymlinks": true }
}
```

If you skip `startScene`, Preview will emit: `无法查到当前场景 JSON 数据(start_scene) = current_scene`.

### 4. Verify

1. Kill Creator, delete `library/` and `temp/` directories
2. Relaunch Creator with the project
3. Check `temp/logs/project.log` for:
   - `[YourTest] Game inited, starting inject retry loop...` ✅
   - No `无法查到当前场景 JSON 数据` error
4. Press Preview ▶️ in Creator

## Pitfalls

- **Module-level auto-injection does NOT work in Preview.** The `game.on(Game.EVENT_GAME_INITED, ...)` hook fires in the Editor's Scene process, not the Preview runtime. `director.getScene()` returns the Editor scene wrapper, not the Preview scene. The two processes are fully isolated — the module code simply runs in the wrong context.
- **`cc.Script` in scene JSON DOES NOT WORK.** Creator 3.8.x silently drops hand-written Script components even though `.meta` shows `imported: true`. Always attach scripts via Creator Editor manually.
- **New Node layer must match Canvas UI_2D layer.** In Creator 3.8.8 runtime, new Nodes default to `layer = 1073741824` (DEFAULT), but the orthographic Camera only has `visibility = 33554432` (UI_2D). UI nodes built in `start()` will be invisible unless `node.layer = canvas.layer` is set.
- **`require()` is not supported.** Creator 3.8.8 Preview runtime uses ES Modules. Use `import { Class } from 'module'` instead.
- **`import { Label } from 'cc'` (imported reference) may differ from `cc.Label` (global reference).** In runtime, `getComponent(Label)` with the imported Label may return null while `getComponent('Label')` (string) works. Use string lookup or `cc.Label` for safety.
- **`require()` is NOT supported in Creator 3.8.8 Preview runtime.** It throws `ReferenceError: require is not defined`. Always use ES `import` statements — `import { MeritEvolutionController } from './MeritValueState'` instead of `const X = require('./module')`.
- **New node layer mismatch is the #1 cause of invisible UI.** In Creator 3.8.8, `new Node()` defaults to `layer = 1073741824` (1<<30 = DEFAULT). The orthographic Camera has `visibility = 33554432` (1<<25 = UI_2D). UI nodes are invisible unless you set `node.layer = canvasNode.layer` (which is usually `33554432`). Always do this right after `.parent =` assignment.
- **`browser_console` via Creator's built-in preview server (port 7456) is the best runtime debug tool.** Navigate to `http://localhost:7456`, then evaluate JS in the page context to inspect the runtime scene graph:
  ```javascript
  // Check scene structure
  const scene = cc.director.getScene();
  const canvas = scene.getChildByName('Canvas');
  canvas.children.map(c => c.name + ' layer=' + c.layer);
  
  // Check camera visibility
  const cam = canvas.getChildByName('Camera').getComponent('Camera');
  {visibility: cam.visibility, orthoHeight: cam._orthoHeight, projection: cam._projection};
  
  // Check UI node details
  const child = canvas.children[2];
  child.components.map(c => ({type: c.constructor.name, string: c.string, color: c.color}));
  ```
- **Gray screen = layer mismatch 90% of the time.** Verify: `camera.visibility` must match all UI nodes' `layer`. If camera visibility is `33554432` (UI_2D) but new Nodes default to `1073741824`, the camera won't render them.
- **`require()` is NOT supported in Creator 3.8.8 Preview runtime.** It silently fails (no error in console). Always use ES `import` statements.
- **Camera `_visibility` must match Canvas node `_layer`.** Check with `camera.visibility === canvasNode.layer`. Common value: `33554432` (1 << 25 = UI_2D bit).
- **Canvas node `_layer` must be `33554432`.** Camera `_visibility` must match (`33554432` = UI_2D bitmask). The orthographic Camera must be a child of the Canvas node.
- **`game.on(Game.EVENT_GAME_INITED)` does NOT give access to the Preview runtime scene.** The event fires in Editor process where `director.getScene()` returns the Editor scene, not the Preview scene.
- **`director.on('scene-loaded', ...) does NOT fire** reliably in Preview mode.
- **Creator keeps cache in `library/` and `temp/`.** Always delete both when scene/script files change. Restarting Creator alone may not pick up changes.
- **Background color** comes from Camera's `_color` setting (`0,0,0` = black). Set it to `r:30,g:30,b:30` for dark gray or change per test needs.
- **`imported: false` in .meta** is normal — Creator sets it to `true` after successful import during startup.
- **Two Creator processes can conflict.** Running `--build` while the GUI Creator is open creates a second process. The `--build` process writes to the same `temp/` and `library/` directories, potentially corrupting the GUI instance's state.
- **`cc.ClassName` globals do NOT work in Creator 3.8.8 ES module runtime.** Writing `cc.Graphics`, `cc.Color`, `cc.Label` etc. inside component methods will throw `ReferenceError: cc is not defined` — the `cc` namespace is only available via explicit `import { ... } from 'cc'`. Always check that every `Graphics`, `Color`, `Node`, etc. used in template literal or direct reference is imported at the top of the file. SILENT FAILURE MODE: If the import line is missing one class but the file compiles (because the class is used in another imported module that re-exports it), Creator may not show an error — the component `start()` runs, console logs appear, but `addComponent('Graphics')` fails silently and no visual output is produced.
- **Black screen with console logs = silent runtime error in a dependent class.** When Creator Preview shows black screen but Bootstrap console.logs appear, the issue is likely a silent error in `start()` of a dynamically-added component (not the Bootstrap itself). Common causes:
  1. **Missing `import { X } from 'cc'`** — using `cc.X` as global instead of imported class
  2. **Circular import** — two modules import each other, Creator runtime silently drops one
  3. **File path typo** — e.g. `import './PracticeMoteCanvas'` when the file is actually named `PracticeMoteCanvas.ts` with different case in the actual filesystem path
  4. **`addComponent('ComponentName')` with string vs class ref** — prefer `addComponent(ImportedClassName)` over string names; string names can silently fail if the internal class name doesn't match (due to decorator name vs export name mismatch)
  
  **Debug protocol for black screen with logs:**
  1. Add a `Graphics` circle+rect at the END of both Bootstrap.start() AND UI.start() — this verifies the Graphics API works on this node
  2. Check that `Graphics` is in the `import { ... } from 'cc'` line of the file where it's used
  3. Verify all import paths resolve to actual files in the filesystem (case-sensitive on macOS!)
  4. Use Creator's console (not browser console) to check for red error messages — look for `Failed to load script`, `Module not found`, or `X is not defined` in the Editor Console panel
