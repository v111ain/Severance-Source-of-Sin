# ADR-0027: 场景管理系统 (Scene Management System) 架构决策

## Status
**Proposed**

## Date
2026-04-12

## Last Updated
2026-04-12

## Context

### Problem Statement

《断绝：罪恶之源》包含多个游戏区域（城市、关卡），场景管理需要处理：
1. **场景加载/卸载**：无缝过渡、异步加载
2. **场景状态**：Loading Screen、Gameplay、Pause
3. **数据持久化**：跨场景数据传递
4. **资源管理**：场景内资源生命周期
5. **World Map 集成**：场景切换与进度追踪

当前项目已有 World Map 系统（ADR-0012），但缺少场景管理层的具体实现决策：
- 场景加载策略（Additive vs Single）
- 异步加载与进度报告
- 跨场景对象生命周期
- 与 Addressables 资源系统的集成

### Constraints

- **引擎约束**：Unity 6.3 LTS
- **平台约束**：PC & PS5
- **性能约束**：场景加载时间 < 3s（非首次）
- **网络约束**：无（单机游戏）
- **ADR-0012 约束**：World Map 系统已定义区域结构（国家→城市→地区）

### Requirements

- **必须**：定义 SceneManagerWrapper 场景加载封装
- **必须**：定义 LoadingScreen 显示/隐藏接口
- **必须**：定义场景过渡动画接口
- **必须**：支持异步加载和进度报告
- **必须**：遵循 ADR-0003 分层（Scene Management 属于 Foundation Layer）
- **必须**：与 World Map 系统（ADR-0012）集成

---

## Decision

### 架构决策

采用**场景封装层 + 异步加载 + 过渡动画**架构：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    场景管理系统架构图                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  SceneManagerWrapper (场景管理器封装)                         │   │
│  │                                                              │   │
│  │  + LoadSceneAsync(sceneId) → AsyncOperation                  │   │
│  │  + UnloadSceneAsync(sceneId) → AsyncOperation                │   │
│  │  + GetActiveScenes() → List<SceneReference>                  │   │
│  │  + SetActiveScene(sceneId)                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  SceneTransitionController (场景过渡控制器)                    │   │
│  │                                                              │   │
│  │  [Idle] ──LoadScene──▶ [Loading] ──Progress──▶ [FadeOut]     │   │
│  │      ▲                   │                      │            │   │
│  │      │                   └───────────────────────│            │   │
│  │      │              [ActivateScene]              │            │   │
│  │      │                                          ▼            │   │
│  │      ◀──────────────────────────────────── [FadeIn]          │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  LoadingScreenController (加载画面控制)                       │   │
│  │  - 进度条显示                                                │   │
│  │  - 加载提示文本                                              │   │
│  │  - Tips 显示                                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  AddressablesSceneLoader (Addressables 场景加载)              │   │
│  │  - 异步加载场景资产                                          │   │
│  │  - 依赖项预加载                                              │   │
│  │  - 内存管理                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1. SceneReference 场景引用

```csharp
// SceneReference.cs
/// <summary>
/// 场景引用（用于序列化）
/// </summary>
[Serializable]
public struct SceneReference
{
    [SerializeField] private string _sceneGuid;
    [SerializeField] private string _sceneName;

    public string SceneName => _sceneName;
    public string SceneGuid => _sceneGuid;

    /// <summary>
    /// 获取场景的 Addressable 地址
    /// </summary>
    public string AddressableAddress => $"scenes/{_sceneName}";

    /// <summary>
    /// 获取 SceneAsset（Editor 专用）
    /// </summary>
#if UNITY_EDITOR
    [SerializeField] private UnityEditor.SceneAsset _sceneAsset;
#endif
}

// SceneReferenceDrawer.cs (Unity Editor)
#if UNITY_EDITOR
[CustomPropertyDrawer(typeof(SceneReference))]
public class SceneReferenceDrawer : PropertyDrawer
{
    public override void OnGUI(Rect position, SerializedProperty property, GUIContent label)
    {
        var sceneAssetProp = property.FindPropertyRelative("_sceneAsset");

        EditorGUI.BeginProperty(position, label, property);
        sceneAssetProp.objectReferenceValue = EditorGUI.ObjectField(
            position, label, sceneAssetProp.objectReferenceValue, typeof(UnityEditor.SceneAsset), false);
        EditorGUI.EndProperty();
    }
}
#endif
```

