# 沉重处决系统 (Gritty Takedowns)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-04-10
> **Implements Pillar**: 沉重、不洁的暴力 (Gritty, Dirty Violence)、环境即武器 (Environment as a Weapon)
> **Revision Notes**: 2026-04-07 修复状态标记为 In Review；澄清公式5中 Alert State < ALERT 的具体含义（指 UNDETECTED/SUSPECT/SEARCH 三态）
> **2026-04-08 协同修订（配合武器系统修复设计审查问题）**：
> - P1: `DamageRequest` 接口添加 `penetration` 参数，说明来自武器数据
> - P2: 新增 `WeaponQueryRequest`/`WeaponQueryResponse` 事件接口
> - P2: 明确 `source` 参数来源于 `WeaponQueryResponse` 的 `weapon_id`
> **2026-04-10 战斗团队评审修复**：
> - P0: 公式3 捆绑时间变量 `NPCSizeMultiplier` 和 `PlayerSkillBonus` 完整定义表
> - P1: `StaggerRecoveryThreshold` 安全范围补全（0.3 ~ 0.7）
> - P1: `DeceptionBaseChance` 公式与 Tuning Knobs 关联明确化
> - P2: `InteractionRange_Search` 默认值与安全范围统一
> **2026-04-10 配合主角背景系统设计审查修复**：
> - P0: 新增 `HasMetFaction[FACTION_ID]` 数据结构到下游依赖和本系统持有数据表
> - P0: 新增 `KillSource` 和 `NPCIdentityType` 枚举定义到事件接口
> - P1: 更新 `KillTagEvent` 负载同时携带 `NPCIdentityType` 和 `KillSource`

## Overview

玩家-NPC 交互系统 (Player-NPC Interaction System) 是管理玩家主动发起、与 NPC 之间所有交互行为的中心系统。它整合了处决、威胁、审讯、转化等玩家对 NPC 的主动动作，是"线索驱动的动态潜行"核心体验的操作层实现。

系统通过"交互类型矩阵"确定可用选项：由 NPC 状态（FREE/UNCONSCIOUS/TIED/DEAD）和玩家获取的信息（身份标签）共同决定玩家可以对当前目标执行哪些交互。

**可用交互对照表**：

| 交互类型 | NPC 状态要求 | 信息要求 | 效果 |
|---------|-------------|---------|------|
| 潜行击杀 | FREE（背面） | 已标记为恶徒 | 即时死亡 |
| 环境处决 | FREE/Staggered/Downed | 已标记为恶徒 | 即时死亡 + 环境效果 |
| 补刀 | UNCONSCIOUS/TIED | 无要求 | 即时死亡 |
| 威胁 | FREE | 无要求 | allegiance 变化 |
| 搜身 | UNCONSCIOUS/DEAD | 无要求 | 获取线索 |
| 审问 | UNCONSCIOUS | 无要求 | 获取情报 |
| 转化 | FREE | 已获取 vulnerability | allegiance 大幅提升 |

## Player Fantasy

**"在黑暗中编织死亡的剧本。"**

玩家在游戏中面对 NPC 时，不是等待系统给予选项，而是通过收集信息（窃听、观察）、制造条件（引诱、分散注意力）、选择手段（威胁、击杀、转化）主动塑造局面。从"信息不对称的猎物"转变为"掌握真相的处刑人"。

核心情感体验是**真相大白时的裁决快感**：
- *恐惧*：我还不知道谁是敌人——每一个阴影背后都可能藏着致命威胁
- *犹豫*：面对一个"求饶的帮凶"，杀还是不杀？
- *策划*：我已经知道他是帮凶——用砖块砸死还是电线勒颈？
- *掌控*：通过威胁和审讯，我让这个 NPC 成了我的线人
- *发现*：通过碎片化的对话拼凑出完整的真相——原来他就是关键人物
- *后悔*：杀错了——他是无辜者，理智值崩溃

参考《Hotline Miami》的致命后果感 + 《The Last of Us》的叙事代入感。

## Detailed Design

### Core Rules

**交互类型矩阵**：

| 类别 | 交互类型 | NPC 状态要求 | 身份标签要求 | 情感对应 |
|------|---------|-------------|-------------|---------|
| **处决类** | 潜行击杀 | FREE（背面） | 已标记恶徒 | 策划、掌控 |
| | 环境处决 | FREE/Staggered/Downed | 已标记恶徒 | 策划、后悔(杀错) |
| | 补刀 | UNCONSCIOUS/TIED | 无 | 掌控 |
| **强制类** | 威胁 | FREE | 无 | 恐惧、掌控 |
| | 贿赂 | FREE | 无 | 掌控 |
| | 欺骗 | FREE | 已知 vulnerability | 掌控 |
| **对峙类** | 保持距离威胁 | FREE | 无 | 恐惧 |
| | 对话选项 | FREE | 无 | 犹豫 |
| **情报类** | 搜身 | UNCONSCIOUS/DEAD | 无 | 发现 |
| | 审问 | UNCONSCIOUS | 无 | 发现 |
| **控制类** | 捆绑 | UNCONSCIOUS | 无 | 掌控 |
| | 解开 | TIED | 无 | — |
| **转化类** | 转化线人 | FREE | 已获取 vulnerability | 掌控 |

