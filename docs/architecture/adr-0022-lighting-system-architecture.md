# ADR-0022: 光照系统 (Lighting System) 架构决策

## Status
**Proposed**

## Date
2026-04-11

## Context

### Problem Statement

光照系统是《断绝：罪恶之源》World Layer 的核心组成，通过动态光照强化潜行氛围并对多个下游系统产生战术影响。系统需要解决：

1. **多系统联动**：光照变化需要影响 NPC AI 感知（阴影中的潜行）、LOS 系统（明暗区域判定）、Sanity/Rage（幽暗恐惧）、Dynamic Post-Processing（光晕/暗角）
2. **时段系统**：支持昼夜循环，多个光照配置文件切换
3. **区域光照**：不同区域可以有独立的光照状态（地下室更暗，街道更亮）
4. **独立运作**：World Layer 系统不调用其他游戏系统，仅通过 Event Bus 广播光照状态

### Constraints

- **架构约束**：遵循 ADR-0003 World Layer 定义 — 代码放在 Core/Environment 子目录，通过 Event Bus 与其他层通信
- **性能约束**：光照更新频率 <= 1Hz（昼夜切换），区域光照可每帧更新
- **兼容性约束**：支持 PC & PS5 平台，需要适配光照探针（Light Probes）
- **美术约束**：提供至少 4 个时段配置（Dawn / Day / Dusk / Night）

### Requirements

- **必须**：定义 4+ 个时段类型（Dawn / Day / Dusk / Night）及光照参数
- **必须**：定义 LightingStateChangedEvent 事件结构
- **必须**：定义光照对 NPC AI 感知的影响参数（暗处潜行加成）
- **必须**：定义光照对 LOS 系统的影响参数（明暗区域声音/视觉判定）
- **必须**：支持区域光照状态（AreaLightingState）
- **必须**：通过 Event Bus 广播光照状态，不直接调用下游系统

---

## Decision

### 架构决策

采用**时段配置 + 区域光照 + 参数化影响系数**架构：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Lighting System 架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【World Layer — 不参与分层调用链，独立运作】                                   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                 LightingSystemManager (MonoBehaviour)                      │   │
│  │  - 持有当前时段状态 (TimeOfDay)                                          │   │
│  │  - 管理区域光照状态表 (Dictionary<areaId, AreaLightingState>)             │   │
│  │  - 发布 LightingStateChangedEvent（时段变化）                            │   │
│  │  - 发布 AreaLightingChangedEvent（区域光照变化）                          │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      LightingStateChangedEvent                         │   │
│  │  {                                                                      │   │
│  │    timeOfDay: TimeOfDay,           // 当前时段                           │   │
│  │    globalIllumination: float,     // 全局光照强度 0.0~1.0               │   │
│  │    shadowIntensity: float,        // 阴影强度 0.0~1.0                  │   │
│  │    perceptionModifier: LightingPerceptionModifier,  // 感知系数         │   │
│  │    timestamp: float               // 事件时间戳                          │   │
│  │  }                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      AreaLightingChangedEvent                          │   │
│  │  {                                                                      │   │
│  │    areaId: string,                // 区域 ID                            │   │
│  │    lightingState: AreaLightingState,  // 区域光照状态                   │   │
│  │    playerPosition: Vector3,       // 玩家当前位置（用于判定）           │   │
│  │  }                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│          ┌─────────────────────────┼─────────────────────────┐              │
│          ▼                         ▼                         ▼              │
│  ┌───────────────┐        ┌───────────────┐        ┌───────────────┐         │
│  │   NPC AI      │        │   LOS System  │        │ Sanity/Rage   │         │
│  │   System      │        │               │        │               │         │
│  │               │        │               │        │               │         │
│  │ ◆ 阴影潜行加成│        │ ◆ 明暗判定   │        │ ◆ 幽暗恐惧   │         │
│  │ ◆ 光照感知   │        │ ◆ 遮挡折扣   │        │ ◆ 氛围反馈   │         │
│  └───────────────┘        └───────────────┘        └───────────────┘         │
│                                                                              │
│  ┌───────────────┐        ┌───────────────┐                                 │
│  │ Dynamic Post  │        │   Audio       │                                 │
│  │ Processing    │        │   System      │                                 │
│  │               │        │               │                                 │
│  │ ◆ 光晕强度   │        │ ◆ 夜晚环境音│                                 │
│  │ ◆ 暗角变化   │        │               │                                 │
│  └───────────────┘        └───────────────┘                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### TimeOfDay 枚举

```csharp
/// <summary>
/// 时段类型枚举
/// </summary>
public enum TimeOfDay
{
    Dawn,    // 黎明 — 光照渐亮，暖色调
    Day,     // 白昼 — 全光照，中性色调
    Dusk,    // 黄昏 — 光照渐暗，橙红色调
    Night    // 夜晚 — 低光照，蓝冷色调
}
```

**定义位置**：`Assets/Game/Core/Environment/Lighting/Types/TimeOfDay.cs`

### AreaLightingState 枚举

