# ADR-0019: Addressables 资源管理系统 (Addressables Asset Management System)

## Status
**Proposed**

## Date
2026-04-11

## Last Updated
2026-04-13

## Context

### Problem Statement

《断绝：罪恶之源》需要管理大量资源：场景、角色模型、纹理、音频、动画等。随着系统数量增加（15+），资源管理问题日益突出：

1. **资源引用混乱**：直接 `Resources.Load` 或场景引用导致内存管理困难
2. **加载时机不明确**：系统自行加载资源，无法统一控制生命周期
3. **内存泄漏风险**：无人知道某资源何时可以卸载
4. **远程内容支持**：DLC/热更新需要远程资源支持，但无统一方案
5. **打包大小**：PC/PS5 平台有严格的包大小限制

### Constraints

- **平台**：PC (Steam) & PS5
- **引擎**：Unity 6.3 LTS
- **包管理器**：Unity Addressables Package (1.21+)
- **网络**：需要支持远程内容更新（DLC、热修复）

### Requirements

- **必须**：统一资源加载/卸载接口
- **必须**：定义资源生命周期管理策略
- **必须**：支持远程内容分发
- **必须**：提供内存预算控制
- **必须**：与场景管理（World Map）集成

---

## Decision

### 架构决策

采用 **Addressables 资源管理系统**，提供统一的资源加载、缓存、卸载接口：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Addressables 资源管理架构                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    ResourceManager (单例)                           │   │
│  │  - 统一的资源加载/卸载入口                                          │   │
│  │  - 生命周期管理（引用计数）                                          │   │
│  │  - 内存预算控制                                                     │   │
│  │  - 远程/本地资源路由                                                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│  ┌─────────────────────────────────┼────────────────────────────────┐   │
│  ▼                                 ▼                                ▼   │
│  ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐   │
│  │  Addressables     │   │  SceneManager      │   │  RemoteContent     │   │
│  │  Provider         │   │  (场景加载)        │   │  Provider         │   │
│  │  (基础资源)        │   │                   │   │  (远程更新/DLC)    │   │
│  └───────────────────┘   └───────────────────┘   └───────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. 资源分类

| 分类 | 加载策略 | 卸载策略 | 示例 |
|------|---------|---------|------|
| **常驻 (Resident)** | 游戏启动时预加载 | 仅在游戏退出时卸载 | 核心 UI、字体、主色调色板 |
| **场景绑定 (SceneBound)** | 进入场景时加载 | 离开场景时卸载 | 场景特有的 NPC、道具 |
| **按需 (OnDemand)** | 首次访问时加载 | 引用计数归零时卸载 | 武器模型、动画片段 |
| **流式 (Streaming)** | 分块渐进加载 | 可随时卸载 | 开放世界地形 |

### 2. ResourceManager 核心

