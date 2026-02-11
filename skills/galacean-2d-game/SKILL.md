---
name: galacean-2d-game
description: 使用 Galacean Engine 开发2D游戏的核心模式，包括真2D（Sprite）和伪2D（3D正交投影）实现
---

# Galacean 2D 游戏开发

## 两种实现方式

Galacean 支持两种2D游戏开发方式：

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **真2D (Sprite)** | 真正的2D渲染，性能好 | API相对简单，功能有限 | 简单2D游戏、UI |
| **伪2D (3D正交)** | 完整3D功能，灵活 | 略重，需要理解3D概念 | 复杂2D效果、伪3D |

---

## 方式一：真2D（SpriteRenderer）

使用 `SpriteRenderer` 组件实现真正的2D渲染。

### 基础设置

```typescript
import { WebGLEngine, Camera, Vector3, BackgroundMode, Color } from "@galacean/engine";

async function init() {
  const engine = await WebGLEngine.create({
    canvas: document.getElementById('canvas'),
    physics: new LitePhysics(), // 如果需要点击检测
  });

  const scene = engine.sceneManager.activeScene;
  scene.background.mode = BackgroundMode.SolidColor;
  scene.background.solidColor = new Color(0.1, 0.1, 0.15, 1);

  // 2D相机：关键！相机必须在Z轴方向看向XY平面
  const cameraEntity = scene.createRootEntity().createChild('camera');
  const camera = cameraEntity.addComponent(Camera);
  camera.orthographicSize = 10;
  camera.nearClipPlane = 0.1;
  camera.farClipPlane = 100;
  camera.aspectRatio = window.innerWidth / window.innerHeight;
  
  // ⚠️ 关键：相机必须在Z轴方向，不能放在Y轴！
  // 这样XY平面上的物体才能被正确看到
  cameraEntity.transform.setPosition(0, 0, 10);
  cameraEntity.transform.lookAt(new Vector3(0, 0, 0));

  engine.run();
}
```

### 创建纯色Sprite

```typescript
import { SpriteRenderer, Sprite, Texture2D, Color } from "@galacean/engine";

function createColorSprite(entity: Entity, color: Color, size: number = 1): void {
  const renderer = entity.addComponent(SpriteRenderer);
  
  // 创建1x1像素纹理
  const texture = new Texture2D(entity.engine, 1, 1);
  const pixel = new Uint8Array([
    color.r * 255, 
    color.g * 255, 
    color.b * 255, 
    255
  ]);
  texture.setPixelBuffer(pixel);
  
  // 创建Sprite
  renderer.sprite = new Sprite(entity.engine, texture);
  renderer.width = size;
  renderer.height = size;
}
```

### 完整的2D方块组件

```typescript
import { Script, SpriteRenderer, Sprite, Texture2D, Color, StaticCollider, BoxColliderShape } from "@galacean/engine";

export class Tile2D extends Script {
  private spriteRenderer: SpriteRenderer;
  private tileType: number;

  init(type: number, size: number = 0.9): void {
    this.tileType = type;
    
    // 添加SpriteRenderer
    this.spriteRenderer = this.entity.addComponent(SpriteRenderer);
    this.setColor(getTileColor(type));
    this.spriteRenderer.width = size;
    this.spriteRenderer.height = size;
    
    // 2D点击检测
    const collider = this.entity.addComponent(StaticCollider);
    const shape = new BoxColliderShape();
    shape.size = new Vector3(size, size, 1);
    collider.addShape(shape);
  }

  setColor(color: Color): void {
    const texture = new Texture2D(this.engine, 1, 1);
    const pixel = new Uint8Array([
      color.r * 255, color.g * 255, color.b * 255, 255
    ]);
    texture.setPixelBuffer(pixel);
    this.spriteRenderer.sprite = new Sprite(this.engine, texture);
  }

  setHighlighted(highlight: boolean): void {
    const color = getTileColor(this.tileType).clone();
    if (highlight) {
      color.r = Math.min(1, color.r + 0.2);
      color.g = Math.min(1, color.g + 0.2);
      color.b = Math.min(1, color.b + 0.2);
      this.spriteRenderer.width = 1.0;
      this.spriteRenderer.height = 1.0;
    } else {
      this.spriteRenderer.width = 0.9;
      this.spriteRenderer.height = 0.9;
    }
    this.setColor(color);
  }
}
```

