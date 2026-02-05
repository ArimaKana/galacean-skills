---
name: galacean-entity
description: 此 Skill 包含了 Galacean Engine 中 Entity（实体）的核心操作，包括创建、层级管理以及 Transform（变换）操作（平移、旋转、缩放）。
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

## 3. 常见脚本示例

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