**关键交互的完整触发条件**：

1. **潜行击杀**：
   - 玩家必须处于潜行状态（Crouch）
   - NPC 必须处于 FREE 状态且 Alert State < ALERT
   - 玩家必须在 NPC 背面（150° 扇形盲区）
   - 玩家必须已通过 LOS 监听确认 NPC 身份为"恶徒"
   - 执行后：即时 Lethal 伤害，无警报扩散

2. **环境处决**：
   - 玩家必须持有可用的环境物件（砖块/钢管/灭火器等）
   - NPC 状态必须为 FREE/Staggered/Downed 任一
   - 玩家必须已确认 NPC 身份为"恶徒"
   - 执行后：即时 Lethal 伤害 + 环境特效

3. **捆绑 (Tie Up)**：
   - 玩家必须在 NPC 交互范围内（1.5m）
   - NPC 必须处于 UNCONSCIOUS 状态
   - 执行后：NPC 转入 TIED 状态，Alert State 冻结并降一级
   - 消耗时间：3-5 秒

4. **解开 (Release)**：
   - 玩家必须在 NPC 交互范围内
   - NPC 必须处于 TIED 状态
   - 执行后：NPC 苏醒，Alert State 降一级

5. **威胁/贿赂/欺骗**：
   - 交互范围：1.5m
   - **欺骗**前置：玩家必须已通过搜身获取 NPC 的 vulnerability

6. **搜身/审问**：
   - 搜身：UNCONSCIOUS/DEAD 状态
   - 审问：仅限 UNCONSCIOUS 状态（DEAD 不可审问）
   - 审问可获取 NPC 的 knowledge（线索列表）

7. **转化线人**：
   - 前置：已获取 NPC vulnerability
   - 类型：临时线人（单次情报）
   - 执行后：allegiance 大幅提升，提供一次情报后恢复原状
   - **超时机制**：转化效果持续 `ConversionEffectDuration`（默认 30 秒），超时后 allegiance 自动恢复，无需等待情报提供

8. **对话选项**：
   - 与 NPC AI 系统的 CONFRONTATION 机制共用
   - 玩家-NPC 交互系统负责发起，NPC AI 系统负责响应分支

### States and Transitions

**玩家交互状态机**：

```
[Idle] ──检测到可交互 NPC──▶ [CanInteract] ──按下交互键──▶ [Interacting]
      ▲                              │                            │
      │                              │                    ┌─────┴─────┐
      │                              │                    ▼           ▼
      │                              │              [选择分支]     [直接执行]
      │                              │                    │           │
      │                              │                    ▼           ▼
      │                              │             [展示选项UI]   [执行结果]
      │                              │                    │           │
      │                              │                    ▼           │
      │                              │              [等待选择]         │
      │                              │                    │           │
      │                              │              [执行结果]◀───────┘
      │                              │                    │
      │                              │                    ▼
      │                              │           [结果分发]
      │                              │                    │
      │                              │     ┌──────────────┼──────────────┐
      │                              │     ▼              ▼              ▼
      │                              │  [返回 Idle]  [锁定/动画]  [被打断]
      │                              │                              │
      │                              │                    ┌──────────┴──────────┐
      │                              │                    ▼                     ▼
      │                              │              [返回 Idle]         [强制中断-死亡]
      │                              │                                          
      └──────────（离开范围）────────┘
```

**状态说明**：

| 状态 | 描述 | 可转移至 |
|------|------|---------|
| Idle | 玩家未与 NPC 交互 | CanInteract（检测到可交互目标） |
| CanInteract | 玩家视野中存在可交互 NPC | Idle（离开范围）、Interacting（按下交互键） |
| Interacting | 玩家执行交互，动画播放中，动作锁定 | 结果分发、Interrupted（被攻击打断） |
| Interrupted | 交互被打断（受到攻击） | 返回 Idle（继续）、死亡（致命伤） |

**与 NPC AI 系统 CONFRONTATION 的对接**：
- 对峙请求 → NPC AI 系统处理 → 返回分支类型（无视/观察/试探/回避）
- 试探分支需要玩家选择对话选项

### Interactions with Other Systems

**数据流入 (Inputs)**：

| 来源系统 | 数据内容 | 用途 |
|---------|---------|------|
| 玩家控制器 | 玩家位置、状态（潜行/站立）、持有物、IsLocked | 判定交互范围和可用选项 |
| NPC AI 系统 | `QueryState(NPC_ID)` - NPC 状态 | 判定可用交互类型 |
| NPC AI 系统 | `QueryAlertState(NPC_ID)` - Alert State | Alert > ALERT 时部分交互不可用 |
| NPC AI 系统 | allegiance, vulnerability, knowledge | 判定交互效果 |
| NPC AI 系统 | Bravery, Courage 属性 | 威胁/欺骗的成功率判定 |
| LOS & Eavesdropping | 身份标签（恶徒/帮凶/受害者/未知） | 判定是否可以安全处决 |
| 环境交互系统 | 可用环境物件列表、持有物状态 | 环境处决选项 |
| **武器系统 (Weapon System)** | `WeaponQueryResponse(weapon_id, WeaponData)` | **环境处决前查询武器数据**；本系统发送 `WeaponQueryRequest` 查询物件对应的 WeaponData；返回的 `weapon_id` 作为 `DamageRequest` 的 `source` 参数 |

