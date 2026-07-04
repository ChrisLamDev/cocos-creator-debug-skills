---
name: cocos-graphics-mote-system
category: software-development
description: Use when replacing sprite particles with procedural graphics rendering.
  triggers:
    - particle
    - mote
    - procedural particle
    - graphics mote

---

# Cocos Graphics Mote (Micro-Particle) System

Use when: you need dozens of small animated particles ("motes" — glowing dots representing other players, blessings, or ambient effects) but have NO sprite/particle assets and want visual feedback **immediately** via Graphics.

## Core Concept

Instead of instantiating Sprite nodes in a pool (which requires a SpriteFrame asset), use a single `cc.Graphics` component that redraws all motes every frame. Each mote is data only — position, color, alpha, radius — rendered procedurally.

## Step-by-Step

### 1. Animated Mote Data Structure

```typescript
interface AnimatedMote {
    id: string;
    x: number;          // current pixel position
    y: number;
    targetX: number;    // drift target
    targetY: number;
    scale: number;      // size multiplier
    alpha: number;      // 0-1 opacity
    life: number;       // 0-1 lifecycle progress
    color: Color;       // mote color
    isSelf: boolean;    // highlight for "me"
    radius: number;     // base radius
}
```

### 2. Component Structure

```typescript
import { _decorator, Component, Graphics, Color, Node } from 'cc';
const { ccclass, property } = _decorator;

@ccclass('MoteCanvas')
export class MoteCanvas extends Component {
    @property public driftSpeed: number = 20;
    @property public updateInterval: number = 0.5;

    private _gfx: Graphics | null = null;
    private _motes: AnimatedMote[] = [];
    private _timer: number = 0;
    private _width: number = 960;
    private _height: number = 640;

    start() {
        this._gfx = this.getComponent(Graphics) || this.addComponent(Graphics);
    }

    update(dt: number) {
        // Sync mote list from data source
        this._timer += dt;
        if (this._timer >= this.updateInterval) {
            this._timer = 0;
            this._syncFromDataSource();
        }

        // Animate positions
        this._updatePositions(dt);
        this._updateLifecycles(dt);
        
        // Redraw all
        this._drawBackground();
        this._drawMotes();
    }
}
```

### 3. Drawing a Single Mote (Glow + Core + Self-Highlight)

```typescript
private _drawMote(g: Graphics, m: AnimatedMote): void {
    if (m.alpha <= 0.01) return;

    const alpha255 = Math.floor(m.alpha * 255);
    const r = m.radius + m.scale * 4;

    // Outer glow (low alpha, bigger circle)
    g.fillColor = new Color(m.color.r, m.color.g, m.color.b, Math.floor(alpha255 * 0.15));
    g.circle(m.x, m.y, r * 2.5);
    g.fill();

    // Main mote (full color)
    g.fillColor = new Color(m.color.r, m.color.g, m.color.b, Math.floor(alpha255 * 0.8));
    g.circle(m.x, m.y, r);
    g.fill();

    // Self mote: gold ring highlight
    if (m.isSelf) {
        g.strokeColor = new Color(255, 215, 0, Math.floor(alpha255 * 0.8));
        g.lineWidth = 2;
        g.circle(m.x, m.y, r + 3);
        g.stroke();
    }
}
```

### 4. Position Drift Animation

```typescript
private _updatePositions(dt: number): void {
    for (const m of this._motes) {
        const dx = m.targetX - m.x;
        const dy = m.targetY - m.y;
        const dist = Math.sqrt(dx * dx + dy * dy);

        if (dist > 2) {
            m.x += (dx / dist) * this.driftSpeed * dt;
            m.y += (dy / dist) * this.driftSpeed * dt;
        } else {
            // Pick new random target within bounds
            const margin = 20;
            const halfW = this._width / 2 - margin;
            const halfH = this._height / 2 - margin;
            m.targetX = m.x + (Math.random() - 0.5) * 80;
            m.targetY = m.y + (Math.random() - 0.5) * 80;
            m.targetX = Math.max(-halfW, Math.min(halfW, m.targetX));
            m.targetY = Math.max(-halfH, Math.min(halfH, m.targetY));
        }
    }
}
```

### 5. Fade In/Out Lifecycle

```typescript
private _updateLifecycles(dt: number): void {
    for (const m of this._motes) {
        const age = (Date.now() - m.birthTime) / 1000;
        const duration = m.lifetime;
        m.life = Math.min(1, age / duration);

        // Fade in: first 0.5s
        if (age < 0.5) {
            m.alpha = age / 0.5;
        }
        // Fade out: last 2s
        else if (age > duration - 2) {
            m.alpha = Math.max(0, (duration - age) / 2);
        } else {
            m.alpha = 1;
        }
    }
    
    // Remove expired motes
    this._motes = this._motes.filter(m => m.life < 1);
}
```

## Color Palette for Activity-Based Motes

A common use case is coloring motes by "practice activity" or "event type":

```typescript
const MOTE_COLORS: Record<string, Color> = {
    meditation: new Color(180, 140, 255, 200),  // purple
    chanting:   new Color(255, 215, 0, 200),     // gold
    offering:   new Color(255, 180, 50, 200),    // orange
    sutra:      new Color(100, 200, 255, 180),   // blue
    circumamb:  new Color(255, 220, 180, 180),   // warm white
};
```

## When to Use vs. Avoid

**Use Graphics motes when:**
- You need visual feedback NOW during development
- No artist/asset pipeline available
- Mote count < 100 (Graphics redraw is cheap at this scale)
- You want unique colors per mote type without texture atlasing
- Target platforms are modern devices (desktop / recent mobile)

**Switch to ParticleSystem / Sprite when:**
- Mote count exceeds 200 (GPU batching becomes important)
- You need texture-based shapes (stars, flames, sparkles)
- Asset pipeline is ready and you have final art
- Targeting low-end mobile devices (Graphics redraw on CPU)

## Key Takeaways (BuddhaHeart Project Learnings)

1. **Always import all classes.** `getComponent(PracticeMoteCanvas)` without `import { PracticeMoteCanvas }` throws `ReferenceError` in ES Module runtime.
2. **One Graphics draws everything.** Unlike Sprite pools (N nodes = N draw calls), a single Graphics component draws N motes in 1 draw call.
3. **Layer must match Canvas.** New Nodes default to `layer = 1073741824` (DEFAULT). Set `node.layer = 33554432` (UI_2D) for the orthographic Camera to see them.
4. **No SpriteFrame needed.** This is the killer feature — you can prototype particle-heavy scenes with zero asset imports.
5. **Handle resize.** If your canvas size changes, update `_width` / `_height` and re-clamp mote positions.
