# ADR-0016: 线索与日志系统 (Clue & Journal) 架构决策

## Status
**Accepted**

## Date
2026-04-10

## Last Updated
2026-04-15 (v7: ADR 评审验证：VulnerabilityUncoveredEvent 订阅关系与 ADR-0018 一致；KeywordCapturedEvent 流向正确)

## Context

### Problem Statement

线索与日志系统是《断绝：罪恶之源》"罪恶的深度"支柱的核心交付系统。玩家通过收集线索来"拼凑真相"，但每一个冲动行为（如杀死 NPC）都可能导致关键线索永久缺失，形成"选择即代价"的情感体验。系统需要处理三种截然不同的线索来源（LOS 窃听关键词、环境物件交互、NPC 知识转移），并维护复杂的依赖/补偿关系。

### Constraints

- **数据一致性约束**：同一线索不能重复添加，NPC 死亡后其已转移的线索状态必须不可逆
- **防止卡关约束**：关键线索缺失时必须触发补偿机制，确保玩家能继续推进
- **性能约束**：单帧处理 10+ 线索添加不造成帧率下降，日志 UI 视图切换 < 100ms
- **叙事一致性约束**：补偿不能消除遗憾，必须保留"[信息缺失—NPC已死亡]"标记

### Requirements

- **必须**：支持三种来源的线索获取（LOS_EAVESDROP / ENVIRONMENT / NPC_DEATH）
- **必须**：定义线索状态机（NOT_DISCOVERED → DISCOVERED → UNLOCKED → COMPLETED / MISSING）
- **必须**：定义补偿机制防止卡关（CRITICAL 线索预设替代路径）
- **必须**：定义 JournalData 查询接口供 UI 系统消费
- **必须**：定义 ClueDiscoveredEvent 向理智系统传递线索发现上下文

---

## Decision

### 架构决策

采用**事件驱动的线索中心架构**，线索系统作为情报枢纽处理来自 LOS 系统、环境交互系统、NPC AI 系统的原始数据，转化为统一的线索实体后分发给下游系统。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Clue & Journal System 架构                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【上游数据源】                                                               │
│                                                                              │
│  ┌──────────────┐    KeywordCapturedEvent    ┌──────────────────────────┐   │
│  │   LOS系统    │ ─────────────────────────▶  │                          │   │
│  └──────────────┘                             │     ClueTemplate        │   │
│                                              │     Repository           │   │
│  ┌──────────────┐    IntelInteracted         │     (线索模板仓库)        │   │
│  │ 环境交互系统  │ ─────────────────────────▶  │                          │   │
│  └──────────────┘                             └────────────┬─────────────┘   │
│                                                              │                 │
│  ┌──────────────┐    NPCStateChangedEvent(DEAD)             ▼                 │
│  │  NPC AI系统  │ ─────────────────────────▶  ┌──────────────────────────┐   │
│  └──────────────┘                             │     ClueFactory          │   │
│                                              │     (线索实例工厂)        │   │
│  【下游消费者】                                └────────────┬─────────────┘   │
│                                                              │                 │
│  ┌──────────────┐    ClueDiscoveredEvent       ┌────────────▼─────────────┐ │
│  │  理智/愤怒系统 │◀────────────────────────── │      JournalManager      │ │
│  └──────────────┘                             │      (日志管理器)          │ │
│                                              │  ◆ MonoBehaviour 单例     │ │
│  ┌──────────────┐    LocationRevealedEvent     │  ◆ 维护 Clue 集合         │ │
│  │  世界地图系统  │◀────────────────────────── │  ◆ 状态机驱动             │ │
│  └──────────────┘                             └────────────┬─────────────┘   │
│                                                              │                 │
│  ┌──────────────┐    JournalData (查询接口)    ┌────────────▼─────────────┐ │
│  │   UI系统     │◀────────────────────────── │     JournalDatabase     │ │
│  └──────────────┘                             │     (持久化存储)          │ │
│                                              └───────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 核心数据结构

#### ClueTemplate（线索模板）

```csharp
/// <summary>
/// 线索模板（由 Narrative Director 策划配置）
/// ClueTemplateRepository 持有所有模板的加载和查询接口
/// </summary>
public class ClueTemplate
{
    public string ClueId;                    // 唯一标识符
    public ClueCategory Category;            // IDENTITY / LOCATION / RELATIONSHIP / ITEM / TRAGEDY
    public string Title;                      // 显示标题
    public string Description;                // 详细内容（解锁后可见）
    public ClueSourceType SourceType;        // LOS_EAVESDROP / ENVIRONMENT / NPC_DEATH_SOURCE / NPC_KNOWLEDGE_BONUS
    public string LocationId;                // 所属地点ID
    public List<string> RelatedNpcIds;      // 关联的NPC ID列表
    public string TaskId;                    // 关联的任务ID
    public ClueCriticality Criticality;      // CRITICAL / IMPORTANT / OPTIONAL
    public List<string> Prerequisites;        // 前置线索ID列表

    /// <summary>
    /// 触发关键词（当 LOS System 捕获到此关键词时，创建对应 Clue 实例）
    /// 仅 SourceType == LOS_EAVESDROP 时使用
    /// </summary>
    public string TriggerKeyword;
}
```

