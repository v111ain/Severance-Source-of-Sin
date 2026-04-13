# ADR-0025: 音频系统 (Audio System) 架构决策

## Status
**Proposed**

## Date
2026-04-12

## Last Updated
2026-04-12

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
- **预算约束**：音频内存 ≤ 64MB（详见 ADR-0019 Addressables）
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

```csharp
// AudioSourcePool.cs
/// <summary>
/// AudioSource 对象池（预分配避免运行时分配）
/// </summary>
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
    /// <param name="mixer">AudioMixer 引用</param>
    /// <param name="poolRoot">对象池根节点 Transform</param>
    public void Initialize(AudioMixer mixer, Transform poolRoot)
    {
        _poolRoot = poolRoot;
        _sfxMixerGroup = mixer.FindMatchingGroups("SFX")[0];
        _ambientMixerGroup = mixer.FindMatchingGroups("Ambient")[0];

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
    }
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
/// </summary>
public struct SFXStopRequest
{
    public string address;  // 停止指定音效
}

public struct SFXStopAllRequest { }  // 停止所有音效

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
    [SerializeField] private Transform _poolRoot;  // 对象池根节点

    private Dictionary<string, AudioClip> _clipCache = new();

    /// <summary>
    /// 正在播放的音效（一个地址可能对应多个 AudioSource）
    /// </summary>
    private Dictionary<string, List<AudioSource>> _playingSources = new();

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

        _sourcePool.Initialize(_audioMixer, _poolRoot);
        ApplyBusConfig();

        // 订阅事件
        EventBus.Instance.Subscribe<SFXPlayRequest>(OnSFXPlayRequest);
        EventBus.Instance.Subscribe<SFXStopRequest>(OnSFXStopRequest);
        EventBus.Instance.Subscribe<SFXStopAllRequest>(OnSFXStopAllRequest);
        EventBus.Instance.Subscribe<MusicTransitionRequest>(OnMusicTransitionRequest);
    }

    private void OnDestroy()
    {
        if (Instance == this)
        {
            EventBus.Instance.Unsubscribe<SFXPlayRequest>(OnSFXPlayRequest);
            EventBus.Instance.Unsubscribe<SFXStopRequest>(OnSFXStopRequest);
            EventBus.Instance.Unsubscribe<MusicTransitionRequest>(OnMusicTransitionRequest);
            Instance = null;
        }
    }

    /// <summary>
    /// 播放 3D 音效
    /// </summary>
    public void Play3D(SFXPlayRequest request)
    {
        StartCoroutine(Play3DCoroutine(request));
    }

    private IEnumerator Play3DCoroutine(SFXPlayRequest request)
    {
        // 异步加载音效
        var op = Addressables.LoadAssetAsync<AudioClip>(request.address);
        yield return op;

        if (op.Result == null)
        {
            Debug.LogWarning($"[AudioManager] Failed to load clip: {request.address}");
            yield break;
        }

        var source = _sourcePool.Rent3D();
        source.clip = op.Result;
        source.transform.position = request.position;
        source.volume = request.volumeScale;
        source.pitch = 1f + request.pitchShift;
        source.loop = request.loop;
        source.Play();

        // 记录到正在播放列表（支持同一音效多实例）
        if (!_playingSources.TryGetValue(request.address, out var list))
        {
            list = new List<AudioSource>();
            _playingSources[request.address] = list;
        }
        list.Add(source);

        // 触发屏幕特效联动（通过 EventBus）
        TriggerScreenEffectSync(request.category);

        if (!request.loop)
        {
            // 非循环音效，播放完毕后归还
            yield return new WaitForSeconds(op.Result.length);
            ReturnSource(request.address, source, true);
        }
    }

    /// <summary>
    /// 播放 2D 音效
    /// </summary>
    public void Play2D(SFXPlayRequest request)
    {
        StartCoroutine(Play2DCoroutine(request));
    }

    private IEnumerator Play2DCoroutine(SFXPlayRequest request)
    {
        var op = Addressables.LoadAssetAsync<AudioClip>(request.address);
        yield return op;

        if (op.Result == null)
        {
            Debug.LogWarning($"[AudioManager] Failed to load clip: {request.address}");
            yield break;
        }

        var source = _sourcePool.Rent2D();
        source.clip = op.Result;
        source.volume = request.volumeScale;
        source.pitch = 1f + request.pitchShift;
        source.loop = request.loop;
        source.Play();

        // 记录到正在播放列表
        if (!_playingSources.TryGetValue(request.address, out var list))
        {
            list = new List<AudioSource>();
            _playingSources[request.address] = list;
        }
        list.Add(source);

        // UI 音效不触发屏幕特效
        if (request.category != SFXCategory.UI)
        {
            TriggerScreenEffectSync(request.category);
        }

        if (!request.loop)
        {
            yield return new WaitForSeconds(op.Result.length);
            _sourcePool.Return2D(source);
            _playingSources.Remove(request.address);
        }
    }

    /// <summary>
    /// 音效与屏幕特效联动
    /// </summary>
    private void TriggerScreenEffectSync(SFXCategory category)
    {
        // 根据音效分类触发对应的屏幕特效
        switch (category)
        {
            case SFXCategory.Explosion:
            case SFXCategory.ExplosionSmall:
                EventBus.Instance.Publish(new ScreenEffectRequestEvent
                {
                    effectType = ScreenEffectType.Shake,
                    intensity = category == SFXCategory.Explosion ? 0.8f : 0.5f,
                    sourceSystem = ScreenEffectSource.AudioSystem,
                    requesterId = $"sfx_{category}_{Time.time}"
                });
                break;

            case SFXCategory.StealthKill:
            case SFXCategory.EnvironmentKill:
                EventBus.Instance.Publish(new ScreenEffectRequestEvent
                {
                    effectType = ScreenEffectType.Shake,
                    intensity = 0.3f,
                    sourceSystem = ScreenEffectSource.AudioSystem,
                    requesterId = $"sfx_kill_{Time.time}"
                });
                break;
        }
    }

    /// <summary>
    /// 切换音频快照（用于状态变化）
    /// </summary>
    public void TransitionToSnapshot(string snapshotName, float transitionTime)
    {
        var snapshot = _audioMixer.FindSnapshot(snapshotName);
        if (snapshot != null)
        {
            snapshot.TransitionTo(transitionTime);
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

    private float LinearToDb(float linear) => linear > 0 ? 20f * Mathf.Log10(linear) : -80f;

    /// <summary>
    /// 事件处理：音效播放请求
    /// </summary>
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
        if (_playingSources.TryGetValue(request.address, out var source))
        {
            source.Stop();
        }
    }

    private void OnMusicTransitionRequest(MusicTransitionRequest request)
    {
        // 音乐切换逻辑
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

### 6. 3D 空间音频配置

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

### 7. Unity 项目结构（Presentation Layer）

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

### 8. 与 ScreenEffects 系统集成

> **注意**：音效与屏幕特效联动已整合到 AudioManager 内部（`TriggerScreenEffectSync` 方法）。
> AudioScreenEffectSync 类不再需要，作为参考保留。

```csharp
// AudioScreenEffectSync.cs (DEPRECATED - 仅供参考)
// 音效与屏幕特效联动功能已移至 AudioManager.TriggerScreenEffectSync()
// 此文件保留作为架构说明，不在实际代码中使用

