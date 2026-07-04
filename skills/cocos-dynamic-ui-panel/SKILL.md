---
name: cocos-dynamic-ui-panel
category: software-development
description: Use when building full runtime UI panels in Cocos Creator 3.x.
  triggers:
    - ui panel
    - dynamic ui
    - cocos ui
    - cocos panel

---

# Cocos Creator 3.x Dynamic UI Panel Pattern

Use when: you need a complex UI panel (buttons, labels, timers, status displays, panel transitions) but want to build it entirely in code without .scene editing or dragging Prefabs. Useful for rapid prototyping and test rigs.

## Core Pattern

1. Create all UI nodes in `start()` using helper methods
2. Store references to dynamic controls (Labels, UITransforms)
3. Use `update(dt)` with a timer accumulator instead of `schedule()`
4. Show/hide panels by toggling `node.active`
5. Always set `node.layer = UI_LAYER` on new nodes

## Boilerplate

```typescript
import { _decorator, Component, Node, Label, Button, UITransform, Color } from 'cc';
const { ccclass } = _decorator;
const UI_LAYER = 33554432; // 1<<25 — UI_2D

@ccclass('MyPanel')
export class MyPanel extends Component {
    // === Label references for dynamic updates ===
    private _statusLabel: Label = null!;
    private _timerLabel: Label = null!;
    
    // === Panel references for switching ===
    private _mainPanel: Node = null!;
    private _sessionPanel: Node = null!;
    
    // === Update timer ===
    private _updateTimer: number = 0;
    private readonly _UPDATE_INTERVAL = 1.0; // seconds

    start() {
        this._buildUI();
    }

    update(dt: number) {
        this._updateTimer += dt;
        if (this._updateTimer >= this._UPDATE_INTERVAL) {
            this._updateTimer = 0;
            this._refreshStatus();
            this._refreshTimer();
        }
    }

    // ===== UI Build Helpers =====
    
    private _buildUI(): void {
        const root = this.node;
        
        this._statusLabel = this._label(root, 'Status text', 0, 200, 20);
        this._timerLabel = this._label(root, '0:00', 0, 100, 36);
        
        this._btn(root, 'Action', 0, -50, () => this._onAction());
        
        // Hidden session panel
        this._sessionPanel = new Node('SessionPanel');
        this._sessionPanel.parent = root;
        this._sessionPanel.layer = UI_LAYER;
        this._sessionPanel.active = false;
        this._label(this._sessionPanel, 'In Session...', 0, 0, 24);
    }
    
    // ===== Helper Methods =====

    private _label(p: Node, text: string, x: number, y: number, size: number): Label {
        const n = new Node();
        n.parent = p;
        n.layer = UI_LAYER;
        n.setPosition(x, y, 0);
        n.addComponent(UITransform).setContentSize(400, 36);
        const l = n.addComponent(Label);
        l.string = text;
        l.fontSize = size;
        l.lineHeight = size + 8;
        l.color = new Color(255, 255, 255, 255);
        return l;
    }

    private _btn(p: Node, text: string, x: number, y: number, cb: () => void): Node {
        const n = new Node();
        n.parent = p;
        n.layer = UI_LAYER;
        n.setPosition(x, y, 0);
        n.addComponent(UITransform).setContentSize(200, 44);
        n.addComponent(Button);
        const l = n.addComponent(Label);
        l.string = text;
        l.fontSize = 20;
        l.lineHeight = 28;
        l.color = new Color(255, 255, 255, 255);
        n.on(Node.EventType.TOUCH_END, cb, this);
        return n;
    }

    // ===== Panel Switching =====

    private switchToSessionPanel(): void {
        this._mainPanel.active = false;
        this._sessionPanel.active = true;
    }

    private switchToMainPanel(): void {
        this._sessionPanel.active = false;
        this._mainPanel.active = true;
    }
}
```

## Why `update(dt)` Instead of `schedule()`

In Creator 3.8 Preview runtime, `this.schedule(callback, interval)` may not fire reliably when called from:
- Components dynamically added via `addComponent()` at runtime
- Non-root components in complex hierarchies

**Safer alternative**: Manual timer in `update(dt)`:

```typescript
private _uiTimer: number = 0;

update(dt: number): void {
    this._uiTimer += dt;
    if (this._uiTimer >= 1.0) {
        this._uiTimer -= 1.0;  // subtract, don't reset (avoids drift)
        this._refreshUI();
    }
}
```

## Code-Driven ScrollView (Cocos 3.x)

Building a ScrollView entirely in code requires precise node hierarchy. **Missing any node or component causes scroll to fail silently.**

### Required Hierarchy

```
ScrollView Node (UITransform + ScrollView)
└── View Node (UITransform + Mask.Type.RECT)
    └── Content Node (UITransform + Layout.Type.VERTICAL)
```

