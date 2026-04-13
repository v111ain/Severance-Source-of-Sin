# ADR-0026: 相机系统 (Camera System) 架构决策

## Status
**Proposed**

## Date
2026-04-12

## Last Updated
2026-04-12

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
public class CameraStateMachine
{
    private CameraState _currentState;
    private CameraState _previousState;

    public CameraState CurrentState => _currentState;
    public CameraState PreviousState => _previousState;

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

    private ICameraBehavior CreateBehavior(CameraState state)
    {
        return state switch
        {
            CameraState.Follow => new FollowCameraBehavior(),
            CameraState.LockOn => new LockOnCameraBehavior(),
            CameraState.Cinematic => new CinematicCameraBehavior(),
            _ => null
        };
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
/// </summary>
public class FollowCameraBehavior : ICameraBehavior
{
    [SerializeField] private Camera _camera;
    [SerializeField] private CameraTuningSO _tuning;

    private Vector3 _currentOffset;
    private float _currentZoom;
    private Vector3 _velocity;

    /// <summary>
    /// 跟随目标（由 CameraManager 在 OnStateEnter 时设置）
    /// </summary>
    private Transform _followTarget;

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

    public void OnStateEnter(CameraState fromState)
    {
        // 初始化偏移
        _currentOffset = _tuning.defaultOffset;
        _currentZoom = _tuning.defaultZoom;
    }

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

        _camera.transform.position = smoothPosition;

        // 始终注视玩家
        _camera.transform.LookAt(_followTarget);

        // 更新缩放（基于玩家移动状态）
        UpdateZoom();
    }

    private void UpdateZoom()
    {
        // 潜行时拉远，战斗时拉近
        float targetZoom = _playerState switch
        {
            PlayerMovementState.CROUCH => _tuning.stealthZoom,      // 拉远
            PlayerMovementState.SPRINT => _tuning.combatZoom,      // 拉近
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
/// </summary>
public class LockOnCameraBehavior : ICameraBehavior
{
    [SerializeField] private Camera _camera;
    [SerializeField] private CameraTuningSO _tuning;

    private Transform _target;
    private float _lockOnDuration;
    private float _elapsed;
    private Vector3 _velocity;

    /// <summary>
    /// 锁定指定目标
    /// </summary>
    /// <param name="target">目标 Transform</param>
    /// <param name="duration">持续时间（秒），-1 表示无超时</param>
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
    }

    public void OnStateExit()
    {
        _target = null;
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
            return;
        }

        // 计算相机位置：在玩家和目标之间，以一定比例偏移
        // 注意：此处使用事件机制获取玩家位置，避免直接依赖 PlayerController
        Vector3 playerPosition = _getPlayerPosition();
        Vector3 midpoint = Vector3.Lerp(playerPosition, _target.position, _tuning.lockOnTargetBias);

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
    /// 获取玩家位置（通过事件机制，避免直接依赖）
    /// </summary>
    private Vector3 _getPlayerPosition()
    {
        // 发布查询事件获取玩家位置
        // 具体实现由 PlayerController 通过事件响应提供
        return default; // 占位，实际通过事件系统获取
    }

    /// <summary>
    /// 锁定超时回调
    /// </summary>
    public event Action OnLockOnTimedOut;

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
    private Vector3 _basePosition;
    private Quaternion _baseRotation;

    /// <summary>
    /// 震动叠加层位置偏移（由 CameraManager 在应用位置后叠加）
    /// </summary>
    public Vector3 ShakeOffset { get; private set; }

    /// <summary>
    /// 震动叠加层旋转偏移
    /// </summary>
    public Quaternion ShakeRotationOffset { get; private set; }

    private void Awake()
    {
        if (Instance != null)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
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

            shakeOffset += CalculateShakeOffset(shake.Mode, intensity);
            shakeRotation *= CalculateShakeRotation(shake.Mode, intensity);
        }

        // 移除完成的震动
        _activeShakes.RemoveAll(s => s.MarkComplete);

        // 更新叠加偏移
        ShakeOffset = shakeOffset;
        ShakeRotationOffset = shakeRotation;
    }

    /// <summary>
    /// 触发相机震动（由其他系统通过 EventBus 调用）
    /// </summary>
    public void TriggerShake(CameraShakeRequest request)
    {
        _activeShakes.Add(new ActiveShake
        {
            Mode = request.Mode,
            Intensity = request.Intensity,
            Duration = request.Duration,
            Elapsed = 0f
        });
    }

    private Vector3 CalculateShakeOffset(ShakeMode mode, float intensity)
    {
        return mode switch
        {
            ShakeMode.Impact => new Vector3(
                Random.Range(-1f, 1f) * intensity * _tuning.impactAmplitudeX,
                Random.Range(-1f, 1f) * intensity * _tuning.impactAmplitudeY,
                0f),

            ShakeMode.Rumble => new Vector3(
                Mathf.Sin(Time.time * _tuning.rumbleFrequency) * intensity * _tuning.rumbleAmplitudeX,
                Mathf.Sin(Time.time * _tuning.rumbleFrequency * 1.3f) * intensity * _tuning.rumbleAmplitudeY,
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

// CameraShakeRequest.cs
public struct CameraShakeRequest
{
    public ShakeMode Mode;
    public float Intensity;  // 0-1
    public float Duration;    // 秒
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
```

### 5. 过场相机控制

```csharp
// CinematicCameraBehavior.cs
/// <summary>
/// 过场相机行为
/// 由过场系统接管，支持路径动画和脚本控制
/// </summary>
public class CinematicCameraBehavior : ICameraBehavior
{
    [SerializeField] private Camera _camera;
    [SerializeField] private CinematicPath _path;

    private float _progress;
    private bool _isPlaying;

    public void OnStateEnter(CameraState fromState)
    {
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

        if (_progress >= 1f)
        {
            _isPlaying = false;
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
    public float globalIntensityScale = 1f;

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
│       │   ├── CinematicCameraBehavior.cs # 过场行为
│       │   └── ShakeCameraBehavior.cs    # 震动行为
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
- [ ] 定义 CameraShakeRequest 事件

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

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — CameraShakeRequest 通过 EventBus 触发
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — **Camera System 属于 Presentation Layer**
- [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) — 锁定聚焦与处决动画同步
- [ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — 震动与屏幕特效协作
- [ADR-0009: 玩家控制器](./adr-0009-player-controller-architecture.md) — **PlayerMovementState 定义位置**
- [共享类型定义](./shared-types.md) — **PlayerMovementState (§7.2)**

---

## 附录：类型依赖说明

| 类型 | 定义位置 | 说明 |
|------|---------|------|
| `PlayerMovementState` | shared-types.md §7.2 | 玩家移动状态枚举 |
| `CameraState` | ADR-0026 本文档 | 相机状态枚举（不含 Shake） |
| `ShakeMode` | ADR-0026 本文档 | 震动模式枚举 |
| `CameraShakeRequest` | ADR-0026 本文档 | 震动请求结构 |
| `ICameraBehavior` | ADR-0026 本文档 | 相机行为接口 |

**设计说明**：
- 震动（Shake）是叠加效果，由 `CameraShakeManager` 独立管理
- `CameraState` 不包含 Shake，因为震动不是独占状态
- `CameraManager` 协调 `CameraStateMachine` 和 `CameraShakeManager`
