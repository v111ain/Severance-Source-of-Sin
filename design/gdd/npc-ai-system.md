# NPC AI系统 (NPC AI System)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-04-06
> **Implements Pillar**: 致命的脆弱感 (Lethal Fragility)
> **Revision Notes**: GDD 定稿。Open Questions (OQ-1,2,4,5,7,8,9,10) 涉及派系完整定义等非 MVP 内容，延后至 Alpha 阶段处理。OQ-3 已解决。

## Overview

NPC AI 系统驱动游戏中所有非玩家角色的身份、行为与协作。NPC 并非单纯的"敌人"——他们是有**隐藏身份的可疑人物**。玩家通过监听对话、观察行为来逐步揭开真相：谁是恶徒，谁是帮凶，谁是无辜者。

**核心交互是"猜"**：猜对了，获得情报优势，解锁新路径或线索；猜错了，承担真实后果——误杀无辜者会让关键线索断裂，误放真凶会让后续关卡难度飙升。辨识不是被动避免惩罚，而是主动获取优势的策略。

NPC 在各自的世界中巡逻、工作、社交，对周围的威胁浑然不觉。他们有盲区、有注意力极限、有记忆——玩家可以利用这些漏洞制造混乱、分化注意力。但当威胁升级，他们之间会迅速共享信息，形成群体响应。战斗一旦爆发，《Hotline Miami》式的一击必杀紧张感随之而来。

系统基调以《Last of Us》的叙事驱动为骨架（60%），融合《Hitman》的社会潜行辨识逻辑（20%）和《Hotline Miami》的失控战斗（20%）。

## Player Fantasy

**地基：信息不对称的紧张感**
你潜入一个陌生的据点，周围满是 NPC——他们看起来都一样，穿着相似的制服，做着相似的工作。你不知道谁是恶徒的核心成员，谁是被胁迫的帮凶，谁是无辜的受害者。在这种信息真空里，每一次移动都伴随着暴露的风险。你必须行动，但你是在黑暗中摸索。

**过程：认知掌控感**
通过监听对话、观察行为、收集碎片化的线索，你逐步拼凑出真相——"那个人是负责看门的"、"这个人在和谁通话"、"那个区域有三个人，但只有两个是武装的"。当最后一个拼图落下，你对整个据点的人员结构了然于胸，那是一种系统性的认知快感——你不只是在猜测，你是在理解。

**过程：智取感**
当你摸清了 NPC 的巡逻路线、注意力盲区和社交网络，你会开始"智取"——利用一个 NPC 吸引另一个的注意，在视线盲区穿行，把一个无害的 NPC 当作移动掩护。不是"利用漏洞"的侥幸，而是"我比这个系统更聪明"的能力确认。

**高潮：处决者的黑暗满足感**
当你从阴影中现身，将一个已确认的恶徒处决，那一刻你完成了从猎物到猎人的身份转变。这种快感不是单纯的暴力释放，而是精心策划后的必然结果——你知道他会死，因为你早就把他标记了。

**代价：后悔与代价感**
当你猜错了——误杀了一个帮凶，甚至是无辜者——后果立即显现。可能是关键线索的断裂，可能是理智值的崩溃，可能是内心的道德愧疚。这不是游戏惩罚，而是真实的人性反应。

**极端情境：Hotline Miami 式肾上腺素**
一旦暴露，平静立即转为混乱。一人倒下，另一人开枪，你只有几秒钟做出生死抉择。一击必杀的规则让每一次操作都命悬一线——这就是失控的代价。

## Detailed Design

### Core Rules

**规则1：NPC 立场向量 (Stance Vector)**

每个 NPC 拥有一个立场向量，定义其身份：

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `affiliation` | 派系列表 | 所属派系（可多属） | ["青龙帮", "赌场保护费"] |
| `allegiance` | 整数 | 对玩家态度 (-100 敌对 ~ +100 友好) | -30, 0, +20 |
| `vulnerability` | 弱点列表 | 玩家可利用的心理/现实弱点 | ["母亲重病", "欠债", "想逃跑"] |
| `knowledge` | 线索ID列表 | 掌握的情报 | ["仓库密码", "接头人姓名"] |
| `state` | 枚举 | 当前状态 | FREE, UNCONSCIOUS, TIED, DEAD |

**规则2：交互触发条件**

玩家可以对特定状态下的 NPC 执行交互：

| 交互层 | 可执行对象状态 | 说明 |
|--------|---------------|------|
| **情报层** | FREE / UNCONSCIOUS | 窃听（FREE）、搜身（UNCONSCIOUS/DEAD）、审问（UNCONSCIOUS） |
| **操控层** | FREE | 威胁、贿赂、欺骗、心理操纵——需要 NPC 能听能反应 |
| **处置层** | 任意 | 击杀、击晕、捆绑、释放——基于当前 state 决定可用选项 |

**规则3：后果传导机制**

每次交互产生后果链，分三类：

- **立即后果**：交互当下发生（如 NPC 态度变化、获得线索）
- **延迟后果**：影响后续事件（如某 NPC 失踪引发派系反应）
- **状态后果**：改变 NPC 的 `state`（如自由→昏迷）

**规则4：派系感知共享**

同派系 NPC 之间会共享感知信息（位置、威胁类型）。派系之间存在敌对关系时，感知信息不会跨边界共享——这是玩家可以利用的漏洞。

示例：青龙帮的 NPC 发现玩家，不会通知相邻的军阀 NPC。玩家可以在军阀地盘制造青龙帮的动静，转移军阀注意力。

**规则5：派系间利益矛盾**

派系之间存在动态的利益关系：
- **敌对**：一个派系的敌人是另一个派系的盟友
- **竞争**：两个派系争夺同一资源（玩家可渔翁得利）
- **中立**：无直接利益冲突

玩家可以主动激化派系矛盾，制造混乱来获取战术优势。

**规则6：allegiance 变化规则**

| 玩家行为 | allegiance 变化 | 备注 |
|----------|---------------|------|
| 成功窃听/获取情报 | +5 | NPC 未察觉 |
| 玩家被发现（未受伤） | -10 |  |
| 威胁 NPC | -20 | 立即变化 |
| 成功贿赂 NPC | +10 | 消耗资源 |
| 转化 NPC 为玩家做事 | +30 | 重大突破 |
| 击杀恶徒 | +5 | 理智可能恢复 |
| 击杀帮凶 | -15 | 理智惩罚 |
| 击杀无辜者 | -50 | 理智崩溃 |

`allegiance` 影响 NPC 的行为反应：敌对（-100~-30）会主动攻击或逃跑；中立（-30~+30）会观望或犹豫；友好（+30~+100）可能在关键时刻帮助玩家或提供情报。

### States and Transitions

### World State 与 Health 状态的映射

所有具有生命体的 NPC 共享 Health 系统的状态定义：

| Health 状态 | NPC AI World State | 行为说明 |
|------------|-------------------|---------|
| Healthy | FREE | 正常行为 |
| **Staggered** | **FREE (特殊标记: STAGGERED)** | 播放硬直动画，期间无法执行主动行为，但可视/可听。玩家可趁机执行处决。 |
| Downed | UNCONSCIOUS | 倒地，昏迷状态，唤醒计时器启动 |
| DEAD | DEAD | 终止态，不参与任何行为系统 |

**Staggered 处理细节**：
- Staggered 期间 NPC AI 的 Alert State **冻结**（不上升也不下降）
- Staggered 持续时间由 Health 系统的 `StaggerDuration` 控制（约 1.5 秒）
- Staggered 结束后恢复原 Alert State
- 如果 Staggered 期间被 Lethal 伤害击中，直接转入 DEAD（无视 Alert State 冻结）

