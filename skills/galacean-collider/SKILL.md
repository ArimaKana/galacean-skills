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

## 5. 碰撞事件系统 (Collision Events)

### 5.1 事件类型对比

Galacean 物理系统提供两类碰撞事件：

| 事件类型 | 触发条件 | 参数类型 | 物理反应 | 使用场景 |
| :--- | :--- | :--- | :--- | :--- |
| **碰撞器事件** | 两个碰撞器产生物理碰撞 | `Collision` | ✅ 有物理效果 | 真实碰撞交互、伤害检测 |
| **触发器事件** | 碰撞器进入触发区域 | `ColliderShape` | ❌ 无物理效果 | 区域检测、道具拾取 |

**关键区别**：
- 碰撞器模式（`isTrigger = false`）：会产生真实的物理碰撞效果，触发 `onCollision*` 事件
- 触发器模式（`isTrigger = true`）：仅检测重叠，不产生物理反应，触发 `onTrigger*` 事件

### 5.2 碰撞器事件 (Collision Events)

**适用场景**：需要基于真实物理碰撞计算伤害、检测冲击力度等。

**重要概念**：
1. 碰撞事件回调参数是 `Collision` 对象，**不是** `Collider`
2. `Collision` 对象包含：碰撞形状 (`shape`)、接触点信息等
3. 通过 `collision.shape.collider` 访问碰撞器，再通过 `collider.entity` 访问实体

**代码模式**：
```typescript
import { Script, Collision, DynamicCollider, Vector3 } from '@galacean/engine';

class DamageDetectionScript extends Script {
  private collider!: DynamicCollider;
  private health: number = 100;
  private lastCollisionTime: number = 0;
  private collisionCooldown: number = 0.1; // 防止同一次碰撞重复计算

  onAwake() {
    this.collider = this.entity.getComponent(DynamicCollider)!;
  }

  onUpdate(deltaTime: number) {
    this.lastCollisionTime += deltaTime;
  }

  /**
   * 碰撞开始时触发（每次新碰撞触发一次）
   * @param collision 碰撞信息对象
   */
  onCollisionEnter(collision: Collision) {
    this.handleCollision(collision);
  }

  /**
   * 碰撞持续时触发（每帧都会触发，需要做频率控制）
   * @param collision 碰撞信息对象
   */
  onCollisionStay(collision: Collision) {
    // 使用冷却时间避免同一次碰撞重复计算伤害
    if (this.lastCollisionTime > this.collisionCooldown) {
      this.handleCollision(collision);
    }
  }

  /**
   * 碰撞结束时触发
   * @param collision 碰撞信息对象
   */
  onCollisionExit(collision: Collision) {
    console.log('碰撞结束');
  }

  private handleCollision(collision: Collision) {
    // 1. 从 Collision 对象获取碰撞器
    const otherCollider = collision.shape.collider;
    if (!otherCollider) return;

    // 2. 从碰撞器获取实体
    const otherEntity = otherCollider.entity;
    if (!otherEntity) return;

    // 3. 判断碰撞对象类型
    const otherName = otherEntity.name;
    if (otherName !== 'bullet' && otherName !== 'enemy') {
      return; // 忽略不相关的碰撞
    }

    // 4. 获取对方的速度（必须是动态碰撞器）
    const otherDynamicCollider = otherCollider instanceof DynamicCollider 
      ? otherCollider 
      : null;
    if (!otherDynamicCollider) return;

    // 5. 计算相对速度（碰撞强度）
    const otherVelocity = otherDynamicCollider.linearVelocity;
    const myVelocity = this.collider.linearVelocity;
    
    const relativeVelocity = new Vector3(
      Math.abs(otherVelocity.x - myVelocity.x),
      Math.abs(otherVelocity.y - myVelocity.y),
      Math.abs(otherVelocity.z - myVelocity.z)
    );

    const impactSpeed = Math.sqrt(
      relativeVelocity.x * relativeVelocity.x +
      relativeVelocity.y * relativeVelocity.y +
      relativeVelocity.z * relativeVelocity.z
    );

    // 6. 根据冲击速度计算伤害
    if (impactSpeed > 2.0) { // 速度阈值
      const damage = (impactSpeed - 2.0) * 10; // 伤害系数
      this.takeDamage(damage);
      this.lastCollisionTime = 0; // 重置冷却
    }
  }

  private takeDamage(damage: number) {
    this.health -= damage;
    console.log(`受到 ${damage.toFixed(1)} 点伤害，剩余生命: ${this.health.toFixed(1)}`);
    
    if (this.health <= 0) {
      this.entity.destroy();
    }
  }
}
```

### 5.3 触发器事件 (Trigger Events)

**适用场景**：区域检测、道具拾取、传送门等不需要物理反应的场景。

**关键步骤**：
1. 将碰撞形状设置为触发器：`shape.isTrigger = true`
2. 使用 `onTrigger*` 系列事件

