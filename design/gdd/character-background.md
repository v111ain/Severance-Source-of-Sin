# 主角背景角色系统 (Character Background System)

> **Status**: Approved (Post-Review Revision)
> **Author**: [user + agents]
> **Last Updated**: 2026-04-10
> **Revision Notes**: 2026-04-10 设计审查修复第六轮：
> - P0: 修复态度矩阵雇佣兵vs灰烬团+25→+35，与边缘情况1首次遭遇加成表（+20）统一
> - P0: 补充边缘情况1叠加计算说明，明确 AttitudeModifier_Mercenary(-5) 也参与计算
> - P1: 澄清 KillSource 枚举定义归属（由 Gritty Takedowns 系统定义），补充实现说明
> - P2: 在 Formulas 章节新增公式4（首次遭遇态度计算），形式化叠加逻辑
> - **P1: 修复接口归属和数据流描述歧义**：明确 QueryOldAcquaintanceBonus 为 Pull 模式，补充 IFirstEncounterBonusProvider 接口调用说明
> - **P2: 补充态度矩阵脚注**，明确为首次遭遇时的初始态度
> - **P2: 补充 PerceptionModifier_Agent 调参风险提示**，防止特工成为唯一最优选择
> - **P2: 修正 Revision Notes 描述**：删除已过时的"PerceptionModifier_Agent 从 1.25x→1.2x"修正记录（原修正已在之前版本完成）
> - **P3: 降低 PerceptionModifier_Agent 安全范围上限**：从 1.2x 降至 1.15x，防止特工感知优势过强
> - **P1: 补充 FactionModifier 独立定义表**，明确其数据来源为 NPC AI System
> - **P2: 明确"战斗力下降15%"的具体数值影响**：移动速度 -15%，攻击准确率 -10%
> - **P2: 明确"线索分析速度"参数的语义**：同时作用于分析时间和分析成功率
> - **P2: 统一感知系数描述**：感知范围+10%~20%（默认+20%，调参上限1.15x）
> - **P1: 补充 Gritty Takedowns 系统为硬依赖**，明确 IFirstEncounterBonusProvider 接口关系
> **Implements Pillar**: 致命的脆弱感 (Lethal Fragility), 沉重、不洁的暴力 (Gritty Violence)

## Overview

主角背景角色系统是《断绝：罪恶之源》开局玩家创建角色时的核心选择系统。它定义了玩家扮演的"复仇者"身份原型——一个被悲剧驱动的父亲，但这个父亲拥有不同的过去。

系统为三个背景提供差异化的参数修正与固有风格：

- **普通人 (Civilian)**：无特殊训练的失亲父亲，潜行与分析能力强，战斗能力最弱
- **特工 (Agent)**：锈蚀的前情报员，感知与分析能力出众，但内心道德挣扎更剧烈
- **雇佣兵 (Mercenary)**：职业杀手，战斗与环境利用能力最强，隐秘能力受限且更易冲动

背景系统在游戏开局选择后**不可更换**，通过差异化参数修正，为玩家提供三种截然不同的游玩风格。所有背景差异均以**参数形式**体现，不设任何独有技能——相同技能在不同背景下有不同效果数值，但本质相同。背景同时影响 NPC 对玩家的对话态度（敌人会因背景而表现出不同反应），但不影响主线结局——结局由玩家的行为（杀/不杀、拯救/无视）决定。

## Player Fantasy

### 三个背景的情感目标

| 背景 | 核心情感体验 | 参考对标 | 玩家感受描述 |
|------|------------|---------|-------------|
| **普通人** | "我不是战士，我是父亲" | 《最后生还者》乔尔 | 用智慧对抗武力，每一次脱身都是胜利，战斗时恐惧与紧张并存 |
| **特工** | "我曾用这些技能伤害人，现在用它救人" | 《分裂细胞》山姆·费舍尔 | 监听和分析带来满足感，但每一次击杀都在重新撕开旧伤疤 |
| **雇佣兵** | "我为杀戮而生，这次是为了我自己" | 《迈阿密热线》 | 高效、残忍、无情的杀戮机器，但这种力量是有代价的 |

### 背景与游戏支柱的对应

> **注**："高/中/低"表示该背景对各支柱的**契合度**（高 = 最能体现该支柱的体验），而非数值大小。

| 支柱 | 普通人 | 特工 | 雇佣兵 |
|------|--------|------|--------|
| **致命的脆弱感** | 高（战斗系数 0.8x，必须谨慎潜行） | 中（战斗系数 0.95x，有反击但不耐打） | 低（战斗系数 1.2x，可正面对抗但仍会死亡） |
| **沉重、不洁的暴力** | 高（理智惩罚系数 0.9x，心理代价明显） | 最高（理智惩罚系数 0.8x，道德挣扎最剧烈） | 中（理智惩罚系数 1.15x，冲动失控倾向高） |
| **环境即武器** | 低（环境交互系数 0.9x，依赖潜行而非环境） | 中（环境交互系数 1.0x） | 高（环境交互系数 1.2x，充分利用一切） |
| **罪恶的深度** | 高（普通人悲剧最具情感共鸣） | 高（锈蚀特工的道德困境最具戏剧性） | 中（雇佣兵黑暗过去叙事张力较弱） |

