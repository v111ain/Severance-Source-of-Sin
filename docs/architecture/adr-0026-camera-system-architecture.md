# ADR-0026: 相机系统 (Camera System) 架构决策

## Status
**Proposed**

## Date
2026-04-12

## Last Updated
2026-04-13

## Context

### Problem Statement

《断绝：罪恶之源》是俯视角潜行游戏，相机系统需要：
1. **俯视角跟随**：玩家移动时相机平滑跟随
2. **锁定聚焦**：处决/交互时相机聚焦目标
3. **动态缩放**：潜行时拉远，战斗时拉近
4. **屏幕震动**：打击、爆炸等事件触发相机震动
5. **过场支持**：剧情过场时接管相机控制

当前缺少系统层面的架构决策，需要定义：
- 相机状态机设计
- 相机与玩家/目标的绑定关系
- 相机震动特效接口
- 与其他 Presentation Layer 系统的协作

### Constraints

- **引擎约束**：Unity 6.3 LTS，使用 Unity Camera System
- **视角约束**：俯视角固定角度（约 45°-60°），不可旋转
- **平台约束**：PC & PS5，支持手柄操作
- **性能约束**：相机更新 < 0.5ms/帧

### Requirements

- **必须**：定义 CameraState 状态机（Follow/LockOn/Cinematic/Shake）
- **必须**：定义相机跟随参数（平滑度、偏移、缩放）
- **必须**：定义相机震动接口（支持多系统触发）
- **必须**：定义相机与 ScreenEffects 的协作机制
- **必须**：遵循 ADR-0003 分层（Camera System 属于 Presentation Layer）

---

## Decision

### 架构决策

采用**状态机驱动相机 + 震动叠加层**架构：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      相机系统架构图                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  CameraStateMachine                                          │   │
│  │                                                              │   │
│  │  [Follow] ──检测到交互目标──▶ [LockOn]                       │   │
│  │      ▲                               │                       │   │
│  │      │                         [完成/被打断]                 │   │
│  │      └───────────────────────────────┘                       │   │
│  │                                                              │   │
│  │  [Any] ──触发过场──▶ [Cinematic]                             │   │
│  │      ▲                                                          │   │
│  │      └────────────────────[过场结束]─────────────────────────│   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  CameraRig (相机本体)                                        │   │
│  │  - Unity Camera                                             │   │
│  │  - Position: FollowBehavior / LockOnBehavior / Cinematic     │   │
│  │  - Rotation: LookAtTarget                                     │   │
│  │  - FOV: DynamicZoom                                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  CameraShakeLayer (震动叠加层 - 非独占)                      │   │
│  │  - 震动模式：Impact/Rumble/Explosion                         │   │
│  │  - 震动强度、持续时间、衰减曲线                               │   │
│  │  - 多震动叠加处理                                            │   │
│  │  - 与 CameraStateMachine 并行运行，叠加在位置之上            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

> **设计说明**：震动（Shake）是叠加在相机位置上的效果，而非独占状态。
> 这样可以在 Follow 或 LockOn 状态下同时触发震动，例如：
> - 潜行击杀时（Follow + 轻微震动）
> - 锁定目标时爆炸（LockOn + 爆炸震动）

### 1. CameraState 定义

```csharp
// CameraState.cs
/// <summary>
/// 相机状态枚举
/// 注意：震动（Shake）不是独占状态，而是叠加在位置上的效果
/// </summary>
public enum CameraState
{
    /// <summary>跟随状态：跟随玩家移动</summary>
    Follow,

    /// <summary>锁定状态：聚焦交互目标（处决、对话）</summary>
    LockOn,

    /// <summary>过场状态：过场动画控制</summary>
    Cinematic

    // 注意：震动（Shake）由独立的 CameraShakeManager 处理，不参与状态机
}

// CameraStateMachine.cs
/// <summary>
/// 相机状态机
/// 注意：震动（Shake）由独立的 CameraShakeManager 处理，不参与状态机
/// </summary>
[System.Serializable]
public class CameraStateMachine
{
    private CameraState _currentState;
    private CameraState _previousState;

    // P2 修复：保存 Camera 和 Tuning 引用，用于在 CreateBehavior 中通过构造函数注入
    private Camera _camera;
    private CameraTuningSO _tuning;

    /// <summary>
    /// 初始化状态机，注入相机和调参配置（由 CameraManager.Start 调用）
    /// </summary>
    public void Initialize(Camera camera, CameraTuningSO tuning)
    {
        _camera = camera;
        _tuning = tuning;
    }

    public CameraState CurrentState => _currentState;
    public CameraState PreviousState => _previousState;

    /// <summary>
    /// 当前行为引用（供 CameraManager 访问）
    /// </summary>
    public ICameraBehavior CurrentBehavior => _behavior;

    private ICameraBehavior _behavior;

    /// <summary>
    /// 切换相机状态
    /// </summary>
    public void TransitionTo(CameraState newState)
    {
        if (_currentState == newState) return;

        _previousState = _currentState;
        _currentState = newState;

        // 退出旧状态
        _behavior?.OnStateExit();

        // 进入新状态
        _behavior = CreateBehavior(newState);
        _behavior?.OnStateEnter(_previousState);

        OnStateChanged?.Invoke(_previousState, _currentState);
    }

    // Behavior 对象池：避免频繁 new/delete 造成的 GC 压力
    // CameraState 只有 3 个状态，池化后始终保持 3 个实例复用
    private Dictionary<CameraState, ICameraBehavior> _behaviorPool = new();

    private ICameraBehavior CreateBehavior(CameraState state)
    {
        // 优先从池中获取已有实例
        if (_behaviorPool.TryGetValue(state, out var behavior))
        {
            // 复用前先重置状态
            if (behavior is FollowCameraBehavior follow)
            {
                follow.Reset();
            }
            else if (behavior is LockOnCameraBehavior lockOn)
            {
                lockOn.Reset();
            }
            else if (behavior is CinematicCameraBehavior cinematic)
            {
                cinematic.Reset();
            }
            return behavior;
        }

        // 池中不存在，创建新实例并加入池
        // P2 修复：通过构造函数注入 Camera 和 Tuning，
        // 避免 [SerializeField] 字段在 new 实例后永远为 null 的问题
        behavior = state switch
        {
            CameraState.Follow => new FollowCameraBehavior(_camera, _tuning),
            CameraState.LockOn => new LockOnCameraBehavior(_camera, _tuning),
            CameraState.Cinematic => new CinematicCameraBehavior(_camera),
            _ => throw new System.ArgumentException($"[CameraStateMachine] Unknown state: {state}")
        };
        _behaviorPool[state] = behavior;
        return behavior;
    }

    /// <summary>
    /// 每帧更新相机行为
    /// </summary>
    public void Update()
    {
        _behavior?.Update();
    }

    public event Action<CameraState, CameraState> OnStateChanged;
}
```

### 1.5. CameraManager 主管理器

