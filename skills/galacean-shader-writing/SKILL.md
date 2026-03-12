---
name: galacean-shader-writing
description: 此 Skill 约定了 Galacean Engine 中自定义着色器的编写方式，包含 Shader.create、BaseMaterial 参数绑定、动画驱动、透明混合与常见排错。
---

# Galacean Shader Writing

## 介绍
本 Skill 用于统一项目里的着色器开发方式，目标是：
1. 快速写出可运行的顶点/片元 Shader。
2. 保持 TypeScript 与 GLSL 参数命名一致。
3. 避免透明混合、UV 方向、动态参数更新中的常见问题。

适用场景：液面波动、遮罩填充、命中特效、扫描线、呼吸发光等 2D/3D 视觉效果。

## 速查入口

- 日常查错优先看：`references/debug-checklist.md`
- 日志打印模板：`references/debug-log-template.md`

## 1. 核心依赖

```typescript
import { Shader, BaseMaterial, Color } from '@galacean/engine'
```

## 2. 命名与结构约定

### Shader 命名
- 使用全局唯一名称，例如：`water-surface-wave-shader-v1`。
- 推荐使用版本后缀（v1/v2/v3），便于迭代时灰度替换。

### Uniform 命名
- 统一使用 `material_` 前缀，例如：
  - `material_BaseColor`
  - `material_Time`
  - `material_WaveStrength`
- TypeScript 中设置时必须和 GLSL 声明完全一致。

### 代码组织
- 用 `getOrCreateXxxShader()` 做缓存，避免重复 `Shader.create(...)`。
- 用 `createXxxMaterial()` 封装默认参数。
- 每帧只更新“动态参数”，不要重复创建材质对象。

## 3. 最小可用模板（可直接改造）

```typescript
const SHADER_NAME = 'custom-effect-shader-v1'

function getOrCreateShader(): Shader {
  let shader = Shader.find(SHADER_NAME)
  if (shader) return shader

  shader = Shader.create(
    SHADER_NAME,
    `
#include <common>
attribute vec3 POSITION;
attribute vec2 TEXCOORD_0;
uniform mat4 renderer_MVPMat;
varying vec2 v_uv;

void main() {
  gl_Position = renderer_MVPMat * vec4(POSITION, 1.0);
  v_uv = TEXCOORD_0;
}
`,
    `
#include <common>
uniform vec4 material_BaseColor;
uniform float material_Time;
uniform float material_Intensity;
varying vec2 v_uv;

void main() {
  float pulse = 0.5 + 0.5 * sin(material_Time * 4.0);
  float alpha = material_BaseColor.a * mix(0.35, 1.0, pulse) * material_Intensity;
  gl_FragColor = vec4(material_BaseColor.rgb, alpha);
}
`
  )

  return shader
}

function createEffectMaterial(engine: any, color: Color): BaseMaterial {
  const material = new BaseMaterial(engine, getOrCreateShader())
  material.isTransparent = true
  material.shaderData.setColor('material_BaseColor', color)
  material.shaderData.setFloat('material_Time', 0)
  material.shaderData.setFloat('material_Intensity', 1)
  return material
}
```

## 4. 动态参数驱动模式

### 帧更新建议
- 时间参数：`material_Time += deltaTime` 或直接传累计时间。
- 强度参数：由状态机或事件系统驱动，范围建议归一化到 [0, 1]。
- 避免每帧 `new BaseMaterial()`；只更新 `shaderData`。

示例：

```typescript
material.shaderData.setFloat('material_Time', elapsed)
material.shaderData.setFloat('material_Intensity', Math.max(0, Math.min(strength, 1)))
```

## 5. 透明与遮罩实践

### 透明对象
- 必须开启：`material.isTransparent = true`。
- 片元输出 alpha 建议显式控制。

### 液体/填充遮罩常用写法
- 用 `smoothstep` 做软边界，避免生硬锯齿。
- 通过 `uvY` 和阈值比较实现“从下到上填充”。

示例片元逻辑：

```glsl
float uvY = 1.0 - v_uv.y;
float fillMask = 1.0 - smoothstep(surfaceY, surfaceY + 0.018, uvY);
float alpha = material_BaseColor.a * fillMask;
```

## 6. 性能建议

1. Shader 全局缓存：用 `Shader.find + Shader.create`。
2. 材质复用：只在颜色或拓扑发生变化时重建。
3. 降低频率：高成本参数可降频更新（如每 2 帧）。
4. 控制分支：片元尽量减少复杂条件和多层 if。
5. 移动端优先：优先使用中低频波形和少量 uniform。

## 7. 常见问题排查

1. 画面全透明：
- 检查 `material.isTransparent`。
- 检查 alpha 是否被乘到 0。

2. 参数不生效：
- 核对 uniform 名称是否完全一致（区分大小写）。
- 确认是更新同一个 material 实例。

3. UV 方向反了：
- 在片元中试 `uvY = 1.0 - v_uv.y`。

4. 波动抖动或跳变：
- 检查时间单位是否统一（秒）。
- 对强度做 clamp，避免异常输入。

## 8. 与项目联动建议

在脚本中推荐按以下分层：
1. `getOrCreate...Shader`：只管 Shader 定义和缓存。
2. `create...Material`：只管默认参数和透明设置。
3. `apply...Shader`：按瓶子或实体动态绑定参数。
4. `restore...Material`：效果结束后恢复基础材质。

这套结构适合当前项目里的倒水液面波动、命中闪白和状态高亮效果。
