# ADR-0011: 沉重处决系统 (Gritty Takedowns) 架构决策

## Status
**Proposed**（依赖 ADR-0010 Weapon System + shared-types.md 的 APPROVED 类型定义）

> **v1.4.1 更新**：修复以下评审问题
> - DECEPTION_RESISTANCE_IMMUNE 配置化
> - COMBAT/ESCAPE 检查逻辑提取
> - PlayerInteractionFSM 状态转换说明
> - Dialogue UI 边界澄清
> - QueryAvailableInteractions 性能策略

> **shared-types.md 类型完整性验证**：以下类型已在 shared-types.md（v1.3.0, APPROVED）中正确定义：
> - `DamageRequest`（§2.3）✅
> - `ExplosionEvent`（§2.4）✅
> - `PlayerDamagedEvent`（§7.5）✅
> - `WeaponQueryRequest/WeaponQueryResponse`（§3.8）✅
> - `AlertStateChangedEvent`（§5.4）✅
> - `NPCStateChangedEvent`（§5.2）✅
> - `WorldState`（§5.1）✅
> - `AlertState`（§5.3）✅
> - `NPCIdentityType`（§5.5）✅
> - `NPCSizeCategory`（§5.6）✅
> - `FactionAllegiance`（§5.7）✅
> - `InteractionType`（§9.1）✅
> - `InteractionResult`（§9.2）✅
> - `PlayerInteractionState`（§9.4）✅
> - `ActionLockType`（§4.1）✅
> - `ActionLockSystem`（§4.2）✅
> - `HealthState`（§6.1）✅

## Date
2026-04-10

## Last Updated
2026-04-11 (v1.4.2 — 评审修复)

## Context

### Problem Statement

Gritty Takedowns（玩家-NPC 交互系统）是《断绝：罪恶之源》"沉重、不洁的暴力"核心体验的操作层实现。系统管理玩家对 NPC 的所有主动交互——处决、威胁、审讯、转化等，是"线索驱动的动态潜行"核心体验的关键支柱。系统需要处理：

1. **交互类型矩阵**：由 NPC 状态（FREE/UNCONSCIOUS/TIED/DEAD）和身份标签决定可用交互
2. **交互状态机**：Idle/CanInteract/Interacting 的玩家交互流程管理
3. **动作锁定**：交互期间通过 ActionLockSystem 接管玩家控制权
4. **伤害请求**：向 Health 系统发送 DamageRequest（含 penetration 参数）
5. **武器数据查询**：向 Weapon System 查询 WeaponData 获取动画标签

### Constraints

- **设计约束**：玩家-NPC 交互系统的操作层职责是"何时交互"和"交互效果"，而非"武器是什么"
- **状态约束**：潜行击杀需要满足（背面 + 潜行 + 已标记恶徒 + Alert State < ALERT）
  - "已标记恶徒"通过 `NPCIdentityType == ENEMY` 判断，身份标签由 LOS System（ADR-0007）维护并通过 NPCController.QueryIdentity() 提供
- **时间约束**：捆绑时间 3-6 秒（受 NPC 体型和玩家技能影响）
  - NPC 体型通过 `NPCController.QuerySizeCategory()` 获取（定义见 shared-types.md §5.6）
- **事件约束**：通过 Event Bus 与 NPC AI 系统解耦，避免循环依赖
- **对话约束**：DialogueTree 数据归 NPC AI 系统所有（Core Layer），Gritty Takedowns 仅负责 UI 渲染
- **性能约束**：单次交互响应 < 1 帧，动画锁定期间 CPU 占用 < 2ms

### Requirements

- **必须**：定义交互类型矩阵（7 类交互：处决/威胁/审讯/转化/捆绑/搜身/对话）
- **必须**：定义交互触发条件（NPC 状态、身份标签、Alert State）
- **必须**：定义玩家交互状态机（Idle/CanInteract/Interacting）
- **必须**：定义与 NPC AI 系统的 Query 接口契约
- **必须**：定义与 Weapon System 的 WeaponQuery 接口
- **必须**：定义 ActionLockSystem 接管机制
- **必须**：遵循 ADR-0003 系统分层定义（Gritty Takedowns 属于 Feature Layer）

---

## Decision

### 架构决策

采用**交互类型矩阵 + 状态机驱动 + 事件解耦**架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 Gritty Takedowns (玩家-NPC 交互系统) 架构                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │              GrittyTakedownsSystem (Feature Layer)                      │   │
│  │  - 管理玩家交互状态机（Idle/CanInteract/Interacting）                │   │
│  │  - 维护交互类型矩阵查询                                              │   │
│  │  - 订阅上游事件，更新可用交互选项                                     │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│          ┌─────────────────────────┴─────────────────────────┐          │
│          ▼                                                   ▼          │
│  ┌───────────────────┐                           ┌───────────────────┐   │
│  │ InteractionMatrix  │                           │  PlayerInteractionFSM │   │
│  │ (交互类型矩阵)      │                           │  (玩家交互状态机)   │   │
│  │                   │                           │                   │   │
│  │ 根据 NPC状态       │                           │ Idle            │   │
│  │ + 身份标签         │                           │   ↓             │   │
│  │ + 玩家持有信息     │                           │ CanInteract      │   │
│  │ + AlertState      │                           │   ↓             │   │
│  │ → 确定可用交互     │                           │ Interacting      │   │
│  │                   │                           │                   │   │
│  └───────────────────┘                           └───────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │              Foundation/Shared/ActionLockSystem                          │   │
│  │  - 单例模式（ActionLockSystem.Instance）                              │   │
│  │  - Gritty Takedowns / Dialogue / Cutscene / Stagger 共用           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Event Subscriptions                             │   │
│  │  - AlertStateChangedEvent (NPC AI) → NPC 警戒状态变化              │   │
│  │  - NPCStateChangedEvent (NPC AI) → NPC 世界状态变化                 │   │
│  │  - PlayerDamagedEvent (Health) → 玩家被攻击，打断交互               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Event Publications                              │   │
│  │  - InteractionEvent → NPC AI/音频                                    │   │
│  │  - DamageRequest → Health 系统                                      │   │
│  │  - KillTagEvent → Sanity 系统                                       │   │
│  │  - KnowledgeGainedEvent → Clue 系统                                 │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Event Queries（请求/响应）                         │   │
│  │  - WeaponQueryRequest → Weapon System                               │   │
│  │    ← WeaponQueryResponse（返回 WeaponData）                         │   │
│  │  - DialogueChoice → NPC AI System（返回 DialogueResult）             │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│                           ▼ 与其他系统交互 ▼                              │
│                                                                          │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐      │
│  │  Weapon System  │   │   Health System  │   │   NPC AI System  │      │
│  │  (ADR-0010)    │   │   (ADR-0008)    │   │   (ADR-0004)    │      │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. 交互类型矩阵

