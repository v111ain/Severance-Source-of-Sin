# ADR-0004: NPC AI 行为架构决策

## Status
**Accepted**

## Date
2026-04-09

## Last Updated
2026-04-15 (ADR 评审修复：补充 NPCStateChangedEvent 订阅；AlertFSM.TransitionTo() 发布 AlertStateChangedEvent；补充 AreaEnteredEvent 订阅)

## Context

### Problem Statement

NPC AI 系统是《断绝：罪恶之源》最高风险的系统（systems-index.md 标注），同时也是所有游戏系统的中枢——它与 LOS、Health、Gritty Takedowns、Sanity/Rage、Clue System 都有交互。俯视角潜行游戏中的 NPC AI 极容易出现两类问题：

1. **"过傻"**：NPC 视野范围设置不合理，感知逻辑有漏洞，玩家可以轻易利用
2. **"过强"**：NPC 视野范围过大，警报传播过快，玩家无处可藏

此外，NPC AI 系统在实现层面面临架构选择：
- **有限状态机 (FSM)**：简单直接，但状态多时难以维护
- **行为树 (Behavior Tree)**：层次清晰，但节点过多时难以调试
- **目标导向规划 (GOAP)**：适合复杂 NPC 决策，但学习和实现成本高

### Constraints

- **平台目标**：PC & PS5，需要在 PS5 上稳定 60fps
- **性能预算**：100 个 NPC 同时运行，单帧感知更新 < 2ms
- **设计约束**：需要支持派系感知网络、Alert State 状态机、World State 管理
- **协作约束**：AI 程序员需要与 gameplay-programmer、systems-designer 协同开发

### Requirements

- **必须**：定义 NPC AI 行为架构的实现模式（FSM/BT/GOAP）
- **必须**：定义感知计算架构（视觉/听觉/记忆的协同机制）
- **必须**：定义派系感知网络的实现方案
- **必须**：定义 NPC 属性系统（Bravery/Courage）的运行时查询接口
- **必须**：提供 Unity 项目代码结构指导

---

## Decision

### 架构决策

采用**混合架构：有限状态机 (FSM) + 行为树 (BT)** 的组合方案：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        NPC AI 行为架构                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     NPCController (MonoBehaviour)                │   │
│  │  - 管理单个 NPC 实例的生命周期                                     │   │
│  │  - 持有 NPCData (身份、派系、属性)                                │   │
│  │  - 持有 FSM (Alert State) + BT (行为执行)                         │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│          ┌─────────────────────────┴─────────────────────────┐          │
│          ▼                                                   ▼          │
│  ┌───────────────────┐                           ┌───────────────────┐   │
│  │   AlertFSM        │                           │   BehaviorTree    │   │
│  │   (Alert State)   │                           │   (行为执行)       │   │
│  │                   │                           │                   │   │
│  │ UNDETECTED        │◄─────────────────────────│  Root Selector    │   │
│  │ SUSPECT           │  状态切换触发行为          │    │             │   │
│  │ SEARCH            │                           │  ┌─┴──┐         │   │
│  │ ALERT             │                           │  │    │         │   │
│  │ ESCAPE            │                           │ Patrol  Investigate│   │
│  │ COMBAT            │                           │ Alert   Combat   │   │
│  └───────────────────┘                           └───────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     PerceptionComponent                           │   │
│  │  - 视觉感知 (LOS System 事件订阅)                                 │   │
│  │  - 听觉感知 (NoiseMadeEvent 事件订阅)                                 │   │
│  │  - 记忆管理 (MemoryScore 衰减)                                    │   │
│  │  - 派系感知 (SharedAlert 接收处理)                                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     FactionNetworkComponent                       │   │
│  │  - 维护派系关系矩阵                                               │   │
│  │  - 处理 SharedAlert 广播/接收                                     │   │
│  │  - 管理"已接收共享"去重列表                                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. Alert State FSM（状态机）

Alert State 使用**有限状态机**，因为：
- 状态数量有限（6个：UNDETECTED/SUSPECT/SEARCH/ALERT/ESCAPE/COMBAT）
- 状态转换条件明确，适合 FSM 的清晰表达
- 状态转换触发 BT 中的对应行为节点

> **重要：AlertStateChangedEvent 发布要求**
> AlertFSM.TransitionTo() 在状态转换时**必须**通过 EventBus 发布 AlertStateChangedEvent，
> 通知所有订阅方（包括 GrittyTakedowns、SanityRage 等系统）。
> 发布方法参考 NPCController.PublishAlertStateChanged()（见 §7 事件订阅与发布关系）。

```csharp
// AlertState.cs
public enum AlertState
{
    UNDETECTED,
    SUSPECT,
    SEARCH,
    ALERT,
    ESCAPE,
    COMBAT
}

public enum WorldState
{
    FREE,
    UNCONSCIOUS,
    TIED,
    DEAD
}

/// <summary>
/// NPC 身份标签类型
/// 定义位置：shared-types.md §5.5
/// 由 LOS System（ADR-0007）维护
/// 用于 Gritty Takedowns 判断交互选项可用性
/// </summary>
/// <remarks>
/// 注意：此枚举已迁移至 shared-types.md 统一管理，ADR-0004 仅引用其定义。
/// 实现时应使用：using Assets.Game.Foundation.Shared.Types.NPCIdentityType;
/// 或直接引用 shared-types.md 中的枚举类型。
/// </remarks>
// NPCIdentityType 定义移至 shared-types.md §5.5
// public enum NPCIdentityType
// {
//     UNKNOWN,     // 未识别
//     ENEMY,       // 恶徒（可处决/威胁）
//     ACCOMPLICE,  // 帮凶（可处决/威胁）
//     VICTIM       // 受害者（不可伤害）
// }

// Faction 枚举定义移至 shared-types.md §5.7
// public enum Faction
// {
//     凋亡议会,
//     锈网,
//     灰烬团,
//     无声者,
//     苍白之手,
//     中立
// }
// 与 FactionAllegiance 结构一起定义

// AlertTransition.cs
public struct AlertTransition
{
    public AlertState From;
    public AlertState To;
    public AlertTrigger Trigger;
    public Func<bool> Condition; // 可选的额外条件
}

public enum AlertTrigger
{
    PERCEPTION,   // 感知触发（视觉/听觉/记忆）
    FACTION,      // 玩家主动行为
    SHARED,       // 派系感知共享
    EXECUTION     // 处决被目击
}
```

