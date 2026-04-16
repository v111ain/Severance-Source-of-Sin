# ADR-0023: 屏幕特效系统 (Screen Effects System) 架构决策

## Status
**Accepted**

## Date
2026-04-11

## Last Updated
2026-04-15 (v2: ScreenEffectType.Jitter→UI_Jitter 重命名；CameraShakeRequestEvent 专用于物理震动；明确优先级范围 0-100)

## Context

### Problem Statement

屏幕特效系统是《断绝：罪恶之源》Infrastructure Layer 的核心组成，负责将游戏事件转化为统一的视觉反馈。多个系统（Sanity/Rage、Health、Weather、Lighting、Environment Interaction）都需要屏幕特效支持，但缺乏统一的效果管理。系统需要解决：

1. **效果统一管理**：所有屏幕特效（暗角、噪点、饱和度、色调、模糊、震动）集中管理，避免各系统重复实现
2. **效果叠加与优先级**：多个系统可能同时请求效果（如 Sanity 低 + 受伤），需要正确的叠加规则
3. **平滑过渡**：效果变化需要平滑插值，避免视觉跳跃
4. **解耦架构**：屏幕特效系统是被动服务，不主动轮询，响应来自各系统的效果请求

### Constraints

- **架构约束**：遵循 ADR-0003 Infrastructure Layer 定义 — 横向基础设施层，被所有层依赖
- **性能约束**：视觉效果必须在 < 1ms 内完成，不影响游戏帧率
- **兼容性约束**：支持 PC & PS5 平台，需要适配不同屏幕比例
- **可调试性**：提供 Debug 界面查看当前活跃效果和参数

### Requirements

- **必须**：定义所有屏幕特效类型（Vignette / Noise / Saturation / Hue / Blur / Jitter / ChromaticAberration / FilmGrain）
- **必须**：定义效果请求结构（ScreenEffectRequestEvent）和效果撤销机制
- **必须**：定义效果叠加规则（多系统同时请求时的仲裁逻辑）
- **必须**：定义平滑插值机制（效果渐变时长和曲线）
- **必须**：定义与各系统的接口（Sanity/Rage、Health、Weather、Lighting、Environment Interaction）

---

## Decision

### 架构决策

采用**请求驱动 + 优先级叠加 + 中心化控制**架构：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Screen Effects System 架构                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【Infrastructure Layer — 横向基础设施，被所有层依赖】                          │
│                                                                              │
│  【效果请求来源】                                                            │
│                                                                              │
│  ┌──────────────────┐    VignetteRequest / NoiseRequest                     │
│  │  Sanity/Rage     │ ──────────────────────────────────────────────────┐  │
│  └──────────────────┘    SaturationRequest / JitterRequest                   │  │
│                                                                            │  │
│  ┌──────────────────┐    DamageVignetteRequest                            │  │
│  │  Health System   │ ──────────────────────────────────────────────────┘  │
│  └──────────────────┘                                                        │
│                                                                            │  │
│  ┌──────────────────┐    WeatherOverlayRequest                             │  │
│  │  Weather System  │ ──────────────────────────────────────────────────┐  │
│  └──────────────────┘                                                      │  │
│                                                                            │  │
│  ┌──────────────────┐    LightingOverlayRequest                            │  │
│  │  Lighting System │ ──────────────────────────────────────────────────┘  │
│  └──────────────────┘                                                        │
│                                                                            │  │
│  ┌──────────────────┐    EnvironmentalFXRequest                           │  │
│  │  Environment     │ ──────────────────────────────────────────────────┐  │
│  └──────────────────┘                                                      │  │
│                                                                            │  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    ScreenEffectsManager (MonoBehaviour)                    │   │
│  │  - 接收所有效果请求                                                     │   │
│  │  - 维护 EffectLayerStack（效果层叠加栈）                                 │   │
│  │  - 计算最终效果参数（按优先级仲裁）                                       │   │
│  │  - 应用到后处理管线                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    后处理管线 (Post-Processing Pipeline)                    │   │
│  │                                                                       │   │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐  │   │
│  │  │ Vignette│ → │ Noise   │ → │Saturation│ → │  Blur   │ → │ Jitter  │  │   │
│  │  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘  │   │
│  │                                                                       │   │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐             │   │
│  │  │ ChromaticAbb │   │  FilmGrain  │   │   Hue Shift     │             │   │
│  │  └─────────────┘   └─────────────┘   └─────────────────┘             │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ScreenEffectSource 枚举

```csharp
/// <summary>
/// 屏幕特效请求来源系统枚举
/// </summary>
public enum ScreenEffectSource
{
    SanityRageSystem,  // Sanity/Rage 心理状态系统
    HealthSystem,       // Health 生命值系统
    WeatherSystem,      // Weather 天气系统
    LightingSystem,     // Lighting 光照系统
    EnvironmentSystem,  // Environment 环境交互系统
    DialogueSystem,     // Dialogue 对话系统
    CombatSystem,       // Combat 战斗系统
    PlayerSystem,       // Player 玩家系统（受伤、死亡等）
    TutorialSystem,     // Tutorial 教程系统
    AudioSystem,        // Audio 音频系统（爆炸/处决等音效触发震动）
    SceneManagement,     // SceneManagement 场景管理系统
    Generic             // 通用/默认来源（用于未分类的其他效果）
}
```

**定义位置**：`Assets/Game/Infrastructure/ScreenEffects/Types/ScreenEffectSource.cs`

### ScreenEffectType 枚举

```csharp
/// <summary>
/// 屏幕特效类型枚举
/// [Flags] 用于表示"需要同时启用的多个效果类型组合"，而非"支持独立强度参数"
///
/// <para><b>强度控制说明：</b></para>
/// <list type="bullet">
///   <item>此枚举仅标识效果种类，不支持各效果独立控制强度</item>
///   <item>当 effectType 组合多个效果时，所有效果共享同一个 intensity 参数</item>
///   <item>如需独立控制各效果强度，应发送多个独立的 ScreenEffectRequestEvent</item>
/// </list>
/// </summary>
[Flags]
public enum ScreenEffectType
{
    None = 0,
    Vignette = 1 << 0,        // 暗角
    Noise = 1 << 1,            // 噪点
    Saturation = 1 << 2,       // 饱和度
    Hue = 1 << 3,              // 色调偏移
    Blur = 1 << 4,             // 模糊
    UI_Jitter = 1 << 5,         // 准星抖动（专负责准星/HUD Transform 抖动，与 CameraShakeManager 的物理震动解耦）
    ChromaticAberration = 1 << 6, // 色差
    FilmGrain = 1 << 7,        // 胶片颗粒
    DamageFlash = 1 << 8,       // 受伤闪红
    ExtractionPulse = 1 << 9,    // 撤离脉冲
    Fade = 1 << 10              // 淡入淡出（用于场景切换）
}
```

