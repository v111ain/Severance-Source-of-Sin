# ADR-0007: 视野与监听系统 (LOS System) 架构决策

> **文档一致性说明**：本文档为 ADR-0007 原版，内容与文件名一致。

## Status
**Accepted**

## Date
2026-04-09

## Last Updated
2026-04-15 (ADR 评审修复：SoundSource 类补充 IsOnScreen 属性；补充 LOS System 对 Lighting/WeatherStateChangedEvent 的订阅说明)

## Context

### Problem Statement

LOS System（Line of Sight & Eavesdropping）是《断绝：罪恶之源》"线索驱动的动态潜行"核心支柱的实现载体。它负责计算玩家在 NPC 视野中的暴露程度，以及专注监听模式下的关键词捕获机制。该系统需要与多个系统交互：

1. **NPC AI System**：暴露值变化时发送 `ExposureValueChangedEvent` 更新感知，暴露值满时发送 `PlayerSpottedEvent` 触发战斗
2. **Lighting System**：查询玩家是否处于阴影中计算隐蔽加成
3. **Clue System**：发送 `KeywordCapturedEvent` 转化关键词为线索；发送 `NPCIdentityConfirmedEvent` 触发线索发现
4. **Player Controller**：读取玩家移动状态和位置；订阅 `NoiseMadeEvent` 进行噪声检测

### Constraints

- **性能约束**：需要在 PS5 上稳定 60fps，100 个 NPC 同时感知
- **解耦约束**：LOS System 与 NPC AI System 通过 Event Bus 通信，不直接调用
- **平台约束**：支持 PC & PS5，手柄和键鼠两种输入模式

### Requirements

- **必须**：定义 LOS 视野计算架构（基于暴露值的累积机制）
- **必须**：定义专注监听模式的关键词捕获机制
- **必须**：定义与 Lighting System 的接口（阴影状态查询）
- **必须**：定义 NPC 身份标签管理架构（Unknown/Enemy/Accomplice/Victim）
- **必须**：支持空间分区优化（避免 O(n) 全实体遍历）

---

## Decision

### 架构决策

采用**暴露值累积 + 专注监听双模式**架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LOS System 架构                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                      LOSSystem (Core Component)                    │   │
│  │  - 管理所有 NPC 的 ExposureTracker 实例                            │   │
│  │  - 每帧调用 Update() 更新所有暴露值                                 │   │
│  │  - 维护 ActiveNPCList（已注册 NPC）                                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│          ┌─────────────────────────┴─────────────────────────┐          │
│          ▼                                                   ▼          │
│  ┌───────────────────┐                           ┌───────────────────┐   │
│  │  ExposureTracker  │                           │  FocusListener    │   │
│  │  (每 NPC 一个)     │                           │  (专注监听模式)    │   │
│  │                   │                           │                   │   │
│  │ - VisualExposure  │                           │ - TagProgress     │   │
│  │ - DistanceFactor  │                           │ - AimAccuracy     │   │
│  │ - StealthBonus    │                           │ - SoundSources[]  │   │
│  │ - IsInShadow      │                           │                   │   │
│  └───────────────────┘                           └───────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     NPCIdentityManager                              │   │
│  │  - 维护所有 NPC 的身份标签状态                                     │   │
│  │  - 管理 Keyword → Identity 映射                                     │   │
│  │  - 广播 KeywordCapturedEvent 到 Clue System                        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     SpatialPartition                              │   │
│  │  - 空间分区优化（Quadrant/Octree）                                 │   │
│  │  - 仅对玩家视野范围内的 NPC 计算暴露值                             │   │
│  │  - 专注模式下声源筛选                                              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. 暴露值计算机制（Visual Exposure）

LOS System 的核心是**暴露值累积**而非传统的视锥检测。

#### 配置常量定义

> 以下常量统一在 `LOSConfigSO` ScriptableObject 中配置，避免代码中的 Magic Numbers：

