# ADR-0025: 音频系统 (Audio System) 架构决策

## Status
**Accepted**

## Date
2026-04-12

## Last Updated
2026-04-15 (ADR 评审修复：OnTakedownComplete 补充 TriggerScreenEffectSync 调用，确保屏幕震动/触觉联动)

## Context

### Problem Statement

《断绝：罪恶之源》需要沉浸式音频体验，包括：
1. **环境音**：关卡氛围、天气音效
2. **游戏音效**：战斗、处决、交互、UI
3. **NPC 音效**：对话、警报、死亡
4. **音乐**：动态背景音乐、剧情触发音乐

当前 `immersive-audio-haptics.md` GDD 已定义音效规格，但缺少**系统架构层面**的决策：
- 音频总线路由设计
- 事件驱动音效触发
- 与 Unity 音频系统集成
- 与其他 Presentation Layer 系统（ScreenEffects、DPP）的协作

### Constraints

- **引擎约束**：Unity 6.3 LTS，使用 Unity 内置音频系统（AudioSource/AudioListener）
- **平台约束**：PC & PS5，PS5 需要支持 DualSense 触控板和自适应扳机音效
- **预算约束**：音频内存 ≤ 64MB（详见 ADR-0019 Addressables），作为 ResourceManager OnDemandBudgetMB 的一部分统一管理
- **第三方约束**：不使用 FMOD/Wwise，使用 Unity 内置音频系统

### Requirements

- **必须**：定义 AudioBus 总线路由（Master/SFX/Ambient/Dialogue/UI）
- **必须**：定义 AudioEvent 到 EventBus 的发布/订阅机制
- **必须**：定义 AudioMixerSnapshot 切换接口（用于状态变化）
- **必须**：支持 3D 空间音频定位
- **必须**：遵循 ADR-0003 分层（Audio System 属于 Presentation Layer）
- **必须**：与 Immersive Audio GDD（§12）定义的音效规格保持一致

---

## Decision

### 架构决策

采用**总线混音架构 + 事件驱动音效触发 + 快照状态管理**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      音频系统架构图                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  AudioSource Pool (预分配 32 个 AudioSource)                │   │
│  │  - 3D 音效源（最多 16 个）                                  │   │
│  │  - 2D 音效源（最多 16 个）                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  AudioBus Router (音频总线路由器)                            │   │
│  │                                                              │   │
│  │  [SFX Bus] ─────────┬──→ [Master Bus] ──→ [AudioListener]   │   │
│  │  [Ambient Bus] ─────┼──→ [Master Bus] ──→ [AudioListener]   │   │
│  │  [Dialogue Bus] ────┼──→ [Master Bus] ──→ [AudioListener]   │   │
│  │  [Music Bus] ───────┴──→ [Master Bus] ──→ [AudioListener]   │   │
│  │  [UI Bus] ──────────┴──→ [Master Bus] ──→ [AudioListener]   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  AudioMixer (Unity AudioMixer)                              │   │
│  │  - MasterMixer                                              │   │
│  │  - SFXMixer (动态范围压缩)                                   │   │
│  │  - AmbientMixer (低通滤波)                                  │   │
│  │  - DialogueMixer                                            │   │
│  │  - MusicMixer                                               │   │
│  │                                                              │   │
│  │  AudioMixerSnapshot (状态快照)                              │   │
│  │  - Normal / Combat / Stealth / Death / Menu                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1. AudioBus 定义

```csharp
// AudioBus.cs
/// <summary>
/// 音频总线枚举
/// </summary>
public enum AudioBus
{
    /// <summary>主总线，所有音频混合输出</summary>
    Master,

    /// <summary>游戏音效总线</summary>
    SFX,

    /// <summary>环境音/氛围音总线</summary>
    Ambient,

    /// <summary>对话/旁白总线</summary>
    Dialogue,

    /// <summary>背景音乐总线</summary>
    Music,

    /// <summary>UI 交互音总线</summary>
    UI
}

// AudioBusConfig.cs
/// <summary>
/// 音频总线配置（对应 Immersive Audio GDD §3.1.1）
/// </summary>
[CreateAssetMenu(menuName = "Game/Audio/BusConfig")]
public class AudioBusConfig : ScriptableObject
{
    [Header("总线音量")]
    public float masterVolume = 1.0f;      // -3 dBFS
    public float sfxVolume = 0.8f;        // -6 dBFS
    public float ambientVolume = 0.7f;     // -9 dBFS
    public float dialogueVolume = 0.9f;    // -6 dBFS
    public float musicVolume = 0.75f;      // -12 dBFS
    public float uiVolume = 0.6f;          // -12 dBFS

    [Header("AudioMixerGroup 引用（运行时直接引用，避免 FindMatchingGroups 查找开销）")]
    public UnityEngine.Audio.AudioMixerGroup masterMixerGroup;
    public UnityEngine.Audio.AudioMixerGroup sfxMixerGroup;
    public UnityEngine.Audio.AudioMixerGroup ambientMixerGroup;
    public UnityEngine.Audio.AudioMixerGroup dialogueMixerGroup;
    public UnityEngine.Audio.AudioMixerGroup musicMixerGroup;
    public UnityEngine.Audio.AudioMixerGroup uiMixerGroup;

    /// <summary>
    /// **音量值说明**：所有音量字段（masterVolume、sfxVolume 等）均为**线性值**（0.0-1.0）。
    /// 代码中使用 LinearToDb() 转换为分贝值：
    /// - 1.0f = 0 dBFS（满音量）
    /// - 0.5f ≈ -6 dBFS
    /// - 0.25f ≈ -12 dBFS
    ///
    /// 配置文件中标注的 dBFS 值（如 -3 dBFS）仅为参考，
    /// 表示该线性值对应的分贝数，便于策划理解音量感。
    /// </summary>

    [Header("动态范围压缩 (SFX Bus)")]
    public bool dynamicRangeEnabled = true;
    public float compressionRatio = 4.0f;
    public float attackTime_ms = 10f;
    public float releaseTime_ms = 100f;
    public float threshold_dB = -20f;

    [Header("低通滤波 (Ambient Bus)")]
    public float ambientLowpassFrequency = 2000f; // Hz
}
```

### 2. AudioSource Pool 管理

> **ADR 评审修复 2026-04-15**：AudioSourcePool 大小已明确写入 ADR（3D:16 + 2D:16 = 32 总源）。

```csharp
// AudioSourcePool.cs
/// <summary>
/// AudioSource 对象池（预分配避免运行时分配）
/// 注意：需要 [Serializable] 才能作为 AudioManager 的 SerializeField 被 Inspector 序列化
///
/// **池大小**（ADR 评审修复 2026-04-15）：
/// - MAX_3D_SOURCES = 16（3D 空间音效源）
/// - MAX_2D_SOURCES = 16（2D 非空间音效源）
/// - 总计 32 个 AudioSource
/// </summary>
[Serializable]
public class AudioSourcePool
{
    private const int MAX_3D_SOURCES = 16;
    private const int MAX_2D_SOURCES = 16;

    private Queue<AudioSource> _pool3D = new();
    private Queue<AudioSource> _pool2D = new();
    private AudioMixerGroup _sfxMixerGroup;
    private AudioMixerGroup _ambientMixerGroup;
    private Transform _poolRoot;  // 对象池根节点

    /// <summary>
    /// 初始化对象池
    /// </summary>
    /// <param name="busConfig">音频总线配置（包含预配置的 AudioMixerGroup 引用）</param>
    /// <param name="poolRoot">对象池根节点 Transform</param>
    public void Initialize(AudioBusConfig busConfig, Transform poolRoot)
    {
        _poolRoot = poolRoot;
        // 使用 AudioBusConfig 中预配置的 AudioMixerGroup 引用，避免运行时 FindMatchingGroups 查找开销
        _sfxMixerGroup = busConfig.sfxMixerGroup;
        _ambientMixerGroup = busConfig.ambientMixerGroup;

        // 预分配 3D AudioSource
        for (int i = 0; i < MAX_3D_SOURCES; i++)
        {
            var source = CreateSource("3D_" + i, true, _sfxMixerGroup);
            _pool3D.Enqueue(source);
        }

        // 预分配 2D AudioSource
        for (int i = 0; i < MAX_2D_SOURCES; i++)
        {
            var source = CreateSource("2D_" + i, false, _sfxMixerGroup);
            _pool2D.Enqueue(source);
        }
    }

    private AudioSource CreateSource(string name, bool spatial, AudioMixerGroup group)
    {
        var go = new GameObject(name);
        go.transform.SetParent(_poolRoot);
        var source = go.AddComponent<AudioSource>();
        source.playOnAwake = false;
        source.spatialBlend = spatial ? 1f : 0f;
        source.maxDistance = spatial ? 50f : float.MaxValue;
        source.outputAudioMixerGroup = group;
        return source;
    }

    /// <summary>
    /// 获取 3D AudioSource
    /// </summary>
    public AudioSource Rent3D()
    {
        if (_pool3D.TryDequeue(out var source))
            return source;

        // 池耗尽，返回临时 AudioSource（应该不发生）
        Debug.LogWarning("[AudioSourcePool] 3D pool exhausted");
        return CreateSource("Temp_3D", true, _sfxMixerGroup);
    }

    /// <summary>
    /// 归还 3D AudioSource
    /// </summary>
    public void Return3D(AudioSource source)
    {
        source.Stop();
        source.clip = null;
        if (_pool3D.Count < MAX_3D_SOURCES)
            _pool3D.Enqueue(source);
        else
            Object.Destroy(source.gameObject); // 池已满（通常是超出预分配的临时实例），销毁避免泄漏
    }

    /// <summary>
    /// 获取 2D AudioSource
    /// </summary>
    public AudioSource Rent2D()
    {
        if (_pool2D.TryDequeue(out var source))
            return source;

        Debug.LogWarning("[AudioSourcePool] 2D pool exhausted");
        return CreateSource("Temp_2D", false, _sfxMixerGroup);
    }

    /// <summary>
    /// 归还 2D AudioSource
    /// </summary>
    public void Return2D(AudioSource source)
    {
        source.Stop();
        source.clip = null;
        if (_pool2D.Count < MAX_2D_SOURCES)
            _pool2D.Enqueue(source);
        else
            Object.Destroy(source.gameObject); // 池已满，销毁避免泄漏
    }
}
```