## Detailed Design

### BackgroundType 枚举定义

本文档与下游系统交互时使用以下枚举定义：

| 枚举值（代码层） | 中文名称 | 说明 |
|----------------|---------|------|
| `CIVILIAN` | 普通人 | 无特殊训练的失亲父亲 |
| `AGENT` | 特工 | 锈蚀的前情报员 |
| `MERCENARY` | 雇佣兵 | 职业杀手 |

> **接口一致性说明**：所有跨系统接口（如 `QueryOldAcquaintanceBonus`、`GetFirstEncounterBonus`、`DialogTreeConfig.player_background`）均使用上述枚举值。文档其他位置使用中文名称是为了可读性，代码实现时应使用枚举值。

### Core Rules

#### 设计原则

背景差异化由两要素构成：**参数修正（核心）** + **NPC 态度（叙事）**。所有参数修饰幅度控制在 **±25%** 以内，确保风格差异而非强度分级。

**设计约束**：
- 三个背景**不使用任何独有技能**——所有游玩效果均通过参数修正体现
- 同一动作在不同背景下有不同效率，但动作本身相同
- 参数设计服务于游戏支柱：致命的脆弱感、沉重不洁的暴力、环境即武器

---

#### 一、参数修正体系（8 项）

| 参数 | 普通人 | 特工 | 雇佣兵 | 说明 |
|------|:------:|:----:|:------:|------|
| **理智惩罚系数** | 0.9x | 0.8x | 1.15x | 普通人略微习惯；特工习惯暴力惩罚最低；雇佣兵最不适应内心黑暗 |
| **狂暴阈值调整** | +8 | ±0 | -8 | 普通人更难愤怒（更迟钝）；雇佣兵更易失控 |
| **潜行系数** | 1.15x | 1.0x | 0.85x | 普通人最擅长潜行（不起眼）；雇佣兵最不擅长 |
| **战斗系数** | 0.8x | 0.95x | 1.2x | 雇佣兵战斗最强；普通人最弱 |
| **感知系数** | 0.9x | 1.2x | 1.0x | 特工监听/分析范围最大（感知范围+10%~20%，默认+20%，调参上限1.15x）；普通人感知最弱 |
| **环境交互系数** | 0.9x | 1.0x | 1.2x | 雇佣兵最擅长利用环境；普通人最不擅长 |
| **线索分析速度** | 1.1x | 1.3x | 0.85x | 普通人善于观察细节；特工情报分析最强；雇佣兵最弱。作用于分析所需时间（时间越短越好）和分析成功率（成功率越高越好），两者均受此乘数影响 |
| **NPC初始态度修正** | +10 | +15 | -5 | 整体修正值（具体矩阵见下节）；特工因旧识关系整体最高 |

**设计说明**：
- 所有系数基于基准值 1.0x，修饰范围 ±25% 以内
- 参数**仅影响数值效果**，不决定可用动作/技能
- 三个背景在功能上等效（可完成相同动作），差异在于效率与手感

**特工"道德挣扎"的机制与叙事一致性说明**：
- 特工的 `SanityPenaltyMultiplier = 0.8x`（理智惩罚最低）是因为特工**习惯暴力行为本身**——他过去执行过无数任务，击杀行为不会让他感到生理上的不适
- 但特工在叙事层面的"道德挣扎"体现在 **NPC 对话态度**上：锈网旧识会触发特殊对话分支，让特工面对"曾同为刽子手"的道德困境
- 理智惩罚系数与叙事道德挣扎是**两个独立的维度**：一个是心理舒适度（习惯杀戮=惩罚低），一个是道德身份认同（面对旧识=道德困境）
- 这种设计让特工在**数值感受上**比普通人更适应暴力，但在**叙事体验上**有独特的道德冲突场景

---

#### 二、NPC 态度矩阵（叙事差异化）

NPC 对玩家的初始态度根据**玩家背景**和**NPC 类型**计算。

##### 2.1 初始态度矩阵（首次遭遇时）

