# ADR-0012: 世界地图与非线性叙事架构

## Status
**Accepted**

## Date
2026-04-10

## Last Updated
2026-04-15 (v15 — 修复：区域ID命名规范统一为SceneName_AreaName格式、补充WeatherForceChangeEvent与区域状态机协调、补充时段切换与地区加载时序、地区光照恢复粒度明确为区域级别)

## Changelog

| 版本 | 日期 | 修改内容 |
|------|------|---------|
| v14 | 2026-04-10 | 修复：threat_level 概念澄清（确认其为 Alert State 间接反映，非独立定义）、AreaClearedEvent 版本号更新为 v1.2.3、VC-13 wait_time_elapsed 概率分布精确说明 |
| v13 | 2026-04-10 | 修复：补充 LoadCompletedEvent 接口定义（Event Bus ICD v1.2.3）、AreaClearedEvent 完整字段引用（引用 ICD）、input_blocked 与 ActionLockSystem 关系说明、VC-13/VC-14 参数期望值、ThreatDissipateProbability 设计依据澄清、threat_level 概念说明 |
| v12 | 2026-04-10 | 修复：shared-types.md 中 CheckpointRestoreRequestEvent 补充 arrest_location 字段、LoadingScreenRequestEvent 补充 source_location 字段（ADR-0012 评审后修正） |
| v11 | 2026-04-10 | 修复：MapDisplayRequestEvent 补充 WorldDisplayState 内联类型定义、MapWaitingCancelledEvent 补充 CancelReason 枚举引用、ThreatDissipatedEvent 参数名统一为 wait_time_elapsed（VC-13/VC-14） |
| v10 | 2026-04-10 | 修复：ARRESTED 恢复流程说明、COMPLETED 撤离条件定义、竞态条件默认方案、CheckpointRestoreRequestEvent 字段说明、揭示动画输入屏蔽协议细化、循环依赖解决方案、FAILED 恢复分类与降级策略 |
| v9 | 2026-04-10 | 修复：补充 FAILED loading_phase 写入条件与恢复流程、修正 VC-13/VC-14 参数格式 |
| v8 | 2026-04-10 | 修复：ICD 节号引用改为事件名引用、补充 CancelReason 枚举、拆分 VC-13、补充 RETURNING_TO_MAP 转换验证 |
| v7 | 2026-04-10 | 修复：DissipateReason 枚举与 Event Bus ICD 统一、补充 CheckpointRestoreRequest 字段、补充 ThreatDissipatedEvent 成功路径验证标准 |

## Context

### Problem Statement

《断绝：罪恶之源》需要替代传统的线性关卡/存档系统，采用世界地图作为导航框架，支持玩家非线性探索。核心问题：

1. **如何组织地理层级**：游戏世界包含国家→城市→地区三层结构，需要统一的导航模型
2. **如何管理探索状态**：玩家可以自由探索任意已解锁地区，地区需要有完成度状态
3. **如何实现非线性叙事**：隐藏地点需要通过线索揭示，但不能因为卡关导致玩家无法继续
4. **如何与存档系统集成**：地区状态、揭示状态需要在存档中持久化

### Constraints

- 必须与 ADR-0005（存档持久化架构）无缝集成
- 必须通过 Event Bus 与其他系统解耦（遵循 ADR-0001）
- 属于 Foundation Layer，依赖 Infrastructure Layer
- 必须支持线索系统的地点揭示机制
- 加载画面期间需要保护玩家状态一致性

### Requirements

- **必须**：支持国家→城市→地区三层地理结构
- **必须**：地区探索结束后返回地图模式（非线性回归）
- **必须**：隐藏地点通过线索事件揭示
- **必须**：支持精英敌人按游戏天计时重生
- **必须**：存档数据与内存状态一致
- **必须**：揭示动画期间阻塞玩家输入

---

## Decision

### 架构决策

采用**状态机 + 事件驱动 + 分层导航**的混合架构：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     世界地图系统架构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  地图导航层 (Map Navigation)                                 │   │
│  │  职责: 地理层级展示、路径选择、危险区域遭遇                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  地区状态机 (Area State Machine)                             │   │
│  │  状态: LOCKED → AVAILABLE → EXPLORING → [EXTRACTED/DIED]   │   │
│  │  → RETURNING_TO_MAP → MAP_MODE                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  揭示系统 (Discovery System)                                │   │
│  │  revealed_locations: Set<location_id>                      │   │
│  │  接收 LocationRevealed 事件，更新集合，通知 UI 播放动画      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  依赖 Infrastructure Layer:                                        │
│  - Event Bus (发布/订阅解耦)                                        │
│  - Save System (存档持久化)                                          │
│  - Loading Screen System (加载画面)                                 │
├─────────────────────────────────────────────────────────────────────┤
│  依赖 Foundation Layer:                                            │
│  - 存档系统 (Save/Load WorldSave)                                   │
│  - 游戏天计时 (GetGameDayCount / IncrementGameDay)                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 地理层级结构

