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

## 6. 完整示例代码

```typescript
import { WebGLEngine, Camera, Vector3 } from "@galacean/engine";

async function initCamera(rootEntity: Entity) {
  const canvas = document.getElementById('canvas') as HTMLCanvasElement;
  
  // 创建相机实体
  const cameraEntity = rootEntity.createChild('camera');
  const camera = cameraEntity.addComponent(Camera);
  
  // 基础设置
  camera.enableFrustumCulling = false;
  
  // 投影设置（二选一）
  // 方案 A：正交投影（俯视游戏推荐）
  camera.orthographicSize = 14;
  camera.aspectRatio = canvas.width / canvas.height;
  cameraEntity.transform.setPosition(0, 25, 0);
  cameraEntity.transform.lookAt(new Vector3(0, 0, 0));
  
  // 方案 B：透视投影
  // camera.fieldOfView = 60;
  // camera.aspectRatio = canvas.width / canvas.height;
  // cameraEntity.transform.setPosition(0, 20, 10);
  // cameraEntity.transform.lookAt(new Vector3(0, 0, 0));
  
  // 窗口适配
  window.addEventListener('resize', () => {
    camera.aspectRatio = window.innerWidth / window.innerHeight;
  });
  
  return camera;
}
```
