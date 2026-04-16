# ADR-0024: 动画系统 (Animation System) 架构决策

## Status
**Proposed**

## Date
2026-04-12

## Last Updated
2026-04-14 (ADR 评审修复：AnimationEventBridge 改为发布 TakedownAnimationCompleteEvent，职责归位)

## Context

### Problem Statement

《断绝：罪恶之源》是一款俯视角潜行动作游戏，动画系统需要支撑：
1. **玩家动画**：移动、潜行、战斗、处决动画
2. **NPC 动画**：巡逻、警戒、追击、死亡、捆绑
3. **环境动画**：物件交互、爆炸、可破坏物
4. **过场动画**：剧情过场、UI 过渡

现有 Unity 项目缺少统一的动画架构规范，需要在 ADR-0003 分层框架下定义：
- 动画状态机如何与游戏状态同步
- 动画层如何管理（玩家/NPC/环境）
- 动画事件如何与事件总线集成

### Constraints

- **引擎约束**：Unity 6.3 LTS，使用 Unity Animation System（Animator + AnimationClip）
- **平台约束**：PC & PS5，支持 PS5 DualSense 自适应扳机
- **性能约束**：动画内存预算 ≤ 64MB，最大同时播放动画数 ≤ 20

### Requirements

- **必须**：定义 AnimationLayer 分类（Base/Action/Overlay）
- **必须**：定义 AnimationState 到游戏状态的映射规则
- **必须**：定义 AnimationEvent 到 EventBus 的转换机制
- **必须**：支持动画重定向（Humanoid）和 2D 骨骼动画
- **必须**：遵循 ADR-0003 系统分层（Animation System 属于 Presentation Layer）

---

## Decision

### 架构决策

采用**分层动画状态机 + 事件驱动动画通知**架构：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    动画系统架构分层图                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Animation Layer 2: Overlay Layer                           │   │
│  │  权重: 0.0-1.0 (Additive)                                   │   │
│  │  内容: 表情动画、武器挂载、披风/衣物物理                      │   │
│  │  控制: 动画层权重，无状态机                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Animation Layer 1: Action Layer                            │   │
│  │  权重: 0.0-1.0 (Additive)                                   │   │
│  │  内容: 处决动画、交互动画、技能动画                           │   │
│  │  控制: ActionStateMachine（由游戏系统触发）                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Animation Layer 0: Base Layer                               │   │
│  │  权重: 1.0 (Override)                                        │   │
│  │  内容: 移动、待机、潜行、死亡                                 │   │
│  │  控制: PlayerController/NPCController 驱动                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1. AnimationLayer 分类

```csharp
// AnimationLayer.cs
public enum AnimationLayer
{
    /// <summary>
    /// 基础层：移动、待机、潜行、死亡
    /// 由 PlayerController/NPCController 驱动
    /// </summary>
    Base = 0,

    /// <summary>
    /// 动作层：处决动画、交互动画、技能动画
    /// 由游戏系统通过 ActionStateMachine 触发
    /// 权重: Additive，通过权重与 Base 层混合（而非完全替代）
    ///
    /// **权重设计说明**：
    /// - Action 层使用 Additive 而非 Override，因为：
    ///   1. 动作动画通常只播放部分身体动作（如上半身处决动画 + 下半身行走）
    ///   2. 动作结束后需要平滑过渡回 Base 层，而非突然切换
    ///   3. Overlay 层需要叠加在 Base+Action 组合之上，Additive 是唯一正确选择
    /// - 运行时权重由 AnimationLayerWeightController 通过 FadeToWeight 控制
    /// - 默认状态：Base=1.0, Action=0.0, Overlay=0.0
    /// - **Animator Controller 配置**：Action 层在 Animator 中必须设置为 Additive 混合模式，
    ///   而非 Override。代码层设置 SetLayerWeight 仅控制权重，混合模式由 Animator Controller 决定。
    /// </summary>
    Action = 1,

    /// <summary>
    /// 覆盖层：表情动画、武器挂载、披风物理
    /// 权重: Additive，叠加在 Base+Action 组合之上
    /// </summary>
    Overlay = 2
}
```

> **初始化顺序说明**：`AnimationStateMachine` 现在是 MonoBehaviour，
> 通过 Unity 生命周期方法自动按正确顺序初始化，不再需要手动注入：
> ```csharp
> // 1. 确保 AnimationLayerWeightController 和 AnimationStateMachine 在同一 GameObject 上
> // 2. AnimationStateMachine.Awake() 会自动获取同 GameObject 上的 LayerWeightController
> // 3. 动画师/程序员只需要在 Inspector 中配置 SerializeField 引用
> ```
>
> **设计优势**：
> - Unity 生命周期保证初始化顺序（Awake → OnEnable → Start）
> - 无需手动调用多个 Set 方法
> - 避免"未初始化就调用"导致的静默错误
> - `Debug.LogWarning` 在运行时检测并警告未初始化的操作
>
> 注意：`AnimationLayerWeightController.IsInitialized` 可用于检查是否已完成初始化。

### 2. AnimationState 定义