```csharp
// InteractionType 定义于 shared-types.md §9.1
// StealthKillAlertMode 定义于 shared-types.md §15.1
using StealthKillAlertMode = SharedTypes.StealthKillAlertMode;

// InteractionAvailability.cs
public struct InteractionAvailability
{
    public InteractionType type;
    public bool is_available;
    public string requirement_hint;  // 不可用时显示原因
}

/// <summary>
/// 玩家交互上下文
/// </summary>
public struct PlayerInteractionContext
{
    public bool hasVulnerability;        // 玩家已获取目标弱点
    public bool isPlayerCrouching;      // 玩家正在潜行
    public bool isBehindTarget;         // 玩家在目标背面
    public bool hasEnvironmentWeapon;  // 玩家持有环境物件
}

public class InteractionMatrix
{
    private readonly GrittyTakedownsTuningSO _tuning;

    /// <summary>
    /// 缓存：上次查询的 NPC ID、上下文哈希和结果
    /// </summary>
    private (int npcId, int contextHash, List<InteractionAvailability> cached) _cache;

    public InteractionMatrix() : this(null) { }

    public InteractionMatrix(GrittyTakedownsTuningSO tuning)
    {
        _tuning = tuning;
    }

    /// <summary>
    /// 计算上下文的哈希值（用于缓存比较）
    /// </summary>
    private int ComputeContextHash(PlayerInteractionContext context)
    {
        // 使用简单组合哈希：可根据需要调整
        int hash = 0;
        hash ^= context.hasVulnerability.GetHashCode();
        hash ^= context.isPlayerCrouching.GetHashCode() << 1;
        hash ^= context.isBehindTarget.GetHashCode() << 2;
        hash ^= context.hasEnvironmentWeapon.GetHashCode() << 3;
        return hash;
    }

    /// <summary>
    /// 更新缓存（内部方法，由 QueryAvailableInteractions 调用）
    /// </summary>
    private void UpdateCache(int npcId, PlayerInteractionContext context, List<InteractionAvailability> result)
    {
        _cache = (npcId, ComputeContextHash(context), result);
    }

    /// <summary>
    /// 查询对特定 NPC 可用的交互类型
    /// </summary>
    /// <remarks>
    /// <b>调用频率限制</b>：此方法应仅在以下时机调用：
    /// 1. 玩家交互范围发生变化时（进入/离开 NPC 范围）
    /// 2. NPC 状态发生变化时（AlertStateChangedEvent、NPCStateChangedEvent）
    /// 3. 玩家持有物品发生变化时（WeaponStateChangedEvent）
    /// 4. 玩家获取新线索时（KnowledgeGainedEvent）
    ///
    /// <b>禁止每帧调用</b>：此方法涉及多次 NPCController 查询，每帧调用会造成性能问题。
    /// 如需每帧更新 UI 显示，应使用缓存的结果或订阅上述事件自行维护缓存。
    ///
    /// <b>内部缓存策略</b>：内部缓存上次查询结果（npcId + context 哈希），
    /// 同一目标且上下文未变化时返回缓存结果，避免重复计算。
    /// </remarks>
    public List<InteractionAvailability> QueryAvailableInteractions(
        NPCController npc,
        PlayerInteractionContext context)
    {
        // 检查缓存：同一目标且上下文未变化时返回缓存结果
        int npcId = npc.GetInstanceID();
        int contextHash = ComputeContextHash(context);
        if (_cache.npcId == npcId && _cache.contextHash == contextHash)
        {
            return _cache.cached;
        }

        // 预取 NPC 状态（避免重复查询）
        WorldState npcState = npc.QueryState();
        NPCIdentityType identity = npc.QueryIdentity();
        AlertState alertState = npc.QueryAlertState();

        // 预构建状态上下文（供 Can* 方法使用）
        var availabilityContext = new AvailabilityContext
        {
            Npc = npc,
            NpcState = npcState,
            Identity = identity,
            AlertState = alertState,
            HasVulnerability = context.hasVulnerability,
            IsPlayerCrouching = context.isPlayerCrouching,
            IsBehindTarget = context.isBehindTarget,
            HasEnvironmentWeapon = context.hasEnvironmentWeapon,
            NpcHealthState = npc.QueryHealthState()
        };

        var results = new List<InteractionAvailability>();

        // 处决类
        results.Add(QueryStealthKill(availabilityContext));
        results.Add(QueryEnvironmentKill(availabilityContext));
        results.Add(QueryFinishOff(availabilityContext));

        // 情报类
        results.Add(QuerySearch(availabilityContext));
        results.Add(QueryInterrogate(availabilityContext));

        // 控制类
        results.Add(QueryTieUp(availabilityContext));
        results.Add(QueryRelease(availabilityContext));

        // 强制类
        results.Add(QueryIntimidate(availabilityContext));
        results.Add(QueryBribe(availabilityContext));
        results.Add(QueryDeceive(availabilityContext));

        // 对峙类
        results.Add(QueryMaintainDistance(availabilityContext));
        results.Add(QueryDialogue(availabilityContext));

        // 转化类
        results.Add(QueryConvert(availabilityContext));

        // 更新缓存
        UpdateCache(npcId, context, results);

        return results;
    }

    #region 交互类型查询方法

    private InteractionAvailability QueryStealthKill(AvailabilityContext ctx)
    {
        bool available = CanStealthKill(ctx.NpcState, ctx.Identity,
            ctx.IsPlayerCrouching, ctx.IsBehindTarget, ctx.AlertState);
        return new InteractionAvailability
        {
            type = InteractionType.StealthKill,
            is_available = available,
            requirement_hint = available ? "" : "需要背面潜行且目标未警觉"
        };
    }

    private InteractionAvailability QueryEnvironmentKill(AvailabilityContext ctx)
    {
        bool available = CanEnvironmentKill(ctx.NpcState, ctx.Identity, ctx.HasEnvironmentWeapon);
        return new InteractionAvailability
        {
            type = InteractionType.EnvironmentKill,
            is_available = available,
            requirement_hint = available ? "" : "需要持有环境物件且目标可被处决"
        };
    }

    private InteractionAvailability QueryFinishOff(AvailabilityContext ctx)
    {
        bool available = CanFinishOff(ctx.NpcState);
        return new InteractionAvailability
        {
            type = InteractionType.FinishOff,
            is_available = available,
            requirement_hint = available ? "" : "目标必须处于无意识或捆绑状态"
        };
    }

    private InteractionAvailability QuerySearch(AvailabilityContext ctx)
    {
        bool available = CanSearch(ctx.NpcState);
        return new InteractionAvailability
        {
            type = InteractionType.Search,
            is_available = available,
            requirement_hint = available ? "" : "目标必须处于无意识或死亡状态"
        };
    }

    private InteractionAvailability QueryInterrogate(AvailabilityContext ctx)
    {
        bool available = CanInterrogate(ctx.NpcState);
        return new InteractionAvailability
        {
            type = InteractionType.Interrogate,
            is_available = available,
            requirement_hint = available ? "" : "目标必须处于无意识状态"
        };
    }

    private InteractionAvailability QueryTieUp(AvailabilityContext ctx)
    {
        bool available = CanTieUp(ctx.NpcState, ctx.NpcHealthState);
        return new InteractionAvailability
        {
            type = InteractionType.TieUp,
            is_available = available,
            requirement_hint = available ? "" : "目标必须处于无意识且可捆绑状态"
        };
    }

    private InteractionAvailability QueryRelease(AvailabilityContext ctx)
    {
        bool available = CanRelease(ctx.NpcState);
        return new InteractionAvailability
        {
            type = InteractionType.Release,
            is_available = available,
            requirement_hint = available ? "" : "目标必须处于捆绑状态"
        };
    }

    private InteractionAvailability QueryIntimidate(AvailabilityContext ctx)
    {
        // ⚠️ AlertState 检查：COMBAT 或 ESCAPE 状态下 NPC 不会响应威胁
        bool available = ctx.NpcState == WorldState.FREE
            && IsNpcInteractable(ctx.AlertState);
        return new InteractionAvailability
        {
            type = InteractionType.Intimidate,
            is_available = available,
            requirement_hint = available ? "" : "目标处于战斗状态，无法威胁"
        };
    }

    private InteractionAvailability QueryBribe(AvailabilityContext ctx)
    {
        // ⚠️ AlertState 检查：COMBAT 或 ESCAPE 状态下 NPC 不会响应贿赂
        bool available = ctx.NpcState == WorldState.FREE
            && IsNpcInteractable(ctx.AlertState)
            && CanBribe(ctx);
        return new InteractionAvailability
        {
            type = InteractionType.Bribe,
            is_available = available,
            requirement_hint = available ? "" : "需要目标可被贿赂"
        };
    }

    private InteractionAvailability QueryDeceive(AvailabilityContext ctx)
    {
        // ⚠️ AlertState 检查：COMBAT 或 ESCAPE 状态下 NPC 不会响应欺骗
        bool available = ctx.NpcState == WorldState.FREE
            && IsNpcInteractable(ctx.AlertState)
            && ctx.HasVulnerability
            && CanDeceive(ctx);
        return new InteractionAvailability
        {
            type = InteractionType.Deceive,
            is_available = available,
            requirement_hint = ctx.HasVulnerability ? "" : "需要先获取目标弱点"
        };
    }

    private InteractionAvailability QueryMaintainDistance(AvailabilityContext ctx)
    {
        bool available = CanMaintainDistance(ctx.NpcState, ctx.AlertState);
        return new InteractionAvailability
        {
            type = InteractionType.MaintainDistance,
            is_available = available,
            requirement_hint = available ? "" : "目标处于战斗状态，无法威胁"
        };
    }

    private InteractionAvailability QueryDialogue(AvailabilityContext ctx)
    {
        // ⚠️ AlertState 检查：COMBAT 或 ESCAPE 状态下 NPC 不会响应对话
        bool available = ctx.NpcState == WorldState.FREE
            && IsNpcInteractable(ctx.AlertState);
        return new InteractionAvailability
        {
            type = InteractionType.Dialogue,
            is_available = available,
            requirement_hint = available ? "" : "目标处于战斗状态，无法对话"
        };
    }

    private InteractionAvailability QueryConvert(AvailabilityContext ctx)
    {
        // ⚠️ AlertState 检查：COMBAT 或 ESCAPE 状态下 NPC 不会响应转化
        // ⚠️ LOYAL 派系检查：派系忠诚度为 LOYAL 时 NPC 不可被转化
        bool available = ctx.NpcState == WorldState.FREE
            && IsNpcInteractable(ctx.AlertState)
            && ctx.HasVulnerability
            && ctx.Npc.QueryFactionAllegiance() != FactionAllegiance.LOYAL;
        return new InteractionAvailability
        {
            type = InteractionType.Convert,
            is_available = available,
            requirement_hint = available ? "" : "需要先获取目标弱点"
        };
    }

    #endregion

    #region 可用性判定内部方法

    /// <summary>
    /// 预取状态上下文（避免重复查询）
    /// </summary>
    private struct AvailabilityContext
    {
        public NPCController Npc;
        public WorldState NpcState;
        public NPCIdentityType Identity;
        public AlertState AlertState;
        public bool HasVulnerability;
        public bool IsPlayerCrouching;
        public bool IsBehindTarget;
        public bool HasEnvironmentWeapon;
        public HealthState NpcHealthState;
    }

    /// <summary>
    /// 检查 NPC 是否处于可交互状态（非战斗/非逃跑）
    /// </summary>
    /// <remarks>
    /// <b>设计理由</b>：COMBAT 或 ESCAPE 状态下 NPC 不会响应威胁、贿赂、欺骗、转化、对话等交互。
    /// 将此检查提取为独立方法，避免多个 Can* 方法中重复相同的检查逻辑。
    /// </remarks>
    /// <param name="alertState">NPC 当前 AlertState</param>
    /// <returns>true = 可交互，false = 处于 COMBAT 或 ESCAPE 状态</returns>
    private bool IsNpcInteractable(AlertState alertState)
    {
        return alertState != AlertState.COMBAT && alertState != AlertState.ESCAPE;
    }

    private bool CanBribe(AvailabilityContext ctx)
    {
        // 贿赂可用条件：
        // 1. NPC 处于 FREE 状态
        // 2. Bravery <= bribeBraveryThreshold（来自 TuningSO）
        // 3. FactionAllegiance != LOYAL（派系忠诚度为 LOYAL 时不可被贿赂）
        //
        // Bravery 由 NPCData 定义（见 ADR-0004 §NPCData），通过 QueryBravery() 接口获取
        // Bravery 范围 [1, 10]，值越小越容易被吓唬/贿赂
        //
        // FactionAllegiance 由 NPCData 定义，通过 QueryFactionAllegiance() 接口获取
        // 派系忠诚度为 LOYAL 时 NPC 不会背叛其派系
        int threshold = _tuning?.bribeBraveryThreshold ?? 3;  // 默认值为 3
        bool braveryCheck = ctx.Npc.QueryBravery() <= threshold;
        bool loyaltyCheck = ctx.Npc.QueryFactionAllegiance() != FactionAllegiance.LOYAL;

        return braveryCheck && loyaltyCheck;
    }

    private bool CanDeceive(AvailabilityContext ctx)
    {
        // 欺骗可用条件：NPC 处于 FREE 状态且玩家已获取其 vulnerability
        // DeceptionResistance 由 NPCData 定义（见 ADR-0004 §NPCData），通过 QueryDeceptionResistance() 接口获取
        // DeceptionResistance 范围 [0, 1]，值越高表示 NPC 越难被欺骗
        //
        // 欺骗免疫阈值由 TuningSO 配置（deceptionResistanceImmuneThreshold，默认 1.0f）
        // 当 npcDeceptionResistance >= deceptionResistanceImmuneThreshold 时 NPC 完全免疫欺骗
        float immuneThreshold = _tuning?.deceptionResistanceImmuneThreshold ?? 1.0f;
        return ctx.Npc.QueryDeceptionResistance() < immuneThreshold;
    }

    private bool CanMaintainDistance(WorldState state, AlertState alert)
    {
        // 保持距离威胁：判定玩家是否可对目标使用"保持距离"交互
        // 可用条件：NPC 处于 FREE 状态且未处于战斗状态
        // 效果说明（见 ADR-0004 NPC AI）：NPC 会进入 GUARDED 状态，暂时停止追击直到警戒超时或发现新目标
        //
        // ⚠️ AlertState 检查：使用 IsNpcInteractable 统一判断
        return state == WorldState.FREE && IsNpcInteractable(alert);
    }

    private bool CanStealthKill(WorldState state, NPCIdentityType identity,
        bool isCrouching, bool isBehind, AlertState alert)
    {
        // 潜行击杀条件：背面 + 潜行 + 已标记恶徒 + 未完全警觉
        //
        // **⚠️ Playtest 重点**：默认 Tight 模式下可能在某些场景下过于困难。
        //   TuningSO.stealthKillAlertMode 控制检查模式（定义于 shared-types.md §15.1）：
        //   - Tight（默认）：仅 UNDETECTED 可执行
        //   - Loose：UNDETECTED / SUSPECT / SEARCH 均可执行，允许玩家失误后补救
        //
        // alert < ALERT 允许在以下状态执行：
        //   - UNDETECTED：完全未被发现，最佳时机
        //   - SUSPECT：NPC 怀疑但未确认，仍有潜行机会（仅 Loose 模式）
        //   - SEARCH：NPC 正在搜索但未发现玩家（仅 Loose 模式）
        //
        // **Loose 模式时间窗口说明**：
        //   Loose 模式允许在 SUSPECT/SEARCH 状态下执行潜行击杀，但这不代表"无限制时间窗口"。
        //   - 在 SUSPECT 状态时，玩家必须在 NPC 确认玩家位置（转入 SEARCH）之前完成击杀
        //   - 在 SEARCH 状态时，玩家必须在 NPC 发现玩家（转入 ALERT）之前完成击杀
        //   - 实际上是一个"紧迫的补救窗口"而非"无限制机会"
        //   - 设计意图：给玩家一个在失误后挽回的途径，但仍然要求快速和精准的行动
        //
        // **设计理由（默认 Tight 模式）**：
        //   - 保证游戏挑战性：潜行击杀应该是高风险高回报的精英操作
        //   - Tight 模式鼓励玩家谨慎行动，避免失误
        //   - 如果 Tight 模式过于困难导致体验下降，可切换到 Loose 模式作为补救
        bool alertCheck = _tuning?.stealthKillAlertMode == StealthKillAlertMode.Tight
            ? alert == AlertState.UNDETECTED
            : alert < AlertState.ALERT;

        return state == WorldState.FREE
            && identity == NPCIdentityType.ENEMY
            && isCrouching
            && isBehind
            && alertCheck;
    }

    /// <summary>
    /// 环境处决可用性判定
    /// </summary>
    /// <remarks>
    /// <b>与 CanTieUp 的关键差异</b>：
    /// - 本方法仅检查 <c>WorldState.UNCONSCIOUS</c>，不检查 <c>HealthState.DOWNED</c>
    /// - 捆绑（CanTieUp）必须同时满足 <c>WorldState.UNCONSCIOUS</c> + <c>HealthState.DOWNED</c>
    ///
    /// **设计理由**：
    /// 处决是终结行为，只需 NPC 失去抵抗能力即可执行；捆绑需要 NPC 处于可安全接近的倒地状态。
    /// 详见 shared-types.md §6.1.1 同步协议。
    /// </remarks>
    private bool CanEnvironmentKill(WorldState state, NPCIdentityType identity, bool hasWeapon)
    {
        // 环境处决可在 FREE（正常状态）或 UNCONSCIOUS（已击倒）状态下执行
        // STAGGERED/DOWNED 状态由 Health System 的 HealthState 管理（见 shared-types.md §6.1）
        // 此处 WorldState 只表示 NPC AI 基础状态，与 HealthState 独立
        //
        // **执行后状态转移**：
        // - 从 FREE 执行：NPC 直接进入 DEAD 状态（通过 HealthSystem 处理 LETHAL 伤害）
        // - 从 UNCONSCIOUS 执行：NPC 直接进入 DEAD 状态（补刀行为）
        //
        // **设计理由**：允许在 UNCONSCIOUS 状态下执行环境处决，因为：
        //   1. 玩家可能先用 BLUNT 武器击倒 NPC（进入 UNCONSCIOUS），再用环境物件补刀
        //   2. 这符合"环境即武器"的核心理念——利用周围一切物品进行击杀
        //   3. 与 FinishOff 的 UNCONSCIOUS/TIED 状态互补（FinishOff 更侧重终结倒地目标）
        //
        // ⚠️ 重要：WorldState.UNCONSCIOUS 与 HealthState.DOWNED 的映射关系
        // 需满足以下条件之一：
        //   1. NPC AI 系统在 NPC 被 Health System 击倒时同步设置 WorldState = UNCONSCIOUS
        //   2. 或 LOS System 的身份标签在 DOWNED 状态下仍然保持 ENEMY（允许补刀）
        // 具体实现见 ADR-0004 NPC AI 和 ADR-0008 Health System 的状态同步协议
        return (state == WorldState.FREE || state == WorldState.UNCONSCIOUS)
            && identity == NPCIdentityType.ENEMY
            && hasWeapon;
    }

    private bool CanFinishOff(WorldState state)
    {
        // 补刀（FinishOff）用于终结已经倒地的 NPC
        // 与 EnvironmentKill 的区别：
        //   - EnvironmentKill：需要玩家持有环境物件，使用物件进行处决
        //   - FinishOff：可以使用徒手或任何武器，终结倒地目标（UNCONSCIOUS/TIED）
        // 设计意图：倒地后的 NPC 无法反抗，补刀无需潜行条件
        return state == WorldState.UNCONSCIOUS || state == WorldState.TIED;
    }

    private bool CanSearch(WorldState state)
    {
        return state == WorldState.UNCONSCIOUS || state == WorldState.DEAD;
    }

    private bool CanInterrogate(WorldState state)
    {
        return state == WorldState.UNCONSCIOUS;
    }

    private bool CanTieUp(WorldState state, HealthState healthState)
    {
        // 捆绑条件：WorldState.UNCONSCIOUS 且 HealthState.DOWNED
        //
        // **⚠️ WorldState ↔ HealthState 同步约束**（shared-types.md §6.1.1）：
        // WorldState.UNCONSCIOUS 仅在 HealthState.DOWNED 时可由 Gritty Takedowns 捆绑。
        // 这是因为捆绑需要 NPC 处于可安全接近的倒地状态。
        //
        // **设计理由**：EnvironmentKill 与 TieUp 的状态要求不同
        // - EnvironmentKill：仅检查 WorldState.UNCONSCIOUS（处决是终结行为，只需 NPC 失去抵抗能力）
        // - TieUp：必须同时检查 WorldState.UNCONSCIOUS + HealthState.DOWNED（捆绑需要可安全接近的倒地状态）
        return state == WorldState.UNCONSCIOUS && healthState == HealthState.DOWNED;
    }

    private bool CanRelease(WorldState state)
    {
        return state == WorldState.TIED;
    }

    // 注意：CanConvert 逻辑已内联到 QueryConvert 中，此处不再需要独立方法
}
```

