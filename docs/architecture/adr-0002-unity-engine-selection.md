# ADR-0002: Unity 引擎技术选型决策

## Status
**Accepted**

## Date
2026-04-09

## Last Updated
2026-04-09

## Context

### Problem Statement

项目初期曾考虑使用 **Godot Engine** 作为开发引擎，但在 `systems-index.md` 2026-04-07 的更新中，决定统一切换到 **Unity 6.3 LTS**。这一决策需要正式记录。

### Constraints

- **平台目标**：PC & PS5（至少）
- **团队技能**：团队 Unity 经验评级：中级（C# 为主），无 Godot 实战经验
- **工期限制**：需要支持快速原型迭代
- **第三方集成**：需要支持主流 SDK（Steam、PSN 等）
- **授权成本**：独立工作室预算，优先选择免费/低授权费用方案

### Requirements

- **必须**：支持 PS5 平台（Sony 官方支持）
- **必须**：支持主流输入设备（手柄、键鼠）
- **必须**：支持主流音频中间件（Wwise、FMOD）
- **必须**：有成熟的 UI 系统（PS5 需要复杂 UI）
- **应该**：有良好的 C# 生态和社区支持
- **应该**：支持 DOTS/ECS（未来性能优化路径）

---

## Decision

### 架构决策

选定 **Unity 6.3 LTS** 作为《断绝：罪恶之源》的唯一游戏引擎。

```
┌─────────────────────────────────────────────────────────────┐
│                  Unity 6.3 LTS                              │
├─────────────────────────────────────────────────────────────┤
│  Platform: PC (Windows/Mac/Linux) + PS5                      │
│  Language: C#                                                │
│  Rendering: URP (Universal Render Pipeline)                  │
│  UI: UI Toolkit (UXML/USS) + UGUI (Canvas)                  │
│  Physics: Unity Physics (Havok backend)                      │
│  Audio: Wwise / FMOD                                        │
│  Networking: Mirror (P2P, phase-synchronous)                 │
└─────────────────────────────────────────────────────────────┘
```

### 技术栈明细

| 类别 | 选择 | 版本 | 说明 |
|------|------|------|------|
| **引擎** | Unity | 6.3 LTS | 长期支持版本，稳定可靠 |
| **语言** | C# | .NET 8 | Unity IL2CPP 兼容 |
| **渲染管线** | URP | 18.x | 轻量级渲染，支持 PS5 |
| **UI 系统** | UI Toolkit + UGUI | 内置 | UI Toolkit 用于新 UI，UGUI 兼容旧系统 |
| **物理引擎** | Unity Physics | 内置 + Havok | 确定性物理 |
| **输入系统** | Input System Package | 1.10+ | 支持手柄、键鼠统一输入 |
| **音频** | Wwise | 2023.2+ | 音频中间件，PS5 认证 |
| **存档** | Unity Save System | 内置 | Cloud Save 集成 |
| **网络** | Mirror | 最新 LTS | **选定 P2P 方案**（相位同步适合潜行游戏，支持存档续连） |

### 平台支持计划

| 平台 | 优先级 | 说明 |
|------|--------|------|
| **PC (Windows)** | P0 | 主要开发平台，首发 |
| **PC (Mac/Linux)** | P1 | 视发行商要求决定 |
| **PS5** | P0 | 主机平台首发 |
| **Xbox Series X|S** | P2 | 视发行商要求决定 |
| **Nintendo Switch** | P2 | 性能挑战较大 |

---

## Alternatives Considered

### Alternative 1: Godot Engine 4.x

- **描述**：使用 Godot 4.x 作为引擎
- **优点**：
  - 开源免费，无授权费用
  - 轻量级，启动快
  - GDScript/C# 双语言支持
  - 2D 支持业界领先
  - 社区活跃
- **缺点**：
  - PS5 支持需要第三方集成（非官方）
  - 第三方 SDK（Steam/PSN）集成需要自行适配
  - 团队可能缺乏 Godot 实战经验
  - 某些 AAA 游戏特性（如 DOTS）尚未成熟
- **拒绝理由**：
  - **PS5 平台支持不稳定**：Godot 对 PS5 的支持需要自行编译和适配，Sony 认证支持不确定
  - **第三方 SDK 风险**：Wwise、Steam、PSN 等主流 SDK 对 Godot 的支持不如 Unity 完善
  - **团队技能**：项目需要快速开发，团队对 Unity 更加熟悉

### Alternative 2: Unreal Engine 5

