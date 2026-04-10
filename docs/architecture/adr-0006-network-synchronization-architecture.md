# ADR-0006: 网络同步架构决策 (Network Synchronization Architecture)

## Status
**Accepted**

## Date
2026-04-09

## Last Updated
2026-04-09

## Context

### Problem Statement

ADR-0002 (Unity 引擎技术选型) 确定了使用 **Mirror P2P** 作为网络方案，但未定义具体实现细节。作为一款俯视角潜行游戏，网络同步面临特殊挑战：

1. **潜行游戏的同步敏感性**：视野/警戒状态的变化需要精确同步，错帧可能导致"穿帮"或"假警报"
2. **相位同步需求**：《Hotline Miami》式的战斗节奏要求所有玩家在同一帧看到击杀
3. **P2P 的天然延迟**：主机-客户端延迟可能高达 100-200ms，需要平滑的预测和纠正机制
4. **存档续连**：《断绝》的存档系统需要支持断线重连后继续游戏

### Constraints

- **平台目标**：PC (Steam) & PS5（PS5 需要 Sony 网络认证）
- **网络条件**：目标用户为家庭宽带，延迟 50-200ms
- **同步精度**：战斗事件需要帧级精度，潜行状态可以接受秒级精度
- **兼容性**：Mirror P2P 需要支持相位同步（phase synchronization）
- **PSN 要求**：PS5 平台需要集成 PSN 网络功能

### Requirements

- **必须**：确定 P2P vs Client-Server 的架构选择
- **必须**：定义相位同步的实现方案
- **必须**：定义玩家输入同步机制
- **必须**：定义 NPC 状态同步策略
- **必须**：定义断线重连和房间管理方案
- **必须**：定义延迟补偿和预测机制

---

## Decision

### 架构决策

选定 **Mirror P2P (Phase-Synchronized)** 作为网络架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Mirror P2P 网络架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                      Host (Player 1)                              │   │
│  │  - 权威游戏状态 (Authoritative Game State)                         │   │
│  │  - 运行完整游戏逻辑 (NPC AI, Combat, etc.)                         │   │
│  │  - 广播相位快照 (Phase Snapshot)                                   │   │
│  │  - 处理所有输入并广播结果                                          │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│                        Phase Snapshot (30Hz)                              │
│                        Input Commands (同步)                             │
│                                    │                                      │
│          ┌─────────────────────────┴─────────────────────────┐          │
│          ▼                                                   ▼          │
│  ┌───────────────────┐                           ┌───────────────────┐   │
│  │  Client (P2)      │                           │  Client (P3)      │   │
│  │                   │                           │                   │   │
│  │  本地预测 + 校正   │                           │  本地预测 + 校正   │   │
│  │  运行完整渲染      │                           │  运行完整渲染      │   │
│  │  延迟补偿          │                           │  延迟补偿          │   │
│  └───────────────────┘                           └───────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     NetworkTopology                                │   │
│  │  - P2P Mesh (全连接，但只有 Host 有权威)                          │   │
│  │  - Host = 游戏状态权威源                                            │   │
│  │  - Clients = 渲染 + 本地预测                                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. 架构选择理由

**P2P vs Client-Server 分析**：

| 维度 | P2P | Client-Server |
|------|-----|---------------|
| **延迟** | 玩家间直连，延迟 = 网络距离 | 所有请求经过服务器，额外 20-50ms |
| **成本** | 无服务器成本 | 需要租用服务器 |
| **一致性** | 需要相位同步机制 | 服务器权威，一致性简单 |
| **PS5 认证** | 需要通过 PSN P2P 认证 | 需要 PSN 服务器认证 |
| **适用游戏** | 2-4 人合作/对战 | MMORPG、大规模多人 |

**选择 P2P 的理由**：
- 潜行合作游戏通常 2-4 人，P2P 延迟更低
- 独立工作室无力承担服务器成本
- Mirror 的 P2P 方案成熟，支持相位同步

### 1.5 游戏状态快照 (GameStateSnapshot)

