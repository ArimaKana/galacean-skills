---
name: galacean-mobile
description: Galacean Engine 移动端游戏开发指南，包含适配、优化和最佳实践
---

# Galacean 移动端游戏开发

## 1. 视口配置（关键）

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
```

完整 HTML 头部配置：

```html
<head>
  <!-- 基础视口 -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
  
  <!-- 禁止自动识别 -->
  <meta name="format-detection" content="telephone=no, date=no, address=no">
  
  <!-- iOS全屏 -->
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  
  <!-- 主题色 -->
  <meta name="theme-color" content="#1a1a2e">
</head>
```

## 2. CSS 移动端适配

```css
* {
  /* 禁止触摸高亮 */
  -webkit-tap-highlight-color: transparent;
  touch-action: none;
  
  /* 禁止文字选中 */
  -webkit-user-select: none;
  user-select: none;
  -webkit-touch-callout: none;
}

html, body {
  /* 禁止橡皮筋效果 */
  overscroll-behavior: none;
  -webkit-overflow-scrolling: none;
  
  /* 全屏显示 */
  width: 100%;
  height: 100%;
  overflow: hidden;
}

/* 安全区域适配（刘海屏） */
#ui {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
}

/* 响应式字体 */
.responsive-text {
  font-size: clamp(14px, 4vw, 18px);
}
```

## 3. 移动端检测

```typescript
// 检测移动端
const isMobile = /iPhone|iPad|iPod|Android|webOS|BlackBerry|Windows Phone/i.test(navigator.userAgent);

// 检测触摸设备
const isTouchDevice = 'ontouchstart' in window || navigator.maxTouchPoints > 0;

// 检测iOS
const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);

// 检测横竖屏
const isPortrait = window.innerHeight > window.innerWidth;
```

## 4. 相机适配

### 基础适配

```typescript
function setupMobileCamera(camera: Camera): void {
  const aspect = window.innerWidth / window.innerHeight;
  
  if (aspect < 1) {
    // 竖屏：缩小视野以适应宽度
    camera.orthographicSize = 9;
  } else {
    // 横屏：根据宽度调整
    camera.orthographicSize = 9 / aspect;
  }
  
  camera.aspectRatio = aspect;
}

// 监听方向变化
window.addEventListener('orientationchange', () => {
  setTimeout(() => {
    setupMobileCamera(camera);
  }, 100);
});
```

### 基于内容尺寸的精确适配（推荐）

根据游戏内容实际尺寸和视窗大小动态计算相机视野，确保内容完整显示：

```typescript
// 游戏内容配置（根据实际游戏设置）
const BOARD_WIDTH = 6;   // 内容宽度（世界单位）
const BOARD_HEIGHT = 6;  // 内容高度（世界单位）
const PADDING = 0.9;     // 边距系数（0-1），越大留白越多

/**
 * 根据视窗尺寸计算相机正交大小
 * 确保游戏内容能完整显示在屏幕内
 */
function calculateOrthoSize(): number {
  const aspect = window.innerWidth / window.innerHeight;
  
  if (aspect < 1) {
    // 竖屏：检查宽度和高度，取较大值确保完整显示
    const heightBasedSize = BOARD_HEIGHT / 2 / PADDING;
    const widthBasedSize = BOARD_WIDTH / 2 / aspect / PADDING;
    return Math.max(heightBasedSize, widthBasedSize);
  } else {
    // 横屏：以高度为基准
    return BOARD_HEIGHT / 2 / PADDING;
  }
}

// 相机初始化
const camera = cameraEntity.addComponent(Camera);
camera.isOrthographic = true;  // 必须启用正交模式！
camera.orthographicSize = calculateOrthoSize();
camera.aspectRatio = window.innerWidth / window.innerHeight;

// 响应式更新
window.addEventListener('resize', () => {
  camera.orthographicSize = calculateOrthoSize();
  camera.aspectRatio = window.innerWidth / window.innerHeight;
});
```

**关键点**：
- 必须设置 `camera.isOrthographic = true`，否则 `orthographicSize` 不会生效
- 竖屏时窄屏幕设备（如手机）需要同时检查宽高，防止内容被截断
- `PADDING` 系数控制四周留白，建议 0.85-0.95

## 5. 触摸事件处理

### 禁止默认行为

```typescript
// 禁止页面滚动
document.addEventListener('touchmove', (e) => {
  e.preventDefault();
}, { passive: false });

// 禁止双击缩放
let lastTouchEnd = 0;
document.addEventListener('touchend', (e) => {
  const now = Date.now();
  if (now - lastTouchEnd <= 300) {
    e.preventDefault();
  }
  lastTouchEnd = now;
}, { passive: false });