```csharp
// AnimationState.cs
/// <summary>
/// 动画状态枚举（按层分组）
/// 注意：枚举值按层分组（Base: 0-31, Action: 32-95, Locomotion: 96-127），值域连续且紧凑
/// </summary>
public enum AnimationState
{
    // ========== Base Layer (0-31) ==========
    /// <summary>无动作（默认）</summary>
    None = 0,

    /// <summary>待机</summary>
    Idle = 1,

    /// <summary>行走</summary>
    Walk = 2,

    /// <summary>冲刺</summary>
    Sprint = 3,

    /// <summary>蹲伏</summary>
    Crouch = 4,

    /// <summary>蹲行</summary>
    CrouchWalk = 5,

    /// <summary>死亡</summary>
    Death = 6,

    // ========== Action Layer (32-95) ==========
    /// <summary>潜行击杀</summary>
    StealthKill = 32,

    /// <summary>环境击杀</summary>
    EnvironmentKill = 33,

    /// <summary>终结</summary>
    FinishOff = 34,

    /// <summary>捆绑</summary>
    TieUp = 35,

    /// <summary>审讯</summary>
    Interrogate = 36,

    /// <summary>威胁</summary>
    Threaten = 37,

    /// <summary>搜身</summary>
    Search = 38,

    // ========== Locomotion Blend (96-127) ==========
    /// <summary>
    /// 移动混合状态（BlendTree 标识符）
    ///
    /// **设计说明**：
    /// - 此枚举值用于 Animator Controller 内部的 BlendTree 状态标识
    /// - Base 层实际状态仍为 Walk/Sprint/CrouchWalk，
    ///   Animator Controller 根据 Speed 参数在 BlendTree 中选择具体动画
    /// - AnimationStateMachine 使用 Locomotion = 96 作为 BlendTree 状态的标识值
    /// - **不建议在代码中切换到此状态**：移动应使用 Walk/Sprint/CrouchWalk 等离散状态
    ///
    /// **与 Base Layer 动画的配合**：
    /// 当 PlayerController 设置 BaseState = Locomotion 时，Animator Controller 中的
    /// Locomotion 状态被激活，BlendTree 根据 Speed 参数在 Walk/Sprint/CrouchWalk 之间混合。
    /// 这种设计允许动画师在 Animator 中精细控制移动动画的过渡和混合。
    /// </summary>
    [System.Obsolete("Locomotion 是 Animator 内部 BlendTree 标识符，不应在代码中切换。移动应使用 Walk/Sprint/CrouchWalk 等离散状态。")]
    Locomotion = 96,

    /// <summary>
    /// 战斗姿态移动混合状态（BlendTree 标识符）
    ///
    /// **设计说明**：
    /// - 与 Locomotion 类似，用于战斗姿态下的移动混合
    /// - 由 PlayerController 切换到 Combat 模式时启用
    /// - **不建议在代码中切换到此状态**：移动应使用 Walk/Sprint/CrouchWalk 等离散状态，
    ///   战斗姿态的移动混合由 Animator Controller 的 BlendTree 自动处理
    /// </summary>
    [System.Obsolete("CombatLocomotion 是 Animator 内部 BlendTree 标识符，不应在代码中切换。移动应使用 Walk/Sprint/CrouchWalk 等离散状态。")]
    CombatLocomotion = 97,
}

// NPCAnimationState.cs
/// <summary>
/// NPC 动画状态枚举（独立于玩家 AnimationState，避免跨职责混用）
/// NPC 使用独立的 Animator Controller，NPCAnimationStateMachine
/// 订阅 NPCStateChangedEvent，将 NPC AI 状态映射到此枚举后
/// 调用 SetBaseState() 更新对应 NPC 的 Animator 参数。
/// </summary>
public enum NPCAnimationState
{
    /// <summary>NPC 待机</summary>
    Idle = 0,

    /// <summary>NPC 巡逻（沿路径行走）</summary>
    Patrol = 1,

    /// <summary>NPC 警戒（发现异样，原地警惕）</summary>
    Alert = 2,

    /// <summary>NPC 搜索（进入搜寻状态，缓慢移动）</summary>
    Search = 3,

    /// <summary>NPC 追击（全速追赶目标）</summary>
    Chase = 4,

    /// <summary>NPC 战斗姿态（已锁定目标，近战/射击准备）</summary>
    Combat = 5,

    /// <summary>NPC 死亡</summary>
    Death = 6,

    /// <summary>NPC 被捆绑（倒地或坐地）</summary>
    Bound = 7,

    /// <summary>NPC 被击倒（短暂晕眩）</summary>
    Stunned = 8
}

// AnimationStateMachine.cs
using System;
using UnityEngine;

/// <summary>
/// 动画状态机
/// 负责管理 Base/Action/Overlay 三层动画状态的切换
/// 此类是 MonoBehaviour，确保生命周期方法按正确顺序执行，
/// 避免 Plain C# class 注入模式带来的初始化顺序陷阱
/// </summary>
public class AnimationStateMachine : MonoBehaviour
{
    private AnimationState _baseState = AnimationState.None;
    private AnimationState _actionState = AnimationState.None;

    /// <summary>
    /// 动作执行中标志（用于防止竞态条件）
    /// 确保 StopCoroutine 和 NotifyActionCompleted 不会重复触发 OnActionComplete
    /// </summary>
    private bool _isActionRunning;

    private System.Collections.IEnumerator _pendingCoroutine;

    /// <summary>
    /// 动画层权重控制器引用
    /// 由 AnimationStateMachine 在 PlayAction/OnActionComplete_Core 中直接调用
    /// FadeToWeight()，无需中间事件。组件在同一 GameObject 上，
    /// 通过 Awake() 的 GetComponent 自动获取。
    /// </summary>
    [SerializeField] private AnimationLayerWeightController _layerWeightController;

    /// <summary>
    /// 初始化完成标志
    /// </summary>
    private bool _isInitialized;

    /// <summary>
    /// Unity 生命周期 Awake - 自动按正确顺序初始化
    /// </summary>
    private void Awake()
    {
        // 自动获取同 GameObject 上的 LayerWeightController
        if (_layerWeightController == null)
        {
            _layerWeightController = GetComponent<AnimationLayerWeightController>();
        }
        _isInitialized = _layerWeightController != null;
    }

    /// <summary>
    /// 检查初始化状态，未初始化时输出警告
    /// </summary>
    private bool ValidateInitialization()
    {
        if (_isInitialized) return true;

        Debug.LogWarning("[AnimationStateMachine] Operation rejected: StateMachine is not initialized. " +
            "Ensure AnimationLayerWeightController is attached to the same GameObject.");
        return false;
    }

    /// <summary>
    /// 动画层权重变化回调
    /// 当 Action 层开始/结束时调用，通知 _layerWeightController 更新层权重
    /// </summary>
    private void OnLayerWeightNeedsUpdate(AnimationLayer layer, float targetWeight, float transitionDuration)
    {
        _layerWeightController?.FadeToWeight(layer, targetWeight, transitionDuration);
    }

    /// <summary>
    /// 设置 Base 层状态（由 PlayerController/NPCController 调用）
    /// </summary>
    public void SetBaseState(AnimationState state)
    {
        if (!ValidateInitialization()) return;
        if (_actionState != AnimationState.None)
            return; // Action 层占用时不能切换 Base

        _baseState = state;
    }

    /// <summary>
    /// 播放动作层动画（由 GrittyTakedowns 等系统调用）
    /// Action 层播放完毕后自动回归 None，触发回调
    /// </summary>
    /// <param name="actionState">动作状态</param>
    /// <param name="fixedDuration">
    /// 固定时长（秒），-1 表示使用 AnimationEvent 触发。
    ///
    /// **模式说明**：
    /// - **fixedDuration >= 0**：固定时长模式，用于动画时间已知且稳定的处决动画。
    ///   超时后强制触发 OnActionComplete，作为安全网。
    /// - **fixedDuration < 0**：动画事件模式，等待 AnimationEventBridge.OnAnimationEvent("OnAnimationEnd") 触发。
    ///   这是主要推荐模式，因为与动画师制作的动画精确同步。
    ///
    /// **线程安全说明**：
    /// _isActionRunning 标志位确保 StopCoroutine 和 NotifyActionCompleted 不会重复触发 OnActionComplete。
    /// </param>
    public void PlayAction(AnimationState actionState, float fixedDuration = -1f)
    {
        if (!ValidateInitialization()) return;

        // 如果已有动作在执行，先中断
        if (_isActionRunning)
        {
            InterruptAction();
        }

        _actionState = actionState;
        _isActionRunning = true;

        // 淡入 Action 层权重（修复 P3：原方法定义但从未调用）
        OnLayerWeightNeedsUpdate(AnimationLayer.Action, 1f, 0.1f);

        if (fixedDuration >= 0)
        {
            // 固定时长模式（用于动画时间已知的处决动画）
            _pendingCoroutine = ActionCompleteCoroutine(fixedDuration);
            StartCoroutine(_pendingCoroutine);
        }
        // 否则等待 AnimationEvent 触发 OnActionComplete
    }

    /// <summary>
    /// 中断当前正在执行的动作
    /// 用于播放新动画时强制打断旧动画
    /// </summary>
    public void InterruptAction()
    {
        if (!_isActionRunning) return;

        // 停止待处理的协程
        if (_pendingCoroutine != null)
        {
            StopCoroutine(_pendingCoroutine);
            _pendingCoroutine = null;
        }

        // 触发完成回调（传递被打断的动作状态）
        var interruptedAction = _actionState;
        _isActionRunning = false;
        OnActionComplete_Core(interruptedAction);
    }

    private System.Collections.IEnumerator ActionCompleteCoroutine(float duration)
    {
        yield return new WaitForSeconds(duration);
        // 线程安全：检查 _isActionRunning 防止协程超时触发时已完成
        if (_isActionRunning)
        {
            OnActionComplete_Core(_actionState);
        }
    }

    /// <summary>
    /// 由外部（AnimationEventBridge）调用，通知动作完成
    /// </summary>
    public void NotifyActionCompleted()
    {
        // 线程安全：检查 _isActionRunning 防止重复触发
        if (!_isActionRunning) return;

        if (_pendingCoroutine != null)
        {
            StopCoroutine(_pendingCoroutine);
            _pendingCoroutine = null;
        }
        OnActionComplete_Core(_actionState);
    }

    /// <summary>
    /// 动作结束事件
    /// 回调参数为刚结束的动作状态（previousAction），而非 None
    ///
    /// **⚠️ 重要语义说明 ⚠️**：
    /// - 回调在 Action 层回归 None **之前**触发，传递的是刚完成的动作状态
    /// - 调用方收到此事件时，Action 层仍处于 previousAction 状态，
    ///   可以安全地读取动作信息（如处决动画的伤害值、持续时间等）
    /// - Action 层真正回归 None 是在回调执行完成后由 OnActionComplete_Core 内部处理
    /// - 如需在 Action 层完全回归 None 后收到通知，应订阅 OnActionStateChanged 事件（如果已实现）
    ///
    /// **使用示例**：
    /// ```csharp
    /// // 正确：在 OnActionEnded 中读取 previousAction 信息
    /// stateMachine.OnActionEnded += (previousAction) =>
    /// {
    ///     // 此时 Action 层仍为 previousAction，可以读取其关联数据
    ///     var damage = GetTakedownDamage(previousAction);
    /// };
    ///
    /// // 错误：在回调中认为 Action 层已回归 None
    /// stateMachine.OnActionEnded += (previousAction) =>
    /// {
    ///     // 此处 GetActionState() 返回的仍是 previousAction，而非 None
    ///     // 如需在 None 状态下执行操作，应使用独立的完成事件
    /// };
    /// ```
    /// </summary>
    public event Action<AnimationState> OnActionEnded;

    /// <summary>
    /// 动作完成核心逻辑（内部方法，供 InterruptAction/NotifyActionCompleted/ActionCompleteCoroutine 调用）
    /// </summary>
    /// <param name="previousAction">刚完成的动作状态（用于回调）</param>
    private void OnActionComplete_Core(AnimationState previousAction)
    {
        _isActionRunning = false;
        _actionState = AnimationState.None;
        _pendingCoroutine = null;

        // 淡出 Action 层权重，回归 Base 层（修复 P3：原方法定义但从未调用）
        OnLayerWeightNeedsUpdate(AnimationLayer.Action, 0f, 0.2f);

        if (previousAction != AnimationState.None)
        {
            // 传递刚完成的 action 状态，而非 None
            // 调用方可以在此处安全地读取 previousAction 的信息
            OnActionEnded?.Invoke(previousAction);
        }
    }

    public AnimationState GetBaseState() => _baseState;
    public AnimationState GetActionState() => _actionState;
}
```

### 2.5 AnimationState ↔ PlayerMovementState 映射

AnimationState（动画系统）与 PlayerMovementState（玩家控制器）需要保持同步：

```csharp
// AnimationStateMapper.cs
/// <summary>
/// AnimationState 与 PlayerMovementState 映射器
/// 负责将玩家移动状态转换为对应的动画状态
/// </summary>
public static class AnimationStateMapper
{
    /// <summary>
    /// 将 PlayerMovementState 映射为 AnimationState
    /// </summary>
    public static AnimationState ToAnimationState(PlayerMovementState movementState)
    {
        return movementState switch
        {
            PlayerMovementState.IDLE => AnimationState.Idle,
            PlayerMovementState.WALK => AnimationState.Walk,
            PlayerMovementState.SPRINT => AnimationState.Sprint,
            PlayerMovementState.CROUCH => AnimationState.Crouch,
            PlayerMovementState.CROUCH_WALK => AnimationState.CrouchWalk,
            PlayerMovementState.ACTION => AnimationState.None,  // ACTION 由动作系统接管
            PlayerMovementState.DEAD => AnimationState.Death,
            _ => AnimationState.Idle
        };
    }