```csharp
// LOSConfigSO.cs
[CreateAssetMenu(fileName = "LOSConfig", menuName = "Game/LOS/LOSConfig")]
public class LOSConfigSO : ScriptableObject
{
    // ========== 暴露值计算 ==========
    [Header("暴露值计算")]
    public float BaseExposureRate = 20f;      // 基础暴露速率（行走时 20%/秒）
    public float DecayRate = 30f;             // 暴露值衰减速率（30%/秒），玩家离开视野后生效
    public float MaxVisionRange = 15f;        // NPC 最大感知范围（米）
    public float VisionConeAngle = 120f;      // NPC 视野锥角度（度）
    public float ProximityThreshold = 1.5f;   // 贴脸判定距离（米）

    // ========== 噪声检测 ==========
    [Header("噪声检测")]
    public float MaxAudioRange = 20f;         // NPC 最大听觉范围（米）
    public float NoiseExposureBase = 10f;     // 噪声基础暴露速率（%/秒）

    // ========== 专注监听模式 ==========
    [Header("专注监听")]
    public float MaxScreenDistance = 200f;     // 屏幕距离阈值（像素）
    public float MinAimAccuracy = 0.5f;       // 最低瞄准精度（0-1）
    public float TagRate = 33.3f;             // 标签捕获速率（%/秒），3秒填满
    public float FocusAlignmentTime = 1f;    // 准星对准后开始累积的延迟（秒）

    // ========== 暴露值乘数 ==========
    [Header("移动状态乘数")]
    public float IdleMultiplier = 0.1f;       // 待机暴露乘数
    public float WalkMultiplier = 1.0f;       // 行走暴露乘数
    public float SprintMultiplier = 3.0f;     // 冲刺暴露乘数
    public float CrouchMultiplier = 0.1f;    // 潜行暴露乘数
    public float ActionMultiplier = 0.0f;    // 动作中暴露乘数
}

// IdentityKeywordsSO.cs
/// <summary>
/// NPC 关键词配置（由 Narrative Director 创建）
/// </summary>
[CreateAssetMenu(fileName = "IdentityKeywords", menuName = "Game/LOS/IdentityKeywords")]
public class IdentityKeywordsSO : ScriptableObject
{
    [Serializable]
    public struct KeywordEntry
    {
        public string Keyword;           // 关键词（如 "掩护我"、"别开枪"）
        public NPCIdentityType IdentityType;  // 对应身份类型
    }

    public List<KeywordEntry> Entries = new();
}

#### 配置常量使用示例

```csharp
// ExposureTracker.cs
public class ExposureTracker
{
    private LOSConfigSO _config;

    public ExposureTracker(LOSConfigSO config)
    {
        _config = config;
    }

    public void Update(Transform player, Transform npc, float deltaTime)
    {
        if (!IsPlayerInVisionCone(player, npc))
        {
            CurrentExposure = Mathf.Max(0, CurrentExposure - _config.DecayRate * deltaTime);
            return;
        }

        float distance = Vector3.Distance(player.position, npc.position);
        float distanceFactor = Mathf.Clamp(1.0f - (distance / _config.MaxVisionRange), 0f, 1f);
        float movementMultiplier = GetMovementMultiplier(_currentPlayerState);

        float deltaExposure = _config.BaseExposureRate * movementMultiplier * distanceFactor * _exposureMultiplier;
        CurrentExposure = Mathf.Clamp(CurrentExposure + deltaExposure * deltaTime, 0f, 100f);
        // ...
    }
}
```

> **事件来源说明**：
> - `PlayerMovementStateChangedEvent` 定义于 [事件总线 ICD](../../engine-reference/event-bus-icd.md)，由 PlayerController 发布
> - `LightingStateChangedEvent` 定义于 [ADR-0022](../../engine-reference/event-bus-icd.md)，由 LightingSystem 发布
> - `PlayerSpottedEvent` 定义于 [事件总线 ICD](../../engine-reference/event-bus-icd.md)

```csharp
// ExposureTracker.cs
public class ExposureTracker
{
    // ========== 暴露值定义（统一为 0-100 浮点值）==========
    // CurrentExposure：当前暴露值（0-100），100 表示完全暴露
    // VisualScore：供 NPC AI 查询的暴露值（0-100），与 CurrentExposure 相同
    // 转换规则：VisualScore = CurrentExposure（直接映射，无转换）
    //
    // 暴露值 → AlertState 映射（供 NPC AI 使用）：
    // | 暴露值 | AlertState  | NPC 行为描述 |
    // |--------|-------------|-------------|
    // | 0-29   | UNDETECTED  | 正常巡逻，无警觉 |
    // | 30-59  | SUSPECT    | 感到可疑，暂停观察 |
    // | 60-79  | SEARCH     | 确认异常，开始搜索 |
    // | 80-99  | ALERT      | 确认威胁，准备战斗 |
    // | ESCAPE | ESCAPE     | 逃离现场（需要医疗/紧急情况触发）|
    // | 100    | COMBAT     | 投入战斗 |

    public float CurrentExposure { get; private set; }  // 0-100%
    public float VisualScore => CurrentExposure;  // 供 NPC AI 查询（0-100）

    private LOSConfigSO _config;  // 配置数据

    // NPC 视野参数（由 LOSSystem 在 RegisterNPC 时注入）
    private float _npcVisionRange;
    private float _npcVisionAngle;

    // 玩家最后暴露值（用于 MemoryScore 衰减计算）
    private float _lastExposureValue;

    // 订阅来自 Player Controller 的移动状态事件
    private void SubscribeToEvents()
    {
        EventBus.Instance.Subscribe<PlayerMovementStateChangedEvent>(OnPlayerMovementStateChanged);
        EventBus.Instance.Subscribe<LightingStateChangedEvent>(OnStealthBonusChanged);
        EventBus.Instance.Subscribe<NoiseMadeEvent>(OnNoiseMade);  // 噪声检测
    }

    private PlayerMovementState _currentPlayerState = PlayerMovementState.IDLE;
    private float _exposureMultiplier = 1.0f;  // 1.0 = 正常暴露，0.77 = 阴影中降低 23% 暴露速度
    private float _noiseMultiplier = 1.0f;     // 噪声乘数（NoiseMadeEvent 影响）

    private void OnPlayerMovementStateChanged(PlayerMovementStateChangedEvent e)
    {
        _currentPlayerState = e.new_state;
    }