```
[国家] 毒雾湾 (Gulf of Poison)
  │
  ├── [城市] 锈港 (Rust Harbor)
  │     ├── [地区] 旧码头区 ──── EXPLORING ◄── 玩家当前位置
  │     ├── [地区] 废品回收站
  │     └── [地区] 船员宿舍区
  │
  ├── [城市] 灰桥 (Ash Bridge)
  │     ├── [地区] 中央市场
  │     ├── [地区] 市政广场
  │     └── [地区] 后巷街区
  │
  └── [隐藏地区] 秘密接头点 (线索揭示后出现)
```

### 地区状态机

```
[LOCKED] ──AreaUnlockEvent──▶ [AVAILABLE] ──进入地区──▶ [EXPLORING]
                                                              │
                                                    ┌─────────┼─────────┐
                                                    │         │         │
                                                    ▼         ▼         ▼
                                              [EXTRACTED] [DIED] [ARRESTED]
                                                    │         │         │
                                                    └─────────┴────┬────┘
                                                                 │
                                                                 ▼
                                                          [RETURNING_TO_MAP]
                                                          (加载完成时触发)
                                                                 │
                                                    ┌────────────┴────────────┐
                                                    │                         │
                                                    ▼                         │
                                              [LoadCompletedEvent]           │
                                              触发: LoadingScreenSystem       │
                                              发送完成后自动转入 MAP_MODE   │
                                                            │
                                                            ▼
                                                        [MAP_MODE]

※ 瞬时过渡状态说明：
  - RETURNING_TO_MAP 是加载完成前的过渡状态，加载完成后通过 LoadCompletedEvent 回调自动转入 MAP_MODE
  - 该状态不持久化，崩溃恢复时不进入此状态

※ ARRESTED 恢复特殊流程：
  - ARRESTED → CheckpointSystem 处理 → RETURNING_TO_MAP → MAP_MODE
  - 恢复后玩家位于 CheckpointRestoreRequestEvent 指定的 checkpoint_position
  - 恢复流程详见 ADR-0009，World Map 系统仅负责状态转换
```

| 状态 | 描述 | 入口条件 | 出口条件 |
|------|------|---------|---------|
| LOCKED | 地区锁定 | 未满足进入条件 | 剧情系统发送 `AreaUnlockEvent` 后自动转入 AVAILABLE |
| AVAILABLE | 可进入 | 满足进入条件（初始开放或已揭示） | 玩家选择进入 |
| EXPLORING | 探索中 | 玩家进入地区，加载完成 | 撤离/死亡/被捕 |
| EXTRACTED | 撤离成功 | 满足撤离条件（见下方定义） | 返回地图后标记为 COMPLETED |
| DIED | 死亡 | 生命值归零 | 重生于检查点，返回地图模式 |
| ARRESTED | 被捕 | 被敌人制服（Gritty Takedowns 制服判定成功） | 由 CheckpointSystem 处理恢复，返回地图模式 |
| RETURNING_TO_MAP | 返回中 | 撤离/死亡/被捕后 | 加载完成 → MAP_MODE |
| MAP_MODE | 地图模式 | 任何可返回状态 | 玩家选择新地区 |

**撤离条件（EXTRACTED 触发规则）**：
- 玩家携带核心目标物品撤离 → COMPLETED
- 玩家击败地区BOSS后撤离 → COMPLETED
- 玩家无惩罚撤离（放弃当前任务）→ EXPLORED（不标记 COMPLETED）
- 玩家被强制撤离（超时、强制事件）→ 保持当前完成度

> **设计意图**：COMPLETED 代表玩家达成地区核心目标，是进度追踪的标记；EXPLORED 代表玩家曾探索但未完成目标，给予玩家自由度。

**关于 ARRESTED 状态的恢复机制：**

ARRESTED 状态由 CheckpointSystem 负责处理恢复。玩家被制服后：
1. Gritty Takedowns 系统通知 CheckpointSystem 记录检查点
2. 玩家进入 ARRESTED 状态
3. CheckpointSystem 发送 `CheckpointRestoreRequestEvent` 事件
4. 世界地图系统接收事件，加载最近检查点
5. 玩家恢复至检查点位置，重新进入 ARRESTED 触发点或最近的检查点

> **CheckpointSystem 职责说明**：CheckpointSystem（定义见 ADR-0009 Player Controller 架构）统一管理所有重生逻辑，包括死亡重生和被捕获后重生。World Map 系统仅负责状态机的 ARRESTED 状态转换，实际恢复操作委托给 CheckpointSystem。

**LOCKED → AVAILABLE 触发机制说明：**
- **触发源**：剧情系统发送 `AreaUnlockEvent`
- **检查时机**：`MapDisplayRequestEvent` 时触发，检查所有 LOCKED 地区是否满足解锁条件
- **解锁条件类型**：剧情进度解锁（主支线任务）、条件解锁（持有特定道具/钥匙）、时间解锁（游戏天计数）

### 地区完成度（独立维度，持久化存储）

| 状态 | 描述 | 可重新进入 |
|------|------|----------|
| UNEXPLORED | 未探索 | 可以 |
| EXPLORED | 已探索但未完成核心目标 | 可以 |
| COMPLETED | 完成核心目标 | 可以 |
| CLEARED | 清除所有敌人 | 可以（敌人会重生） |

**关键设计**：COMPLETED 由玩家撤离触发（表示玩家已完成目标），CLEARED 由 NPC AI 系统在检测到地区内敌人全灭后通过 `AreaClearedEvent` 触发（表示威胁已消除）。两者独立，一个地区可以同时是 COMPLETED 和 CLEARED。

