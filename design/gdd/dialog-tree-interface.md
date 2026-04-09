# DialogTree 接口协议 (DialogTree Interface Protocol)

> **Status**: Approved
> **Author**: AI Programmer + Game Designer
> **Created**: 2026-04-08
> **Last Updated**: 2026-04-08
> **Priority**: MVP
> **Layer**: Interface / Cross-System
> **Interacts With**: Gritty Takedowns, NPC AI System
> **Depends On**: NPC AI System (DialogueTree 数据所有权)

---

## Overview

本协议定义了 **沉重处决系统 (Gritty Takedowns)** 与 **NPC AI 系统 (NPC AI System)** 之间关于对话树 (DialogueTree) 的接口规范。

两个系统存在双向依赖：Gritty Takedowns 需要 NPC AI 系统提供的对话数据进行对峙交互，NPC AI 系统需要 Gritty Takedowns 传来的玩家选择结果来驱动 NPC 行为变化。为避免循环依赖，采用事件驱动架构解耦。

**核心设计原则**：
- NPC AI 系统拥有并存储所有对话树数据
- Gritty Takedowns 系统负责 UI 渲染和玩家输入处理
- 通过明确定义的事件和查询接口进行通信

---

## Player Fantasy

**"对峙时的紧张对话，是一场心理博弈。"**

当玩家与 NPC 进入对峙状态时，NPC 可能发出试探性对话（如"你在看什么？"）。玩家需要从中获取信息、施加压力或寻找破绽。每一个对话选项都会影响 NPC 的态度——选择得当可兵不血刃，选择失误则可能激化局势。

---

## Detailed Design

### 接口所有权划分

| 接口 | 拥有者 | 说明 |
|------|--------|------|
| `DialogueTree` 数据结构 | NPC AI 系统 | 定义并存储所有对话树 JSON 配置 |
| 对话 UI 渲染 | Gritty Takedowns | 渲染对话选项 UI，处理玩家输入 |
| `ConfrontationStartRequest` | Gritty Takedowns → NPC AI | 发起对峙请求 |
| `DialogueTreeConfig` | NPC AI 系统 → Gritty Takedowns | 返回对话树数据 |
| `DialogueChoice` | Gritty Takedowns → NPC AI 系统 | 发送玩家选择 |
| `DialogueResult` | NPC AI 系统 → Gritty Takedowns | 返回处理结果 |

### 对话树数据结构 (DialogueTreeConfig)

JSON 格式（由策划编辑，NPC AI 系统反序列化）：