**定义位置**：`Assets/Game/Infrastructure/ScreenEffects/Types/ScreenEffectType.cs`

### ScreenEffectRequestEvent 结构

```csharp
/// <summary>
/// 屏幕特效请求事件
/// 由各游戏系统通过 EventBus 发送，ScreenEffectsManager 接收并处理
/// 注意：此结构直接作为 EventBus 的事件类型使用，
/// 包含完整的请求信息（来源、目标效果、强度、持续时间等），
/// 符合"结构化事件"模式。
///
/// 事件命名规范：遵循 *Event 后缀约定，与 WeatherStateChangedEvent、LightingStateChangedEvent 保持一致。
/// </summary>
public struct ScreenEffectRequestEvent
{
    /// <summary>请求来源系统（类型安全，使用枚举）</summary>
    public ScreenEffectSource sourceSystem;

    /// <summary>请求来源的唯一请求者 ID（如 npc_id、area_id）</summary>
    public string requesterId;

    /// <summary>效果类型（[Flags] 可组合枚举）</summary>
    public ScreenEffectType effectType;

    /// <summary>
    /// 效果强度（0.0~1.0）
    ///
    /// 使用场景说明：
    /// - **单效果请求**（推荐）：effectType 为单一效果（如 Vignette），
    ///   intensity 直接控制该效果的强度。
    ///
    /// - **多效果请求**（慎用）：effectType 可组合多个效果（如 Vignette | Noise），
    ///   但此时所有效果**共享同一个 intensity 值**。
    ///   适用于效果需要同步变化的场景（如 "极端恐惧" 同时触发暗角+噪点+色差）。
    ///
    /// - **独立强度控制**：如需不同效果有不同强度，应发送多个独立的 ScreenEffectRequestEvent。
    ///
    /// [Flags] 枚举设计用于表示"需要同时启用的多个效果类型"，而非"多个独立请求"。
    /// </summary>
    public float intensity;

    /// <summary>
    /// 持续时间（秒）
    ///
    /// 语义说明：
    /// - **permanent = true** 时：效果永久生效，直到被显式撤销（RevokeScreenEffect）
    /// - **permanent = false** 时：效果在 duration 秒后自动淡出并移除
    /// </summary>
    public float duration;

    /// <summary>
    /// 是否永久生效（覆盖 duration 的隐式语义）
    /// - true：永久生效，由请求方通过 RevokeScreenEffect 撤销
    /// - false：使用 duration 控制生命周期
    ///
    /// 典型使用场景：
    /// - 天气效果：permanent=true（由天气系统管理撤销）
    /// - 区域效果：permanent=true（区域离开时撤销）
    /// - 受伤闪红：permanent=false，duration=0.3s
    /// </summary>
    public bool permanent;

    /// <summary>
    /// 超时时间（秒），安全网机制
    ///
    /// 语义说明：
    /// - **timeout > 0**：效果在 timeout 秒后强制移除（防止请求方崩溃未撤销）
    /// - **timeout <= 0**：永不超时，仅受 permanent/duration 控制
    ///
    /// 典型使用场景：
    /// - 受伤闪红：timeout=1.0s（安全网，防止异常未撤销）
    /// - 天气效果：timeout<=0（永不超时，由天气系统管理撤销）
    ///
    /// **重要**：timeout 是安全网，不是正常的生命周期控制。正常撤销应通过 RevokeScreenEffect。
    /// </summary>
    public float timeout;

    /// <summary>优先级（数值越高越优先）</summary>
    public int priority;

    /// <summary>特效参数（根据 effectType 不同而不同）</summary>
    public ScreenEffectParams parameters;

    /// <summary>请求时间戳</summary>
    public float timestamp;
}

/// <summary>
/// 屏幕特效参数
/// </summary>
[System.Serializable]
public struct ScreenEffectParams
{
    /// <summary>暗角颜色（RGBA）</summary>
    public Color vignetteColor;

    /// <summary>暗角强度（0.0~1.0）</summary>
    public float vignetteIntensity;

    /// <summary>噪点颜色</summary>
    public Color noiseColor;

    /// <summary>噪点强度（0.0~1.0）</summary>
    public float noiseIntensity;

    /// <summary>色调偏移（0.0~1.0，对应 0°~360°）</summary>
    public float hueShift;

    /// <summary>饱和度乘数（0.0=灰度，1.0=正常）</summary>
    public float saturationMultiplier;

    /// <summary>模糊半径（0.0~1.0）</summary>
    public float blurRadius;

    /// <summary>模糊质量（1/2/3）</summary>
    public int blurQuality;

    /// <summary>震动方向（2D 向量）</summary>
    public Vector2 shakeDirection;

    /// <summary>震动强度（0.0~8.0）</summary>
    public float shakeIntensity;

    /// <summary>震动频率（Hz）</summary>
    public float shakeFrequency;

    /// <summary>色差强度（0.0~1.0）</summary>
    public float chromaticIntensity;

    /// <summary>胶片颗粒密度（0.0~1.0）</summary>
    public float grainDensity;

    /// <summary>淡入时长（秒）</summary>
    public float fadeInDuration;

    /// <summary>淡出时长（秒）</summary>
    public float fadeOutDuration;
}
```

**定义位置**：`Assets/Game/Infrastructure/ScreenEffects/Events/ScreenEffectRequestEvent.cs`

