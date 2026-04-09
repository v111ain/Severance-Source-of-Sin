# 世界地图与非线性叙事系统 (World Map & Non-Linear Progression)

> **Status**: Approved
> **Author**: Game Designer
> **Last Updated**: 2026-04-08 (P0/P1/P2修复 + 设计审查修复)
> **Implements Pillar**: 致命的脆弱感 (Lethal Fragility)、罪恶的深度 (Depth of Sin)
>
> **2026-04-08 设计审查修复**：
> - ✅ 高优先级：npc-ai-system.md 双向依赖修复（补充 `AreaCleared` 事件流出目标）
> - ✅ 中优先级：游戏天计时机制澄清（移除"进入地区推进"的矛盾表述）
> - ✅ 中优先级：敌人类型命名统一（BOSS/关键NPC → 关键NPC）
> - ✅ 中优先级：`initial_areas` 变量定义补充说明
> - ✅ 低优先级：揭示动画输入屏蔽职责明确（UI系统负责屏蔽）

## Overview

世界地图与非线性叙事系统是《断绝：罪恶之源》的导航与进度框架。玩家在连续俯视角的世界地图上自由移动，探索一个被犯罪网络笼罩的小型国家。游戏世界以「国家 → 城市 → 地区」三层地理结构组织——玩家可以看到整个国家的版图，在城市间徒步穿行，当进入特定城市或地区时触发加载画面。地图完全对玩家开放，玩家可以自由选择追查哪条线索、前往哪个区域，但某些隐藏地点（如秘密接头点、帮派藏身处）需要通过收集特定线索来揭示其存在。玩家的复仇之路由自己的好奇心和判断驱动，而非线性的任务序列。

**与游戏支柱的连接**：
- **致命的脆弱感**：地图上存在高危区域（如帮派火拼中的地盘），玩家在赶路时也可能遭遇危险
- **罪恶的深度**：地图展现的是一个人口贩卖网络笼罩的城市生态，玩家在地图上的每一次移动都能感受到这个世界的腐败氛围

## Player Fantasy

玩家面对世界地图时的核心情感是**「在黑暗中摸索的猎人」**——他们能看到整个猎场的轮廓，知道目标就在这片土地的某处，但需要靠自己去嗅出线索、拼凑路径。

**三种核心感受**：

| 阶段 | 情感 | 来源 |
|------|------|------|
| **凝视地图** | 好奇心 + 压迫感 | 地图展示了犯罪网络的规模，这种"庞然大物"感既是挑战也是动力 |
| **自由穿行** | 掌控感 + 脆弱感 | 玩家自己选择去哪，但地图上的危险区域提醒他们这不是观光旅行 |
| **揭示隐藏** | 揭秘的快感 + 道德纠结 | 发现新地点时既是收获，也可能面对"这里发生过什么悲剧"的沉重 |

**设计意图**：世界地图不只是导航工具，它是「罪恶的深度」支柱的第一层体验——玩家在踏上地图的那一刻就已经置身于这个腐烂的世界之中。

## Detailed Design

### Core Rules

#### 3.1 地理层级结构

```
[国家] 小型虚构国家 "毒雾湾"（Gulf of Poison）
  │
  ├── [城市1] 锈港（Rust Harbor）— 港口工业区
  │     ├── 地区1.1 旧码头区
  │     ├── 地区1.2 废品回收站
  │     └── 地区1.3 船员宿舍区
  │
  ├── [城市2] 灰桥（Ash Bridge）— 商业中心
  │     ├── 地区2.1 中央市场
  │     ├── 地区2.2 市政广场
  │     └── 地区2.3 后巷街区
  │
  ├── [城市3] ...（共5-8个城市）
  │
  └── [隐藏地区] 秘密接头点（需线索揭示）
```

**地理层级定义**：

| 层级 | 定义 | 玩家交互 |
|------|------|---------|
| **国家** | 整个可探索区域的总览 | 显示在世界地图上，可看到城市轮廓和路线 |
| **城市** | 具有独特氛围和NPC派系的中等区域 | 进入时触发加载画面，城市内有多个可探索地区 |
| **地区** | 具体的关卡场景（俯视角潜行区域） | 进入时触发加载画面，是实际游戏发生的地点 |
| **隐藏地区** | 在地图上不可见，需要线索揭示的特殊地点 | 揭示后如同普通地区，但有其独特进入条件 |

#### 3.2 地区探索流程

**标准流程**：
```
[地图模式] ──选择地区──▶ [加载画面] ──▶ [地区内模式] ──▶ [撤离成功] ──▶ [加载画面] ──▶ [地图模式]
                              │
                              │
                         [死亡/被捕]
                              │
                              ▼
                         [加载画面] ──▶ [最近检查点/地图模式]
```

**关键设计**：地区探索结束后，玩家**返回地图模式**，而非重新开始。这是替代线性关卡的核心——进度不会因为死亡而丢失。

#### 3.3 解锁机制

| 地区类型 | 解锁方式 | 示例 |
|---------|---------|------|
| **初始可见可进** | 无条件 | 锈港的所有地区、灰桥的所有地区 |
| **初始可见但有进入条件** | 需要满足条件（如钥匙、帮派关系） | 被封锁的帮派总部 |
| **初始不可见，线索揭示** | 收集特定线索后揭示 | 秘密接头点、人口贩卖转运点 |

**线索揭示机制**：
- 当玩家收集特定线索时，系统检查该线索是否关联隐藏地点
- 如果关联，则在地图上揭示该地点（播放发现动画）
- 揭示后的隐藏地区与普通地区行为一致

#### 3.4 地图移动与遭遇

| 遭遇类型 | 描述 | 玩家选择 |
|---------|------|---------|
| **环境威胁** | 帮派火拼区，玩家需要绕道或等待 | 绕道（消耗时间 `DangerZoneMoveDelay`）/ 强行通过（触发战斗/风险）/ 等待（原地消耗时间后威胁消散） |
| **NPC对话** | 路人NPC的对话，包含线索或情报 | 窃听 / 忽略 |
| **可选战斗** | 遭遇落单的敌人，可以选择清除或避开 | 击杀（获得道具/影响派系）/ 潜行通过 |
| **安全移动** | 无事件，正常移动到目的地 | 无 |

