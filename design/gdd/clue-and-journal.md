# 线索与日志系统 (Clue & Journal System)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-04-06
> **Implements Pillar**: 罪恶的深度 (Depth of Sin)

## Overview

线索与日志系统 (Clue & Journal System) 是《断绝：罪恶之源》中管理玩家收集的所有情报碎片的中心枢纽。

**三种线索来源与转换流程**：

| 来源 | 原始数据 | 转换关系 |
|------|---------|---------|
| LOS系统窃听 | 关键词 (KeyWords) | 需经过"破译-加工"流程转化为线索 |
| 环境交互系统 | 情报类物件 (手机/账本/文件) | 本身就是成品线索，无需加工 |
| NPC死亡 | NPC的`knowledge` (线索ID列表) | 本身就是线索ID列表，直接转移给玩家 |

**核心功能**：系统接收来自LOS系统的关键词（原材料），通过监听专注模式中的破译进度条完成"原材料→线索"的加工；直接从环境物件获取成品线索；NPC死亡时将其掌握的`knowledge`转移给玩家。每条线索存入日志，日志按**地点-人物双维度**组织（地点为主视图），帮助玩家追踪任务进度并拼凑犯罪网络的完整真相。

**防止卡关**：系统维护每个任务的"最低线索阈值"——当关键NPC死亡导致线索缺失时，系统触发补偿机制（给予替代线索或开放其他调查路径），防止玩家卡关。日志中仍会显示"[信息缺失—NPC已死亡]"，保留"罪恶的深度"支柱的情感张力。

## Player Fantasy

玩家在收集线索时的核心情感体验应当是**"拼图者的成就感"**与**"无法挽回的遗憾"**之间的交织。

* **发现者的狂喜**（参考：《Her Story》）：当玩家在暗处窃听并成功提取关键词，或翻找尸体获得关键情报时，那种"我知道了他不知道我知道"的认知优势带来强烈的掌控感。每一条线索都是一块拼图，逐步揭示犯罪网络的真相。这种**智识满足感**是玩家持续探索的核心驱动力。

* **缺失者的遗憾**（参考：《Return of the Obra Dinn》）：当玩家因冲动击杀而失去某个NPC掌握的关键线索时，日志中醒目的"[信息缺失—NPC已死亡]"标签带来真实的后悔感。这种"因为我选择错误导致永远无法知道真相"的体验，是"罪恶的深度"支柱的核心交付——**选择即代价**。

* **道德困境的重量**：当NPC本身有值得同情的故事（被迫参与的帮凶、家属被威胁的守卫），杀他们的情感代价会更大。线索的缺失不仅是信息上的空白，更是道德选择的后果——玩家杀死的每一个NPC都可能是一个悲剧的结束，而非单纯的战术动作。

**核心设计意图**：玩家不是在"完成任务"，而是在"拼凑真相"并为之付出代价。日志中的每一条记录都是选择的结果，包含着"知道真相的庆幸"或"永远错过的遗憾"。

## Detailed Design

### Core Rules

**规则1：线索分类 (Clue Categories)**

每条线索属于以下类别之一：

| 类别 | 描述 | 示例 | 获取来源 |
|------|------|------|---------|
| **身份线索 (Identity)** | 揭示NPC真实身份的信息 | "张三是青龙帮的二把手" | LOS窃听/搜身 |
| **位置线索 (Location)** | 指向其他据点或人物位置的信息 | "下一个仓库在码头东区" | LOS窃听/NPC知识转移 |
| **关系线索 (Relationship)** | 揭示派系或人物间关系的信息 | "李四是王五的马仔" | 环境物件/NPC知识转移 |
| **物品线索 (Item)** | 与任务相关的物件位置 | "账本藏在保险柜里" | 环境物件/搜身 |
| **悲剧线索 (Tragedy)** | 揭示NPC背景故事或受害者的信息 | "这个女人是被强迫的" | LOS窃听/NPC知识转移 |

**规则2：线索数据结构 (Clue Data Structure)**

```python
Clue:
    clue_id: String           # 唯一标识符，如 "ID_张三分_仓库位置"
    category: Enum           # Identity/Location/Relationship/Item/Tragedy
    title: String             # 显示标题，如 "张三的身份"
    description: String       # 详细内容（解锁后可见）
    source_type: Enum         # LOS_EAVESDROP / ENVIRONMENT / NPC_DEATH
    source_id: String         # 来源标识（如NPC_ID或物件ID）
    location_id: String       # 所属地点ID
    related_npc_ids: []       # 关联的NPC ID列表
    task_id: String           # 关联的任务ID
    criticality: Enum         # CRITICAL / IMPORTANT / OPTIONAL
    prerequisites: []         # 前置线索ID列表，必须全部UNLOCKED后才能解锁本线索
    is_discovered: Bool       # 是否已发现
    is_locked: Bool           # 是否已解锁（需前置线索）
    locked_reason: String     # 解锁条件描述
    missing_reason: String    # 若NPC已死亡，记录缺失原因
```