**数据流出 (Outputs)**：

| 目标系统 | 数据内容 | 触发条件 |
|---------|---------|---------|
| NPC AI 系统 | 交互事件（威胁/击杀/捆绑等） | 交互执行后 |
| NPC AI 系统 | `ExecutionWitnessed(npc_id, witness_npc_id)` | 处决被第三方目击 |
| NPC AI 系统 | CONFRONTATION 请求 | 对峙类交互 |
| Health & Lethality | 伤害请求（Lethal/Blunt） | 处决类交互 |
| Clue & Journal | 获取的 knowledge 列表 | 搜身/审问后 |
| Sanity/Rage System | 击杀的 NPC Tag | 处决后（异步） |

## Formulas

### 公式1：威胁/贿赂/欺骗后的 allegiance 变化

NPC AI 系统已定义基础变化表，本系统直接引用：

| 交互类型 | BaseChange | 说明 |
|---------|-----------|------|
| 威胁 | -20 | 立即变化 |
| 贿赂 | +10 | 消耗资源 |
| 欺骗成功 | +10 | 利用已知弱点 |
| 心理操纵成功 | +20 | 利用已知弱点 |
| 欺骗失败 | -5 | NPC 感到被轻视 |

最终变化量：`AllegianceDelta = BaseChange * InteractionTypeMultiplier * ContextMultiplier * RelationshipMultiplier`
- InteractionTypeMultiplier：威胁=-1.0, 贿赂=1.0, 欺骗=0.8, 心理操纵=1.2
- ContextMultiplier：UNDETECTED=1.0, SUSPECT=1.2, SEARCH=1.5
- RelationshipMultiplier：派系敌对=0.8, 中立=1.0, 友好=1.2

### 公式2：欺骗成功判定

`DeceptionSuccess = (HasVulnerability == true) AND (Random() < DeceptionBaseChance)`

| 变量 | 定义 | 默认值 |
|------|------|--------|
| HasVulnerability | 玩家是否已获取 NPC 的 vulnerability | true/false |
| DeceptionBaseChance | 欺骗基础成功率 | 0.6 |

**设计说明**：欺骗成功仅取决于玩家是否已获取目标的 vulnerability。不引入额外的玩家属性加成，以保持机制简洁性和可预测性。若玩家获取了 vulnerability 但选择使用错误的弱点信息，则欺骗失败（视为普通威胁处理）。

### 公式3：捆绑时间消耗

`TieUpDuration = BaseTieUpTime * (1.0 + NPCSizeMultiplier - PlayerSkillBonus)`

| 变量 | 定义 | 数据来源 | 典型值 |
|------|------|---------|--------|
| BaseTieUpTime | 基础捆绑时间 | Tuning Knobs | 4.0 秒 |
| NPCSizeMultiplier | NPC 体型对捆绑时间的加成 | NPC AI - `EntityData.size_category` | 小型=0.0, 中型=0.25, 大型=0.5 |
| PlayerSkillBonus | 玩家技能对捆绑时间的减免 | 玩家技能系统 `PlayerSkillTree.tie_up_efficiency` | 基础=0.0, 进阶=0.15, 专家=0.3 |

**NPCSizeMultiplier 定义表**：

| NPC 体型 | NPCSizeMultiplier | 说明 |
|---------|------------------|------|
| 小型 (Small) | 0.0 | 捆得更快 |
| 中型 (Medium) | 0.25 | 标准 |
| 大型 (Large) | 0.5 | 体型大，捆绑耗时长 |

**PlayerSkillBonus 定义表**：

| 技能等级 | PlayerSkillBonus | 说明 |
|---------|-----------------|------|
| 基础 (Tier 1) | 0.0 | 无加成 |
| 进阶 (Tier 2) | 0.15 | 捆绑效率提升 |
| 专家 (Tier 3) | 0.3 | 最高效率加成 |

安全范围验证：
- 最短捆绑时间：`4.0 × (1.0 + 0.0 - 0.3) = 2.8s`（接近 3.0s 安全下限）
- 最长捆绑时间：`4.0 × (1.0 + 0.5 - 0.0) = 6.0s`（等于 6.0s 安全上限）

安全范围：3.0 秒 ~ 6.0 秒

### 公式4：转化线人的 allegiance 提升

`ConversionAllegianceGain = BaseConversionBoost * VulnerabilityDepthMultiplier`

| 变量 | 定义 | 默认值 |
|------|------|--------|
| BaseConversionBoost | 基础转化提升 | +30（与 NPC AI 系统公式1一致） |
| VulnerabilityDepthMultiplier | 弱点深度乘数（知道越多，提升越多） | 1.0 ~ 1.5 |

示例：
- 仅知道"母亲重病"：×1.0 = +30
- 知道"母亲重病 + 欠债 + 想逃跑"：×1.5 = +45

转化上限：`Clamp(Allegiance + ConversionAllegianceGain, -100, +100)`

### 公式5：潜行击杀的警报判定

`AlertBlocked = (IsStealthKill == true) AND (IsBehindTarget == true) AND (TargetAlertState < ALERT)`