    private void OnStealthBonusChanged(LightingStateChangedEvent e)
    {
        // exposureMultiplier: 1.0 = 无加成（光照良好）, 0.77 = 阴影中（降低 23% 暴露速度）
        // shadowStealthBonus 范围是 0.0~1.0+，值越高表示阴影中的潜行效果越好（NPC越难检测到玩家）
        // 我们需要将其转换为 exposureMultiplier：shadowStealthBonus = 1.0 时 exposureMultiplier = 1.0
        // shadowStealthBonus = 1.5 时 exposureMultiplier = 0.77（降低 23% 暴露速度）
        _exposureMultiplier = 1.0f / e.perceptionModifier.shadowStealthBonus;
        _exposureMultiplier = Mathf.Clamp(_exposureMultiplier, 0.5f, 1.0f);  // 限制范围
    }

    /// <summary>
    /// 玩家噪声检测：订阅 NoiseMadeEvent，通过 exposureMultiplier 影响暴露速度
    /// 噪声暴露增量 = 基础暴露速率 × 噪声强度 × 距离因子 × exposureMultiplier
    /// </summary>
    private void OnNoiseMade(NoiseMadeEvent e)
    {
        if (_npcId == 0) return;  // 未注册

        float distance = Vector3.Distance(_trackedNpcPosition, e.position);
        if (distance > _config.MaxAudioRange) return;  // 超出听觉范围

        // 噪声强度（0-100）转换为暴露乘数
        // 强度越高，暴露速度越快
        float noiseFactor = e.intensity / 100f;
        _noiseMultiplier = 1.0f + noiseFactor * 2.0f;  // 噪声时暴露速度可增加至 3x
    }

    // 暴露值更新（每帧调用）
    public void Update(Transform player, Transform npc, float deltaTime)
    {
        _trackedNpcPosition = npc.position;
        bool wasInVision = IsPlayerInVisionCone(player, npc);

        if (!wasInVision)
        {
            // ========== 玩家离开视野：暴露值衰减规则 ==========
            // 玩家离开视野后，暴露值按 DecayRate=30%/秒 衰减
            // 衰减公式：CurrentExposure = max(0, CurrentExposure - DecayRate * deltaTime)
            // _lastExposureValue 保存衰减前的暴露值，供 NPC AI 的 MemoryScore 使用
            _lastExposureValue = CurrentExposure;
            CurrentExposure = Mathf.Max(0, CurrentExposure - _config.DecayRate * deltaTime);

            // 发布 ExposureValueChangedEvent（供 NPC AI 更新 MemoryScore）
            PublishExposureChangedEvent(npc);

            // 重置噪声乘数
            _noiseMultiplier = 1.0f;
            return;
        }

        // ========== 玩家在视野内：暴露值累积 ==========
        // 计算新增暴露值
        float distance = Vector3.Distance(player.position, npc.position);
        float distanceFactor = Mathf.Clamp(1.0f - (distance / _npcVisionRange), 0f, 1f);
        float movementMultiplier = GetMovementMultiplier(_currentPlayerState);

        // 暴露增量 = 基础暴露速率 × 移动乘数 × 距离因子 × 阴影乘数 × 噪声乘数
        float deltaExposure = _config.BaseExposureRate * movementMultiplier * distanceFactor * _exposureMultiplier * _noiseMultiplier;
        CurrentExposure = Mathf.Clamp(CurrentExposure + deltaExposure * deltaTime, 0f, 100f);

        // 发布 ExposureValueChangedEvent（暴露值变化时实时同步）
        PublishExposureChangedEvent(npc);

        // 暴露值满，触发发现事件
        if (CurrentExposure >= 100f)
        {
            EventBus.Instance.Publish(new PlayerSpottedEvent
            {
                player_id = PlayerController.Instance.GetPlayerId(),
                npc_id = npc.GetComponent<NPCController>().NpcId,
                spot_time = Time.time
            });
        }
    }

    /// <summary>
    /// 发布 ExposureValueChangedEvent，通知 NPC AI 感知系统
    /// NPC AI 订阅此事件更新 MemoryScore（而非直接同步）
    /// </summary>
    private void PublishExposureChangedEvent(Transform npc)
    {
        EventBus.Instance.Publish(new ExposureValueChangedEvent
        {
            npc_id = npc.GetComponent<NPCController>().NpcId,
            exposure_value = CurrentExposure,
            last_exposure_value = _lastExposureValue  // 衰减前的值，供 MemoryScore 使用
        });
    }

    // 移动状态乘数映射（从配置读取）
    private float GetMovementMultiplier(PlayerMovementState state)
    {
        return state switch
        {
            PlayerMovementState.IDLE => _config.IdleMultiplier,
            PlayerMovementState.WALK => _config.WalkMultiplier,
            PlayerMovementState.SPRINT => _config.SprintMultiplier,
            PlayerMovementState.CROUCH or PlayerMovementState.CROUCH_WALK => _config.CrouchMultiplier,
            PlayerMovementState.ACTION => _config.ActionMultiplier,
            _ => 1.0f
        };
    }