```csharp
/// <summary>
/// 屏幕特效撤销事件
/// 与 ScreenEffectRequestEvent 对称，同样通过 EventBus 发布，保持解耦
/// 请求方（如 DialogueSystem）在效果结束时发布此事件，
/// 无需持有 ScreenEffectsManager 引用
/// </summary>
public struct ScreenEffectRevokeEvent
{
    /// <summary>撤销来源系统</summary>
    public ScreenEffectSource sourceSystem;

    /// <summary>要撤销的请求者 ID（与原 ScreenEffectRequestEvent.requesterId 一致）</summary>
    public string requesterId;

    /// <summary>事件时间戳</summary>
    public float timestamp;
}
```

**定义位置**：`Assets/Game/Infrastructure/ScreenEffects/Events/ScreenEffectRequestEvent.cs`（与 ScreenEffectRequestEvent 共文件）

```csharp
/// <summary>
/// 效果层（管理单个来源的效果请求）
/// </summary>
internal class ScreenEffectLayer
{
    public ScreenEffectSource sourceSystem;  // 来源系统（枚举）
    public string requesterId;                // 请求者 ID
    public ScreenEffectRequestEvent request;        // 当前请求
    public float elapsedTime;                 // 已过时间
    public float currentIntensity;             // 当前强度（用于插值）
    public float timeout;                      // 超时时间，<= 0 表示永不超时
    public bool isExpired;                     // 是否已过期（用于标记待移除）
}
```

### ScreenEffectsManager 核心逻辑