#### 有效状态组合

| World State | Alert State | 有效性 | 行为描述 |
|------------|-------------|--------|---------|
| **FREE** | UNDETECTED | 有效 | 正常巡逻/工作/社交，对玩家存在完全不知情。 |
| **FREE** | SUSPECT | 有效 | 感到不安，暂停当前动作，开始观察周围。更频繁扫视，但尚未确认威胁。 |
| **FREE** | SEARCH | 有效 | 已确认有异常，主动搜索。沿可疑路径移动，查看掩体，但尚未呼叫增援。 |
| **FREE** | ALERT | 有效 | 确认威胁存在，大声呼叫增援，通知派系成员，但尚未主动追击。 |
| **FREE** | ESCAPE | 有效 | 威胁过大，选择逃离并呼叫增援。不主动攻击，但会引导增援到玩家位置。 |
| **FREE** | COMBAT | 有效 | 主动追击并尝试消灭威胁。射击/近战，持续呼叫增援。 |
| **UNCONSCIOUS** | *(冻结→降级)* | 条件有效 | 保留原 Alert State，唤醒时降一级（如 COMBAT→ALERT）。 |
| **TIED** | *(冻结→降级)* | 条件有效 | 保留原 Alert State，解开时降一级。 |
| **DEAD** | N/A | 终止态 | NPC 死亡，无任何状态。派系感知共享终止。 |

#### Alert State 转换条件

| 当前状态 | 目标状态 | 触发条件 | 转换前提 |
|---------|---------|---------|---------|
| **UNDETECTED** | SUSPECT | 感知异常（听到脚步声、看到物体移动）；看到其他 NPC 处于 SUSPECT+；派系感知共享收到"可疑"情报 | NPC 处于 FREE |
| **SUSPECT** | SEARCH | 发现直接证据（看到玩家轮廓、发现新鲜尸体）；派系感知共享收到"确认"情报；SUSPECT 持续 > 3秒且仍有异常 | 已触发过 SUSPECT |
| **SUSPECT** | UNDETECTED | 3秒内无进一步异常；派系感知共享收到"解除警报" | 已触发过 SUSPECT |
| **SEARCH** | ALERT | 直接看到玩家；发现关键证据（大量血迹、多个同伴死亡） | 已触发过 SEARCH |
| **SEARCH** | UNDETECTED | 8秒内未发现威胁；派系感知共享收到"解除警报" | 已触发过 SEARCH |
| **ALERT** | COMBAT | 玩家进入攻击范围；玩家主动攻击；ALERT 持续 > 5秒且玩家未被驱逐 | 已触发过 ALERT |
| **ALERT** | ESCAPE | 威胁超出应对能力（如同时面对3个以上敌人）；看到同伴被快速击杀 | 已触发过 ALERT |
| **ALERT** | SEARCH | 玩家逃离视野 > 2秒 | 已触发过 ALERT |
| **ESCAPE** | SEARCH | 成功逃离当前区域，到达安全位置 | 已触发过 ESCAPE |
| **ESCAPE** | ALERT | 逃离后增援到达，重新集结 | 已触发过 ESCAPE |
| **COMBAT** | SEARCH | 玩家逃离视野 > 10秒；所有敌人死亡 | 已触发过 COMBAT |
| **COMBAT** | ESCAPE | 伤亡过大（同伴死亡 > 50%）且玩家仍存活 | 已触发过 COMBAT |

#### World State 转换

| 当前状态 | 目标状态 | 触发条件 | 备注 |
|---------|---------|---------|------|
| **FREE** | UNCONSCIOUS | Blunt 伤害导致 Downed；苏醒计时 15秒 | 触发"冻结 Alert→降一级" |
| **FREE** | DEAD | Lethal 伤害 | Alert State 终止 |
| **FREE** | TIED | 玩家执行"捆绑"交互 | 触发"冻结 Alert→降一级" |
| **UNCONSCIOUS** | FREE | 计时到期自动苏醒 | 恢复降级后的 Alert State |
| **UNCONSCIOUS** | DEAD | 补刀（Lethal 伤害） | — |
| **TIED** | FREE | 玩家执行"解开"交互 | 恢复降级后的 Alert State |
| **TIED** | UNCONSCIOUS | 受到 Blunt 伤害 | 保持 TIED，增加昏迷层级 |
| **TIED** | DEAD | 补刀 | — |
| **DEAD** | — | 无（终止态） | — |

#### 降级规则（UNCONSCIOUS/TIED 唤醒时）

| 唤醒前的 Alert State | 唤醒后的 Alert State |
|---------------------|---------------------|
| UNDETECTED | UNDETECTED（无变化） |
| SUSPECT | UNDETECTED |
| SEARCH | SUSPECT |
| ALERT | SEARCH |
| ESCAPE | SEARCH |
| COMBAT | ALERT |

#### 派系感知共享规则

| 派系关系 | 共享概率 | 共享延迟 | 内容降级 |
|---------|---------|---------|---------|
| 同派系 | 100% | 2秒 | 无 |
| 联盟派系 | 50% | 4秒 | 降一级 |
| 敌对派系 | 0% | — | — |
| 中立派系 | 10% | 6秒 | 降一级 |

### Interactions with Other Systems

#### 数据流入 (Inputs)

| 来源系统 | 数据内容 | 说明 |
|---------|---------|------|
| **Player Controller** | 玩家世界坐标、运动状态、移动速度 | 用于计算 NPC 对玩家的感知距离和方向 |
| **LOS & Eavesdropping** | `PlayerSpottedEvent` 事件 | 玩家被发现的暴露事件，触发 NPC Alert State 上升 |
| **Health & Lethality** | 伤害结果通知 (内部接口) | Health 系统计算伤害后，通过内部接口通知 NPC AI 系统更新 World State；NPC AI 系统随后广播 `NPCStateChangedEvent` |
| **Gritty Takedowns** | `ExecutionWitnessed(npc_id, witness_npc_id)` | 处决被第三方目击，触发 witness NPC 的 Alert State 变化（参见边缘情况6） |
| **Environment Interaction** | 环境事件 | 如某扇门被打开、某物体被移动——NPC 可能感知到并触发 SUSPECT |

#### 数据流出 (Outputs)

| 目标系统 | 数据内容 | 说明 |
|---------|---------|------|
| **LOS & Eavesdropping** | NPC 对话文本、关键词 | NPC 的语音/对话内容，供 LOS 系统提取和过滤 |
| **Health & Lethality** | `QueryState(NPC_ID)` | 查询接口，返回 NPC 当前 World State |
| **Health & Lethality** | `NPCStateChangedEvent` | NPC 状态转移（FREE→UNCONSCIOUS/DEAD）广播 |
| **Sanity/Rage System** | 被击杀 NPC 的 `Tag` | 击杀事件携带的 NPC 身份标签（Enemy/Accomplice/Victim/Innocent） |
| **Clue & Journal** | NPC 的 `knowledge` | NPC 掌握的线索列表，搜身/审问后可获取 |

**`knowledge` 数据结构**：
```python
knowledge: List[clue_id: String]  # 线索ID列表，如 ["ID_张三的身份", "ID_仓库位置"]
```
- `knowledge` 是**线索ID列表**，不是完整的线索内容
- Clue & Journal 系统收到后会查询完整的线索详情
| **Gritty Takedowns** | `QueryAlertState(NPC_ID)` | 查询接口，返回 NPC 当前 Alert State |
| **派系感知网络** | `SharedAlert` 广播 | 同派系 NPC 之间的感知信息共享 |

#### 接口所有权

