# ADR-0021: 天气系统 (Weather System) 架构决策

## Status
**Proposed**

## Date
2026-04-11

## Last Updated
2026-04-15 (v3: 统一PerceptionModifier字段命名、补充WeatherForceChangeEvent时序规则、补充PitchBlack区域Fog视觉效果；补充 EnvironmentalEvent 订阅说明) [已修复]

## Context

### Problem Statement

天气系统是《断绝：罪恶之源》World Layer 的核心组成，通过环境氛围强化沉浸感并对多个下游系统产生战术影响。系统需要解决：

1. **多系统联动**：天气变化需要影响 NPC AI 感知（视野折扣）、LOS 系统（声音/视觉传播）、Sanity/Rage（氛围反馈）、Dynamic Post-Processing（天气视觉效果）、Audio（环境音变化）
2. **平滑过渡**：天气切换不能突变，需要渐变过渡避免视觉跳跃
3. **性能考量**：天气效果需要与渲染管线解耦，不能每帧更新天气状态
4. **独立运作**：World Layer 系统不调用其他游戏系统，仅通过 Event Bus 广播环境变化

### Constraints

- **架构约束**：遵循 ADR-0003 World Layer 定义 — 代码放在 Core/Environment 子目录，通过 Event Bus 与其他层通信
- **性能约束**：天气效果更新频率 <= 1Hz，避免每帧计算
- **兼容性约束**：支持 PC & PS5 平台
- **美术约束**：提供至少 4 种基础天气类型，每种有独特的视觉和听觉签名

### Requirements

- **必须**：定义 4+ 种天气类型（Clear / Rain / Fog / Storm / Snow 等）
- **必须**：定义 WeatherStateChangedEvent 事件结构
- **必须**：定义天气对 NPC AI 感知的影响参数（视野折扣、听觉折扣）
- **必须**：定义天气对 LOS 系统的影响参数（声音传播折扣、可视距离折扣）
- **必须**：定义天气过渡机制（渐变时长、插值曲线）
- **必须**：通过 Event Bus 广播天气状态，不直接调用下游系统

---

## Decision

### 架构决策

采用**事件广播 + 参数化影响系数**架构：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Weather System 架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【World Layer — 不参与分层调用链，独立运作】                                   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                 WeatherSystemManager (MonoBehaviour)                       │   │
│  │  - 持有当前天气状态 (WeatherState)                                        │   │
│  │  - 管理天气过渡动画 (Lerp over Duration)                                  │   │
│  │  - 每秒更新一次感知系数（而非每帧）                                        │   │
│  │  - 发布 WeatherStateChangedEvent（唯一输出）                              │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      WeatherStateChangedEvent                          │   │
│  │  {                                                                      │   │
│  │    weather_type: WeatherType,      // 当前天气                          │   │
│  │    intensity: float,               // 强度 0.0~1.0                      │   │
│  │    transition_progress: float,      // 过渡进度 0.0~1.0                  │   │
│  │    perception_modifier: PerceptionModifier,  // 感知系数               │   │
│  │    timestamp: float                 // 事件时间戳                        │   │
│  │  }                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│          ┌─────────────────────────┼─────────────────────────┐              │
│          ▼                         ▼                         ▼              │
│  ┌───────────────┐        ┌───────────────┐        ┌───────────────┐         │
│  │   NPC AI      │        │   LOS System  │        │ Sanity/Rage   │         │
│  │   System      │        │               │        │               │         │
│  │               │        │               │        │               │         │
│  │ ◆ 视野折扣    │        │ ◆ 声音折扣    │        │ ◆ 氛围效果   │         │
│  │ ◆ 听觉折扣    │        │ ◆ 可视距离   │        │ ◆ 暗角/噪点  │         │
│  └───────────────┘        └───────────────┘        └───────────────┘         │
│                                                                              │
│  ┌───────────────┐        ┌───────────────┐                                 │
│  │ Dynamic Post  │        │   Audio       │                                 │
│  │ Processing    │        │   System      │                                 │
│  │               │        │               │                                 │
│  │ ◆ 雨滴镜头    │        │ ◆ 雨声/雷声  │                                 │
│  │ ◆ 雾气效果    │        │ ◆ 风声层次   │                                 │
│  │ ◆ 闪电闪烁    │        │               │                                 │
│  └───────────────┘        └───────────────┘                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### WeatherType 枚举

```csharp
/// <summary>
/// 天气类型枚举
/// </summary>
public enum WeatherType
{
    Clear,    // 晴朗 — 无修改
    Rain,     // 雨天 — 视野/听觉折扣，雨滴后处理
    Fog,      // 雾天 — 视野严重折扣，朦胧后处理
    Storm,    // 暴风雨 — 听觉严重折扣，闪电/雨/风
    Snow      // 雪天 — 视野/听觉中等折扣，雪花后处理
}
```

**定义位置**：`Assets/Game/Core/Environment/Weather/Types/WeatherType.cs`

### 天气转换规则

天气状态机采用**有限状态机（FSM）** + **时间/区域触发**的混合模式：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Weather State Machine                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│    ┌─────────┐  transition  ┌─────────┐  transition  ┌─────────┐          │
│    │  Clear  │ ───────────► │   Fog   │ ───────────► │  Storm  │          │
│    └─────────┘ ◄─────────── └─────────┘ ◄─────────── └─────────┘          │
│         ▲              │              │              │                      │
│         │              ▼              ▼              │                      │
│         │         ┌─────────┐  ┌─────────┐           │                      │
│         └─────────│  Rain   │◄─│  Snow   │───────────┘                      │
│                   └─────────┘  └─────────┘                                  │
│                                                                              │
│  触发模式：                                                                   │
│  - 时间驱动：按游戏时间表生成（如每天有固定天气模式）                          │
│  - 区域触发：进入特定区域时强制天气（如进入洞穴区域强制 Clear）                 │
│  - 事件触发：游戏事件（如剧情触发）强制切换天气                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### WeatherTransitionConfig 天气过渡配置

