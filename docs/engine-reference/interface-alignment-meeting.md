# 跨系统接口对齐会议准备材料

> **会议日期**: 2026-04-07
> **议题**: 统一事件总线规范 & 创建接口控制文档
> **参会人员**: 架构师、各系统设计者、研发代表

---

## 1. 会议目标

1. ✅ 确认统一事件命名规范
2. ✅ 消除命名冲突（`NPCDeath`/`NPCKilled`/`StateChanged` 等）
3. ✅ 补充缺失接口定义（`QueryNPCIdentity`、`KeywordCapturedEvent`）
4. ✅ 消除轮询机制，统一改为事件订阅
5. ✅ 批准接口控制文档 (ICD)

---

## 2. 问题清单

### 2.1 命名冲突问题（需立即解决）

| # | 冲突描述 | 当前状态 | 解决方案 |
|---|---------|---------|---------|
| **P0-1** | NPC 死亡事件有两个名称 | `NPCDeath`（存档系统用）vs `NPCKilled`（Clue 系统用） | 统一为 `NPCStateChangedEvent` |
| **P0-2** | 状态变化事件对外对内命名不同 | 对内 `StateChanged`，对外 `NPCStateChanged` | 统一为 `NPCStateChangedEvent` |
| **P0-3** | 警觉状态事件命名不一致 | `AlertStateChanged` vs `NPCAlertStateChanged` | 统一为 `AlertStateChangedEvent` |
| **P0-4** | 玩家受伤事件命名缺失后缀 | `PlayerDamaged` vs `PlayerDamagedEvent` | 统一为 `PlayerDamagedEvent` |

### 2.2 接口缺失问题（需立即定义）

| # | 接口描述 | 问题 | 解决方案 |
|---|---------|------|---------|
| **P0-5** | `QueryNPCIdentity` | 在 Gritty Takedowns 被引用，但 LOS 文档未定义 | 在 LOS 文档中添加完整定义 |
| **P0-6** | `KeywordCapturedEvent` | 在 Clue 系统被引用，但 LOS 文档未定义 | 在 LOS 文档中添加完整定义 |

### 2.3 轮询vs事件问题（需澄清）

| # | 问题描述 | 当前状态 | 解决方案 |
|---|---------|---------|---------|
| **P0-7** | Gritty Takedowns 使用轮询获取 AlertState | NPC AI 文档已定义 `AlertStateChanged` 事件，但 Gritty Takedowns 声称"尚未定义" | 确认事件已定义，Gritty Takedowns 改为订阅 |

---

## 3. 提案：统一事件命名规范

### 3.1 命名规则

```
[Subject] + [Did/Does] + [Context] + [Event/Request/Query/Response]
```

**示例**:
- `PlayerDamagedEvent` (正确)
- `NPCStateChangedEvent` (正确)
- `AlertStateChangedEvent` (正确)
- `KeywordCapturedEvent` (正确)

### 3.2 强制要求

1. **所有外部事件必须包含 `Event` 后缀**
2. **名词优先于动词开头**：`NPCStateChangedEvent` 而非 `ChangedNPCStateEvent`
3. **使用完整单词**：不缩写

### 3.3 废弃别名对照表

| 废弃名称 | 标准名称 |
|---------|---------|
| `NPCDeath` | `NPCStateChangedEvent` |
| `NPCKilled` | `NPCStateChangedEvent` |
| `StateChanged`（外部使用） | `NPCStateChangedEvent` |
| `AlertStateChanged`（外部引用） | `AlertStateChangedEvent` |
| `PlayerDamaged` | `PlayerDamagedEvent` |
| `NPCAlertStateChanged` | `AlertStateChangedEvent` |
| `NPCStateChanged`（对外） | `NPCStateChangedEvent` |

---

## 4. 提案：新增接口定义

### 4.1 LOS 系统新增 QueryNPCIdentity

```csharp
QueryNPCIdentity(npc_id: int) -> NPCIdentity

NPCIdentity:
    npc_id: int
    identity: NPCIdentityType  // UNKNOWN / ENEMY / ACCOMPLICE / VICTIM
    confidence: float          // 置信度 0.0 - 1.0
    source_keywords: List[string]
    last_update_time: float
```