```json
{
  "dialogue_id": "string",
  "npc_id": "string",
  "root_branch": "branch_id",
  "branches": {
    "branch_id": {
      "speaker": "string",
      "text": "string",
      "emotion": "enum (NEUTRAL/AGITATED/SCARED/ANGRY)",
      "choices": [
        {
          "choice_id": "string",
          "text": "string",
          "next_branch": "branch_id | null",
          "allegiance_change": "int",
          "requires_vulnerability": "bool",
          "result_type": "enum (CONTINUE/INTIMIDATE/BRIBE/DECOY)"
        }
      ]
    }
  }
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `dialogue_id` | string | 对话树唯一标识符 |
| `npc_id` | string | 关联的 NPC ID |
| `root_branch` | string | 起始分支 ID |
| `branches` | object | 所有分支的字典 |
| `speaker` | string | 说话者名称/ID |
| `text` | string | 对话文本内容 |
| `emotion` | enum | NPC 说话时的情绪状态 |
| `choices` | array | 可选的对话选项列表 |
| `choice_id` | string | 选项唯一标识符 |
| `next_branch` | string \| null | 选择后跳转的分支，null 表示对话结束 |
| `allegiance_change` | int | 选择导致的 allegiance 变化值 |
| `requires_vulnerability` | bool | 是否需要玩家已获取 NPC 的 vulnerability |
| `result_type` | enum | 选项结果类型，影响后续处理 |

### 情绪枚举 (DialogueEmotion)

| 值 | 说明 | UI 表现 |
|----|------|---------|
| `NEUTRAL` | 普通 | 标准对话气泡 |
| `AGITATED` | 激动 | 气泡边缘抖动 |
| `SCARED` | 恐惧 | 气泡颤抖 + 颜色变淡 |
| `ANGRY` | 愤怒 | 气泡变红 + 边缘锯齿 |

### 结果类型枚举 (DialogueResultType)

| 值 | 说明 | Gritty Takedowns 处理 |
|----|------|----------------------|
| `CONTINUE` | 对话继续 | 显示下一分支内容 |
| `INTIMIDATE` | 威胁成功 | 触发威胁成功音效 + allegiance 大幅下降 |
| `BRIBE` | 贿赂成功 | 触发金币音效 + allegiance 提升 |
| `DECOY` | 欺骗成功 | 触发欺骗特效 + allegiance 变化 |

---

## 接口调用流程

### 时序图

```
┌─────────────────────┐                              ┌─────────────────────┐
│  Gritty Takedowns   │                              │    NPC AI System    │
└──────────┬──────────┘                              └──────────┬──────────┘
           │                                                    │
           │  1. ConfrontationStartRequest(npc_id)            │
           │──────────────────────────────────────────────────►│
           │                                                    │
           │                                                    │  查询 DialogueTreeConfig
           │                                                    │  根据 npc_id 加载对应对话树
           │                                                    │
           │  2. DialogueTreeConfig (JSON)                     │
           │◄──────────────────────────────────────────────────│
           │                                                    │
           │  3. 渲染对话 UI（显示 text + choices）              │
           │                                                    │
           │  4. 玩家选择选项                                    │
           │                                                    │
           │  5. DialogueChoice(choice_id, npc_id)              │
           │──────────────────────────────────────────────────►│
           │                                                    │
           │                                                    │  查找 choice_id 对应选项
           │                                                    │  计算 allegiance_change
           │                                                    │  查找 next_branch
           │                                                    │  发送 KnowledgeGained（如有）
           │                                                    │
           │  6. DialogueResult(result)                        │
           │◄──────────────────────────────────────────────────│
           │                                                    │
           │  7a. result.next_branch != null                   │
           │      → 渲染下一分支 UI（回到步骤 3）                  │
           │                                                    │
           │  7b. result.next_branch == null                   │
           │      → 对话结束，发送 InteractionEvent              │
           │                                                    │
           └────────────────────────────────────────────────────┘
```

### 详细步骤说明

**步骤 1：发起对峙请求**

```
ConfrontationStartRequest(npc_id: string) → Event
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `npc_id` | string | 目标 NPC 的唯一标识符（与 NPC AI 系统中的 EntityID 一致） |

**触发条件**：
- 玩家与 NPC 处于对峙距离（≤ 5 米）
- NPC 处于 FREE 状态
- NPC Alert State < ALERT（UNDETECTED/SUSPECT/SEARCH 三态之一）
- 注：Alert State 枚举定义详见 `npc-ai-system.md`

**步骤 2：返回对话树配置**

```
DialogueTreeConfig → Event
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `dialogue_id` | string | 对话树 ID |
| `npc_id` | string | NPC ID |
| `root_branch` | string | 起始分支 ID |
| `branches` | object | 分支字典 |

**数据来源**：NPC AI 系统根据 `npc_id` 查找对应的对话树 JSON 配置。

**步骤 3：渲染对话 UI**

Gritty Takedowns 系统负责：
- 显示 NPC 台词气泡
- 显示可用对话选项（2-4 个）
- 选项根据 `requires_vulnerability` 过滤（未获取 vulnerability 时不显示）

**步骤 4：玩家选择**

玩家通过键盘/手柄选择选项。

**输入路由规则**：
| 输入设备 | 选择方式 | 路由规则 |
|---------|---------|---------|
| 键盘 | 数字键 1-4 / 方向键 + Enter | 数字键直接索引（1=第1个选项），方向键移动焦点，Enter 确认 |
| 手柄 | 方向键 + A/X 确认 | 方向键移动焦点，A/X 确认 |
| 鼠标 | 点击选项 | 直接点击选项，无须先选中 |

**输入解析逻辑**：
1. 玩家按数字键 N 时，直接发送 `DialogueChoice(choice[N-1], npc_id)`
2. 玩家按方向键时，更新 UI 焦点位置（不发事件）
3. 玩家按确认键时，发送当前焦点选项的 `DialogueChoice`
4. 玩家点击选项时，直接发送该选项的 `DialogueChoice`

**选项过滤规则**：
- `requires_vulnerability = true` 的选项在玩家未获取对应 vulnerability 时不显示
- 所有可用选项按 `choice_id` 字母顺序排列后，依次分配 1-N 索引

**步骤 5：发送玩家选择**

```
DialogueChoice(choice_id: string, npc_id: string) → Event
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `choice_id` | string | 玩家选择的选项 ID |
| `npc_id` | string | 目标 NPC ID（与 ConfrontationStartRequest 中的 npc_id 一致） |