```csharp
/// <summary>
/// 天气过渡配置
/// 定义天气之间的过渡规则
/// </summary>
[System.Serializable]
public struct WeatherTransitionConfig
{
    /// <summary>目标天气类型</summary>
    public WeatherType targetWeather;

    /// <summary>过渡时长（秒）</summary>
    public float transitionDuration;

    /// <summary>触发权重（用于随机选择）</summary>
    public float weight;

    /// <summary>触发条件（可选）</summary>
    public string requiredGameFlag;
}

/// <summary>
/// 天气转换规则表
/// 每张表对应一个源天气类型，存储该源天气可转换到的所有目标配置
/// 源天气类型由资产文件名决定（如 WeatherTransitionTable_Clear.asset → Clear）
/// </summary>
[CreateAssetMenu(fileName = "WeatherTransitionTable_Clear", menuName = "Game/Weather/Transition Table")]
public class WeatherTransitionTable : ScriptableObject
{
    /// <summary>
    /// 此表对应的源天气
    /// 由资产文件名派生（如 "WeatherTransitionTable_Clear" → WeatherType.Clear）
    /// Inspector 中只读显示，不编辑
    /// </summary>
    public WeatherType SourceWeather => DeriveSourceWeather();

    /// <summary>
    /// 从资产文件名派生源天气类型
    /// 文件名格式：WeatherTransitionTable_{WeatherType}
    /// </summary>
    private WeatherType DeriveSourceWeather()
    {
        // 从资产文件名提取源天气类型
        // 文件名格式：WeatherTransitionTable_{WeatherType}
        string assetName = name;
        int underscoreIndex = assetName.LastIndexOf('_');
        if (underscoreIndex >= 0 && underscoreIndex < assetName.Length - 1)
        {
            string weatherName = assetName.Substring(underscoreIndex + 1);
            if (Enum.TryParse<WeatherType>(weatherName, out var result))
                return result;
        }
        // Fallback：尝试从 _transitions 中推断（如果所有配置的 targetWeather 都相同）
        if (_transitions != null && _transitions.Count > 0)
        {
            return _transitions[0].targetWeather;
        }
        Debug.LogWarning($"[WeatherTransitionTable] 无法从文件名 '{assetName}' 派生源天气类型，请重命名资产文件（格式：WeatherTransitionTable_{{WeatherType}}）");
        return WeatherType.Clear;
    }

    /// <summary>
    /// 从源天气出发可转换到的目标配置列表
    /// </summary>
    [SerializeField] private List<WeatherTransitionConfig> _transitions;

    /// <summary>
    /// 获取此表对应的源天气的所有可能转换目标
    /// </summary>
    /// <returns>转换配置数组</returns>
    public WeatherTransitionConfig[] GetPossibleTransitions()
    {
        return _transitions?.ToArray() ?? Array.Empty<WeatherTransitionConfig>();
    }
}
```

**定义位置**：`Assets/Game/Core/Environment/Weather/Config/WeatherTransitionTable.cs`

**资产命名规范**：
- 文件名格式：`WeatherTransitionTable_{SourceWeather}.asset`
- 示例：`WeatherTransitionTable_Clear.asset`、`WeatherTransitionTable_Rain.asset`、`WeatherTransitionTable_Storm.asset`
- 源天气类型由文件名决定，不得在 Inspector 中手动选择

#### 天气生成器接口