```csharp
/// <summary>
/// 区域光照状态
/// </summary>
public enum AreaLightingState
{
    Bright,      // 明亮 — 标准光照，无潜行加成
    Normal,      // 正常 — 标准光照
    Dim,         // 昏暗 — 潜行加成，轻微氛围效果
    PitchBlack   // 漆黑 — 最大潜行加成，Sanity 负面影响
}
```

**定义位置**：`Assets/Game/Core/Environment/Lighting/Types/AreaLightingState.cs`

### LightingPerceptionModifier 结构

```csharp
/// <summary>
/// 光照对感知的影响系数
/// 由 LightingSystem 计算，广播给下游系统
/// </summary>
public struct LightingPerceptionModifier
{
    /// <summary>NPC 在阴影中的潜行检测难度乘数（越高越难检测到玩家）</summary>
    public float shadowStealthBonus;

    /// <summary>NPC 光照感知灵敏度乘数（越低越难看到玩家）</summary>
    public float npcLightSensitivity;

    /// <summary>LOS 遮挡判定折扣（影响阴影对视野的遮挡效果）</summary>
    public float losShadowOcclusionDiscount;

    /// <summary>玩家脚步声在暗处的可听闻距离乘数</summary>
    public float playerNoiseAudibleDistanceInDark;
}
```

**定义位置**：`Assets/Game/Core/Environment/Lighting/Types/LightingPerceptionModifier.cs`

### LightingStateChangedEvent

```csharp
/// <summary>
/// 时段光照状态变化事件
/// 由 LightingSystem 发布，所有相关系统订阅
/// </summary>
public struct LightingStateChangedEvent
{
    /// <summary>时段类型</summary>
    public TimeOfDay timeOfDay;

    /// <summary>全局光照强度（0.0 = 漆黑，1.0 = 满光照）</summary>
    public float globalIllumination;

    /// <summary>阴影强度（0.0 = 无阴影，1.0 = 浓重阴影）</summary>
    public float shadowIntensity;

    /// <summary>色调温度（以 Kelvin 为单位，参考：阳光 5500K，烛光 1800K，月光 4100K）</summary>
    public float colorTemperature;

    /// <summary>色调偏移（0.0~1.0，对应 0°~360°）</summary>
    public float hueShift;

    /// <summary>感知影响系数</summary>
    public LightingPerceptionModifier perceptionModifier;

    /// <summary>视觉效果参数（用于 Screen Effects 系统）</summary>
    public LightingVisualParams visualParams;

    /// <summary>事件时间戳</summary>
    public float timestamp;
}

/// <summary>
/// 光照视觉效果参数
/// 用于 Screen Effects 系统请求后处理效果
/// </summary>
public struct LightingVisualParams
{
    /// <summary>光晕效果强度（0.0~1.0）</summary>
    public float bloomIntensity;

    /// <summary>暗角强度（0.0~1.0）</summary>
    public float vignetteIntensity;

    /// <summary>色调映射 LUT 强度（用于冷暖色调转换）</summary>
    public float lutIntensity;
}
```

**定义位置**：`Assets/Game/Core/Environment/Lighting/Events/LightingEvents.cs`

**发布者**：LightingSystemManager
**订阅者**：NPC AI System、LOS System、Sanity/Rage System、Screen Effects System、Audio System

### AreaLightingChangedEvent

```csharp
/// <summary>
/// 区域光照状态变化事件
/// 由 AreaLightingDetector（区域光照检测器）发布
/// LightingSystemManager 维护区域光照状态表，但区域触发检测由独立的检测器负责
/// </summary>
public struct AreaLightingChangedEvent
{
    /// <summary>区域 ID</summary>
    public string areaId;

    /// <summary>区域光照状态</summary>
    public AreaLightingState lightingState;

    /// <summary>玩家当前处于该区域</summary>
    public bool isPlayerInside;

    /// <summary>事件时间戳</summary>
    public float timestamp;
}
```

**定义位置**：`Assets/Game/Core/Environment/Lighting/Events/LightingEvents.cs`

**发布者**：AreaLightingDetector（独立检测器组件，挂载在区域触发器上）
**订阅者**：NPC AI System、LOS System（判定玩家所处光照条件）、Sanity/Rage System、Screen Effects System

**职责说明**：
- `LightingSystemManager` 负责维护全局光照状态（时段）和区域光照配置表（每个区域的基础光照状态）
- `AreaLightingDetector` 负责检测玩家是否进入/离开某区域，并在玩家进入时发布 `AreaLightingChangedEvent`
- 下游系统直接使用事件中携带的 `lightingState` 参数进行判定
  - 如需获取更详细的光照系数（如 `ShadowStealthBonus`），可通过 `areaId` 查询 `LightingSystemManager` 的区域配置表
  - 事件中的 `isPlayerInside` 标志用于判定玩家是否真正在该区域内

---

## 时段详细参数

### 默认光照系数