/// <summary>
/// 音频与屏幕特效同步（已废弃）
/// 原功能已整合到 AudioManager 内部
/// </summary>
[Obsolete("Use AudioManager.TriggerScreenEffectSync instead")]
public class AudioScreenEffectSync : MonoBehaviour
{
    // 已废弃，不再使用
}
```
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
| **Memory** | ≤ 64MB | 音频资源内存预算（见 ADR-0019） |
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
- [Immersive Audio & Haptics GDD](../../design/gdd/immersive-audio-haptics.md) — **音效规格定义**
- [共享类型定义](./shared-types.md) — WeaponUsedEvent (§3.7) 定义

---

## 附录：类型依赖说明

| 类型 | 定义位置 | 说明 |
|------|---------|------|
| `ScreenEffectSource` | ADR-0023 §X | 屏幕特效来源枚举，需包含 `Audio` 值 |
| `ScreenEffectType` | ADR-0023 | 屏幕特效类型枚举 |
| `ScreenEffectRequestEvent` | ADR-0023 | 屏幕特效请求事件 |
| `ScreenEffectRevokeEvent` | ADR-0023 | 屏幕特效撤销事件 |
| `WeaponUsedEvent` | shared-types.md §3.7 | 武器使用事件 |
| `SFXCategory` | ADR-0025 本文档 | 音效分类枚举 |

**注意**：`ScreenEffectSource.Audio` 枚举值需要在 ADR-0023 的 ScreenEffectSource 枚举中添加。

---

## 附录：音效规格映射

| GDD 章节 | 音效类型 | AudioManager 接口 |
|----------|---------|------------------|
| §3.2.1 | 处决音效 | `Play3D(SFXPlayRequest{address="sfx/stealth_kill"})` |
| §3.2.2 | 环境交互音效 | `Play3D(SFXPlayRequest{address="sfx/pickup_metal"})` |
| §3.2.3 | UI 音效 | `Play2D(SFXPlayRequest{address="sfx/ui_click"})` |
| §3.2.4 | NPC 音效 | `Play3D(SFXPlayRequest{address="sfx/npc_footstep"})` |
| §3.4 | 氛围音 | `Play3D(SFXPlayRequest{address="ambient/warehouse", loop=true})` |
| §3.3 | 震动反馈 | 通过 HapticFeedbackManager（见 ADR-0010 §9） |