**规则3：线索解锁机制 (Clue Unlock)**

* 线索分为**已发现(Discovered)**和**未发现(Not Discovered)**两种状态
* 某些线索有前置要求（`is_locked: true`），需要先收集其他线索才能解锁
* NPC死亡时，该NPC掌握的线索如果未被获取，直接标记为`missing_reason: "NPC已死亡"`
* 解锁前置规则：必须按 `task_id` 顺序解锁；跨任务线索可并行收集

**规则4：日志数据结构 (Journal Data Structure)**

```python
Journal:
    locations: Dictionary[location_id, LocationEntry]
    clues: Dictionary[clue_id, Clue]
    missing_clues: List[Clue]  # 永久缺失的线索

LocationEntry:
    location_id: String
    location_name: String       # 如 "码头仓库"
    clues: List[clue_id]       # 该地点的所有线索ID
    npcs: List[npc_id]         # 该地点的NPC ID
    completion_percentage: Float # 线索完成度

JournalView:
    mode: Enum                 # LOCATION / NPC / TIMELINE
    current_location_filter: String
    current_npc_filter: String
```

**规则5：任务进度验证 (Task Progress Validation)**

系统在以下时机检查任务进度：
1. 新线索被解锁时
2. NPC死亡时
3. 玩家主动请求检查时

检查逻辑：
```
if clues_discovered_for_task >= min_clues_required_for_task:
    unlock_next_task()
else if critical_clue_missing:
    trigger_compensation()  # 防止卡关的补偿机制
```

**规则6：补偿机制 (Compensation Mechanism)**

当关键线索因NPC死亡永久缺失时，系统触发补偿：

| 缺失类型 | 补偿方式 | 实现优先级 |
|---------|---------|-----------|
| 关键线索缺失（任务无法推进） | 自动给予1条替代线索（可从其他NPC或环境获取） | P0 必须 |
| 叙事完整性缺失（影响完美收集） | 日志中保留"[信息缺失—NPC已死亡]"标记 | P1 应该 |

补偿触发条件：
- `criticality == CRITICAL` AND `missing_reason != null`
- 且当前任务 `clues_discovered < min_clues_required`

**补偿线索池生成规则（OQ-3 已解决）**：

补偿线索池采用**混合策略**，在设计阶段由设计师手动预设关键线索的补偿路径：

| 线索类别 | 生成方式 | 说明 |
|---------|---------|------|
| **CRITICAL** | 设计师手动预设 | 每条 CRITICAL 线索必须预设至少 1 条替代补偿线索，确保叙事完整性 |
| **IMPORTANT** | 系统自动生成 | 从同任务、同类别、未被玩家发现的线索中自动抽取 |
| **OPTIONAL** | 系统自动生成 | 从同任务、未被玩家发现的所有线索中抽取 |

**设计师预设补偿线索的数据结构**：
```python
CompensationEntry:
    source_clue_id: String        # 原始关键线索 ID
    compensation_clue_ids: List[String]  # 补偿线索 ID 列表（按优先级排序）
    fallback_pool_size: int        # 自动填充的备用池大小（默认 2）
```

**补偿选择优先级**：
1. 优先使用设计师预设的补偿线索（按 `compensation_clue_ids` 顺序尝试）
2. 预设线索全部已获取时，使用系统自动生成的备用线索
3. 备用池为空或无法生成时，触发设计错误日志，允许临时通关

**示例**：
```json
{
  "source_clue_id": "ID_仓库密码",
  "compensation_clue_ids": ["ID_接头人透露密码", "ID_账本记载密码"],
  "fallback_pool_size": 2
}
```

### States and Transitions

#### 线索状态机 (Clue State Machine)

每条线索的独立状态机：

```
[NOT_DISCOVERED] ──获取来源触发──▶ [DISCOVERED] ──前置满足──▶ [UNLOCKED]
     │                                        │                      │
     │                                        ▼                      ▼
     │                                 [MISSING]              [COMPLETED]
     │                                    │
     │                              (NPC死亡触发)
     │                                       
[UNKNOWN ──NPC拥有该线索但未被发现──▶ MISSING]
```