**Alert State 层级说明**：根据 NPC AI 系统定义，Alert State 层级如下（从低到高）：
- UNDETECTED（未察觉）→ SUSPECT（怀疑）→ SEARCH（搜索）→ ALERT（警戒）→ ESCAPE（逃跑）→ COMBAT（战斗）

**公式含义**：只有当目标 NPC 处于 UNDETECTED/SUSPECT/SEARCH 三态之一时（即尚未确认威胁存在），潜行击杀才不会触发警报扩散。

**ESCAPE 状态特殊说明**：
- ESCAPE 状态表示 NPC 已确认威胁存在并正在逃离
- 从 ESCAPE 状态 NPC 的背面击杀**会触发警报扩散**（因为 NPC 已知道威胁存在）
- 从 COMBAT 状态 NPC 的背面击杀**会触发警报扩散**（同 ESCAPE）
- 这符合"威胁已经暴露"的设计意图——一旦 NPC 知道了你的存在，即使从背后偷袭，警报仍会扩散

只有当上述三个条件**全部满足**时，才不会触发警报扩散。

## Edge Cases

### 边缘情况1：交互期间 NPC 状态变化

**问题**：玩家正准备处决，NPC 突然从 FREE 转入 ALERT

**处理**：
- 交互不自动取消
- ALERT 状态下击杀会触发警报
- **反馈机制**：当背面条件失效时，NPC 播放"警觉转身"动画 + 警示音效，让玩家明确理解条件变化

### 边缘情况2：玩家在 Interacting 状态中被攻击

**问题**：玩家处于处决动画中，被其他 NPC 攻击

**处理**：
- 处决期间**无无敌帧**
- Lethal → 死亡，处决中断；Blunt → 硬直，处决中断
- **被处决 NPC 的存活判定**：
  - 动画进度 < 50%：处决无效，NPC 从 Staggered 恢复
  - 动画进度 ≥ 50%：处决成功，NPC 死亡（即使玩家同时死亡）

### 边缘情况3：同时面对多个 NPC

**问题**：玩家被多个 NPC 发现，但交互系统针对单个 NPC

**处理**：
- 只处理当前锁定的 NPC；其他 NPC 正常 AI 运作
- **锁定切换机制**：
  - 触发条件：按下方向键（朝向目标方向）
  - 冷却时间：切换后 0.5 秒内不能再次切换
  - 优先级：自动选择视野内最近的 NPC
- **Interacting 状态下的威胁升级**：
  - 当玩家处于 Interacting 状态时，非锁定 NPC 获得 threat priority 提升
  - 它们更快从 SUSPECT 进入 ALERT/COMBAT

### 边缘情况4：审问期间 NPC 醒来

**问题**：玩家正在审问 UNCONSCIOUS NPC，唤醒计时器到期

**处理**：
- 审问期间唤醒计时器**暂停**
- 每完成一轮审问（获取一条情报），唤醒计时器恢复
- 计时器到期 NPC 苏醒，审问**强制中断**
- 已获取的信息**保留**，但不可继续审问
- NPC 醒来后按降级规则处理（距离 < 2m → SEARCH）

### 边缘情况5：捆绑后 NPC 被其他 NPC 发现

**处理**：与 NPC AI 系统边缘情况一致
- 发现同伴 TIED → 立即进入 ALERT
- 被捆绑 NPC 不参与派系感知
- 玩家可利用这一点"切断"感知链

### 边缘情况6：欺骗时玩家不知道任何 vulnerability

**处理**：
- 欺骗选项在 UI 中**不显示**
- 如果玩家已获取 vulnerability 但选择错误的弱点 → 欺骗失败，触发正常失败惩罚

### 边缘情况7：转化线人后 NPC 被攻击

**问题**：转化 NPC 被同派系攻击却不反击，派系逻辑矛盾

**处理**：
- 转化只改变 allegiance（+50），不改变派系关系
- **转化 NPC 的行为**：
  - 在 allegiance 生效期间（同派系眼中仍是"友军"），如果被攻击会正常触发逃跑/自卫行为
  - 逃跑成功后，allegiance 在下个行为循环恢复正常，NPC 行为恢复正常
- **提供情报后**：allegiance 恢复原值，NPC 不会"记仇"
- **风险**：转化 NPC 在提供情报后可能因接近敌人而陷入危险

### 边缘情况8：玩家在 CanInteract 状态中目标死亡

**处理**：
- 检测到 DEAD 状态自动退出 CanInteract
- **提示形式**：屏幕中央短暂显示"NPC 已死亡"（白色文字，1.5秒淡出）+ 低沉音效
- 如果玩家正在子菜单（威胁/欺骗选项）中，选择**立即关闭**，带 0.2 秒渐隐过渡动画

### 边缘情况9：处决被第三方目击

**问题**：玩家执行处决时，第三方 NPC 正在观看

**处理原则**：处决完成后，本系统向 NPC AI 系统发送 `ExecutionWitnessed(npc_id, witness_npc_id)` 事件，由 NPC AI 系统根据其派系感知规则决定如何响应。

**与 NPC AI 系统边缘情况6（发现同伴处于非 FREE 状态）的职责划分**：
- **Gritty Takedowns 负责**：检测第三方目击，发送 `ExecutionWitnessed` 事件
- **NPC AI 系统负责**：根据 witness 的 Alert State、派系关系、感知共享规则决定最终行为

