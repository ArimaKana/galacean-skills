---
name: galacean-animation
description: 此 Skill 包含了 Galacean Engine 动画系统的核心模式，包括 Animator 组件的使用、动画播放、过渡以及状态管理
---

# Galacean Animation

## 介绍
Galacean 使用 `Animator` 组件来管理和播放动画。通常这些动画来源于外部导入的 GLTF 模型。

## 1. 核心依赖 (Imports)

```typescript
import { Animator, AnimationClip, AnimatorController, AnimatorControllerLayer, AnimatorStateMachine, AnimatorState } from "@galacean/engine";
```

## 2. 播放 GLTF 动画 (Playing GLTF Animation)

**适用场景**：最常见的用法，播放模型自带的动画（如 "Walk", "Run", "Idle"）。

**前提**：GLTF 加载后，其根节点或特定子节点上通常会自动包含 `Animator` 组件，且 `animations` 数据已准备好。

**代码模式**：

```typescript
// 假设 gltfResource 是加载好的 GLTF 资源
const modelRoot = gltfResource.defaultSceneRoot;
rootEntity.addChild(modelRoot);

// 获取 Animator 组件
const animator = modelRoot.getComponent(Animator);

// 播放指定名称的动画
// 这里的 "Run" 必须匹配 gltfResource.animations 数组中 clip 的名字
if (animator) {
    animator.play("Run"); 
}

// 带有过渡的播放 (Cross Fade)
// 第二个参数是过渡时间(秒)- 0.5s 从当前动作过渡到 Walk
animator.crossFade("Walk", 0.5);
```

## 3. 手动创建动画状态机 (Animator Controller)

**适用场景**：需要更精细地控制动画逻辑，或者需要代码动态组装动画时。

**概念**：
*   **AnimatorController**：动画控制器 asset。
*   **Layer**：动画层，支持多层混合（如一边跑一边射击）。
*   **StateMachine**：状态机。
*   **State**：动画状态，绑定一个 `AnimationClip`。

**代码模式**：

```typescript
// 1. 创建 AnimatorController
const animatorController = new AnimatorController();
const layer = new AnimatorControllerLayer("BaseLayer");
animatorController.addLayer(layer);

const stateMachine = layer.stateMachine;

// 2. 创建状态
const idleState = stateMachine.addState("Idle");
const runState = stateMachine.addState("Run");

// 3. 关联 AnimationClip (假设已有 clip 对象)
// idleState.clip = idleClip;
// runState.clip = runClip;

// 4. 将 Controller 赋值给 Animator 组件
const animator = entity.addComponent(Animator);
animator.animatorController = animatorController;

// 5. 播放
animator.play("Run");
```

## 4. 动画事件 (Animation Events)

**适用场景**：在动画播放到特定帧时触发回调（如脚步声、攻击判定）。

**代码模式**：

```typescript
// 假设有一个 clip
const clip = gltfResource.animations.find(a => a.name === "Attack");

// 定义事件
const event = new AnimationEvent();
event.functionName = "onAttackHit"; // 回调函数名
event.time = 0.5; // 触发时间 (秒)
event.parameter = "Heavy"; // 参数

// 添加事件到 clip
clip.addEvent(event);

// 在 Script 中实现回调
class PlayerScript extends Script {
    onAttackHit(param: string) {
        console.log("Attack Hit triggered with:", param);
    }
}
```