```csharp
// CameraManager.cs
/// <summary>
/// 相机系统主管理器
/// 协调 CameraStateMachine 和 CameraShakeManager，负责相机更新顺序和事件分发
/// </summary>
public class CameraManager : MonoBehaviour
{
    public static CameraManager Instance { get; private set; }

    [SerializeField] private Camera _camera;
    [SerializeField] private CameraStateMachine _stateMachine;
    [SerializeField] private CameraShakeManager _shakeManager;
    [SerializeField] private CameraTuningSO _tuning;
    [SerializeField] private CameraBoundary _boundary;
    [SerializeField] private CameraBoundaryHandler _boundaryHandler;

    /// <summary>
    /// 相机引用（供外部系统使用）
    /// </summary>
    public Camera Camera => _camera;

    private void Awake()
    {
        if (Instance != null)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;

        // CameraManager 和 CameraShakeManager 都设置 DontDestroyOnLoad，保持生命周期一致。
        // 若其中一个跨场景存活而另一个被销毁，CameraManager.ApplyShakeToCamera 会访问悬空引用。
        // 两者必须同步 DontDestroyOnLoad 状态，或同步不使用（在每个场景单独挂载）。
        DontDestroyOnLoad(gameObject);
    }

    private void Start()
    {
        // [已修复] 订阅玩家位置更新事件（由 PlayerController 在每帧 Update 结束时发布）
        // PlayerPositionUpdatedEvent 用于 LockOnCameraBehavior 计算相机中点
        EventBus.Instance.Subscribe<PlayerPositionUpdatedEvent>(OnPlayerPositionUpdated);

        // [已修复] 订阅 ExecutionCameraRequestEvent（由动画系统在处决动画前发送）
        // GrittyTakedowns 在播放处决动画前发布此事件，请求相机切换到 LockOn 状态
        EventBus.Instance.Subscribe<ExecutionCameraRequestEvent>(OnExecutionCameraRequest);

        // P2 修复：先注入 Camera 和 Tuning，再初始化状态机（确保 CreateBehavior 构造函数参数有效）
        _stateMachine.Initialize(_camera, _tuning);

        // 初始化边界处理器（如果配置了边界）
        if (_boundary != null)
        {
            _boundaryHandler.Initialize(_boundary);
        }

        // 必须先 TransitionTo(Follow) 创建 FollowCameraBehavior 实例，
        // 才能通过 CurrentBehavior 拿到有效引用。
        // 原代码在 TransitionTo 之前调用 CurrentBehavior（此时为 null），
        // 导致 SetBoundaryHandler 永远不会被执行。
        _stateMachine.TransitionTo(CameraState.Follow);

        // TransitionTo 已创建 FollowCameraBehavior，此时 CurrentBehavior 有效
        var followBehavior = _stateMachine.CurrentBehavior as FollowCameraBehavior;
        followBehavior?.SetBoundaryHandler(_boundaryHandler);

        // 订阅 CinematicCameraBehavior 的完成事件，以便过场结束后自动切回 Follow
        // 注意：此事件在 Cinematic 状态下才需要订阅，Follow/LockOn 状态下 behavior 是其他类型
        if (_stateMachine.CurrentBehavior is CinematicCameraBehavior cinematic)
        {
            cinematic.OnCinematicComplete -= OnCinematicCompleteHandler;
            cinematic.OnCinematicComplete += OnCinematicCompleteHandler;
        }
    }

    /// <summary>
    /// 过场完成处理：自动切回 Follow 状态
    /// </summary>
    private void OnCinematicCompleteHandler()
    {
        _stateMachine.TransitionTo(CameraState.Follow);
    }

    private void OnDestroy()
    {
        EventBus.Instance.Unsubscribe<PlayerPositionUpdatedEvent>(OnPlayerPositionUpdated);
        EventBus.Instance.Unsubscribe<ExecutionCameraRequestEvent>(OnExecutionCameraRequest);

        // 确保 LockOnCameraBehavior 的事件订阅被正确清理
        if (_stateMachine.CurrentBehavior is IDisposable disposableBehavior)
        {
            disposableBehavior.Dispose();
        }

        if (Instance == this) Instance = null;
    }

    /// <summary>
    /// [已修复] 处理 PlayerPositionUpdatedEvent 事件
    /// 供 LockOnCameraBehavior 计算相机中点
    /// </summary>
    private void OnPlayerPositionUpdated(PlayerPositionUpdatedEvent evt)
    {
        var behavior = _stateMachine.CurrentBehavior;
        if (behavior is LockOnCameraBehavior lockOn)
        {
            lockOn.UpdatePlayerPosition(evt.position);
        }
    }

    /// <summary>
    /// [已修复] 处理 ExecutionCameraRequestEvent 事件
    /// 由动画系统在处决动画前发送，请求相机切换到 LockOn 状态
    /// </summary>
    private void OnExecutionCameraRequest(ExecutionCameraRequestEvent evt)
    {
        if (_stateMachine.CurrentState == CameraState.LockOn)
        {
            // 已处于 LockOn，无需重复设置
            return;
        }

        // 请求切换到 LockOn 状态
        _stateMachine.TransitionTo(CameraState.LockOn);
        var lockOn = _stateMachine.CurrentBehavior as LockOnCameraBehavior;
        lockOn?.SetTarget(evt.target, evt.expectedDuration);

        // [已修复] 相机 LockOn 状态自动提升 GrittyTakedowns 的输入优先级
        // 当相机进入 LockOn 状态时，GrittyTakedowns 应该已经调用 SetActiveConsumer(GrittyTakedowns, 30)
        // InputArbitrator 的优先级机制确保 GrittyTakedowns 接收输入
    }

    private void Update()
    {
        // 更新相机行为
        _stateMachine.Update();
    }

    private void LateUpdate()
    {
        // 应用震动叠加（在所有 Update 之后）
        // 注意：CameraManager.LateUpdate 必须是相机更新的最后一步。
        // 所有其他相机位置修改应在各自的 Update 中完成，不要在 LateUpdate 中修改相机位置，
        // 否则会与震动叠加冲突（震动偏移会被覆盖）。
        _shakeManager.UpdateShake();
        ApplyShakeToCamera();
    }

    /// <summary>
    /// 应用震动偏移到相机
    /// </summary>
    /// <remarks>
    /// **警告**：此方法在 LateUpdate 中调用，直接修改 camera.transform.position 和 rotation。
    /// 其他相机位置修改必须通过 CameraManager.RequestShake() 而非直接修改 transform，
    /// 否则震动偏移会被覆盖。建议所有相机修改都通过 CameraManager 协调。
    ///
    /// **正确用法**：
    /// ```csharp
    /// CameraManager.Instance.RequestShake(new CameraShakeRequestEvent { ... });
    /// ```
    ///
    /// **错误用法**（会覆盖震动效果）：
    /// ```csharp
    /// CameraManager.Instance.Camera.transform.position = newPosition;  // 错误！
    /// ```
    /// </remarks>
    private void ApplyShakeToCamera()
    {
        Vector3 shakeOffset = _shakeManager.GetShakeOffset();
        Quaternion shakeRotation = _shakeManager.GetShakeRotationOffset();

        _camera.transform.position += shakeOffset;
        _camera.transform.rotation *= shakeRotation;
    }

    /// <summary>
    /// 切换相机状态
    /// </summary>
    public void TransitionTo(CameraState newState)
    {
        _stateMachine.TransitionTo(newState);

        // 切换到 Cinematic 状态时，订阅过场完成事件以便结束后自动切回 Follow
        // 先取消旧的事件订阅，防止切换时残留
        if (_stateMachine.CurrentBehavior is CinematicCameraBehavior cinematic)
        {
            cinematic.OnCinematicComplete -= OnCinematicCompleteHandler;
            cinematic.OnCinematicComplete += OnCinematicCompleteHandler;
        }
    }

    /// <summary>
    /// 请求相机震动
    /// </summary>
    public void RequestShake(CameraShakeRequestEvent request)
    {
        _shakeManager.TriggerShake(request);
    }

    /// <summary>
    /// 设置锁定目标（用于 LockOn 状态）
    /// </summary>
    /// <param name="target">锁定目标 Transform</param>
    /// <param name="duration">持续时间（秒），-1 表示无超时</param>
    /// <returns>是否成功设置锁定目标（当相机处于 LockOn 状态时返回 true）</returns>
    public bool SetLockOnTarget(Transform target, float duration = -1f)
    {
        if (_stateMachine.CurrentState != CameraState.LockOn)
        {
            Debug.LogWarning($"[CameraManager] SetLockOnTarget called but camera is not in LockOn state (current: {_stateMachine.CurrentState})");
            return false;
        }

        var behavior = _stateMachine.CurrentBehavior as LockOnCameraBehavior;
        if (behavior == null) return false;

        behavior.SetTarget(target, duration);

        // 订阅超时回调，确保 LockOn 超时后状态机自动切回 Follow
        // 使用 lambda 确保只订阅一次（SetLockOnTarget 每次调用时重新绑定）
        behavior.OnLockOnTimedOut -= OnLockOnBehaviorTimedOut;
        behavior.OnLockOnTimedOut += OnLockOnBehaviorTimedOut;
        return true;
    }

    private void OnLockOnBehaviorTimedOut()
    {
        // 取消订阅，防止 behavior 对象池复用时重复触发
        if (_stateMachine.CurrentBehavior is LockOnCameraBehavior lockOn)
        {
            lockOn.OnLockOnTimedOut -= OnLockOnBehaviorTimedOut;
        }
        _stateMachine.TransitionTo(CameraState.Follow);
    }

    /// <summary>
    /// 当前行为（用于外部查询）
    /// </summary>
    public ICameraBehavior CurrentBehavior => _stateMachine.CurrentBehavior;
}
```

### 2. 相机跟随行为

```csharp
// ICameraBehavior.cs
public interface ICameraBehavior
{
    void OnStateEnter(CameraState fromState);
    void OnStateExit();
    void Update();
}

