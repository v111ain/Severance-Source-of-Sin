# ADR-0009: 玩家控制器 (Player Controller) 架构决策

## Status
**Accepted** — [已修复] 本文档已更新，InputHandler 相关内容已废弃，统一使用 [ADR-0020](./adr-0020-input-system-architecture.md) 的 InputManager。

### 迁移状态
- [x] PlayerController.InputHandler → 已废弃，使用 InputManager.GetMoveInput() 等接口
- [x] PlayerMovementStateChangedEvent 发布者保持不变（仍由 PlayerController 发布）
- [x] InputArbitrator 与 InputManager 集成（已在 ADR-0020 中实现）
- [x] PlayerController 正确发布 PlayerPositionUpdatedEvent（见 §9 主控制器）

### 相关决策
- [ADR-0020: Input System](./adr-0020-input-system-architecture.md) — **输入系统的权威实现**

## Date
2026-04-09

## Last Updated
2026-04-15 (ADR 评审修复：CheckpointSystem 事件订阅修正 — 玩家死亡应订阅 PlayerDiedEvent，非 NPCStateChangedEvent)

## Context

### Problem Statement

Player Controller 是《断绝：罪恶之源》最基础的输入响应与状态管理系统。它负责将玩家操作转化为游戏世界中主角"父亲"的实际位移与状态切换。本游戏严格遵循"致命的脆弱感"支柱——移动沉重且谨慎，不支持无敌帧翻滚。作为连接玩家意图与所有交互系统的核心枢纽，它需要与以下系统交互：

1. **LOS System**：提供玩家位置和面朝方向以计算暴露值
2. **NPC AI System**：基于移动状态广播 NoiseEvent
3. **Environment Interaction**：提供射线检测结果
4. **Gritty Takedowns**：提供动作锁定（IsLocked）接口供外部接管

### Constraints

- **输入约束**：支持键鼠（WASD）和手柄（左摇杆）
- **移动约束**：无无敌帧，必须实现"沉重感"
- **锁定约束**：外部系统（处决等）需要能接管玩家控制权
- **性能约束**：输入响应必须 < 1 帧延迟

### Requirements

- **必须**：定义移动状态机（Idle/Walk/Sprint/Crouch/Action）
- **必须**：定义体力（Stamina）消耗与恢复系统
- **必须**：定义动作锁定（IsLocked）接口供外部系统接管
- **必须**：定义噪声广播（NoiseMadeEvent）机制
- **必须**：定义射线检测接口供 Environment Interaction 读取
- **必须**：定义与 Gritty Takedowns 的控制权移交机制

---

## Decision

### 架构决策

采用**状态机 + 输入处理 + 锁定接口**架构，**统一使用 Unity MonoBehaviour 单例模式**：

> **单例模式规范**：所有需要全局访问的子系统统一使用 `MonoBehaviour + Instance` 单例模式，避免"穷人的单例"（无参构造函数 + static Instance）导致的初始化顺序问题。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Player Controller 架构                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    PlayerController (MonoBehaviour)                  │   │
│  │  - 单例模式，全局访问                                              │   │
│  │  - 管理玩家实体生命周期                                             │   │
│  │  - 协调所有子系统                                                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│          ┌─────────────────────────┼─────────────────────────┐          │
│          ▼                         ▼                         ▼          │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐   │
│  │  InputManager      │ │  MovementSystem   │ │  StaminaSystem    │   │
│  │  (输入抽象层)       │ │  (移动系统)        │ │  (体力系统)        │   │
│  │  ★ 统一输入系统    │ │  ◆ 普通类        │ │  ◆ MonoBehaviour  │   │
│  │  (见 ADR-0020)     │ │                   │ │                   │   │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     MovementStateMachine                          │   │
│  │  ◆ MonoBehaviour 单例（统一单例模式）                              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐   │
│  │  ActionLockSystem  │ │  RaycastSystem    │ │  NoiseBroadcaster │   │
│  │  (普通类)          │ │  (普通类)          │ │  (普通类)          │   │
│  │ - Acquire/Release │ │ - Cone Raycast    │ │ - 状态变化时发布  │   │
│  │ - 超时自动释放     │ │ - 提供查询接口     │ │                   │   │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2. 输入处理（已废弃）

> **⚠️ 废弃通知**：以下 `InputHandler` 实现已废弃，统一使用 [ADR-0020](./adr-0020-input-system-architecture.md) 的 `InputManager`。
>
> 保留此段仅作为历史参考，实现时请使用 InputManager。

```csharp
// InputHandler.cs — 已废弃，请使用 InputManager
[Obsolete("Use InputManager instead. See ADR-0020.")]
public class InputHandler : MonoBehaviour
{
    // ... 已废弃的实现细节 ...
}
```

### 3. 移动状态机

> **注意**：`PlayerMovementState` 枚举已在 `shared-types.md §7.2` 统一定义，本 ADR 仅引用：
> ```csharp
> using PlayerMovementState = GlobalNamespace.PlayerMovementState;
> ```