### 2.5. AudioClipCacheManager 音效缓存管理

```csharp
// AudioClipCacheManager.cs
/// <summary>
/// 音效资源缓存管理器
/// 使用 LRU (Least Recently Used) 策略管理已加载的 AudioClip
///
/// **序列化说明**：
/// 此类标记了 [Serializable]，使 AudioManager 可以通过 Inspector 的 [SerializeField]
/// 显示并配置它。普通 C# 类如果没有 [Serializable]，Inspector 序列化的字段永远为 null。
/// </summary>
[Serializable]
public class AudioClipCacheManager
{
    /// <summary>
    /// 缓存条目
    /// </summary>
    private class CacheEntry
    {
        public AudioClip Clip;
        public float LastAccessTime;  // 使用时间戳替代帧号（协程中 Time.time 可靠）
        public int UseCount;  // 使用计数（用于热点音效优化）
    }

    private Dictionary<string, CacheEntry> _cache = new();

    /// <summary>
    /// 缓存配置
    /// P1 修复：原设计为 AudioClipCacheManager 的嵌套类，但嵌套类无法继承 ScriptableObject
    /// （Unity 无法为嵌套类创建资产，[CreateAssetMenu] 对嵌套类无效）。
    /// 已提取为独立顶级类 AudioClipCacheConfig，见文件末尾定义。
    /// </summary>
    private AudioClipCacheConfig _config;
    private float _lastCleanupTime;

    /// <summary>
    /// 初始化缓存管理器
    /// </summary>
    public void Initialize(AudioClipCacheConfig config)
    {
        _config = config;
        _lastCleanupTime = Time.time;
    }

    /// <summary>
    /// 获取缓存的 AudioClip（同步）
    /// </summary>
    /// <returns>缓存的 clip，如果未缓存则返回 null</returns>
    public AudioClip GetCachedClip(string address)
    {
        if (_cache.TryGetValue(address, out var entry))
        {
            entry.LastAccessTime = Time.time;
            entry.UseCount++;
            return entry.Clip;
        }
        return null;
    }

    /// <summary>
    /// 添加到缓存
    /// </summary>
    /// <remarks>
    /// **缓存策略**：
    /// 1. 检查缓存是否已满
    /// 2. 如果已满，淘汰最低优先级条目（使用计数最少 + 最近未访问）
    /// 3. 大文件音效（> maxClipSizeKB）不缓存
    /// </remarks>
    public void AddToCache(string address, AudioClip clip)
    {
        if (clip == null) return;

        // 检查文件大小
        // 注意：不能使用 clip.length * clip.frequency * clip.channels 计算，这是未压缩大小。
        // OGG/MP3 等压缩格式实际内存占用差异大。应使用 Profiler.GetRuntimeMemorySizeLong 获取实际运行时内存。
        int clipSizeKB = (int)(Profiler.GetRuntimeMemorySizeLong(clip) / 1024);
        if (clipSizeKB > _config.maxClipSizeKB)
        {
            Debug.Log($"[AudioClipCache] Skipping cache for large clip: {address} ({clipSizeKB}KB)");
            return;
        }

        // 如果已存在，更新访问信息
        if (_cache.TryGetValue(address, out var existing))
        {
            existing.LastAccessTime = Time.time;
            existing.UseCount++;
            return;
        }

        // 缓存已满，执行 LRU 淘汰
        if (_cache.Count >= _config.maxCacheCount)
        {
            EvictLRU();
        }

        _cache[address] = new CacheEntry
        {
            Clip = clip,
            LastAccessTime = Time.time,
            UseCount = 1
        };
    }

    /// <summary>
    /// LRU 淘汰：移除最低优先级条目
    /// 优先级 = UseCount * 权重 + (当前时间 - LastAccessTime) * 时间衰减
    /// </summary>
    private void EvictLRU()
    {
        string toEvict = null;
        float lowestPriority = float.MaxValue;
        float currentTime = Time.time;

        foreach (var kvp in _cache)
        {
            // 热点音效（使用次数 >= minUseCountForPriority）保护不淘汰
            if (kvp.Value.UseCount >= _config.minUseCountForPriority)
                continue;

            float priority = kvp.Value.UseCount * 1000f + (currentTime - kvp.Value.LastAccessTime);
            if (priority < lowestPriority)
            {
                lowestPriority = priority;
                toEvict = kvp.Key;
            }
        }

        // 如果所有条目都是热点，淘汰使用次数最少的
        if (toEvict == null)
        {
            foreach (var kvp in _cache)
            {
                if (kvp.Value.UseCount < lowestPriority)
                {
                    lowestPriority = kvp.Value.UseCount;
                    toEvict = kvp.Key;
                }
            }
        }

        if (toEvict != null)
        {
            _cache.Remove(toEvict);
            Debug.Log($"[AudioClipCache] Evicted: {toEvict}");
        }
    }

    /// <summary>
    /// 预加载音效到缓存（协程版本）
    ///
    /// **调用方式**：此方法为 IEnumerator，需通过 MonoBehaviour.StartCoroutine 调用：
    /// ```csharp
    /// // 在 AudioManager 中调用：
    /// StartCoroutine(_cacheManager.PreloadCoroutine(addresses));
    /// ```
    /// **原设计问题**：原版 async Task 方法在 Unity 中存在异常被吞掉（未在主线程处理）
    /// 以及无法使用部分 Unity API 的风险，改为 IEnumerator 确保所有操作在主线程执行。
    /// </summary>
    public IEnumerator PreloadCoroutine(string[] addresses)
    {
        foreach (var address in addresses)
        {
            if (_cache.ContainsKey(address)) continue;

            var op = Addressables.LoadAssetAsync<AudioClip>(address);
            yield return op;

            if (op.Status == AsyncOperationStatus.Succeeded && op.Result != null)
            {
                AddToCache(address, op.Result);
            }
        }
    }

    /// <summary>
    /// 清空缓存
    /// </summary>
    public void Clear()
    {
        _cache.Clear();
    }

    /// <summary>
    /// 获取缓存统计信息
    /// </summary>
    public string GetCacheStats()
    {
        return $"[AudioClipCache] Count: {_cache.Count}/{_config.maxCacheCount}";
    }
}

// AudioClipCacheConfig.cs
/// <summary>
/// 音效资源缓存配置（P1 修复：从 AudioClipCacheManager 嵌套类提取为独立顶级类）
/// 嵌套类继承 ScriptableObject 在 Unity 中不受支持（[CreateAssetMenu] 对嵌套类无效）
/// </summary>
[CreateAssetMenu(menuName = "Game/Audio/CacheConfig")]
public class AudioClipCacheConfig : ScriptableObject
{
    [Tooltip("最大缓存音效数量")]
    public int maxCacheCount = 64;

    [Tooltip("单音效最大内存（KB），超过此大小的音效不缓存")]
    public int maxClipSizeKB = 512;