// FollowCameraBehavior.cs
/// <summary>
/// 跟随相机行为
/// P2 修复：原设计在 Plain C# class 上使用 [SerializeField]，
/// [SerializeField] 仅对 MonoBehaviour/ScriptableObject 有效，
/// 通过 new 实例化的对象其 [SerializeField] 字段永远为 null。
/// 改为构造函数注入，由 CameraStateMachine.CreateBehavior 传入依赖。
/// </summary>
public class FollowCameraBehavior : ICameraBehavior
{
    private readonly Camera _camera;
    private readonly CameraTuningSO _tuning;
    private CameraBoundaryHandler _boundaryHandler;

    public FollowCameraBehavior(Camera camera, CameraTuningSO tuning)
    {
        _camera = camera;
        _tuning = tuning;
    }

    private Vector3 _currentOffset;
    private float _currentZoom;
    private Vector3 _velocity;

    /// <summary>
    /// 跟随目标（由 CameraManager 在 OnStateEnter 时设置，或自动从 PlayerController 获取）
    /// </summary>
    private Transform _followTarget;

    /// <summary>
    /// 是否已尝试自动获取玩家目标
    /// </summary>
    private bool _hasAutoAcquiredPlayer;

    /// <summary>
    /// 当前玩家状态（由 CameraManager 通过事件更新）
    /// </summary>
    private PlayerMovementState _playerState = PlayerMovementState.IDLE;

    /// <summary>
    /// 设置跟随目标
    /// </summary>
    public void SetFollowTarget(Transform target)
    {
        _followTarget = target;
    }

    /// <summary>
    /// 更新玩家状态（由 CameraManager 订阅事件后调用）
    /// </summary>
    public void UpdatePlayerState(PlayerMovementState state)
    {
        _playerState = state;
    }

    /// <summary>
    /// 重置状态（供对象池复用时调用）
    /// </summary>
    public void Reset()
    {
        _currentOffset = _tuning?.defaultOffset ?? Vector3.zero;
        _currentZoom = _tuning?.defaultZoom ?? 60f;
        _velocity = Vector3.zero;
        _hasAutoAcquiredPlayer = false;
    }

    /// <summary>
    /// 设置边界处理器（用于约束相机不超出场景边界）
    /// </summary>
    public void SetBoundaryHandler(CameraBoundaryHandler handler)
    {
        _boundaryHandler = handler;
    }

    public void OnStateEnter(CameraState fromState)
    {
        // 初始化偏移
        _currentOffset = _tuning.defaultOffset;
        _currentZoom = _tuning.defaultZoom;

        // 自动获取玩家目标（如果尚未设置）
        // 注意：这是安全网，确保 Follow 状态总是有目标可跟随
        if (_followTarget == null && !_hasAutoAcquiredPlayer)
        {
            TryAutoAcquirePlayer();
        }
    }

    /// <summary>
    /// 尝试自动从 PlayerController 获取玩家 Transform
    ///
    /// **⚠️ 使用场景说明 ⚠️**：
    /// 此方法是**开发期间的临时安全网**，仅用于以下场景：
    /// 1. 早期开发阶段，EventBus 系统尚未就绪
    /// 2. 独立测试 FollowCameraBehavior 时，无需完整游戏环境
    ///
    /// **生产环境应使用 EventBus 方案**：
    /// PlayerController 在 Start 时发布 PlayerRegisteredEvent（包含自己的 Transform），
    /// FollowCameraBehavior 订阅该事件获取玩家引用。
    /// 这样可以避免 Unity 查找开销，保持系统解耦。
    ///
    /// **FindGameObjectWithTag 警告**：
    /// 这是 Unity 中较慢的查找方式之一。仅在 EventBus 方案不可用时作为备用。
    /// 运行时不应调用此方法（OnStateEnter 时应已有 FollowTarget）。
    /// </summary>
    #if UNITY_EDITOR
    private void TryAutoAcquirePlayer()
    {
        // ⚠️ 仅用于早期开发或测试：EventBus 方案不可用时的备用
        // 生产代码应通过 EventBus 订阅 PlayerRegisteredEvent 获取玩家引用
        var player = GameObject.FindGameObjectWithTag("Player");
        if (player != null)
        {
            _followTarget = player.transform;
            _hasAutoAcquiredPlayer = true;
            Debug.Log($"[FollowCameraBehavior] Auto-acquired player via fallback method: {player.name}. " +
                "Consider using EventBus PlayerRegisteredEvent for production.");
        }
        else
        {
            Debug.LogError("[FollowCameraBehavior] Could not auto-acquire player. " +
                "Call SetFollowTarget() manually or ensure Player has 'Player' tag.");
        }
    }
    #endif

    public void OnStateExit()
    {
        // 保存当前偏移用于恢复
    }

    public void Update()
    {
        if (_followTarget == null) return;

        // 计算目标位置
        Vector3 targetPosition = _followTarget.position + _currentOffset;

        // 平滑跟随（使用 SmoothDamp）
        Vector3 smoothPosition = Vector3.SmoothDamp(
            _camera.transform.position,
            targetPosition,
            ref _velocity,
            _tuning.followSmoothTime);

        // 应用边界约束
        smoothPosition = _boundaryHandler?.ConstrainPosition(smoothPosition, _camera) ?? smoothPosition;

        _camera.transform.position = smoothPosition;

        // 始终注视玩家
        _camera.transform.LookAt(_followTarget);

        // 更新缩放（基于玩家移动状态）
        UpdateZoom();
    }