### 2. Behavior Tree（行为树）

行为执行使用**行为树**，因为：
- 每个 Alert State 下的行为需要层次化组织
- 行为节点（巡逻、追击、搜索等）可以复用
- BT 的可视化和调试工具成熟（Unity Behavior Tree 插件）

```csharp
// BT 节点类型定义
public abstract class BTNode
{
    public enum NodeState { SUCCESS, FAILURE, RUNNING }
    public abstract NodeState Execute(NPCController npc);
}

// BT Composite Nodes
public class Selector : BTNode { }   // 选择第一个成功的子节点
public class Sequence : BTNode { }  // 全部成功才返回成功
public class Parallel : BTNode { }   // 并行执行所有子节点

// BT Leaf Nodes
public class PatrolNode : BTNode { }     // 巡逻行为
public class InvestigateNode : BTNode { } // 调查可疑位置
public class AlertNode : BTNode { }      // 发出警报
public class CombatNode : BTNode { }     // 战斗行为
public class EscapeNode : BTNode { }     // 逃跑行为
```

### 3. PerceptionComponent（感知组件）

感知计算独立为组件，与 Alert FSM 解耦：

```csharp
// PerceptionComponent.cs
public class PerceptionComponent
{
    // 感知评分（由 LOS System 和 Player Controller 事件驱动）
    // VisualScore：来自 LOS System 的 ExposureValueChangedEvent（0-100）
    // AudioScore：来自 Player Controller 的 NoiseMadeEvent（0-100）
    // MemoryScore：通过 ExposureValueChangedEvent 事件更新，非直接同步
    public float VisualScore { get; private set; }
    public float AudioScore { get; private set; }
    public float MemoryScore { get; private set; }

    // ========== 暴露值 → AlertState 映射表 ==========
    // 暴露值（0-100）到 AlertState 阈值的映射关系：
    //
    // | 暴露值范围 | AlertState  | 说明                    |
    // |------------|-------------|-------------------------|
    // | 0-29       | UNDETECTED  | 未被发现                 |
    // | 30-59      | SUSPECT     | 怀疑，玩家可能存在       |
    // | 60-79      | SEARCH      | 搜索，确认玩家位置       |
    // | 80-99      | ALERT       | 警戒，准备战斗           |
    // | 100        | COMBAT      | 进入战斗                 |
    //
    // 映射公式：
    // - VisualScore → 直接对应暴露值
    // - AudioScore → 噪声检测，累积到 MemoryScore
    // - MemoryScore → 记忆残留，由 ExposureValueChangedEvent 衰减触发
    //
    // AlertState 转换优先级：COMBAT > ALERT > SEARCH > SUSPECT > UNDETECTED

    // 感知阈值（Tuning Knobs）
    public float SuspectThreshold => 30f;    // 暴露值 30 → SUSPECT
    public float SearchThreshold => 60f;     // 暴露值 60 → SEARCH
    public float AlertThreshold => 80f;      // 暴露值 80 → ALERT
    public float CombatThreshold => 100f;    // 暴露值 100 → COMBAT

    // 记忆衰减配置
    private const float MemoryDecayRate = 30f;  // 记忆衰减率（%/秒），与 LOS System DecayRate 同步

    /// <summary>
    /// 订阅 LOS System 发布的暴露值变化事件
    /// MemoryScore 通过此事件更新，而非直接同步
    /// </summary>
    public void SubscribeToEvents()
    {
        EventBus.Instance.Subscribe<ExposureValueChangedEvent>(OnExposureValueChanged);
        EventBus.Instance.Subscribe<NoiseMadeEvent>(OnNoiseDetected);
        EventBus.Instance.Subscribe<NPCIdentityConfirmedEvent>(OnIdentityConfirmed);
    }

    private void OnExposureValueChanged(ExposureValueChangedEvent e)
    {
        if (e.npc_id != _ownerNpcId) return;

        // VisualScore 直接来自暴露值（0-100）
        VisualScore = e.exposure_value;

        // MemoryScore 逻辑说明：
        // 当玩家在视野内时，VisualScore 实时反映暴露值，MemoryScore 由 Update() 中的衰减逻辑处理
        // 当玩家离开视野时（e.exposure_value <= 0），记录最后的暴露值到 last_exposure_value
        // MemoryScore 的衰减在 Update() 中统一处理：从当前 MemoryScore 开始按 DecayRate 衰减
        // 此设计确保 MemoryScore 在玩家离开视野的第一帧就开始自然衰减，而非被重置后立即衰减
        if (e.exposure_value <= 0)
        {
            // 玩家离开视野时，如果当前 MemoryScore 为 0 或低于最后暴露值，则继承最后暴露值作为记忆残留
            // 否则保持 MemoryScore 不变，让 Update() 中的衰减逻辑自然处理
            MemoryScore = Mathf.Max(MemoryScore, e.last_exposure_value);
        }
    }

    private void OnNoiseDetected(NoiseMadeEvent e)
    {
        // 玩家噪声检测：噪声强度（0-100）直接累加到 AudioScore
        // 噪声距离NPC越近，AudioScore 累积越快
        float distance = Vector3.Distance(_ownerTransform.position, e.position);
        float distanceFactor = Mathf.Clamp(1.0f - (distance / MaxAudioRange), 0f, 1f);

        // 噪声暴露增量 = 基础暴露速率 × 噪声强度 × 距离因子 × deltaTime
        float noiseExposure = e.intensity * distanceFactor * Time.deltaTime;
        AudioScore = Mathf.Clamp(AudioScore + noiseExposure, 0f, 100f);
    }

    private void OnIdentityConfirmed(NPCIdentityConfirmedEvent e)
    {
        if (e.npc_id != _ownerNpcId) return;

        // NPCIdentityConfirmedEvent 触发后，NPC AI 应升级为 Enemy 状态
        // 此事件由 LOS System 的 NPCIdentityManager 发布
        // 订阅者：NPCAI（升级为 Enemy）、ClueJournal（触发线索发现）
        _isEnemyConfirmed = true;
    }

    // 每帧/定时更新
    public void Update(NPCController npc, float deltaTime)
    {
        // MemoryScore 衰减逻辑：当玩家不在视野内时，记忆值按 DecayRate=30%/秒衰减
        if (!npc.IsPlayerInSight)
        {
            MemoryScore = Mathf.Max(0, MemoryScore - MemoryDecayRate * deltaTime);
        }

        // AudioScore 也随时间衰减（如果长时间没有噪声）
        AudioScore = Mathf.Max(0, AudioScore - MemoryDecayRate * 0.5f * deltaTime);
    }

    /// <summary>
    /// 获取当前综合感知评分（0-100）
    /// </summary>
    /// <remarks>
    /// 评分公式：取即时感知（Visual/Audio的最大值）与Memory的加权组合
    /// - 即时感知权重0.9：表示当前实时感知占主导
    /// - 记忆残留权重0.1：表示历史感知作为辅助参考
    ///
    /// 为什么不直接相加：因为Visual/Audio/Memory都是0-100范围，直接相加最高可达300，
    /// 这在语义上不合理（不能同时100%看到+100%听到+100%记住同一个目标）
    /// </remarks>
    public float GetPerceptionScore()
    {
        float immediateScore = Mathf.Max(VisualScore, AudioScore);
        return Mathf.Clamp(immediateScore * 0.9f + MemoryScore * 0.1f, 0f, 100f);
    }

    /// <summary>
    /// 根据感知评分获取对应的 AlertState
    /// </summary>
    public AlertState GetAlertStateFromScore()
    {
        float score = GetPerceptionScore();
        if (score >= CombatThreshold) return AlertState.COMBAT;
        if (score >= AlertThreshold) return AlertState.ALERT;
        if (score >= SearchThreshold) return AlertState.SEARCH;
        if (score >= SuspectThreshold) return AlertState.SUSPECT;
        return AlertState.UNDETECTED;
    }

    private int _ownerNpcId;
    private Transform _ownerTransform;
    private bool _isEnemyConfirmed;
    private const float MaxAudioRange = 20f;  // 最大听觉范围（米）
}
```

