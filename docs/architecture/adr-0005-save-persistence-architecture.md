# ADR-0005: 存档/持久化架构 (Save & Persistence Architecture)

## Status
**Accepted**

## Date
2026-04-09

## Last Updated
2026-04-15 (v3 — QueryBus 取消修复：BuildSaveData 改为事件订阅模式) [已修复]

## Context

### Problem Statement

《断绝：罪恶之源》的世界地图与非线性叙事系统（World Map & Non-Linear Progression）定义了存档数据的结构和业务逻辑，但存档系统的**技术架构**尚未明确。存档系统是游戏的核心基础设施——所有进度都依赖它持久化，一旦出问题将导致玩家数据丢失。

需要确定的关键决策：
1. **存档格式**：明文 JSON、二进制、还是加密格式？
2. **存档时机**：自动存档触发点、手动存档、还是两者都有？
3. **存储位置**：本地存储、云存储（Steam Cloud / PSN）、还是两者都支持？
4. **版本迁移**：游戏更新后旧存档如何兼容？
5. **加密方案**：是否需要加密？用什么方案？

### Constraints

- **平台目标**：PC (Steam) & PS5，需要同时支持 Steam Cloud 和 PSN Cloud Save
- **数据安全**：玩家进度是核心资产，存档损坏或丢失将严重损害体验
- **性能约束**：存档操作不能导致游戏卡顿（异步保存）
- **离线支持**：需要支持完全离线游戏（PS5 可能有网络要求）
- **法规要求**：PS5 平台有强制性的云存档要求

### Requirements

- **必须**：定义存档数据结构（World Save / Player Save / System Save）
- **必须**：定义存档存储位置和同步策略
- **必须**：定义存档时机和触发条件
- **必须**：定义版本迁移策略
- **必须**：定义存档加密方案
- **应该**：支持多槽位存档（至少 3 个槽位）

---

## Decision

### 架构决策

采用**分层存档架构**，将存档分为三个独立域：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        分层存档架构                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     SaveManager (单例)                            │   │
│  │  - 统一的存档入口/出口                                            │   │
│  │  - 协调 WorldSave / PlayerSave / SystemSave 的保存顺序            │   │
│  │  - 处理存档验证和错误恢复                                          │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│          ┌─────────────────────────┼─────────────────────────┐          │
│          ▼                         ▼                         ▼          │
│  ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐   │
│  │   WorldSave       │   │   PlayerSave      │   │   SystemSave      │   │
│  │                   │   │                   │   │                   │   │
│  │ - 地区探索状态     │   │ - 玩家属性        │   │ - 设置选项        │   │
│  │ - 城市状态        │   │ - 武器/道具       │   │ - 键位映射        │   │
│  │ - 揭示地点        │   │ - 理智/愤怒值     │   │ - 成就进度        │   │
│  │ - 当前进度位置    │   │ - 线索收集进度    │   │ - 统计信息        │   │
│  │                   │   │ - NPC 关系        │   │                   │   │
│  └───────────────────┘   └───────────────────┘   └───────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     IStorageAdapter                               │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │   │
│  │  │ LocalStorage    │  │ SteamCloud     │  │ PSNCloud       │     │   │
│  │  │ (PC 本地)       │  │ (Steam)        │  │ (PS5)          │     │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘     │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. 存档数据结构

