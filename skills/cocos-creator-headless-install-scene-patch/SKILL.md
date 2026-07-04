---
name: cocos-creator-headless-install-scene-patch
description: Use when installing Cocos Creator 3.x and setting up a headless test bootstrap.
version: 2.0.0
author: Hermes Agent
tags: [cocos-creator, headless, install, bootstrap, scene, patch]
  triggers:
    - cocos headless
    - scene patch
    - headless install

---

# Cocos Creator Headless Install + Scene Patch

Install Cocos Creator 3.x without GUI, then set up a runtime test bootstrap scene.

## Headless Install

```bash
# Download latest Cocos Creator from official site:
# https://www.cocos.com/en/creator-download
# (Download link changes with each version, use the official page)
curl -L "$(curl -s https://www.cocos.com/creator-download 2>/dev/null | grep -oP 'https://download\.cocos\.com/CocosCreator/v[0-9.]+\.[0-9]+/CocosCreator-v[0-9.]+\.dmg' | head -1 || echo 'https://www.cocos.com/en/creator-download')" -o /tmp/Cocos.dmg || echo "Please download manually from https://www.cocos.com/en/creator-download"
# Mount and copy
hdiutil attach /tmp/Cocos.dmg
cp -R "/Volumes/CocosCreator/CocosCreator.app" /Applications/
hdiutil detach /Volumes/CocosCreator
```

## Bootstrap Scene Setup

Create `assets/scenes/BootScene.scene` with minimum nodes:
1. Canvas node (UI root)
2. Camera as child of Canvas (for UI rendering)
3. BootEntry component on Canvas node

See `references/scene-template.md` for the minimal BootScene JSON template.

## CLI Build

```bash
/Applications/CocosCreator.app/Contents/MacOS/CocosCreator --project . --build "platform=web-mobile;debug=true"
```

The `--headless` flag may be needed for CI environments.
