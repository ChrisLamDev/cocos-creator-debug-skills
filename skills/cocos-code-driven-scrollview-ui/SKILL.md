---
name: cocos-code-driven-scrollview-ui
description: Use when building code-driven scrollable UI panels in Cocos Creator 3.x.
version: 1.0.0
author: Hermes Agent
tags: [cocos-creator, code-driven, ui, scrollview, dynamic-node, phase7]
  triggers:
    - scrollview
    - scroll view
    - cocos scroll
    - list scroll

---

# Cocos Creator 3.x Code-Driven ScrollView UI Panel

## When To Use

When you need to build a complex UI panel (ScrollView + TabBar + Header + interactive cards) in Cocos Creator 3.x **without any Editor manual binding**. The user only needs to create one empty Node in the scene and attach the script.

Best for:
- Phase 7 content migration panels (Scripture Library, Story Engine, Recitation Hall)
- Any temporary test UI that shouldn't clutter the Editor scene
- Panels that need clean lifecycle management (create/destroy without leaks)

## Architecture Pattern

### File Structure (3-file module)
```
assets/scripts/ModuleName/
├── ModuleTypes.ts    — Constants, enums, state interfaces
├── ModuleManager.ts  — Singleton Cocos Component, data loading + query API
└── ModuleUI.ts       — Cocos Component, all dynamic node generation + rendering
```

### Full Node Hierarchy (code-generated in onLoad)

```
ScripturePanel (ScriptureUI Component)
├── header (Node)
│   └── title (Label) — "📚 藏經閣"
├── tabBar (Node)
│   ├── tab_classic1 (Label) — "📜 無量壽經"
│   ├── tab_classic2 (Label) — "📜 觀經"
│   └── ...
├── scrollView (ScrollView)
│   └── view (UITransform + Mask Type.RECT)
│       └── content (UITransform + Layout Type.VERTICAL, ResizeMode.CONTAINER)
│           └── [dynamic child nodes: cards, headings, text, spacers]
└── closeBtn (Label) — "✕"
```

## ScrollView Construction (Critical)

The correct node hierarchy for a working Cocos 3.x ScrollView is:

```typescript
// 1. ScrollView outer node
const svNode = new Node('scrollView');
svNode.setPosition(x, y, 0);
parent.addChild(svNode);
const svTrans = svNode.addComponent(UITransform);
svTrans.width = PANEL_W;
svTrans.height = svH;

// 2. ScrollView component (not on child — on the outer node)
const scrollView = svNode.addComponent(ScrollView);
scrollView.horizontal = false;
scrollView.vertical = true;
scrollView.inertia = true;
scrollView.brake = 0.85;
scrollView.elastic = true;
scrollView.verticalScrollBar = null as any; // hide scrollbar

// ⚠️ CRITICAL: View node MUST have Mask for clipping
const viewNode = new Node('view');
viewNode.setPosition(0, 0, 0);
svNode.addChild(viewNode);
const viewTrans = viewNode.addComponent(UITransform);
viewTrans.width = PANEL_W;
viewTrans.height = svH;
const mask = viewNode.addComponent(Mask);
mask.type = Mask.Type.RECT;

// ⚠️ CRITICAL: Content anchor MUST be (0.5, 1) for vertical scroll from top
const contentNode = new Node('content');
contentNode.setPosition(0, svH / 2, 0); // top-center of view
viewNode.addChild(contentNode);
const contentTrans = contentNode.addComponent(UITransform);
contentTrans.width = PANEL_W - PADDING * 2;
contentTrans.height = svH; // initial, auto-resized by Layout
contentTrans.anchorX = 0.5;
contentTrans.anchorY = 1;

// ⚠️ CRITICAL: Layout with CONTAINER resize mode auto-grows content
const layout = contentNode.addComponent(Layout);
layout.type = Layout.Type.VERTICAL;
layout.resizeMode = Layout.ResizeMode.CONTAINER;
layout.horizontalDirection = Layout.HorizontalDirection.LEFT_TO_RIGHT;
layout.verticalDirection = Layout.VerticalDirection.TOP_TO_BOTTOM;
layout.paddingTop = 30;
layout.paddingLeft = PADDING;
layout.paddingRight = PADDING;
layout.spacingY = 20;

// Link scrollView.content
scrollView.content = contentNode;
```

### ScrollView Pitfalls

| Mistake | Symptom | Fix |
|---------|---------|-----|
| No Mask on View node | Content overflows visible area | Add `viewNode.addComponent(Mask)` with `type = Mask.Type.RECT` |
| Content anchor (0.5, 1) wrong | Content starts from center instead of top | Set `contentTrans.anchorX = 0.5; anchorY = 1` |
| Missing Layout CONTAINER | Content doesn't auto-grow | Add Layout, set `resizeMode = CONTAINER` |
| Setting content height manually | Layout CONTAINER + manual height conflict | Don't set content height — Layout handles it |
| Not linking scrollView.content | ScrollView doesn't know what to scroll | `scrollView.content = contentNode` |
| Forgetting `as any` on scrollBar | null assignment type error | `scrollView.verticalScrollBar = null as any` |
| `scrollView` placed on wrong node | ScrollView doesn't scroll | ScrollView component goes on the OUTER node, NOT the view node |

## Touch Dispatch Pattern