// 禁止长按菜单
document.addEventListener('contextmenu', (e) => e.preventDefault());
```

### 触摸反馈

```typescript
class MobileButton extends Script {
  private onTouchStart(): void {
    // 按下效果：缩小
    this.entity.transform.setScale(0.95, 0.95, 1);
  }
  
  private onTouchEnd(): void {
    // 松开效果：恢复
    this.entity.transform.setScale(1, 1, 1);
  }
  
  onAwake(): void {
    const canvas = document.getElementById('canvas');
    
    canvas?.addEventListener('touchstart', () => this.onTouchStart(), { passive: true });
    canvas?.addEventListener('touchend', () => this.onTouchEnd(), { passive: true });
  }
}
```

## 6. 性能优化

### DPR 限制

```typescript
// 限制DPR避免性能问题
const dpr = Math.min(window.devicePixelRatio || 1, 2);
engine.canvas.resizeByClientSize(dpr);
```

### 动画优化

```typescript
// 移动端使用更短的动画时间
const animationDuration = isMobile ? 0.2 : 0.3;

// 减少同时进行的动画数量
const MAX_CONCURRENT_ANIMATIONS = isMobile ? 10 : 20;
```

## 7. 横竖屏适配

### CSS 媒体查询

```css
/* 竖屏 */
@media screen and (orientation: portrait) {
  #game-container {
    width: 100vw;
    height: 100vh;
  }
}

/* 横屏 */
@media screen and (orientation: landscape) {
  #game-container {
    width: 100vw;
    height: 100vh;
  }
}

/* 强制竖屏提示（小屏幕横屏时） */
@media screen and (orientation: landscape) and (max-height: 500px) {
  #orientation-hint {
    display: flex;
  }
  #game-container {
    display: none;
  }
}
```

### 横竖屏切换处理

```typescript
class OrientationManager extends Script {
  onAwake(): void {
    this.checkOrientation();
    window.addEventListener('orientationchange', () => {
      setTimeout(() => this.checkOrientation(), 100);
    });
  }
  
  private checkOrientation(): void {
    const isPortrait = window.innerHeight > window.innerWidth;
    const hint = document.getElementById('orientation-hint');
    
    if (!isPortrait && window.innerHeight < 500) {
      // 小屏幕横屏，显示提示
      hint?.style.setProperty('display', 'flex');
    } else {
      hint?.style.setProperty('display', 'none');
    }
  }
}
```

## 8. UI 适配

### 响应式布局

```typescript
// 根据屏幕尺寸调整UI
function adaptUIForMobile(): void {
  const width = window.innerWidth;
  const isMobile = width < 768;
  
  const scoreElement = document.getElementById('score');
  if (scoreElement) {
    scoreElement.style.fontSize = isMobile ? '16px' : '22px';
    scoreElement.style.padding = isMobile ? '8px 16px' : '10px 30px';
  }
}
```

### 触摸友好的按钮

```css
.mobile-button {
  min-width: 44px;  /* iOS推荐最小触摸尺寸 */
  min-height: 44px;
  padding: 12px 24px;
  
  /* 防止误触 */
  margin: 8px;
  
  /* 触摸反馈 */
  transition: transform 0.1s;
}

.mobile-button:active {
  transform: scale(0.95);
}
```

## 9. 常见移动端问题

### 问题1：点击延迟

**解决**：设置正确的 viewport
```html
<meta name="viewport" content="width=device-width, user-scalable=no">
```

### 问题2：iOS 橡皮筋效果

**解决**：
```css
body {
  overflow: hidden;
  position: fixed;
  width: 100%;
  height: 100%;
}
```

### 问题3：刘海屏适配

**解决**：
```css
#ui {
  padding-top: max(10px, env(safe-area-inset-top));
  padding-bottom: max(10px, env(safe-area-inset-bottom));
}
```

### 问题4：虚拟键盘

**解决**：
```typescript
const initialHeight = window.innerHeight;
window.addEventListener('resize', () => {
  const keyboardHeight = initialHeight - window.innerHeight;
  if (keyboardHeight > 150) {
    // 键盘弹出，调整布局
  }
});
```

## 10. 测试清单

- [ ] iOS Safari 正常显示
- [ ] Android Chrome 正常显示
- [ ] 横竖屏切换正常
- [ ] 触摸点击响应正常
- [ ] 无双击缩放
- [ ] 无页面滚动
- [ ] 刘海屏适配正常
- [ ] 性能流畅（60fps）
