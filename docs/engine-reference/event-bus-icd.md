# 接口控制文档：事件总线规范 (Event Bus ICD)

> **版本**: 1.2.2
> **创建日期**: 2026-04-07
> **更新日期**: 2026-04-10
> **状态**: APPROVED - 已通过跨系统接口对齐会议确认
> **基于**: 各系统 GDD 设计文档
> **维护者**: 架构师 + 各系统设计者

---

## 1. 概述

本文档是《断绝：罪恶之源》跨系统事件总线和接口的权威定义。所有系统设计文档中的事件和接口定义应与本文档保持一致。

### 1.1 目标

- 统一事件命名规范，消除歧义
- 明确事件所有权和订阅关系
- 防止接口重复定义或遗漏
- 为程序员提供明确的实现规范

### 1.2 规范状态说明

| 状态 | 含义 |
|------|------|
| **DRAFT** | 草案，待团队确认 |
| **APPROVED** | 已批准，各系统必须遵循 |
| **DEPRECATED** | 已废弃，不应使用 |

---

## 2. 事件命名规范

### 2.1 命名规则

| 类型 | 规则 | 示例 |
|------|------|------|
| **事件 (Event)** | `Subject` + `Did` + `Context` + `Event` | `PlayerDamagedEvent`, `NPCStateChangedEvent` |
| **查询 (Query)** | `Query` + `Subject` | `QueryNPCIdentity`, `QueryAlertState` |
| **请求 (Request)** | `Subject` + `Request` | `DamageRequest`, `DialogueStartRequest` |
| **响应 (Response)** | `Subject` + `Response` | `DialogueResponse` |
| **广播 (Broadcast)** | `Subject` + `Broadcast` | `SharedAlertBroadcast`（仅限内部使用） |

### 2.2 强制要求

1. **所有外部事件必须包含 `Event` 后缀**
2. **使用完整单词**，不使用缩写（如 `Damaged` 而非 `Dmg`）
3. **同一事件只有一个标准名称**，其他均为别名
4. **事件名使用 PascalCase**

### 2.3 事件类型后缀

| 后缀 | 用于 | 示例 |
|------|------|------|
| `Event` | 状态变化/触发事件 | `PlayerDamagedEvent`, `AlertStateChangedEvent` |
| `Request` | 单向请求 | `DamageRequest`, `LevelLoadRequest` |
| `Response` | 请求的响应 | `DialogueResponse` |
| `Query` | 查询接口 | `QueryNPCIdentity` |

---

## 3. 废弃别名列表

以下名称为废弃别名，各系统 GDD 应统一使用标准名称：

| 废弃名称 | 标准名称 | 废弃原因 |
|---------|---------|---------|
| `NPCDeath` | `NPCStateChangedEvent` | NPC死亡是状态变化的一种 |
| `NPCKilled` | `NPCStateChangedEvent` | 统一命名规范 |
| `StateChanged`（外部使用） | `NPCStateChangedEvent` | 对外广播应使用完整命名 |
| `AlertStateChanged`（外部引用） | `AlertStateChangedEvent` | 统一添加 Event 后缀 |
| `PlayerDamaged` | `PlayerDamagedEvent` | 统一添加 Event 后缀 |
| `NPCAlertStateChanged` | `AlertStateChangedEvent` | 统一命名规范 |
| `NPCStateChanged`（对外） | `NPCStateChangedEvent` | 统一命名规范 |

---

## 4. 事件所有权表

每个事件只有一个**拥有者系统**，拥有者负责：
- 定义事件的完整数据结构
- 决定事件何时触发
- 维护事件的文档说明