**Keyword→ClueTemplate 映射机制**：

ClueFactory 订阅 `KeywordCapturedEvent` 后，通过 ClueTemplateRepository 查询匹配的 ClueTemplate：

```csharp
// ClueTemplateRepository.cs
public class ClueTemplateRepository
{
    // ClueId → ClueTemplate 映射（用于精确查找）
    private Dictionary<string, ClueTemplate> _templates = new();

    // TriggerKeyword → ClueTemplate 映射（用于 Keyword 触发查找）
    private Dictionary<string, ClueTemplate> _keywordToTemplate = new();

    /// <summary>
    /// 根据触发关键词查找对应的 ClueTemplate
    /// </summary>
    public ClueTemplate GetTemplateByKeyword(string keyword)
    {
        if (_keywordToTemplate.TryGetValue(keyword, out var template))
            return template;

        Debug.LogWarning($"[ClueTemplateRepository] 未找到 keyword='{keyword}' 对应的 ClueTemplate");
        return null;
    }

    private void BuildKeywordIndex()
    {
        _keywordToTemplate.Clear();
        foreach (var template in _templates.Values)
        {
            if (!string.IsNullOrEmpty(template.TriggerKeyword))
            {
                _keywordToTemplate[template.TriggerKeyword] = template;
            }
        }
    }
}

// ClueFactory.cs
private void OnKeywordCaptured(KeywordCapturedEvent evt)
{
    var template = _repository.GetTemplateByKeyword(evt.keyword);
    if (template == null) return;

    var clue = new Clue
    {
        ClueId = template.ClueId,
        Category = template.Category,
        Title = template.Title,
        Description = template.Description,
        SourceType = ClueSourceType.LOS_EAVESDROP,
        SourceId = evt.npc_id.ToString(),
        LocationId = template.LocationId,
        RelatedNpcIds = template.RelatedNpcIds,
        TaskId = template.TaskId,
        Criticality = template.Criticality,
        Prerequisites = template.Prerequisites,
        State = ClueState.NOT_DISCOVERED
    };

    Journal.AddClue(clue);
}
```

#### Clue 实体

```csharp
public class Clue
{
    public string ClueId;                    // 唯一标识符
    public ClueCategory Category;            // IDENTITY / LOCATION / RELATIONSHIP / ITEM / TRAGEDY
    public string Title;                      // 显示标题
    public string Description;                // 详细内容（解锁后可见）
    public ClueSourceType SourceType;        // LOS_EAVESDROP / ENVIRONMENT / NPC_DEATH_SOURCE / NPC_KNOWLEDGE_BONUS（定义见 shared-types.md §14.1）
    public string SourceId;                  // 来源标识（NPC_ID 或物件ID）
    public string LocationId;                // 所属地点ID
    public List<string> RelatedNpcIds;      // 关联的NPC ID列表
    public string TaskId;                    // 关联的任务ID
    public ClueCriticality Criticality;      // CRITICAL / IMPORTANT / OPTIONAL
    public List<string> Prerequisites;        // 前置线索ID列表
    public ClueState State;                  // 当前状态
    public MissingReasonType MissingReason;  // 线索缺失原因（取代 string 类型）
    public bool IsPallidReplacement;        // 是否为苍白替代（补偿获得）
}

public enum ClueState
{
    NOT_DISCOVERED,
    DISCOVERED,
    UNLOCKED,
    COMPLETED,
    MISSING
}

/// <summary>
/// 线索类别（定义于 shared-types.md §11.1）
/// </summary>
/// <remarks>
/// ClueCategory 枚举已在 shared-types.md 中统一定义，本文档仅做引用说明：
/// - IDENTITY：身份线索
/// - LOCATION：位置线索
/// - RELATIONSHIP：关系线索
/// - ITEM：物品线索
/// - TRAGEDY：悲剧线索
/// </remarks>
public enum ClueCategory
{
    IDENTITY,
    LOCATION,
    RELATIONSHIP,
    ITEM,
    TRAGEDY
}

/// <summary>
/// 线索缺失原因类型
/// 定义于 shared-types.md §5.8
/// </summary>
public enum MissingReasonType
{
    NONE,           // 正常状态，无缺失
    NPC_DEAD,        // NPC 已死亡，线索随其知识一同消失
    PLAYER_CHOICE,  // 玩家主动放弃（如拒绝任务）
    ZONE_LOCKED,    // 区域未解锁
    QUEST_FAILED     // 关联任务失败
}
```

#### Journal 聚合根

```csharp
public class Journal
{
    public Dictionary<string, LocationEntry> Locations;  // 地点 → 线索索引
    public Dictionary<string, Clue> Clues;                 // ClueId → Clue 实体
    // 注意：missing_clue_ids 无需单独维护，可通过 Clues.Values.Where(c => c.MissingReason != null) 推导
}

public class LocationEntry
{
    public string LocationId;
    public string LocationName;
    public List<string> ClueIds;
    public float CompletionPercentage;
}
```

#### 组件职责定义

**ClueTemplateRepository（线索模板仓库）**
- 职责：负责 ClueTemplate 的加载、验证、存储
- 在加载时执行循环依赖检测（HasCircularDependency）
- 提供模板查询接口（GetTemplateById）
- Build 时 + 运行时加载前各执行一次验证

