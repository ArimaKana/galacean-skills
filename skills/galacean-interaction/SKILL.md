---
name: galacean-interaction
description: 此 Skill 包含了在 Galacean Engine 中通过 Script 脚本处理物体交互（如点击）的核心模式，基于 onPointerClick 实现
---

# Galacean Interaction

## 介绍
在 Galacean Engine 中，通过编写继承自 `Script` 的脚本组件并实现 `onPointerClick` 等生命周期函数，可以轻松实现物体的点击交互。

## 1. 核心依赖 (Imports)

需引入 Script 类以及可能需要操作的碰撞体组件：

```typescript
import { Script, Entity, DynamicCollider, StaticCollider } from '@galacean/engine'
```

## 2. 点击交互实现 (Click Interaction)

**适用场景**：物体被鼠标或触摸点击时触发逻辑（如消除方块、触发跳跃）。

**核心机制**：
1. **Script 组件**：继承 `Script` 并实现 `onPointerClick()`。
2. **Collider 依赖**：实体必须挂载 `DynamicCollider` 或 `StaticCollider`，引擎的输入系统会通过物理射线检测来触发点击事件。

**代码模式**：

```typescript
class InteractionScript extends Script {
  // 可选：缓存 Collider 组件以便在点击时修改其状态
  private _collider?: DynamicCollider;

  onAwake() {
    // 获取同级挂载的 Collider 组件
    this._collider = this.entity.getComponent(DynamicCollider)!;
  }

  /**
   * 当指针（鼠标/触摸）点击该物体时触发
   */
  onPointerClick() {
    // 1. 执行交互逻辑，例如修改游戏状态
    console.log("Clicked:", this.entity.name);

    // 2. 操作 Collider (例如禁用，防止再次被点击)
    if (this._collider) {
      this._collider.enabled = false;
    }

    // 3. 触发其他行为 (如动画、移动)
    // this.doSomething();
  }
}
```

## 3. 常见用法示例

结合实际应用场景，点击交互常包含以下步骤：

1. **状态检查**：检查游戏状态是否允许交互（如堆叠上限）。
2. **禁用碰撞**：点击后立即禁用碰撞体，防止重复触发。
3. **视觉反馈**：修改 Transform (缩放) 或触发 Tween 动画。
4. **数据更新**：通知全局 Store 更新数据。

```typescript
// 示例：点击方块逻辑
onPointerClick() {
  // Check game state
  if (!canInteract) return;

  // Disable physics interaction
  if (this._collider) {
    this._collider.enabled = false;
  }

  // Visual feedback
  this.entity.transform.setScale(0.8, 0.8, 0.8);

  // Trigger movement or logic
  this.startMoveAnimation();
}
```

## 4. 全局点击实现 (Global Click)

**适用场景**：需要监听整个屏幕的点击事件（如点击屏幕任意位置发射子弹、点击屏幕继续游戏），而不需要判定点击了哪个具体 3D 物体。

**实现方式**：直接对 Canvas 元素添加事件监听器。

**代码模式**：

建议在专门的控制脚本（如 `GameControlScript`）中管理全局事件：

```typescript
export class GameControlScript extends Script {
  
  onAwake() {
    // 1. 获取 Canvas 元素 (假设 ID 为 'canvas')
    const canvas = document.getElementById('canvas');
    
    // 2. 绑定原生事件
    if (canvas) {
      canvas.addEventListener('pointerdown', this.handlePointerDown);
    }
  }

  // 使用箭头函数以保留 this 上下文
  handlePointerDown = (e: PointerEvent) => {
    // 3. 处理点击逻辑
    console.log("Global Click:", e.clientX, e.clientY);
    
    // 示例：根据点击位置发射子弹或处理 UI
    // this.shoot(e.clientX, e.clientY);
  }

  onDestroy() {
    // 4. 清理事件监听，防止内存泄漏
    const canvas = document.getElementById('canvas');
    if (canvas) {
      canvas.removeEventListener('pointerdown', this.handlePointerDown);
    }
  }
}
```

## 5. 摇杆/触摸控制最佳实践

**适用场景**：移动设备上的虚拟摇杆控制，需要全屏响应触摸事件。

### 核心要点

1. **全屏触摸**：不要在 touchstart 中限制触摸区域（如只响应左半边），让用户可以在屏幕任意位置开始控制
2. **事件处理**：同时处理 `touchend` 和 `touchcancel`，避免意外情况下摇杆卡住
3. **坐标转换**：使用 `getBoundingClientRect()` 获取准确的触摸位置

### 代码模式

