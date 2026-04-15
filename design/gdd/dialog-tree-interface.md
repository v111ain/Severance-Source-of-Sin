# DialogTree 接口协议 (DialogTree Interface Protocol)

> **Status**: Approved
> **Author**: AI Programmer + Game Designer
> **Created**: 2026-04-08
> **Last Updated**: 2026-04-15
> **Priority**: MVP
> **Layer**: Interface / Cross-System
> **Interacts With**: Gritty Takedowns, NPC AI System
> **Depends On**: NPC AI System (DialogueTree 数据所有权)

---

## Overview

本协议定义了 **沉重处决系统 (Gritty Takedowns)** 与 **NPC AI 系统 (NPC AI System)** 之间关于对话树 (DialogueTree) 的接口规范。

两个系统存在双向依赖：Gritty Takedowns 需要 NPC AI 系统提供的对话数据进行对峙交互，NPC AI 系统需要 Gritty Takedowns 传来的玩家选择结果来驱动 NPC 行为变化。为避免循环依赖，采用事件驱动架构解耦。

**核心设计原则**：
- NPC AI 系统拥有并存储所有对话树数据
- Gritty Takedowns 系统负责 UI 渲染和玩家输入处理
- 通过明确定义的事件和查询接口进行通信

---

## 术语对照说明

本协议使用 `allegiance` 表示 **NPC 对玩家的态度值**，这是一个独立于叙事系统 `moral_standing`（玩家道德立场）的概念：

| 维度 | `allegiance`（本协议 / NPC AI 系统） | `moral_standing`（叙事系统） |
|------|-------------------------------------|------------------------------|
| **追踪对象** | NPC 对玩家的态度（友好/敌意） | 玩家对 NPC 的道德评判（善/恶） |
| **值域** | -100（极度敌意）~ +100（极度友好） | -100（残暴）~ +100（仁慈） |
| **所属系统** | NPC AI 系统 | 叙事系统（Narrative System） |
| **影响对象** | NPC 行为（攻击/逃跑/合作） | 对话变体、叙事分支、结局 |

**设计意图**：
- `allegiance` 由对话选项的 `allegiance_change` 累加计算，用于驱动 NPC 的即时行为决策
- `moral_standing` 由击杀/放过事件累加计算，用于叙事层面的玩家角色道德评估
- 两者独立追踪、不直接联动，确保对话层面的战术选择与叙事层面的道德评估解耦

---

## Player Fantasy

**"对峙时的紧张对话，是一场心理博弈。"**

当玩家与 NPC 进入对峙状态时，NPC 可能发出试探性对话（如"你在看什么？"）。玩家需要从中获取信息、施加压力或寻找破绽。每一个对话选项都会影响 NPC 的态度——选择得当可兵不血刃，选择失误则可能激化局势。

### 情感体验层次

**第一层：对峙时的心理压力**

玩家站在 NPC 面前，距离不过数米。NPC 的眼神在打量你，你不知道他已经知道了什么——他是单纯在警惕陌生人，还是已经察觉到你的真实身份？这种信息不对称制造了持续的紧张感。玩家需要在瞬间判断：这个 NPC 能被说服吗？他知道多少？我应该花时间套话，还是直接动手？这种决策的压力让每一次对话都像是一场没有退路的赌博。

**第二层：决策后果的不确定性**

对话选项呈现时，玩家看不到每个选择背后的完整后果链条。选择"威胁"可能让 NPC 屈服供出情报，也可能让他呼叫增援；选择"欺骗"可能套出真相，也可能因 NPC 曾经听过类似的谎言而彻底暴露。每一个选项都有多重可能性，玩家的选择是基于有限信息的博弈。这种不确定性让对话结果难以预测——但也正是这种难以预测，让成功说服一个顽固的 NPC 变得格外有成就感。

**第三层：掌控对话节奏的满足感**

当玩家摸清了 NPC 的性格和弱点，通过试探性对话逐步引导 NPC 进入预设的话题轨迹——这种感觉就像一个经验丰富的话术高手在操控对话的走向。玩家从 NPC 的只言片语中拼凑出他的恐惧（"他最近欠了债"）、他的软肋（"他母亲还在等他回家"），然后精准地在关键时刻祭出这些信息，换取 NPC 的妥协。这种"我知道你不知道我知道"的认知优势，带来的是智取者独有的掌控感。

**第四层：背叛与宽恕之间的道德挣扎**

当对话深入，玩家面临真正的道德抉择：对一个已经透露弱点、流露出人性一面的 NPC，是按照原计划将其击杀获取关键线索，还是给予他一个活下去的机会？这种抉择不是游戏机制层面的"最优解"选择，而是玩家内在价值观与游戏情境的碰撞。背叛一个曾经信任你的 NPC——哪怕他是个恶徒——会在事后带来残余的道德重量；选择宽恕，则可能让后续的任务变得更加困难。玩家在这两种选择之间的犹豫，构成了游戏最深刻的情感张力。

---