```csharp
/// <summary>
/// 屏幕特效管理器
/// 生命周期：由 GameBootstrap 在场景加载时创建，跨场景持久化（ DontDestroyOnLoad）
/// 通过 EventBus 订阅 ScreenEffectRequestEvent，被各系统调用
/// </summary>
public class ScreenEffectsManager : MonoBehaviour
{
    // 效果层栈（按优先级排序）
    private List<ScreenEffectLayer> _effectLayers = new();

    // 当前最终效果值
    private ScreenEffectParams _finalParams = new();

    // 默认超时时间
    [SerializeField] private float _defaultTimeout = 10f;

    // 排序脏标记：避免每帧排序
    private bool _isDirty = false;

    private void Start()
    {
        // 同时订阅 Request 和 Revoke 事件，保持接口对称、调用方无需持有本组件引用
        EventBus.Instance.Subscribe<ScreenEffectRequestEvent>(OnScreenEffectRequested);
        EventBus.Instance.Subscribe<ScreenEffectRevokeEvent>(OnScreenEffectRevoked);
    }

    private void OnDestroy()
    {
        EventBus.Instance.Unsubscribe<ScreenEffectRequestEvent>(OnScreenEffectRequested);
        EventBus.Instance.Unsubscribe<ScreenEffectRevokeEvent>(OnScreenEffectRevoked);
    }

    /// <summary>
    /// 获取当前计算出的最终效果参数
    /// 供 ScreenEffectsVolumeController（或其他后处理驱动组件）在 LateUpdate 中调用
    /// </summary>
    public ScreenEffectParams GetFinalParams() => _finalParams;

    private void OnScreenEffectRequested(ScreenEffectRequestEvent request)
    {
        RequestScreenEffect(request);
    }

    private void OnScreenEffectRevoked(ScreenEffectRevokeEvent revoke)
    {
        RevokeScreenEffect(revoke.sourceSystem, revoke.requesterId);
    }

    // 低优先级效果层叠加衰减系数（可配置）
    [Header("Tuning")]
    [SerializeField] private float _additiveLayerDecay = 0.5f;

    // 效果强度上限（可配置）
    [SerializeField] private float _maxVignetteIntensity = 1.0f;
    [SerializeField] private float _maxNoiseIntensity = 1.0f;
    [SerializeField] private float _maxShakeIntensity = 8.0f;

    /// <summary>
    /// 接收效果请求
    ///
    /// 调用方式（任选其一）：
    /// 1. **通过 EventBus（推荐）**：发布 ScreenEffectRequestEvent，由 ScreenEffectsManager 订阅处理
    /// 2. **直接调用**：直接调用此方法，适用于需要立即生效的场景
    ///
    /// 两种方式效果相同。EventBus 方式是异步的，直接调用是同步的。
    /// </summary>
    /// <param name="request">效果请求</param>
    public void RequestScreenEffect(ScreenEffectRequestEvent request)
    {
        // 检查是否已有该请求者的效果层
        var existingLayer = _effectLayers.FirstOrDefault(l =>
            l.sourceSystem == request.sourceSystem &&
            l.requesterId == request.requesterId);

        if (existingLayer != null)
        {
            // 更新现有效果层
            existingLayer.request = request;
            existingLayer.elapsedTime = 0f;
            existingLayer.isExpired = false;
            existingLayer.timeout = request.timeout > 0 ? Time.time + request.timeout : -1f;
        }
        else
        {
            // 添加新效果层
            _effectLayers.Add(new ScreenEffectLayer
            {
                sourceSystem = request.sourceSystem,
                requesterId = request.requesterId,
                request = request,
                elapsedTime = 0f,
                currentIntensity = 0f,
                timeout = request.timeout > 0 ? Time.time + request.timeout : -1f,
                isExpired = false
            });
        }

        // 标记需要排序（延迟到 Update 中排序）
        _isDirty = true;
    }

    /// <summary>
    /// 撤销效果请求（内部实现，外部通过 EventBus 发布 ScreenEffectRevokeEvent 触发）
    /// </summary>
    private void RevokeScreenEffect(ScreenEffectSource sourceSystem, string requesterId)
    {
        _effectLayers.RemoveAll(l =>
            l.sourceSystem == sourceSystem &&
            l.requesterId == requesterId);
        _isDirty = true;
    }

    /// <summary>
    /// 每帧更新
    /// </summary>
    /// <remarks>
    /// duration 与 fadeOutDuration 交互规则：
    /// - duration > 0：效果在指定时长后开始淡出，淡出时长由 fadeOutDuration 控制
    /// - duration <= 0：效果视为"永久效果"，忽略 fadeOutDuration（永不就绪触发淡出）
    ///   永久效果只能通过显式撤销（RevokeScreenEffect）或超时（timeout）移除
    /// </remarks>
    private void Update()
    {
        // 仅在脏标记为真时排序
        if (_isDirty)
        {
            _effectLayers.Sort((a, b) => b.request.priority.CompareTo(a.request.priority));
            _isDirty = false;
        }

        // 检查超时
        CheckTimeouts();

        // 更新每个效果层的时间和强度
        foreach (var layer in _effectLayers)
        {
            layer.elapsedTime += Time.deltaTime;

            // 计算当前强度（使用 SmoothStep 曲线进行淡入/淡出）
            float fadeIn = layer.elapsedTime < layer.request.parameters.fadeInDuration
                ? SmoothStep(layer.elapsedTime / layer.request.parameters.fadeInDuration)
                : 1.0f;

            // fadeOut：仅当 permanent=false 时生效（永久效果不触发淡出）
            float fadeOut = 1.0f;
            if (!layer.request.permanent &&
                layer.elapsedTime > layer.request.duration - layer.request.parameters.fadeOutDuration)
            {
                fadeOut = SmoothStep((layer.request.duration - layer.elapsedTime) / layer.request.parameters.fadeOutDuration);
            }

            layer.currentIntensity = layer.request.intensity * fadeIn * fadeOut;
        }

        // 移除已过期且强度已衰减到 0 的效果层
        // 注意：对于设置了 fadeOutDuration 的效果层，
        // 需要等待 currentIntensity 衰减到 0 才能移除（确保淡出完成）
        if (_effectLayers.RemoveAll(l =>
            l.isExpired && l.currentIntensity <= 0) > 0)
        {
            _isDirty = true;
        }

        // 计算最终效果参数（按优先级叠加）
        CalculateFinalParams();

        // 应用到后处理管线
        ApplyToPostProcessing();
    }

    /// <summary>
    /// 检查效果层超时
    /// </summary>
    private void CheckTimeouts()
    {
        foreach (var layer in _effectLayers)
        {
            if (!layer.isExpired && layer.timeout > 0 && Time.time > layer.timeout)
            {
                layer.isExpired = true;
                Debug.LogWarning($"[ScreenEffects] Layer timeout: {layer.sourceSystem}/{layer.requesterId}");
            }
        }
    }

    /// <summary>
    /// SmoothStep 插值曲线
    /// 确保平滑过渡，避免线性插值带来的视觉跳跃
    /// </summary>
    private float SmoothStep(float t)
    {
        return t * t * (3.0f - 2.0f * t);
    }

    /// <summary>
    /// 计算最终效果参数（核心叠加逻辑）
    /// 注意：仅处理未过期的效果层，过期层在移除前仍保留在列表中但被跳过
    /// </summary>
    private void CalculateFinalParams()
    {
        // 重置
        _finalParams = new ScreenEffectParams();

        // 过滤出未过期且未完成的效果层（使用 IsEffectActive 判断）
        var activeLayers = _effectLayers.Where(l => IsEffectActive(l)).ToList();

        // 获取最高优先级的效果层作为基准
        if (activeLayers.Count > 0)
        {
            var topLayer = activeLayers[0];
            _finalParams = BlendParams(_finalParams, topLayer.request.parameters, topLayer.currentIntensity);
        }

        // 低优先级效果层的强度叠加（使用 Add 混合，并乘以衰减系数）
        for (int i = 1; i < activeLayers.Count; i++)
        {
            var layer = activeLayers[i];
            _finalParams = BlendParamsAdditive(_finalParams, layer.request.parameters, layer.currentIntensity * _additiveLayerDecay);
        }

        // 移除已过期且强度已衰减到 0 的效果层
        // 注意：对于设置了 fadeOutDuration 的效果层，
        // 需要等待 currentIntensity 衰减到 0 才能移除（确保淡出完成）
        if (_effectLayers.RemoveAll(l =>
            l.isExpired && l.currentIntensity <= 0) > 0)
        {
            _isDirty = true;
        }
    }

    /// <summary>
    /// 检查效果层是否处于活跃状态
    /// 活跃 = 未过期，且（permanent=true 表示永久 或 elapsedTime<duration 表示仍在持续）
    /// </summary>
    private bool IsEffectActive(ScreenEffectLayer layer)
    {
        if (layer.isExpired) return false;
        // permanent = true 表示永久效果（不因时间流逝而结束）
        if (layer.request.permanent) return true;
        return layer.elapsedTime < layer.request.duration;
    }

    /// <summary>
    /// 按优先级混合效果参数（高优先级覆盖）
    /// </summary>
    private ScreenEffectParams BlendParams(ScreenEffectParams baseParams, ScreenEffectParams overlayParams, float intensity)
    {
        if (intensity <= 0f) return baseParams;

        return new ScreenEffectParams
        {
            vignetteColor = Color.Lerp(baseParams.vignetteColor, overlayParams.vignetteColor, intensity),
            vignetteIntensity = Mathf.Lerp(baseParams.vignetteIntensity, overlayParams.vignetteIntensity, intensity),

            noiseColor = Color.Lerp(baseParams.noiseColor, overlayParams.noiseColor, intensity),
            noiseIntensity = Mathf.Lerp(baseParams.noiseIntensity, overlayParams.noiseIntensity, intensity),
            grainDensity = Mathf.Lerp(baseParams.grainDensity, overlayParams.grainDensity, intensity),

            hueShift = Mathf.Lerp(baseParams.hueShift, overlayParams.hueShift, intensity),
            saturationMultiplier = Mathf.Lerp(baseParams.saturationMultiplier, overlayParams.saturationMultiplier, intensity),

            blurRadius = Mathf.Lerp(baseParams.blurRadius, overlayParams.blurRadius, intensity),
            blurQuality = Mathf.Max(baseParams.blurQuality, overlayParams.blurQuality), // MaxQuality 模式，取最高质量

            // Jitter: PriorityDominant 混合
            // 高优先级层主导：方向和频率直接覆盖（baseParams 被忽略），
            // 强度按 intensity 在两层之间 Lerp 插值
            // 设计理由：震动方向应与高优先级效果一致（如受伤时的震动方向应主导）
            shakeDirection = overlayParams.shakeDirection; // 高优先级直接覆盖方向
            shakeIntensity = Mathf.Lerp(baseParams.shakeIntensity, overlayParams.shakeIntensity, intensity);
            shakeFrequency = overlayParams.shakeFrequency; // 高优先级直接覆盖频率

            chromaticIntensity = Mathf.Lerp(baseParams.chromaticIntensity, overlayParams.chromaticIntensity, intensity),

            fadeInDuration = overlayParams.fadeInDuration, // Override
            fadeOutDuration = overlayParams.fadeOutDuration  // Override
        };
    }

    /// <summary>
    /// 叠加混合效果参数（低优先级效果累加）
    /// </summary>
    private ScreenEffectParams BlendParamsAdditive(ScreenEffectParams baseParams, ScreenEffectParams overlayParams, float intensity)
    {
        if (intensity <= 0f) return baseParams;

        return new ScreenEffectParams
        {
            // Vignette: Additive 混合，但有上限
            vignetteColor = Color.Lerp(baseParams.vignetteColor, overlayParams.vignetteColor, intensity),
            vignetteIntensity = Mathf.Min(_maxVignetteIntensity, baseParams.vignetteIntensity + overlayParams.vignetteIntensity * intensity),

            // Noise: Additive 混合，有上限
            noiseColor = Color.Lerp(baseParams.noiseColor, overlayParams.noiseColor, intensity),
            noiseIntensity = Mathf.Min(_maxNoiseIntensity, baseParams.noiseIntensity + overlayParams.noiseIntensity * intensity),

            // GrainDensity: Additive 混合（FilmGrain 和 Noise 共享此参数）
            grainDensity = baseParams.grainDensity + overlayParams.grainDensity * intensity,

            // Hue: 叠加偏移（AdditiveWrap 模式，超出 0~360 范围时回绕）
            hueShift = Mathf.Repeat(baseParams.hueShift + overlayParams.hueShift * intensity, 1f),

            // Saturation: Lerp 混合（按强度插值，避免低 intensity 下 Min 把饱和度压到接近 0）
            // intensity 已经乘了 _additiveLayerDecay，低优先级层对饱和度的影响自然衰减
            saturationMultiplier = Mathf.Lerp(baseParams.saturationMultiplier, overlayParams.saturationMultiplier, intensity),

            // Blur: Additive，但有上限
            blurRadius = Mathf.Min(1.0f, baseParams.blurRadius + overlayParams.blurRadius * intensity),
            blurQuality = baseParams.blurQuality, // 不被 overlay 影响

            // Jitter: Additive 混合（低优先级层贡献）
            // 低优先级层不覆盖方向和频率（由 BlendParams 中的高优先级层决定），
            // 仅累加强度（有上限）
            // 设计理由：抖动方向应由最高优先级层决定，低优先级层仅增加抖动强度
            // 注意：字段命名统一为 shake*（与 ScreenEffectParams 结构体保持一致）
            shakeDirection = baseParams.shakeDirection, // 保持高优先级层的主导方向
            shakeIntensity = Mathf.Min(_maxShakeIntensity, baseParams.shakeIntensity + overlayParams.shakeIntensity * intensity),
            shakeFrequency = baseParams.shakeFrequency, // 保持高优先级层的主导频率

            // Chromatic: Additive，有上限
            chromaticIntensity = Mathf.Min(1.0f, baseParams.chromaticIntensity + overlayParams.chromaticIntensity * intensity),

            fadeInDuration = baseParams.fadeInDuration,
            fadeOutDuration = baseParams.fadeOutDuration
        };
    }
}
```