**"等待"机制详解**：

当玩家在危险区域选择"等待"时：
1. 玩家在原地停止移动，弹出等待UI（显示剩余等待时间和威胁消散概率）
2. 每帧更新等待计时器，按 `WaitCheckInterval` 检测威胁是否消散
3. 等待超过最大阈值（`MaxWaitTime`）后，威胁强制消散
4. 等待期间玩家保持隐蔽状态（不触发敌人发现）

**等待机制实现细节**：

```lua
OnPlayerChooseWait():
    player.SetMovementEnabled(false)
    wait_timer = 0.0
    threat_active = true

    while threat_active AND wait_timer < MaxWaitTime:
        Wait(WaitCheckInterval)
        wait_timer += WaitCheckInterval

        if RandomFloat(0.0, 1.0) < ThreatDissipateProbability:
            threat_active = false
            TriggerThreatDissipatedEvent()
        else:
            UpdateWaitUI(wait_timer, MaxWaitTime)

    if threat_active AND wait_timer >= MaxWaitTime:
        -- 超时强制消散
        threat_active = false
        TriggerThreatDissipatedEvent()

    player.SetMovementEnabled(true)
```

**并发攻击处理**：
- 等待期间，如果敌人主动接近玩家并进入攻击范围，威胁判定**立即升级**
- 玩家可以选择"中止等待"（取消隐蔽状态）进行战斗或继续等待
- 等待期间如果玩家主动移动，自动退出等待状态

| 参数 | 默认值 | 说明 |
|------|-------|------|
| `WaitCheckInterval` | 0.5秒 | 检测威胁消散的间隔（服务器端Tick） |
| `ThreatDissipateProbability` | 0.3 | 每次检测时威胁消散的概率 |
| `MaxWaitTime` | 30.0秒 | 最大等待时间，超时后威胁强制消散 |

**设计意图**：地图移动不强制战斗，但提供有意义的选择——这强化了玩家的「掌控感」，同时「环境威胁」保留了「致命的脆弱感」。

### States and Transitions

#### 地区探索状态机

```
[LOCKED] ──满足条件──▶ [AVAILABLE] ──进入地区──▶ [EXPLORING]
                                                    │
                                          ┌─────────┼─────────┐
                                          │         │         │
                                          ▼         ▼         ▼
                                    [EXTRACTED] [DIED] [CAPTURED]
                                          │         │         │
                                          └─────────┴─────────┘
                                                        │
                                                        ▼
                                                 [RETURNING_TO_MAP]
                                                        │
                                                        ▼
                                                   [MAP_MODE]
```

| 状态 | 描述 | 入口条件 | 出口条件 | 行为 |
|------|------|---------|---------|------|
| LOCKED | 地区锁定，不可进入 | 未满足进入条件 | 满足条件后自动转入 AVAILABLE | 显示锁定图标 |
| AVAILABLE | 地区可用，可以进入 | 满足进入条件（初始开放或线索揭示） | 玩家选择进入 | 显示可进入状态 |
| EXPLORING | 玩家正在探索地区 | 玩家进入地区，加载完成 | 撤离成功/死亡/被捕 | 正常游戏逻辑 |
| EXTRACTED | 玩家成功撤离 | 满足撤离条件（无敌人或击杀所有敌人） | 返回地图后标记为 COMPLETED | 保存进度，返回地图 |
| DIED | 玩家死亡 | 玩家生命值归零 | 等待复活/加载完成 | 显示死亡画面，重生于检查点 |
| CAPTURED | 玩家被捕获 | 玩家被敌人制服 | 被迫读取进度 | 显示被捕动画 |
| RETURNING_TO_MAP | 返回地图中 | 撤离成功/死亡/被捕后 | 加载完成 | 显示加载画面 |
| MAP_MODE | 地图模式 | 任何可以返回地图的状态 | 玩家选择新地区 | 世界地图界面 |

#### 地区完成状态

| 状态 | 描述 | 可否重新进入 |
|------|------|------------|
| UNEXPLORED | 未探索 | 可以 |
| EXPLORED | 已探索但未完成核心目标 | 可以 |
| COMPLETED | 完成核心目标（如击杀关键NPC、获取关键道具） | 可以（重复探索） |
| CLEARED | 清除所有敌人 | 可以（敌人会重生） |

**状态区别说明**：
- `COMPLETED` 由玩家主动触发（通常通过触发撤离点），与敌人清除无关
- `CLEARED` 由NPC AI系统发送 `AreaCleared` 事件，仅表示敌人全灭
- 一个地区可以同时是 COMPLETED 和 CLEARED（完成了核心目标且清除了敌人）
- 一个地区可以是 CLEARED 但不是 COMPLETED（清除了敌人但没完成核心目标）

**关键设计**：地区不会"永久完成"——玩家可以随时返回任何已探索的地区。这是非线性叙事的基础。

#### 3.5 敌人重生机制

**敌人分类**：

| 类型 | 描述 | 重生规则 |
|------|------|---------|
| **普通敌人** | 地图上随机生成的杂兵 | 每次进入地区时重新生成 |
| **精英敌人** | 区域性固定敌人 | 每7游戏天重生一次（如果被击杀） |
| **BOSS/关键NPC** | 剧情关键角色 | 永久死亡，不重生 |

**清除判定**：

```
OnAreaCleared(area_id):
    area_save.exploration_state = CLEARED
    area_save.enemies_killed_count += enemies_defeated_this_visit
```

**重生生效条件**：
- 玩家离开地区后再次进入
- 距离上次清除经过了一定时间（精英敌人）

```
CanRespawnEnemies(area_id) =
    area_save.exploration_state == CLEARED
    AND last_cleared_time + respawn_interval <= current_time
    AND enemy_type != BOSS
```

