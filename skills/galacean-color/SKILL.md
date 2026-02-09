---
name: galacean-color
description: 此 Skill 包含了 Galacean Engine 中的颜色配置与使用，包括 Color 类、背景色、材质颜色及线性空间转换
---

# Galacean Color Configuration

## 介绍
在 Galacean Engine 中，颜色通常通过 `Color` 类来表示。颜色配置广泛应用于场景背景、材质基色、灯光颜色以及后处理效果中。理解颜色的线性空间 (Linear Space) 和伽马空间 (Gamma Space) 转换对于从美术资产获得正确渲染结果至关重要。

## 1. 核心依赖 (Imports)

使用颜色相关功能通常需要引入：

```typescript
import { Color, Camera, PBRMaterial, DirectLight, WebGLEngine } from "@galacean/engine";
```

## 2. Color 类基础 (Color Basics)

**适用场景**：创建颜色实例，可以在任何需要颜色的 API 中使用。

**核心步骤**：
1.  **实例化**：`new Color(r, g, b, a)`，参数范围通常为 0.0 - 1.0。
2.  **修改值**：直接修改 `r`, `g`, `b`, `a` 属性，或使用 `setValue` 方法。

**代码模式**：

```typescript
// 创建红色 (不透明)
const redColor = new Color(1, 0, 0, 1);

// 创建半透明蓝色
const semiTransparentBlue = new Color(0, 0, 1, 0.5);

// 修改颜色值
const myColor = new Color();
myColor.setValue(0.5, 0.5, 0.5, 1.0); // 灰色
myColor.r = 0.8; // 修改红色通道
```

## 3. 设置场景背景颜色 (Background Color)

**适用场景**：改变渲染画布的清除颜色（Clear Color），即没有物体遮挡时显示的背景色。

**核心步骤**：
1.  获取 `Camera` 组件。
2.  访问 `camera.background`。
3.  设置 `solidColor` 属性。

**代码模式**：

```typescript
// 假设已获取 camera 组件
const camera = rootEntity.addComponent(Camera);

// 设置背景为纯色模式 (通常默认即为 Color)
// camera.background.mode = BackgroundMode.Color; (如果需要显式设置)

// 设置背景颜色为深灰色
camera.background.solidColor = new Color(0.1, 0.1, 0.1, 1);

// 或者修改现有对象
camera.background.solidColor.setValue(0.2, 0.3, 0.4, 1);
```

## 4. 设置材质颜色 (Material Color)

**适用场景**：修改物体表面的基础颜色，常见于 PBR 材质 (`PBRMaterial`) 或无光照材质 (`UnlitMaterial`)。

**核心步骤**：
1.  创建或获取材质。
2.  修改材质的 `baseColor` 属性。

**代码模式**：

```typescript
const mtl = new PBRMaterial(engine);

// 设置基础颜色为金色
mtl.baseColor = new Color(1, 0.84, 0, 1);

// 注意：如果有贴图 (baseTexture)，baseColor 通常会与贴图颜色相乘
```

## 5. 设置灯光颜色 (Light Color)

**适用场景**：改变光源发出的光线颜色，影响被照射物体的视觉效果。

**核心步骤**：
1.  创建或获取灯光组件 (如 `DirectLight`, `PointLight`, `SpotLight`)。
2.  修改 `color` 属性。

**代码模式**：

```typescript
const lightNode = rootEntity.createChild("light");
const directLight = lightNode.addComponent(DirectLight);

// 设置暖色光
directLight.color = new Color(1, 0.9, 0.7, 1);

// 调整光照强度 (通常与颜色分开控制，但在物理单位下颜色值可能超 1)
directLight.intensity = 1.0; 
```

## 6. 线性与 Gamma 空间 (Linear vs Gamma)

**适用场景**：当颜色看起来比预期“灰”或“亮”时，通常涉及色彩空间问题。Galacean 默认通常工作在线性空间。如果从取色器（Gamma 空间）拿到的颜色值，可能需要转换。

**核心步骤**：
1.  使用 `toLinear()` 将 Gamma 空间颜色转换为线性空间。
2.  使用 `toGamma()` 将线性空间颜色转换为 Gamma 空间。

**代码模式**：

```typescript
const color = new Color();

// 假设 hex 字符串或取色器拿到的值是 Gamma 空间的 (例如 sRGB)
color.setValue(0.5, 0.5, 0.5, 1); 

// 转换为线性空间用于渲染
color.toLinear();

// 如果需要导出或显示在 UI 上，可能需要转回 Gamma
// color.toGamma();
```