**预期行为参考**（NPC AI 系统应实现）：
| 第三方 NPC 的 Alert State | 预期处理方式 |
|--------------------------|-------------|
| UNDETECTED | 立即进入 SUSPECT（感知到异常但未确认） |
| SUSPECT | 立即进入 SEARCH（确认有异常） |
| SEARCH | 立即进入 ALERT 并触发派系感知共享 |
| ALERT/COMBAT | 警报立即扩散，呼叫增援 |

**注意**：如果是潜行击杀（背面 + 未被发现），且第三方 NPC 处于 UNDETECTED 时，NPC AI 系统应评估为"未被发现的可疑迹象"，进入 SUSPECT 而非更高级别。具体行为由 NPC AI 系统派系感知机制决定。

## Dependencies

### 架构决策：事件驱动解耦

**问题**：本系统与 NPC AI 系统存在双向硬依赖，形成循环依赖风险。

**解决方案**：采用事件驱动架构，通过 Unity 的 `C# Events` 或 `ScriptableObject Events` 系统解耦。

```
修订前（循环依赖）：
玩家-NPC 交互系统 ──(硬依赖)──► NPC AI 系统
     ▲                                  │
     └────────(硬依赖：接收交互事件)─────┘

修订后（解耦）：
玩家-NPC 交互系统 ──(事件总线)──► NPC AI 系统
     │                                  │
     └────── InteractionEvent ◀────────┘
```

### 上游依赖（本系统依赖谁）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **玩家控制器 (Player Controller)** | 硬依赖 | 读取玩家位置、状态（潜行/站立）、持有物、`IsLocked`；接管 `IsLocked` 进行动作锁定 |
| **NPC AI 系统 (NPC AI System)** | 软依赖（事件订阅 + 查询） | 订阅 `AlertStateChangedEvent` 获取 NPC Alert State 变化；订阅 `NPCStateChangedEvent` 获取 NPC World State 变化；通过 `QueryAlertState(NPC_ID)` 查询当前 Alert State 用于初始判定。 |
| **LOS & Eavesdropping 系统** | 软依赖（查询） | 查询身份标签（恶徒/帮凶/受害者/未知）；仅处决类交互需要，不阻塞其他交互 |
| **环境交互系统 (Environment Interaction)** | 硬依赖 | 查询可用环境物件列表；调用物件效果进行环境处决 |
| **Health & Lethality 系统** | 软依赖（请求/回调） | 发送 `DamageRequest`；通过 `DamageResult` 事件接收结果 |

### 下游依赖（谁依赖本系统）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **NPC AI 系统 (NPC AI System)** | 软依赖（事件订阅） | 订阅本系统的 `InteractionEvent`（威胁/击杀/捆绑等），触发 NPC 状态变化 |
| **Health 系统** | 硬依赖 | 接收 `DamageRequest`（含 penetration）执行伤害计算；穿透值来自武器数据 |
| **Clue & Journal 系统** | 软依赖 | 接收 `KnowledgeGainedEvent` 搜身/审问获取的线索 |
| **Sanity/Rage 系统** | 软依赖 | 接收 `KillTagEvent` 击杀事件及 NPC 身份（异步后置） |
| **沉浸式音频系统** | 软依赖 | 订阅 `InteractionEvent` 用于音效触发 |
| **叙事系统 (Narrative System)** | 软依赖 | 接收 `KillTagEvent` 用于道德计算；接收 `TieUpEvent` 用于仁慈行为记录 |
| **Character Background 系统** | 数据依赖 | 本系统持有并管理 `HasMetFaction[FACTION_ID]` flag，用于首次遭遇加成判定；Character Background 系统通过 `IFirstEncounterBonusProvider` 接口提供首次遭遇加成数据 |

### 本系统持有的玩家状态数据

| 数据结构 | 类型 | 说明 |
|---------|------|------|
| `HasMetFaction[FACTION_ID]` | bool | 标记玩家是否已与该派系发生过对峙/交互。首次遭遇时设为 `true`。用于 Character Background 系统首次遭遇加成计算。 |

### 核心事件接口

#### 本系统发出的事件

| 事件名 | 方向 | 负载 | 说明 |
|--------|------|------|------|
| `InteractionEvent` | → 事件总线 → NPC AI/音频 | `{type, target_id, source, result}` | 所有交互的通用事件 |
| `DamageRequest` | → Health 系统 | `{target_id, damage_type, penetration, source}` | 处决类交互的伤害请求；`penetration` 来自 WeaponData；`source` 来自 WeaponQueryResponse 的 `weapon_id` |
| `KillTagEvent` | → Sanity 系统 | `{kill_tag: NPCIdentityType, kill_source: KillSource}` | 击杀事件（异步后置），携带被击杀NPC的身份标签和死亡方式。详见下方枚举定义 |
| `KnowledgeGainedEvent` | → Clue 系统 | `{npc_id, knowledge_list}` | 搜身/审问获取的线索 |
| `WeaponQueryRequest` | → 武器系统 | `{weapon_id}` | 环境处决前查询武器数据 |
| `WeaponQueryResponse` | ← 武器系统 | `{weapon_id, WeaponData}` | 返回武器数据（含 damage_type、penetration、animation_tags）；`weapon_id` 用作 DamageRequest 的 `source` |