| 事件名称 | 拥有者 | 事件类型 | 说明 |
|---------|-------|---------|------|
| `PlayerDamagedEvent` | Health System | Health State | 玩家受到伤害 |
| `NPCStateChangedEvent` | Health System | Health State | NPC 状态变化（由 Health System 直接广播，NPC AI 消费） |
| `ArmorDestroyedEvent` | Health System | Health State | NPC 护甲被破坏 |
| `ExplosionAlertEvent` | Health System | Health State | 爆炸造成 NPC 死亡时的警报 |
| `FriendlyFireExplosionEvent` | Health System | Health State | 友军误伤爆炸 |
| `AlertStateChangedEvent` | NPC AI System | AI State | NPC 警觉状态变化 |
| `PlayerSpottedEvent` | LOS System | Perception | 玩家被 NPC 发现 |
| `KeywordCapturedEvent` | LOS System | Perception | 玩家捕获窃听关键词 |
| `NPCIdentityConfirmedEvent` | LOS System | Perception | NPC 身份标签被确认 |
| `PlayerMovementStateChangedEvent` | Player Controller | Player State | 玩家移动状态变化 |
| `LightingStealthBonusChangedEvent` | Lighting System | World State | 阴影隐蔽加成变化 |
| `InteractionEvent` | Gritty Takedowns | Gameplay | 玩家-NPC 交互通用事件 |
| `DamageRequest` | Gritty Takedowns | Request | 伤害请求 |
| `KillTagEvent` | Gritty Takedowns | Gameplay | 击杀标签（带 NPC 身份） |
| `KnowledgeGainedEvent` | Gritty Takedowns | Gameplay | 搜身/审问获取的知识 |
| `ExecutionWitnessedEvent` | Gritty Takedowns | Gameplay | 处决被第三方目击 |
| `EnvironmentalEvent` | Environment System | Gameplay | 环境交互触发 |
| `ClueDiscoveredEvent` | Clue System | Narrative | 发现新线索 |
| `TaskProgressUpdatedEvent` | Clue System | Narrative | 任务进度更新 |
| `NoiseEvent` | Player Controller | Perception | 玩家产生噪音 |
| `VignetteRequest` | Sanity/Rage | Request | 暗角视觉效果请求 |
| `NoiseRequest` | Sanity/Rage | Request | 噪点视觉效果请求 |
| `SaturationRequest` | Sanity/Rage | Request | 饱和度视觉效果请求 |
| `ShakeRequest` | Sanity/Rage | Request | 准星抖动效果请求 |
| `MovementSpeedMultiplier` | Sanity/Rage | Request | 移动速度修改请求 |
| `PsychologicalState` | Sanity/Rage | Event | 心理状态通知 |
| `AudioHapticEvent` | Immersive Audio | Audio | 音频/震动事件触发确认 |
| `AmbienceLayerChangedEvent` | Immersive Audio | Audio | 氛围音层变化 |
| `SaveCompletedEvent` | Save System | Persistence | 存档完成事件 |
| `LoadCompletedEvent` | Save System | Persistence | 加载完成事件 |
| `SaveCorruptedEvent` | Save System | Persistence | 存档损坏事件 |
| `PlayerJoinedEvent` | Network | Multiplayer | 玩家加入事件 |
| `PlayerLeftEvent` | Network | Multiplayer | 玩家离开事件 |
| `PlayerDisconnectedEvent` | Network | Multiplayer | 玩家断线事件 |
| `PlayerReconnectedEvent` | Network | Multiplayer | 玩家重连成功事件 |
| `ReconnectFailedEvent` | Network | Multiplayer | 重连失败事件 |
| `ReconnectTimeoutEvent` | Network | Multiplayer | 重连超时事件 |
| `HostMigrationStartedEvent` | Network | Multiplayer | Host 迁移开始事件 |

---

## 5. 完整事件定义

### 5.1 Health System 事件

#### PlayerDamagedEvent

**发送方**: Health System
**订阅方**: Gritty Takedowns, Sanity/Rage System, Immersive Audio

```csharp
PlayerDamagedEvent:
    player_id: int              // 玩家实体 ID
    damage_type: DamageType     // LETHAL / BLUNT
    hit_location: HitLocation   // HEAD / TORSO / LIMBS
    source_entity_id: int       // 伤害来源实体 ID
    is_lethal: bool             // 是否为致命伤害
```

#### NPCStateChangedEvent

**发送方**: NPC AI System（由 Health System 触发后广播）
**订阅方**: Gritty Takedowns, Clue System, Sanity/Rage System, Immersive Audio

```csharp
NPCStateChangedEvent:
    npc_id: int                 // NPC 实体 ID
    entity_type: EntityType     // NPC（固定值）
    old_state: HealthState      // Healthy / Staggered / Downed / Dead
    new_state: HealthState
    damage_type: DamageType     // LETHAL / BLUNT / NONE（用于死亡）
```

#### ArmorDestroyedEvent

**发送方**: Health System
**订阅方**: VFX System, Audio System

```csharp
ArmorDestroyedEvent:
    npc_id: int                 // NPC 实体 ID
    position: Vector3          // 护甲破坏位置（用于视觉效果放置）
```

#### ExplosionAlertEvent

**发送方**: Health System
**订阅方**: NPC AI System, Audio System

```csharp
ExplosionAlertEvent:
    position: Vector3          // 爆炸位置
    radius: float             // 爆炸半径
    victim_id: int            // 死亡 NPC 的 ID
    killer_is_player: bool     // 凶手是否为玩家
```

#### FriendlyFireExplosionEvent

**发送方**: Health System
**订阅方**: Faction System, UI System

```csharp
FriendlyFireExplosionEvent:
    victim_id: int            // 受害方 NPC ID
    killer_id: int           // 攻击方 NPC ID（友军）
    faction_relation: FactionRelation  // 派系关系
    explosion_position: Vector3
```

---

### 5.2 NPC AI System 事件

#### AlertStateChangedEvent

**发送方**: NPC AI System
**订阅方**: Gritty Takedowns