| 状态 | 描述 | 可转移至 | 触发条件 |
|------|------|---------|---------|
| NOT_DISCOVERED | 线索尚未被玩家发现 | DISCOVERED | LOS窃听关键词完成/环境物件交互 |
| DISCOVERED | 线索已被发现但未解锁（需前置线索） | UNLOCKED | 前置线索已UNLOCKED |
| UNLOCKED | 线索已解锁，可阅读详细内容 | COMPLETED | 玩家打开日志查看该线索 |
| COMPLETED | 玩家已阅读线索 | 无（终止态） | 阅读动作完成 |
| MISSING | NPC死亡导致线索无法获取 | 无（终止态） | NPC死亡时，该NPC的knowledge中尚未DISCOVERED的线索直接标记为MISSING |

**状态转移规则**：
- NPC死亡时，如果该NPC的knowledge包含尚未DISCOVERED的线索 → 直接标记为 MISSING（不经过NOT_DISCOVERED状态）
- MISSING状态的线索不计入任务完成度，但会在日志中显示 `[信息缺失—NPC已死亡]`

#### 补偿触发状态

```
[COMPENSATION_PENDING] ──补偿条件满足──▶ [COMPENSATION_ACTIVE]
                                                │
                                                ▼
                                        [COMPENSATION_COMPLETE]
```

| 状态 | 描述 | 触发条件 |
|------|------|---------|
| COMPENSATION_PENDING | 等待补偿检查 | 关键线索缺失且任务无法推进 |
| COMPENSATION_ACTIVE | 补偿正在执行 | 系统给予替代线索 |
| COMPENSATION_COMPLETE | 补偿完成 | 替代线索已添加到日志 |

#### 任务进度状态 (Task Progress State)

| 状态 | 描述 | 转移条件 |
|------|------|---------|
| TASK_LOCKED | 任务未解锁 | 前置任务未完成 |
| TASK_ACTIVE | 任务进行中 | 玩家已进入该任务 |
| TASK_COMPLETE | 任务完成 | 已发现足够线索 |
| TASK_FAILED | 任务失败（关键线索永久缺失） | 补偿机制也无法恢复 |

### Interactions with Other Systems

#### 数据流入 (Inputs)

| 来源系统 | 数据内容 | 处理方式 |
|---------|---------|---------|
| **LOS系统** | `KeywordCapturedEvent` | 接收关键词，与已定义的线索模板匹配。如匹配成功，创建 `Clue` 实例，状态设为 DISCOVERED |
| **环境交互系统** | `IntelObjectInteracted(object_id, object_type, location_id)` | 情报类物件被交互时触发。直接创建对应 Clue，状态设为 DISCOVERED |
| **NPC AI系统** | `NPCStateChangedEvent` | NPC死亡时（new_state=DEAD）触发。遍历 NPC 的 knowledge，对于尚未 DISCOVERED 的线索，创建 Clue 并标记 `missing_reason: "NPC已死亡"` |
| **玩家控制器** | `PlayerEnteredLocation(location_id)` | 玩家进入新地点时，更新 Journal 的 `current_location_filter` |

#### 数据流出 (Outputs)

| 目标系统 | 发送数据 | 说明 |
|---------|---------|------|
| **任务/关卡系统** | `TaskProgressUpdated(task_id, completion_percentage)` | 任务进度变化时通知，用于解锁/锁定关卡目标 |
| **理智/愤怒系统** | `ClueDiscoveredEvent(clue_id, clue_category)` | 新线索被发现时通知，用于计算理智值变化（如发现悲剧线索可能导致理智下降） |
| **UI系统** | `JournalData(journal_view, current_clues)` | 日志UI需要的数据结构，按 LOCATION / NPC / TIMELINE 三种视图组织 |
| **LOS系统** | `RequestKeywordTemplate(clue_id)` | 请求LOS系统提供特定线索对应的关键词模板（用于窃听匹配） |

#### 接口所有权

| 接口 | 拥有者 | 流向 |
|------|-------|------|
| `KeywordCapturedEvent` | LOS系统 | LOS → 线索系统 |
| `IntelObjectInteracted` 事件 | 环境交互系统 | 环境 → 线索系统 |
| `NPCStateChangedEvent` | NPC AI系统 | NPC AI → 线索系统（NPC死亡时触发） |
| `ClueDiscoveredEvent` 事件 | 线索系统 | 线索系统 → 理智系统 |
| `TaskProgressUpdated` 事件 | 线索系统 | 线索系统 → 任务系统 |
| `JournalData` 查询接口 | 线索系统 | UI系统 ← 线索系统 |

#### 跨系统边界澄清