    private void UpdateZoom()
    {
        // 潜行时拉远，战斗时拉近
        // 枚举值遵循 shared-types.md §7.2 全大写命名规范（与 ADR-0024 AnimationStateMapper 一致）
        float targetZoom = _playerState switch
        {
            PlayerMovementState.CROUCH => _tuning.stealthZoom,      // 拉远
            PlayerMovementState.SPRINT => _tuning.combatZoom,       // 拉近
            _ => _tuning.defaultZoom
        };

        _currentZoom = Mathf.Lerp(_currentZoom, targetZoom, Time.deltaTime * _tuning.zoomLerpSpeed);
        _camera.fieldOfView = Mathf.Lerp(_camera.fieldOfView, _currentZoom, Time.deltaTime * _tuning.zoomLerpSpeed);
    }
}
```

### 3. 锁定聚焦行为

```csharp
// LockOnCameraBehavior.cs
/// <summary>
/// 锁定相机行为（用于处决、对话等交互）
/// P2 修复：同 FollowCameraBehavior，[SerializeField] 对 Plain C# class 无效，改为构造函数注入。
/// </summary>
public class LockOnCameraBehavior : ICameraBehavior, IDisposable
{
    private readonly Camera _camera;
    private readonly CameraTuningSO _tuning;

    public LockOnCameraBehavior(Camera camera, CameraTuningSO tuning)
    {
        _camera = camera;
        _tuning = tuning;
    }

    private Transform _target;
    private float _lockOnDuration;
    private float _elapsed;
    private Vector3 _velocity;
    private Vector3 _playerPosition;
    private bool _hasPlayerPosition;

    /// <summary>
    /// 销毁标志，防止重复释放
    /// </summary>
    private bool _disposed;

    /// <summary>
    /// 锁定指定目标
    /// </summary>
    /// <param name="target">目标 Transform</param>
    /// <param name="duration">持续时间（秒），-1 表示无超时</param>
    /// <remarks>
    /// **超时保护**：如果 PlayerPositionUpdatedEvent 从未被发送（如 PlayerController 根本没发布这个事件），
    /// 相机将永远等待。SetTarget 时会设置 _lockOnDuration，如果 duration > 0，则超时后自动切换回 Follow 状态。
    /// </remarks>
    public void SetTarget(Transform target, float duration = -1f)
    {
        _target = target;
        _lockOnDuration = duration;
        _elapsed = 0f;
    }

    public void OnStateEnter(CameraState fromState)
    {
        // 尝试从交互系统获取目标
        // 注意：此处使用事件机制，交互目标通过 SetTarget 方法设置
        _elapsed = 0f;
        _hasPlayerPosition = false;

        // 注意：不在此订阅 PlayerPositionUpdatedEvent。
        // CameraManager 已在 Start() 中统一订阅，并通过 UpdatePlayerPosition() 转发给本类。
        // 若此处也订阅，会造成双重处理（CameraManager 转发一次 + 本类直接处理一次）。
    }

    public void OnStateExit()
    {
        _target = null;
        _hasPlayerPosition = false;
    }

    /// <summary>
    /// 实现 IDisposable（保留接口兼容性，当前无需清理事件订阅）
    /// CameraManager 销毁时调用
    /// </summary>
    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        // 事件订阅由 CameraManager 统一管理，LockOnCameraBehavior 不直接订阅 EventBus
    }

    /// <summary>
    /// 重置状态（供对象池复用时调用）
    /// </summary>
    public void Reset()
    {
        _target = null;
        _lockOnDuration = -1f;
        _elapsed = 0f;
        _velocity = Vector3.zero;
        _hasPlayerPosition = false;
        _disposed = false;
    }

    /// <summary>
    /// 更新玩家位置（由 CameraManager 从 PlayerPositionUpdatedEvent 转发调用）
    /// </summary>
    public void UpdatePlayerPosition(Vector3 position)
    {
        _playerPosition = position;
        _hasPlayerPosition = true;
    }

    public void Update()
    {
        if (_target == null) return;

        _elapsed += Time.deltaTime;

        // 检查超时
        if (_lockOnDuration > 0 && _elapsed >= _lockOnDuration)
        {
            // 通知状态机切换回 Follow
            OnLockOnTimedOut?.Invoke();
            // ⚠️ 注意：OnLockOnEnded 在超时时会与 OnLockOnTimedOut 同时触发
            // 这是因为 OnLockOnEnded 表示"锁定以任何原因结束"，超时是结束原因之一
            // 调用方应使用 OnLockOnTimedOut 区分超时和其他结束原因
            OnLockOnEnded?.Invoke();
            return;
        }

        // 检查是否获取到玩家位置
        if (!_hasPlayerPosition)
        {
            return; // 等待玩家位置更新
        }

        // 计算相机位置：在玩家和目标之间，以一定比例偏移
        Vector3 midpoint = Vector3.Lerp(_playerPosition, _target.position, _tuning.lockOnTargetBias);

        // 拉高偏移（俯视角）
        Vector3 offset = new Vector3(0, _tuning.lockOnHeightOffset, -_tuning.lockOnDepthOffset);
        Vector3 targetPosition = midpoint + offset;

        // 平滑移动到目标位置
        _camera.transform.position = Vector3.SmoothDamp(
            _camera.transform.position,
            targetPosition,
            ref _velocity,
            _tuning.lockOnSmoothTime);

        // 注视目标
        _camera.transform.LookAt(_target.position + Vector3.up * 0.5f);
    }

    /// <summary>
    /// 锁定超时回调
    /// 仅在锁定因超时（_lockOnDuration > 0 且时间到达）结束时触发
    ///
    /// **使用场景**：
    /// - 处决动画超时自动取消锁定
    /// - 交互超时自动返回跟随状态
    /// </summary>
    public event Action OnLockOnTimedOut;

    /// <summary>
    /// 锁定结束回调（任何原因导致锁定结束时触发）
    ///
    /// **触发时机**：
    /// - 超时时：与 OnLockOnTimedOut 同时触发
    /// - 外部调用 EndLockOn() 时触发
    /// - 目标被销毁时触发（通过 IDisposable 机制）
    ///
    /// **与 OnLockOnTimedOut 的区别**：
    /// - OnLockOnTimedOut：仅在超时原因结束时触发
    /// - OnLockOnEnded：任何结束原因都触发，包括超时、打断、手动结束
    ///
    /// **使用示例**：
    /// ```csharp
    /// // 区分处理超时和打断
    /// lockOn.OnLockOnTimedOut += () => Debug.Log("锁定超时");
    /// lockOn.OnLockOnEnded += () => Debug.Log("锁定结束（原因未知）");
    /// ```
    /// </summary>
    public event Action OnLockOnEnded;

    /// <summary>
    /// 主动结束锁定（由外部系统调用，如交互取消、处决开始）
    /// </summary>
    public void EndLockOn()
    {
        OnLockOnEnded?.Invoke();
    }

    /// <summary>
    /// 检查锁定是否超时
    /// </summary>
    public bool IsTimedOut => _lockOnDuration > 0 && _elapsed >= _lockOnDuration;
}
```

### 4. 相机震动管理器（叠加层）

```csharp
// CameraShakeManager.cs
/// <summary>
/// 相机震动效果管理器（叠加层）
/// 震动是叠加在相机位置上的效果，而非独占状态
/// 可以与 Follow/LockOn/Cinematic 状态并行运行
/// </summary>
public class CameraShakeManager : MonoBehaviour
{
    public static CameraShakeManager Instance { get; private set; }