```csharp
AlertStateChangedEvent:
    npc_id: int                 // NPC 实体 ID
    old_state: AlertState       // UNDETECTED / SUSPECT / SEARCH / ALERT / ESCAPE / COMBAT
    new_state: AlertState
    trigger: AlertTrigger       // PERCEPTION / FACTION / SHARED / EXECUTION
```

**AlertTrigger 枚举值**:

| 值 | 说明 |
|----|------|
| `PERCEPTION` | 感知触发（视觉/听觉/记忆评分变化） |
| `FACTION` | 玩家主动行为（威胁、攻击等） |
| `SHARED` | 派系感知共享 |
| `EXECUTION` | 处决被目击（收到 ExecutionWitnessedEvent） |

---

### 5.3 LOS System 事件

#### PlayerSpottedEvent

**发送方**: LOS System
**订阅方**: NPC AI System

```csharp
PlayerSpottedEvent:
    player_id: int
    npc_id: int                // 发现玩家的 NPC ID
    spot_time: float           // 时间戳
```

#### KeywordCapturedEvent

**发送方**: LOS System
**订阅方**: Clue System, UI System

```csharp
KeywordCapturedEvent:
    keyword: string           // 捕获的关键词文本
    npc_id: int              // 来源 NPC 的 ID
    location_id: string       // 当前位置 ID
    category: KeywordCategory // IDENTITY / LOCATION / RELATIONSHIP / ITEM / TRAGEDY
    capture_timestamp: float  // 捕获时间戳
```

#### NPCIdentityConfirmedEvent

**发送方**: LOS System
**订阅方**: NPC AI System, UI System

```csharp
NPCIdentityConfirmedEvent:
    npc_id: int                    // NPC 实体 ID
    identity_type: NPCIdentityType // ENEMY / ACCOMPLICE / VICTIM
```

---

### 5.4 Gritty Takedowns 事件

#### InteractionEvent

**发送方**: Gritty Takedowns
**订阅方**: NPC AI System, Immersive Audio

```csharp
InteractionEvent:
    type: InteractionType      // STEALTH_KILL / ENVIRONMENT_EXECUTE / THREATEN / BRIBE / SEARCH / INTERROGATE / TIE_UP / RELEASE / CONVERT
    target_id: int            // 目标 NPC ID
    source: System            // 发送系统标识
    result: InteractionResult // SUCCESS / FAIL / INTERRUPTED
```

#### DamageRequest

**发送方**: Gritty Takedowns
**接收方**: Health System

```csharp
DamageRequest:
    target_id: int            // 目标实体 ID
    damage_type: DamageType   // LETHAL / BLUNT
    source: string            // 来源系统标识
```

#### KillTagEvent

**发送方**: Gritty Takedowns
**订阅方**: Sanity/Rage System

```csharp
KillTagEvent:
    npc_id: int
    npc_tag: NPCIdentityType // ENEMY / ACCOMPLICE / VICTIM / UNKNOWN
    is_mistake: bool          // 是否为误杀
```

#### KnowledgeGainedEvent

**发送方**: Gritty Takedowns
**订阅方**: Clue System

```csharp
KnowledgeGainedEvent:
    npc_id: int
    knowledge_list: List[string]  // clue_id 列表
```

#### ExecutionWitnessedEvent

**发送方**: Gritty Takedowns
**订阅方**: NPC AI System

```csharp
ExecutionWitnessedEvent:
    npc_id: int              // 被处决的 NPC
    witness_npc_id: int       // 目击者 NPC
```

---

### 5.5 Environment System 事件

#### EnvironmentalEvent

**发送方**: Environment System
**订阅方**: NPC AI System, Immersive Audio

```csharp
EnvironmentalEvent:
    event_type: EnvEventType // SOUND / EXPLOSION / DESTRUCTION / DISTRACTION / BLOCKING
    position: Vector3        // 事件发生位置
    radius: float            // 影响半径（米）
    duration: float           // 持续时间（秒）
    source_object_id: int     // 来源物件 ID
```

**EnvEventType 枚举值**:

| 值 | 说明 | NPC 反应 |
|----|------|---------|
| `SOUND` | 声响（如打破玻璃） | 短暂注意 |
| `EXPLOSION` | 爆炸 | 惊吓→搜索→战斗 |
| `DESTRUCTION` | 破坏 | 注意→搜索 |
| `DISTRACTION` | 诱饵（如投掷物落地） | 移动到声源 |
| `BLOCKING` | 障碍（如推倒架子） | 绕行/移除 |

---

### 5.6 Clue System 事件

#### ClueDiscoveredEvent

**发送方**: Clue System
**订阅方**: Sanity/Rage System, UI System

```csharp
ClueDiscoveredEvent:
    clue_id: string
    clue_category: ClueCategory // IDENTITY / LOCATION / RELATIONSHIP / ITEM / TRAGEDY
    source_type: ClueSourceType // LOS_EAVESDROP / ENVIRONMENT / NPC_DEATH
    related_npc_ids: List[int]   // 关联 NPC ID 列表
```

