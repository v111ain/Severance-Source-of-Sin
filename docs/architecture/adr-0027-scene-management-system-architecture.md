# ADR-0027: 场景管理系统 (Scene Management System) 架构决策

## Status
**Proposed**

## Date
2026-04-12

## Last Updated
2026-04-15 (ADR 评审修复：SceneStateChangedEvent 字段命名统一为 snake_case，与项目规范一致)

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

### 0.5 SceneManagerWrapper 与 ResourceManager 职责边界

> **重要澄清**（ADR 评审修复 2026-04-14）：
> 本节明确定义 SceneManagerWrapper 与 ResourceManager（ADR-0019）的职责边界，解决两者都操作 Addressables 场景句柄可能导致的竞态条件问题。

#### 职责划分原则

| 组件 | 职责 | 不负责 |
|------|------|--------|
| **SceneManagerWrapper** | Addressables 场景加载/卸载的底层封装，管理场景实例生命周期 | 场景内资源、引用计数 |
| **ResourceManager** | 场景内 Addressable 资源（纹理、模型、音频等）的加载/卸载和引用计数 | 场景实例本身 |

#### 关键约束

1. **SceneManagerWrapper 是 Addressables 场景句柄的唯一所有者**
   - `LoadSceneAsync()` 返回的 `AsyncOperationHandle<SceneInstance>` 由 SceneManagerWrapper 持有
   - `UnloadSceneAsync()` 是释放场景句柄的唯一入口
   - ResourceManager 不直接调用 `Addressables.UnloadSceneAsync()`

2. **ResourceManager 仅通过事件感知场景卸载**
   - SceneManagerWrapper 在场景卸载完成后发布 `SceneUnloadedEvent`
   - ResourceManager 订阅此事件，清理场景绑定的资源
   - ResourceManager 不持有 Addressables 场景句柄

3. **禁止的调用模式**
   ```
   // ❌ 错误：ResourceManager 直接卸载场景
   Addressables.UnloadSceneAsync(sceneHandle);

   // ✅ 正确：通过 SceneManagerWrapper 卸载
   SceneManagerWrapper.Instance.UnloadSceneAsync(scene);
   ```

#### 事件订阅关系

```csharp
// ResourceManager（正确做法）
public void OnSceneUnloaded(SceneUnloadedEvent e)
{
    // 清理该场景绑定的资源，但不释放场景句柄
    CleanupSceneBoundResources(e.Scene.SceneGuid);
}
EventBus.Subscribe<SceneUnloadedEvent>(OnSceneUnloaded);
```

#### 与 ADR-0019 的接口约定

| 接口 | 方向 | 说明 |
|------|------|------|
| `RegisterSceneBound(handle, assets)` | SceneManagerWrapper → ResourceManager | 场景加载后，注册场景内的资源供 ResourceManager 管理 |
| `SceneUnloadedEvent` | SceneManagerWrapper → ResourceManager | 场景卸载通知，触发资源清理 |
| `UnloadSceneAsync(scene)` | 调用方 → SceneManagerWrapper | 唯一允许的场景卸载入口 |

---

### 1. SceneReference 场景引用

```csharp
// SceneReference.cs
/// <summary>
/// 场景引用（用于序列化）
/// 注意：场景的 Addressable 地址由 _sceneGuid 决定，而非 _sceneName。
/// 这是因为 Addressables 使用 GUID 作为地址，名称可能变化但 GUID 唯一。
/// </summary>
[Serializable]
public struct SceneReference
{
    [SerializeField] private string _sceneGuid;
    [SerializeField] private string _sceneName;

    /// <summary>
    /// 构造函数（P1 修复：私有字段无法在外部用对象初始化器赋值，需通过构造函数创建）
    /// </summary>
    public SceneReference(string sceneGuid, string sceneName)
    {
        _sceneGuid = sceneGuid;
        _sceneName = sceneName;
#if UNITY_EDITOR
        _sceneAsset = null;
#endif
    }

    public string SceneName => _sceneName;
    public string SceneGuid => _sceneGuid;

    /// <summary>
    /// 检查场景引用是否有效（Guid 不为空）
    /// </summary>
    public bool IsValid => !string.IsNullOrEmpty(_sceneGuid);

    /// <summary>
    /// 获取场景的 Addressable 地址
    /// 注意：Addressables 地址是用户定义的字符串（如 "scenes/warehouse_block_a"），不是 Unity 内部 GUID
    /// SceneReference 需要在 Editor 时解析 GUID 到实际 Addressables 地址
    /// </summary>
    /// <exception cref="InvalidOperationException">当 SceneGuid 为空时抛出</exception>
    public string AddressableAddress => GetAddressableAddress();

    /// <summary>
    /// 获取可配置前缀的 Addressable 地址
    /// 注意：此处应解析 _sceneGuid 为实际的 Addressables 地址，而非直接拼接
    /// 地址格式为 "scenes/{addressable_name}"，由用户在 Addressables 窗口配置
    /// </summary>
    /// <exception cref="InvalidOperationException">当 SceneGuid 为空时抛出</exception>
    public string GetAddressableAddress(string prefix = "scenes/")
    {
        if (!IsValid)
            throw new InvalidOperationException($"[SceneReference] Cannot get Addressable address for invalid scene reference (sceneName: {_sceneName})");

        // P0 修复：Addressables 地址是用户定义的字符串，不是 Unity GUID
        // 正确做法：在 Editor 时通过 UnityEditor.AddressableAssets.AddressableAssetSettings
        // 将 _sceneGuid 解析为实际的 Addressables 地址
        // 示例：如果用户在 Addressables 窗口将 "warehouse_block_a" 场景标记为 "scenes/warehouse_block_a"
        // 则解析后返回 "scenes/warehouse_block_a"（而非 "scenes/2c4a8f3e9b1d..."）
        //
        // 临时实现（待 Editor 脚本补充）：
        return $"{prefix}{_sceneName.ToLower().Replace(" ", "_")}";
    }

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

> **SceneReference 地址前缀约定** [已修复]：
> 所有场景的 Addressable 地址统一使用 `scenes/` 前缀 + 用户定义名称，例如：
> - 正确地址：`scenes/warehouse_block_a`
> - 错误地址：`scenes/2c4a8f3e9b1d4f6a8c7e9b2d4f6a8c7e`（这是 Unity 内部 GUID，不是 Addressables 地址）
>
> **约定来源**：此约定与 ADR-0019 Addressables 系统的 `scenes/` 前缀规范保持一致。
> SceneReference.GetAddressableAddress() 需要在 Editor 时解析场景 GUID 为实际的 Addressables 地址。

### 2. SceneManagerWrapper 场景管理器封装

> **P2 修复说明**：SceneManagerWrapper 改为 MonoBehaviour + Instance 单例模式，与 ADR-0003 规范保持一致。
> ADR-0003 规定所有需要全局访问的子系统统一使用 MonoBehaviour 单例模式，
> 避免"穷人的单例"（无参构造函数 + static Instance）导致的初始化顺序问题。

```csharp
// SceneManagerWrapper.cs
/// <summary>
/// Unity SceneManager 封装层
/// 提供场景加载/卸载的异步接口和进度报告
/// </summary>
public class SceneManagerWrapper : MonoBehaviour
{
    /// <summary>
    /// 单例实例（MonoBehaviour + Instance 模式）
    /// </summary>
    public static SceneManagerWrapper Instance { get; private set; }