    [SerializeField] private Camera _camera;
    [SerializeField] private CameraShakeTuningSO _tuning;

    private List<ActiveShake> _activeShakes = new();

    /// <summary>
    /// 震动基础位置（预留用于震动前保存状态，当前版本未使用）
    /// 保留字段以便未来支持更复杂的震动偏移计算
    /// </summary>
    private Vector3 _basePosition;
    private Quaternion _baseRotation;

    /// <summary>
    /// 震动叠加层位置偏移（由 CameraManager 在 LateUpdate 中读取并叠加）
    /// 通过方法暴露，防止外部直接修改
    /// </summary>
    private Vector3 _shakeOffset;

    /// <summary>
    /// 震动叠加层旋转偏移
    /// </summary>
    private Quaternion _shakeRotationOffset;

    /// <summary>
    /// 获取震动位置偏移（只读）
    /// </summary>
    public Vector3 GetShakeOffset() => _shakeOffset;

    /// <summary>
    /// 获取震动旋转偏移（只读）
    /// </summary>
    public Quaternion GetShakeRotationOffset() => _shakeRotationOffset;

    private void Awake()
    {
        if (Instance != null)
        {
            Debug.LogWarning("[CameraShakeManager] Duplicate instance detected, destroying. " +
                "Ensure only one CameraShakeManager exists in the scene.");
            Destroy(gameObject);
            return;
        }
        Instance = this;
        DontDestroyOnLoad(gameObject);  // 保持跨场景存在

        // 订阅 CameraShakeRequestEvent 事件（通过 EventBus 触发震动）
        EventBus.Instance.Subscribe<CameraShakeRequestEvent>(OnCameraShakeRequested);
    }

    private void OnDestroy()
    {
        EventBus.Instance.Unsubscribe<CameraShakeRequestEvent>(OnCameraShakeRequested);
    }

    /// <summary>
    /// CameraShakeRequestEvent 事件处理（由 EventBus 调用）
    /// </summary>
    private void OnCameraShakeRequested(CameraShakeRequestEvent request)
    {
        TriggerShake(request);
    }

    /// <summary>
    /// 每帧更新震动
    /// 由 CameraManager 在应用位置后调用
    /// </summary>
    public void UpdateShake()
    {
        // 计算所有震动叠加
        Vector3 shakeOffset = Vector3.zero;
        Quaternion shakeRotation = Quaternion.identity;
        float totalIntensity = 0f;

        foreach (var shake in _activeShakes)
        {
            shake.Elapsed += Time.deltaTime;
            if (shake.Elapsed >= shake.Duration)
            {
                shake.MarkComplete = true;
                continue;
            }

            float progress = shake.Elapsed / shake.Duration;
            float intensity = shake.Intensity * (1f - progress) * _tuning.globalIntensityScale;

            // 累加总强度，用于多震动叠加上限检查
            totalIntensity += intensity;

            shakeOffset += CalculateShakeOffset(shake.Mode, intensity, shake.Elapsed);
            shakeRotation *= CalculateShakeRotation(shake.Mode, intensity);
        }

        // 多震动叠加时应用上限，防止效果过强
        if (totalIntensity > _tuning.maxShakeIntensity)
        {
            float scale = _tuning.maxShakeIntensity / totalIntensity;
            shakeOffset *= scale;
        }

        // 移除完成的震动
        _activeShakes.RemoveAll(s => s.MarkComplete);

        // 更新叠加偏移
        _shakeOffset = shakeOffset;
        _shakeRotationOffset = shakeRotation;
    }

    /// <summary>
    /// 触发相机震动（由其他系统通过 EventBus 调用）
    /// </summary>
    public void TriggerShake(CameraShakeRequestEvent request)
    {
        _activeShakes.Add(new ActiveShake
        {
            mode = request.mode,
            Intensity = request.Intensity,
            Duration = request.Duration,
            Elapsed = 0f
        });
    }

    private Vector3 CalculateShakeOffset(ShakeMode mode, float intensity, float elapsed)
    {
        return mode switch
        {
            ShakeMode.Impact => new Vector3(
                Random.Range(-1f, 1f) * intensity * _tuning.impactAmplitudeX,
                Random.Range(-1f, 1f) * intensity * _tuning.impactAmplitudeY,
                0f),

            ShakeMode.Rumble =>
                // 使用 shake.Elapsed 而非 Time.time，确保多个 Rumble 震动独立计算
                // 避免 Time.time 全局时钟导致的震动同步问题
                new Vector3(
                    Mathf.Sin(elapsed * _tuning.rumbleFrequency) * intensity * _tuning.rumbleAmplitudeX,
                    Mathf.Sin(elapsed * _tuning.rumbleFrequency * 1.3f) * intensity * _tuning.rumbleAmplitudeY,
                    0f),

            ShakeMode.Explosion => new Vector3(
                Random.Range(-1f, 1f) * intensity * _tuning.explosionAmplitudeX,
                Random.Range(-1f, 1f) * intensity * _tuning.explosionAmplitudeY,
                0f),

            _ => Vector3.zero
        };
    }

    private Quaternion CalculateShakeRotation(ShakeMode mode, float intensity)
    {
        // 轻微旋转震动（用于爆炸等大型冲击）
        if (mode == ShakeMode.Explosion)
        {
            float roll = Random.Range(-1f, 1f) * intensity * _tuning.explosionRollAmplitude;
            return Quaternion.Euler(0f, 0f, roll);
        }
        return Quaternion.identity;
    }

    private class ActiveShake
    {
        public ShakeMode Mode;
        public float Intensity;
        public float Duration;
        public float Elapsed;
        public bool MarkComplete;
    }
}

> **CameraShakeManager 职责边界说明**：CameraShakeManager 专负责**物理震动**（影响相机 Transform 位置，如爆炸冲击导致画面晃动）。**准星/UI 抖动**（影响 HUD 准星 Transform）由 ScreenEffectsManager 的 UI_Jitter 效果处理（见 ADR-0023 §6）。
>
> **效果选择指南**：
> - 爆炸/撞击/处决命中 → CameraShakeRequestEvent → CameraShakeManager
> - Rage >= 50 准星抖动 → ScreenEffectRequestEvent(effectType=UI_Jitter) → ScreenEffectsManager
>
> 两者独立运作，不会重复触发。

// CameraShakeRequestEvent.cs
/// <summary>
/// 相机震动请求事件
/// 由其他系统通过 EventBus 发布，CameraShakeManager 订阅处理
/// 定义位置：shared-types.md §22（权威定义）
/// </summary>
public struct CameraShakeRequestEvent
{
    /// <summary>
    /// 震动类型（用于震动曲线选择）
    /// </summary>
    public CameraShakeType shake_type;

    /// <summary>
    /// 震动模式（影响震动衰减方式）
    /// </summary>
    public ShakeMode mode;

    /// <summary>
    /// 震动强度（0-1）
    /// </summary>
    public float Intensity;

    /// <summary>
    /// 震动持续时间（秒）
    /// </summary>
    public float Duration;
}

/// <summary>
/// 相机震动类型枚举
/// </summary>
public enum CameraShakeType
{
    /// <summary>爆炸冲击：强烈的一次性震动</summary>
    Explosion,

    /// <summary>处决震动：中等强度的平滑震动</summary>
    Execute,