## Detailed Design

### 接口所有权划分

| 接口 | 拥有者 | 说明 |
|------|--------|------|
| `DialogueTree` 数据结构 | NPC AI 系统 | 定义并存储所有对话树 JSON 配置 |
| 对话 UI 渲染 | Gritty Takedowns | 渲染对话选项 UI，处理玩家输入 |
| `ConfrontationStartRequest` | Gritty Takedowns → NPC AI | 发起对峙请求 |
| `DialogueTreeConfig` | NPC AI 系统 → Gritty Takedowns | 返回对话树数据 |
| `DialogueChoice` | Gritty Takedowns → NPC AI 系统 | 发送玩家选择 |
| `DialogueResult` | NPC AI 系统 → Gritty Takedowns | 返回处理结果 |

### 对话树数据结构 (DialogueTreeConfig)

JSON 格式（由策划编辑，NPC AI 系统反序列化）：

```json
{
  "dialogue_id": "string",
  "npc_id": "string",
  "player_background": "enum (CIVILIAN/AGENT/MERCENARY)",
  "root_branch": "branch_id",
  "variant_id": "string | null",
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
          "is_betrayal_option": "bool",
          "result_type": "enum (CONTINUE/INTIMIDATE/BRIBE/DECOY/PSYCHOLOGICAL_MANIPULATION/BETRAYAL)",
          "betrayal_type_hint": "enum (BETRAYAL_TYPE_STANDARD/BETRAYAL_TYPE_MERCY/BETRAYAL_TYPE_TACTICAL/BETRAYAL_TYPE_PROVOKED) | null"
        }
      ]
    }
  }
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `dialogue_id` | string | 对话树唯一标识符 |
| `npc_id` | string | 关联的 NPC ID |
| `player_background` | enum | 玩家当前背景类型（CIVILIAN/AGENT/MERCENARY），用于 Gritty Takedowns 过滤不可用的对话选项（如旧识对话分支） |
| `root_branch` | string | 起始分支 ID |
| `variant_id` | string \| null | 当前对话变体 ID（MERCY/CRUEL/CALCULATING/CAUTIOUS），由 NPC AI 系统根据 `QueryDominantTrait()` 选择后填充，传递给 GrittyTakedowns 用于 UI 渲染。**null 表示无变体（默认对话）** |
| `branches` | object | 所有分支的字典 |

**branches.*.choices 数组字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `choice_id` | string | 选项唯一标识符 |
| `text` | string | 选项显示文本 |
| `next_branch` | string \| null | 下一分支 ID，null 表示对话结束 |
| `allegiance_change` | int | 选择后 NPC 对玩家态度变化值 |
| `requires_vulnerability` | bool | 是否需要先获取 NPC 的 vulnerability 才能显示此选项 |
| `is_betrayal_option` | bool | **背叛选项标记**：true 表示此选项为背叛选项。Gritty Takedowns 在玩家选择后设置 `BetrayalEvent` 通知叙事系统 |
| `result_type` | enum | 结果类型（CONTINUE/INTIMIDATE/BRIBE/DECOY/PSYCHOLOGICAL_MANIPULATION/BETRAYAL） |
| `betrayal_type_hint` | enum \| null | **背叛类型提示**（可选，仅当 `result_type == BETRAYAL` 时有效）：BETRAYAL_TYPE_STANDARD/BETRAYAL_TYPE_MERCY/BETRAYAL_TYPE_TACTICAL/BETRAYAL_TYPE_PROVOKED。Gritty Takedowns 在发送 `BetrayalEvent` 时使用此字段填充 `betrayal_type`。若为 null，则由 Gritty Takedowns 根据对话上下文自行判断背叛类型 |

**variant_id 选择与传递机制**

variant_id 的选择和传递遵循以下流程：

| 步骤 | 执行者 | 动作 |
|------|-------|------|
| 1 | GrittyTakedowns | 发送 `ConfrontationStartRequest(npc_id, player_background)` |
| 2 | NPC AI 系统 | 加载 `DialogueTreeConfig` |
| 3 | NPC AI 系统 | 调用 `NarrativeSystem.QueryDominantTrait()` 获取 `MoralTrait` |
| 4 | NPC AI 系统 | 根据 `MoralTrait` 映射到对应的 `variant_id`（详见下表） |
| 5 | NPC AI 系统 | 将 `variant_id` 填入 `DialogueTreeConfig.variant_id` 字段 |
| 6 | NPC AI 系统 | 返回完整 `DialogueTreeConfig`（包含 `variant_id`）给 GrittyTakedowns |
| 7 | GrittyTakedowns | 根据 `variant_id` 过滤并渲染对应的对话分支 UI |

**MoralTrait → variant_id 映射表**：

| MoralTrait | variant_id | 说明 |
|------------|------------|------|
| `DOMINANT_MERCY` | `"MERCY"` | 仁慈主导特质 |
| `DOMINANT_CRUELTY` | `"CRUEL"` | 残暴主导特质 |
| `DOMINANT_CALCULATING` | `"CALCULATING"` | 计算主导特质 |
| `DOMINANT_CAUTIOUS` | `"CAUTIOUS"` | 谨慎主导特质 |
| `null`（零行为玩家） | `"CAUTIOUS"` | 零行为玩家默认变体 |

**InteractionEvent 与 variant_id 的关系**

对话结束时发送的 `InteractionEvent` **不需要包含 variant_id**，原因如下：

- GrittyTakedowns 在对话开始时已收到完整的 `DialogueTreeConfig`（包含 `variant_id`）
- GrittyTakedowns 在本地维护当前对话会话状态，包括 `variant_id`
- 对话结束事件只需标识 `DIALOG_COMPLETE` 和 `npc_id`，GrittyTakedowns 可自行关联到本地会话状态
- 如需查询 variant_id，可通过 `DialogueTreeConfig.variant_id` 获取（由 NPC AI 系统在对话开始时传递）

```
InteractionEvent 格式：
{
    event_type: "DIALOG_COMPLETE",
    npc_id: "string",
    // 无需 variant_id 字段
}
```
### 情绪枚举 (DialogueEmotion)

| 值 | 说明 | UI 表现 |
|----|------|---------|
| `NEUTRAL` | 普通 | 标准对话气泡 |
| `AGITATED` | 激动 | 气泡边缘抖动 |
| `SCARED` | 恐惧 | 气泡颤抖 + 颜色变淡 |
| `ANGRY` | 愤怒 | 气泡变红 + 边缘锯齿 |

### 结果类型枚举 (DialogueResultType)

| 值 | 说明 | Gritty Takedowns 处理 |
|----|------|----------------------|
| `CONTINUE` | 对话继续 | 显示下一分支内容 |
| `INTIMIDATE` | 威胁成功 | 触发威胁成功音效 + allegiance 大幅下降 |
| `BRIBE` | 贿赂成功 | 触发金币音效 + allegiance 提升 |
| `DECOY` | 欺骗成功 | 触发欺骗特效 + allegiance 变化 |
| `PSYCHOLOGICAL_MANIPULATION` | 心理操纵成功 | 触发心理操纵特效 + allegiance 大幅变化（+20），利用已知弱点 |
| `BETRAYAL` | 背叛成功 | 触发背叛选项特有的后续处理 |

> **枚举一致性说明**：`PSYCHOLOGICAL_MANIPULATION` 作为独立交互类型（BaseChange=+20），区别于 `DECOY`（欺骗成功，BaseChange=+10）。两者都依赖已获取的 vulnerability，但效果强度不同。

---

## 接口调用流程

### 时序图

```
┌─────────────────────┐                              ┌─────────────────────┐
│  Gritty Takedowns   │                              │    NPC AI System    │
└──────────┬──────────┘                              └──────────┬──────────┘
           │                                                    │
           │  1. ConfrontationStartRequest(npc_id, player_background) │
           │──────────────────────────────────────────────────►│
           │                                                    │
           │                                                    │  查询 DialogueTreeConfig
           │                                                    │  根据 npc_id 加载对应对话树
           │                                                    │  根据 player_background 过滤旧识对话分支
           │                                                    │
           │  2. DialogueTreeConfig (JSON)                     │
           │    （含 player_background 用于 UI 选项过滤）        │
           │◄──────────────────────────────────────────────────│
           │                                                    │
           │  3. 渲染对话 UI（显示 text + choices）              │
           │                                                    │
           │  4. 玩家选择选项                                    │
           │                                                    │
           │  5. DialogueChoice(choice_id, npc_id)              │
           │──────────────────────────────────────────────────►│
           │                                                    │
           │                                                    │  查找 choice_id 对应选项
           │                                                    │  计算 allegiance_change
           │                                                    │  查找 next_branch
           │                                                    │  发送 KnowledgeGained（如有）
           │                                                    │
           │  6. DialogueResult(result)                        │
           │◄──────────────────────────────────────────────────│
           │                                                    │
           │  7a. result.next_branch != null                   │
           │      → 渲染下一分支 UI（回到步骤 3）                  │
           │                                                    │
           │  7b. result.next_branch == null                   │
           │      → 对话结束，发送 InteractionEvent              │
           │                                                    │
           └────────────────────────────────────────────────────┘