### 3.1. 暴露值 → AlertState 数据流

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        暴露值数据流                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  LOS System (ExposureTracker)                                           │
│  ├── 暴露值计算：BaseExposureRate × 乘数 × 距离因子                      │
│  ├── 发布 ExposureValueChangedEvent（暴露值变化）                        │
│  └── 发布 PlayerSpottedEvent（暴露值达到100）                            │
│                    │                                                     │
│                    ▼                                                     │
│  NPC AI (PerceptionComponent)                                           │
│  ├── 订阅 ExposureValueChangedEvent → 更新 VisualScore/MemoryScore       │
│  ├── 订阅 NoiseMadeEvent → 更新 AudioScore（噪声检测）                   │
│  └── 订阅 NPCIdentityConfirmedEvent → 确认 Enemy 身份                    │
│                    │                                                     │
│                    ▼                                                     │
│  AlertFSM ← GetAlertStateFromScore()                                     │
│  ├── 30 → SUSPECT                                                       │
│  ├── 60 → SEARCH                                                         │
│  ├── 80 → ALERT                                                         │
│  └── 100 → COMBAT                                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2. FactionNetworkComponent 与 LOS System 协调

```csharp
// FactionNetworkComponent.cs（扩展）
public class FactionNetworkComponent
{
    // 派系关系矩阵（静态配置）
    private static readonly Dictionary<Faction, Dictionary<Faction, FactionRelation>> Relations = new()
    {
        { Faction.凋亡议会, new() {
            { Faction.锈网, FactionRelation.ALLIED },
            { Faction.灰烬团, FactionRelation.ALLIED },
            { Faction.无声者, FactionRelation.HOSTILE },
            { Faction.苍白之手, FactionRelation.NEUTRAL }
        }},
        // ... 其他派系关系
    };

    // "已接收共享"去重列表（来源 NPC ID + 时间戳）
    private List<(int SourceId, float Timestamp)> _receivedAlerts = new();

    /// <summary>
    /// 订阅 LOS System 的感知共享事件
    /// 当派系 NPC 发现玩家后，通过派系网络共享感知信息
    /// </summary>
    public void SubscribeToLOSEvents()
    {
        EventBus.Instance.Subscribe<PlayerSpottedEvent>(OnFactionMemberSpottedPlayer);
        EventBus.Instance.Subscribe<ExposureValueChangedEvent>(OnFactionMemberExposureChanged);
    }

    /// <summary>
    /// 派系成员发现玩家时，通过派系网络广播共享感知
    /// </summary>
    private void OnFactionMemberSpottedPlayer(PlayerSpottedEvent e)
    {
        var sourceNpc = NPCManager.Instance.GetNPC(e.npc_id);
        if (sourceNpc == null) return;

        // 检查是否为同派系 NPC
        if (!IsSameFaction(sourceNpc, _ownerNpc)) return;

        // 构建威胁信息并广播
        var threat = new ThreatInfo
        {
            Position = PlayerController.Instance.transform.position,
            Intensity = 100f,  // 直接发现为最高强度
            SourceId = e.npc_id
        };

        BroadcastSharedAlert(_ownerNpc, threat);
    }

    /// <summary>
    /// 派系成员暴露值变化时，渐进式共享感知
    /// </summary>
    private void OnFactionMemberExposureChanged(ExposureValueChangedEvent e)
    {
        var sourceNpc = NPCManager.Instance.GetNPC(e.npc_id);
        if (sourceNpc == null) return;

        // 只有暴露值达到 SUSPECT 阈值以上才共享
        // 使用 last_exposure_value（衰减前的记忆残留值）而非当前 exposure_value
        if (e.last_exposure_value < 30f) return;
        if (!IsSameFaction(sourceNpc, _ownerNpc)) return;

        var threat = new ThreatInfo
        {
            Position = PlayerController.Instance.transform.position,
            Intensity = e.last_exposure_value,  // 使用衰减前的记忆残留值
            SourceId = e.npc_id
        };

        BroadcastSharedAlert(_ownerNpc, threat);
    }

    // 发送感知共享
    public void BroadcastSharedAlert(NPCController npc, ThreatInfo threat)
    {
        var faction = npc.Data.Faction;
        foreach (var otherNpc in GetSameFactionNPCs(npc))
        {
            if (otherNpc.NpcId == npc.NpcId) continue;  // 跳过自己

            var relation = Relations[faction][otherNpc.Data.Faction];
            var delay = CalculateSharedDelay(relation, npc, otherNpc);
            var degradedThreat = ApplyDegradation(threat, relation);

            // 延迟广播
            npc.StartCoroutine(DelayedAlertCoroutine(otherNpc, degradedThreat, delay));
        }
    }

    // 处理接收到的共享（去重）
    public bool ShouldProcessSharedAlert(int sourceId)
    {
        var now = Time.time;
        // 5秒内同一来源只处理一次
        _receivedAlerts.RemoveAll(x => now - x.Timestamp > 5f);
        if (_receivedAlerts.Any(x => x.SourceId == sourceId)) return false;
        _receivedAlerts.Add((sourceId, now));
        return true;
    }

    private bool IsSameFaction(NPCController a, NPCController b)
    {
        return a?.Data?.Faction == b?.Data?.Faction;
    }
}
```