**ClueFactory（线索实例工厂）**
- 职责：负责将事件转换为 Clue 实例
- 订阅上游事件（KeywordCapturedEvent / IntelInteracted / NPCStateChanged）
- 根据 SourceType 创建对应 Clue 实例
- 处理 NPC 死亡时的 MISSING 标记逻辑

**JournalManager（日志管理器）**
- 职责：维护 Journal 聚合根，管理 Clue 状态机
- 订阅 ClueFactory 产生的 Clue 实例
- 执行补偿机制判定
- 提供 JournalData 查询接口

> **类型定义说明**：本 ADR 中引用的以下枚举/事件已统一定义于 shared-types.md：
> - `MissingReasonType`（§5.8）
> - `ClueCategory`（§11.1）
> - `DiscoveryStage`（§11.2）
> - `NarrativeSignificance`（§11.3）
> - `ClueDiscoveredEvent`（§11.4）
> - `ClueSourceType`（§14.1）
>
> 实现时应从 shared-types 引用，而非在本 ADR 中重复定义。

#### CompensationEntry 补偿映射

```csharp
public class CompensationEntry
{
    public string SourceClueId;              // 原始关键线索 ID
    public List<string> CompensationClueIds; // 补偿线索 ID 列表（支持 1:N 补偿池）
}
```

> **P0 验证要求**：`SourceClueId` 对应的线索必须是 `Criticality.CRITICAL` 类型。
> 在加载 `CompensationEntry` 时必须验证此约束，若不满足则输出错误日志并拒绝加载。
> 实现示例：
> ```csharp
> public void AddCompensation(CompensationEntry entry)
> {
>     var sourceClue = GetClue(entry.SourceClueId);
>     if (sourceClue == null || sourceClue.Criticality != ClueCriticality.CRITICAL)
>     {
>         Debug.LogError($"[ClueJournal] Compensation source clue {entry.SourceClueId} must be CRITICAL");
>         return;
>     }
>     // ... 后续逻辑
> }
> ```

### 线索状态机

**状态转移图（文字版）**：
```
NOT_DISCOVERED --获取来源触发--> DISCOVERED --前置满足--> UNLOCKED
      |                               |                        |
      |                               v                        v
      |                          [MISSING]              [COMPLETED]
      |                             |                     (终止态)
      |                       (NPC死亡触发)
      |                             |
      |                             v
      |                    [补偿触发] --> [添加新 Clue，is_pallid_replacement=true]
      |                             |      原 MISSING 标记保留，不发生状态转移
      |                       (副作用)
```

**状态转移表**：

| 当前状态 | 触发条件 | 目标状态 | 说明 |
|---------|---------|---------|------|
| NOT_DISCOVERED | 获取来源触发 | DISCOVERED | 玩家获取到线索来源 |
| DISCOVERED | 前置线索满足 | UNLOCKED | 前置条件达成，线索可阅读 |
| DISCOVERED | NPC死亡且该NPC是来源 | MISSING | 线索随NPC知识一同消失（仅限NOT_DISCOVERED状态的线索） |
| UNLOCKED | 玩家确认/任务完成/超时 | COMPLETED | 玩家消化完线索信息（终止态） |
| MISSING | 补偿触发 | - | 不改变原状态，作为副作用添加补偿线索 |

**状态转移规则**：
- NPC 死亡时，如果该 NPC 的 knowledge 包含状态为 `NOT_DISCOVERED` 的线索 → 直接标记为 MISSING（终止态）
  > **范围说明**：仅检查 `NOT_DISCOVERED` 状态。已处于 `DISCOVERED`/`UNLOCKED`/`COMPLETED` 状态的线索不受影响，维持原状态。这是合理的，因为已"发现"的线索已经存在于玩家记忆中，不会因 NPC 死亡而消失。
- MISSING 是逻辑终止态，补偿触发**不改变原线索状态**，而是作为**副作用**向 Journal 添加一条新的 `is_pallid_replacement=true` 的补偿线索
- 原 MISSING 标记**不消失**，确保玩家始终意识到"原本可以知道但现在永远不知道"的遗憾

### 线索完成机制（COMPLETED 触发条件）

`COMPLETED` 是线索的逻辑终点，表示玩家已完整消化该线索的信息。

**触发条件优先级**（高优先级先检查）：
| 优先级 | 触发条件 | 说明 |
|--------|---------|------|
| 1（最高） | **玩家主动确认** | 玩家在 Journal UI 中打开线索详情并点击"确认"按钮，标记为已阅读 |
| 2 | **关联任务完成** | 线索关联的任务（`task_id`）达到 `COMPLETED` 状态时，自动标记 |
| 3 | **时间自动完成** | 线索处于 `UNLOCKED` 状态超过预设时间阈值（默认 5 分钟）且玩家未查看，自动标记 |