#### 城市状态

| 状态 | 描述 |
|------|------|
| REVEALED | 城市在地图上可见 |
| ACCESSIBLE | 玩家可以前往（无进入条件限制） |
| LOCKED | 玩家暂时无法前往（如剧情锁定） |

### Interactions with Other Systems

#### 数据流入 (Inputs)

| 来源系统 | 数据内容 | 处理方式 |
|---------|---------|---------|
| **线索系统** | `LocationRevealed(location_id)` | 将 location_id 加入 revealed_locations，播放发现动画 |
| **玩家控制器** | `PlayerDeath()` | 触发 DIED 状态，返回地图 |
| **NPC AI系统** | `AreaCleared(area_id)` | 更新地区状态为 CLEARED。当地区内所有敌人被清除时，NPC AI系统发送此事件 |
| **环境交互系统** | `ExitZoneReached()` | 触发撤离判定，满足条件则 EXTRACTED |
| **加载画面系统** | `LoadingComplete(destination)` | 加载完成后，确认玩家已进入目标地区，触发 EXPLORING 状态 |

#### 数据流出 (Outputs)

| 目标系统 | 发送数据 | 说明 |
|---------|---------|------|
| **存档系统** | `AreaStateUpdate(area_id, state)` | 地区状态变更时同步（内部事件，触发存档保存） |
| **存档系统** | `LocationRevealed(location_id)` | 隐藏地点揭示时同步 |
| **UI系统** | `MapDisplayRequest(world_state)` | 请求显示世界地图 |
| **UI系统** | `AreaInfoRequest(area_id)` | 请求显示地区详情 |
| **加载画面系统** | `LoadingScreenRequest(destination)` | 进入/离开地区时请求加载 |
| **NPC AI系统** | `AreaEntered(area_id)` | 玩家进入某地区，NPC AI系统订阅此事件后自行决定是否生成敌人 |

#### 接口所有权

| 接口 | 拥有者 | 流向 |
|------|-------|------|
| `LocationRevealed` 事件 | 线索系统 | 线索系统 → 世界地图系统 |
| `AreaStateUpdate` 事件 | 世界地图系统 | 世界地图系统 → 存档系统（内部事件） |
| `LocationRevealed` 事件（存档） | 世界地图系统 | 世界地图系统 → 存档系统 |
| `MapDisplayRequest` 查询 | UI系统 | 世界地图系统 → UI系统 |

#### 关键事件定义

**AreaEntered 事件**

| 属性 | 值 |
|------|-----|
| 发送者 | 世界地图系统 |
| 接收者 | NPC AI系统 |
| 触发时机 | 加载画面完成后，玩家正式进入地区时 |
| 携带数据 | `area_id: String` — 进入的地区ID |
| 用途 | NPC AI系统根据 `area_id` 决定生成哪些敌人 |
| 响应方式 | NPC AI系统自行决定生成逻辑，世界地图系统不等待响应 |

**LocationRevealed 事件**

| 属性 | 值 |
|------|-----|
| 发送者 | 世界地图系统 |
| 接收者 | UI系统、存档系统 |
| 触发时机 | 线索系统发送 `LocationRevealed` 事件后，世界地图系统处理完毕 |
| 携带数据 | `location_id: String` — 新揭示的地点ID |
| 响应方式 | UI系统播放揭示动画；存档系统持久化 |

**DiscoveryAnimationComplete 事件**

| 属性 | 值 |
|------|-----|
| 发送者 | UI系统 |
| 接收者 | 世界地图系统 |
| 触发时机 | 揭示动画播放完毕（正常完成或被玩家中断） |
| 携带数据 | `location_id: String` — 动画播放完毕的地点ID |
| 响应方式 | 世界地图系统解除对玩家输入的阻塞 |

## Formulas

### 4.1 存档数据结构

```lua
SaveData:
    save_version: String                    -- 存档格式版本，如 "1.0.0"
    timestamp: Integer                      -- Unix时间戳
    playtime_seconds: Integer               -- 累计游玩时间

    world: WorldSave                        -- 世界探索状态
    revealed_locations: Set<location_id>   -- 已揭示的隐藏地点（世界地图系统所有）

WorldSave:
    cities: Dict[city_id, CitySave]
    areas: Dict[area_id, AreaSave]
    current_location: location_id          -- 玩家最后所在位置

CitySave:
    city_id: String
    is_accessible: Boolean                 -- 是否可以进入

AreaSave:
    area_id: String
    exploration_state: Enum                -- UNEXPLORED / EXPLORED / COMPLETED / CLEARED
    completion_count: Integer               -- 完成次数
    is_unlocked: Boolean                   -- 是否解锁
```

### 4.2 地区进入判定

```lua
CanEnterArea(area_id) =
    is_unlocked(area_id) == true
    AND MeetsEntryRequirements(area_id)
    AND (area_id ∈ revealed_locations OR area_id ∈ initial_areas)

MeetsEntryRequirements(area_id) =
    ALL requirements FOR area_id ARE satisfied

-- 进入条件类型示例：
-- 钥匙条件：PlayerHasItem(required_item_id)
-- 派系关系：PlayerFactionRelation(faction_id) >= required_relation
-- 剧情状态：StoryFlagIsSet(story_flag_id)
```

| 变量 | 类型 | 说明 |
|------|------|------|
| is_unlocked | Boolean | 地区是否已解锁（剧情/进度锁定） |
| requirements | List | 进入地区需要满足的条件列表 |
| revealed_locations | Set | 已揭示的隐藏地点集合 |
| initial_areas | Set | 初始开放地区集合（由策划在关卡编辑器中配置） |

**进入条件类型**：

| 条件类型 | 检查方式 | 示例 |
|---------|---------|------|
| **钥匙/道具** | `PlayerHasItem(item_id)` | 需要锈港钥匙才能进入帮派总部 |
| **派系关系** | `PlayerFactionRelation(faction) >= threshold` | 与凋亡议会关系达到"友好"才能进入议会大厅 |
| **剧情标记** | `StoryFlagIsSet(flag_id)` | 完成前置任务后自动解锁 |
| **时间窗口** | `CurrentTimeInRange(start, end)` | 只在夜间开放的地点 |