```csharp
// GameStateSnapshot.cs - 用于相位同步和断线重连的游戏状态快照
[Serializable]
public struct GameStateSnapshot
{
    public int SnapshotVersion;              // 快照版本号
    public long Timestamp;                  // 快照时间戳
    public int Phase;                       // 快照对应的相位

    // 玩家状态
    // ⚠️ 容量限制: 最多 MaxPlayers (4) 个玩家
    public List<PlayerSnapshot> Players;

    // NPC 状态（仅关键状态同步）
    // ⚠️ 容量限制: 最多 100 个 NPC（与 NPCManager.NPCsPerFrame 一致）
    public List<NPCSnapshot> NPCs;

    // 世界状态（用于存档续连）
    public WorldSnapshot World;

    // 存档状态
    public SaveSlotId CurrentSaveSlot;
}

[Serializable]
public struct PlayerSnapshot
{
    public int PlayerId;
    public Vector3 Position;
    public float Rotation;
    public AlertState AlertState;
    public WorldState WorldState;
    public HealthState HealthState;
    public int Sanity;
    public int Rage;
    public int LastProcessedPhase;         // 最后处理的相位（用于去重）
}

[Serializable]
public struct NPCSnapshot
{
    public int NPCId;
    public Vector3 Position;
    public float Rotation;
    public AlertState AlertState;
    public WorldState WorldState;
    // NPC 完整状态在 Host 端，不同步到客户端
}

[Serializable]
public struct WorldSnapshot
{
    public string AreaId;                  // 当前所在地区
    public Dictionary<string, AreaSave> AreaStates;
    public Dictionary<string, CitySave> CityStates;
    public HashSet<string> RevealedLocations;
    public int GameDayCount;
}

// 快照序列化/反序列化
public static class GameStateSnapshotSerializer
{
    public static byte[] Serialize(GameStateSnapshot snapshot)
    {
        var json = JsonUtility.ToJson(snapshot);
        return Encoding.UTF8.GetBytes(json);
    }

    public static GameStateSnapshot Deserialize(byte[] data)
    {
        var json = Encoding.UTF8.GetString(data);
        return JsonUtility.FromJson<GameStateSnapshot>(json);
    }
}

// ReconnectCheckpoint - 用于断线重连的完整检查点
[Serializable]
public class ReconnectCheckpoint
{
    public string SessionId;                // 会话 ID
    public long Timestamp;                 // 检查点时间戳
    public GameStateSnapshot GameState;    // 游戏状态快照
    public string SaveSlotId;              // 对应的存档槽位
    public int HostPlayerId;               // 当前 Host 的玩家 ID
    public List<int> ConnectedPlayerIds;   // 断线前的玩家 ID 列表
}
```

相位同步是潜行游戏的关键——所有玩家必须在同一时刻看到相同的状态。

```csharp
// PhaseManager.cs
public class PhaseManager : NetworkBehaviour
{
    // 相位同步参数
    private const int PHASE_RATE = 30;  // 每秒 30 个相位
    private const float PHASE_INTERVAL = 1f / PHASE_RATE;
    private const int PHASE_BUFFER_SIZE = 3;  // 客户端缓冲 3 帧

    // 相位状态
    private int _currentPhase;
    private double _lastPhaseTime;
    private readonly Queue<PhaseSnapshot> _pendingSnapshots = new();
    private float _phaseTimer;

    // NetworkIdentity 组件用于 Mirror 同步
    private NetworkIdentity _networkIdentity;

    public struct PhaseSnapshot
    {
        public int Phase;
        public double Timestamp;
        public GameStateSnapshot State;  // 使用新增的 GameStateSnapshot
    }

    public override void OnStartServer()
    {
        // Host 启动时初始化相位循环
        _phaseTimer = 0f;
        _currentPhase = 0;
        _lastPhaseTime = NetworkTime.time;
    }

    // Host: 使用计时器驱动相位循环（不依赖 InvokeRepeating）
    [Server]
    private void Update()
    {
        _phaseTimer += Time.deltaTime;

        while (_phaseTimer >= PHASE_INTERVAL)
        {
            _phaseTimer -= PHASE_INTERVAL;
            OnPhaseTick();
        }
    }

    // Host: 广播相位快照
    [Server]
    public void OnPhaseTick()
    {
        _currentPhase++;
        _lastPhaseTime = NetworkTime.time;

        var snapshot = new PhaseSnapshot
        {
            Phase = _currentPhase,
            Timestamp = _lastPhaseTime,
            State = CaptureGameStateSnapshot()
        };

        // 广播给所有客户端
        RpcBroadcastSnapshot(snapshot);
    }

    // Client: 接收相位快照
    [ClientRpc]
    private void RpcBroadcastSnapshot(PhaseSnapshot snapshot)
    {
        if (!isServer)
        {
            _pendingSnapshots.Enqueue(snapshot);

            // 保持缓冲大小
            while (_pendingSnapshots.Count > PHASE_BUFFER_SIZE)
                _pendingSnapshots.Dequeue();

            // 如果落后太多，请求完整同步
            if (_pendingSnapshots.Count > PHASE_BUFFER_SIZE * 2)
                CmdRequestFullStateSync();
        }
    }

    // Client: 请求完整状态同步
    [Command]
    private void CmdRequestFullStateSync()
    {
        // Host 发送完整 GameStateSnapshot
        var fullSnapshot = CaptureGameStateSnapshot();
        TargetSendFullSnapshot(connectionToClient, fullSnapshot);
    }

    [TargetRpc]
    private void TargetSendFullSnapshot(NetworkConnection target, GameStateSnapshot snapshot)
    {
        // 客户端清空缓冲，应用完整快照
        _pendingSnapshots.Clear();
        ApplyFullSnapshot(snapshot);
    }

    // Client: 获取当前相位游戏状态
    public GameStateSnapshot GetInterpolatedState()
    {
        if (_pendingSnapshots.Count == 0)
            return GetLastKnownSnapshot();

        return _pendingSnapshots.Peek().State;
    }

    // Host: 捕获完整游戏状态快照
    [Server]
    private GameStateSnapshot CaptureGameStateSnapshot()
    {
        var snapshot = new GameStateSnapshot
        {
            SnapshotVersion = 1,
            Timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds(),
            Phase = _currentPhase,
            Players = CaptureAllPlayerSnapshots(),
            NPCs = CaptureAllNPCSnapshots(),
            World = CaptureWorldSnapshot()
        };
        return snapshot;
    }
}
```