    // 贴脸判定（使用 NPC 级别的感知范围）
    private bool IsPlayerInProximity(Transform player, Transform npc)
    {
        return Vector3.Distance(player.position, npc.position) < _config.ProximityThreshold;
    }

    // 视锥检测实现（使用 NPC 级别的视野参数）
    private bool IsPlayerInVisionCone(Transform player, Transform npc)
    {
        // 检查距离是否在 NPC 最大视野范围内
        float distance = Vector3.Distance(player.position, npc.position);
        if (distance > _npcVisionRange)
            return false;

        // 检查玩家是否在 NPC 正前方锥形范围内
        Vector3 npcForward = npc.forward;
        Vector3 directionToPlayer = (player.position - npc.position).normalized;

        float angle = Vector3.Angle(npcForward, directionToPlayer);
        if (angle > _npcVisionAngle / 2f)
            return false;

        // 检查是否有障碍物遮挡视线（Phase 2 特性，启用后增强真实性）
        // 注意：启用遮挡检测会增加性能开销，PS5 100 NPC 场景下需谨慎
        // if (Physics.Raycast(npc.position, directionToPlayer, out var hit, distance, obstacleLayer))
        //     return false;

        // 贴脸判定优先于视锥检测
        if (IsPlayerInProximity(player, npc))
            return true;

        // 通过所有检测，玩家在 NPC 视野范围内
        return true;
    }

    private int _npcId;
    private Vector3 _trackedNpcPosition;
}

// ========== 新增事件定义 ==========

/// <summary>
/// 暴露值变化事件（由 ExposureTracker 发布，NPC AI 订阅）
/// 用于 NPC AI 更新 MemoryScore，而非直接同步 VisualScore
/// </summary>
public struct ExposureValueChangedEvent
{
    public int npc_id;              // NPC ID
    public float exposure_value;    // 当前暴露值（0-100）
    public float last_exposure_value;  // 衰减前的暴露值（供 MemoryScore 使用）
}

// 注意：
// - LightingStateChangedEvent 定义于 [ADR-0022](../../architecture/adr-0022-lighting-system-architecture.md)
// - PlayerMovementStateChangedEvent 定义于 [事件总线 ICD](../../engine-reference/event-bus-icd.md)
// - AreaLightingChangedEvent 定义于 [ADR-0022](../../architecture/adr-0022-lighting-system-architecture.md)
// - NoiseMadeEvent 定义于 [事件总线 ICD](../../engine-reference/event-bus-icd.md)
// - NPCIdentityConfirmedEvent 定义于 [shared-types.md](./shared-types.md)
```

### 2. 专注监听模式（Focus Mode）

```csharp
// FocusListener.cs
public class FocusListener
{
    public bool IsActive { get; private set; }
    public float CurrentTagProgress { get; private set; }  // 0-100%

    private LOSConfigSO _config;  // 配置数据

    // 声源数据结构
    private class SoundSource
    {
        public int NpcId;
        public Vector3 WorldPosition;
        public Vector2 ScreenPosition;  // 投影到屏幕的像素坐标
        public bool IsOnScreen;  // 是否在屏幕内可见
    }

    private List<SoundSource> _activeSoundSources = new();
    private Camera _mainCamera;  // 缓存相机引用，避免每帧查找 Camera.main
    private float _alignmentTimer = 0f;  // 累计对准时间

    public void Initialize(LOSConfigSO config)
    {
        _config = config;
        _mainCamera = Camera.main;
        // 订阅 NPCSpeakingChangedEvent，持续维护 _activeSoundSources 列表
        EventBus.Instance.Subscribe<NPCSpeakingChangedEvent>(OnNPCSpeakingChanged);
    }

    public void Dispose()
    {
        EventBus.Instance.Unsubscribe<NPCSpeakingChangedEvent>(OnNPCSpeakingChanged);
    }

    /// <summary>
    /// 响应 NPC 发声状态变化，维护活跃声源列表
    /// </summary>
    private void OnNPCSpeakingChanged(NPCSpeakingChangedEvent evt)
    {
        if (evt.is_speaking)
        {
            // 避免重复添加
            if (_activeSoundSources.All(s => s.NpcId != evt.npc_id))
            {
                _activeSoundSources.Add(new SoundSource
                {
                    NpcId = evt.npc_id,
                    WorldPosition = evt.position
                });
            }
            else
            {
                // 更新位置（NPC 可能移动）
                var source = _activeSoundSources.Find(s => s.NpcId == evt.npc_id);
                if (source != null) source.WorldPosition = evt.position;
            }
        }
        else
        {
            _activeSoundSources.RemoveAll(s => s.NpcId == evt.npc_id);
        }
    }

    /// <summary>
    /// 获取主相机引用（每次调用时检查有效性，支持运行时相机切换）
    /// </summary>
    private Camera GetMainCamera()
    {
        if (_mainCamera == null)
            _mainCamera = Camera.main;
        return _mainCamera;
    }

    public void EnterFocusMode()
    {
        IsActive = true;
        // _activeSoundSources 通过 NPCSpeakingChangedEvent 订阅持续维护，无需手动刷新
    }

    public void ExitFocusMode()
    {
        IsActive = false;
        CurrentTagProgress = 0f;
    }