| TimeOfDay | GlobalIllumination | ShadowIntensity | ShadowStealthBonus | NPCLightSensitivity |
|-----------|-------------------|-----------------|-------------------|-------------------|
| Dawn      | 0.4               | 0.3             | 1.2               | 0.8               |
| Day       | 1.0               | 0.5             | 1.0               | 1.0               |
| Dusk      | 0.3               | 0.6             | 1.3               | 0.7               |
| Night     | 0.1               | 0.9             | 1.5               | 0.5               |

**说明**：
- `ShadowStealthBonus`：玩家在阴影中的潜行检测难度乘数，Night 时 1.5 表示 NPC 检测玩家难度增加 50%
- `NPCLightSensitivity`：NPC 对光照变化的敏感度，Night 时 0.5 表示 NPC 更难发现玩家

### 区域光照叠加

区域光照状态在时段基础上叠加计算。定义统一的设计原则：**光照越暗，潜行加成越高**。

| AreaLightingState | IlluminationOffset | ShadowStealthMultiplier | 典型场景 |
|-------------------|-------------------|------------------------|---------|
| Bright            | +0.3              | 0.8                    | 室内灯、路灯 |
| Normal            | 0.0               | 1.0                    | 普通室内 |
| Dim               | -0.2              | 1.3                    | 走廊、地下室 |
| PitchBlack        | -0.5              | 2.0                    | 封闭房间、洞穴 |

**计算公式**：
```
finalIllumination = Clamp(timeOfDayIllumination + illuminationOffset, 0.0, 1.0)
finalStealthBonus = timeOfDayShadowStealthBonus * shadowStealthMultiplier
```

**感知系数上限**：叠加后的 `finalStealthBonus` 理论最大值可达 `Night(1.5) * PitchBlack(2.0) = 3.0`。当超过安全上限（默认 3.0）时，由下游系统自行 clamp。例如 NPC AI 感知系统应确保 `stealthBonus` 不超过 `[1.0, 3.0]` 区间。

**设计一致性说明**：
- `PitchBlack` 的 `illuminationOffset = -0.5` 与 `shadowStealthMultiplier = 2.0` 保持一致
- 当 timeOfDayIllumination = 0.1（Night）且区域为 PitchBlack 时：
  - `finalIllumination = Clamp(0.1 - 0.5, 0, 1) = 0.0`（完全漆黑）
  - `finalStealthBonus = 1.5 * 2.0 = 3.0`（最大潜行加成）
- 这确保了"越暗 = 越难被看到"的设计原则在数值上保持一致

---

## 昼夜循环机制

### 时段切换参数

| 参数 | 默认值 | 安全范围 | 说明 |
|------|--------|---------|------|
| `TimeScale` | 1.0x | 0.0~10.0x | 游戏内时间流速（可调节） |
| `DawnStartHour` | 6 | 4~8 | 黎明开始时间 |
| `DayStartHour` | 8 | 6~10 | 白昼开始时间 |
| `DuskStartHour` | 18 | 16~20 | 黄昏开始时间 |
| `NightStartHour` | 20 | 18~22 | 夜晚开始时间 |
| `TransitionDuration` | 30.0s | 15.0~60.0s | 时段切换的渐变时长 |
| `NightShadowStealthBonus` | 1.5 | 1.2~2.0 | 夜晚阴影潜行加成乘数 |
| `PitchBlackStealthBonus` | 2.0 | 1.5~3.0 | 漆黑区域潜行加成乘数 |

### 游戏时间依赖