    /// <summary>
    /// 检查是否为移动相关状态（用于 BlendTree 控制）
    /// </summary>
    public static bool IsMovementState(PlayerMovementState movementState)
    {
        return movementState is
            PlayerMovementState.WALK or
            PlayerMovementState.SPRINT or
            PlayerMovementState.CROUCH_WALK;
    }
}
```

> **同步机制**：AnimationParameterUpdater 订阅 PlayerMovementStateChangedEvent，
> 当状态变化时调用 AnimationStateMapper.ToAnimationState() 转换为动画状态，
> 并通过 SetBaseState() 同步到 Animator Controller。

```csharp
// AnimationEventBridge.cs
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// 动画事件桥接器：将 Animator AnimationEvent 转换为 EventBus 事件
/// 解决 AnimationEvent 无法直接发布到 EventBus 的问题
/// </summary>
public class AnimationEventBridge : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private AnimationStateMachine _stateMachine;

    // 事件映射表
    private Dictionary<string, Action> _eventMap = new();

    private void Start()
    {
        // 注册事件映射
        // 注意：InteractionEvent 已定义于 shared-types.md §9.3
        // 注意：需要 targetId 的事件通过 Lambda 闭包捕获参数，不使用字典预注册

        // 动画结束事件触发状态机回调
        _eventMap["OnAnimationEnd"] = () =>
        {
            _stateMachine?.NotifyActionCompleted();
        };
    }

    /// <summary>
    /// 由 AnimationEvent 触发，查找并执行对应的 EventBus 事件
    /// </summary>
    /// <param name="eventName">事件名称</param>
    /// <param name="entityId">实体 ID（用于交互类事件，如 StealthKill、EnvironmentKill）</param>
    /// <param name="weaponId">武器 ID（用于武器类事件，如 WeaponSwing）</param>
    public void OnAnimationEvent(string eventName, int entityId = 0, int weaponId = 0)
    {
        // 交互类事件：使用 entityId
        // 注意：OnStealthKillHit 只转发动画命中事件，不发布 InteractionEvent
        // InteractionEvent 由 GrittyTakedowns 在动画完成后发布
        if (eventName == "OnStealthKillHit")
        {
            EventBus.Instance.Publish(new TakedownAnimationCompleteEvent
            {
                EntityId = entityId,
                AnimationHash = 0, // 由调用方填充
                TakedownType = InteractionType.StealthKill
            });
            return;
        }

        // 武器类事件：使用 weaponId
        if (eventName == "OnWeaponSwing")
        {
            EventBus.Instance.Publish(new WeaponUsedEvent
            {
                weapon_id = weaponId > 0 ? weaponId.ToString() : "unknown",
                usage_type = "melee"
            });
            return;
        }

        // 其他事件通过字典查找
        if (_eventMap.TryGetValue(eventName, out var action))
        {
            action.Invoke();
        }
    }
}