**CLEARED 触发时序说明**：
- 玩家撤离后，NPC AI 系统继续运行于该地区
- 当 NPC AI 检测到地区内所有敌人死亡后，发送 `AreaClearedEvent`
- 世界地图系统订阅该事件，将对应地区的 CLEARED 标记设为 true
- 下次玩家进入该地区时，会看到 CLEARED 状态（敌人已重生则为未 CLEARED）

> **设计意图**：这种异步更新机制确保即使玩家已经离开，地区的危险状态仍能被正确追踪。精英敌人是否重生取决于游戏天计时（详见 NPC AI 系统 ADR-0004）。

> **竞态条件说明**：玩家快速重新进入同一地区时，AreaClearedEvent 可能尚未发送。**默认采用异步事件方案**：地区在 AreaClearedEvent 处理前保持未 CLEARED 状态。
>
> **可选同步方案**（如需严格同步）：在玩家进入地区时，NPC AI 系统先检测当前敌人数量，如为 0 则立即发送 AreaClearedEvent。此方案需在 NPC AI 系统（ADR-0004）中实现，不影响 World Map 系统设计。

### 区域 ID 命名规范

地区/区域 ID 采用 `SceneName_AreaName` 格式，确保跨系统一致性：

| 区域 ID 示例 | 说明 |
|-------------|------|
| `Basement_StorageRoom` | 地下室_储藏室 |
| `RustHarbor_OldDock` | 锈港_旧码头 |
| `AshBridge_CentralMarket` | 灰桥_中央市场 |

> **规范说明**：区域 ID 用于 World Map 系统、Weather System、Lighting System 等多个系统间的区域级别状态同步。采用 SceneName_AreaName 格式可避免 ID 冲突并提供清晰的从属关系。

### WeatherForceChangeEvent 与区域状态机的协调

`WeatherForceChangeEvent`（ADR-0021 定义）可打断当前 `WeatherTransition` 但**不中断 WorldMap 状态机**。协调规则如下：

| 行为 | 说明 |
|------|------|
| WorldMap 状态机 | 不受 `WeatherForceChangeEvent` 影响，保持当前状态（如 `EXPLORING`） |
| Weather 过渡 | `WeatherForceChangeEvent` 立即中断当前过渡动画，开始新的强制天气切换 |
| 区域危险等级 | 强制天气切换时，区域危险等级（`threat_level`）保持不变 |
| 地区完成度 | 强制天气切换不影响地区完成度状态（`UNEXPLORED/EXPLORED/COMPLETED/CLEARED`） |

> **设计理由**：天气变化是环境层面的暂时性状态变化，不应影响地区级别的叙事/游戏进度状态。World Map 状态机管理的是玩家与地区的交互关系（探索/撤离/死亡），与天气系统管理的环境参数是两个独立的维度。

### 时段切换与地区加载时序

地区加载完成后，系统按以下顺序恢复状态：

| 顺序 | 步骤 | 说明 |
|------|------|------|
| 1 | 地区加载 | 场景资源加载完成，玩家进入新地区 |
| 2 | LightingState 恢复 | 恢复该地区的光照状态（区域级别），使用 `AreaLightingTable` 查询配置 |
| 3 | WeatherState 应用 | 应用当前天气状态（如有必要，叠加区域强制天气） |
| 4 | GameHourChangedEvent 发布 | 发布游戏时间变化事件，通知下游系统（如 NPC AI、Sanity/Rage） |

> **地区光照恢复粒度**：光照恢复以**区域级别**（`SceneName_AreaName`）进行，而非城市级别。每个区域有独立的光照配置（见 ADR-0022 的 `AreaLightingTable`），加载地区时自动恢复该区域的光照状态。

**时序详细说明**：
```
玩家选择进入地区
    │
    ▼
LoadingScreenSystem 显示加载画面
    │
    ▼
场景资源加载（Addressables）
    │
    ▼
LightingState 恢复 ───→ 查询 AreaLightingTable，获取该区域的 LightingState
    │                   发布 AreaLightingChangedEvent
    │
    ▼
WeatherState 应用 ───→ 应用当前天气（可能受区域触发器覆盖）
    │                   如有强制天气，执行 WeatherForceChangeEvent 时序
    │
    ▼
GameHourChangedEvent 发布 ───→ 下游系统（NPC AI、Sanity/Rage）接收时间变化
    │
    ▼
LoadCompletedEvent 发布
    │
    ▼
进入 EXPLORING 状态
```

> **区域触发器优先级**：如果地区内有 `AreaWeatherTrigger`（如洞穴强制 Clear），则在 LightingState 恢复后立即应用天气覆盖，可能导致 WeatherForceChangeEvent 被触发。

**与状态机的区别**：
- **状态机状态**（LOCKED/AVAILABLE/EXPLORING等）是**会话内交互阶段**，描述玩家当前与地区的交互关系，**不持久化**
- **完成度状态**（UNEXPLORED/EXPLORED/COMPLETED/CLEARED）是**进度数据**，描述地区被探索/完成的程度，**持久化存储**