#### TaskProgressUpdatedEvent

**发送方**: Clue System
**订阅方**: Mission System

```csharp
TaskProgressUpdatedEvent:
    task_id: string
    completion_percentage: float  // 0.0 - 1.0
    clues_discovered: int
    clues_total: int
```

---

### 5.7 Player Controller 事件

#### NoiseEvent

**发送方**: Player Controller
**订阅方**: NPC AI System

```csharp
NoiseEvent:
    position: Vector3        // 噪音发生的世界坐标
    radius: float            // 噪音广播半径（米）
    noise_type: NoiseType    // WALK / CROUCH / SPRINT / INTERACTION / NONE
    duration: float          // 噪音持续时间（秒），默认 0.5s
    can_interrupt: bool      // 是否可被打断（默认 true）
    source_entity_id: int    // 产生噪音的实体 ID（玩家或其他）
```

#### PlayerMovementStateChangedEvent

**发送方**: Player Controller
**订阅方**: LOS System

```csharp
PlayerMovementStateChangedEvent:
    old_state: PlayerMovementState  // IDLE / WALK / SPRINT / CROUCH / CROUCH_WALK / ACTION
    new_state: PlayerMovementState
```

---

### 5.8 Lighting System 事件

#### LightingStealthBonusChangedEvent

**发送方**: Lighting System
**订阅方**: LOS System

```csharp
LightingStealthBonusChangedEvent:
    exposure_multiplier: float  // 暴露值乘数（1.0 = 无加成，0.77 = 阴影中降低 23%）
```

---

### 5.8 Sanity/Rage System 事件/请求

#### 输出请求（至其他系统）

| 请求名称 | 接收方 | 参数 |
|---------|-------|------|
| `VignetteRequest` | ScreenEffects | intensity: float (0.0 - 0.8) |
| `NoiseRequest` | ScreenEffects | intensity: float (0.0 - 0.5) |
| `SaturationRequest` | ScreenEffects | multiplier: float (0.3 - 1.0) |
| `ShakeRequest` | ScreenEffects | intensity: float (0.0 - 8.0) |
| `MovementSpeedMultiplier` | Player Controller | multiplier: float (1.0 - 1.1) |

#### PsychologicalState

**发送方**: Sanity/Rage System
**订阅方**: Dynamic Post-Processing

```csharp
PsychologicalState:
    state: PsychologicalStateType // CALM / UNEASY / AGITATED / BROKEN / FRENZIED
    sanity_value: int            // 0-100
    rage_value: int              // 0-100
```

---

## 6. 查询接口定义

### 6.1 接口所有权

| 查询名称 | 拥有者 | 返回类型 | 说明 |
|---------|-------|---------|------|
| `QueryState` | NPC AI System | WorldState | 查询 NPC 当前世界状态 |
| `QueryAlertState` | NPC AI System | AlertState | 查询 NPC 警觉状态 |
| `QueryNPCIdentity` | LOS System | NPCIdentity | 查询 NPC 身份标签 |
| `QueryAllegiance` | NPC AI System | int | 查询 NPC 对玩家态度值 |
| `QueryVulnerability` | NPC AI System | List[string] | 查询 NPC 已知弱点列表 |
| `QueryKnowledge` | NPC AI System | List[string] | 查询 NPC 掌握的线索 ID 列表 |

### 6.2 详细接口定义

#### QueryNPCIdentity (LOS System)

```csharp
/// <summary>
/// 查询 NPC 的身份标签（恶徒/帮凶/无辜者/未知）
/// </summary>
/// <param name="npc_id">NPC 的唯一标识符</param>
/// <returns>NPC 身份信息</returns>
QueryNPCIdentity(npc_id: int) -> NPCIdentity

NPCIdentity:
    npc_id: int
    identity: NPCIdentityType  // UNKNOWN / ENEMY / ACCOMPLICE / VICTIM
    confidence: float          // 置信度 0.0 - 1.0
    source_keywords: List[string]  // 导致该身份确认的关键词列表
    last_update_time: float   // 最后更新时间戳
```

#### QueryState / QueryAlertState (NPC AI System)

```csharp
/// <summary>
/// 查询 NPC 当前世界状态
/// </summary>
QueryState(npc_id: int) -> WorldState
// 返回: FREE, UNCONSCIOUS, TIED, DEAD

/// <summary>
/// 查询 NPC 当前警觉状态
/// </summary>
QueryAlertState(npc_id: int) -> AlertState
// 返回: UNDETECTED, SUSPECT, SEARCH, ALERT, ESCAPE, COMBAT
```

---

## 7. 订阅关系矩阵

