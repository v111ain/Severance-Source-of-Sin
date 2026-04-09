# Systems Index: 断绝：罪恶之源 (Severance: Source of Sin)

> **Status**: Approved
> **Created**: 2026-03-29
> **Last Updated**: 2026-04-09
> **Source Concept**: design/gdd/game-concept.md
> **Engine**: Unity 6.3 LTS (PC & PS5)
> **Revision Notes**: 2026-04-07 批量修复已发现的设计问题：
> - 引擎统一为 Unity 6.3 LTS（修复 Godot 引用冲突）
> - 玩家控制器（射线检测职责划分、IsLocked所有权澄清）
> - LOS与监听（AimAccuracy计算方法、专注对准时间阈值、感知职责划分）
> - 沉重处决（ESCAPE状态行为澄清、AlertStateChanged事件说明、DialogueTree数据结构定义、PlayerIntelligenceBonus移除）
> - 理智/愤怒（双轨并行模型澄清、状态优先级定义）
> - 线索系统（关键词→线索匹配机制详细定义）
> - 存档系统（撤离判定逻辑与战斗暂停逻辑统一）
> - 脆弱度系统（穿透伤害公式定义）
> - 音频系统（PlayerDamagedEvent广播职责澄清）
> - 所有 Godot 引用已替换为 Unity
> - game-concept.md 补充验收标准（Formulas/Edge Cases/Dependencies/Tuning Knobs/Acceptance Criteria）
>
> **2026-04-07 设计审查修复**：
> - ✅ P0-1: LOS系统 `SoundSourceUIPosition` → `SoundSourceScreenPosition`，补充3D投影计算说明
> - ✅ P0-2: 线索系统 `ContextMultiplier` → `ClueContextMultiplier`，避免与理智系统命名冲突
> - ✅ P0-3: Gritty Takedowns `KillTagEvent{npc_tag, is_mistake}` → `KillTagEvent{kill_tag: NPCIdentityType}`
> - ✅ P1-4: NPC AI系统 锁喉机制明确为 Alpha 范围，MVP 简化方案已定义
> - ✅ P1-5: 理智系统 悲剧线索惩罚曲线重新评估（递减机制）
> - ✅ P1-6: 环境交互系统 新增物件动画标签体系定义
> - ✅ P2-7: 理智系统 愤怒消散速率调整（5秒→3秒，-1→-2），新增狂暴阈值锁定机制
> - ✅ P2-8: 动态视觉滤镜系统 创建设计文档框架（draft）
>
> **2026-04-08 设计审查修复**：
> - ✅ P0-*: Screen Effects 系统确立为独立 Infrastructure 系统
> - ✅ P0-*: DPP 状态映射表删除，改为直接引用 Sanity/Rage 系统状态枚举
> - ✅ P1-*: 线索系统理智惩罚计算统一为 NarrativeSignificanceMultiplier 模式
> - ✅ P1-*: NPC AI 系统 OQ-1 派系体系简化方案确立（5派系：凋亡议会、锈网、灰烬团、无声者、苍白之手）

**2026-04-08 系统重构**：
> - ⚠️ **关卡与存档系统 (Progression & Save)** → **已被替代**（由世界地图系统替代）
> - ✅ 新增「世界地图与非线性叙事系统 (World Map & Non-Linear Progression)」：采用国家→城市→地区三层结构，支持非线性探索

**2026-04-09 设计审查修复**：
> - ✅ 武器系统 (Weapon System) 全部 P0/P1/P2 问题已修复，状态更新为 Approved：
>   - P0: 修正爆炸伤害公式示例计算错误
>   - P1: 澄清 DESTRUCT_GLASS 标签语义一致性
>   - P1: 补充爆炸物音效倒计时详细规格
>   - P2: 补充 Haptic Feedback 实现规格

**2026-04-09 系统补充**：
> - ✅ UI 系统 (UI System) 已纳入索引：作为 Presentation 层 MVP 系统，负责 HUD、菜单、Alert Layer 和输入屏蔽