// 在 Unity Animator 中配置 AnimationEvent：
// - Object: 绑定 AnimationEventBridge 组件
// - Function: 选择 "OnAnimationEvent"
// - String Parameter: 传入事件名称（eventName）
// - Int Parameter: 用于 entityId（交互类）或 weaponId（武器类），按事件类型选用
```

### 4. 动画参数定义

```csharp
// AnimationParameters.cs
/// <summary>
/// Animator Controller 使用的浮点参数
/// 命名遵循 Unity 规范（首字母小写 + 驼峰）
/// </summary>
public static class AnimationParameters
{
    // Float Parameters
    public static readonly int Speed = Animator.StringToHash("Speed");
    public static readonly int MoveDirection = Animator.StringToHash("MoveDirection");
    public static readonly int CrouchAmount = Animator.StringToHash("CrouchAmount");
    public static readonly int ActionProgress = Animator.StringToHash("ActionProgress");

    // Int Parameters
    public static readonly int BaseState = Animator.StringToHash("BaseState");
    public static readonly int ActionState = Animator.StringToHash("ActionState");

    // Bool Parameters
    public static readonly int IsMoving = Animator.StringToHash("IsMoving");
    public static readonly int IsCrouching = Animator.StringToHash("IsCrouching");
    public static readonly int IsDead = Animator.StringToHash("IsDead");
    public static readonly int InCombat = Animator.StringToHash("InCombat");
}

// AnimationParameterUpdater.cs
using UnityEngine;

/// <summary>
/// 负责将游戏状态同步到 Animator 参数
/// 挂载在角色 GameObject 上
/// </summary>
public class AnimationParameterUpdater : MonoBehaviour
{
    [SerializeField] private Animator _animator;
    [SerializeField] private Rigidbody _rigidbody;  // 用于获取速度
    [SerializeField] private bool _isCrouching;    // 由 PlayerController 通过方法设置

    /// <summary>
    /// 由 PlayerController 调用，更新当前是否为潜行状态
    /// </summary>
    public void SetCrouching(bool isCrouching)
    {
        _isCrouching = isCrouching;
        _animator.SetBool(AnimationParameters.IsCrouching, _isCrouching);
    }

    private void Update()
    {
        if (_rigidbody != null)
        {
            // 同步移动状态（使用水平速度）
            // 注意：Unity 6 中 Rigidbody.velocity 已弃用，改用 linearVelocity
            Vector3 horizontalVelocity = new Vector3(_rigidbody.linearVelocity.x, 0, _rigidbody.linearVelocity.z);
            _animator.SetFloat(AnimationParameters.Speed, horizontalVelocity.magnitude);
            _animator.SetBool(AnimationParameters.IsMoving, horizontalVelocity.magnitude > 0.1f);
        }
    }

    /// <summary>
    /// 设置基础动画状态（由 PlayerController/NPCController 调用）
    /// </summary>
    public void SetBaseState(AnimationState state)
    {
        _animator.SetInteger(AnimationParameters.BaseState, (int)state);
    }