> **优先级说明**：当多个条件同时满足时，按优先级高优先处理。例如，当玩家主动确认线索后，关联任务也完成了，不会重复触发状态转移（因为线索已进入 COMPLETED 终止态）。任务完成和时间自动完成两个条件在玩家未主动确认的情况下作为"兜底"机制。
>
> **任务状态订阅机制**：JournalManager 通过 EventBus 订阅 `TaskStateChangedEvent{task_id, new_state}`。当 `new_state == COMPLETED` 时，遍历 Journal 中所有 `task_id` 匹配的线索，将状态为 `UNLOCKED` 的线索标记为 `COMPLETED`。无需主动轮询，由任务系统在状态变化时推送事件。

> **MVP 简化策略**：初期仅实现"玩家主动确认"机制，其他两种作为未来扩展预留。

### 补偿路径选择逻辑

`CompensationEntry.CompensationClueIds` 支持 1:N 补偿池，选择规则如下：

**补偿选择策略**：
```
从 CompensationClueIds 中选择尚未在 Journal 中的线索
- 如果只有 1 条补偿线索 → 直接使用
- 如果有多条补偿线索 → 随机选择 1 条（保证每次游戏体验的差异性）
- 如果所有补偿线索都已在 Journal 中 → 此次补偿不添加新线索，但 Miss 标记保留
```

> **为什么是"多选一"而非"多选多"**：
> - 避免 Journal 污染（补偿过多会稀释"选择即代价"的情感）
> - 随机选择保证重复可玩性
> - 如需更精细的控制（如根据玩家行为选择特定补偿），可扩展为"多选多"，但 MVP 采用简化策略

### 递归补偿防护机制

**问题场景**：如果补偿线索的来源 NPC（`CompensationEntry.SourceClueId` 对应的 NPC）也死亡，导致补偿线索本身被标记为 MISSING，可能引发递归补偿。

**防护规则**：
```
补偿线索（IsPallidReplacement = true）不参与补偿触发判定
```
- 补偿线索是为了缓解"选择即代价"的情感惩罚而添加的，它们本身已经是"补偿"的结果
- 如果补偿线索再次被标记为 MISSING（如补偿来源 NPC 也死亡），**不触发新的补偿**
- 玩家需要承受多重选择的后果，这是"选择即代价"设计理念的延伸

**实现方式**：
```csharp
bool ShouldCompensate(string taskId)
{
    // 获取该任务关联的 CRITICAL 线索，排除补偿线索
    var criticalClues = Journal.Clues.Values
        .Where(c => c.TaskId == taskId
                 && c.Criticality == ClueCriticality.CRITICAL
                 && !c.IsPallidReplacement)  // 排除补偿线索
        .ToList();

    // 检查是否存在 MISSING 状态
    var missingClues = criticalClues.Where(c => c.State == ClueState.MISSING).ToList();
    if (missingClues.Count == 0) return false;

    // 检查已发现线索数量
    var discoveredCount = criticalClues
        .Count(c => c.State == ClueState.DISCOVERED
                 || c.State == ClueState.UNLOCKED
                 || c.State == ClueState.COMPLETED);

    return discoveredCount < MinCluesRequired;
}
```

### 补偿机制扩展说明

当前 `CompensationEntry.CompensationClueIds` 已支持 1:N 补偿池（多选一策略），MVP 即可使用多补偿路径。

**扩展方向**（非 MVP 范围）：
- **动态补偿**：补偿线索根据玩家行为动态生成，而非预设
- **多选多补偿**：某些极端场景（如连续杀死关键 NPC）可触发多选多补偿

### 补偿机制（MVP 简化版）

**触发条件**：
```
ShouldCompensate = (critical_clue_missing == true) AND (clues_discovered_for_task < MinCluesRequired)
```

**补偿触发条件变量定义**：
| 变量 | 类型 | 定义 | 说明 |
|------|------|------|------|
| `critical_clue_missing` | `bool` | 当前任务关联的 CRITICAL 线索中**存在** State == MISSING | 任务关联的 CRITICAL 线索是否存在 MISSING 状态 |
| `clues_discovered_for_task` | `int` | 当前任务关联的所有线索中 State ∈ {DISCOVERED, UNLOCKED, COMPLETED} 的**计数**（**不包括 NOT_DISCOVERED 和 MISSING**） | 任务已消化线索数量。注意：MISSING 线索不计入，因为它们是"原本可以知道但现在永远不知道"的遗憾标记，不属于玩家已获取的信息 |
| `MinCluesRequired` | `int` | **可配置**，默认值 3 | 单个任务最低线索数，低于此值且存在 MISSING 时触发补偿检查。<br>**配置位置**：`CompensationConfigSO.MinCluesRequired`（策划配置表） |
| `MaxCompensationPerTask` | `int` | **可配置**，默认值 2 | 单个任务最大补偿次数上限，防止 Journal 污染。<br>**配置位置**：`CompensationConfigSO.MaxCompensationPerTask` |

> **策划配置说明**：MinCluesRequired 和 MaxCompensationPerTask 作为可调参数存储在 `CompensationConfigSO` 中，策划可根据任务难度和叙事需求调整具体数值，而无需修改代码逻辑。

**逻辑说明**：仅当「任务关联的 CRITICAL 线索中确实存在 MISSING 状态」**且**「该任务已发现的线索数量尚未达到最低要求」时，才触发补偿机制。

