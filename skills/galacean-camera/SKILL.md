---
name: galacean-camera
description: 此 Skill 包含了 Galacean Engine 相机的配置方法，包括透视/正交投影设置、俯视视角配置、窗口适配等
---

# Galacean Camera 相机配置

## 1. 相机基础概念

Galacean 的 `Camera` 组件决定场景如何渲染到屏幕上。关键属性：

| 属性 | 类型 | 说明 |
|------|------|------|
| `aspectRatio` | number | 宽高比（width / height） |
| `fieldOfView` | number | 透视投影的视场角（度） |
| `orthographicSize` | number | 正交投影的垂直视野大小 |
| `nearClipPlane` | number | 近裁剪面 |
| `farClipPlane` | number | 远裁剪面 |
| `isOrthographic` | boolean | 是否正交投影（只读，通过设置 orthographicSize 切换） |

## 2. 俯视视角游戏推荐配置

### 2.1 正交投影（推荐用于俯视游戏）

正交投影不会产生透视变形，物体大小不随距离变化，适合俯视视角。

```typescript
const cameraEntity = rootEntity.createChild('camera');
const camera = cameraEntity.addComponent(Camera);

// 基础设置
camera.enableFrustumCulling = false;

// 正交投影关键设置
camera.orthographicSize = 14; // 垂直视野范围（单位：世界单位）
camera.aspectRatio = canvas.width / canvas.height;

// 相机位置：正上方俯视
cameraEntity.transform.setPosition(0, 25, 0);
cameraEntity.transform.lookAt(new Vector3(0, 0, 0));

// 窗口适配
window.addEventListener('resize', () => {
  camera.aspectRatio = window.innerWidth / window.innerHeight;
});
```

### 2.2 透视投影（带透视效果）

```typescript
const cameraEntity = rootEntity.createChild('camera');
const camera = cameraEntity.addComponent(Camera);

// 透视投影设置
camera.fieldOfView = 60; // 视场角（度）
camera.aspectRatio = canvas.width / canvas.height;

// 相机位置：倾斜俯视角度
cameraEntity.transform.setPosition(0, 20, 10);
cameraEntity.transform.lookAt(new Vector3(0, 0, 0));

// 窗口适配
window.addEventListener('resize', () => {
  camera.aspectRatio = window.innerWidth / window.innerHeight;
});
```

## 3. 关键问题：避免模型被压扁

### 问题原因
正交投影下，如果 `aspectRatio` 设置不正确（如使用默认的屏幕比例但没有随窗口变化更新），会导致画面比例失调，模型看起来被压扁。

### 正确做法

```typescript
// ❌ 错误：不设置 aspectRatio 或设置固定值
camera.orthographicSize = 15;
// camera 会使用默认的 aspectRatio，可能不正确

// ✅ 正确：动态设置 aspectRatio
camera.orthographicSize = 15;
camera.aspectRatio = window.innerWidth / window.innerHeight;

// 并监听窗口变化
window.addEventListener('resize', () => {
  camera.aspectRatio = window.innerWidth / window.innerHeight;
});
```

### 移动端竖屏适配

```typescript
// 强制竖屏游戏的相机设置
function setupCameraForPortrait(camera: Camera): void {
  // 竖屏时，高度应该看到更多内容
  const isPortrait = window.innerHeight > window.innerWidth;
  
  if (isPortrait) {
    // 竖屏：调整 orthographicSize 适应
    camera.orthographicSize = 16;
  } else {
    // 横屏：调整 orthographicSize 适应
    camera.orthographicSize = 10;
  }
  
  camera.aspectRatio = window.innerWidth / window.innerHeight;
}

window.addEventListener('resize', () => setupCameraForPortrait(camera));
```

## 4. 常用相机模式

### 4.1 固定俯视相机