**枚举定义**：

| 枚举 | 值 | 说明 |
|------|-----|------|
| **NPCIdentityType** | `ENEMY` | 已标记为恶徒 |
| | `ACCOMPLICE` | 已标记为帮凶 |
| | `VICTIM` | 已标记为无辜者 |
| | `UNKNOWN` | 未标记，击杀前未通过LOS监听确认身份 |
| **KillSource** | `DIRECT_KILL` | 玩家直接攻击击杀 |
| | `EXECUTION_KILL` | 环境处决 |
| | `INDIRECT_KILL` | 玩家攻击导致倒地后环境致死 |
| | `ACCIDENTAL_KILL` | 第三方NPC误杀 |
| | `SELF_DEFENSE` | NPC自卫导致玩家死亡/NPC撤离 |

> **KillSource 用途说明**：KillSource 用于追踪死亡责任，判断"收益丢失"等边缘情况。与 NPCIdentityType（用于理智惩罚计算）共同构成完整的击杀追踪体系。

#### 本系统订阅的事件

| 事件名 | 来源 | 用途 |
|--------|------|------|
| `AlertStateChangedEvent` | NPC AI 系统 | 监听 NPC 的 Alert State 变化。用于边缘情况1（交互期间状态变化）。 |
| `PlayerDamagedEvent` | Health 系统 | 监听玩家被攻击，用于边缘情况2（Interacting 中被攻击） |
| `NPCStateChangedEvent` | NPC AI 系统 | 监听 NPC 的 World State 变化（FREE→UNCONSCIOUS/DEAD） |

### CONFRONTATION 接口边界（明确所有权）

**问题**：CONFRONTATION 机制涉及本系统和 NPC AI 系统，需要明确 DialogueTree 所有权。

**决策**：

| 接口 | 拥有者 | 说明 |
|------|--------|------|
| `DialogueTree` 数据 | NPC AI 系统 | NPC AI 系统定义并存储所有对话树数据 |
| `DialogueChoice` 发送 | 本系统 | 玩家选择后，本系统发送选中的选项索引 |
| `DialogueResult` 接收 | 本系统 | NPC AI 系统处理后返回结果（allegiance 变化、获取信息等） |
| 对话 UI 渲染 | 本系统 | 本系统负责渲染对话选项 UI，NPC AI 系统提供文本内容 |

**数据流**：
```
本系统（UI渲染+玩家输入） → DialogueChoice → NPC AI 系统（处理+返回结果）
                                              ↓
本系统（显示结果） ← DialogueResult ←
```

### 数据流摘要