* **LOS系统 vs 线索系统**：LOS负责"提取关键词"，线索系统负责"将关键词转化为线索"。关键词是原材料，线索是成品。
* **NPC AI系统 vs 线索系统**：NPC的`knowledge`字段本身已经是"线索ID列表"（已加工的成品），所以NPC死亡时是直接转移线索，而非再加工。
* **环境交互系统 vs 线索系统**：情报类物件交互后直接产出成品线索，无需经过"关键词→线索"的加工流程。

## Formulas

**公式1：线索解锁前置判定**

判断某条线索是否满足前置条件可以解锁：

```
CanUnlock = All(precondition_clue_ids.every(clue_id =>
    Clue[clue_id].state == COMPLETED
))
```

其中 `precondition_clue_ids` 来自线索数据结构的 `prerequisites` 字段。

**公式2：任务完成度计算**

```
TaskCompletionPercentage = (
    clues_discovered_count + clues_missing_count
) / total_clues_in_task * 100
```

注意：`missing_reason != null` 的线索计入已完成基数，但不计入可用线索。

**公式3：补偿触发判定**

```
ShouldCompensate = (
    critical_clue_missing == true
) AND (
    clues_discovered_for_task < min_clues_required
)
```

**公式4：补偿替代线索选择**

当触发补偿时，从预设的补偿线索池中选择替代线索：

```
AvailableCompensationClues = CompensationPool.Filter(clue =>
    clue.task_id == current_task_id
    AND clue not in Journal
).Take(1)

AddToJournal(AvailableCompensationClues.First())
```

补偿线索池由策划预设，确保每条关键线索都有至少一条替代路径。

**公式5：地点完成度计算**

```
LocationCompletion = clues_in_location.Filter(clue =>
    clue.state == COMPLETED
).Count() / clues_in_location.TotalCount() * 100
```

**公式6：理智值影响计算（与理智系统联动）**

```
SanityDelta = BaseValue[clue.category] * ClueContextMultiplier
```

**BaseValue（与理智系统对齐）**：

| 线索类别 | BaseValue | 说明 |
|---------|-----------|------|
| Identity (身份线索) | +5 | 确认敌人身份 = 复仇进展 |
| Location (位置线索) | +10 | 获得新地点 = 重大进展 |
| Relationship (关系线索) | 0 | 中性信息 |
| Item (物品线索) | +5 | 获得关键物品 |
| Tragedy (悲剧线索) | -10 ~ -20 | 发现悲剧 = 情感代价 |

**ClueContextMultiplier**（避免与理智系统的 ContextMultiplier 混淆）：
- first_discovery: ×1.0
- related_to_main_target: ×1.5
- reveals_npc_sympathy: ×2.0

**与理智系统的数值对齐**：
- 悲剧线索的 -10 ~ -20 范围与理智系统的 `TragedyClueSanityPenalty_Min/Max` 参数一致
- 身份/位置线索的 +5/+10 与理智系统的 BaseValue 一致
- 本系统的 `ClueDiscoveredEvent` 事件携带 `clue_category`，理智系统据此计算具体的 SanityDelta

## Edge Cases

**边缘情况1：多条前置线索同时解锁**

*问题*：某条线索有多个前置条件，这些前置条件在同一帧内全部完成解锁。

*处理*：前置判定使用 `All()` 条件，同一帧完成多个前置也会触发解锁。无需特殊处理。

---

**边缘情况2：NPC死亡但部分线索已被玩家获取**

*问题*：NPC死亡时，其knowledge列表中的部分线索已经被玩家收集（状态为COMPLETED），剩余线索尚未收集。

*处理*：只对尚未DISCOVERED的线索创建MISSING条目。已完成的线索不受影响。

---

**边缘情况3：补偿线索池为空**

*问题*：触发补偿机制时，当前任务的补偿池中没有可用线索。

*处理*：记录设计错误（`DesignError` 日志），临时给予直接通关权限（`OverrideTaskCompletion()`）。这种情况不应在生产环境中发生，说明设计时遗漏了该关键线索的补偿路径。

---

**边缘情况4：同一NPC多次死亡（不可能发生但需防御）**

*问题*：理论上NPC死亡后不应再触发第二次 `NPCStateChangedEvent`（new_state=DEAD）。

*处理*：NPC AI系统保证只触发一次死亡事件。线索系统收到重复事件时忽略并记录警告。

---

**边缘情况5：线索解锁前置形成循环依赖**

*问题*：线索A需要线索B的前置，线索B需要线索A的前置（理论上不应发生）。