- **描述**：使用 Unreal Engine 5 作为引擎
- **优点**：
  - PS5 原生支持，Sony 官方认证
  - 蓝图系统快速原型
  - 画质业界领先（Nanite、Lumen）
  - 完善的 AAA 游戏工具链
- **缺点**：
  - C++ 学习曲线陡峭
  - 引擎体积大，编译慢
  - 对 2D 俯视角游戏略显重量
  - 授权费用较高
- **拒绝理由**：
  - **过度重量**：项目是俯视角潜行游戏，不需要 Nanite/Lumen 的画质能力
  - **团队技能**：团队 C++ 经验较少，Unity C# 更适合快速迭代
  - **开发效率**：Unreal 的编译速度严重影响快速原型迭代效率

### Alternative 3: Unity 2023.x (Non-LTS)

- **描述**：使用 Unity 2023.x 最新版
- **优点**：
  - 最新的 DOTS/ECS 支持
  - 最新的渲染特性
- **缺点**：
  - 非 LTS 版本，可能有稳定性问题
  - 需要频繁升级
  - 不适合长期项目
- **拒绝理由**：
  - **稳定性优先**：游戏开发周期长，需要 LTS 版本的稳定性保障
  - **PS5 认证**：LTS 版本经过更多平台测试

---

## Consequences

### Positive

- **平台稳定性**：Unity LTS 对 PS5 支持经过充分测试
- **团队效率**：团队 C# 经验评级为中级，可快速上手 Unity
- **第三方支持**：Wwise、Steam、PSN 等主流 SDK 官方支持
- **社区资源**：大量教程、插件、解决方案可用
- **授权透明**：Unity Personal（免费）→ Plus（年费）→ Pro（月费）分层清晰
- **未来扩展**：DOTS/ECS 路径为性能优化提供空间

### Negative

- **授权成本**：
  - Personal（收入 < 10万美元）：免费
  - Plus（收入 10-20万美元）：约 $399/年
  - Pro（收入 > 20万美元）：约 $1800/月
  - 本项目预计 < 10万美元，可使用 Personal 免费版本
- **渲染能力**：不如 Unreal 的 Nanite/Lumen，适合但不顶尖
- **引擎体积**：相比 Godot 较大（安装包约 5-10GB）

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **Unity 版本升级** | LTS 版本过期后需要升级 | 规划在 LTS 周期内完成开发；必要时使用 Unity Prolong 支持 |
| **第三方 SDK 兼容性** | SDK 更新可能导致兼容性问题 | 在 Tech Spike 中验证关键 SDK；锁定次要版本 |
| **性能瓶颈** | 俯视角游戏可能遇到 Draw Call 等性能问题 | 使用 URP BatchRenderer；考虑 DOTS 优化路径 |
| **平台认证延迟** | PS5 平台认证可能遇到问题 | 提前与 Sony 对接；使用 Unity 官方认证服务 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | 良好 | DOTS/Job System 可处理大量 NPC |
| **Memory** | 良好 | URP 比 HDRP 更节省内存 |
| **Load Time** | 良好 | Addressables 优化资源加载 |
| **GPU** | 中等 | URP 足够满足俯视角游戏画质需求 |

---

## Migration Plan

### Phase 1: 引擎迁移
- [ ] 废弃所有 Godot 引用
- [ ] 更新 `systems-index.md` 引擎标识为 Unity 6.3 LTS
- [ ] 更新所有 GDD 文档中的引擎引用

### Phase 2: 技术验证
- [ ] 创建 Unity 6.3 LTS 项目骨架
- [ ] 验证 URP 在目标平台的表现
- [ ] 验证 Wwise 与 Unity 的集成
- [ ] 验证 Input System 的手柄支持
- [ ] 验证 Mirror P2P 网络同步（相位同步适合潜行游戏）

### Phase 3: 原型开发
- [ ] 使用 Unity 重做核心玩法原型
- [ ] 验证性能目标能否达成

---

## Validation Criteria

1. **平台验证**：Unity 项目能在 PS5 模拟器上运行核心玩法
2. **性能验证**：60fps@1080p 在 PC 平台稳定运行
3. **输入验证**：手柄 + 键鼠操作流畅，无输入延迟
4. **SDK 验证**：Wwise 音频能正确触发，存档系统正常工作

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 跨系统通信架构
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — Unity 项目结构指导
- [系统索引文档](../../design/gdd/systems-index.md) — 所有系统设计文档引用此 ADR，所有系统应遵循本 ADR 定义的 Unity 6.3 LTS 技术栈