**步骤 6：返回处理结果**

```
DialogueResult {
    result_type: DialogueResultType,
    allegiance_change: int,
    knowledge_gained: List[string] | null,
    next_branch: string | null,
    npc_emotion: DialogueEmotion
} → Event
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `result_type` | enum | 结果类型 |
| `allegiance_change` | int | allegiance 变化值 |
| `knowledge_gained` | array \| null | 审问获取的线索列表（如有） |
| `next_branch` | string \| null | 下一分支 ID，null 表示对话结束 |
| `npc_emotion` | enum | NPC 新的情绪状态 |

**步骤 7：结束或继续**

- `next_branch != null`：返回步骤 3，渲染下一分支
- `next_branch == null`：对话结束，发送 `InteractionEvent(DIALOGUE_COMPLETE, npc_id)` 到事件总线

---

## Edge Cases

### 边缘情况 1：NPC 在对话期间状态变化

**问题**：对话进行中，NPC 被第三方攻击或发现其他威胁。

**处理**：
- NPC AI 系统发送 `NPCStateChangedEvent` 或 `AlertStateChangedEvent`
- Gritty Takedowns 收到事件后，**立即中断对话**
- 显示"NPC 被打断！"提示
- 对话进度**不保存**，下次对峙从头开始

### 边缘情况 2：对话树配置缺失

**问题**：请求的 `npc_id` 没有对应的对话树配置。

**处理**：
- NPC AI 系统返回空的 `DialogueTreeConfig`（`branches` 为空）
- Gritty Takedowns 检测到空配置，显示默认选项：
  - "威胁" → 触发威胁交互
  - "离开" → 取消对峙

### 边缘情况 3：玩家选择需要 vulnerability 但未获取

**问题**：选项的 `requires_vulnerability = true`，但玩家尚未获取 NPC 的 vulnerability。

**处理**：
- 该选项在 UI 中**不显示**
- Gritty Takedowns 在渲染前过滤选项列表

### 边缘情况 4：对话选项全部被过滤

**问题**：由于 `requires_vulnerability` 限制，没有任何选项可显示。

**处理**：
- Gritty Takedowns 显示默认选项：
  - "离开" → 取消对峙
- NPC 播放一条额外台词："看什么看？"

### 边缘情况 5：NPC 在对话期间被击杀

**问题**：玩家选择期间，第三方将 NPC 击杀。

**处理**：
- Gritty Takedowns 收到 `NPCStateChangedEvent(DEAD)`
- 立即关闭对话 UI
- 显示"NPC 已死亡"提示（1.5 秒）
- 玩家自动进入 Idle 状态

### 边缘情况 6：allegiance 变化触发 NPC 行为立即变化

**问题**：对话结束后，allegiance 变化导致 NPC 立即逃跑/攻击。

**处理**：
- NPC AI 系统在发送 `DialogueResult` 后，正常处理 allegiance 变化
- Gritty Takedowns 侧：
  - 对话结束动画（0.3 秒）
  - 动画完成后，NPC 根据新 allegiance 执行对应行为

---

## Dependencies

### 上游依赖

| 系统 | 接口类型 | 依赖性质 | 说明 |
|------|---------|---------|------|
| NPC AI 系统 | 事件接收 + 数据查询 | 硬依赖 | 接收 `ConfrontationStartRequest`，返回 `DialogueTreeConfig`，处理 `DialogueChoice` |

### 下游依赖

| 系统 | 接口类型 | 依赖性质 | 说明 |
|------|---------|---------|------|
| Gritty Takedowns | 事件发送 | 硬依赖 | 发送 `DialogueChoice`，接收 `DialogueResult`，渲染对话 UI |

### 依赖关系图

```
┌─────────────────────────────────────────────────────────────┐
│                    Cross-System Interface                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   NPC AI System ──────► DialogueTreeConfig ──────► Gritty   │
│         ▲                (数据所有权)              │  Takedowns  │
│         │                                           │            │
│         │◄──── DialogueResult ◄──── DialogueChoice │            │
│         │                                           │            │
│         │◄──── InteractionEvent ◄──── (对话结束)   │            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 循环依赖解决