**2026-04-10 设计审查修复**：
> - ✅ P0: DPP (Dynamic Post-Processing) 状态阈值直接引用 sanity-rage-meter.md 公式4，避免重复定义
> - ✅ P0: LOS/Eavesdropping 系统补充 MaxScreenDistance Tuning Knob 定义
> - ✅ P1: Health & Lethality 穿透公式统一使用 `>=` 比较，爆炸伤害边界使用 `<`
> - ✅ P1: 武器系统 OQ-1 库存问题已解决（固定栏位方案）
> - ✅ P1: 沉重处决系统 OQ-2 转化线人超时机制已定义
> - ✅ P1: UI 系统 OQ-1 小地图问题已解决（不需要小地图）
> - ✅ P1: DialogTree 接口 npc_id 类型统一为 string，Alert State 枚举引用 npc-ai-system.md

---

## Overview

《断绝：罪恶之源》的核心机制围绕“线索驱动的动态潜行”和“沉重、不洁的暴力”展开。因此，系统架构强依赖于稳健的“视野与监听”逻辑、深度的“NPC AI行为”，以及支撑起这一切的极简但高致死率的底层规则。作为一款俯视角游戏，它的系统重心不在于复杂的物理模拟，而在于精准的状态管理（玩家状态、NPC警戒状态、线索状态、理智值状态）和强烈的情绪视听反馈。

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | 玩家控制器 (Player Controller) | Core | MVP | Approved | design/gdd/player-controller.md | — |
| 2 | 脆弱度与伤害系统 (Health & Lethality) | Core | MVP | Approved | design/gdd/health-lethality.md | — |
| 3 | ~~关卡与存档系统 (Progression & Save)~~ | ~~Persistence~~ | ~~Vertical Slice~~ | ~~Superseded~~ | ~~design/gdd/progression-save.md~~ | ~~—~~ |
| 4 | **世界地图与非线性叙事系统 (World Map & Non-Linear Progression)** | **Navigation** | **MVP** | **Designed** | **design/gdd/world-map-progression.md** | **—** |
| 5 | 视野与监听系统 (LOS & Eavesdropping) | Gameplay | MVP | Approved | design/gdd/los-eavesdropping.md | Player Controller |
| 6 | 环境交互系统 (Environment Interaction) | Gameplay | MVP | Approved | design/gdd/environment-interaction.md | Player Controller |
| 7 | NPC AI系统 (NPC AI System) | Gameplay | MVP | Approved | design/gdd/npc-ai-system.md | LOS & Eavesdropping, Health & Lethality |
| 8 | 线索与日志系统 (Clue & Journal) | Narrative | MVP | Approved | design/gdd/clue-and-journal.md | Environment Interaction, NPC AI System |
| 9 | **武器系统 (Weapon System)** | **Gameplay** | **MVP** | Approved | design/gdd/weapon-system.md | Player Controller, Environment Interaction |
| 10 | 沉重处决系统 (Gritty Takedowns) | Gameplay | MVP | Approved | design/gdd/gritty-takedowns.md | Player Controller, NPC AI System, **Weapon System** |
| 11| 理智/愤怒系统 (Sanity/Rage Meter) | Meta | Vertical Slice | Approved | design/gdd/sanity-rage-meter.md | Gritty Takedowns, Clue & Journal |
| 12| 沉浸式音频与震动 (Immersive Audio & Haptics) | Audio | Vertical Slice | Approved | design/gdd/immersive-audio-haptics.md | Gritty Takedowns, Environment Interaction |
| 13| 动态视觉滤镜系统 (Dynamic Post-Processing) | Presentation | Alpha | Approved | design/gdd/dynamic-post-processing.md | Sanity/Rage Meter, Screen Effects |
| 14| UI 系统 (UI System) | Presentation | MVP | Approved | design/gdd/ui-system.md | — |
| — | **屏幕特效系统 (Screen Effects)** | **Infrastructure** | **Infrastructure** | **Approved** | design/gdd/screen-effects.md | — |
| — | **DialogTree 接口协议** | **Interface** | **MVP** | **Approved** | design/gdd/dialog-tree-interface.md | Gritty Takedowns, NPC AI System |

---

## Categories