    [Tooltip("最小使用次数，达到此次数的音效优先级保留")]
    public int minUseCountForPriority = 3;
}
```

### 3. 音效播放接口

```csharp
// AudioEvents.cs
/// <summary>
/// 音效分类枚举（用于音效触发和屏幕特效联动）
/// </summary>
public enum SFXCategory
{
    /// <summary>处决音效</summary>
    StealthKill,
    /// <summary>环境处决音效</summary>
    EnvironmentKill,
    /// <summary>爆炸音效</summary>
    Explosion,
    /// <summary>小型爆炸音效</summary>
    ExplosionSmall,
    /// <summary>脚步声</summary>
    Footstep,
    /// <summary>UI 交互音效</summary>
    UI,
    /// <summary>NPC 行为音效</summary>
    NPC,
    /// <summary>环境物件音效</summary>
    Environment,
    /// <summary>武器/战斗音效</summary>
    Weapon,
    /// <summary>其他</summary>
    Other
}

/// <summary>
/// 音效播放事件（发送到 EventBus）
/// </summary>
public struct SFXPlayRequest
{
    /// <summary>音效资源地址（Addressables）</summary>
    public string address;

    /// <summary>播放位置（3D 音效）</summary>
    public Vector3 position;

    /// <summary>音量缩放 [0-1]</summary>
    public float volumeScale;

    /// <summary>音调偏移 [-0.5, +0.5]</summary>
    public float pitchShift;

    /// <summary>是否循环</summary>
    public bool loop;

    /// <summary>音效分类（用于屏幕特效联动）</summary>
    public SFXCategory category;
}

/// <summary>
/// 音效停止事件
///
/// **多实例行为说明**：
/// 当前实现中，`SFXStopRequest(address)` 会停止该地址的**所有**播放实例。
/// 这是**有意为之的设计**，因为大多数音效（环境音、UI 音效等）按地址管理即可。
///
/// **使用场景**：
/// - 停止特定类型的循环音效（如 "ambient/warehouse"）
/// - 停止所有脚步声（按地址 "sfx/footstep"）
///
/// **限制场景**：
/// 如果需要精确控制单个实例（如停止某个特定位置的脚步声），当前设计无法区分。
/// 扩展方案：使用 instanceId 追踪每个播放实例：
/// ```csharp
/// Dictionary<string, List<(AudioSource source, int instanceId)>> _playingSources
/// ```
/// 此设计变更需同步修改 Return3D/Return2D 的实例追踪逻辑。
/// </summary>
public struct SFXStopRequest
{
    /// <summary>
    /// 要停止的音效地址（Addressables 地址）
    /// 所有使用此地址播放的音效实例都将被停止
    /// </summary>
    public string address;
}

/// <summary>
/// 停止所有音效事件
/// 用于游戏暂停、死亡等需要立即停止所有音效的场景
/// </summary>
public struct SFXStopAllRequest { }

/// <summary>
/// 音乐切换事件
/// </summary>
public struct MusicTransitionRequest
{
    public string musicAddress;      // 新音乐地址
    public float fadeDuration;        // 淡入淡出时长
    public MusicTransitionType type; // 切换类型
}

public enum MusicTransitionType
{
    Crossfade,   // 交叉淡入淡出
    Cut,         // 立即切换
    SnapshotFade // 快照切换（用于状态变化）
}
```

### 4. AudioManager 核心类

```csharp
// AudioManager.cs
// 必要 using：
// using System.Collections.Generic;
// using System.Threading;          // Interlocked.Increment 需要
// using UnityEngine.AddressableAssets;
// using UnityEngine.ResourceManagement.AsyncOperations;

/// <summary>
/// 音频系统管理器
/// </summary>
public class AudioManager : MonoBehaviour
{
    public static AudioManager Instance { get; private set; }

    [SerializeField] private AudioMixer _audioMixer;
    [SerializeField] private AudioSourcePool _sourcePool;
    [SerializeField] private AudioBusConfig _busConfig;
    [SerializeField] private AudioTuningSO _tuning;
    [SerializeField] private AudioClipCacheManager _cacheManager;
    // P1 修复：CacheConfig 已提取为独立顶级类 AudioClipCacheConfig
    [SerializeField] private AudioClipCacheConfig _cacheConfig;
    [SerializeField] private Transform _poolRoot;  // 对象池根节点

    /// <summary>
    /// 正在播放的音效（address → AudioSource 列表）
    ///
    /// **设计说明**：
    /// - key 是音效地址，同一地址的多个播放实例共享同一个 key
    /// - SFXStopRequest(address) 会停止该地址的**所有**播放实例
    /// - 此设计简化了实例追踪，按 address 管理足够满足大多数场景需求
    ///   如果将来需要精确控制单个实例，可扩展为：
    ///   `Dictionary<string, List<(AudioSource source, int instanceId)>>`
    /// </summary>
    private Dictionary<string, List<AudioSource>> _playingSources = new();

    /// <summary>
    /// 专用音乐播放 AudioSource（Music Bus）
    /// </summary>
    private AudioSource _musicSource;

    /// <summary>
    /// 下一个屏幕特效请求的唯一 ID（用于 requesterId，避免 Time.time 导致的 ID 不稳定）
    /// 使用 static long + Interlocked.Increment 避免跨系统 ID 冲突
    /// 注意：静态字段在所有 AudioManager 实例间共享，确保 ID 全局唯一
    /// </summary>
    private static long _nextEffectId = 0;

    private void Awake()
    {
        if (Instance != null)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;

        // 创建对象池根节点
        if (_poolRoot == null)
        {
            var poolGo = new GameObject("[AudioSourcePool]");
            _poolRoot = poolGo.transform;
            DontDestroyOnLoad(poolGo);
        }

        _sourcePool.Initialize(_busConfig, _poolRoot);
        ApplyBusConfig();
        InitializeMusicSource();

        // 初始化缓存管理器
        if (_cacheManager != null && _cacheConfig != null)
        {
            _cacheManager.Initialize(_cacheConfig);
        }

        // 订阅事件
        EventBus.Instance.Subscribe<SFXPlayRequest>(OnSFXPlayRequest);
        EventBus.Instance.Subscribe<SFXStopRequest>(OnSFXStopRequest);
        EventBus.Instance.Subscribe<SFXStopAllRequest>(OnSFXStopAllRequest);
        EventBus.Instance.Subscribe<MusicTransitionRequest>(OnMusicTransitionRequest);
        EventBus.Instance.Subscribe<TakedownAnimationCompleteEvent>(OnTakedownComplete);  // 处决动画完成 → 播放处决音效
        EventBus.Instance.Subscribe<CombatStateChangedEvent>(OnCombatStateChanged);  // 战斗状态变化 → 自动切换战斗音乐/氛围音
        EventBus.Instance.Subscribe<PauseMenuOpenedEvent>(OnPauseMenuOpened);  // 菜单打开 → 音乐淡出
        EventBus.Instance.Subscribe<PauseMenuClosedEvent>(OnPauseMenuClosed);   // 菜单关闭 → 音乐淡入
    }

    private void OnDestroy()
    {
        if (Instance == this)
        {
            EventBus.Instance.Unsubscribe<SFXPlayRequest>(OnSFXPlayRequest);
            EventBus.Instance.Unsubscribe<SFXStopRequest>(OnSFXStopRequest);
            EventBus.Instance.Unsubscribe<SFXStopAllRequest>(OnSFXStopAllRequest);
            EventBus.Instance.Unsubscribe<MusicTransitionRequest>(OnMusicTransitionRequest);
            EventBus.Instance.Unsubscribe<TakedownAnimationCompleteEvent>(OnTakedownComplete);
            EventBus.Instance.Unsubscribe<CombatStateChangedEvent>(OnCombatStateChanged);
            EventBus.Instance.Unsubscribe<PauseMenuOpenedEvent>(OnPauseMenuOpened);
            EventBus.Instance.Unsubscribe<PauseMenuClosedEvent>(OnPauseMenuClosed);
            Instance = null;
        }
    }

    /// <summary>
    /// 播放 3D 音效
    /// </summary>
    public void Play3D(SFXPlayRequest request) => StartCoroutine(PlayClipCoroutine(request, is3D: true));

    /// <summary>
    /// 播放 2D 音效
    /// </summary>
    public void Play2D(SFXPlayRequest request) => StartCoroutine(PlayClipCoroutine(request, is3D: false));