### 4.3 线索揭示判定

> **设计说明**：线索 → 地点映射表（`clue_reveals: Dict[clue_id, location_id]`）由**线索系统**维护。线索系统根据此映射在适当时机发送 `LocationRevealed(location_id)` 事件。世界地图系统只负责接收事件并更新 `revealed_locations`。

```lua
OnLocationRevealed(location_id):
    if location_id ∉ revealed_locations:
        AddToSet(revealed_locations, location_id)
        TriggerDiscoveryEvent(location_id)  -- 通知UI系统播放发现动画
        SaveToSaveData()                    -- 持久化

-- TriggerDiscoveryEvent 由世界地图系统发送，由UI系统接收并负责播放动画
-- UI系统在接收到事件后：
-- 1. 执行揭示动画（地图聚焦、标记弹出、脉冲闪烁）
-- 2. 动画完成后发送 DiscoveryAnimationComplete 事件
-- 3. 动画可被玩家中断（点击确认直接完成）

-- DiscoveryAnimationComplete 事件定义
-- 发送者：UI系统
-- 接收者：世界地图系统
-- 携带数据：location_id
-- 用途：动画完成后，世界地图系统解除对玩家的输入阻塞
```

**数据流出澄清**：
- `AreaStateUpdate(area_id, state)` 是世界地图系统**内部**的状态变更操作，通过存档系统的内部接口触发保存，**不是**独立的发布-订阅事件
- 对外发布的事件只有 `LocationRevealed(location_id)`，用于通知存档系统和UI系统
- 接口表中的 `AreaStateUpdate` 流向是指"存档系统需要响应此数据变更"，而非"接收一个独立事件"

**揭示动画期间输入屏蔽职责**：
- **职责归属**：UI系统负责在揭示动画播放期间屏蔽玩家输入
- **屏蔽范围**：地图交互输入（选择地区、查看详情等）
- **解除时机**：UI系统播放完毕后发送 `DiscoveryAnimationComplete` 事件，世界地图系统收到后解除对玩家输入的阻塞
```

**数据归属**：
- `clue_reveals` 映射表：归线索系统所有（存储在线索系统数据中）
- `revealed_locations` 集合：归世界地图系统所有（存储在世界地图存档中）
- `LocationRevealed` 事件：由线索系统主动推送，世界地图被动接收

### 4.4 地区状态转换

```lua
-- 探索完成时
OnAreaExtracted(area_id):
    if area_save.exploration_state == UNEXPLORED:
        area_save.exploration_state = EXPLORED
    else if area_save.exploration_state == EXPLORED:
        area_save.exploration_state = COMPLETED
    -- 如果已经是 COMPLETED 或 CLEARED 状态，保持不变（不重复升级）

    area_save.completion_count += 1

-- 地区清除时
OnAreaCleared(area_id):
    area_save.exploration_state = CLEARED
    -- 清除状态优先级高于完成状态
    -- 即使地区已完成核心目标，被清除后仍标记为 CLEARED
```

### 4.5 存储空间估算

```
EstimatedSaveSize =
    1KB                                              -- save_version, timestamp, playtime
    + cities_count * 200 bytes                       -- CitySave: city_id + is_accessible
    + areas_count * 150 bytes                       -- AreaSave: area_id + state + count + unlocked
    + revealed_locations_count * 64 bytes            -- String Set overhead (64-bit hash + pointer)
```

| 数据项 | 每条估算大小 | 示例 |
|--------|------------|------|
| 基本信息 | ~1KB | 存档头、时间戳、游玩时间 |
| 每个城市 | ~200 bytes | city_id + is_accessible |
| 每个地区 | ~150 bytes | area_id + state + count + unlocked |
| 每条揭示地点 (Set entry) | ~64 bytes | String引用 + Set结构开销 |

**MVP场景估算**：8城市 × 200 = 1.6KB + 32地区 × 150 = 4.8KB + 揭示地点20 × 64 = 1.28KB ≈ **7.7KB**

**存储模型说明**：
- `revealed_locations` 使用 `Set<location_id>`（唯一集合，无重复），按 `location_id` 字符串长度估算
- `clue_reveals` 由**线索系统**自己序列化和保存，不在世界地图系统的存档数据中
- 存档系统负责实际序列化，世界地图系统只提供原始数据结构

### 4.6 存档保存与恢复

**世界状态保存 (SaveWorldState)**

```lua
SaveWorldState() =
    world_save: WorldSave
    world_save.cities = {}
    for each city_id in game_world.cities:
        city_save: CitySave
        city_save.city_id = city_id
        city_save.is_accessible = city.is_accessible
        world_save.cities[city_id] = city_save

    world_save.areas = {}
    for each area_id in game_world.areas:
        area_save: AreaSave
        area_save.area_id = area_id
        area_save.exploration_state = area.exploration_state
        area_save.completion_count = area.completion_count
        area_save.is_unlocked = area.is_unlocked
        world_save.areas[area_id] = area_save

    world_save.revealed_locations = CopySet(game_world.revealed_locations)
    world_save.current_location = game_world.current_location

    return world_save
```

**世界状态恢复 (LoadWorldState)**

```lua
LoadWorldState(world_save: WorldSave) =
    -- 恢复城市状态
    for each city_id, city_save in world_save.cities:
        if city_id exists in game_world.cities:
            city = game_world.cities[city_id]
            city.is_accessible = city_save.is_accessible

    -- 恢复地区状态
    for each area_id, area_save in world_save.areas:
        if area_id exists in game_world.areas:
            area = game_world.areas[area_id]
            area.exploration_state = area_save.exploration_state
            area.completion_count = area_save.completion_count
            area.is_unlocked = area_save.is_unlocked

    -- 恢复揭示地点
    game_world.revealed_locations = CopySet(world_save.revealed_locations)

    -- 恢复当前位置
    game_world.current_location = world_save.current_location