### Boilerplate

```typescript
import { ScrollView, Mask, Layout, UITransform, v3 } from 'cc';

private _buildScrollView(parent: Node, width: number, height: number, padding = 30, spacing = 20): ScrollView {
    const svNode = new Node('scrollView');
    svNode.setPosition(v3(0, 0, 0));
    parent.addChild(svNode);
    const svTrans = svNode.addComponent(UITransform);
    svTrans.width = width;
    svTrans.height = height;

    const scrollView = svNode.addComponent(ScrollView);
    scrollView.horizontal = false;
    scrollView.vertical = true;
    scrollView.inertia = true;
    scrollView.brake = 0.85;
    scrollView.elastic = true;
    scrollView.verticalScrollBar = null as any; // hide scrollbar

    // View node — required child with Mask for clipping
    const viewNode = new Node('view');
    viewNode.setPosition(v3(0, 0, 0));
    svNode.addChild(viewNode);
    const viewTrans = viewNode.addComponent(UITransform);
    viewTrans.width = width;
    viewTrans.height = height;
    viewNode.addComponent(Mask).type = Mask.Type.RECT;

    // Content node — must have anchorY=1 for correct scroll start
    const contentNode = new Node('content');
    contentNode.setPosition(v3(0, height / 2, 0));
    viewNode.addChild(contentNode);
    const contentTrans = contentNode.addComponent(UITransform);
    contentTrans.width = width;
    contentTrans.anchorX = 0.5;
    contentTrans.anchorY = 1;  // ⚠️ CRITICAL

    // Layout — auto-expands content height as children are added
    const layout = contentNode.addComponent(Layout);
    layout.type = Layout.Type.VERTICAL;
    layout.resizeMode = Layout.ResizeMode.CONTAINER;
    layout.horizontalDirection = Layout.HorizontalDirection.LEFT_TO_RIGHT;
    layout.verticalDirection = Layout.VerticalDirection.TOP_TO_BOTTOM;
    layout.paddingTop = padding;
    layout.paddingLeft = padding;
    layout.paddingRight = padding;
    layout.spacingY = spacing;

    scrollView.content = contentNode;
    return scrollView;
}
```

### ⚠️ Critical Gotchas

1. **Content `anchorY = 1` is mandatory** — without this, scroll starts from the wrong position.
2. **Content needs `Layout` with `ResizeMode.CONTAINER`** — children auto-expand the content height.
3. **View needs `Mask.Type.RECT`** — without it, content bleeds outside the visible area.
4. **`verticalScrollBar = null as any`** — hide scrollbar explicitly (Creator 3.x defaults show one).
5. **`scrollView.content = contentNode` must be set** — otherwise nothing scrolls.
6. **Use `new Node()` not `instantiate`** — for content children in dynamic UIs.
7. **Children added to content need `UITransform` with width** — otherwise Layout can't arrange them.

### Touch Detection on ScrollView Children (Name-Based Dispatching)

Use Node name-based detection to avoid Button-component interference with scroll gestures:

```typescript
this.node.on(Node.EventType.TOUCH_END, (event: EventTouch) => {
    const target = event.target;
    if (!target) return;
    const name = target.name;
    
    // Tab/button using name prefix tagging
    if (name.startsWith('tab_')) { this._onTabSelect(name.substring(4)); }
    if (name.startsWith('card_')) { this._onCardClick(name.substring(5)); }
    if (name === 'btn_back') { this._goBack(); }
    if (name === 'btn_close') { this.node.destroy(); }
}, this);
```

## Key Takeaways (BuddhaHeart Project)

1. **Always import classes used in `getComponent()`.** `getComponent(PracticeMoteCanvas)` without `import { PracticeMoteCanvas }` throws `ReferenceError` in Creator 3.8 ES Module runtime.
2. **`schedule()` is unreliable for runtime-added components.** Use manual `update(dt)` timer accumulators in 3.8.8.
3. **Panel switching via `active` flag works well.** Pre-create both panels in `start()`, toggle visibility. Avoid creating/destroying nodes during gameplay.
4. **Button Node hierarchy matters.** `Button` component must be on the same node as `UITransform`. Label should also be on the same node (or a child) — but same node works fine.
5. **`Node.EventType.TOUCH_END` is the click event** in Creator 3.x. Not `CLICK` (which requires a Collider/EventTrigger). Use `TOUCH_END` for button interactions.
6. **Dynamic Labels with changing content** — just assign `label.string = newValue` directly. No need to recreate nodes.
7. **Layout responsibility is yours.** Unlike scene-editor UIs with Widget/Layout components, dynamic UIs must position each node manually via `setPosition(x, y, 0)`.