```typescript
export class JoystickScript extends Script {
  private joystickActive: boolean = false;
  private joystickCenter: { x: number, y: number } = { x: 0, y: 0 };
  private joystickRadius: number = 60;

  onAwake() {
    const canvas = document.getElementById('canvas');
    if (!canvas) return;

    // 触摸开始 - 全屏任意位置都可触发
    canvas.addEventListener('touchstart', (e: TouchEvent) => {
      const touch = e.touches[0];
      const rect = canvas.getBoundingClientRect();
      const x = touch.clientX - rect.left;
      const y = touch.clientY - rect.top;

      // 不要在 here 限制区域，让全屏都可以触发摇杆
      this.joystickActive = true;
      this.joystickCenter = { x: touch.clientX, y: touch.clientY };
      
      // 显示摇杆UI在触摸位置
      this.showJoystickAt(x, y);
    });

    // 触摸移动
    canvas.addEventListener('touchmove', (e: TouchEvent) => {
      if (!this.joystickActive) return;
      e.preventDefault(); // 防止页面滚动
      
      const touch = e.touches[0];
      this.updateJoystick(touch.clientX, touch.clientY);
    });

    // 触摸结束
    canvas.addEventListener('touchend', () => {
      this.resetJoystick();
    });
    
    // 触摸取消（如来电打断）
    canvas.addEventListener('touchcancel', () => {
      this.resetJoystick();
    });
  }

  private updateJoystick(touchX: number, touchY: number): void {
    const dx = touchX - this.joystickCenter.x;
    const dy = touchY - this.joystickCenter.y;
    const distance = Math.sqrt(dx * dx + dy * dy);

    // 限制摇杆移动范围
    let normalizedX = dx;
    let normalizedY = dy;
    if (distance > this.joystickRadius) {
      normalizedX = (dx / distance) * this.joystickRadius;
      normalizedY = (dy / distance) * this.joystickRadius;
    }

    // 计算归一化方向 (-1 到 1)
    const dirX = normalizedX / this.joystickRadius;
    const dirY = normalizedY / this.joystickRadius;
    
    // 应用移动逻辑
    this.applyMovement(dirX, dirY);
  }

  private resetJoystick(): void {
    this.joystickActive = false;
    this.hideJoystick();
    this.applyMovement(0, 0); // 停止移动
  }

  // 抽象方法：由子类实现
  showJoystickAt(x: number, y: number): void {}
  hideJoystick(): void {}
  applyMovement(x: number, y: number): void {}
}
```

## 6. 移动端触摸松开事件处理 (Touch Release)

### 6.1 问题描述

在使用 Galacean 的 `inputManager.pointers` 处理拖拽交互时，移动端触摸松开（touchend）会遇到一个关键问题：

**问题**：当手指从屏幕上松开时，`pointers` 数组会立即变为空，导致 `PointerPhase.Up` 事件无法被捕获。

**症状**：
- 鼠标操作（PC端）：拖拽松开正常工作 ✅
- 触摸操作（移动端）：拖拽后松手无反应，物体卡在拖拽状态 ❌

### 6.2 错误的实现方式

```typescript
// ❌ 这种方式在移动端会失败
onUpdate() {
  const pointers = this.engine.inputManager.pointers;
  
  if (pointers && pointers.length > 0) {
    const pointer = pointers[0];
    
    if (pointer.phase === PointerPhase.Down) {
      this.startDrag();
    }
    else if (pointer.phase === PointerPhase.Move && this.isDragging) {
      this.updateDrag();
    }
    else if (pointer.phase === PointerPhase.Up && this.isDragging) {
      // ❌ 在移动端，手指松开后 pointers 数组为空
      // 这段代码永远不会执行！
      this.endDrag();
    }
  }
}
```

**问题分析**：
1. 触摸按下时：指针加入 `pointers` 数组，`phase = Down`
2. 触摸移动时：指针仍在数组中，`phase = Move`
3. 触摸松开时：指针从数组中移除，`pointers.length = 0`，导致整个 `if` 块不执行

### 6.3 正确的实现方式

**核心思路**：当 `pointers` 为空但 `isDragging` 仍为 `true` 时，说明手指刚刚松开。