`PlayerMovementState` 定义位置：`Assets/Game/Foundation/Shared/Types/PlayerMovementState.cs`
详细定义见 [shared-types.md §7.2](../architecture/shared-types.md#72-playermovementstate)

```csharp
// 状态机管理器（统一使用 Unity MonoBehaviour 单例模式）
public class MovementStateMachine : MonoBehaviour
{
    public static MovementStateMachine Instance { get; private set; }

    public PlayerMovementState CurrentState { get; private set; }

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        CurrentState = PlayerMovementState.IDLE;
    }

    // ForceState 设计说明：
    // - 虽然是 public 方法，但设计为由 PlayerController 内部调用
    // - 外部系统应通过 PlayerController.ForceMovementState() 中转
    // - 这样保持状态机封装性，便于未来可能的重构
    public void ForceState(PlayerMovementState newState)
    {
        PlayerMovementState oldState = CurrentState;
        CurrentState = newState;

        // 强制状态变更也触发事件通知，确保 LOS System 等订阅者能感知
        EventBus.Instance.Publish(new PlayerMovementStateChangedEvent
        {
            old_state = oldState,
            new_state = newState
        });
    }

    public void Update(
        PlayerMovementState current,
        bool hasDirectionalInput,
        bool isSprintHeld,
        bool isCrouchToggled,
        bool isActionLocked)
    {
        // 动作锁定优先级最高
        if (isActionLocked)
        {
            CurrentState = PlayerMovementState.ACTION;
            return;
        }

        // 潜行切换
        if (isCrouchToggled)
        {
            CurrentState = current switch
            {
                PlayerMovementState.IDLE => PlayerMovementState.CROUCH,
                PlayerMovementState.WALK => PlayerMovementState.CROUCH_WALK,
                PlayerMovementState.SPRINT => PlayerMovementState.CROUCH,  // 急停蹲下
                PlayerMovementState.CROUCH => PlayerMovementState.IDLE,
                PlayerMovementState.CROUCH_WALK => PlayerMovementState.WALK,
                _ => current
            };
            return;
        }

        // 冲刺判定（必须在有方向输入时按住冲刺键，且体力允许）
        if (hasDirectionalInput && isSprintHeld && StaminaSystem.Instance.CanSprint)
        {
            CurrentState = current switch
            {
                PlayerMovementState.IDLE => PlayerMovementState.SPRINT,
                PlayerMovementState.WALK => PlayerMovementState.SPRINT,
                PlayerMovementState.SPRINT => PlayerMovementState.SPRINT,
                _ => current
            };
            return;
        }

        // 方向输入 → 行走
        if (hasDirectionalInput)
        {
            CurrentState = current switch
            {
                PlayerMovementState.IDLE => PlayerMovementState.WALK,
                PlayerMovementState.WALK => PlayerMovementState.WALK,
                PlayerMovementState.SPRINT => PlayerMovementState.WALK,  // 体力耗尽降为行走
                PlayerMovementState.CROUCH => PlayerMovementState.CROUCH_WALK,
                _ => current
            };
            return;
        }

        // 无输入 → 待机
        CurrentState = current switch
        {
            PlayerMovementState.WALK => PlayerMovementState.IDLE,
            PlayerMovementState.SPRINT => PlayerMovementState.IDLE,
            PlayerMovementState.CROUCH_WALK => PlayerMovementState.CROUCH,
            _ => current
        };
    }
}
```

### 2. 输入处理

> **废弃说明**：以下 InputHandler 实现已废弃，统一使用 [ADR-0020](./adr-0020-input-system-architecture.md) 的 `InputManager`。
> 完整的历史实现已移至附录。

**InputHandler 历史实现**：见本文档末尾 [附录 E：废弃的 InputHandler 实现](#附录e废弃的-inputhandler-实现)。

### 4. 移动系统

```csharp
// MovementSystem.cs
public class MovementSystem
{
    public float CurrentSpeed { get; private set; }
    public Vector3 CurrentVelocity { get; private set; }
    public Vector3 FacingDirection { get; private set; }  // 面朝方向

    private CharacterController _characterController;

    // 配置参数（从 PlayerControllerConfigSO 读取，不再硬编码）
    private float _baseSpeed;
    private float _sprintMultiplier;
    private float _crouchMultiplier;

    /// <summary>
    /// 初始化移动系统
    /// </summary>
    /// <param name="characterController">CharacterController 组件引用（必须非 null）</param>
    /// <param name="config">配置数据（可选，传入 null 时使用默认值）</param>
    /// <exception cref="ArgumentNullException">当 characterController 为 null 时抛出</exception>
    public void Initialize(CharacterController characterController, PlayerControllerConfigSO config = null)
    {
        _characterController = characterController ?? throw new ArgumentNullException(nameof(characterController));

        // 从配置读取，无配置时使用默认值
        _baseSpeed = config?.BaseSpeed ?? 5f;
        _sprintMultiplier = config?.SprintMultiplier ?? 1.6f;
        _crouchMultiplier = config?.CrouchMultiplier ?? 0.5f;
    }

    public void Update(PlayerInput input, PlayerMovementState currentState, float deltaTime)
    {
        // 计算速度
        float multiplier = currentState switch
        {
            PlayerMovementState.WALK => 1.0f,
            PlayerMovementState.SPRINT => _sprintMultiplier,
            PlayerMovementState.CROUCH or PlayerMovementState.CROUCH_WALK => _crouchMultiplier,
            _ => 0f
        };

        CurrentSpeed = _baseSpeed * multiplier;

        // 计算移动向量
        Vector3 moveDirection = new Vector3(input.MoveDirection.x, 0, input.MoveDirection.y);
        CurrentVelocity = moveDirection * CurrentSpeed;

        // 应用移动（ACTION 状态下不执行移动，由外部接管）
        if (currentState != PlayerMovementState.ACTION)
        {
            _characterController.Move(CurrentVelocity * deltaTime);
        }

        // 更新面朝方向（即使在 ACTION 状态也更新，除非被外部接管）
        // 注意：外部系统接管时需调用 SetFacingDirectionExternal() 锁定面朝方向
        if (moveDirection.magnitude > 0.1f)
        {
            FacingDirection = moveDirection.normalized;
        }
    }

    // 供外部系统（如 Gritty Takedowns）临时接管面朝方向
    public void SetFacingDirection(Vector3 direction)
    {
        FacingDirection = direction.normalized;
    }

    public bool CanStandUp()
    {
        // 向上发射射线检测
        if (Physics.Raycast(
            _characterController.transform.position + Vector3.up * 0.1f,
            Vector3.up,
            out var hit,
            _characterController.height))
        {
            return false;  // 有障碍物，不能站起
        }
        return true;
    }
}
```

### 5. 体力系统

> **循环依赖解决方案**：StaminaSystem 通过 Initialize() 注入 PlayerController 引用，避免在 Awake 中直接访问单例导致初始化顺序问题。

```csharp
// StaminaSystem.cs
// 【设计说明】继承 MonoBehaviour 是为了利用 Unity 生命周期和单例模式
// 但不使用 Unity Update() 是因为体力更新需要由 PlayerController 显式调用（受控更新顺序）
// 这种"受控 MonoBehaviour"模式在需要与其他子系统协调更新顺序时很有用
public class StaminaSystem : MonoBehaviour
{
    public static StaminaSystem Instance { get; private set; }

    public float CurrentStamina { get; private set; }
    public float MaxStamina => 100f;
    public bool IsExhausted => _isExhausted;
    public bool CanSprint => CurrentStamina > 0 && !_isExhausted;

    private bool _isExhausted = false;

    // 配置参数（从 PlayerControllerConfigSO 读取，不再硬编码）
    private float _drainRate;
    private float _regenRate;
    private float _regenDelay;
    private float _exhaustionThreshold;
    private float _timeSinceLastSprint = 0f;

    // 注入的 PlayerController 引用（用于 ForceMovementState 调用）
    private PlayerController _playerController;

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        CurrentStamina = MaxStamina;
    }

    /// <summary>
    /// 初始化体力系统（由 PlayerController 在 Awake 中调用）
    /// </summary>
    /// <param name="playerController">PlayerController 引用（非 null）</param>
    /// <param name="config">配置数据（可选，传入 null 时使用默认值）</param>
    public void Initialize(PlayerController playerController, PlayerControllerConfigSO config = null)
    {
        _playerController = playerController ?? throw new ArgumentNullException(nameof(playerController));

        // 从配置读取，无配置时使用默认值
        _drainRate = config?.StaminaDrainRate ?? 20f;
        _regenRate = config?.StaminaRegenRate ?? 15f;
        _regenDelay = config?.StaminaRegenDelay ?? 2f;
        _exhaustionThreshold = config?.ExhaustionThreshold ?? 30f;
    }

    public void Update(PlayerMovementState state, float deltaTime)
    {
        switch (state)
        {
            case PlayerMovementState.SPRINT:
                // 消耗体力
                CurrentStamina = Mathf.Max(0, CurrentStamina - _drainRate * deltaTime);
                _timeSinceLastSprint = 0f;

                if (CurrentStamina <= 0)
                {
                    _isExhausted = true;
                    // 强制降为行走（通过注入的 PlayerController 引用调用，避免循环依赖）
                    _playerController.ForceMovementState(PlayerMovementState.WALK);
                }
                break;

            default:
                // 非冲刺状态，延迟后开始恢复
                // 注意：如果在恢复期间玩家再次冲刺，_timeSinceLastSprint 会立即重置为 0，
                // 导致恢复中断并等待下一个 _regenDelay 后才能继续恢复。这是预期行为。
                _timeSinceLastSprint += deltaTime;

                if (_timeSinceLastSprint >= _regenDelay)
                {
                    // 检查是否从 exhaustion 中恢复
                    if (_isExhausted)
                    {
                        if (CurrentStamina >= _exhaustionThreshold)
                            _isExhausted = false;
                        else
                            CurrentStamina += _regenRate * deltaTime;
                    }
                    else
                    {
                        CurrentStamina = Mathf.Min(MaxStamina, CurrentStamina + _regenRate * deltaTime);
                    }
                }
                break;
        }
    }
}
```

### 6. 动作锁定系统

> **重要**：ActionLockSystem 统一定义在 [shared-types.md](./shared-types.md) 中。
> 此处仅说明 Player Controller 如何使用，不重复实现细节。

```csharp
// PlayerController.cs - 动作锁定集成
private void Update()
{
    // ...

    // 8. 更新动作锁定超时（调用共享的 ActionLockSystem）
    ActionLockSystem.Instance.Update();
}

// 对外接口：动作锁定（供外部系统调用）
public bool AcquireActionLock(string requester, float expectedDuration)
{
    return ActionLockSystem.Instance.AcquireLock(requester, ActionLockType.Interaction, expectedDuration);
}

public bool ReleaseActionLock(string requester)
{
    return ActionLockSystem.Instance.ReleaseLock(requester);
}
```

### 6.5. [已修复] InputArbitrator 集成说明

> **职责边界说明**：
> - **ActionLockSystem**（shared-types.md §4.2）：管理玩家控制权的**锁定/解锁**，用于 GrittyTakedowns 等系统需要完全接管玩家输入的场景
> - **InputArbitrator**（ADR-0020 §3）：管理输入**路由**，决定哪个消费者（Consumer）接收输入
> 两者协同工作：ActionLockSystem 控制是否启用输入，InputArbitrator 控制输入路由到哪个系统

**InputArbitrator.SetActiveConsumer() 调用时机和调用者**：

| 调用者 | 调用时机 | 设置的 Consumer | 优先级 |
|--------|----------|-----------------|--------|
| **PlayerController** | 正常游戏时 | `InputConsumer.PlayerController` | 10 |
| **LOSSystem** | 进入专注监听模式时 | `InputConsumer.LOSSystem` | 20 |
| **GrittyTakedowns** | 开始处决/交互时 | `InputConsumer.GrittyTakedowns` | 30 |
| **DialogueSystem** | 开始对话时 | `InputConsumer.DialogueSystem` | 40 |
| **UISystem** | 打开 UI 菜单时 | `InputConsumer.UISystem` | 50 |
| **PauseMenu** | 打开暂停菜单时 | `InputConsumer.PauseMenu` | 100 |

**优先级设置规则**：
- 数字越大优先级越高
- 仅当新消费者优先级 >= 当前消费者优先级时才能切换
- 详见 [ADR-0020 输入仲裁层](./adr-0020-input-system-architecture.md#3-输入仲裁层)

**PlayerController 与 InputArbitrator 集成示例**：
```csharp
// 在 PlayerController 初始化时注册
InputArbitrator.Instance.RegisterConsumer(InputConsumer.PlayerController, 10);

// 在 Update 中检查是否应处理输入
if (InputArbitrator.Instance.ShouldRouteTo(InputConsumer.PlayerController))
{
    // 处理玩家输入...
}

// 当玩家进入 ACTION 状态时，InputArbitrator 自动路由到当前激活的消费者
// （由 GrittyTakedowns 等系统调用 SetActiveConsumer 设置）
```

### 7. 射线检测系统

```csharp
// IPlayerRaycastProvider.cs
/// <summary>
/// 玩家射线检测结果提供接口
/// 供 Environment Interaction 等外部系统订阅当前射线检测结果
/// 使用接口解耦，避免直接依赖 PlayerController
/// </summary>
public interface IPlayerRaycastProvider
{
    /// <summary>
    /// 获取当前射线检测结果
    /// </summary>
    RaycastResult GetCurrentRaycastResult();
}

// RaycastSystem.cs
public class RaycastSystem
{
    // 射线检测结果
    public struct RaycastResult
    {
        public bool hit;
        public int objectId;
        public InteractableType objectType;
        public float distance;
        public Vector3 normal;
    }

    private RaycastResult _currentResult;
    private float _rayLength = 2f;
    private float _coneAngle = 60f;  // 锥形角度
    private LayerMask _interactableLayer;  // 可交互物体所在层级

    public void SetInteractableLayer(LayerMask layerMask)
    {
        _interactableLayer = layerMask;
    }

    // 提供给 Environment Interaction 读取
    public RaycastResult GetCurrentResult() => _currentResult;

    public void Update(Vector3 origin, Vector3 direction)
    {
        _currentResult = new RaycastResult { hit = false };

        // 使用球形区域检测 + 锥形过滤
        // 球形检测可以精确控制检测范围，避免 RaycastAll 检测到无关物体
        Collider[] colliders = Physics.OverlapSphere(origin, _rayLength, _interactableLayer);

        float nearestDistance = float.MaxValue;

        foreach (var collider in colliders)
        {
            var interactable = collider.GetComponent<InteractableObject>();
            if (interactable == null)
                continue;

            // 计算是否在锥形范围内
            Vector3 toTarget = collider.transform.position - origin;
            float angle = Vector3.Angle(direction, toTarget);

            if (angle <= _coneAngle / 2f)
            {
                float distance = toTarget.magnitude;
                if (distance < nearestDistance)
                {
                    nearestDistance = distance;
                    _currentResult = new RaycastResult
                    {
                        hit = true,
                        objectId = interactable.ObjectId,
                        objectType = interactable.Type,
                        distance = distance,
                        normal = collider.transform.forward
                    };
                }
            }
        }
    }
}
```

### 8. 噪声广播系统

```csharp
// NoiseBroadcaster.cs
public class NoiseBroadcaster
{
    // 配置参数
    private float _walkRadius = 3f;
    private float _sprintRadius = 8f;
    private float _crouchRadius = 0f;  // 静音
    private float _noiseDuration = 0.5f;

    // 上一帧状态，用于检测状态变化
    private PlayerMovementState _previousState = PlayerMovementState.IDLE;

    public void Update(PlayerMovementState state, Vector3 position)
    {
        // 仅在状态改变时发布事件，避免每帧无意义广播
        if (state == _previousState)
            return;

        _previousState = state;

        float radius = state switch
        {
            PlayerMovementState.WALK => _walkRadius,
            PlayerMovementState.SPRINT => _sprintRadius,
            PlayerMovementState.CROUCH_WALK => _crouchRadius,
            _ => 0f
        };

        if (radius > 0f)
        {
            EventBus.Instance.Publish(new NoiseMadeEvent
            {
                position = position,
                radius = radius,
                noise_type = StateToNoiseType(state),
                duration = _noiseDuration,
                can_interrupt = true,
                source_entity_id = PlayerController.Instance.GetPlayerId()
            });
        }
    }

    private NoiseType StateToNoiseType(PlayerMovementState state)
    {
        return state switch
        {
            PlayerMovementState.WALK => NoiseType.WALK,
            PlayerMovementState.SPRINT => NoiseType.SPRINT,
            PlayerMovementState.CROUCH_WALK => NoiseType.CROUCH,
            PlayerMovementState.ACTION => NoiseType.INTERACTION,
            _ => NoiseType.NONE
        };
    }
}
```

### 9. 主控制器

> **子系统初始化规范**：
> - MonoBehaviour 子系统（InputHandler、MovementSystem、StaminaSystem、MovementStateMachine）：通过 GetComponent 获得，确保在同一个 GameObject 上
> - 普通类子系统（ActionLockSystem、RaycastSystem、NoiseBroadcaster）：使用 new 实例化
> - 所有子系统通过 Initialize() 方法注入依赖，避免在构造函数或 Awake 中直接依赖

```csharp
// PlayerController.cs
public class PlayerController : MonoBehaviour, IPlayerRaycastProvider
{
    public static PlayerController Instance { get; private set; }

    public int GetPlayerId() => _playerId;
    private int _playerId = 0;  // 玩家实体 ID，可由 EntityManager 分配

    // 子系统声明
    // ★ InputHandler 已废弃，统一使用 InputManager（见 ADR-0020）
    private MovementSystem _movementSystem;
    private StaminaSystem _staminaSystem;
    // ActionLockSystem 使用共享单例（见 shared-types.md）
    private RaycastSystem _raycastSystem;
    private NoiseBroadcaster _noiseBroadcaster;
    private MovementStateMachine _stateMachine;

    private void Awake()
    {
        // 单例初始化
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;

        // 初始化 MonoBehaviour 子系统
        // ★ InputHandler 已废弃，使用 InputManager（ADR-0020）
        _staminaSystem = GetComponent<StaminaSystem>() ?? gameObject.AddComponent<StaminaSystem>();
        _stateMachine = GetComponent<MovementStateMachine>() ?? gameObject.AddComponent<MovementStateMachine>();

        // 初始化普通类子系统（使用 new 实例化）
        // 注意：ActionLockSystem 使用共享单例，无需实例化（见 shared-types.md）
        _movementSystem = new MovementSystem();
        _raycastSystem = new RaycastSystem();
        _noiseBroadcaster = new NoiseBroadcaster();

        // 获取配置（可从 Resources 加载或 Inspector 指定）
        var config = GetComponent<PlayerControllerConfigSO>();

        // 初始化依赖注入（按依赖顺序）
        var characterController = GetComponent<CharacterController>();

        // 初始化移动系统
        if (_movementSystem != null)
            _movementSystem.Initialize(characterController, config);

        // 初始化体力系统（依赖 PlayerController 引用）
        if (_staminaSystem != null)
            _staminaSystem.Initialize(this, config);
    }

    private void Update()
    {
        // 1. 获取输入（使用 InputManager，详见 ADR-0020）
        var moveInput = InputManager.Instance.GetMoveInput();
        bool isSprintHeld = InputManager.Instance.IsSprintHeld();
        bool wasCrouchToggled = InputManager.Instance.WasCrouchToggled();

        // 2. 更新状态机
        // 检查 GrittyTakedowns 是否持有交互锁（详见 shared-types.md §4.2）
        bool isActionLocked = ActionLockSystem.Instance.HasLock("GrittyTakedowns");
        PlayerMovementState oldState = _stateMachine.CurrentState;
        _stateMachine.Update(
            _stateMachine.CurrentState,
            moveInput.magnitude > 0.1f,
            isSprintHeld,
            wasCrouchToggled,
            isActionLocked);

        // 2.5. 发布移动状态变化事件（供 LOS System 订阅）
        if (_stateMachine.CurrentState != oldState)
        {
            EventBus.Instance.Publish(new PlayerMovementStateChangedEvent
            {
                old_state = oldState,
                new_state = _stateMachine.CurrentState
            });
        }

        // 3. 更新体力
        _staminaSystem.Update(_stateMachine.CurrentState, Time.deltaTime);

        // 4. 更新移动
        // 构建 PlayerInput 结构体（兼容 MovementSystem）
        var input = new MovementSystem.PlayerInput
        {
            MoveDirection = moveInput,
            SprintHeld = isSprintHeld,
            CrouchToggled = wasCrouchToggled,
            ActionPressed = InputManager.Instance.WasActionPressed(),
            FocusPressed = InputManager.Instance.IsFocusHeld()
        };
        _movementSystem.Update(input, _stateMachine.CurrentState, Time.deltaTime);

        // 5. 头顶碰撞检测（防止低矮空间站起）
        if (_stateMachine.CurrentState == PlayerMovementState.CROUCH)
        {
            if (!_movementSystem.CanStandUp())
            {
                // 强制保持蹲下状态
                _stateMachine.ForceState(PlayerMovementState.CROUCH);
            }
        }
        else if (_stateMachine.CurrentState == PlayerMovementState.CROUCH_WALK && moveInput.magnitude <= 0.1f)
        {
            // 潜行移动中松手时，检查是否可以站起
            if (!_movementSystem.CanStandUp())
            {
                // 强制保持蹲下状态
                _stateMachine.ForceState(PlayerMovementState.CROUCH);
            }
        }

        // 6. 更新射线检测
        _raycastSystem.Update(
            transform.position,
            _movementSystem.FacingDirection);

        // 7. 广播噪声
        _noiseBroadcaster.Update(_stateMachine.CurrentState, transform.position);

// 8. 更新动作锁定超时（调用共享单例）
        ActionLockSystem.Instance.Update();

        // 9. [已修复] 发布玩家位置更新事件（供 Camera System 的 LockOnCameraBehavior 订阅）
        // 注意：此事件由 PlayerController 在每帧 Update 结束时发布，LockOnCameraBehavior 依赖此事件
        // 计算相机中点。事件发布频率由 PlayerController 控制（建议每帧一次）。
        EventBus.Instance.Publish(new PlayerPositionUpdatedEvent
        {
            position = transform.position,
            entityId = _playerId,
            timestamp = Time.time
        });
    }

    // 对外接口：动作锁定
    public bool AcquireActionLock(string requester, float expectedDuration)
    {
        return ActionLockSystem.Instance.AcquireLock(requester, ActionLockType.Interaction, expectedDuration);
    }

    public bool ReleaseActionLock(string requester)
    {
        return ActionLockSystem.Instance.ReleaseLock(requester);
    }

    // 对外接口：射线结果查询（供 Environment Interaction 使用）
    public RaycastSystem.RaycastResult GetCurrentRaycastResult()
    {
        return _raycastSystem.GetCurrentResult();
    }

    // 对外接口：获取当前移动状态（供 LOS System 订阅）
    public PlayerMovementState GetCurrentMovementState()
    {
        return _stateMachine.CurrentState;
    }

    // 对内/对外接口：强制切换移动状态（供 StaminaSystem 在体力耗尽时调用）
    // 注意：外部系统应通过此方法中转，不应直接调用 MovementStateMachine
    public void ForceMovementState(PlayerMovementState newState)
    {
        _stateMachine.ForceState(newState);
    }
}
```

### 9. Unity 项目结构

```
Assets/Game/
├── Foundation/
│   └── PlayerController/
│       ├── PlayerController.cs          # 主控制器（单例）
│       ├── Movement/
│       │   ├── MovementSystem.cs        # 移动系统
│       │   ├── PlayerMovementState.cs    # 状态枚举
│       │   └── MovementStateMachine.cs   # 状态机
│       ├── Stamina/
│       │   └── StaminaSystem.cs         # 体力系统
│       ├── ActionLock/
│       │   └── ActionLockSystem.cs      # 动作锁定系统
│       ├── Raycast/
│       │   └── RaycastSystem.cs         # 射线检测系统
│       ├── Noise/
│       │   └── NoiseBroadcaster.cs      # 噪声广播系统
│       └── Config/
│           └── PlayerControllerConfigSO.cs  # 配置 ScriptableObject
│
├── Infrastructure/
│   └── Input/
│       └── InputManager.cs              # 统一输入管理（见 ADR-0020）
```

#### PlayerControllerConfigSO 定义

```csharp
// PlayerControllerConfigSO.cs
[CreateAssetMenu(fileName = "PlayerControllerConfig", menuName = "Game/Player/Config")]
public class PlayerControllerConfigSO : ScriptableObject
{
    // ========== 移动参数 ==========
    [Header("移动参数")]
    public float BaseSpeed = 5f;
    public float SprintMultiplier = 1.6f;
    public float CrouchMultiplier = 0.5f;

    // ========== 体力参数 ==========
    [Header("体力参数")]
    public float StaminaDrainRate = 20f;        // 每秒消耗
    public float StaminaRegenRate = 15f;         // 每秒恢复
    public float StaminaRegenDelay = 2f;         // 停止冲刺后延迟恢复
    public float ExhaustionThreshold = 30f;      // 耗尽后需恢复到此阈值才能冲刺
}
```

### 11. Package 依赖

本模块依赖以下 Unity 包：
- **Unity.InputSystem** (1.9.0+) — 统一处理键鼠和手柄输入

---

## Alternatives Considered

### Alternative 1: 角色控制器完全托管给 Unity CharacterController

- **描述**：使用 Unity 自带的 CharacterController 而不自己实现移动逻辑
- **Pros**：实现简单，Unity 帮你处理碰撞
- **Cons**：
  - 无法精确控制冲刺/潜行的速度乘数
  - 头顶碰撞检测需要额外处理
- **拒绝理由**：
  - 需要精确控制速度、噪声半径、体力消耗的协同
  - 自定义状态机更灵活

### Alternative 2: 使用自定义 InputHandler 作为输入处理核心

- **描述**：直接使用 Unity 的 Input System Package，无统一抽象层
- **Pros**：实现简单
- **Cons**：
  - 输入处理分散在多个组件中
  - 平台差异处理重复
  - 输入重映射难以统一
- **选择理由（已废弃）**：
  - **已被 [ADR-0020](./adr-0020-input-system-architecture.md) 的 InputManager 统一抽象层取代**
  - ADR-0020 提供了完整的输入抽象、仲裁和重映射支持

---

## Consequences

### Positive

- **状态清晰**：移动状态机覆盖所有移动场景
- **体力约束**：冲刺不能滥用，需要策略性
- **锁定安全**：动作锁定机制防止死锁
- **噪声反馈**：NPC 能感知玩家移动状态
- **射线接口解耦**：Environment Interaction 读取而非自己检测

### Negative

- **多系统协调**：输入、移动、体力、锁定需要精心协调
- **潜行起身限制**：头顶碰撞检测增加了起身状态的复杂度
- **体力 UI**：需要显示体力槽

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **体力耗尽抖动** | 体力 0 附近反复横跳 | 引入 exhaustion 状态，必须恢复至 30% 才能冲刺 |
| **低矮空间卡死** | 头顶碰撞导致无法站起 | 始终允许蹲下；站起时检测 |
| **动作锁定死锁** | 持有锁定的系统崩溃未释放 | 5 秒超时自动释放 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 0.1ms/帧 | 移动计算简单，无复杂物理 |
| **Memory** | < 5MB | 纯逻辑系统，无额外数据 |
| **Input Latency** | < 1 帧 | ProcessInput 最早执行 |

---

## Migration Plan

### Phase 1: 基础框架
- [x] 创建 PlayerController 单例
- [x] ~~创建 InputHandler~~ — **已废弃，使用 [InputManager](./adr-0020-input-system-architecture.md)**
- [x] 创建 PlayerMovementState 枚举
- [x] 创建 MovementStateMachine 状态机

### Phase 2: 移动系统
- [ ] 实现 MovementSystem
- [ ] 实现头顶碰撞检测
- [ ] 实现面朝方向更新

### Phase 3: 体力系统
- [ ] 实现 StaminaSystem（含单例 Instance）
- [ ] 实现 exhaustion 惩罚机制
- [ ] 集成体力 UI

### Phase 4: 动作锁定
- [ ] 实现 ActionLockSystem（含超时释放机制）
- [ ] 与 Gritty Takedowns 系统确认 AcquireLock/ReleaseLock 接口契约
- [ ] 验证 Gritty Takedowns 能正确接管玩家控制权

### Phase 5: 射线和噪声
- [ ] 实现 RaycastSystem
- [ ] 实现 NoiseBroadcaster
- [ ] 订阅 PlayerMovementStateChangedEvent（供 LOS System 获取移动状态）

### Phase 6: 跨系统集成
- [ ] 与 Gritty Takedowns 集成：验证动作锁定、处决动画、状态广播
- [ ] 与 LOS System 集成：验证暴露值计算、移动状态订阅
- [ ] 与 NPC AI System 集成：验证噪声广播触发感知
- [ ] 与 Environment Interaction 集成：验证射线检测结果读取

---

## Validation Criteria

1. **8 向移动**：WASD 对角线移动速度不异常增加
2. **潜行切换**：Crouch 键正确切换站立/蹲下状态，碰撞体高度改变
3. **冲刺限制**：体力 0 时强制退出冲刺并锁定
4. **面朝保持**：停止移动时保持最后面朝方向
5. **动作锁定**：攻击期间移动输入被忽略
6. **噪声广播**：不同移动状态广播正确半径的 NoiseMadeEvent
7. **头顶碰撞**：低矮空间无法站起
8. **手柄支持**：手柄输入与键鼠行为一致

---

## CheckpointSystem

> **章节说明**：CheckpointSystem 负责管理所有重生逻辑，包括死亡重生和被捕获后重生。它与 World Map 系统（ADR-0012）紧密协作，通过 CheckpointRestoreRequestEvent 事件通知 World Map 系统恢复玩家状态。

### 1. 系统概述

CheckpointSystem 是《断绝：罪恶之源》的统一重生管理系统，负责：
- 记录玩家检查点位置
- 处理死亡后的重生逻辑
- 处理被捕（ARRESTED）后的恢复逻辑
- 向 World Map 系统发送恢复请求事件

### 2. 检查点数据结构

```csharp
// CheckpointData.cs
public struct CheckpointData
{
    public string area_id;           // 检查点所在地区 ID
    public Vector3 position;         // 检查点位置（世界坐标）
    public Vector3 facing_direction; // 检查点面朝方向
    public float timestamp;          // 检查点创建时间戳
}
```

### 3. 检查点记录时机

| 触发条件 | 描述 |
|---------|------|
| 玩家进入新地区 | AreaEnteredEvent 触发后，记录该地区的默认检查点 |
| 玩家到达安全位置 | NPC AI 系统检测到威胁消除后 |
| 剧情里程碑 | Narrative System 发送特定事件后 |
| 手动存档点 | 玩家激活存档点物件时 |

### 4. 重生恢复流程

#### 死亡恢复流程（DIED 状态）

> **重要修正 (2026-04-15)**：`NPCStateChangedEvent` 是 NPC 状态变化事件，不应用于玩家死亡。玩家死亡应使用 `PlayerDiedEvent`。

```
玩家死亡 → Health System 广播 PlayerDiedEvent
    │
    ▼
CheckpointSystem 接收事件（订阅 PlayerDiedEvent）
    │
    ▼
CheckpointSystem 发送 CheckpointRestoreRequestEvent(area_id, checkpoint_position)
    │
    ▼
World Map 系统接收事件，加载对应检查点
    │
    ▼
玩家在最近检查点重生
```

#### 被捕恢复流程（ARRESTED 状态）

> **待实现 (2026-04-15)**：`PlayerArrestedEvent` 尚未在 GrittyTakedowns 系统中定义。需在 ADR-0011 中新增此事件，发布者是 GrittyTakedowns，订阅者是 CheckpointSystem。

```
玩家被制服 → Gritty Takedowns 系统判定成功
    │
    ▼
Gritty Takedowns 发布 PlayerArrestedEvent
    │
    ▼
CheckpointSystem 接收事件（订阅 PlayerArrestedEvent），记录被捕位置
    │
    ▼
CheckpointSystem 发送 CheckpointRestoreRequestEvent(area_id, checkpoint_position, arrest_location)
    │
    ▼
World Map 系统接收事件，加载最近检查点
    │
    ▼
玩家恢复至检查点位置，重新进入 ARRESTED 触发点或最近的检查点
```

### 5. 关键接口

#### CheckpointRestoreRequestEvent

**发送方**: CheckpointSystem
**订阅方**: World Map System

```csharp
CheckpointRestoreRequestEvent:
    area_id: string               // 恢复目标地区 ID
    checkpoint_position: Vector3  // 检查点位置
    arrest_location: Vector3     // 被捕/死亡位置（用于记录/统计）
```

> **注意**：World Map 系统接收到此事件后，负责加载对应场景并将玩家传送到 checkpoint_position。CheckpointSystem 本身不直接操作场景加载。

### 6. 与其他系统的关系

| 系统 | 关系 | 说明 |
|------|------|------|
| Health System | 订阅者 | 接收 **PlayerDiedEvent** 得知玩家死亡（注：NPCStateChangedEvent 用于 NPC，不适用于玩家） |
| Gritty Takedowns | 发布者 | 发布 **PlayerArrestedEvent**（待实现）通知玩家被制服 |
| World Map System | 发布目标 | 发送 CheckpointRestoreRequestEvent 触发恢复 |
| Save System | 依赖 | 检查点数据需要持久化 |

### 7. 验证标准

| ID | 标准 | 测试方法 |
|----|------|---------|
| CS-1 | 死亡后重生到检查点 | 在地区内死亡，验证重生位置为最近检查点 |
| CS-2 | 被捕后恢复 | 被敌人制服，验证恢复位置为检查点而非被捕位置 |
| CS-3 | 检查点数据持久化 | 创建检查点后存档，重新加载，验证检查点数据一致 |
| CS-4 | CheckpointRestoreRequestEvent 发送 | 死亡/被捕后，验证事件正确发送并包含正确字段 |

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — Player Controller 通过 Event Bus 广播 NoiseEvent
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — Player Controller 属于 Foundation Layer
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC AI 订阅 NoiseMadeEvent
- [ADR-0007: LOS System 架构](./adr-0007-los-system-architecture.md) — LOS System 需要 Player Controller 提供位置和移动状态
- [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) — Gritty Takedowns 使用 ActionLockSystem 接管玩家控制权
- [ADR-0020: Input System 输入系统架构](./adr-0020-input-system-architecture.md) — **InputManager 统一输入抽象层（已废弃本 ADR 中的 InputHandler）**
- [共享类型定义](./shared-types.md) — **ActionLockSystem、PlayerMovementState 等跨 ADR 类型统一定义在此**
- [共享常量定义](./shared-constants.md) — 单例模式规范等跨 ADR 约定
- [Player Controller GDD](../../design/gdd/player-controller.md) — 本 ADR 的设计依据
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 事件定义的权威文档（PlayerMovementStateChangedEvent、NoiseEvent 等）

---

## 附录 E：废弃的 InputHandler 实现

> **废弃日期**：2026-04-15
> **废弃原因**：统一使用 [ADR-0020](./adr-0020-input-system-architecture.md) 的 `InputManager` 替代。
> **保留目的**：历史参考，不应用于新实现。

```csharp
// InputHandler.cs
// [已废弃] 请使用 InputManager（见 ADR-0020）
using UnityEngine.InputSystem;

public class InputHandler : MonoBehaviour
{
    // 输入数据结构
    public struct PlayerInput
    {
        public Vector2 MoveDirection;   // WASD / 左摇杆
        public bool SprintHeld;          // 冲刺键按住
        public bool CrouchToggled;       // 潜行键切换
        public bool ActionPressed;       // 攻击/互动键按下
        public bool FocusPressed;        // 专注监听键按下
    }

    private InputAction _moveAction;
    private InputAction _sprintAction;
    private InputAction _crouchAction;
    private InputAction _actionAction;
    private InputAction _focusAction;

    private PlayerInput _currentInput;
    private PlayerInput _previousInput;

    private void Awake()
    {
        SetupInputActions();
    }

    private void SetupInputActions()
    {
        // 键盘 + 手柄输入统一为 InputAction
        _moveAction = new InputAction("Move", InputActionType.Value);
        _moveAction.AddBinding("<Keyboard>/w")
                   .AddBinding("<Keyboard>/a")
                   .AddBinding("<Keyboard>/s")
                   .AddBinding("<Keyboard>/d")
                   .AddBinding("<Gamepad>/leftStick");

        _sprintAction = new InputAction("Sprint", InputActionType.Button);
        _sprintAction.AddBinding("<Keyboard>/leftShift")
                     .AddBinding("<Gamepad>/leftTrigger");

        _crouchAction = new InputAction("Crouch", InputActionType.Button);
        _crouchAction.AddBinding("<Keyboard>/leftCtrl")
                    .AddBinding("<Gamepad>/buttonB");

        _actionAction = new InputAction("Action", InputActionType.Button);
        _actionAction.AddBinding("<Keyboard>/e")
                    .AddBinding("<Gamepad>/buttonA");

        _focusAction = new InputAction("Focus", InputActionType.Button);
        _focusAction.AddBinding("<Keyboard>/v")
                    .AddBinding("<Gamepad>/buttonY");
    }

    public PlayerInput GetCurrentInput()
    {
        _previousInput = _currentInput;

        Vector2 rawMove = _moveAction.ReadValue<Vector2>();

        // 对角线归一化处理：
        // - 键盘 WASD：按下多个键时 magnitude 会 > 1（如同时按 W+D = sqrt(2) ≈ 1.414）
        //   需要归一化防止对角线移动速度异常
        // - 手柄摇杆：Unity Input System 默认 deadzone 为 0.125，摇杆边缘位置 magnitude
        //   接近 1.0，这是正常的手柄响应特性，无需额外处理
        // 注意：如果手柄 deadzone 设置不是默认的 0.125，可能需要额外处理
        bool isKeyboardInput = _moveAction.activeControl?.device is Keyboard;
        if (isKeyboardInput && rawMove.magnitude > 1f)
            rawMove.Normalize();

        _currentInput = new PlayerInput
        {
            MoveDirection = rawMove,
            SprintHeld = _sprintAction.IsPressed(),
            CrouchToggled = _crouchAction.WasPressedThisFrame(),  // 边缘触发：仅在按键按下那一帧为 true，抬起后为 false，直到下次按下
            ActionPressed = _actionAction.WasPressedThisFrame(),
            FocusPressed = _focusAction.IsPressed()
        };

        return _currentInput;
    }
}
```