**问题**：Gritty Takedowns 和 NPC AI 系统互相依赖。

**解决方案**：通过事件总线解耦。

```
修订前（循环依赖）：
Gritty Takedowns ──(硬依赖)──► NPC AI 系统
     ▲                                  │
     └────────(硬依赖)────────────────────┘

修订后（解耦）：
Gritty Takedowns ──(事件总线)──► NPC AI 系统
     │                                  │
     └────── DialogueChoice ◄───────────┘
```

---

## Formulas

### allegiance 变化计算

对话选项导致的 allegiance 变化由 NPC AI 系统计算：

```
AllegianceDelta = BaseChange * ContextMultiplier * RelationshipMultiplier
```

| 变量 | 定义 | 值 |
|------|------|-----|
| `BaseChange` | 对话选项中定义的值 | -20 ~ +20 |
| `ContextMultiplier` | NPC 当前 Alert State | UNDETECTED=1.0, SUSPECT=1.2, SEARCH=1.5 |
| `RelationshipMultiplier` | 派系关系 | 敌对=0.8, 中立=1.0, 友好=1.2 |

最终变化：`Clamp(Allegiance + AllegianceDelta, -100, +100)`

---

## Tuning Knobs

### 对话 UI 参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `DialogueFadeInDuration` | float | 0.2s | 0.1~0.5s | 对话气泡淡入时长 |
| `DialogueFadeOutDuration` | float | 0.15s | 0.1~0.3s | 对话气泡淡出时长 |
| `DialogueChoiceMaxDisplay` | int | 4 | 2~6 | 最多显示选项数 |
| `DialogueEmotionShakeIntensity` | float | 0.05 | 0.02~0.1 | 激动情绪气泡抖动幅度 |

### 时序参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `DialogueEndAnimationDuration` | float | 0.3s | 0.2~0.5s | 对话结束动画时长 |
| `DialogueInterruptionFadeDuration` | float | 0.15s | 0.1~0.3s | 对话中断淡出时长 |

---

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 玩家触发对峙后，对话 UI 正确显示 NPC 台词和选项 | 进入对峙距离，观察 UI |
| AC-2 | 需要 vulnerability 的选项在未获取时不显示 | 检查 UI 选项数量 |
| AC-3 | 玩家选择后，allegiance 正确变化 | 选择前后查询 `QueryAllegiance` |
| AC-4 | 对话结束后，NPC 根据新 allegiance 执行对应行为 | 变化 allegiance 后观察 NPC |
| AC-5 | 对话期间 NPC 被击杀，对话 UI 正确关闭 | 在对话中击杀 NPC，验证 UI |

### 集成验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-6 | Gritty Takedowns 正确发送 `ConfrontationStartRequest` | 监控事件总线 |
| AC-7 | NPC AI 系统正确返回 `DialogueTreeConfig` | 监控事件总线 |
| AC-8 | Gritty Takedowns 正确发送 `DialogueChoice` | 选择选项后监控事件 |
| AC-9 | NPC AI 系统正确返回 `DialogueResult` | 选择后监控事件 |
| AC-10 | 对话结束正确发送 `InteractionEvent` | 对话结束时监控事件 |

### 边缘情况验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-11 | NPC 对话期间被攻击，对话立即中断 | 对话中攻击 NPC，验证中断 |
| AC-12 | 无对话树配置时显示默认选项 | 对无配置 NPC 触发对峙 |
| AC-13 | 所有选项被过滤时显示"离开"选项 | 获取 vulnerability 后验证 |

---

## Open Questions

| # | 问题 | 负责人 | 目标日期 |
|---|------|--------|---------|
| OQ-1 | 对话树是否需要支持条件分支（根据玩家属性显示不同选项） | 游戏设计师 | Vertical Slice 设计时 |
| OQ-2 | NPC 主动发起的对话（非对峙触发）如何处理 | AI 设计师 | Alpha 阶段 |

---

## Change Log

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-08 | 0.1 | 初稿创建，定义 DialogTree 接口协议 | AI Programmer Agent |