| 接口类型 | 拥有者 | 说明 |
|---------|-------|------|
| `PlayerSpottedEvent` 事件 | LOS System | LOS 系统检测到玩家暴露时触发，NPC AI 系统订阅此事件 |
| `DamageResult` (内部接口) | Health System | Health 系统计算伤害后，通过内部接口传递伤害结果给 NPC AI 系统，由 NPC AI 系统更新状态并广播 |
| `NPCStateChangedEvent` | NPC AI System | NPC AI 系统拥有，Health 系统触发后由 NPC AI 系统广播。NPC 世界状态变化（FREE→UNCONSCIOUS/DEAD）时触发。 |
| **`AlertStateChangedEvent`** | NPC AI System | NPC AI 系统在 Alert State 转换时触发，通知外部系统（如 Gritty Takedowns）。参见下方事件定义。 |
| `QueryState/QueryAlertState` | NPC AI System | 查询接口，其他系统通过接口获取 NPC 状态 |
| `SharedAlert` 广播 | NPC AI System | NPC AI 系统内部派系共享机制，对其他系统透明 |
| `InteractionEvent` | NPC AI System | 本系统订阅来自沉重处决系统的交互事件（威胁/击杀/捆绑等），触发相应 NPC 行为变化 |

#### 查询接口规范

以下接口由 NPC AI 系统提供，供其他系统实时查询 NPC 状态：

**`QueryState(NPC_ID: int) -> WorldState`**
| 参数 | 类型 | 说明 |
|------|------|------|
| `NPC_ID` | int | NPC 的唯一标识符 |
| **返回值** | `WorldState` (枚举) | NPC 的当前世界状态：`FREE`, `UNCONSCIOUS`, `TIED`, `DEAD` |

**`QueryAlertState(NPC_ID: int) -> AlertState`**
| 参数 | 类型 | 说明 |
|------|------|------|
| `NPC_ID` | int | NPC 的唯一标识符 |
| **返回值** | `AlertState` (枚举) | NPC 的当前警觉状态：`UNDETECTED`, `SUSPECT`, `SEARCH`, `ALERT`, `ESCAPE`, `COMBAT` |

#### 事件接口定义

**`AlertStateChangedEvent` 事件**

NPC AI 系统在 Alert State 发生转换时触发此事件，通知外部系统（如 Gritty Takedowns）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `npc_id` | int | 状态变化的 NPC 标识符 |
| `old_state` | AlertState | 变化前的警觉状态 |
| `new_state` | AlertState | 变化后的警觉状态 |
| `trigger` | AlertTrigger | 触发原因，枚举值见下表 |

**AlertTrigger 枚举值**：

| 值 | 说明 |
|----|------|
| `PERCEPTION` | 感知触发（视觉/听觉/记忆评分变化） |
| `FACTION` | 玩家主动行为（威胁、攻击等） |
| `SHARED` | 派系感知共享 |
| `EXECUTION` | 处决被目击（收到 `ExecutionWitnessed` 事件） |

**触发时机**：Alert State 表格中的**任何**状态转换时均触发。

**`QueryAllegiance(NPC_ID: int) -> int`**
| 参数 | 类型 | 说明 |
|------|------|------|
| `NPC_ID` | int | NPC 的唯一标识符 |
| **返回值** | `int` | NPC 对玩家的态度值，范围 -100 ~ +100 |

**`QueryVulnerability(NPC_ID: int) -> List[String]`**
| 参数 | 类型 | 说明 |
|------|------|------|
| `NPC_ID` | int | NPC 的唯一标识符 |
| **返回值** | `List[String]` | NPC 已知的弱点列表，如 `["母亲重病", "欠债"]` |

**`QueryKnowledge(NPC_ID: int) -> List[String]`**
| 参数 | 类型 | 说明 |
|------|------|------|
| `NPC_ID` | int | NPC 的唯一标识符 |
| **返回值** | `List[String]` | NPC 掌握的线索 ID 列表，如 `["ID_仓库密码", "ID_接头人"]` |

## Formulas

### 公式1：allegiance 变化计算

NPC 对玩家的态度（allegiance）根据玩家交互行为实时更新。

`AllegianceDelta = BaseChange * InteractionTypeMultiplier * ContextMultiplier * RelationshipMultiplier`

| 变量 | 定义 | 范围/值 |
|------|------|---------|
| `BaseChange` | 基础变化值 | 见交互类型表 |
| `InteractionTypeMultiplier` | 交互类型乘数 | 威胁=-1.0, 贿赂=1.0, 欺骗=0.8, 心理操纵=1.2 |
| `ContextMultiplier` | 情境乘数 | NPC 当时 Alert State：UNDETECTED=1.0, SUSPECT=1.2, SEARCH=1.5 |
| `RelationshipMultiplier` | 关系乘数 | 派系敌对=0.8, 派系中立=1.0, 派系友好=1.2 |

**立即后果变化量表（示例）**：

| 玩家行为 | BaseChange | 说明 |
|----------|-----------|------|
| 成功窃听/获取情报 | +5 | NPC 未察觉 |
| 玩家被发现（未受伤） | -10 | |
| 威胁 NPC | -20 | |
| 成功贿赂 NPC | +10 | 消耗资源 |
| 欺骗成功 | +10 | |
| 心理操纵成功 | +20 | 利用已知弱点 |
| 转化 NPC 为玩家做事 | +30 | 重大突破 |
| 击杀恶徒 (Enemy) | +5 | 理智可能恢复 |
| 击杀帮凶 (Accomplice) | -15 | 理智惩罚 |
| 击杀无辜者 (Victim) | -50 | 理智崩溃 |

**allegiance 范围**：`Clamp(Allegiance + AllegianceDelta, -100, +100)`

**基于 allegiance 的行为阈值**：

| allegiance 范围 | NPC 行为倾向 |
|----------------|-------------|
| -100 ~ -30 | 主动攻击或逃跑（敌对） |
| -30 ~ +30 | 观望或犹豫（中立） |
| +30 ~ +100 | 可能帮助玩家或提供情报（友好） |

---

### 公式2：威胁评分公式

当 NPC 感知到多个潜在威胁时（如同时看到玩家和听到远处枪声），使用威胁评分决定优先级响应。

`ThreatScore = DistanceScore + SeverityScore + UrgencyScore`

| 变量 | 定义 | 计算方式 |
|------|------|---------|
| `DistanceScore` | 距离评分 | `1.0 - (DistanceToThreat / MaxPerceptionRange)`，范围 0.0~1.0 |
| `SeverityScore` | 严重性评分 | 视觉威胁=1.0, 听觉威胁=0.6, 间接证据=0.3 |
| `UrgencyScore` | 紧迫性评分 | 基于威胁类型：直接射击=1.0, 接近中=0.8, 静止=0.4 |

**威胁类型权重**：

| 威胁类型 | DistanceScore 权重 | SeverityScore 权重 | UrgencyScore 权重 |
|---------|-------------------|-------------------|------------------|
| 直接看到玩家 | 高（距离近权重更高） | 1.0 | 高 |
| 听到枪声 | 中 | 0.6 | 中 |
| 发现同伴尸体 | 低（与距离无关） | 0.3 | 低 |
| 听到脚步声 | 中 | 0.6 | 中 |

**多威胁优先级判定**：选择 ThreatScore 最高的威胁作为主要响应目标。

---

### 公式3：派系感知共享延迟

同派系 NPC 之间的感知信息共享不是瞬时的，延迟基于距离计算。

`SharedAlertDelay = BaseDelay + (DistanceBetweenNPCs / SharedAlertSpeed)`

| 变量 | 定义 | 默认值 |
|------|------|-------|
| `BaseDelay` | 基础延迟 | 2.0 秒 |
| `DistanceBetweenNPCs` | 两个 NPC 之间的距离 | 动态计算 |
| `SharedAlertSpeed` | 感知共享传递速度 | 10.0 米/秒 |