```csharp
// ResourceManager.cs
/// <summary>
/// 资源管理器 - 采用 MonoBehaviour 单例模式
/// 注意：与 EventBus（ScriptableObject 单例）不同，
/// ResourceManager 需要 Update() 生命周期且频繁访问，
/// 使用 MonoBehaviour 单例可简化生命周期管理并减少 GC 开销。
/// </summary>
public class ResourceManager : MonoBehaviour
{
    public static ResourceManager Instance { get; private set; }

    // 内存预算（可配置）
    [Header("Memory Budget")]
    public long TotalMemoryBudgetMB = 512;           // 总预算 512MB
    public long ResidentBudgetMB = 128;              // 常驻资源预算
    public long SceneBoundBudgetMB = 256;           // 场景绑定预算
    public long OnDemandBudgetMB = 128;            // 按需加载预算
    public long StreamingBudgetMB = 128;           // 流式资源预算

    private Dictionary<string, ResourceHandle> _loadedAssets = new();
    private Dictionary<string, int> _referenceCounts = new();  // 显式初始化，便于理解引用计数语义
    private long _currentMemoryUsage;

    /// <summary>
    /// 初始化 ResourceManager 单例
    /// 由 ResourceManagerBootstrap 在 Awake 中显式调用
    /// 注意：此方法仅应由 Bootstrap 调用，Awake() 不再重复调用
    /// </summary>
    public void Initialize()
    {
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    // Awake() 不再调用 Initialize()，由 Bootstrap 显式初始化
    // 这样可以确保 ResourceManager 的生命周期完全由 Bootstrap 控制
    private void Awake()
    {
        // 此 MonoBehaviour 由 Bootstrap.NewGameObject() 创建
        // 初始化由 Bootstrap.Awake() 显式调用 Initialize() 完成
    }

    private void OnDestroy()
    {
        // 停止弱引用清理协程
        if (_cleanupCoroutine != null)
            StopCoroutine(_cleanupCoroutine);

        if (Instance == this)
        {
            CleanupAllSceneHandlers();
            Instance = null;
        }
    }

    /// <summary>
    /// 生成资源的唯一键
    /// 格式："{TypeName}_{address}"
    /// </summary>
    private string GetKey<T>(string address) => $"{typeof(T).Name}_{address}";

    // 异步加载
    public async Task<T> LoadAsync<T>(string address, ResourceCategory category = ResourceCategory.OnDemand)
        where T : Object
    {
        // 检查预算
        if (!CheckBudget(category))
        {
            Debug.LogWarning($"[ResourceManager] Budget exceeded for category {category}. Unloading unused assets.");
            await UnloadUnusedAsync(category);
        }

        var key = GetKey<T>(address);

        // 已加载：增加引用计数
        if (_loadedAssets.TryGetValue(key, out var handle))
        {
            _referenceCounts[key]++;
            return handle.Asset as T;
        }

        // 异步加载
        var operation = Addressables.LoadAssetAsync<T>(address);
        var asset = await operation.Task;

        // 估算内存占用
        long estimatedSize = EstimateMemorySize(asset);
        _currentMemoryUsage += estimatedSize;

        // 注册
        _loadedAssets[key] = new ResourceHandle
        {
            Address = address,
            Asset = asset,
            Handle = operation,
            Category = category,
            EstimatedSize = estimatedSize,
            LoadTime = Time.time
        };
        _referenceCounts[key] = 1;

#if DEVELOPMENT_BUILD
        Debug.Log($"[ResourceManager] Loaded {address} ({estimatedSize / 1024}KB). Total: {_currentMemoryUsage / 1024}MB");
#endif

        // 发布资源加载完成事件
        EventBus.Instance.Publish(new AssetLoadedEvent
        {
            Address = address,
            AssetType = typeof(T).Name,
            EstimatedSizeBytes = estimatedSize
        });

        return asset;
    }

    // 释放（引用计数减一）
    public void Release(string address, Type assetType)
    {
        var key = $"{assetType.Name}_{address}";

        if (!_referenceCounts.TryGetValue(key, out var count))
            return;

        _referenceCounts[key] = count - 1;

        if (_referenceCounts[key] <= 0)
        {
            // 引用计数归零时发布事件，然后同步卸载
            // 注意：此处直接调用 UnloadInternal() 是同步行为
            // 原因：引用计数归零表示资源已无引用，延迟卸载没有意义
            //       同步卸载可以立即释放内存，避免积压待卸载资源
            //       AssetUnloadedEvent 会在卸载完成后发布，通知订阅者资源已从内存移除
            EventBus.Instance.Publish(new AssetReleaseEvent
            {
                Address = address,
                AssetType = assetType.Name
            });

            _referenceCounts.Remove(key);
            UnloadInternal(key);
        }
    }

    // Release 的类型安全重载
    public void Release<T>(string address) where T : Object
    {
        Release(address, typeof(T));
    }

    // 释放所有指定类别资源
    public async Task UnloadUnusedAsync(ResourceCategory category)
    {
        var toUnload = _loadedAssets
            .Where(kv => kv.Value.Category == category && _referenceCounts[kv.Key] <= 0)
            .Select(kv => kv.Key)
            .ToList();

        foreach (var key in toUnload)
        {
            UnloadInternal(key);
        }

        await Resources.UnloadUnusedAssets();
    }

    // 场景绑定资源自动管理（使用 WeakReference 防止内存泄漏）
    private readonly Dictionary<string, WeakReference<Action<Scene>>> _sceneUnloadHandlers = new();

    public void RegisterSceneBound(string address, AsyncOperationHandle<SceneInstance> handle)
    {
        var key = $"Scene_{address}";  // Scene 类型使用特殊前缀
        _loadedAssets[key] = new ResourceHandle
        {
            Address = address,
            Asset = null,  // Scene 实例不是 Object，无法存储
            SceneHandle = handle,
            Category = ResourceCategory.SceneBound,
            EstimatedSize = 0,  // 场景内存由 Unity 管理
            LoadTime = Time.time
        };
        _referenceCounts[key] = 1;

        // 监听场景卸载（使用 WeakReference 防止内存泄漏）
        Action<Scene> handler = _ => OnSceneUnloaded(key);
        _sceneUnloadHandlers[key] = new WeakReference<Action<Scene>>(handler);
        SceneManager.sceneUnloaded += handler;
    }

    private void OnSceneUnloaded(string key)
    {
        if (_loadedAssets.TryGetValue(key, out var handle) && handle.Category == ResourceCategory.SceneBound)
        {
            UnloadInternal(key);
        }

        // 注销委托（通过 WeakReference 查找并清理）
        CleanupSceneUnloadHandler(key);
    }

    private void CleanupSceneUnloadHandler(string key)
    {
        // 弱引用需要定期清理已失效的条目
        var keysToRemove = new List<string>();

        foreach (var kvp in _sceneUnloadHandlers)
        {
            if (!kvp.Value.TryGetTarget(out _))
            {
                keysToRemove.Add(kvp.Key);
            }
        }

        foreach (var k in keysToRemove)
        {
            _sceneUnloadHandlers.Remove(k);
        }
    }

    /// <summary>
    /// 清理所有场景卸载处理器（在 ResourceManagerBootstrap.OnDestroy 中调用）
    /// </summary>
    public void CleanupAllSceneHandlers()
    {
        foreach (var kvp in _sceneUnloadHandlers)
        {
            if (kvp.Value.TryGetTarget(out var handler))
            {
                SceneManager.sceneUnloaded -= handler;
            }
        }
        _sceneUnloadHandlers.Clear();
    }

    private void UnloadInternal(string key)
    {
        if (_loadedAssets.TryGetValue(key, out var handle))
        {
            if (handle.EstimatedSize > 0)
                _currentMemoryUsage -= handle.EstimatedSize;

            // 释放 Addressables 句柄
            if (handle.Handle.IsValid())
                Addressables.Release(handle.Handle);

            // 释放场景句柄
            if (handle.SceneHandle.IsValid())
                Addressables.Release(handle.SceneHandle);

#if DEVELOPMENT_BUILD
            Debug.Log($"[ResourceManager] Unloaded {handle.Address}. Total: {_currentMemoryUsage / 1024}MB");
#endif

            // 发布资源卸载事件（使用原始 Address，而非合成 Key）
            EventBus.Instance.Publish(new AssetUnloadedEvent { Address = handle.Address });

            _loadedAssets.Remove(key);
            _referenceCounts.Remove(key);
        }
    }

    /// <summary>
    /// 获取当前内存使用量（字节）
    /// </summary>
    public long GetCurrentMemoryUsage() => _currentMemoryUsage;

    /// <summary>
    /// 获取指定类别的当前内存使用量
    /// </summary>
    public long GetCategoryMemoryUsage(ResourceCategory category)
    {
        return _loadedAssets
            .Where(kv => kv.Value.Category == category)
            .Sum(kv => kv.Value.EstimatedSize);
    }

    /// <summary>
    /// 检查资源是否已加载
    /// </summary>
    public bool IsLoaded(string address, Type assetType)
    {
        var key = $"{assetType.Name}_{address}";
        return _loadedAssets.ContainsKey(key) && _referenceCounts.GetValueOrDefault(key, 0) > 0;
    }

    private bool CheckBudget(ResourceCategory category)
    {
        var categoryLimit = category switch
        {
            ResourceCategory.Resident => ResidentBudgetMB * 1024 * 1024,
            ResourceCategory.SceneBound => SceneBoundBudgetMB * 1024 * 1024,
            ResourceCategory.OnDemand => OnDemandBudgetMB * 1024 * 1024,
            ResourceCategory.Streaming => StreamingBudgetMB * 1024 * 1024,
            _ => OnDemandBudgetMB * 1024 * 1024
        };

        // 计算该类别当前使用量
        long categoryUsage = _loadedAssets
            .Where(kv => kv.Value.Category == category)
            .Sum(kv => kv.Value.EstimatedSize);

        return categoryUsage < categoryLimit;
    }

    // 纹理压缩格式字节计算（每像素 bytes）
    // 注意：不同平台使用不同压缩格式
    // - PC (DXT/BC): DXT1, DXT5, BC7
    // - PS5 (ASTC): ASTC_4x4, ASTC_8x8, ASTC_6x6, ASTC_8x8
    // - 通用: ETC2 (Android/Switch)
    // 注意：ASTC_8x8 使用 0.5 bytes/pixel，使用 float 存储避免截断
    private static readonly Dictionary<TextureFormat, float> _textureFormatBytes = new()
    {
        { TextureFormat.DXT1, 1f },       // 8 bits per pixel = 1 byte
        { TextureFormat.DXT5, 2f },       // 16 bits per pixel = 2 bytes
        { TextureFormat.BC7, 2f },        // 16 bits per pixel = 2 bytes
        { TextureFormat.ASTC_4x4, 2f },   // 16 bits per pixel = 2 bytes
        { TextureFormat.ASTC_6x6, 1f },   // 8 bits per pixel = 1 byte
        { TextureFormat.ASTC_8x8, 0.5f }, // 4 bits per pixel = 0.5 bytes (ASTC_8x8 压缩比)
        { TextureFormat.ETC2_RGBA8, 2f },  // 16 bits per pixel = 2 bytes
        { TextureFormat.RGBA32, 4f },     // 32 bits per pixel = 4 bytes
        { TextureFormat.RGBA16, 2f },     // 16 bits per pixel = 2 bytes
        { TextureFormat.R8, 1f },         // 8 bits per pixel = 1 byte
        { TextureFormat.RG32, 4f },       // 32 bits per pixel = 4 bytes
        { TextureFormat.RGB24, 3f },      // 24 bits per pixel = 3 bytes
    };

    // 平台默认压缩格式（使用预编译指令，在编译时确定平台）
    private static TextureFormat GetPlatformDefaultFormat()
    {
#if UNITY_PS5
        return TextureFormat.ASTC_6x6;
#elif UNITY_PS4
        return TextureFormat.DXT5;
#elif UNITY_STANDALONE
        return TextureFormat.DXT5;
#else
        return TextureFormat.DXT5;  // 未知平台默认使用 DXT5
#endif
    }

    // mipmap 因子（考虑链式 mipmap 额外占用约 1/3）
    private const float MIPMAP_FACTOR = 1.33f;

    private long EstimateMemorySize(Object asset)
    {
        if (asset is Texture texture)
        {
            return EstimateTextureSize(texture);
        }

        if (asset is GameObject go)
        {
            long total = 0;
            var processedMaterials = new HashSet<int>();

            foreach (var renderer in go.GetComponentsInChildren<Renderer>())
            {
                foreach (var mat in renderer.sharedMaterials)
                {
                    if (mat == null) continue;
                    if (!processedMaterials.Add(mat.GetInstanceID())) continue;

                    if (mat.mainTexture is Texture tex)
                        total += EstimateTextureSize(tex);
                    total += 4 * 1024; // Material overhead
                }
            }

            foreach (var animator in go.GetComponentsInChildren<Animator>())
            {
                if (animator.runtimeAnimatorController != null)
                {
                    foreach (var clip in animator.runtimeAnimatorController.animationClips)
                    {
                        total += EstimateAnimationClipSize(clip);
                    }
                }
            }

            foreach (var mf in go.GetComponentsInChildren<MeshFilter>())
            {
                if (mf.sharedMesh != null)
                    total += EstimateMeshSize(mf.sharedMesh);
            }

            return Math.Max(total, 1024 * 100);
        }

        if (asset is AnimationClip clip)
            return EstimateAnimationClipSize(clip);

        if (asset is AudioClip audio)
            return EstimateAudioSize(audio);

        if (asset is Mesh mesh)
            return EstimateMeshSize(mesh);

        return 1024 * 50;
    }

    private long EstimateTextureSize(Texture texture)
    {
        int width = texture.width;
        int height = texture.height;
        float bytesPerPixel = 4f; // 默认 RGBA

        // 检测压缩格式
        if (texture is Texture2D tex2D)
        {
            if (_textureFormatBytes.TryGetValue(tex2D.format, out var bytes))
            {
                bytesPerPixel = bytes;
            }
            else
            {
                // 未知格式，使用运行时检测的平台默认压缩格式
                var defaultFormat = GetPlatformDefaultFormat();
                bytesPerPixel = _textureFormatBytes.GetValueOrDefault(defaultFormat, 4f);
            }
        }

        // 计算实际像素数（考虑 mipmap）
        long totalPixels = 0;
        int mipWidth = width;
        int mipHeight = height;
        while (mipWidth >= 1 && mipHeight >= 1)
        {
            totalPixels += mipWidth * mipHeight;
            mipWidth >>= 1;
            mipHeight >>= 1;
        }

        // 如果没有 mipmap，使用简单估算
        if (totalPixels == width * height)
            totalPixels = (long)(width * height * MIPMAP_FACTOR);

        return (long)(totalPixels * bytesPerPixel);
    }

    private long EstimateMeshSize(Mesh mesh)
    {
        // 顶点数据：position(12) + normal(12) + uv(8) + tangent(16) ≈ 48 bytes/vertex
        long vertexSize = mesh.vertexCount * 48;

        // 索引数据
        long indexSize = mesh.triangles.Length *
            (mesh.indexFormat == IndexFormat.UInt16 ? 2 : 4);

        // 骨骼权重（如果有）
        long boneWeightSize = 0;
        if (mesh.boneWeights != null && mesh.boneWeights.Length > 0)
            boneWeightSize = mesh.boneWeights.Length * 16; // 4 weights * 4 bytes each

        return vertexSize + indexSize + boneWeightSize;
    }

    private long EstimateAnimationClipSize(AnimationClip clip)
    {
        // 每帧约 32 bytes（包含关键帧数据、曲线、事件）
        return (long)(clip.length * clip.frameRate * 32);
    }

    private long EstimateAudioSize(AudioClip audio)
    {
        // 估算：duration * sampleRate * channels * bytesPerSample
        // 假设 16-bit PCM，如果压缩则更小
        long rawSize = (long)(audio.length * audio.sampleRate * audio.channels * 2);

        // 考虑压缩格式（Vorbis/opus 通常压缩到原始大小的 10-20%）
        // 这里使用 20% 作为保守估计
        return rawSize / 5;
    }
}

// 资源句柄
public class ResourceHandle
{
    public string Address;
    public Object Asset;                        // 普通资源的 Unity Object
    public AsyncOperationHandle Handle;        // Addressables 操作句柄（用于 Release）
    public AsyncOperationHandle<SceneInstance> SceneHandle;  // 场景专用句柄
    public ResourceCategory Category;
    public long EstimatedSize;
    public float LoadTime;
}

// 资源分类
public enum ResourceCategory
{
    Resident,      // 常驻
    SceneBound,   // 场景绑定
    OnDemand,      // 按需
    Streaming      // 流式
}

// ==================== 资源加载/卸载事件 ====================
// 注意：这些事件类型应同步添加到 shared-types.md

// ==================== 事件时序说明 ====================
// 资源生命周期中，事件按以下顺序发布：
//
// 1. AssetReleaseEvent （引用计数归零）
//    - 时机：资源的引用计数从 1 变为 0 时发布
//    - 含义：资源可以被卸载，但尚未从内存移除
//    - 用途：订阅者可以执行清理操作（如取消注册监听），此时资源仍可用
//
// 2. AssetUnloadedEvent （卸载完成）
//    - 时机：资源从内存完全移除后发布
//    - 含义：资源已不可访问，内存已被释放
//    - 用途：订阅者可以确认资源已完全卸载
//
// 注意：正常 Release() 路径两者几乎同时发布（同步卸载）
//
// **设计决策说明**：将生命周期拆分为 Release 和 Unloaded 两个独立事件，支持订阅者
// 在资源真正卸载前执行清理逻辑。如果只需要一个通知，使用 AssetUnloadedEvent 即可。

/// <summary>
/// 资源加载完成事件
/// 由 ResourceManager 发布，订阅者可以响应资源加载完成
/// </summary>
public struct AssetLoadedEvent
{
    /// <summary>
    /// 资源的 Addressables 地址
    /// </summary>
    public string Address;

    /// <summary>
    /// 资源类型名称
    /// </summary>
    public string AssetType;

    /// <summary>
    /// 估算的内存占用（字节）
    /// </summary>
    public long EstimatedSizeBytes;
}

/// <summary>
/// 资源卸载完成事件
/// 由 ResourceManager 发布，订阅者可以响应资源卸载完成
/// </summary>
public struct AssetUnloadedEvent
{
    /// <summary>
    /// 被卸载资源的地址
    /// </summary>
    public string Address;
}

/// <summary>
/// 资源引用计数归零事件
/// 由 ResourceManager 发布，当资源的引用计数从 1 变为 0 时触发（表示资源可以被卸载）
/// 与 AssetUnloadedEvent 的区别：此事件在引用计数归零时发布，卸载可能稍后进行
/// </summary>
public struct AssetReleaseEvent
{
    /// <summary>
    /// 被释放资源的地址
    /// </summary>
    public string Address;

    /// <summary>
    /// 资源的原始类型名称
    /// </summary>
    public string AssetType;
}
```