```
┌─────────────────────────────────────────────────────────────────────┐
│                    玩家-NPC 交互系统                                   │
├─────────────────────────────────────────────────────────────────────┤
│  Inputs（订阅事件）:                                                    │
│    - AlertStateChangedEvent (NPC AI)  → 监听 NPC 警戒状态变化             │
│    - NPCStateChangedEvent (NPC AI)   → 监听 NPC 世界状态变化             │
│    - PlayerDamagedEvent (Health)     → 监听玩家被攻击                   │
│                                                                     │
│  Outputs（发送事件）:                                                  │
│    - InteractionEvent (→ 事件总线 → NPC AI/音频)                      │
│    - DamageRequest (→ Health 系统)                                   │
│    - KillTag (→ Sanity 系统，异步)                                   │
│    - KnowledgeGained (→ Clue 系统)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 与上游系统的依赖记录（需在其他系统 GDD 中同步更新）

| 系统 | 需补充的依赖记录 |
|------|-----------------|
| LOS & Eavesdropping 系统 | 下游依赖：`玩家-NPC 交互系统`（身份标签查询） |
| 环境交互系统 | 下游依赖：`玩家-NPC 交互系统`（物件效果调用） |
| NPC AI 系统 | 上游依赖：本系统订阅其事件（解耦后） |
| **叙事系统** | 下游依赖：`玩家-NPC 交互系统`（接收 KillTagEvent 和 TieUpEvent） |

## Tuning Knobs

*以下参数暴露给策划在引擎 Inspector 中直接调整，无需修改代码。*

### 交互范围参数

| 参数 | 类型 | 默认值 | 安全范围 | 说明 |
|------|------|--------|---------|------|
| `InteractionRange_Melee` | float | 1.5m | 1.0m ~ 3.0m | 威胁/贿赂/欺骗的交互范围 |
| `InteractionRange_Search` | float | 1.0m | 0.5m ~ 2.0m | 搜身/审问的交互范围 |
| `InteractionRange_TieUp` | float | 1.5m | 1.0m ~ 2.5m | 捆绑/解开的交互范围 |
| `StealthBackAngle` | float | 150° | 120° ~ 180° | 潜行击杀的背面判定角度（越小越严格） |

### 时序参数

| 参数 | 类型 | 默认值 | 安全范围 | 说明 |
|------|------|--------|---------|------|
| `BaseTieUpTime` | float | 4.0s | 3.0s ~ 6.0s | 基础捆绑时间 |
| `TargetSwitchCooldown` | float | 0.5s | 0.3s ~ 1.0s | 切换目标的冷却时间 |
| `StaggerRecoveryThreshold` | float | 0.5 | 0.3 ~ 0.7 | 处决中断时 NPC 存活判定（动画进度比例）。低于0.3处决窗口过短，高于0.7几乎总会失败 |

### 欺骗判定参数

| 参数 | 类型 | 默认值 | 安全范围 | 说明 |
|------|------|--------|---------|------|
| `DeceptionBaseChance` | float | 0.6 | 0.4 ~ 0.8 | 欺骗基础成功率 |
| `DeceptionIntelligenceBonus` | float | 0.2 | 0.0 ~ 0.3 | 玩家属性加成上限 |

### 转化参数

| 参数 | 类型 | 默认值 | 安全范围 | 说明 |
|------|------|--------|---------|------|
| `ConversionAllegianceBoost` | int | +50 | +30 ~ +80 | 转化线人的基础 allegiance 提升 |
| `VulnerabilityDepthMultiplier_Max` | float | 1.5 | 1.0 ~ 2.0 | 弱点深度最大乘数 |
| `ConversionEffectDuration` | float | 30.0s | 20.0s ~ 60.0s | 转化效果持续时间，超时后 allegiance 自动恢复 |

### UI 反馈参数

| 参数 | 类型 | 默认值 | 安全范围 | 说明 |
|------|------|--------|---------|------|
| `DeathNotificationDuration` | float | 1.5s | 1.0s ~ 3.0s | "NPC 已死亡"提示显示时长 |
| `SubmenuFadeDuration` | float | 0.2s | 0.1s ~ 0.5s | 子菜单渐隐动画时长 |

### 调参风险提示

| 参数 | 风险 |
|------|------|
| `StealthBackAngle` 设置过大 | 背面击杀太容易，削弱潜行挑战 |
| `BaseTieUpTime` 设置过长 | 捆绑时间成本过高，玩家倾向直接击杀 |
| `DeceptionBaseChance` 设置过高 | 欺骗过于安全，信息收集价值降低 |
| `TargetSwitchCooldown` 设置过短 | 玩家可高速切换目标导致 AI 无法反应 |

## Visual/Audio Requirements

*详见 `design/ux/player-npc-interaction-ux.md` 和 `design/ux/player-npc-interaction-visual.md`*

### 视觉风格

**漫画/风格化** — 线条分明，高对比度。参考《Fortnite》《Borderlands》

### 颜色系统

| 用途 | 颜色 | Hex |
|------|------|-----|
| 处决/敌人 | Crimson | `#E63946` |
| 转化/完成 | Jade | `#2ECC71` |
| 搜身/情报 | Cobalt | `#3498DB` |
| 威胁/帮凶 | Amber | `#F39C12` |
| 捆绑/禁用 | Slate | `#5D6D7E` |
| 描边 | Outline | `#0D0D0D` |

### 图标规范

| 图标类型 | 形状 | 颜色 | 尺寸 |
|---------|------|------|------|
| 处决 | 匕首轮廓 | `#E63946` + 银白刃部 | 32x32px |
| 威胁/贿赂/欺骗 | 对话气泡，锯齿边缘 | `#F39C12` | 28x28px |
| 搜身/审问 | 放大镜 | `#3498DB` | 30x30px |
| 捆绑 | 绳索打结 | `#5D6D7E` | 26x26px |
| 转化 | 握手 | `#2ECC71` | 28x28px |

### 动画规范

| 状态 | 动画类型 | 参数 | 时长 |
|------|---------|------|------|
| 头顶图标出现 | 缩放弹入 | 0.0→1.2→1.0 | 200ms |
| 头顶图标消失 | 淡出 + 缩小 | opacity 1→0, scale 1.0→0.8 | 300ms |
| 当前目标脉动 | 微弱呼吸 | scale 1.0→1.05→1.0 | 1.0s 循环 |
| 次级目标 | 静止 | — | — |
| 进度条脉动 | 亮度±10%, 缩放±3% | ease-in-out | 1.0s 周期 |
| 径向菜单展开 | 从中心向外弹出，依次延迟 30ms | ease-out back | 250ms |
| 方向指示器呼吸 | opacity 0.6→0.8 | — | 1.0s 周期 |

### 屏幕反馈

| 效果 | 参数 |
|------|------|
| 顿帧 (Hit Stop) | 2-3 帧 |
| 震动 | 强度由 ScreenEffects 系统控制 |
| 镜头缩放 | 由 ScreenEffects 系统控制 |
| 暗角 | 强度 0.5 |

### 资产需求