```typescript
// ✅ 兼容 PC 和移动端的正确实现
export class DragScript extends Script {
  private isDragging: boolean = false;
  private currentDragPos: Vector3 = new Vector3();

  onUpdate() {
    const inputManager = this.engine.inputManager;
    const pointers = inputManager.pointers;

    if (pointers && pointers.length > 0) {
      const pointer = pointers[0];

      // 开始拖拽（鼠标按下 / 触摸开始）
      if (pointer.phase === PointerPhase.Down) {
        this.handlePointerDown(pointer);
      }
      // 拖拽中（鼠标移动 / 触摸移动）
      else if (pointer.phase === PointerPhase.Move && this.isDragging) {
        this.handlePointerMove(pointer);
      }
      // 结束拖拽（鼠标松开 - 仅 PC 端会触发）
      else if (pointer.phase === PointerPhase.Up && this.isDragging) {
        this.handlePointerUp();
      }
    }
    // ✅ 关键：处理移动端触摸松开
    // 当 pointers 为空但还在拖拽状态时，说明手指刚松开
    else if (this.isDragging) {
      this.handlePointerUp();
    }
  }

  private handlePointerDown(pointer: any) {
    // 检查是否点击在目标区域
    const worldPos = this.screenToWorld(pointer.position.x, pointer.position.y);
    if (this.isInTargetArea(worldPos)) {
      this.isDragging = true;
    }
  }

  private handlePointerMove(pointer: any) {
    const worldPos = this.screenToWorld(pointer.position.x, pointer.position.y);
    if (worldPos) {
      this.currentDragPos.copyFrom(worldPos);
      this.updateDragPosition(worldPos);
    }
  }

  private handlePointerUp() {
    if (!this.isDragging) return;
    
    this.isDragging = false;
    
    // 使用 currentDragPos 计算松手时的位置/速度
    // 注意：此时无法从 pointer 获取位置，必须使用之前保存的值
    this.performAction(this.currentDragPos);
  }

  private performAction(finalPos: Vector3) {
    // 执行松手后的逻辑（如发射、释放等）
    console.log('Released at:', finalPos);
  }
}
```

### 6.4 关键要点

1. **保存拖拽状态**：
   - 使用 `isDragging` 标志跟踪拖拽状态
   - 使用 `currentDragPos` 保存最后的拖拽位置

2. **双重松开检测**：
   ```typescript
   // PC 端：pointer.phase === PointerPhase.Up
   if (pointer.phase === PointerPhase.Up && this.isDragging) {
     this.handlePointerUp();
   }
   
   // 移动端：pointers 为空 + isDragging 为 true
   else if (this.isDragging) {
     this.handlePointerUp();
   }
   ```

3. **不依赖 pointer 参数**：
   - `handlePointerUp()` 不应依赖 `pointer` 参数
   - 松手时需要的数据应该提前保存（如 `currentDragPos`）

4. **状态重置**：
   - 在 `handlePointerUp()` 中立即设置 `isDragging = false`
   - 防止下一帧重复触发

### 6.5 实际应用场景

**弹弓发射**（Angry Birds 风格）：
```typescript
export class SlingshotScript extends Script {
  private isDragging: boolean = false;
  private dragStartPos: Vector3 = new Vector3();
  private currentDragPos: Vector3 = new Vector3();

  onUpdate() {
    const pointers = this.engine.inputManager.pointers;

    if (pointers && pointers.length > 0) {
      const pointer = pointers[0];

      if (pointer.phase === PointerPhase.Down) {
        const worldPos = this.screenToWorld(pointer.position);
        if (this.isNearBird(worldPos)) {
          this.isDragging = true;
          this.dragStartPos.copyFrom(worldPos);
        }
      }
      else if (pointer.phase === PointerPhase.Move && this.isDragging) {
        const worldPos = this.screenToWorld(pointer.position);
        this.currentDragPos.copyFrom(worldPos);
        this.updateBirdPosition(worldPos);
      }
      else if (pointer.phase === PointerPhase.Up && this.isDragging) {
        this.launchBird();
      }
    }
    // ✅ 移动端松手处理
    else if (this.isDragging) {
      this.launchBird();
    }
  }

  private launchBird() {
    this.isDragging = false;
    
    // 使用保存的位置计算发射速度
    const velocity = new Vector3();
    Vector3.subtract(this.dragStartPos, this.currentDragPos, velocity);
    velocity.scale(5);
    
    this.bird.launch(velocity);
  }
}
```

### 6.6 调试技巧

如果遇到触摸事件问题，可以添加日志：

```typescript
onUpdate() {
  const pointers = this.engine.inputManager.pointers;
  
  console.log('Pointers:', pointers?.length, 'Dragging:', this.isDragging);
  
  if (pointers && pointers.length > 0) {
    console.log('Pointer phase:', pointers[0].phase);
  }
  
  // ... 其他逻辑
}
```

**预期输出**：
- 触摸按下：`Pointers: 1 Dragging: false` → `Pointers: 1 Dragging: true`
- 触摸移动：`Pointers: 1 Dragging: true`
- 触摸松开：`Pointers: 0 Dragging: true` → `Pointers: 0 Dragging: false`

### 6.7 最佳实践总结

| 场景 | 推荐做法 | 避免做法 |
|------|---------|---------|
| **检测松手** | 检查 `pointers.length === 0 && isDragging` | 只检查 `pointer.phase === Up` |
| **保存位置** | 在 Move 阶段持续保存 `currentDragPos` | 在 Up 时才从 pointer 读取 |
| **状态管理** | 使用 `isDragging` 布尔标志 | 依赖 pointers 数组判断状态 |
| **参数依赖** | `handlePointerUp()` 使用保存的数据 | `handlePointerUp(pointer)` 依赖参数 |

````