> **MonoBehaviour 单例生命周期说明**：
> `ResourceManager` 继承 `MonoBehaviour`，其生命周期由 Unity 管理：
> - `Awake()` 中执行初始化，`OnDestroy()` 中执行清理
> - `DontDestroyOnLoad` 确保跨场景持久化
> - `ResourceManagerBootstrap` 确保单例在首个场景加载前完成初始化
>
> **⚠️ Script Execution Order 配置**：
> 为确保 `ResourceManagerBootstrap` 在首个场景加载前完成初始化，需在 Unity 中配置执行顺序：
> - `ResourceManagerBootstrap` 执行顺序设置为 **-100**（最早）
> - `ResourceManager` 执行顺序设置为 **-99**
> - 建议在 `Project Settings > Script Execution Order` 中添加配置

> **⚠️ SceneManager.sceneUnloaded 静态事件注意事项**：
> `SceneManager.sceneUnloaded` 是静态事件，跨场景持久化：
> - 回调方法需确保在场景切换后仍然有效
> - 使用弱引用（WeakReference）或显式注销防止内存泄漏
> - 必须在 `ResourceManager.OnDestroy` 中统一清理所有静态事件监听

### 3. ResourceManagerBootstrap（生命周期管理）

```csharp
// ResourceManagerBootstrap.cs
/// <summary>
/// ResourceManager 生命周期管理器
/// 确保 ResourceManager 在首个场景加载前完成初始化
/// </summary>
public class ResourceManagerBootstrap : MonoBehaviour
{
    private static ResourceManagerBootstrap _instance;
    private static readonly object _lock = new();

    public static ResourceManagerBootstrap Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                    {
                        var go = new GameObject("ResourceManagerBootstrap");
                        _instance = go.AddComponent<ResourceManagerBootstrap>();
                        DontDestroyOnLoad(go);
                    }
                }
            }
            return _instance;
        }
    }

    private void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }
        _instance = this;

        // 创建 ResourceManager 并初始化
        var resourceManagerGO = new GameObject("ResourceManager");
        var resourceManager = resourceManagerGO.AddComponent<ResourceManager>();
        resourceManager.Initialize();

        // 初始化远程内容管理器（如果配置存在）
        InitializeRemoteContent();
    }

    private void InitializeRemoteContent()
    {
        // 使用 Resources.Load 而非 Addressables 加载 RemoteContentConfigSO
        //
        // 设计决策说明（重要）：
        // RemoteContentConfigSO 是 ResourceManagerBootstrap 初始化时就需要读取的配置，
        // 用于决定是否启用 Addressables 远程功能。此配置本身很小（只包含 URL 和开关），
        // 且必须在 Addressables 初始化之前读取，因此使用同步的 Resources.Load。
        //
        // 为什么不通过 Addressables 加载？
        // 1. Addressables.LoadAssetAsync 是异步的，无法在 Bootstrap 阶段同步等待
        // 2. 此配置的加载时机在 Addressables 系统初始化之前（Bootstrap 负责初始化 Addressables）
        // 3. 配置内容是纯数据（URL、开关），无需热更新能力
        //
        // 如果未来需要热更新此配置，可考虑：
        // - 在首个场景加载后通过 Addressables 异步更新内存中的配置
        // - 或使用远程 JSON 配置文件（通过 UnityWebRequest 下载）
        var config = Resources.Load<RemoteContentConfigSO>("Config/RemoteContentConfig");
        if (config != null)
        {
            var remoteContentManager = new RemoteContentManager();
            remoteContentManager.Initialize(config);
        }
        else
        {
            Debug.Log("[ResourceManagerBootstrap] RemoteContentConfig not found, skipping remote content initialization.");
        }
    }

    private void OnDestroy()
    {
        if (_instance == this)
        {
            _instance = null;
        }
    }
}

/// <summary>
/// 远程内容管理器
/// 采用依赖注入模式，由 ResourceManagerBootstrap 在启动时初始化
/// 注意：此类不是单例，实例由 Bootstrap 创建并持有
/// </summary>
public class RemoteContentManager
{
    /// <summary>
    /// 默认构造函数
    /// </summary>
    public RemoteContentManager()
    {
    }

    private RemoteContentConfigSO _config;
    private string REMOTE_CATALOG_URL => _config?.remoteCatalogUrl ?? "https://cdn.severance-game.com/content/";
    private string VERSION_MANIFEST => _config?.versionManifestFileName ?? "version_manifest.json";

    // 本地缓存的远程资源版本信息
    private Dictionary<string, long> _remoteVersions = new();

    // 本地资源版本内存缓存（避免频繁文件 IO）
    private Dictionary<string, long> _localVersionCache = new();
}
```