```

### 详细步骤说明

**步骤 1：发起对峙请求**

```
ConfrontationStartRequest(npc_id: string, player_background: BackgroundType) → Event
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `npc_id` | string | 目标 NPC 的唯一标识符（与 NPC AI 系统中的 EntityID 一致） |
| `player_background` | BackgroundType | 玩家当前背景类型（CIVILIAN/AGENT/MERCENARY），由 GrittyTakedowns 从 Character Background 系统获取，用于 NPC AI 系统过滤旧识对话分支 |

> **BackgroundType 枚举定义**（来自 Character Background 系统）：
> - `CIVILIAN`：普通人
> - `AGENT`：特工
> - `MERCENARY`：雇佣兵

**触发条件**：
- 玩家与 NPC 处于对峙距离（≤ 5 米）
- NPC 处于 FREE 状态
- NPC Alert State < ALERT（UNDETECTED/SUSPECT/SEARCH 三态之一）
- 注：Alert State 枚举定义详见 `npc-ai-system.md`

**步骤 2：返回对话树配置**

```
DialogueTreeConfig → Event
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `dialogue_id` | string | 对话树 ID |
| `npc_id` | string | NPC ID |
| `root_branch` | string | 起始分支 ID |
| `branches` | object | 分支字典 |

**数据来源**：NPC AI 系统根据 `npc_id` 查找对应的对话树 JSON 配置，并根据 `player_background` 过滤旧识对话分支后返回。

**步骤 3：渲染对话 UI**

Gritty Takedowns 系统负责：
- 显示 NPC 台词气泡
- 显示可用对话选项（2-4 个）
- 选项根据 `requires_vulnerability` 过滤（未获取 vulnerability 时不显示）
- 选项根据 `player_background` 过滤旧识对话选项（如特工专属的锈网旧识对话分支）

**步骤 4：玩家选择**

玩家通过键盘/手柄选择选项。

**输入路由规则**：
| 输入设备 | 选择方式 | 路由规则 |
|---------|---------|---------|
| 键盘 | 数字键 1-4 / 方向键 + Enter | 数字键直接索引（1=第1个选项），方向键移动焦点，Enter 确认 |
| 手柄 | 方向键 + A/X 确认 | 方向键移动焦点，A/X 确认 |
| 鼠标 | 点击选项 | 直接点击选项，无须先选中 |

**输入解析逻辑**：
1. 玩家按数字键 N 时，直接发送 `DialogueChoice(choice[N-1], npc_id)`
2. 玩家按方向键时，更新 UI 焦点位置（不发事件）
3. 玩家按确认键时，发送当前焦点选项的 `DialogueChoice`
4. 玩家点击选项时，直接发送该选项的 `DialogueChoice`

**选项过滤规则**：
- `requires_vulnerability = true` 的选项在玩家未获取对应 vulnerability 时不显示
- 所有可用选项按 `choice_id` 字母顺序排列后，依次分配 1-N 索引

**选项数量超限处理机制（UI 层面）**：

当过滤后的可用选项数量超过 `DialogueChoiceMaxDisplay`（默认=4）时，采用以下 UI 处理机制：

| 情况 | 处理方式 |
|------|---------|
| 选项数量 = 5~6 | 启用垂直滚动列表，向上/下箭头或方向键滚动显示超出选项 |
| 选项数量 > 6 | 分页显示，每页最多显示 4 个选项，使用左右方向键或数字键切换页面 |

**滚动/分页交互规则**：

| 输入设备 | 滚动操作 | 分页操作 |
|---------|---------|---------|
| 键盘 | 上/下方向键移动焦点（超出可见区域时滚动列表） | 左/右方向键切换页面，数字键 5-9 直接跳转至对应页面 |
| 手柄 | 上/下方向键移动焦点并滚动 | LB/RB 或左/右方向键切换页面 |
| 鼠标 | 滚轮滚动列表 | 不支持鼠标分页操作（需使用键盘/手柄） |

**选项索引分配规则（扩展）**：
- 可见选项按字母顺序分配 1-4 索引
- 不可见的滚动/分页选项在滚动到可见位置后，显示其对应索引
- 数字键 1-4 始终对应当前可见页面/滚动位置的前 4 个选项

**示例**：
假设有 7 个选项（按字母顺序）：A, B, C, D, E, F, G
- 第一页显示：A(1), B(2), C(3), D(4)，按右方向键或按5切换到第二页
- 第二页显示：E(1), F(2), G(3)，按左方向键或按1返回第一页

**步骤 5：发送玩家选择**

```
DialogueChoice(choice_id: string, npc_id: string) → Event
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `choice_id` | string | 玩家选择的选项 ID |
| `npc_id` | string | 目标 NPC ID（与 ConfrontationStartRequest 中的 npc_id 一致） |