### 3.3. SpatialPartition 共享组件引用

> **跨 ADR 共享组件**：`SpatialPartition` 与 ADR-0007 中的 `SpatialPartition` 功能相同，
> 应统一使用共享组件 `SpatialGrid`，统一定义在 `Assets/Game/Foundation/Shared/Spatial/` 目录下。
> 详见 [ADR-0007](./adr-0007-los-system-architecture.md) 的 SpatialPartition 说明。

```csharp
// SpatialPartition 使用示例（引用共享组件）
using Assets.Game.Foundation.Shared.Spatial;

public class NPCManager
{
    private SpatialGrid _spatialGrid;  // 引用共享组件，而非重复定义

    public void Initialize()
    {
        _spatialGrid = new SpatialGrid(GameConstants.SPATIAL_GRID_SIZE);
    }

    public void RegisterNPC(NPCController npc)
    {
        _spatialGrid.Register(npc.NpcId, npc.transform.position);
    }

    public List<int> GetNearbyNPCs(Vector3 position, float range)
    {
        return _spatialGrid.GetEntitiesInRange(position, range);
    }
}
```

### 4. FactionNetworkComponent（派系感知网络）

```csharp
// FactionNetworkComponent.cs
public class FactionNetworkComponent
{
    // 派系关系矩阵（静态配置）
    private static readonly Dictionary<Faction, Dictionary<Faction, FactionRelation>> Relations = new()
    {
        { Faction.凋亡议会, new() {
            { Faction.锈网, FactionRelation.ALLIED },
            { Faction.灰烬团, FactionRelation.ALLIED },
            { Faction.无声者, FactionRelation.HOSTILE },
            { Faction.苍白之手, FactionRelation.NEUTRAL }
        }},
        // ... 其他派系关系
    };

    // "已接收共享"去重列表（来源 NPC ID + 时间戳）
    private List<(int SourceId, float Timestamp)> _receivedAlerts = new();

    // 发送感知共享
    public void BroadcastSharedAlert(NPCController npc, ThreatInfo threat)
    {
        var faction = npc.Data.Faction;
        foreach (var otherNpc in GetSameFactionNPCs(npc))
        {
            var relation = Relations[faction][otherNpc.Data.Faction];
            var delay = CalculateSharedDelay(relation, npc, otherNpc);
            var degradedThreat = ApplyDegradation(threat, relation);

            // 延迟广播
            npc.StartCoroutine(DelayedAlertCoroutine(otherNpc, degradedThreat, delay));
        }
    }

    // 处理接收到的共享（去重）
    public bool ShouldProcessSharedAlert(int sourceId)
    {
        var now = Time.time;
        // 5秒内同一来源只处理一次
        _receivedAlerts.RemoveAll(x => now - x.Timestamp > 5f);
        if (_receivedAlerts.Any(x => x.SourceId == sourceId)) return false;
        _receivedAlerts.Add((sourceId, now));
        return true;
    }
}
```

### 5. Unity 项目结构

```
Assets/Game/
├── Core/NPCAI/
│   ├── NPCController.cs              # NPC 实例 MonoBehaviour
│   ├── NPCData.cs                    # NPC 身份数据（ScriptableObject）
│   ├── AlertFSM/
│   │   ├── AlertFSM.cs              # 状态机核心
│   │   ├── AlertTransitions.cs       # 转换规则表
│   │   └── AlertState.cs             # 状态枚举
│   ├── BehaviorTree/
│   │   ├── BTNode.cs                 # 节点基类
│   │   ├── BTSelector.cs             # 选择节点
│   │   ├── BTSequence.cs             # 序列节点
│   │   ├── BTRoot.cs                 # 根节点
│   │   ├── Actions/                  # 行为节点
│   │   │   ├── BTPatrol.cs
│   │   │   ├── BTInvestigate.cs
│   │   │   ├── BTAlert.cs
│   │   │   ├── BTCombat.cs
│   │   │   └── BTEscape.cs
│   │   └── Conditions/               # 条件节点
│   │       ├── BTCanSeePlayer.cs
│   │       └── BTCanHearNoise.cs
│   ├── Perception/
│   │   ├── PerceptionComponent.cs    # 感知组件
│   │   ├── VisualPerception.cs        # 视觉感知
│   │   ├── AudioPerception.cs        # 听觉感知
│   │   └── MemoryComponent.cs        # 记忆管理
│   └── Faction/
│       ├── FactionNetworkComponent.cs # 派系感知网络
│       ├── FactionRelations.cs       # 派系关系矩阵
│       └── SharedAlertQueue.cs        # 共享警报队列
```

### 6. NPCData ScriptableObject

```csharp
// NPCSizeCategory.cs
// 定义位置：shared-types.md §5.6（唯一数据源）
// 此处仅作为代码引用参考，枚举定义以 shared-types.md 为准

// NPCData.cs (ScriptableObject)
[CreateAssetMenu(menuName = "Game/NPC/NPCData")]
public class NPCData : ScriptableObject
{
    [Header("Identity")]
    public string npcId;
    public Faction faction;
    public NPCIdentityType identity; // ENEMY/ACCOMPLICE/VICTIM/UNKNOWN

    [Header("Attributes")]
    [Range(1, 10)] public int Bravery = 5;  // 影响抵抗/屈服判定
    [Range(1, 10)] public int Courage = 5;   // 影响回避/逃跑行为
    public NPCSizeCategory sizeCategory = NPCSizeCategory.Medium;  // 影响捆绑时间（引用 shared-types.md §5.6）

    [Header("Perception")]
    public float maxPerceptionRange = 15f;   // 最大感知范围（米）
    public float fieldOfView = 90f;           // 视野角度

    [Header("Patrol")]
    public List<Vector3> patrolPoints;
    public float patrolSpeed = 2f;
}
```

### 7. 事件订阅与发布关系