### 4. 弱引用自动清理机制

```csharp
// ResourceManager.cs 中新增

/// <summary>
/// 协程：每 30 秒定期清理 _sceneUnloadHandlers 中已失效的弱引用条目
/// </summary>
private Coroutine _cleanupCoroutine;

private void Start()
{
    _cleanupCoroutine = StartCoroutine(CleanupWeakReferencesLoop());
}

private System.Collections.IEnumerator CleanupWeakReferencesLoop()
{
    while (true)
    {
        yield return new WaitForSeconds(30f);
        CleanupAllStaleWeakReferences(); // 清理所有已失效的弱引用（GC 已回收的条目）
    }
}

/// <summary>
/// 清理所有已失效的弱引用条目
/// 与 CleanupSceneUnloadHandler(key) 不同：此方法遍历全量，用于定期维护
/// </summary>
private void CleanupAllStaleWeakReferences()
{
    var keysToRemove = new List<string>();
    foreach (var kvp in _sceneUnloadHandlers)
    {
        if (!kvp.Value.TryGetTarget(out _))
            keysToRemove.Add(kvp.Key);
    }
    foreach (var k in keysToRemove)
        _sceneUnloadHandlers.Remove(k);
}

// OnDestroy 已在上方定义，包含 StopCoroutine + CleanupAllSceneHandlers
} // end of ResourceManager class
```