| Category | Description | Typical Systems |
|----------|-------------|-----------------|
| **Core** | Foundation systems everything depends on | Player controller, input, physics, camera, scene management, state machine |
| **Gameplay** | The systems that make the game fun | Combat, AI, stealth, movement abilities, interaction |
| **Persistence** | Save state and continuity | Save/load, settings, cloud sync, profile management |
| **Narrative** | Story and dialogue delivery | Dialogue system, quest tracking, cutscenes, journal, lore entries |
| **Meta** | Systems outside the core game loop | Analytics, tutorials/onboarding, accessibility options, photo mode |
| **Presentation**| Player-facing visual feedback displays | Dynamic Post-Processing, HUD, menus, UI |
| **Audio** | Sound and music systems | Music manager, SFX bus, ambient audio, adaptive music, voice |

---

## Priority Tiers

| Tier | Definition | Target Milestone | Design Urgency |
|------|------------|------------------|----------------|
| **MVP** | Required for the core loop to function. Without these, you can't test "is this fun?" | First playable prototype | Design FIRST |
| **Vertical Slice** | Required for one complete, polished area. Demonstrates the full experience. | Vertical slice / demo | Design SECOND |
| **Alpha** | All features present in rough form. Complete mechanical scope, placeholder content OK. | Alpha milestone | Design THIRD |
| **Full Vision** | Polish, edge cases, nice-to-haves, and content-complete features. | Beta / Release | Design as needed |

---

## Dependency Map

### Foundation Layer (no dependencies)

1. **玩家控制器 (Player Controller)** — 处理玩家输入和基础状态，是一切交互的根基。
2. **脆弱度与伤害系统 (Health & Lethality)** — 确立”一击必杀”的核心底层规则，AI和处决都需要调用它。
3. **世界地图与非线性叙事系统 (World Map & Non-Linear Progression)** — 管理游戏世界的导航层级结构，替代原关卡系统，支持非线性探索。

### Core Layer (depends on foundation)

1. **视野与监听系统 (LOS & Eavesdropping)** — depends on: 玩家控制器
2. **环境交互系统 (Environment Interaction)** — depends on: 玩家控制器
3. **NPC AI系统 (NPC AI System)** — depends on: 视野与监听系统, 脆弱度与伤害系统
4. **武器系统 (Weapon System)** — depends on: 玩家控制器, 环境交互系统

### Feature Layer (depends on core)

1. **线索与日志系统 (Clue & Journal)** — depends on: 环境交互系统, NPC AI系统
2. **沉重处决系统 (Gritty Takedowns)** — depends on: 玩家控制器, NPC AI系统, **武器系统**, 环境交互系统

### Meta/Narrative Layer (depends on features)

1. **理智/愤怒系统 (Sanity/Rage Meter)** — depends on: 沉重处决系统, 线索与日志系统 (玩家的行为反馈于此)

### Presentation Layer (depends on meta/features)

1. **沉浸式音频与震动 (Immersive Audio & Haptics)** — depends on: 沉重处决系统, 环境交互系统等 (行为的物理反馈)
2. **动态视觉滤镜系统 (Dynamic Post-Processing)** — depends on: 理智/愤怒系统 (心理状态的视觉外显)
3. **UI 系统 (UI System)** — depends on: 所有游戏系统（数据消费者，HUD/菜单/Alert Layer/输入屏蔽）

---

## Recommended Design Order

| Order | System | Priority | Layer | Agent(s) | Est. Effort |
|-------|--------|----------|-------|----------|-------------|
| 1 | 玩家控制器 (Player Controller) | MVP | Foundation | gameplay-programmer | S |
| 2 | 脆弱度与伤害系统 (Health & Lethality) | MVP | Foundation | systems-designer | S |
| 3 | **~~关卡与存档系统~~** | ~~Vertical Slice~~ | ~~Foundation~~ | ~~tools-programmer~~ | ~~M~~ |
| 4 | **世界地图与非线性叙事系统 (World Map & Non-Linear Progression)** | **MVP** | **Foundation** | **game-designer** | **M** |
| 5 | 视野与监听系统 (LOS & Eavesdropping) | MVP | Core | systems-designer | M |
| 6 | 环境交互系统 (Environment Interaction) | MVP | Core | gameplay-programmer | S |
| 7 | NPC AI系统 (NPC AI System) | MVP | Core | ai-programmer | L |
| 8 | **武器系统 (Weapon System)** | **MVP** | **Core** | **game-designer / systems-designer** | **M** |
| 9 | 沉重处决系统 (Gritty Takedowns) | MVP | Feature | technical-artist / game-designer | M |
| 10 | 线索与日志系统 (Clue & Journal) | MVP | Feature | narrative-director | M |
| 11 | 理智/愤怒系统 (Sanity/Rage Meter) | Vertical Slice| Meta | systems-designer | M |
| 12| 沉浸式音频与震动 (Immersive Audio & Haptics) | Vertical Slice| Presentation| sound-designer | S |
| 13| 动态视觉滤镜系统 (Dynamic Post-Processing) | Alpha | Presentation| technical-artist | S |
| 14| UI 系统 (UI System) | MVP | Presentation| ui-programmer / ux-designer | M |