### 3. 玩家输入同步

```csharp
// PlayerInputSync.cs
public class PlayerInputSync : NetworkBehaviour
{
    // 输入命令结构（客户端 → Host）
    public struct InputCommand
    {
        public int Phase;              // 命令生效的相位
        public float MoveX;            // 移动 X (-1 ~ 1)
        public float MoveY;            // 移动 Y (-1 ~ 1)
        public InputFlags Flags;       // 蹲下/冲刺/互动 等标志位
        public float LookAngle;        // 准星角度
        public float Timestamp;        // 发送时间戳
    }

    [Flags]
    public enum InputFlags
    {
        None = 0,
        Crouch = 1 << 0,
        Sprint = 1 << 1,
        Interact = 1 << 2,
        Attack = 1 << 3,
        Reload = 1 << 4
    }

    // 客户端：发送输入命令
    [Client]
    public void SendInput(InputCommand cmd)
    {
        cmd.Phase = _phaseManager.CurrentPhase + 1;  // 下一相位生效
        cmd.Timestamp = Time.time;

        CmdSendInput(cmd);
    }

    // Host：接收并处理输入命令
    [Command]
    private void CmdSendInput(InputCommand cmd)
    {
        // 存储命令，等待相位处理
        _pendingInputs[cmd.ClientId].Enqueue(cmd);
    }

    // Host：在相位开始时处理所有输入
    [Server]
    private void ProcessInputsForPhase(int phase)
    {
        foreach (var kvp in _pendingInputs)
        {
            var clientId = kvp.Key;
            var queue = kvp.Value;

            while (queue.Count > 0 && queue.Peek().Phase == phase)
            {
                var cmd = queue.Dequeue();
                ApplyInputToPlayer(clientId, cmd);
            }
        }
    }
}
```

### 4. NPC 状态同步策略

NPC AI 在 Host 上运行，客户端只接收必要的状态更新。

```csharp
// NPCStateSync.cs
public class NPCStateSync : NetworkBehaviour
{
    // NPC 同步数据结构
    [SyncVar]
    public AlertState CurrentAlertState;

    [SyncVar]
    public WorldState CurrentWorldState;

    [SyncVar]
    public Vector3 NetworkPosition;

    [SyncVar]
    public float NetworkRotation;

    // NPC 行为结果（用于客户端预测）
    [SyncVar]
    public int LastProcessedPhase;

    // 仅 Host 运行 NPC AI
    [Server]
    private void Update()
    {
        // NPC AI 在 Host 上更新
        // ... (NPC AI 逻辑)

        // 同步关键状态到客户端
        CurrentAlertState = _npcController.AlertState;
        CurrentWorldState = _npcController.WorldState;
        NetworkPosition = _npcController.transform.position;
    }

    // 客户端：接收状态更新
    public override void OnStartClient()
    {
        // 客户端不运行 NPC AI，只渲染
        enabled = false;  // 禁用 Update
    }
}
```