**代码模式**：
```typescript
class TriggerScript extends Script {
  /**
   * 进入触发器区域时调用
   * @param shape 进入的碰撞形状
   */
  onTriggerEnter(shape: ColliderShape) {
    const entity = shape.collider.entity;
    if (entity.name === 'player') {
      console.log('玩家进入区域');
    }
  }

  /**
   * 停留在触发器区域时持续调用
   */
  onTriggerStay(shape: ColliderShape) {
    // 持续检测逻辑
  }

  /**
   * 离开触发器区域时调用
   */
  onTriggerExit(shape: ColliderShape) {
    const entity = shape.collider.entity;
    if (entity.name === 'player') {
      console.log('玩家离开区域');
    }
  }
}

// 创建触发器
function createTrigger(engine: WebGLEngine) {
  const entity = new Entity(engine);
  const collider = entity.addComponent(StaticCollider); // 或 DynamicCollider
  
  const triggerShape = new BoxColliderShape();
  triggerShape.size.set(5, 5, 5);
  triggerShape.isTrigger = true; // 设置为触发器模式
  
  collider.addShape(triggerShape);
  entity.addComponent(TriggerScript);
  
  return entity;
}
```

### 5.4 ❌ 反模式：定时轮询检测碰撞

**错误做法**：
```typescript
// ❌ 不要这样做！会漏掉快速碰撞
class BadDamageScript extends Script {
  private checkTimer: number = 0;

  onUpdate(deltaTime: number) {
    this.checkTimer += deltaTime;
    
    // 每 0.1 秒检查一次速度
    if (this.checkTimer > 0.1) {
      this.checkTimer = 0;
      const velocity = this.collider.linearVelocity;
      const speed = velocity.length();
      
      // 问题：如果碰撞发生在两次检查之间，就会错过！
      if (speed > 5.0) {
        this.takeDamage(10);
      }
    }
  }
}
```

**问题**：
- ❌ 碰撞发生在检查间隔之间会被遗漏
- ❌ 无法准确获取碰撞时的瞬时速度
- ❌ 无法区分碰撞对象类型
- ❌ 无法获取碰撞接触点信息

**正确做法**：
```typescript
// ✅ 使用事件驱动的碰撞检测
class GoodDamageScript extends Script {
  onCollisionEnter(collision: Collision) {
    // 每次碰撞都会准确触发
    this.calculateDamage(collision);
  }
}
```

### 5.5 关键要点总结

1. **参数类型**：
   - `onCollision*` 接收 `Collision` 对象
   - `onTrigger*` 接收 `ColliderShape` 对象

2. **访问路径**：
   ```typescript
   collision.shape          // 碰撞的形状
   collision.shape.collider // 碰撞器
   collision.shape.collider.entity // 实体
   ```

3. **事件触发条件**：
   - 至少有一方必须是 `DynamicCollider`
   - 双方都需要有 `Collider` 组件
   - 碰撞器模式触发 `onCollision*`
   - 触发器模式触发 `onTrigger*`

4. **性能优化**：
   - 使用冷却时间避免 `onCollisionStay` 频繁触发
   - 尽早返回（early return）过滤不相关碰撞
   - 使用 `instanceof` 检查碰撞器类型

## 6. 关键概念速查

| 概念 | 类名 | 描述 |
| :--- | :--- | :--- |
| **碰撞体组件** | `DynamicCollider` | **受力运动**。有质量，受重力影响。 |
| | `StaticCollider` | **静止不动**。质量视为无限大，不受力。 |
| **碰撞形状** | `BoxColliderShape` | 盒子形状，拥有 `size` 属性。 |
| | `PlaneColliderShape` | 无限平面，常用于地面。 |
| | `SphereColliderShape` | 球体，拥有 `radius` 属性。 |
| **碰撞事件** | `onCollisionEnter(collision)` | 碰撞开始时触发，参数类型 `Collision` |
| | `onCollisionStay(collision)` | 碰撞持续时触发，需做频率控制 |
| | `onCollisionExit(collision)` | 碰撞结束时触发 |
| **触发器事件** | `onTriggerEnter(shape)` | 进入触发区域，参数类型 `ColliderShape` |
| | `onTriggerStay(shape)` | 停留在触发区域 |
| | `onTriggerExit(shape)` | 离开触发区域 |
| **碰撞信息获取** | `collision.shape.collider` | 从 `Collision` 对象获取碰撞器 |
| | `collision.shape.collider.entity` | 从碰撞器获取实体 |
| | `collider.linearVelocity` | 获取碰撞器的线性速度 |
| **复合碰撞** | N/A | 一个 Collider 组件通过 `addShape()` 添加多个 Shape。 |

## 7. 最佳实践

### ✅ 推荐做法

1. **使用事件驱动**：通过 `onCollision*` 事件检测碰撞，而非定时轮询
2. **类型检查**：使用 `instanceof` 判断碰撞器类型
3. **早期返回**：在事件处理开始时过滤不相关碰撞
4. **冷却控制**：为 `onCollisionStay` 添加冷却时间避免重复计算
5. **相对速度**：计算伤害时使用相对速度而非绝对速度

### ❌ 避免的做法

1. **定时轮询**：不要用 `onUpdate` 轮询速度来判断碰撞
2. **错误参数类型**：不要将 `Collider` 作为 `onCollisionEnter` 的参数类型
3. **错误访问路径**：不要直接从 `Collision` 尝试获取 `entity`，应该通过 `collision.shape.collider.entity`
4. **忽略类型转换**：获取速度前要先检查是否为 `DynamicCollider`