```csharp
// NPCController 事件订阅设置
private void SetupEventSubscriptions()
{
    // ========== 来自 LOS System ==========
    EventBus.Instance.Subscribe<PlayerSpottedEvent>(OnPlayerSpotted);
    EventBus.Instance.Subscribe<ExposureValueChangedEvent>(OnExposureValueChanged);  // 视觉暴露值同步

    // ========== 来自 LOS System - NPCIdentityManager ==========
    // NPCIdentityManager 确认身份后发布此事件
    // NPCAI 收到此事件后应升级为 Enemy 状态
    // ClueJournal 收到此事件后触发线索发现
    EventBus.Instance.Subscribe<NPCIdentityConfirmedEvent>(OnNPCIdentityConfirmed);

    // ========== 来自 Player Controller ==========
    EventBus.Instance.Subscribe<NoiseMadeEvent>(OnNoiseDetected);

    // ========== 来自 Gritty Takedowns ==========
    EventBus.Instance.Subscribe<ExecutionWitnessedEvent>(OnExecutionWitnessed);

    // ========== 来自 Environment System ==========
    EventBus.Instance.Subscribe<EnvironmentalEvent>(OnEnvironmentEvent);

    // ========== 来自 Health System - NPC 状态变化 ==========
    // NPCStateChangedEvent 由 Health System 发布（通过 NPCStateManager 协调）
    // 订阅此事件以同步 WorldState ↔ HealthState 协议
    // 注意：NPCStateChangedEvent 的 new_world_state 字段表示 NPC 的世界状态（FREE/UNCONSCIOUS/TIED/DEAD）
    // 当 new_world_state 为 DEAD/UNCONSCIOUS/TIED 时，NPC AI 应停止当前行为
    EventBus.Instance.Subscribe<NPCStateChangedEvent>(OnNPCStateChanged);

    // ========== 来自 World Layer - Weather System（通过事件订阅获取感知系数缓存）==========
    EventBus.Instance.Subscribe<WeatherStateChangedEvent>(OnWeatherStateChanged);

    // ========== 来自 World Layer - Lighting System ==========
    EventBus.Instance.Subscribe<LightingStateChangedEvent>(OnLightingStateChanged);

    // ========== 来自 World Layer - Area Lighting ==========
    EventBus.Instance.Subscribe<AreaLightingChangedEvent>(OnAreaLightingChanged);

    // ========== 来自 WorldMap System - 区域进入事件 ==========
    // AreaEnteredEvent 由 WorldMap System 发布，通知 NPC AI 玩家进入了新区域
    // NPC AI 收到此事件后可触发敌人生成逻辑（见 ADR-0012 §CLEARED触发时序）
    EventBus.Instance.Subscribe<AreaEnteredEvent>(OnAreaEntered);

    // ========== 订阅 PerceptionComponent 到 LOS System 事件 ==========
    _perceptionComponent.SubscribeToEvents();

    // ========== 订阅 FactionNetworkComponent 到 LOS System 事件 ==========
    // 派系感知共享需要接收来自同派系 NPC 的感知信息
    _factionComponent.SubscribeToLOSEvents();
}

/// <summary>
/// NPCIdentityConfirmedEvent 处理实现
/// 当 LOS System 的 NPCIdentityManager 确认 NPC 身份后发布此事件
/// NPCAI 收到此事件后应升级为 Enemy 状态
/// ClueJournal 订阅此事件触发线索发现
/// </summary>
private void OnNPCIdentityConfirmed(NPCIdentityConfirmedEvent e)
{
    if (e.npc_id != NpcId) return;

    // NPC 身份被确认为 Enemy/Accomplice 时，触发 AlertState 升级
    // Victim 类型不影响 AlertState
    if (e.identity_type == NPCIdentityType.ENEMY ||
        e.identity_type == NPCIdentityType.ACCOMPLICE)
    {
        // 升级为 ALERT 状态（假设确认身份意味着玩家已被发现）
        _alertFSM.TransitionTo(AlertState.ALERT);
    }
}

/// <summary>
/// EnvironmentalEvent 处理实现
/// NPC AI 订阅 Environment System（ADR-0013）发布的 EnvironmentalEvent
/// </summary>
private void OnEnvironmentEvent(EnvironmentalEvent evt)
{
    // 计算 NPC 与事件源的距离
    float distance = Vector3.Distance(transform.position, evt.Position);
    if (distance > evt.Radius)
        return; // 超出影响范围

    switch (evt.Type)
    {
        case EnvironmentalEventType.EXPLOSION:
        case EnvironmentalEventType.FIRE:
        case EnvironmentalEventType.DESTRUCTION:
            // 强烈事件：立即进入 ALERT
            _alertFSM.TransitionTo(AlertState.ALERT);
            break;

        case EnvironmentalEventType.CHAOS:
            // CHAOS 事件：额外增加 3 秒混乱持续时间
            _alertFSM.ExtendConfusionDuration(3.0f);
            _alertFSM.TransitionTo(AlertState.ALERT);
            break;

        case EnvironmentalEventType.ALERT:
            // 警戒事件：NPC 会关注该位置
            _alertFSM.SetInvestigatePoint(evt.Position);
            _alertFSM.TransitionTo(AlertState.SEARCH);
            break;

        case EnvironmentalEventType.DISTRACTION:
            // 干扰事件：NPC 会短暂分心
            if (_alertFSM.CurrentState == AlertState.UNDETECTED)
            {
                _alertFSM.SetInvestigatePoint(evt.Position);
                // 不改变 AlertState，但行为树会处理调查
            }
            break;
    }
}

/// <summary>
/// NPCStateChangedEvent 处理实现
/// 由 Health System 发布（通过 NPCStateManager 协调）
/// 当 NPC 的 WorldState 变化时（FREE/UNCONSCIOUS/TIED/DEAD），停止当前 AI 行为
/// 定义位置：shared-types.md §5.2
/// </summary>
private void OnNPCStateChanged(NPCStateChangedEvent e)
{
    if (e.npc_id != NpcId) return;

    // 根据 WorldState 变化调整 NPC AI 行为
    switch (e.new_world_state)
    {
        case WorldState.DEAD:
        case WorldState.UNCONSCIOUS:
        case WorldState.TIED:
            // NPC 处于非自由状态，停止当前 AI 行为
            // 注意：这里不直接修改 AlertState，而是通过禁用行为树来停止 AI
            enabled = false;
            break;
        case WorldState.FREE:
            // NPC 恢复自由状态，重新启用 AI
            enabled = true;
            break;
    }
}

/// <summary>
/// AreaEnteredEvent 处理实现
/// 由 WorldMap System 发布，通知玩家进入了新区域
/// NPC AI 收到此事件后可触发敌人生成逻辑（见 ADR-0012 §CLEARED触发时序）
/// 定义位置：shared-types.md §10.3
/// </summary>
private void OnAreaEntered(AreaEnteredEvent e)
{
    // 触发敌人生成逻辑
    // 具体实现取决于区域类型和游戏阶段
    // NPCManager 负责实际的敌人实例化
    NPCManager.Instance.OnPlayerEnteredArea(e.area_id, transform.position);
}

// NPCManager 事件发布（输出）
// CombatStateChangedEvent 由 NPCManager 在检测到战斗状态变更时发布
private void SetupEventPublications()
{
    // 发布给 Sanity/Rage 系统（ADR-0017）
    EventBus.Instance.Publish(new CombatStateChangedEvent { IsInCombat = CheckAnyNPCInCombat() });
}

/// <summary>
/// AlertStateChangedEvent 发布
/// AlertFSM.TransitionTo() 内部在状态转换时调用此方法发布事件
/// 通知 GrittyTakedowns（ADR-0011）等系统 NPC 警戒状态已变化
/// 定义位置：shared-types.md §5.3
/// </summary>
private void PublishAlertStateChanged(AlertState oldState, AlertState newState)
{
    EventBus.Instance.Publish(new AlertStateChangedEvent
    {
        npc_id = NpcId,
        old_state = oldState,
        new_state = newState
    });
}

/// <summary>
/// CombatStateChangedEvent
/// 在任意 NPC AlertState 进入 ALERT 及以上状态时发布（进入战斗）
/// 在所有 NPC AlertState 回到 UNDETECTED 时发布（退出战斗）
/// 消费者：SanityRageMeter (ADR-0017)
/// 定义位置：shared-types.md §5.10
/// </summary>
// 完整结构体定义见 shared-types.md §5.10
```