**感知共享距离衰减**：如果两个 NPC 之间有墙体遮挡，`SharedAlertSpeed` 降低 50%。

**示例计算**：
- 两个相邻（同区域）的同派系 NPC：延迟 ≈ 2.0 + (5 / 10) = 2.5 秒
- 两个距离 20 米的同派系 NPC：延迟 ≈ 2.0 + (20 / 10) = 4.0 秒
- 中间有墙体的 NPC：延迟 ≈ 2.0 + (20 / 5) = 6.0 秒

---

### 公式4：NPC 感知范围计算

NPC 对玩家的感知综合评分，用于判定 Alert State 转换。

`PerceptionScore = VisualScore + AudioScore + MemoryScore`

| 变量 | 定义 | 计算方式 | 与 LOS 系统的关系 |
|------|------|---------|------------------|
| `VisualScore` | 视觉感知评分 | 直接读取 `LOS.CurrentExposure / 100`，范围 0.0~1.0 | LOS 系统计算玩家的暴露进度 |
| `AudioScore` | 听觉感知评分 | 静止=0, 潜行=0.3, 行走=0.6, 冲刺=1.0 | NPC AI 独立计算 |
| `MemoryScore` | 记忆残留评分 | 离开视野后 10 秒内=0.8, 10-30 秒=0.4, 30 秒+=0 | NPC AI 独立计算 |

**与 LOS 系统的职责划分**：
- **LOS 系统**：计算玩家被 NPC 发现的**暴露进度** (0-100%)
- **NPC AI 系统**：计算 NPC 对玩家的**综合感知评分** (0.0~1.0)
- 当 LOS 的 `CurrentExposure >= 100` 时，NPC AI 收到 `PlayerSpottedEvent` 事件，VisualScore 瞬间设为 1.0

**感知评分到 Alert State 转换的触发条件映射**：

| 触发条件 | VisualScore | AudioScore | MemoryScore | PerceptionScore | 触发 Alert State |
|---------|-------------|------------|-------------|-----------------|-----------------|
| 感知异常（脚步声/物体移动） | 0.1 | 0.3 | 0 | 0.4 | SUSPECT |
| 看到玩家轮廓 | 0.5 | 0 | 0 | 0.5 | SUSPECT |
| 发现新鲜尸体 | 0.3 | 0 | 0.1 | 0.4 | SUSPECT |
| 直接看到玩家（PlayerSpottedEvent 事件） | 1.0 | 0 | 0 | 1.0 | ALERT |
| 发现关键证据（大量血迹） | 0.5 | 0 | 0.3 | 0.8 | ALERT |

**感知阈值**：

| PerceptionScore 范围 | 触发 Alert State | 说明 |
|---------------------|-----------------|------|
| 0 ~ 0.3 | UNDETECTED | 无感知 |
| 0.3 ~ 0.6 | SUSPECT | 感到不安 |
| 0.6 ~ 0.8 | SEARCH | 主动搜索 |
| > 0.8 | ALERT | 确认威胁 |

### 公式5：ProximityScore（对峙距离评分）

用于判定对峙（CONFRONTATION）交互的触发条件。

`ProximityScore = 1.0 - (DistanceToNPC / MaxConfrontationRange)`

| 变量 | 定义 | 默认值 |
|------|------|-------|
| `DistanceToNPC` | 玩家到 NPC 的欧几里得距离 | 动态计算 |
| `MaxConfrontationRange` | 最大对峙触发距离 | 5米（见 Tuning Knobs `MaxConfrontationRange`） |

**示例计算**：
- 距离 1 米：`ProximityScore = 1.0 - (1/5) = 0.8` → 高威胁，对峙触发 SUSPECT
- 距离 3 米：`ProximityScore = 1.0 - (3/5) = 0.4` → 中等威胁
- 距离 5 米：`ProximityScore = 1.0 - (5/5) = 0.0` → 低于对峙阈值

**用途**：
- 对峙触发 SUSPECT 的条件之一：`ProximityScore > ProximityScoreThreshold`（默认 0.7）

## Edge Cases

### 边缘情况1：多个 NPC 同时感知到玩家

**问题**：多个 NPC 同时看到玩家，Alert State 如何计算？

**处理方案**：
- Alert State 对每个 NPC 独立计算
- 派系感知共享会在同派系 NPC 之间同步，但每个 NPC 有独立的"已接收共享"列表，防止重复接收同一警报
- 当多个 NPC 同时感知到玩家时，系统选择 ThreatScore 最高的 NPC 作为"主导威胁"，其他 NPC 参考主导威胁的判断

---

### 边缘情况2：NPC 在状态转换瞬间受到致命伤害

**问题**：NPC 在从 SUSPECT → SEARCH 的一瞬间被击杀，Alert State 如何处理？

**处理方案**：
- 死亡优先级最高。任何 Lethal 伤害，无论 NPC 当前处于何种状态，立即转为 DEAD
- **锁喉机制 (Choke-Out Mechanism)**：NPC 在死亡前会尝试广播 SharedAlert，但如果玩家在广播完成前完成击杀，警报不会发出
  - 这是 "锁喉" 机制——玩家可以用快速击杀来阻止警报扩散
  - 广播完成时间由 `ChokeOutWindow` 控制（默认 1.0 秒）
  - 玩家必须在 NPC 完成广播前完成击杀，才能阻止警报扩散

**范围说明 (MVP vs Alpha)**：
- 锁喉机制属于 **Alpha 阶段** 实现内容
- **MVP 简化方案**：NPC 死亡后**不等待广播完成**，直接死亡，警报在下一帧根据派系共享规则正常扩散
- 理由：锁喉机制的帧级精确判定实现成本高，建议在 Alpha 阶段有更多时间调试时再实现

---

### 边缘情况3：NPC 在状态转换动画期间受到伤害

**问题**：NPC 正在从 SUSPECT 转向 SEARCH 时被攻击——动画会被打断吗？

**处理方案**：
- 所有 NPC 的状态转换动画可被打断
- Lethal 伤害：立即终止动画，进入 DEAD
- Blunt 伤害：立即终止动画，累积硬直/倒地判定

---

### 边缘情况4：多条件同时满足

**问题**：NPC 同时满足 "SUSPECT → UNDETECTED"（3秒无异常）和 "SUSPECT → SEARCH"（看到玩家）——优先级是什么？

**处理方案**：
- **发现证据的条件立即覆盖衰减条件**。任何 "发现证据" 的条件会立即覆盖 "无异常衰减"
- 同理，ALERT 发现条件 > ALERT 衰减条件，COMBAT 发现条件 > COMBAT 衰减条件
- **注意**：这与 ThreatScore 优先级不冲突。ThreatScore 用于在**多个威胁**之间选择主要响应目标；Alert State 转移条件用于在**同一威胁的不同状态**之间决定升降

---

### 边缘情况5：派系感知共享的循环依赖

**问题**：A 通知 B，B 通知 C，C 又通知 A——如何防止无限循环？

**处理方案**：
- 每个 NPC 维护独立的 "已接收共享" 列表（来源 NPC + 时间戳）
- 同一来源的 SharedAlert 在 5 秒内只处理一次
- 超过 5 秒的旧警报自动过期

---

### 边缘情况6：NPC 发现同伴处于非 FREE 状态

**问题**：NPC 发现同伴处于 TIED/UNCONSCIOUS/DEAD 状态——Alert State 如何变化？

**处理方案**：
- 发现同伴 **UNCONSCIOUS** 或 **TIED**：立即进入 **ALERT** 状态（确认有威胁，但未确认是玩家）
- 发现同伴 **DEAD**：立即进入 **SEARCH** 状态（确认有威胁存在）
- 这不受派系感知共享延迟影响，是即时反应