---

## Circular Dependencies

- [None found]

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| **NPC AI系统** | Design / Technical | 俯视角下的潜行AI很容易显得”过傻”（看不到旁边的人）或”过强”（千里眼）。同时需要处理多状态切换（巡逻/求饶/逃跑），状态机/行为树容易臃肿。 | 在设计GDD时，严格定义视锥体参数和状态转移条件；在开始铺设大量内容前，优先运行 `/prototype` 制作包含三种状态的基础AI测试用例。 |
| **沉重处决系统** | Scope / Technical | 强调”环境即武器”，需要制作大量依赖环境上下文的特定动画，如果每个物体都做独立动画，工作量会爆炸。 | ✅ **已解决**：武器系统统一管理”通用标签系统”（重物、长柄、投掷物），动画模块化复用。 |
| **武器系统（爆炸物设计）** | Balance | 爆炸物混合型设计需要精确调参：lethal_radius_ratio 过大导致秒杀范围过大，过小导致控制效果不足。 | 在 Vertical Slice 阶段专项测试爆炸物平衡性，按 OQ-6 设计 C4 任务关卡时验证。 |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 14 (含1个已替代系统) |
| Active design docs | 14 |
| Design docs reviewed | 12 |
| Design docs with P0 issues fixed | 12 |
| Design docs in revision | 0 |
| Design docs approved | 12 (活跃系统) + 1 superseded |
| Design docs in review | 0 |
| Design docs with P1/P2 improvements | 6 (LOS/线索/理智/环境交互/NPC AI/DPP) |
| MVP systems (10 total) | 10/10 (全部 Approved) |
| Vertical Slice systems (3 total) | 3/3 (all Approved) |
| Alpha systems (1 total) | 1/1 (DPP ✅ Approved) |

---

## Next Steps

- [x] ✅ 引擎统一为 Unity 6.3 LTS
- [x] ✅ 修复所有 Godot 引用为 Unity
- [x] ✅ 解决 Gritty Takedowns 的 DialogueTree 数据结构定义
- [x] ✅ 解决 Clue & Journal 的关键词匹配机制
- [x] ✅ 解决 progression-save 撤离判定逻辑冲突
- [x] ✅ 解决 sanity-rage-meter 双轨并行模型澄清
- [x] ✅ 移除 gritty-takedowns 的 PlayerIntelligenceBonus
- [x] ✅ 定义 health-lethality 穿透伤害公式
- [x] ✅ 澄清 immersive-audio PlayerDamagedEvent 广播职责
- [x] ✅ 补充 game-concept.md 验收标准
- [x] ✅ P0/P1/P2 设计审查问题修复（2026-04-07）
- [x] ✅ 动态视觉滤镜系统设计框架启动
- [ ] Run `/gate-check pre-production` to check if you're ready to start building
- [ ] Prototype the highest-risk system (`/prototype NPC AI系统`)
- [ ] 完善动态视觉滤镜系统设计（确定渲染方案）
- [ ] UI 系统 OQ-1 交叉问题：LOS 系统是否需要提供小地图数据？（待 LOS 系统解答）

**2026-04-09 更新**：
- ✅ UI 系统已纳入索引（#14）
- ✅ 动态视觉滤镜系统 (DPP) 补充 Tuning Knobs 章节，状态更新为 Approved