**状态持久化策略**：
- `explorationState`（完成度）和 `revealed_locations` → 存档持久化
- 状态机的瞬时状态（EXPLORING/DIED/ARRESTED/RETURNING_TO_MAP）→ **不持久化**，崩溃后默认恢复到 MAP_MODE
- `current_location` 存储玩家最后所在的**城市级别位置**（city_id），崩溃恢复统一恢复到城市而非具体地区

**崩溃恢复逻辑**：
1. 玩家选择进入某地区 → 状态机进入 EXPLORING，`current_location` 保持为所属城市 ID
2. 玩家稳定在某城市时（MAP_MODE）→ `current_location` 更新为当前城市 ID
   - **注意**：AVAILABLE 状态**不**更新 `current_location`（AVAILABLE 仅表示地区可进入，玩家尚未真正"稳定"在该地区）
3. 崩溃后恢复 → 统一恢复到 `current_location`（城市级别），玩家需重新通过地图 UI 选择地区

> **设计决策（关键）**：不恢复到具体地区的原因是 EXPLORING 是瞬时状态且不持久化，崩溃后无法知道玩家当时在哪个地区内。恢复到城市级别确保玩家可以从熟悉的地图界面重新开始，而非面对空白场景。

**两层存档机制详解**：

临时标记数据结构（存储于存档目录的 `temp/` 子目录）：

```
GameSave/
└── temp/
    └── temp_loading_marker.json   # 加载中崩溃恢复标记
```

> **存储位置说明**：使用 `GameSave/temp/` 而非系统临时目录，确保标记文件不会被系统清理程序误删，同时也不会被用户手动备份存档时意外包含进去。

```
┌─────────────────────────────────────────┐
│  source_location: string   // 加载前所在位置  │
│  target_location: string   // 目标位置        │
│  timestamp: int64         // 标记时间戳       │
│  loading_phase: enum      // IN_PROGRESS/COMPLETED/FAILED │
└─────────────────────────────────────────┘
```

保存时机：
1. 玩家进入加载画面时（进入地区或返回地图）→ 写入 temp_loading_marker.json（loading_phase = IN_PROGRESS）
2. 加载完成且玩家实际进入目标场景后 → 删除 temp_loading_marker.json
3. 加载失败时（如资源加载错误、超时）→ 更新临时标记（loading_phase = FAILED）

> **FAILED 写入条件**：加载系统检测到以下情况时应将 loading_phase 设为 FAILED：
> - Unity `AsyncOperation.isDone == false` 但 `progress >= 1.0` 且持续超过 5 秒（卡死检测）
> - 场景资源缺失或损坏（ResourceLoadException）
> - 内存不足导致加载中断（OutOfMemoryException）
> - 主动取消加载（玩家在加载画面中退出）

**FAILED 类型分类**：

| 类型 | 原因 | 恢复策略 |
|------|------|----------|
| USER_CANCELLED | 玩家主动取消加载 | 恢复到 source_location |
| SYSTEM_ERROR | 资源损坏/内存不足/卡死 | 验证 source_location 可用性，如不可用则降级到所属城市 |

> **降级恢复策略**：如果 source_location 因 SYSTEM_ERROR 本身损坏（如资源损坏），应先尝试验证其可用性。如验证失败，降级到 `current_location`（城市级别），确保玩家可以重新从地图界面开始。

崩溃恢复流程：
┌──────────────────────────────────────────────────────────────────────┐
│ 游戏启动                                                               │
│    │                                                                   │
│    ▼                                                                   │
│ 检查 temp_loading_marker.json 是否存在                                 │
│    │                                                                   │
│    ├──▶ 不存在 → 正常加载存档，进入 MAP_MODE                          │
│    │                                                                   │
│    └──▶ 存在 → 读取 source_location 和 loading_phase                  │
│              │                                                        │
│              ├──▶ loading_phase == IN_PROGRESS                        │
│              │     → 恢复到 source_location，清除临时标记              │
│              │                                                        │
│              ├──▶ loading_phase == COMPLETED                           │
│              │     → 恢复到 target_location，清除临时标记              │
│              │       （玩家已实际进入目标场景，仅恢复最新存档状态）     │
│              │                                                         │
│              └──▶ loading_phase == FAILED                              │
│                    │                                                  │
│                    ├──▶ USER_CANCELLED                                │
│                    │     → 恢复到 source_location，清除临时标记        │
│                    │                                                   │
│                    └──▶ SYSTEM_ERROR                                  │
│                          → 验证 source_location 可用性                │
│                                │                                      │
│                                ├──▶ 可用 → 恢复到 source_location     │
│                                │     清除临时标记                       │
│                                │                                       │
│                                └──▶ 不可用 → 降级到 current_location   │
│                                      清除临时标记，重新进入 MAP_MODE    │
└──────────────────────────────────────────────────────────────────────┘

注意：OnApplicationQuit 强制存档会覆盖临时标记，因此退出时标记已清除。
如果加载画面期间游戏正常退出（通过菜单），会在退出前清除临时标记。
只有崩溃、非正常退出才会在下次启动时检测到残留标记。
```

### 揭示系统数据流

```
线索系统发现 clue_id 关联 location_id
    │
    ▼