| 事件 | 发送方 | 订阅方 | 用途 |
|------|-------|-------|------|
| `PlayerDamagedEvent` | Health | Gritty Takedowns | Interacting 状态中被攻击检测 |
| `PlayerDamagedEvent` | Health | Sanity/Rage | 受伤时愤怒上升 |
| `PlayerDamagedEvent` | Health | Immersive Audio | 玩家受伤震动反馈 |
| `NPCStateChangedEvent` | NPC AI | Gritty Takedowns | NPC 世界状态变化（FREE→UNCONSCIOUS/DEAD） |
| `NPCStateChangedEvent` | NPC AI | Clue System | NPC 死亡触发线索缺失标记 |
| `NPCStateChangedEvent` | NPC AI | Sanity/Rage | 目睹 NPC 死亡 |
| `NPCStateChangedEvent` | NPC AI | Immersive Audio | NPC 死亡音效触发 |
| `AlertStateChangedEvent` | NPC AI | Gritty Takedowns | 交互期间状态变化检测 |
| `PlayerSpottedEvent` | LOS | NPC AI | 触发 Alert State 上升 |
| `KeywordCapturedEvent` | LOS | Clue System | 关键词转化为线索 |
| `KeywordCapturedEvent` | LOS | UI | 破译进度显示 |
| `InteractionEvent` | Gritty Takedowns | NPC AI | 威胁/击杀/捆绑触发行为变化 |
| `InteractionEvent` | Gritty Takedowns | Immersive Audio | 交互音效触发 |
| `DamageRequest` | Gritty Takedowns | Health | 执行伤害 |
| `KillTagEvent` | Gritty Takedowns | Sanity/Rage | 击杀类型影响理智 |
| `KnowledgeGainedEvent` | Gritty Takedowns | Clue | 搜身/审问获取线索 |
| `ExecutionWitnessedEvent` | Gritty Takedowns | NPC AI | 处决被目击触发警戒 |
| `EnvironmentalEvent` | Environment | NPC AI | 环境事件触发感知 |
| `EnvironmentalEvent` | Environment | Immersive Audio | 环境音效触发 |
| `ClueDiscoveredEvent` | Clue | Sanity/Rage | 悲剧线索影响理智 |
| `ClueDiscoveredEvent` | Clue | UI | 新线索提示显示 |
| `TaskProgressUpdatedEvent` | Clue | Mission System | 任务进度更新 |
| `NoiseEvent` | Player Controller | NPC AI | 噪音触发 NPC 警觉 |

---

## 8. 数据类型枚举定义

### 8.1 DamageType

```csharp
enum DamageType:
    LETHAL    // 致命伤害（枪击、刺穿、背刺）
    BLUNT     // 钝击伤害（拳击、棍击）
    NONE      // 无伤害（用于状态变化）
```

### 8.2 HitLocation

```csharp
enum HitLocation:
    HEAD
    TORSO
    LIMBS
```

### 8.3 EntityType

```csharp
enum EntityType:
    PLAYER
    NPC
```

### 8.4 HealthState

```csharp
enum HealthState:
    HEALTHY    // 健康
    STAGGERED  // 硬直
    DOWNED     // 倒地/重伤
    DEAD       // 死亡
```

### 8.5 AlertState

```csharp
enum AlertState:
    UNDETECTED  // 未察觉
    SUSPECT     // 怀疑
    SEARCH       // 搜索
    ALERT        // 警戒
    ESCAPE       // 逃跑
    COMBAT       // 战斗
```

### 8.6 NPCIdentityType

```csharp
enum NPCIdentityType:
    UNKNOWN     // 未知
    ENEMY       // 恶徒
    ACCOMPLICE  // 帮凶
    VICTIM      // 无辜者/受害者
```

### 8.7 InteractionType

```csharp
enum InteractionType:
    STEALTH_KILL           // 潜行击杀
    ENVIRONMENT_EXECUTE     // 环境处决
    FINISH_OFF             // 补刀
    THREATEN               // 威胁
    BRIBE                  // 贿赂
    DECEIVE                // 欺骗
    SEARCH                  // 搜身
    INTERROGATE            // 审问
    TIE_UP                 // 捆绑
    RELEASE                 // 解开
    CONVERT                // 转化线人
```

### 8.8 ClueCategory

```csharp
enum ClueCategory:
    IDENTITY        // 身份线索
    LOCATION       // 位置线索
    RELATIONSHIP   // 关系线索
    ITEM           // 物品线索
    TRAGEDY        // 悲剧线索
```

---