```typescript
// 完全固定的俯视相机
const cameraEntity = rootEntity.createChild('camera');
const camera = cameraEntity.addComponent(Camera);

camera.orthographicSize = 15;
camera.aspectRatio = window.innerWidth / window.innerHeight;
cameraEntity.transform.setPosition(0, 30, 0);
cameraEntity.transform.rotation.set(90, 0, 0); // 完全朝下
```

### 4.2 跟随玩家的相机

```typescript
class CameraFollowScript extends Script {
  private target: Entity | null = null;
  private offset: Vector3 = new Vector3(0, 20, 10);
  private smoothSpeed: number = 5;

  setTarget(target: Entity): void {
    this.target = target;
  }

  onUpdate(deltaTime: number): void {
    if (!this.target) return;

    const targetPos = this.target.transform.position;
    const desiredPos = new Vector3(
      targetPos.x + this.offset.x,
      targetPos.y + this.offset.y,
      targetPos.z + this.offset.z
    );

    // 平滑跟随
    const currentPos = this.entity.transform.position;
    const lerpFactor = this.smoothSpeed * deltaTime;
    
    currentPos.x += (desiredPos.x - currentPos.x) * lerpFactor;
    currentPos.y += (desiredPos.y - currentPos.y) * lerpFactor;
    currentPos.z += (desiredPos.z - currentPos.z) * lerpFactor;
    
    this.entity.transform.position = currentPos;
    this.entity.transform.lookAt(targetPos);
  }
}
```

## 5. 相机参数速查

### 正交投影参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `orthographicSize` | 10-20 | 越大视野越大 |
| `nearClipPlane` | 0.1 | 近裁剪面 |
| `farClipPlane` | 100 | 远裁剪面 |
| `aspectRatio` | 动态计算 | width / height |

### 透视投影参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `fieldOfView` | 45-60 | 越大视野越广 |
| `nearClipPlane` | 0.1 | 近裁剪面 |
| `farClipPlane` | 1000 | 远裁剪面 |

## 6. 2D正交投影相机配置

### 重要说明：伪2D vs 真2D
Galacean 是 3D 引擎，"2D游戏"通常有两种实现方式：
1. **伪2D（3D正交投影）**：使用 3D 立方体/平面，相机从正面看，这是本节的配置方式
2. **真2D（Sprite）**：使用 SpriteRenderer 渲染 2D 图片（ Galacean 的 2D 能力在演进中）

### 2D相机配置（伪2D）

对于消消乐、塔防等棋盘类游戏，使用正交投影从 Z 轴方向看向 XY 平面：

```typescript
const cameraEntity = rootEntity.createChild('camera');
const camera = cameraEntity.addComponent(Camera);

camera.enableFrustumCulling = false;

// 正交投影 - 关键：orthographicSize 决定视野大小
camera.orthographicSize = 10; // 根据棋盘大小调整
camera.nearClipPlane = 0.1;
camera.farClipPlane = 100;
camera.aspectRatio = window.innerWidth / window.innerHeight;

// 相机在 Z 轴正方向，看向 XY 平面（标准的2D视角）
cameraEntity.transform.setPosition(0, 0, 10);
cameraEntity.transform.lookAt(new Vector3(0, 0, 0));

// 窗口适配
window.addEventListener('resize', () => {
  camera.aspectRatio = window.innerWidth / window.innerHeight;
});
```

### orthographicSize 计算

对于棋盘类游戏，`orthographicSize` 应略大于棋盘高度的一半：

```typescript
// 棋盘参数
const BOARD_ROWS = 8;
const TILE_SIZE = 1.0;
const TILE_SPACING = 1.1;
const BOARD_HEIGHT = BOARD_ROWS * TILE_SPACING;

// orthographicSize 是半高，所以要除以2
// 额外加一些边距
const PADDING = 1.0;
camera.orthographicSize = BOARD_HEIGHT / 2 + PADDING;
```