```csharp
/// <summary>
/// 天气生成器接口
/// 支持多种天气生成策略：时间表、随机、区域触发
/// </summary>
public interface IWeatherGenerator
{
    /// <summary>
    /// 生成下一个天气状态
    /// </summary>
    /// <param name="currentWeather">当前天气</param>
    /// <param name="gameHour">当前游戏时间（小时，0.0~24.0）
    ///
    /// <para><b>预留参数：</b></para>
    /// <list type="bullet">
    ///   <item>当前实现中此参数未使用，仅用于日志记录和调试</item>
    ///   <item>预留用于未来扩展：按游戏时间表生成天气（如夜间更易生成 Storm）</item>
    ///   <item>实现者如需使用此参数，应在方法体开头添加注释说明</item>
    /// </list>
    /// </param>
    /// <returns>下一个天气配置（含目标天气和过渡时长）</returns>
    WeatherTransitionConfig GenerateNextWeather(WeatherType currentWeather, float gameHour);
}

/// <summary>
/// 天气转换规则映射
/// Key: 源天气类型，Value: 该源天气可转换到的目标配置数组
/// 由 WeatherSystemManager 在初始化时从 WeatherTransitionTable[] 资产构建
/// </summary>
public class WeatherTransitionMap : Dictionary<WeatherType, WeatherTransitionConfig[]>
{
    /// <summary>
    /// 从 ScriptableObject 数组构建映射表
    /// WeatherSystemManager.Awake() 中调用：
    ///   _transitionMap = WeatherTransitionMap.Build(_transitionTables);
    /// </summary>
    public static WeatherTransitionMap Build(WeatherTransitionTable[] tables)
    {
        var map = new WeatherTransitionMap();
        foreach (var table in tables)
        {
            // 每张表的 SourceWeather 作为 key，GetPossibleTransitions() 返回的数组作为 value
            var transitions = table.GetPossibleTransitions();
            if (transitions.Length > 0)
                map[table.SourceWeather] = transitions;
        }

        // 空数组警告日志：帮助开发者快速定位配置问题
        if (map.Count == 0)
        {
            Debug.LogWarning("[WeatherTransitionMap] 警告: 构建的转换映射表为空。"
                + "请确保传入有效的 WeatherTransitionTable[] 资产。"
                + "WeatherSystem 将 fallback 到保持当前天气的保底行为。");
        }

        return map;
    }
}

/// <summary>
/// 时间表驱动天气生成器
/// 不继承 MonoBehaviour，配置通过构造函数注入（而非 [SerializeField]）
/// 由 WeatherSystemManager 持有并在 Awake() 中实例化
/// </summary>
public class TimedWeatherGenerator : IWeatherGenerator
{
    private readonly float _minWeatherDuration;  // 最少持续 2 分钟
    private readonly float _maxWeatherDuration;  // 最多持续 10 分钟
    private readonly WeatherTransitionMap _transitionMap;

    public TimedWeatherGenerator(float minDuration, float maxDuration, WeatherTransitionMap transitionMap)
    {
        _minWeatherDuration = minDuration;
        _maxWeatherDuration = maxDuration;
        _transitionMap = transitionMap;
    }

    public WeatherTransitionConfig GenerateNextWeather(WeatherType currentWeather, float gameHour)
    {
        // 直接从字典获取转换配置
        if (!_transitionMap.TryGetValue(currentWeather, out var transitions) || transitions.Length == 0)
        {
            // Fallback: 保持当前天气
            return new WeatherTransitionConfig
            {
                targetWeather = currentWeather,
                transitionDuration = _minWeatherDuration
            };
        }

        // 按权重随机选择
        float totalWeight = transitions.Sum(t => t.weight);
        float random = UnityEngine.Random.Range(0f, totalWeight);

        float cumulative = 0f;
        foreach (var transition in transitions)
        {
            cumulative += transition.weight;
            if (random <= cumulative)
            {
                float duration = UnityEngine.Random.Range(_minWeatherDuration, _maxWeatherDuration);
                return new WeatherTransitionConfig
                {
                    targetWeather = transition.targetWeather,
                    transitionDuration = duration,
                    weight = transition.weight
                };
            }
        }

        // Fallback: 保持当前天气
        return new WeatherTransitionConfig
        {
            targetWeather = currentWeather,
            transitionDuration = _minWeatherDuration
        };
    }
}
```

**定义位置**：`Assets/Game/Core/Environment/Weather/Config/WeatherTransitionTable.cs`

### PerceptionModifier 结构

```csharp
/// <summary>
/// 天气对感知的影响系数
/// 由 WeatherSystem 计算，广播给下游系统
/// </summary>
public struct PerceptionModifier
{
    /// <summary>视野距离乘数（0.0~1.0），影响 NPC 的感知范围</summary>
    public float visionDistanceMultiplier;

    /// <summary>天气光照灵敏度乘数（0.0~1.0），影响 NPC 对光照变化的感知阈值（与 Lighting.lightSensitivityMultiplier 区分命名）[已修复]</summary>
    public float weatherLightSensitivityMultiplier;

    /// <summary>听觉灵敏度乘数（0.0~1.0），影响 NPC 对噪声的感知阈值</summary>
    public float hearingSensitivityMultiplier;

    /// <summary>声音传播距离乘数（0.0~1.0），影响噪声事件的有效半径</summary>
    public float soundPropagationMultiplier;

    /// <summary>
    /// 在两个感知系数之间按强度插值
    /// 使用 SmoothStep 曲线确保平滑过渡
    /// </summary>
    /// <param name="from">起始感知系数</param>
    /// <param name="to">目标感知系数</param>
    /// <param name="t">插值因子（0.0~1.0）</param>
    /// <returns>插值后的感知系数</returns>
    public static PerceptionModifier Lerp(PerceptionModifier from, PerceptionModifier to, float t)
    {
        // SmoothStep 曲线：t * t * (3.0 - 2.0 * t)
        float smoothT = t * t * (3.0f - 2.0f * t);

        return new PerceptionModifier
        {
            visionDistanceMultiplier = Mathf.Lerp(from.visionDistanceMultiplier, to.visionDistanceMultiplier, smoothT),
            weatherLightSensitivityMultiplier = Mathf.Lerp(from.weatherLightSensitivityMultiplier, to.weatherLightSensitivityMultiplier, smoothT),
            hearingSensitivityMultiplier = Mathf.Lerp(from.hearingSensitivityMultiplier, to.hearingSensitivityMultiplier, smoothT),
            soundPropagationMultiplier = Mathf.Lerp(from.soundPropagationMultiplier, to.soundPropagationMultiplier, smoothT)
        };
    }

    /// <summary>
    /// 获取指定天气类型和强度对应的感知系数
    /// 用于计算中间强度的感知系数（通过 Lerp 插值）
    /// </summary>
    /// <param name="weatherType">天气类型</param>
    /// <param name="intensity">天气强度（0.5 或 1.0 档位）</param>
    /// <returns>感知系数</returns>
    public static PerceptionModifier GetFor(WeatherType weatherType, float intensity)
    {
        // 定义各天气类型在 0.5 和 1.0 强度下的感知系数
        return (weatherType, intensity >= 0.75f) switch
        {
            (WeatherType.Clear, _) => new PerceptionModifier
            {
                visionDistanceMultiplier = 1.0f,
                weatherLightSensitivityMultiplier = 1.0f,
                hearingSensitivityMultiplier = 1.0f,
                soundPropagationMultiplier = 1.0f
            },
            (WeatherType.Rain, false) => new PerceptionModifier
            {
                visionDistanceMultiplier = 0.8f,
                weatherLightSensitivityMultiplier = 0.9f,
                hearingSensitivityMultiplier = 0.6f,
                soundPropagationMultiplier = 0.7f
            },
            (WeatherType.Rain, true) => new PerceptionModifier
            {
                visionDistanceMultiplier = 0.6f,
                weatherLightSensitivityMultiplier = 0.8f,
                hearingSensitivityMultiplier = 0.4f,
                soundPropagationMultiplier = 0.5f
            },
            (WeatherType.Fog, false) => new PerceptionModifier
            {
                visionDistanceMultiplier = 0.5f,
                weatherLightSensitivityMultiplier = 0.7f,
                hearingSensitivityMultiplier = 0.9f,
                soundPropagationMultiplier = 0.9f
            },
            (WeatherType.Fog, true) => new PerceptionModifier
            {
                visionDistanceMultiplier = 0.3f,
                weatherLightSensitivityMultiplier = 0.5f,
                hearingSensitivityMultiplier = 0.95f,
                soundPropagationMultiplier = 0.95f
            },
            (WeatherType.Storm, false) => new PerceptionModifier
            {
                visionDistanceMultiplier = 0.7f,
                weatherLightSensitivityMultiplier = 0.7f,
                hearingSensitivityMultiplier = 0.3f,
                soundPropagationMultiplier = 0.4f
            },
            (WeatherType.Storm, true) => new PerceptionModifier
            {
                visionDistanceMultiplier = 0.4f,
                weatherLightSensitivityMultiplier = 0.5f,
                hearingSensitivityMultiplier = 0.15f,
                soundPropagationMultiplier = 0.2f
            },
            (WeatherType.Snow, false) => new PerceptionModifier
            {
                visionDistanceMultiplier = 0.75f,
                weatherLightSensitivityMultiplier = 0.85f,
                hearingSensitivityMultiplier = 0.7f,
                soundPropagationMultiplier = 0.8f
            },
            (WeatherType.Snow, true) => new PerceptionModifier
            {
                visionDistanceMultiplier = 0.5f,
                weatherLightSensitivityMultiplier = 0.7f,
                hearingSensitivityMultiplier = 0.5f,
                soundPropagationMultiplier = 0.6f
            },
            _ => new PerceptionModifier
            {
                visionDistanceMultiplier = 1.0f,
                weatherLightSensitivityMultiplier = 1.0f,
                hearingSensitivityMultiplier = 1.0f,
                soundPropagationMultiplier = 1.0f
            }
        };
    }
}
```