```csharp
// SaveData.cs
[Serializable]
public class SaveData
{
    public string saveVersion;        // 存档格式版本 "1.0.0"
    public long timestamp;           // Unix 时间戳
    public int playtimeSeconds;      // 累计游玩时间
    public string slotId;            // 槽位 ID "slot_0" / "slot_1" / "slot_2"

    public WorldSave world;
    public PlayerSave player;
    public SystemSave system;
}

// WorldSave.cs - 世界探索状态
[Serializable]
public class WorldSave
{
    public Dictionary<string, CitySave> cities;
    public Dictionary<string, AreaSave> areas;
    public string currentLocationId;
    public HashSet<string> revealedLocations; // 揭示的隐藏地点
    public int gameDayCount;            // 游戏天数（用于精英敌人重生判定）
}

/// <summary>
/// 【P1-4 修复】WorldSave 与 WorldSnapshot 命名映射说明
///
/// WorldSave（存档系统）使用不同的字段名来存储相同的数据结构：
/// - WorldSave.cities ↔ WorldSnapshot.CityStates（城市状态字典）
/// - WorldSave.areas ↔ WorldSnapshot.AreaStates（地区状态字典）
/// - WorldSave.revealedLocations ↔ WorldSnapshot.RevealedLocations
/// - WorldSave.gameDayCount ↔ WorldSnapshot.GameDayCount
///
/// 在重连恢复时，需要进行字段映射转换。以下是显式转换方法：
/// </summary>

// WorldSave 与 WorldSnapshot 转换扩展方法
public static class WorldSaveExtensions
{
    /// <summary>
    /// WorldSave → WorldSnapshot 转换（用于存档保存时构建快照）
    /// </summary>
    public static WorldSnapshot ToWorldSnapshot(this WorldSave worldSave)
    {
        return new WorldSnapshot
        {
            AreaId = worldSave.currentLocationId,
            AreaStates = worldSave.areas,
            CityStates = worldSave.cities,
            RevealedLocations = worldSave.revealedLocations,
            GameDayCount = worldSave.gameDayCount
        };
    }

    /// <summary>
    /// WorldSnapshot → WorldSave 转换（用于快照恢复存档）
    /// </summary>
    public static WorldSave ToWorldSave(this WorldSnapshot snapshot)
    {
        return new WorldSave
        {
            currentLocationId = snapshot.AreaId,
            areas = snapshot.AreaStates,
            cities = snapshot.CityStates,
            revealedLocations = snapshot.RevealedLocations,
            gameDayCount = snapshot.GameDayCount
        };
    }
}

// AreaSave.cs - 地区状态
[Serializable]
public class AreaSave
{
    public string areaId;
    public ExplorationState explorationState; // UNEXPLORED / EXPLORED / COMPLETED / CLEARED
    public bool isUnlocked;
    public long lastClearedTimestamp;    // 上次清除时间（Unix）
}

// PlayerSave.cs - 玩家进度
[Serializable]
public class PlayerSave
{
    public string playerId;
    // 注意：health 不是数值型 HP，而是 HealthState 枚举
    // 根据 Health System GDD，本游戏采用状态机模型（Healthy/Staggered/Downed/Dead）
    // 玩家存档只需保存玩家当前所在的地区 ID，health 状态由 Health System 管理
    // 存档加载时根据存档的 area_id 重新初始化 Health State
    public HealthState healthState;       // 当前健康状态
    public int sanity;                    // 当前理智值 (0-100)
    public int rage;                     // 当前愤怒值 (0-100)

    public List<string> collectedClues;   // 已收集线索 ID 列表
    public List<string> acquiredKnowledge;// 已获取知识 ID 列表

    public Dictionary<string, int> npcRelationships; // NPC ID → 好感度
    public Dictionary<string, int> inventory;         // 道具 ID → 数量
    public List<string> unlockedWeapons;               // 已解锁武器 ID 列表

    // 位置线索映射（clue_id → location_id）由 Clue System 维护
    public Dictionary<string, string> clueLocationMap;
}

// SystemSave.cs - 系统设置
[Serializable]
public class SystemSave
{
    public float masterVolume;
    public float musicVolume;
    public float sfxVolume;
    public float dialogueVolume;

    public bool subtitlesEnabled;
    public bool hapticFeedbackEnabled;
    public string controlScheme; // "keyboard_mouse" / "gamepad"

    public Dictionary<string, bool> achievements; // 已解锁成就
    public GameStatistics statistics;              // 统计信息
}

// 存档槽位元数据（不随存档数据存储，用于 UI 显示）
[Serializable]
public class SaveSlotMetadata
{
    public string slotId;
    public string displayName;        // 玩家自定义名称
    public long lastPlayedTimestamp;  // 最后游玩时间
    public int playtimeSeconds;
    public string thumbnailPath;      // 缩略图路径
    public bool isEmpty;
}

// 元数据存储位置：%APPDATA%/Severance/slot_metadata.json
// （独立于存档文件，用于 UI 快速加载槽位列表）
public class SaveSlotMetadataManager
{
    private const string METADATA_FILE = "slot_metadata.json";

    public static List<SaveSlotMetadata> LoadAllSlots()
    {
        var path = GetMetadataPath();
        if (!File.Exists(path)) return new List<SaveSlotMetadata>();

        var json = File.ReadAllText(path);
        return JsonUtility.FromJson<SaveSlotMetadataList>(json).slots;
    }

    public static void SaveAllSlots(List<SaveSlotMetadata> slots)
    {
        var dir = Path.GetDirectoryName(GetMetadataPath());
        Directory.CreateDirectory(dir);

        var json = JsonUtility.ToJson(new SaveSlotMetadataList { slots = slots });
        File.WriteAllText(GetMetadataPath(), json);
    }

    private static string GetMetadataPath()
    {
        return Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
            "Severance",
            METADATA_FILE
        );
    }

    [Serializable]
    private class SaveSlotMetadataList
    {
        public List<SaveSlotMetadata> slots;
    }
}
```

### 2. 存档格式

采用 **JSON + 二进制头部 + AES-256 加密**的混合方案：

```
┌─────────────────────────────────────────────────────────────┐
│                      Save File Format                        │
├─────────────────────────────────────────────────────────────┤
│  [Header: 32 bytes]                                        │
│  - Magic: "SAVE" (4 bytes)                                 │
│  - Version: uint16 (2 bytes)                               │
│  - Checksum: CRC32 (4 bytes)                               │
│  - EncryptedSize: uint32 (4 bytes)                         │
│  - Timestamp: int64 (8 bytes)                              │
│  - Reserved: (10 bytes)                                    │
├─────────────────────────────────────────────────────────────┤
│  [Encrypted Body]                                           │
│  - JSON 序列化后的 SaveData                                 │
│  - 使用 AES-256-GCM 加密                                    │
│  - 加密密钥由平台存储（Steam: SteamEncrypted$, PSN: PSN API）│
└─────────────────────────────────────────────────────────────┘
```