**知识获取与 ClueDiscoveredEvent 的关系**

当对话选项的 `result_type` 为特定类型（如 INTIMIDATE/BRIBE/DECOY）时，NPC 可能在对话中透露信息（knowledge_gained）。

DialogueResult 中的 `knowledge_gained: List[string] | null` 字段表示通过审问获取的线索 ID 列表。

**与 Clue&Journal 系统 ClueDiscoveredEvent 的关系**：

| 事件/字段 | 来源系统 | 用途 | 性质 |
|---------|---------|------|------|
| `DialogueResult.knowledge_gained` | NPC AI 系统（DialogTree 协议） | 返回给 Gritty Takedowns 的内部数据字段，包含审问获取的线索 ID 列表 | 协议内部数据 |
| `ClueDiscoveredEvent` | Clue&Journal 系统 | 跨系统通知事件，通知理智系统等下游系统有线索被发现 | 跨系统事件 |

**处理流程**：

1. NPC AI 系统在 `DialogueResult.knowledge_gained` 中返回线索 ID 列表
2. Gritty Takedowns 收到 DialogueResult 后：
   - 将 `knowledge_gained` 中的线索 ID 转发给 Clue&Journal 系统
   - Clue&Journal 系统为每条线索创建 Clue 实例（状态设为 DISCOVERED）
   - Clue&Journal 系统发送 `ClueDiscoveredEvent` 到事件总线

