---
name: cocos-graphics-visual-prototyping
category: software-development
description: Use when doing zero-asset visual prototyping with Cocos Graphics component.
  triggers:
    - cocos graphics
    - visual prototype
    - procedural graphics

---

# Cocos Creator 3.x Graphics Visual Prototyping

Use when: you need runtime-visible visual feedback for a game feature but have NO image assets yet. The `cc.Graphics` component draws procedurally — circles, ellipses, lines, polygons — using only code.

## Two Approaches: Editor Binding vs. Self-Contained VisualUI Wrapper

### Approach A: Editor Binding (@property) — Use when you have Editor access

1. Add empty `Node` children to Canvas (e.g., `PondNode`, `TreeNode`, `LotusNode`, `AuraNode`)
2. Use `@property({type: Node})` in your Component to expose these for Editor binding
3. In `start()`, get or auto-add `Graphics` component on each node
4. Define a **draw config array** — one entry per game state
5. Call `graphics.clear()` + draw commands + `graphics.fill()`/`graphics.stroke()` on state change
6. Use `Tween` for smooth transitions between states

### Approach B: Self-Contained VisualUI Wrapper — Use when you want zero Editor steps

Create a wrapper Component that **dynamically creates all Graphics nodes + Labels + Buttons** in `start()`, then self-binds to an existing Controller. No Editor drag-drop, no @property setup — just `addComponent(VisualUI)` and it works.

```typescript
// MeritForestVisualUI.ts — Self-contained wrapper
import { _decorator, Component, Node, Label, Button, UITransform, Color, Vec3, tween, Tween, Graphics } from 'cc';
import { MyController } from '../control/MyController';

const { ccclass } = _decorator;
const UI_LAYER = 33554432;

@ccclass('MeritForestVisualUI')
export class MeritForestVisualUI extends Component {
    private _controller: MyController | null = null;
    private _meritLabel: Label | null = null;
    private _pondNode: Node = null!;
    private _treeNode: Node = null!;

    start() {
        this._buildUI();
        this._initController();
    }

    private _buildUI(): void {
        const root = new Node('VisualRoot');
        root.parent = this.node;
        root.layer = UI_LAYER;

        // Graphics nodes (no Editor binding needed — created dynamically)
        this._pondNode = this._createGfxNode('Pond', 0, -40, 200, 150);
        this._treeNode = this._createGfxNode('Tree', 0, 60, 200, 150);

        // Labels
        this._meritLabel = this._createLabel('功德值：0', 0, 180, 32);
        
        // Buttons
        this._createButton('+100', -100, -160, () => this._addMerit(100));
    }

    private _createGfxNode(name: string, x: number, y: number, w: number, h: number): Node {
        const n = new Node(name);
        n.parent = this._root;
        n.layer = UI_LAYER;
        n.setPosition(x, y, 0);
        n.addComponent(UITransform).setContentSize(w, h);
        n.addComponent(Graphics);
        return n;
    }

    private _initController(): void {
        this._controller = MyController.instance;
        if (!this._controller) {
            this._controller = this.node.addComponent(MyController);
        }
        // Self-bind: pass our dynamically-created nodes to the Controller
        this._controller.bindPondNode(this._pondNode);
        this._controller.bindTreeNode(this._treeNode);
    }

    private _addMerit(amount: number): void {
        this._controller?.addMerit(amount);
    }

    private _createLabel(text: string, x: number, y: number, size: number): Label {
        const n = new Node();
        n.parent = this._root;
        n.layer = UI_LAYER;
        n.setPosition(x, y, 0);
        n.addComponent(UITransform).setContentSize(400, 30);
        const l = n.addComponent(Label);
        l.string = text;
        l.fontSize = size;
        l.lineHeight = size + 6;
        l.horizontalAlign = Label.HorizontalAlign.CENTER;
        l.color = new Color(255, 215, 0, 255);
        return l;
    }

    private _createButton(text: string, x: number, y: number, cb: () => void): void {
        const n = new Node();
        n.parent = this._root;
        n.layer = UI_LAYER;
        n.setPosition(x, y, 0);
        n.addComponent(UITransform).setContentSize(95, 36);
        n.addComponent(Button);
        const l = n.addComponent(Label);
        l.string = text;
        l.fontSize = 14;
        l.lineHeight = 20;
        l.horizontalAlign = Label.HorizontalAlign.CENTER;
        l.color = new Color(255, 255, 255, 220);
        n.on(Node.EventType.TOUCH_END, cb, this);
    }

    onDestroy(): void {
        // Clean up event subscriptions
        MeritVisualEventBus.clear();
    }
}
```