### 5. 延迟补偿和预测

```csharp
// ClientPrediction.cs
public class ClientPrediction : NetworkBehaviour
{
    // 延迟补偿参数
    private const int INPUT_DELAY = 2;  // 客户端延迟 2 个相位执行输入

    // 本地预测状态
    private PlayerState _predictedState;
    private Queue<InputCommand> _unconfirmedInputs = new();

    // 客户端：预测执行输入
    [Client]
    private void PredictInput(InputCommand cmd)
    {
        _predictedState = SimulateInput(_predictedState, cmd);
        _unconfirmedInputs.Enqueue(cmd);

        // 应用预测渲染
        ApplyStateToPlayer(_predictedState);
    }

    // 客户端：校正（收到 Host 确认）
    [Client]
    public void OnHostStateReceived(PlayerState hostState, int confirmedPhase)
    {
        // 移除已确认的输入
        while (_unconfirmedInputs.Count > 0 &&
               _unconfirmedInputs.Peek().Phase <= confirmedPhase)
        {
            _unconfirmedInputs.Dequeue();
        }

        // 重新模拟未确认的输入
        var baseState = hostState;
        foreach (var cmd in _unconfirmedInputs)
        {
            baseState = SimulateInput(baseState, cmd);
        }

        // 平滑校正到 Host 状态
        var correction = baseState.Position - _predictedState.Position;
        if (correction.magnitude > SNAP_THRESHOLD)
        {
            // 超过阈值，直接跳转到正确位置
            _predictedState.Position = baseState.Position;
        }
        else
        {
            // 小偏差平滑插值
            _predictedState.Position = Vector3.Lerp(
                _predictedState.Position,
                baseState.Position,
                CORRECTION_LERP * Time.deltaTime
            );
        }
    }

    // 状态模拟（纯函数，无副作用）
    private PlayerState SimulateInput(PlayerState state, InputCommand cmd)
    {
        // 根据输入模拟下一状态
        // ... (与 Host 相同的模拟逻辑)
    }
}
```

### 6. 房间管理和配对

```csharp
// NetworkRoomManager.cs
public class NetworkRoomManager : NetworkManager
{
    // 房间设置
    public int MaxPlayers = 4;
    public int MinPlayers = 2;
    public string RoomName;

    // 房间状态
    public int CurrentPlayerCount => _connectedPlayers.Count;

    private readonly List<NetworkConnection> _connectedPlayers = new();

    // Steam P2P 配对
    public async Task CreateRoomAsync(string roomName)
    {
        RoomName = roomName;

        // Steam: 创建邀请代码
        if (SteamManager.IsInitialized)
        {
            var lobby = SteamMatchmaking.CreateLobby(MaxPlayers);
            lobby.SetPublic();
            lobby.SetJoinable(true);

            // 存储本地房间信息
            PlayerPrefs.SetString("LastRoomCode", lobby.GetLobbyID().ToString());
        }

        // PSN: 创建房间
        if (PSNSystem.IsSignedIn)
        {
            await PSNSession.CreateAsync(roomName, MaxPlayers);
        }

        StartHost();
    }

    public async Task JoinRoomAsync(string roomCode)
    {
        // Steam: 通过邀请码加入
        if (SteamManager.IsInitialized)
        {
            var lobbyID = SteamMatchmaking.GetLobbyFromID(ulong.Parse(roomCode));
            SteamMatchmaking.JoinLobby(lobbyID);
        }

        // PSN: 通过邀请码加入
        if (PSNSystem.IsSignedIn)
        {
            await PSNSession.JoinAsync(roomCode);
        }

        StartClient();
    }

    // Host 迁移（Host 离开时）
    [Server]
    public void OnServerDisconnect(NetworkConnection conn)
    {
        // 如果 Host 断开，尝试迁移到新 Host
        if (conn == NetworkServer.localConnection)
        {
            if (_connectedPlayers.Count > 1)
            {
                // 选择连接质量最佳、延迟最低的玩家作为新 Host
                var newHost = SelectBestHostCandidate();
                if (newHost != null)
                {
                    MigrateHostTo(newHost);
                }
                else
                {
                    Debug.LogWarning("[Network] No suitable host candidate found");
                    NetworkServer.Shutdown();
                }
            }
            else
            {
                // 最后一个玩家，结束游戏
                NetworkServer.Shutdown();
            }
        }

        _connectedPlayers.Remove(conn);
    }

    [Server]
    private NetworkConnection SelectBestHostCandidate()
    {
        // 优先选择连接时间最长、延迟最低的玩家
        return _connectedPlayers
            .Where(c => c != NetworkServer.localConnection)
            .OrderByDescending(c => GetConnectionUptime(c))
            .ThenBy(c => GetConnectionLatency(c))
            .FirstOrDefault();
    }

    [Server]
    private float GetConnectionUptime(NetworkConnection conn)
    {
        // 返回连接持续时间
        return Time.time - _connectionStartTimes.GetValueOrDefault(conn, Time.time);
    }

    [Server]
    private float GetConnectionLatency(NetworkConnection conn)
    {
        return NetworkTime.GetPing() * 2;
    }
}
```