Lighting System 的游戏时间来源定义于 `IGameTimeProvider` 接口（与 Weather System 共用，**统一定义见 [shared-types.md §22](./shared-types.md#22-游戏时间接口与-world-layer-时间事件)**，此处仅说明依赖关系）：

**依赖关系**：
- Lighting System **不直接依赖 WorldMap System**，而是通过接口 `IGameTimeProvider` 解耦
- WorldMap System 实现 `IGameTimeProvider` 并发布 `GameHourChangedEvent`
- LightingSystemManager 订阅 `GameHourChangedEvent` 获取时间，而非主动轮询

### 时段检测逻辑

```csharp
void UpdateTimeOfDay()
{
    float hour = GetCurrentGameHour(); // 0.0 ~ 24.0

    TimeOfDay newTimeOfDay;

    // 时段判定边界（采用左闭右开区间 [start, end)）
    // Night: [20, 24) ∪ [0, 6) — 覆盖 20-24 和 0-6
    if (hour >= NightStartHour || hour < DawnStartHour)
        newTimeOfDay = TimeOfDay.Night;
    // Dawn: [6, 8)
    else if (hour >= DawnStartHour && hour < DayStartHour)
        newTimeOfDay = TimeOfDay.Dawn;
    // Day: [8, 18)
    else if (hour >= DayStartHour && hour < DuskStartHour)
        newTimeOfDay = TimeOfDay.Day;
    // Dusk: [18, 20)
    else // hour >= DuskStartHour && hour < NightStartHour
        newTimeOfDay = TimeOfDay.Dusk;

    if (newTimeOfDay != _currentTimeOfDay)
    {
        _currentTimeOfDay = newTimeOfDay;
        PublishLightingStateChanged();
    }
}
```

**边界过渡说明**：
- 时段切换触发后，`LightingSystemManager` 发布 `LightingStateChangedEvent`
- 下游系统收到事件后，执行平滑的视觉过渡（使用 `TransitionDuration` 配置的渐变时长）
- 时段判定使用离散边界（如 6:00 精确切换到 Dawn），过渡动画掩盖了这个跃变
- 如果需要更平滑的感知系数变化，可以将 `UpdateFrequency` 提高到 2Hz，并在感知系数层面进行插值

**边界覆盖验证**：
| 时刻 | 小时值 | 判定结果 |
|------|--------|---------|
| 凌晨 3:00 | 3.0 | Night ✓ |
| 黎明 6:00 | 6.0 | Dawn ✓ |
| 上午 9:00 | 9.0 | Day ✓ |
| 傍晚 18:00 | 18.0 | Dusk ✓ |
| 夜晚 21:00 | 21.0 | Night ✓ |
| 深夜 23:30 | 23.5 | Night ✓ |

---

## Alternatives Considered

### Alternative 1: 直接管理光源对象

- **描述**：Lighting System 直接持有所有 Light 组件的引用，主动设置参数
- **优点**：控制精确，性能开销小
- **缺点**：
  - 引入对渲染系统的直接依赖
  - 违反 World Layer 不调用其他游戏系统的原则
  - 光源数量多时管理复杂
- **拒绝理由**：违反 ADR-0003 World Layer 定义，应该通过事件广播光照状态

### Alternative 2: 静态光照配置（无动态变化）

- **描述**：光照在场景加载时固定，不支持昼夜循环
- **优点**：实现简单，性能最优
- **缺点**：缺乏动态氛围变化，削弱沉浸感
- **拒绝理由**：动态光照是沉浸式潜行体验的重要组成部分

### Alternative 3: 仅影响视觉效果（无感知影响）

- **描述**：光照系统仅控制视觉亮度，不影响 NPC AI 感知
- **优点**：实现简单，美术可控性强
- **缺点**：光照缺乏战术意义
- **拒绝理由**：光照应该同时影响视觉效果和游戏玩法

---

## Consequences

### Positive

- **解耦架构**：Lighting System 不依赖任何游戏系统，仅通过 Event Bus 广播
- **可扩展性**：新增时段只需扩展枚举和参数配置
- **性能优化**：时段更新频率低（<= 1Hz），区域光照通过触发式更新
- **战术深度**：光照变化影响 NPC 感知，提供潜行策略选择

### Negative

- **调参复杂性**：多个系数组合，平衡难度高
- **美术工作量**：每个时段需要独立的 Lighting Profile
- **区域配置**：大量区域的灯光状态需要策划配置

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 光照感知失衡 | 夜晚导致游戏过难 | 提供难度调节选项，允许 NPC 夜视能力 |
| 性能开销 | 大量动态光源更新 | 使用烘焙光照 + 少量动态光源 |
| 区域漏光 | 区域切换时光照跳跃 | 使用 Light Probes 插值平滑过渡 |

---

## Performance Implications

| 指标 | 影响 | 说明 |
|------|------|------|
| **CPU** | 低 | 时段切换 <= 1Hz，区域光照事件驱动 |
| **Memory** | 低 | 仅存储当前状态和感知系数 |
| **Load Time** | 中 | 时段切换时可能加载光照探针数据 |
| **Network** | 无 | 本地事件广播 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 `LightingSystemManager` MonoBehaviour
- [ ] 定义 `TimeOfDay`、`AreaLightingState` 枚举
- [ ] 实现 `LightingStateChangedEvent` 和 `AreaLightingChangedEvent`

### Phase 2: 时段系统
- [ ] 实现 Dawn / Day / Dusk / Night 四个时段配置
- [ ] 配置每种时段的光照和感知系数
- [ ] 实现时段切换过渡

### Phase 3: 区域光照
- [ ] 设计区域光照状态表数据结构
- [ ] 实现区域光照判定逻辑（玩家位置检测）
- [ ] 发布区域光照变化事件

### 区域光照编辑器支持

为方便策划配置区域光照状态，提供以下编辑器工具：

```csharp
/// <summary>
/// 区域光照配置数据
/// </summary>
[System.Serializable]
public struct AreaLightingConfig
{
    public string areaId;
    public AreaLightingState defaultState;

    /// <summary>
    /// 各时段的光照状态覆盖映射
    /// Key: 时段类型，Value: 覆盖的光照状态
    /// 如设置为 PitchBlack，则该区域在该时段自动变为漆黑
    /// 为空或不存在该时段 key 时使用 defaultState
    /// </summary>
    [SerializeField] private List<TimeOfDayStateOverride> _timeOfDayOverrides;

    /// <summary>
    /// 可选的区域色调覆盖颜色
    /// 仅当 alpha > 0 时生效
    /// </summary>
    public Color overrideTintColor;

    /// <summary>
    /// 获取指定时段的光照覆盖状态
    /// </summary>
    public AreaLightingState? GetStateOverride(TimeOfDay timeOfDay)
    {
        if (_timeOfDayOverrides == null || _timeOfDayOverrides.Count == 0) return null;
        // 单次遍历：找到即返回，避免 FirstOrDefault + Any 的双重遍历
        foreach (var o in _timeOfDayOverrides)
            if (o.timeOfDay == timeOfDay) return o.state;
        return null;
    }
}

/// <summary>
/// 时段-光照状态覆盖条目
/// </summary>
[System.Serializable]
public struct TimeOfDayStateOverride
{
    public TimeOfDay timeOfDay;
    public AreaLightingState state;
}

/// <summary>
/// 区域光照状态系数表
/// 集中管理所有 AreaLightingState 对应的系数，避免硬编码分散
/// 与"区域光照叠加"表格（见上文"区域光照叠加"章节）保持同步
/// </summary>
public static class AreaLightingCoefficientsTable
{
    /// <summary>
    /// 获取指定光照状态的系数
    /// </summary>
    /// <param name="state">光照状态</param>
    /// <returns>对应的系数</returns>
    public static AreaLightingCoefficients Get(AreaLightingState state)
    {
        return state switch
        {
            AreaLightingState.Bright => new AreaLightingCoefficients
            {
                lightingState = AreaLightingState.Bright,
                shadowStealthMultiplier = 0.8f,
                lightSensitivityMultiplier = 1.1f,
                finalIllumination = 0.8f
            },
            AreaLightingState.Normal => new AreaLightingCoefficients
            {
                lightingState = AreaLightingState.Normal,
                shadowStealthMultiplier = 1.0f,
                lightSensitivityMultiplier = 1.0f,
                finalIllumination = 1.0f
            },
            AreaLightingState.Dim => new AreaLightingCoefficients
            {
                lightingState = AreaLightingState.Dim,
                shadowStealthMultiplier = 1.3f,
                lightSensitivityMultiplier = 0.85f,
                finalIllumination = 0.6f
            },
            AreaLightingState.PitchBlack => new AreaLightingCoefficients
            {
                lightingState = AreaLightingState.PitchBlack,
                shadowStealthMultiplier = 2.0f,
                lightSensitivityMultiplier = 0.5f,
                finalIllumination = 0.0f
            },
            _ => new AreaLightingCoefficients
            {
                lightingState = AreaLightingState.Normal,
                shadowStealthMultiplier = 1.0f,
                lightSensitivityMultiplier = 1.0f,
                finalIllumination = 1.0f
            }
        };
    }
}

/// <summary>
/// 区域光照配置表 ScriptableObject
/// 存储所有区域的默认光照配置
/// </summary>
[CreateAssetMenu(fileName = "AreaLightingTable", menuName = "Game/Lighting/Area Table")]
public class AreaLightingTable : ScriptableObject
{
    [SerializeField] private List<AreaLightingConfig> _areaConfigs;

    /// <summary>
    /// 获取区域的默认光照状态
    /// 如果 areaId 未配置，返回 <see cref="AreaLightingState.Normal"/> 作为 silent fallback，
    /// 同时记录警告（供编辑器校验使用）。
    ///
    /// <para><b>Silent Fallback 设计理由：</b></para>
    /// <list type="bullet">
    ///   <item>运行时不应因配置缺失而导致游戏功能异常</item>
    ///   <item>警告日志仍记录到开发日志，供策划修复配置</item>
    ///   <item>编辑器工具可额外使用 <c>Debug.LogWarning</c> 检测配置完整性</item>
    /// </list>
    /// </summary>
    public AreaLightingState GetDefaultState(string areaId)
    {
        foreach (var config in _areaConfigs)
        {
            if (config.areaId == areaId) return config.defaultState;
        }
        Debug.LogWarning($"[AreaLightingTable] areaId '{areaId}' 未在配置表中找到，fallback 到 Normal。请检查策划配置。");
        return AreaLightingState.Normal;
    }

    /// <summary>
    /// 获取区域在特定时段的光照状态覆盖
    /// </summary>
    /// <param name="areaId">区域 ID</param>
    /// <param name="timeOfDay">当前时段</param>
    /// <returns>如果该时段有覆盖则返回覆盖状态，否则返回 null</returns>
    public AreaLightingState? GetStateOverride(string areaId, TimeOfDay timeOfDay)
    {
        var config = _areaConfigs.FirstOrDefault(c => c.areaId == areaId);
        return config.GetStateOverride(timeOfDay);
    }

    /// <summary>
    /// 获取区域的色调覆盖（如有）
    /// </summary>
    public Color? GetTintColor(string areaId)
    {
        var config = _areaConfigs.FirstOrDefault(c => c.areaId == areaId);
        return config.overrideTintColor.a > 0 ? config.overrideTintColor : (Color?)null;
    }

    /// <summary>
    /// 获取区域的光照系数
    /// 用于下游系统查询
    /// </summary>
    /// <param name="areaId">区域 ID</param>
    /// <param name="timeOfDay">当前时段</param>
    /// <returns>光照系数</returns>
    public AreaLightingCoefficients LookupCoefficients(string areaId, TimeOfDay timeOfDay)
    {
        var config = _areaConfigs.FirstOrDefault(c => c.areaId == areaId);
        if (config.areaId == null)
        {
            Debug.LogWarning($"[AreaLightingTable] areaId '{areaId}' 未在配置表中找到，fallback 到 Normal 系数。");
        }

        // 获取基础光照状态（优先使用时段覆盖）
        var stateOverride = config.GetStateOverride(timeOfDay);
        AreaLightingState effectiveState = stateOverride ?? config.defaultState;

        // 委托给 AreaLightingCoefficientsTable 获取系数
        return AreaLightingCoefficientsTable.Get(effectiveState);
    }
}
```

**编辑器工作流**：
1. 策划在 `Assets/Game/Core/Environment/Lighting/Config/` 下创建 `AreaLightingTable` 资产
2. 在 Inspector 中添加所有区域的光照配置
3. `LightingSystemManager` 引用该资产，自动填充区域状态表
4. 场景切换时，从存档恢复玩家最后所在区域的光照状态

**区域 ID 命名规范**：
- 使用场景名称 + 区域名称：`SceneName_AreaName`（如 `Basement_StorageRoom`）
- AreaLightingDetector 的 `_areaId` 必须与 Table 中的配置一致

### Screen Effects System 接口

Lighting System 通过 EventBus 发布 `ScreenEffectRequestEvent`，ScreenEffectsManager 订阅并处理请求：

```csharp
// LightingSystemManager 发布 Screen Effect 请求
public class LightingSystemManager : MonoBehaviour
{
    // 注意：LightingSystemManager 遵循 World Layer 原则，不直接引用 ScreenEffectsManager
    // 而是通过 EventBus 发布请求，由 ScreenEffectsManager 订阅处理

    private void OnLightingStateChanged(LightingStateChangedEvent evt)
    {
        // 请求暗角效果
        var vignetteRequest = new ScreenEffectRequestEvent
        {
            sourceSystem = ScreenEffectSource.LightingSystem,
            requesterId = $"lighting_{evt.timeOfDay}",
            effectType = ScreenEffectType.Vignette,
            intensity = 1.0f - evt.globalIllumination,
            duration = 0f,
            permanent = true, // 永久生效，由系统管理撤销
            timeout = 0f,
            priority = 30, // 低于天气效果
            parameters = new ScreenEffectParams
            {
                vignetteIntensity = evt.visualParams.vignetteIntensity,
                // 色调映射：将 Kelvin 温度映射到暗角颜色
                // 参考值：1800K 烛光（暖）→ 4100K 月光（冷）→ 5500K 日光（中性）
                vignetteColor = CalculateVignetteColor(evt.colorTemperature),
                fadeInDuration = 1.0f,
                fadeOutDuration = 1.0f
            },
            timestamp = Time.time
        };

        EventBus.Instance.Publish(vignetteRequest);
    }

    /// <summary>
    /// 将色温（Kelvin）映射到暗角颜色
    /// 暖色温偏橙红，冷色温偏蓝紫
    /// </summary>
    /// <remarks>
    /// 色温范围选择依据：
    /// - 1500K：烛光/篝火（暖色光源下限）
    /// - 4100K：月光/阴天日光（冷暖过渡点）
    /// - 5500K：日光（标准参考白点）
    /// - 10000K：晴天阴影/阴天天空（冷色光源上限）
    /// 归一化区间选择 1500K~10000K 覆盖游戏内所有可能的光照色温场景
    /// </remarks>
    private Color CalculateVignetteColor(float colorTemperature)
    {
        // 色温归一化到 [0, 1]，参考范围 1500K ~ 10000K
        float t = Mathf.Clamp01((colorTemperature - 1500f) / 8500f);

        // 1500K: 暖橙红 (0.8, 0.3, 0.1)
        // 4100K: 冷蓝 (0.3, 0.3, 0.8)
        // 10000K: 极冷紫 (0.2, 0.2, 1.0)
        Color warmColor = new Color(0.8f, 0.3f, 0.1f, 1f);
        Color coolColor = new Color(0.3f, 0.3f, 0.8f, 1f);
        Color extremeColor = new Color(0.2f, 0.2f, 1f, 1f);

        if (t < 0.4f)
        {
            // 1500K ~ 4900K：暖到中性
            return Color.Lerp(warmColor, coolColor, t / 0.4f);
        }
        else
        {
            // 4900K ~ 10000K：中性到冷
            return Color.Lerp(coolColor, extremeColor, (t - 0.4f) / 0.6f);
        }
    }
}
```

**效果请求对应关系**：

| TimeOfDay | VignetteIntensity | 色调 | 视觉效果 |
|-----------|------------------|------|---------|
| Dawn | 0.1 | 暖橙 | 黎明微光 |
| Day | 0.0 | 中性 | 正常光照 |
| Dusk | 0.2 | 橙红 | 黄昏暮色 |
| Night | 0.4 | 蓝冷 | 夜间暗角 |

---

### 区域光照检测器 (AreaLightingDetector)

区域光照检测器是独立的 `MonoBehaviour` 组件，挂载在区域触发器（Trigger）上：

> **IPlayer 接口来源**：IPlayer 接口定义于 [shared-types.md §7.1](./shared-types.md#71-ihu-player-接口)，由 PlayerController 实现。

```csharp
/// <summary>
/// 区域光照检测器
/// 负责检测玩家进入/离开区域，并发布 AreaLightingChangedEvent
/// </summary>
public class AreaLightingDetector : MonoBehaviour
{
    [SerializeField] private string _areaId;
    [SerializeField] private AreaLightingState _defaultLightingState = AreaLightingState.Normal;
    [SerializeField] private AreaLightingTable _areaLightingTable; // 可选：用于查询区域默认光照状态

    private int _playerEnterCount = 0; // 进入计数，避免依赖对象引用

    private void OnTriggerEnter(Collider other)
    {
        if (TryGetPlayer(other, out var player))
        {
            _playerEnterCount++;
            if (_playerEnterCount == 1) // 首次进入
            {
                EventBus.Instance.Publish(new AreaLightingChangedEvent
                {
                    areaId = _areaId,
                    lightingState = _defaultLightingState,
                    isPlayerInside = true,
                    timestamp = Time.time
                });
            }
        }
    }

    private void OnTriggerExit(Collider other)
    {
        if (TryGetPlayer(other, out var player))
        {
            _playerEnterCount = Mathf.Max(0, _playerEnterCount - 1);
            if (_playerEnterCount == 0) // 完全离开
            {
                // 查询该区域的默认光照状态
                // GetDefaultState 内部已有 silent fallback 返回 Normal，无需二次 ?? 保护
                var defaultState = _areaLightingTable != null
                    ? _areaLightingTable.GetDefaultState(_areaId)
                    : AreaLightingState.Normal;

                EventBus.Instance.Publish(new AreaLightingChangedEvent
                {
                    areaId = _areaId,
                    lightingState = defaultState,
                    isPlayerInside = false,
                    timestamp = Time.time
                });
            }
        }
    }

    /// <summary>
    /// 尝试获取 Player 组件
    /// </summary>
    private bool TryGetPlayer(Collider other, out IPlayer player)
    {
        player = other.GetComponent<IPlayer>();
        if (player != null)
            return true;

        // 兼容 LayerMask 检测（备用方案）
        if (other.gameObject.layer == LayerMask.NameToLayer("Player"))
        {
            // 手动查找 IPlayer 实现
            player = other.GetComponentInParent<IPlayer>();
            return player != null;
        }

        return false;
    }
}
```

**职责划分**：
- `LightingSystemManager`：维护 `Dictionary<string, AreaLightingState> _areaLightingStateCache`，存储每个区域的运行时光照状态
- `AreaLightingDetector`：挂载在场景触发器上，检测玩家进出事件并发布 `AreaLightingChangedEvent`
- 下游系统（NPC AI、LOS、Sanity/Rage）：订阅 `AreaLightingChangedEvent` 后，通过 `AreaLightingTable.LookupCoefficients()` 方法获取光照系数

**下游系统获取系数的方式**：

> **说明**：World Layer 系统不持有下游系统引用，但下游系统可以通过 `LightingSystemQueryBus` 查询上游状态。这是单向依赖，下游持有上游的查询接口而非直接引用。

```csharp
/// <summary>
/// 光照系统查询接口
/// 下游系统通过此接口查询当前的光照系数
/// 实现：LightingSystemQueryBus : IQueryBus
/// </summary>
public struct QueryAreaLightingCoefficients
{
    /// <summary>区域 ID</summary>
    public string areaId;
}

/// <summary>
/// 光照系数查询响应
/// </summary>
public struct AreaLightingCoefficients
{
    /// <summary>该区域的光照状态</summary>
    public AreaLightingState lightingState;

    /// <summary>阴影潜行加成乘数</summary>
    public float shadowStealthMultiplier;

    /// <summary>光照灵敏度乘数</summary>
    public float lightSensitivityMultiplier;

    /// <summary>最终光照强度（0.0~1.0）</summary>
    public float finalIllumination;
}
```

**调用示例**（NPC AI System）：
```csharp
var coeffs = QueryBus.Instance.Query<QueryAreaLightingCoefficients, AreaLightingCoefficients>(
    new QueryAreaLightingCoefficients { areaId = playerAreaId }
);
float stealthBonus = coeffs.shadowStealthMultiplier;
```

**区域重叠处理**：
当玩家同时处于多个区域时（如站在门口），采用以下仲裁规则：
1. 获取所有 `isPlayerInside=true` 的区域
2. 使用 `AreaLightingStateExtensions.GetWorst()` 获取最严重的光照状态：
   - `PitchBlack (severity=4) > Dim (severity=3) > Normal (severity=2) > Bright (severity=1)`
3. 使用最严重的光照状态作为最终判定

**叠加计算公式**：
```
finalIllumination = Clamp(timeOfDayIllumination + illuminationOffset, 0.0, 1.0)
finalStealthBonus = timeOfDayShadowStealthBonus × shadowStealthMultiplier
```

**数值示例**：
| 场景 | timeOfDayIllumination | illuminationOffset | finalIllumination | timeOfDayStealthBonus | shadowStealthMultiplier | finalStealthBonus |
|------|---------------------|-------------------|-------------------|----------------------|------------------------|-------------------|
| Day + Normal | 1.0 | 0.0 | 1.0 | 1.0 | 1.0 | 1.0 |
| Night + Normal | 0.1 | 0.0 | 0.1 | 1.5 | 1.0 | 1.5 |
| Day + Dim | 1.0 | -0.2 | 0.8 | 1.0 | 1.3 | 1.3 |
| Night + PitchBlack | 0.1 | -0.5 | 0.0 | 1.5 | 2.0 | 3.0 |
| Dawn + Bright | 0.4 | +0.3 | 0.7 | 1.2 | 0.8 | 0.96 |

**Weather + Lighting 感知系数叠加规则**：

详细组合规则见 [shared-types.md §22.4](./shared-types.md#224-感知系数叠加规则weather-×-lighting)。

**本 ADR 简述**：
- `shadowStealthMultiplier`：由 Lighting System 独立维护，Weather System 不提供对应折扣
- `lightSensitivityMultiplier`：与 Weather 的 `visionAngleMultiplier` 乘法组合
- `visionDistanceMultiplier`：由 Weather System 独立维护

**叠加示例**：Night + PitchBlack + Storm
```
最终视野角度 = Weather.visionAngleMultiplier × Lighting.lightSensitivityMultiplier
            = 0.5 × 0.5 = 0.25

最终潜行加成 = Lighting.shadowStealthMultiplier
            = 2.0（与 Weather 无关）
```

```csharp
/// <summary>
/// 区域光照状态严重程度
/// 用于区域重叠时的仲裁
/// 注意：数值越大表示光照越暗、潜行加成越高
/// </summary>
public static class AreaLightingStateExtensions
{
    /// <summary>
    /// 获取光照状态的严重程度（用于区域重叠仲裁）
    /// 数值越大 = 光照越暗 = 潜行加成越高 = 越优先
    /// 注意：Bright=1 为最亮（数值最小），PitchBlack=4 为最暗（数值最大）
    /// </summary>
    public static int GetSeverity(this AreaLightingState state)
    {
        return state switch
        {
            AreaLightingState.Bright => 1,      // 最亮，潜行加成最低
            AreaLightingState.Normal => 2,       // 正常
            AreaLightingState.Dim => 3,          // 昏暗，较高潜行加成
            AreaLightingState.PitchBlack => 4,   // 最暗，最高潜行加成
            _ => 2
        };
    }

    /// <summary>
    /// 获取最严重的光照状态
    /// </summary>
    public static AreaLightingState GetWorst(params AreaLightingState[] states)
    {
        AreaLightingState result = AreaLightingState.Normal;
        int maxSeverity = 0;
        foreach (var state in states)
        {
            int severity = state.GetSeverity();
            if (severity > maxSeverity)
            {
                maxSeverity = severity;
                result = state;
            }
        }
        return result;
    }
}
```

**定义位置**：`Assets/Game/Core/Environment/Lighting/Extensions/AreaLightingStateExtensions.cs`

> **说明**：扩展方法放在独立文件中，而非 `AreaLightingState.cs` 本身，是为了保持类型定义的纯净性。扩展方法作为"工具类"性质的代码，与核心枚举定义分离。
```

### Phase 4: 下游集成
- [ ] NPC AI System 订阅光照事件，实现阴影潜行加成
- [ ] LOS System 订阅光照事件，实现明暗区域判定
- [ ] Sanity/Rage System 订阅光照事件，实现幽暗恐惧
- [ ] Screen Effects System 订阅光照事件，实现暗角/色调变化

---

## Validation Criteria

| ID | 验收条件 |
|----|---------|
| VC-1 | 时段切换时，所有下游系统正确接收 `LightingStateChangedEvent` |
| VC-2 | 玩家进入 PitchBlack 区域时，NPC AI 的阴影潜行加成生效 |
| VC-3 | 夜晚时段，NPC 光照感知灵敏度下降，玩家可更容易潜行 |
| VC-4 | 区域切换光照状态时，过渡平滑无跳跃 |
| VC-5 | 性能测试：时段更新时 CPU 占用 < 0.1ms |

---

## Related Decisions

- [ADR-0003: 系统分层架构](./adr-0003-system-layers.md) — World Layer 定义
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC AI 感知系统
- [ADR-0007: LOS 系统架构](./adr-0007-los-system-architecture.md) — LOS 感知折扣
- [ADR-0017: Sanity/Rage 系统](./adr-0017-sanity-rage-meter-architecture.md) — 氛围反馈
- [ADR-0021: Weather System](./adr-0021-weather-system-architecture.md) — 天气与光照交互
- [ADR-0023: Screen Effects 系统](./adr-0023-screen-effects-system-architecture.md) — 光照视觉效果请求
- [shared-types.md §22](./shared-types.md#22-游戏时间接口与-world-layer-时间事件) — IGameTimeProvider、GameHourChangedEvent、ScreenEffectRevokeEvent、感知系数叠加规则权威定义
