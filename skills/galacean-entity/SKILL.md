---
name: galacean-entity
description: 此 Skill 包含了 Galacean Engine 中 Entity（实体）的核心操作，包括创建、层级管理、Transform（变换）操作以及全局状态管理
---

# Galacean Entity 操作

## 介绍
`Entity` 是 Galacean Engine 场景中的基本单元。所有的游戏对象（相机、灯光、模型、脚本等）都依附于 Entity。本 Skill 介绍了如何创建实体以及操作其 `Transform` 组件来进行平移、旋转和缩放。

## 1. 实体创建与层级管理

### 创建实体
可以通过 `new Entity()` 创建，或者通过父实体 `createChild()` 创建。

```typescript
import { Entity, WebGLEngine } from "@galacean/engine";

// 方式 1: 直接 new (需要手动添加到场景中)
const entity = new Entity(engine, "MyEntity");
rootEntity.addChild(entity);

// 方式 2: 通过父节点创建 (推荐)
const childEntity = rootEntity.createChild("ChildEntity");
```

### 销毁实体
使用 `destroy()` 方法销毁实体及其所有子节点和组件。

```typescript
entity.destroy();
```

### 查找实体
- `findByName(name)`: 查找子层级中指定名称的实体。
- `parent`: 获取父实体。
- `children`: 获取子实体列表。

```typescript
const target = rootEntity.findByName("TargetName");
```

## 2. 变换操作 (Transform)

每个 Entity 自动包含一个 `Transform` 组件。

### 获取 Transform
```typescript
const transform = entity.transform;
```

### 平移 (Position)

可以直接修改坐标，或使用辅助方法。

```typescript
// 直接设置世界坐标
transform.position.set(10, 0, 0);

// 或者分别设置
transform.position.x = 10;

// 设置相对父节点的局部坐标
transform.position.set(5, 0, 0); // 此时默认修改的是 position，即 localPosition

// 沿当前朝向移动 (比如向前移动 10 个单位)
transform.translate(0, 0, 10, true); // true 表示相对于局部坐标系
```

**注意**: `transform.position` 实际上是引用，但在脚本中修改 `x, y, z` 能够生效。如果想设置世界坐标，可以使用 `transform.worldPosition = new Vector3(...)` (不推荐高频创建对象) 或者使用 `transform.worldPosition.set(...)`。通常直接操作 `position` (局部) 较多，世界坐标通过 `worldPosition` 访问。

### 旋转 (Rotation)

通常使用欧拉角（度数）或四元数。

```typescript
// 设置欧拉角 (角度制) - 局部旋转
transform.rotation.set(0, 90, 0); // 绕 Y 轴旋转 90 度

// 沿特定轴旋转
transform.rotate(0, 1, 0); // 绕 Y 轴转 1 度 (默认局部空间)

// 设置世界旋转 (四元数)
// transform.worldRotationQuaternion.set(...) 
```

### 缩放 (Scale)

设置缩放倍数。

```typescript
// 整体缩放 2 倍
transform.setScale(2, 2, 2);

// 或者
transform.scale.set(2, 2, 2);
```

## 3. 全局状态管理（获取 GameManager）

### 问题背景

Galacean Engine **没有**提供类似 `scene.findComponentByType(GameManager)` 的 API。如果多个脚本需要访问同一个管理器（如 GameManager），需要使用**全局状态模式**。

### 解决方案：单例模式

```typescript
// ==================== GameState.ts ====================
export class GameState {
  private static instance: GameState;
  private _gameManager: GameManager | null = null;

  static getInstance(): GameState {
    if (!GameState.instance) {
      GameState.instance = new GameState();
    }
    return GameState.instance;
  }

  setGameManager(manager: GameManager): void {
    this._gameManager = manager;
  }

  getGameManager(): GameManager | null {
    return this._gameManager;
  }
}

// ==================== GameManager.ts ====================
import { Script } from "@galacean/engine";
import { GameState } from "./GameState";

export class GameManager extends Script {
  onAwake(): void {
    // 注册自己到全局状态
    GameState.getInstance().setGameManager(this);
  }

  gameOver(isWin: boolean): void {
    // 游戏结束逻辑
  }
}

// ==================== EnemyScript.ts ====================
import { Script } from "@galacean/engine";
import { GameState } from "./GameState";

export class EnemyScript extends Script {
  onUpdate(deltaTime: number): void {
    // 检测是否接触玩家...
    
    if (接触玩家) {
      // 通过 GameState 获取 GameManager
      const gameManager = GameState.getInstance().getGameManager();
      if (gameManager) {
        gameManager.gameOver(false);
      }
    }
  }
}
```

### 使用场景

| 场景 | 使用方式 |
|------|----------|
| 敌人碰到玩家，触发游戏结束 | `GameState.getInstance().getGameManager()?.gameOver(false)` |
| 暂停游戏 | `GameState.getInstance().getGameManager()?.pause()` |
| 获取游戏状态 | `GameState.getInstance().getGameManager()?.isPlaying` |

### 注意事项

1. **避免循环依赖**：不要在 GameState 中引用具体脚本类型
2. **类型安全**：getGameManager() 可能返回 null，使用时需要判断
3. **生命周期**：在 Script 的 `onAwake()` 中注册，在 `onDestroy()` 中清理

## 4. 常见脚本示例

```typescript
import { Script, Vector3 } from "@galacean/engine";

export class RotateScript extends Script {
  
  // 每帧调用
  onUpdate(deltaTime: number) {
    // 旋转: 每秒绕 Y 轴旋转 90 度
    // deltaTime 是秒
    this.entity.transform.rotate(0, 90 * deltaTime, 0);
    
    // 移动: 沿自身 Z 轴向前移动
    this.entity.transform.translate(0, 0, 5 * deltaTime, true);
  }
}
```

## 5. 实用工具函数

### 遍历实体层级

```typescript
// 递归遍历所有子实体
function traverseEntities(entity: Entity, callback: (e: Entity) => void): void {
  callback(entity);
  for (const child of entity.children) {
    traverseEntities(child, callback);
  }
}

// 查找特定名称的实体（递归）
function findEntityByName(root: Entity, name: string): Entity | null {
  if (root.name === name) return root;
  for (const child of root.children) {
    const found = findEntityByName(child, name);
    if (found) return found;
  }
  return null;
}
```

### 安全获取组件

```typescript
// 安全获取组件，如果不存在返回 null
function safeGetComponent<T extends Script>(
  entity: Entity,
  componentType: new () => T
): T | null {
  return entity.getComponent(componentType);
}

// 使用示例
const script = safeGetComponent(entity, MyScript);
if (script) {
  script.doSomething();
}
```