## 9. 事件流向图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           事件总线 (Event Bus)                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐     PlayerDamagedEvent      ┌──────────────────┐    │
│  │ Health       │────────────────────────────▶│ Gritty Takedowns │    │
│  │ System       │     NPCStateChangedEvent      │                  │───▶│ NPC AI
│  └──────────────┘────────────────────────────▶│                  │───▶│ Immersive Audio │
│                                                └──────────────────┘    │
│                                                                          │
│  ┌──────────────┐     AlertStateChangedEvent   ┌──────────────────┐    │
│  │ NPC AI      │────────────────────────────▶  │ Gritty Takedowns │    │
│  │ System      │     PlayerSpottedEvent        │                  │    │
│  └──────────────┘────────────────────────────▶  └──────────────────┘    │
│         │                                                            │
│         │     KeywordCapturedEvent                                   │
│         ▼                                                            │
│  ┌──────────────┐                            ┌──────────────────┐    │
│  │ LOS System   │────────────────────────────▶│ Clue System      │    │
│  └──────────────┘                            └──────────────────┘    │
│                                                                          │
│  ┌──────────────┐     EnvironmentalEvent      ┌──────────────────┐    │
│  │ Environment  │────────────────────────────▶  │ NPC AI           │    │
│  │ System       │                            │ System           │    │
│  └──────────────┘                            └──────────────────┘    │
│                                                                          │
│  ┌──────────────┐     NoiseEvent              ┌──────────────────┐    │
│  │ Player      │─────────────────────────────▶  │ NPC AI           │    │
│  │ Controller  │                            │ System           │    │
│  └──────────────┘                            └──────────────────┘    │
│                                                                          │
│  ┌──────────────┐     ClueDiscoveredEvent     ┌──────────────────┐    │
│  │ Clue        │────────────────────────────▶  │ Sanity/Rage      │    │
│  │ System      │                            │ System           │    │
│  └──────────────┘                            └──────────────────┘    │
│                                                                          │
│  ┌──────────────┐     Vignette/Noise/Shake    ┌──────────────────┐    │
│  │ Sanity/Rage │─────────────────────────────▶  │ Screen Effects   │    │
│  │ System      │     MovementSpeedMultiplier──▶ │ Player Controller│    │
│  └──────────────┘                            └──────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. 网络系统事件定义

### 10.1 Save System 事件

#### SaveCompletedEvent

**发送方**: Save System
**订阅方**: UI System

```csharp
SaveCompletedEvent:
    slot_id: string          // 存档槽位 ID
    is_auto_save: bool       // 是否为自动存档
```

#### SaveCorruptedEvent

**发送方**: Save System
**订阅方**: UI System

```csharp
SaveCorruptedEvent:
    slot_id: string          // 损坏的存档槽位 ID
    recovery_status: RecoveryStatus // SUCCESS / FAILED / NO_BACKUP
```

### 10.2 Network System 事件

#### PlayerJoinedEvent

**发送方**: Network
**订阅方**: UI System, All Game Systems

```csharp
PlayerJoinedEvent:
    player_id: int           // 玩家 ID
    player_name: string     // 玩家名称
    is_local: bool          // 是否为本地玩家
```

#### PlayerLeftEvent

**发送方**: Network
**订阅方**: UI System, All Game Systems

```csharp
PlayerLeftEvent:
    player_id: int           // 离开的玩家 ID
    reason: LeaveReason     // DISCONNECTED / KICKED / LEFT
```

#### PlayerDisconnectedEvent

**发送方**: Network
**订阅方**: UI System

```csharp
PlayerDisconnectedEvent:
    player_id: int           // 断线的玩家 ID
    reconnect_timeout: float // 重连超时时间（秒）
```

#### PlayerReconnectedEvent

**发送方**: Network
**订阅方**: UI System, All Game Systems

```csharp
PlayerReconnectedEvent:
    player_id: int           // 重连成功的玩家 ID
    reconnect_time: float    // 重连耗时（秒）
```

#### ReconnectFailedEvent

**发送方**: Network
**订阅方**: UI System

```csharp
ReconnectFailedEvent:
    player_id: int           // 重连失败的玩家 ID
    reason: string           // 失败原因
```

#### ReconnectTimeoutEvent

**发送方**: Network
**订阅方**: UI System

```csharp
ReconnectTimeoutEvent:
    player_id: int           // 超时的玩家 ID
    timeout_seconds: float   // 超时时间
```

#### HostMigrationStartedEvent

**发送方**: Network
**订阅方**: UI System, All Game Systems

```csharp
HostMigrationStartedEvent:
    old_host_id: int         // 原 Host ID
    new_host_id: int        // 新 Host ID
    migration_duration: float // 预计迁移耗时
```

---

## 11. World Map System 事件定义

### 11.1 World Map System 事件所有权