---

## ScreenEffectSource 与 ScreenEffectType 正交性说明

`ScreenEffectSource`（请求来源）与 `ScreenEffectType`（效果类型）是**正交维度**：

- **Source**：标识"谁请求了效果"，用于撤销（RevokeEvent）和调试
- **Type**：标识"什么效果"，决定如何渲染和混合

**规则**：
1. 任意 `ScreenEffectSource` 可以请求任意 `ScreenEffectType`（无限制映射）
2. 同一 Source + requesterId + Type 组合仅保留最新请求（覆盖更新）
3. 撤销时需精确匹配 sourceSystem + requesterId（与 Type 无关）

**典型 Source → Type 映射**（非强制约束，仅供参考）：

| Source | 典型请求的 Type |
|--------|---------------|
| SanityRageSystem | Vignette, Noise, Saturation, Blur |
| HealthSystem | Vignette (红), DamageFlash |
| WeatherSystem | Noise, Blur |
| LightingSystem | Vignette |
| DialogueSystem | Blur |
| CombatSystem | Jitter, ChromaticAberration |
| Generic | 用于未分类的临时效果（如调试期间的效果请求） |

**Generic 特殊说明**：
`Generic` 适用于临时性、调试性或无法归类的效果请求。使用场景包括：
- 开发期间的临时效果测试
- 不属于任何特定系统的全局效果（如全局屏幕闪烁）
- 未分类系统的一次性效果请求

---

## 效果类型详细定义

### 1. Vignette（暗角）

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `color` | Color | 暗角颜色 | (0, 0, 0, 1) |
| `intensity` | float | 暗角强度 0.0~1.0 | 0.0 |

**使用场景**：
- Sanity < 50 时触发（强度 = 1.0 - Sanity/100）
- 受伤时闪红（颜色切换为红色）
- 撤离脉冲动画

### 2. Noise（噪点）

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `color` | Color | 噪点颜色 | (0.5, 0.5, 0.5, 1) |
| `intensity` | float | 噪点强度 0.0~1.0 | 0.0 |
| `grainDensity` | float | 颗粒密度 | 1.0 |

**使用场景**：
- Sanity < 40 时触发（强度 = (40 - Sanity) / 40）
- 极端天气（暴风雨）

### 3. Saturation（饱和度）

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `multiplier` | float | 饱和度乘数 0.0~1.0 | 1.0 |

**使用场景**：
- Sanity < 20 时触发（multiplier = Sanity / 40）
- 夜晚氛围效果

### 4. Hue（色调偏移）

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `shift` | float | 色调偏移角度 0~360 | 0.0 |

**使用场景**：
- 特殊氛围效果（如中毒时绿色偏移）

### 5. Blur（模糊）

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `radius` | float | 模糊半径 0.0~1.0 | 0.0 |
| `quality` | int | 模糊质量（1/2/3） | 2 |

**使用场景**：
- 对话时轻微模糊背景
- Sanity = 0 时模糊增强

### 6. UI_Jitter（准星抖动）

> **命名说明**：`ScreenEffectType.Jitter` 已重命名为 `ScreenEffectType.UI_Jitter`，明确其职责范围。

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `direction` | Vector2 | 震动方向 | (0, 0) |
| `intensity` | float | 震动强度 0.0~8.0 | 0.0 |
| `frequency` | float | 震动频率 | 15.0 |

