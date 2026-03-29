# Systems Index: 断绝：罪恶之源 (Severance: Source of Sin)

> **Status**: Approved
> **Created**: 2026-03-29
> **Last Updated**: 2026-03-29
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

《断绝：罪恶之源》的核心机制围绕“线索驱动的动态潜行”和“沉重、不洁的暴力”展开。因此，系统架构强依赖于稳健的“视野与监听”逻辑、深度的“NPC AI行为”，以及支撑起这一切的极简但高致死率的底层规则。作为一款俯视角游戏，它的系统重心不在于复杂的物理模拟，而在于精准的状态管理（玩家状态、NPC警戒状态、线索状态、理智值状态）和强烈的情绪视听反馈。

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | 玩家控制器 (Player Controller) (inferred) | Core | MVP | Designed | design/gdd/player-controller.md | — |
| 2 | 脆弱度与伤害系统 (Health & Lethality) (inferred) | Core | MVP | Designed | design/gdd/health-lethality.md | — |
| 3 | 关卡与存档系统 (Progression & Save) (inferred) | Persistence | Vertical Slice | Not Started | — | — |
| 4 | 视野与监听系统 (LOS & Eavesdropping) | Gameplay | MVP | Designed | design/gdd/los-eavesdropping.md | Player Controller |
| 5 | 环境交互系统 (Environment Interaction) | Gameplay | MVP | Not Started | — | Player Controller |
| 6 | NPC AI系统 (NPC AI System) | Gameplay | MVP | Not Started | — | LOS & Eavesdropping, Health & Lethality |
| 7 | 线索与日志系统 (Clue & Journal) (inferred) | Narrative | MVP | Not Started | — | Environment Interaction, NPC AI System |
| 8 | 沉重处决系统 (Gritty Takedowns) | Gameplay | MVP | Not Started | — | Player Controller, NPC AI System, Environment Interaction |
| 9 | 理智/愤怒系统 (Sanity/Rage Meter) | Meta | Vertical Slice | Not Started | — | Gritty Takedowns, Clue & Journal |
| 10| 沉浸式音频与震动 (Immersive Audio & Haptics) (inferred) | Audio | Vertical Slice | Not Started | — | Gritty Takedowns, Environment Interaction |
| 11| 动态视觉滤镜系统 (Dynamic Post-Processing) (inferred) | Presentation | Alpha | Not Started | — | Sanity/Rage Meter |

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
2. **脆弱度与伤害系统 (Health & Lethality)** — 确立“一击必杀”的核心底层规则，AI和处决都需要调用它。
3. **关卡与存档系统 (Progression & Save)** — 管理游戏场景的生命周期，脱离具体玩法独立存在。

### Core Layer (depends on foundation)

1. **视野与监听系统 (LOS & Eavesdropping)** — depends on: 玩家控制器
2. **环境交互系统 (Environment Interaction)** — depends on: 玩家控制器
3. **NPC AI系统 (NPC AI System)** — depends on: 视野与监听系统, 脆弱度与伤害系统

### Feature Layer (depends on core)

1. **线索与日志系统 (Clue & Journal)** — depends on: 环境交互系统, NPC AI系统
2. **沉重处决系统 (Gritty Takedowns)** — depends on: 玩家控制器, NPC AI系统, 环境交互系统

### Meta/Narrative Layer (depends on features)

1. **理智/愤怒系统 (Sanity/Rage Meter)** — depends on: 沉重处决系统, 线索与日志系统 (玩家的行为反馈于此)

### Presentation Layer (depends on meta/features)

1. **沉浸式音频与震动 (Immersive Audio & Haptics)** — depends on: 沉重处决系统, 环境交互系统等 (行为的物理反馈)
2. **动态视觉滤镜系统 (Dynamic Post-Processing)** — depends on: 理智/愤怒系统 (心理状态的视觉外显)

---

## Recommended Design Order

| Order | System | Priority | Layer | Agent(s) | Est. Effort |
|-------|--------|----------|-------|----------|-------------|
| 1 | 玩家控制器 (Player Controller) | MVP | Foundation | gameplay-programmer | S |
| 2 | 脆弱度与伤害系统 (Health & Lethality) | MVP | Foundation | systems-designer | S |
| 3 | 视野与监听系统 (LOS & Eavesdropping) | MVP | Core | systems-designer | M |
| 4 | 环境交互系统 (Environment Interaction) | MVP | Core | gameplay-programmer | S |
| 5 | NPC AI系统 (NPC AI System) | MVP | Core | ai-programmer | L |
| 6 | 沉重处决系统 (Gritty Takedowns) | MVP | Feature | technical-artist / game-designer | M |
| 7 | 线索与日志系统 (Clue & Journal) | MVP | Feature | narrative-director | M |
| 8 | 关卡与存档系统 (Progression & Save) | Vertical Slice| Foundation | tools-programmer | M |
| 9 | 理智/愤怒系统 (Sanity/Rage Meter) | Vertical Slice| Meta | systems-designer | M |
| 10| 沉浸式音频与震动 (Immersive Audio & Haptics) | Vertical Slice| Presentation| sound-designer | S |
| 11| 动态视觉滤镜系统 (Dynamic Post-Processing) | Alpha | Presentation| technical-artist | S |

---

## Circular Dependencies

- [None found]

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| **NPC AI系统** | Design / Technical | 俯视角下的潜行AI很容易显得“过傻”（看不到旁边的人）或“过强”（千里眼）。同时需要处理多状态切换（巡逻/求饶/逃跑），状态机/行为树容易臃肿。 | 在设计GDD时，严格定义视锥体参数和状态转移条件；在开始铺设大量内容前，优先运行 `/prototype` 制作包含三种状态的基础AI测试用例。 |
| **沉重处决系统** | Scope / Technical | 强调“环境即武器”，需要制作大量依赖环境上下文的特定动画，如果每个物体都做独立动画，工作量会爆炸。 | 在系统设计阶段建立“通用标签系统”（如重物、长柄、投掷物），将动画进行模块化复用，限制必须制作特殊动画的物件数量。 |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 11 |
| Design docs started | 0 |
| Design docs reviewed | 0 |
| Design docs approved | 0 |
| MVP systems designed | 0/7 |
| Vertical Slice systems designed | 0/3 |

---

## Next Steps

- [ ] Design MVP-tier systems first (use `/design-system [system-name]`)
- [ ] Prototype the highest-risk system early (`/prototype NPC AI系统` or `沉重处决系统`)
- [ ] Run `/sprint-plan new` to organize the first development sprint