### 2. 玩家交互状态机

> **状态转换触发条件说明**
>
> 玩家交互状态机的状态转换由 **GrittyTakedownsSystem** 驱动，触发条件如下：
>
> | 当前状态 | 目标状态 | 触发条件 |
> |---------|---------|---------|
> | Idle | CanInteract | 玩家按下交互键（E）且准星范围内存在有效 NPC 目标 |
> | CanInteract | Idle | 玩家按下 ESC/取消键，或目标离开范围，或目标死亡 |
> | CanInteract | Interacting | 玩家按下交互键（E）确认执行交互 |
> | Interacting | Idle | 交互动画播放完成（正常结束），或交互被打断（被打、被杀） |
>
> **关键设计**：
> - **自动检测 vs 手动触发**：GrittyTakedownsSystem 每帧检测范围内有效 NPC，自动更新 CanInteract 状态
> - **交互确认**：CanInteract → Interacting 需要玩家再次按键确认，防止误触
> - **打断机制**：PlayerDamagedEvent 打断当前交互，强制转入 Idle

```csharp
// PlayerInteractionState 定义于 shared-types.md §9.4

// PlayerInteractionFSM.cs
public class PlayerInteractionFSM
{
    private PlayerInteractionState _currentState = PlayerInteractionState.Idle;
    private int _lockedTargetId = -1;
    private InteractionType _interactionType;  // 使用 InteractionType 枚举
    private bool _actionLockAcquired = false;   // 追踪锁获取状态

    public void Transition(PlayerInteractionState newState, int targetId = -1, InteractionType interactionType = default)
    {
        var oldState = _currentState;
        _currentState = newState;

        switch (newState)
        {
            case PlayerInteractionState.Idle:
                ReleaseActionLock();
                _lockedTargetId = -1;
                _interactionType = default;
                break;

            case PlayerInteractionState.CanInteract:
                _lockedTargetId = targetId;
                break;

            case PlayerInteractionState.Interacting:
                _interactionType = interactionType;
                // 尝试获取动作锁，失败时取消交互
                if (!TryAcquireActionLock())
                {
                    Debug.LogWarning($"[GrittyTakedowns] 无法获取动作锁，目标 {targetId}，交互类型 {interactionType}");
                    _currentState = PlayerInteractionState.Idle;
                    EventBus.Instance.Publish(new InteractionStateChangedEvent(oldState, PlayerInteractionState.Idle));
                    return;
                }
                // 启动交互动画
                PlayInteractionAnimation(interactionType, targetId);
                break;
        }

        EventBus.Instance.Publish(new InteractionStateChangedEvent(oldState, newState));
    }

    /// <summary>
    /// 尝试获取动作锁（带失败处理）
    /// </summary>
    /// <returns>true = 成功获取锁，false = 获取失败（被其他系统占用）</returns>
    private bool TryAcquireActionLock()
    {
        // 通过 ActionLockSystem 接管玩家控制
        // ActionLockType 定义于 shared-types.md §4.1
        bool success = ActionLockSystem.Instance.TryAcquireLock(ActionLockType.Interaction, 0.1f);
        _actionLockAcquired = success;
        return success;
    }

    private void AcquireActionLock()
    {
        // 内部使用，不检查返回值（调用前已通过 TryAcquireActionLock 验证）
        ActionLockSystem.Instance.AcquireLock(ActionLockType.Interaction);
        _actionLockAcquired = true;
    }

    private void ReleaseActionLock()
    {
        if (_actionLockAcquired)
        {
            ActionLockSystem.Instance.ReleaseLock(ActionLockType.Interaction);
            _actionLockAcquired = false;
        }
    }
}
```