**使用场景**：
- Rage >= 50 时准星抖动（UI 层准星，非相机位移）
- 极端心理状态视觉反馈

> **ScreenEffect.UI_Jitter 职责边界说明**：ScreenEffect.UI_Jitter 专负责**准星/UI 抖动**（影响 HUD 准星 Transform），例如愤怒值高时准星不受控制地晃动。**物理震动**（爆炸冲击导致画面晃动）由 CameraShakeManager 处理（见 ADR-0026 §6 CameraShakeRequestEvent）。

### 7. Chromatic Aberration（色差）

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `intensity` | float | 色差强度 0.0~1.0 | 0.0 |

**使用场景**：
- 高速移动时轻微色差
- 受伤时闪动

### 8. Film Grain（胶片颗粒）

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `density` | float | 颗粒密度 0.0~1.0 | 0.0 |
| `response` | float | 响应曲线 | 0.8 |

**使用场景**：
- 复古风格氛围
- 夜间场景增强

---

## 优先级与叠加规则

### 优先级定义

| 优先级 | 来源系统 | 效果类型 | 说明 |
|--------|---------|---------|------|
| 100 | Health System | DamageFlash | 受伤闪红最高优先级 |
| 80 | Sanity/Rage | SOUL_SPLIT 状态 | 极端心理状态 |
| 70 | Sanity/Rage | FRENZIED/BROKEN | 高压心理状态 |
| 60 | Sanity/Rage | AGITATED/UNEASY | 中压心理状态 |
| 50 | Environment | 爆炸/火灾特效 | 环境事件 |
| 40 | Weather | 暴风雨/大雾 | 天气效果 |
| 30 | Lighting | 极暗区域 | 区域光照 |
| 20 | Dialogue | 对话模糊 | UI 交互 |
| 10 | Generic | 默认 | 最低优先级 |

### 效果混合规则

| 效果类型 | 混合模式 | 说明 |
|---------|---------|------|
| Vignette | LerpBlend | 高优先级层按 intensity 插值覆盖，低优先级层贡献衰减后的叠加值 |
| Noise | Additive | 所有层叠加（但有上限 1.0） |
| Saturation | LerpBlend | 高优先级层按 intensity 插值覆盖 |
| Blur | MaxQuality | 取最高模糊质量（高质量优先） |
| UI_Jitter | PriorityDominant | 高优先级层的方向和频率直接覆盖；强度按 intensity 插值；低优先级层仅通过 Additive 叠加强度 |
| ChromaticAberration | Additive | 所有层叠加（但有上限 1.0） |
| FilmGrain | Additive | 所有层叠加（无上限，密度可叠加） |
| Hue | AdditiveWrap | 色调偏移叠加，超出 0~360 范围时回绕 |

---

## 与各系统的接口

### Sanity/Rage 系统接口

```csharp
// 请求暗角
var vignetteRequest = new ScreenEffectRequestEvent
{
    sourceSystem = ScreenEffectSource.SanityRageSystem,
    requesterId = "sanity_vignette",
    effectType = ScreenEffectType.Vignette,
    intensity = 1.0f - sanity / 100f,
    duration = 0f,
    permanent = true, // 永久生效，由系统管理撤销
    priority = (int)(sanity < 20 ? 80 : (sanity < 40 ? 70 : 60))
};

// 请求噪点
var noiseRequest = new ScreenEffectRequestEvent
{
    sourceSystem = ScreenEffectSource.SanityRageSystem,
    requesterId = "sanity_noise",
    effectType = ScreenEffectType.Noise,
    intensity = sanity < 40 ? (40 - sanity) / 40 : 0f,
    duration = 0f,
    permanent = true,
    priority = (int)(sanity < 20 ? 80 : 60)
};
```

### Health System 接口

```csharp
// 受伤闪红
var damageFlash = new ScreenEffectRequestEvent
{
    sourceSystem = ScreenEffectSource.HealthSystem,
    requesterId = $"damage_{target_id}",
    effectType = ScreenEffectType.Vignette | ScreenEffectType.DamageFlash,
    intensity = 1.0f,
    duration = 0.3f,
    timeout = 1f, // 防止异常未撤销
    priority = 100, // 最高优先级
    parameters = new ScreenEffectParams
    {
        vignetteColor = new Color(1f, 0f, 0f, 1f),
        fadeInDuration = 0.05f,
        fadeOutDuration = 0.25f
    }
};
```

### Weather System 接口

```csharp
// 雨天效果
var rainOverlay = new ScreenEffectRequestEvent
{
    sourceSystem = ScreenEffectSource.WeatherSystem,
    requesterId = "rain",
    effectType = ScreenEffectType.Noise | ScreenEffectType.Blur,
    intensity = weatherIntensity,
    duration = 0f,
    permanent = true,
    priority = 40
};
```

### Lighting System 接口

```csharp
// 极暗区域
var darkOverlay = new ScreenEffectRequestEvent
{
    sourceSystem = ScreenEffectSource.LightingSystem,
    requesterId = $"area_{area_id}",
    effectType = ScreenEffectType.Vignette,
    intensity = lightingState == AreaLightingState.PitchBlack ? 0.5f : 0f,
    duration = 0f,
    permanent = true,
    priority = 30
};
```

### Dialogue System 接口

```csharp
// 对话时轻微模糊背景
var dialogueOverlay = new ScreenEffectRequestEvent
{
    sourceSystem = ScreenEffectSource.DialogueSystem,
    requesterId = $"dialogue_{dialogue_id}",
    effectType = ScreenEffectType.Blur,
    intensity = 0.3f,
    duration = 0f,
    permanent = true, // 永久生效，由系统管理撤销，持续到对话结束
    priority = 20, // 低优先级
    parameters = new ScreenEffectParams
    {
        blurRadius = 0.2f,
        blurQuality = 2, // 中等质量
        fadeInDuration = 0.3f,
        fadeOutDuration = 0.5f
    }
};

// 对话结束撤销效果（通过 EventBus 发布 RevokeEvent，无需持有 ScreenEffectsManager 引用）
EventBus.Instance.Publish(new ScreenEffectRevokeEvent
{
    sourceSystem = ScreenEffectSource.DialogueSystem,
    requesterId = $"dialogue_{dialogue_id}",
    timestamp = Time.time
});
```

