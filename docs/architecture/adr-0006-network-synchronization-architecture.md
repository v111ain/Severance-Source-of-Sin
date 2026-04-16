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

    // P0 修复：添加完整 PlayerSave 数据，确保重连时能恢复所有玩家进度
    // 注意：这些字段在相位同步时可能为空（节省带宽），仅在检查点快照时填充
    public List<string> CollectedClues;               // 已收集线索 ID 列表
    public List<string> AcquiredKnowledge;           // 已获取知识 ID 列表
    public Dictionary<string, int> NPCRelationships; // NPC ID → 好感度
    public List<string> UnlockedWeapons;            // 已解锁武器 ID 列表
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
    // 注意：SaveSlotId 已移除（ADR-0005 评审修复 2026-04-15）
    // 断线重连检查点使用 "reconnect_checkpoint" 专用槽位（见 ADR-0005 §4.2）
    // 不再需要 SaveSlotId 字段标识关联的存档槽位
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

#### 参数设计依据

| 参数 | 值 | 计算依据 | 性能影响 |
|------|-----|----------|----------|
| `INPUT_DELAY` | 2 | 客户端延迟 = (RTT / 2) / phase_duration。假设 RTT = 100ms，phase = 33.3ms，则延迟 ≈ 1.5 相位，取整为 2 | 额外 66.6ms 输入延迟 |
| `SNAP_THRESHOLD` | 0.5m | 玩家可感知的位置跳跃阈值。超过 0.5m 的偏差会导致明显跳跃感 | 低于阈值使用插值平滑 |
| `CORRECTION_LERP` | 10f | 每秒校正 10m 的速度。CORRECTION_LERP / frame_rate ≈ 0.17m/frame (60fps) | 保证平滑但不迟钝 |

> **延迟容忍度**：本设计支持 < 300ms RTT 的网络环境。超过 300ms RTT 时：
> - `INPUT_DELAY` 自动调整为 ceil(RTT / (2 * phase_duration)) + 1
> - 额外输入延迟 = 客户端 RTT / 2
> - 最大容忍延迟：500ms（之后客户端进入"卡顿补偿模式"）

```csharp
// ClientPrediction.cs
public class ClientPrediction : NetworkBehaviour
{
    // 延迟补偿参数
    // 计算公式: INPUT_DELAY = ceil(RTT / (2 * phase_duration)) + safety_margin
    // 假设 RTT = 100ms, phase_duration = 33.3ms, safety_margin = 1
    // 则 INPUT_DELAY = ceil(100 / (2 * 33.3)) + 1 = ceil(1.5) + 1 = 3
    // 实际使用中取保守值 2 以优化手感
    private const int INPUT_DELAY = 2;

    // 校正参数
    private const float SNAP_THRESHOLD = 0.5f;   // 超过 0.5m 直接跳转
    private const float CORRECTION_LERP = 10f;  // 校正插值速度

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
        // 注意：SaveSlotId 已移除，ReconnectCheckpoint 不再存储存档槽位信息
        // 断线重连检查点使用 "reconnect_checkpoint" 专用槽位（ADR-0005 §4.2）
        _lastCheckpoint = new ReconnectCheckpoint
        {
            SessionId = _sessionId,
            Timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds(),
            GameState = _phaseManager.CaptureGameStateSnapshot(),
            HostPlayerId = GetHostPlayerId(),
            ConnectedPlayerIds = GetAllConnectedPlayerIds()
        };

        // 通过存档系统保存检查点（使用 "reconnect_checkpoint" 专用槽位）
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
        if (conn == null || !conn.isReady) return 0f;

        // 使用 NetworkTime.GetPing() 获取 RTT（毫秒）
        var rtt = NetworkTime.GetPing(conn) * 2; // 单程延迟 = RTT/2
        var packetLoss = conn.packetLoss; // 丢包率 0-1

        // 将 RTT 转换为质量分数 (0-1)
        // 假设 0ms = 1.0, 500ms+ = 0.0
        var rttQuality = Mathf.Clamp(1f - (rtt / 500f), 0f, 1f);

        // 丢包率直接影响质量
        var lossQuality = 1f - packetLoss;

        // 综合质量 = 0.7*RTT + 0.3*丢包
        return 0.7f * rttQuality + 0.3f * lossQuality;
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
```

**`reconnect_checkpoint` 专用槽位策略说明**：