    /// <summary>战斗震动：持续的轻微晃动</summary>
    Combat
}

public enum ShakeMode
{
    /// <summary>冲击震动：单次强烈震动（处决命中）</summary>
    Impact,

    /// <summary>持续震动：低频 rumble（引擎噪音）</summary>
    Rumble,

    /// <summary>爆炸震动：强烈衰减震动（爆炸）</summary>
    Explosion
}

> **CameraShake 职责边界说明**：CameraShake 专负责**物理位移震动**（影响相机 Transform 位置），例如爆炸冲击导致画面整体晃动。**准星抖动**属于 UI 层，由 ScreenEffect.Shake 处理（见 ADR-0023 §6）。

### 4.5. 相机边界约束

```csharp
// CameraBoundary.cs
/// <summary>
/// 相机边界约束
/// 防止相机跟随玩家超出场景边界
/// </summary>
/// **⚠️ 设计假设说明**：
/// 本实现假设相机为**固定俯视角**（约 45°-60°，朝向地面），
/// 相机只会在 XZ 平面移动，Y 轴高度由相机 offset 固定。
///
/// 如果将来需要支持**自由视角**或**旋转相机**：
/// 1. 当前硬边界/软边界计算需要重新设计（基于视锥而非固定平面）
/// 2. CameraBoundary 需要扩展支持 3D 边界框（AABB 或 OBB）
/// 3. ConstrainPosition 方法需要改为基于相机朝向计算
[CreateAssetMenu(menuName = "Game/Camera/Boundary")]
public class CameraBoundary : ScriptableObject
{
    [Header("边界范围")]
    public Vector2 minBounds = new Vector2(-50f, -50f);
    public Vector2 maxBounds = new Vector2(50f, 50f);

    [Header("软边界参数")]
    [Tooltip("相机超出边界多少时开始减速（屏幕边缘宽度）")]
    public float softBoundaryWidth = 5f;

    [Tooltip("相机在软边界内的最大偏移")]
    public float maxSoftOffset = 3f;

    /// <summary>
    /// 约束相机位置在边界内
    /// </summary>
    /// <param name="desiredPosition">期望的相机位置（未约束）</param>
    /// <param name="cameraWidth">相机视锥宽度（用于软边界计算）</param>
    /// <returns>约束后的相机位置</returns>
    public Vector3 ConstrainPosition(Vector3 desiredPosition, float cameraWidth)
    {
        // 俯视角相机的 XZ 平面约束（Y 由相机高度固定）
        float halfCameraWidth = cameraWidth * 0.5f;

        float minX = minBounds.x + halfCameraWidth;
        float maxX = maxBounds.x - halfCameraWidth;
        float minZ = minBounds.y + halfCameraWidth;
        float maxZ = maxBounds.y - halfCameraWidth;

        // 硬边界：直接限制在范围内
        float constrainedX = Mathf.Clamp(desiredPosition.x, minX, maxX);
        float constrainedZ = Mathf.Clamp(desiredPosition.z, minZ, maxZ);

        // 如果在软边界内，应用额外平滑
        Vector3 constrained = new Vector3(constrainedX, desiredPosition.y, constrainedZ);

        // 计算软边界偏移
        float softOffsetX = CalculateSoftOffset(desiredPosition.x, minX, maxX, halfCameraWidth);
        float softOffsetZ = CalculateSoftOffset(desiredPosition.z, minZ, maxZ, halfCameraWidth);

        constrained.x += softOffsetX;
        constrained.z += softOffsetZ;

        return constrained;
    }

    private float CalculateSoftOffset(float position, float minBound, float maxBound, float halfCameraWidth)
    {
        float softMin = minBound + softBoundaryWidth;
        float softMax = maxBound - softBoundaryWidth;

        // 左侧软边界
        if (position < softMin)
        {
            float t = 1f - (position - minBound) / (softMin - minBound);
            return -maxSoftOffset * (1f - t) * (1f - t);
        }

        // 右侧软边界
        if (position > softMax)
        {
            float t = 1f - (maxBound - position) / (maxBound - softMax);
            return maxSoftOffset * (1f - t) * (1f - t);
        }

        return 0f;
    }
}

// CameraBoundaryHandler.cs
/// <summary>
/// 相机边界处理器
/// 在 FollowCameraBehavior 更新后调用，确保相机位置在边界内
/// </summary>
public class CameraBoundaryHandler
{
    private CameraBoundary _boundary;

    public void Initialize(CameraBoundary boundary)
    {
        _boundary = boundary;
    }

    /// <summary>
    /// 约束相机位置
    /// </summary>
    /// <param name="desiredPosition">期望位置（跟随计算后的位置）</param>
    /// <param name="camera">相机引用（用于计算视锥宽度）</param>
    /// <returns>约束后的位置</returns>
    public Vector3 ConstrainPosition(Vector3 desiredPosition, Camera camera)
    {
        if (_boundary == null) return desiredPosition;

        // 计算相机视锥宽度（基于 FOV 和 aspect ratio）
        float halfHeight = camera.nearClipPlane * Mathf.Tan(camera.fieldOfView * 0.5f * Mathf.Deg2Rad);
        float cameraWidth = halfHeight * camera.aspect * 2f;

        return _boundary.ConstrainPosition(desiredPosition, cameraWidth);
    }
}
```

> **设计说明**：相机边界约束使用软边界（Soft Boundary）设计，当玩家接近边界时，
> 相机开始减速而非突然停止，提供更自然的体验。软边界宽度和最大偏移可通过
> `CameraBoundary` ScriptableObject 配置。

### 5. 过场相机控制

```csharp
// CinematicCameraBehavior.cs
/// <summary>
/// 过场相机行为
/// 由过场系统接管，支持路径动画和脚本控制
/// P2 修复：[SerializeField] 对 Plain C# class 无效，Camera 改为构造函数注入。
/// CinematicPath 通过 SetPath() 方法在运行时由场景/过场系统动态设置。
/// </summary>
public class CinematicCameraBehavior : ICameraBehavior
{
    private readonly Camera _camera;
    private CinematicPath _path;

    public CinematicCameraBehavior(Camera camera)
    {
        _camera = camera;
    }

    /// <summary>
    /// 设置过场路径（由过场系统在切换到 Cinematic 状态后调用）
    /// </summary>
    public void SetPath(CinematicPath path)
    {
        _path = path;
    }

    private float _progress;
    private bool _isPlaying;

    public void OnStateEnter(CameraState fromState)
    {
        if (_path == null)
        {
            Debug.LogWarning("[CinematicCameraBehavior] CinematicPath is not assigned. Please assign a CinematicPath in the Inspector.");
            _isPlaying = false;
            return;
        }

        _isPlaying = true;
        _progress = 0f;
        _path?.Initialize(_camera);
    }

    public void OnStateExit()
    {
        _isPlaying = false;
        _path?.Stop();
    }

    public void Update()
    {
        if (!_isPlaying || _path == null) return;

        _progress += Time.deltaTime / _path.Duration;
        _progress = Mathf.Clamp01(_progress);

        _path.UpdateAt(_progress);
        _path.ApplyToCamera(_camera);

        if (_progress >= 1f)
        {
            _isPlaying = false;
            _path.Stop();
            OnCinematicComplete?.Invoke();
        }
    }

    /// <summary>
    /// 过场完成回调
    /// </summary>
    public event Action OnCinematicComplete;

    /// <summary>
    /// 跳过过场
    /// </summary>
    public void Skip()
    {
        _progress = 1f;
        _isPlaying = false;
    }

    /// <summary>
    /// 重置状态（供对象池复用时调用）
    /// </summary>
    public void Reset()
    {
        _progress = 0f;
        _isPlaying = false;
        _path = null;
    }
}
```

