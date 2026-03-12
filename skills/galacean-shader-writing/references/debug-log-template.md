# Shader 日志模板片段（可直接粘贴）

用途：快速确认“材质是否挂载成功、uniform 是否真实写入、当前渲染对象是否就是你要调试的对象”。

## 1) 最小日志工具（建议放到 Script 类中）

```typescript
import { BaseMaterial, SpriteRenderer } from '@galacean/engine'

private debugShaderLogEnabled = true
private debugShaderLogFrameInterval = 20
private _debugFrameCount = 0

private logShaderState(
  tag: string,
  renderer: SpriteRenderer | null,
  material: BaseMaterial | null,
  uniforms: Array<{ name: string; type: 'float' | 'color' }>
) {
  if (!this.debugShaderLogEnabled) return
  this._debugFrameCount += 1
  if (this._debugFrameCount % this.debugShaderLogFrameInterval !== 0) return

  console.group(`[ShaderDebug] ${tag}`)

  if (!renderer) {
    console.warn('renderer is null')
    console.groupEnd()
    return
  }

  console.log('renderer.enabled =', renderer.enabled)
  console.log('entity.name =', renderer.entity?.name)

  if (!material) {
    console.warn('material is null')
    console.groupEnd()
    return
  }

  console.log('material.instanceId =', (material as any).instanceId ?? 'N/A')
  console.log('material.isTransparent =', material.isTransparent)

  for (const uniform of uniforms) {
    try {
      if (uniform.type === 'float') {
        const value = material.shaderData.getFloat(uniform.name)
        console.log(`${uniform.name} (float) =`, value)
      } else {
        const value = material.shaderData.getColor(uniform.name)
        if (!value) {
          console.warn(`${uniform.name} (color) = null`)
        } else {
          console.log(
            `${uniform.name} (color) =`,
            `r=${value.r.toFixed(3)} g=${value.g.toFixed(3)} b=${value.b.toFixed(3)} a=${value.a.toFixed(3)}`
          )
        }
      }
    } catch (error) {
      console.warn(`read uniform failed: ${uniform.name}`, error)
    }
  }

  console.groupEnd()
}
```

## 2) 每帧调试调用示例

```typescript
// 例如在 onUpdate 中，对目标水层材质做采样日志
const waters = bottleEntity.findByName('waters')
const top = waters?.children[waters.children.length - 1]
const renderer = top?.getComponent(SpriteRenderer) ?? null
const material = renderer?.getMaterial() as BaseMaterial | null

this.logShaderState('target-top-water', renderer, material, [
  { name: 'material_Time', type: 'float' },
  { name: 'material_WaveStrength', type: 'float' },
  { name: 'material_ImpactCenter', type: 'float' },
  { name: 'material_BaseColor', type: 'color' }
])
```

## 3) 事件前后对比日志模板

```typescript
private logPourEventSnapshot(fromId: number, toId: number, fromMat: BaseMaterial | null, toMat: BaseMaterial | null) {
  console.group(`[ShaderDebug] pourEvent from=${fromId} to=${toId}`)
  console.log('from material id =', (fromMat as any)?.instanceId ?? 'N/A')
  console.log('to material id =', (toMat as any)?.instanceId ?? 'N/A')

  if (fromMat) {
    console.log('from waveStrength =', fromMat.shaderData.getFloat('material_WaveStrength'))
  }
  if (toMat) {
    console.log('to waveStrength =', toMat.shaderData.getFloat('material_WaveStrength'))
  }

  console.groupEnd()
}
```

## 4) 使用注意

1. 线上构建前关闭：`debugShaderLogEnabled = false`。
2. 不要每帧全量打印，建议按帧间隔采样。
3. 优先打印“你怀疑失效的那 2-4 个 uniform”，避免日志噪声。