    /// <summary>
    /// 通用音效播放协程（Play3D / Play2D 的共同实现）
    /// is3D=true：从 3D 对象池租用，设置空间位置；is3D=false：从 2D 对象池租用
    /// </summary>
    private IEnumerator PlayClipCoroutine(SFXPlayRequest request, bool is3D)
    {
        AudioClip clip = null;

        // 1. 尝试从缓存获取
        if (_cacheManager != null)
        {
            clip = _cacheManager.GetCachedClip(request.address);
        }

        // 2. 缓存未命中，异步加载
        if (clip == null)
        {
            var op = Addressables.LoadAssetAsync<AudioClip>(request.address);
            yield return op;

            if (op.Result == null)
            {
                Debug.LogWarning($"[AudioManager] Failed to load clip: {request.address}");
                yield break;
            }

            clip = op.Result;
            _cacheManager?.AddToCache(request.address, clip);
        }

        // 3. 租用 AudioSource 并配置
        var source = is3D ? _sourcePool.Rent3D() : _sourcePool.Rent2D();
        source.clip = clip;
        if (is3D) source.transform.position = request.position;
        source.volume = request.volumeScale;
        source.pitch = 1f + request.pitchShift;
        source.loop = request.loop;
        source.Play();

        // 记录到正在播放列表（address → 多实例 source 列表）
        if (!_playingSources.ContainsKey(request.address))
            _playingSources[request.address] = new List<AudioSource>();
        _playingSources[request.address].Add(source);

        // 4. 触发屏幕特效 / Haptics 联动（UI 音效跳过）
        if (is3D || request.category != SFXCategory.UI)
        {
            TriggerScreenEffectSync(request.category);
        }

        // 5. 非循环音效：播放完毕后归还到对象池
        if (!request.loop)
        {
            yield return new WaitForSeconds(clip.length);

            if (source != null && source.isActiveAndEnabled)
            {
                if (is3D) _sourcePool.Return3D(source);
                else _sourcePool.Return2D(source);
            }

            if (_playingSources.TryGetValue(request.address, out var list))
            {
                list.Remove(source);
                if (list.Count == 0)
                    _playingSources.Remove(request.address);
            }
        }
    }

    /// <summary>
    /// 音效与屏幕特效联动
    /// </summary>
    private void TriggerScreenEffectSync(SFXCategory category)
    {
        // 根据音效分类触发对应的物理相机震动（CameraShakeRequestEvent）和触觉反馈（HapticRequest）
        // 注意：爆炸/处决的物理震动应使用 CameraShakeRequestEvent，而非 ScreenEffectRequestEvent
        // ScreenEffectType.Jitter 仅用于 UI 准星抖动，见 ADR-0023 §6 职责边界说明
        switch (category)
        {
            case SFXCategory.Explosion:
            case SFXCategory.ExplosionSmall:
                EventBus.Instance.Publish(new CameraShakeRequestEvent
                {
                    shake_type = CameraShakeType.Explosion,
                    intensity = category == SFXCategory.Explosion ? 0.8f : 0.5f,
                    duration = 0.4f,
                    mode = ShakeMode.Explosive
                });
                // 爆炸触发 DualSense 强烈震动
                EventBus.Instance.Publish(new HapticRequest
                {
                    type = HapticType.Explosion,
                    intensity = category == SFXCategory.Explosion ? 1.0f : 0.6f
                });
                break;

            case SFXCategory.StealthKill:
            case SFXCategory.EnvironmentKill:
                EventBus.Instance.Publish(new CameraShakeRequestEvent
                {
                    shake_type = CameraShakeType.Execute,
                    intensity = 0.3f,
                    duration = 0.25f,
                    mode = ShakeMode.Subtle
                });
                // 处决触发 DualSense 自适应扳机最大阻力（ADR-0010 §9）
                EventBus.Instance.Publish(new HapticRequest
                {
                    type = HapticType.Execution,
                    intensity = 1.0f
                });
                break;

            case SFXCategory.Weapon:
                // 武器音效不直接触发屏幕震动，由调用方根据具体攻击类型自行决定是否触发。
                // 例如：重型武器攻击可触发轻微震动，轻型武器则不触发。
                // 调用方应在发布 SFXPlayRequest 前发布对应的 ScreenEffectRequestEvent。
                // 武器音效触发 DualSense 扳机震动（ADR-0010 §9）
                EventBus.Instance.Publish(new HapticRequest
                {
                    type = HapticType.Combat,
                    intensity = 0.7f
                });
                break;
        }
    }

    /// <summary>
    /// 快照缓存（避免每帧 FindSnapshot 查找开销）
    /// 初始化时预加载所有已知快照，运行时直接使用缓存引用
    /// </summary>
    private Dictionary<string, AudioMixerSnapshot> _snapshotCache = new();

    /// <summary>
    /// 初始化快照缓存
    /// 在 AudioManager 初始化时调用，预加载所有游戏状态快照
    /// </summary>
    public void InitializeSnapshotCache(string[] snapshotNames)
    {
        foreach (var name in snapshotNames)
        {
            var snapshot = _audioMixer.FindSnapshot(name);
            if (snapshot != null)
            {
                _snapshotCache[name] = snapshot;
            }
            else
            {
                Debug.LogWarning($"[AudioManager] Snapshot '{name}' not found in AudioMixer");
            }
        }
    }

    /// <summary>
    /// 切换音频快照（用于状态变化）
    /// 使用缓存的快照引用，避免每帧查找开销
    /// </summary>
    public void TransitionToSnapshot(string snapshotName, float transitionTime)
    {
        if (_snapshotCache.TryGetValue(snapshotName, out var snapshot))
        {
            snapshot.TransitionTo(transitionTime);
        }
        else
        {
            // 快照未缓存，尝试运行时查找（并缓存结果）
            var found = _audioMixer.FindSnapshot(snapshotName);
            if (found != null)
            {
                _snapshotCache[snapshotName] = found;
                found.TransitionTo(transitionTime);
            }
            else
            {
                Debug.LogWarning($"[AudioManager] Snapshot '{snapshotName}' not found in AudioMixer");
            }
        }
    }

    private void ApplyBusConfig()
    {
        _audioMixer.SetFloat("MasterVolume", LinearToDb(_busConfig.masterVolume));
        _audioMixer.SetFloat("SFXVolume", LinearToDb(_busConfig.sfxVolume));
        _audioMixer.SetFloat("AmbientVolume", LinearToDb(_busConfig.ambientVolume));
        _audioMixer.SetFloat("DialogueVolume", LinearToDb(_busConfig.dialogueVolume));
        _audioMixer.SetFloat("MusicVolume", LinearToDb(_busConfig.musicVolume));
        _audioMixer.SetFloat("UIVolume", LinearToDb(_busConfig.uiVolume));
    }

    /// <summary>
    /// 初始化专用音乐播放 AudioSource
    /// </summary>
    private void InitializeMusicSource()
    {
        var musicGo = new GameObject("[MusicSource]");
        musicGo.transform.SetParent(_poolRoot);
        _musicSource = musicGo.AddComponent<AudioSource>();
        _musicSource.playOnAwake = false;
        _musicSource.spatialBlend = 0f;  // 2D 音乐
        _musicSource.loop = true;
        // 使用 AudioBusConfig 中预配置的 AudioMixerGroup 引用
        _musicSource.outputAudioMixerGroup = _busConfig.musicMixerGroup;
    }

    private float LinearToDb(float linear) => linear > 0 ? 20f * Mathf.Log10(linear) : -80f;

    /// <summary>
    /// 事件处理：音效播放请求
    /// </summary>
    /// <remarks>
    /// **SFXStopRequest 多实例行为说明**：
    /// 当前实现中，SFXStopRequest(address) 会停止该地址的**所有**播放实例。
    /// 这是**有意为之的设计**，因为大多数音效（环境音、UI 音效等）按地址管理即可。
    ///
    /// 若需要精确控制单个实例（如停止某个特定位置的脚步声），应使用 instanceId 追踪：
    /// ```csharp
    /// Dictionary<string, List<(AudioSource source, int instanceId)>> _playingSources
    /// ```
    /// 此设计变更需同步修改 Return3D/Return2D 的实例追踪逻辑。
    /// </remarks>
    private void OnSFXPlayRequest(SFXPlayRequest request)
    {
        if (request.position != default && request.position != Vector3.zero)
        {
            Play3D(request);
        }
        else
        {
            Play2D(request);
        }
    }

    private void OnSFXStopRequest(SFXStopRequest request)
    {
        // 停止所有匹配地址的 AudioSource（支持多实例）
        // **防御性清理**：检查 AudioSource 是否有效，避免因异常终止导致的列表泄漏
        if (_playingSources.TryGetValue(request.address, out var sourcesToStop))
        {
            foreach (var source in sourcesToStop)
            {
                if (source != null && source.isActiveAndEnabled)
                {
                    source.Stop();
                }
            }
            _playingSources.Remove(request.address);
        }
    }