```csharp
// SaveFormat.cs
public static class SaveFormat
{
    public const string MAGIC = "SAVE";
    public const ushort CURRENT_VERSION = 1;
    public const int HEADER_SIZE = 32;

    public struct Header
    {
        public ushort Version;
        public uint Checksum;
        public uint EncryptedSize;
        public long Timestamp;
    }

    // 序列化
    public static byte[] Serialize(SaveData data)
    {
        var json = JsonUtility.ToJson(data);
        var jsonBytes = Encoding.UTF8.GetBytes(json);

        var header = new Header
        {
            Version = CURRENT_VERSION,
            Checksum = CRC32.Calculate(jsonBytes),
            EncryptedSize = (uint)jsonBytes.Length,
            Timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds()
        };

        var encrypted = AES256.Encrypt(jsonBytes, GetEncryptionKey());
        return Combine(SerializeHeader(header), encrypted);
    }

    // 反序列化
    public static SaveData Deserialize(byte[] bytes)
    {
        var header = DeserializeHeader(bytes);

        // 版本检查
        if (header.Version > CURRENT_VERSION)
            throw new SaveCorruptedException("Save version too new");

        var encrypted = bytes.Skip(HEADER_SIZE).ToArray();
        var decrypted = AES256.Decrypt(encrypted, GetEncryptionKey());

        var json = Encoding.UTF8.GetString(decrypted);
        return JsonUtility.FromJson<SaveData>(json);
    }
}
```

### 3. 存储适配器架构

```csharp
// IStorageAdapter.cs
public interface IStorageAdapter
{
    bool IsCloudAvailable { get; }
    Task<bool> ExistsAsync(string slotId);
    Task<byte[]> LoadAsync(string slotId);
    Task SaveAsync(string slotId, byte[] data);
    Task DeleteAsync(string slotId);
    Task<List<string>> ListSlotsAsync();
}

// LocalStorageAdapter.cs (PC 本地)
public class LocalStorageAdapter : IStorageAdapter
{
    private readonly string _basePath;

    public bool IsCloudAvailable => false;

    public async Task<byte[]> LoadAsync(string slotId)
    {
        var path = GetSavePath(slotId);
        return await File.ReadAllBytesAsync(path);
    }

    public async Task SaveAsync(string slotId, byte[] data)
    {
        var path = GetSavePath(slotId);
        var directory = Path.GetDirectoryName(path);
        Directory.CreateDirectory(directory);
        await File.WriteAllBytesAsync(path, data);
    }
}

// SteamCloudAdapter.cs (Steam)
// TODO: 需要 Steamworks SDK 实现，以下为占位桩代码
public class SteamCloudAdapter : IStorageAdapter
{
    public bool IsCloudAvailable => SteamManager.IsInitialized;

    public async Task<byte[]> LoadAsync(string slotId)
    {
        // TODO: await SteamCloud.LoadAsync($"save_{slotId}.bin");
        throw new NotImplementedException("SteamCloudAdapter.LoadAsync requires Steamworks SDK integration");
    }

    public async Task SaveAsync(string slotId, byte[] data)
    {
        // TODO: await SteamCloud.SaveAsync($"save_{slotId}.bin", data);
        throw new NotImplementedException("SteamCloudAdapter.SaveAsync requires Steamworks SDK integration");
    }
}

// PSNCloudAdapter.cs (PS5)
// TODO: 需要 PSN SDK 实现，以下为占位桩代码
public class PSNCloudAdapter : IStorageAdapter
{
    public bool IsCloudAvailable => PSNSystem.IsSignedIn;

    public async Task<byte[]> LoadAsync(string slotId)
    {
        // TODO: await PSNSystem.LoadAsync($"save_{slotId}.bin");
        throw new NotImplementedException("PSNCloudAdapter.LoadAsync requires PSN SDK integration");
    }

    public async Task SaveAsync(string slotId, byte[] data)
    {
        // TODO: await PSNSystem.SaveAsync($"save_{slotId}.bin", data);
        throw new NotImplementedException("PSNCloudAdapter.SaveAsync requires PSN SDK integration");
    }
}
```

### 4. SaveManager 核心