### 2. SceneManagerWrapper 场景管理器封装

```csharp
// SceneManagerWrapper.cs
/// <summary>
/// Unity SceneManager 封装层
/// 提供场景加载/卸载的异步接口和进度报告
/// </summary>
public class SceneManagerWrapper
{
    public static SceneManagerWrapper Instance { get; private set; }

    private List<SceneReference> _activeScenes = new();
    private SceneReference _currentMainScene;

    public IReadOnlyList<SceneReference> ActiveScenes => _activeScenes;
    public SceneReference CurrentMainScene => _currentMainScene;

    /// <summary>
    /// 异步加载场景
    /// </summary>
    /// <param name="scene">目标场景引用</param>
    /// <param name="loadSceneMode">加载模式（Single/Additive）</param>
    /// <returns>场景加载操作</returns>
    public AsyncOperation LoadSceneAsync(SceneReference scene, LoadSceneMode loadSceneMode = LoadSceneMode.Additive)
    {
        var address = scene.AddressableAddress;

        // 使用 Addressables 异步加载
        var handle = UnityEngine.AddressableAssets.Addressables.LoadSceneAsync(
            address,
            loadSceneMode,
            activateOnLoad: true);

        // 订阅进度更新
        handle.Completed += op =>
        {
            if (op.Status == AsyncOperationStatus.Succeeded)
            {
                var loadedScene = new SceneReference
                {
                    _sceneName = op.Result.Scene.name,
                    _sceneGuid = scene.SceneGuid
                };

                if (loadSceneMode == LoadSceneMode.Single)
                {
                    _activeScenes.Clear();
                    _currentMainScene = loadedScene;
                }

                _activeScenes.Add(loadedScene);
                EventBus.Instance.Publish(new SceneLoadedEvent { Scene = loadedScene });
            }
            else
            {
                EventBus.Instance.Publish(new SceneLoadFailedEvent { Scene = scene, Error = op.OperationException?.Message });
            }
        };

        return handle;
    }

    /// <summary>
    /// 异步卸载场景
    /// </summary>
    public AsyncOperation UnloadSceneAsync(SceneReference scene)
    {
        var address = scene.AddressableAddress;

        var handle = UnityEngine.AddressableAssets.Addressables.UnloadSceneAsync(address);

        handle.Completed += op =>
        {
            if (op.Status == AsyncOperationStatus.Succeeded)
            {
                _activeScenes.Remove(scene);
                if (_currentMainScene.SceneGuid == scene.SceneGuid)
                {
                    _currentMainScene = default;
                }
                EventBus.Instance.Publish(new SceneUnloadedEvent { Scene = scene });
            }
        };

        return handle;
    }

    /// <summary>
    /// 获取场景加载进度（0-1）
    /// </summary>
    public float GetLoadProgress(AsyncOperation operation)
    {
        if (operation is Progress<float> progress)
            return progress.Current / 100f;
        return operation.progress;
    }
}

public enum LoadSceneMode
{
    Single,    // 卸载其他场景，只保留目标
    Additive   // 添加到当前场景
}
```

### 3. SceneTransitionController 场景过渡控制器