**跨系统说明**：Gritty Takedowns 系统在处决被第三方目击时，会发送 `ExecutionWitnessed(npc_id, witness_npc_id)` 事件。本边缘情况6描述的是 NPC "主动发现"同伴异常的处理；当收到 `ExecutionWitnessed` 事件时，视为"被动通知"，处理逻辑同上（根据 witness 的当前状态决定行为）。

---

### 边缘情况7：NPC 唤醒时玩家就在旁边

**问题**：玩家在昏迷 NPC 旁边等待，NPC 唤醒后立即感知到玩家——进入什么 Alert State？

**处理方案**：
- 唤醒后，NPC 首先评估玩家距离
  - **距离 < 2 米**：立即进入 **SEARCH**（威胁迫在眉睫）
  - **距离 2~5 米**：进入 **SUSPECT**（感到不安）
  - **距离 > 5 米**：按正常感知评分计算
- 如果 NPC 唤醒前已经有降级后的 Alert State（如 COMBAT→ALERT），优先恢复该状态再叠加距离判定

---

### 边缘情况8：allegiance 变化后立即进行交互

**问题**：NPC 正在 COMBAT 中（allegiance=-50），玩家成功贿赂后立即变为 +30——它会立即停手吗？

**处理方案**：
- **不立即停手**。allegiance 变化在下一个行为循环评估时生效
- 当前行为循环（通常是 1~2 秒）继续执行
- 如果新 allegiance 低于 -30，行为循环结束后保持敌对；如果高于 -30，下一个循环转为友好行为

---

### 边缘情况9：贿赂/威胁失败的处理

**问题**：玩家资源不足导致贿赂失败，或威胁被 NPC 识破——后果是什么？

**处理方案**：
- **贿赂失败**：allegiance -5（NPC 感到被轻视），NPC 行为不变
- **威胁识破**：allegiance -15，NPC 有 50% 概率进入 SUSPECT 状态（如果玩家在视野内）

---

### 边缘情况10：UNCONSCIOUS/TIED NPC 不参与派系感知

**问题**：昏迷/被捆绑的 NPC 是否继续参与派系感知网络？

**处理方案**：
- **不参与**。UNCONSCIOUS 和 TIED 状态的 NPC 不发送也不接收 SharedAlert
- 这是核心战术选择：击晕 NPC 可以切断感知链，让派系成员陷入 "信息孤岛"
- 但这也意味着昏迷的 NPC 醒来后，派系可能已经因 "失联" 而进入更高警戒状态

---

### 边缘情况11：派系关系动态变化时 pending 共享的处理

**问题**：派系关系从敌对变为友好（或反之）时，pending 中的 SharedAlert 是否立即生效/作废？

**处理方案**：
- SharedAlert 发送时的派系关系决定其是否被接收
- 发送后，即使派系关系变化，已发送的共享**立即作废**
- 新派系关系只影响后续的共享

---

### 边缘情况12：对峙/强制交互的威胁程度分级

**问题**：玩家接近 NPC 进行威胁时，不同程度的威胁如何量化并产生不同后果？

**处理方案**：

**对峙（CONFRONTATION）**：
- 玩家从一定距离外接近，保持在 NPC 视野边缘
- 触发 SUSPECT 的条件（AND 逻辑）：
  - `ProximityScore > 0.7`（距离极近）
  - NPC 处于 UNDETECTED
  - `Random() < 0.3 + TimeScore * 0.2`

**TimeScore 定义**：
- `TimeScore = Clamp(TimeInConfrontation / MaxConfrontationTime, 0.0, 1.0)`
- `TimeInConfrontation`：玩家保持在对峙距离内的累计时间（秒）
- `MaxConfrontationTime`：最大对峙时间阈值，默认 3.0 秒
- 含义：玩家对峙越久，NPC 越容易产生怀疑（3秒后概率 = 0.3 + 1.0 × 0.2 = 0.5）
- 分支：
  - **无视**：Allegiance < -30 或 Bravery > 5 → 继续工作
  - **观察**：→ 暂停动作，仔细观察
  - **试探**（需要玩家输入对话选项）：Allegiance > +30 → 发出对话
  - **回避**：Courage < 3 → 慢慢远离，30% 概率低声告知同派系

**强制交互（FORCED_INTERACTION）**：
- 玩家直接对 NPC 执行威胁交互
- 立即触发 SUSPECT；满足条件时升级到 SEARCH/ALERT
- 分支（基于 allegiance 和 Bravery 纯属性判定）：
  - **屈服**：Allegiance > +20 或 Courage < 2 → 举手/后退
  - **抵抗**：Allegiance < -30 或 Bravery > 7 → 反击/呼救
  - **沟通/求饶**：Allegiance -30~+30，Courage 2-7 → 说出台词
  - **欺骗/反击**：NPC 有 "说谎者" vulnerability → 假装屈服，突袭反击

**派系感知共享触发**：
| 分支结果 | 威胁等级 | 共享延迟 | 内容降级 |
|----------|---------|---------|---------|
| 对峙-无视/观察 | 无 | — | — |
| 对峙-试探 | 极低 | 6秒 | 降两级 |
| 对峙-回避 | 低 | 4秒 | 降一级 |
| 强制-屈服 | 中 | 2秒 | 无 |
| 强制-抵抗 | 高 | 1秒 | 无 |
| 强制-沟通 | 中 | 3秒 | 降一级 |
| 强制-欺骗 | 不共享 | — | — |

## Dependencies

### 上游依赖 (依赖谁)

| 系统 | 接口类型 | 依赖性质 | 说明 |
|------|---------|---------|------|
| **玩家控制器 (Player Controller)** | 数据读取 | 硬依赖 | 读取玩家世界坐标、运动状态、移动速度，用于计算感知距离和方向 |
| **视野与监听系统 (LOS & Eavesdropping)** | 事件接收 | 硬依赖 | 接收 `PlayerSpottedEvent` 事件触发 Alert State 变化；NPC 对话文本作为 LOS 系统的输入源 |
| **脆弱度与伤害系统 (Health & Lethality)** | 内部接口 | 硬依赖 | 通过内部接口接收 Health 系统的伤害结果通知，触发 World State 更新；提供 `QueryState(NPC_ID)` 查询接口 |

### 下游依赖 (谁依赖本系统)

| 系统 | 接口类型 | 依赖性质 | 说明 |
|------|---------|---------|------|
| **视野与监听系统 (LOS & Eavesdropping)** | 数据发送 | 硬依赖 | 提供NPC对话文本供LOS系统提取关键词 |
| **线索与日志系统 (Clue & Journal)** | 数据发送 | 软依赖 | 接收 NPC 的 `knowledge`（掌握的线索列表）；NPC 死亡可能导致线索永久丢失 |
| **沉重处决系统 (Gritty Takedowns)** | 查询接口 + 事件订阅 | 软依赖 | 提供 `QueryAlertState(NPC_ID)` 和 `QueryState(NPC_ID)`；订阅 `InteractionEvent`（威胁/击杀/捆绑等）触发 NPC 状态变化 |
| **理智/愤怒系统 (Sanity/Rage)** | 数据发送 | 软依赖 | 接收被击杀 NPC 的 `Tag`（Enemy/Accomplice/Victim）以计算理智增减 |
| **环境交互系统 (Environment Interaction)** | 事件接收 | 软依赖 | 接收环境交互触发的事件（如某扇门被打开），NPC 可能感知到并触发 SUSPECT |

### 依赖关系矩阵