### 5.5 CinematicPath 过场路径

```csharp
// CinematicPath.cs
/// <summary>
/// 过场相机路径
/// 支持位置和旋转的关键帧插值，支持多种曲线插值模式
/// </summary>
[Serializable]
public class CinematicPath
{
    /// <summary>
    /// 插值模式
    /// </summary>
    public enum InterpolationMode
    {
        /// <summary>线性插值（Lerp/Slerp）</summary>
        Linear,
        /// <summary>Catmull-Rom 样条插值（平滑曲线）</summary>
        CatmullRom,
        /// <summary>贝塞尔曲线插值（需要额外控制点）</summary>
        Bezier
    }

    [Serializable]
    public class PathKeyframe
    {
        public float time;           // 时间点（0-1）
        public Vector3 position;     // 位置
        public Quaternion rotation;   // 旋转
        public float fov;            // 视野

        /// <summary>
        /// 插值模式（覆盖全局模式）
        /// </summary>
        public InterpolationMode interpolationMode = InterpolationMode.Linear;

        /// <summary>
        /// 贝塞尔控制点（仅在 interpolationMode = Bezier 时使用）
        /// </summary>
        public Vector3 controlPoint1;  // 出点控制点
        public Vector3 controlPoint2;  // 入点控制点
    }

    public List<PathKeyframe> keyframes = new();
    /// <summary>
    /// 总时长（秒）。字段名使用 PascalCase 以与 ICameraBehavior 调用约定一致。
    /// </summary>
    public float Duration = 5f;

    /// <summary>
    /// 全局插值模式（当 PathKeyframe 未指定模式时使用）
    /// </summary>
    public InterpolationMode globalInterpolationMode = InterpolationMode.CatmullRom;

    /// <summary>
    /// 初始化路径
    /// </summary>
    public void Initialize(Camera camera)
    {
        // 初始化逻辑
    }

    /// <summary>
    /// 停止路径动画
    /// </summary>
    public void Stop()
    {
        _isPlaying = false;
    }

    /// <summary>
    /// 在指定进度更新相机（内部保存进度状态）
    /// </summary>
    /// <param name="progress">进度（0-1）</param>
    public void UpdateAt(float progress)
    {
        _currentProgress = Mathf.Clamp01(progress);
    }

    /// <summary>
    /// 将当前进度的插值结果应用到相机
    /// </summary>
    public void ApplyToCamera(Camera camera)
    {
        if (keyframes.Count < 2 || camera == null) return;

        // 查找当前关键帧和下一关键帧，同时记录段索引（Catmull-Rom 需要相邻控制点）
        PathKeyframe startKey = null;
        PathKeyframe endKey = null;
        int segIndex = -1;

        for (int i = 0; i < keyframes.Count - 1; i++)
        {
            if (_currentProgress >= keyframes[i].time && _currentProgress <= keyframes[i + 1].time)
            {
                startKey = keyframes[i];
                endKey = keyframes[i + 1];
                segIndex = i;
                break;
            }
        }

        if (startKey == null || endKey == null) return;

        float localProgress = (_currentProgress - startKey.time) / (endKey.time - startKey.time);

        // 使用关键帧指定的插值模式，或全局默认模式
        InterpolationMode mode = startKey.interpolationMode != InterpolationMode.Linear
            ? startKey.interpolationMode
            : globalInterpolationMode;

        // Catmull-Rom 需要前一个和后一个关键帧作为切线控制点：
        //   p0 = 前一段的起点（边界时退化为 startKey 自身）
        //   p3 = 后一段的终点（边界时退化为 endKey 自身）
        Vector3 prevPos = segIndex > 0
            ? keyframes[segIndex - 1].position
            : startKey.position;
        Vector3 nextPos = segIndex < keyframes.Count - 2
            ? keyframes[segIndex + 2].position
            : endKey.position;

        // 根据插值模式计算位置
        camera.transform.position = InterpolatePosition(startKey, endKey, localProgress, mode, prevPos, nextPos);

        // 旋转始终使用 Slerp（线性插值不适用于旋转）
        camera.transform.rotation = Quaternion.Slerp(startKey.rotation, endKey.rotation, localProgress);
        camera.fieldOfView = Mathf.Lerp(startKey.fov, endKey.fov, localProgress);
    }

    /// <summary>
    /// 根据插值模式计算位置
    /// </summary>
    /// <param name="prevPos">前一段起点位置（Catmull-Rom p0，边界时等于 start.position）</param>
    /// <param name="nextPos">后一段终点位置（Catmull-Rom p3，边界时等于 end.position）</param>
    private Vector3 InterpolatePosition(
        PathKeyframe start, PathKeyframe end, float t, InterpolationMode mode,
        Vector3 prevPos, Vector3 nextPos)
    {
        return mode switch
        {
            InterpolationMode.Linear =>
                Vector3.Lerp(start.position, end.position, t),

            // Catmull-Rom 四控制点：[p0=前段起, p1=当前起, p2=当前终, p3=后段终]
            // 曲线从 p1 到 p2，p0/p3 仅影响端点处的切线方向，不经过
            InterpolationMode.CatmullRom =>
                InterpolateCatmullRom(prevPos, start.position, end.position, nextPos, t),

            InterpolationMode.Bezier =>
                InterpolateBezier(start.position, start.controlPoint2, end.controlPoint1, end.position, t),

            _ => Vector3.Lerp(start.position, end.position, t)
        };
    }

    /// <summary>
    /// Catmull-Rom 样条插值
    /// </summary>
    private Vector3 InterpolateCatmullRom(Vector3 p0, Vector3 p1, Vector3 p2, Vector3 p3, float t)
    {
        // Catmull-Rom 公式
        float t2 = t * t;
        float t3 = t2 * t;

        return 0.5f * (
            (2f * p1) +
            (-p0 + p2) * t +
            (2f * p0 - 5f * p1 + 4f * p2 - p3) * t2 +
            (-p0 + 3f * p1 - 3f * p2 + p3) * t3
        );
    }

    /// <summary>
    /// 三次贝塞尔曲线插值
    /// </summary>
    private Vector3 InterpolateBezier(Vector3 p0, Vector3 p1, Vector3 p2, Vector3 p3, float t)
    {
        float u = 1f - t;
        float tt = t * t;
        float uu = u * u;
        float uuu = uu * u;
        float ttt = tt * t;

        return uuu * p0 +
               3f * uu * t * p1 +
               3f * u * tt * p2 +
               ttt * p3;
    }

    /// <summary>
    /// 是否正在播放
    /// </summary>
    public bool IsPlaying => _isPlaying;

    private float _currentProgress;
    private bool _isPlaying;
}

// CinematicSequence.cs
/// <summary>
/// 过场序列
/// 管理多个 CinematicPath 的播放顺序
/// </summary>
[CreateAssetMenu(menuName = "Game/Camera/CinematicSequence")]
public class CinematicSequence : ScriptableObject
{
    public CinematicPath[] paths;
    public bool loop;

    /// <summary>
    /// 获取路径数量
    /// </summary>
    public int PathCount => paths?.Length ?? 0;
}
```

### 6. 相机调参配置