### 5. 场景加载集成

```csharp
// SceneLoadRequest.cs
/// <summary>
/// 场景加载请求结构体
/// 用于 SceneManagerWrapper.LoadScene 的参数封装
/// </summary>
public struct SceneLoadRequest
{
    /// <summary>
    /// Addressables 场景地址
    /// </summary>
    public string SceneAddress;

    /// <summary>
    /// 加载模式：Single（替换当前场景）或 Additive（叠加场景）
    /// </summary>
    public LoadSceneMode Mode;

    /// <summary>
    /// 加载完成后是否自动激活场景
    /// true = Addressables 自动激活，false = 需手动调用 ActivateAsync()
    /// </summary>
    public bool AutoActivate;

    /// <summary>
    /// 加载进度回调（每帧调用，直到加载完成）
    /// </summary>
    public Action<AsyncOperation> OnProgress;

    /// <summary>
    /// 目标场景类型（用于 LoadCompletedEvent.destination_type）
    /// </summary>
    public SceneDestinationType DestinationType;
}

/// <summary>
/// 场景目标类型枚举
/// </summary>
public enum SceneDestinationType
{
    Area,       // 区域/关卡
    City,       // 城市
    LoadingTip, // 加载提示
    Safehouse   // 安全屋
}

// SceneManagerWrapper.cs
public class SceneManagerWrapper : MonoBehaviour
{
    public static SceneManagerWrapper Instance { get; private set; }

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }

    public async Task LoadScene(SceneLoadRequest request)
    {
        // 1. 通知 UI 显示加载画面
        EventBus.Instance.Publish(new LoadingScreenRequestEvent
        {
            destination = request.SceneAddress,
            destination_type = "area",
            source_location = ""
        });

        // 2. 卸载上一个场景的资源（引用计数归零的会被自动释放）
        if (request.Mode == LoadSceneMode.Single)
        {
            await ResourceManager.Instance.UnloadUnusedAsync(ResourceCategory.SceneBound);
        }

        // 3. 异步加载新场景
        // 注意：autoActivate 参数控制是否在加载完成后自动激活场景
        // 如果 AutoActivate = false，则需要手动调用 handle.Result.ActivateAsync() 激活场景
        // Addressables.LoadScene 不接受泛型参数，返回 AsyncOperationHandle<SceneInstance>
        var handle = Addressables.LoadScene(
            request.SceneAddress,
            request.Mode,
            request.AutoActivate);

        // 4. 进度更新
        while (!handle.IsDone)
        {
            request.OnProgress?.Invoke(handle);
            await Task.Yield();
        }

        // 5. 如果未自动激活，手动激活场景
        if (!request.AutoActivate && handle.Result.Scene.IsValid())
        {
            await handle.Result.ActivateAsync();
        }

        // 6. 注册场景资源
        if (handle.Result.Scene.IsValid())
        {
            ResourceManager.Instance.RegisterSceneBound(request.SceneAddress, handle);
        }

        // 7. 通知加载完成
        EventBus.Instance.Publish(new LoadCompletedEvent
        {
            destination = request.SceneAddress,
            destination_type = request.DestinationType.ToString().ToLowerInvariant(),
            was_successful = handle.Result.Scene.IsValid()
        });
    }
}
```