    /// <summary>
    /// 设置动作动画状态（由 GrittyTakedowns 等系统调用）
    /// </summary>
    public void SetActionState(AnimationState state)
    {
        _animator.SetInteger(AnimationParameters.ActionState, (int)state);
    }
}
```

### 5. 动画层权重控制

```csharp
// AnimationLayerWeightController.cs
using System.Collections;
using UnityEngine;

/// <summary>
/// 动画层权重控制器（MonoBehaviour）
/// 挂载在与 AnimationStateMachine 相同的 GameObject 上，
/// 由 AnimationStateMachine.Awake() 通过 GetComponent 自动获取。
///
/// **设计变更说明（P1 修复）**：
/// 原设计为 Plain C# class，通过 SetAnimator/SetCoroutineRunner 手动注入。
/// 问题在于 AnimationStateMachine.Awake() 使用 GetComponent 查找它，
/// 而 Plain C# class 无法通过 GetComponent 获取，导致运行时永远为 null。
/// 改为 MonoBehaviour 后，GetComponent 可正确找到组件，
/// 协程也可直接通过 StartCoroutine 运行，无需外部注入运行器。
/// </summary>
public class AnimationLayerWeightController : MonoBehaviour
{
    [SerializeField] private Animator _animator;

    /// <summary>
    /// 检查是否已初始化（Animator 已赋值）
    /// </summary>
    public bool IsInitialized => _animator != null;

    /// <summary>
    /// 每个动画层的当前淡入淡出协程引用
    /// key: AnimationLayer 枚举整数值
    /// 每次 FadeToWeight 调用前先停止同层旧协程，防止多次调用时权重值被多个协程同时修改
    /// </summary>
    private Dictionary<int, Coroutine> _lerpCoroutines = new();

    /// <summary>
    /// 设置指定动画层的权重
    /// </summary>
    public void SetLayerWeight(AnimationLayer layer, float weight)
    {
        if (_animator == null)
        {
            Debug.LogWarning("[AnimationLayerWeightController] Animator not assigned in Inspector");
            return;
        }
        int layerIndex = (int)layer;
        _animator.SetLayerWeight(layerIndex, Mathf.Clamp01(weight));
    }

    /// <summary>
    /// 平滑过渡到目标权重
    /// 自动取消同层未完成的旧过渡协程，防止多次调用时出现权重闪烁
    /// </summary>
    public void FadeToWeight(AnimationLayer layer, float targetWeight, float duration)
    {
        int key = (int)layer;

        // 停止该层上正在执行的旧协程
        if (_lerpCoroutines.TryGetValue(key, out var existing) && existing != null)
        {
            StopCoroutine(existing);
        }

        _lerpCoroutines[key] = StartCoroutine(LerpWeightCoroutine(layer, targetWeight, duration));
    }

    private System.Collections.IEnumerator LerpWeightCoroutine(
        AnimationLayer layer, float target, float duration)
    {
        int layerIndex = (int)layer;
        float start = _animator.GetLayerWeight(layerIndex);
        float elapsed = 0f;

        while (elapsed < duration)
        {
            // P0 修复：使用 unscaledDeltaTime 确保不受 Time.timeScale 影响
            // 这样暂停菜单或慢动作时，动画层权重过渡仍能正常完成
            elapsed += Time.unscaledDeltaTime;
            float t = elapsed / duration;
            _animator.SetLayerWeight(layerIndex, Mathf.Lerp(start, target, t));
            yield return null;
        }

        _animator.SetLayerWeight(layerIndex, target);

        // 协程自然完成，从字典移除引用
        _lerpCoroutines.Remove(layerIndex);
    }
}
```

### 6. 动画层控制器

```csharp
// BaseLayerController.cs
/// <summary>
/// Base 层动画控制器
/// 负责管理角色基础动画状态（移动、待机、潜行、死亡）
/// 由 PlayerController/NPCController 驱动
/// </summary>
public class BaseLayerController
{
    private readonly Animator _animator;

    public BaseLayerController(Animator animator)
    {
        _animator = animator;
    }

    /// <summary>
    /// 设置基础动画状态
    /// </summary>
    public void SetState(AnimationState state)
    {
        _animator.SetInteger(AnimationParameters.BaseState, (int)state);
    }

    /// <summary>
    /// 设置移动速度（用于 BlendTree）
    /// </summary>
    public void SetSpeed(float speed)
    {
        _animator.SetFloat(AnimationParameters.Speed, speed);
    }

    /// <summary>
    /// 设置是否蹲伏
    /// </summary>
    public void SetCrouching(bool isCrouching)
    {
        _animator.SetBool(AnimationParameters.IsCrouching, isCrouching);
    }
}

// ActionLayerController.cs
/// <summary>
/// Action 层动画控制器
/// 负责管理角色动作动画（处决、交互动画、技能动画）
/// 由游戏系统通过 ActionStateMachine 触发
/// 权重: Additive，通过 AnimationLayerWeightController 控制权重
/// </summary>
public class ActionLayerController
{
    private readonly Animator _animator;
    private readonly AnimationLayerWeightController _weightController;

    public ActionLayerController(Animator animator, AnimationLayerWeightController weightController)
    {
        _animator = animator;
        _weightController = weightController;
    }

    /// <summary>
    /// 播放动作动画
    /// Action 层使用 Additive 混合，权重从 0 渐变到 1
    /// </summary>
    public void PlayAction(AnimationState actionState, float fadeInDuration = 0.1f)
    {
        _animator.SetInteger(AnimationParameters.ActionState, (int)actionState);
        _weightController?.FadeToWeight(AnimationLayer.Action, 1f, fadeInDuration);
    }

    /// <summary>
    /// 结束动作动画
    /// Action 层权重渐变回 0，状态回归 None
    /// </summary>
    public void EndAction(float fadeOutDuration = 0.2f)
    {
        _weightController?.FadeToWeight(AnimationLayer.Action, 0f, fadeOutDuration);
        _animator.SetInteger(AnimationParameters.ActionState, (int)AnimationState.None);
    }
}

// OverlayLayerController.cs
/// <summary>
/// Overlay 层动画控制器
/// 负责管理覆盖层动画（表情动画、武器挂载、披风物理）
/// 权重: Additive，叠加在 Base+Action 组合之上
/// </summary>
public class OverlayLayerController
{
    private readonly Animator _animator;
    private readonly AnimationLayerWeightController _weightController;