**定义位置**：`Assets/Game/Core/Environment/Weather/Types/PerceptionModifier.cs`

### WeatherStateChangedEvent

```csharp
/// <summary>
/// 天气状态变化事件
/// 由 WeatherSystem 发布，所有相关系统订阅
/// 在天气过渡期间，以 1Hz 频率广播带进度字段的事件
/// </summary>
public struct WeatherStateChangedEvent
{
    /// <summary>天气类型</summary>
    public WeatherType weatherType;

    /// <summary>天气强度（0.0 = 轻微，1.0 = 极端）</summary>
    public float intensity;

    /// <summary>过渡进度（0.0 = 开始，1.0 = 完成）</summary>
    public float transitionProgress;

    /// <summary>是否正在过渡中（true 时 transitionProgress 表示当前进度）</summary>
    public bool isTransitioning;

    /// <summary>感知影响系数</summary>
    public PerceptionModifier perceptionModifier;

    /// <summary>视觉效果参数（用于 Screen Effects 系统）</summary>
    public WeatherVisualParams visualParams;

    /// <summary>事件时间戳</summary>
    public float timestamp;
}

/// <summary>
/// 天气视觉效果参数
/// Weather System 发布原始天气视觉效果参数，Screen Effects System（ADR-0023）负责将其转换为最终的后处理效果
/// </summary>
/// <remarks>
/// <para><b>与 ScreenEffectParams 的边界：</b></para>
/// <list type="bullet">
///   <item>WeatherVisualParams：Weather System 发布的原始参数（rainIntensity、fogIntensity 等）</item>
///   <item>ScreenEffectParams（ADR-0023）：Screen Effects System 的最终效果参数（noiseIntensity、blurRadius 等）</item>
///   <item>转换逻辑由 ScreenEffectsManager 执行：WeatherVisualParams → ScreenEffectParams</item>
/// </list>
/// 定义位置：<see cref="WeatherEvents"/>`Assets/Game/Core/Environment/Weather/Events/WeatherEvents.cs`
/// </remarks>
public struct WeatherVisualParams
{
    /// <summary>雨滴效果强度（0.0~1.0）</summary>
    public float rainIntensity;

    /// <summary>雾气效果强度（0.0~1.0）</summary>
    public float fogIntensity;

    /// <summary>闪电效果频率（0.0 = 无闪电，1.0 = 最高频率）</summary>
    public float lightningFrequency;

    /// <summary>雪花效果强度（0.0~1.0）</summary>
    public float snowIntensity;

    /// <summary>风效果强度（0.0~1.0，影响粒子系统方向）</summary>
    public float windIntensity;
}
```

**定义位置**：`Assets/Game/Core/Environment/Weather/Events/WeatherEvents.cs`

**发布规则**：
- **状态变化时**：天气类型切换时立即广播一次 `isTransitioning=true`
- **过渡期间**：以 **1Hz 频率** 广播带更新 `transitionProgress` 的事件
- **过渡完成时**：广播 `isTransitioning=false`，`transitionProgress=1.0`

**发布者**：WeatherSystemManager
**订阅者**：NPC AI System、LOS System、Sanity/Rage System、Screen Effects System、Audio System、Environment Interaction System（用于雨灭火等环境响应）