```csharp
// CameraTuningSO.cs
[CreateAssetMenu(menuName = "Game/Camera/Tuning")]
public class CameraTuningSO : ScriptableObject
{
    [Header("跟随参数")]
    public float followSmoothTime = 0.15f;
    public Vector3 defaultOffset = new Vector3(0f, 15f, -8f);
    public float defaultZoom = 60f;

    [Header("缩放参数")]
    public float stealthZoom = 50f;        // 潜行时拉远
    public float combatZoom = 70f;         // 战斗时拉近
    public float zoomLerpSpeed = 2f;

    [Header("锁定参数")]
    public float lockOnSmoothTime = 0.3f;
    public float lockOnTargetBias = 0.3f;  // 目标在画面中的位置
    public float lockOnHeightOffset = 8f;
    public float lockOnDepthOffset = -5f;
}

// CameraShakeTuningSO.cs
[CreateAssetMenu(menuName = "Game/Camera/ShakeTuning")]
public class CameraShakeTuningSO : ScriptableObject
{
    [Header("全局设置")]
    [Tooltip("全局震动强度缩放因子。默认值 1.0 让策划直接控制震动强度，无隐式削弱")]
    [Range(0.1f, 1f)]
    public float globalIntensityScale = 1.0f;

    [Tooltip("单次震动最大强度上限，防止多震动叠加后效果过强")]
    [Range(0f, 1f)]
    public float maxShakeIntensity = 1.0f;

    [Header("Impact 震动")]
    public float impactAmplitudeX = 0.3f;
    public float impactAmplitudeY = 0.2f;

    [Header("Rumble 震动")]
    public float rumbleFrequency = 25f;
    public float rumbleAmplitudeX = 0.05f;
    public float rumbleAmplitudeY = 0.05f;

    [Header("Explosion 震动")]
    public float explosionAmplitudeX = 0.8f;
    public float explosionAmplitudeY = 0.5f;
    public float explosionRollAmplitude = 5f;
}
```

### 7. Unity 项目结构（Presentation Layer）

```
Assets/Game/
├── Presentation/
│   └── Camera/
│       ├── CameraManager.cs               # 主管理器
│       ├── CameraStateMachine.cs         # 状态机
│       ├── Behaviors/
│       │   ├── ICameraBehavior.cs        # 行为接口
│       │   ├── FollowCameraBehavior.cs   # 跟随行为
│       │   ├── LockOnCameraBehavior.cs  # 锁定行为
│       │   └── CinematicCameraBehavior.cs # 过场行为
│       ├── CameraShakeManager.cs         # 震动管理器
│       ├── Cinematic/
│       │   ├── CinematicPath.cs          # 过场路径
│       │   └── CinematicSequence.cs      # 过场序列
│       ├── Config/
│       │   ├── CameraTuningSO.cs         # 相机调参
│       │   └── CameraShakeTuningSO.cs    # 震动调参
│       └── Events/
│           ├── CameraEvents.cs           # 相机事件
│           └── CameraEventIds.cs         # 事件 ID
```

---

## Alternatives Considered

### Alternative 1: 使用 Cinemachine

- **描述**：使用 Unity Cinemachine 包替代自研相机
- **Pros**：
  - 功能完善（FreeLook、Impulse、Confiner 等）
  - 社区资源丰富
  - 与 Timeline 集成好
- **Cons**：
  - 额外包依赖
  - 高度封装，定制受限
  - 学习曲线
- **拒绝理由**：Cinemachine 功能过重，俯视角相机的简单跟随需求不值得引入完整方案

### Alternative 2: 固定相机（无跟随）

- **描述**：相机固定在场景中央，不跟随玩家
- **Pros**：实现简单
- **Cons**：
  - 玩家远离相机边缘时体验差
  - 无法支持锁定聚焦
- **拒绝理由**：俯视角潜行游戏需要跟随玩家查看周围环境

---

## Consequences

### Positive

- **状态清晰**：状态机设计易于理解和扩展
- **震动灵活**：多震动叠加，支持多种震动模式
- **过场支持**：Timeline 集成支持复杂过场

### Negative

- **调试复杂**：多状态切换需要仔细调试
- **参数多**：调参配置项较多

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **震动叠加混乱** | 多震动同时触发导致效果怪异 | 实现震动优先级和衰减机制 |
| **过场打断** | 玩家操作打断过场 | 设计过场跳过机制 |
| **平台差异** | 手柄震动 API 差异 | 抽象 HapticAdapter |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 0.5ms/帧 | 相机更新 |
| **Memory** | < 1MB | 配置和状态 |
| **GPU** | 无影响 | 无额外渲染 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 CameraStateMachine
- [ ] 创建 FollowCameraBehavior
- [ ] 创建 CameraManager 单例

### Phase 2: 锁定与震动
- [ ] 创建 LockOnCameraBehavior
- [ ] 创建 CameraShakeManager
- [ ] 定义 CameraShakeRequestEvent 事件

### Phase 3: 过场支持
- [ ] 创建 CinematicCameraBehavior
- [ ] 集成 Unity Timeline
- [ ] 定义过场路径配置

### Phase 4: 系统集成
- [ ] 与 PlayerController 集成（状态同步）
- [ ] 与 ScreenEffects 集成（震动触发）
- [ ] 与 GrittyTakedowns 集成（锁定目标）

---

## Validation Criteria

1. **跟随验证**：玩家移动时相机平滑跟随
2. **锁定验证**：处决时相机正确聚焦目标
3. **震动验证**：震动效果正确应用，支持叠加
4. **过场验证**：过场相机正确接管和释放
5. **性能验证**：相机更新 < 0.5ms/帧

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — CameraShakeRequestEvent 通过 EventBus 触发
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — **Camera System 属于 Presentation Layer**
- [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) — 锁定聚焦与处决动画同步
- [ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — 震动与屏幕特效协作
- [ADR-0024: 动画系统](./adr-0024-animation-system-architecture.md) — **动画-相机协调协议（ExecutionCameraRequestEvent）**
- [ADR-0009: 玩家控制器](./adr-0009-player-controller-architecture.md) — **PlayerMovementState 定义位置**
- [共享类型定义](./shared-types.md) — **PlayerMovementState (§7.2)**

---

## 附录：类型依赖说明

| 类型 | 定义位置 | 说明 |
|------|---------|------|
| `PlayerMovementState` | shared-types.md §7.2 | 玩家移动状态枚举 |
| `PlayerPositionUpdatedEvent` | shared-types.md §7.4 | 玩家位置更新事件 |
| `CameraState` | ADR-0026 本文档 | 相机状态枚举（不含 Shake） |
| `ShakeMode` | ADR-0026 本文档 | 震动模式枚举 |
| `CameraShakeRequestEvent` | ADR-0026 本文档 | 震动请求事件（通过 EventBus 发布） |
| `ICameraBehavior` | ADR-0026 本文档 | 相机行为接口 |
| `CinematicPath` | ADR-0026 本文档 | 过场相机路径 |
| `CinematicSequence` | ADR-0026 本文档 | 过场序列配置 |

**设计说明**：
- 震动（Shake）是叠加效果，由 `CameraShakeManager` 独立管理
- `CameraState` 不包含 Shake，因为震动不是独占状态
- `CameraManager` 协调 `CameraStateMachine` 和 `CameraShakeManager`
- `LockOnCameraBehavior` 通过 `PlayerPositionUpdatedEvent` 获取玩家位置，避免直接依赖 PlayerController
