# ADR-0024: 动画系统 (Animation System) 架构决策

## Status
**Proposed**

## Date
2026-04-12

## Last Updated
2026-04-12

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
│  │  权重: 1.0 (Override)                                        │   │
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
    /// 权重: Override，完全替代 Base 层
    /// </summary>
    Action = 1,

    /// <summary>
    /// 覆盖层：表情动画、武器挂载、披风物理
    /// 权重: Additive，叠加在 Base/Action 之上
    /// </summary>
    Overlay = 2
}
```

### 2. AnimationState 定义

```csharp
// AnimationState.cs
/// <summary>
/// 动画状态枚举（按层分组，值域连续）
/// 注意：枚举值按层分组，便于 Animator Controller 参数映射和调试
/// </summary>
public enum AnimationState
{
    // ========== Base Layer (0-99) ==========
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

    // ========== Action Layer (100-199) ==========
    /// <summary>潜行击杀</summary>
    StealthKill = 100,

    /// <summary>环境击杀</summary>
    EnvironmentKill = 101,

    /// <summary>终结</summary>
    FinishOff = 102,

    /// <summary>捆绑</summary>
    TieUp = 103,

    /// <summary>审讯</summary>
    Interrogate = 104,

    /// <summary>威胁</summary>
    Threaten = 105,

    /// <summary>搜身</summary>
    Search = 106,

    // ========== Movement Blend (200-255) ==========
    /// <summary>移动混合（行走/冲刺/蹲行）</summary>
    Locomotion = 200,

    /// <summary>战斗姿态移动</summary>
    CombatLocomotion = 201
}

// AnimationStateMachine.cs
/// <summary>
/// 动画状态机
/// 负责管理 Base/Action/Overlay 三层动画状态的切换
/// 注意：此类是非 MonoBehaviour 的 Plain C# class，
/// 协程通过注入的 MonoBehaviour runner 执行
/// </summary>
public class AnimationStateMachine
{
    private AnimationLayer _currentLayer;
    private AnimationState _baseState = AnimationState.None;
    private AnimationState _actionState = AnimationState.None;

    private MonoBehaviour _coroutineRunner;
    private System.Collections.IEnumerator _pendingCoroutine;

    /// <summary>
    /// 注入协程运行器（通常是角色身上的 MonoBehaviour）
    /// </summary>
    public void SetCoroutineRunner(MonoBehaviour runner)
    {
        _coroutineRunner = runner;
    }

    /// <summary>
    /// 设置 Base 层状态（由 PlayerController/NPCController 调用）
    /// </summary>
    public void SetBaseState(AnimationState state)
    {
        if (_actionState != AnimationState.None)
            return; // Action 层占用时不能切换 Base

        _baseState = state;
    }

    /// <summary>
    /// 播放动作层动画（由 GrittyTakedowns 等系统调用）
    /// Action 层播放完毕后自动回归 None，触发回调
    /// </summary>
    /// <param name="actionState">动作状态</param>
    /// <param name="fixedDuration">固定时长（秒），-1 表示使用 AnimationEvent</param>
    public void PlayAction(AnimationState actionState, float fixedDuration = -1f)
    {
        _actionState = actionState;

        if (fixedDuration > 0)
        {
            // 固定时长模式（用于动画时间已知的处决动画）
            _pendingCoroutine = ActionCompleteCoroutine(fixedDuration);
            _coroutineRunner?.StartCoroutine(_pendingCoroutine);
        }
        // 否则等待 AnimationEvent 触发 OnActionComplete
    }

    private System.Collections.IEnumerator ActionCompleteCoroutine(float duration)
    {
        yield return new WaitForSeconds(duration);
        OnActionComplete();
    }

    /// <summary>
    /// 由外部（AnimationEventBridge）调用，通知动作完成
    /// </summary>
    public void NotifyActionCompleted()
    {
        if (_pendingCoroutine != null && _coroutineRunner != null)
        {
            _coroutineRunner.StopCoroutine(_pendingCoroutine);
            _pendingCoroutine = null;
        }
        OnActionComplete();
    }