*处理*：在关卡设计验证阶段（`/balance-check`）检测循环依赖并报错。运行时不做循环检测，假设设计阶段已排除。

---

**边缘情况6：玩家在查看线索时触发补偿**

*问题*：玩家正在阅读日志时，系统触发补偿并添加新线索。

*处理*：新添加的线索在当前会话中不立即显示在列表中（避免打乱阅读顺序）。在玩家关闭并重新打开日志后可见。

---

**边缘情况7：同一关键词匹配多条线索模板**

*问题*：LOS系统传递的关键词可能同时匹配多条线索。

*处理*：关键词匹配是**一对多**关系。一个关键词可以解锁多条线索。遍历所有匹配的线索模板，逐一创建Clue实例。

---

**边缘情况8：环境物件交互发生在NPC死亡之后**

*问题*：玩家先杀死NPC获得其knowledge，然后才与环境物件交互。如果物件与NPC的knowledge有重叠线索，是否重复？

*处理*：线索系统维护 `discovered_clue_ids` 集合。创建新Clue前检查 `clue_id` 是否已存在。已存在的直接忽略，保证幂等性。

---

**边缘情况9：任务跨多个地点**

*问题*：单个任务可能跨越多个地点，地点完成度如何计算？

*处理*：任务完成度基于**线索数量**而非地点完成度。地点完成度仅用于UI显示该地点的探索进度。

---

**边缘情况10：玩家删除/回滚存档导致线索状态不一致**

*问题*：玩家加载旧存档，导致线索状态与当前游戏世界状态不一致。

*处理*：每次保存存档时，同时保存 `Journal` 的完整快照。加载存档时恢复Journal状态，确保线索状态与存档一致。

## Dependencies

### 上游依赖（线索系统依赖谁）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **LOS系统** | 硬依赖 | 接收 `KeywordCapturedEvent`，获取窃听关键词作为线索原材料 |
| **环境交互系统** | 硬依赖 | 接收 `IntelObjectInteracted` 事件，获取情报类物件线索 |
| **NPC AI系统** | 硬依赖 | 接收 `NPCStateChangedEvent`（new_state=DEAD），获取NPC的knowledge转移 |
| **玩家控制器** | 软依赖 | 接收 `PlayerEnteredLocation` 事件，更新当前地点过滤 |

### 下游依赖（谁依赖线索系统）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **理智/愤怒系统** | 软依赖 | 接收 `ClueDiscoveredEvent` 事件，根据线索类型计算理智值变化 |
| **任务/关卡系统** | 硬依赖 | 接收 `TaskProgressUpdated` 事件，根据线索完成度解锁关卡 |
| **UI系统** | 硬依赖 | 提供 `JournalData` 查询接口，供日志UI渲染 |
| **LOS系统** | 软依赖 | 接收 `RequestKeywordTemplate` 请求，提供线索对应的关键词模板 |

### 依赖关系矩阵

```
┌────────────────┐
│   LOS系统      │────KeywordCapturedEvent──▶┌────────────────┐
│                │                        │               │
│   NPC AI系统   │────NPCStateChangedEvent─▶│ 线索与日志系统 │
│                │                        │               │
│   环境交互系统 │────IntelInteracted───▶ │               │
└────────────────┘                        │               │
                                          │               ▼
                                          │        ┌───────────┐
                                          │        │ 理智系统   │◀──ClueDiscoveredEvent
                                          │        └───────────┘
                                          │               │
                                          │               ▼
                                          │        ┌───────────┐
                                          └───────▶│ 任务系统   │◀──TaskProgressUpdated
                                                   └───────────┘
```

### 关键设计约束

1. **线索系统不处理身份标记**：NPC的身份标签（Enemy/Accomplice/Victim）由LOS系统处理，线索系统只负责情报内容
2. **线索系统不拥有任务进度**：任务进度的最终判定由任务/关卡系统控制，线索系统只提供完成度数据
3. **理智值变化是单向通知**：线索系统通知理智系统"发现了什么"，但不等待理智系统的响应（避免循环依赖）

### 关键词匹配规则表（OQ-2 已解决）

**匹配机制**：LOS系统捕获的关键词（Keyword）与线索模板中的关键词列表进行**一对一匹配**。

```python
# 匹配伪代码
def MatchKeywordToClue(keyword: string, npc_id: string, location_id: string) -> List[Clue]:
    matched_clues = []
    for clue_template in ClueTemplates:
        # 筛选来源为LOS且关键词列表包含该keyword的模板
        if clue_template.source_type == LOS_EAVESDROP:
            if keyword in clue_template.keyword_list:
                # 一对多：同一个关键词可能匹配多条线索
                matched_clues.append(CreateClueFromTemplate(clue_template, npc_id, location_id))
    return matched_clues
```