---

## 方式二：伪2D（3D正交投影）

使用扁平的3D几何体模拟2D效果。

### 相机设置

```typescript
const camera = cameraEntity.addComponent(Camera);
camera.orthographicSize = 10;
camera.aspectRatio = window.innerWidth / window.innerHeight;
cameraEntity.transform.setPosition(0, 0, 10);
cameraEntity.transform.lookAt(new Vector3(0, 0, 0));
```

### 创建扁平方块

```typescript
import { MeshRenderer, PrimitiveMesh, UnlitMaterial } from "@galacean/engine";

function createFlatTile(entity: Entity, color: Color, size: number = 1): void {
  const renderer = entity.addComponent(MeshRenderer);
  // 厚度很小的立方体
  renderer.mesh = PrimitiveMesh.createCuboid(entity.engine, size, size, 0.1);
  
  const material = new UnlitMaterial(entity.engine);
  material.baseColor = color;
  renderer.setMaterial(material);
}
```

---

## 消消乐坐标映射技巧

### 3D坐标系理解（关键）

```
在3D空间中（相机从Z轴看）：

    Y+ （屏幕上方）
    ↑
    |  row=7 (Y=+3.5)
    |  row=6
    |  row=5
    |  row=4
    |  row=3
    |  row=2
    |  row=1
    |  row=0 (Y=-3.5)
    |
    └────────→ X+
```

**关键**：消消乐中，row=0 在底部（Y负方向），row=7 在顶部（Y正方向）。

### 正确的坐标映射

```typescript
const BOARD_OFFSET_Y = -(GRID_ROWS - 1) * TILE_SPACING / 2; // = -3.5

function getTilePosition(row: number, col: number): Vector3 {
  const x = BOARD_OFFSET_X + col * TILE_SPACING;
  // 直接映射：row=0 -> Y=-3.5 (底部)，row=7 -> Y=+3.5 (顶部)
  const y = BOARD_OFFSET_Y + row * TILE_SPACING;
  return new Vector3(x, y, 0);
}
```

### 常见陷阱

**陷阱1：以为"下落"是row减小**
```typescript
// ❌ 错误理解
// "下落"应该是 row 从 7->0（减小）
// 实际上 Y 从 +3.5 -> -3.5，视觉上就是向下！
```

**正确理解**：
- "下落" = 从**大row**（上方）落到**小row**（下方）
- row=7（Y=+3.5）-> row=0（Y=-3.5）
- 动画：Y 坐标从正到负

**陷阱2：翻转row值**
```typescript
// ❌ 错误：翻转row导致方向相反
const flippedRow = (GRID_ROWS - 1) - row; // row=0变成7
const y = BOARD_OFFSET_Y + flippedRow * TILE_SPACING;
// 结果是row=0显示在屏幕上方！
```

### 棋盘坐标对应关系

| 逻辑坐标 | Y坐标 | 视觉效果 | 说明 |
|----------|-------|----------|------|
| row=0 | -3.5 | 底部 | 重力方向 |
| row=7 | +3.5 | 顶部 | 远离重力 |
| 新方块 | +6.5 (row=10) | 从上方进入 | 屏幕外进入 |

### 新方块入场动画

```typescript
// 新方块从屏幕上方（row=10）进入
const startRow = GRID_ROWS + 2; // = 10
const startPos = getTilePosition(startRow, col); // Y = 6.5
tile.position = startPos;
await tile.moveTo(getTilePosition(targetRow, col)); // 下落到目标位置
```

### 下落动画逻辑（关键）

消消乐的核心：**所有列同时下落**