**执行流程**：
1. 查找预设的 `CompensationEntry`（设计师手动预设，每条 CRITICAL 线索至少 1 条）
2. 检查当前任务补偿次数是否已达 `MaxCompensationPerTask`，若已达上限则跳过
3. 遍历 `CompensationEntry.CompensationClueIds`（支持多选一或多选多的补偿池），将尚未在 Journal 中的补偿线索添加并标记为 `IsPallidReplacement = true`
4. 显示警示文字"——你的选择留下了无法愈合的伤口——"
5. 原 MISSING 标记保留，实现"遗憾不可消除"

**补偿机制与 KillTagEvent（ADR-0011）联动说明**：
> 补偿机制**仅针对玩家主动击杀 NPC 导致的线索缺失**。与威胁/审讯/转化获取情报的区别如下：

| 获取方式 | 触发机制 | 补偿触发 |
|----------|----------|----------|
| **玩家主动击杀 NPC**（StealthKill/EnvironmentKill/FinishOff） | `KillTagEvent` → `NPCStateChangedEvent{DEAD}` → 标记线索为 MISSING | **会触发补偿**（因为是击杀导致的永久信息缺失） |
| 审问（Interrogate）已捆绑 NPC | `KnowledgeGainedEvent` → 获取 vulnerability/knowledge | **不触发补偿**（NPC 仍存活，信息已成功获取） |
| 搜身（Search）已死亡 NPC | `KnowledgeGainedEvent` → 获取 vulnerability/knowledge | **不触发补偿**（玩家主动搜索，信息已成功获取） |
| 威胁/贿赂/欺骗 NPC | `DialogueChoiceRequestEvent` → `DialogueResultEvent` | **不触发补偿**（NPC 存活，不涉及信息缺失） |

> **设计理由**：补偿机制的核心是"玩家冲动行为（杀死 NPC）导致关键线索永久缺失"这一遗憾体验。威胁/审讯/转化是玩家与 NPC 交互的手段，不会导致线索缺失，因此不在补偿范围内。

### 关键接口定义

> **统一规范**：所有 UI 面板的事件订阅必须遵循 shared-types.md §13.x UI系统事件订阅规范。Clue Journal UI 作为游戏数据消费者，订阅来自其他系统的事件。

#### 事件订阅（Inputs）

| 事件 | 来源 | 处理逻辑 |
|------|------|----------|
| `KeywordCapturedEvent{keyword, npc_id, location_id}` | LOS 系统 | 与 ClueTemplate 匹配，创建 Clue 实例 |
| `IntelObjectInteractedEvent{object_id, object_type, location_id}` | 环境交互系统 | 直接创建 Clue 实例（已是成品线索） |
| `NPCStateChangedEvent{npc_id, new_state: DEAD}` | NPC AI 系统 | 遍历 NPC 的 knowledge，标记尚未 DISCOVERED 的线索为 MISSING，MissingReason = NPC_DEAD |

> **已确定的设计决策**（原 OQ-4/5/6）：
>
> | # | 原问题描述 | 确定方案 | 决策日期 |
> |---|------------|----------|----------|
> | OQ-4 | 日志 UI 美术风格 | **默认"复古笔记本"风格**（深棕色皮革纹理背景、手写体字体、泛黄纸张）。实现路径：UI 系统（ADR-0015）定义 UXML/USS 样式模板 | 2026-04-11 |
> | OQ-5 | 时间线视图 | **MVP 默认不实现**。时间线视图复杂度高，对核心体验贡献有限。JournalData 接口在 Phase 4+ 扩展 `timeline_view` 相关字段（暂不预留在当前数据结构中） | 2026-04-11 |
> | OQ-6 | NPC 人物视图信息层级 | **MVP 默认显示基础信息**（姓名、状态、已知线索数）。不显示完整背景故事。NPC 人物视图详细信息在 Phase 4+ 扩展 | 2026-04-11 |

#### 事件发布（Outputs）

| 事件 | 目标 | 数据内容 |
|------|------|----------|
| `ClueDiscoveredEvent{clue_id, category, discovery_stage, narrative_significance, source_id}` | 理智系统 | 线索发现上下文，用于计算理智惩罚（完整定义见 shared-types.md §11.4） |
| `VulnerabilityUncoveredEvent{npc_id, source, vulnerability, clue_id, timestamp}` | Gritty Takedowns 系统 | 线索发现触发 vulnerability 解锁（完整定义见 shared-types.md §11.5） |
| `LocationRevealedEvent{location_id}` | 世界地图 | 新地点被发现，在地图上显示标记 |
| `JournalData{journal_view, current_clues}` | UI 系统 | 日志渲染所需数据（按 LOCATION/NPC 组织，TIMELINE 为 Phase 4+ 预留） |

> **注意**：`ClueDiscoveredEvent` 和 `VulnerabilityUncoveredEvent` 的完整字段定义见 shared-types.md §11.4 和 §11.5。本表格仅列出关键字段。

---

## Alternatives Considered

### Alternative 1: 同步查询模式（而非事件驱动）

- **描述**：LOS/环境系统通过直接查询接口传递数据，而非发布事件
- **Pros**：数据类型明确，调用关系清晰
- **Cons**：形成强耦合；多系统同时查询时难以保证事务一致性；扩展困难
- **Rejection Reason**：不符合 ADR-0001 事件驱动架构原则；限制了系统的可组合性