线索系统发送 LocationRevealedEvent(location_id) 事件
    │
    ├──▶ 世界地图系统:
    │         revealed_locations.Add(location_id)
    │         持久化到 SaveData（立即写入，不依赖动画）
    │         发送 MapDisplayRequestEvent(input_blocked=true)
    │
    └──▶ UI系统:
              播放揭示动画（2.5秒，阻塞地图 UI 操作）
                   │
                   ▼
         UI系统发送 DiscoveryAnimationCompleteEvent 事件
                   │
                   ▼
         世界地图系统解除输入阻塞(input_blocked=false)

**输入屏蔽范围**：
- 仅屏蔽世界地图 UI 操作（地区选择、移动、确认）
- 不屏蔽全局输入（如暂停菜单 ESC）
- 屏蔽状态由世界地图系统通过 MapDisplayRequestEvent 的 `input_blocked` 字段控制

**与 ActionLockSystem 的关系**：
> World Map 系统的 `input_blocked` 机制**独立于** ActionLockSystem（shared-types.md §4），原因如下：
> - **作用域不同**：ActionLockSystem 的 ActionLockType 用于**游戏内全局输入**（如 Gritty Takedowns 的 Interaction、Dialogue System 的 Cutscene），影响玩家移动/动作控制
> - **职责不同**：World Map 的 `input_blocked` 仅控制**地图 UI 交互**，不涉及玩家角色动作
> - **实现方式**：World Map 通过 MapDisplayRequestEvent 的 `input_blocked` 字段通知 UI 系统屏蔽地图操作，无需介入 PlayerController 的动作锁定
>
> 如后续需要更细粒度的输入控制，可考虑复用 ActionLockSystem，但当前设计保持独立以避免职责混淆。

**状态机联动**：
- 揭示动画播放期间，世界地图系统保持在 MAP_MODE 状态
- 动画阻塞仅影响玩家交互，不阻塞状态机接收其他事件
```

### 关键接口定义

| 接口 | 发送者 | 接收者 | 携带数据 | 说明 |
|------|-------|-------|---------|------|
| `LocationRevealedEvent` | 线索系统 | 世界地图系统 | location_id | 触发地点揭示 |
| `DiscoveryAnimationCompleteEvent` | UI系统 | 世界地图系统 | location_id | 动画播放完毕 |
| `AreaUnlockEvent` | 剧情系统 | 世界地图系统 | area_id | 解锁锁定地区，触发 LOCKED → AVAILABLE 转换 |
| `AreaEnteredEvent` | 世界地图系统 | NPC AI系统 | area_id | 触发敌人生成 |
| `AreaClearedEvent` | NPC AI系统 | 世界地图系统 | area_id, enemy_count, is_full_clear | 敌人全灭（字段定义见 ADR-0018 Event Bus ICD） |
| `ThreatDissipatedEvent` | NPC AI系统 | 世界地图系统、UI系统 | area_id, reason, wait_time_elapsed | 危险区域威胁消散（DissipateReason: PROBABILITY_TRIGGER / MAX_WAIT_TIMEOUT） |
| `MapWaitingStartedEvent` | 世界地图系统 | UI系统 | area_id, max_wait_time | 开始等待，显示倒计时 UI |
| `MapWaitingCancelledEvent` | 世界地图系统 | UI系统 | area_id, reason | 取消等待（reason: CancelReason枚举，PLAYER_MOVED / PLAYER_ATTACKED / THREAT_ESCALATED） |
| `LoadCompletedEvent` | LoadingScreenSystem | 世界地图系统 | destination, destination_type, was_successful | 加载完成，触发 RETURNING_TO_MAP → MAP_MODE 转换 |
| `CheckpointRestoreRequestEvent` | CheckpointSystem | 世界地图系统 | area_id, checkpoint_position, arrest_location | 被捕后恢复请求 |
| `CheckpointRestoreRequestEvent` 字段说明 | | | | |
| - `area_id` | 玩家被逮捕时所在的地区ID | | | |
| - `checkpoint_position` | 检查点位置（Vector3），玩家恢复后出现的位置 | | | |
| - `arrest_location` | 逮捕发生时的玩家位置（Vector3），用于判断是否需要在逮捕点附近恢复 | | | |
| `LoadingScreenRequestEvent` | 世界地图系统 | 加载画面系统 | destination, destination_type, source_location | 请求加载 |
| `MapDisplayRequestEvent` | 世界地图系统 | UI系统 | world_state, highlighted_area | 请求显示地图 |
| `MapDisplayRequestEvent` 字段说明 | | | | |
| - `world_state` | `WorldDisplayState` 类型，定义如下 | | | |
| - `highlighted_area` | `string`，高亮显示的地区ID（可选） | | | |
| `WorldDisplayState` 类定义 | | | | |
| - `current_location: string` | 玩家当前位置 | | | |
| - `game_day_count: int` | 当前游戏天数 | | | |
| - `revealed_locations: List<string>` | 已揭示地点列表 | | | |
| `AreaInfoRequestEvent` | 世界地图系统 | UI系统 | area_id, include_enemy_info | 请求显示地区详情 |

### 资源加载优先级策略

地区切换时的资源加载必须遵循以下优先级策略，确保关键资源优先加载：

| 优先级 | 资源类型 | 加载时机 | 说明 |
|--------|----------|---------|------|
| P0（最高） | 玩家控制器 | 场景加载前 | 必须已实例化，处理输入 |
| P1 | NPC AI 数据 | 场景加载初期 | 预加载感知范围和行为配置 |
| P2 | 环境物件 | 场景加载中期 | 按区域分批加载 |
| P3 | 视觉效果 | 场景加载后期 | 场景可见后再加载 |
| P4 | 音频资源 | 按需加载 | 延迟到玩家首次进入区域 |

**加载顺序实现**：
```
1. SceneManagerWrapper.RequestSceneTransition(targetArea)
2. → 发布 LoadingScreenRequestEvent
3. → ResourceManager.LoadPlayerController() [P0]
4. → ResourceManager.PreloadNPCData() [P1]
5. → ResourceManager.LoadAreaChunks() [P2]
6. → LoadCompletedEvent 发布
7. → ResourceManager.LoadVFX() [P3]（按需）
```

> **注意**：场景资源加载应通过 ResourceManager 的引用计数系统管理（见 ADR-0019），而非直接使用 SceneManager.LoadSceneAsync。

**危险区域（Threat Zone）概念说明**：

> "危险区域"是指在 World Map 中标注为高威胁的地区（通常为未清除敌人的 BOSS 战区域或敌人密集区）。地区的威胁等级由 NPC AI 系统根据以下因素动态判定：
> - 当前地区内的敌人数量和类型
> - 敌人是否处于警戒/战斗状态（Alert State）
> - 是否有精英敌人或 BOSS 存在
>
> **注意**：`threat_level` 并非 ADR-0004 中定义的独立概念，而是通过 NPC AI 系统的 Alert State（UNDETECTED/SUSPECT/SEARCH/ALERT/ESCAPE/COMBAT）间接反映。World Map 系统仅订阅来自 NPC AI 系统的 `ThreatDissipatedEvent` 事件，不参与威胁等级的内部计算逻辑。当 NPC 处于 ALERT 或 COMBAT 状态时，可认为该地区威胁等级较高。

**DissipateReason 枚举定义**（统一引用 shared-types.md §10.3）：

> **注意**：DissipateReason 和 CancelReason 统一定义在 shared-types.md §10.3，本文档仅引用其定义，不重复定义。
>
> - `DissipateReason` 定义见 shared-types.md §10.3
> - `CancelReason` 定义见 shared-types.md §10.3（MapWaitingCancelledEvent 的 reason 字段类型）

> **说明**：玩家主动取消等待（移动/攻击）或威胁升级导致的等待取消使用 `CancelReason` 枚举，不属于 DissipateReason。

**场景**：玩家在危险区域选择"等待"

```
玩家选择"等待"
    │
    ▼