**重要说明**：DialogTree 接口协议本身不直接发送 ClueDiscoveredEvent，该事件由 Clue&Journal 系统在接收到线索数据后负责发送。DialogTree 只负责通过 `knowledge_gained` 字段传递线索 ID 列表。

**步骤 6：返回处理结果**

```
DialogueResult {
    result_type: DialogueResultType,
    allegiance_change: int,
    knowledge_gained: List[string] | null,
    next_branch: string | null,
    npc_emotion: DialogueEmotion
} → Event
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `result_type` | enum | 结果类型 |
| `allegiance_change` | int | allegiance 变化值 |
| `knowledge_gained` | array \| null | 审问获取的线索列表（如有） |
| `next_branch` | string \| null | 下一分支 ID，null 表示对话结束 |
| `npc_emotion` | enum | NPC 新的情绪状态 |

**步骤 7：结束或继续**

- `next_branch != null`：返回步骤 3，渲染下一分支
- `next_branch == null`：对话结束，发送 `InteractionEvent(DIALOGUE_COMPLETE, npc_id)` 到事件总线

---

## Edge Cases

### 边缘情况 1：NPC 在对话期间状态变化

**问题**：对话进行中，NPC 被第三方攻击或发现其他威胁。

**处理**：
- NPC AI 系统发送 `NPCStateChangedEvent` 或 `AlertStateChangedEvent`
- Gritty Takedowns 收到事件后，**立即中断对话**
- 显示"NPC 被打断！"提示
- 对话进度**不保存**，下次对峙从头开始

### 边缘情况 2：对话树配置缺失

**问题**：请求的 `npc_id` 没有对应的对话树配置。

**处理**：
- NPC AI 系统返回空的 `DialogueTreeConfig`（`branches` 为空）
- Gritty Takedowns 检测到空配置，显示默认选项：
  - "威胁" → 触发威胁交互
  - "离开" → 取消对峙

### 边缘情况 3：玩家选择需要 vulnerability 但未获取

**问题**：选项的 `requires_vulnerability = true`，但玩家尚未获取 NPC 的 vulnerability。

**处理**：
- 该选项在 UI 中**不显示**
- Gritty Takedowns 在渲染前过滤选项列表

### 边缘情况 4：对话选项全部被过滤

**问题**：由于 `requires_vulnerability` 限制，没有任何选项可显示。

**处理**：
- Gritty Takedowns 显示默认选项：
  - "离开" → 取消对峙
- NPC 播放一条额外台词："看什么看？"

### 边缘情况 5：NPC 在对话期间被击杀

**问题**：玩家选择期间，第三方将 NPC 击杀。

**处理**：
- Gritty Takedowns 收到 `NPCStateChangedEvent(DEAD)`
- 立即关闭对话 UI
- 显示"NPC 已死亡"提示（1.5 秒）
- 玩家自动进入 Idle 状态

### 边缘情况 6：allegiance 变化触发 NPC 行为立即变化

**问题**：对话结束后，allegiance 变化导致 NPC 立即逃跑/攻击。

**处理**：
- NPC AI 系统在发送 `DialogueResult` 后，正常处理 allegiance 变化
- Gritty Takedowns 侧：
  - 对话结束动画（0.3 秒）
  - 动画完成后，NPC 根据新 allegiance 执行对应行为

### 边缘情况 7：BetrayalEvent 背叛选项触发机制

**问题**：叙事系统定义 BetrayalEvent 由 Gritty Takedowns 在对话中选择"背叛"选项时设置，但 DialogTree 文档未明确说明触发机制。

**`betrayal_type` 枚举定义**：

| 枚举值 | 说明 | 使用场景 |
|--------|------|---------|
| `BETRAYAL_TYPE_STANDARD` | 标准背叛 | 玩家选择标准的背叛选项，背叛一个已建立信任的NPC |
| `BETRAYAL_TYPE_MERCY` | 仁慈背叛 | 玩家以"仁慈"为由欺骗NPC后背叛（如假装放过但实际击杀） |
| `BETRAYAL_TYPE_TACTICAL` | 战术背叛 | 玩家为获取情报/优势而暂时假装合作，随后背弃承诺 |
| `BETRAYAL_TYPE_PROVOKED` | 受激背叛 | 玩家因NPC的挑衅或威胁而选择背叛（有限度） |

> **枚举使用说明**：`betrayal_type` 由 Gritty Takedowns 系统在发送 `BetrayalEvent` 时填充。DialogTree 系统仅负责通过 `result_type: "BETRAYAL"` 标记背叛选项，不负责 `betrayal_type` 的具体值。

**背叛选项（betrayal_option）触发流程**：

| 步骤 | 执行者 | 动作 |
|------|-------|------|
| 1 | Gritty Takedowns | 渲染对话 UI，显示包含背叛选项的 choice（result_type: BETRAYAL） |
| 2 | 玩家 | 选择背叛选项 |
| 3 | Gritty Takedowns | 发送 `DialogueChoice(choice_id, npc_id)` 到 NPC AI 系统 |
| 4 | NPC AI 系统 | 处理选择，计算 allegiance 变化 |
| 5 | Gritty Takedowns | **设置 `BetrayalEvent{npc_id, betrayal_type}`**，通知叙事系统 |
| 6 | NPC AI 系统 | 返回 `DialogueResult` |

**关键说明**：
- **DialogTree 不负责判断背叛逻辑**：DialogTree 系统只负责传递对话选项的数据结构，不判断选项是否构成"背叛"
- **Gritty Takedowns 负责设置 BetrayalEvent**：背叛选项的判断和事件设置由 Gritty Takedowns 系统在玩家选择后触发
- **DialogueTreeConfig 中的 betrayal_option 标记**：`choices` 数组中的 `result_type: "BETRAYAL"` 字段标识该选项为背叛选项，Gritty Takedowns 根据此标记在玩家选择后设置 `BetrayalEvent`
- **`BETRAYAL` 仅作为选择标记**：当 `result_type == BETRAYAL` 时，DialogueResult 本身**不包含**背叛信息。Gritty Takedowns 必须**额外发送独立的 `BetrayalEvent`** 事件到 Narrative 系统。两者是不同的数据传递机制：DialogueResult 携带对话处理结果（如 allegiance 变化），BetrayalEvent 携带背叛事件元数据（betrayal_type）。

**数据结构扩展**：

```json
{
  "choices": [
    {
      "choice_id": "string",
      "text": "string",
      "next_branch": "branch_id | null",
      "allegiance_change": "int",
      "requires_vulnerability": "bool",
      "result_type": "enum (CONTINUE/INTIMIDATE/BRIBE/DECOY/PSYCHOLOGICAL_MANIPULATION/BETRAYAL)"
    }
  ]
}
```

> **注意**：`BETRAYAL` 是 `DialogueResultType` 的新增枚举值，表示该选项会导致 NPC 被背叛。当 `result_type` 为 `BETRAYAL` 时，Gritty Takedowns 在处理完对话结果后设置 `BetrayalEvent`。

---

## Dependencies

### 上游依赖

| 系统 | 接口类型 | 依赖性质 | 说明 |
|------|---------|---------|------|
| NPC AI 系统 | 事件接收 + 数据查询 | 硬依赖 | 接收 `ConfrontationStartRequest`，返回 `DialogueTreeConfig`，处理 `DialogueChoice` |

### 下游依赖

| 系统 | 接口类型 | 依赖性质 | 说明 |
|------|---------|---------|------|
| Gritty Takedowns | 事件发送 | 硬依赖 | 发送 `DialogueChoice`，接收 `DialogueResult`，渲染对话 UI |
| **叙事系统 (Narrative System)** | **INarrativeMoralQuery 接口查询** | 硬依赖 | NPC AI 系统通过 `INarrativeMoralQuery.QueryDominantTrait()` 获取玩家主导特质，用于选择对话变体分支 |
| **主角背景角色系统 (Character Background)** | 接口查询 | 硬依赖 | 接收 `BackgroundType`，用于判断对话选项的可见性和旧识对话分支的触发 |

### 依赖关系图

```
┌─────────────────────────────────────────────────────────────┐
│                    Cross-System Interface                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   NPC AI System ──────► DialogueTreeConfig ──────► Gritty   │
│         ▲                (数据所有权)              │  Takedowns  │
│         │                                           │            │
│         │◄──── DialogueResult ◄──── DialogueChoice │            │
│         │                                           │            │
│         │◄──── InteractionEvent ◄──── (对话结束)   │            │
│         │                                           │            │
│         │◄── QueryDominantTrait() ─────────────────│            │
│         │         (INarrativeMoralQuery 接口)              │
│         │                                           │            │
│         ▼                                           │            │
│   ┌─────────────────────────────────────────┐      │            │
│   │   叙事系统 (Narrative System)           │◄─────┘            │
│   │   - Moral Profile                       │                   │
│   │   - dominant_trait 计算                │                   │
│   └─────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