```csharp
// SaveManager.cs
public class SaveManager : MonoBehaviour
{
    public static SaveManager Instance { get; private set; }

    private IStorageAdapter _primaryStorage;
    private IStorageAdapter _backupStorage;

    // 并发控制：防止同时多次保存导致数据竞争
    // 使用 SemaphoreSlim 而非 Mutex，确保协程中可以异步等待
    private SemaphoreSlim _saveLock = new(1, 1);

    // 存档槽位
    private const int MAX_SLOTS = 3;
    private SaveSlotMetadata[] _slotMetadatas;

    // 自动存档触发时机
    private enum AutoSaveTrigger
    {
        AreaEntered,        // 进入地区时
        AreaExtracted,      // 撤离成功时
        ClueCollected,      // 收集到线索时
        GameDayAdvanced     // 游戏天数推进时（每 1 分钟）
        // 注意：OnApplicationQuit 是 Unity 生命周期回调，不通过 Event Bus 触发
    }

    private void Awake()
    {
        Instance = this;

        // 根据平台选择存储适配器
        _primaryStorage = PlatformSelect();
        _backupStorage = new LocalStorageAdapter(); // 始终本地备份
    }

    // 异步保存（不阻塞游戏）
    public async Task SaveAsync(string slotId, bool isManual = true)
    {
        // 并发控制：等待之前的保存完成
        await _saveLock.WaitAsync();
        try
        {
            var saveData = BuildSaveData();

            var bytes = SaveFormat.Serialize(saveData);

            // 同时写入主存储和本地备份
            await _primaryStorage.SaveAsync(slotId, bytes);
            await _backupStorage.SaveAsync(slotId, bytes); // 本地备份

            // 更新槽位元数据
            await UpdateSlotMetadata(slotId, isManual ? "manual" : "auto");

            EventBus.Instance.Publish(new SaveCompletedEvent(slotId, isManual));
        }
        finally
        {
            _saveLock.Release();
        }
    }

    // 存档续连专用检查点保存（供 Network 系统调用）
    // 注意：此方法使用 "reconnect_checkpoint" 专用槽位，与 SaveCheckpointAsync 配合使用
    // ReconnectCheckpoint.SaveSlotId 预留给其他存档操作（如完整存档快照）使用，
    // 当前断线重连检查点使用独立槽位，不依赖 SaveSlotId 字段
    public async Task SaveCheckpointAsync()
    {
        // 使用专用槽位存储重连检查点
        // SaveSlotId 字段在 Network 层的 ReconnectCheckpoint 中预留，
        // 用于未来可能的"从完整存档恢复"场景
        await SaveAsync("reconnect_checkpoint", isManual: false);
    }

    // 加载重连检查点（供 Network 系统调用）
    public async Task<SaveData> LoadCheckpointAsync()
    {
        return await LoadAsync("reconnect_checkpoint");
    }

    // Unity 生命周期回调 - 离开游戏时强制存档
    // 使用 StartCoroutine 而非 async void，确保协程在 OnApplicationQuit 期间完成执行
    // 注意：OnApplicationQuit 的等待时间有限，协程必须在数秒内完成
    private void OnApplicationQuit()
    {
        StartCoroutine(SaveOnQuitCoroutine());
    }

    private IEnumerator SaveOnQuitCoroutine()
    {
        // 等待一小段时间让其他 OnApplicationQuit 处理程序完成
        yield return new WaitForSeconds(0.5f);

        string targetSlot = GetCurrentActiveSlot();

        // 使用 _saveLock 确保不与正在进行的保存操作冲突
        yield return _saveLock.WaitAsync();
        try
        {
            var saveData = BuildSaveData();
            var bytes = SaveFormat.Serialize(saveData);

            // 写入主存储和备份
            var primaryTask = _primaryStorage.SaveAsync(targetSlot, bytes);
            var backupTask = _backupStorage.SaveAsync(targetSlot, bytes);

            yield return new WaitUntil(() => primaryTask.IsCompleted && backupTask.IsCompleted);

            Debug.Log($"[SaveManager] Auto-saved on application quit to slot {targetSlot}.");
        }
        finally
        {
            _saveLock.Release();
        }
    }

    // 获取当前活跃槽位（临时实现，待完善多槽位管理）
    private string GetCurrentActiveSlot()
    {
        // TODO：实现槽位追踪，返回当前玩家正在使用的槽位
        // 临时返回 slot_0，建议实现 SaveSlotManager.TrackActiveSlot()
        return "slot_0";
    }

    // 异步加载
    public async Task<SaveData> LoadAsync(string slotId)
    {
        try
        {
            // 优先从主存储加载，失败则从本地备份恢复
            byte[] bytes;
            if (await _primaryStorage.ExistsAsync(slotId))
            {
                bytes = await _primaryStorage.LoadAsync(slotId);
            }
            else if (await _backupStorage.ExistsAsync(slotId))
            {
                bytes = await _backupStorage.LoadAsync(slotId);
                // 从备份恢复
                await _primaryStorage.SaveAsync(slotId, bytes);
            }
            else
            {
                throw new SaveNotFoundException(slotId);
            }

            var saveData = SaveFormat.Deserialize(bytes);

            // 版本迁移
            if (saveData.saveVersion != SaveFormat.CURRENT_VERSION.ToString())
            {
                saveData = MigrateSaveData(saveData);
            }

            return saveData;
        }
        catch (Exception ex)
        {
            Debug.LogError($"Failed to load save: {ex.Message}");
            throw;
        }
    }

    // 自动存档（异步，不阻塞）
    public async void TriggerAutoSave(AutoSaveTrigger trigger)
    {
        // 特定触发点才存档
        if (!ShouldAutoSave(trigger)) return;

        // 使用 "quicksave" 槽位
        await SaveAsync("quicksave", isManual: false);
    }

    private bool ShouldAutoSave(AutoSaveTrigger trigger)
    {
        // 避免存档过于频繁
        if (_lastAutoSaveTime > DateTime.Now.AddSeconds(-30)) return false;

        return trigger switch
        {
            AutoSaveTrigger.AreaExtracted => true,  // 撤离成功必须存档
            AutoSaveTrigger.ClueCollected => true,   // 收集线索必须存档
            AutoSaveTrigger.AreaEntered => true,     // 进入地区存档
            AutoSaveTrigger.GameDayAdvanced => true,  // 游戏天数推进存档
            _ => false
        };
    }

    private SaveData BuildSaveData()
    {
        // 事件订阅模式：使用各系统缓存的存档数据
        // SaveManager 订阅各系统发布的状态变更事件，在回调中更新缓存
        // BuildSaveData 直接使用缓存的数据构建存档
        //
        // QueryBus 取消说明（2026-04-15）：
        // 原 QueryBus 模式已取消，详见 ADR-0018 §QueryBus 同步查询模式已取消。
        //
        // 新模式说明：
        // - WorldMap System 发布 WorldStateChangedEvent → SaveManager 缓存 worldSaveData
        // - PlayerController 发布 PlayerStateChangedEvent → SaveManager 缓存 playerSaveData
        // - Settings System 发布 SystemSettingsChangedEvent → SaveManager 缓存 systemSaveData
        // - 或定义 QuerySaveDataEvent，各系统订阅后响应自己的存档数据片段

        return new SaveData
        {
            saveVersion = SaveFormat.CURRENT_VERSION.ToString(),
            timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds(),
            playtimeSeconds = (int)GameTimer.TotalSeconds,
            world = _cachedWorldSave,
            player = _cachedPlayerSave,
            system = _cachedSystemSave
        };
    }

    // ========== 存档数据缓存（事件订阅模式）==========
    // 注意：以下缓存机制替代了原 QueryBus 模式

    private WorldSave _cachedWorldSave;
    private PlayerSave _cachedPlayerSave;
    private SystemSave _cachedSystemSave;

    /// <summary>
    /// 初始化存档数据事件订阅（在 SaveManager.Awake 中调用）
    /// </summary>
    public void InitializeSaveEventSubscriptions()
    {
        // WorldMap System 订阅：地区探索状态、揭示地点等
        EventBus.Instance.Subscribe<WorldStateChangedEvent>(OnWorldStateChanged);

        // PlayerController 订阅：玩家属性、道具、理智/愤怒值等
        EventBus.Instance.Subscribe<PlayerStateChangedEvent>(OnPlayerStateChanged);

        // Settings System 订阅：音量、键位等系统设置
        EventBus.Instance.Subscribe<SystemSettingsChangedEvent>(OnSystemSettingsChanged);

        // ClueJournal 订阅：线索收集进度
        EventBus.Instance.Subscribe<ClueDiscoveredEvent>(OnClueDiscovered);
    }

    private void OnWorldStateChanged(WorldStateChangedEvent evt)
    {
        // 缓存世界探索状态
        _cachedWorldSave = new WorldSave
        {
            currentLocationId = evt.currentLocationId,
            revealedLocations = evt.revealedLocations,
            gameDayCount = evt.gameDayCount,
            // cities 和 areas 需要从 WorldMap System 的内部状态获取
            // 通过事件携带完整数据或从 AreaLightingTable 等配置表查询
        };
    }

    private void OnPlayerStateChanged(PlayerStateChangedEvent evt)
    {
        // 缓存玩家状态
        _cachedPlayerSave = new PlayerSave
        {
            healthState = evt.healthState,
            sanity = evt.sanity,
            rage = evt.rage,
            inventory = evt.inventory,
            unlockedWeapons = evt.unlockedWeapons,
            npcRelationships = evt.npcRelationships,
            collectedClues = evt.collectedClues,
            acquiredKnowledge = evt.acquiredKnowledge,
            clueLocationMap = evt.clueLocationMap
        };
    }

    private void OnSystemSettingsChanged(SystemSettingsChangedEvent evt)
    {
        // 缓存系统设置
        _cachedSystemSave = new SystemSave
        {
            masterVolume = evt.masterVolume,
            musicVolume = evt.musicVolume,
            sfxVolume = evt.sfxVolume,
            dialogueVolume = evt.dialogueVolume,
            subtitlesEnabled = evt.subtitlesEnabled,
            hapticFeedbackEnabled = evt.hapticFeedbackEnabled,
            controlScheme = evt.controlScheme,
            achievements = evt.achievements,
            statistics = evt.statistics
        };
    }

    private void OnClueDiscovered(ClueDiscoveredEvent evt)
    {
        // 更新线索收集进度
        if (_cachedPlayerSave.collectedClues == null)
            _cachedPlayerSave.collectedClues = new List<string>();

        if (!_cachedPlayerSave.collectedClues.Contains(evt.clue_id))
            _cachedPlayerSave.collectedClues.Add(evt.clue_id);
    }

    /// <summary>
    /// 清理事件订阅（在 SaveManager.OnDestroy 中调用）
    /// </summary>
    public void CleanupSaveEventSubscriptions()
    {
        EventBus.Instance.Unsubscribe<WorldStateChangedEvent>(OnWorldStateChanged);
        EventBus.Instance.Unsubscribe<PlayerStateChangedEvent>(OnPlayerStateChanged);
        EventBus.Instance.Unsubscribe<SystemSettingsChangedEvent>(OnSystemSettingsChanged);
        EventBus.Instance.Unsubscribe<ClueDiscoveredEvent>(OnClueDiscovered);
    }

    // ========== 事件定义（补充）==========
    // 注意：以下事件类型应同步添加到 shared-types.md

    /// <summary>
    /// 世界状态变更事件（WorldMap System 发布）
    /// </summary>
    public struct WorldStateChangedEvent
    {
        public string currentLocationId;
        public HashSet<string> revealedLocations;
        public int gameDayCount;
    }

    /// <summary>
    /// 玩家状态变更事件（PlayerController 发布）
    /// </summary>
    public struct PlayerStateChangedEvent
    {
        public HealthState healthState;
        public int sanity;
        public int rage;
        public Dictionary<string, int> inventory;
        public List<string> unlockedWeapons;
        public Dictionary<string, int> npcRelationships;
        public List<string> collectedClues;
        public List<string> acquiredKnowledge;
        public Dictionary<string, string> clueLocationMap;
    }

    /// <summary>
    /// 系统设置变更事件（Settings System 发布）
    /// </summary>
    public struct SystemSettingsChangedEvent
    {
        public float masterVolume;
        public float musicVolume;
        public float sfxVolume;
        public float dialogueVolume;
        public bool subtitlesEnabled;
        public bool hapticFeedbackEnabled;
        public string controlScheme;
        public Dictionary<string, bool> achievements;
        public GameStatistics statistics;
    }
}
```

