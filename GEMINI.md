# Claude Code & Gemini CLI Game Studios 指南

欢迎来到 **Game Studios**。这是一个高度结构化的游戏开发框架。

本项目现已全面支持 **Claude Code** 和 **Gemini CLI**。

---

## Technology Stack

- **Engine**: Godot 4.6.1
- **Language**: GDScript (primary), C++ via GDExtension (performance-critical)
- **Version Control**: Git with trunk-based development
- **Build System**: SCons (engine), Godot Export Templates
- **Asset Pipeline**: Godot Import System + custom resource pipeline

> **注意**: 针对 Godot、Unity 和 Unreal 存在引擎专家代理及其子专家。请使用与你引擎匹配的一套代理。

## Project Structure

@.gemini/docs/directory-structure.md

## Engine Version Reference

@docs/engine-reference/godot/VERSION.md

## Technical Preferences

@.gemini/docs/technical-preferences.md

## Coordination Rules

@.gemini/docs/coordination-rules.md

## Coding Standards

@.gemini/docs/coding-standards.md

## Context Management

@.gemini/docs/context-management.md

---

## 协作协议

**用户驱动的协作，而非自主执行。**
每项任务遵循：**提问 -> 方案 -> 决策 -> 草案 -> 批准**

- 代理在调用写入/编辑工具前，必须询问“我可以将其写入 [文件路径] 吗？”
- 代理在请求批准前，必须展示草案或摘要。
- 多文件变更需要对整个变更集进行明确批准。
- 未经用户指令，严禁进行 commit。

详见 `docs/COLLABORATIVE-DESIGN-PRINCIPLE.md` 了解完整的协议和示例。

## 常用命令 (Skills)

- `/start`: 启动引导流程。
- `/setup-engine <engine> <version>`: 配置游戏引擎。
- `/brainstorm`: 创意头脑风暴。
- `/design-system <system-name>`: 编写游戏设计文档 (GDD)。
- `/sprint-plan new`: 开启新冲刺计划。

> **Gemini CLI 用户**: 请使用 `activate_skill("skill-name")` 来激活上述功能。

## 目录结构
- `.gemini/`: Gemini CLI 框架核心（代理定义、技能、钩子、规则）。
- `src/`: 游戏源代码。
- `assets/`: 游戏资源。
- `design/`: 设计文档 (GDD)。
- `production/`: 生产管理（冲刺计划、里程碑）。
- `tests/`: 测试套件。
