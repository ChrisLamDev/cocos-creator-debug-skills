---
name: cocos-standalone-test
description: Use when testing Cocos Creator code as a standalone HTML without the IDE.
  triggers:
    - cocos test
    - standalone test
    - cocos preview

---

# 🪷 Cocos Creator Standalone HTML 快速測試

## 背景
佛心項目嘅遊戲用 Cocos Creator 3.x 開發，需要 Cocos Creator IDE 先可以 compile TypeScript。但好多時候我哋唔方便開 IDE，或者想快速驗證邏輯先再做正式 build。

呢個 workflow 提供一個快速測試方法。

## 步驟

### 1. 確認需求
師兄想測試某個遊戲功能。如果只係邏輯層面（劇情引擎、選擇系統、功德系統等），可以用 standalone HTML 做快速驗證。

### 2. 建立測試 HTML
喺 `BuddhaHeart_Game/build/web-mobile/` 建立 `test-story.html`（或 `test-xxx.html`），做以下轉換：

**TypeScript → JavaScript 轉換規則：**
| TypeScript（Cocos） | JavaScript（Standalone） |
|---------------------|------------------------|
| `import { ... } from 'cc'` | 移除，改用純 JS |
| `@_decorator.ccclass('X')` | 移除，用 class / object |
| `@_decorator.property(...)` | 移除，直接初始化 |
| `Component, Node, Label, Button, tween` | 用 DOM API + CSS 動畫代替 |
| `this.node.emit(...)` | 用自訂 EventTarget / callback |
| TypeScript type annotation | 移除 |

### 3. 核心 UI 元素對應
| Cocos 元素 | HTML 對應 |
|-----------|-----------|
| Scene node | `<div id="app">` |
| Label | `<div>` or `<p>` with CSS font styling |
| Button | `<button>` with CSS |
| Sprite / color | CSS `background`, `color` |
| tween | CSS `transition` + `animation` |
| Button.EventType.CLICK | `element.addEventListener('click', ...)` |

### 4. 啟動測試
```bash
cd BuddhaHeart_Game/build/web-mobile && python3 -m http.server 8080
```
然後用 browser 打開 `http://localhost:8080/test-story.html`

### 5. 測試方法
- 用 browser tool 操作 UI（click 按鈕、睇文字變化）
- 用 browser_snapshot 睇 DOM 狀態（睇到文字、按鈕等內容）
- 用 browser_vision 截圖確認視覺效果（可 Send 俾師兄睇）
- 檢查邏輯是否正確（功德值計算、劇情分支、結局條件）
- 行晒所有可能路線，確保每個分支同結局都正常

### 6. 完成後
- Kill server：`process(action='kill', session_id='...')`
- 知會師兄測試結果
- 如果測試 pass，將邏輯保留返喺正式 `.ts` 檔案中
- 測試用嘅 `test-story.html` 可以保留或刪除

## No Compile, No Commit — TypeScript 強制編譯規則

Cocos Creator 項目改 code 後，commit 前必須做 TypeScript compile check：

1. 確保 `node_modules/typescript` 已安裝：`npm install typescript`
2. 確保 `types/cc.d.ts` 存在（mock Cocos 3.x 類型宣告，因 cocos-creator-3d 不在 npm registry）
3. 確保 `tsconfig.phase6.json` 存在（獨立 config，用 typeRoots 指向 types/，只 include 新 code 排除舊 code）
4. Run: `npx tsc --noEmit -p tsconfig.phase6.json`
5. **零 error 先可以 commit**。自行修復所有報錯。

常見 TS error 同修復：
| Error | 原因 | 修復 |
|-------|------|------|
| `cc.Vec3` not found | 用咗舊 Cocos 2.x `cc.xxx` 語法 | 改用 ES6 import + `Vec3` |
| `'scale' not in Partial<Node>` | Tween type 唔夠闊 | 在 cc.d.ts 加 `TweenProps` interface |
| `property X not exist` | spread 咗唔喺 interface 嘅 field | `as any` cast |
| `?? unreachable` | `?.` operator 順序問題 | 改用明確變量 + if-else |

## 注意事項
- Standalone HTML 用純 JS，**唔可以**直接用 Cocos API（`cc` namespace）
- 動畫效果會比 Cocos 版本簡陋，呢個係正常嘅
- 只適合測試邏輯，UI 精緻度要用 Cocos IDE 先做到
- 如果師兄想見到正式畫面，需要 Cocos Creator 打開 project build

## 輸出長度限制處理（關鍵技巧）

大嘅 standalone 測試頁（SocialUI + 控制邏輯，>30KB）好容易超出模型 output 上限。**解決方法：分步寫入法**

### 步驟 A：先寫 HTML 骨架 + CSS（write_file）
一次性寫 HTML head、body 結構、style block、DOM element 骨架。呢 part 約 8-15KB。

### 步驟 B：分開 patch JS adapter 部分
JS block（inline adapter、managers、控制邏輯）佔最大篇幅。用多個 `patch` 命令分段加入：
1. **第一次 patch**：SocialUI inline renderer（UI 生成函數 + CSS getter）
2. **第二次 patch**：SocialIntegration bridge（createSocialIntegration, helper functions, event binding）
3. **第三次 patch**：初始化代碼 + 快速操作按鈕 + 測試頁尾部

咁樣每個 patch 約 8-12KB，遠低於 output 上限，唔會爆長度。

### 通用規則
- 任何超過 15KB 嘅檔案，考慮分拆為「骨架 + N次 patch」
- TypeScript adapter（繞過 Cocos decorator 嘅 JS class）係最大舊嘅，要放最後 patch
- 多個 managers 各自 patch 會更穩定