```

**版本迁移 (OnVersionMismatch)**

```lua
OnVersionMismatch(current_version, save_version) =
    if current_version.major != save_version.major:
        -- 主版本不兼容，拒绝加载
        return LOAD_RESULT_CORRUPTED

    if current_version.minor > save_version.minor:
        -- 次版本更高，执行迁移
        migrated_world = MigrateWorldSave(save_version, current_version)
        LoadWorldState(migrated_world)
        return LOAD_RESULT_MIGRATED

    return LOAD_RESULT_COMPATIBLE
```

**数据归属澄清**：
- `clue_reveals: Dict[clue_id, location_id]` 的所有权属于**线索系统**，由线索系统自己序列化/反序列化
- `revealed_locations: Set<location_id>` 的所有权属于**世界地图系统**，由 `LocationRevealed` 事件触发时写入，由世界地图系统序列化
- 两个系统的数据各自独立存档，加载时各自恢复自己的数据

### 4.7 敌人密度星级计算

```
EnemyDensityRating(area_id) =
    expected_enemy_count = GetExpectedEnemyCount(area_id)
    normalized = Clamp(expected_enemy_count / baseline_enemy_count, 0.0, 2.0)
    stars = Round(normalized * 5)

-- 星级映射：
-- ★☆☆☆☆ = 1-3 普通敌人
-- ★★☆☆☆ = 4-6 普通敌人
-- ★★★☆☆ = 7-10 普通敌人
-- ★★★★☆ = 11-15 普通敌人
-- ★★★★★ = 16+ 普通敌人
```

| 变量 | 类型 | 默认值 | 说明 |
|------|------|-------|------|
| `baseline_enemy_count` | Integer | 5 | 标准敌人数量基准（策划配置） |
| `expected_enemy_count` | Integer | — | 地区内配置的预期敌人数量（由关卡编辑器配置） |

**配置方式**：敌人密度由策划在关卡编辑器中直接配置（`expected_enemy_count`），星级根据该数值自动计算并显示在UI上。

## Edge Cases

**边缘情况1：玩家在加载画面期间退出游戏**

- 问题：进入/离开地区时触发加载，玩家可能在加载中关闭游戏
- 处理：使用**两层存档机制**
  - 玩家进入加载画面时，先将"目标地区"写入临时标记
  - 加载完成进入地区后，清除临时标记
  - 如果加载完成前崩溃，下次启动检测到临时标记，恢复到加载前状态（最后所在地图）
- 风险等级：中

**边缘情况2：线索A揭示地点X，线索B也揭示同一地点X**

- 问题：重复揭示
- 处理：
  - 系统维护 `Set<location_id>`，重复添加无效
  - 线索收集时检查是否已揭示，是则跳过
- 风险等级：低

**边缘情况3：玩家持有揭示地点X的线索，但在揭示前死亡**

- 问题：下次加载时，线索仍然存在，但地点是否应该保持揭示？
- 处理：
  - 揭示状态保存在 `revealed_locations` 中，独立于线索状态
  - 玩家收集线索时立即揭示，不依赖"何时死亡"
- 风险等级：低

**边缘情况4：玩家回到未完成的地区，但敌人已全部清除**

- 问题：地区状态为CLEARED时，敌人是否会重生？
- 处理：按敌人类型分别处理

| 敌人类型 | CLEARED状态时行为 | 重生条件 |
|---------|------------------|---------|
| **普通敌人** | 每次重新进入地区时重新生成 | 进入新实例时 |
| **精英敌人** | 标记为"待重生"，在以下条件满足时重生 | 距上次清除≥7游戏天 |
| **关键NPC** | 永久死亡，不重生 | 不适用 |

**详细规则**：

```lua
OnEnterArea(area_id):
    for each enemy in area.enemies:
        if enemy.type == NORMAL:
            SpawnEnemy(enemy)                    -- 每次进入都重新生成
        else if enemy.type == ELITE:
            if TimeSinceLastCleared(area_id) >= 7_game_days:
                SpawnEnemy(enemy)                -- 7游戏天后在游戏内重生
            else:
                -- 精英敌人仍未到重生时间，地区保持"部分清除"状态
                Pass
        else if enemy.type == KEY_NPC:
            -- 关键NPC永久死亡，不生成
            Pass