Since all nodes are code-generated, use node **name string prefix** for touch identification (no Editor event system):

```typescript
// Constants for node name prefixes
const TAG_TAB = 'tab_';
const TAG_CARD = 'card_';
const TAG_BACK = 'btn_back';
const TAG_CLOSE = 'btn_close';

onLoad(): void {
  this.node.on(Node.EventType.TOUCH_END, this._onTouchEnd, this);
}

// When creating nodes
const tabNode = new Node(TAG_TAB + tab.id);  // e.g. "tab_classic1"
const cardNode = new Node(TAG_CARD + classic.id);  // e.g. "card_classic1"
backBtn.name = TAG_BACK;
closeBtn.name = TAG_CLOSE;

private _onTouchEnd(event: EventTouch): void {
  const target = event.target;
  if (!target) return;
  const name = target.name;

  if (name === TAG_CLOSE) { this.node.destroy(); return; }
  if (name === TAG_BACK) { this._showCatalogue(); return; }
  if (name.startsWith(TAG_TAB)) { ... }
  if (name.startsWith(TAG_CARD)) { ... }
}
```

## Lifecycle Management

```typescript
// Clean content (remove all children) when switching views
private _clearContent(): void {
  if (this._contentNode) {
    this._contentNode.removeAllChildren();
  }
}

// Clean full destruction
// When user clicks close: this.node.destroy()
// Cocos handles recursive child destruction
```

## cc.d.ts Declarations Needed

If your project has a custom `types/cc.d.ts` for `tsc --noEmit`, you need these declarations:

```typescript
// ===== ScrollView =====
export class ScrollView extends Component {
  public content: Node | null;
  public horizontal: boolean;
  public vertical: boolean;
  public inertia: boolean;
  public brake: number;
  public elastic: boolean;
  public verticalScrollBar: Node | null;
}

// ===== Layout =====
export namespace Layout {
  enum Type { NONE = 0, HORIZONTAL = 1, VERTICAL = 2, GRID = 3 }
  enum ResizeMode { NONE = 0, CONTAINER = 1, CHILDREN = 2 }
  enum HorizontalDirection { LEFT_TO_RIGHT = 0, RIGHT_TO_LEFT = 1 }
  enum VerticalDirection { TOP_TO_BOTTOM = 0, BOTTOM_TO_TOP = 1 }
}
export class Layout extends Component { ... }

// ===== Mask =====
export namespace Mask { enum Type { RECT = 0, ELLIPSE = 1, IMAGE_STENCIL = 2 } }
export class Mask extends Component { public type: Mask.Type; }

// ===== Node setPosition overload =====
// Accept both Vec3 and (x, y, z)
public setPosition(x: number | Vec3, y?: number, z?: number): void;

// ===== UITransform setContentSize overload =====
public setContentSize(w: number, h?: number): void;
public setContentSize(s: Size): void;

// ===== EventTouch =====
export class EventTouch { public target: Node; }
```

## Label Creation Pattern (no `isSystemFontUsed`)

Cocos 3.x Label does NOT have `isSystemFontUsed` property in all versions. Use this safe pattern:

```typescript
private _makeLabel(text: string, fontSize: number, hexColor: string): Node {
  const node = new Node('label');
  const label = node.addComponent(Label);
  label.string = text;
  label.fontSize = fontSize;
  label.lineHeight = fontSize * 1.6;
  label.color = this._hexToColor(hexColor);
  label.horizontalAlign = Label.HorizontalAlign.CENTER;
  const trans = node.getComponent(UITransform);
  if (trans) trans.width = CARD_W;
  return node;
}

private _hexToColor(hex: string): Color {
  return Color.fromHEX(new Color(), hex);
}

private _wrapText(text: string, charsPerLine: number): string {
  // Simple char-count wrapping
  let result = '', count = 0;
  for (const ch of text) { result += ch; count++; if (count >= charsPerLine) { result += '\n'; count = 0; } }
  return result;
}
```

## Dark Mode Color Constants

Place at module top-level for easy global adjustment:

```typescript
const C_BG = '#1A1A2E';
const C_SURFACE = '#16213E';
const C_TEXT_PRIMARY = '#F0EAD6';
const C_TEXT_SECONDARY = '#B8B0A0';
const C_GOLD = '#E8C96A';
const C_PINK = '#D98C7A';
const C_TAB_INACTIVE = '#4A5A7A';
const C_TAB_ACTIVE = '#E8C96A';
```

## Steps to Add a New Code-Driven Panel

1. **Create `Types.ts`** — Tab definitions, UI state interface, defaults
2. **Create `Manager.ts`** — Singleton Component with data loading (async) + query API
3. **Create `UI.ts`** — Component with:
   - `onLoad()`: buildHeader(), buildTabBar(), buildScrollView(), buildCloseButton()
   - Display methods: showCatalogue(), showClassicContent(), showDailySutra()
   - Helper methods: makeLabel(), makeSpacer(), wrapText(), hexToColor()
   - Touch handler: _onTouchEnd() with name-based dispatch
4. **Update Bootstrap.ts** — Add import + switch case
5. **Update cc.d.ts** — Add any missing Cocos API declarations
6. **Run `tsc --noEmit`** — Confirm zero errors on new files
7. **Git commit**
8. **User test**: Open Creator → Preview → click button
