---
name: cocos-creator-install-and-debug
description: Use when installing Cocos Creator 3.x and debugging scene import errors.
version: 2.0.0
author: Hermes Agent
tags: [cocos-creator, install, macos, scene-import, debug]
  triggers:
    - install cocos
    - cocos setup
    - cocos install

---

# Cocos Creator 3.x Install & Debug

Install Cocos Creator 3.x on macOS from terminal and fix common scene import errors.

## Install

```bash
# Download from official site
open "https://www.cocos.com/en/creator-download"
# Or use direct .dmg
open /path/to/CocosCreator_v3.x.dmg
```

Install to `/Applications/CocosCreator.app`.

## Common Scene Import Errors

| Error | Likely Cause | Quick Fix |
|-------|-------------|-----------|
| Error 1222/1223 | Corrupted scene JSON | Rebuild scene from backup |
| `scene._load is not a function` | Broken __id__ references | Don't manually edit scene JSON |
| Gray screen on preview | Script component not loaded | Check Canvas node exists in Hierarchy |
| Missing textures | Asset bundle not imported | Re-import assets in Editor |

## Debug Flow

1. Check Console in Cocos Creator (Window → Console)
2. Verify scene loads: `LoadScene MainScene.scene: XXms` in output
3. Read scene JSON integrity: `python3 -c "import json; json.load(open('assets/scenes/MainScene.scene'))"`
4. If scene corrupt, restore from git backup or rebuild

See `references/full-debug.md` for complete error catalog and fixes.
