
---

## 全局语言与沟通规范 (Language & Communication Mandate)

> **最高优先级指令**：本章节定义的语言规范具有最高法律效力，覆盖所有代理（Agents）及其子任务。

- **强制语言**：所有代理与用户之间的沟通、方案建议、任务报告及交互界面必须**全量使用中文**。
- **情景识别与术语**：必须精准识别"游戏工作室"开发场景，使用地道、专业的中文行业术语。
- **自然口语表达**：回复应自然、口语化，模拟工作室资深同事间的真实协作情景。识别并适配特定语境，严禁生硬的"翻译腔"。
- **二次核对机制**：在输出任何内容前，代理必须进行内部审查，确保表达习惯符合中文母语使用者的直觉，逻辑严密且无语病。

---

## Technology Stack

> **引擎与语言**：Unity 6.3 LTS (6000.3) — C#
> **构建系统**：Unity Build Pipeline
> **资源管线**：Unity Asset Import Pipeline + Addressables
> **测试框架**：NUnit (Unity Test Framework)
> **目标平台**：PC (Steam) & PS5

详细技术规范见 [`.claude/docs/technical-preferences.md`](.claude/docs/technical-preferences.md)。

---

## 核心规范索引

| 类别 | 文档 | 说明 |
|------|------|------|
| **技术规范** | [`.claude/docs/technical-preferences.md`](.claude/docs/technical-preferences.md) | 命名规范、性能预算、测试框架 |
| **引擎参考** | [`docs/engine-reference/`](docs/engine-reference/) | Unity 6.3 API 注意事项、事件总线协议 |
| **架构决策** | [`docs/architecture/`](docs/architecture/) | 28 个 ADR，覆盖核心系统设计 |
| **游戏设计** | [`design/gdd/`](design/gdd/) | 19 个 GDD，系统级设计文档 |
| **系统索引** | [`design/gdd/systems-index.md`](design/gdd/systems-index.md) | 系统依赖关系、优先级、MVP 范围 |
| **游戏概念** | [`design/gdd/game-concept.md`](design/gdd/game-concept.md) | 核心玩法、支柱、设计约束 |

---

