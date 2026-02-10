---
name: galacean-init
description: 此 Skill 包含了 Galacean Engine 的初始化流程，包括 Engine 创建、场景自适应、相机与灯光的基本设置
---

# Galacean Initialization

## 介绍
在 Galacean Engine 中，创建一个 3D 场景通常需要初始化引擎 (`WebGLEngine`)，设置画布自适应，并配置基本的相机 (`Camera`) 和灯光 (`DirectLight`)。

## 1. 核心依赖 (Imports)

初始化通常需要以下核心类：

```typescript
import { WebGLEngine, Camera, DirectLight, Vector3, Color } from "@galacean/engine";
```

## 2. 引擎初始化 (Engine Setup)

**适用场景**：项目的入口文件，启动游戏循环。

**核心步骤**：
1.  **创建 Engine**。
2.  **启动运行**：调用 `engine.run()`。

**代码模式**：

```typescript
// 引入物理引擎包
import { LitePhysics } from '@galacean/engine-physics-lite';
import { PhysXPhysics, PhysXRuntimeMode } from '@galacean/engine-physics-physx';

function createCanvas(width?: number, height?: number): HTMLCanvasElement {
  const canvas = document.createElement('canvas')
  canvas.id = 'canvas'
  const aspect = window.innerWidth / window.innerHeight
  canvas.width = width || 1280
  canvas.height = height || canvas.width / aspect
  return canvas
}

async function initEngine() {
  const canvas = createCanvas();
  document.body.appendChild(canvas);
  
  // 获取设备像素比 (DPR) - 用于解决高分辨率屏幕模糊问题
  const dpr = window.devicePixelRatio || 1;
  
  // 设置 canvas 实际像素尺寸以匹配设备像素比
  const resizeCanvas = () => {
    const rect = canvas.getBoundingClientRect();
    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;
  };
  resizeCanvas();
  
  const engine = await WebGLEngine.create({
    canvas,
    // 场景 1: 需要真实的物理反馈（如重力、反弹、复杂的物理交互）
    // 需要引用完整的 physics (PhysX)
    physics: new PhysXPhysics(PhysXRuntimeMode.Auto, {
        wasmModeUrl: './libs/physx.release.js',
        javaScriptModeUrl: './libs/physx.release.downgrade.js'
    }),

    // 场景 2: 只需要基础的碰撞检测（Trigger）、点击事件（Raycast）
    // 可以引用轻量的物理引擎 (LitePhysics)
    // physics: new LitePhysics(),

    // 场景 3: 不需要点击事件、碰撞检测
    // 不配置 physics
  });

  // 设置引擎的像素比，确保渲染清晰度
  engine.canvas.resizeByClientSize(dpr);

  canvas.style.touchAction = 'none'
  const scene = engine.sceneManager.activeScene
  const rootEntity = scene.createRootEntity()
  
  // 窗口大小变化时更新 canvas 尺寸
  window.addEventListener('resize', () => {
    engine.canvas.resizeByClientSize(dpr);
  });

  engine.run()
}
```

## 3. 高分辨率屏幕适配 (DPR 处理)

**适用场景**：解决在 Retina 屏、手机等高 DPR (Device Pixel Ratio) 设备上画面模糊的问题。

**核心机制**：
- 获取设备的 `devicePixelRatio`（通常为 1, 2, 3）
- 设置 Canvas 的实际像素尺寸为 CSS 尺寸的 DPR 倍
- 调用引擎的 `resizeByClientSize(dpr)` 方法通知引擎适配

**代码模式**：

```typescript
async function initHighDPREngine() {
  const canvas = document.getElementById('canvas') as HTMLCanvasElement;
  
  // 1. 获取设备像素比
  const dpr = window.devicePixelRatio || 1;
  
  // 2. 初始化时设置 canvas 实际像素尺寸
  const resizeCanvas = () => {
    const rect = canvas.getBoundingClientRect();
    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;
  };
  resizeCanvas();
  
  // 3. 创建引擎
  const engine = await WebGLEngine.create({
    canvas,
    physics: new LitePhysics(),
  });
  
  // 4. 关键：设置引擎的像素比
  engine.canvas.resizeByClientSize(dpr);
  
  // 5. 窗口变化时重新适配
  window.addEventListener('resize', () => {
    engine.canvas.resizeByClientSize(dpr);
    // 如果有相机，也需要更新 aspectRatio
    // camera.aspectRatio = window.innerWidth / window.innerHeight;
  });
  
  return engine;
}
```

**注意事项**：
- 必须在创建引擎后调用 `resizeByClientSize(dpr)`
- 如果手动设置了 `canvas.width/height`，需要与 `resizeByClientSize` 保持一致
- 在 resize 事件中需要同时更新引擎和相机参数

## 4. 场景与环境 (Scene)

- **职责**：设置背景、相机、灯光、后处理效果。
- **相机**：创建 Camera Entity，设置位置和裁剪。
- **灯光**：设置环境光 `scene.ambientLight` 和直射光 `DirectLight`。
- **后处理**：如需 Bloom 等效果，挂载 `PostProcess` 组件。

**代码模式**：

```typescript
export async function createEnvironment(engine: WebGLEngine, rootEntity: Entity) {
  const scene = engine.sceneManager.activeScene

  // 设置背景为透明
  const { background } = scene
  background.mode = BackgroundMode.SolidColor
  background.solidColor.set(1, 1, 1, 0)

  // 创建背景精灵
  const bg = rootEntity.createChild('bg')
  const bgSprite = await engine.resourceManager
    .load({
      url:'xxx.png',
      type: AssetType.Texture2D
    })
  const bgSpriteRenderer = bg.addComponent(SpriteRenderer)
  bgSpriteRenderer.sprite = new Sprite(engine, bgSprite, new Rect(0, 0, 1, 1))
  bgSpriteRenderer.width = 7
  bgSpriteRenderer.height = 13

  // 创建相机
  const cameraEntity = rootEntity.createChild('camera')
  const camera = cameraEntity.addComponent(Camera)
  camera.enableFrustumCulling = false
  console.log('camera', camera)
  cameraEntity.transform.setPosition(0, 0, 15)

  // 添加后处理效果（Bloom）
  const postEntity = rootEntity.createChild('postProcess')
  const postProcess = postEntity.addComponent(PostProcess)
  const bloomEffect = postProcess.addEffect(BloomEffect)
  bloomEffect.intensity.value = 2
  bloomEffect.scatter.value = 0.8
  bloomEffect.threshold.value = 0.5
  console.log('postProcess', postProcess)

  // 设置场景灯光
  scene.ambientLight.diffuseSolidColor.set(1, 1, 1, 1)
  scene.ambientLight.diffuseIntensity = 0.6
  const lightEntity = rootEntity.createChild('light')
  lightEntity.addComponent(DirectLight)
  lightEntity.transform.setPosition(1, 1, 1)
  lightEntity.transform.lookAt(new Vector3())
}
```