    private void OnSFXStopAllRequest(SFXStopAllRequest request)
    {
        // **防御性清理**：遍历时检查 AudioSource 有效性
        foreach (var sources in _playingSources.Values.ToList())  // ToList() 避免迭代中修改
        {
            foreach (var source in sources)
            {
                if (source != null && source.isActiveAndEnabled)
                {
                    source.Stop();
                }
            }
        }
        _playingSources.Clear();
    }

    private void OnMusicTransitionRequest(MusicTransitionRequest request)
    {
        // 音乐切换逻辑
        // 根据 transition.type 执行不同切换方式
        switch (request.type)
        {
            case MusicTransitionType.Crossfade:
                StartCoroutine(CrossfadeMusicCoroutine(request));
                break;
            case MusicTransitionType.Cut:
                CutToMusic(request.musicAddress);
                break;
            case MusicTransitionType.SnapshotFade:
                // 快照切换由 AudioSnapshotController 处理
                break;
        }
    }

    /// <summary>
    /// 处决动画完成事件处理：播放对应的处决音效
    /// </summary>
    /// <remarks>
    /// TakedownAnimationCompleteEvent 由 AnimationEventBridge 发布（ADR-0024 §9.6）。
    /// AudioManager 订阅此事件以在处决动画完成时播放音效/触觉反馈。
    ///
    /// **事件流转**：AnimationEventBridge.OnStealthKillHit → TakedownAnimationCompleteEvent →
    /// GrittyTakedowns → SFXPlayRequest(StealthKill) → AudioManager（此路径为备用，由 AudioManager 直接订阅可确保音效不丢失）
    /// </remarks>
    private void OnTakedownComplete(TakedownAnimationCompleteEvent evt)
    {
        // 根据 TakedownType 获取对应的音效地址
        string sfxAddress = evt.TakedownType switch
        {
            InteractionType.StealthKill => "audio/sfx/combat/stealth_kill",
            InteractionType.EnvironmentKill => "audio/sfx/combat/environment_kill",
            InteractionType.TieUp => "audio/sfx/combat/tie_up",
            _ => "audio/sfx/combat/takedown_default"
        };

        // 播放 2D 处决音效
        Play2D(new SFXPlayRequest { address = sfxAddress });

        // 触发屏幕音效同步（压低其他音频）
        SFXCategory category = evt.TakedownType switch
        {
            InteractionType.StealthKill => SFXCategory.StealthKill,
            InteractionType.EnvironmentKill => SFXCategory.EnvironmentKill,
            _ => SFXCategory.Default
        };
        TriggerScreenEffectSync(category);
    }

    /// <summary>
    /// 战斗状态变化事件处理：自动切换战斗/非战斗音乐和氛围音
    /// 事件来源：NPC AI System（ADR-0004），通过 CombatStateChangedEvent 发布
    /// </summary>
    private void OnCombatStateChanged(CombatStateChangedEvent evt)
    {
        if (evt.IsInCombat)
        {
            // 进入战斗：切换到战斗音乐，氛围音切换到战斗氛围
            StartCoroutine(CrossfadeMusicCoroutine(new MusicTransitionRequest
            {
                musicAddress = "audio/music/combat/combat_loop",
                type = MusicTransitionType.Crossfade,
                fadeOutDuration = 0.5f,
                fadeInDuration = 0.3f
            }));
        }
        else
        {
            // 退出战斗：恢复探索音乐
            StartCoroutine(CrossfadeMusicCoroutine(new MusicTransitionRequest
            {
                musicAddress = "audio/music/exploration/exploration_loop",
                type = MusicTransitionType.Crossfade,
                fadeOutDuration = 0.5f,
                fadeInDuration = 0.3f
            }));
        }
    }

        /// <summary>
    /// 菜单打开事件处理：音乐淡出
    /// 事件来源：UI System（ADR-0015），通过 PauseMenuOpenedEvent 发布
    /// </summary>
    private void OnPauseMenuOpened(PauseMenuOpenedEvent evt)
    {
        StartCoroutine(MusicFadeOutCoroutine(0.3f));
    }

    /// <summary>
    /// 菜单关闭事件处理：音乐淡入
    /// 事件来源：UI System（ADR-0015），通过 PauseMenuClosedEvent 发布
    /// </summary>
    private void OnPauseMenuClosed(PauseMenuClosedEvent evt)
    {
        StartCoroutine(MusicFadeInCoroutine(0.3f));
    }

    private IEnumerator MusicFadeOutCoroutine(float duration)
    {
        if (_musicSource == null) yield break;
        float startVolume = _musicSource.volume;
        float elapsed = 0f;
        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            _musicSource.volume = Mathf.Lerp(startVolume, 0f, elapsed / duration);
            yield return null;
        }
        _musicSource.volume = 0f;
    }

    private IEnumerator MusicFadeInCoroutine(float duration)
    {
        if (_musicSource == null) yield break;
        float startVolume = _musicSource.volume;
        _musicSource.volume = 0f;
        // 重新播放之前淡出的音乐
        if (!_musicSource.isPlaying) _musicSource.Play();
        float elapsed = 0f;
        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            _musicSource.volume = Mathf.Lerp(0f, startVolume, elapsed / duration);
            yield return null;
        }
        _musicSource.volume = startVolume;
    }

    /// <summary>
    /// 正在进行的交叉淡入淡出协程（用于中断时清理）
    /// </summary>
    private Coroutine _crossfadeCoroutine;

    private IEnumerator CrossfadeMusicCoroutine(MusicTransitionRequest request)
    {
        // 如果有正在进行的交叉淡入淡出，先停止它
        if (_crossfadeCoroutine != null)
        {
            StopCoroutine(_crossfadeCoroutine);
            _crossfadeCoroutine = null;
        }

        var previousSource = _musicSource;
        GameObject newSourceGameObject = null;
        AudioSource newSource = null;

        try
        {
            // Step 1: 异步加载新音乐
            var op = Addressables.LoadAssetAsync<AudioClip>(request.musicAddress);
            yield return op;

            // **异常保护**：加载失败时确保旧音乐继续播放
            if (op.Result == null)
            {
                Debug.LogWarning($"[AudioManager] Failed to load music: {request.musicAddress}");
                // 恢复旧音乐音量（如果 previousSource 仍有效）
                if (previousSource != null && previousSource.isActiveAndEnabled)
                {
                    previousSource.volume = _busConfig.musicVolume;
                }
                _crossfadeCoroutine = null;
                yield break;
            }

            // Step 2: 创建新的音乐 AudioSource（用于淡入）
            newSourceGameObject = new GameObject("[MusicSource_New]");
            newSourceGameObject.transform.SetParent(_poolRoot);
            newSource = newSourceGameObject.AddComponent<AudioSource>();
            newSource.clip = op.Result;
            newSource.volume = 0f;
            newSource.spatialBlend = 0f;
            newSource.loop = true;
            newSource.outputAudioMixerGroup = _busConfig.musicMixerGroup;
            newSource.Play();

            // Step 3: 交叉淡入淡出
            float elapsed = 0f;
            while (elapsed < request.fadeDuration)
            {
                elapsed += Time.deltaTime;
                float t = elapsed / request.fadeDuration;

                // **防御性检查**：确保 AudioSource 在淡入过程中仍然有效
                if (previousSource != null && previousSource.isActiveAndEnabled)
                    previousSource.volume = _busConfig.musicVolume * (1f - t);
                if (newSource != null && newSource.isActiveAndEnabled)
                    newSource.volume = _busConfig.musicVolume * t;

                yield return null;
            }

            // Step 4: 切换完成，销毁旧源
            if (previousSource != null && previousSource.isActiveAndEnabled)
            {
                previousSource.Stop();
                if (previousSource.gameObject != null)
                    Destroy(previousSource.gameObject);
            }

            _musicSource = newSource;
        }
        finally
        {
            // 协程结束时清理：
            // - 如果在加载阶段失败（yield break 之前），销毁可能已创建的新 GameObject
            // - 如果在淡入期间发生异常，确保引用被清理
            _crossfadeCoroutine = null;
            if (newSourceGameObject != null && newSource != null && newSource != _musicSource)
            {
                Destroy(newSourceGameObject);
            }
        }
    }

    private void CutToMusic(string musicAddress)
    {
        // 记录当前音乐源用于淡出后销毁
        var previousSource = _musicSource;

        // 异步加载并立即播放新音乐
        StartCoroutine(CutToMusicCoroutine(musicAddress, previousSource));
    }

    private IEnumerator CutToMusicCoroutine(string musicAddress, AudioSource previousSource)
    {
        var op = Addressables.LoadAssetAsync<AudioClip>(musicAddress);
        yield return op;

        if (op.Result == null)
        {
            Debug.LogWarning($"[AudioManager] Failed to load music: {musicAddress}");
            yield break;
        }

        var musicGo = new GameObject("[MusicSource]");
        musicGo.transform.SetParent(_poolRoot);
        _musicSource = musicGo.AddComponent<AudioSource>();
        _musicSource.clip = op.Result;
        _musicSource.volume = _busConfig.musicVolume;
        _musicSource.spatialBlend = 0f;
        _musicSource.loop = true;
        _musicSource.outputAudioMixerGroup = _busConfig.musicMixerGroup;
        _musicSource.Play();

        // 在新音乐开始播放后再销毁旧音乐源，避免短暂静音
        if (previousSource != null)
        {
            previousSource.Stop();
            Destroy(previousSource.gameObject);
        }
    }
}
```

### 5. Snapshot 状态管理

```csharp
// AudioSnapshotController.cs
/// <summary>
/// 音频快照状态控制器
/// 对应 Immersive Audio GDD §3.1.2 音效分类优先级
/// </summary>
public class AudioSnapshotController
{
    private AudioMixer _mixer;
    private AudioTuningSO _tuning;

