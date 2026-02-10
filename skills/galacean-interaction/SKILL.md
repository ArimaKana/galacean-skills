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
```