---

## Alternatives Considered

### Alternative 1: 纯 FSM（无行为树）

- **描述**：使用纯有限状态机实现所有 NPC 行为，每个状态内嵌行为逻辑
- **优点**：
  - 实现简单，状态转换清晰
  - 调试直观，状态变化容易追踪
  - 性能开销最小
- **缺点**：
  - 状态内行为复杂时，状态机会臃肿
  - 行为复用困难（复制粘贴代码）
  - 难以处理"同一状态不同情况"的分支逻辑
- **拒绝理由**：
  - NPC AI 行为复杂度高，纯 FSM 会导致单个状态类代码膨胀
  - 行为复用是刚需（巡逻、搜索、追击等行为多处使用）

### Alternative 2: 纯 GOAP（目标导向规划）

- **描述**：使用 GOAP (Goal-Oriented Action Planning) 实现 NPC 决策
- **优点**：
  - NPC 行为由目标驱动，更"智能"
  - 自然处理多目标冲突
  - 行为库可复用
- **缺点**：
  - 实现复杂度高，学习曲线陡峭
  - 调试困难，行为规划不透明
  - 对于俯视角潜行游戏，GOAP 的"智能"可能是过度设计
- **拒绝理由**：
  - 团队无 GOAP 实战经验
  - 俯视角潜行游戏的 NPC 行为相对模式化，FSM+BT 足够
  - GOAP 的调试困难会拖慢开发进度

### Alternative 3: 状态机 + 行为树（当前选择）

- **优点**：
  - FSM 管理高层状态转换，清晰可控
  - BT 管理状态内行为，复用性强
  - 工具支持成熟（Unity Behavior Tree 插件）
  - 性能可接受（BT 每帧只执行一条路径）
- **缺点**：
  - 两套系统需要协调
  - BT 节点过多时需要管理工具
- **选择理由**：
  - NPC AI 系统状态明确（6 Alert State），适合 FSM
  - 每个状态下的行为需要层次化，BT 完美契合
  - 性能和复杂度的平衡点

---

## Consequences

### Positive

- **清晰的状态转换**：FSM 使得 Alert State 变化一目了然
- **可复用的行为节点**：BT 的节点可被多个 NPC 实例共享
- **感知逻辑解耦**：PerceptionComponent 独立处理感知计算，与行为逻辑分离
- **派系网络模块化**：FactionNetworkComponent 独立管理派系关系，易于测试
- **性能可控**：BT 每帧只执行一条路径，单帧开销小

### Negative

- **双系统复杂度**：需要维护 FSM + BT 两套机制
- **BT 节点管理**：需要配合版本控制和行为树编辑器
- **FSM 状态膨胀风险**：如果新增状态，需要同步更新 FSM 和 BT

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **BT 节点膨胀** | BT 节点过多难以维护 | 按功能分区（Actions/Conditions），每个节点 < 50 行 |
| **FSM 状态冲突** | 多个状态转换条件同时满足 | 明确的优先级规则：发现证据 > 衰减条件 |
| **派系感知循环** | 共享循环 A→B→C→A | 去重列表 + 5秒过期机制 |
| **性能瓶颈** | 100 NPC 同时更新 | 空间分区 + 分帧计算 + 距离预筛选 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 2ms/帧 (100 NPC) | 分帧计算 + 空间分区 |
| **Memory** | < 50MB | NPCData ScriptableObject 共享 |
| **GameObject 数量** | 1/NPC | 每个 NPC 一个 NPCController |

### 性能优化策略

> **⚠️ 示意代码**：以下 NPCManager 实现为示意代码，用于说明分帧计算的概念。生产代码需要经过性能基准测试验证后投入使用。