    public void Update(Vector2 crosshairPosition, float deltaTime)
    {
        if (!IsActive || _activeSoundSources.Count == 0)
        {
            _alignmentTimer = 0f;
            return;
        }

        // 找出最靠近准星的声源
        SoundSource nearest = null;
        float nearestDistance = float.MaxValue;
        float nearestAccuracy = 0f;

        foreach (var source in _activeSoundSources)
        {
            Vector3 screenPos = GetMainCamera().WorldToScreenPoint(source.WorldPosition);
            // 相机裁剪检查：z < 0 表示目标在相机后方，屏幕坐标无效
            if (screenPos.z < 0)
            {
                source.ScreenPosition = new Vector3(float.NaN, float.NaN, 0);
                source.IsOnScreen = false;
                continue;
            }
            source.ScreenPosition = screenPos;
            source.IsOnScreen = true;
            float dist = Vector2.Distance(crosshairPosition, source.ScreenPosition);

            if (dist < nearestDistance)
            {
                nearestDistance = dist;
                nearest = source;
                nearestAccuracy = Mathf.Clamp(1.0f - (dist / _config.MaxScreenDistance), 0f, 1f);
            }
        }

        // 准星精准度阈值判定
        if (nearestAccuracy < _config.MinAimAccuracy)
        {
            // 未对准，重置对准计时器
            _alignmentTimer = 0f;
            return;
        }

        // 对准时间累计
        _alignmentTimer += deltaTime;
        if (_alignmentTimer < _config.FocusAlignmentTime)
        {
            // 未达到对准延迟，不累积进度
            return;
        }

        // 破译进度累积
        CurrentTagProgress = Mathf.Clamp(
            CurrentTagProgress + _config.TagRate * nearestAccuracy * deltaTime,
            0f, 100f);

        // 进度满，触发身份标签翻转
        if (CurrentTagProgress >= 100f)
        {
            TriggerIdentityTag(nearest);
        }
    }

    private void TriggerIdentityTag(SoundSource source)
    {
        string keyword = NPCIdentityManager.Instance.GetNextKeyword(source.NpcId);
        if (string.IsNullOrEmpty(keyword))
        {
            // 所有关键词已捕获，不触发翻转
            CurrentTagProgress = 0f;
            return;
        }

        // 向 NPCIdentityManager 请求翻转该 NPC 的身份标签
        EventBus.Instance.Publish(new KeywordCapturedEvent
        {
            keyword = keyword,
            npc_id = source.NpcId,
            location_id = LocationManager.Instance.GetCurrentLocationId(),
            category = ClueCategory.IDENTITY,
            capture_timestamp = Time.time
        });

        CurrentTagProgress = 0f;
    }
}
```

### 3. NPC 身份标签管理

NPC 身份标签（Identity）采用**置信度渐进式演变**机制：

- **UNKNOWN（未确认）**：Confidence < 1.0，NPC 行为不受影响
- **Confirmed 状态**：Confidence == 1.0，身份类型已确认（Enemy/Accomplice/Victim）
  - Enemy：NPC 会主动追击玩家
  - Accomplice：NPC 会报警或通知目标
  - Victim：NPC 不会攻击玩家，可作为盟友

```csharp
// NPCIdentityManager.cs
public class NPCIdentityManager : MonoBehaviour
{
    public static NPCIdentityManager Instance { get; private set; }

    // NPC 身份数据
    private Dictionary<int, NPCIdentity> _identities = new();

    // 关键词 → 身份映射（由 Narrative Director 通过 IdentityKeywordsSO 配置）
    private Dictionary<string, NPCIdentityType> _keywordToIdentity = new();

    /// <summary>
    /// 从配置数据初始化关键词映射
    /// </summary>
    /// <param name="keywordsSO">关键词配置 ScriptableObject（由 Narrative Director 创建）</param>
    public void LoadKeywordConfiguration(IdentityKeywordsSO keywordsSO)
    {
        _keywordToIdentity.Clear();
        foreach (var entry in keywordsSO.Entries)
        {
            _keywordToIdentity[entry.Keyword] = entry.IdentityType;
        }
    }

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }

    public NPCIdentity GetIdentity(int npcId) => _identities[npcId];

    public string GetNextKeyword(int npcId)
    {
        // 返回下一个未捕获的关键词
        if (!_identities.TryGetValue(npcId, out var identity))
            return null;

        foreach (var keyword in identity.PotentialKeywords)
        {
            if (!identity.CapturedKeywords.Contains(keyword))
                return keyword;
        }
        return null;  // 所有关键词已捕获
    }

    public void ApplyKeyword(int npcId, string keyword)
    {
        if (_keywordToIdentity.TryGetValue(keyword, out var type))
        {
            var identity = _identities[npcId];
            identity.Confidence = 1.0f;  // 关键词确认，置信度满
            identity.CapturedKeywords.Add(keyword);
            identity.Type = type;

            // 置信度达到 1.0 时，触发 NPC 行为变化
            EventBus.Instance.Publish(new NPCIdentityConfirmedEvent
            {
                npc_id = npcId,
                identity_type = type
            });
        }
    }
}