```typescript
// ❌ 错误：按列顺序下落
for (let col = 0; col < GRID_COLS; col++) {
  const promises = [];
  // ... 收集列内动画
  await Promise.all(promises); // 等待这一列完成
}

// ✅ 正确：所有列同时下落
const allPromises: Promise<void>[] = [];
for (let col = 0; col < GRID_COLS; col++) {
  // ... 收集列内动画
  allPromises.push(...columnPromises); // 添加到总列表
}
await Promise.all(allPromises); // 统一等待
```

---

## 2D动画

### 位置动画

```typescript
class TileAnimator extends Script {
  private isMoving = false;
  private startPos = new Vector3();
  private targetPos = new Vector3();
  private moveTime = 0;
  private duration = 0.3;
  private onComplete?: () => void;

  moveTo(target: Vector3): Promise<void> {
    return new Promise((resolve) => {
      this.startPos.copyFrom(this.entity.transform.position);
      this.targetPos.copyFrom(target);
      this.moveTime = 0;
      this.isMoving = true;
      this.onComplete = resolve;
    });
  }

  onUpdate(deltaTime: number): void {
    if (!this.isMoving) return;
    
    this.moveTime += deltaTime;
    const t = Math.min(this.moveTime / this.duration, 1);
    const easeT = 1 - Math.pow(1 - t, 3);
    
    const pos = this.entity.transform.position;
    pos.x = this.startPos.x + (this.targetPos.x - this.startPos.x) * easeT;
    pos.y = this.startPos.y + (this.targetPos.y - this.startPos.y) * easeT;
    
    if (t >= 1) {
      this.isMoving = false;
      this.onComplete?.();
    }
  }
}
```

### 缩放动画（Sprite）

```typescript
// 选中效果
select() {
  this.spriteRenderer.width = 1.0;
  this.spriteRenderer.height = 1.0;
}

// 消除效果
async destroyAnimate() {
  const startSize = this.spriteRenderer.width;
  for (let t = 0; t <= 1; t += 0.1) {
    const scale = 1 - t;
    this.spriteRenderer.width = startSize * scale;
    this.spriteRenderer.height = startSize * scale;
    await delay(16);
  }
}
```

---

## 性能优化

### 对象池（必做）

消消乐方块频繁创建销毁：

```typescript
class TilePool {
  private pool: Tile2D[] = [];
  private active: Set<Tile2D> = new Set();

  get(type: number): Tile2D {
    const tile = this.pool.pop() || this.createNew();
    tile.reset(type);
    this.active.add(tile);
    return tile;
  }

  recycle(tile: Tile2D): void {
    this.active.delete(tile);
    tile.hide();
    this.pool.push(tile);
  }

  private createNew(): Tile2D {
    const entity = this.engine.createEntity();
    return entity.addComponent(Tile2D);
  }
}
```

### 纹理复用

```typescript
// 缓存纯色纹理
const colorTextureCache = new Map<string, Texture2D>();

function getColorTexture(engine: Engine, color: Color): Texture2D {
  const key = `${color.r},${color.g},${color.b}`;
  if (!colorTextureCache.has(key)) {
    const texture = new Texture2D(engine, 1, 1);
    const pixel = new Uint8Array([
      color.r * 255, color.g * 255, color.b * 255, 255
    ]);
    texture.setPixelBuffer(pixel);
    colorTextureCache.set(key, texture);
  }
  return colorTextureCache.get(key)!;
}
```

---

## 常见问题

### Q: Sprite不显示？
- 检查相机位置是否在Z轴
- 检查Sprite的width/height是否设置
- 检查纹理是否正确创建

### Q: 点击检测不准？
- 添加Collider时，Z轴厚度要足够（至少0.5）
- 检查相机和物体的Z坐标关系

### Q: 动画卡顿？
- 避免每帧创建新的Vector3/Color
- 使用对象池复用Tile
- 减少draw call（合并相同颜色的Sprite）

---

## 推荐选择

| 场景 | 推荐方案 |
|------|----------|
| 简单消消乐 | **真2D Sprite** |
| 需要复杂特效 | 伪2D + 粒子系统 |
| 2D角色动画 | 真2D + Sprite动画 |
| 2.5D效果 | 伪2D + 3D旋转 |