```csharp
// SceneTransitionController.cs
/// <summary>
/// 场景过渡状态机
/// 管理 LoadingScreen 显示和 Fade 动画
/// </summary>
public class SceneTransitionController
{
    public static SceneTransitionController Instance { get; private set; }

    private enum TransitionState
    {
        Idle,
        Loading,
        FadingOut,
        Activating,
        FadingIn
    }

    private TransitionState _state = TransitionState.Idle;
    private SceneReference _targetScene;
    private float _transitionProgress;

    [SerializeField] private float fadeDuration = 0.5f;
    [SerializeField] private LoadingScreenController _loadingScreen;

    public bool IsTransitioning => _state != TransitionState.Idle;

    /// <summary>
    /// 执行场景切换
    /// </summary>
    public void TransitionTo(SceneReference targetScene, LoadSceneMode mode = LoadSceneMode.Single)
    {
        if (_state != TransitionState.Idle)
        {
            Debug.LogWarning("[SceneTransition] Already transitioning");
            return;
        }

        _targetScene = targetScene;
        _transitionProgress = 0f;

        StartTransitionSequence(mode);
    }

    private async void StartTransitionSequence(LoadSceneMode mode)
    {
        // Phase 1: 显示 Loading Screen
        _state = TransitionState.Loading;
        _loadingScreen.Show();
        _loadingScreen.SetProgress(0f);

        EventBus.Instance.Publish(new LoadingScreenRequestEvent
        {
            destination = _targetScene.SceneName,
            destination_type = "area"  // 来自 World Map 的区域类型
        });

        // Phase 2: Fade Out（如果有活跃场景）
        if (SceneManagerWrapper.Instance.ActiveScenes.Count > 0)
        {
            _state = TransitionState.FadingOut;
            await FadeOutAsync();

            // Phase 3: 卸载旧场景
            foreach (var scene in SceneManagerWrapper.Instance.ActiveScenes.ToList())
            {
                SceneManagerWrapper.Instance.UnloadSceneAsync(scene);
            }
        }

        // Phase 4: 加载新场景
        _state = TransitionState.Activating;
        var loadOp = SceneManagerWrapper.Instance.LoadSceneAsync(_targetScene, mode);

        // 更新加载进度
        while (!loadOp.isDone)
        {
            _transitionProgress = SceneManagerWrapper.Instance.GetLoadProgress(loadOp);
            _loadingScreen.SetProgress(_transitionProgress);
            await Task.Delay(100); // 每 100ms 更新一次
        }

        // Phase 5: Fade In
        _state = TransitionState.FadingIn;
        await FadeInAsync();

        // Phase 6: 隐藏 Loading Screen
        _loadingScreen.Hide();
        _state = TransitionState.Idle;

        EventBus.Instance.Publish(new LoadCompletedEvent
        {
            destination = _targetScene.SceneName,
            was_successful = true
        });
    }

    private async Task FadeOutAsync()
    {
        // 触发 ScreenFade（通过 ScreenEffects 系统）
        EventBus.Instance.Publish(new ScreenEffectRequestEvent
        {
            effectType = ScreenEffectType.Fade,
            intensity = 1f,
            sourceSystem = ScreenEffectSource.SceneManagement,
            requesterId = "scene_transition"
        });

        await Task.Delay((int)(fadeDuration * 1000));
    }

    private async Task FadeInAsync()
    {
        // 等待 Fade Out 完成后的一半时间（对称过渡）
        await Task.Delay((int)(fadeDuration * 1000));

        // 触发 Fade In
        EventBus.Instance.Publish(new ScreenEffectRevokeEvent
        {
            sourceSystem = ScreenEffectSource.SceneManagement,
            requesterId = "scene_transition"
        });
    }
}
```

### 4. LoadingScreenController 加载画面控制

```csharp
// LoadingScreenController.cs
/// <summary>
/// Loading Screen UI 控制
/// </summary>
public class LoadingScreenController : MonoBehaviour
{
    [SerializeField] private CanvasGroup _canvasGroup;
    [SerializeField] private UnityEngine.UI.Slider _progressSlider;
    [SerializeField] private TMPro.TextMeshProUGUI _progressText;
    [SerializeField] private TMPro.TextMeshProUGUI _tipText;
    [SerializeField] private string[] _loadingTips;

    private float _targetProgress;
    private float _displayProgress;

    public void Show()
    {
        gameObject.SetActive(true);
        _canvasGroup.alpha = 1f;
        _targetProgress = 0f;
        _displayProgress = 0f;

        // 随机显示一个 Tip
        if (_loadingTips != null && _loadingTips.Length > 0)
        {
            _tipText.text = _loadingTips[Random.Range(0, _loadingTips.Length)];
        }
    }

    public void Hide()
    {
        StartCoroutine(HideCoroutine());
    }

    private IEnumerator HideCoroutine()
    {
        float duration = 0.3f;
        float elapsed = 0f;

        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            _canvasGroup.alpha = 1f - (elapsed / duration);
            yield return null;
        }

        gameObject.SetActive(false);
    }

    public void SetProgress(float progress)
    {
        _targetProgress = Mathf.Clamp01(progress);
    }

    private void Update()
    {
        // 平滑显示进度
        _displayProgress = Mathf.Lerp(_displayProgress, _targetProgress, Time.deltaTime * 10f);
        _progressSlider.value = _displayProgress;
        _progressText.text = $"Loading... {(_displayProgress * 100f):F0}%";
    }
}
```