> **Weather × Environment 双向通信**：Weather System 订阅 Environment Interaction System 发布的环境事件（如 `ExplosionEvent`、`FireExtinguishedEvent`），以响应环境变化（如火灾触发雾气、大雨浇灭火焰）。具体事件类型和订阅关系见 ADR-0013 §3.5 EnvironmentalEvent 定义。
>
> **注意**：ADR-0013 §4 不存在（原引用错误），Environment→Weather 的具体订阅事件类型应在 ADR-0013 中明确定义。参见 shared-types.md §16 的 `EnvironmentalEventType` 枚举定义。

### WeatherForceChangeEvent

```csharp
/// <summary>
/// 强制天气切换来源枚举
/// </summary>
public enum WeatherForceSource
{
    Area,       // 区域触发（如玩家进入洞穴）
    GameEvent   // 游戏事件触发（如剧情 Boss 战）
}

/// <summary>
/// 强制天气切换事件
/// 由区域触发器或游戏事件发布，高优先级打断当前天气过渡
/// WeatherSystemManager 订阅此事件，收到后立即中断当前过渡
/// </summary>
public struct WeatherForceChangeEvent
{
    /// <summary>目标天气类型</summary>
    public WeatherType targetWeather;

    /// <summary>过渡时长（秒）</summary>
    public float transitionDuration;

    /// <summary>触发来源类型</summary>
    public WeatherForceSource source;

    /// <summary>
    /// 区域 ID（当 source = Area 时有效）
    /// </summary>
    public string areaId;

    /// <summary>
    /// 游戏事件名称（当 source = GameEvent 时有效）
    /// </summary>
    public string eventName;

    /// <summary>事件时间戳</summary>
    public float timestamp;
}
```

**使用场景**：
- 玩家进入特定区域（如洞穴强制 Clear）
- 剧情事件触发（如 Boss 战强制 Storm）
- 特殊游戏模式（如潜行任务中禁用 Fog）

**实现说明**：
- `AreaWeatherTrigger` 组件挂载在区域触发器上，检测到玩家进入时发布 `WeatherForceChangeEvent`
- `WeatherSystemManager` 订阅 `WeatherForceChangeEvent`，收到后**立即中断当前过渡动画**，无论过渡进行到哪个阶段
- 打断行为：
  1. 当前正在进行的过渡被取消，`transitionProgress` 重置
  2. 旧天气的视觉效果直接撤销（发布 `ScreenEffectRevokeEvent`）
  3. 新的强制天气立即开始过渡（如果有过渡时长）或立即生效（如果 `transitionDuration=0`）
  4. 下游系统收到新的 `WeatherStateChangedEvent`（`isTransitioning=true`）
- 强制切换完成后，WeatherSystemManager 恢复自然天气循环，下次 `GenerateNextWeather` 自然生成新天气，不再有强制状态残留

**WeatherForceChangeEvent 时序规则**：

| 顺序 | 步骤 | 说明 |
|------|------|------|
| 1 | 区域触发器检测 | `AreaWeatherTrigger` 检测到玩家进入特定区域 |
| 2 | 发布 WeatherForceChangeEvent | 携带 `targetWeather`、`areaId`、`source=Area` |
| 3 | WeatherSystemManager 接收 | 立即中断当前 `WeatherTransition` 动画 |
| 4 | 撤销旧天气效果 | 发布 `ScreenEffectRevokeEvent` |
| 5 | 应用新天气 | 开始新的强制天气过渡或立即生效 |
| 6 | 发布 WeatherStateChangedEvent | 下游系统收到 `isTransitioning=true` 的事件 |

> **与 WorldMap 状态机的协调**：`WeatherForceChangeEvent` 打断天气过渡，但**不中断 WorldMap 状态机**（保持 `EXPLORING` 等状态）。强制天气切换时，区域危险等级不变。详见 ADR-0012。

**PitchBlack 区域的 Fog 天气视觉效果**：

当区域 `AreaLightingState = PitchBlack` 且天气为 `Fog` 时：

| 效果 | 说明 |
|------|------|
| 雾效叠加 | Fog 天气效果在 PitchBlack 基础上额外叠加 |
| 最终视觉 | 玩家视野极度受限，但仍保留微弱轮廓识别能力 |
| NPC 感知 | `fogIntensity * PitchBlack.visibilityMultiplier` 进一步降低感知 |
| 特殊效果 | 可选：开启"盲区"模式（仅声音提示敌人方向） |

> **设计理由**：PitchBlack 区域已是视野最低状态，Fog 不应完全覆盖而应叠加效果，保持"伸手不见五指但有微弱光感"的氛围。

闪电事件，由 WeatherSystem 发布，Lighting System 订阅以叠加闪电光照效果。

**定义位置**：`Assets/Game/Core/Environment/Weather/Events/WeatherEvents.cs`

```csharp
/// <summary>
/// 闪电事件
/// 由 WeatherSystem 发布（仅 Storm 天气时），Lighting System 订阅以叠加闪电光照效果
/// </summary>
public struct LightningFlashEvent
{
    /// <summary>
    /// 闪电中心位置（用于光照计算）
    /// </summary>
    public Vector3 position;

    /// <summary>
    /// 闪电强度（影响 globalIllumination 提升幅度，0.0~1.0）
    /// </summary>
    public float intensityBonus;

    /// <summary>
    /// 闪电持续时间（秒）
    /// </summary>
    public float duration;

    /// <summary>
    /// 事件时间戳
    /// </summary>
    public float timestamp;
}
```

**发布条件**：
- 仅在 `WeatherType = Storm` 时，WeatherSystemManager 随机生成闪电事件
- 发布频率由 `lightningFrequency` 参数控制（Storm 高强度时最高可达 0.5 次/秒）

**订阅者**：Lighting System

**使用场景**：
- 闪电击中时触发全局光照脉冲效果（叠加在时段基础光照之上）
- 配合雷声 AudioEvent（由 Audio System 订阅）