---

## Alternatives Considered

### Alternative 1: 各系统自行实现屏幕特效

- **描述**：Sanity/Rage、Health 等系统各自实现自己的视觉效果
- **优点**：开发简单，各系统独立
- **缺点**：
  - 效果实现重复
  - 无法统一管理叠加规则
  - 可能冲突（如多个暗角效果叠加）
- **拒绝理由**：屏幕特效应该有统一管理，避免重复实现和冲突

### Alternative 2: 直接操作后处理组件

- **描述**：各系统直接获取后处理组件并修改参数
- **优点**：调用直接，性能略好
- **缺点**：
  - 引入对渲染系统的直接依赖
  - 无叠加机制
  - 无生命周期管理
- **拒绝理由**：应该通过请求-响应模式解耦

### Alternative 3: 单一全局效果

- **描述**：所有效果合并为一个全局参数
- **优点**：实现简单
- **缺点**：无法区分不同来源的效果
- **拒绝理由**：不同来源的效果需要独立管理和撤销

---

## Consequences

### Positive

- **统一管理**：所有屏幕特效集中管理，避免重复实现
- **优先级机制**：多系统同时请求时，有明确的仲裁规则
- **平滑过渡**：内置淡入/淡出机制，视觉体验流畅
- **可调试性**：提供 Debug 界面查看当前活跃效果
- **可扩展性**：新增效果类型只需扩展枚举和实现

### Negative

- **性能开销**：效果叠加计算有一定开销（但 < 1ms）
- **调参复杂性**：多效果组合需要仔细平衡
- **依赖管理**：所有特效请求需要显式撤销

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 效果叠加过度 | 极端情况下效果叠加过强，看不清画面 | 设置 `MaxVignetteIntensity` 等效果强度上限 |
| 忘记撤销请求 | 系统崩溃或异常时未撤销效果请求 | 提供超时自动撤销机制（`EffectTimeoutDefault`） |
| 性能峰值 | 多效果同时请求时帧率下降 | 限制 `MaxEffectLayers`，低优先级效果衰减叠加 |
| 忘记设置 permanent | 永久效果未设置 permanent=true，可能被 duration 机制误移除 | 设置 `permanent = true` 表示永久 |

---

## Performance Implications

| 指标 | 影响 | 说明 |
|------|------|------|
| **CPU** | < 1ms | 效果层数量通常 < 10，每帧更新计算量小 |
| **Memory** | 低 | 仅存储活跃效果层列表 |
| **GPU** | 中 | 取决于同时启用的效果数量 |
| **Network** | 无 | 本地系统 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 `ScreenEffectsManager` MonoBehaviour
- [ ] 定义 `ScreenEffectType` 枚举
- [ ] 实现 `ScreenEffectRequestEvent`、`ScreenEffectRevokeEvent` 和 `ScreenEffectLayer` 数据结构
- [ ] 实现请求接收逻辑（EventBus 订阅 + 公开方法直接调用）
- [ ] 实现撤销逻辑（EventBus 发布 `ScreenEffectRevokeEvent`）
- [ ] **实现 Debug 可视化界面**（显示所有活跃效果层、优先级、强度参数）

### Phase 2: 优先级与叠加
- [ ] 实现优先级排序机制（含脏标记优化）
- [ ] 实现效果混合规则
- [ ] 实现平滑插值（淡入/淡出）
- [ ] 实现超时自动撤销机制

### Phase 3: 后处理集成
- [ ] 集成 Unity Post-Processing Stack v2
- [ ] 实现各个特效 shader 参数映射
- [ ] 完善 Debug 界面（支持实时调节参数、开关单个效果层）

### 后处理管线实现

#### ScreenEffectsVolumeProfile

使用 Unity 的 Volume Framework，为每个效果类型创建对应的 Volume Component：

```csharp
/// <summary>
/// 屏幕特效 Volume Component
/// 挂载在 Volume 上，由 ScreenEffectsVolumeController 驱动参数
/// 注意：只需继承 VolumeComponent 并标注 [Serializable]，Unity URP 自动识别
/// （不存在 VolumeComponentFor 等非标准 Attribute，请勿添加）
/// </summary>
[System.Serializable]
public class ScreenEffectsVolumeComponent : VolumeComponent
{
    public ClampedFloatParameter vignetteIntensity = new(0f, 0f, 1f);
    public ColorParameter vignetteColor = new(Color.black);
    public ClampedFloatParameter noiseIntensity = new(0f, 0f, 1f);
    public ColorParameter noiseColor = new(Color.white);
    public ClampedFloatParameter saturationMultiplier = new(1f, 0f, 1f);
    public ClampedFloatParameter hueShift = new(0f, 0f, 1f);
    public ClampedFloatParameter blurRadius = new(0f, 0f, 1f);
    public ClampedIntParameter blurQuality = new(2, 1, 3);
    public ClampedFloatParameter shakeIntensity = new(0f, 0f, 8f);
    public ClampedFloatParameter shakeFrequency = new(15f, 5f, 30f);
    public ClampedFloatParameter chromaticIntensity = new(0f, 0f, 1f);
    public ClampedFloatParameter grainDensity = new(0f, 0f, 1f);
}

/// <summary>
/// 屏幕特效桥接器（原名 ScreenEffectsVolume，已重命名避免与 VolumeComponent 子类冲突）
/// 负责将 ScreenEffectsManager 的计算结果写入 VolumeProfile 参数
/// 挂载在 Camera 下的 Volume GameObject 上
/// </summary>
[RequireComponent(typeof(Volume))]
public class ScreenEffectsVolumeController : MonoBehaviour
{
    [SerializeField] private ScreenEffectsManager _manager;
    [SerializeField] private VolumeProfile _profile;

    private void LateUpdate()
    {
        if (_manager == null || _profile == null) return;

        var p = _manager.GetFinalParams();

        // 应用到 Volume Component（使用重命名后的 ScreenEffectsVolumeComponent）
        if (_profile.TryGet<ScreenEffectsVolumeComponent>(out var screenEffects))
        {
            screenEffects.vignetteIntensity.value = p.vignetteIntensity;
            screenEffects.vignetteColor.value = p.vignetteColor;
            screenEffects.noiseIntensity.value = p.noiseIntensity;
            screenEffects.saturationMultiplier.value = p.saturationMultiplier;
            screenEffects.hueShift.value = p.hueShift;
            screenEffects.blurRadius.value = p.blurRadius;
            screenEffects.blurQuality.value = p.blurQuality;
            screenEffects.shakeIntensity.value = p.shakeIntensity;
            screenEffects.shakeFrequency.value = p.shakeFrequency;
            screenEffects.chromaticIntensity.value = p.chromaticIntensity;
            screenEffects.grainDensity.value = p.grainDensity;
        }
    }
}
```