public struct NPCIdentity
{
    public int NpcId;
    public NPCIdentityType Type;  // UNKNOWN / ENEMY / ACCOMPLICE / VICTIM
    public float Confidence;  // 0.0 - 1.0
    public List<string> PotentialKeywords;  // 可能揭示身份的关键词
    public List<string> CapturedKeywords;  // 已捕获的关键词
}

/// <summary>
/// NPCIdentityConfirmedEvent
/// 当 NPC 身份被确认时由 NPCIdentityManager 发布
///
/// 订阅者：
/// - NPCAI：收到此事件后应升级为 Enemy 状态（见 ADR-0004 §3）
/// - ClueJournal：收到此事件后触发线索发现
/// </summary>
public struct NPCIdentityConfirmedEvent
{
    public int npc_id;
    public NPCIdentityType identity_type;
}
```

### 4. 空间分区优化

> **格子大小论证**：选择 10m 格子基于以下分析：
> - NPC 最大感知范围（MaxVisionRange）通常为 15-20m
> - 每个格子覆盖 10m × 10m，任意感知范围内的查询最多遍历 3×3 = 9 个格子
> - 100 个 NPC 均匀分布时，每格子约 1 个 NPC，遍历开销极低
> - 10m 格子既避免了过小格子导致的内存开销（100 NPC 需要 100+ 格子），也避免了过大格子导致的精确度问题

```csharp
// SpatialPartition.cs
using UnityEngine;

public class SpatialPartition
{
    /// <summary>
    /// 格子大小（米），基于性能分析选择 10m：
    /// - 覆盖最大感知范围（15-20m）时，查询最多遍历 3×3=9 格
    /// - 100 NPC 均匀分布时每格约 1 NPC，遍历效率高
    /// - 统一使用 GameConstants.SPATIAL_GRID_SIZE（见 shared-constants.md）
    /// </summary>
    private const int GRID_SIZE = GameConstants.SPATIAL_GRID_SIZE;
    private Dictionary<Vector2Int, List<int>> _grid = new();  // 格子 → NPC ID 列表
    private Vector3 _playerLastPosition;

    public void RegisterNPC(int npcId, Vector3 position)
    {
        var cell = WorldToCell(position);
        if (!_grid.ContainsKey(cell))
            _grid[cell] = new List<int>();
        _grid[cell].Add(npcId);
    }

    public List<int> GetNPCsInRange(Vector3 playerPos, float range)
    {
        var result = new List<int>();
        int cellRange = Mathf.CeilToInt(range / GRID_SIZE);
        var playerCell = WorldToCell(playerPos);

        for (int x = -cellRange; x <= cellRange; x++)
        {
            for (int z = -cellRange; z <= cellRange; z++)
            {
                var cell = new Vector2Int(playerCell.x + x, playerCell.y + z);
                if (_grid.TryGetValue(cell, out var npcs))
                {
                    foreach (var npcId in npcs)
                        result.Add(npcId);
                }
            }
        }
        return result;
    }

    private Vector2Int WorldToCell(Vector3 pos)
    {
        return new Vector2Int(
            Mathf.FloorToInt(pos.x / GRID_SIZE),
            Mathf.FloorToInt(pos.z / GRID_SIZE));
    }
}
```

> **⚠️ 跨 ADR 共享组件 [已修复]**
>
> `SpatialPartition` 已替换为共享组件 `SpatialGrid`（见 [ADR-0030](./adr-0030-spatial-grid-shared-component.md)）。
> 两个 ADR 共用 `Assets/Game/Foundation/Shared/Spatial/SpatialGrid.cs` 实现。

### 5. LOSSystem 主控制器

```csharp
// LOSSystem.cs
/// <summary>
/// LOS System 主控制器 - 管理所有 NPC 的暴露值追踪和专注监听
/// 使用 MonoBehaviour 单例模式
/// </summary>
public class LOSSystem : MonoBehaviour
{
    public static LOSSystem Instance { get; private set; }