### dominant_trait 对话变体选择机制

**概述**：DialogTree 系统通过 `INarrativeMoralQuery` 接口向叙事系统查询玩家的主导特质，用于在发起对话时选择正确的变体分支。

**接口定义**（由叙事系统实现）：

```csharp
interface INarrativeMoralQuery {
    MoralTrait QueryDominantTrait();     // 返回当前主导特质
    int QueryMoralStanding();             // 返回当前道德立场值
    bool QueryHasTrait(MoralTrait trait); // 检查是否具有特定特质
}
```

**调用时机**：在 `ConfrontationStartRequest` 到达后、加载 `DialogueTreeConfig` 前，NPC AI 系统调用 `QueryDominantTrait()`。

**流程**：

| 步骤 | 执行者 | 动作 |
|------|-------|------|
| 1 | GrittyTakedowns | 发送 `ConfrontationStartRequest(npc_id)` |
| 2 | NPC AI 系统 | 调用 `NarrativeSystem.QueryDominantTrait()` |
| 3 | NPC AI 系统 | 根据 dominant_trait 选择对应分支，设置 `DialogueTreeConfig.variant_id` |
| 4 | NPC AI 系统 | 返回 `DialogueTreeConfig`（包含变体信息） |
| 5 | GrittyTakedowns | 渲染对话 UI，按 `variant_id` 显示对应变体 |