**关键词模板数据结构**：
```python
KeywordTemplate:
    clue_id: String           # 关联的线索ID
    keyword_list: List[String] # 触发该线索的关键词列表（OR关系）
    min_evidence_count: int    # 最少需要捕获的关键词数量（默认1）
```

**一对一与一对多匹配示例**：

| 关键词 | 匹配的线索 | 说明 |
|--------|-----------|------|
| "货" | ID_仓库位置, ID_运输计划 | 一对多：同一关键词揭示多条线索 |
| "张三" + "二把手" | ID_张三身份 | 一对多：多个关键词共同指向同一线索（需全部捕获） |
| "母亲" | ID_帮凶背景 | 一对一 |

**匹配优先级**：
1. 精确匹配 > 部分匹配（"仓库密码" 优先于 "仓库"）
2. 多关键词全部匹配 > 部分匹配
3. 同NPC来源 > 不同NPC来源

**关键词捕获后的处理流程**：
```
LOS系统捕获关键词
    │
    ▼
过滤出来自同一NPC的关键词
    │
    ▼
检查已捕获关键词是否满足线索模板的 min_evidence_count
    │
    ├── 满足 ──▶ 创建Clue实例，状态设为 DISCOVERED
    │
    └── 不满足 ──▶ 累积到下一关键词捕获时再判定
```

### 双向依赖检查

| 系统对 | 依赖关系 | 是否双向 | 解决方案 |
|--------|---------|---------|---------|
| 线索 ↔ 理智 | 线索通知理智 | 否 | 线索系统主动推送，理智系统被动接收 |
| 线索 ↔ 任务 | 线索更新任务进度 | 否 | 线索系统主动推送，任务系统被动接收 |
| 线索 ↔ LOS | 线索请求关键词模板 | 是 | 仅在需要时查询，不形成循环 |

**不存在双向循环依赖**：线索系统是数据的"消费者"（接收LOS/NPC/环境的线索）和"分发者"（向理智/任务/UI推送数据），不依赖下游系统的回调。

## Tuning Knobs

*以下参数暴露给策划在引擎 Inspector 中直接调整，无需修改代码。*

### 线索数量参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `CluesPerLocation_Min` | int | 3 | 1-10 | 每个地点的最少线索数量 |
| `CluesPerLocation_Max` | int | 8 | 5-20 | 每个地点的最多线索数量 |
| `CriticalCluesPerTask_Ratio` | float | 0.3 | 0.1-0.5 | 关键线索占总线索的比例 |
| `OptionalCluesPerTask_Ratio` | float | 0.4 | 0.2-0.6 | 可选线索占总线索的比例 |

### 任务进度参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MinCluesToCompleteTask` | int | 2 | 1-5 | 完成任务所需的最少线索数 |
| `CompensationCluesPoolSize` | int | 3 | 1-10 | 每个任务的补偿线索池大小 |

### 解锁与前置参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MaxPrerequisitesPerClue` | int | 3 | 1-5 | 每条线索的最大前置数量 |
| `MaxChainDepth` | int | 5 | 2-10 | 线索链的最大深度（防止极端设计） |

### UI参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `JournalSortDefault` | enum | LOCATION | — | 默认排序方式 |
| `MaxCluesDisplayedPerPage` | int | 20 | 10-50 | 日志每页最多显示线索数 |

### 理智值联动参数（占位符，待理智系统确认）

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `SanityDelta_Identity` | int | +5 | -10~+10 | 发现身份线索的理智变化 |
| `SanityDelta_Location` | int | +10 | -10~+20 | 发现位置线索的理智变化 |
| `SanityDelta_Tragedy` | int | -15 | -30~-5 | 发现悲剧线索的理智变化 |
| `SanityDelta_FirstDiscoveryMultiplier` | float | 1.0 | 0.5-2.0 | 首次发现乘数 |
| `SanityDelta_MainTargetMultiplier` | float | 1.5 | 1.0-3.0 | 与主要目标相关乘数 |

### 调参风险提示

- `MinCluesToCompleteTask` 设置过低 → 玩家可跳过大量内容，削弱探索深度
- `MinCluesToCompleteTask` 设置过高 → 可能导致卡关（尤其在有缺失线索的情况下）
- `CriticalCluesPerTask_Ratio` 设置过高 → 补偿机制压力增大
- `OptionalCluesPerTask_Ratio` 设置过高 → 玩家可能错过核心信息

## Visual/Audio Requirements

### 视觉反馈

