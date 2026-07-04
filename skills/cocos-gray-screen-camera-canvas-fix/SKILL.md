---
name: cocos-gray-screen-camera-canvas-fix
description: Use when Cocos Creator shows gray/blue screen on Play or Preview.
version: 2.0.0
author: Hermes Agent
tags: [cocos-creator, gray-screen, camera, canvas, preview]
  triggers:
    - cocos gray
    - cocos black
    - 灰屏
    - 黑屏

---

# Cocos Gray Screen — Camera + Canvas Fix

Systematic debug for Cocos Creator gray/blue screen — covers Camera and Canvas configuration issues.

## Quick Check

```bash
# 1. Check Camera._near in scene JSON
python3 -c "import json; d=json.load(open('assets/scenes/MainScene.scene')); c=[o for o in d if 'Camera' in str(o.get('_type',''))]; print([(o.get('_name'), o.get('_near')) for o in c])"

# 2. Check Canvas exists in Hierarchy (not just in JSON)
# Open Cocos Creator → Hierarchy panel → look for Canvas node

# 3. Check gfx-webgl2 is enabled
cat settings/engine.json | python3 -c "import sys,json; print(json.load(sys.stdin).get('gfx-webgl2'))"
```

## Common Pattern: Canvas in JSON but not in Editor

If Canvas node exists in `MainScene.scene` JSON but is not visible in Editor Hierarchy:
- Scene serialization corruption
- UUID mismatch on component reference
- **Fix**: Manually create Canvas via Editor (right-click → Create → UI → Canvas)

See `references/deep-diagnostics.md` for comprehensive debugging.
