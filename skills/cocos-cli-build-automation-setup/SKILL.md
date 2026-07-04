---
name: cocos-cli-build-automation-setup
title: Cocos Creator CLI Build Automation Setup
description: Use when setting up a GUI-free CLI build pipeline for Cocos Creator 3.x.
tags: [cocos, creator, build, cli, automation, wechatgame, macOS]
  triggers:
    - cocos ci
    - build automation
    - cocos cli build

---

# Cocos Creator CLI Build Automation Setup

## When to Use

Use this skill when:
- The user wants to **stop using Cocos Creator GUI** for builds entirely
- Setting up a "one command" build (e.g., `npm run build`) for a Cocos Creator 3.x project
- Cleaning up `builder.json` — removing stale/dead scenes from the scenes array
- Fixing `project.json` UUID mismatches (UUID set to placeholder strings like `"main-scene"`)
- Creating a repeatable, documented build script for new developers or CI

## Prerequisites

- Cocos Creator 3.8.x installed at `/Applications/Cocos/Creator/3.8.x/CocosCreator.app/`
- Project is a valid Cocos Creator 3.x project with `.meta` files
- Access to terminal on macOS

## Step-by-Step: Setting Up CLI Build Automation

### Step 1: Locate the Project

Cocos projects may live in various paths. Search commonly:

```bash
find /Users -maxdepth 4 -type f -name "project.json" 2>/dev/null
```

Or use `session_search` to find past project path references.

### Step 2: Audit the Builder Config

```bash
cat profiles/v2/packages/builder.json
```

**Check these items:**

1. **`startScene`** — UUID must match `MainScene.scene` (or your actual main scene). Get the UUID from the `.meta` file:
   ```bash
   cat assets/scenes/MainScene.scene.meta | grep uuid
   ```

2. **`scenes` array** — Should contain ONLY the scene you want to build. Remove:
   - Any test/debug scenes (`Test_MeritForest`, etc.)
   - Scenes that no longer exist (files deleted but still referenced here)
   - Scenes that are not needed in the build output

3. **`outputName`** — Set to a clean name like `"wechatgame"` (not `"wechatgame-001"`, `"-005"`)

### Step 3: Fix project.json UUID

Cocos Creator 3.x's `project.json` (at project root) may have placeholder UUIDs:

```json
// ❌ BAD — "main-scene" is not a real UUID
"scenes": {
  "db://assets/scenes/MainScene.scene": {
    "uuid": "main-scene"
  }
}
```

Fix with the real UUID from the `.meta` file:

```json
// ✅ GOOD
"scenes": {
  "db://assets/scenes/MainScene.scene": {
    "uuid": "1f597e16-a4fc-43e1-bfb0-d8c96e240617"
  }
}
```

### Step 4: Create build-game.sh

Create a build script at the project root:

```bash
#!/bin/bash
set -e

PROJECT_DIR="$(cd "$(dirname "$0")" && pwd)"
COCOS_APP="/Applications/Cocos/Creator/3.8.8/CocosCreator.app/Contents/MacOS/CocosCreator"

echo "=== Build Start: $(date) ==="

# Verify CLI exists
if [ ! -f "$COCOS_APP" ]; then
    echo "ERROR: Cocos Creator not found at $COCOS_APP"
    exit 1
fi

# Run build
cd "$PROJECT_DIR"
"$COCOS_APP" --project . --build "platform=wechatgame"

if [ $? -ne 0 ]; then
    echo "BUILD FAILED (exit code: $?)"
    exit 1
fi

echo "=== BUILD SUCCESS: $(date) ==="
```

Make it executable:
```bash
chmod +x build-game.sh
```

**Variations for other platforms:**
- **Web mobile**: `"$COCOS_APP" --project . --build "platform=web-mobile"`
- **Android**: `"$COCOS_APP" --project . --build "platform=android"`
- **iOS**: `"$COCOS_APP" --project . --build "platform=iphone"`

### Step 5: Configure npm scripts

Add to `package.json`:

```json
"scripts": {
  "build": "bash build-game.sh",
  "build:wechat": "bash build-game.sh",
  "build:web": "bash build-game.sh web-mobile",
  "test": "echo \"Error: no test specified\" && exit 1"
}
```

### Step 6: Verify Everything Consistent

```bash
# Check builder.json
python3 -c "
import json
d = json.load(open('profiles/v2/packages/builder.json'))
print('startScene:', d['common']['startScene'])
print('scenes:', json.dumps(d['common']['scenes'], indent=2))
"

# Check project.json
python3 -c "
import json
d = json.load(open('project.json'))
print('startScene:', d.get('startScene'))
scenes = d.get('scenes', {})
if scenes:
    print('scenes UUID:', list(scenes.values())[0]['uuid'])
"
```

### Step 7: Teach the User

Tell the user:

```
你以後只需要一行指令就搞掂：

  npm run build

呢行指令會自動做：
1. ✅ 確認 Cocos Creator CLI 存在
2. ✅ 自動讀取 builder.json 配置
3. ✅ 用 MainScene 做 startScene 建置
4. ✅ 完成後顯示最新 build 目錄路徑

然後喺微信 DevTools 只需要 Cmd+R 刷新就得 — 完全唔使再開 Cocos Creator GUI！
```

## Pitfalls

### 🚩 Builder JSON has stale cached tasks
The `BuildTaskManager.taskMap` section in `builder.json` contains old build task records. These don't affect the build output but can be confusing. They're safe to leave; the build will use the `common` section.

### 🚩 wrong `outputName` can cause confusion old build folder
If `outputName` is `"wechatgame-005"` but user expects `"wechatgame"`, the build output goes to a different folder. Always set `outputName` to a simple clean value.

### 🚩 Cocos Creator CLI requires the Cocos app to be a valid bundle
Always verify `COCOS_APP` path exists before attempting a build. The path differs by version (3.8.8 vs 3.8.3 etc.).

### 🚩 Hard Stop: Do NOT manually edit scene JSON arrays
If you need to fix a corrupted scene file, don't manually adjust `__id__` array indices — Cocos uses them as internal pointers. Instead, rewrite the entire scene JSON or use the script-driven approach.

## Verification Checklist

- [ ] `builder.json`: `startScene` UUID matches `MainScene.scene.meta`
- [ ] `builder.json`: `scenes` array has ONLY the main scene
- [ ] `project.json`: scene UUID is a real UUID (not `"main-scene"`)
- [ ] `build-game.sh` exists and is executable
- [ ] `package.json` has `"build"` script
- [ ] `npm run build` works end-to-end (run at least once)