**注意**：NPC AI 系统不直接访问叙事系统的内部状态，仅通过 `INarrativeMoralQuery` 接口查询。

### 循环依赖解决

**问题**：Gritty Takedowns 和 NPC AI 系统互相依赖。

**解决方案**：通过事件总线解耦。

```
修订前（循环依赖）：
Gritty Takedowns ──(硬依赖)──► NPC AI 系统
     ▲                                  │
     └────────(硬依赖)────────────────────┘

修订后（解耦）：
Gritty Takedowns ──(事件总线)──► NPC AI 系统
     │                                  │
     └────── DialogueChoice ◄───────────┘
```

---

## Formulas

> **公式所有权说明**：本节完整内联了 `allegiance` 变化计算公式（引用自 npc-ai-system.md Section 4.1 公式1），完整变量定义见该章节。本节仅提供调用接口的上下文摘要。

### allegiance 变化计算

对话选项导致的 allegiance 变化由 NPC AI 系统计算：

```
AllegianceDelta = BaseChange × |InteractionTypeMultiplier| × ContextMultiplier × RelationshipMultiplier
```

**变量完整定义**：

| 变量 | 定义 | 值 | 来源 |
|------|------|-----|------|
| `BaseChange` | 对话选项中定义的基准变化值 | 威胁=-10, 贿赂=+10, 欺骗=+10, 心理操纵=+20, 背叛=N/A（由 BetrayalEvent 处理） | DialogTreeConfig.choices[].allegiance_change |
| `InteractionTypeMultiplier` | 交互类型乘数（**乘数取绝对值**，负值仅表示态度变化方向） | 威胁=-1.0, 贿赂=1.0, 欺骗=0.8, 心理操纵=1.2 | npc-ai-system.md 公式1 |
| `ContextMultiplier` | NPC 当前 Alert State 乘数 | UNDETECTED=1.0, SUSPECT=1.2, SEARCH=1.5, ALERT=2.0 | npc-ai-system.md 公式1 |
| `RelationshipMultiplier` | 玩家与 NPC 的派系关系等级 | 敌对=0.8, 中立=1.0, 友好=1.2 | npc-ai-system.md 公式1 |

**ContextMultiplier 完整值域表**：

| Alert State | ContextMultiplier | 说明 |
|-------------|-------------------|------|
| UNDETECTED | 1.0 | NPC 未察觉玩家 |
| SUSPECT | 1.2 | NPC 怀疑玩家 |
| SEARCH | 1.5 | NPC 正在搜索玩家 |
| ALERT | 2.0 | NPC 已发现玩家并处于警戒状态 |

> **说明**：对话期间 NPC 进入 ALERT 状态时，根据边缘情况 1 的处理规则，对话会立即中断。但在极端情况下（如 NPC 在对话选项呈现期间从 SEARCH 升级到 ALERT），如果对话尚未中断，ALERT 状态的乘数按 2.0 计算。

**最终变化**：`Clamp(Allegiance + AllegianceDelta, -100, +100)`

> **公式引用链**：
> - 公式定义：`npc-ai-system.md Section 4.1 公式1`
> - 公式所有权：NPC AI System 唯一定义
> - DialogTree 接口协议仅引用计算结果（`DialogueResult.allegiance_change`）

---

## Tuning Knobs

### 对话 UI 参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `DialogueFadeInDuration` | float | 0.2s | 0.1~0.5s | 对话气泡淡入时长 |
| `DialogueFadeOutDuration` | float | 0.15s | 0.1~0.3s | 对话气泡淡出时长 |
| `DialogueChoiceMaxDisplay` | int | 4 | 2~6 | 最多显示选项数 |
| `DialogueEmotionShakeIntensity` | float | 0.05 | 0.02~0.1 | 激动情绪气泡抖动幅度 |