```

**游戏天定义**：
- 1 游戏天 = 玩家在地图模式累计停留 1 分钟
- 游戏天计数器由**存档系统**持久化保存
- 游戏天与现实时间无关，只与游戏内活动相关

**防刷机制**：
- 精英敌人重生后，上一次清除时间被重置
- 玩家反复刷同一地区时，精英敌人会持续保持"已清除"状态（不会每次进入都重置）
- 只有经过足够的游戏内时间（7游戏天）后，精英敌人才会重生

**风险等级**：中 — 需要与NPC AI系统协调，确保精英敌人重生时间与游戏日历同步

**边缘情况5：玩家尝试进入已锁定的地区**

- 问题：玩家在地图上点击锁定地区
- 处理：
  - 显示锁定原因（"需要钥匙"、"需要特定线索"、"剧情锁定"）
  - 不触发加载，直接返回地图
- 风险等级：低

**边缘情况6：游戏版本更新后，地区结构发生变化**

- 问题：例如删除了某个地区，重命名了城市名
- 处理：
  - 存档时保存 `world_version`
  - 加载时检测版本不匹配，执行数据迁移
  - 迁移策略：旧版本有但新版本没有的地区 → 标记为 DEPRECATED
- 风险等级：中

**边缘情况7：非线性叙事导致玩家卡关**

- 问题：玩家可能在没有足够线索的情况下进入困难地区，死亡后无法回到简单地区
- 处理：
  - 所有地区初始开放（玩家不会被困住）
  - 线索作为"提示"而非"必需"
  - 提供基地/安全屋作为检查点
- 风险等级：低

**边缘情况9：揭示动画播放期间游戏崩溃**

- 问题：地点已通过 `LocationRevealed` 事件标记为已揭示，但揭示动画尚未播放完成时玩家退出游戏
- 处理：
  - `revealed_locations` 在事件处理时立即更新并持久化，不依赖动画是否播放完成
  - 崩溃后重新加载时，地点已标记为揭示（但动画可能需要重新播放或跳过）
  - UI系统应检测地点是否已揭示过，对已揭示地点直接显示，不重复播放动画
- 风险等级：低

**边缘情况8：玩家在地图移动中遭遇战斗，死亡后是否返回地图？**

- 问题：地图移动中触发的可选战斗，死亡后是返回地图还是当前位置？
- 处理：
  - 地图移动遭遇战（落单敌人）→ 玩家进入临时战斗场景（独立于地区），死亡后返回地图
  - 危险区域强制遭遇（帮派火拼）→ 玩家可能被卷入强制战斗区域，死亡后返回地图
  - 地区内战斗死亡 → 返回地图（不消耗进度，但撤离不成功，地区状态保持原样）
- 地图战斗死亡不视为"完成地区"，不触发 `OnAreaExtracted`
- 风险等级：低

## Dependencies

### 上游依赖（世界地图系统依赖谁）

| 系统 | 依赖类型 | 接口说明 | 事件/接口 |
|------|---------|---------|----------|
| **存档系统** | 硬依赖 | 从存档恢复世界状态；保存揭示状态和地区完成度 | `SaveWorldState()` / `LoadWorldState()` |
| **存档系统** | 硬依赖 | 游戏天计时器持久化；世界地图系统查询当前游戏天数用于精英敌人重生判定 | `GetGameDayCount()` / `IncrementGameDay()` |
| **线索系统** | 硬依赖 | 接收 `LocationRevealed(location_id)` 事件，触发地点揭示 | 事件订阅 |
| **加载画面系统** | 硬依赖 | 地区进入/离开时触发加载 | `LoadingScreenRequest()` |
| **UI系统** | 硬依赖 | 地图显示、地区信息展示 | `MapDisplayRequest()` / `AreaInfoRequest()` |
| **NPC AI系统** | 信息依赖 | 接收 `AreaCleared(area_id)` 事件更新地区状态 | 事件订阅 |

**说明**：NPC AI系统与世界地图系统的关系是**信息消费者**而非依赖者。NPC AI系统在敌人全灭时主动发送 `AreaCleared` 事件，世界地图系统被动接收以更新状态。这是发布-订阅模式，NPC AI不依赖世界地图系统。

### 下游依赖（谁依赖世界地图系统）

| 系统 | 依赖类型 | 接口说明 | 事件/接口 |
|------|---------|---------|----------|
| **存档系统** | 硬依赖 | 接收 `AreaStateUpdate`、`LocationRevealed` 事件 | 事件订阅 |
| **线索系统** | 硬依赖 | 接收 `LocationRevealed` 事件用于揭示关联线索地点；世界地图高亮显示与线索相关的地区 | 事件订阅 + 查询 |
| **NPC AI系统** | 信息依赖 | 接收 `AreaEntered(area_id)` 事件以触发敌人生成逻辑 | 事件订阅 |
| **UI系统** | 硬依赖 | 世界地图是UI的核心界面之一 | 数据推送 |

### 关键设计约束

1. **世界地图系统不拥有游戏逻辑**：它只管理"玩家在哪"和"可以去哪"，不决定"某地区有多少敌人"
2. **世界地图系统不处理叙事**：叙事内容由线索系统和事件系统管理，世界地图只是呈现和导航
3. **存档是揭示状态的唯一真相来源**：即使内存中已揭示某地点，崩溃后也应从存档恢复

### 双向依赖检查

| 系统对 | 依赖关系 | 是否双向 | 解决方案 |
|--------|---------|---------|---------|
| 世界地图 ↔ 存档 | 存档保存/恢复世界状态 | 是 | 存档系统保存 `revealed_locations` 和 `AreaSave` |
| 世界地图 ↔ 线索 | 线索触发揭示 + 地点揭示触发线索高亮 | 是 | 线索系统推送事件，世界地图被动处理；世界地图揭示地点后通知线索系统高亮相关地区 |
| 世界地图 ↔ NPC AI | 世界地图发送 `AreaEntered`；NPC AI发送 `AreaCleared` | 是（单向数据流） | `AreaEntered` 由世界地图发布，NPC AI订阅；`AreaCleared` 由NPC AI发布，世界地图订阅 |
| 世界地图 ↔ UI | UI显示地图 | 是 | UI负责渲染，但布局数据由世界地图提供 |

### Area Exploration ↔ Clue Reveal 触发关系说明

**触发链路**：
```
玩家在地区内收集线索(clue_id)
    ↓
线索系统检查 clue_reveals 映射表
    ↓
发现 clue_id 关联 location_id
    ↓
线索系统发送 LocationRevealed(location_id) 事件
    ↓
世界地图系统接收事件，更新 revealed_locations
    ↓