### Alternative 2: 线索存储在 NPC 实体上（分布式）

- **描述**：每个 NPC 自己管理其 knowledge 列表，线索系统按需查询
- **Pros**：数据物理上贴近生产者
- **Cons**：NPC 死亡后数据需要立即转移，否则丢失；分散存储难以做任务进度验证；补偿机制实现复杂
- **Rejection Reason**：设计文档明确要求集中式 Journal；NPC 死亡可能发生在玩家未保存进度时

### Alternative 3: 无补偿机制（纯惩罚）

- **描述**：关键线索缺失直接导致任务失败，不提供替代路径
- **Pros**：简化实现；强化"选择即代价"
- **Cons**：严重卡关风险；玩家可能因为一条线索失去整个任务的完成机会；设计意图过于残酷
- **Rejection Reason**：GDD 明确要求"防止卡关"；MVP 阶段需要可玩的完整性验证

---

## Consequences

### Positive

- **集中式数据管理**：Journal 作为单一数据源，便于存档/加载和任务进度验证
- **事件驱动解耦**：上游系统无需知道线索系统的存在，下游系统按需订阅
- **可逆的遗憾**：补偿机制确保玩家始终能推进，同时保留情感代价标记
- **可测试性**：状态机清晰，每个转移条件可独立验证

### Negative

- **单点故障**：JournalManager 是单例，崩溃将丢失所有线索数据（需配合存档系统）
- **补偿池维护成本**：每条 CRITICAL 线索需要设计师预设补偿路径，增加内容制作工作量
- **内存占用**：完整 Journal 包含所有地点和 NPC 的线索数据，内存占用随内容增加

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **R-1** | 补偿池配置遗漏导致关键线索无补偿路径 | 自动化检测工具（balance-check）验证每条 CRITICAL 线索的 CompensationEntry 存在 |
| **R-2** | 存档回滚后 Journal 与游戏世界状态不一致 | 存档时保存 Journal 完整快照；加载时执行 MISSING 线索验证（检查 SourceId 对应实体状态）；若 SourceId 对应实体仍存活但线索为 MISSING，自动修正为 DISCOVERED（边界情况容错） |
| **R-3** | 循环前置依赖导致线索永远无法解锁 | 设计验证阶段（balance-check）检测循环依赖并报错 |

---

## Performance Implications

| 维度 | 影响 | 说明 |
|------|------|------|
| **CPU** | 低 | 线索添加是 O(1) 查找；前置判定使用 Set 交集；状态转移无复杂计算 |
| **Memory** | 中 | 每个 Clue 约 200 字节；1000 条线索约 200KB；可接受 |
| **Load Time** | 低 | Journal 在存档加载时恢复，无额外异步加载 |
| **Network** | 无 | 纯本地系统 |

### 循环依赖检测算法

为防止玩家配置的前置线索形成循环导致线索永远无法解锁，使用**拓扑排序检测环**：

```csharp
// 检测所有线索的前置依赖是否形成循环
// 使用 Kahn 算法（BFS 拓扑排序）
// 时间复杂度：O(V + E)，V = 线索数，E = 前置依赖边数

public bool HasCircularDependency(List<Clue> clues)
{
    var inDegree = new Dictionary<string, int>();
    var adjacency = new Dictionary<string, List<string>>();

    // 初始化
    foreach (var clue in clues)
    {
        inDegree[clue.clue_id] = 0;
        adjacency[clue.clue_id] = new List<string>();
    }

    // 构建图
    foreach (var clue in clues)
    {
        foreach (var prereq in clue.prerequisites)
        {
            if (adjacency.ContainsKey(prereq))
            {
                adjacency[prereq].Add(clue.clue_id);
                inDegree[clue.clue_id]++;
            }
        }
    }

    // BFS 拓扑排序
    var queue = new Queue<string>();
    foreach (var kvp in inDegree)
        if (kvp.Value == 0) queue.Enqueue(kvp.Key);

    int visited = 0;
    while (queue.Count > 0)
    {
        var current = queue.Dequeue();
        visited++;
        foreach (var neighbor in adjacency[current])
        {
            inDegree[neighbor]--;
            if (inDegree[neighbor] == 0) queue.Enqueue(neighbor);
        }
    }

    // 如果不能访问所有节点，说明存在环
    return visited != clues.Count;
}
```

> **验证时机**：在 `ClueTemplateRepository` 加载时执行检测。
> - **Build 时验证**（内容制作阶段）：策划配置线索数据时，CI/CD 流水线执行一次完整拓扑排序，检测到环时拒绝合并，从源头阻止问题进入游戏包体。
> - **运行时加载前验证**（兜底）：游戏启动时再次执行验证，防止配置在 Build 后被篡改（理论上 Build 后文件不可变，但作为防御性编程保留）。
>
> 如两次验证结果不一致（Build 通过但运行时失败），以 Build 时结果为准，运行时输出警告日志建议重新执行 Build。

### 存档加载验证逻辑