### 3. ActionLockSystem（共享服务）

> **重要**：ActionLockSystem 统一定义在 [shared-types.md](./shared-types.md) 中。
> 此处仅说明 Gritty Takedowns 如何使用，不重复实现细节。
>
> **实例获取方式**：使用 `ActionLockSystem.Instance` 单例模式（与 shared-types.md §4.2 保持一致）。

```csharp
// 使用示例（单例模式）
public class GrittyTakedownsSystem
{
    private void StartInteraction()
    {
        // 请求交互锁定
        ActionLockSystem.Instance.AcquireLock("GrittyTakedowns", ActionLockType.Interaction, expectedDuration: 2f);
    }

    private void EndInteraction()
    {
        // 释放交互锁定
        ActionLockSystem.Instance.ReleaseLock("GrittyTakedowns");
    }
}
```
```

### 4. 捆绑时间计算

> **⚠️ WorldState ↔ HealthState 同步协议**
>
> NPC 状态涉及两套独立的状态机，详细同步规则见 [shared-types.md §6.1.1](./shared-types.md#61-health-system-types)。
>
> **⚠️ 关键约束**：
> - **NPCController 是 WorldState 的唯一数据主权方**。Gritty Takedowns 只能查询 WorldState，不得直接修改。
> - **HealthSystem 是 HealthState 的唯一数据主权方**。其他系统不得直接修改 HealthState。
> - 两系统间同步由 NPCController 协调，通过订阅 HealthSystem 的 `HealthStateChangedEvent`（ADR-0008 §6.5）实现。
>
> **Gritty Takedowns 可见性约束**：WorldState.UNCONSCIOUS 仅在 HealthState.DOWNED 时可由 Gritty Takedowns 捆绑。
>
> **与 EnvironmentKill 的差异**：
> - **捆绑**要求 HealthState.DOWNED（确保 NPC 处于可安全接近的倒地状态）
> - **环境处决**（见 §1 CanEnvironmentKill）仅检查 WorldState.UNCONSCIOUS，不强制要求 HealthState.DOWNED
> - 这是因为处决是终结行为，只需 NPC 失去抵抗能力即可执行

```csharp
// Foundation/Shared/TieUpCalculator.cs
// NPCSizeCategory 定义在 Core/NPCAI/NPCSizeCategory.cs（由 shared-types.md §5.6 索引）
// Gritty Takedowns 通过 NPCController.QuerySizeCategory() 获取
// QuerySizeCategory() 方法定义在 ADR-0004 NPCController.cs 的 Query 接口部分

public class TieUpCalculator
{
    // 调参引用（必须提供）
    private readonly GrittyTakedownsTuningSO _tuning;

    public TieUpCalculator(GrittyTakedownsTuningSO tuning)
    {
        _tuning = tuning ?? throw new System.ArgumentNullException(nameof(tuning));
    }