    /// <summary>
    /// 快照缓存（避免每次 FindSnapshot 查找开销）
    /// 与 AudioManager._snapshotCache 设计一致：首次访问时缓存，后续复用
    /// </summary>
    private Dictionary<string, AudioMixerSnapshot> _snapshotCache = new();

    /// <summary>
    /// 初始化快照控制器
    /// </summary>
    /// <param name="mixer">AudioMixer 引用（用于 FindSnapshot）</param>
    /// <param name="tuning">调参配置</param>
    public void Initialize(AudioMixer mixer, AudioTuningSO tuning)
    {
        _mixer = mixer;
        _tuning = tuning;
        _snapshotCache.Clear();
    }

    public enum GameAudioState
    {
        Normal,     // 正常游戏
        Combat,     // 战斗状态
        Stealth,    // 潜行状态
        Death,      // 死亡状态
        Menu        // 菜单状态
    }

    /// <summary>
    /// 根据游戏状态切换音频快照
    /// </summary>
    public void TransitionTo(GameAudioState state)
    {
        string snapshotName = state switch
        {
            GameAudioState.Combat => "CombatSnapshot",
            GameAudioState.Stealth => "StealthSnapshot",
            GameAudioState.Death => "DeathSnapshot",
            GameAudioState.Menu => "MenuSnapshot",
            _ => "NormalSnapshot"
        };

        float fadeTime = GetTransitionTime(state);
        TransitionToSnapshot(snapshotName, fadeTime);
    }

    /// <summary>
    /// 切换到指定名称的音频快照
    /// 使用本地缓存避免每帧 FindSnapshot 查找开销（与 AudioManager._snapshotCache 一致）
    /// </summary>
    /// <param name="snapshotName">快照名称（与 AudioMixer 中定义的快照一致）</param>
    /// <param name="transitionTime">过渡时间（秒）</param>
    public void TransitionToSnapshot(string snapshotName, float transitionTime)
    {
        if (_mixer == null)
        {
            Debug.LogWarning("[AudioSnapshotController] AudioMixer not initialized, call Initialize first");
            return;
        }

        // 优先从缓存获取
        if (!_snapshotCache.TryGetValue(snapshotName, out var snapshot))
        {
            snapshot = _mixer.FindSnapshot(snapshotName);
            if (snapshot != null)
            {
                _snapshotCache[snapshotName] = snapshot;
            }
        }

        if (snapshot != null)
        {
            snapshot.TransitionTo(transitionTime);
        }
        else
        {
            Debug.LogWarning($"[AudioSnapshotController] Snapshot '{snapshotName}' not found in AudioMixer");
        }
    }

    private float GetTransitionTime(GameAudioState state)
    {
        return state switch
        {
            GameAudioState.Combat => _tuning.combatTransitionTime,  // 0.3s
            GameAudioState.Death => _tuning.deathTransitionTime,      // 0.5s
            GameAudioState.Menu => _tuning.menuTransitionTime,        // 0.2s
            _ => _tuning.defaultTransitionTime                        // 0.5s
        };
    }
}
```

### 6. 音频优先级与 Ducking 机制

```csharp
// AudioDuckingController.cs
/// <summary>
/// 音频优先级与 Ducking 控制器
/// 当高优先级音效播放时，自动降低低优先级音效的音量
///
/// **Ducking 语义说明**（对应 Immersive Audio GDD §4.4）：
/// - 战斗音乐播放时，环境音（Ambient）自动降低到 30%
/// - 对话播放时，背景音乐自动降低到 20%
/// - 处决音效播放时，所有其他音效临时静音
///
/// **架构说明**：
/// AudioDuckingController 由 AudioManager 持有，与 AudioSourcePool 同级。
/// SFX 播放时自动触发 Ducking，无需外部调用。
///
/// **音量来源说明**：
/// Ducking 的"原始音量"直接从 AudioBusConfig 读取（而非缓存 AudioMixer 当前值），
/// 确保玩家在设置界面调整音量后，Ducking 恢复能使用最新值。
/// </summary>
public class AudioDuckingController
{
    private AudioMixer _audioMixer;
    private AudioTuningSO _tuning;
    private AudioTuningSO.DuckingConfig _duckingConfig;

    /// <summary>
    /// 总线配置引用（原始音量来源，避免缓存过时问题）
    /// </summary>
    private AudioBusConfig _busConfig;

    /// <summary>
    /// 当前活跃的 Ducking 任务
    /// key: AudioBus 枚举值（转换为整数）
    /// </summary>
    private Dictionary<int, ActiveDucking> _activeDuckings = new();

    /// <summary>
    /// 初始化 Ducking 控制器
    /// </summary>
    /// <param name="audioMixer">AudioMixer 引用</param>
    /// <param name="tuning">调参配置</param>
    /// <param name="busConfig">总线配置（原始音量来源，确保恢复时使用玩家最新设置值）</param>
    public void Initialize(AudioMixer audioMixer, AudioTuningSO tuning, AudioBusConfig busConfig)
    {
        _audioMixer = audioMixer;
        _tuning = tuning;
        _duckingConfig = tuning.duckingConfig;
        _busConfig = busConfig;
    }

    /// <summary>
    /// 请求 Ducking（压低其他音频）
    /// 由 AudioManager 在 SFX 播放时自动调用
    /// </summary>
    public void RequestDucking(AudioDuckingRequest request)
    {
        if (_audioMixer == null || _tuning == null) return;

        int triggerKey = (int)request.triggerCategory;

        // 如果已有相同触发类别的 Ducking，更新持续时间
        if (_activeDuckings.TryGetValue(triggerKey, out var existing))
        {
            existing.Duration = request.duration;
            existing.Elapsed = 0f;
            existing.DuckToLevel = request.duckToLevel;
            return;
        }

        // 创建新的 Ducking 任务
        var ducking = new ActiveDucking
        {
            TriggerCategory = request.triggerCategory,
            TargetBuses = request.targetBuses,
            DuckToLevel = request.duckToLevel,
            Duration = request.duration,
            Elapsed = 0f
        };

        _activeDuckings[triggerKey] = ducking;

        // 立即应用 Ducking
        ApplyDucking(ducking);
    }

    /// <summary>
    /// 释放特定触发类别的 Ducking
    /// </summary>
    public void ReleaseDucking(SFXCategory triggerCategory)
    {
        int key = (int)triggerCategory;
        if (_activeDuckings.TryGetValue(key, out var ducking))
        {
            RestoreBusVolumes(ducking.TargetBuses);
            _activeDuckings.Remove(key);
        }
    }

    /// <summary>
    /// 每帧更新（检查 Ducking 超时）
    /// 由 AudioManager 在 Update 中调用
    /// </summary>
    public void Update()
    {
        var toRemove = new List<int>();

        foreach (var kvp in _activeDuckings)
        {
            var ducking = kvp.Value;

            // 永久 Ducking（duration < 0）不自动移除
            if (ducking.Duration < 0f) continue;

            ducking.Elapsed += Time.deltaTime;

            if (ducking.Elapsed >= ducking.Duration)
            {
                toRemove.Add(kvp.Key);
            }
        }

        foreach (var key in toRemove)
        {
            if (_activeDuckings.TryGetValue(key, out var ducking))
            {
                RestoreBusVolumes(ducking.TargetBuses);
                _activeDuckings.Remove(key);
            }
        }
    }