```csharp
/// <summary>
/// 存档加载时验证 MISSING 线索与游戏世界状态的一致性
/// </summary>
public void ValidateJournalOnLoad(Journal journal, IEntityRegistry entityRegistry)
{
    foreach (var clue in journal.Clues.Values.Where(c => c.State == ClueState.MISSING))
    {
        // 只有 NPC_KNOWLEDGE_BONUS 类型才需要验证
        if (clue.SourceType != ClueSourceType.NPC_KNOWLEDGE_BONUS)
            continue;

        var sourceEntity = entityRegistry.GetEntity(clue.SourceId);
        if (sourceEntity == null)
            continue; // SourceId 不存在，跳过

        // 如果 NPC 仍然存活，说明是存档回滚，修正线索状态
        if (sourceEntity.IsAlive)
        {
            clue.State = ClueState.DISCOVERED;
            clue.MissingReason = MissingReasonType.NONE;
            Debug.LogWarning($"[Journal] Corrected MISSING clue {clue.ClueId} - source NPC {clue.SourceId} is alive");
        }
    }
}
```

> **验证时机**：存档加载后立即执行（`JournalManager.Initialize` 时）。MISSING 标记不可逆仅针对"真正因 NPC 死亡导致的 MISSING"，存档回滚的边界情况需要容错。

---

## Implementation Dependencies

本系统的实现依赖于以下 ADR 定义的系统：

| 依赖系统 | 依赖关系 | 说明 |
|----------|----------|------|
| [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) | 必须 | ClueFactory 通过 EventBus 订阅上游事件 |
| [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) | 必须 | NPCStateChangedEvent 来源；NPC.knowledge 数据结构定义 |
| [ADR-0006: 网络同步架构](./adr-0006-network-synchronization-architecture.md) | 必须 | 存档/加载时 Journal 状态同步 |
| [ADR-0008: 脆弱度与伤害系统](./adr-0008-health-lethality-architecture.md) | 可选 | 击杀事件与线索转移的联动（如果线索来源包含战斗场景） |
| [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) | 可选 | KillTagEvent 触发理智/愤怒变化 |
| [ADR-0012: 世界地图与非线性叙事架构](./adr-0012-world-map-nonlinear-progression.md) | 必须 | LocationRevealedEvent 事件消费者 |
| [ADR-0015: UI 系统](./adr-0015-ui-system-architecture.md) | 必须 | JournalData 查询接口消费者 |
| [ADR-0017: 理智/愤怒系统](./adr-0017-sanity-rage-meter-architecture.md) | 必须 | ClueDiscoveredEvent 消费者（下游依赖）。注：此为接口声明式依赖，非实现依赖；两系统通过 EventBus 解耦，无循环引用问题 |

### Journal UI MVP 布局

```
┌─────────────────────────────────────────────────────────┐
│  [LOCATION]           Journal            [Filters ▼]   │
├──────────────┬──────────────────────────────────────────┤
│              │                                          │
│  LOCATION   │   Current View (Clue/NPC/Location)       │
│  ────────── │                                          │
│  • 锈港      │   ┌────────────────────────────────┐   │
│  • 灰桥      │   │  Clue Card                      │   │
│  • ...       │   │  ────────────────────────────    │   │
│              │   │  Title: 码头废弃文件              │   │
│  NPC         │   │  Category: 文档                   │   │
│  ────────── │   │  Significance: 中                │   │
│  • NPC-A     │   │  ────────────────────────────    │   │
│  • NPC-B     │   │  Description text...             │   │
│  • ...       │   │                                  │   │
│              │   └────────────────────────────────┘   │
│              │                                          │
└──────────────┴──────────────────────────────────────────┘
```

> **布局说明**：
> - 左侧 LOCATION/NPC 列表为固定宽度（约 200px），垂直滚动
> - 右侧为可变宽度内容区，显示当前选中项的线索卡片
> - LOCATION 视图：按地区分组线索，显示完成百分比
> - NPC 视图：按 NPC 分组线索，显示 NPC 头像 + 姓名 + 状态图标
> - **NPC 状态图标定义**（来源 ADR-0004 §AlertState）：
>   - 🟢 UNDETECTED（未发现）、🟡 SUSPECT（怀疑）、🔴 ALERT（警戒）、⚠️ ESCAPE（逃跑）、⚔️ COMBAT（战斗）
>   - 💀 DEAD（死亡）— NPC 死亡后状态图标变灰并显示覆盖层
> - TIMELINE 视图为 Phase 4+ 预留，MVP 仅实现 LOCATION/NPC

### Clue Journal UI 输入屏蔽规则

当 Clue Journal UI 打开时，UI 系统通过 Input Blocking Layer 屏蔽游戏输入：

```
Clue Journal UI 打开流程：
1. 玩家按下 Journal 快捷键（PC: J / PS5: Touchpad）
2. UI 系统接收 JournalOpenRequest 事件
3. UI 系统设置 Input Blocking Layer 为 active，Priority = 50（Menu Layer 范围）
4. 玩家输入被屏蔽（移动/攻击/交互/跳跃/蹲伏）
5. UI 导航和确认键保持可用（确保玩家仍能操作 Journal UI）
6. 玩家按下关闭键（ESC/Options）或点击关闭按钮
7. UI 系统设置 Input Blocking Layer 为 inactive
8. 游戏输入恢复
```