UI系统播放揭示动画
```

**关键约束**：
- 揭示触发**独立于**地区探索状态（玩家可以在任何地区收集线索来揭示其他地区的隐藏地点）
- `clue_reveals` 映射表由**线索系统所有**，世界地图系统不维护此映射
- 世界地图系统只负责接收 `LocationRevealed` 事件并更新 `revealed_locations` 集合
- 揭示状态变更通过存档系统持久化，崩溃后可恢复

**示例场景**：
1. 玩家在"旧码头区"收集到线索"CLUE_接头点位置"
2. 线索系统检查 `clue_reveals` 发现 `CLUE_接头点位置 → "secret_shandian"`
3. 线索系统发送 `LocationRevealed("secret_shandian")`
4. 世界地图系统将 "secret_shandian" 加入 `revealed_locations`
5. UI系统在地图上播放"新地点发现"动画

## Tuning Knobs

### 地图结构参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `WorldMapScale` | Float | 1.0 | 0.5~2.0 | 地图整体缩放，影响移动时间 |
| `CityCount` | Integer | 6 | 3~12 | 城市数量 |
| `AreasPerCity` | Integer | 3 | 2~6 | 每个城市的地区数量 |
| `InitialOpenAreas` | Set | 全部初始地区 | — | 初始开放地区ID列表 |

### 地区进入参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `LoadingScreenDuration` | Float | 2.0 | 0.5~5.0 | 加载画面最短显示时间（秒） |
| `EnableLoadingScreenTips` | Boolean | true | — | 加载画面是否显示提示 |

### 地图遭遇参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `EncounterProbability` | Float | 0.15 | 0.05~0.3 | 地图移动时遭遇概率 |
| `DangerZoneMoveDelay` | Float | 10.0 | 5.0~30.0 | 危险区域绕行延迟时间（秒） |
| `WaitCheckInterval` | Float | 0.5 | 0.1~2.0 | 检测威胁消散的间隔（秒） |
| `ThreatDissipateProbability` | Float | 0.3 | 0.1~0.5 | 每次检测时威胁消散的概率 |
| `MaxWaitTime` | Float | 30.0 | 15.0~60.0 | 最大等待时间（秒），超时后威胁强制消散 |

**EncounterProbability 校准依据**：

| 场景 | 预期玩家体验 | 对应概率范围 |
|------|-------------|-------------|
| 高度紧张（频繁战斗） | 每次移动都可能触发战斗 | 0.25 ~ 0.3 |
| **当前设定（平衡）** | **偶尔遭遇，保持压迫感但不打断探索** | **0.15** |
| 轻松探索 | 几乎不遭遇，地图主要是导航 | 0.05 ~ 0.1 |
| 安全区域 | 无遭遇 | 0.0 |

**设计考量**：
- `0.15` 意味着玩家平均每 6-7 次地图移动会遭遇 1 次战斗
- 考虑到 [game-concept.md](../game-concept.md) 定义的核心体验是"谨慎观察"而非"频繁战斗"，此值不应超过 0.3
- 危险区域（如帮派火拼区）不受此概率影响，使用固定的 `DangerZoneMoveDelay`

### 调参风险提示

- `EncounterProbability` 设置过高 → 玩家在地图上频繁战斗，失去探索感
- `EncounterProbability` 设置过低 → 地图移动过于安全，缺乏「致命的脆弱感」
- `LoadingScreenDuration` 设置过长 → 频繁加载打断节奏感
- `LoadingScreenDuration` 设置过短 → 加载未完成可能闪现画面

## Visual/Audio Requirements

### 视觉反馈

| 事件 | 视觉反馈 | 优先级 | 动画时长 |
|------|---------|--------|---------|
| 进入地区 | 加载画面 + 地区名称淡入 | 高 | 加载时间决定 |
| 揭示新地点 | 地图上弹出"新发现"标记 + 紫色闪烁动画 + 地图中心偏移 | 高 | 2.5秒（播放完毕后解除阻塞） |
| 危险区域 | 区域显示红色边框 + 火焰图标 | 中 | 持续 |
| 当前位置 | 玩家图标 + 脉冲光晕 | 高 | 持续 |
| 地区完成 | 绿色勾号标记 | 中 | 持续 |
| 锁定地区 | 锁图标 + 灰色遮罩 | 低 | 持续 |
| 揭示动画被打断 | 立即淡出，地点以普通样式显示 | — | 0.3秒 |

**揭示动画详细流程**：

```
1. 地图聚焦：镜头平移至新地点（0.5秒）
2. 标记弹出：紫色问号标记从透明→不透明放大（0.3秒）
3. 脉冲闪烁：标记进行3次脉冲（每次0.4秒，共1.2秒）
4. 发现文本：显示"新地点发现"文字（0.3秒）
5. 动画完成：解除玩家输入阻塞

总时长：2.5秒（动画期间玩家输入被暂时屏蔽）
```

**动画可中断**：玩家点击"确认"可提前结束动画，直接进入步骤5

### 地图视觉风格

- **整体色调**：暗色调，低饱和度，体现犯罪笼罩的氛围
- **城市标记**：简化的建筑轮廓，不同城市用不同颜色区分派系
- **路线**：虚线表示推荐路径，实线表示已探索路径
- **危险区域**：红色半透明覆盖层，带有动态烟雾效果

### 加载画面设计

```
┌────────────────────────────────────────┐
│                                        │
│         [地区名称]                      │
│         [城市名]                        │
│                                        │
│     ░░░░░░░░░░░░░░░░░░░░░░░░░░░░     │  ← 进度条
│                                        │
│   "正在加载：旧码头区..."               │  ← 提示文字
│                                        │
└────────────────────────────────────────┘
```

### 音频反馈

| 事件 | 音效类型 | 示例 |
|------|---------|------|
| 进入地区 | 加载音效 | 低沉的机械运作声 |
| 揭示新地点 | 发现音效 | 神秘的"叮"声 + 环境音淡入 |
| 危险区域接近 | 警告音效 | 低沉的警笛声或心跳加速 |
| 选择可进入地区 | 确认音效 | 柔和的"咔嗒"声 |
| 选择锁定地区 | 拒绝音效 | 沉闷的"咚"声 |
| 地图移动 | 脚步声/环境音 | 脚步、远处人声、车流声 |

### 环境音乐

- **地图模式**：低沉、压抑的环境音，带有隐约的不和谐音
- **危险区域**：增加紧张感的低频脉冲
- **城市间移动**：可以听到不同城市的特色环境音（如港口的汽笛声）

## UI Requirements

### 世界地图界面布局

```
┌─────────────────────────────────────────────────────────┐
│ [玩家位置]  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
│                                        │  [菜单]  [存档] │
│                                        │               │
│      ┌─────────────────────────┐       │               │
│      │                         │       │   [地图信息]   │
│      │      世界地图            │       │   当前: 锈港   │
│      │   (可缩放/拖拽)          │       │   地区: 3/12  │
│      │                         │       │               │
│      │    [城市A]    [城市B]    │       │   [快速操作]   │
│      │      │         │        │       │               │
│      │    [地区]   [地区]      │       │               │
│      │                         │       │               │
│      └─────────────────────────┘       │               │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ [线索] 线索面板 (可折叠)                              │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 地区信息面板