> **⚠️ 重要说明**：
> - 本表为玩家**首次**与某派系 NPC 遭遇时的初始态度（`HasMetFaction[派系] == false`）
> - 后续遭遇不再应用首次遭遇加成，使用公式3计算
> - 本表中特工 vs 锈网（+40）和雇佣兵 vs 灰烬团（+35）和雇佣兵 vs 凋亡议会（+30）的数值是**含旧识加成的最终叠加值**，而非单纯的背景态度修正。
>
> - 特工 vs 锈网 +40 = `AttitudeModifier_Agent`(+15) + `OldAcquaintanceBonus`(+25) + `FirstEncounterBonus`(+15)
> - 雇佣兵 vs 灰烬团 +35 = `AttitudeModifier_Mercenary`(-5) + `OldAcquaintanceBonus`(+20) + 首次遭遇加成(+20)
> - 雇佣兵 vs 凋亡议会 +30 = `AttitudeModifier_Mercenary`(-5) + `OldAcquaintanceBonus`(+15) + 首次遭遇加成(+15)
>
> 基础态度修正值（不含旧识加成和首次遭遇加成）见 Section 2.3 旧识关系修正表和边缘情况1的首次遭遇加成表。

| NPC 类型 | 普通人 | 特工 | 雇佣兵 |
|---------|:------:|:----:|:------:|
| 黑帮普通成员 | -10（轻视） | -20（警惕） | +10（恐惧） |
| 黑帮小头目 | -30（蔑视） | -25（警惕） | +5（平等） |
| 受害者/线人 | +25（信任） | +10（谨慎信任） | -20（恐惧） |
| 凋亡议会成员 | -40（蔑视） | -30（警惕） | -35（警惕） |
| 锈网成员 | -10（中立） | **+40（信任）** | +5（平等） |
| 灰烬团成员 | -20（警惕） | -5（中立） | **+35（好奇/尊重）** |
| 苍白之手成员 | -15（警惕） | -10（警惕） | -15（警惕） |
| 无声者成员 | +15（好奇） | +20（好奇） | +5（中立） |

**设计说明**：
- **特工 vs 锈网**：特工曾是锈网的线人/合作伙伴，有"旧识"关系
- **雇佣兵 vs 灰烬团**：灰烬团是军阀化私人武装，雇佣兵可能是"同行"；首次遭遇时还有额外的+20首次遭遇加成
- **雇佣兵 vs 凋亡议会**：雇佣兵曾是凋亡议会的雇佣者，首次遭遇时还有额外的+15首次遭遇加成
- **雇佣兵 vs 受害者**：职业杀手形象让受害者本能恐惧
- 整体态度修正值受 **NPC初始态度修正参数** 二次调整（见参数表）

##### 2.2 态度对行为的影响

| 态度 | 对话选项影响 | 行为变化 |
|------|------------|---------|
| 蔑视 | 无说服选项，只能威胁/处决 | 立即呼叫增援（1.5秒内） |
| 警惕 | 质问/对峙选项 | 武器戒备，给你 3 秒解释时间 |
| 中立偏疑 | 解释/请求可用 | 保持距离观察 |
| 好奇/信任 | 所有对话类型可用 | 态度软化，愿意配合 |
| 恐惧 | 敌人可能投降/逃跑 | 移动速度下降 15%，攻击准确率下降 10% |

> **注**："战斗力下降 15%"具体指：移动速度 -15%，近战/远程攻击准确率 -10%。由 Health&Lethality 系统在计算敌人作战能力时应用。

##### 2.3 旧识关系修正

特工/雇佣兵与特定派系存在旧识关系，享有额外态度加成：

| 触发条件 | 背景 | NPC 派系 | 关系性质 | 态度加成 |
|---------|------|---------|---------|:--------:|
| SAME_FACTION | Agent | 锈网 | 前同僚/线人 | +25 |
| MERCE_TO_MERCE | Mercenary | 灰烬团 | 同行 | +20 |
| MERCE_EMPLOYER | Mercenary | 凋亡议会 | 曾是雇主 | +15 |

**旧识对话效果**：可快速获取关键情报，但可能触发道义困境或身份暴露风险（旧识对话分支本身由 DialogTree 系统实现，背景参数不直接控制）。

### States and Transitions

背景系统是**静态系统**（开局选择后不可更换），**无状态机**。

玩家选择背景后，BackgroundType 和 BackgroundParameters 固定传递到各系统，贯穿整个游戏流程。背景本身不持有任何临时状态。

**状态转移图**：无（静态系统）

```
[开局选择] ──背景确定──▶ [游戏中持续生效] ──游戏结束──▶ [无变化]
```

### Interactions with Other Systems

#### 数据流出 (Outputs)

| 目标系统 | 数据内容 | 用途 |
|---------|---------|------|
| **NPC AI系统** | `AttitudeModifier` | NPC 初始态度计算 |
| **NPC AI系统** | `QueryOldAcquaintanceBonus`（Pull 接口） | 旧识关系额外态度加成（NPC AI 系统主动查询，本系统返回） |
| **DialogTree** | `BackgroundType` | 对话选项可见性、旧识对话分支判断 |
| **理智/愤怒系统** | `SanityPenaltyMultiplier` | 理智惩罚修正 |
| **理智/愤怒系统** | `FrenzyThresholdModifier` | 狂暴阈值调整 |