### 7. 断线重连和存档续连

```csharp
// ReconnectionManager.cs
public class ReconnectionManager : MonoBehaviour
{
    // 断线重连超时
    private const float RECONNECT_TIMEOUT = 30f;

    private string _sessionId;
    private float _disconnectTime;
    private bool _isReconnecting;
    private int _disconnectedPlayerId;
    private NetworkConnection _disconnectedConnection;

    // 存档续连点（仅 Host 使用）
    private ReconnectCheckpoint _lastCheckpoint;

    // 事件发布
    public event System.Action<int> OnPlayerReconnected;      // 玩家重连成功
    public event System.Action<int> OnReconnectFailed;          // 重连失败（参数：玩家ID）

    // ========== Host 端：保存检查点 ==========

    // Host: 在检测到客户端断线时保存检查点
    [Server]
    public void OnClientDisconnect(NetworkConnection conn)
    {
        _disconnectedConnection = conn;
        _disconnectTime = Time.time;
        _disconnectedPlayerId = GetPlayerId(conn);

        // 保存检查点到存档系统（使用专用槽位）
        SaveCheckpointToSaveSystem();

        // 启动重连等待
        _isReconnecting = true;

        // 通知 UI 显示重连提示
        EventBus.Instance.Publish(new PlayerDisconnectedEvent(_disconnectedPlayerId));
    }

    [Server]
    private async void SaveCheckpointToSaveSystem()
    {
        // 构建检查点
        _lastCheckpoint = new ReconnectCheckpoint
        {
            SessionId = _sessionId,
            Timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds(),
            GameState = _phaseManager.CaptureGameStateSnapshot(),
            // ⚠️ SaveSlotId 应从当前游戏会话获取，表示当前游戏所在的存档槽位
            // SaveCheckpointAsync 使用 "reconnect_checkpoint" 专用槽位，不使用此字段
            SaveSlotId = "slot_0", // TODO: 从 NetworkRoomManager 或游戏会话获取实际槽位
            HostPlayerId = GetHostPlayerId(),
            ConnectedPlayerIds = GetAllConnectedPlayerIds()
        };

        // 通过存档系统保存检查点
        await SaveManager.Instance.SaveCheckpointAsync();
    }

    // ========== 客户端：尝试重连 ==========

    [Client]
    public async void AttemptReconnect()
    {
        if (!_isReconnecting) return;

        _isReconnecting = true;
        var playerId = LocalPlayerId;

        try
        {
            // 重新连接到 Host
            await NetworkManager.singleton.client.ConnectAsync();

            // 请求同步检查点
            CmdRequestCheckpoint();

            Debug.Log($"[Reconnect] Player {playerId} reconnected successfully");
        }
        catch (Exception ex)
        {
            Debug.LogError($"[Reconnect] Player {playerId} reconnect failed: {ex.Message}");
            OnReconnectFailed(playerId);
        }
    }

    [Command]
    private void CmdRequestCheckpoint(NetworkConnectionToClient sender)
    {
        if (_lastCheckpoint == null)
        {
            // 没有检查点，发送完整状态
            var fullSnapshot = _phaseManager.CaptureGameStateSnapshot();
            TargetSendFullSnapshot(sender, fullSnapshot);
            return;
        }

        TargetSendCheckpoint(sender, _lastCheckpoint);
    }

    [TargetRpc]
    private void TargetSendCheckpoint(NetworkConnection target, ReconnectCheckpoint checkpoint)
    {
        // 恢复游戏状态
        ApplyCheckpoint(checkpoint);
        _isReconnecting = false;

        EventBus.Instance.Publish(new PlayerReconnectedEvent(_disconnectedPlayerId));
        OnPlayerReconnected?.Invoke(_disconnectedPlayerId);
    }

    [TargetRpc]
    private void TargetSendFullSnapshot(NetworkConnection target, GameStateSnapshot snapshot)
    {
        ApplyFullSnapshot(snapshot);
        _isReconnecting = false;

        EventBus.Instance.Publish(new PlayerReconnectedEvent(_disconnectedPlayerId));
        OnPlayerReconnected?.Invoke(_disconnectedPlayerId);
    }

    // ========== 重连失败处理 ==========

    [Client]
    private void OnReconnectFailed(int playerId)
    {
        _isReconnecting = false;

        // 弹出提示，让玩家选择后续操作
        EventBus.Instance.Publish(new ReconnectFailedEvent(playerId));

        // 通知 UI 显示选项菜单
        // 选项 1: 返回主菜单
        // 选项 2: 重新连接（新游戏）
        // 选项 3: 加载存档
        OnReconnectFailed?.Invoke(playerId);
    }

    [Server]
    private void OnServerReconnectTimeout()
    {
        // 服务器端超时：强制结束等待
        _isReconnecting = false;

        // 通知所有客户端重连失败
        EventBus.Instance.Publish(new ReconnectTimeoutEvent(_disconnectedPlayerId));

        // 如果是 Host 断线，询问是否迁移
        if (_disconnectedPlayerId == GetHostPlayerId())
        {
            // 尝试迁移 Host
            if (TryMigrateHost())
            {
                Debug.Log($"[Reconnect] Host migrated successfully");
            }
            else
            {
                Debug.Log($"[Reconnect] Host migration failed, ending session");
                NetworkServer.Shutdown();
            }
        }
    }

    private void Update()
    {
        if (_isReconnecting)
        {
            if (Time.time - _disconnectTime > RECONNECT_TIMEOUT)
            {
                _isReconnecting = false;
                OnServerReconnectTimeout();
            }
        }
    }

    // ========== Host 迁移 ==========

    [Server]
    private bool TryMigrateHost()
    {
        if (_connectedPlayers.Count <= 1) return false;

        // 选择新 Host：优先选择连接时间最长、延迟最低的玩家
        var candidates = _connectedPlayers
            .Where(c => c != _disconnectedConnection)
            .OrderByDescending(c => GetConnectionQuality(c))
            .ThenBy(c => GetConnectionLatency(c))
            .ToList();

        if (candidates.Count == 0) return false;

        var newHost = candidates[0];
        MigrateHostTo(newHost);
        return true;
    }

    [Server]
    private void MigrateHostTo(NetworkConnection newHostConnection)
    {
        // 1. 保存当前状态为检查点
        SaveCheckpointToSaveSystem();

        // 2. 通知所有客户端 Host 即将迁移
        EventBus.Instance.Publish(new HostMigrationStartedEvent(
            _disconnectedPlayerId,
            GetPlayerId(newHostConnection)
        ));

        // 3. 停止当前 Host 的权威
        NetworkServer.ChangeHost(newHostConnection);

        // 4. 更新 Host ID
        UpdateHostId(GetPlayerId(newHostConnection));

        Debug.Log($"[Network] Host migrated from {_disconnectedPlayerId} to {GetPlayerId(newHostConnection)}");
    }

    // ========== 辅助方法 ==========

    private int GetPlayerId(NetworkConnection conn)
    {
        // 从 NetworkConnection 获取玩家 ID
        return conn.identity.netId;
    }

    private int GetHostPlayerId()
    {
        return NetworkServer.localConnection.identity.netId;
    }

    private List<int> GetAllConnectedPlayerIds()
    {
        return _connectedPlayers.Select(c => GetPlayerId(c)).ToList();
    }

    private float GetConnectionQuality(NetworkConnection conn)
    {
        // 计算连接质量（基于延迟和丢包率）
        // 返回 0-1，1 为最佳
        // ⚠️ TODO: 当前实现返回固定值 1f，实际应基于 NetworkTelemetry 计算
        return 1f;
    }

    private float GetConnectionLatency(NetworkConnection conn)
    {
        // 返回 RTT 延迟（毫秒）
        return NetworkTime.GetPing() * 2;
    }

    private void ApplyCheckpoint(ReconnectCheckpoint checkpoint)
    {
        ApplyFullSnapshot(checkpoint.GameState);
    }

    private void ApplyFullSnapshot(GameStateSnapshot snapshot)
    {
        // 恢复所有玩家状态
        foreach (var playerSnap in snapshot.Players)
        {
            var player = FindPlayer(playerSnap.PlayerId);
            if (player != null)
            {
                player.transform.position = playerSnap.Position;
                player.transform.rotation = Quaternion.Euler(0, playerSnap.Rotation, 0);
                // 恢复其他状态...
            }
        }

        // 恢复世界状态
        // ... (调用 WorldMapSystem 等)
    }
}

// 事件定义
public class PlayerDisconnectedEvent
{
    public int PlayerId { get; }
    public PlayerDisconnectedEvent(int playerId) => PlayerId = playerId;
}

public class PlayerReconnectedEvent
{
    public int PlayerId { get; }
    public PlayerReconnectedEvent(int playerId) => PlayerId = playerId;
}

public class ReconnectFailedEvent
{
    public int PlayerId { get; }
    public ReconnectFailedEvent(int playerId) => PlayerId = playerId;
}

public class ReconnectTimeoutEvent
{
    public int PlayerId { get; }
    public ReconnectTimeoutEvent(int playerId) => PlayerId = playerId;
}

public class HostMigrationStartedEvent
{
    public int OldHostId { get; }
    public int NewHostId { get; }
    public HostMigrationStartedEvent(int oldHostId, int newHostId)
    {
        OldHostId = oldHostId;
        NewHostId = newHostId;
    }
}
```