```csharp
// NPCManager.cs - NPC 更新调度器
public class NPCManager
{
    private List<NPCController> _allNPCs = new();
    private int _currentIndex = 0;
    private int _processedCount = 0;
    private const int NPCsPerFrame = 20; // 每帧最多更新 20 个 NPC

    private void Update()
    {
        // 分帧计算：每帧只处理一部分 NPC
        int toProcess = Mathf.Min(NPCsPerFrame, _allNPCs.Count);
        for (int i = 0; i < toProcess; i++)
        {
            // 安全地获取下一个有效 NPC
            NPCController npc = null;
            int safetyCounter = 0;
            while (npc == null && safetyCounter < _allNPCs.Count)
            {
                _currentIndex = (_currentIndex + 1) % _allNPCs.Count;
                if (_allNPCs[_currentIndex] != null && _allNPCs[_currentIndex].isActiveAndEnabled)
                {
                    npc = _allNPCs[_currentIndex];
                }
                safetyCounter++;
            }

            if (npc != null)
            {
                npc.PerceptionComponent.Update(npc, Time.deltaTime);
                npc.AlertFSM.Update(npc);
            }
        }
    }

    public void RegisterNPC(NPCController npc)
    {
        if (!npc.IsPlayer) // 排除玩家
            _allNPCs.Add(npc);
    }

    public void UnregisterNPC(NPCController npc)
    {
        _allNPCs.Remove(npc);
        if (_currentIndex >= _allNPCs.Count)
            _currentIndex = 0;
    }

    /// <summary>
    /// 获取所有 NPC 列表（供 LOS System 初始化使用）
    /// </summary>
    public List<NPCController> GetAllNPCs()
    {
        return _allNPCs.ToList();  // 返回副本避免外部修改
    }

    /// <summary>
    /// 获取当前正在发声的 NPC 列表（供 LOS System 专注监听模式使用）
    /// </summary>
    public List<NPCController> GetActiveSoundSources()
    {
        var activeNPCs = new List<NPCController>();
        foreach (var npc in _allNPCs)
        {
            if (npc != null && npc.IsActiveAndEnabled && npc.IsCurrentlySpeaking)
            {
                activeNPCs.Add(npc);
            }
        }
        return activeNPCs;
    }

    /// <summary>
    /// 根据 NPC ID 获取 NPC 引用
    /// </summary>
    public NPCController GetNPC(int npcId)
    {
        return _allNPCs.FirstOrDefault(n => n != null && n.NpcId == npcId);
    }
}
```

### 8. 查询接口定义

NPCController 暴露以下查询接口，供其他系统（主要是 Gritty Takedowns）通过 Event Bus 的 Query 机制同步调用：

```csharp
// NPCController.cs - Query 接口暴露
public partial class NPCController : MonoBehaviour
{
    // ========== NPC 状态属性 ==========

    /// <summary>
    /// NPC 是否正在说话/发声（供 LOS System 专注监听模式使用）
    /// 由行为树执行"说话"节点时更新
    /// </summary>
    public bool IsCurrentlySpeaking { get; private set; }

    // 供 LOS System 读取的位置信息
    public Vector3 CurrentPosition => transform.position;

    // ========== Query 接口（供外部系统调用）==========

    /// <summary>
    /// 查询 NPC 当前 Alert State
    /// </summary>
    public AlertState QueryAlertState() => _alertFSM.CurrentState;

    /// <summary>
    /// 查询 NPC 当前 World State
    /// </summary>
    public WorldState QueryState() => _worldState;

    /// <summary>
    /// 查询 NPC 对玩家的态度值
    /// </summary>
    public int QueryAllegiance() => _allegiance;

    /// <summary>
    /// 查询 NPC 的派系忠诚度（用于 GrittyTakedowns 贿赂判定）
    /// </summary>
    public FactionAllegiance QueryFactionAllegiance() => new FactionAllegiance
    {
        faction = _npcData.faction,
        loyalty = _allegiance
    };

    /// <summary>
    /// 查询 NPC 的勇气值（用于贿赂等交互判定）
    /// Bravery 范围 [1, 10]，值越小越容易被吓唬/贿赂
    /// </summary>
    public int QueryBravery() => _npcData.Bravery;

    /// <summary>
    /// 查询 NPC 已知的弱点列表
    /// </summary>
    public List<string> QueryVulnerability() => _vulnerabilities;

    /// <summary>
    /// 查询 NPC 掌握的线索 ID 列表
    /// </summary>
    public List<string> QueryKnowledge() => _knowledge;

    /// <summary>
    /// 查询 NPC 的身份标签
    /// </summary>
    public NPCIdentityType QueryIdentity() => _npcData.identity;

    /// <summary>
    /// 查询 NPC 所属派系
    /// </summary>
    public Faction QueryFaction() => _npcData.faction;

    /// <summary>
    /// 查询 NPC 体型分类（用于 Gritty Takedowns 捆绑时间计算）
    /// </summary>
    public NPCSizeCategory QuerySizeCategory() => _npcData.sizeCategory;

    /// <summary>
    /// 查询 NPC 当前感知评分（用于调试/UI）
    /// </summary>
    public PerceptionScore QueryPerceptionScore()
    {
        return new PerceptionScore
        {
            Visual = _perceptionComponent.VisualScore,
            Audio = _perceptionComponent.AudioScore,
            Memory = _perceptionComponent.MemoryScore,
            Total = _perceptionComponent.GetPerceptionScore()
        };
    }

    // ========== 天气/光照感知折扣查询 ==========

    /// <summary>
    /// 查询 NPC 的有效感知范围（考虑天气和光照影响）
    /// 计算公式：MaxPerceptionRange × visionDistanceMultiplier × shadowStealthMultiplier
    /// 最低保留 30% 的基础感知范围
    ///
    /// **架构约束**：WeatherSystem 和 LightingSystem 属于 World Layer，
    /// 不得被其他系统直接调用。必须通过事件订阅机制获取感知系数。
    /// NPC AI 通过订阅 WeatherStateChangedEvent 和 LightingStateChangedEvent 缓存最新值。
    ///
    /// **感知系数来源**（按 shared-types.md §22.4 叠加规则）：
    /// - visionDistanceMultiplier：来自 WeatherStateChangedEvent.perceptionModifier
    /// - shadowStealthMultiplier：来自 LightingStateChangedEvent.perceptionModifier
    /// </summary>
    public float QueryEffectivePerceptionRange()
    {
        var baseRange = _npcData.maxPerceptionRange;
        // 从缓存中读取（由 OnWeatherStateChanged/OnLightingStateChanged 事件回调更新）
        var weatherMod = _cachedWeatherModifier?.visionDistanceMultiplier ?? 1.0f;
        var lightMod = _cachedLightModifier?.shadowStealthMultiplier ?? 1.0f;

        var effectiveRange = baseRange * weatherMod * lightMod;
        return Mathf.Max(effectiveRange, baseRange * 0.3f);
    }

    /// <summary>
    /// 查询玩家是否在 NPC 的阴影隐蔽区域内
    ///
    /// **架构约束**：不得直接持有 LightingSystem 引用。
    /// 通过 AreaLightingChangedEvent 事件回调缓存当前区域的光照状态。
    /// </summary>
    public bool QueryIsPlayerInShadow()
    {
        // 从缓存中读取（由 OnAreaLightingChanged 事件回调更新）
        return _cachedShadowStealthBonus > 1.0f;
    }

    // ========== 事件订阅（World Layer 感知系数缓存） ==========
    // NPC AI 订阅 WeatherStateChangedEvent 和 LightingStateChangedEvent，
    // 在事件回调中缓存最新的感知系数，供 QueryEffectivePerceptionRange 使用。
    // 这样遵守了 World Layer 不得被直接调用的架构约束。

    private PerceptionModifier? _cachedWeatherModifier;
    private AreaLightingCoefficients? _cachedLightModifier;
    private float _cachedShadowStealthBonus = 1.0f;

    /// <summary>
    /// 天气状态变化事件回调（应在 SetupEventSubscriptions 中订阅）
    /// </summary>
    private void OnWeatherStateChanged(WeatherStateChangedEvent evt)
    {
        _cachedWeatherModifier = evt.perception_modifier;
    }

    /// <summary>
    /// 光照状态变化事件回调（应在 SetupEventSubscriptions 中订阅）
    /// </summary>
    private void OnLightingStateChanged(LightingStateChangedEvent evt)
    {
        _cachedLightModifier = evt.perceptionModifier;
        _cachedShadowStealthBonus = evt.perceptionModifier.shadowStealthBonus;
    }

    /// <summary>
    /// 区域光照变化事件回调（应在 SetupEventSubscriptions 中订阅）
    /// AreaLightingChangedEvent 不携带感知系数，需通过 AreaLightingCoefficientsTable 查表获取
    /// </summary>
    private void OnAreaLightingChanged(AreaLightingChangedEvent evt)
    {
        if (evt.isPlayerInside)
        {
            // AreaLightingChangedEvent 仅携带 lightingState，感知系数需查 AreaLightingCoefficientsTable
            var coefficients = AreaLightingCoefficientsTable.Get(evt.lightingState);
            _cachedShadowStealthBonus = coefficients.shadowStealthMultiplier;
        }
    }

    // ========== GrittyTakedowns 专用接口 ==========

    /// <summary>
    /// 查询 NPC 的欺骗抗性（用于判断贿赂/欺骗是否成功）
    /// 范围 [0, 1]，值越高越难被欺骗
    /// </summary>
    public float QueryDeceptionResistance() => _npcData.deceptionResistance;

    /// <summary>
    /// 查询 NPC 的对话树数据（供 DialogueUIManager 渲染对话选项）
    /// </summary>
    public DialogueTreeConfig QueryDialogueTree() => _npcData.dialogueTree;

    /// <summary>
    /// 查询 NPC 的健康状态（用于判断是否可以捆绑）
    /// </summary>
    public HealthState QueryHealthState() => _healthComponent.CurrentState;
}

public struct PerceptionScore
{
    public float Visual;
    public float Audio;
    public float Memory;
    public float Total;
}
```