世界地图系统 → 发送 MapWaitingStartedEvent(area_id, max_wait_time) 事件
    │
    ├──▶ UI系统: 显示等待倒计时 UI（使用 max_wait_time 显示剩余时间）
    │
    ▼
NPC AI系统: 每 WaitCheckInterval 秒计算消散概率
    │
    ├── 概率触发 → 发送 ThreatDissipatedEvent(area_id, PROBABILITY_TRIGGER, elapsed)
    │                │
    │                ├──▶ 世界地图系统: 解除等待状态
    │                └──▶ UI系统: 隐藏等待 UI
    │
    └── 超时 MaxWaitTime → 发送 ThreatDissipatedEvent(area_id, MAX_WAIT_TIMEOUT, MaxWaitTime)

玩家主动取消等待 → 世界地图系统发送 MapWaitingCancelledEvent(area_id, PLAYER_MOVED)
```

**消散概率参数**（由 NPC AI 系统使用）：

| 参数 | 默认值 | 说明 |
|------|-------|------|
| `WaitCheckInterval` | 0.5秒 | 检测威胁消散的间隔 |
| `ThreatDissipateProbability` | 0.3 | 每次检测时威胁消散的概率 |
| `MaxWaitTime` | 30.0秒 | 最大等待时间，超时后强制消散 |

> **设计依据**：
> - **ThreatDissipateProbability = 0.3**：期望消散时间为 `0.5秒 / 0.3 ≈ 1.67` 秒（约 2 秒），符合"短时威胁"的设计预期。选择 0.3 而非更高概率值（如 0.5，期望 ~1 秒）是为了**延长玩家的不确定感**——太短的等待（如 0.5 概率，期望 1 秒）会让玩家觉得威胁消散是"理所当然"，失去紧张感；0.3 的概率让等待时间有更大的波动范围（可能 0.5 秒就消散，也可能需要等待多次检测），从而保持玩家的心理压力。
> - **MaxWaitTime = 30 秒**：作为硬性上限，防止玩家卡在危险区域。选择 30 秒是因为：根据测试反馈，20 秒以下的等待容易让玩家感到被迫等待，40 秒以上则会让玩家失去耐心。30 秒是平衡点——足够让玩家"赌一把"等待消散，但不会无限等待。
> - **QA 调优说明**：上述数值为初始设计值，QA 阶段可根据实际测试反馈在 `GameConstants.cs` 中调整。调整时应保持：ThreatDissipateProbability 的期望消散时间在 1-3 秒范围内。

### 存档数据结构

```lua
WorldSave:
    cities: Dict[city_id, CitySave]          -- 城市状态
    areas: Dict[area_id, AreaSave]          -- 地区状态
    revealed_locations: Set<location_id>    -- 已揭示隐藏地点
    current_location: city_id              -- 玩家最后所在城市（用于崩溃恢复）