| 事件 | 视觉反馈 |
|------|---------|
| 新线索被发现 | 屏幕边缘闪烁淡蓝色光晕，日志图标出现脉冲动画 |
| 线索从 DISCOVERED → UNLOCKED | 线索卡片从灰色渐变为正常颜色，伴随轻微弹跳 |
| 线索被标记为 MISSING | 卡片显示红色"[信息缺失]"标签，带删除线效果 |
| 任务完成 | 屏幕中央弹出"线索收集完成"提示，伴随金色光效 |
| 触发补偿机制 | 屏幕边缘橙色闪烁，日志图标出现"+1"浮出动画 |

### 视觉层级

- **未发现线索**：完全隐藏，不显示
- **已发现但未解锁**：灰色卡片，显示标题和锁定图标
- **已解锁未阅读**：正常颜色，卡片右上角有"NEW"标签
- **已阅读完成**：正常颜色，无标签
- **缺失线索**：灰色卡片，显示"[信息缺失—NPC已死亡]"，带红色删除线

### 音效反馈

| 事件 | 音效类型 | 示例 |
|------|---------|------|
| 新线索被发现 | 收集音效 | 清脆的"叮"声，略带神秘感 |
| 线索解锁 | 解锁音效 | 低沉的"咔嗒"声 |
| 线索标记为缺失 | 缺失音效 | 沉重的"咚"声，带回音 |
| 任务完成 | 成就音效 | 上升音阶的弦乐 |
| 补偿触发 | 补偿音效 | 警报音+柔和的补偿音效 |

### 色调规范

| 线索状态 | 主色调 | 辅助色 |
|---------|-------|--------|
| 身份线索 | 深红 | 暗金 |
| 位置线索 | 蓝色 | 银色 |
| 关系线索 | 紫色 | 灰色 |
| 物品线索 | 绿色 | 铜色 |
| 悲剧线索 | 灰紫色 | 暗红 |

## UI Requirements

### 日志UI结构

```
┌──────────────────────────────────────────────────┐
│  [地点视图] [人物视图] [时间线视图]    [关闭]    │  ← 视图切换Tab
├──────────────────────────────────────────────────┤
│ ┌─────────┐ ┌─────────────────────────────────┐ │
│ │ 地点列表 │ │         线索列表                 │ │
│ │         │ │                                 │ │
│ │ ▼码头仓库│ │ ┌─────────────────────────────┐ │ │
│ │  ▸ 3/5  │ │ │ [身份] 张三的身份            │ │ │
│ │         │ │ │ 张三是青龙帮的二把手          │ │ │
│ │  ▼码头住宅│ │ └─────────────────────────────┘ │ │
│ │   ▸ 1/3 │ │ ┌─────────────────────────────┐ │ │
│ │         │ │ │ [缺失] 李四的背景    [信息缺失]│ │ │
│ │  ▼未知区域│ │ │ NPC已死亡 — 无法获取         │ │ │
│ │   ▸ 0/2 │ │ └─────────────────────────────┘ │ │
│ └─────────┘ └─────────────────────────────────┘ │
├──────────────────────────────────────────────────┤
│ 任务进度: 2/5 线索                              │
└──────────────────────────────────────────────────┘
```

### 三种视图模式

| 视图 | 描述 | 适用场景 |
|------|------|---------|
| **地点视图** | 按关卡/据点组织线索（主视图） | 追踪探索进度 |
| **人物视图** | 按NPC组织线索 | 追踪人物关系和悲剧故事 |
| **时间线视图** | 按发现顺序组织线索 | 回顾调查过程 |

### 线索卡片内容

| 字段 | 显示规则 |
|------|---------|
| 类别标签 | 左上角彩色标签（如[身份]） |
| 标题 | 加粗字体 |
| 描述 | 普通字体，解锁后可见 |
| 来源信息 | 右下角小字（"来自：张三的knowledge"） |
| 缺失标记 | 红色"[信息缺失—NPC已死亡]"，带删除线 |

### 交互提示

- **线索卡片悬停**：显示完整描述（如果已UNLOCKED）
- **点击线索卡片**：展开/收起详情（如果已UNLOCKED）
- **未解锁线索点击**：显示"需要先发现：[前置线索标题]"
- **缺失线索点击**：显示"该NPC已死亡，信息无法获取"

### HUD元素

- **日志图标**：屏幕右下角，显示已收集线索数量
- **新线索提示**：图标脉冲动画 + 屏幕边缘光晕
- **任务进度**：进入日志时显示当前任务的完成度

## Acceptance Criteria