**归属**: LOS System（LOS 维护身份标签）

### 4.2 LOS 系统新增 KeywordCapturedEvent

```csharp
KeywordCapturedEvent:
    keyword: string
    npc_id: int
    location_id: string
    category: KeywordCategory
    capture_timestamp: float
```

**归属**: LOS System（LOS 负责监听和关键词提取）

---

## 5. 提案：消除轮询，统一为事件订阅

### 5.1 当前状态

Gritty Takedowns 文档 (line 377) 声称：
> "NPC AI 系统尚未定义 `AlertStateChanged` 事件，当前实现使用轮询机制"

### 5.2 实际情况

NPC AI 文档 (line 248-268) 已明确定义：
> `AlertStateChanged(NPC_ID, old_state, new_state, trigger)`

OQ-11 (line 939) 也已解决：
> "✅ **已解决**：采用事件广播方案"

### 5.3 解决方案

Gritty Takedowns 改为订阅 `AlertStateChangedEvent`，删除所有轮询引用。

---

## 6. 会议议程（30分钟）

| 时间 | 议题 | 负责人 |
|------|------|--------|
| 0-5min | 开场：问题陈述 | 架构师 |
| 5-10min | 命名规范提案确认 | 架构师 |
| 10-15min | 新增接口定义确认（LOS） | LOS 系统代表 |
| 15-20min | 轮询→事件订阅确认 | Gritty Takedowns 代表 |
| 20-25min | ICD 文档批准 | 全体 |
| 25-30min | 后续行动分配 | 架构师 |

---

## 7. 后续行动

| # | 行动 | 负责人 | 截止日期 |
|---|------|--------|---------|
| 1 | 更新 `los-eavesdropping.md`：添加 QueryNPCIdentity 和 KeywordCapturedEvent | LOS 系统设计者 | TBD |
| 2 | 更新 `gritty-takedowns.md`：消除轮询，改为订阅 AlertStateChangedEvent | Gritty Takedowns 设计者 | TBD |
| 3 | 更新 `npc-ai-system.md`：统一事件命名 | NPC AI 设计者 | TBD |
| 4 | 更新 `clue-and-journal.md`：统一事件命名 | Clue 系统设计者 | TBD |
| 5 | 更新 `health-lethality.md`：明确 NPCStateChangedEvent 所有权 | Health 系统设计者 | TBD |
| 6 | 更新 `environment-interaction.md`：补充 EnvironmentalEvent 结构 | Environment 设计者 | TBD |
| 7 | 更新 `immersive-audio-haptics.md`：统一事件命名 | Audio 设计者 | TBD |

---

## 8. 参考文件

| 文件 | 说明 |
|------|------|
| `docs/engine-reference/event-bus-icd.md` | 接口控制文档（草案） |
| `design/gdd/health-lethality.md` | Health 系统 GDD |
| `design/gdd/npc-ai-system.md` | NPC AI 系统 GDD |
| `design/gdd/los-eavesdropping.md` | LOS 系统 GDD |
| `design/gdd/gritty-takedowns.md` | Gritty Takedowns GDD |
| `design/gdd/environment-interaction.md` | Environment 系统 GDD |
| `design/gdd/clue-and-journal.md` | Clue 系统 GDD |
| `design/gdd/immersive-audio-haptics.md` | Immersive Audio GDD |

---

## 9. 决策记录（会议中填写）

| # | 决策 | 决议 | 日期 |
|---|------|------|------|
| 1 | 事件命名规范 | ✅ 确认 / ❌ 否决 | 2026-04-07 |
| 2 | QueryNPCIdentity 归属 | ✅ LOS / ❌ 变更 | 2026-04-07 |
| 3 | KeywordCapturedEvent 定义 | ✅ 确认 / ❌ 否决 | 2026-04-07 |
| 4 | Gritty Takedowns 改为事件订阅 | ✅ 确认 / ❌ 否决 | 2026-04-07 |
| 5 | ICD 文档批准 | ✅ 批准 / ❌ 退回修改 | 2026-04-07 |