---

## 天气类型详细参数

### 默认感知系数

| WeatherType | Intensity | VisionDistance | VisionAngle | HearingSensitivity | SoundPropagation |
|-------------|-----------|----------------|-------------|-------------------|-------------------|
| Clear       | —         | 1.0            | 1.0         | 1.0               | 1.0               |
| Rain        | 0.5       | 0.8            | 0.9         | 0.6               | 0.7               |
| Rain        | 1.0       | 0.6            | 0.8         | 0.4               | 0.5               |
| Fog         | 0.5       | 0.5            | 0.7         | 0.9               | 0.9               |
| Fog         | 1.0       | 0.3            | 0.5         | 0.95              | 0.95              |
| Storm       | 0.5       | 0.7            | 0.7         | 0.3               | 0.4               |
| Storm       | 1.0       | 0.4            | 0.5         | 0.15              | 0.2               |
| Snow        | 0.5       | 0.75           | 0.85        | 0.7               | 0.8               |
| Snow        | 1.0       | 0.5            | 0.7         | 0.5               | 0.6               |

**Intensity 字段说明**：
- `—` 表示 Clear 天气无强度概念，始终为基准值
- 0.5 / 1.0 表示该天气类型的两个预设强度档位
- 设计师可通过插值公式计算任意强度的感知系数

**WindIntensity 参考值**：
| WeatherType | Intensity | WindIntensity | 说明 |
|-------------|-----------|---------------|------|
| Clear       | -         | 0.0           | 无风 |
| Rain        | 0.5       | 0.2           | 轻微风声 |
| Rain        | 1.0       | 0.4           | 较强风声 |
| Fog         | -         | 0.1           | 几乎无风 |
| Storm       | 0.5       | 0.6           | 强风 |
| Storm       | 1.0       | 0.9           | 暴风 |
| Snow        | 0.5       | 0.3           | 寒风 |
| Snow        | 1.0       | 0.5           | 风雪 |

**插值规则**：中间强度使用 `PerceptionModifier.Lerp()` 方法进行 SmoothStep 曲线插值。

例如 Rain 强度 0.75：
```csharp
var from = PerceptionModifier.GetFor(WeatherType.Rain, 0.5f);
var to = PerceptionModifier.GetFor(WeatherType.Rain, 1.0f);
float t = (0.75f - 0.5f) / 0.5f; // 0.5
var result = PerceptionModifier.Lerp(from, to, t);
// result.visionDistanceMultiplier == 0.7
```

**代码示例**（WeatherSystemManager 中的实现）：
```csharp
// 在天气过渡时计算当前感知系数
private PerceptionModifier CalculateCurrentPerceptionModifier(
    WeatherType fromWeather, WeatherType toWeather,
    float fromIntensity, float toIntensity,
    float transitionProgress)
{
    var from = PerceptionModifier.GetFor(fromWeather, fromIntensity);
    var to = PerceptionModifier.GetFor(toWeather, toIntensity);
    return PerceptionModifier.Lerp(from, to, transitionProgress);
}
```

**Intensity 与 PerceptionModifier 的关系**：
- `intensity` 是天气的**外在强度表现**（影响视觉、听觉等主观感受）
- `PerceptionModifier` 是天气对 NPC 感知的**客观影响系数**
- 两者**独立变化**：例如同一 Rain 强度 0.5，在不同区域可能对 NPC 有不同的感知折扣（取决于地形遮挡）
- 设计师通过调整 PerceptionModifier 表来平衡游戏难度，而非直接修改 intensity