| 事件名称 | 拥有者 | 事件类型 | 说明 |
|---------|-------|---------|------|
| `LocationRevealedEvent` | Clue System | Narrative | 线索揭示新地点 |
| `DiscoveryAnimationCompleteEvent` | UI System | UI | 揭示动画播放完毕 |
| `AreaUnlockEvent` | Narrative System | World State | 解锁锁定地区 |
| `AreaEnteredEvent` | World Map System | World State | 玩家进入地区 |
| `AreaClearedEvent` | NPC AI System | World State | 地区内敌人全灭 |
| `ThreatDissipatedEvent` | NPC AI System | World State | 危险区域威胁消散 |
| `MapWaitingStartedEvent` | World Map System | UI | 玩家开始等待 |
| `MapWaitingCancelledEvent` | World Map System | UI | 玩家取消等待 |
| `LoadingScreenRequestEvent` | World Map System | System | 请求显示加载画面 |
| `MapDisplayRequestEvent` | World Map System | UI | 请求显示世界地图 |
| `AreaInfoRequestEvent` | World Map System | UI | 请求显示地区详情 |
| `LoadCompletedEvent` | Loading Screen System | System | 加载完成，触发状态机转换 |
| `CheckpointRestoreRequestEvent` | Checkpoint System | World State | 被捕后恢复请求 |

---

### 11.2 LocationRevealedEvent

**发送方**: Clue System
**订阅方**: World Map System, Save System

```csharp
LocationRevealedEvent:
    location_id: string           // 新揭示的地点ID
    source_clue_id: string        // 触发揭示的线索ID（可选）
```

---

### 11.3 DiscoveryAnimationCompleteEvent

**发送方**: UI System
**订阅方**: World Map System

```csharp
DiscoveryAnimationCompleteEvent:
    location_id: string           // 动画播放完毕的地点ID
    was_interrupted: bool         // 是否被玩家中断
```

---

### 11.4 AreaUnlockEvent

**发送方**: Narrative System
**订阅方**: World Map System

```csharp
AreaUnlockEvent:
    area_id: string               // 被解锁的地区ID
    unlock_condition: UnlockCondition  // 解锁条件类型
```

```csharp
enum UnlockCondition:
    STORY_PROGRESS    // 剧情进度解锁
    ITEM_REQUIRED     // 需要持有特定道具
    TIME_BASED        // 时间解锁（游戏天计数）
```

---

### 11.5 AreaEnteredEvent

**发送方**: World Map System
**订阅方**: NPC AI System

```csharp
AreaEnteredEvent:
    area_id: string               // 进入的地区ID
    player_id: int                // 玩家实体ID
```

---

### 11.6 AreaClearedEvent

**发送方**: NPC AI System
**订阅方**: World Map System

```csharp
AreaClearedEvent:
    area_id: string               // 被清除的地区ID
    enemy_count: int              // 本次清除的敌人数
    is_full_clear: bool           // 是否为完全清除
```

---

### 11.7 ThreatDissipatedEvent

**发送方**: NPC AI System
**订阅方**: World Map System, UI System

```csharp
ThreatDissipatedEvent:
    area_id: string               // 威胁消散的地区ID
    reason: DissipateReason       // 消散原因
    wait_time_elapsed: float      // 实际等待时间
```

```csharp
enum DissipateReason:
    PROBABILITY_TRIGGER  // 概率触发（每次检测有 30% 概率消散）
    MAX_WAIT_TIMEOUT     // 最大等待时间（30秒）超时后强制消散
```

---

### 11.8 MapWaitingStartedEvent

**发送方**: World Map System
**订阅方**: UI System

```csharp
MapWaitingStartedEvent:
    area_id: string               // 等待所在地区ID
    max_wait_time: float          // 最大等待时间
```

---

### 11.9 MapWaitingCancelledEvent

**发送方**: World Map System
**订阅方**: UI System

```csharp
MapWaitingCancelledEvent:
    area_id: string               // 等待所在地区ID
    reason: CancelReason          // 取消原因
```

```csharp
enum CancelReason:
    PLAYER_MOVED      // 玩家主动移动
    PLAYER_ATTACKED   // 玩家被攻击
    THREAT_ESCALATED  // 威胁升级
```

---

### 11.10 LoadingScreenRequestEvent

**发送方**: World Map System
**订阅方**: Loading Screen System

```csharp
LoadingScreenRequestEvent:
    destination: string           // 目标地区ID
    destination_type: LocationType // 目标类型
    source_location: string       // 来源位置ID
```

```csharp
enum LocationType:
    CITY        // 城市
    AREA        // 地区
    HIDDEN      // 隐藏地点
    MAP         // 返回地图
```

---

### 11.11 MapDisplayRequestEvent

**发送方**: World Map System
**订阅方**: UI System

```csharp
MapDisplayRequestEvent:
    world_state: WorldDisplayState // 世界地图显示状态
    highlighted_area: string      // 高亮显示的地区ID（可选）
```

```csharp
class WorldDisplayState:
    current_location: string       // 玩家当前位置
    game_day_count: int           // 当前游戏天数
    revealed_locations: List[string]  // 已揭示地点列表
```

---

### 11.12 AreaInfoRequestEvent

**发送方**: World Map System
**订阅方**: UI System

```csharp
AreaInfoRequestEvent:
    area_id: string               // 地区ID
    include_enemy_info: bool     // 是否包含敌人密度信息
```

---

### 11.13 CheckpointRestoreRequestEvent