```
玩家控制器 ──────► NPC AI 系统
     │                    │
     │                    ├──► 线索与日志系统
     │                    ├──► 沉重处决系统
     │                    ├──► 理智/愤怒系统
     ▼                    └──► 环境交互系统
LOS 系统 ──────► 　　　
     │
Health 系统 ───► 　　　
```

### 关键设计约束

1. **NPC AI 系统不拥有生命值/状态**：World State 的实际状态由 Health 系统管理，NPC AI 系统通过查询接口获取
2. **NPC AI 系统不直接处理伤害计算**：伤害由 Health 系统计算后，以事件形式通知 NPC AI 系统
3. **NPC AI 系统不直接播放动画**：状态转换通过事件广播，由 gameplay-programmer 实现具体的动画逻辑

### NPC 属性生成 (Bravery / Courage)

每个 NPC 实例化时，从预设模板或随机分布中获取 Bravery 和 Courage 值：

| 属性 | 范围 | 生成方式 | 影响 |
|------|------|---------|------|
| `Bravery` | 1~10 | 正态分布，均值5，标准差2 | 影响抵抗/屈服判定。Bravery × 10 = 抵抗所需 allegiance 阈值 |
| `Courage` | 1~10 | 正态分布，均值5，标准差2 | 影响回避/逃跑行为。Courage < 3 = 容易触发回避 |

**NPC 类型模板示例**（详细设计见 OQ-3）：

| NPC类型 | Bravery 均值 | Courage 均值 | 典型行为 |
|---------|-------------|-------------|---------|
| 帮派核心成员 | 7 | 6 | 抵抗倾向高，不易逃跑 |
| 普通守卫 | 5 | 5 | 中等抵抗力 |
| 胆小职员 | 3 | 4 | 容易屈服，容易逃跑或求饶 |
| 无辜平民 | 2 | 3 | 极易屈服，极易逃跑或求饶 |

**属性影响的行为判定**：

| 判定类型 | 公式 | 示例 |
|---------|------|------|
| 抵抗判定 | `Allegiance < -30 OR Bravery × 10 > CurrentAllegianceChange` | Bravery=7 时，需要 allegiance < -70 才能抵抗 |
| 屈服判定 | `Allegiance > +20 OR Courage < 2` | Courage=2 时，任何 allegiance 都可能屈服 |
| 回避判定 | `Courage < 3` → 触发回避行为 | 低勇气 NPC 会主动远离玩家 |

## Tuning Knobs

### 警觉状态 (Alert State) 相关

| 参数名 | 默认值 | 安全范围 | 极端行为 | 说明 |
|--------|-------|---------|---------|------|
| `SuspectHoldTime` | 3 秒 | 1~8 秒 | 过短=频繁波动，过长=过于迟钝 | 保持 SUSPECT 无刺激则开始衰减 |
| `SuspectDecayTime` | 2 秒 | 1~5 秒 | 过短=快速恢复正常，过长=长时间紧张 | 从 SUSPECT 衰减回 UNDETECTED |
| `SearchTime` | 8 秒 | 4~15 秒 | 过短=太快放弃搜索，过长=玩家难以甩掉 | SEARCH 状态持续时间 |
| `SearchDecayTime` | 3 秒 | 1~8 秒 | 过短=搜索不彻底，过长=过度搜索 | 从 SEARCH 衰减回 UNDETECTED |
| `AlertHoldTime` | 5 秒 | 2~10 秒 | 过短=快速撤退，过长=长时间警戒 | ALERT 状态在看不到玩家时保持 |
| `AlertEscapeTime` | 2 秒 | 1~5 秒 | 过短=频繁切换，过长=玩家难以脱离 | ALERT 状态下玩家逃离后转入 SEARCH |
| `CombatEscapeTime` | 10 秒 | 5~20 秒 | 过短=战斗节奏太快，过长=玩家被困住 | COMBAT 状态下玩家逃离后转入 SEARCH |

---

### 感知系统相关

| 参数名 | 默认值 | 安全范围 | 极端行为 | 说明 |
|--------|-------|---------|---------|------|
| `MaxPerceptionRange` | 15 米 | 8~30 米 | 过短=NPC 很迟钝，过长=无处可藏 | NPC 最大感知距离 |
| `MaxConfrontationRange` | 5 米 | 3~10 米 | 过短=难以对峙，过长=太容易触发 | 对峙威胁的触发距离 |
| `SharedAlertBaseDelay` | 2 秒 | 1~5 秒 | 过短=警报传播太快，过长=玩家有时间差 | 同派系感知共享基础延迟 |
| `SharedAlertSpeed` | 10 米/秒 | 5~20 米/秒 | 过短=跨区域警报，过长=本地化感知 | 感知共享传递速度 |
| `PerceptionSuspectThreshold` | 0.3 | 0.2~0.5 | 过低=太容易怀疑，过高=太迟钝 | 触发 SUSPECT 的感知评分阈值 |
| `PerceptionSearchThreshold` | 0.6 | 0.4~0.8 | 过低=太容易搜索，过高=难以触发 | 触发 SEARCH 的感知评分阈值 |
| `PerceptionAlertThreshold` | 0.8 | 0.6~1.0 | 过低=太容易警报，过高=难以触发 | 触发 ALERT 的感知评分阈值 |

---

### 派系感知共享相关

| 参数名 | 默认值 | 安全范围 | 极端行为 | 说明 |
|--------|-------|---------|---------|------|
| `AllianceShareProbability` | 50% | 30~70% | 过低=联盟感知弱，过高=联盟感知过强 | 联盟派系感知共享概率 |
| `AllianceShareDelayMultiplier` | 2x | 1.5~3x | 过低=联盟反应快，过高=联盟反应慢 | 联盟派系共享延迟倍率 |
| `NeutralShareProbability` | 10% | 5~20% | 过低=中立感知弱，过高=世界太透明 | 中立派系感知共享概率 |
| `NeutralShareDelayMultiplier` | 3x | 2~5x | 同上 | 中立派系共享延迟倍率 |
| `SharedAlertExpireTime` | 5 秒 | 3~10 秒 | 过短=频繁重复，过长=旧警报堆积 | SharedAlert 重复接收过滤时间 |

---

### 勇气值 (Bravery) 相关

| 参数名 | 默认值 | 安全范围 | 极端行为 | 说明 |
|--------|-------|---------|---------|------|
| `BraveryResistThreshold` | Bravery * 10 | — | — | NPC 反击所需 allegiance 阈值 |
| `CourageFleeThreshold` | 3 | 1~5 | 过低=太容易逃跑，过高=从不逃跑 | 触发回避行为的 Courage 上限 |

---

### 威胁交互相关

| 参数名 | 默认值 | 安全范围 | 极端行为 | 说明 |
|--------|-------|---------|---------|------|
| `ConfrontationBaseThreat` | -8 | -15~-3 | 过低=威胁太轻，过高=威胁太重 | 对峙基础 allegiance 变化 |
| `ForcedInteractionBaseThreat` | -25 | -40~-10 | 过低=强制交互不够有效，过高=太容易屈服 | 强制交互基础 allegiance 变化 |
| `ProximityScoreThreshold` | 0.7 | 0.5~0.9 | 过低=太容易触发 SUSPECT，过高=难以触发 | 对峙触发 SUSPECT 的距离阈值 |
| `SurrenderAllegianceThreshold` | +20 | +10~+30 | 过低=太容易屈服，过高=太难屈服 | 屈服所需的最低 allegiance |
| `ResistBraveryThreshold` | 7 | 5~9 | 过低=太容易抵抗，过高=太难抵抗 | 抵抗所需的最低 Bravery |
| `MaxConfrontationTime` | 3.0 秒 | 2.0~6.0 秒 | 过短=太快产生怀疑，过长=难以触发试探 | TimeScore 计算的最大对峙时间阈值 |