### 8. 同步精度分级

不同游戏事件需要不同的同步精度：

| 事件类型 | 同步精度 | 机制 | 示例 |
|---------|---------|------|------|
| **战斗事件** | 帧级 (30Hz) | 相位快照 | 击杀、伤害、处决 |
| **移动事件** | 帧级 | 位置同步 + 预测 | 玩家移动 |
| **警戒状态** | 秒级 | 状态变化时同步 | Alert State 变化 |
| **NPC 状态** | 秒级 | 状态变化时同步 | NPC 死亡、昏迷 |
| **环境交互** | 秒级 | 事件驱动 | 门被打开 |
| **对话/UI** | 秒级 | 事件驱动 | 对话选择 |

### 9. Unity 项目结构

```
Assets/Game/Infrastructure/Network/
├── Core/
│   ├── NetworkBootstrap.cs          # 网络初始化单例
│   ├── NetworkRoomManager.cs         # 房间管理
│   └── NetworkConstants.cs           # 网络参数常量
├── Synchronization/
│   ├── PhaseManager.cs              # 相位同步管理
│   ├── PlayerInputSync.cs           # 玩家输入同步
│   ├── ClientPrediction.cs          # 客户端预测和校正
│   └── NPCStateSync.cs              # NPC 状态同步
├── Reconnection/
│   ├── ReconnectionManager.cs       # 断线重连管理
│   ├── ReconnectCheckpoint.cs       # 重连检查点数据
│   └── SaveStateBridge.cs           # 存档系统桥接（存档续连）
├── StorageAdapters/
│   ├── SteamNetworkAdapter.cs        # Steam P2P 配对
│   └── PSNNetworkAdapter.cs         # PSN P2P 配对
└── Events/
    ├── PlayerJoinedEvent.cs         # 玩家加入事件
    ├── PlayerLeftEvent.cs           # 玩家离开事件
    ├── HostMigratedEvent.cs         # Host 迁移事件
    └── PhaseSyncEvent.cs            # 相位同步事件
```