    public OverlayLayerController(Animator animator, AnimationLayerWeightController weightController)
    {
        _animator = animator;
        _weightController = weightController;
    }

    /// <summary>
    /// 播放覆盖层动画
    /// </summary>
    public void PlayOverlay(string overlayName, float fadeInDuration = 0.15f)
    {
        // Overlay 层动画通过 Animator Controller 的 Trigger 参数触发
        _animator.SetTrigger(Animator.StringToHash(overlayName));
        _weightController?.FadeToWeight(AnimationLayer.Overlay, 1f, fadeInDuration);
    }

    /// <summary>
    /// 停止覆盖层动画
    /// </summary>
    public void StopOverlay(float fadeOutDuration = 0.15f)
    {
        _weightController?.FadeToWeight(AnimationLayer.Overlay, 0f, fadeOutDuration);
    }

    /// <summary>
    /// 设置披风物理模拟强度
    /// </summary>
    public void SetCapePhysics(float intensity)
    {
        _animator.SetFloat(Animator.StringToHash("CapePhysicsIntensity"), intensity);
    }
}
```

### 6.5. 骨骼动画与 IK 支持

```csharp
// IKSolver.cs
using UnityEngine;

/// <summary>
/// IK 求解器接口
/// 用于动画 IK 处理的抽象
/// </summary>
public interface IKSolver
{
    void SetRightHandWeapon(Transform weaponMount);
    void SetLeftHandWeapon(Transform weaponMount);
    void SetLookAtTarget(Transform target);
    void SetFootIKTarget(AvatarIKGoal foot, Transform target);
    void ClearIKTargets();
}

// AnimationIKHandler.cs
using UnityEngine;

/// <summary>
/// 处理骨骼 IK（Inverse Kinematics）
/// 支持：武器瞄准、脚步 IK、环境交互对齐
/// </summary>
public class AnimationIKHandler : MonoBehaviour, IKSolver
{
    [SerializeField] private Animator _animator;
    [SerializeField] private Transform _leftHandTarget;
    [SerializeField] private Transform _rightHandTarget;
    [SerializeField] private Transform _lookAtTarget;

    /// <summary>
    /// 左脚 IK 目标
    /// </summary>
    private Transform _leftFootTarget;

    /// <summary>
    /// 右脚 IK 目标
    /// </summary>
    private Transform _rightFootTarget;

    /// <summary>
    /// IK 处理的动画层索引
    /// 默认为 Base Layer (0)
    ///
    /// **设计说明**：Unity 的 OnAnimatorIK 会在每个启用 IK 的层上调用。
    /// 大多数 IK 逻辑（如武器瞄准、注视）应在 Base Layer (0) 上处理，
    /// 因为 Base Layer 始终处于激活状态。如果需要其他层支持 IK，
    /// 可在 Inspector 中调整此值。
    /// </summary>
    [SerializeField] private int _ikLayerIndex = 0;

    private void OnAnimatorIK(int layerIndex)
    {
        // IK 仅在配置的层上处理（默认 Base Layer）
        if (layerIndex != _ikLayerIndex) return;

        // 武器瞄准 IK（右手）
        if (_rightHandTarget != null)
        {
            _animator.SetIKPositionWeight(AvatarIKGoal.RightHand, 1f);
            _animator.SetIKRotationWeight(AvatarIKGoal.RightHand, 1f);
            _animator.SetIKPosition(AvatarIKGoal.RightHand, _rightHandTarget.position);
            _animator.SetIKRotation(AvatarIKGoal.RightHand, _rightHandTarget.rotation);
        }
        else
        {
            // 清除右手 IK（防止上一帧的目标残留）
            _animator.SetIKPositionWeight(AvatarIKGoal.RightHand, 0f);
            _animator.SetIKRotationWeight(AvatarIKGoal.RightHand, 0f);
        }

        // 左手 IK
        if (_leftHandTarget != null)
        {
            _animator.SetIKPositionWeight(AvatarIKGoal.LeftHand, 1f);
            _animator.SetIKRotationWeight(AvatarIKGoal.LeftHand, 1f);
            _animator.SetIKPosition(AvatarIKGoal.LeftHand, _leftHandTarget.position);
            _animator.SetIKRotation(AvatarIKGoal.LeftHand, _leftHandTarget.rotation);
        }
        else
        {
            _animator.SetIKPositionWeight(AvatarIKGoal.LeftHand, 0f);
            _animator.SetIKRotationWeight(AvatarIKGoal.LeftHand, 0f);
        }

        // 头部注视 IK
        if (_lookAtTarget != null)
        {
            _animator.SetLookAtWeight(1f);
            _animator.SetLookAtPosition(_lookAtTarget.position);
        }
        else
        {
            _animator.SetLookAtWeight(0f);
        }

        // 脚部 IK（仅在有目标时启用）
        if (_leftFootTarget != null)
        {
            _animator.SetIKPositionWeight(AvatarIKGoal.LeftFoot, 1f);
            _animator.SetIKRotationWeight(AvatarIKGoal.LeftFoot, 1f);
            _animator.SetIKPosition(AvatarIKGoal.LeftFoot, _leftFootTarget.position);
            _animator.SetIKRotation(AvatarIKGoal.LeftFoot, _leftFootTarget.rotation);
        }
        else
        {
            _animator.SetIKPositionWeight(AvatarIKGoal.LeftFoot, 0f);
            _animator.SetIKRotationWeight(AvatarIKGoal.LeftFoot, 0f);
        }

        if (_rightFootTarget != null)
        {
            _animator.SetIKPositionWeight(AvatarIKGoal.RightFoot, 1f);
            _animator.SetIKRotationWeight(AvatarIKGoal.RightFoot, 1f);
            _animator.SetIKPosition(AvatarIKGoal.RightFoot, _rightFootTarget.position);
            _animator.SetIKRotation(AvatarIKGoal.RightFoot, _rightFootTarget.rotation);
        }
        else
        {
            _animator.SetIKPositionWeight(AvatarIKGoal.RightFoot, 0f);
            _animator.SetIKRotationWeight(AvatarIKGoal.RightFoot, 0f);
        }
    }

    /// <summary>
    /// 设置右手武器挂载点（由武器系统调用）
    /// </summary>
    public void SetRightHandWeapon(Transform weaponMount)
    {
        _rightHandTarget = weaponMount;
    }