#### 跨系统接口定义

```
CharacterBackground（纯数据输出系统，不接收外部事件）
    │
    ├──► NPC AI System
    │       ├── 提供: AttitudeModifier（推送）
    │       └── 提供: QueryOldAcquaintanceBonus（Pull 接口，NPC AI 系统主动调用）
    │
    ├──► DialogTree
    │       └── 提供: BackgroundType（推送）
    │
    └──► Sanity/Rage Meter
            └── 提供: SanityPenaltyMultiplier, FrenzyThresholdModifier（推送）
```

#### OldAcquaintanceBonus 接口规范

本系统提供 `QueryOldAcquaintanceBonus` 接口供 NPC AI 系统调用：

```csharp
interface IOldAcquaintanceQuery {
    int QueryOldAcquaintanceBonus(NPC_ID npc_id, BackgroundType player_background);
}
```

**接口实现逻辑**：

> **实现方式说明**：本接口通过**静态查表**实现，无复杂业务逻辑。

1. NPC AI 系统调用接口时传入 `npc_id` 和 `player_background`
2. 本系统根据 NPC 所属派系查询旧识关系修正表（见下方返回值查表）
3. 返回对应的态度加成值（无旧识关系则返回 0）

**返回值查表**：

| 触发条件 | 背景 | NPC 派系 | 关系性质 | 返回值 |
|---------|------|---------|---------|:------:|
| `SAME_FACTION` | Agent | 锈网 | 前同僚/线人 | +25 |
| `MERCE_TO_MERCE` | Mercenary | 灰烬团 | 同行 | +20 |
| `MERCE_EMPLOYER` | Mercenary | 凋亡议会 | 曾是雇主 | +15 |
| 无匹配 | 任意 | 任意 | — | 0 |

## Formulas

### 公式1：理智惩罚计算

```
EffectiveSanityPenalty = BaseSanityPenalty × SanityPenaltyMultiplier
```

| 变量 | 值 | 来源 |
|------|---|------|
| `BaseSanityPenalty` | 来自各事件（如击杀 -5~-15） | Sanity/Rage 系统 |
| `SanityPenaltyMultiplier` | 普通人=0.9, 特工=0.8, 雇佣兵=1.15 | Character Background |

**示例**：特工击杀帮凶（基准惩罚 -8）
```
EffectiveSanityPenalty = -8 × 0.8 = -6.4
```
（特工比普通人少受 20% 的心理惩罚，因为他习惯暴力）

### 公式2：狂暴阈值计算

```
EffectiveFrenzyThreshold = BaseFrenzyThreshold + FrenzyThresholdModifier
```

| 变量 | 值 | 来源 |
|------|---|------|
| `BaseFrenzyThreshold` | 70 | Sanity/Rage 系统 |
| `FrenzyThresholdModifier` | 普通人=+8, 特工=±0, 雇佣兵=-8 | Character Background |

**示例**：雇佣兵触发狂暴
```
EffectiveFrenzyThreshold = 70 + (-8) = 62
```
（雇佣兵在 Rage > 62 时进入 FRENZIED，比普通人早 8 点）

### 公式3：NPC 初始态度计算

```
InitialAllegiance = BaseAllegiance + BackgroundAttitudeModifier + FactionModifier + OldAcquaintanceBonus
```

| 变量 | 说明 | 数据来源 |
|------|------|---------|
| `BaseAllegiance` | NPC 基准态度（来自 NPC AI 系统） | NPC AI System |
| `BackgroundAttitudeModifier` | 背景态度修正（见态度矩阵，直接查表取值） | Character Background System |
| `FactionModifier` | 基于 NPC 派系的基准态度修正（见下表） | NPC AI System |
| `OldAcquaintanceBonus` | 旧识关系加成（特工/雇佣兵专属，见旧识关系修正表，最高 +25） | Character Background System |

**FactionModifier 定义表（由 NPC AI System 提供）**：

| NPC 类型 | FactionModifier | 说明 |
|---------|----------------|------|
| 黑帮普通成员 | -10 | 基础敌意 |
| 黑帮小头目 | -20 | 帮派管理层 |
| 受害者/线人 | +15 | 弱势群体 |
| 凋亡议会成员 | -30 | 核心敌人 |
| 锈网成员 | -5 | 灰色地带 |
| 灰烬团成员 | -10 | 武装组织 |
| 苍白之手成员 | -5 | 观望态度 |
| 无声者成员 | +10 | 潜在盟友 |

### 公式4：首次遭遇态度计算（含首次遭遇加成）

```
FirstEncounterAllegiance = BaseAllegiance + BackgroundAttitudeModifier + OldAcquaintanceBonus + FirstEncounterBonus
```

