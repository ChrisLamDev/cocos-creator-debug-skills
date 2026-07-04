# Cocos Creator Debug Skills

<div align="center">

[![GitHub stars](https://img.shields.io/github/stars/ChrisLamDev/cocos-creator-debug-skills?style=for-the-badge&logo=github)](https://github.com/ChrisLamDev/cocos-creator-debug-skills/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/ChrisLamDev/cocos-creator-debug-skills?style=for-the-badge&logo=github)](https://github.com/ChrisLamDev/cocos-creator-debug-skills/forks)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)
[![Skills](https://img.shields.io/badge/Skills-17-8A2BE2?style=for-the-badge)](skills/)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-ready-8A2BE2?style=for-the-badge&logo=anthropic)](https://claude.ai)
[![OpenAI Codex](https://img.shields.io/badge/OpenAI%20Codex-ready-8A2BE2?style=for-the-badge&logo=openai)](https://openai.com)
[![Cursor](https://img.shields.io/badge/Cursor-ready-8A2BE2?style=for-the-badge&logo=cursor)](https://cursor.sh)
[![Hermes Agent](https://img.shields.io/badge/Hermes%20Agent-ready-8A2BE2?style=for-the-badge&logo=python)](https://github.com/NousResearch/hermes-agent)

</div>

**Your Cocos Creator 3.x keeps crashing with gray screens? Been there.**

## 📸 Demo

<div align="center">
  <img src="screenshots/demo.svg" alt="cocos-creator-debug-skills overview" width="100%">
  <br><br>
  <img src="screenshots/demo-real.png" alt="cocos-creator-debug-skills live demo" width="90%" style="border-radius: 8px; border: 1px solid #30363d;">
  <br>
  <em>Cocos Creator gray-screen-debug-flow terminal output — step-by-step scene file integrity, component bindings, and render pipeline diagnostics</em>
</div>

<br>



This collection of 17 executable AI agent skills documents every Cocos Creator 3.x bug I've encountered and the exact steps to fix them — gray screens, black previews, corrupt scene files, code-driven UI rendering issues, CLI build failures, and more.

Each skill is a proven workflow your AI agent can load and execute immediately.

## 💡 Why These Skills?

Cocos Creator 3.x debugging is trial-and-error — gray screens, corrupt scenes, broken component bindings take hours to diagnose. You check the camera, then the canvas, then the render pipeline, then search five GitHub issues, then re-import the project. Every Cocos developer has burned a full afternoon on a problem that turned out to be a single missing UUID or a `_near` value set to zero.

These 17 skills encode hard-won debugging patterns so your AI agent fixes in seconds instead of hours. Each one captures a real bug we fixed — the exact symptoms, root cause, and the precise sequence of commands and checks that resolved it. Hand your agent a skill file and it knows exactly where to look, what to run, and how to interpret the output. No more context-switching between the Cocos docs, Stack Overflow, and your terminal.

## You've definitely been here:

```
- Gray/black screen on Play and you have no idea if it's the camera, canvas, or rendering pipeline
- Code-driven UI nodes render as gray boxes with no text
- ScrollView content never displays properly
- Scene file corrupts with Error 1222/1223 and you're about to redo everything
- CLI build fails but the error message tells you nothing useful
- Standalone test requires opening the full Cocos Creator IDE
```

## Highlight Skills

| Skill | One-liner |
|-------|-----------|
| **cocos-gray-screen-debug-flow** | Step-by-step gray/black screen diagnosis, from common to obscure |
| **cocos-dynamic-ui-panel** | Build full UI panels in pure TypeScript — no scene editor needed |
| **cocos-nirvana-rebuild-scene** | Recover from corrupt scene Error 1222/1223 |
| **cocos-creator-cli-build-debug** | Debug CLI build failures scene-by-scene |
| **cocos-code-driven-scrollview-ui** | Scrollable leaderboards and long lists in code |
| **cocos-graphics-mote-system** | Replace sprite particles with procedural graphics |
| **cocos-standalone-test** | Test Cocos code as standalone HTML — no IDE required |

## The 17 Skills

| # | Skill | Problem It Solves |
|---|-------|-------------------|
| 1 | **cocos-creator-install-and-debug** | New install scene import errors — version conflicts + plugin issues |
| 2 | **cocos-creator-cli-build-debug** | CLI build fails — scene/library/module errors one-by-one |
| 3 | **cocos-creator-headless-install-scene-patch** | Headless mode setup without GUI |
| 4 | **cocos-nirvana-rebuild-scene** | Corrupt scene Error 1222/1223 — full recovery flow |
| 5 | **cocos-gray-screen-debug-flow** | Gray/black screen — camera → canvas → rendering pipeline |
| 6 | **cocos-gray-screen-camera-canvas-fix** | Camera gray screen — config + canvas + render texture |
| 7 | **cocos-dynamic-ui-panel** | Full UI panels in pure TypeScript, no scene editor |
| 8 | **cocos-dynamic-ui-layer-color-font** | UI nodes as gray boxes with no text — AddChild timing + UIOpacity |
| 9 | **cocos-code-driven-scrollview-ui** | ScrollView content not displaying fully |
| 10 | **cocos-code-driven-ui-common-pitfalls** | Coordinate system + Anchor + parent-child timing |
| 11 | **cocos-scene-json-component-binding** | Component-scene binding failures |
| 12 | **cocos-graphics-visual-prototyping** | Zero-asset visual prototyping — Graphics component |
| 13 | **cocos-graphics-mote-system** | Replace sprite particles with procedural rendering |
| 14 | **cocos-standalone-test** | Test Cocos code as standalone HTML, no IDE |
| 15 | **cocos-ts-compile-check-sop** | TypeScript compile check — `tsc --noEmit` |
| 16 | **cocos-cli-build-automation-setup** | GUI-free build pipeline for CI/CD |
| 17 | **cocos-creator-scene-less-test-bootstrap** | Runtime test bootstrap without .scene files |

## 🧬 Skill Anatomy

Each skill lives in its own directory under `skills/` and follows a consistent file format that AI agents can parse and execute programmatically.

```
skills/
├── cocos-gray-screen-debug-flow/
│   ├── SKILL.md                    ← main skill file (required)
│   └── references/                 ← supporting docs (optional)
│       ├── full-debug-flow.md
│       └── uuid-debug.md
├── cocos-dynamic-ui-panel/
│   └── SKILL.md
├── cocos-nirvana-rebuild-scene/
│   └── SKILL.md
├── ...
```

Every `SKILL.md` uses this structure:

**YAML frontmatter** — agent-readable metadata for discovery and routing:

```yaml
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
```

**Markdown body** — human-readable, step-by-step instructions with numbered steps, case branches, inline commands, and references to deeper docs:

```markdown
# Cocos Creator 3.x Gray/Black Screen Debug Flow

## Step 1: Check Symptom Type
Turn on **Show FPS** in Preview toolbar.

### Case A: FPS = 60, Draw Call = 0, Game Logic = 0ms
**Root cause:** Engine runs but script components aren't executing.
See `references/case-a-script-failure.md`.

### Case B: FPS = 0 or extremely low
**Root cause:** Engine or rendering pipeline failure.
See `references/case-b-render-failure.md`.
```

The `triggers` field in the frontmatter is the agent's entry point — when you say "gray screen" or "black screen" in a Cocos Creator context, the agent matches these keywords, loads the matching `SKILL.md`, and follows the steps. The numbered steps are written as pass/fail checkpoints so the agent can report progress and loop back if a fix doesn't take.

## Installation

```bash
# Hermes Agent
# Copy to your skills directory:
cp -r skills/* ~/.hermes/skills/

# Claude Code
cp -r skills/* ~/.claude/skills/

# Cursor
cp -r skills/* ~/.cursor/skills/
```

## Compatibility

- **Cocos Creator:** 3.x (tested on 3.6+)
- **AI Agents:** Hermes Agent, Claude Code, Cursor, any agent with skill-loading support
- **Platform:** macOS (some skills Windows-compatible)

## License

MIT — use freely, contribute back when you can.

---

> Built from real Cocos Creator debugging battles. Every skill here documents an actual bug we fixed.
