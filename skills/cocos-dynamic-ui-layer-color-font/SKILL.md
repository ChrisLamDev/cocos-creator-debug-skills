---
name: cocos-dynamic-ui-layer-color-font
category: software-development
description: Use when dynamic UI nodes in Cocos Creator appear as gray/blue boxes.
  triggers:
    - cocos gray ui
    - ui not show
    - layer color
    - font not show

---

# Cocos Creator 3.x Dynamic UI 修復三部曲

## 症狀

用 `new Node()` 動態生成嘅 UI 節點顯示為暗黑剪影、文字極暗黑灰（只有 Emoji 有顏色），或純粹睇唔到字。

## 根因三部曲（必須做齊）

### 第一步：Layer — 強制設定 UI_2D

Cocos Creator 3.x 所有 `new Node()` 預設 layer 係 **DEFAULT（3D 層）**，被 3D Camera 渲染，失去 UI 2D 專屬材質與混色。

```typescript
import { Layers } from 'cc';

let node = new Node('MyUINode');
node.layer = Layers.Enum.UI_2D;  // 必須設定！

// 所有子節點會 inherit parent layer，所以 parent node set 咗 layer 就夠
```

### 第二步：Color — 必須用 `new Color(r, g, b, 255)`，唔可以用 Hex string

Cocos Creator 3.x 嘅 `Label.color` 同 `Sprite.color` **唔可以直接接受 Hex string**（`'#FFFFFF'`）或者 ThemeManager 嘅 hex constant。

```typescript
import { Color, Label } from 'cc';

let label = node.getComponent(Label) || node.addComponent(Label);

// ✅ 正確
label.color = new Color(255, 255, 255, 255);  // 純白
label.color = new Color(255, 215, 0, 255);     // 金色

// ❌ 錯誤 — 靜默失敗，文字變黑或透明
label.color = '#FFFFFF';
label.color = TEXT_DEEP_INK;  // ThemeManager hex constant
label.color = someHexString;
```

**注意：** `Color.fromHEX()` 喺某啲 Cocos 版本可能都有 bug，最穩陣係直接用 `new Color(r, g, b, 255)`。

### 第三步：useSystemFont — 確保 Label 有字體

純 code 生成嘅 Label 如果冇 set `useSystemFont = true`，可能會因為搵唔到 Font Asset 而唔渲染。

```typescript
let label = node.getComponent(Label) || node.addComponent(Label);
label.useSystemFont = true;  // 必須設定！
label.string = "要顯示嘅文字";
label.color = new Color(255, 255, 255, 255);
```

## 完整範例

```typescript
import { Node, Label, Color, Layers, UITransform, Size } from 'cc';

function createUILabel(parent: Node, text: string): Node {
    let node = new Node('Label');
    node.layer = Layers.Enum.UI_2D;     // ✅ Step 1
    
    let label = node.addComponent(Label);
    label.useSystemFont = true;         // ✅ Step 2
    label.string = text;
    label.color = new Color(255, 255, 255, 255);  // ✅ Step 3
    
    let transform = node.addComponent(UITransform);
    transform.contentSize = new Size(200, 40);
    
    parent.addChild(node);
    return node;
}
```

## 排查 checklist（由最高機率排列）

1. [ ] 所有 `new Node()` 嘅 UI 節點有冇 `layer = Layers.Enum.UI_2D`？
2. [ ] 所有 `label.color = xxx` 係用 `new Color(r, g, b, 255)` 定係 hex string？
3. [ ] 所有 Label 有冇 `useSystemFont = true`？
4. [ ] 如果用咗 `Color.fromHEX()`，有冇換做 `new Color()` 試過？
5. [ ] Scene 嘅 Canvas node 嘅 Layer 係唔係 `UI_2D`？

## 關聯檔案（BuddhaHeart_Game 項目）

- `assets/scripts/control/GameBootstrapper.ts` — 主入口 UI
- `assets/scripts/control/CollectivePracticeHallUI.ts` — 共修大廳（第一個出現嘅 UI）
- `assets/scripts/control/*.ts` — 其他 UI 檔案都需要 check
- `assets/scripts/core/StoryEngineUI.ts` — 禪修室 UI
