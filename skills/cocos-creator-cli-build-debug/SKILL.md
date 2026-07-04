---
name: cocos-creator-cli-build-debug
description: Use when Cocos Creator 3.x CLI build fails with scene or library errors.
version: 2.0.0
author: Hermes Agent
tags: [cocos-creator, cli, build, debug, error]
  triggers:
    - cocos build
    - build error
    - compile fail
    - build fail

---

# Cocos Creator CLI Build Debug

Debug Cocos Creator 3.x CLI build failures — scene JSON corruption, library import errors, and resource missing errors.

## Common CLI Build Errors

| Error | Likely Cause | Check |
|-------|-------------|-------|
| Error 1222/1223 | Scene JSON corrupt | `python3 -c "import json; json.load(open('assets/scenes/MainScene.scene'))"` |
| `resource.ac` not found | Asset bundle missing | Check `assets/bundle/` exists |
| Module import failed | TS compile error | Run `npx tsc --noEmit` |
| Library version mismatch | Cocos version conflict | Check `package.json` `@cocos/creator` version |

## WeChat Mini Game (wechatgame) Known Issues

### `Cannot set property 'w' of undefined` (2026-06-16)
- **Symptom**: Console shows `TypeError: Cannot set property 'w' of undefined at trt (virtual cc-XXX.js:2)`
- **Root cause**: Cocos engine rendering pipeline initialization fail → `FRAMEBUFFER_INCOMPLETE_MISSING_ATTACHMENT` in OpenGL framebuffer
- **NOT fixed by**: Disabling video/webview engine modules, delaying bootstrap init, or changing camera settings
- **Status**: Confirmed as Cocos 3.8.8 engine bug on WeChat mini game platform
- **Workaround**: Not found yet — needs Cocos official patch or engine config deep-dive
- **Evidence**: Bootstrapper mounts successfully, 7 menu buttons created, project starts — but black screen due to rendering pipeline fail

### Build Performance
- First build with full engine modules: ~42 seconds
- After disabling unused modules: ~7 seconds
- Key modules to keep for 2D zen game: base, 2d, ui, graphics, gfx-webgl, animation, audio, tween, particle-2d
- Modules safe to disable: video, webview, 3d (if 2D only), physics-3d, gfx-webgpu

### Scene Config Tips
- `project.json` `startScene` must match the actual scene UUID
- Build uses `profiles/v2/packages/builder.json` `scenes[]` list — all listed scenes get built
- `overwriteProjectSettings` in builder.json can override engine module settings
- After changing engine.json modules, run a full rebuild (not incremental)

## Build Command (CLI, no GUI needed)

```bash
/Applications/Cocos/Creator/3.8.8/CocosCreator.app/Contents/MacOS/CocosCreator --project . --build "platform=wechatgame"
```

## Recovery Flow

1. Fix any TS compile errors first
2. If scene corrupt: restore from git or rebuild in Editor
3. Clear library cache: `rm -rf library/`
4. Re-import assets: Open project in Cocos Creator (GUI)

See `references/error-catalog.md` for complete error code reference.