**Camera 配置**：

```
Main Camera
└── Volume (GameObject)
    ├── ScreenEffectsVolumeController  ← MonoBehaviour，从 ScreenEffectsManager 读参数
    └── VolumeProfile
        ├── ScreenEffectsVolumeComponent  ← VolumeComponent 子类，持有所有效果参数
        ├── Vignette (URP 内置)
        ├── ChromaticAberration (URP 内置)
        └── FilmGrain (URP 内置)
```

> **注意**：`ScreenEffectsVolumeController` 在 `LateUpdate` 中驱动 `ScreenEffectsVolumeComponent` 的参数，不要将两者混为一个类。

**效果到 Volume Component 映射**：

| ScreenEffectType | Volume Component | 驱动参数 |
|-----------------|-----------------|---------|
| Vignette | Vignette | intensity, color |
| Noise | Noise (自定义 Shader) | intensity, color |
| Saturation | LiftGammaGain (饱和度通过 saturationMultiplier 控制) | - |
| Hue | ColorAdjustments | hueShift |
| Blur | DepthOfField (模糊模式) | focusDistance, aperture |
| UI_Jitter | 自定义 Shader + Camera offset | - |
| ChromaticAberration | ChromaticAberration | intensity |
| FilmGrain | FilmGrain | intensity |
| DamageFlash | Vignette (color=red) | color, intensity |

---

### Phase 4: 系统集成
- [ ] Sanity/Rage 系统集成
- [ ] Health System 集成
- [ ] Weather System 集成
- [ ] Lighting System 集成

---

## Tuning Knobs

| 参数 | 默认值 | 安全范围 | 说明 |
|------|--------|---------|------|
| `MaxEffectLayers` | 20 | 10~50 | 同时存在的最大效果层数 |
| `DefaultFadeInDuration` | 0.2s | 0.1~0.5s | 默认淡入时长 |
| `DefaultFadeOutDuration` | 0.3s | 0.1~0.5s | 默认淡出时长 |
| `MaxVignetteIntensity` | 1.0 | 0.8~1.0 | 暗角强度上限 |
| `MaxNoiseIntensity` | 1.0 | 0.8~1.0 | 噪点强度上限 |
| `MaxUI_JitterIntensity` | 8.0 | 4.0~12.0 | 准星抖动强度上限（原 MaxShakeIntensity，已重命名） |
| `EffectTimeoutDefault` | 10.0s | 5.0~30.0s | 默认效果超时时间 |
| `AdditiveLayerDecay` | 0.5 | 0.3~0.7 | 低优先级效果层叠加衰减系数。<br/>设计意图：0.5 可确保高优先级效果主导视觉体验的同时，低优先级效果仍有可感知的贡献（如天气噪点隐约可见）。**待 UX 测试验证后确认默认值。** |
| `DebugOverlayEnabled` | false | - | 是否启用 Debug 界面 |

**优先级定义补充**：

| 优先级 | 来源系统 | 效果类型 | 说明 |
|--------|---------|---------|------|
| 100 | Health System | DamageFlash | 受伤闪红最高优先级 |
| 80 | Sanity/Rage | SOUL_SPLIT 状态 | 极端心理状态 |
| 70 | Sanity/Rage | FRENZIED/BROKEN | 高压心理状态 |
| 60 | Sanity/Rage | AGITATED/UNEASY | 中压心理状态 |
| 50 | Environment | 爆炸/火灾特效 | 环境事件 |
| 40 | Weather | 暴风雨/大雾 | 天气效果 |
| 30 | Lighting | 极暗区域 | 区域光照 |
| 20 | Dialogue | 对话模糊 | UI 交互 |
| 10 | Generic | 默认 | 最低优先级 |

> **优先级范围说明**：ScreenEffect 优先级范围为 **0-100**，与 Input Blocking 优先级范围（ADR-0015 §UI 层级优先级）保持一致，便于统一管理和调试。

**Vignette 优先级仲裁规则**：

当同一效果类型（Vignette）被多个系统请求时：
1. 获取所有请求 Vignette 的效果层
2. 按 `sourceSystem` 的优先级（见上表）排序
3. 最高优先级的效果层决定 Vignette 的 `vignetteColor` 和 `vignetteIntensity`
4. 其他低优先级层的 `vignetteIntensity` 通过 `AdditiveLayerDecay` 衰减后叠加，但有 `MaxVignetteIntensity` 上限

---

## Validation Criteria

| ID | 验收条件 |
|----|---------|
| VC-1 | 多个系统同时请求效果时，优先级最高的效果正确显示 |
| VC-2 | 效果请求撤销后，下一个优先级的效果正确恢复 |
| VC-3 | 效果强度变化时，平滑过渡无跳变 |
| VC-4 | 性能测试：10 个效果层同时运行时 CPU < 1ms |
| VC-5 | Debug 界面正确显示所有活跃效果和参数 |

---

## Related Decisions

- [ADR-0003: 系统分层架构](./adr-0003-system-layers.md) — Infrastructure Layer 定义
- [ADR-0017: Sanity/Rage 系统](./adr-0017-sanity-rage-meter-architecture.md) — 视觉效果请求
- [ADR-0021: Weather System](./adr-0021-weather-system-architecture.md) — 天气视觉效果
- [ADR-0022: Lighting System](./adr-0022-lighting-system-architecture.md) — 光照视觉效果
- [shared-types.md §22](./shared-types.md#22-游戏时间接口与-world-layer-时间事件) — ScreenEffectRevokeEvent、感知系数叠加规则权威定义