**发送方**: Checkpoint System
**订阅方**: World Map System

> **说明**：此事件由 CheckpointSystem（定义见 ADR-0009）发送，用于在被捕（ARRESTED）状态恢复时通知 World Map System。玩家被制服后，CheckpointSystem 记录检查点并发送此事件，World Map System 据此加载对应位置。

```csharp
CheckpointRestoreRequestEvent:
    area_id: string               // 恢复目标地区ID
    checkpoint_position: Vector3  // 检查点位置
    arrest_location: Vector3     // 被捕位置（用于记录）
```

---

### 11.14 LoadCompletedEvent

**发送方**: Loading Screen System
**订阅方**: World Map System

> **说明**：`RETURNING_TO_MAP` 状态结束后，加载完成时由 Loading Screen System 发送此事件，通知 World Map System 将状态转换为 `MAP_MODE`。此事件替代原有的状态轮询机制，实现事件驱动的状态转换。

```csharp
LoadCompletedEvent:
    destination: string           // 加载完成的目标位置ID
    destination_type: LocationType // 目标位置类型（AREA/CITY）
    was_successful: bool          // 加载是否成功
```

---

### 11.15 LocationType 枚举

```csharp
public enum LocationType
{
    AREA,   // 地区级别
    CITY    // 城市级别
}
```

> **说明**：用于 `LoadingScreenRequestEvent` 和 `LoadCompletedEvent` 的 destination_type 字段，标识目标位置的地理层级。

---

## 12. 修改日志

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-07 | 0.1 | 初稿创建 | 架构师 Agent |
| 2026-04-07 | 1.0.1 | 通过跨系统接口对齐会议，确认所有事件命名规范并更新相关 GDD 文档 | 架构师 Agent |
| 2026-04-10 | 1.1.0 | 补充新增事件定义：ArmorDestroyedEvent, ExplosionAlertEvent, FriendlyFireExplosionEvent, NPCIdentityConfirmedEvent, PlayerMovementStateChangedEvent, LightingStealthBonusChangedEvent；更新 NPCStateChangedEvent 所有权为 Health System | 架构师 Agent |
| 2026-04-10 | 1.2.0 | 补充 World Map System 事件定义：LocationRevealedEvent, DiscoveryAnimationCompleteEvent, AreaUnlockEvent, AreaEnteredEvent, AreaClearedEvent, ThreatDissipatedEvent, MapWaitingStartedEvent, MapWaitingCancelledEvent, LoadingScreenRequestEvent, MapDisplayRequestEvent, AreaInfoRequestEvent | 架构师 Agent |
| 2026-04-10 | 1.2.1 | 补充 CheckpointRestoreRequestEvent 事件定义，完善 World Map System 事件所有权表 | 架构师 Agent |
| 2026-04-10 | 1.2.2 | 补充 DissipateReason 枚举完整定义（PROBABILITY_TRIGGER / MAX_WAIT_TIMEOUT） | 架构师 Agent |
| 2026-04-10 | 1.2.3 | 新增 LoadCompletedEvent 和 LocationType 枚举，完善 World Map 状态转换触发机制 | 架构师 Agent |

---

## 13. 待确认问题

1. [x] `EnvironmentalEvent` 的 `EnvEventType` 枚举值 - ✅ 已定义
2. [x] `KeywordCategory` 与 Clue System 的 `ClueCategory` - ✅ 已统一为 `ClueCategory`
3. [ ] `InteractionEvent.type` 是否需要细分更多类型？ - 待确认
4. [ ] `QueryNPCIdentity` 的 `confidence` 字段如何计算？ - 待确认
5. [x] 网络系统事件定义 - ✅ 已添加（SaveCompletedEvent, PlayerDisconnectedEvent 等）
6. [x] 存档系统事件定义 - ✅ 已添加（SaveCompletedEvent, SaveCorruptedEvent 等）
7. [x] 新增 6 个跨 ADR 依赖事件定义 - ✅ 已补充
8. [x] World Map System 事件定义 - ✅ 已添加（LocationRevealedEvent, AreaClearedEvent, CheckpointRestoreRequestEvent 等）
9. [x] CheckpointRestoreRequestEvent - ✅ 已在 v1.2.1 中添加

---

## 13. 参考文档

- `design/gdd/health-lethality.md`
- `design/gdd/npc-ai-system.md`
- `design/gdd/los-eavesdropping.md`
- `design/gdd/gritty-takedowns.md`
- `design/gdd/environment-interaction.md`
- `design/gdd/clue-and-journal.md`
- `design/gdd/sanity-rage-meter.md`
- `design/gdd/immersive-audio-haptics.md`
- `design/gdd/player-controller.md`
- `docs/architecture/adr-0005-save-persistence-architecture.md` — 存档系统的技术架构
- `docs/architecture/adr-0006-network-synchronization-architecture.md` — 网络同步的技术架构