    /// <summary>
    /// 计算捆绑时间
    /// </summary>
    /// <param name="npcSize">NPC体型分类</param>
    /// <param name="playerSkillBonus">玩家技能加成 [0, 1]</param>
    /// <returns>捆绑所需时间（秒）</returns>
    public float CalculateTieUpDuration(NPCSizeCategory npcSize, float playerSkillBonus)
    {
        float baseTime = _tuning.baseTieUpTime;
        float minDuration = _tuning.minTieUpDuration;
        float maxDuration = _tuning.maxTieUpDuration;

        // 体型系数：确保各体型覆盖不同的难度区间
        // Small:   [minDuration, minDuration + 1.0s] - 基础难度
        // Medium:  [minDuration + 0.5s, minDuration + 1.5s] - 中等难度
        // Large:   [minDuration + 1.0s, maxDuration] - 高难度
        float npcSizeMultiplier = npcSize switch
        {
            NPCSizeCategory.Small => 0.0f,
            NPCSizeCategory.Medium => 0.25f,
            NPCSizeCategory.Large => 1.0f,
            _ => 0.0f
        };

        // playerSkillBonus 范围 [0, 1]，负值或大于1时 clamp 到 [0, 1]
        playerSkillBonus = Mathf.Clamp01(playerSkillBonus);

        float duration = baseTime * (1.0f + npcSizeMultiplier - playerSkillBonus);
        return Mathf.Clamp(duration, minDuration, maxDuration);
    }
}

/// <summary>
/// 捆绑时间计算结果
/// </summary>
public struct TieUpDurationResult
{
    public float minDuration;  // 最小时间（最高技能）
    public float maxDuration;  // 最大时间（零技能）
    public float npcSizeBonus; // 体型加成（秒）
}

/// <summary>
/// 获取捆绑时间范围信息（供 UI 显示预估时间）
/// </summary>
public class TieUpDurationProvider
{
    private readonly GrittyTakedownsTuningSO _tuning;

    public TieUpDurationProvider(GrittyTakedownsTuningSO tuning)
    {
        _tuning = tuning ?? throw new System.ArgumentNullException(nameof(tuning));
    }

    public TieUpDurationResult GetTieUpDurationRange(NPCSizeCategory npcSize)
    {
        float baseTime = _tuning.baseTieUpTime;
        float minDuration = _tuning.minTieUpDuration;
        float maxDuration = _tuning.maxTieUpDuration;

        float npcSizeMultiplier = npcSize switch
        {
            NPCSizeCategory.Small => 0.0f,
            NPCSizeCategory.Medium => 0.25f,
            NPCSizeCategory.Large => 1.0f,
            _ => 0.0f
        };

        return new TieUpDurationResult
        {
            minDuration = Mathf.Clamp(baseTime * (1.0f + npcSizeMultiplier - 1.0f), minDuration, maxDuration),
            maxDuration = Mathf.Clamp(baseTime * (1.0f + npcSizeMultiplier - 0.0f), minDuration, maxDuration),
            npcSizeBonus = baseTime * npcSizeMultiplier
        };
    }
}
```

/// <summary>
/// 捆绑取消检测器
/// 在捆绑过程中持续检查玩家是否满足继续捆绑的条件
/// 参数由 GrittyTakedownsTuningSO 提供（见 §9）
/// </summary>
/// <remarks>
/// <b>数据流说明</b>：
/// GrittyTakedownsSystem 在捆绑过程中每帧调用 <see cref="ShouldCancel"/>。
/// 调用时需传入：
/// - <c>playerPosition</c>：通过 <c>PlayerController.Instance.transform.position</c> 获取
/// - <c>npcPosition</c>：通过 <c>NPCController.transform.position</c> 获取
/// - <c>playerFacing</c>：通过 <c>PlayerController.Instance.transform.forward</c> 获取
/// - <c>playerToNpc</c>：通过 <c>(npcPosition - playerPosition).normalized</c> 计算
/// </remarks>
public class TieUpCancelChecker
{
    private readonly float _maxCancelDistance;  // 超过此距离取消捆绑
    private readonly float _minCancelAngle;     // 视角偏离角度（度），超过此角度取消

    /// <summary>
    /// 使用指定参数构造
    /// </summary>
    public TieUpCancelChecker(float maxCancelDistance, float minCancelAngle)
    {
        _maxCancelDistance = maxCancelDistance;
        _minCancelAngle = minCancelAngle;
    }

    /// <summary>
    /// 使用 TuningSO 中的参数构造（推荐）
    /// </summary>
    /// <param name="tuning">调参配置（必须提供）</param>
    public TieUpCancelChecker(GrittyTakedownsTuningSO tuning)
    {
        if (tuning == null) throw new System.ArgumentNullException(nameof(tuning));
        _maxCancelDistance = tuning.maxTieUpCancelDistance;
        _minCancelAngle = tuning.minTieUpCancelAngle;
    }

    /// <summary>
    /// 检查是否应该取消捆绑
    /// </summary>
    /// <param name="playerPosition">玩家当前位置</param>
    /// <param name="npcPosition">NPC 位置</param>
    /// <param name="playerFacing">玩家面朝方向</param>
    /// <param name="playerToNpc">从玩家到 NPC 的方向向量</param>
    /// <returns>true = 取消捆绑，false = 继续捆绑</returns>
    public bool ShouldCancel(
        Vector3 playerPosition,
        Vector3 npcPosition,
        Vector3 playerFacing,
        Vector3 playerToNpc)
    {
        // 距离检测：使用水平距离（仅 XZ 平面），避免垂直方向影响判定
        float dx = npcPosition.x - playerPosition.x;
        float dz = npcPosition.z - playerPosition.z;
        float horizontalDistance = Mathf.Sqrt(dx * dx + dz * dz);

        if (horizontalDistance > _maxCancelDistance)
        {
            Debug.Log($"[TieUp] 取消：水平距离 {horizontalDistance:F2}m > {_maxCancelDistance}m");
            return true;
        }

        // 角度检测：视角偏离（仅计算水平方向，忽略垂直分量）
        // 提取水平方向向量
        Vector3 playerFacingFlat = new Vector3(playerFacing.x, 0, playerFacing.z).normalized;
        Vector3 playerToNpcFlat = new Vector3(playerToNpc.x, 0, playerToNpc.z).normalized;
        float angle = Vector3.Angle(playerFacingFlat, playerToNpcFlat);
        if (angle > _minCancelAngle)
        {
            Debug.Log($"[TieUp] 取消：视角偏离 {angle:F1}° > {_minCancelAngle}°");
            return true;
        }

        return false;
    }

    /// <summary>
    /// 检查目标 NPC 是否仍然处于可捆绑状态
    /// </summary>
    public bool IsNpcTieable(WorldState npcState)
    {
        // 只能在 NPC 处于 UNCONSCIOUS 状态时捆绑
        return npcState == WorldState.UNCONSCIOUS;
    }
}
```

### 5. 潜行击杀警报判定

```csharp
// StealthKillAlertChecker.cs
public bool ShouldBlockAlert(AlertState targetAlertState)
{
    // 只有当目标处于 UNDETECTED/SUSPECT/SEARCH 时，潜行击杀才不触发警报
    // ALERT/ESCAPE/COMBAT 状态下击杀会触发警报
    // AlertState 定义见 shared-types.md §5.3
    return targetAlertState switch
    {
        AlertState.UNDETECTED => true,
        AlertState.SUSPECT => true,
        AlertState.SEARCH => true,
        _ => false
    };
}
```

### 6. 欺骗成功判定

```csharp
// DeceptionChecker.cs
// NPC欺骗抗性来源：NPCData.deceptionResistance（0-1，越高越难欺骗）
// 由调用方（InteractionMatrix.CanDeceive）从 NPCController.QueryDeceptionResistance() 获取
// 注意：使用统一的 RandomService 以支持 seed 控制和确定性测试

/// <summary>
/// 欺骗成功判定结果
/// </summary>
public struct DeceptionResult
{
    public bool Success { get; init; }
    public float SuccessChance { get; init; }  // 本次判定的成功率（用于调试/日志）
    public float Roll { get; init; }           // 本次判定的随机数（用于调试/日志）
}

public DeceptionResult CheckDeceptionSuccess(
    bool hasVulnerability,
    float npcDeceptionResistance,
    IRandomService randomService)  // 注入随机数服务（非 UnityEngine.Random）
{
    // 欺骗成功条件：玩家已获取 vulnerability 且随机数小于 (1 - npcDeceptionResistance)
    //
    // **欺骗免疫判定**：当 npcDeceptionResistance >= 1.0 时，SuccessChance = 0，完全免疫欺骗
    if (!hasVulnerability)
        return new DeceptionResult { Success = false, SuccessChance = 0f, Roll = 0f };

    float successChance = 1.0f - npcDeceptionResistance;  // 抗性 >= 1.0 时 successChance = 0，完全免疫
    float roll = randomService.Value;
    bool success = roll < successChance;

    return new DeceptionResult { Success = success, SuccessChance = successChance, Roll = roll };
}

/// <summary>
/// 统一随机数服务接口（供外部注入）
/// </summary>
public interface IRandomService
{
    float Value { get; }  // 返回 [0, 1) 范围的随机浮点数
}

/// <summary>
/// 默认随机数实现（使用 UnityEngine.Random）
/// 可在测试时替换为确定性实现
/// </summary>
public class UnityRandomService : IRandomService
{
    public float Value => UnityEngine.Random.value;
}
```

