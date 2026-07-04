---
name: cocos-ts-compile-check-sop
description: Use when running TypeScript compile check on Cocos Creator 3.x project.
version: 2.0.0
author: Hermes Agent
tags: [cocos-creator, typescript, compile, check, sop]
  triggers:
    - typescript compile
    - ts error
    - type error
    - compile check

---

# Cocos Creator TS Compile Check SOP

Mandatory TypeScript compile check before committing code to a Cocos Creator 3.x project.

## Command

```bash
npx tsc --noEmit
```

Must pass with **zero errors** before any commit.

## Common Errors & Fixes

| Error | Fix |
|-------|-----|
| `Cannot find module 'cc'` | Add `@cocos/creator-types` to tsconfig |
| `Property 'xxx' does not exist` | Check Cocos API version compatibility |
| `Type 'X' is not assignable` | Check Cocos 3.x type differences from 2.x |

## tsconfig.json Checklist

- [ ] `compilerOptions.target`: `"ES2015"` or higher
- [ ] `compilerOptions.module`: `"commonjs"` or `"ES2015"`
- [ ] `compilerOptions.strict`: `true` recommended
- [ ] `compilerOptions.skipLibCheck`: `true` for faster checks
- [ ] `include`: covers all `assets/**/*.ts`

See `references/tsconfig-template.md` for a complete working tsconfig.json.
