---
name: galacean-collider
description: 此 Skill 包含了在 Galacean Engine 中创建物理碰撞体的核心模式，涵盖动态碰撞体 (DynamicCollider) 和 静态碰撞体 (StaticCollider) 的实现
---

# Galacean Collider

## 介绍
为了实现各种碰撞场景，需要给实体添加碰撞行为。

## 1. 核心依赖 (Imports)

在使用物理组件前，需引入以下核心模块：

```typescript
import {
  Entity,
  WebGLEngine,
  StaticCollider,       // 静态碰撞体（如地面、墙壁）
  DynamicCollider,      // 动态碰撞体（如受重力影响的物体）
  BoxColliderShape,     // 盒形碰撞形状
  PlaneColliderShape,   // 无限平面碰撞形状
  SphereColliderShape,  // 球形碰撞形状（可选）
  Vector3
} from '@galacean/engine'
```

## 2. 物理引擎初始化

在创建 `WebGLEngine` 时，需要根据需求配置 `physics` 属性：

```typescript
// 引入物理引擎包
import { LitePhysics } from '@galacean/engine-physics-lite';
import { PhysXPhysics, PhysXRuntimeMode } from '@galacean/engine-physics-physx';

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
```

## 3. 动态碰撞体实现 (Dynamic Collider)

**适用场景**：受重力影响、需要产生物理反应的物体（如掉落的方块）。

**代码模式**：
```typescript
/**
 * 创建一个带有动态物理属性的立方体
 * @param engine 引擎实例
 * @param position 初始位置
 * @param size 尺寸
 */
function createDynamicBox(engine: WebGLEngine, position: Vector3, size: Vector3 = new Vector3(1, 1, 1)) {
  const entity = new Entity(engine);
  entity.transform.position = position;

  // 1. 添加 DynamicCollider 组件
  const dynamicCollider = entity.addComponent(DynamicCollider);
  
  // 2. 配置物理属性
  dynamicCollider.mass = 10.0; // 设置质量
  // dynamicCollider.linearDamping = 0.0; // 线性阻尼
  // dynamicCollider.angularDamping = 0.0; // 角阻尼

  // 3. 创建碰撞形状 (Shape)
  const boxShape = new BoxColliderShape();
  boxShape.size=size
  
  // 4. 将形状添加到碰撞体中
  dynamicCollider.addShape(boxShape);

  return entity;
}
```

## 4. 静态碰撞体实现 (Static Collider)

**适用场景**：虽然参与碰撞但不移动的物体（如地面、墙壁、障碍物）。
**高级技巧**：可以使用一个 StaticCollider 挂载多个 Shape 来构建复杂的环境。

**代码模式**：
```typescript
/**
 * 创建静态环境（包含无限地面和四周墙壁）
 * @param rootEntity 根节点
 */
function createStaticEnvironment(rootEntity: Entity) {
  const groundEntity = rootEntity.createChild('ground');
  
  // 1. 添加 StaticCollider 组件
  const staticCollider = groundEntity.addComponent(StaticCollider);

  // --- 地面 (无限平面) ---
  const planeShape = new PlaneColliderShape();
  // 默认平面法线朝上，通常不需要额外旋转即可作为地面
  staticCollider.addShape(planeShape);

  // --- 墙壁 (使用 BoxShape 构建围栏) ---
  // 通过调整 Shape 的 position (局部偏移) 和 size，在一个实体上构建四面墙
  const width = 4;
  const length = 6;
  const height = 100;
  const thickness = 0.1;

  // 前墙 (Z轴正方向)
  const frontWall = new BoxColliderShape();
  frontWall.size.set(width, height, thickness);
  frontWall.position.set(0, 0, length / 2); // 设置相对于 groundEntity 的偏移
  staticCollider.addShape(frontWall);

  // 后墙 (Z轴负方向)
  const backWall = new BoxColliderShape();
  backWall.size.set(width, height, thickness);
  backWall.position.set(0, 0, -length / 2);
  staticCollider.addShape(backWall);

  // 左/右墙同理...
  
  return groundEntity;
}
```

## 5. 碰撞回调与触发器 (Collision Callbacks)

**适用场景**：需要在发生碰撞或进入触发区域时执行逻辑（如子弹击中敌人、踏入陷阱）。
**关键前提**：
1. 碰撞双方都必须有 `Collider` 组件。
2. 至少有一方必须是 `DynamicCollider`（动态碰撞体）。

**代码模式**：
```typescript
class BulletScript extends Script {
  /**
   * 触发器进入事件：当其他碰撞体进入当前物体的触发区域时调用
   * @param other 碰撞到的对方形状
   */
  onTriggerEnter(other: ColliderShape) {
    // 过滤逻辑：通过 entity.name 判断击中了谁
    // 如果不是 boss 也不是 enemy，则忽略
    if (other.collider.entity.name !== 'boss' && other.collider.entity.name !== 'enemy') {
      return;
    }
    
    console.log('Bullet hit:', other.collider.entity.name);
    // 在此处添加销毁子弹或扣除生命值的逻辑
  }

  // 其他常用回调：
  // onTriggerExit(other: ColliderShape) {}    // 离开触发区域
  // onCollisionEnter(collision: Collision) {} // 发生物理碰撞（非 Trigger）
}
```

## 6. 关键概念速查

| 概念 | 类名 | 描述 |
| :--- | :--- | :--- |
| **碰撞体组件** | `DynamicCollider` | **受力运动**。有质量，受重力影响。 |
| | `StaticCollider` | **静止不动**。质量视为无限大，不受力。 |
| **碰撞形状** | `BoxColliderShape` | 盒子形状，拥有 `size` 属性。 |
| | `PlaneColliderShape` | 无限平面，常用于地面。 |
| | `SphereColliderShape` | 球体，拥有 `radius` 属性。 |
| **复合碰撞** | N/A | 一个 Collider 组件通过 `addShape()` 添加多个 Shape。 |