> **感知系数叠加规则**：Weather × Lighting 的感知系数叠加规则见 [shared-types.md §22.4](./shared-types.md#224-感知系数叠加规则weather-×-lighting)。

---

## 天气过渡机制

### 过渡参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `TransitionDuration` | 30.0s | 天气切换的渐变时长 |
| `MinTransitionInterval` | 60.0s | 两次天气切换的最小间隔 |
| `UpdateFrequency` | 1.0Hz | 感知系数的更新频率（非每帧） |

### 过渡曲线

使用 **SmoothStep** 插值曲线，确保过渡平滑：
```
t_smooth = t * t * (3.0 - 2.0 * t)
currentValue = Lerp(fromValue, toValue, t_smooth)
```

---

## Alternatives Considered

### Alternative 1: 每帧轮询天气状态（主动推送）

- **描述**：天气系统每帧检查天气状态，主动调用下游系统的接口
- **优点**：实现简单，状态同步即时
- **缺点**：
  - 违反 World Layer 独立性原则
  - 引入对下游系统的直接依赖
  - 每帧调用增加 CPU 开销
- **拒绝理由**：违反 ADR-0003 World Layer 定义，需要解耦

### Alternative 2: 天气系统直接管理视觉特效

- **描述**：天气系统直接持有后处理效果的引用，主动设置参数
- **优点**：减少事件广播层次
- **缺点**：
  - 引入对渲染系统的直接依赖
  - 违反 World Layer 不调用其他游戏系统的原则
- **拒绝理由**：Weather System 应该广播环境变化，而非管理特效实现

### Alternative 3: 天气仅影响视觉效果（无感知系数）

- **描述**：天气系统仅控制视觉和听觉效果，不影响 NPC AI 感知
- **优点**：实现简单，性能开销小
- **缺点**：天气变化缺乏战术意义，削弱"环境即战术工具"的设计理念
- **拒绝理由**：天气应该同时影响视觉效果和游戏玩法，提供战术深度

---

## Consequences

### Positive

- **解耦架构**：Weather System 不依赖任何游戏系统，仅通过 Event Bus 广播
- **可扩展性**：新增天气类型只需扩展枚举和参数配置，无需修改下游系统
- **性能优化**：感知系数更新频率 <= 1Hz，避免每帧计算
- **战术深度**：天气变化影响 NPC 感知，提供潜行策略选择

### Negative

- **调参复杂性**：每种天气类型需要配置多个感知系数，平衡难度高
- **调试难度**：异步事件广播，状态变化链路较长
- **美术资源**：每种天气需要独立的视觉和听觉资源

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 感知系数失衡 | 极端天气导致游戏过难或过简 | 提供难度调节选项，允许禁用天气感知影响 |
| 过渡突兀 | 天气渐变不自然 | 使用 SmoothStep 曲线，增加过渡时长可配置 |
| 资源加载 | 天气切换时加载资源可能卡顿 | 使用 Addressables 异步加载，预加载下一个天气资源 |

---

## Performance Implications

| 指标 | 影响 | 说明 |
|------|------|------|
| **CPU** | 低 | 每秒更新一次感知系数，计算量小 |
| **Memory** | 低 | 仅存储当前天气状态和过渡进度 |
| **Load Time** | 中 | 天气切换时异步加载资源，使用 Addressables |
| **Network** | 无 | 本地事件广播 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 `WeatherSystemManager` MonoBehaviour
- [ ] 定义 `WeatherType` 枚举和 `PerceptionModifier` 结构
- [ ] 实现 `WeatherStateChangedEvent` 事件广播

### Phase 2: 天气类型
- [ ] 实现 Clear / Rain / Fog / Storm / Snow 五种天气
- [ ] 配置每种天气的感知系数
- [ ] 实现 SmoothStep 过渡曲线

### Phase 3: 下游集成
- [ ] NPC AI System 订阅天气事件，实现视野/听觉折扣
- [ ] LOS System 订阅天气事件，实现声音/视觉折扣
- [ ] Sanity/Rage System 订阅天气事件，实现氛围反馈

### Phase 4: 效果增强
- [ ] Dynamic Post-Processing 集成雨滴/雾气/闪电效果
- [ ] Audio System 集成环境音过渡

## Screen Effects System 接口

Weather System 通过 EventBus 发布 `ScreenEffectRequestEvent`，ScreenEffectsManager 订阅并处理请求。

**设计说明**：Weather System 直接发布完整的 `ScreenEffectRequestEvent`（含 `ScreenEffectParams`），而非仅发布 `WeatherVisualParams`。这是因为 ADR-0023 定义的 `ScreenEffectRequestEvent` 已经包含了完整的视觉效果参数，Weather System 负责填充这些参数，`ScreenEffectsManager` 负责叠加和渲染。

> **架构选择说明**：当前设计让 Weather System 了解 `ScreenEffectRequestEvent` 结构，形成类型耦合。
> 这是一种**架构权衡**：
> - **替代方案 A（严格分层）**：Weather 发布 `WeatherVisualRequestEvent`，ScreenEffectsManager 订阅并转换为 `ScreenEffectRequestEvent`。优点：完全解耦；缺点：需要额外的转换逻辑，ScreenEffectsManager 需要了解 Weather 的视觉效果参数。
> - **替代方案 B（当前设计）**：Weather 直接发布 `ScreenEffectRequestEvent`。优点：简单直接，Weather 完全控制视觉效果参数；缺点：类型耦合。
>
> **决策**：采用方案 B。原因：
> 1. `ScreenEffectParams` 是纯数据容器，不包含行为，耦合成本低
> 2. Weather 和 ScreenEffectsManager 都在 Infrastructure Layer 边界处，紧耦合可接受
> 3. 如果未来需要完全解耦，可在 ScreenEffectsManager 层添加 `WeatherVisualRequestEvent → ScreenEffectRequestEvent` 转换器

```csharp
// WeatherSystemManager 发布 Screen Effect 请求
public class WeatherSystemManager : MonoBehaviour
{
    // 注意：WeatherSystemManager 遵循 World Layer 原则，不直接引用 ScreenEffectsManager
    // 而是通过 EventBus 发布请求，由 ScreenEffectsManager 订阅处理

    // 在天气状态更新完成后调用此方法，将视觉效果请求推送到 EventBus
    // 命名使用 Publish 前缀，明确表示这是主动发布而非响应回调
    private void PublishWeatherScreenEffects(WeatherStateChangedEvent evt)
    {
        // 构建天气对应的效果类型和参数
        var (effectType, params) = GetWeatherScreenEffectParams(evt);

        var weatherFXRequest = new ScreenEffectRequestEvent
        {
            sourceSystem = ScreenEffectSource.WeatherSystem,
            requesterId = $"weather_{evt.weatherType}",
            effectType = effectType,
            intensity = evt.intensity,
            duration = 0f, // 永久，直到下一个天气请求
            timeout = 0f,   // 永不超时，由天气系统管理撤销
            priority = 40,  // 天气效果优先级
            parameters = params
        };

        EventBus.Instance.Publish(weatherFXRequest);
    }

    /// <summary>
    /// 根据天气类型获取对应的屏幕效果类型和参数
    /// </summary>
    private (ScreenEffectType effectType, ScreenEffectParams @params) GetWeatherScreenEffectParams(
        WeatherStateChangedEvent evt)
    {
        var p = evt.visualParams;
        return evt.weatherType switch
        {
            WeatherType.Clear => (ScreenEffectType.None, default),

            WeatherType.Rain => (
                ScreenEffectType.Noise,
                new ScreenEffectParams
                {
                    noiseIntensity = p.rainIntensity,
                    noiseColor = new Color(0.5f, 0.5f, 0.5f, 1f),
                    fadeInDuration = 1.0f,
                    fadeOutDuration = 2.0f
                }
            ),

            WeatherType.Fog => (
                ScreenEffectType.Blur,
                new ScreenEffectParams
                {
                    blurRadius = p.fogIntensity * 0.3f,
                    blurQuality = 2,
                    fadeInDuration = 1.5f,
                    fadeOutDuration = 2.0f
                }
            ),

            WeatherType.Storm => (
                ScreenEffectType.Noise,  // 暴风雨使用 Noise 效果模拟雨滴/噪点
                new ScreenEffectParams
                {
                    noiseIntensity = p.rainIntensity,
                    noiseColor = new Color(0.5f, 0.5f, 0.5f, 1f),
                    fadeInDuration = 0.5f,
                    fadeOutDuration = 2.0f
                }
            ),

            WeatherType.Snow => (
                ScreenEffectType.Noise | ScreenEffectType.FilmGrain,
                new ScreenEffectParams
                {
                    noiseIntensity = p.snowIntensity * 0.3f,
                    noiseColor = Color.white,
                    grainDensity = p.snowIntensity * 0.5f,
                    fadeInDuration = 2.0f,
                    fadeOutDuration = 3.0f
                }
            ),

            _ => (ScreenEffectType.None, default)
        };
    }

    /// <summary>
    /// 撤销天气屏幕效果（当天气切换时，旧天气的效果需要撤销）
    /// 由 WeatherSystemManager 在发布新的天气效果请求前自动调用
    /// </summary>
    private void RevokeWeatherScreenEffects(WeatherType oldWeather)
    {
        EventBus.Instance.Publish(new ScreenEffectRevokeEvent
        {
            sourceSystem = ScreenEffectSource.WeatherSystem,
            requesterId = $"weather_{oldWeather}",
            timestamp = Time.time
        });
    }
}
```

**效果请求对应关系**：

| WeatherType | ScreenEffectType | 主要视觉效果 |
|------------|------------------|-------------|
| Clear | None | 无 |
| Rain | Noise | 雨滴噪点效果 |
| Fog | Blur | 朦胧模糊效果 |
| Storm | Noise + Shake | 暴风雨噪点 + 闪电震动 |
| Snow | Noise + FilmGrain | 雪花颗粒效果 |

**撤销时机**：
- 天气切换时，先发布 `ScreenEffectRevokeEvent` 撤销旧天气效果，再发布新的 `ScreenEffectRequestEvent`
- `WeatherForceChangeEvent` 触发强制切换时，同样先撤销当前天气效果

---

## Tuning Knobs

| 参数 | 默认值 | 安全范围 | 说明 |
|------|--------|---------|------|
| `TransitionDuration` | 30.0s | 15.0~60.0s | 天气切换的渐变时长 |
| `MinTransitionInterval` | 60.0s | 30.0~120.0s | 两次天气切换的最小间隔 |
| `UpdateFrequency` | 1.0Hz | 0.5~2.0Hz | 感知系数的更新频率 |
| Storm HearingSensitivity Min | 0.15 | 0.1~0.3 | 暴风雨时 NPC 听觉灵敏度下限（防止过易潜行） |
| Fog VisionDistance Max | 0.3 | 0.2~0.4 | 浓雾时视野距离乘数下限 |
| Rain VisionDistance Max | 0.6 | 0.4~0.8 | 大雨时视野距离乘数下限 |

---

## 游戏时间依赖

Weather System 的游戏时间来源定义于 `IGameTimeProvider` 接口（**统一定义见 [shared-types.md §22](./shared-types.md#22-游戏时间接口与-world-layer-时间事件)**，此处仅说明依赖关系）：

**依赖关系**：
- Weather System **不直接依赖 WorldMap System**，而是通过接口 `IGameTimeProvider` 解耦
- WorldMap System 实现 `IGameTimeProvider` 并发布 `GameHourChangedEvent`
- WeatherSystemManager 订阅 `GameHourChangedEvent` 获取时间，而非主动轮询

**环境事件订阅 (2026-04-15 修复)**：
- WeatherSystemManager 订阅 `EnvironmentalEvent`（来自 EnvironmentInteractionSystem）以响应环境变化
- 订阅的事件类型：`EnvironmentalEventType.EXPLOSION`、`EnvironmentalEventType.FIRE`、`EnvironmentalEventType.DESTRUCTION`
- 具体订阅逻辑见 ADR-0013 §3.5

---

## Validation Criteria

| ID | 验收条件 |
|----|---------|
| VC-1 | 天气切换时，所有下游系统正确接收 `WeatherStateChangedEvent` |
| VC-2 | 天气强度变化时，NPC AI 的感知范围平滑过渡 |
| VC-3 | 暴风雨天气下，NPC 听觉灵敏度显著下降，玩家可趁机潜行 |
| VC-4 | 天气切换动画流畅，无视觉跳跃 |
| VC-5 | 性能测试：每秒更新感知系数时，CPU 占用 < 0.1ms |

---

## Related Decisions

- [ADR-0003: 系统分层架构](./adr-0003-system-layers.md) — World Layer 定义
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC AI 感知系统
- [ADR-0007: LOS 系统架构](./adr-0007-los-system-architecture.md) — LOS 感知折扣
- [ADR-0017: Sanity/Rage 系统](./adr-0017-sanity-rage-meter-architecture.md) — 氛围反馈
- [ADR-0023: Screen Effects 系统](./adr-0023-screen-effects-system-architecture.md) — 天气视觉效果请求
- [shared-types.md §22](./shared-types.md#22-游戏时间接口与-world-layer-时间事件) — IGameTimeProvider、GameHourChangedEvent、ScreenEffectRevokeEvent
- [shared-types.md §22.4](./shared-types.md#224-感知系数叠加规则weather-×-lighting) — **感知系数叠加规则权威定义**（Weather × Lighting 组合公式）