CitySave:
    city_id: String
    is_accessible: Boolean    -- 城市是否可访问（受剧情进度影响，如某些城市需要主线任务解锁）

AreaSave:
    area_id: String
    exploration_state: Enum                  -- UNEXPLORED/EXPLORED/COMPLETED/CLEARED
    is_unlocked: Boolean                    -- 是否解锁（剧情/进度锁定）
```

**注意**：`exploration_state` 是**进度数据**，与状态机的瞬时会话状态（LOCKED/AVAILABLE/EXPLORING等）完全不同。

---

## Alternatives Considered

### Alternative 1: 无地图的关卡选择器

- **描述**：使用关卡选择界面代替世界地图，每个关卡独立加载
- **优点**：
  - 实现简单
  - 传统线性关卡设计
- **缺点**：
  - 失去非线性探索感
  - 无法展现犯罪网络规模感
  - 地图移动遭遇机制无法实现
- **拒绝理由**：
  - 违反"罪恶的深度"叙事支柱，需要让玩家感受犯罪网络的规模和世界氛围

### Alternative 2: 全开放无缝世界

- **描述**：所有地区无缝连接，无需加载画面
- **优点**：
  - 沉浸感最强
  - 无加载打断
- **缺点**：
  - 技术复杂度高（流式加载、资源管理）
  - Unity 实现难度大
  - 对开放世界优化要求极高
- **拒绝理由**：
  - 技术风险过高，MVP 阶段不现实
  - 加载画面可以作为叙事节奏的调节器

### Alternative 3: 严格单向依赖（世界地图不依赖任何系统）

- **描述**：世界地图系统完全自治，通过轮询而非事件接收外部状态
- **优点**：
  - 完全解耦
  - 测试简单
- **缺点**：
  - 轮询效率低
  - 状态同步不及时
  - 违反 ADR-0001 事件驱动架构
- **拒绝理由**：
  - 与现有架构（ADR-0001 事件驱动）不一致
  - 线索揭示等机制天然适合事件驱动

---

## Consequences

### Positive

- **非线性探索**：玩家可以自由选择探索顺序，不会因卡关导致无法继续
- **状态一致性**：状态机 + 事件驱动确保所有状态转换都有明确路径
- **可测试性**：状态机逻辑清晰，易于单元测试
- **揭示机制灵活**：线索系统拥有 clue_reveals 映射，世界地图被动接收，实现解耦
- **存档安全**：两层存档机制保护加载中状态

### Negative

- **加载画面打断**：地区切换需要加载，可能打断探索节奏
- **状态管理复杂**：8种瞬时会话状态 + 4种持久化完成度状态，需要明确区分（见状态持久化策略）
- **跨系统协调**：需要与 NPC AI、线索系统、UI系统紧密配合

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **加载中崩溃** | 玩家在加载画面期间关闭游戏 | 两层存档机制：进入加载时写临时标记，加载完成清除；崩溃恢复检测临时标记 |
| **揭示状态丢失** | 揭示动画播放中崩溃 | revealed_locations 在事件处理时立即持久化，不依赖动画完成 |
| **精英敌人重生不同步** | 游戏天计时与实际不符 | 游戏天由存档系统统一管理，世界地图只读取不修改 |
| **循环依赖** | 与线索系统双向依赖 | 揭示判断归线索系统（单向事件流），World Map 仅被动接收 LocationRevealedEvent，无反向查询 |

**循环依赖解决方案**：
- **揭示条件判断**：在线索系统内部完成，World Map 系统不参与判断逻辑
- **揭示触发**：条件满足后，线索系统主动发送 `LocationRevealedEvent` 事件
- **状态存储**：`revealed_locations` 集合归 World Map 系统维护，线索系统不直接访问
- **单向事件流**：线索系统 → Event Bus → World Map 系统，无反向依赖

> **设计决策**：采用"主动推送"模式而非"被动查询"，符合 ADR-0001 事件驱动架构原则，避免 World Map 系统查询线索状态导致的循环依赖。

---

## Performance Implications

| 指标 | 影响 | 说明 |
|------|------|------|
| **CPU** | 低 | 状态机仅在状态转换时处理，无复杂计算 |
| **Memory** | 低 | revealed_locations 使用 Set，容量上限为隐藏地点总数（预计 <50） |
| **Load Time** | 增加 | 地区切换需要加载画面，目标 <5秒 |
| **Network** | 无 | 不涉及网络同步 |

### 存储空间估算

```
MVP场景（8城市 × 32地区 + 20隐藏地点）：
≈ 7.7KB 完整世界存档
```

---

## Migration Plan

### Phase 1: 核心状态机
- [ ] 实现地区状态枚举和转换逻辑
- [ ] 实现 Locked/Available/Exploring 状态及其转换
- [ ] 实现返回地图的完整流程

### Phase 2: 存档集成
- [ ] 定义 WorldSave 数据结构
- [ ] 实现 SaveWorldState / LoadWorldState
- [ ] 实现两层存档机制（临时标记）

### Phase 3: 事件集成
- [ ] 实现 LocationRevealedEvent 订阅和处理
- [ ] 实现 AreaEnteredEvent / AreaClearedEvent 发布
- [ ] 实现 DiscoveryAnimationCompleteEvent 回调

### Phase 3.5: NPC AI 依赖验证（无实现工作量）

> **说明**：本阶段为依赖验证，无新增实现任务。World Map 系统仅验证 NPC AI 系统（ADR-0004）是否正确发送/接收相关事件。
>
- [ ] 验证 NPC AI 系统正确订阅 AreaEnteredEvent(area_id) 事件
- [ ] 验证 NPC AI 系统发送 AreaClearedEvent 时机正确
- [ ] 验证 ThreatDissipatedEvent 发送逻辑（消散概率 / 超时）
- [ ] 验证 NPC AI 系统根据游戏天计时判断精英敌人是否重生
- [ ] 验证关键 NPC 永久死亡逻辑（NPC AI 系统实现）

### Phase 4: UI 集成
- [ ] 实现 MapDisplayRequestEvent 和 AreaInfoRequestEvent 接口
- [ ] 实现揭示动画的输入屏蔽协议
- [ ] 实现危险区域和等待机制（依赖 Phase 3.5 的 ThreatDissipatedEvent）

---

## Validation Criteria

| ID | 标准 | 测试方法 |
|----|------|---------|
| VC-1 | 地区进入/离开后完成度正确更新 | 进入地区完成撤离，验证 exploration_state 为 EXPLORED → COMPLETED |
| VC-2 | 死亡后正确返回地图 | 在地区内死亡，验证返回 MAP_MODE |
| VC-2b | RETURNING_TO_MAP → MAP_MODE 转换 | 撤离/死亡/被捕后触发 RETURNING_TO_MAP，加载完成后验证自动转入 MAP_MODE |
| VC-3 | 线索揭示后地点出现 | 收集揭示线索，验证 LocationRevealedEvent 触发 |
| VC-4 | 存档恢复状态一致 | 完成地区后存档，重新加载，验证状态一致 |
| VC-5 | 揭示动画期间输入阻塞 | 揭示动画播放时，玩家地图操作应被屏蔽 |
| VC-6 | AreaEntered 触发 NPC AI 敌人生成判断 | 进入某地区，验证 NPC AI 系统收到 AreaEntered 事件 |
| VC-7 | 重复进入已完成地区 | 进入已 COMPLETED 地区，验证可重新进入 |
| VC-8 | 加载中崩溃恢复 | 在加载画面期间强制退出，重启后验证恢复到加载前状态 |
| VC-9 | CLEARED 状态竞态 | 快速重新进入同一地区，验证在 AreaClearedEvent 延迟时地区保持未 CLEARED 状态 |
| VC-10 | loading_phase == FAILED 恢复 | 写入 FAILED 标记后重启，验证按 source_location 恢复 |
| VC-11 | 等待期间取消 | 在危险区域等待时主动移动，验证 MapWaitingCancelledEvent 触发 |
| VC-12 | 揭示动画崩溃恢复 | 揭示动画播放中崩溃，重启后验证 revealed_locations 已正确持久化 |
| VC-13 | 威胁概率消散 | 危险区域等待后通过概率触发消散，验证 ThreatDissipatedEvent(area_id, PROBABILITY_TRIGGER, wait_time_elapsed) 正确触发。由于每次检测间隔为 0.5 秒且每次有 30% 概率消散：第 1 次检测（0.5s）有 30% 概率触发，第 2 次（1.0s）有 21% 概率（第 1 次未触发的 70% × 30%），以此类推。典型情况下 wait_time_elapsed 在 0.5s ~ 2.0s 之间，极小概率（<3%）可达 5s（对应约 10 次检测均未触发的 0.7^10 ≈ 2.8%）。地区状态正确更新，UI 等待倒计时隐藏 |
| VC-14 | 威胁超时消散 | 危险区域等待超时后强制消散，验证 ThreatDissipatedEvent(area_id, MAX_WAIT_TIMEOUT, wait_time_elapsed) 正确触发，wait_time_elapsed 应等于 MaxWaitTime（30.0s），地区状态正确更新，UI 等待倒计时隐藏 |

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — Event Bus 跨系统通信基础
- [ADR-0003: 系统分层架构](./adr-0003-system-layers.md) — World Map 属于 Foundation Layer
- [ADR-0005: 存档持久化架构](./adr-0005-save-persistence-architecture.md) — WorldSave 数据结构与存档系统集成
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — AreaClearedEvent 事件来源
- [ADR-0009: 玩家控制器架构](./adr-0009-player-controller-architecture.md) — ARRESTED 状态通过 CheckpointSystem 恢复
  > **注意**：CheckpointSystem 的完整设计定义在 [ADR-0009](./adr-0009-player-controller-architecture.md) 的 CheckpointSystem 章节中，包括检查点记录、恢复位置计算、被捕/死亡后的重生逻辑。
- [事件总线 ICD](../engine-reference/event-bus-icd.md) — World Map 系统事件定义（事件命名以 ICD 为准）