### 7. 核心事件接口定义

```csharp
// ⚠️ 以下类型统一定义在 shared-types.md 中，此处仅列出 GrittyTakedowns 专用类型：

// ===== 共享类型（定义见 shared-types.md）=====
// - DamageRequest          → shared-types.md §2.3
// - ExplosionEvent         → shared-types.md §2.4
// - WeaponQueryRequest / WeaponQueryResponse → shared-types.md §3.8
// - PlayerDamagedEvent     → shared-types.md §7.6
// - AlertStateChangedEvent → shared-types.md §5.4
// - NPCStateChangedEvent   → shared-types.md §5.2
// - WorldState             → shared-types.md §5.1
// - AlertState             → shared-types.md §5.3
// - NPCIdentityType        → shared-types.md §5.5
// - EntityType             → shared-types.md §8.1
// - DamageType             → shared-types.md §2.1
// - HitLocation            → shared-types.md §2.2
// - InteractionType        → shared-types.md §9.1
// - InteractionResult      → shared-types.md §9.2

// ===== GrittyTakedowns 专用事件 =====

// 交互事件（通用）
public struct InteractionEvent
{
    public InteractionType type;       // 定义见 shared-types.md §9.1
    public int target_id;
    public string source;              // weapon_id 或 "bare_hands"
    public InteractionResult result;    // 定义见 shared-types.md §9.2
}

// 击杀标签（发送到 Sanity 系统）
public struct KillTagEvent
{
    public int npc_id;
    public NPCIdentityType kill_tag;  // ENEMY(恶徒)/ACCOMPLICE(帮凶)/VICTIM(受害者)
    // 定义见 shared-types.md §5.5
    // Sanity 系统根据 kill_tag 计算不同的疯狂值增长
    // 击杀恶徒：低疯狂值增长
    // 击杀帮凶：中等疯狂值增长
    // 击杀受害者：高疯狂值增长（强烈不推荐）
}

// 线索获取（发送到 Clue 系统）
public struct KnowledgeGainedEvent
{
    public int npc_id;
    public List<string> knowledge_list;
}

// 处决被目击（发送到 NPC AI 系统）
public struct ExecutionWitnessedEvent
{
    public int victim_id;
    public int witness_id;
}

// 玩家交互状态变化（内部事件）
public struct InteractionStateChangedEvent
{
    public PlayerInteractionState old_state;  // 定义见 shared-types.md §9.4
    public PlayerInteractionState new_state;
}

// 对话选项选择（发送到 NPC AI 系统处理）
public struct DialogueChoice
{
    public string dialogue_id;
    public string choiceId;  // 统一使用 PascalCase（符合 C# 命名规范）
}

// NPC AI 系统返回的对话结果
public struct DialogueResult
{
    public string dialogue_id;
    public bool success;
    public int allegiance_change;
    public List<string> knowledge_gained;
}
```

### 8. DialogueTree 接口边界

> **所有权声明**：
> - NPC AI System（Core Layer）：负责 DialogueTree 的**数据定义、存储和逻辑处理**
> - GrittyTakedowns（Feature Layer）：通过 Query 接口获取数据，**仅负责 UI 渲染和玩家输入**
>
> **接口边界定义**：
> | 职责 | 所属系统 |
> |------|---------|
> | DialogueTreeConfig 数据结构定义 | NPC AI System |
> | 对话选项文本（choice_text、response_text） | NPC AI System |
> | allegiance_change 计算 | NPC AI System |
> | knowledge_gained 列表 | NPC AI System |
> | DialogueUIManager UI 渲染 | GrittyTakedowns |
> | 玩家输入处理（选择选项） | GrittyTakedowns |
> | DialogueChoice 事件发送 | GrittyTakedowns |
>
> **接口约束**：GrittyTakedowns 只能通过 **Query 接口**（NPCController.QueryDialogueTree）获取只读数据，
> 不得直接修改 DialogueTreeConfig 的任何字段。NPC AI System 返回的 DialogueTreeConfig 应为只读拷贝或接口实现。

```csharp
/// <summary>
/// 对话树配置（由 NPC AI System 定义和维护）
/// </summary>
public class DialogueTreeConfig
{
    public string dialogue_id;
    public string npc_name;
    public List<DialogueBranch> branches;  // 对话选项分支
}

/// <summary>
/// 对话分支
/// </summary>
public class DialogueBranch
{
    public string choiceId;              // 统一使用 PascalCase（符合 C# 命名规范）
    public string choice_text;           // 显示给玩家的选项文本
    public string response_text;         // NPC 的回应文本
    public int allegiance_change;         // 选择后的派系态度变化
    public List<string> knowledge_gained; // 选择后获取的线索 ID 列表
}

/// <summary>
/// 对话 UI 管理器
/// </summary>
public class DialogueUIManager
{
    private DialogueTreeConfig _currentDialogue;  // 由 NPC AI 通过事件注入

    // 显示对话选项（由 NPC AI 系统提供文本内容）
    // 调用 NPCController.QueryDialogueTree() 获取对话树数据
    public void ShowDialogue(DialogueTreeConfig dialogueTree)
    {
        _currentDialogue = dialogueTree;
        // 使用 NPC AI 系统提供的 Branch 数据渲染 UI
    }

    // 玩家选择后，发送选中选项到 NPC AI 系统处理
    public void OnChoiceSelected(string choiceId)
    {
        EventBus.Instance.Publish(new DialogueChoice
        {
            dialogue_id = _currentDialogue.dialogue_id,
            choiceId = choiceId
        });
    }

    // 接收 NPC AI 系统返回的结果
    public void OnDialogueResult(DialogueResult result)
    {
        // 显示结果（allegiance 变化、获取信息等）
    }
}
```

### 9. 调参配置（GrittyTakedownsTuningSO）