### 4. 远程内容支持

```csharp
// RemoteContentConfigSO.cs
[CreateAssetMenu(fileName = "RemoteContentConfig", menuName = "Game/Resource/RemoteContentConfig")]
public class RemoteContentConfigSO : ScriptableObject
{
    [Header("Remote Content URLs")]
    public string remoteCatalogUrl = "https://cdn.severance-game.com/content/";
    public string versionManifestFileName = "version_manifest.json";

    [Header("Build Settings")]
    public bool forceLocalCatalog = false;  // 开发环境强制使用本地 Catalog

    [Header("Fallback")]
    public bool useLocalFallbackOnError = true;
}
```

> **注意**：`RemoteContentConfigSO` 是 ScriptableObject 资源文件，需放置在 `Resources/Config/` 目录下。此配置通过 `Resources.Load` 同步加载，原因见上方 `InitializeRemoteContent()` 方法的设计决策说明。

// RemoteContentManager.cs
/// <summary>
/// 远程内容管理器
/// 采用依赖注入模式，由 ResourceManagerBootstrap 在启动时初始化
/// 注意：此类不是单例，实例由 Bootstrap 创建并持有
/// </summary>
public class RemoteContentManager
{
    // 本地默认 InternalIdTransform（用于 fallback）
    private static string DefaultInternalIdTransform(OriginalUri original) => original.PrimaryKey;
    /// <summary>
    /// 默认构造函数
    /// </summary>
    public RemoteContentManager()
    {
    }

    // 远程内容配置（由 Initialize 注入）
    private RemoteContentConfigSO _config;
    private string REMOTE_CATALOG_URL => _config?.remoteCatalogUrl ?? "https://cdn.severance-game.com/content/";
    private string VERSION_MANIFEST => _config?.versionManifestFileName ?? "version_manifest.json";

    // 本地缓存的远程资源版本信息
    private Dictionary<string, long> _remoteVersions = new();

    // 本地资源版本内存缓存（避免频繁文件 IO）
    private Dictionary<string, long> _localVersionCache = new();

    // 版本清单结构
    [Serializable]
    public class VersionManifest
    {
        /// <summary>
        /// 清单格式版本（用于检测清单格式变更，不等于内容版本）
        /// 当清单格式发生结构性变更时递增，触发强制重新解析
        /// </summary>
        public long manifestVersion;

        /// <summary>
        /// 资源版本列表（使用 List 替代 Dictionary 以确保 JsonUtility 兼容性）
        /// 序列化/反序列化时与 Dictionary 互相转换
        /// </summary>
        public List<AssetVersionEntry> assetVersions = new();
    }

    /// <summary>
    /// 资源版本条目（JsonUtility 兼容的 K/V 结构）
    /// </summary>
    [Serializable]
    public struct AssetVersionEntry
    {
        public string address;
        public long version;
    }

    /// <summary>
    /// 初始化远程内容管理器
    /// 由 ResourceManagerBootstrap.Awake 调用，确保在 ResourceManager 之前初始化
    /// </summary>
    /// <param name="config">远程内容配置文件</param>
    public void Initialize(RemoteContentConfigSO config)
    {
        _config = config;

        // 开发环境强制使用本地 Catalog
        if (_config?.forceLocalCatalog == true && Application.isEditor)
        {
            Debug.Log("[RemoteContentManager] Force local catalog mode enabled");
            // 即使强制本地 Catalog，也需要设置 InternalIdTransform 指向默认行为
            // 否则 Addressables 会尝试使用上一次设置的 transform（如果有）
            Addressables.InternalIdTransform = DefaultInternalIdTransform;
            return;
        }

        // 配置 Addressables 远程目录
        Addressables.InternalIdTransform = OnInternalIdTransform;

        // 加载本地缓存的版本信息
        LoadLocalVersionManifest();
    }

    private void LoadLocalVersionManifest()
    {
        var path = Path.Combine(Application.persistentDataPath, VERSION_MANIFEST);
        if (File.Exists(path))
        {
            var json = File.ReadAllText(path);
            var manifest = JsonUtility.FromJson<VersionManifest>(json);
            if (manifest != null && manifest.assetVersions != null)
            {
                _remoteVersions = manifest.assetVersions.ToDictionary(e => e.address, e => e.version);
            }
        }
    }