    /// <summary>
    /// 应用 Ducking 到目标总线
    /// 当多个 Ducking 同时影响同一总线时，取 duckToLevel 最低值（最强压制优先），
    /// 与文档 §1582 描述的语义一致
    ///
    /// **原始音量来源**：从 AudioBusConfig 读取（而非缓存值），
    /// 确保玩家修改音量设置后 Ducking 仍正确计算。
    /// </summary>
    private void ApplyDucking(ActiveDucking ducking)
    {
        foreach (var bus in ducking.TargetBuses)
        {
            if (bus == AudioBus.Master) continue;

            string paramName = GetBusVolumeParamName(bus);

            // 取所有活跃 Ducking 中对此总线的最低 duckToLevel（最强压制优先）
            float minDuckLevel = ducking.DuckToLevel;
            foreach (var active in _activeDuckings.Values)
            {
                if (System.Array.IndexOf(active.TargetBuses, bus) >= 0)
                {
                    minDuckLevel = Mathf.Min(minDuckLevel, active.DuckToLevel);
                }
            }

            // 从 AudioBusConfig 读取当前配置的线性音量（始终与玩家设置同步）
            float originalLinear = GetBusConfigLinear(bus);
            float duckedLinear = originalLinear * minDuckLevel;
            float duckedDb = LinearToDb(duckedLinear);
            _audioMixer.SetFloat(paramName, duckedDb);
        }
    }

    /// <summary>
    /// 恢复目标总线的音量
    /// </summary>
    private void RestoreBusVolumes(AudioBus[] buses)
    {
        foreach (var bus in buses)
        {
            if (bus == AudioBus.Master) continue;

            string paramName = GetBusVolumeParamName(bus);
            // 从 AudioBusConfig 读取，保证恢复值是玩家当前设置的目标值
            float originalDb = LinearToDb(GetBusConfigLinear(bus));
            _audioMixer.SetFloat(paramName, originalDb);
        }
    }

    /// <summary>
    /// 从 AudioBusConfig 读取指定总线的线性音量值
    /// </summary>
    private float GetBusConfigLinear(AudioBus bus)
    {
        if (_busConfig == null) return 0.8f;  // 安全默认值

        return bus switch
        {
            AudioBus.SFX      => _busConfig.sfxVolume,
            AudioBus.Ambient  => _busConfig.ambientVolume,
            AudioBus.Dialogue => _busConfig.dialogueVolume,
            AudioBus.Music    => _busConfig.musicVolume,
            AudioBus.UI       => _busConfig.uiVolume,
            _                 => 0.8f
        };
    }

    private string GetBusVolumeParamName(AudioBus bus)
    {
        return bus switch
        {
            AudioBus.Master => "MasterVolume",
            AudioBus.SFX => "SFXVolume",
            AudioBus.Ambient => "AmbientVolume",
            AudioBus.Dialogue => "DialogueVolume",
            AudioBus.Music => "MusicVolume",
            AudioBus.UI => "UIVolume",
            _ => throw new ArgumentException($"[AudioDuckingController] Unknown AudioBus: {bus}")
        };
    }

    private float LinearToDb(float linear) => linear > 0 ? 20f * Mathf.Log10(linear) : -80f;

    private class ActiveDucking
    {
        public SFXCategory TriggerCategory;
        public AudioBus[] TargetBuses;
        public float DuckToLevel;
        public float Duration;
        public float Elapsed;
    }
}

/// <summary>
/// Ducking 请求结构
/// </summary>
public struct AudioDuckingRequest
{
    /// <summary>
    /// 触发 Ducking 的音效类别
    /// </summary>
    public SFXCategory triggerCategory;

    /// <summary>
    /// Ducking 目标总线（哪些总线需要被压低）
    /// </summary>
    public AudioBus[] targetBuses;

    /// <summary>
    /// 压低到目标音量的百分比（0.0-1.0）
    /// 例如：0.3f 表示降低到原始音量的 30%
    /// </summary>
    public float duckToLevel;

    /// <summary>
    /// Ducking 持续时间（秒）
    /// - 0f：立即恢复（瞬时 Ducking）
    /// - > 0f：持续指定时间后自动恢复
    /// - < 0f：永久 Ducking，需要手动调用 ReleaseDucking
    /// </summary>
    public float duration;
}
```

**AudioManager 集成说明**：

在 `AudioManager.cs` 中添加 Ducking 支持：

```csharp
// AudioManager.cs 片段
[SerializeField] private AudioDuckingController _duckingController;

private void Awake()
{
    // ... 现有初始化代码 ...

    // 初始化 Ducking 控制器
    if (_duckingController != null && _audioMixer != null && _tuning != null && _busConfig != null)
    {
        // 传入 _busConfig，确保恢复时始终使用玩家当前设置的音量（而非初始化时缓存的值）
        _duckingController.Initialize(_audioMixer, _tuning, _busConfig);
    }
}

private void Update()
{
    // ... 其他更新逻辑 ...

    // 更新 Ducking 控制器
    _duckingController?.Update();
}

// SFX 播放时自动触发 Ducking（示例：爆炸音效）
private void TriggerScreenEffectSync(SFXCategory category)
{
    switch (category)
    {
        case SFXCategory.Explosion:
        case SFXCategory.ExplosionSmall:
            // 爆炸时压低环境音和音乐
            _duckingController?.RequestDucking(new AudioDuckingRequest
            {
                triggerCategory = category,
                targetBuses = new[] { AudioBus.Ambient, AudioBus.Music },
                duckToLevel = 0.3f,
                duration = 0.5f
            });
            break;

        case SFXCategory.StealthKill:
        case SFXCategory.EnvironmentKill:
            // 处决时压低所有其他音效
            _duckingController?.RequestDucking(new AudioDuckingRequest
            {
                triggerCategory = category,
                targetBuses = new[] { AudioBus.Ambient, AudioBus.Music, AudioBus.Dialogue, AudioBus.UI },
                duckToLevel = 0.1f,
                duration = 0.3f
            });
            break;
    }
}
```

**Ducking 配置（AudioTuningSO 扩展）**：

```csharp
// AudioTuningSO.cs 扩展
[System.Serializable]
public class DuckingConfig
{
    [Header("战斗状态 Ducking")]
    [Tooltip("战斗时环境音降低到的百分比")]
    public float combatAmbientDuckLevel = 0.3f;

    [Tooltip("战斗时环境音 Ducking 持续时间")]
    public float combatDuckingDuration = 1.0f;

    [Header("对话 Ducking")]
    [Tooltip("对话时背景音乐降低到的百分比")]
    public float dialogueMusicDuckLevel = 0.2f;

    [Tooltip("对话时背景音乐 Ducking 持续时间")]
    public float dialogueDuckingDuration = -1f; // 永久，直到对话结束

    [Header("处决 Ducking")]
    [Tooltip("处决时压低其他音效的百分比")]
    public float executionDuckLevel = 0.1f;

    [Tooltip("处决 Ducking 持续时间")]
    public float executionDuckingDuration = 0.3f;
}
```

**Ducking 优先级规则**：
- 同一总线可以被多个 Ducking 影响，取**最低**的 duckToLevel
- 例如：战斗 Ducking (30%) + 爆炸 Ducking (10%) → 最终音量 10%
- 通过在 ApplyDucking 时检查现有值，取最小值实现

### 7. 3D 空间音频配置

> **§7.1 距离衰减配置**：distanceCoefficient、distanceCurve
> **§7.2 环境遮蔽配置**：environmentMultiplier
> **§7.3 空间混音配置**：centerOffset、spatialBlendRange
> 各参数详细定义见代码块。

```csharp
// SpatialAudioConfig.cs
/// <summary>
/// 3D 空间音频配置
/// 对应 Immersive Audio GDD §4.1 音量衰减公式
/// </summary>
[CreateAssetMenu(menuName = "Game/Audio/SpatialConfig")]
public class SpatialAudioConfig : ScriptableObject
{
    [Header("距离衰减")]
    [Tooltip("距离衰减系数，越大衰减越快")]
    public float distanceCoefficient = 0.2f;

    [Tooltip("距离衰减曲线")]
    public AnimationCurve distanceCurve = AnimationCurve.EaseInOut(0f, 1f, 100f, 0f);

    [Header("环境遮蔽")]
    [Tooltip("环境遮蔽系数（墙壁/障碍物阻隔）")]
    public float environmentMultiplier = 0.7f;

    [Header("空间混音")]
    [Tooltip("空间音频中心点偏移")]
    public float centerOffset = 1f;