    private LOSConfigSO _config;
    private SpatialPartition _spatialPartition;
    private FocusListener _focusListener;
    private NPCIdentityManager _identityManager;
    private Dictionary<int, ExposureTracker> _exposureTrackers = new();

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }

    public void Initialize(LOSConfigSO config)
    {
        _config = config;
        _spatialPartition = new SpatialPartition();
        _focusListener = new FocusListener();
        _focusListener.Initialize(config);
        _identityManager = NPCIdentityManager.Instance;

        // 初始化所有 NPC 的 ExposureTracker
        foreach (var npc in NPCManager.Instance.GetAllNPCs())
        {
            RegisterNPC(npc);
        }
    }

    public void RegisterNPC(NPCController npc)
    {
        var tracker = new ExposureTracker(_config)
        {
            NpcVisionRange = npc.Data.maxPerceptionRange,
            NpcVisionAngle = npc.Data.fieldOfView
        };
        tracker.SubscribeToEvents();  // 【关键】确保事件订阅被调用
        _exposureTrackers[npc.NpcId] = tracker;
        _spatialPartition.RegisterNPC(npc.NpcId, npc.transform.position);
    }

    private void Update()
    {
        var player = PlayerController.Instance.transform;
        var nearbyNPCs = _spatialPartition.GetNPCsInRange(player.position, _config.MaxVisionRange);

        foreach (var npcId in nearbyNPCs)
        {
            if (_exposureTrackers.TryGetValue(npcId, out var tracker))
            {
                var npc = NPCManager.Instance.GetNPC(npcId);
                if (npc != null)
                {
                    tracker.Update(player, npc.transform, Time.deltaTime);
                }
            }
        }

        // 更新专注监听模式
        if (_focusListener.IsActive)
        {
            // 获取当前准星位置（从 UI 系统传入）
            _focusListener.Update(GetCrosshairPosition(), Time.deltaTime);
        }
    }

    private Vector2 GetCrosshairPosition()
    {
        // 从 UIManager 获取准星屏幕位置（异步查询接口，避免循环依赖）
        // 注意：UIManager 需实现 GetCrosshairPosition() 公共接口
        return UIManager.Instance?.GetCrosshairPosition() ?? Vector2.zero;
    }

    // 供 UI 层调用
    public void EnterFocusMode() => _focusListener.EnterFocusMode();
    public void ExitFocusMode() => _focusListener.ExitFocusMode();
    public bool IsFocusActive => _focusListener.IsActive;
}
```

### 5.1. ExposureTracker 扩展

```csharp
// ExposureTracker.cs 扩展 - 添加 NPC 视野参数和事件订阅
public partial class ExposureTracker
{
    public float NpcVisionRange { get; set; }
    public float NpcVisionAngle { get; set; }

    // 【关键】暴露事件订阅必须在 LOSSystem 初始化时调用
    public void SubscribeToEvents()
    {
        EventBus.Instance.Subscribe<PlayerMovementStateChangedEvent>(OnPlayerMovementStateChanged);
        EventBus.Instance.Subscribe<LightingStateChangedEvent>(OnLightingStateChanged);
        // 订阅光照潜行加成变化事件（用于更细粒度的感知系数更新）
        EventBus.Instance.Subscribe<LightingStealthBonusChangedEvent>(OnStealthBonusChanged);
    }

    private void OnLightingStateChanged(LightingStateChangedEvent e)
    {
        // exposureMultiplier = 1.0 / shadowStealthBonus（shadowStealthBonus 越高，exposureMultiplier 越低）
        _exposureMultiplier = 1.0f / e.perceptionModifier.shadowStealthBonus;
        _exposureMultiplier = Mathf.Clamp(_exposureMultiplier, 0.5f, 1.0f);
    }