> **适用条件**：仅当 `HasMetFaction[NPC.faction] == false` 时（玩家首次与该派系遭遇）才应用首次遭遇加成。

| 变量 | 说明 |
|------|------|
| `FirstEncounterBonus` | 首次遭遇加成（见边缘情况1首次遭遇加成表） |
| `HasMetFaction[FACTION_ID]` | 标记玩家是否已与该派系发生过对峙/交互（由 Gritty Takedowns 系统持有） |

**首次遭遇加成查表**：

| 背景 | NPC 派系 | FirstEncounterBonus | 叠加后态度总值 |
|------|---------|---------------------|---------------|
| 普通人 | 锈网 | +15 | +15（无旧识加成时） |
| 普通人 | 灰烬团 | +10 | +10（无旧识加成时） |
| **特工** | **锈网** | **+15** | **+40**（旧识+25 + 首次+15） |
| 特工 | 凋亡议会 | +15 | +15（无旧识加成时） |
| **雇佣兵** | **灰烬团** | **+20** | **+35**（旧识+20 + 首次+20） |
| **雇佣兵** | **凋亡议会** | **+15** | **+30**（旧识+15 + 首次+15） |

> **注意**：首次窗口期只触发一次。触发后 `HasMetFaction` 设为 `true`，后续遭遇不再应用首次遭遇加成。

## Edge Cases

### 边缘情况 1：背景选择后首次遭遇特定 NPC

**场景**：玩家选择特工后，首次遭遇锈网成员。

**首次遭遇判定机制**：

| 数据结构 | 类型 | 说明 |
|---------|------|------|
| `HasMetFaction[FACTION_ID]` | bool | 标记玩家是否已与该派系发生过对峙/交互（首次遭遇后设为 true） |

> **实现说明**：
> - 此 flag 由 **Gritty Takedowns 系统** 持有并管理（存储于 Gritty Takedowns 系统内部的玩家状态数据结构中）
> - 设置时机：`ConfrontationStartRequest` 发送时检查 `HasMetFaction[NPC.faction]`，若为 false 则触发首次遭遇加成并设为 true
> - 首次遭遇加成数据由 **Character Background 系统** 提供（见下表）

**首次遭遇加成规则**（由 Character Background 系统定义，供 Gritty Takedowns 查询）：

| 背景 | 派系 | 首次遭遇加成组成 | 触发条件 |
|------|------|----------------|---------|
| 普通人 | 锈网 | +15（信任好奇） | `HasMetFaction[锈网] == false` |
| 普通人 | 灰烬团 | +10（中立好奇） | `HasMetFaction[灰烬团] == false` |
| **特工** | **锈网** | **旧识加成 +25** + **首次遭遇 +15 = +40（信任）** | `HasMetFaction[锈网] == false` |
| 特工 | 凋亡议会 | +15（警惕） | `HasMetFaction[凋亡议会] == false` |
| **雇佣兵** | **灰烬团** | **旧识加成 +20** + **首次遭遇 +20 = +35（好奇/尊重）** | `HasMetFaction[灰烬团] == false` |
| **雇佣兵** | **凋亡议会** | **旧识加成 +15** + **首次遭遇 +15 = +30（旧雇主）** | `HasMetFaction[凋亡议会] == false` |

> **叠加计算说明**：
> - 对于特工 vs 锈网：首次遭遇时总加成为 +40 = OldAcquaintanceBonus(+25，来自旧识关系修正表) + FirstEncounterBonus(+15，来自本表首次遭遇部分)
> - 对于雇佣兵 vs 灰烬团：首次遭遇时总加成为 +35 = AttitudeModifier_Mercenary(-5) + OldAcquaintanceBonus(+20) + FirstEncounterBonus(+20)
> - 对于雇佣兵 vs 凋亡议会：首次遭遇时总加成为 +30 = AttitudeModifier_Mercenary(-5) + OldAcquaintanceBonus(+15) + FirstEncounterBonus(+15)
> - 首次窗口期只触发一次。旧识对话（如特工 vs 锈网）有独立的触发逻辑，在首次加成之后根据对话结果调整态度。

**处理流程**：
1. 玩家发起 `ConfrontationStartRequest(npc_id, player_background)` 到 Gritty Takedowns
2. Gritty Takedowns 向 Character Background 查询 `GetFirstEncounterBonus(player_background, npc_faction)`
3. Gritty Takedowns 查询本地 `HasMetFaction[NPC.faction]` flag
4. 如果 `== false`：
   - 将 Character Background 返回的首次遭遇加成应用到态度计算
   - 将 flag 设为 `true`
5. 如果 `== true`：不应用首次遭遇加成（已过首次窗口期）
6. 后续计算：`InitialAllegiance = BaseAllegiance + BackgroundAttitudeModifier + OldAcquaintanceBonus`