**Key differences from Approach A:**
- ❌ No `@property` decorators needed
- ❌ No Editor steps (no drag-drop, no Add Component, no config)
- ✅ The Controller must expose `bindXxx()` methods (not just `@property` slots)
- ✅ The Controller must be a Singleton or already exist on `this.node`
- ✅ Best for test rigs / Bootstrap patterns where you want instant Preview

## Step-by-Step

### 1. Expose Visual Nodes as @property

```typescript
import { _decorator, Component, Node, Graphics, Color, Vec3, Tween, tween } from 'cc';
const { ccclass, property } = _decorator;

@ccclass('MyVisualController')
export class MyVisualController extends Component {
    @property({ type: Node, tooltip: '樹節點（掛 Graphics）' })
    public treeNode: Node | null = null;

    @property({ type: Node, tooltip: '蓮花節點（掛 Graphics）' })
    public lotusNode: Node | null = null;

    private _treeGfx: Graphics | null = null;
    private _lotusGfx: Graphics | null = null;

    start() {
        // Auto-get-or-add Graphics on each bound node
        this._treeGfx = this._getOrAddGraphics(this.treeNode);
        this._lotusGfx = this._getOrAddGraphics(this.lotusNode);
        this._redrawAll();
    }

    private _getOrAddGraphics(node: Node | null): Graphics | null {
        if (!node) return null;
        return node.getComponent(Graphics) || node.addComponent(Graphics);
    }
}
```

### 2. Define State-to-Visual Config Array

Each state has its own draw functions:

```typescript
interface VisualConfig {
    pondColor: string;       // hex color string
    drawTree: (g: Graphics) => void;
    drawLotus: ((g: Graphics) => void) | null;
    drawAura: ((g: Graphics) => void) | null;
}

const VISUAL_CONFIGS: VisualConfig[] = [
    {  // Stage 0: Barren — gray ground, dead branches
        pondColor: '#3a3a3a',
        drawTree: (g) => {
            g.strokeColor = Color.hex('#504028');
            g.lineWidth = 2;
            g.moveTo(0, -60); g.lineTo(10, -30);
            g.moveTo(0, -60); g.lineTo(-10, -30);
            g.stroke();
        },
        drawLotus: null,
        drawAura: null,
    },
    {  // Stage 1: Seed — green dot
        pondColor: '#5a7a5a',
        drawTree: (g) => {
            g.fillColor = Color.hex('#50B450');
            g.circle(0, -30, 8); g.fill();
            g.strokeColor = Color.hex('#78643C');
            g.lineWidth = 1;
            g.moveTo(0, -22); g.lineTo(0, -60); g.stroke();
        },
        drawLotus: null,
        drawAura: null,
    },
    // ... more stages ...
];
```

### 3. Redraw on State Change

```typescript
private _currentStage = 0;

private _redrawAll(): void {
    const cfg = VISUAL_CONFIGS[this._currentStage];
    this._redrawTree(cfg);
    this._redrawLotus(cfg);
    this._redrawAura(cfg);
}

private _redrawTree(cfg: VisualConfig): void {
    if (!this._treeGfx) return;
    const g = this._treeGfx;
    g.clear();
    cfg.drawTree(g);
}

// Animate transition
private _animateTransition(): void {
    const nodes = [this.treeNode, this.lotusNode, this.auraNode];
    for (const node of nodes) {
        if (!node) continue;
        node.setScale(0.95, 0.95, 1);
        Tween.stopAllByTarget(node);
        tween(node)
            .to(0.6, { scale: new Vec3(1, 1, 1) }, {
                onComplete: () => this._redrawAll()
            })
            .start();
    }
}
```