`SaveCheckpointAsync()` 使用的 `"reconnect_checkpoint"` 专用槽位具有以下特性：

| 特性 | 策略 |
|------|------|
| **槽位数量** | 始终单槽，不参与多槽位轮换 |
| **本地备份** | 是，保留在本地存储 |
| **云同步** | **不上云**，避免与本地重连检查点冲突 |
| **覆盖时机** | 每次新的断线重连检查点保存时自动覆盖 |
| **重连成功后** | 由 Network 系统决定是否清除（通常保留用于下次快速重连） |

> **设计理由**：断线重连检查点是瞬态数据（transient），不应该与玩家的持久存档混淆，因此使用独立槽位且不上云。

> **重连检查点与完整存档的合并策略**（ADR 评审修复 2026-04-15）：
> 当玩家重连后选择"继续游戏"时，检查点数据与完整存档的合并逻辑如下：
> 1. **优先使用检查点**：如果存在有效的 `reconnect_checkpoint`，使用检查点的 `GameStateSnapshot` 恢复游戏状态
> 2. **完整存档作为 Fallback**：如果检查点已过期或损坏，回退到玩家最近一次完整存档
> 3. **数据选择性合并**：`GameStateSnapshot` 中的玩家状态、NPC 状态直接采用；世界状态（如地区探索进度）从完整存档读取并与检查点合并
> 4. **检查点清理**：成功重连后，`reconnect_checkpoint` 槽位被标记为可覆盖（但保留直到下次保存覆盖）

> **网络同步事件定义**：以下网络同步专用事件定义于 `shared-types.md §21`，本 ADR 仅做索引引用：
> - `PlayerDisconnectedEvent` — 玩家断开连接
> - `PlayerReconnectedEvent` — 玩家重连成功
> - `ReconnectFailedEvent` — 重连失败
> - `ReconnectTimeoutEvent` — 重连超时
> - `HostMigrationStartedEvent` — Host 迁移开始
>
> 实现时应引用 `shared-types.md` 中的权威定义，本文档不做重复定义。
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
| **Host 作弊** | Host 可以修改本地状态 | 关键状态需 Host 验证；报告机制；输入时间戳校验；关键状态 Server 校验框架（见 §10） |
| **NAT 穿透失败** | 某些网络无法建立 P2P | Steam/PSN 中继服务器作为 Fallback |
| **Host 迁移卡顿** | Host 离开时游戏短暂卡顿 | 平滑迁移流程；提前通知 |
| **状态不同步** | 网络波动导致状态不一致 | 相位同步 + 重连检查点恢复 |

### 10. 反作弊技术实现

> **设计说明**：P2P 架构下 Host 具有天然优势（本地状态可修改）。本节定义关键验证点，降低作弊发生率。完整反作弊系统需在上线前持续迭代，本节提供基础框架。

#### 10.1 AntiCheatManager