---

## Alternatives Considered

### Alternative 1: Client-Server (有服务器)

- **描述**：使用专用服务器作为游戏状态权威源
- **优点**：
  - 服务器权威，一致性简单
  - 无 Host 迁移问题
  - 防作弊更容易
- **缺点**：
  - 需要租用服务器，成本高
  - 额外延迟（所有请求经过服务器）
  - 服务器需要 Sony/Steam 认证
- **拒绝理由**：
  - 独立工作室无力承担服务器月费
  - 2-4 人合作游戏，P2P 延迟更低

### Alternative 2: 无相位同步的 P2P

- **描述**：使用标准 Mirror 状态同步，不使用相位同步
- **优点**：
  - Mirror 默认方案，实现简单
  - 状态同步自然
- **缺点**：
  - 潜行游戏需要精确的状态同步时机
  - 可能出现"穿帮"（客户端 A 看到敌人，客户端 B 没有）
  - 战斗事件不同步（你杀了他，他没死）
- **拒绝理由**：
  - 潜行游戏要求所有玩家在同一时刻看到相同状态
  - 《Hotline Miami》式的战斗需要帧级精度

### Alternative 3: 帧同步 (Lockstep)

- **描述**：所有客户端运行相同帧，完美同步
- **优点**：
  - 完美一致性
  - 无延迟补偿问题