    /// <summary>
    /// 设置左手武器挂载点（由武器系统调用）
    /// </summary>
    public void SetLeftHandWeapon(Transform weaponMount)
    {
        _leftHandTarget = weaponMount;
    }

    /// <summary>
    /// 设置注视目标（由相机系统调用）
    /// </summary>
    public void SetLookAtTarget(Transform target)
    {
        _lookAtTarget = target;
    }

    /// <summary>
    /// 设置脚部 IK 目标（由环境交互系统调用）
    /// </summary>
    /// <param name="foot">AvatarIKGoal.LeftFoot 或 AvatarIKGoal.RightFoot</param>
    /// <param name="target">脚部目标 Transform（如地面上的 IK 标记点）</param>
    public void SetFootIKTarget(AvatarIKGoal foot, Transform target)
    {
        switch (foot)
        {
            case AvatarIKGoal.LeftFoot:
                _leftFootTarget = target;
                break;
            case AvatarIKGoal.RightFoot:
                _rightFootTarget = target;
                break;
            default:
                Debug.LogWarning($"[AnimationIKHandler] SetFootIKTarget: unsupported foot {foot}");
                break;
        }
    }

    /// <summary>
    /// 清除所有 IK 目标
    /// </summary>
    public void ClearIKTargets()
    {
        _rightHandTarget = null;
        _leftHandTarget = null;
        _lookAtTarget = null;
        _leftFootTarget = null;
        _rightFootTarget = null;
    }
}
```

#### IK LOD 策略

| LOD Level | IK 类型 | 触发条件 | 性能开销 |
|------------|---------|----------|----------|
| **LOD 0 (Full)** | All IKs + LookAt + Foot IK | 距离 < 3m 或处于交互状态 | 高 |
| **LOD 1 (Partial)** | Hand IK + LookAt | 距离 3-8m 或 NPC 正面对玩家 | 中 |
| **LOD 2 (Minimal)** | Hand IK only | 距离 8-15m | 低 |
| **LOD 3 (None)** | No IK | 距离 > 15m 或相机看不到角色背面 | 无 |

**切换逻辑**：
```csharp
public void UpdateIKLOD(Vector3 characterPosition, Vector3 cameraPosition, bool isInInteraction)
{
    var distance = Vector3.Distance(characterPosition, cameraPosition);

    if (isInInteraction || distance < 3f)
        _currentIKLOD = IKLODLevel.Full;
    else if (distance < 8f)
        _currentIKLOD = IKLODLevel.Partial;
    else if (distance < 15f)
        _currentIKLOD = IKLODLevel.Minimal;
    else
        _currentIKLOD = IKLODLevel.None;
}
```

> **设计说明**：IK 计算是 CPU 密集型操作，PC 平台上大量角色同时进行完整 IK 计算会严重影响帧率。通过 LOD 策略，远距离角色的 IK 计算可以被跳过或简化，保持帧率稳定。PS5 版本可考虑使用 GPU Skinning 进一步优化。

### 6.6-6.8 节缺失说明

> **§6.6-6.8 说明**：骨骼动画与 IK 支持的内容已在 §6.5 中完整定义。
> 原有 §6.6（动画事件桥接）、§6.7（动画状态映射）、§6.8（调参配置）
> 已分别整合到 §2.5（AnimationState ↔ PlayerMovementState 映射）和 §4（动画参数定义）中。
> 本文档结构现已统一为：基础定义(§1-§2) → 控制器(§3-§5) → IK支持(§6.5) → 项目结构(§7)

### 7. Unity 项目结构（Presentation Layer）

```
Assets/Game/
├── Presentation/
│   └── Animation/
│       ├── AnimationStateMachine.cs      # 状态机核心
│       ├── AnimationLayerWeightController.cs
│       ├── AnimationParameterUpdater.cs
│       ├── AnimationIKHandler.cs
│       ├── AnimationEventBridge.cs      # AnimationEvent → EventBus 桥接
│       ├── Layers/
│       │   ├── BaseLayerController.cs   # Base 层控制
│       │   ├── ActionLayerController.cs # Action 层控制
│       │   └── OverlayLayerController.cs
│       ├── Parameters/
│       │   └── AnimationParameters.cs  # 参数名称常量
│       └── Config/
│           └── AnimationTuningSO.cs     # 调参配置
```

> **注意**：相机震动（Shake）功能由 ADR-0026 相机系统的 CameraShakeManager 统一管理，
> 不属于动画系统的职责。动画系统通过 AnimationEvent 触发 CameraShakeRequestEvent 事件来间接控制震动。

---

## Alternatives Considered

### Alternative 1: 单一 Animator 状态机

- **描述**：所有动画状态放在一个 Animator Controller 中
- **Pros**：简单直观，调试方便
- **Cons**：
  - 状态数膨胀（移动 x 方向 x 姿态 x 武器组合）
  - 难以管理动作层优先级
  - 动画师协作困难
- **拒绝理由**：俯视角潜行游戏需要多层动画叠加，单一状态机无法优雅处理

### Alternative 2: 第三方动画中间件（Motion Matching）

- **描述**：使用 Motion Matching 库（如 Kawaii Physics、Motion Sensor）
- **Pros**：动画自然流畅
- **Cons**：
  - 引入额外依赖
  - 内存/CPU 开销增加
  - 与 Unity Animation System 集成复杂度高
- **拒绝理由**：团队首次使用 Motion Matching 学习成本高，不适合当前阶段

### Alternative 3: 纯代码动画（无 Animator）

- **描述**：使用 DOTween/LeanTween 直接操作 Transform
- **Pros**：代码控制精确，无 Animator 开销
- **Cons**：
  - 无法使用 Unity 的动画分层、IK、BlendTree
  - 复杂动画状态逻辑难以维护
- **拒绝理由**：需要支持骨骼 IK 和动画层，纯代码方案无法满足

---

## Consequences

### Positive

- **分层清晰**：Base/Action/Overlay 分工明确，便于动画师和程序员协作
- **事件驱动**：AnimationEvent 转换为 EventBus，支持跨系统动画通知
- **可扩展**：支持 Humanoid 重定向和 2D 骨骼动画
- **IK 支持**：武器瞄准、注视等功能有标准实现路径

### Negative

- **Animator 管理复杂**：多层权重控制需要仔细调试
- **动画事件桥接层**：额外代码增加维护成本
- **调参工作量**：BlendTree 参数需要策划和动画师反复调整

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **动画层冲突** | Base 和 Action 层同时播放导致姿态异常 | Action 层结束时强制设置权重为 0 |
| **IK 性能** | 大量角色同时 IK 计算开销大 | PS5 使用 GPU Skinning，PC 限制 IK 数量 |
| **动画事件丢失** | 动画帧率高时事件可能被跳过 | 使用 FixedUpdate 同步，使用标志位防抖 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 2ms/帧 | 20 个动画同时播放 |
| **Memory** | ≤ 64MB | 动画资产内存预算 |
| **GPU** | < 5% | 骨骼动画 Skinning |
| **Load Time** | < 2s | 关键动画预加载 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 AnimationStateMachine 核心类
- [ ] 创建 AnimationParameters 常量定义
- [ ] 创建 AnimationEventBridge 桥接器

### Phase 2: 动画层实现
- [ ] 实现 BaseLayerController（移动、待机）
- [ ] 实现 ActionLayerController（处决动画触发）
- [ ] 实现 OverlayLayerController（武器挂载）

### Phase 3: 事件集成
- [ ] 集成 AnimationParameterUpdater 到 PlayerController
- [ ] 集成 AnimationEventBridge 到 GrittyTakedowns
- [ ] 测试 AnimationEvent → EventBus 转换

### Phase 4: IK 支持
- [ ] 实现 AnimationIKHandler
- [ ] 与武器系统集成右手 IK
- [ ] 与相机系统集成注视 IK

### Phase 5: 动画-相机协调协议
- [ ] 实现 `ExecutionCameraRequest` 事件（见下节）
- [ ] 在 GrittyTakedowns 处决动画播放前发送协调事件
- [ ] CameraManager 订阅并响应协调事件

---

### 动画-相机协调协议

> **⚠️ 问题背景**：`LockOnCameraBehavior` 的 `duration` 参数与动画系统的处决动画时长相互独立，可能导致相机已切回 Follow 但动画仍在播放，或反之。

#### 协调事件定义

```csharp
// ExecutionCameraRequestEvent.cs
/// <summary>
/// 动画系统向相机系统发送的协调请求
/// 在播放需要相机配合的动画（如处决动画）前发送
/// </summary>
public struct ExecutionCameraRequestEvent
{
    /// <summary>
    /// 请求的相机目标（被处决的 NPC）
    /// </summary>
    public Transform target;