当玩家悬停/选择地区时显示：

```
┌─────────────────────────────┐
│ 旧码头区                    │
│ ─────────────────────────── │
│ 锈港 · 港口工业区            │
│                             │
│ 状态: 已探索                 │
│ 完成度: 1次                  │
│                             │
│ 敌人密度: ★★★☆☆             │
│ 线索: 2/5 已收集            │
│                             │
│ [进入地区]                   │
└─────────────────────────────┘
```

### 锁定地区面板

```
┌─────────────────────────────┐
│ 封锁帮派总部                │
│ ─────────────────────────── │
│ 锈港 · 港口工业区            │
│                             │
│ 状态: 🔒 锁定                │
│ 解锁条件: 需要锈港钥匙       │
│                             │
│ [查看线索提示]               │
└─────────────────────────────┘
```

### 发现新地点通知

当线索揭示新地点时：

```
┌─────────────────────────────────────┐
│  🗺️  新地点发现                      │
│                                     │
│  秘密接头点                          │
│  位于: 锈港 · 旧码头区               │
│                                     │
│  在地图上显示为可疑地点               │
│                                     │
│           [确认]                    │
└─────────────────────────────────────┘
```

### 地图导航控制

| 操作 | PC | PS5 |
|------|-----|-----|
| 移动地图 | WASD / 鼠标拖拽 | 左摇杆 |
| 缩放 | 滚轮 | 右摇杆上下 |
| 选择地区 | 左键 | A/X |
| 查看详情 | 右键 | B/O |
| 打开菜单 | ESC | Options |
| 快速存档 | Q | L1 |

### 色调规范

| 元素 | 颜色 | 用途 |
|------|------|------|
| 安全区域 | #2D5A27 | 可进入地区背景 |
| 危险区域 | #8B0000 | 高危区域覆盖 |
| 已完成地区 | #1E90FF | 蓝色标记 |
| 锁定地区 | #4A4A4A | 灰色遮罩 |
| 当前位置 | #FFD700 | 金色脉冲光晕 |
| 揭示地点 | #9932CC | 紫色闪烁 |

## Acceptance Criteria

### 功能验收

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-1 | **世界地图显示正确**：所有初始城市和地区正确显示在地图上 | 启动游戏，在地图模式验证所有初始地点可见 |
| AC-2 | **地区进入/离开**：选择地区后触发加载，进入后正确进入地区内模式 | 选择一个地区，验证加载画面，进入后验证NPC/环境正确生成 |
| AC-3 | **撤离返回地图**：完成地区后返回地图，地区状态正确更新 | 进入地区，完成撤离，验证返回地图且地区状态为 EXPLORED/COMPLETED |
| AC-4 | **死亡返回地图**：在地区内死亡后正确返回地图 | 在地区内故意死亡，验证返回地图而非重新开始 |
| AC-5 | **线索揭示隐藏地点**：收集正确线索后隐藏地点在地图上显示 | 收集揭示某地点的线索，验证地图上出现新标记 |
| AC-6 | **存档恢复**：重新加载存档后，世界探索状态正确恢复 | 完成一些地区，保存，退出，重新加载，验证状态一致 |
| AC-7 | **重复进入已探索地区**：可以重新进入任何已探索的地区 | 进入一个已完成地区，验证可以重新进入 |
| AC-8 | **地图遭遇**：在地图上移动时正确触发遭遇系统 | 在地图上多次移动，验证遭遇按概率正确触发 |

### 跨系统验收

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-9 | **存档系统集成**：地区状态变更正确保存到存档系统 | 修改地区状态，检查存档文件是否正确更新 |
| AC-10 | **线索系统集成**：线索收集正确触发地点揭示事件 | 收集应揭示地点的线索，验证揭示事件正确发出 |
| AC-11 | **UI系统集成**：地图UI正确显示所有必要信息 | 检查地图UI是否显示城市、地区、危险区域、当前位置 |

### 性能验收

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-12 | **加载时间**：地区进入/离开加载时间 < 5秒 | 测量加载画面持续时间 |
| AC-13 | **地图渲染**：地图模式下帧率稳定 60FPS | 使用帧率检测工具 |
| AC-14 | **存档大小**：完整存档大小 < 100KB | 序列化完整存档并测量大小 |

## Open Questions

| # | 问题 | 状态 | 负责人 | 说明 |
|---|------|------|--------|------|
| OQ-1 | **地图是否需要缩放功能** | 待确认 | UX设计师 | 缩放可以看更多细节，但可能影响性能 |
| OQ-2 | **基地/安全屋的位置** | 待确认 | 游戏设计师 | 基地应该是地图上的一个固定点还是有多个？ |
| OQ-3 | **不同城市的视觉风格差异** | 待确认 | 美术总监 | 每个城市应有独特的视觉语言来区分派系 |
| OQ-4 | **隐藏地区的揭示方式** | 待确认 | 叙事设计师 | 是自动揭示还是需要玩家主动调查地图？ |
| OQ-5 | **快速旅行（未来扩展）** | 预留 | — | 当前设计不支持快速旅行，未来版本可能需要 |

## Open Questions Resolution Log

| 日期 | 问题 | 最终方案 | 决定者 |
|------|------|---------|--------|
| 2026-04-08 | 非线性解锁机制 | 混合模式：线索驱动+开放世界 | 用户 |
| 2026-04-08 | 地图规模 | 小型（5-8城市） | 用户 |
| 2026-04-08 | 快速旅行 | 不需要 | 用户 |
| 2026-04-08 | 地图表现形式 | 连续俯视角地图 | 用户 |
| 2026-04-08 | 地区过渡 | 加载画面 | 用户 |
| 2026-04-08 | 初始开放 | 全开放 | 用户 |