### 4.1 存档事件定义

> **事件统一定义**：以下事件完整定义见 shared-types.md §10.1，本文档仅引用不重复定义。

- `SaveCompletedEvent`：定义于 shared-types.md §10.1
- `SaveCorruptedEvent`：定义于 shared-types.md §10.1
- `LoadCompletedEvent`：定义于 shared-types.md §10.1

### 4.2 并发控制策略

> **存档触发与网络检查点保存的并发控制**（ADR 评审修复 2026-04-15）：
> SaveManager 使用 `SemaphoreSlim _saveLock` 实现互斥访问，确保以下场景不会发生数据竞争：

| 场景 | 控制策略 |
|------|----------|
| **常规存档** | 通过 `_saveLock` 互斥锁保护，`SaveAsync()` 等待锁释放后执行 |
| **OnApplicationQuit 存档** | 使用 `StartCoroutine(SaveOnQuitCoroutine())` 替代 `async void`，协程内部等待 `_saveLock` |
| **SaveCheckpointAsync（网络检查点）** | 与常规存档共用同一 `_saveLock`，防止同时写入同一槽位 |
| **TriggerAutoSave（自动存档）** | 调用 `SaveAsync()` 自动等待锁，确保不会打断正在进行的存档 |

> **设计决策**：
> - 使用 `SemaphoreSlim` 而非 `Mutex`，因为协程中可以 `yield return _saveLock.WaitAsync()`
> - 同一时刻只允许一个存档操作，防止"reconnect_checkpoint"槽位被常规存档覆盖
> - 锁等待超时默认 30 秒，超时后放弃等待并记录警告