    public event Action<AnimationState> OnActionCompleted;

    /// <summary>
    /// 动作完成回调
    /// </summary>
    private void OnActionComplete()
    {
        var previousAction = _actionState;
        _actionState = AnimationState.None;
        _pendingCoroutine = null;

        if (previousAction != AnimationState.None)
        {
            OnActionCompleted?.Invoke(_baseState);
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
    /// <param name="targetId">目标实体ID（用于 InteractionEvent）</param>
    public void OnAnimationEvent(string eventName, int targetId = 0)
    {
        // 需要 targetId 的事件直接处理，不走字典映射
        if (eventName == "OnStealthKillHit")
        {
            EventBus.Instance.Publish(new InteractionEvent
            {
                type = InteractionType.StealthKill,
                target_id = targetId,  // 由 AnimationEvent 的 int 参数传入
                source = "animation",
                result = InteractionResult.Success
            });
            return;
        }

        if (eventName == "OnWeaponSwing")
        {
            EventBus.Instance.Publish(new WeaponUsedEvent
            {
                weapon_id = "unknown",
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
// - String Parameter: 传入事件名称
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
            Vector3 horizontalVelocity = new Vector3(_rigidbody.velocity.x, 0, _rigidbody.velocity.z);
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
public class AnimationLayerWeightController
{
    private Animator _animator;

    /// <summary>
    /// 设置指定动画层的权重
    /// </summary>
    public void SetLayerWeight(AnimationLayer layer, float weight)
    {
        int layerIndex = (int)layer;
        _animator.SetLayerWeight(layerIndex, Mathf.Clamp01(weight));
    }

    /// <summary>
    /// 平滑过渡到目标权重
    /// </summary>
    public void FadeToWeight(AnimationLayer layer, float targetWeight, float duration)
    {
        StartCoroutine(LerpWeightCoroutine(layer, targetWeight, duration));
    }

    private System.Collections.IEnumerator LerpWeightCoroutine(
        AnimationLayer layer, float target, float duration)
    {
        int layerIndex = (int)layer;
        float start = _animator.GetLayerWeight(layerIndex);
        float elapsed = 0f;

        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            float t = elapsed / duration;
            _animator.SetLayerWeight(layerIndex, Mathf.Lerp(start, target, t));
            yield return null;
        }

        _animator.SetLayerWeight(layerIndex, target);
    }
}
```

### 6. 骨骼动画与 IK 支持

```csharp
// AnimationIKHandler.cs
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

    private void OnAnimatorIK(int layerIndex)
    {
        if (layerIndex != (int)AnimationLayer.Base) return;

        // 武器瞄准 IK
        if (_rightHandTarget != null)
        {
            _animator.SetIKPositionWeight(AvatarIKGoal.RightHand, 1f);
            _animator.SetIKRotationWeight(AvatarIKGoal.RightHand, 1f);
            _animator.SetIKPosition(AvatarIKGoal.RightHand, _rightHandTarget.position);
            _animator.SetIKRotation(AvatarIKGoal.RightHand, _rightHandTarget.rotation);
        }

        // 头部注视 IK
        if (_lookAtTarget != null)
        {
            _animator.SetLookAtWeight(1f);
            _animator.SetLookAtPosition(_lookAtTarget.position);
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
    /// 设置注视目标（由相机系统调用）
    /// </summary>
    public void SetLookAtTarget(Transform target)
    {
        _lookAtTarget = target;
    }
}
```

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
> 不属于动画系统的职责。动画系统通过 AnimationEvent 触发 CameraShakeRequest 事件来间接控制震动。

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
| `AnimationState` | ADR-0024 本文档 | 动画状态枚举 |
| `InteractionType` | shared-types.md §9.1 | 交互类型枚举 |
| `InteractionResult` | shared-types.md §9.2 | 交互结果枚举 |
| `WeaponUsedEvent` | shared-types.md §3.7 | 武器使用事件 |
| `Rigidbody` | Unity Engine | 提供速度数据用于动画参数同步 |
