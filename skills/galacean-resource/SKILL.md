---
name: galacean-resource
description: 此 Skill 包含了在 Galacean Engine 中通过 ResourceManager 加载资源（GLTF 模型、纹理等）的核心模式
---

# Galacean Resource Loading

## 介绍
Galacean Engine 使用 `ResourceManager` 来管理资源的加载。支持 GLTF/GLB 模型、纹理 (Texture2D)、材质等多种资源的异步加载。

## 1. 核心依赖 (Imports)

需引入引擎和资源加载相关的类型：

```typescript
import { WebGLEngine, GLTFResource, Texture2D, AssetType } from "@galacean/engine";
```

## 2. 基础加载模式 (Basic Loading)

**适用场景**：在游戏初始化阶段加载必要的 3D 模型或图片。

**核心机制**：
*   **engine.resourceManager.load()**：通用的加载方法，支持传入 URL 字符串或配置对象。
*   **Promise**：加载过程是异步的，推荐使用 `async/await` 或 `.then()`。

**代码模式**：

```typescript
// 假设在 init 函数中
async function loadResources(engine: WebGLEngine, rootEntity: Entity) {
  // 1. 加载 GLTF 模型
  // load 方法会根据后缀自动推断类型，也可以通过 type 指定
  const gltfResource = await engine.resourceManager.load<GLTFResource>("https://example.com/duck.glb");
  
  // 实例化模型到场景中
  const defaultSceneRoot = gltfResource.defaultSceneRoot;
  rootEntity.addChild(defaultSceneRoot);
  
  // 2. 加载纹理图片
  const texture = await engine.resourceManager.load<Texture2D>("https://example.com/texture.png");
  
  console.log("Resources loaded successfully");
}
```

## 3. 并行加载 (Parallel Loading)

**适用场景**：同时加载多个资源以提高初始化效率。

**代码模式**：

```typescript
const [duckGLTF, terrainGLTF, bgTexture] = await Promise.all([
  engine.resourceManager.load<GLTFResource>("https://path/to/duck.glb"),
  engine.resourceManager.load<GLTFResource>("https://path/to/terrain.glb"),
  engine.resourceManager.load<Texture2D>("https://path/to/bg.png")
]);

// 添加到场景
rootEntity.addChild(duckGLTF.defaultSceneRoot);
rootEntity.addChild(terrainGLTF.defaultSceneRoot);
```

## 4. 带配置的加载 (Advanced Loading)

**适用场景**：需要指定资源类型（当 URL 无后缀时）或设置重试次数等参数。

**代码模式**：

```typescript
engine.resourceManager.load({
  url: "https://api.example.com/get-model-data",
  type: AssetType.GLTF, // 显式指定类型
  retryCount: 3         // 失败重试次数
}).then((resource) => {
    const gltf = resource as GLTFResource;
    rootEntity.addChild(gltf.defaultSceneRoot);
});
```

## 5. GLTF 结构处理

**适用场景**：加载模型后需要播放动画或修改材质。

**代码模式**：

```typescript
engine.resourceManager.load<GLTFResource>("model.gltf").then((gltf) => {
  // 1. 添加到场景
  rootEntity.addChild(gltf.defaultSceneRoot);
  
  // 2. 获取动画 (如果有)
  const animations = gltf.animations;
  if (animations && animations.length > 0) {
     const animator = gltf.defaultSceneRoot.getComponent(Animator);
     animator.play(animations[0].name);
  }
  
  // 3. 获取材质/纹理等其他资源
  // gltf.materials, gltf.textures, etc.
});
```