### 滚动/分页 UI 参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `DialogueChoiceScrollThreshold` | int | 5 | 3~6 | 启用滚动的最小选项数（超过此值启用滚动UI） |
| `DialogueChoicePageSize` | int | 4 | 2~6 | 分页模式每页显示的选项数（与 DialogueChoiceMaxDisplay 联动） |
| `DialogueScrollAnimationDuration` | float | 0.15s | 0.1~0.3s | 滚动动画时长 |
| `DialoguePageTransitionDuration` | float | 0.2s | 0.1~0.4s | 分页切换动画时长 |

### 时序参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `DialogueEndAnimationDuration` | float | 0.3s | 0.2~0.5s | 对话结束动画时长 |
| `DialogueInterruptionFadeDuration` | float | 0.15s | 0.1~0.3s | 对话中断淡出时长 |

---

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 玩家触发对峙后，对话 UI 正确显示 NPC 台词和选项 | 进入对峙距离，观察 UI |
| AC-2 | 需要 vulnerability 的选项在未获取时不显示 | 检查 UI 选项数量 |
| AC-3 | 玩家选择后，allegiance 正确变化 | 选择前后查询 `QueryAllegiance` |
| AC-4 | 对话结束后，NPC 根据新 allegiance 执行对应行为 | 变化 allegiance 后观察 NPC |
| AC-5 | 对话期间 NPC 被击杀，对话 UI 正确关闭 | 在对话中击杀 NPC，验证 UI |

### 集成验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-6 | Gritty Takedowns 正确发送 `ConfrontationStartRequest` | 监控事件总线 |
| AC-7 | NPC AI 系统正确返回 `DialogueTreeConfig` | 监控事件总线 |
| AC-8 | Gritty Takedowns 正确发送 `DialogueChoice` | 选择选项后监控事件 |
| AC-9 | NPC AI 系统正确返回 `DialogueResult` | 选择后监控事件 |
| AC-10 | 对话结束正确发送 `InteractionEvent` | 对话结束时监控事件 |

### 边缘情况验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-11 | NPC 对话期间被攻击，对话立即中断 | 对话中攻击 NPC，验证中断 |
| AC-12 | 无对话树配置时显示默认选项 | 对无配置 NPC 触发对峙 |
| AC-13 | 所有选项被过滤时显示"离开"选项 | 获取 vulnerability 后验证 |

---

## Open Questions

| # | 问题 | 负责人 | 目标日期 |
|---|------|--------|---------|
| OQ-1 | 对话树是否需要支持条件分支（根据玩家属性显示不同选项） | Narrative Director | 2026-04-20 |
| OQ-2 | NPC 主动发起的对话（非对峙触发）如何处理 | Narrative Director | 2026-04-20 |

---

## Change Log

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-08 | 0.1 | 初稿创建，定义 DialogTree 接口协议 | AI Programmer Agent |
| 2026-04-10 | 0.2 | 设计审查修复：补充 dominant_trait 对话变体选择机制，添加 INarrativeMoralQuery 接口定义和调用流程说明；更新依赖关系图 | Claude Code |
| 2026-04-10 | 0.3 | **P1修复**：DialogueTreeConfig 添加 variant_id 字段，解决与叙事系统的 variant_id 传递机制不一致问题 | Claude Code |
| 2026-04-13 | 0.4 | 格式修复：删除 InteractionEvent 格式后的错误表格行（speaker、text、emotion 等字段）；枚举修复：在 DialogueResultType 枚举和 DialogueTreeConfig.result_type 中添加 BETRAYAL 选项 | Claude Code |
| 2026-04-13 | 0.5 | **问题1修复**：Player Fantasy 章节扩展，补充对峙心理压力、决策后果不确定性、掌控对话节奏满足感、背叛vs宽恕道德挣扎四个层次的情感体验描述 | Claude Code |
| 2026-04-14 | 0.6 | **P1修复：KnowledgeGained与ClueDiscoveredEvent关系明确**：在步骤5添加"知识获取与ClueDiscoveredEvent的关系"说明，澄清knowledge_gained是DialogResult内部数据字段，ClueDiscoveredEvent由Clue&Journal系统在接收到线索数据后发送；两者不是同一事件，不重叠 |
| 2026-04-14 | 0.6 | **P1修复：ContextMultiplier值域完整**：在Formulas章节补充完整的Alert State乘数表，新增ALERT=2.0状态乘数，并说明ALERT状态出现在对话期间时的处理规则 |
| 2026-04-14 | 0.6 | **P2修复：超过4个选项时的UI处理**：在Detailed Design章节添加"选项数量超限处理机制"说明（滚动/分页UI、交互规则、索引分配），在Tuning Knobs章节新增滚动/分页相关参数（DialogueChoiceScrollThreshold、DialogueChoicePageSize、DialogueScrollAnimationDuration、DialoguePageTransitionDuration） |
| 2026-04-15 | 0.7 | **P1修复：Formulas章节独立性与引用链明确**：添加完整的公式引用链说明（引用npc-ai-system.md Section 4.1 公式1），内联所有变量定义和值域表，确保文档独立可读；添加公式引用链注释，明确所有权归属 | Claude Code |