---

### 世界状态 (World State) 相关

| 参数名 | 默认值 | 安全范围 | 极端行为 | 说明 |
|--------|-------|---------|---------|------|
| `DownedRecoveryTime` | 15 秒 | 10~30 秒 | 过短=太快苏醒，过长=玩家有太多时间 | UNCONSCIOUS 后自动苏醒时间 |
| `AlertStateDemotionOnWake` | 1 级 | — | — | 唤醒时 Alert State 降级级数 |
| `MaxUnconsciousDepth` | 3 | 2~5 | 过少=太容易苏醒，过多=几乎无法苏醒 | 昏迷最大累积层数 |

### 锁喉机制 (Choke-Out Mechanism)

| 参数名 | 默认值 | 安全范围 | 极端行为 | 说明 |
|--------|-------|---------|---------|------|
| `ChokeOutWindow` | 1.0 秒 | 0.5~2.0 秒 | 过短=玩家几乎总能阻止警报，过长=无法阻止 | 玩家必须在 NPC 广播 SharedAlert 完成前完成击杀，才能阻止警报扩散 |

**锁喉机制时序精确定义**：
- T+0.0s：NPC 进入 ALERT 状态，广播计时开始
- T+0.0s：NPC 立即向调度器注册延迟广播任务（延迟=派系共享延迟）
- T+ChokeOutWindow：锁喉窗口关闭。如果玩家在此之前完成击杀，广播任务被取消
- T+派系延迟（如未被击杀）：广播任务执行，警报扩散

## Visual/Audio Requirements

### 视觉反馈

| NPC 状态 | 头顶标识颜色 | 行为视觉表现 |
|---------|------------|------------|
| **UNDETECTED** | 灰色 "?" | NPC 正常巡逻/工作，无特殊标识 |
| **SUSPECT** | 黄色 "?" 闪烁 | NPC 暂停动作，头部频繁转动扫视 |
| **SEARCH** | 橙色 "!" | NPC 沿路径移动，表情警觉，手持武器戒备 |
| **ALERT** | 红色 "!" + 感叹号图标 | NPC 大声呼叫，武器举起，向同伴示意 |
| **ESCAPE** | 红色 "!!" | NPC 向出口奔跑，表情惊恐 |
| **COMBAT** | 红色 "!!" + 准星 | NPC 射击/追击，动作激烈 |

### NPC 头顶标识系统

- **身份标签**（通过 LOS 监听获得）：恶徒=红，帮凶=黄，无辜者=绿
- **身份标签**优先级高于警觉状态：即一个"已标记的恶徒"在 UNDETECTED 时显示红色标签

### 派系感知共享的视觉表现

- 当 NPC 发送 SharedAlert 时，向共享方向的 NPC 头部出现**虚线箭头**动画
- 接收方 NPC 收到共享后，短暂出现**问号气泡**

### 威胁交互的视觉反馈

| 交互类型 | 视觉表现 |
|---------|---------|
| **对峙** | 玩家靠近时，NPC 头顶出现**虚线圆圈**（警戒范围提示） |
| **强制交互** | 玩家进入交互范围时，NPC 头顶出现**红色警告三角** |
| **屈服** | NPC 举手，角色颜色变灰白 |
| **抵抗** | NPC 举起武器，红色闪烁 |
| **沟通/求饶** | NPC 双手颤抖，台词气泡出现 |

---

### 听觉反馈

| NPC 状态 | 音效要求 |
|---------|---------|
| **UNDETECTED** | 正常环境音：脚步声、呼吸声、对话声 |
| **SUSPECT** | 警觉音效：NPC 暂停对话，发出"嗯？"、"谁在那里？"等语音 |
| **SEARCH** | 搜索音效：NPC 低声呼叫我同伴、吹哨、脚步声加快 |
| **ALERT** | 警报音效：NPC 大喊"发现入侵者！"、"所有人警戒！"、呼叫增援 |
| **ESCAPE** | 逃跑音效：NPC 尖叫、惊恐喊叫、奔跑脚步声 |
| **COMBAT** | 战斗音效：武器开火、近战打击、呼救声 |

### 威胁交互的听觉反馈

| 交互类型 | 音效要求 |
|---------|---------|
| **对峙** | 低沉的环境音，玩家脚步声被压低 |
| **强制交互** | 紧张音效：心跳声、NPC 紧张呼吸 |
| **屈服** | NPC 求饶语音、放下武器音效 |
| **抵抗** | NPC 反击呐喊、武器拔出音效 |
| **沟通/求饶** | NPC 颤抖语音（"别杀我"、"我什么都说"） |

## UI Requirements

### NPC 头顶标识

| 元素 | 位置 | 显示内容 | 优先级 |
|------|------|---------|--------|
| **身份标签** | NPC 头顶 | 恶徒(红)/帮凶(黄)/无辜者(绿)/未知(灰?) | 高 |
| **警觉状态** | NPC 头顶 | UNDETECTED/SUSPECT/SEARCH/ALERT/ESCAPE/COMBAT 的视觉图标 | 高 |
| **派系标识** | NPC 头顶 | 派系图标（如帮派徽章） | 中 |

### 玩家 HUD

| 元素 | 位置 | 显示内容 | 触发条件 |
|------|------|---------|---------|
| **对峙范围提示** | 玩家准星周围 | 虚线圆圈标识对峙触发范围 | 玩家靠近 NPC |
| **交互提示** | 屏幕中央 | "对峙/威胁/搜身/审问" 等可用交互图标 | 可执行交互时 |
| **威胁程度指示** | 交互提示上方 | 低(黄)/中(橙)/高(红) 三级 | 执行威胁交互时 |
| **NPC 头顶标签** | NPC 头顶 | 实时显示 NPC 的 Tag、派系、警觉状态 | 始终显示 |

### 对话选择 UI（对峙-试探分支）

| 元素 | 位置 | 显示内容 | 说明 |
|------|------|---------|------|
| **对话选项** | 屏幕下方 | 2-4 个玩家可选择的对话选项 | 玩家触发试探分支时 |
| **NPC 台词气泡** | NPC 头顶 | 当前 NPC 说的话 | 对话进行中 |
| **选项后果预览** | 选项旁边 | 可选的简化后果提示 | 专家模式可开启 |

### 派系感知可视化（可选）

| 元素 | 位置 | 显示内容 | 说明 |
|------|------|---------|------|
| **感知网络图** | 暂停菜单 | 显示派系关系和感知连接 | 可选开启 |
| **警报传播路径** | 游戏画面 | SharedAlert 传播路径的淡出箭头 | 调试用 |

### UI 不显示内容

- **不显示 NPC 的精确 allegiance 数值**：玩家通过 NPC 行为和对话来感知态度
- **不显示 NPC 的 Bravery/Courage 数值**：作为隐藏属性影响 NPC 反应
- **不显示感知范围圆锥**：保持 LOS 系统的视觉隐蔽性

## Acceptance Criteria

### 基础功能测试

| # | 测试条件 | 验证方法 |
|---|---------|---------|
| AC-1 | NPC 处于 UNDETECTED 状态时，玩家在视野外移动不会触发任何反应 | 在 NPC 视野外移动，验证 NPC 仍处于 UNDETECTED |
| AC-2 | NPC 在 SUSPECT 状态持续 3 秒无刺激后，衰减回 UNDETECTED | 触发 SUSPECT，计时 3 秒后验证状态 |
| AC-3 | NPC 看到玩家轮廓后，从 SUSPECT 进入 SEARCH | NPC 看到玩家身影，验证状态转换 |
| AC-4 | NPC 发现同伴尸体，从 SEARCH 进入 ALERT | 手动制造同伴死亡事件，验证 NPC 反应 |
| AC-5 | 玩家进入攻击范围，NPC 从 ALERT 进入 COMBAT | 玩家接近至攻击范围，验证 NPC 主动攻击 |