**输入屏蔽范围**：
- **屏蔽**：移动(WASD)、攻击(鼠标左键/F)、交互(E/F)、跳跃(空格/C)、蹲伏(C/Ctrl)、菜单(ESC)、快速存档(Q)、地图拖拽
- **保持可用**：UI 导航（方向键/左摇杆）、确认(Enter/A/X)、取消(ESC/Options/O)

> **与 World Map 揭示动画的 Input Blocking 优先级说明**：
> - Clue Journal 的 Input Blocking Priority = 50（Menu Layer）
> - World Map 揭示动画的 Input Blocking Priority = 60（高于 Menu Layer）
> - 如果揭示动画期间玩家打开 Journal，揭示动画的 Input Blocking 不会被 Journal 覆盖（60 > 50）
> - 揭示动画完成后，Journal 的 Input Blocking 才能生效

### ClueViewedEvent 事件说明

> **设计说明**：`ClueViewedEvent` 用于区分"发现线索"（DISCOVERED 状态触发）与"阅读线索"（UNLOCKED 状态触发）。

```csharp
/// <summary>
/// 玩家阅读线索事件
/// 当玩家在 Journal UI 中打开线索详情并阅读时发布
/// 用于区分"发现"与"阅读"的行为追踪
/// </summary>
public struct ClueViewedEvent
{
    public string clue_id;
    public string player_id;
    public float timestamp;
}
```

| 事件 | 触发时机 | 使用场景 |
|------|---------|---------|
| `ClueDiscoveredEvent` | 线索状态变为 DISCOVERED 时（发现线索来源） | Sanity/Rage 系统计算理智惩罚 |
| `ClueViewedEvent` | 玩家主动打开 Journal 并阅读线索详情时（UNLOCKED 状态） | 任务进度追踪、成就系统、COMPLETED 状态触发 |

> **COMPLETED 触发条件补充**：玩家主动确认线索后，线索状态从 UNLOCKED 变为 COMPLETED。详见 §线索完成机制。

---

## Migration Plan

本决策不涉及对现有代码的迁移（无现有实现）。实施步骤：

1. **Phase 1**：实现 `ClueTemplateRepository` 和 `ClueFactory`（数据定义层）
2. **Phase 2**：实现 `JournalManager` 状态机和事件订阅逻辑
3. **Phase 3**：实现补偿机制和任务进度验证
4. **Phase 4**：实现 `JournalData` 查询接口供 UI 使用
5. **Phase 5**：与 LOS 系统、环境交互系统、NPC AI 系统联调

---

## Validation Criteria

| ID | 标准 | 测试方法 |
|----|------|----------|
| VC-1 | LOS 窃听关键词后，对应线索正确添加 | 模拟 KeywordCapturedEvent，检查 Journal 中 Clue 数量 +1 |
| VC-2 | NPC 死亡后，其未收集的 knowledge 标记为 MISSING | 杀死 NPC，检查 MissingReason == "NPC已死亡" |
| VC-3 | 补偿机制在关键线索缺失时正确触发 | 模拟 critical_clue_missing + 任务无法推进，检查补偿线索添加 |
| VC-4 | 苍白替代线索保留 [信息缺失] 标记 | 触发补偿，检查 IsPallidReplacement == true 且 MissingReason 不变 |
| VC-5 | 前置线索满足时自动解锁 | 添加所有前置线索，检查目标线索 State == UNLOCKED |
| VC-6 | 同一线索不重复添加 | 触发两次相同来源事件，检查 Clues 集合大小不变 |
| VC-7 | 日志 UI 视图切换 < 100ms | 计时器测量 LOCATION/NPC 切换响应时间（TIMELINE 为 Phase 4+ 预留） |
| VC-8 | 循环前置依赖被正确检测 | 创建 A→B→C→A 配置，验证 HasCircularDependency 返回 true |
| VC-9 | 无环配置通过检测 | 创建 A→B→C 线性依赖，验证 HasCircularDependency 返回 false |
| VC-10 | 补偿次数达到上限后不再添加 | 杀死多个关键 NPC，验证补偿线索数不超过 MaxCompensationPerTask |
| VC-11 | 存档回滚后存活的 NPC 线索状态自动修正 | 加载存档后，如果 NPC 仍存活但线索为 MISSING，验证自动修正为 DISCOVERED |

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 本系统遵循事件驱动原则
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC 的 knowledge 数据结构定义
- [ADR-0006: 网络同步架构](./adr-0006-network-synchronization-architecture.md) — 存档/加载时 Journal 状态同步
- [ADR-0008: 脆弱度与伤害系统](./adr-0008-health-lethality-architecture.md) — 击杀事件与线索转移的联动
- [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) — KillTagEvent 触发理智/愤怒变化
- [ADR-0012: 世界地图与非线性叙事架构](./adr-0012-world-map-nonlinear-progression.md) — LocationRevealedEvent 事件消费者
- [ADR-0017: 理智/愤怒系统](./adr-0017-sanity-rage-meter-architecture.md) — ClueDiscoveredEvent 消费者
- [ADR-0015: UI 系统](./adr-0015-ui-system-architecture.md) — JournalData 查询接口消费者
- [Shared Types: 伤害与命中类型](./shared-types.md) — ClueCategory 枚举定义
