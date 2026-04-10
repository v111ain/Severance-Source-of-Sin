# ADR-0003: 系统分层架构定义（System Layer Architecture）

## Status
**Accepted**

## Date
2026-04-09

## Last Updated
2026-04-09

## Context

### Problem Statement

《断绝：罪恶之源》包含 15+ 个游戏系统（如 Health, NPC AI, LOS, Gritty Takedowns 等），如果没有清晰的分层架构，会导致：

1. **依赖混乱**：系统间依赖关系不明确，不知道谁可以调用谁
2. **循环依赖**：A→B→C→A 形成循环，无法独立测试
3. **开发顺序混乱**：不知道先做哪个系统，后做哪个
4. **代码组织**：代码文件夹结构混乱，难以定位

### Constraints

- 必须支持多系统并行开发
- 必须支持独立测试（Unit Test）
- 必须防止循环依赖
- 必须符合 Unity 项目结构最佳实践

### Requirements

- **必须**：定义所有系统的分层（Layer）
- **必须**：定义层间依赖规则（同层不能互调，上层可以调用下层）
- **必须**：明确每层的职责
- **必须**：提供 Unity 项目文件夹结构映射

---

## Decision

### 架构决策

采用**分层架构（Layered Architecture）**，将所有系统归入 7 个层次：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        分层架构图                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Presentation Layer (表现层)                                 │   │
│  │  职责: HUD、菜单、视觉效果、音频播放                        │   │
│  │  系统: UI System, Dynamic Post-Processing, Immersive Audio │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│                              │ 调用                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Meta / Narrative Layer (元叙事层)                         │   │
│  │  职责: 玩家心理状态、线索收集、任务进度                    │   │
│  │  系统: Sanity/Rage Meter, Clue & Journal                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│                              │ 调用                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Feature Layer (功能层)                                     │   │
│  │  职责: 核心玩法功能，组合基础能力                           │   │
│  │  系统: Gritty Takedowns, Weapon System                     │   │
│  │  依赖: Foundation + Core                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│                              │ 调用                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Core Layer (核心层)                                         │   │
│  │  职责: AI、感知、环境交互                                   │   │
│  │  系统: NPC AI System, LOS & Eavesdropping, Environment      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│                              │ 调用                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Foundation Layer (基础层)                                  │   │
│  │  职责: 玩家输入、伤害计算、世界导航                         │   │
│  │  系统: Player Controller, Health & Lethality, World Map      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  跨层横向系统（独立于分层体系外，被所有层依赖，但不参与分层调用）   │
│  Infrastructure Layer: Event Bus, Save System, Addressables,        │
│                       Screen Effects, Network System                │
├─────────────────────────────────────────────────────────────────────┤
│  World Layer（影响多个层的环境系统，独立调用）                       │
│  World Layer: Weather System, Lighting System                       │
└─────────────────────────────────────────────────────────────────────┘
```

### World Layer 说明

World Layer 包含影响多个层的环境系统。这些系统的特点：
- **独立于分层调用链**：不调用其他游戏系统，也不被调用
- **通过 Event Bus 影响其他系统**：如天气变化触发 NPC 感知折扣、视觉滤镜调整等
- **实现位置**：代码放在 Core Layer 的 Environment 子目录，但逻辑上属于 World Layer

### 层次详细定义

#### 0. Infrastructure Layer（横向基础设施层）

| 特性 | 说明 |
|------|------|
| **职责** | 跨层通用能力，不属于具体游戏逻辑，是所有层的公共依赖 |
| **依赖** | 无（独立于分层体系外） |
| **被依赖** | 所有层都依赖它（但它不调用任何层） |
| **不属于调用链** | 它是被动服务，不主动调用任何系统 |
| **系统** | Event Bus, Save System, Addressables, Screen Effects, Network System |

#### 1. Foundation Layer（基础层）

| 特性 | 说明 |
|------|------|
| **职责** | 游戏底层规则，所有其他系统依赖的基石 |
| **依赖** | Infrastructure Layer |
| **调用** | 可被 Core/Feature/Meta/Presentation 层调用 |
| **系统** | Player Controller, Health & Lethality, World Map & Progression |

#### 2. Core Layer（核心层）

| 特性 | 说明 |
|------|------|
| **职责** | 游戏核心能力（AI、感知、环境） |
| **依赖** | Foundation Layer |
| **调用** | 可被 Feature/Meta/Presentation 层调用 |
| **系统** | NPC AI System, LOS & Eavesdropping, Environment Interaction |

#### 2.5. World Layer（世界层）

| 特性 | 说明 |
|------|------|
| **职责** | 影响多个层的环境系统，通过 Event Bus 被动影响 |
| **依赖** | Infrastructure Layer；影响 Core/Feature/Meta/Presentation 层 |
| **调用** | 不主动调用其他系统；通过 Event Bus 广播环境变化事件 |
| **系统** | Weather System, Lighting System |
| **实现位置** | 代码放在 Core/Environment 子目录，但逻辑上属于 World Layer |
| **职责** | 游戏核心能力（AI、感知、环境） |
| **依赖** | Foundation Layer |
| **调用** | 可被 Feature/Meta/Presentation 层调用 |
| **系统** | NPC AI System, LOS & Eavesdropping, Environment Interaction |

#### 3. Feature Layer（功能层）

| 特性 | 说明 |
|------|------|
| **职责** | 组合基础能力形成完整玩法 |
| **依赖** | Foundation + Core Layer |
| **调用** | 可被 Meta/Presentation 层调用 |
| **系统** | Gritty Takedowns, Weapon System |

#### 4. Meta / Narrative Layer（元叙事层）

| 特性 | 说明 |
|------|------|
| **职责** | 玩家心理状态、叙事内容 |
| **依赖** | Feature Layer |
| **调用** | 可被 Presentation 层调用 |
| **系统** | Sanity/Rage Meter, Clue & Journal |

#### 5. Presentation Layer（表现层）

| 特性 | 说明 |
|------|------|
| **职责** | 玩家可见/可听的一切 |
| **依赖** | 所有下层 |
| **调用** | 调用下层，不被任何层调用 |
| **系统** | UI System, Dynamic Post-Processing, Immersive Audio |

### Unity 项目结构映射

```
Assets/
├── Game/
│   │
│   ├── Foundation/              # Foundation Layer
│   │   ├── PlayerController/
│   │   ├── Health/
│   │   └── WorldMap/
│   │
│   ├── Core/                    # Core Layer
│   │   ├── NPCAI/
│   │   ├── LOS/
│   │   └── Environment/
│   │
│   ├── Features/               # Feature Layer
│   │   ├── GrittyTakedowns/
│   │   └── WeaponSystem/
│   │
│   ├── Meta/                   # Meta Layer
│   │   ├── SanityRage/
│   │   └── ClueJournal/
│   │
│   ├── World/                  # World Layer (代码放在此，实际逻辑归属 World Layer)
│   │   ├── Weather/
│   │   └── Lighting/
│   │
│   └── Presentation/           # Presentation Layer
│       ├── UI/
│       ├── DynamicPostProcessing/
│       └── Audio/
│
├── Infrastructure/              # 横向基础设施（不属于分层调用链）
│   ├── EventBus/
│   ├── SaveSystem/
│   ├── Network/
│   └── Addressables/
│
├── ThirdParty/                  # 第三方 SDK
│   ├── Wwise/
│   └── InputSystem/
│
└── Resources/                   # Unity 资源
```

> **注意**：Weather System 作为 World 层系统，影响 Core/Meta/Presentation 多层，其实现代码应放在 Core 层（Environment 子目录），但通过 Event Bus 与其他层通信。

### 依赖规则（必须遵守）

```
✅ 允许的依赖方向：
  Foundation → Core → Feature → Meta → Presentation
  ↑ 被所有层依赖（但不调用任何层）
  Infrastructure Layer（横向基础设施）
  - 所有层都可以引用 Infrastructure Layer 的服务
  - 但 Infrastructure Layer 本身不参与分层调用链