### 5. 版本迁移策略

```csharp
// SaveMigration.cs
public static class SaveMigration
{
    private static readonly Dictionary<string, Action<SaveData>> Migrators = new()
    {
        { "1.0.0_to_1.1.0", Migrate_1_0_0_to_1_1_0 },
        { "1.1.0_to_1.2.0", Migrate_1_1_0_to_1_2_0 },
    };

    public static SaveData MigrateSaveData(SaveData saveData)
    {
        var currentVersion = saveData.saveVersion;

        // 逐级迁移
        foreach (var migrator in Migrators)
        {
            if (IsVersionLessThan(currentVersion, migrator.Key.Split("_to_")[1]))
            {
                migrator.Value(saveData);
                currentVersion = migrator.Key.Split("_to_")[1];
                Debug.Log($"Migrated save from {currentVersion} to {migrator.Key.Split("_to_")[1]}");
            }
        }

        saveData.saveVersion = SaveFormat.CURRENT_VERSION.ToString();
        return saveData;
    }

    private static void Migrate_1_0_0_to_1_1_0(SaveData save)
    {
        // 示例：1.0.0 到 1.1.0 的迁移
        // 新增字段的默认值处理
        if (!save.system.achievements.ContainsKey("first_blood"))
        {
            save.system.achievements["first_blood"] = false;
        }
    }
}
```

### 6. 存档时机定义

| 时机 | 触发条件 | 存档类型 | 备注 |
|------|---------|---------|------|
| **撤离成功** | 玩家满足撤离条件离开地区 | Auto | 玩家主要进度保存点 |
| **进入地区** | 加载画面完成后 | Auto | 防止加载崩溃导致进度丢失 |
| **收集线索** | ClueDiscoveredEvent 触发 | Auto | 防止关键剧情进度丢失 |
| **游戏天推进** | 地图模式停留 1 分钟 | Auto | 精英敌人重生判定依赖 |
| **手动存档** | 玩家按快捷键/菜单保存 | Manual | 玩家主动行为 |
| **离开游戏** | OnApplicationQuit | Auto | 必须存档，防止退出时丢失 |

### 7. 多槽位管理