    [Tooltip("空间混音范围")]
    public float spatialBlendRange = 50f;
}
```

### 7. 调参配置 (AudioTuningSO)

```csharp
// AudioTuningSO.cs
/// <summary>
/// 音频调参配置
/// 包含 Snapshot 切换时间和 DualSense 手柄配置
/// </summary>
[CreateAssetMenu(menuName = "Game/Audio/Tuning")]
public class AudioTuningSO : ScriptableObject
{
    [Header("Snapshot 切换时间")]
    public float combatTransitionTime = 0.3f;
    public float deathTransitionTime = 0.5f;
    public float menuTransitionTime = 0.2f;
    public float defaultTransitionTime = 0.5f;

    [Header("PS5 DualSense 配置")]
    public DualSenseConfig dualSenseConfig = new();
}

/// <summary>
/// PS5 DualSense 手柄触控板和自适应扳机音效配置
///
/// **架构说明**：
/// - DualSenseConfig 仅存储静态配置参数（扳机阻力等级、振动强度等）
/// - 实际 haptics 播放由 HapticFeedbackManager（ADR-0010 §9）统一管理
/// - AudioManager 在适当时机发布 HapticRequest 事件，由 HapticFeedbackManager 处理平台差异
/// - 此设计解耦音频和触觉反馈，便于跨平台适配
/// </summary>
[System.Serializable]
public class DualSenseConfig
{
    [Header("触控板振动配置")]
    public bool touchpadHapticsEnabled = true;
    public float touchpadIntensity = 0.7f;
    public float touchpadFrequency = 60f;  // Hz

    [Header("自适应扳机配置")]
    public bool triggerHapticsEnabled = true;

    [Header("扳机阻力等级 (0-8)")]
    [Range(0, 8)]
    public int triggerResistanceLevel = 4;

    [Header("战斗场景扳机阻力")]
    [Range(0, 8)]
    public int combatTriggerResistance = 7;

    [Header("潜行场景扳机阻力")]
    [Range(0, 8)]
    public int stealthTriggerResistance = 2;

    [Header("处决场景扳机阻力")]
    [Range(0, 8)]
    public int executionTriggerResistance = 8;
}
```

### 8. Unity 项目结构（Presentation Layer）

```
Assets/Game/
├── Presentation/
│   └── Audio/
│       ├── AudioManager.cs              # 主管理器（单例）
│       ├── AudioSourcePool.cs           # AudioSource 对象池
│       ├── AudioBusRouter.cs            # 总线路由逻辑
│       ├── AudioSnapshotController.cs    # 快照状态管理
│       ├── Events/
│       │   ├── AudioEvents.cs           # 音频事件定义
│       │   └── AudioEventIds.cs        # 事件 ID 常量
│       ├── Config/
│       │   ├── AudioBusConfig.cs        # 总线配置
│       │   ├── SpatialAudioConfig.cs   # 空间音频配置
│       │   └── AudioTuningSO.cs         # 调参配置
│       ├── Mixer/
│       │   └── MasterMixer.mixer        # Unity AudioMixer Asset
│       └── Clips/                       # 音效资源（Addressables）
```

---

## Alternatives Considered

### Alternative 1: 使用 FMOD 中间件

- **描述**：使用 FMOD Studio 替代 Unity 内置音频系统
- **Pros**：
  - 更强大的音效管理工具
  - 动态音乐系统成熟
  - 跨平台优化更好
- **Cons**：
  - 引入额外授权费用
  - 团队需要学习 FMOD
  - 与 Unity 原生系统集成复杂度增加
- **拒绝理由**：项目预算不允许额外中间件授权，Unity 内置音频系统可满足需求

### Alternative 2: 使用 Wwise 中间件

- **理由**：同上
- **拒绝理由**：同上

### Alternative 3: 简化架构（无 AudioSource Pool）

- **描述**：运行时动态创建/销毁 AudioSource
- **Pros**：内存管理简单
- **Cons**：
  - GC 开销大
  - 音频卡顿风险
- **拒绝理由**：性能要求高，需要预分配对象池

---

## Consequences

### Positive

- **性能优化**：AudioSource Pool 避免运行时分配
- **总线清晰**：5 总线设计符合 Immersive Audio GDD 规范
- **状态驱动**：Snapshot 切换支持游戏状态变化
- **事件驱动**：EventBus 解耦，支持多系统协同

### Negative

- **内存占用**：AudioSource Pool 预分配占用一定内存
- **调参复杂**：AudioMixer 和 Snapshot 需要仔细调试
- **资源管理**：音效资源通过 Addressables 异步加载，需要缓存策略

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **音频卡顿** | 异步加载导致音效延迟 | 关键音效预加载，其他异步 |
| **总线干扰** | 高优先级音效打断低优先级 | 实现 Ducking 机制（见 GDD §4.4） |
| **平台差异** | 不同平台音频 API 行为差异 | 抽象 PlatformAudioAdapter |
| **内存溢出** | 音效资源缓存过多 | 实现 LRU 缓存淘汰 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 1ms/帧 | 32 个 AudioSource 同时播放 |
| **Memory** | ≤ 64MB（总音频预算），AudioSourcePool 预分配 32 个 AudioSource（3D:16 + 2D:16），属于 ResourceManager OnDemandBudgetMB 的一部分 |
| **Load Time** | < 1s | 关键音效预加载 |
| **Network** | 无 | 单机游戏 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 AudioManager 单例
- [ ] 创建 AudioSourcePool
- [ ] 创建 AudioMixer（5 总线）
- [ ] 定义 AudioEvents

### Phase 2: 音效播放
- [ ] 实现 Play3D/Play2D 接口
- [ ] 集成 Addressables 资源加载
- [ ] 实现音效缓存策略

### Phase 3: 状态管理
- [ ] 创建 AudioSnapshotController
- [ ] 定义各状态 Snapshot
- [ ] 集成 GameStateManager

### Phase 4: 系统集成
- [ ] 与 ScreenEffects 集成
- [ ] 与 Weather System 集成（环境音）
- [ ] 与 GrittyTakedowns 集成（处决音效）
- [ ] 与 UI System 集成（UI 音效）

---

## Validation Criteria

1. **总线验证**：5 个总线音量独立可调
2. **3D 验证**：空间音效随距离正确衰减
3. **Snapshot 验证**：状态切换时音频快照正确过渡
4. **事件验证**：SFXPlayRequest 正确触发音效播放
5. **并发验证**：32 个音效同时播放无卡顿

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — Audio 系统通过 EventBus 通信
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — **Audio System 属于 Presentation Layer**
- [ADR-0019: Addressables 系统架构](./adr-0019-addressables-system-architecture.md) — 音频资源通过 Addressables 管理
- [ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — **ScreenEffectSource 枚举和 ScreenEffectRequestEvent 定义**
- [共享类型定义](./shared-types.md) — **HapticType (§20)、HapticRequest (§20)**、WeaponUsedEvent (§3.7) 定义
- [Immersive Audio & Haptics GDD](../../design/gdd/immersive-audio-haptics.md) — **音效规格定义**

---

## 附录：类型依赖说明

| 类型 | 定义位置 | 说明 |
|------|---------|------|
| `ScreenEffectSource` | ADR-0023 §ScreenEffectSource 枚举 | 屏幕特效来源枚举，已包含 `AudioSystem` 值 |
| `ScreenEffectType` | ADR-0023 | 屏幕特效类型枚举 |
| `ScreenEffectRequestEvent` | ADR-0023 | 屏幕特效请求事件 |
| `ScreenEffectRevokeEvent` | ADR-0023 | 屏幕特效撤销事件 |
| `WeaponUsedEvent` | shared-types.md §3.7 | 武器使用事件 |
| `SFXCategory` | ADR-0025 本文档 | 音效分类枚举 |
| `HapticRequest` | **shared-types.md §20** | 触觉反馈请求事件 |
| `HapticType` | **shared-types.md §20** | 触觉反馈类型枚举（Explosion/Execution/Combat） |

---

## 附录：音效规格映射

| GDD 章节 | 音效类型 | AudioManager 接口 |
|----------|---------|------------------|
| §3.2.1 | 处决音效 | `Play3D(SFXPlayRequest{address="sfx/stealth_kill"})` |
| §3.2.2 | 环境交互音效 | `Play3D(SFXPlayRequest{address="sfx/pickup_metal"})` |
| §3.2.3 | UI 音效 | `Play2D(SFXPlayRequest{address="sfx/ui_click"})` |
| §3.2.4 | NPC 音效 | `Play3D(SFXPlayRequest{address="sfx/npc_footstep"})` |
| §3.4 | 氛围音 | `Play3D(SFXPlayRequest{address="ambient/warehouse", loop=true})` |
| §3.3 | 震动反馈 | 通过 HapticFeedbackManager（见 shared-types.md §20） |