> **注意**：首次遭遇加成与 OldAcquaintanceBonus 在首次遭遇时**叠加计算**，用于增强首次接触的戏剧效果。详见上表叠加计算说明。首次窗口期只触发一次。旧识对话（如特工 vs 锈网）有独立的触发逻辑，在首次加成之后根据对话结果调整态度。

**接口定义**：

```csharp
interface IFirstEncounterBonusProvider {
    /// <summary>
    /// 返回首次遭遇加成原始值。
    /// 注意：此接口不检查 HasMetFaction flag，调用方需先检查 HasMetFaction[npc_faction] == false
    /// 再将返回值应用到态度计算。若 HasMetFaction[npc_faction] == true，应使用公式3而非公式4。
    /// </summary>
    int GetFirstEncounterBonus(BackgroundType player_background, Faction npc_faction);
}
```

> Character Background 系统实现此接口，Gritty Takedowns 在 `ConfrontationStartRequest` 时调用。
> 调用流程：Gritty Takedowns 先检查 `HasMetFaction[NPC.faction]`，若为 false 则调用此接口获取首次遭遇加成并应用；若为 true 则不调用，直接使用公式3计算。

### 边缘情况 2：旧识对话触发后 NPC 死亡

**场景**：特工与锈网成员的旧识对话进行中，该 NPC 死亡

**死亡责任追踪**：

| 死亡类型 | KillSource | 效果 |
|---------|-----------|------|
| 玩家主动攻击击杀 | `DIRECT_KILL` | 收益丢失，理智惩罚正常计算 |
| 玩家环境处决 | `EXECUTION_KILL` | 收益丢失，理智惩罚正常计算 |
| 玩家攻击导致 NPC 倒地后被环境杀死 | `INDIRECT_KILL` | 收益丢失，理智惩罚正常计算 |
| 对话期间第三方 NPC 误杀 | `ACCIDENTAL_KILL` | 收益丢失，不计入理智惩罚 |
| 对话期间玩家被击杀导致 NPC 撤离 | `SELF_DEFENSE` | 收益保留（NPC 自己离开） |

**旧识对话收益定义**：
- 线索：直接获得，不受死亡影响
- 态度变化：立即生效，不受死亡影响
- 后续对话分支解锁：如果 NPC 存活则保留，死亡则失效

### 边缘情况 3：主动击杀 vs 意外死亡定义

**场景**：需要区分玩家行为导致的 NPC 死亡类型

**KillSource 枚举定义**（由 Gritty Takedowns 系统定义和管理）：

| 值 | 场景 | 理智惩罚 | 收益丢失 |
|----|------|---------|---------|
| `DIRECT_KILL` | 玩家直接攻击击杀 | 正常计算 | 是 |
| `EXECUTION_KILL` | 环境处决 | 正常计算 | 是 |
| `INDIRECT_KILL` | 玩家攻击导致倒地后环境致死 | 正常计算 | 是 |
| `ACCIDENTAL_KILL` | 第三方 NPC 误杀 | 不计入 | 是 |
| `SELF_DEFENSE` | NPC 自卫导致玩家死亡/NPC 撤离 | 不适用 | 否（NPC 存活） |

> **实现说明**：KillSource 枚举在实际实现中由 Gritty Takedowns 系统管理（定义于 gritty-takedowns.md 的 `KillSource` 枚举），通过 `KillTagEvent` 事件传递。理智惩罚计算使用 `NPCIdentityType`（ENEMY/ACCOMPLICE/VICTIM/UNKNOWN），而 KillSource 用于追踪更细粒度的死亡责任，用于判断"收益丢失"等边缘情况。

**设计意图**：精确追踪死亡责任，确保"玩家主动选择击杀"才承担理智惩罚。

### 边缘情况 4：参数边界检查

**场景**：调参时某项系数设置超出安全范围

**处理**：
- 所有参数在游戏启动时进行边界检查
- 超出安全范围的参数自动钳制到边界值
- 日志记录违规参数和修正后的值

**安全范围**：所有系数 0.75 ~ 1.25（狂暴阈值调整为 ±15）

## Dependencies

### 架构说明

本系统为**纯数据输出系统**，不持有任何游戏状态，仅在玩家选择背景后向下游系统传递参数修正值。本系统不监听任何外部事件。

### 上游依赖（系统依赖谁）

本系统**不依赖任何其他系统**。背景选择由玩家在开局时手动完成，选择后背景数据被推送至各下游系统。

