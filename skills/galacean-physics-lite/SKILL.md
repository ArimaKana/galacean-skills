---
name: galacean-physics-lite
description: 此 Skill 包含了使用 Galacean Engine LitePhysics 物理引擎的注意事项、距离检测替代方案以及常见问题解决方案
---

# Galacean LitePhysics 使用指南

## 1. LitePhysics 简介

LitePhysics 是 Galacean Engine 的轻量级物理引擎，适合移动端和简单物理场景。

### 支持的功能
- 基础碰撞检测
- 射线检测（Raycast）
- 简单点击事件

### 不支持的功能

| 功能 | 支持状态 | 替代方案 |
|------|----------|----------|
| `isTrigger` 触发器模式 | ❌ 不支持 | 使用距离检测 |
| `shape.material` 物理材质 | ❌ 不支持 | 手动实现 |
| 复杂物理模拟（重力、反弹） | ❌ 不支持 | 使用 PhysX 或手动实现 |

## 2. 距离检测替代物理碰撞

由于 LitePhysics 不支持触发器，推荐使用**纯距离检测**实现碰撞。

### 基础距离检测

```typescript
function checkDistanceCollision(
  pos1: Vector3, 
  pos2: Vector3, 
  radius1: number, 
  radius2: number
): boolean {
  const dx = pos2.x - pos1.x;
  const dz = pos2.z - pos2.z;
  const dist = Math.sqrt(dx * dx + dz * dz);
  return dist < (radius1 + radius2);
}
```

### 子弹碰撞检测示例

```typescript
class BulletScript extends Script {
  private radius: number = 0.3;

  onUpdate(deltaTime: number): void {
    this.move(deltaTime);
    this.checkCollision();
  }

  private checkCollision(): void {
    const myPos = this.entity.transform.position;
    const root = this.engine.sceneManager.activeScene.getRootEntity();

    // 遍历所有敌人
    const checkEnemies = (entity: Entity): boolean => {
      for (const child of entity.children) {
        if (child.name === 'enemy' && child.enabled) {
          const enemyPos = child.transform.position;
          const dx = enemyPos.x - myPos.x;
          const dz = enemyPos.z - myPos.z;
          const dist = Math.sqrt(dx * dx + dz * dz);

          // 子弹半径 + 敌人半径
          if (dist < 0.8) {
            const enemyScript = child.getComponent(EnemyScript);
            if (enemyScript) enemyScript.die();
            this.recycle();
            return true;
          }
        }
        if (child.children.length > 0) {
          if (checkEnemies(child)) return true;
        }
      }
      return false;
    };

    checkEnemies(root);
  }
}
```

### 碰撞半径参考

| 对象类型 | 推荐半径 | 说明 |
|----------|----------|------|
| 玩家 | 0.5 | 边长为 1 的立方体 |
| 敌人 | 0.5 | 边长为 1 的立方体 |
| 子弹 | 0.3 | 半径为 0.3 的球体 |

## 3. 常见错误

### 错误："Physics-lite don't support setIsTrigger"

**原因**：尝试设置 `shape.isTrigger = true`

**解决**：移除碰撞体，使用距离检测

```typescript
// ❌ 错误
const collider = entity.addComponent(DynamicCollider);
const shape = new BoxColliderShape();
shape.isTrigger = true;  // LitePhysics 不支持！
collider.addShape(shape);

// ✅ 正确：不使用碰撞体，纯距离检测
// 在 Script 的 onUpdate 中手动检测距离
```

### 错误："collider.getShapes is not a function"

**原因**：使用了错误的 API，LitePhysics 的 Collider 没有 `getShapes()` 方法

**解决**：使用 `shapes` 属性或 `clearShapes()` 方法

```typescript
// ❌ 错误
const shapes = collider.getShapes();

// ✅ 正确
const shapes = collider.shapes;  // 属性，不是方法
collider.clearShapes();  // 清除所有形状
```

## 4. 推荐的物理实现方案

### 俯视射击游戏方案

```typescript
// 不使用任何物理碰撞组件
// 完全依赖距离检测

class PlayerScript extends Script {
  private radius: number = 0.5;
  
  onUpdate(deltaTime: number): void {
    // 移动
    this.handleMovement(deltaTime);
    
    // 碰撞检测（距离检测）
    this.checkEnemyCollision();
  }
  
  private checkEnemyCollision(): void {
    const myPos = this.entity.transform.position;
    
    // 遍历所有敌人检测距离
    // 如果距离 < (playerRadius + enemyRadius)，游戏结束
  }
}

class BulletScript extends Script {
  private radius: number = 0.3;
  
  onUpdate(deltaTime: number): void {
    this.move(deltaTime);
    
    // 检测与敌人的距离
    // 如果距离 < (bulletRadius + enemyRadius)，消灭敌人
  }
}

class EnemyScript extends Script {
  private radius: number = 0.5;
  
  onUpdate(deltaTime: number): void {
    // 向玩家移动
    this.moveTowardsPlayer(deltaTime);
    
    // 检测与玩家的距离
    // 如果距离 < (enemyRadius + playerRadius)，游戏结束
  }
}
```

## 5. 何时使用 LitePhysics

| 场景 | 推荐方案 |
|------|----------|
| 简单点击交互 | ✅ LitePhysics |
| 2D/俯视游戏碰撞 | ✅ LitePhysics + 距离检测 |
| 需要触发器功能 | ❌ 使用 PhysX 或距离检测 |
| 3D 物理模拟 | ❌ 使用 PhysX |
| 复杂碰撞回调 | ❌ 使用 PhysX 或距离检测 |

## 6. 迁移到 PhysX

如果需要更复杂的物理功能，可以迁移到 PhysX：

```typescript
import { PhysXPhysics, PhysXRuntimeMode } from '@galacean/engine-physics-physx';

const engine = await WebGLEngine.create({
  canvas,
  physics: new PhysXPhysics(PhysXRuntimeMode.Auto, {
    wasmModeUrl: './libs/physx.release.js',
    javaScriptModeUrl: './libs/physx.release.downgrade.js'
  }),
});
```

## 7. 关键 API 速查

| API | 说明 | LitePhysics 支持 |
|-----|------|------------------|
| `shape.isTrigger` | 触发器模式 | ❌ 不支持 |
| `collider.shapes` | 获取形状数组 | ✅ 支持 |
| `collider.addShape()` | 添加形状 | ✅ 支持 |
| `collider.removeShape()` | 移除形状 | ✅ 支持 |
| `collider.clearShapes()` | 清除所有形状 | ✅ 支持 |
| `collider.isKinematic` | 运动学模式 | ✅ 支持 |