```csharp
// AntiCheatManager.cs
public class AntiCheatManager : MonoBehaviour
{
    // 输入时间戳检测窗口（毫秒）
    private const int INPUT_TIMESTAMP_WINDOW = 500;

    // 关键状态 Server 校验阈值
    private const float HEALTH_DEVIATION_THRESHOLD = 0.02f; // 2% 偏差允许（原 10% 过宽）
    private const float MAX_ALLOWED_SPEED = 10f; // m/s，超过视为瞬移

    /// <summary>
    /// 验证玩家输入时间戳是否合理（防止输入延迟作弊）
    /// </summary>
    public bool ValidateInputTimestamp(float clientTimestamp, float serverTime)
    {
        var deviation = Mathf.Abs(serverTime - clientTimestamp);
        if (deviation > INPUT_TIMESTAMP_WINDOW / 1000f)
        {
            Debug.LogWarning($"[AntiCheat] Input timestamp deviation {deviation:F3}s exceeds threshold");
            return false;
        }
        return true;
    }

    /// <summary>
    /// 校验玩家关键状态（由 Host 在关键节点调用）
    /// </summary>
    public bool ValidateCriticalState(int playerId, PlayerSnapshot clientState, PlayerSnapshot serverState)
    {
        // 校验生命值
        var healthDeviation = Mathf.Abs(clientState.Health - serverState.Health) / Mathf.Max(serverState.Health, 1f);
        if (healthDeviation > HEALTH_DEVIATION_THRESHOLD)
        {
            Debug.LogWarning($"[AntiCheat] Player {playerId} health deviation {healthDeviation:P} exceeds threshold");
            EventBus.Instance.Publish(new PlayerCheatDetectedEvent(playerId, CheatType.HEALTH_TAMPERING));
            return false;
        }

        // 校验位置（瞬移检测）
        // 使用 timestamp 计算速度 = distance / timeDelta
        var distance = Vector3.Distance(clientState.Position, serverState.Position);
        var timeDelta = (clientState.Timestamp - serverState.Timestamp) / 1000f; // 转换为秒
        var speed = timeDelta > 0 ? distance / timeDelta : 0f;

        if (speed > MAX_ALLOWED_SPEED)
        {
            Debug.LogWarning($"[AntiCheat] Player {playerId} potential teleport detected: {speed:F1}m/s over {timeDelta:F3}s");
            EventBus.Instance.Publish(new PlayerCheatDetectedEvent(playerId, CheatType.TELEPORT));
            return false;
        }

        return true;
    }

    /// <summary>
    /// 校验客户端上报的伤害值是否合理
    /// </summary>
    public bool ValidateDamageRequest(int attackerId, float damage, Vector3 hitPosition)
    {
        // 校验伤害值上限（防止放大伤害）
        var maxDamage = GetWeaponMaxDamage(attackerId); // 从权威数据获取
        if (damage > maxDamage * 1.2f) // 20% 容差
        {
            Debug.LogWarning($"[AntiCheat] Player {attackerId} suspicious damage value: {damage:F1} > max {maxDamage:F1}");
            return false;
        }
        return true;
    }
}

public enum CheatType
{
    HEALTH_TAMPERING,
    TELEPORT,
    SPEED_HACK,
    AIMBOT_SUSPECTED,
    DAMAGE_AMPLIFICATION
}

public struct PlayerCheatDetectedEvent
{
    public int PlayerId;
    public CheatType CheatType;
    public float Timestamp;
}
```

#### 10.2 调用时机

| 校验点 | 调用时机 | 校验内容 |
|--------|----------|----------|
| 输入时间戳 | 每次收到 `PlayerInputCommand` | `ValidateInputTimestamp` |
| 伤害事件 | 每次收到 `DamageEvent` | `ValidateDamageRequest` |
| 状态快照同步 | 每次 `GameStateSnapshot` 接收（关键节点） | `ValidateCriticalState` |

#### 10.3 局限性

- **无法防止 Host 本地完整状态修改**：Host 可修改一切本地数据，Server 校验只能检测异常模式
- **合理阈值需要上线后调优**：`HEALTH_DEVIATION_THRESHOLD` 和 `MAX_ALLOWED_SPEED` 需要根据实际数据调整
- **隐蔽作弊难以检测**：如"轻微加速"、"小幅生命修改"等可能绕过检测

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

## Dependencies [已修复]

### Network 与 SaveManager 依赖关系

Network 系统依赖 SaveManager 实现断线重连检查点保存：

| 依赖方向 | 说明 |
|---------|------|
| Network → SaveManager | Network 的 `ReconnectionManager` 调用 `SaveManager.Instance.SaveCheckpointAsync()` 保存重连检查点 |
| SaveManager → ResourceManager | SaveManager 依赖 ResourceManager 加载存档资源（场景数据、角色状态等） |

**依赖实现位置**：
- `ReconnectionManager.SaveCheckpointToSaveSystem()`（第 682-698 行）调用 `SaveManager.Instance.SaveCheckpointAsync()`
- 详见 ADR-0005 §4.2 关于 `reconnect_checkpoint` 专用槽位的定义

**调用时序**：
1. 客户端断开连接 → `ReconnectionManager.OnClientDisconnect()`
2. 保存检查点 → `SaveManager.Instance.SaveCheckpointAsync()`
3. 等待重连（30 秒超时）

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 网络事件通过 Event Bus 发布
- [ADR-0002: Unity 引擎技术选型](./adr-0002-unity-engine-selection.md) — 确定使用 Mirror P2P
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — 网络系统属于 Infrastructure Layer
- [ADR-0005: 存档/持久化架构](./adr-0005-save-persistence-architecture.md) — 存档续连依赖存档系统
- [NPC AI System GDD](../../design/gdd/npc-ai-system.md) — NPC AI 在 Host 上运行，状态同步给客户端
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 网络系统事件（PlayerJoinedEvent, HostMigrationStartedEvent 等）定义