---

### 派系感知共享测试

| # | 测试条件 | 验证方法 |
|---|---------|---------|
| AC-6 | 同派系 NPC 发现威胁，2 秒后邻近同派系 NPC 收到警报 | 触发 NPC-A 进入 ALERT，计时验证 NPC-B 在 2 秒内进入 ALERT |
| AC-7 | 联盟派系 NPC 收到警报概率约 50%，延迟翻倍 | 多次测试，统计联盟派系接收率和延迟 |
| AC-8 | 敌对派系 NPC 不会收到警报 | 验证敌对派系 NPC 始终不进入警戒状态 |
| AC-9 | UNCONSCIOUS/TIED NPC 不参与派系感知 | 昏迷 NPC 不发送也不接收 SharedAlert |

---

### 世界状态转换测试

| # | 测试条件 | 验证方法 |
|---|---------|---------|
| AC-10 | NPC 受到 Lethal 伤害立即死亡 | 对 NPC 施加 Lethal 伤害，验证立即 DEAD |
| AC-11 | NPC 受到 Blunt 伤害进入 UNCONSCIOUS | 对 NPC 施加 Blunt 伤害，验证 UNCONSCIOUS |
| AC-12 | 昏迷 NPC 15 秒后自动苏醒 | 触发 UNCONSCIOUS，计时 15 秒后验证苏醒 |
| AC-13 | 昏迷 NPC 唤醒后 Alert State 降一级 | 在 COMBAT 状态下击晕，唤醒后验证是 ALERT |

---

### 威胁交互测试

| # | 测试条件 | 验证方法 |
|---|---------|---------|
| AC-14 | 对峙不自动触发 Alert State 变化 | 玩家接近但不进入交互范围，验证 NPC 保持 UNDETECTED |
| AC-15 | 强制交互立即触发 SUSPECT | 执行强制交互，验证 NPC 立即进入 SUSPECT |
| AC-16 | Bravery=8 的 NPC 在 allegiance=-20 时会抵抗 | 设置正确参数，执行强制交互，验证抵抗行为 |
| AC-17 | Bravery=2 的 NPC 在 allegiance=+10 时会屈服 | 设置正确参数，执行强制交互，验证屈服行为 |
| AC-18 | 屈服分支触发派系感知共享（2 秒延迟） | 触发屈服，验证邻近同派系 NPC 在 2 秒后进入警戒 |

---

### 锁定机制测试

| # | 测试条件 | 验证方法 |
|---|---------|---------|
| AC-19 | 玩家在 NPC 广播前完成击杀，警报不发出 | 在 SharedAlert 广播完成前击杀 NPC，验证周围 NPC 未进入警戒 |
| AC-20 | 玩家在 NPC 广播完成后完成击杀，警报正常发出 | 等待 1.5 秒后击杀，验证周围 NPC 进入 ALERT |

---

### 边缘情况测试

| # | 测试条件 | 验证方法 |
|---|---------|---------|
| AC-21 | 多个 NPC 同时感知玩家，ThreatScore 最高的主导 | 多 NPC 同时看到玩家，验证行为协调性 |
| AC-22 | NPC 唤醒时玩家在 2 米内，立即进入 SEARCH | 玩家在昏迷 NPC 旁等待苏醒，验证立即 SEARCH |
| AC-23 | COMBAT 中的 NPC 被贿赂后，下个循环转为友好 | 在 COMBAT 中贿赂，验证当前循环结束后行为变化 |
| AC-24 | 派系关系从敌对变为友好后，pending 警报作废 | 改变派系关系，验证 pending 警报不生效 |

---

### 性能预算

| 指标 | 预算 | 说明 |
|------|------|------|
| 单帧 NPC 状态更新耗时 | < 2ms | 100 个 NPC 同时更新的目标 |
| 派系感知广播传播时间 | < 16ms | 8 层派系链路的最大延迟 |
| 状态机转换响应时间 | < 1 帧 | Alert State 变化立即响应 |

## Open Questions

| # | 问题 | 负责人 | 目标日期 | 说明 |
|---|------|--------|---------|------|
| OQ-1 | **派系体系的完整定义** | 游戏设计师 | TBD | 尚未定义具体的派系列表（如青龙帮、军阀等）、派系之间的关系（敌对/联盟/中立）、以及派系之间的利益矛盾设计 |
| OQ-2 | **派系利益矛盾的具体机制** | 游戏设计师 | TBD | 如何利用派系间争斗达到获取线索或过关条件？需要设计具体的触发条件和结果链 |
| OQ-3 | ~~**Bravery/Courage 属性在 NPC 生成时的分布**~~ | ~~系统设计师~~ | ~~TBD~~ | ✅ **已解决**：Bravery/Courage 使用正态分布（均值5，标准差2），辅以 NPC 类型模板预设值。详见第565-590行「NPC 属性生成」章节。 |
| OQ-4 | **NPC 的 vulnerability 如何被玩家发现** | 游戏设计师 | TBD | 玩家如何知道 NPC 的弱点？是通过监听对话、搜身、还是审问？ |
| OQ-5 | **NPC 醒来后的行为脚本** | AI 程序员 | TBD | 昏迷 NPC 苏醒后，除了 Alert State 降级外，是否有特定的行为脚本（如去查看同伴状态）？ |
| OQ-6 | **"锁喉"机制的时序精确性** | 系统设计师 | TBD | 玩家必须在 1 秒内完成击杀才能阻止广播——这个时序是否需要更精确的帧级判定？ |
| OQ-7 | **对话选项的后果预览程度** | UX 设计师 | TBD | 试探分支中，对话选项的后果应该预览多少？完全不预览（高风险）还是部分预览（中等风险）？ |
| OQ-8 | **NPC 转化（Convert）机制** | 游戏设计师 | TBD | 玩家将 NPC 转化为"线人"后，该 NPC 的后续行为如何定义？是一次性帮助还是持续配合？ |
| OQ-9 | **环境作为 NPC 行为触发器** | 关卡设计师 | TBD | 环境交互（如触发警报器）如何与 NPC AI 联动？是否需要定义环境事件到 NPC 行为的映射表？ |
| OQ-10 | **NPC 数量上限与性能** | AI 程序员 | TBD | 单个区域最多允许多少 NPC 同时运行状态机？需要性能测试后确定 |
| OQ-11（跨系统） | ~~**AlertStateChangedEvent 事件定义**~~ | ~~AI 程序员~~ | ~~下次对齐会议~~ | ✅ **已解决**：采用事件广播方案。已定义 `AlertStateChangedEvent(NPC_ID, old_state, new_state, trigger)` 事件，详见「事件接口定义」章节。Gritty Takedowns 订阅此事件替代轮询。 |

---

### 已解决的设计决策（供参考）

| 决策项 | 最终方案 | 决定日期 |
|--------|---------|---------|
| UNCONSCIOUS NPC 是否参与感知 | 不参与（切断感知链=有效战术） | 2026-04-05 |
| 死亡时 SharedAlert 原子性 | 可被打断（锁喉机制） | 2026-04-05 |
| 威胁交互分级 | 对峙 vs 强制交互（威胁程度不同） | 2026-04-05 |
| 勇气值 (Bravery) | 引入，影响抵抗/屈服判定 | 2026-04-05 |
| 对峙试探分支 | 需要玩家选择对话选项 | 2026-04-05 |
| 屈服/抵抗判定 | 纯属性判定（无 QTE） | 2026-04-05 |
| Alert State 降级规则 | 唤醒时降一级 | 2026-04-05 |