```csharp
// SaveSlotManager.cs
public class SaveSlotManager
{
    public const int MAX_SLOTS = 3;

    public List<SaveSlotMetadata> GetAllSlots()
    {
        var slots = new List<SaveSlotMetadata>();
        for (int i = 0; i < MAX_SLOTS; i++)
        {
            var slotId = $"slot_{i}";
            slots.Add(GetSlotMetadata(slotId));
        }
        return slots;
    }

    public bool IsSlotEmpty(string slotId)
    {
        return !_primaryStorage.ExistsAsync(slotId).Result;
    }

    public string CreateNewSlot()
    {
        for (int i = 0; i < MAX_SLOTS; i++)
        {
            var slotId = $"slot_{i}";
            if (IsSlotEmpty(slotId))
                return slotId;
        }
        return null; // 无空槽位
    }

    public bool DeleteSlot(string slotId)
    {
        if (IsSlotEmpty(slotId)) return false;

        _primaryStorage.DeleteAsync(slotId);
        _backupStorage.DeleteAsync(slotId);
        DeleteSlotMetadata(slotId);
        return true;
    }
}
```

### 8. 加密方案

```csharp
// SaveEncryption.cs
public static class SaveEncryption
{
    // AES-256-GCM 是当前业界标准，PS5/Steam 都支持
    private const int KEY_SIZE = 256;
    private const int NONCE_SIZE = 12;

    public static byte[] Encrypt(byte[] plainText)
    {
        var key = GetEncryptionKey();
        var nonce = GenerateSecureRandom(NONCE_SIZE);

        using var aes = new AesGcm(key, KEY_SIZE / 8);
        var cipherText = new byte[plainText.Length];
        var tag = new byte[16];

        aes.Encrypt(nonce, plainText, cipherText, tag);

        // 格式: nonce + tag + ciphertext
        return Combine(nonce, tag, cipherText);
    }

    public static byte[] Decrypt(byte[] encrypted)
    {
        var key = GetEncryptionKey();

        var nonce = encrypted.Take(NONCE_SIZE).ToArray();
        var tag = encrypted.Skip(NONCE_SIZE).Take(16).ToArray();
        var cipherText = encrypted.Skip(NONCE_SIZE + 16).ToArray();

        using var aes = new AesGcm(key, KEY_SIZE / 8);
        var plainText = new byte[cipherText.Length];
        aes.Decrypt(nonce, cipherText, tag, plainText);

        return plainText;
    }

    private static byte[] GetEncryptionKey()
    {
        // PC (Steam): 使用 Steamworks API 获取加密密钥
        if (Application.platform == RuntimePlatform.WindowsPlayer ||
            Application.platform == RuntimePlatform.OSXPlayer)
        {
            return Steamworks.SteamRemoteStorage.GetEncryptionKey();
        }

        // PS5: 使用 PSN API 获取加密密钥
        if (Application.platform == RuntimePlatform.PS5)
        {
            return PSNSystem.GetSaveEncryptionKey();
        }

        // ⚠️ 安全警告: Editor / Fallback 使用内置密钥（仅用于测试）
        // 生产环境必须确保 Steamworks/PSN API 可用，否则存档将以弱密钥加密
        // 建议在获取密钥失败时抛出异常而非使用 Fallback
        return GetFallbackKey();
    }
}
```

### 9. Unity 项目结构

```
Assets/Game/Infrastructure/SaveSystem/
├── SaveManager.cs                    # 单例，协调存档入口/出口
├── SaveSlotManager.cs                # 多槽位管理
├── SaveFormat.cs                     # 存档文件格式定义
├── SaveEncryption.cs                 # AES-256-GCM 加密
├── SaveMigration.cs                  # 版本迁移逻辑
├── StorageAdapters/
│   ├── IStorageAdapter.cs           # 存储适配器接口
│   ├── LocalStorageAdapter.cs       # PC 本地存储
│   ├── SteamCloudAdapter.cs         # Steam Cloud
│   └── PSNCloudAdapter.cs           # PS5 Cloud Save
├── DataClasses/
│   ├── SaveData.cs                  # 根存档数据结构
│   ├── WorldSave.cs                  # 世界探索状态
│   ├── PlayerSave.cs                 # 玩家进度
│   ├── SystemSave.cs                 # 系统设置
│   └── SaveSlotMetadata.cs           # 槽位元数据
└── Events/
    ├── SaveCompletedEvent.cs         # 存档完成事件
    ├── LoadCompletedEvent.cs         # 加载完成事件
    └── SaveCorruptedEvent.cs         # 存档损坏事件
```

---

## Alternatives Considered

### Alternative 1: 纯明文 JSON

- **描述**：存档直接序列化为明文 JSON 文件
- **优点**：
  - 实现简单，调试方便
  - 人类可读，易于排查问题
  - 无加密开销
- **缺点**：
  - 玩家可以手动修改存档（作弊）
  - 无版本校验，损坏不易发现
- **拒绝理由**：
  - 严重影响游戏寿命和经济系统平衡
  - 玩家修改存档可能导致 bug，难以排查
  - 轻微的版本不匹配也难以检测

### Alternative 2: Unity PlayerPrefs

- **描述**：使用 Unity 内置的 PlayerPrefs 存储存档
- **优点**：
  - Unity 内置，无需额外代码
  - 跨平台自动适配
- **缺点**：
  - 容量有限（不适合大量数据）
  - 明文存储，无加密
  - 无结构化查询能力
  - 非异步，存档时可能卡顿
