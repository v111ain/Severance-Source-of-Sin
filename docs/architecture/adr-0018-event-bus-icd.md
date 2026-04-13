# ADR-0018: 事件总线接口契约 (Event Bus ICD)

## Status
**Proposed**

## Date
2026-04-11

## Last Updated
2026-04-13

## Context

### Problem Statement

ADR-0001 定义了事件驱动架构的**概述**，但缺少完整的**接口契约文档 (ICD)**。在联调过程中发现：

1. **事件定义分散**：每个 ADR 自行定义事件，无统一索引
2. **订阅/发布关系不明确**：无法快速确认某事件的完整订阅者列表
3. **事件命名不一致**：ADR-0001 要求 `Subject + Did + Context + Event` 格式，但实际存在 `NPCDeath`、`NPCKilled` 等非正式称呼（实为 `NPCStateChangedEvent{new_state=WorldState.DEAD}` 的语义描述，非独立事件）
4. **Query/Request/Response 模式未标准化**：系统间调用缺乏统一规范

### Constraints

- 必须兼容 ADR-0001 定义的单例 ScriptableObject EventBus 实现
- 必须支持 shared-types.md 中已定义的所有事件类型
- 必须支持 Unity 6.3 LTS 的生命周期

### Requirements

- **必须**：完整列出所有事件，包含 Owner、Subscribers、Event Data 结构
- **必须**：标准化 Query/Request/Response 模式
- **必须**：定义事件兼容性规则（向后兼容、版本演进）
- **必须**：提供 Event Bus Debug 可视化接口

---

## Decision

### 事件分类

事件分为三类，对应不同的通信模式：

| 类型 | 模式 | 说明 | 示例 |
|------|------|------|------|
| **Event** | Publish/Subscribe | 单向广播，发送方不等待响应 | `NPCStateChangedEvent` |
| **Query** | Request/Response | 同步查询，调用方等待响应 | `QueryNPCIdentity` |
| **Request** | Fire-and-Forget | 单向请求，不等待响应 | `DamageRequest` |

### 完整事件索引

#### 3.1 Foundation Layer 事件

| Event | Owner | Subscribers | Event Data |
|-------|-------|------------|------------|
| `PlayerDamagedEvent` | Health System | GrittyTakedowns, SanityRage, ImmersiveAudio | `player_id`, `damage_type`, `damage_amount`, `hit_location`, `source_entity_id` |
| `PlayerMovementStateChangedEvent` | PlayerController | LOSSystem | `old_state`, `new_state` |
| `NoiseMadeEvent` | PlayerController | NPCAI | `position`, `radius`, `noise_type`, `duration`, `can_interrupt`, `source_entity_id` |
> **注意**：`NoiseMadeEvent` 原名为 `NoiseEvent`（已废弃别名），于 2026-04-12 正式更名。实现时应使用 `NoiseMadeEvent`
| `DamageRequest` | WeaponSystem, GrittyTakedowns | HealthSystem | `target_id`, `damage_type`, `damage_amount`, `penetration`, `hit_location`, `source_entity_id`, `source` |
| `ExplosionEvent` | WeaponSystem | HealthSystem, NPCAI, GrittyTakedowns | `position`, `radius`, `lethal_ratio`, `base_damage` |
| `NPCStateChangedEvent` | Health System | NPCAI, GrittyTakedowns | `npc_id`, `entity_type`, `old_state`, `new_state`, `damage_type` |
| `AlertStateChangedEvent` | NPCAI | GrittyTakedowns | `npc_id`, `old_state`, `new_state` |
> **注意**：`AlertStateChangedEvent` 与 ADR-0011 中定义的 `AlertStateChanged` 为同一事件，后者为已废弃别名，实现时应使用 `AlertStateChangedEvent`
>
> **别名映射**：
> - `AlertStateChanged` → `AlertStateChangedEvent`（已废弃，应使用后者）
| `ArmorDestroyedEvent` | HealthSystem | VFXSystem | `npc_id`, `position` |
| `ExplosionAlertEvent` | Health System | NPC AI | `position`, `radius`, `victim_id`, `KillerIsPlayer` |

#### 3.2 Core Layer 事件