| 类型 | 资产 |
|------|------|
| 纹理 | 图标集、进度条纹理、径向菜单背景、方向指示器、NPC 身份标签 |
| 字体 | 位图字体 (ui_font_tag.fnt, ui_font_numbers.fnt) |
| 着色器 | 进度条脉动效果、双层描边效果 |
| 音效 | 进入专注模式低音效、锁定声源滴声、进度累积电流声、50%咔声、完成成功提示音 |

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 潜行击杀在满足全部条件时可正常执行并击杀目标 | 在背面 + 潜行状态 + 已标记恶徒 + Alert State < ALERT（即 UNDETECTED/SUSPECT/SEARCH）时执行，验证击杀 |
| AC-2 | 潜行击杀在任一条件不满足时触发警报或拒绝执行 | 分别破坏每个条件，验证系统响应 |
| AC-3 | 环境处决在持有可用物件时可正常执行 | 拾取环境物件，对目标执行处决，验证物件效果触发 |
| AC-4 | 捆绑交互在 3-6 秒内将 UNCONSCIOUS NPC 转为 TIED 状态 | 执行捆绑，计时，验证 NPC 状态和 Alert 降级 |
| AC-5 | 审问在 NPC UNCONSCIOUS 时可获取 knowledge | 执行审问，验证获取的 knowledge 列表 |
| AC-6 | 审问在 NPC DEAD 时不可用（只能搜身） | 对 DEAD NPC 尝试审问，验证无选项显示 |
| AC-7 | 欺骗选项仅在玩家已获取 vulnerability 时显示 | 未获取时验证无欺骗选项；获取后验证显示 |
| AC-8 | 转化线人执行后 allegiance 提升并在情报提供后恢复 | 执行转化，验证 allegiance 变化和情报获取 |

### 边缘情况验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-9 | 玩家在 Interacting 状态中被 Lethal 伤害导致死亡，处决根据动画进度判定结果 | 在处决中途受伤，验证死亡和处决结果 |
| AC-10 | 玩家在 CanInteract 状态中目标死亡，自动退出并显示提示 | NPC 死亡，验证状态退出和 UI 提示 |
| AC-11 | 处决被第三方 NPC 目击，根据其 Alert State 触发正确响应 | 在不同 Alert 状态下执行目击处决，验证响应 |
| AC-12 | 审问期间唤醒计时器正确暂停和恢复 | 执行审问，验证计时器行为 |
| AC-13 | 多个 NPC 时方向键切换目标有冷却限制 | 快速切换，验证冷却机制 |
| AC-14 | 玩家处于 Interacting 时非锁定 NPC threat priority 提升 | Interacting 状态中观察其他 NPC 行为变化 |

### 跨系统验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-15 | 交互事件正确发送到事件总线 | 监控事件总线，验证 InteractionEvent |
| AC-16 | DamageRequest 正确发送到 Health 系统 | 监控 Health 系统，验证伤害请求 |
| AC-17 | 击杀事件正确发送到 Sanity 系统 | 监控 Sanity 系统，验证 KillTag |
| AC-18 | 搜身/审问正确发送到 Clue 系统 | 监控 Clue 系统，验证 KnowledgeGained |

### 性能预算

| 指标 | 预算 | 说明 |
|------|------|------|
| 单次交互响应时间 | < 1 帧 | 交互选项显示应即时 |
| 动画锁定期间 CPU 占用 | < 2ms | 不影响其他系统运行 |
| 事件系统处理延迟 | < 16ms | 跨系统事件传递 |

## Open Questions

| # | 问题 | 负责人 | 目标日期 |
|---|------|--------|---------|
| OQ-1（已解决） | **DialogueTree 的具体数据结构定义** | NPC AI 系统设计者 | ✅ 已解决 |
| OQ-2 | ✅ **已解决**：转化线人的"临时"持续时间设置超时机制。详见核心规则第7条。 | 已解决 | — |
| OQ-3（已解决） | **环境处决的物件效果优先级**：当多个物件同时可用时，选择距离最近的物件。玩家可通过轻微移动切换选择目标。 | 已解决 | — |
| OQ-4 | 是否需要"非致命威胁"选项（如打伤但不杀） | 游戏设计师 | MVP 评审时 |
| OQ-5 | 玩家在审问/捆绑时 NPC 苏醒，动画如何过渡 | 技术艺术 + 动画师 | Vertical Slice 时 |
| OQ-6（已澄清） | **LOS 身份标签查询机制**：身份标签（Enemy/Accomplice/Victim）由 LOS 系统维护并提供查询接口 `QueryNPCIdentity(npc_id)`。处决系统通过此接口查询，不直接访问 NPC AI 系统。 | 已澄清 | — |

### DialogueTree 数据结构定义（OQ-1 已解决）

**数据存储格式**：JSON 文件（可由策划编辑，由 `DialogueTreeConfig` 类反序列化）

```json
{
  "dialogue_id": "string",
  "npc_id": "string",
  "root_branch": "branch_id",
  "branches": {
    "branch_id": {
      "speaker": "string",
      "text": "string",
      "emotion": "enum (NEUTRAL/AGITATED/SCARED/ANGRY)",
      "choices": [
        {
          "choice_id": "string",
          "text": "string",
          "next_branch": "branch_id | null",
          "allegiance_change": "int",
          "requires_vulnerability": "bool",
          "result_type": "enum (CONTINUE/INTIMIDATE/BRIBE/DECOY)"
        }
      ]
    }
  }
}
```

**接口调用方式**：
1. 玩家触发对峙 → 玩家-NPC 交互系统向 NPC AI 系统发送 `ConfrontationStartRequest(npc_id)`
2. NPC AI 系统返回 `DialogueTreeConfig` 数据
3. 玩家-NPC 交互系统负责 UI 渲染和玩家选择
4. 玩家选择后，发送 `DialogueChoice(choice_id, npc_id)` 到 NPC AI 系统
5. NPC AI 系统处理并返回 `DialogueResult(allegiance_change, knowledge_gained)`