❌ 禁止的依赖方向：
  - 同层之间不能互调（Core 不能直接调 Core）
  - 下层不能调用上层（Core 不能调用 Feature）
  - 跨层跳级调用（如 Foundation 直接调用 Meta）应避免

🔄 例外（需架构师批准）：
  - Meta 层可调用 Presentation，例如：
    - 成就解锁时触发 UI 动画
    - 剧情触发时播放过场动画
  - Event Bus 全局可访问（订阅/发布无需考虑分层）
```

### 循环依赖检测规则

如果发现循环依赖（如 A→B→C→A），解决方案：

1. **提取公共部分到 Infrastructure**：如 A 和 B 都需要 X，把 X 提到 Infrastructure
2. **引入接口抽象**：A 调用 IB 接口，B 实现 IB，具体依赖通过依赖注入解耦
3. **重新划分层次**：检查是否层次划分不合理

---

## Alternatives Considered

### Alternative 1: 混沌架构（无分层）

- **描述**：所有系统平铺，无层次划分
- **优点**：
  - 简单直接
  - 灵活不受限
- **缺点**：
  - 系统依赖混乱
  - 无法并行开发
  - 难以测试
  - 循环依赖风险高
- **拒绝理由**：
  - 系统数量 15+，无分层无法管理

### Alternative 2: 严格六边形架构（Hexagonal Architecture）

- **描述**：所有系统通过 Port/Adapter 与外部交互
- **优点**：
  - 完全解耦
  - 高度可测试
  - 依赖方向严格控制
- **缺点**：
  - 过度设计
  - 概念复杂，团队学习成本高
  - 对于游戏系统，过度的架构抽象反而降低开发效率
- **拒绝理由**：
  - 团队规模不需要六边形架构的复杂度
  - 游戏系统的"端口"概念不如企业应用清晰

### Alternative 3: 仅有 3 层（基础/游戏/表现）

- **描述**：只分 3 层，更粗粒度
- **优点**：
  - 简单易理解
  - 决策快速
- **缺点**：
  - 粒度过粗
  - Gritty Takedowns（功能层）和 NPC AI（核心层）被混在一起
  - 无法清晰表达依赖顺序
- **拒绝理由**：
  - 6 层能更好表达系统的依赖顺序和职责划分
  - 已有 `systems-index.md` 定义了 6 层分类，直接沿用

---

## Consequences

### Positive

- **依赖清晰**：每个系统知道自己该依赖谁，不该依赖谁
- **并行开发**：可以同时开发不同层的系统
- **独立测试**：可以单独测试某个系统
- **代码定位**：文件夹结构即分层，快速定位代码
- **新人友好**：新成员能快速理解系统边界

### Negative

- **约束严格**：开发时需要时刻注意依赖方向
- **分层开销**：有时候跨层调用一个小功能需要绕路
- **接口抽象**：解耦需要引入接口，增加代码量

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **违反分层** | 开发者为图方便跨层调用 | Code Review 检查依赖；CI 自动化检测循环依赖 |
| **层次过深** | 系统跨越多层调用 | 尽量保持2层以内调用链 |
| **接口膨胀** | 每个调用都创建接口 | 接口仅在有多于1个实现时才创建 |

---

## Performance Implications

| 指标 | 影响 | 说明 |
|------|------|------|
| **CPU** | 无直接影响 | 分层是架构概念，不影响运行时性能 |
| **Memory** | 无直接影响 | 分层不引入额外对象 |
| **Load Time** | 无直接影响 | 分层不增加加载时间 |
| **Network** | 无影响 | 不涉及网络 |

---

## Migration Plan

### Phase 1: 建立文件夹结构
- [ ] 在 `Assets/Game/` 下建立 6 层文件夹结构（Foundation/Core/Features/Meta/Presentation）
- [ ] 在 `Assets/Infrastructure/` 下建立横向基础设施文件夹
- [ ] 将 Weather System 实现代码放入 Core/Environment 目录

### Phase 2: 依赖规则文档化
- [ ] 在 `systems-index.md` 中明确定义 6 层和依赖关系
- [ ] 创建依赖图可视化

### Phase 3: 工具支持
- [ ] 引入 Unity 依赖分析工具（如 Dependency Graph）
- [ ] CI 阶段检测循环依赖

### Phase 4: 代码迁移
- [ ] 审计现有代码的依赖关系
- [ ] 重构违反分层规则的代码

---

## Validation Criteria

1. **文件夹检查**：`Assets/Game/` 下有且仅有 5 个分层文件夹
2. **依赖图检查**：使用工具生成依赖图，确认无循环依赖
3. **GDD 一致性**：每个系统的 GDD 在 Dependencies 章节正确标注依赖层级
4. **Code Review**：所有 PR 检查依赖方向

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 跨层通信通过 Event Bus 实现
- [ADR-0002: Unity 引擎技术选型](./adr-0002-unity-engine-selection.md) — 项目结构遵循 Unity 最佳实践
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC AI 属于 Core Layer，依赖 World Layer 的天气/光照系统
- [ADR-0005: 存档/持久化架构](./adr-0005-save-persistence-architecture.md) — 存档系统属于 Infrastructure Layer
- [ADR-0006: 网络同步架构](./adr-0006-network-synchronization-architecture.md) — 网络系统属于 Infrastructure Layer
- [系统索引文档](../../design/gdd/systems-index.md) — 所有系统的分层归属定义