## Graphics API Quick Reference

| Method | Purpose |
|--------|---------|
| `g.clear()` | Erase all drawings |
| `g.fillColor = Color.hex('#RRGGBB')` | Set fill color |
| `g.strokeColor = Color.hex('#RRGGBB')` | Set stroke color |
| `g.lineWidth = N` | Line thickness |
| `g.circle(x, y, radius)` | Add circle path |
| `g.ellipse(x, y, rx, ry)` | Add ellipse path |
| `g.moveTo(x, y)` | Move pen (start path) |
| `g.lineTo(x, y)` | Draw line to (x,y) |
| `g.fill()` | Fill the current path |
| `g.stroke()` | Stroke the current path |
| `g.close()` | Close current path |
| `g.rect(x, y, w, h)` | Rectangle path |
| `g.roundRect(x, y, w, h, r)` | Rounded rect path (radius `r` for all corners) — confirmed available in Creator 3.8.8 |

**Note**: `Color.hex()` takes a 6-char hex string WITHOUT `#`. For colors with alpha, use `new Color(r, g, b, a)`.

## Common Pitfalls

- **New Nodes default to wrong layer.** In Creator 3.8.8, `new Node()` gets `layer = 1073741824` (DEFAULT). The orthographic Camera only sees `layer = 33554432` (UI_2D). Always set `node.layer = canvas.layer` after creating visual nodes.
- **Always import all referenced classes.** In Creator 3.8 runtime (ES Module), `getComponent('ClassName')` with a string may work, but `getComponent(UnimportedClass)` or `node.getComponent(PracticeMoteCanvas)` without `import { PracticeMoteCanvas } from '...'` throws `ReferenceError: PracticeMoteCanvas is not defined`. Every class used in `getComponent()` / `addComponent()` must be explicitly imported at the top of the file.
- **`schedule()` may not fire as expected** in Creator 3.8 Preview runtime when called from a non-root component. For reliable periodic UI updates, use an `update(dt)` loop with a manual timer accumulator instead: `if ((this._timer += dt) >= 1.0) { this._timer = 0; /* update UI */ }`.
- **Graphics draws in node local space.** Position (0,0) is the node's pivot. Use `node.setPosition(x, y, 0)` to place the drawing on screen.
- **Graphics has no depth sorting within one node.** Draw order = call order. Draw background (pond) first, then foreground (lotus) last.
- **`node.active = false` hides Graphics.** The component does not auto-clear on deactivation. Call `g.clear()` before hiding if needed.
- **Tween on scale works well for transitions.** Combine with `onComplete` callback to redraw after the animation finishes. This masks the instant `g.clear()` + redraw with a smooth scale bounce.
- **For complex scenes, use multiple Graphics nodes** (one per visual element) so they can be independently positioned, scaled, and toggled.
- **`@property({type: Node})` slots are visible in Editor Inspector.** The user drags the corresponding node from Hierarchy into the slot. This replaces manual `bindXxx()` methods.

## Example: Director's Cut (6-Stage Forest Evolution from BuddhaHeart)

The six stages from the BuddhaHeart project demonstrate full Graphics visual range:

| Stage | Merit | Pond | Tree | Lotus | Aura |
|-------|-------|------|------|-------|------|
| 0 Barren | 0 | Dark gray | Dead branches | None | None |
| 1 Seed | 100 | Greenish | Green dot + root | None | None |
| 2 Sprout | 500 | Blue | Stem + 2 leaves | None | None |
| 3 Flourish | 2000 | Blue | Trunk + multi-layer canopy | Pink bud | None |
| 4 Bloom | 8000 | Light blue | Large tree + golden dots | 5-petal pink flower | Gold ring |
| 5 Divine | 30000 | Gold | Golden tree + 12 light dots | 8-petal gold lotus | 3 rings + 16 rays |

Each stage's draw functions are stored in a config array — adding a 7th stage requires only appending to the array.