```csharp
// Config/GrittyTakedownsTuningSO.cs
// StealthKillAlertMode 定义于 shared-types.md §15.1
using StealthKillAlertMode = SharedTypes.StealthKillAlertMode;

[CreateAssetMenu(menuName = "Game/GrittyTakedowns/Tuning")]
public class GrittyTakedownsTuningSO : ScriptableObject
{
    /// <summary>
    /// 设计说明：EnvironmentKill 与 TieUp 的状态要求差异
    ///
    /// - EnvironmentKill（环境处决）：仅检查 WorldState.UNCONSCIOUS
    ///   处决是终结行为，只需 NPC 失去抵抗能力即可执行
    ///
    /// - TieUp（捆绑）：必须同时检查 WorldState.UNCONSCIOUS + HealthState.DOWNED
    ///   捆绑需要 NPC 处于可安全接近的倒地状态
    ///
    /// 此差异反映了两种交互的不同风险等级：
    /// - 处决可在 NPC 仍处于 FREE 状态时发起（背刺），也可在击倒后补刀
    /// - 捆绑必须在 NPC 完全倒地后才可进行，以确保玩家安全
    /// </summary>

    [Header("目标切换冷却")]
    [Tooltip("快速切换目标时的冷却时间（秒），防止玩家过快切换导致 NPC AI 无法反应")]
    public float targetSwitchCooldown = 0.5f;

    [Header("捆绑时间参数")]
    [Tooltip("基础捆绑时间（秒），体型 Small 的 NPC 所需最短时间")]
    public float baseTieUpTime = 4.0f;

    [Tooltip("最小捆绑时间（秒），最高技能 + Small NPC")]
    public float minTieUpDuration = 3.0f;

    [Tooltip("最大捆绑时间（秒），零技能 + Large NPC")]
    public float maxTieUpDuration = 6.0f;

    [Header("捆绑取消阈值")]
    [Tooltip("捆绑过程中，玩家与 NPC 之间的最大允许距离（米）")]
    public float maxTieUpCancelDistance = 3.0f;

    [Tooltip("捆绑过程中，视角偏离角度阈值（度），超过此角度取消捆绑")]
    [Tooltip("仅计算水平方向视角偏离，垂直方向暂不纳入判定")]
    public float minTieUpCancelAngle = 120f;

    [Header("潜行击杀")]
    [Tooltip("潜行击杀时打断动画播放的最小进度阈值（0-1），低于此值处决不生效")]
    public float stealthKillInterruptThreshold = 0.5f;

    [Tooltip("潜行击杀警戒状态检查模式（默认 Tight）：")]
    [Tooltip("Tight（推荐）：仅 UNDETECTED 状态可执行潜行击杀，确保游戏挑战性")]
    [Tooltip("Loose：UNDETECTED / SUSPECT / SEARCH 均可执行，允许玩家失误后补救")]
    [Tooltip("")]
    [Tooltip("切换条件（Playtest 评估）：")]
    [Tooltip("如果平均每场战斗死亡次数 > 3 次，或潜行击杀成功率 < 20%，考虑切换到 Loose 模式")]
    public StealthKillAlertMode stealthKillAlertMode = StealthKillAlertMode.Tight;

    [Header("审问")]
    [Tooltip("审问冷却时间（秒），NPC 被审问后的沉默时长")]
    public float interrogateCooldown = 30f;

    [Header("贿赂")]
    [Tooltip("贿赂可用性阈值：NPC Bravery <= 此值时可以被贿赂（Bravery 范围 [1, 10]，值越小越容易被贿赂）")]
    public int bribeBraveryThreshold = 3;

    [Header("欺骗")]
    [Tooltip("欺骗免疫阈值：NPC DeceptionResistance >= 此值时完全免疫欺骗（范围 [0, 1]，默认 1.0f 完全免疫）")]
    [Range(0f, 1f)]
    public float deceptionResistanceImmuneThreshold = 1.0f;
}
```
```

### 10. Unity 项目结构（Feature Layer + Foundation 共享）

```
Assets/Game/
├── Features/GrittyTakedowns/           # Feature Layer（遵循 ADR-0003）
│   ├── GrittyTakedownsSystem.cs         # 主系统管理器
│   ├── InteractionMatrix.cs             # 交互类型矩阵
│   ├── PlayerInteractionFSM.cs           # 玩家交互状态机
│   ├── InteractionType.cs               # 交互类型枚举
│   ├── InteractionState.cs              # 玩家交互状态枚举
│   ├── Handlers/
│   │   ├── StealthKillHandler.cs        # 潜行击杀处理
│   │   ├── EnvironmentKillHandler.cs    # 环境处决处理
│   │   ├── TieUpHandler.cs              # 捆绑处理
│   │   ├── InterrogateHandler.cs        # 审问处理
│   │   ├── ConvertHandler.cs            # 转化处理
│   │   ├── IntimidateHandler.cs         # 威胁处理
│   │   ├── BribeHandler.cs              # 贿赂处理
│   │   └── DeceiveHandler.cs            # 欺骗处理
│   ├── UI/
│   │   ├── InteractionPromptUI.cs        # 交互提示 UI
│   │   ├── RadialMenuUI.cs              # 径向菜单 UI
│   │   └── DialogueUIManager.cs         # 对话 UI 管理
│   ├── Events/
│   │   ├── GrittyTakedownsEvents.cs     # 所有交互事件定义
│   │   └── GrittyTakedownsEventIds.cs   # 事件 ID 常量
│   └── Config/
│       └── GrittyTakedownsTuningSO.cs   # 调参配置
│
├── Foundation/Shared/                   # Foundation Layer 共享服务（ADR-0003）
│   ├── ActionLockSystem.cs             # 动作锁定系统（跨层共享）
│   ├── TieUpCalculator.cs              # 捆绑时间计算器
│   └── NPCConstants.cs                 # NPC 常量（与 ADR-0004 共享）
```

---

## Alternatives Considered

### Alternative 1: Gritty Takedowns 直接访问 NPC AI 状态

- **描述**：Gritty Takedowns 直接查询和修改 NPC AI 内部状态，而非通过 Query 接口
- **Pros**：实现简单，减少事件传递层级
- **Cons**：
  - 违反关注点分离原则
  - 循环依赖风险（NPC AI 需要知道交互结果，交互需要知道 NPC 状态）
- **拒绝理由**：
  - ADR-0004 NPC AI 决策明确使用事件驱动解耦
  - 保持系统边界清晰

### Alternative 2: 交互选项由 NPC AI 系统计算

- **描述**：NPC AI 系统根据自身状态计算出可用交互列表，Gritty Takedowns 只负责执行
- **Pros**：NPC AI 状态变化时自动更新交互选项
- **Cons**：
  - 交互选项涉及玩家持有物（武器、线索），这不属于 NPC AI 系统职责
  - 会导致 Gritty Takedowns 和 NPC AI 职责边界模糊
- **拒绝理由**：
  - 交互选项需要综合考虑 NPC 状态、玩家持有物、玩家已获取线索等多维度信息
  - Gritty Takedowns 作为玩家交互系统，是这些信息的汇聚点

### Alternative 3: Gritty Takedowns 自行管理 WeaponData

- **描述**：Gritty Takedowns 维护一份 WeaponData 副本，不通过 Weapon System 查询
- **Pros**：减少跨系统调用
- **Cons**：
  - 违反单一数据源原则
  - 环境物件武器化逻辑重复
- **拒绝理由**：
  - Weapon System 已经是 WeaponData 的唯一管理者
  - 通过 WeaponQuery 接口解耦，保持清晰职责

---

## Consequences

### Positive

- **清晰的职责边界**：Gritty Takedowns 只负责"何时交互"和"交互效果"，不涉及武器数据和 NPC 内部状态
- **事件驱动解耦**：通过 Event Bus 与 NPC AI、Weapon、Health、Sanity、Clue 系统通信，避免循环依赖
- **交互选项动态化**：InteractionMatrix 根据 NPC 状态、身份标签、玩家持有物实时计算可用选项
- **动作锁定安全**：ActionLockSystem 作为共享服务，确保交互期间玩家控制权被正确接管和释放
- **DialogueTree 所有权清晰**：NPC AI 系统负责数据，Gritty Takedowns 负责 UI 渲染

### Negative

- **交互选项计算复杂性**：需要综合多个数据源，边缘情况多
- **ActionLock 优先级管理**：多个锁类型共存时需要明确的优先级规则
- **对话 UI 与 NPC AI 的协调**：需要建立良好的接口契约

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **交互被打断** | 玩家在动画中被打断，导致状态不一致 | 边缘情况处理：检查动画进度决定是否完成处决（阈值由 TuningSO 配置） |
| **多目标切换** | 快速切换目标导致 AI 无法反应 | `targetSwitchCooldown` 冷却机制（由 GrittyTakedownsTuningSO 配置，默认 0.5秒） |
| **捆绑超时** | 捆绑时间过长导致玩家不耐烦 | 最短 3 秒，最长 6 秒的安全范围（TieUpCalculator 硬编码保证） |
| **审问被唤醒** | 审问期间 NPC 醒来 | 审问期间暂停唤醒计时器 |
| **WorldState/HealthState 状态映射** | NPC 被击倒时 WorldState 与 HealthState 同步失败 | 已在 §4 明确同步协议，由 NPCController 负责协调 |
| **审问冷却参数泄露** | 审问 cooldown 持续时间未公开导致玩家困惑 | UI 提示显示冷却状态，`interrogateCooldown` 由 TuningSO 配置 |
| **转化派系冲突** | LOYAL 派系的 NPC 被转化可能与游戏 lore 冲突 | CanConvert 已添加 LOYAL 派系检查，转化选项对 LOYAL NPC 不可用（v1.4.2 修复） |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 2ms/帧 | 交互状态机简单，动画锁定期间无额外计算 |
| **Memory** | < 5MB | InteractionMatrix 数据结构 |
| **Network** | 无影响 | 单机游戏 |
| **交互响应** | < 1 帧 | 交互选项显示即时 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] **InteractionType、WorldState、AlertState、NPCIdentityType 等枚举已在 shared-types.md 中定义，直接引用**
- [ ] **PlayerInteractionState 枚举在 shared-types.md §9.4 中定义，直接引用**
- [ ] 创建 ActionLockSystem（Foundation/Shared/）
- [ ] 创建 PlayerInteractionFSM
- [ ] 定义所有 GrittyTakedowns 专用事件类型

### Phase 2: 交互矩阵
- [ ] 实现 InteractionMatrix
- [ ] 实现所有交互类型的可用性判定
- [ ] 与 LOS System 集成（身份标签查询）
- [ ] 与 Environment Interaction 集成（可用物件查询）

### Phase 3: 交互处理器
- [ ] 实现 StealthKillHandler
- [ ] 实现 EnvironmentKillHandler
- [ ] 实现 TieUpHandler
- [ ] 实现 InterrogateHandler
- [ ] 实现 ConvertHandler
- [ ] 实现 IntimidateHandler
- [ ] 实现 BribeHandler
- [ ] 实现 DeceiveHandler

### Phase 4: Weapon System 集成
- [ ] 实现 WeaponQueryRequest/WeaponQueryResponse 接口
- [ ] 验证环境处决能正确获取 WeaponData

### Phase 5: NPC AI System 集成
- [ ] 实现事件订阅（AlertStateChangedEvent、NPCStateChangedEvent）
- [ ] 实现 Query 接口调用
- [ ] 实现 DialogueTree UI 渲染

### Phase 6: Health System 集成
- [ ] 实现 DamageRequest 发送
- [ ] 订阅 PlayerDamagedEvent（与 ADR-0008 对齐签名）
- [ ] 验证处决伤害被 Health 系统正确处理

### Phase 7: UI 开发
- [ ] 实现 InteractionPromptUI
- [ ] 实现 RadialMenuUI
- [ ] 实现 DialogueUIManager

### Phase 8: 完整流程测试
- [ ] 验证交互状态机所有路径
- [ ] 验证边缘情况处理
- [ ] 验证跨系统事件流

---

## Validation Criteria

### 功能验收
1. **潜行击杀条件判定**：
   - **默认 Tight 模式**：仅 UNDETECTED 状态可执行（确保游戏挑战性）
   - Loose 模式：UNDETECTED / SUSPECT / SEARCH 均可执行
2. **⚠️ Playtest Required（ Loose 模式调优）**：
   - 当前默认 Tight 模式（仅 UNDETECTED 可执行）
   - **Playtest 评估**：如果 Tight 模式过于困难导致游戏体验下降，可切换到 Loose 模式
   - Loose 模式下 SUSPECT/SEARCH 状态均可执行潜行击杀，允许玩家"失误后补救"
   - 调优参数：`GrittyTakedownsTuningSO.stealthKillAlertMode`
3. **环境处决流程**：持有物件 → 查询 WeaponData → 发送 DamageRequest → Health 处理
4. **捆绑时间正确**：NPC 体型和玩家技能正确影响捆绑时间（3-6 秒范围）
5. **审问线索获取**：审问成功后 KnowledgeGainedEvent 正确发送到 Clue 系统
6. **欺骗选项显示**：仅在玩家已获取 vulnerability 时显示欺骗选项
7. **转化效果**：allegiance 提升 + 情报提供后恢复；LOYAL 派系 NPC 转化选项不可用
8. **贿赂功能完整**：Bravery 检查和 FactionAllegiance 检查同时生效（LOYAL 不可被贿赂）

### 边缘情况验收
9. **交互被打断**：玩家在动画中被攻击，根据 `stealthKillInterruptThreshold`（TuningSO）决定处决结果
10. **目标死亡**：CanInteract 状态中目标死亡，自动退出并显示提示
11. **处决被目击**：ExecutionWitnessedEvent 正确发送到 NPC AI 系统
12. **审问期间唤醒**：唤醒计时器在审问期间暂停
13. **动作锁获取失败**：当其他系统占用锁时，交互正确取消而非卡死
14. **⚠️ AlertState 边界检查**：Intimidate/Bribe/Deceive/Convert/MaintainDistance/Dialogue 在 NPC 进入 COMBAT 或 ESCAPE 状态时正确禁用
15. **Loose 模式时间窗口**：SUSPECT/SEARCH 状态下可执行潜行击杀，但必须在 NPC 升级警戒之前完成（紧迫补救窗口）
16. **捆绑距离检测**：使用水平距离（XZ 平面）判定，楼梯等垂直场景不误判

### 跨系统验收
16. **交互事件广播**：InteractionEvent 正确发送到事件总线
17. **伤害请求处理**：DamageRequest 被 Health 系统正确处理
18. **击杀标签传递**：KillTagEvent 正确发送到 Sanity 系统
19. **线索系统集成**：搜身/审问的 KnowledgeGainedEvent 被 Clue 系统接收
20. **PlayerDamagedEvent 对齐**：与 ADR-0008/shared-types.md §7.6 定义的签名完全一致
    ```
    public struct PlayerDamagedEvent
    {
        public int player_id;
        public DamageType damage_type;
        public HitLocation hit_location;
        public int source_entity_id;
    }
    ```

### shared-types.md 类型存在性验收
21. **shared-types.md 完整性**：验证以下类型已在 shared-types.md 中正确定义：
    - `DamageRequest`（§2.3）
    - `ExplosionEvent`（§2.4）
    - `PlayerDamagedEvent`（§7.5）
    - `WeaponQueryRequest` / `WeaponQueryResponse`（§3.8）
    - `AlertStateChangedEvent`（§5.4）
    - `NPCStateChangedEvent`（§5.2）
    - `WorldState`（§5.1）
    - `AlertState`（§5.3）
    - `NPCIdentityType`（§5.5）
    - `InteractionType`（§9.1）
    - `InteractionResult`（§9.2）
    - `PlayerInteractionState`（§9.4）
    - `ActionLockType`（§4.1）
    - `NPCSizeCategory`（§5.6）
    - `HealthState`（§6.1）
    - `FactionAllegiance`（§5.7，含 Bravery 和 DeceptionResistance 属性说明）

### NPCData 属性验收（ADR-0004 定义）
> **注意**：以下 NPCController Query 接口的具体定义见 [ADR-0004 §8](./adr-0004-npc-ai-behavior-architecture.md#8-query-接口)。

22. **NPCController.QueryBravery()**：返回 NPC 的勇气值（范围 [1, 10]，影响贿赂可用性）
23. **NPCController.QueryDeceptionResistance()**：返回 NPC 的欺骗抗性（范围 [0, 1]）
24. **NPCController.QuerySizeCategory()**：返回 NPC 的体型分类（Small/Medium/Large）
25. **NPCController.QueryDialogueTree()**：返回该 NPC 的对话树数据，供 DialogueUIManager 渲染对话选项
26. **NPCController.QueryFactionAllegiance()**：返回 NPC 的派系忠诚度（影响贿赂可用性，LOYAL 不可被贿赂）
27. **NPCController.QueryHealthState()**：返回 NPC 的 HealthState（用于 CanTieUp 的 HealthState.DOWNED 检查）

### 跨系统状态同步协议验收
28. **WorldState ↔ HealthState 同步**：验证 NPCController 在 HealthSystem 发送 DOWNED 事件后正确设置 WorldState = UNCONSCIOUS（详见 §4 同步协议）
29. **捆绑状态检查完整性**：CanTieUp 同时检查 WorldState.UNCONSCIOUS 和 HealthState.DOWNED（详见 CanTieUp 方法注释）

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — Gritty Takedowns 通过 Event Bus 与其他系统通信
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — **Gritty Takedowns 属于 Feature Layer**；**ActionLockSystem 属于 Foundation/Shared/**
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC AI 系统订阅 InteractionEvent，DialogueTree 数据归 NPC AI 所有
- [ADR-0007: LOS 系统架构](./adr-0007-los-system-architecture.md) — 身份标签由 LOS 系统维护并提供查询接口
- [ADR-0008: 脆弱度与伤害系统](./adr-0008-health-lethality-architecture.md) — Health 系统接收 DamageRequest；**PlayerDamagedEvent 定义来源**
- [ADR-0010: 武器系统](./adr-0010-weapon-system-architecture.md) — Weapon System 提供 WeaponQuery 接口
- [共享类型定义](./shared-types.md) — **DamageRequest、WeaponQuery、ActionLockSystem、PlayerDamagedEvent 等跨 ADR 类型统一定义在此**
- [Gritty Takedowns GDD](../../design/gdd/gritty-takedowns.md) — 本 ADR 的设计依据
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 事件定义的权威文档

## 附录：v1.4.x 修改日志

| 日期 | 版本 | 修改内容 | 评审修复 |
|------|------|---------|----------|
| 2026-04-11 | v1.4.2 | StealthKillAlertMode 补充切换条件说明（Playtest 评估指标） | - |
| 2026-04-11 | v1.4.2 | TuningSO 补充 EnvironmentKill/TieUp 状态要求差异注释 | - |
| 2026-04-11 | v1.4.2 | TieUpCancelChecker 距离检测改为水平距离（XZ 平面） | - |
| 2026-04-11 | v1.4.2 | CanConvert 添加 LOYAL 派系不可转化检查 | - |
| 2026-04-11 | v1.4.1 | DECEPTION_RESISTANCE_IMMUNE 硬编码改为 TuningSO 配置化 | #6 修复 |
| 2026-04-11 | v1.4.1 | COMBAT/ESCAPE 检查逻辑提取为 IsNpcInteractable 方法 | #7 修复 |
| 2026-04-11 | v1.4.1 | PlayerInteractionFSM 状态转换触发条件补充说明 | #12 修复 |
| 2026-04-11 | v1.4.1 | Dialogue UI 边界明确为只读 Query 接口 | #10 修复 |
| 2026-04-11 | v1.4.1 | QueryAvailableInteractions 调用频率限制和缓存策略说明 | #11 修复 |
| 2026-04-11 | v1.4.1 | InteractionMatrix 添加内部缓存机制 | #11 修复 |