### 下游依赖（谁依赖本系统）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **NPC AI系统** | 硬依赖 | 调用 `QueryOldAcquaintanceBonus` 获取旧识关系加成；读取 `AttitudeModifier` 计算 NPC 初始态度 |
| **DialogTree** | 硬依赖 | 读取 `BackgroundType` 判断对话选项可见性和旧识对话分支过滤 |
| **理智/愤怒系统** | 硬依赖 | 读取 `SanityPenaltyMultiplier` 和 `FrenzyThresholdModifier` 计算惩罚和狂暴状态 |
| **叙事系统 (Narrative System)** | 软依赖 | 接收背景类型用于 SKILL 系列模块解锁条件判断（BackgroundType 影响叙事模块的可用性） |
| **Gritty Takedowns 系统** | 硬依赖 | 本系统实现 `IFirstEncounterBonusProvider` 接口，供 Gritty Takedowns 在 `ConfrontationStartRequest` 时调用；Gritty Takedowns 持有 `HasMetFaction` flag 并管理首次遭遇加成的应用时机 |

### 依赖关系矩阵

```
┌─────────────────────────────────────────────────────────────────┐
│                        Character Background                      │
│  （纯数据输出，开局选择后向以下系统推送参数）                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ──AttitudeModifier───────────────► NPC AI System               │
│  ──OldAcquaintanceBonus*──────────► NPC AI System（Query 接口） │
│  ──BackgroundType─────────────────► DialogTree                  │
│  ──SanityPenaltyMultiplier───────► Sanity/Rage Meter           │
│  ──FrenzyThresholdModifier────────► Sanity/Rage Meter           │
│  ──BackgroundType─────────────────► Narrative System（软依赖）  │
│                                                                  │
│  * OldAcquaintanceBonus 通过 QueryOldAcquaintanceBonus 接口提供 │
└─────────────────────────────────────────────────────────────────┘
```

## Tuning Knobs

背景系统的可调参数为 8 项核心参数修正，无特殊能力相关参数。

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `SanityPenaltyMultiplier_Civilian` | float | 0.9 | 0.75~1.0 | 普通人理智惩罚系数 |
| `SanityPenaltyMultiplier_Agent` | float | 0.8 | 0.7~0.95 | 特工理智惩罚系数 |
| `SanityPenaltyMultiplier_Mercenary` | float | 1.15 | 1.0~1.25 | 雇佣兵理智惩罚系数 |
| `FrenzyThresholdModifier_Civilian` | int | +8 | +3~+15 | 普通人狂暴阈值调整 |
| `FrenzyThresholdModifier_Agent` | int | ±0 | ±0 | 特工狂暴阈值调整 |
| `FrenzyThresholdModifier_Mercenary` | int | -8 | -15~-3 | 雇佣兵狂暴阈值调整 |
| `StealthModifier_Civilian` | float | 1.15 | 1.0~1.25 | 普通人潜行系数 |
| `StealthModifier_Agent` | float | 1.0 | 0.9~1.1 | 特工潜行系数 |
| `StealthModifier_Mercenary` | float | 0.85 | 0.75~1.0 | 雇佣兵潜行系数 |
| `CombatModifier_Civilian` | float | 0.8 | 0.7~0.95 | 普通人战斗系数 |
| `CombatModifier_Agent` | float | 0.95 | 0.85~1.05 | 特工战斗系数 |
| `CombatModifier_Mercenary` | float | 1.2 | 1.0~1.25 | 雇佣兵战斗系数 |
| `PerceptionModifier_Civilian` | float | 0.9 | 0.75~1.05 | 普通人感知系数 |
| `PerceptionModifier_Agent` | float | 1.2 | 1.1~1.15 | 特工感知系数（感知范围+10%~15%，上限1.15x以保持挑战平衡） |
| `PerceptionModifier_Mercenary` | float | 1.0 | 0.9~1.1 | 雇佣兵感知系数 |
| `EnvironmentModifier_Civilian` | float | 0.9 | 0.8~1.0 | 普通人环境交互系数 |
| `EnvironmentModifier_Agent` | float | 1.0 | 0.9~1.1 | 特工环境交互系数 |
| `EnvironmentModifier_Mercenary` | float | 1.2 | 1.0~1.25 | 雇佣兵环境交互系数 |
| `ClueAnalysisModifier_Civilian` | float | 1.1 | 1.0~1.2 | 普通人线索分析速度 |
| `ClueAnalysisModifier_Agent` | float | 1.3 | 1.15~1.4 | 特工线索分析速度 |
| `ClueAnalysisModifier_Mercenary` | float | 0.85 | 0.75~1.0 | 雇佣兵线索分析速度 |
| `AttitudeModifier_Civilian` | int | +10 | +5~+15 | 普通人NPC初始态度修正 |
| `AttitudeModifier_Agent` | int | +15 | +10~+20 | 特工NPC初始态度修正 |
| `AttitudeModifier_Mercenary` | int | -5 | -10~+5 | 雇佣兵NPC初始态度修正 |
| `OldAcquaintanceBonus_Max` | int | 25 | 20~30 | 旧识关系最大态度加成 |