> **接口调用说明**：以上 Query 接口通过 Event Bus 的 Query 模式调用，具体见 `event-bus-icd.md` 中定义的 `QueryAlertState`、`QueryState` 等查询接口。所有接口均为同步调用，返回当前帧的即时状态。


---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 NPCData ScriptableObject 定义
- [ ] 创建 AlertState/WorldState 枚举
- [ ] 创建 AlertFSM 状态机核心
- [ ] 创建 NPCController 基础结构

### Phase 2: 感知系统
- [ ] 创建 PerceptionComponent
- [ ] 实现 VisualScore 更新（订阅 PlayerSpottedEvent）
- [ ] 实现 AudioScore 更新（订阅 NoiseMadeEvent）
- [ ] 实现 MemoryScore 衰减逻辑

### Phase 3: 行为树
- [ ] 创建 BT 节点基类
- [ ] 实现 Composite Nodes (Selector, Sequence)
- [ ] 实现基础 Action Nodes (Patrol, Investigate, Combat, Escape)
- [ ] 实现基础 Condition Nodes

### Phase 4: 派系网络
- [ ] 创建 FactionRelations 派系关系矩阵
- [ ] 实现 FactionNetworkComponent
- [ ] 实现 SharedAlert 广播/接收逻辑
- [ ] 实现去重机制

### Phase 5: 集成测试
- [ ] Alert State 转换测试
- [ ] 感知评分计算测试
- [ ] 派系感知共享测试
- [ ] 100 NPC 性能压力测试

---

## Validation Criteria

1. **状态转换正确性**：UNDETECTED→SUSPECT→SEARCH→ALERT→ESCAPE→COMBAT 的所有转换路径可正常执行
2. **感知评分准确性**：VisualScore/AudioScore/MemoryScore 的计算结果符合设计文档公式
3. **派系感知去重**：同一来源的 SharedAlert 在 5 秒内只处理一次
4. **性能达标**：100 NPC 同时运行，单帧感知更新 < 2ms
5. **事件订阅正确**：所有事件（PlayerSpottedEvent/NoiseMadeEvent/ExecutionWitnessedEvent）正确订阅
6. **BT 行为可切换**：不同 Alert State 下 BT 根选择器返回不同行为分支
7. **Query 接口正确**：QueryAlertState/QueryState/QueryAllegiance 等接口返回正确值
8. **天气/光照折扣正确**：QueryEffectivePerceptionRange 正确应用天气和光照折扣

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — NPC AI 通过 Event Bus 与其他系统通信
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — NPC AI 属于 Core Layer，Weather/Lighting 属于 World Layer
- [ADR-0005: 存档/持久化架构](./adr-0005-save-persistence-architecture.md) — NPC AI 发送 AreaCleared 事件触发存档
- [ADR-0017: 理智/愤怒系统](./adr-0017-sanity-rage-meter-architecture.md) — NPC AI 发布 CombatStateChangedEvent 供理智/愤怒系统判断 IsInCombat
- [NPC AI System GDD](../../design/gdd/npc-ai-system.md) — 本 ADR 的设计依据
- [Weather System GDD](../../design/gdd/weather-system.md) — World Layer 系统，影响 NPC 感知范围
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — NPC AI 订阅/发布的事件定义，以及 Query 接口定义