### 功能验收

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-1 | LOS窃听提取关键词后，对应线索正确添加到日志 | 监听NPC完成关键词提取，检查日志中是否出现对应线索 |
| AC-2 | 环境物件交互后，对应线索正确添加到日志 | 交互情报类物件，检查日志中是否出现对应线索 |
| AC-3 | NPC死亡后，其knowledge中的线索正确标记为MISSING | 杀死NPC，检查日志中是否出现"[信息缺失—NPC已死亡]"标记 |
| AC-4 | 线索解锁前置条件满足时，线索自动从DISCOVERED转为UNLOCKED | 完成前置线索收集，检查后置线索是否自动解锁 |
| AC-5 | 补偿机制在关键线索缺失且任务无法推进时正确触发 | 模拟关键NPC死亡且无替代线索，验证补偿线索是否添加 |
| AC-6 | 日志UI三种视图模式正确切换显示 | 在日志中切换地点/人物/时间线视图，验证显示内容正确 |
| AC-7 | 任务完成度正确计算并通知任务系统 | 收集不同数量线索，验证完成度百分比和通知时机 |

### 跨系统验收

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-8 | 线索被发现时，理智系统接收到ClueDiscoveredEvent | 监听事件总线，验证事件格式和触发时机 |
| AC-9 | 任务进度更新时，任务系统接收到TaskProgressUpdated | 监听事件，验证任务系统收到正确的完成度 |
| AC-10 | UI系统能正确查询并渲染JournalData | 打开日志，检查显示内容与数据结构一致 |

### 边缘情况验收

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-11 | 同一关键词匹配多条线索时，全部正确添加 | 设置一个关键词对应多条线索的模板，验证全部添加 |
| AC-12 | 已存在的线索不会因重复事件重复添加 | 触发两次相同来源的事件，验证只有一条线索 |
| AC-13 | NPC死亡时，已被收集的线索不受影响 | 先收集某NPC的knowledge，再杀死该NPC，验证已收集线索状态不变 |

### 性能验收

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-14 | 单帧内处理10条线索添加不造成帧率下降 | 批量触发多个线索添加事件，观察帧率 |
| AC-15 | 日志UI切换视图响应时间 < 100ms | 切换视图，测量响应时间 |

### 边界条件验收

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-16 | 补偿线索池为空时，系统不崩溃并记录DesignError | 清空补偿池后触发补偿，验证系统行为 |
| AC-17 | 存档加载后，线索状态与存档一致 | 保存存档，修改游戏状态，加载存档，验证线索状态恢复 |

## Open Questions

| # | 问题 | 状态 | 负责人 | 说明 |
|---|------|------|--------|------|
| OQ-1（已解决） | **理智值变化数值确认** | ✅ 已解决 | 理智系统设计师 | BaseValue 已与理智系统对齐。悲剧线索 -10~-20 与理智系统的 TragedyClueSanityPenalty_Min/Max 一致。Identity +5, Location +10 与理智系统 BaseValue 一致。 |
| OQ-2（已解决） | **关键词→线索的匹配机制** | ✅ 已解决 | — | 见下方「关键词匹配规则表」详细定义 |
| OQ-3 | ~~**补偿线索池的生成策略**~~ | ✅ **已解决** | 游戏设计师 | **混合策略**：CRITICAL 线索由设计师手动预设补偿线索（每条至少 1 条替代路径，确保叙事完整性）；IMPORTANT/OPTIONAL 线索由系统自动从同任务、同类别的未使用线索中抽取。详见下方「补偿线索池生成规则」。 |
| OQ-4 | **日志UI的美术风格** | 待确认 | 美术设计师 | 日志UI的具体美术风格尚未确定——是复古笔记本风格还是现代UI风格？ |
| OQ-5 | **时间线视图的实现复杂度** | 待确认 | 游戏设计师/UI设计师 | MVP是否需要实现时间线视图？如果需要，MVP版本是否可以简化为"按获取顺序显示"？ |
| OQ-6 | **NPC人物视图的信息层级** | 待确认 | 叙事设计师 | 人物视图是否需要显示NPC的完整背景故事？还是只显示与线索相关的信息？ |

### 已解决的设计决策

| 决策项 | 最终方案 | 决定日期 |
|--------|---------|----------|
| Keywords vs knowledge 语义边界 | Keywords是LOS窃听的原材料，knowledge是NPC已掌握的成品线索，两者独立 | 2026-04-06 |
| 补偿线索池生成策略（OQ-3） | CRITICAL线索由设计师手动预设补偿路径；IMPORTANT/OPTIONAL线索由系统自动从同任务未发现线索中抽取 | 2026-04-07 |
