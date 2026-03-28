# Gemini CLI Game Studios -- 游戏工作室代理架构

通过 48 个协同的 Gemini CLI 子代理管理的独立游戏开发框架。
每个代理拥有特定领域，强制执行关注点分离和质量控制。

## 技术栈

- **引擎**: [选择: Godot 4 / Unity / Unreal Engine 5]
- **语言**: [选择: GDScript / C# / C++ / Blueprint]
- **版本控制**: Git (遵循 Trunk-based 模式)
- **构建系统**: [选择引擎后指定]
- **资产流水线**: [选择引擎后指定]

> **注意**: 针对 Godot、Unity 和 Unreal 存在引擎专家代理及其子专家。请使用与你引擎匹配的一套代理。

## 项目结构

参见 `.gemini/docs/directory-structure.md`

## 引擎版本参考

参见 `docs/engine-reference/godot/VERSION.md`

## 技术偏好

参见 `.gemini/docs/technical-preferences.md`

## 协同规则

参见 `.gemini/docs/coordination-rules.md`

## 协作协议

**用户驱动的协作，而非自主执行。**
每项任务遵循：**提问 -> 方案 -> 决策 -> 草案 -> 批准**

- 代理在调用写入/编辑工具前，必须询问“我可以将其写入 [文件路径] 吗？”
- 代理在请求批准前，必须展示草案或摘要。
- 多文件变更需要对整个变更集进行明确批准。
- 未经用户指令，严禁进行 commit。

详见 `docs/COLLABORATIVE-DESIGN-PRINCIPLE.md` 了解完整的协议和示例。

> **首次会话？** 如果项目尚未配置引擎且没有游戏概念，请运行 `/start`（通过 `activate_skill("start")`）开始引导式入职流程。

## 编码标准

参见 `.gemini/docs/coding-standards.md`

## 上下文管理

参见 `.gemini/docs/context-management.md`