    /// <summary>
    /// 期望的相机持续时间（秒）
    /// 相机应保持 LockOn 状态直到动画完成或超时
    /// </summary>
    public float expectedDuration;

    /// <summary>
    /// 请求来源系统
    /// </summary>
    public string sourceSystem;
}
```

#### 协调流程

```
GrittyTakedowns 准备处决动画
        │
        ▼
发布 ExecutionCameraRequestEvent(expectedDuration)
        │
        ▼
CameraManager 接收事件，设置 LockOn(target, expectedDuration)
        │
        ▼
开始播放处决动画（AnimationSystem）
        │
        ▼
动画完成 → AnimationEventBridge.OnAnimationEvent("OnAnimationEnd")
        │
        ▼
GrittyTakedowns 发送 CameraTransitionRequest(CameraState.Follow)
        │
        ▼
相机切换回 Follow 状态
```

#### CameraManager 响应协调事件

```csharp
// 在 CameraManager 中订阅 ExecutionCameraRequestEvent
EventBus.Instance.Subscribe<ExecutionCameraRequestEvent>(OnExecutionCameraRequest);

private void OnExecutionCameraRequest(ExecutionCameraRequestEvent evt)
{
    if (_stateMachine.CurrentState == CameraState.LockOn)
    {
        // 已处于 LockOn，无需重复设置
        return;
    }

    // 请求切换到 LockOn 状态
    _stateMachine.RequestState(CameraState.LockOn);
    var lockOn = _stateMachine.CurrentBehavior as LockOnCameraBehavior;
    lockOn.SetTarget(evt.target, evt.expectedDuration);
}
```

#### 注意事项

1. **超时保护**：`expectedDuration` 仅作为参考，实际超时由 `LockOnCameraBehavior` 的 `duration` 参数控制
2. **状态检查**：如果相机已在 LockOn 状态，忽略重复请求
3. **动画取消**：如果动画被中断（如玩家受伤），GrittyTakedowns 应发送 `CameraTransitionRequest(CameraState.Follow)` 恢复相机

---

## Validation Criteria

1. **分层验证**：Base 层和 Action 层可以独立切换，权重正确
2. **事件验证**：AnimationEvent 正确转换为 EventBus 事件
3. **状态验证**：AnimationState 与 PlayerMovementState/NPCState 一致
4. **IK 验证**：武器瞄准和注视 IK 正常工作
5. **性能验证**：20 个 NPC 同时播放动画时帧率稳定

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — AnimationEventBridge 通过 EventBus 与其他系统通信
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — **Animation System 属于 Presentation Layer**
- [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) — Action 层处决动画触发
- [ADR-0012: 世界地图与非线性叙事](./adr-0012-world-map-nonlinear-progression.md) — 过场动画与 World Map 集成
- [共享类型定义](./shared-types.md) — **PlayerMovementState (§7.2)、InteractionEvent (§9.3)、WeaponUsedEvent (§3.7)**

---

## 附录：类型依赖说明

| 类型 | 定义位置 | 说明 |
|------|---------|------|
| `PlayerMovementState` | shared-types.md §7.2 | 玩家移动状态枚举 |
| `AnimationLayer` | ADR-0024 本文档 | 动画层枚举 |
| `AnimationState` | ADR-0024 本文档 | 玩家/动作动画状态枚举 |
| `NPCAnimationState` | ADR-0024 本文档 | NPC 动画状态枚举（独立定义，勿与 AnimationState 混用） |
| `InteractionType` | shared-types.md §9.1 | 交互类型枚举 |
| `InteractionResult` | shared-types.md §9.2 | 交互结果枚举 |
| `WeaponUsedEvent` | shared-types.md §3.7 | 武器使用事件 |
| `Rigidbody` | Unity Engine | 提供速度数据用于动画参数同步 |