| Event | Owner | Subscribers | Event Data |
|-------|-------|------------|------------|
| `PlayerSpottedEvent` | LOSSystem | NPCAI | `player_id`, `npc_id`, `spot_time` |
| `QuerySoundSourceScreenPosition` | LOSSystem | UI（Query Handler 直连，通过 QueryBus） | `npc_id` → `Vector3`（屏幕空间位置） |
| `EnvironmentalEvent` | EnvironmentInteraction | NPCAI | `type`, `position`, `radius`, `duration`, `intensity`, `source_object_id` |
| `ObjectStateChangedEvent` | EnvironmentInteraction | WeaponSystem | `object_id`, `object_category`, `new_state`, `position` |
| `WeaponAwarenessEvent` | WeaponSystem | NPCAI | `weapon_id`, `position`, `weapon_type` |
| `WeaponStateChangedEvent` | Weapon System | UI | `weapon_id`, `old_state`, `new_state` |
| `WeaponUsedEvent` | WeaponSystem | Audio | `weapon_id`, `usage_type` |

> **命名规范**：`WeaponAwarenessEvent` 已于 2026-04-12 正式命名，原 `WeaponAwareness` 为已废弃别名

### NPCDeath/NPCKilled 代码使用示例

`NPCDeath` 和 `NPCKilled` 不是独立事件，而是 `NPCStateChangedEvent` 的语义描述。当需要监听 NPC 死亡时，应订阅 `NPCStateChangedEvent` 并检查 `new_state` 字段：

```csharp
// 错误示例（不应使用）
// EventBus.Instance.Subscribe<NPCDeathEvent>(OnNPCDeath);

// 正确示例
EventBus.Instance.Subscribe<NPCStateChangedEvent>(OnNPCStateChanged);

private void OnNPCStateChanged(NPCStateChangedEvent evt)
{
    // 检查是否死亡状态
    if (evt.new_state == WorldState.DEAD)
    {
        Debug.Log($"NPC {evt.npc_id} died");
        // 执行死亡相关逻辑
    }
}

// WorldState 枚举定义（参考 shared-types.md）
public enum WorldState
{
    Alive,
    DEAD,       // NPC 死亡状态
    Unconscious // NPC 失去意识
}
```

**别名映射说明**：
- `NPCDeath` → `NPCStateChangedEvent{new_state=WorldState.DEAD}`
- `NPCKilled` → 同上（强调击杀来源，通常配合 `source_entity_id` 使用）

#### 3.3 Feature Layer 事件

| Event | Owner | Subscribers | Event Data |
|-------|-------|------------|------------|
| `InteractionEvent` | GrittyTakedowns | NPCAI, ClueJournal, SanityRage | `type`, `target_id`, `source`, `result` |
| `InteractionStateChangedEvent` | Gritty Takedowns | UI | `old_state`, `new_state` |
| `KillTagEvent` | GrittyTakedowns | SanityRage | `npc_id`, `kill_tag` |
| `KnowledgeGainedEvent` | GrittyTakedowns | ClueJournal | `npc_id`, `knowledge_list` |
| `ExecutionWitnessedEvent` | GrittyTakedowns | NPCAI | `victim_id`, `witness_id` |
| `DialogueChoice` | GrittyTakedowns | NPCAI | `dialogue_id`, `choice_id` |
| `DialogueResult` | NPCAI | GrittyTakedowns | `dialogue_id`, `success`, `allegiance_change`, `knowledge_gained` |
| `ConfrontationStartRequest` | GrittyTakedowns | NPCAI | `npc_id`, `source` |

> **注意**：`DialogueChoice` 和 `DialogueResult` 的完整定义见 shared-types.md §9.8。注意：ADR-0018 早期版本曾使用 `DialogueChoiceEvent` 和 `DialogueResultEvent` 名称，现已统一为 shared-types.md 中的正式名称。

| `ClueDiscoveredEvent` | ClueJournal | SanityRage | `clue_id`, `category`, `discovery_stage`, `narrative_significance`, `source_id` |
| `LocationRevealedEvent` | ClueJournal | WorldMap | `location_id` |

#### 3.4 Meta Layer 事件

| Event | Owner | Subscribers | Event Data |
|-------|-------|------------|------------|
| `PsychologicalStateEvent` | SanityRage | UI, DPP | `State`, `Reason` |
| `ScreenEffectRequestEvent` | SanityRage, Weather, Lighting, etc. | ScreenEffectsManager | `source_system`, `requester_id`, `effect_type`, `intensity`, `duration`, `timeout`, `priority`, `parameters` |
| `ScreenEffectRevokeEvent` | SanityRage, Weather, Lighting, etc. | ScreenEffectsManager | `source_system`, `requester_id`, `timestamp` |