    private void OnStealthBonusChanged(LightingStealthBonusChangedEvent e)
    {
        // 使用更细粒度的光照潜行加成事件更新暴露乘数
        // exposure_multiplier 越高 = NPC 越容易检测到玩家（阴影加成低）
        _exposureMultiplier = e.exposure_multiplier;
        _exposureMultiplier = Mathf.Clamp(_exposureMultiplier, 0.5f, 1.0f);
    }
}
```

### 5. Unity 项目结构

```
Assets/Game/
├── Core/
│   ├── LOS/
│   │   ├── LOSSystem.cs              # 主系统管理器
│   │   ├── ExposureTracker.cs        # 单个 NPC 的暴露值追踪
│   │   ├── FocusListener.cs          # 专注监听模式
│   │   ├── NPCIdentityManager.cs     # NPC 身份标签管理
│   │   ├── SpatialPartition.cs       # 空间分区优化
│   │   ├── NoiseSourceDetector.cs    # 声源检测（专注模式用）
│   │   ├── VisionConeChecker.cs      # 视锥检测实现
│   │   └── LOSConfigSO.cs            # 配置 ScriptableObject
│   │
│   └── NPCAI/
│       └── ... (见 ADR-0004)
```

---

## Alternatives Considered

### Alternative 1: 传统视锥检测 (Cone-based Vision)

- **描述**：使用扇形/锥体检测玩家是否在 NPC 视野内
- **Pros**：概念简单，调试直观
- **Cons**：无法表达"暴露程度"，只能判断 0/1 发现
- **拒绝理由**：
  - 无法支持本游戏的暴露值累积机制（玩家可以在被完全发现前撤退）
  - 无法支持移动状态乘数（静止=低暴露，奔跑=高暴露）
  - 传统视锥无法表达"逐渐警觉"的游戏体验

### Alternative 2: 纯射线检测 (Raycast-only)

- **描述**：仅使用射线检测判断是否有视线遮挡
- **Pros**：性能好，检测精确
- **Cons**：无法处理"察觉但不确定"的状态
- **拒绝理由**：
  - 本游戏需要"暴露值"作为渐进式反馈
  - 射线无法表达玩家移动状态对暴露的影响

---

## Consequences

### Positive

- **渐进式反馈**：暴露值机制让玩家有"还来得及撤退"的紧张感
- **专注模式解谜感**：关键词捕获提供侦探式的收集乐趣
- **身份标签翻转**：通过窃听揭示 NPC 真面目，增强叙事深度
- **性能可控**：空间分区确保 100 NPC 时依然 60fps

### Negative

- **双系统复杂度**：暴露值计算和专注监听是两个独立机制
- **调参复杂度**：6 个感知相关参数（BaseExposureRate, DecayRate, MaxVisionRange 等）需要精细平衡
- **身份标签数据依赖**：需要 Narrative Director 提前配置关键词映射

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **暴露值过快** | 玩家在阴影中依然被快速发现 | StealthBonus 参数调优：阴影中 StealthBonus = 0.77；上线前大量 playtest 验证 |
| **暴露值参数调试复杂** | 6 个参数相互影响 | 使用 Unity Inspector 暴露所有参数；建立自动化冒烟测试 |
| **关键词误触发** | 玩家意外捕获错误关键词 | AimAccuracy 阈值过滤（需 >= 0.5）；FocusAlignmentTime 累计 1 秒后才能开始累积进度 |
| **空间分区精度** | 大格子导致漏检，小格子内存高 | 动态格子大小（远处用大格，近处用小格）；上线前性能测试验证 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 1ms/帧 (100 NPC) | 空间分区 + 分帧计算 |
| **Memory** | < 20MB | ExposureTracker 按需分配 |
| **GPU** | 无直接影响 | 专注模式 UI 由 UGUI 渲染 |

> **性能预算说明**：LOS System (< 1ms) 是 NPC AI 感知更新 (< 2ms) 的子集。
> NPC AI 的 < 2ms 预算包含：LOS System 感知计算 (< 1ms) + 行为树决策 + 派系网络通信。
> 空间分区优化确保 100 NPC 场景下 LOS System 可稳定运行在 1ms 以内。

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 LOSConfigSO ScriptableObject
- [ ] 创建 ExposureTracker 基础逻辑
- [ ] 创建 NPCIdentityManager 基础结构

### Phase 2: 暴露值机制
- [ ] 实现暴露值累积公式
- [ ] 实现视野锥检测
- [ ] 订阅 PlayerController 的 PlayerMovementStateChangedEvent
- [ ] 订阅 LightingSystem 的 LightingStateChangedEvent（通过 Event Bus）

### Phase 3: 专注监听
- [ ] 实现 FocusListener
- [ ] 实现屏幕坐标投影
- [ ] 实现关键词捕获进度

### Phase 4: 优化
- [ ] 实现 SpatialPartition
- [ ] 分帧计算优化
- [ ] 100 NPC 性能测试

---

## Validation Criteria

1. **暴露值累积**（基于 LOSConfigSO 配置）：
   - 静止（Idle）状态下暴露值增长 = BaseExposureRate × IdleMultiplier = 20 × 0.1 = 2/秒
   - 行走（Walk）状态下暴露值增长 = BaseExposureRate × WalkMultiplier = 20 × 1.0 = 20/秒
   - 冲刺（Sprint）状态下暴露值增长 = BaseExposureRate × SprintMultiplier = 20 × 3.0 = 60/秒
   - 阴影中（StealthBonus=0.77）时暴露值增长速度降低 23%
2. **发现触发**：暴露值达到 100 时正确发送 `PlayerSpottedEvent`，间隔 < 1 帧
3. **贴脸判定**：进入 ProximityThreshold（1.5m）范围内无论状态立即暴露（CurrentExposure → 100）
4. **专注模式 UI**：由 Presentation Layer 的 UI System 实现，LOS System 仅负责提供 FocusListener.Active 状态
5. **关键词捕获**：准星对准声源（AimAccuracy ≥ MinAimAccuracy）持续 3 秒（100% / TagRate）后正确触发身份翻转
6. **隔墙监听**：隔着普通墙体可成功窃听（射线检测不遮挡声音）
7. **性能达标**：100 NPC 同时感知时 LOS System < 1ms/帧（NPC AI 总预算 < 2ms/帧）
8. **配置一致性**：所有感知相关参数均从 LOSConfigSO 读取，无硬编码 Magic Numbers
9. **暴露值衰减**：玩家离开视野后暴露值按 DecayRate=30%/秒 衰减，正确发布 ExposureValueChangedEvent
10. **噪声检测**：NoiseMadeEvent 通过 exposureMultiplier 影响暴露速度，正确累加 AudioScore
11. **身份确认事件**：NPCIdentityManager 确认身份后正确发布 NPCIdentityConfirmedEvent，ClueJournal 正确订阅触发线索发现
12. **VisualScore 统一**：VisualScore = CurrentExposure（0-100 浮点值），NPC AI 无需转换

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — LOS System 通过 Event Bus 与 NPC AI、Clue System 通信
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — LOS System 属于 Core Layer
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC AI 订阅 PlayerSpottedEvent
- [ADR-0009: 玩家控制器架构](./adr-0009-player-controller-architecture.md) — Player Controller 发布 PlayerMovementStateChangedEvent
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — PlayerMovementStateChangedEvent 的权威定义
- [LOS & Eavesdropping GDD](../../design/gdd/los-eavesdropping.md) — 本 ADR 的设计依据
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 事件定义的权威文档（PlayerMovementStateChangedEvent、LightingStateChangedEvent 等）
- [共享常量定义](./shared-constants.md) — GRID_SIZE 等跨 ADR 常量
