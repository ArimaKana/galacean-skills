# Galacean Skills specifically for AI Coding Assistants

本项目包含了一系列针对 Galacean Engine 的 AI 编程助手 Skills 配置。这些 Skills 旨在帮助 AI 更好地理解和生成符合 Galacean Engine 最佳实践的代码。

## 包含的 Skills

本项目目前包含以下 Skills，涵盖了 Galacean 引擎开发的核心模块：

| Skill 名称 | 描述 | 包含内容 |
| :--- | :--- | :--- |
| **galacean-init** | 引擎初始化 | Engine 创建、画布自适应、场景/相机/灯光的基础设置。通常是项目的起点。 |
| **galacean-entity** | 实体与变换 | Entity（实体）的创建、层级树管理、Transform（位置/旋转/缩放）操作。 |
| **galacean-resource** | 资源管理 | 使用 `ResourceManager` 加载常见资产（如 GLTF 模型、纹理、Sprite 等）。 |
| **galacean-animation** | 动画系统 | `Animator` 组件的使用、动画状态机管理、动画剪辑播放与过渡。 |
| **galacean-collider** | 物理碰撞 | `StaticCollider` 与 `DynamicCollider` 的创建与配置，物理形状的添加。 |
| **galacean-interaction** | 交互系统 | 脚本组件中的输入事件处理，例如物体点击 (`onPointerClick`) 等交互逻辑。 |

## 使用方法

这些 Skill 文件通常位于 `.claude/skills` 目录下。根据你使用的 AI 助手的配置方式，将对应的路径注册到上下文中即可生效。

## 贡献

欢迎提交 PR 补充更多 Galacean 功能模块的 Skill 定义。
