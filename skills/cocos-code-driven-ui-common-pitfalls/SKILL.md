---
name: cocos-code-driven-ui-common-pitfalls
description: Use when code-driven UI nodes in Cocos Creator 3.x fail to render text or colors.
  triggers:
    - cocos ui bug
    - ui not appear
    - code driven ui
    - dynamic node

---

# 🪷 Cocos Creator 3.x 純代碼驅動 UI 常見陷阱排錯

## 背景
佛心項目使用全 code-driven 方式建立 UI（無 Editor 綁定），`new Node()` + `addComponent(Label)` 動態生成。這種方式容易遇到幾個 Cocos 3.x 特有的坑。

## 必檢查清單（按優先級）

### 1. ✅ Layer 必須明確設置為 UI_2D
動態 `new Node()` 預設的 layer 是 **DEFAULT**（即 3D 層），會被 Main Camera 以 3D 方式渲染，導致 UI 文字變成暗黑/灰色剪影。

```typescript
const UI_2D_LAYER = 1 << 25; // 33554432

const myNode = new Node('MyText');
myNode.layer = UI_2D_LAYER; // ← 每個動態 UI Node 都必須加！
```

**重要：** 子節點不會繼承父節點的 layer！每個 `new Node()` 都要單獨設置，即使父節點已經是 UI_2D。

### 2. ✅ 所有 Label 必須加 useSystemFont = true
動態生成的 Label 如果沒有指定 font asset，**可能完全不會渲染文字**（顯示為空白或暗黑）。

```typescript
const label = node.addComponent(Label);
label.string = '文字內容';
label.useSystemFont = true; // ← 必須加！
label.fontSize = 20;
```

### 3. ✅ Label.color 必須用 new Color(r, g, b, a) 直接賦值
`Color.fromHEX()` 在 Cocos 3.8.8 中可能有不穩定的問題。最穩妥的寫法是直接使用 `new Color()` 實例化。

```typescript
// ✅ 正確 (強烈推薦)
label.color = new Color(255, 255, 255, 255); // 純白
label.color = new Color(255, 215, 0, 255);   // 金色

// ❌ 可能失敗
label.color = this._hexToColor('#FFFFFF');

// ❌ 絕對錯誤 — 不能直接賦值 hex string
label.color = '#FFFFFF';
```

### 4. ⚠️ 場景 JSON 不可手寫 — 極高風險！(2026-06-03 Update)

Cocos 3.x 的 `.scene` JSON 是一個 flat array，使用 **array index 作為 implicit `__id__`**。這意味著：

- ✅ `__id__` 值必須為 `null`（不能用數字）
- ✅ 位置 `[i]` 的 object 就是 `__id__: i`
- ✅ `__id__` 指向另一個 object 時，值是 array index（如 `"__id__": 2` 表示 array[2]）
- ✅ 最大 `__id__` reference 值必須 < array length

**極常見錯誤：手寫 custom script component**

錯誤寫法（自定義 type name + `__scriptAsset__`）：
```json
{
  "__type__": "GameBootstrapper",      // ❌ 錯誤！Cocos 不認
  "node": { "__id__": 2 },
  "__scriptAsset__": { "__uuid__": "..." }  // ❌
}
```

正確寫法（`cc.Script` + `_scriptAsset`）：
```json
{
  "__type__": "cc.Script",             // ✅ 必須用 cc.Script
  "_name": "GameBootstrapper",
  "node": { "__id__": 2 },
  "_scriptAsset": {                    // ✅ 用 _scriptAsset（底線）
    "__uuid__": "df04b42a-..."
  },
  "_id": "zen-bootstrapper-001",
}
```

**Array index (__id__) 規則（真實案例）**：
- Array 長度 17 個 entries (index 0-16)
- Canvas node 的 `_components[]` 指向 `__id__: 5` (UITransform)、`__id__: 6` (Canvas)
- 如果你要加第三個 component（GameBootstrapper），必須用 `__id__: 16`（因為 index 16 是最後一個 entry）
- **用 `__id__: 17` 會 crash** — `Cannot read properties of undefined (reading '__type__')`

**正確做法（按優先級）：**
1. 🥇 **完全避開 Scene JSON** — 用純 code-driven：`new Node()` + `addComponent(GameBootstrapper)` 動態掛載（Cocos auto-mount pattern）
2. 🥈 **用 Editor 操作** — Add/Remove node/component（最安全）
3. 🥉 **完整重建 scene JSON** — 只用喺極端情況，嚴格遵守 `cc.Script` format 同 array index 規則

**錯誤做法：**
- ❌ 用 `patch()` 編輯 `.scene` JSON — escaped quotes (`\\\"`) 會 corrupt 檔案
- ❌ 從 JSON array splice 刪除元素 — 後面所有 `__id__` 錯位
- ❌ 自定義 `__type__` 名稱（非 `cc.Script`）— deserializer 會 crash
- ❌ 用 python `json.dump` 保留 `__id__` 的 numeric value — 必須設為 `null`

### 5. ✅ Scene cache 問題
直接修改 `.scene` 檔案後，Cocos Editor 可能仍使用 cache，導致「Load current scene data failed」錯誤。

**解決方法：** 關閉並重開 Cocos Creator（File → Quit → 重新打開 project）。

### 6. ✅ 強制 TypeScript 重新編譯
修改 `.ts` 檔案後如果 Play mode 不生效，可能因為 Editor 未觸發重新編譯。

**強制刷新方法：**
- Developer → Compile（手動觸發）
- 或者 Save Scene（Command+S）後等待自動 compile
- 或者 Build 任意平台來觸發完整 compile
- 查看 Console Panel 是否有紅色錯誤

## 完整範例：一個正確的 code-driven Label

```typescript
import { _decorator, Component, Node, Label, Color, UITransform } from 'cc';

const UI_2D_LAYER = 1 << 25;

export class MyUI extends Component {
  start() {
    const root = new Node('MyRoot');
    root.layer = UI_2D_LAYER;
    this.node.addChild(root);

    const labelNode = new Node('MyLabel');
    labelNode.layer = UI_2D_LAYER; // ← 每個子節點都要
    root.addChild(labelNode);

    const label = labelNode.addComponent(Label);
    label.string = '清晰可見的文字';
    label.useSystemFont = true;    // ← 必須
    label.fontSize = 24;
    label.color = new Color(255, 255, 255, 255); // ← 用 new Color()
  }
}
```

## 排錯流程（按順序）

1. **Layer 檢查** — 所有 `new Node()` 有冇 `layer = UI_2D_LAYER`？
2. **Font 檢查** — 所有 Label 有冇 `useSystemFont = true`？
3. **Color 檢查** — 所有 `.color =` 係咪用 `new Color(r, g, b, a)`？
4. **Scene 檢查** — 有冇手動改過 `.scene` JSON 的 `__id__`？
5. **Cache 檢查** — 有冇試過重開 Cocos Creator？
6. **Compile 檢查** — Console 有冇 compile error？

## 佛心項目特殊備註
- 背景色使用深黑 `#0A0A12`（對應 RGBA: new Color(10, 10, 18, 255)）
- 金色 `#FFD700`（對應: new Color(255, 215, 0, 255)）
- 純白文字 `#FFFFFF`（對應: new Color(255, 255, 255, 255)）
- Button 背景方塊: `new Color(40, 40, 80, 220)`