### 5. 场景事件定义

```csharp
// SceneEvents.cs
public struct SceneLoadedEvent
{
    public SceneReference Scene;
}

public struct SceneLoadFailedEvent
{
    public SceneReference Scene;
    public string Error;
}

public struct SceneUnloadedEvent
{
    public SceneReference Scene;
}

public struct LoadCompletedEvent
{
    public string destination;
    public bool was_successful;
}

public struct SceneStateChangedEvent
{
    public SceneState oldState;
    public SceneState newState;
}

public enum SceneState
{
    /// <summary>加载中</summary>
    Loading,

    /// <summary>游戏进行中</summary>
    Gameplay,

    /// <summary>暂停</summary>
    Paused
}
```

### 6. World Map 集成

```csharp
// WorldMapSceneIntegration.cs
/// <summary>
/// World Map 与 Scene Management 的集成
/// </summary>
public class WorldMapSceneIntegration : MonoBehaviour
{
    [SerializeField] private WorldMapSystem _worldMap;
    [SerializeField] private SceneTransitionController _transitionController;

    private void Start()
    {
        // 订阅 World Map 区域进入事件
        EventBus.Instance.Subscribe<AreaEnteredEvent>(OnAreaEntered);
    }

    private void OnAreaEntered(AreaEnteredEvent evt)
    {
        var areaData = _worldMap.GetAreaData(evt.area_id);
        if (areaData?.sceneReference != null)
        {
            // 触发场景切换
            _transitionController.TransitionTo(areaData.sceneReference, LoadSceneMode.Single);
        }
    }

    private void OnDisable()
    {
        EventBus.Instance.Unsubscribe<AreaEnteredEvent>(OnAreaEntered);
    }
}
```

### 7. 场景配置数据

```csharp
// AreaSceneData.cs
/// <summary>
/// 区域场景配置数据
/// </summary>
[CreateAssetMenu(menuName = "Game/WorldMap/AreaSceneData")]
public class AreaSceneData : ScriptableObject
{
    public string areaId;
    public SceneReference sceneReference;

    [Header("预加载资源")]
    public string[] preloadAddresses;  // Addressables 地址

    [Header("场景设置")]
    public bool enableDayNightCycle = true;
    public bool enableWeatherSystem = true;

    [Header("游戏规则")]
    public bool resetPlayerState = false;
    public bool preserveInventory = true;
}

// WorldMapSystem.cs（扩展）
public partial class WorldMapSystem
{
    /// <summary>
    /// 获取区域的场景数据
    /// </summary>
    public AreaSceneData GetAreaSceneData(string areaId)
    {
        // 从配置中查找
        return _areaSceneDataDict.GetValueOrDefault(areaId);
    }

    private Dictionary<string, AreaSceneData> _areaSceneDataDict = new();
}
```

### 8. Unity 项目结构（Foundation Layer）

```
Assets/Game/
├── Foundation/
│   └── SceneManagement/
│       ├── SceneManagerWrapper.cs          # 场景管理器封装
│       ├── SceneTransitionController.cs     # 过渡状态机
│       ├── LoadingScreenController.cs       # Loading Screen UI
│       ├── WorldMapSceneIntegration.cs     # World Map 集成
│       ├── Data/
│       │   ├── SceneReference.cs           # 场景引用
│       │   └── AreaSceneData.cs           # 区域场景配置
│       ├── Events/
│       │   ├── SceneEvents.cs              # 场景事件
│       │   └── SceneEventIds.cs           # 事件 ID
│       └── Config/
│           └── SceneManagementTuningSO.cs  # 调参配置
```