- **缺点**：
  - 任何延迟都导致卡顿
  - 需要锁定帧率
  - 不适合互联网环境
- **拒绝理由**：
  - 帧同步对网络要求过于严格
  - 潜行游戏可以接受秒级状态同步，无需帧级

---

## Consequences

### Positive

- **低延迟**：P2P 直连减少服务器中转延迟
- **帧级精度**：相位同步确保战斗和潜行状态精确同步
- **成本低**：无需服务器，降低运营成本
- **存档续连**：检查点机制支持断线重连后继续

### Negative

- **Host 优势**：Host 有轻微的延迟优势（本地预测）
- **Host 迁移复杂**：Host 离开需要完整迁移流程
- **NAT 穿透问题**：某些网络环境下 P2P 连接困难

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **Host 作弊** | Host 可以修改本地状态 | 关键状态需 Host 验证；报告机制 |
| **NAT 穿透失败** | 某些网络无法建立 P2P | Steam/PSN 中继服务器作为 Fallback |
| **Host 迁移卡顿** | Host 离开时游戏短暂卡顿 | 平滑迁移流程；提前通知 |
| **状态不同步** | 网络波动导致状态不一致 | 相位同步 + 重连检查点恢复 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **带宽占用** | < 50KB/s 每客户端 | 相位快照 + 输入命令 |
| **延迟容忍** | < 300ms | 超过后进入"卡顿"模式 |
| **同步频率** | 30Hz (相位) | 足够 60fps 游戏 |
| **CPU 占用** | < 5% | 同步逻辑开销小 |

---

## Migration Plan

### Phase 1: 基础 P2P
- [ ] 集成 Mirror Networking
- [ ] 实现基础房间管理（创建/加入/离开）
- [ ] 实现 Steam P2P 配对
- [ ] 实现 PSN P2P 配对

### Phase 2: 同步机制
- [ ] 实现 PhaseManager 相位同步
- [ ] 实现 PlayerInputSync 输入同步
- [ ] 实现 NPCStateSync NPC 同步
- [ ] 实现 ClientPrediction 客户端预测

### Phase 3: 高级功能
- [ ] 实现 ReconnectionManager 断线重连
- [ ] 实现 Host 迁移
- [ ] 实现存档续连桥接
- [ ] 实现延迟补偿

### Phase 4: 测试和优化
- [ ] 局域网测试（低延迟）
- [ ] 互联网模拟测试（高延迟）
- [ ] NAT 穿透测试
- [ ] PS5 平台认证测试

---

## Validation Criteria

1. **基本连接**：2-4 名玩家可以成功创建/加入房间
2. **相位同步精度**：所有玩家在同一相位看到相同的 Alert State 变化
3. **战斗同步**：击杀事件在所有客户端同时生效
4. **断线重连**：客户端断线 30 秒内重新连接可恢复游戏
5. **Host 迁移**：Host 离开后游戏继续，新 Host 正确接管
6. **存档续连**：断线重连后加载正确的检查点状态
7. **带宽控制**：每客户端带宽 < 50KB/s
8. **GameStateSnapshot 完整性**：快照包含所有必要的玩家/NPC/世界状态，且不超过容量限制（4 玩家 + 100 NPC）
9. **重连超时处理**：30 秒超时后正确触发 OnReconnectFailed 并提供玩家选择
10. **Host 迁移选择**：新 Host 选择基于连接质量（GetConnectionQuality TODO 实现后验证）

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 网络事件通过 Event Bus 发布
- [ADR-0002: Unity 引擎技术选型](./adr-0002-unity-engine-selection.md) — 确定使用 Mirror P2P
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — 网络系统属于 Infrastructure Layer
- [ADR-0005: 存档/持久化架构](./adr-0005-save-persistence-architecture.md) — 存档续连依赖存档系统
- [NPC AI System GDD](../../design/gdd/npc-ai-system.md) — NPC AI 在 Host 上运行，状态同步给客户端
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 网络系统事件（PlayerJoinedEvent, HostMigrationStartedEvent 等）定义