    private void SaveLocalVersionManifest()
    {
        var manifest = new VersionManifest
        {
            manifestVersion = 1,
            assetVersions = _remoteVersions.Select(kv => new AssetVersionEntry { address = kv.Key, version = kv.Value }).ToList()
        };
        var path = Path.Combine(Application.persistentDataPath, VERSION_MANIFEST);
        File.WriteAllText(path, JsonUtility.ToJson(manifest));
    }

    private string OnInternalIdTransform(OriginalUri original)
    {
        // 决定使用本地还是远程
        // 本地开发环境使用本地路径
        if (Application.isEditor || _config?.forceLocalCatalog == true)
            return original.PrimaryKey;

        // 检查远程是否有更新版本
        var remoteVersion = CheckRemoteVersion(original.PrimaryKey);
        if (remoteVersion > GetLocalVersion(original.PrimaryKey))
        {
            return REMOTE_CATALOG_URL + original.PrimaryKey;
        }

        return original.PrimaryKey;
    }

    // 版本检查（Catalog 比对）
    private long CheckRemoteVersion(string assetKey)
    {
        return _remoteVersions.TryGetValue(assetKey, out var v) ? v : 0;
    }

    // 获取本地缓存的资源版本（首次为 0）
    // 使用清单文件统一存储所有资源版本，避免每个资源单独存储
    private long GetLocalVersion(string assetKey)
    {
        // 直接从版本清单获取（已加载到内存）
        if (_remoteVersions.TryGetValue(assetKey, out var version))
        {
            return version;
        }
        return 0;
    }

    private void SetLocalVersion(string assetKey, long version)
    {
        // 更新版本清单（内存）
        _remoteVersions[assetKey] = version;

        // 持久化到清单文件
        SaveLocalVersionManifest();
    }

    // 热更新检查：对比远程 Catalog 与本地缓存的版本清单
    public async Task<bool> CheckForUpdatesAsync()
    {
        const int MAX_RETRIES = 3;
        const int TIMEOUT_SECONDS = 10;

        for (int retry = 0; retry < MAX_RETRIES; retry++)
        {
            try
            {
                // 1. 加载远程版本清单
                var remoteJson = await GetRemoteTextAsync(REMOTE_CATALOG_URL + VERSION_MANIFEST, TIMEOUT_SECONDS);
                var remoteManifest = JsonUtility.FromJson<VersionManifest>(remoteJson);

                if (remoteManifest == null)
                    return false;

                // 2. 如果远程清单版本 <= 本地清单版本，跳过更新检查
                // manifestVersion 用于检测清单格式变更，不用于内容版本比对
                if (remoteManifest.manifestVersion <= 1)
                {
                    Debug.Log("[RemoteContentManager] Remote manifest version not newer, skipping update check.");
                    return false;
                }

                bool hasUpdate = false;

                // 3. 遍历远程清单，与本地版本比对
                foreach (var entry in remoteManifest.assetVersions)
                {
                    var localVersion = GetLocalVersion(entry.address);
                    if (entry.version > localVersion)
                    {
                        _remoteVersions[entry.address] = entry.version;
                        hasUpdate = true;
                    }
                }

                if (hasUpdate)
                {
                    SaveLocalVersionManifest();
                }

                return hasUpdate;
            }
            catch (Exception ex)
            {
                Debug.LogWarning($"[RemoteContentManager] Update check attempt {retry + 1}/{MAX_RETRIES} failed: {ex.Message}");
                if (retry < MAX_RETRIES - 1)
                {
                    // 指数退避重试：1s, 2s, 4s
                    await Task.Delay((int)Math.Pow(2, retry) * 1000);
                }
            }
        }

        Debug.LogWarning("[RemoteContentManager] All update check attempts failed. Proceeding with local content.");
        return false;
    }

    private async Task<string> GetRemoteTextAsync(string url, int timeoutSeconds = 10)
    {
        using var client = new UnityWebRequest(url);
        client.downloadHandler = new DownloadHandlerBuffer();
        client.timeout = timeoutSeconds;

        var asyncOp = client.SendWebRequest();

        // UnityWebRequestAsyncOperation 没有 WaitAsync()
        // 使用 TaskCompletionSource 将 completed 回调桥接为 Task
        var tcs = new TaskCompletionSource<bool>();
        asyncOp.completed += _ => tcs.TrySetResult(true);  // 使用 TrySetResult 防止重复设置

        var timeoutTask = Task.Delay(TimeSpan.FromSeconds(timeoutSeconds));

        // 等待请求完成或超时，取先完成者
        await Task.WhenAny(tcs.Task, timeoutTask);

        // 清理 timeoutTask（避免 Task.Delay 内部资源泄漏）
        timeoutTask.Dispose();

        if (!asyncOp.isDone)
        {
            client.Abort();
            throw new TimeoutException($"Request to {url} timed out after {timeoutSeconds} seconds");
        }

        if (client.result != UnityWebRequest.Result.Success)
        {
            throw new Exception($"Request failed: {client.error}");
        }

        return client.downloadHandler.text;
    }
}
```

### 5. 资源标签规范

所有 Addressables 资源必须遵循以下标签规范（使用小写蛇形命名）：

| 标签 | 用途 | 示例 |
|------|------|------|
| `characters/[name]` | 角色资源 | `characters/npc_goon_a` |
| `weapons/[type]` | 武器资源 | `weapons/pistol_01` |
| `environments/[area]` | 环境资源 | `environments/warehouse_block_a` |
| `ui/[type]` | UI 资源 | `ui/hud_main` |
| `audio/[category]/[name]` | 音频资源 | `audio/sfx/explosion_01` |
| `animations/[type]/[name]` | 动画资源 | `animations/takedowns/env_01` |

> **命名规范**：
> - 所有标签路径使用小写蛇形命名（lowercase_snake_case）
> - 复数形式用于分类标签（characters, weapons, environments, audio, animations）
> - 示例中的 `pistol_01`、`env_01` 等使用序号便于版本迭代

> **⚠️ Addressables Groups 配置说明**：
> 在 Unity Editor 中配置 Addressables Groups 时需注意：
> - **Local vs Remote**：本地资源打包到 StreamingAssets，远程资源上传到 CDN
> - **Build Path / Load Path**：需在 Addressables Groups 窗口中正确配置
> - **Editor 调试**：开发环境下可使用 `Application.isEditor` 判断，使用本地路径
> - **PS5 平台**：需在 Player Settings 中配置远程目录白名单
> - 所有标签路径使用小写蛇形命名（lowercase_snake_case）
> - 复数形式用于分类标签（characters, weapons, environments, audio, animations）
> - 示例中的 `pistol_01`、`env_01` 等使用序号便于版本迭代

### 6. 内存预算配置

```csharp
// ResourceBudgetConfigSO.cs
[CreateAssetMenu(fileName = "ResourceBudgetConfig", menuName = "Game/Resource/BudgetConfig")]
public class ResourceBudgetConfigSO : ScriptableObject
{
    [Header("Total Budget")]
    public long TotalMemoryBudgetMB = 512;