### 常见2D相机问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 棋盘显示不完整 | orthographicSize 太小 | 增大 orthographicSize |
| 内容被压扁 | aspectRatio 不正确 | 使用 window 尺寸而非 canvas 尺寸 |
| 点击检测不准 | 相机位置/朝向问题 | 确保相机在 Z 轴，lookAt(0,0,0) |
| 物体闪烁/消失 | near/far 裁剪面设置 | 调整 nearClipPlane 和 farClipPlane |
| 内容不显示 | 相机位置错误 | 2D相机必须在Z轴方向，如(0,0,10) |

### 消消乐坐标映射（关键）

在3D空间中，Y轴向上为正：

```
Y=+3.5  ← 屏幕上方
Y=0     ← 中心
Y=-3.5  ← 屏幕下方
```

消消乐设计：
```typescript
// 映射关系：row=0（底部）-> Y=-3.5（屏幕下方）
//          row=7（顶部）-> Y=+3.5（屏幕上方）
const BOARD_OFFSET_Y = -(GRID_ROWS - 1) * TILE_SPACING / 2;

function getTilePosition(row: number, col: number): Vector3 {
  const x = BOARD_OFFSET_X + col * TILE_SPACING;
  const y = BOARD_OFFSET_Y + row * TILE_SPACING; // row=0 -> Y=-3.5
  return new Vector3(x, y, 0);
}
```

**关键理解**：
- `row=0` 对应 `Y=-3.5`（视觉底部）
- `row=7` 对应 `Y=+3.5`（视觉顶部）
- 下落动画：row增大 -> Y增大 -> 视觉上**向上移动**？
- **不对！** 相机从Z轴看，Y正方向就是屏幕上方，所以：
  - 下落应该是 **row减小**（从7降到0）
  - 或者 Y坐标**从正到负**（从+3.5降到-3.5）

**相机必须在Z轴方向**：
```typescript
// ✅ 正确：相机在Z轴，看向XY平面
cameraEntity.transform.setPosition(0, 0, 10);
cameraEntity.transform.lookAt(new Vector3(0, 0, 0));

// ❌ 错误：相机在Y轴，看不到XY平面上的内容
cameraEntity.transform.setPosition(0, -10, 0);
```

## 7. 完整示例代码

```typescript
import { WebGLEngine, Camera, Vector3 } from "@galacean/engine";

async function initCamera(rootEntity: Entity) {
  const canvas = document.getElementById('canvas') as HTMLCanvasElement;
  
  // 创建相机实体
  const cameraEntity = rootEntity.createChild('camera');
  const camera = cameraEntity.addComponent(Camera);
  
  // 基础设置
  camera.enableFrustumCulling = false;
  
  // 投影设置（三选一）
  
  // 方案 A：正交投影 - 俯视3D（塔防、RTS）
  // camera.orthographicSize = 14;
  // camera.aspectRatio = window.innerWidth / window.innerHeight;
  // cameraEntity.transform.setPosition(0, 25, 0);
  // cameraEntity.transform.lookAt(new Vector3(0, 0, 0));
  
  // 方案 B：透视投影（3D游戏）
  // camera.fieldOfView = 60;
  // camera.aspectRatio = window.innerWidth / window.innerHeight;
  // cameraEntity.transform.setPosition(0, 20, 10);
  // cameraEntity.transform.lookAt(new Vector3(0, 0, 0));
  
  // 方案 C：正交投影 - 2D（消消乐、棋类）
  camera.orthographicSize = 10;
  camera.nearClipPlane = 0.1;
  camera.farClipPlane = 100;
  camera.aspectRatio = window.innerWidth / window.innerHeight;
  cameraEntity.transform.setPosition(0, 0, 10);
  cameraEntity.transform.lookAt(new Vector3(0, 0, 0));
  
  // 窗口适配
  window.addEventListener('resize', () => {
    camera.aspectRatio = window.innerWidth / window.innerHeight;
  });
  
  return camera;
}
```