> **⚠️ Playtest 验证重点**：`CombatModifier_Mercenary` 是影响"致命脆弱感"支柱的关键参数。建议在首次 playtest 时重点验证：雇佣兵战斗能力增强是否导致"脆弱感"体验显著下降。若玩家反馈雇佣兵战斗过于简单，应将上限下调至 1.1x。

**调参风险提示**：

| 参数 | 风险 |
|------|------|
| `SanityPenaltyMultiplier_*` 设置过低 | 玩家杀戮代价降低，削弱理智系统的情感冲击力 |
| `FrenzyThresholdModifier_Mercenary` 设置过低（如 -15） | 雇佣兵过早进入狂暴，可能导致"狂暴流"玩法固化 |
| `PerceptionModifier_Agent` 设置过高 | 特工感知范围和信息优势过强，在潜行/监听为主要手段的游戏中可能导致特工成为唯一最优背景选择。已将安全范围上限降至 1.15x，首次 playtest 重点验证三个背景的吸引力平衡 |
| `CombatModifier_Mercenary` 设置过高 | 雇佣兵战斗能力过强，破坏"致命脆弱感"支柱 |

## Visual/Audio Requirements

### 背景选择界面视觉要求

| 元素 | 要求 |
|------|------|
| **背景图标** | 每个背景一个标志性图标（普通人：父亲剪影；特工：耳机/通讯器；雇佣兵：战术手套） |
| **背景描述卡** | 显示背景名称、简短描述、参数预览 |
| **参数预览** | 悬停时显示详细参数说明（8项参数） |
| **选择确认** | 选中后高亮显示，确认按钮变为可用 |

### HUD 元素

| 元素 | 显示位置 | 显示条件 |
|------|---------|---------|
| **背景标识** | HUD 左下角 | 游戏内常驻显示当前背景图标和名称 |

## UI Requirements

### 背景选择界面

| 信息 | 显示位置 | 更新频率 | 条件 |
|-------------|-----------------|-----------------|--------|
| **背景名称** | 屏幕顶部中央 | 静态 | 无 |
| **背景图标** | 角色创建界面中央 | 静态 | 无 |
| **背景描述** | 背景图标下方 | 静态 | 无 |
| **参数列表** | 界面左侧 | 静态 | 无 |

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 三个背景可在开局选择界面正确显示 | 打开创建角色界面，验证三个背景选项可见 |
| AC-2 | 选择背景后，BackgroundType 和 BackgroundParameters 正确传递到各系统 | 监听 NPC AI/DialogTree/Sanity/Rage 接口，验证数据正确 |
| AC-3 | NPC 对不同背景产生正确初始态度 | 创建三个角色，对比同一 NPC 的初始态度是否符合态度矩阵 |
| AC-4 | 理智惩罚根据背景系数正确计算 | 击杀后检查理智变化值是否符合公式 |
| AC-5 | 狂暴阈值根据背景正确调整 | 积累 Rage 到不同背景的阈值，验证狂暴触发时机 |

### 跨系统验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-6 | DialogTree 根据背景显示/隐藏对话选项 | 对比三个背景的可用对话选项 |
| AC-7 | 旧识关系在特工/雇佣兵遭遇特定派系时触发 | 测试特工遭遇锈网、雇佣兵遭遇灰烬团时的额外态度加成 |
| AC-8 | 参数修改在游戏启动时正确加载 | 修改参数文件后启动游戏，验证效果变化 |

### 边缘情况验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-9 | 参数超出安全范围时自动钳制 | 设置系数为 0.5 或 1.5，验证启动后被钳制到 0.75 或 1.25 |
| AC-10 | 旧识对话 NPC 死亡后收益正确处理 | 在对话中击杀 NPC，验证收益丢失 |

## Open Questions

| 问题 | 负责人 | 截止日期 | 状态 | 解决方案 |
|------|-------|----------|------|---------|
| 背景选择界面的美术风格 | Art Director | 待定 | Open | 需要确定是写实风格还是符号化图标 |
| 不同背景的配音/台词差异 | Narrative Director | 待定 | Open | 是否需要为每个背景录制专属的内心独白？ |
| 背景是否影响结局分支 | Narrative Director | 待定 | Open | 当前设计是结局由玩家行为决定，背景不影响。但是否需要差异化结局？ |
| ~~背景选择后是否可查看详细属性~~ | ~~UX Designer~~ | ~~待定~~ | ~~已解决~~ | ✅ **已解决**：采用**悬停显示**方案，在背景描述卡上悬停时显示详细参数说明（8项参数），无需专用详情面板 |
| 各参数的默认值是否需要根据实际测试调整 | Game Designer | 待定 | Open | 试玩测试后可能需要调整参数以达到预期手感；`CombatModifier_Mercenary` 需重点验证 |