    [Header("Category Budgets")]
    public long ResidentBudgetMB = 128;
    public long SceneBoundBudgetMB = 256;
    public long OnDemandBudgetMB = 128;
    public long StreamingBudgetMB = 128;  // Streaming 独立预算（Phase 5 完成）

    [Header("Streaming Budgets")]
    public long TerrainBudgetMB = 256;
    public long TextureBudgetMB = 384;
}
```

---

## Alternatives Considered

### Alternative 1: Unity Resources 系统

- **描述**：使用 Unity 内置 `Resources.Load`
- **Pros**：简单，Unity 原生
- **Cons**：
  - 无法热更新
  - 无法单独卸载
  - 所有资源打包在一个二进制中
  - 无法控制加载时机
- **拒绝理由**：不支持远程内容分发和精细内存管理

### Alternative 2: 自定义 Resource System

- **描述**：自己实现资源加载/缓存/卸载
- **Pros**：完全可控
- **Cons**：
  - 重复造轮子
  - 需要处理平台差异（PC/PS5）
  - 维护成本高
- **拒绝理由**：Addressables 已经解决了这些问题

### Alternative 3: Asset Bundles

- **描述**：使用 Unity Asset Bundles（Addressables 的前身）
- **Pros**：成熟稳定
- **Cons**：
  - 需要手动管理依赖
  - 无内置内存管理
  - 热更新支持弱
- **拒绝理由**：Addressables 是 Asset Bundles 的升级版，提供更完整的解决方案

---

## Consequences

### Positive

- **统一接口**：所有系统通过 ResourceManager 加载资源
- **内存可控**：预算控制防止内存溢出
- **生命周期明确**：引用计数确保资源不会过早/过晚卸载
- **远程支持**：Addressables 内置 CDN 支持
- **依赖管理**：Addressables 自动处理资源依赖

### Negative

- **学习曲线**：团队需要学习 Addressables 工作流
- **构建时间**：Addressables catalog 生成增加构建时间
- **调试复杂性**：异步加载增加了调试难度

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **内存溢出** | 资源估算不准确导致超预算 | 保守估算，定期压测 |
| **引用泄漏** | Release 未被调用导致资源无法卸载 | 代码审查检查 Load/Release 配对 |
| **加载阻塞** | 同步加载导致卡顿 | 强制异步加载，UI 显示进度 |
| **远程失败** | CDN 不可用时资源加载失败 | Fallback 到本地资源 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **加载速度** | 取决于资源大小和网络 | 异步加载不阻塞主线程 |
| **内存峰值** | 约等于 TotalMemoryBudgetMB | 预算控制防止超支 |
| **GC** | 资源卸载时触发 | 对象池减少 GC |

---

## Migration Plan

### Phase 1: 基础设施
- [ ] 配置 Addressables Groups（按资源类型分组）
- [ ] 实现 ResourceManager 单例
- [ ] 实现引用计数系统

### Phase 2: 场景集成
- [ ] 实现 SceneManagerWrapper
- [ ] 与 World Map 系统集成
- [ ] 实现 Loading Screen 进度回调

### Phase 3: 远程支持
- [ ] 配置远程 Catalog
- [ ] 实现 RemoteContentManager
- [ ] 实现版本检查和热更新

### Phase 4: 迁移现有代码
- [ ] 审查所有 `Resources.Load` 调用
- [ ] 迁移到 ResourceManager
- [ ] 添加 Addressables 标签

### Phase 5: Streaming 独立预算
- [x] 将 `ResourceCategory.Streaming` 从复用 `OnDemandBudgetMB` 改为独立预算字段
- [x] 在 `ResourceBudgetConfigSO` 中补充 `StreamingBudgetMB` 配置项
- [x] 在 `CheckBudget()` 中对应 `Streaming` 分支使用新字段

---

## Validation Criteria

1. **加载成功**：所有通过 ResourceManager 加载的资源能正确使用
2. **卸载正确**：引用计数归零后资源被正确卸载
3. **预算控制**：内存使用不超过配置预算
4. **场景切换**：场景加载/卸载正确管理资源生命周期
5. **远程加载**：CDN 可用时正确加载远程资源
6. **Fallback**：CDN 不可用时正确 Fallback 到本地资源

---

## Related Decisions

- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — Infrastructure Layer 的核心系统
- [ADR-0005: 存档/持久化架构](./adr-0005-save-persistence-architecture.md) — 存档系统依赖 ResourceManager 加载资源
- [ADR-0012: 世界地图与非线性叙事系统](./adr-0012-world-map-nonlinear-progression.md) — 场景加载依赖 ResourceManager
