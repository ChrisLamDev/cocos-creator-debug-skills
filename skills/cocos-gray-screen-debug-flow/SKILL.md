---
name: cocos-gray-screen-debug-flow
description: Use when Cocos Creator 3.x shows gray or black screen in Preview mode.
version: 2.0.0
author: Hermes Agent
tags: [cocos-creator, debug, gray-screen, preview, serialization]
  triggers:
    - cocos gray
    - cocos black
    - 灰屏
    - 黑屏

---

# Cocos Creator 3.x Gray/Black Screen Debug Flow

Use when Cocos Creator Preview or WeChat DevTools shows a completely gray/black/blue screen with no visible UI, even though the scene loads successfully.

## Step 1: Check Symptom Type

Turn on **Show FPS** in Preview toolbar.

### Case A: FPS = 60, Draw Call = 0, Game Logic = 0ms
**Root cause:** Engine runs but script components aren't executing. See `references/case-a-script-failure.md`.

### Case B: FPS = 0 or extremely low
**Root cause:** Engine or rendering pipeline failure. See `references/case-b-render-failure.md`.

## Step 2: Run Scene Integrity Check

```bash
python3 -c "import json; data=json.load(open('assets/scenes/MainScene.scene')); print(f'Objects: {len(data)}'); [print(f'  [{i}] {o.get(\"__type__\",\"\")[:40]}', o.get('_name','')) for i,o in enumerate(data)]"
```

## Step 3: Quick Fixes (try in order)

1. **Quit Cocos Creator (Don't Save) and reopen** — clears stale cache
2. **Manually re-create Canvas node** — right-click Hierarchy → Create → UI → Canvas, add BootEntry
3. **Check Scene JSON for dead UUID references** — see `references/uuid-debug.md`
4. **Check `Camera._near > 0`** in scene JSON
5. **Set `gfx-webgl2: true`** in engine.json (macOS M1)

See `references/full-debug-flow.md` for complete detailed steps.