> **说明**：`ScreenEffectRequestEvent` / `ScreenEffectRevokeEvent` 均为 **Request-Fire-Forget 模式**（见 2.3 节），成对使用。请求方通过 EventBus 发布 Request 激活效果，通过 EventBus 发布 Revoke 撤销效果，**不持有 ScreenEffectsManager 引用**。
>
> **parameters 字段类型**：`ScreenEffectParams` 结构体定义于 [ADR-0023 §ScreenEffectRequestEvent](./adr-0023-screen-effects-system-architecture.md#screeneffectrequestevent)，包含完整的字段定义。本 ADR 仅引用其类型，完整实现细节见 ADR-0023。

> **重要**：`ScreenEffectParams` 的完整定义（字段列表、默认值、序列化方式）统一在 ADR-0023 `ScreenEffectRequestEvent` 结构章节维护。ADR-0018 仅引用其类型，不重复定义。

#### 3.5 Presentation Layer 事件

| Event | Owner | Subscribers | Event Data |
|-------|-------|------------|------------|
| `PauseMenuOpenedEvent` | UI | GameTime, Audio | `reason` |
| `PauseMenuClosedEvent` | UI | GameTime, Audio | `reason` |

#### 3.6 Infrastructure Layer 事件

| Event | Owner | Subscribers | Event Data |
|-------|-------|------------|------------|
| `SaveCompletedEvent` | SaveManager | UI | `slot_id` |
| `LoadCompletedEvent` | SaveManager | WorldMap | `destination`, `destination_type`, `was_successful` |
| `SaveCorruptedEvent` | SaveManager | UI | `slot_id` |
| `AssetLoadedEvent` | ResourceManager | Any System | `Address`, `AssetType`, `EstimatedSizeBytes` |
| `AssetUnloadedEvent` | ResourceManager | Any System | `Address` |
| `AssetReleaseEvent` | ResourceManager | Any System | `Address`, `AssetType` |
| `InputDeviceChangedEvent` | InputManager | Any System | `Device` |
| `InputRemappedEvent` | InputRemapManager | InputManager | `ActionName`, `Device` |

#### 3.7 World Layer 事件

| Event | Owner | Subscribers | Event Data |
|-------|-------|------------|------------|
| `GameHourChangedEvent` | WorldMap System (IGameTimeProvider 实现) | WeatherSystem, LightingSystem, SanityRage* | `current_hour`, `time_scale` |

> **注意**：`SanityRage*` 订阅 GameHourChangedEvent 用于根据游戏内时间（如昼夜节律）调整心理状态恢复速率。具体订阅关系以代码实现为准，此处标注仅供参考。
| `WeatherStateChangedEvent` | WeatherSystem | NPCAI, LOS, SanityRage, ScreenEffects, Audio | `weather_type`, `intensity`, `transition_progress`, `perception_modifier`, `visual_params`, `timestamp` |
| `WeatherForceChangeEvent` | AreaTrigger, GameEvent | WeatherSystem | `target_weather`, `transition_duration`, `source` |
| `LightingStateChangedEvent` | LightingSystem | NPCAI, LOS, SanityRage, ScreenEffects, Audio | `time_of_day`, `global_illumination`, `shadow_intensity`, `perception_modifier`, `visual_params`, `timestamp` |
| `AreaLightingChangedEvent` | AreaLightingDetector | NPCAI, LOS, SanityRage, ScreenEffects | `area_id`, `lighting_state`, `is_player_inside`, `timestamp` |

---

### Query 模式标准化

Query 是同步查询模式，调用方通过独立的 `QueryBus` 单例获取响应：

> **重要澄清**：Query 操作通过独立的 `QueryBus` MonoBehaviour 单例调用，而非通过 `EventBus` 发布。`QueryBus` 与 `EventBus` 是两个独立的单例组件，各自通过 `Awake()` 初始化。虽然两者挂载在同一 GameObject 上以便于管理生命周期，但通信模式完全不同：
> - `EventBus`：Publish/Subscribe 模式（异步/单向）
> - `QueryBus`：Request/Response 模式（同步/双向）
>
> `QueryBus` 是**唯一**通过 `QueryBus.Query<TRequest, TResult>()` 而非 `EventBus.Instance.Publish()` 通信的模式。
>
> **字段命名约定**：Event Data 字段统一使用 PascalCase（与 C# 属性命名一致），避免与 shared-types.md 中的实际结构体定义冲突。

```csharp
// QueryBus.cs
// 注意：QueryBus 是独立的 MonoBehaviour 单例，与 EventBus（ScriptableObject 单例）是两个不同的组件
// 两者通过在同一 GameObject 上挂载便于管理生命周期，但通信模式完全独立
//
// 线程安全：所有 public 方法（Register/Query）均使用 lock 保护，
// 可在 Unity 主线程及 Job System 中安全调用。
public class QueryBus : MonoBehaviour
{
    public static QueryBus Instance { get; private set; }

    private Dictionary<Type, Delegate> _handlers = new();
    private readonly object _lock = new();

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }

    public void Register<TRequest, TResult>(Func<TRequest, TResult> handler)
    {
        lock (_lock)
        {
            _handlers[typeof(TRequest)] = handler;
        }
    }

    public TResult Query<TRequest, TResult>(TRequest request)
    {
        // 先取出 handler，释放锁后再调用，避免 handler 内部递归调用 Query 时死锁
        Delegate handler;
        lock (_lock)
        {
            _handlers.TryGetValue(typeof(TRequest), out handler);
        }
        if (handler != null)
            return ((Func<TRequest, TResult>)handler)(request);
        throw new QueryHandlerNotFoundException(typeof(TRequest));
    }
}

/// <summary>
/// Query 处理器未找到异常
/// </summary>
public class QueryHandlerNotFoundException : Exception
{
    public Type RequestType { get; }

    public QueryHandlerNotFoundException(Type requestType)
        : base($"No handler registered for Query type: {requestType.Name}")
    {
        RequestType = requestType;
    }
}
```

**已定义的 Query**：

| Query | Handler | Request Data | Response Data |
|-------|---------|--------------|---------------|
| `QueryNPCIdentity` | NPCAI | `npc_id` | `NPCIdentityType` |
| `QueryAlertState` | NPCAI | `npc_id` | `AlertState` |
| `QueryNPCSizeCategory` | NPCAI | `npc_id` | `NPCSizeCategory` |
| `QueryWeaponData` | WeaponSystem | `weapon_id` | `WeaponData` |
| `QueryCurrentWeather` | WeatherSystem | — | `WeatherData` |
| `QueryEffectiveVisionRange` | LOSSystem | `npc_id` | `float` (vision range in meters) |

---

### Request 模式标准化

Request 是 Fire-and-Forget 模式，发送方不等待响应：

```csharp
// 使用 EventBus.Publish 发送，与 Event 相同的发布接口
// 但语义上区分：Request 的处理方会产生副作用
EventBus.Instance.Publish(new DamageRequest { ... });
EventBus.Instance.Publish(new ConfrontationStartRequest { ... });
```

---

### 事件 Data 字段类型定义

所有 Event Data 字段遵循以下类型约定：

| 字段后缀 | 类型 | 说明 |
|----------|------|------|
| `_id` | `int` | 实体 ID（玩家、NPC、武器等） |
| `_type` | `Enum` | 枚举类型（如 `DamageType`、`NoiseType`） |
| `_state` | `Enum` | 状态枚举（如 `AlertState`、`WorldState`） |
| `_position` | `Vector3` | 世界空间坐标 |
| `_radius` | `float` | 范围半径（米） |
| `_duration` | `float` | 持续时间（秒） |
| `_amount` / `_value` | `float` | 数值量 |
| `_ratio` | `float` | 比例值（0.0-1.0） |
| `_progress` | `float` | 进度值（0.0-1.0） |
| `_timestamp` | `float` | 时间戳（游戏时间秒数） |
| `_time_scale` | `float` | 时间缩放倍率 |
| `_intensity` | `float` | 强度值（0.0-1.0） |
| `_priority` | `int` | 优先级（数值越大优先级越高） |
| `_source` | `string` | 来源系统标识 |

> **注意**：具体字段类型以 shared-types.md 中的实际结构体定义为准，此表仅作为命名约定参考。

### 事件向后兼容规则

1. **禁止删除已发布事件的字段**：只能新增可选字段
2. **禁止修改字段类型**：如需修改，创建新事件
3. **事件版本通过 shared-types.md 版本追踪**：修改 shared-types 时更新其版本号
4. **废弃事件使用 `[Obsolete]` 标记**：并在新 ADR 中说明替代方案

### 事件命名规范

事件命名应遵循 ADR-0001 定义的 `Subject + Did + Context + Event` 格式：
- ✅ `PlayerDamagedEvent` — 符合规范
- ✅ `NPCStateChangedEvent` — 符合规范
- ⚠️ `NoiseEvent` — 应改为 `NoiseMadeEvent` 或 `PlayerDidMakeNoiseEvent`
- ⚠️ `WeaponAwareness` — 应改为 `WeaponAwarenessEvent`
- ℹ️ `NPCDeath` / `NPCKilled` — 非独立事件，为 `NPCStateChangedEvent{new_state=WorldState.DEAD}` 的语义描述，无需定义别名

**迁移计划**：
1. **Phase 1（已完成 2026-04-12）**：正式命名
   - `NoiseEvent` → `NoiseMadeEvent`（已废弃别名，仅文档记录）
   - `WeaponAwareness` → `WeaponAwarenessEvent`（已废弃别名，仅文档记录）
   - `NPCDeath` / `NPCKilled` — 确认非独立事件，无需迁移（见上方命名规范说明）
2. **Phase 2（首次发布前）**：统一代码中的使用，移除别名
3. **强制执行**：Code Review 检查新增事件命名是否符合规范

### 别名实现（过渡期兼容）

```csharp
// EventBusExtensions.cs
// 临时别名类，提供命名兼容（首次发布后移除）

/// <summary>
/// [已废弃] 请使用 NoiseMadeEvent
/// </summary>
[Obsolete("Use NoiseMadeEvent instead. Removed after v1.0.")]
public struct NoiseEvent
{
    public Vector3 position;
    public float radius;
    public NoiseType noise_type;
    public float duration;
    public bool can_interrupt;
    public int source_entity_id;

    /// <summary>
    /// 隐式转换到新名称
    /// </summary>
    public static implicit operator NoiseMadeEvent(NoiseEvent old)
    {
        return new NoiseMadeEvent
        {
            position = old.position,
            radius = old.radius,
            noise_type = old.noise_type,
            duration = old.duration,
            can_interrupt = old.can_interrupt,
            source_entity_id = old.source_entity_id
        };
    }
}

/// <summary>
/// [已废弃] 请使用 WeaponAwarenessEvent
/// </summary>
[Obsolete("Use WeaponAwarenessEvent instead. Removed after v1.0.")]
public struct WeaponAwareness
{
    public int weapon_id;
    public Vector3 position;
    public WeaponType weapon_type;

    public static implicit operator WeaponAwarenessEvent(WeaponAwareness old)
    {
        return new WeaponAwarenessEvent
        {
            weapon_id = old.weapon_id,
            position = old.position,
            weapon_type = old.weapon_type
        };
    }
}
```

---

### Debug 可视化接口

```csharp
// EventBusDebug.cs
#if UNITY_EDITOR
using UnityEditor;
#endif

public class EventBusDebug : MonoBehaviour
{
    private Queue<EventRecord> _eventHistory = new();

    [Header("Debug Settings")]
    [Tooltip("最大历史记录条数，可通过 Inspector 配置")]
    [SerializeField] private int _maxHistory = 500;  // 增大历史记录容量，便于复杂调试

    // 编译宏控制：仅在 Unity Editor 或 DEVELOPMENT_BUILD 时启用
    // 注意：EventBusDebug 核心逻辑通过编译时 #if 分支排除，在其他构建配置下完全不产生开销

    public struct EventRecord
    {
        public Type EventType;
        public string EventName;
        public string Source;
        public float Timestamp;
        public string PayloadSummary;  // JSON 摘要（前 200 字符）
    }

    public IReadOnlyCollection<EventRecord> EventHistory => _eventHistory;

    /// <summary>
    /// 最大历史记录条数（供 Editor Inspector 使用）
    /// </summary>
    public int MaxHistory => _maxHistory;

    public void RecordEvent<T>(T eventData, string source)
    {
#if !UNITY_EDITOR && !DEVELOPMENT_BUILD
        return;  // 编译时已确定不启用，避免运行时分支
#endif

        if (_eventHistory.Count >= _maxHistory)
            _eventHistory.Dequeue();  // Queue.Dequeue 是 O(1)，避免 List.RemoveAt(0) 的 O(n) 开销

        var json = JsonUtility.ToJson(eventData);
        _eventHistory.Enqueue(new EventRecord
        {
            EventType = typeof(T),
            EventName = typeof(T).Name,
            Source = source,
            Timestamp = Time.time,
            PayloadSummary = json?.Length > 200 ? json.Substring(0, 200) : json
        });
    }

    /// <summary>
    /// 清除历史记录（调试用）
    /// </summary>
    public void ClearHistory()
    {
        _eventHistory.Clear();
    }

    /// <summary>
    /// 导出历史记录为 JSON（用于自动化测试）
    /// JsonUtility 不支持顶层 Collection，需包裹在对象中
    /// </summary>
    [Serializable]
    private class HistoryExport { public List<EventRecord> events; }

    public string ExportHistoryAsJson()
    {
        return JsonUtility.ToJson(new HistoryExport { events = _eventHistory.ToList() });
    }
}

// ==================== Unity Editor Inspector 可视化面板 ====================
#if UNITY_EDITOR
[CustomEditor(typeof(EventBusDebug))]
public class EventBusDebugEditor : Editor
{
    private EventBusDebug _target;
    private string _filter = "";
    private Vector2 _scrollPosition;

    void OnEnable()
    {
        _target = (EventBusDebug)target;
    }

    public override void OnInspectorGUI()
    {
        // 标题
        EditorGUILayout.LabelField("Event Bus Debugger", EditorStyles.boldLabel);
        EditorGUILayout.Space();

        // 工具栏
        EditorGUILayout.BeginHorizontal();
        if (GUILayout.Button("Clear History", GUILayout.Width(100)))
        {
            _target.ClearHistory();
        }

        _filter = EditorGUILayout.TextField("Filter:", _filter, GUILayout.Width(200));

        int eventCount = _target.EventHistory.Count;
        EditorGUILayout.LabelField($"Events: {eventCount}/{_target.MaxHistory}");
        EditorGUILayout.EndHorizontal();

        EditorGUILayout.Space();

        // 事件列表
        _scrollPosition = EditorGUILayout.BeginScrollView(_scrollPosition, GUILayout.Height(400));

        // IReadOnlyCollection<T> 不支持索引访问，转为数组后逆序遍历显示（最新事件在最上方）
        var history = _target.EventHistory.ToArray();
        for (int i = history.Length - 1; i >= 0; i--)
        {
            var record = history[i];

            // 过滤器
            if (!string.IsNullOrEmpty(_filter) &&
                !record.EventName.Contains(_filter, StringComparison.OrdinalIgnoreCase) &&
                !record.Source.Contains(_filter, StringComparison.OrdinalIgnoreCase))
            {
                continue;
            }

            // 事件条目
            EditorGUILayout.BeginHorizontal("box");
            {
                // 时间戳
                EditorGUILayout.LabelField($"{record.Timestamp:F2}s", GUILayout.Width(60));

                // 事件类型（带颜色）
                GUI.color = GetEventColor(record.EventName);
                EditorGUILayout.LabelField(record.EventName, EditorStyles.boldLabel, GUILayout.Width(200));
                GUI.color = Color.white;

                // 来源
                EditorGUILayout.LabelField($"[{record.Source}]", GUILayout.Width(100));

                // 摘要（可展开）
                bool hasPayload = !string.IsNullOrEmpty(record.PayloadSummary);
                if (hasPayload)
                {
                    EditorGUILayout.LabelField(record.PayloadSummary.Length > 50
                        ? record.PayloadSummary.Substring(0, 50) + "..."
                        : record.PayloadSummary);
                }
                else
                {
                    EditorGUILayout.LabelField("(no payload)");
                }
            }
            EditorGUILayout.EndHorizontal();
        }

        EditorGUILayout.EndScrollView();

        // 导出按钮
        EditorGUILayout.Space();
        if (GUILayout.Button("Export History as JSON"))
        {
            string path = EditorUtility.SaveFilePanel(
                "Export Event History",
                Application.dataPath,
                $"event_history_{DateTime.Now:yyyyMMdd_HHmmss}.json",
                "json");

            if (!string.IsNullOrEmpty(path))
            {
                System.IO.File.WriteAllText(path, _target.ExportHistoryAsJson());
                EditorUtility.DisplayDialog("Export Complete", $"Exported to {path}", "OK");
            }
        }
    }

    /// <summary>
    /// 根据事件类型返回颜色（便于快速识别）
    /// </summary>
    private Color GetEventColor(string eventName)
    {
        if (eventName.EndsWith("Request")) return Color.yellow;
        if (eventName.StartsWith("Query")) return Color.cyan;
        if (eventName.Contains("Changed")) return Color.green;
        if (eventName.Contains("Damaged") || eventName.Contains("Killed") || eventName.Contains("Death"))
            return Color.red;
        if (eventName.Contains("Alert")) return Color.magenta;
        return Color.white;
    }
}
#endif
```

### World Layer 与 Meta Layer 事件订阅交叉索引

| 事件 | World Layer 订阅者 | Meta Layer 订阅者 |
|------|-------------------|-------------------|
| `GameHourChangedEvent` | WeatherSystem, LightingSystem | SanityRage* |
| `WeatherStateChangedEvent` | NPCAI, LOSSystem, Audio | SanityRage, ScreenEffects |
| `LightingStateChangedEvent` | NPCAI, LOSSystem, Audio | SanityRage, ScreenEffects |
| `AreaLightingChangedEvent` | NPCAI, LOSSystem | SanityRage, ScreenEffects |
| `ScreenEffectRequestEvent` | — | ScreenEffectsManager（唯一订阅者，统一处理） |
| `ScreenEffectRevokeEvent` | — | ScreenEffectsManager（唯一订阅者，触发内部撤销） |

> **SanityRage 订阅 GameHourChangedEvent 说明**：SanityRage 根据游戏内时间调整心理状态恢复速率。例如：夜间恢复速率降低，黎明后逐渐恢复。具体逻辑以 ADR-0017 和代码实现为准。

> **说明**：`ScreenEffectRequestEvent` / `ScreenEffectRevokeEvent` 由 SanityRage、Weather、Lighting 等系统发布，ScreenEffectsManager 统一处理，无需 World Layer 系统直接订阅。`GameHourChangedEvent` 由 WorldMap System 发布，Weather/Lighting 系统订阅，用于时段切换和天气生成触发。

### 补充定义的事件

以下事件在本 ADR 之前未被正式定义，此处补充其完整结构：

#### LoadingScreenRequestEvent

**Owner**: WorldMap System
**Subscribers**: UI System
**Layer**: Infrastructure Layer（跨层通信事件）

```csharp
public struct LoadingScreenRequestEvent
{
    public string destination;           // 目标场景/位置
    public string destination_type;      // "city" / "area" / "loading_tip"
    public string source_location;       // 来源位置
}
```

**定义位置**：`Assets/Game/Infrastructure/EventBus/Events/LoadingScreenRequestEvent.cs`

#### QuerySoundSourceScreenPosition

**Owner**: LOSSystem
**Type**: Query（同步查询）
**Subscribers**: N/A（Query Handler 直连）

```csharp
// Query 请求
public struct QuerySoundSourceScreenPosition
{
    public int NpcId;
}

// Query 响应（通过 QueryBus 返回 Vector3）
public struct QuerySoundSourceScreenPositionResponse
{
    public Vector3 ScreenPosition;  // 屏幕空间位置
    public bool IsValid;             // NPC 是否可见
}

// 注意：Query 模式使用 QueryBus.Query<QuerySoundSourceScreenPosition, QuerySoundSourceScreenPositionResponse>()
// 返回值为屏幕空间位置和有效性标志
```

---

## Alternatives Considered

### Alternative 1: 保持现状（无独立 ICD）

- **描述**：事件定义继续分散在各 ADR 中，通过 shared-types.md 引用
- **Pros**：现有工作流不变
- **Cons**：
  - 无法快速确认完整订阅关系
  - 事件命名规范难以强制执行
  - 新开发者难以理解全系统事件流
- **拒绝理由**：
  - 已有证据表明接口不一致（GrittyTakedowns 误以为 AlertStateChanged 未定义）
  - 15+ 系统的联调需要统一索引

### Alternative 2: 完整中介者模式

- **描述**：引入完整中介者，所有通信必须通过中介者
- **Pros**：高度解耦
- **Cons**：过度设计，同 ADR-0001 的评估结果
- **拒绝理由**：Event Bus 已足够，且已实现

---

## Consequences

### Positive

- **完整索引**：开发者可快速查找任何事件的定义和用途
- **明确订阅关系**：避免遗漏订阅导致的 bug
- **标准化 Query/Request**：系统间调用有一致模式
- **Debug 友好**：事件历史记录便于排查问题

### Negative

- **维护负担**：新增/修改事件需要同步更新 ICD
- **文档同步延迟**：代码和文档可能暂时不一致

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **文档过期** | 代码变更未同步更新 ICD | CI 阶段检查 `git diff` 是否同步修改 ICD |
| **事件泛滥** | 大量小事件难以管理 | 定期审查，合并相似事件 |

---

## Performance Implications

| 指标 | 影响 | 说明 |
|------|------|------|
| **CPU** | 无直接影响 | Debug 记录有轻微开销，可通过编译宏关闭 |
| **Memory** | Debug 模式增加 | 100 条记录约 50KB，可控制 |

---

## Migration Plan

### Phase 1: 建立索引
- [ ] 迁移 shared-types.md 中所有事件定义到本 ICD
- [ ] 按 Owner 系统分组
- [ ] 补充缺失的 Subscribers 列表

### Phase 2: Query 标准化
- [ ] 实现 QueryBus
- [ ] 注册所有已定义的 Query

### Phase 3: Debug 工具
- [ ] 实现 EventBusDebug
- [ ] 实现 Editor Inspector 面板

### Phase 4: 规范执行
- [ ] Code Review 检查新增事件是否同步更新 ICD
- [ ] CI 阶段检查（可选）

### Phase 5: 命名规范迁移
- [x] 将 `NoiseEvent` 迁移为 `NoiseMadeEvent` 并添加别名映射（见上方"事件命名规范"章节）
- [x] 将 `WeaponAwareness` 迁移为 `WeaponAwarenessEvent` 并添加别名映射（见上方"事件命名规范"章节）
- [x] 确认 `MovementStateChangedEvent` 是否为 `PlayerMovementStateChangedEvent` 的别名——已确认：shared-types.md 中仅存在 `PlayerMovementStateChangedEvent`，无独立别名

---

## Validation Criteria

1. **完整性**：shared-types.md 中的每个事件类型都映射到 ICD 中的一个条目
2. **正确性**：每个条目的 Owner/Subscribers 与代码实现一致
3. **可追溯性**：[ ] 通过 `EventBusICDValidator` 单元测试验证所有 `EventBus.Instance.Publish` 调用在 ICD 中有对应条目（测试自动扫描代码并与 ICD 交叉验证）— **TODO：实现指南见附录 A**
4. **Query 覆盖**：所有系统间同步查询都能找到对应 Query Handler
5. **命名合规**：[ ] 新增事件通过 `EventNamingConventionTest` 测试检查是否符合 `Subject + Did + Context + Event` 格式 — **TODO：实现指南见附录 B**

> **EventNamingConventionTest 检查规则**：
> - 事件类型名必须以 `Event` 结尾
> - 名称必须由 2+ 个单词组成（避免单数无意义命名）
> - 对于表示动作的事件，动词应使用过去式（Did）或现在分词（Doing）形式
> - Query 类型必须以 `Query` 开头
> - Request 类型必须以 `Request` 结尾
> - 禁用示例：`StateChanged`（应为 `StateDidChangeEvent` 或 `StateChangedEvent`）、`Killed`（应为 `WasKilledEvent` 或 `KillEvent`）

### 附录 A：EventBusICDValidator 实现指南

```csharp
// EventBusICDValidator.cs
// 放置于 Assets/Game/Tests/Validation/
// 通过反射扫描所有继承自结构体的事件类型，验证 ICD 索引完整性

public class EventBusICDValidator
{
    private static readonly HashSet<string> IcdEventNames = new()
    {
        // 从 ICD 文档解析的事件名列表
        "PlayerDamagedEvent", "PlayerMovementStateChangedEvent", "NoiseMadeEvent",
        "DamageRequest", "ExplosionEvent", "NPCStateChangedEvent",
        "AlertStateChangedEvent", "ArmorDestroyedEvent", "ExplosionAlertEvent",
        // ... 完整列表见 ICD §3
    };

    [UnityTest]
    public IEnumerator AllPublishedEventsMustBeInICD()
    {
        var eventBusType = typeof(EventBus);
        var publishMethods = eventBusType.GetMethods(BindingFlags.Public | BindingFlags.Static);

        foreach (var method in publishMethods)
        {
            if (method.Name == "Publish")
            {
                var genericMethod = method.GetGenericArguments();
                // 验证事件类型在 ICD 中有定义
            }
        }
        yield return null;
    }
}
```

### 附录 B：EventNamingConventionTest 实现指南

```csharp
// EventNamingConventionTest.cs
// 放置于 Assets/Game/Tests/Validation/
// 扫描所有事件类型，验证命名是否符合规范

public class EventNamingConventionTest
{
    private static readonly Regex EventNamePattern = new(@"^(Did|Is|Has|Will|Can)\w+Event$|^[A-Z][a-zA-Z]+(Did|Has|Is|Will|Can)\w*Event$");

    [Test]
    public void AllEventTypesMustEndWithEvent()
    {
        var eventTypes = Assembly.GetExecutingAssembly()
            .GetTypes()
            .Where(t => t.IsValueType && t.Name.EndsWith("Event"));

        foreach (var type in eventTypes)
        {
            Assert.IsTrue(
                EventNamePattern.IsMatch(type.Name),
                $"Event {type.Name} does not match naming convention");
        }
    }
}
```

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 本 ICD 的基础，EventBus 单例实现规范
- [shared-types.md](./shared-types.md) — 事件类型的代码定义位置（包含废弃警告）
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — 事件跨层通信的层级规范
- [ADR-0021: 天气系统](./adr-0021-weather-system-architecture.md) — WeatherStateChangedEvent, WeatherForceChangeEvent 定义
- [ADR-0022: 光照系统](./adr-0022-lighting-system-architecture.md) — LightingStateChangedEvent, AreaLightingChangedEvent 定义
- [ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — ScreenEffectRequestEvent / ScreenEffectRevokeEvent 统一接口定义
- [shared-types.md §22](./shared-types.md#22-游戏时间接口与-world-layer-时间事件) — IGameTimeProvider、GameHourChangedEvent、ScreenEffectRevokeEvent 权威类型定义