- **拒绝理由**：
  - 存档数据量大（世界状态、玩家进度、线索等）
  - PlayerPrefs 容量和性能都不足以支撑
  - 明文存储风险同"纯 JSON"方案

### Alternative 3: 二进制 Protobuf + 平台加密

- **描述**：使用 Protobuf 序列化 + 平台原生加密（iOS Keychain / Android Keystore）
- **优点**：
  - Protobuf 序列化效率高
  - 平台原生加密安全性高
- **缺点**：
  - Protobuf 需要 .proto 定义文件，增加工具链复杂度
  - 多平台适配（Steam/PSN）需要抽象层
  - 调试困难，二进制数据不直观
- **拒绝理由**：
  - 引入 Protobuf 增加工具链复杂度
  - 当前存档数据量 JSON 序列化完全可接受
  - 已有完整的存储适配器抽象，多平台支持已解决

---

## Consequences

### Positive

- **数据安全**：AES-256-GCM 加密防止篡改，本地备份防止云同步失败
- **跨平台支持**：存储适配器抽象层支持 PC (Steam) 和 PS5
- **版本迁移**：可扩展的迁移系统支持未来版本兼容
- **性能优良**：异步存档不阻塞游戏主线程
- **调试友好**：本地备份可用于数据恢复，JSON 格式便于问题排查

### Negative

- **加密开销**：AES-256 加密/解密有轻微 CPU 开销（约 1-2ms 每存档）
- **存储空间**：加密后文件比纯 JSON 略大（约 5-10%）
- **密钥管理**：需要平台 API 支持，不同平台实现不同

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **存档损坏** | 写入时崩溃导致存档损坏 | 双写策略（主存储 + 本地备份）；写入前校验；崩溃恢复 |
| **版本不兼容** | 游戏更新后旧存档无法加载 | 版本迁移系统；主版本不兼容时拒绝加载并提示 |
| **云同步冲突** | 多设备同时存档导致冲突 | PS5/Steam 自动处理冲突；保留本地备份用于恢复 |
| **密钥丢失** | 平台 API 密钥获取失败 | Fallback 内置密钥（仅测试用）；严格错误处理 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **Save() 耗时** | < 50ms (异步) | AES-256 + JSON 序列化 |
| **Load() 耗时** | < 100ms | AES-256 + JSON 反序列化 |
| **存档大小** | < 100KB | 包含世界、玩家、系统三部分 |
| **内存占用** | < 1MB | SaveData 对象大小 |

---

## Migration Plan

### Phase 1: 基础架构
- [ ] 创建 SaveData 数据结构
- [ ] 实现 SaveFormat 序列化/反序列化
- [ ] 实现 SaveEncryption AES-256-GCM 加密
- [ ] 实现 LocalStorageAdapter

### Phase 2: 存储适配
- [ ] 实现 SteamCloudAdapter
- [ ] 实现 PSNCloudAdapter
- [ ] 实现 SaveManager 协调器

### Phase 3: 自动存档
- [ ] 集成 WorldMapSystem 事件（AreaEntered, AreaExtracted）
- [ ] 集成 ClueSystem 事件（ClueDiscoveredEvent）
- [ ] 实现游戏天计时器存档

### Phase 4: 多槽位和 UI
- [ ] 实现 SaveSlotManager
- [ ] 集成 UI System 的存档菜单
- [ ] 实现存档缩略图功能

### Phase 5: 错误处理和恢复
- [ ] 实现存档损坏检测和恢复
- [ ] 实现版本迁移系统
- [ ] 实现云同步冲突处理

---

## Validation Criteria

1. **存档完整性**：存档文件包含所有必要数据，加载后游戏状态一致
2. **加密有效性**：存档文件无法被人类阅读，无法被手动篡改
3. **跨平台一致**：PC 和 PS5 的存档格式兼容
4. **版本迁移正确**：旧版本存档加载后正确迁移
5. **自动存档可靠**：所有定义的自动存档触发点正确工作
6. **错误恢复**：存档损坏时能从本地备份恢复
7. **性能达标**：存档/加载操作不阻塞游戏主线程
8. **检查点保存**：SaveCheckpointAsync 能正确保存 GameStateSnapshot 到专用槽位
9. **退出存档**：OnApplicationQuit 使用协程同步保存，不依赖 async void
10. **并发安全**：多个存档触发点不会同时写入，造成数据竞争

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 存档触发依赖事件总线
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — 存档系统属于 Infrastructure Layer
- [ADR-0006: 网络同步架构](./adr-0006-network-synchronization-architecture.md) — 存档续连依赖存档系统，SaveCheckpointAsync 使用 "reconnect_checkpoint" 专用槽位
- [NPC AI System GDD](../../design/gdd/npc-ai-system.md) — NPC AI 发送 AreaClearedEvent 触发自动存档
- [World Map & Non-Linear Progression GDD](../../design/gdd/world-map-progression.md) — 存档数据结构定义依据
- [NPC AI System GDD](../../design/gdd/npc-ai-system.md) — NPC AI 发送 AreaCleared 事件触发存档
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 存档系统事件（SaveCompletedEvent, SaveCorruptedEvent）定义