---

## Alternatives Considered

### Alternative 1: 直接使用 Unity SceneManager

- **描述**：不使用封装层，直接调用 UnityEngine.SceneManagement
- **Pros**：简单直接
- **Cons**：
  - 无法集成 Addressables
  - 进度报告困难
  - 过渡动画难以实现
- **拒绝理由**：需要 Addressables 集成和过渡动画

### Alternative 2: 使用 SceneManagement 插件

- **描述**：使用第三方场景管理插件
- **Pros**：功能完善
- **Cons**：
  - 额外依赖
  - 定制受限
- **拒绝理由**：自研方案已满足需求

---

## Consequences

### Positive

- **封装清晰**：SceneManagerWrapper 抽象底层 API
- **异步友好**：支持进度报告和取消
- **过渡平滑**：Fade In/Out 提供良好体验
- **World Map 集成**：场景切换与进度追踪联动

### Negative

- **复杂度增加**：多层封装需要更多代码
- **调试难度**：异步操作调试复杂

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **内存泄漏** | 场景卸载不彻底 | 使用 Addressables 引用计数 |
| **进度卡顿** | Loading Screen 更新阻塞主线程 | 使用 async/await |
| **场景依赖** | 场景间依赖复杂 | 定义场景依赖表 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 2ms/帧 | 场景加载 |
| **Memory** | 动态 | 取决于场景内容 |
| **Load Time** | < 3s（非首次） | 异步加载 + 进度显示 |
| **Network** | 无 | 单机游戏 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 SceneReference 和 SceneReferenceDrawer
- [ ] 创建 SceneManagerWrapper
- [ ] 创建 SceneTransitionController

### Phase 2: Loading Screen
- [ ] 创建 LoadingScreenController UI
- [ ] 集成 ScreenEffects Fade
- [ ] 实现进度报告

### Phase 3: World Map 集成
- [ ] 创建 AreaSceneData 配置
- [ ] 实现 WorldMapSceneIntegration
- [ ] 测试区域切换流程

### Phase 4: 优化
- [ ] 实现场景预加载
- [ ] 实现资源引用计数
- [ ] 优化加载速度

---

## Validation Criteria

1. **加载验证**：场景切换时 Loading Screen 正确显示进度
2. **过渡验证**：Fade In/Out 平滑，无黑屏闪烁
3. **内存验证**：场景卸载后内存正确释放
4. **World Map 验证**：区域进入触发正确场景加载
5. **进度验证**：加载进度 0-100% 正确更新

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 场景事件通过 EventBus 发布
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — **Scene Management 属于 Foundation Layer**
- [ADR-0012: 世界地图与非线性叙事](./adr-0012-world-map-nonlinear-progression.md) — **World Map 系统已定义区域结构、AreaEnteredEvent**
- [ADR-0019: Addressables 系统架构](./adr-0019-addressables-system-architecture.md) — 场景资源通过 Addressables 加载
- [ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — Loading Screen 过渡使用 Fade 效果、ScreenEffectSource 定义
- [共享类型定义](./shared-types.md) — **LoadingScreenRequestEvent (§19.2)、LoadCompletedEvent (§10.3)**

---

## 附录：类型依赖说明

| 类型 | 定义位置 | 说明 |
|------|---------|------|
| `LoadingScreenRequestEvent` | shared-types.md §19.2 | 加载画面请求事件 |
| `LoadCompletedEvent` | shared-types.md §10.3 | 加载完成事件 |
| `AreaEnteredEvent` | ADR-0012 | World Map 区域进入事件 |
| `SceneReference` | ADR-0027 本文档 | 场景引用结构 |
| `SceneState` | ADR-0027 本文档 | 场景状态枚举 |
| `LoadSceneMode` | ADR-0027 本文档 | 加载模式枚举 |
| `ScreenEffectSource` | ADR-0023 | 屏幕特效来源枚举 |