    /// <summary>
    /// 重置单例（仅用于测试场景，生产代码勿调用）
    /// </summary>
    public static void Reset()
    {
        if (Instance != null)
        {
            Destroy(Instance.gameObject);
            Instance = null;
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
        // SceneManagerWrapper 需要跨场景持久化，因为场景过渡过程中不能被销毁
        DontDestroyOnLoad(gameObject);
    }

    // 使用 Dictionary + List 混合结构：
    // - Single 模式：_currentMainScene 存储当前主场景，_mainSceneCache 作为 GUID→SceneReference 映射缓存
    // - Additive 模式：_additiveScenes 存储叠加场景列表（支持同一场景多次加载）
    private Dictionary<string, SceneReference> _mainSceneCache = new();
    private List<SceneReference> _additiveScenes = new();
    private SceneReference _currentMainScene;

    public IReadOnlyCollection<SceneReference> ActiveScenes
    {
        get
        {
            // 优化：避免每次访问都 new List<>()，改用可复用的缓存列表
            // 注意：返回的是只读视图，调用者不应修改
            _cachedActiveScenes.Clear();
            if (_currentMainScene.IsValid) _cachedActiveScenes.Add(_currentMainScene);
            _cachedActiveScenes.AddRange(_additiveScenes);
            return _cachedActiveScenes;
        }
    }
    private List<SceneReference> _cachedActiveScenes = new();

    public SceneReference CurrentMainScene => _currentMainScene;

    /// <summary>
    /// 异步加载场景
    /// </summary>
    /// <param name="scene">目标场景引用</param>
    /// <param name="loadSceneMode">加载模式（Single/Additive）</param>
    /// <returns>Addressables 场景加载句柄（P1 修复：返回 AsyncOperationHandle 而非 AsyncOperation，
    /// 以保留 Status/OperationException 等 Addressables 专有属性）</returns>
    public AsyncOperationHandle<SceneInstance> LoadSceneAsync(SceneReference scene, SceneLoadMode loadSceneMode = SceneLoadMode.Additive)
    {
        var address = scene.GetAddressableAddress();

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
                // **GUID 一致性保证**：
                // Addressables 加载场景后，op.Result.Scene 的 GUID 与传入的 SceneReference._sceneGuid 一致。
                // 这里保留原始 _sceneGuid 而非从 Result 获取，因为：
                // 1. SceneReference 在序列化和反序列化后，_sceneGuid 已经过验证
                // 2. 避免 Addressables 内部实现差异导致的 GUID 不一致
                // 3. 确保 SceneReference 在整个游戏生命周期内保持稳定引用
                //
                // P1 修复：使用构造函数代替对象初始化器（私有字段无法在外部直接赋值）
                var loadedScene = new SceneReference(scene.SceneGuid, op.Result.Scene.name);

                if (loadSceneMode == SceneLoadMode.Single)
                {
                    // Single 模式：清空所有叠加场景
                    _additiveScenes.Clear();
                    _currentMainScene = loadedScene;
                    _mainSceneCache[loadedScene.SceneGuid] = loadedScene;
                }
                else
                {
                    // Additive 模式：添加到叠加场景列表
                    // 注意：同一场景可以被加载多次，每次加载都添加一个新条目
                    _additiveScenes.Add(loadedScene);
                }

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
    /// 注意：Additive 模式下如果同一场景被加载多次，只卸载最近加载的那个实例
    /// 返回 AsyncOperationHandle（而非 AsyncOperation），以访问 Addressables 专有的
    /// Status/OperationException 等属性，并与 LoadSceneAsync 的返回类型保持一致。
    /// </summary>
    public AsyncOperationHandle UnloadSceneAsync(SceneReference scene)
    {
        var address = scene.GetAddressableAddress();

        var handle = UnityEngine.AddressableAssets.Addressables.UnloadSceneAsync(address);

        handle.Completed += op =>
        {
            if (op.Status == AsyncOperationStatus.Succeeded)
            {
                // 从 Single 场景移除
                _mainSceneCache.Remove(scene.SceneGuid);
                if (_currentMainScene.SceneGuid == scene.SceneGuid)
                {
                    _currentMainScene = default;
                }

                // 从 Additive 场景列表移除（只移除最后一个匹配项）
                for (int i = _additiveScenes.Count - 1; i >= 0; i--)
                {
                    if (_additiveScenes[i].SceneGuid == scene.SceneGuid)
                    {
                        _additiveScenes.RemoveAt(i);
                        break;  // 只移除一个
                    }
                }

                EventBus.Instance.Publish(new SceneUnloadedEvent { Scene = scene });
            }
            else
            {
                Debug.LogError($"[SceneManagerWrapper] Failed to unload scene: {scene.SceneName}, Error: {handle.OperationException?.Message}");
                EventBus.Instance.Publish(new SceneUnloadFailedEvent { Scene = scene, Error = handle.OperationException?.Message });
            }
        };

        return handle;
    }

    /// <summary>
    /// 获取场景加载进度（0-1）
    /// 使用 AsyncOperationHandle.PercentComplete 替代已弃用的 AsyncOperation.progress
    /// </summary>
    public float GetLoadProgress(AsyncOperationHandle<SceneInstance> handle)
    {
        return handle.PercentComplete;
    }
}

public enum SceneLoadMode
{
    Single,    // 卸载其他场景，只保留目标
    Additive   // 添加到当前场景
}
```

### 3. SceneTransitionController 场景过渡控制器

```
┌─────────────────────────────────────────────────────────────────────┐
│                    场景过渡状态转换图                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐                                                       │
│  │   Idle   │◄─────────────────────────────────────┐               │
│  └────┬─────┘                                       │               │
│       │                                              │               │
│       │ TransitionTo()                               │               │
│       ▼                                              │               │
│  ┌──────────┐    有活跃场景     ┌──────────┐        │               │
│  │  Loading  │────────────────► │ FadingOut│        │               │
│  └────┬─────┘                   └────┬─────┘        │               │
│       │                              │               │               │
│       │                              │ Fade 完成后   │               │
│       │                              ▼               │               │
│       │                        ┌──────────┐         │               │
│       │                        │ Unload   │         │               │
│       │                        └────┬─────┘         │               │
│       │                              │               │               │
│       │                              │ 卸载完成后    │               │
│       │                              ▼               │               │
│       │                        ┌──────────┐         │               │
│       │◄────────────────────────│ Activating│────────┘ FadingIn完成后│
│       │                         └────┬─────┘                   │     │
│       │                               │                         │     │
│       │                               │ 加载完成后              │     │
│       │                               ▼                         │     │
│       │                         ┌──────────┐                    │     │
│       │◄─────────────────────────│ FadingIn │────────────────────┘     │
│                                  └──────────┘                          │
│                                                                     │
│  **状态说明**：                                                      │
│  - Idle：初始状态，无过渡进行中                                       │
│  - Loading：显示 Loading Screen，准备过渡                             │
│  - FadingOut：Fade 到黑色（如果有活跃场景）                          │
│  - Unload：卸载旧场景                                               │
│  - Activating：加载新场景                                           │
│  - FadingIn：Fade 到正常显示                                        │
│                                                                     │
│  **中断处理**：                                                       │
│  - CancelTransition() 可在任何状态调用，取消当前过渡                 │
│  - 回到 Idle 状态，Loading Screen 隐藏                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 状态转换条件

| 当前状态 | 触发条件 | 下一状态 | 说明 |
|---------|---------|---------|------|
| Idle | `TransitionTo()` | Loading | 开始过渡 |
| Loading | FadeOut 完成（如果有场景）/ 直接跳转 | FadingOut/Unload | 有活跃场景才 FadeOut |
| FadingOut | Fade 完成 | Unload | 卸载旧场景 |
| Unload | 卸载完成 | Activating | 加载新场景 |
| Activating | 加载完成 | FadingIn | FadeIn 显示 |
| FadingIn | Fade 完成 | Idle | 过渡结束 |
| Any | `CancelTransition()` | Idle | 中断过渡 |

```csharp
// SceneTransitionController.cs
using System.Collections;
using UnityEngine;

/// <summary>
/// 场景过渡状态机
/// 管理 LoadingScreen 显示和 Fade 动画
///
/// **协程方案说明**：
/// 本类使用协程（IEnumerator）替代 async Task 实现异步等待。
/// 协程在 Unity 主线程上执行，所有 Unity API 调用（如 EventBus.Publish、UI 操作）
/// 均保证在主线程执行，不存在线程安全问题。
///
/// **CancellationToken 处理**：
/// 协程中无法直接使用 `cancellationToken.ThrowIfCancellationRequested()`，
/// 改为检查 `cancellationToken.IsCancellationRequested` 后 `yield break` 退出协程。
/// </summary>
public class SceneTransitionController : MonoBehaviour
{
    public static SceneTransitionController Instance { get; private set; }

    private void Awake()
    {
        if (Instance != null)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;

        // SceneTransitionController 需要跨场景持久化，因为过渡过程中玩家操作可能触发场景切换
        // 如果在过渡过程中被销毁，可能导致永久黑屏或 Loading Screen 无法消失
        DontDestroyOnLoad(gameObject);
    }

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

    /// <summary>
    /// 当前转换的取消令牌（用于在转换中途取消）
    /// </summary>
    private CancellationTokenSource _transitionCts;

    /// <summary>
    /// 对象销毁标志（用于防止 MonoBehaviour 销毁后 async 方法继续执行）
    /// </summary>
    private bool _isDisposed;

    public bool IsTransitioning => _state != TransitionState.Idle;

    /// <summary>
    /// 执行场景切换
    /// </summary>
    /// <param name="targetScene">目标场景</param>
    /// <param name="mode">加载模式：
    /// - Single：卸载所有现有场景，加载目标场景（用于 World Map 区域切换）
    /// - Additive：叠加到现有场景（用于临时场景如教程覆盖层）</param>
    /// <param name="cancellationToken">取消令牌（可选）</param>
    public void TransitionTo(SceneReference targetScene, SceneLoadMode mode = SceneLoadMode.Single, CancellationToken cancellationToken = default)
    {
        if (_state != TransitionState.Idle)
        {
            Debug.LogWarning("[SceneTransition] Already transitioning");
            return;
        }

        if (_loadingScreen == null)
        {
            Debug.LogError("[SceneTransition] LoadingScreen is not assigned, cannot transition");
            return;
        }

        _targetScene = targetScene;
        _transitionProgress = 0f;
        _transitionCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);

        StartCoroutine(StartTransitionSequence(mode, _transitionCts.Token));
    }

    /// <summary>
    /// 取消正在进行的场景切换（通常由玩家操作触发）
    /// </summary>
    public void CancelTransition()
    {
        if (_state == TransitionState.Idle) return;

        Debug.Log("[SceneTransition] Transition cancelled");
        _transitionCts?.Cancel();

        // 清理状态
        _loadingScreen?.Hide();
        _state = TransitionState.Idle;
        _transitionCts?.Dispose();
        _transitionCts = null;
    }

    /// <summary>
    /// MonoBehaviour 销毁时调用
    /// 防止 async 方法在对象销毁后继续执行
    /// </summary>
    public void OnDestroy()
    {
        _isDisposed = true;
        _transitionCts?.Cancel();
        _transitionCts?.Dispose();
        _transitionCts = null;
    }

    private IEnumerator StartTransitionSequence(SceneLoadMode mode, CancellationToken cancellationToken)
    {
        if (cancellationToken.IsCancellationRequested)
        {
            Debug.LogWarning("[SceneTransition] Transition aborted: controller is being destroyed");
            yield break;
        }

        // **注意：C# 规范不允许在 catch/finally 子句中使用 yield。**
        // 这里用标志位替代 catch 中的 yield break，确保代码可编译。
        // try 块内的 yield return 在 C# 7+ 是合法的，无需规避。
        bool transitionAborted = false;

        try
        {
            // Phase 1: 显示 Loading Screen
            _state = TransitionState.Loading;
            _loadingScreen.Show();
            _loadingScreen.SetTargetProgress(0f);

            if (cancellationToken.IsCancellationRequested) yield break;

            EventBus.Instance.Publish(new LoadingScreenRequestEvent
        {
            destination = _targetScene.SceneName,
            destination_type = "area"  // 来自 World Map 的区域类型
        });

        // Phase 2: Fade Out（如果有活跃场景）
        if (SceneManagerWrapper.Instance.ActiveScenes.Count > 0)
        {
            if (cancellationToken.IsCancellationRequested) yield break;

            _state = TransitionState.FadingOut;
            yield return FadeOutAsync(cancellationToken);

            if (cancellationToken.IsCancellationRequested) yield break;

            // Phase 3: 卸载旧场景（等待卸载完成后再加载新场景）
            var unloadOps = new List<AsyncOperationHandle>();
            foreach (var scene in SceneManagerWrapper.Instance.ActiveScenes.ToList())
            {
                var unloadOp = SceneManagerWrapper.Instance.UnloadSceneAsync(scene);
                unloadOps.Add(unloadOp);
            }

            foreach (var op in unloadOps)
            {
                yield return op;
            }

            if (cancellationToken.IsCancellationRequested) yield break;
        }

        // Phase 4: 加载新场景
        if (cancellationToken.IsCancellationRequested) yield break;

        _state = TransitionState.Activating;

        // P1 修复：使用 ConsumePreload 尝试复用预加载的句柄
        // 如果没有预加载，SceneManagerWrapper.LoadSceneAsync 会正常加载
        var preloadHandle = _preloader?.ConsumePreload(_targetScene);
        AsyncOperationHandle<SceneInstance> loadOp;

        if (preloadHandle.HasValue && preloadHandle.Value.IsValid())
        {
            // 使用预加载的句柄激活场景
            loadOp = preloadHandle.Value;
            // 激活已加载但不激活的场景实例
            yield return loadOp.Result.ActivateAsync();

            // P0 修复：预加载句柄必须在激活完成后释放，避免内存泄漏
            // （与 ADR-0019 释放契约保持一致）
            Addressables.Release(loadOp);
        }
        else
        {
            // 没有预加载，正常加载
            loadOp = SceneManagerWrapper.Instance.LoadSceneAsync(_targetScene, mode);

            // P1 修复：AsyncOperationHandle 没有 IsDone 属性，使用 Status != AsyncOperationStatus.None 判断
            while (loadOp.Status == AsyncOperationStatus.None)
            {
                if (cancellationToken.IsCancellationRequested) yield break;

                _transitionProgress = SceneManagerWrapper.Instance.GetLoadProgress(loadOp);
                _loadingScreen.SetTargetProgress(_transitionProgress);
                yield return null;  // 每帧轮询，避免 WaitForSeconds 引入的延迟累积
            }
        }

        _loadingScreen.SetTargetProgress(1f);

        // Phase 5: Fade In
        if (cancellationToken.IsCancellationRequested) yield break;

        _state = TransitionState.FadingIn;
        yield return FadeInAsync(cancellationToken);

        if (cancellationToken.IsCancellationRequested) yield break;

        // Phase 6: 隐藏 Loading Screen
        _loadingScreen.Hide();
        _state = TransitionState.Idle;

        EventBus.Instance.Publish(new LoadCompletedEvent
        {
            destination = _targetScene.SceneName,
            destination_type = "area",  // 与 LoadingScreenRequestEvent.destination_type 保持一致
            was_successful = true
        });
        }
        catch (OperationCanceledException)
        {
            // yield break 不允许在 catch 块中使用（CS1631），改用标志位
            Debug.Log("[SceneTransition] Transition was cancelled");
            transitionAborted = true;
        }
        catch (Exception ex)
        {
            // yield break 不允许在 catch 块中使用（CS1631），改用标志位
            Debug.LogException(ex);
            transitionAborted = true;
        }
        finally
        {
            _transitionCts?.Dispose();
            _transitionCts = null;
        }

        // catch 块中无法 yield break，在 try/catch/finally 全部结束后统一处理
        if (transitionAborted)
        {
            _loadingScreen?.Hide();
            _state = TransitionState.Idle;
        }
    }

    private IEnumerator FadeOutAsync(CancellationToken cancellationToken)
    {
        // 触发 ScreenFade（通过 ScreenEffects 系统）
        // 设置 permanent=true 使 fade 保持黑色，直到显式撤销
        EventBus.Instance.Publish(new ScreenEffectRequestEvent
        {
            effectType = ScreenEffectType.Fade,
            intensity = 1f,
            permanent = true,
            sourceSystem = ScreenEffectSource.SceneManagement,
            requesterId = "scene_transition"
        });

        yield return new WaitForSeconds(fadeDuration);
    }

    private IEnumerator FadeInAsync(CancellationToken cancellationToken)
    {
        // 等待场景激活完成
        yield return new WaitForSeconds(fadeDuration);

        // 撤销旧的永久黑色 fade 效果
        //
        // **ScreenEffectRevokeEvent 语义说明**：
        // 此处撤销的是 FadeInAsync 之前的 FadeOutAsync 设置的 permanent=true fade。
        // revocation 逻辑：撤销所有 sourceSystem=SceneManagement 且 requesterId="scene_transition" 的效果。
        // 由于场景切换流程中每个步骤使用不同的 requesterId，revoke 只会影响"保持黑色"的那个永久 fade，
        // 不会影响其他 screen effects。
        //
        // **依赖验证**：
        // ADR-0023 ScreenEffectsManager.RevokeScreenEffect() 已验证支持按 sourceSystem + requesterId 精确撤销：
        // ```csharp
        // private void RevokeScreenEffect(ScreenEffectSource sourceSystem, string requesterId)
        // {
        //     _effectLayers.RemoveAll(l =>
        //         l.sourceSystem == sourceSystem &&
        //         l.requesterId == requesterId);
        // }
        // ```
        // 因此本实现与 ADR-0023 的实现一致，不存在"永久黑屏"或"闪白"风险。
        //
        // **时序说明**：
        // ScreenEffectsManager.RevokeScreenEffect 是同步操作（立即从列表移除），
        // 无需等待回调。WaitForSeconds(0.05f) 是为了确保撤销完成后 fade-in 请求才发出，
        // 避免极罕见的竞态条件。
        EventBus.Instance.Publish(new ScreenEffectRevokeEvent
        {
            sourceSystem = ScreenEffectSource.SceneManagement,
            requesterId = "scene_transition"
        });

        // 等待撤销生效（确保屏幕不会闪白）
        yield return new WaitForSeconds(0.05f);

        // 请求新的 fade-in 效果：逐渐将屏幕从黑色过渡到可见
        //
        // **Fade 效果说明**：详见 ADR-0023 ScreenEffectsManager.BlendParams 方法。
        // 简单来说：先请求 intensity=1 的黑色 fade（permanent=true），然后撤销，
        // 再请求 intensity=0 的 fade（permanent=false, duration），系统会从当前强度（1）渐变到 0。
        EventBus.Instance.Publish(new ScreenEffectRequestEvent
        {
            effectType = ScreenEffectType.Fade,
            intensity = 0f,  // 目标强度为 0（完全可见）
            duration = fadeDuration,
            permanent = false,  // 使用 duration 自动移除
            sourceSystem = ScreenEffectSource.SceneManagement,
            requesterId = "scene_transition_fadein"
        });

        // 等待渐变完成
        yield return new WaitForSeconds(fadeDuration);
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
        if (_canvasGroup == null || _progressSlider == null || _progressText == null)
        {
            Debug.LogWarning("[LoadingScreenController] UI components not assigned, cannot show loading screen");
            return;
        }

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
        if (_canvasGroup == null) return;
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

    /// <summary>
    /// 设置目标加载进度（实际显示值通过 Update 中的 Lerp 平滑过渡）
    /// </summary>
    /// <param name="progress">目标进度值 [0, 1]</param>
    public void SetTargetProgress(float progress)
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

/// <summary>
/// 场景卸载失败事件
/// </summary>
public struct SceneUnloadFailedEvent
{
    public SceneReference Scene;
    public string Error;
}

/// <summary>
/// 场景状态变化事件
/// </summary>
public struct SceneStateChangedEvent
{
    public SceneState old_state;  // snake_case，与项目命名规范一致
    public SceneState new_state;  // snake_case，与项目命名规范一致
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

### 5.5. 场景预加载机制

```csharp
// ScenePreloader.cs
/// <summary>
/// 场景预加载器
/// 在适当时机预加载即将需要的场景资源，减少加载等待时间
/// </summary>
public class ScenePreloader
{
    /// <summary>
    /// 预加载优先级
    /// </summary>
    public enum PreloadPriority
    {
        /// <summary>低优先级：后台预加载，可被中断</summary>
        Low,
        /// <summary>中优先级：标准预加载</summary>
        Normal,
        /// <summary>高优先级：立即预加载</summary>
        High
    }

    /// <summary>
    /// 预加载任务状态
    /// P1 修复：Handle 类型改为 AsyncOperationHandle<SceneInstance>，
    /// 以访问 Addressables 专有的 Status/OperationException 属性
    /// </summary>
    private class PreloadTask
    {
        public SceneReference Scene;
        public PreloadPriority Priority;
        /// <summary>
        /// P1 修复：类型改为 AsyncOperationHandle&lt;SceneInstance&gt;，
        /// 以访问 Addressables 专有的 Status/OperationException 属性，
        /// 并持有已加载的场景实例句柄（不 Unload，供正式切换时复用）
        /// </summary>
        public AsyncOperationHandle<SceneInstance> Handle;
        public float StartTime;
        public bool IsCompleted;
    }

    private ScenePreloaderConfig _config;
    private List<PreloadTask> _activeTasks = new();
    private Dictionary<string, PreloadTask> _completedTasks = new();

    /// <summary>
    /// 初始化预加载器
    /// </summary>
    public void Initialize(ScenePreloaderConfig config)
    {
        _config = config;
    }

    /// <summary>
    /// 请求预加载指定场景
    /// </summary>
    /// <param name="scene">目标场景</param>
    /// <param name="priority">预加载优先级</param>
    /// <returns>是否成功发起预加载请求</returns>
    public bool RequestPreload(SceneReference scene, PreloadPriority priority = PreloadPriority.Normal)
    {
        // 检查是否已完成预加载
        if (_completedTasks.ContainsKey(scene.SceneGuid))
        {
            return true;
        }

        // 检查是否正在预加载
        if (_activeTasks.Any(t => t.Scene.SceneGuid == scene.SceneGuid))
        {
            // 如果新请求优先级更高，提升现有任务优先级
            var existing = _activeTasks.First(t => t.Scene.SceneGuid == scene.SceneGuid);
            if (priority > existing.Priority)
            {
                existing.Priority = priority;
            }
            return true;
        }

        // 检查并发限制
        if (_activeTasks.Count >= _config.maxConcurrentPreloads)
        {
            // 如果当前有低优先级任务，取消最低优先级任务
            var lowestPriority = _activeTasks.OrderBy(t => t.Priority).FirstOrDefault();
            if (lowestPriority != null && lowestPriority.Priority < priority)
            {
                CancelPreload(lowestPriority);
            }
            else
            {
                return false;  // 无法接受新任务
            }
        }

        // 创建预加载任务
        var task = new PreloadTask
        {
            Scene = scene,
            Priority = priority,
            StartTime = Time.time
        };

        // 开始预加载：使用 Additive 模式 + activateOnLoad=false
        // 场景加载到内存但不激活，句柄由 task.Handle 持有直到正式切换或超时清理
        task.Handle = Addressables.LoadSceneAsync(
            scene.GetAddressableAddress(),
            SceneLoadMode.Additive,
            activateOnLoad: false);
        // P1 修复：AsyncOperationHandle 使用大写 .Completed（而非 AsyncOperation 的小写 .completed）
        task.Handle.Completed += op => OnPreloadCompleted(task);

        _activeTasks.Add(task);
        return true;
    }

    /// <summary>
    /// 检查场景是否已预加载完成
    /// </summary>
    public bool IsPreloaded(SceneReference scene)
    {
        return _completedTasks.ContainsKey(scene.SceneGuid);
    }

    /// <summary>
    /// 获取预加载进度
    /// </summary>
    /// <param name="scene">目标场景</param>
    /// <returns>0.0 - 1.0 的进度值，如果未开始预加载返回 -1</returns>
    public float GetPreloadProgress(SceneReference scene)
    {
        var activeTask = _activeTasks.FirstOrDefault(t => t.Scene.SceneGuid == scene.SceneGuid);
        if (activeTask != null)
        {
            return activeTask.Handle.PercentComplete;  // AsyncOperationHandle 使用 PercentComplete
        }

        if (_completedTasks.ContainsKey(scene.SceneGuid))
        {
            return 1.0f;
        }

        return -1f;
    }

    /// <summary>
    /// 取消指定场景的预加载
    /// </summary>
    public void CancelPreload(SceneReference scene)
    {
        var task = _activeTasks.FirstOrDefault(t => t.Scene.SceneGuid == scene.SceneGuid);
        if (task != null)
        {
            CancelPreload(task);
        }
    }

    private void CancelPreload(PreloadTask task)
    {
        // 注意：Addressables 不支持取消场景加载，标记后同步释放句柄避免内存泄漏
        task.IsCompleted = true;  // 标记为已完成（即使未完成）
        if (task.Handle.IsValid())
        {
            Addressables.ReleaseInstance(task.Handle);
        }
        _activeTasks.Remove(task);
    }

    private void OnPreloadCompleted(PreloadTask task)
    {
        task.IsCompleted = true;
        _activeTasks.Remove(task);

        if (task.Handle.Status == AsyncOperationStatus.Succeeded)
        {
            // **正确的预加载策略**：保留已加载的 SceneInstance 句柄（不 Unload），
            // 下次正式切换时通过 activateOnLoad=true 激活该场景，
            // 跳过重新加载，直接激活已在内存中的场景实例。
            //
            // ⚠️ 之前设计的"加载后立即 Unload"策略是错误的：
            // Addressables.UnloadSceneAsync 会从内存中彻底释放场景，
            // 下次仍然需要从磁盘重新加载，预加载毫无意义。
            //
            // 句柄由 _completedTasks 持有，当超时清理时同步 Release 句柄：
            // Addressables.ReleaseInstance(task.Handle)
            _completedTasks[task.Scene.SceneGuid] = task;
        }
        else
        {
            Debug.LogWarning($"[ScenePreloader] Preload failed for scene: {task.Scene.SceneName}, error: {task.Handle.OperationException?.Message}");
            // 不添加到 _completedTasks，允许后续重新预加载
        }

        CleanupOldCompletedTasks();
    }

    /// <summary>
    /// 清理超时的已完成任务记录，并释放对应的 Addressables 句柄
    /// </summary>
    private void CleanupOldCompletedTasks()
    {
        var timeout = Time.time - _config.preloadedTimeout;
        var toRemove = _completedTasks
            .Where(kvp => kvp.Value.StartTime < timeout)
            .Select(kvp => kvp.Key)
            .ToList();

        foreach (var key in toRemove)
        {
            if (_completedTasks.TryGetValue(key, out var task))
            {
                // P0 修复：通过 ResourceManager 释放句柄，而非直接调用 Addressables
                // 这样可以与 ADR-0019 的 ResourceManager 是唯一 Addressables 句柄管理者的原则保持一致
                // 注意：场景预加载的句柄通过 ResourceManager.UnloadScenePreload() 统一释放
                if (task.Handle.IsValid())
                {
                    ResourceManager.Instance.UnloadScenePreload(task.Handle);
                }
                Debug.Log($"[ScenePreloader] Cleaned up stale preload for scene: {key}");
            }
            _completedTasks.Remove(key);
        }
    }

}

/// <summary>
/// 预加载配置（P1 修复：从 ScenePreloader 嵌套类提取为独立顶级类）
/// 嵌套类继承 ScriptableObject 在 Unity 中不受支持（[CreateAssetMenu] 对嵌套类无效），
/// 与 ADR-0025 AudioClipCacheConfig 的 P1 修复采用相同模式。
/// </summary>
[CreateAssetMenu(menuName = "Game/Scene/PreloaderConfig")]
public class ScenePreloaderConfig : ScriptableObject
{
    [Tooltip("最大并发预加载任务数。建议值：2-4。值过大会导致：1) 内存峰值过高；2) 加载带宽竞争导致所有任务变慢。")]
    public int maxConcurrentPreloads = 2;

    [Tooltip("预加载场景在内存中保留的最长时间（秒）。超过此时间的预加载记录会被清理，场景需要重新从磁盘加载。")]
    public float preloadedTimeout = 30f;

    [Tooltip("触发预加载的距离阈值（米）。当玩家与目标场景的距离小于此值时触发预加载。")]
    public float triggerDistance = 100f;

    [Tooltip("取消预加载的距离阈值（米）。当玩家与已预加载场景的距离大于此值时，取消预加载并释放内存。")]
    public float cancelDistance = 150f;

    [Tooltip("预加载超时时间（秒）。超过此时间未正式加载的预加载任务会被取消。")]
    public float preloadTimeout = 10f;
}

// ScenePreloadRequestEvent.cs
/// <summary>
/// 场景预加载请求事件
/// 由 WorldMapSystem 在玩家接近区域边界时发布
/// </summary>
public struct ScenePreloadRequestEvent
{
    /// <summary>目标场景引用</summary>
    public SceneReference scene;

    /// <summary>预加载优先级</summary>
    public ScenePreloader.PreloadPriority priority;
}
```

> **设计说明**：场景预加载采用"加载并持有句柄，不激活"的策略：
> 1. 场景资产和场景实例被加载到内存（`activateOnLoad: false`）
> 2. `AsyncOperationHandle<SceneInstance>` 由 `_completedTasks` 持有，确保内存不被释放
> 3. 正式切换场景时，通过 `ConsumePreload()` 获取预加载的句柄并释放，避免重复加载
> 4. 超时未使用的预加载会被清理，并通过 `Addressables.ReleaseInstance` 释放句柄
>
> ⚠️ **错误模式**：加载后立即 `UnloadSceneAsync` 会彻底释放内存，下次仍需从磁盘读取，预加载意义全失。
>
> 预加载时机由 WorldMapSystem 控制，当玩家接近某个区域的边界时，
> 发布 `ScenePreloadRequestEvent` 触发预加载。

### 5.6. 预加载句柄消费机制

```csharp
// ScenePreloader.cs 补充方法
/// <summary>
/// 消费预加载：正式切换场景时调用此方法获取预加载的句柄
/// 返回预加载的 AsyncOperationHandle，如果未预加载则返回无效句柄
/// 调用方负责在场景激活完成后释放句柄（调用 Addressables.ReleaseInstance）
///
/// **使用示例**：
/// ```csharp
/// // 在 SceneTransitionController.StartTransitionSequence 中调用
/// var preloadHandle = _preloader.ConsumePreload(targetScene);
/// if (preloadHandle.IsValid())
/// {
///     // 使用预加载的句柄激活场景（跳过重新加载）
///     preloadHandle.Result.ActivateAsync();
/// }
/// else
/// {
///     // 没有预加载，正常加载
///     SceneManagerWrapper.Instance.LoadSceneAsync(targetScene, mode);
/// }
/// ```
/// </summary>
/// <param name="scene">目标场景</param>
/// <returns>预加载句柄，如果未预加载则返回无效句柄（调用方应检查 IsValid()）</returns>
public AsyncOperationHandle<SceneInstance> ConsumePreload(SceneReference scene)
{
    if (_completedTasks.TryGetValue(scene.SceneGuid, out var task))
    {
        _completedTasks.Remove(scene.SceneGuid);
        return task.Handle;
    }

    return default;  // 返回无效句柄，调用方应检查 IsValid()
}
```

> **⚠️ 重要**：如果 `SceneTransitionController` 设置了 `DontDestroyOnLoad`，则 `LoadingScreenController`
> 也必须保持同一生命周期（作为子对象或同样设置为 `DontDestroyOnLoad`），
> 否则 `LoadingScreen` UI 在场景切换后会丢失引用。

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
        // P2 修复：GetAreaData 方法名有误，正确方法为 GetAreaSceneData（见 WorldMapSystem 扩展）
        var areaData = _worldMap.GetAreaSceneData(evt.area_id);
        if (areaData?.sceneReference != null)
        {
            // 触发场景切换
            _transitionController.TransitionTo(areaData.sceneReference, SceneLoadMode.Single);
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
    [Tooltip("玩家状态重置模式：None=不重置，PositionOnly=仅重置位置，Full=完全重置")]
    public PlayerStateResetMode playerStateResetMode = PlayerStateResetMode.None;
    [Tooltip("切换区域时是否保留背包物品")]
    public bool preserveInventory = true;
}

/// <summary>
/// 玩家状态重置模式
/// </summary>
public enum PlayerStateResetMode
{
    /// <summary>不重置玩家状态</summary>
    None,
    /// <summary>仅重置玩家位置到区域入口，不重置生命值/状态等</summary>
    PositionOnly,
    /// <summary>完全重置玩家状态（位置、生命值、buff 等）</summary>
    Full
}

// WorldMapSystem.cs（扩展）
/// <summary>
/// 注意：本类为 partial class WorldMapSystem 的扩展部分。
/// WorldMapSystem 的主定义位于 World Map 系统模块（ADR-0012）。
/// 扩展方法引用主定义中的私有字段 _areaSceneDataDict，该字段由 WorldMapSystem 主类初始化。
/// </summary>
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
| **进度卡顿** | Loading Screen 更新阻塞主线程 | 使用协程轮询进度 |
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
  - 实现 GUID 序列化和 Editor GUI
  - 验证 Addressables 地址解析
- [ ] 创建 SceneManagerWrapper
  - 实现 Addressables 场景加载封装
  - 实现 Single/Additive 模式切换
  - 添加场景加载进度报告
- [ ] 创建 SceneTransitionController
  - 实现状态机（Idle/Loading/FadeOut/Activating/FadeIn）
  - 集成 CancellationToken 支持取消
  - 与 ScreenEffects 联动实现 Fade

### Phase 2: Loading Screen
- [ ] 创建 LoadingScreenController UI
  - 实现进度条滑动、文本显示
  - 实现 Tips 轮播
  - 实现淡出动画
- [ ] 集成 ScreenEffects Fade
  - 通过 EventBus 发布 ScreenEffectRequestEvent
  - 实现 FadeOut/FadeIn 时序
- [ ] 实现进度报告
  - 从 AsyncOperationHandle 获取 PercentComplete
  - 平滑进度条更新（避免跳跃）

### Phase 3: World Map 集成
- [ ] 创建 AreaSceneData ScriptableObject 配置
  - 定义场景引用、预加载资源、玩家状态重置模式
  - 配置区域场景映射表
- [ ] 实现 WorldMapSceneIntegration
  - 订阅 AreaEnteredEvent
  - 调用 SceneTransitionController.TransitionTo
  - 处理 PlayerStateResetMode
- [ ] 测试区域切换流程
  - 验证加载画面显示
  - 验证 Fade 平滑
  - 验证玩家状态重置

### Phase 4: 优化
- [ ] 实现场景预加载（ScenePreloader）
  - 实现 ScenePreloadRequestEvent 事件处理
  - 实现预加载优先级调度
  - 实现 LRU 缓存清理
- [ ] 实现资源引用计数
  - 与 Addressables 引用计数系统集成
  - 防止场景卸载后资源被意外释放
- [ ] 优化加载速度
  - 分析加载瓶颈
  - 实现依赖项预加载
  - 优化场景激活时机

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
- [ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — Loading Screen 过渡使用 Fade 效果、ScreenEffectSource 定义；**ScreenEffectRevokeEvent 支持按 sourceSystem+requesterId 精确撤销（已验证）**
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
| `SceneLoadMode` | ADR-0027 本文档 | 加载模式枚举 |
| `ScreenEffectSource` | ADR-0023 | 屏幕特效来源枚举 |
