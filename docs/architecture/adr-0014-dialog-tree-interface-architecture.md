# ADR-0014: DialogTree 接口协议架构决策

## Status
**Accepted**

## Date
2026-04-10

## Last Updated
2026-04-10 (v2: 增加超时机制、循环引用检测、vulnerability 动态刷新、可调性修正)

## Context

### Problem Statement

DialogTree 接口协议定义了**沉重处决系统 (Gritty Takedowns)** 与**NPC AI 系统 (NPC AI System)** 之间关于对话树的接口规范。两个系统存在双向依赖：
- Gritty Takedowns 需要 NPC AI 系统提供的对话数据进行对峙交互
- NPC AI 系统需要 Gritty Takedowns 传来的玩家选择结果来驱动 NPC 行为变化

需要解决：
1. **数据所有权**：对话树数据由哪个系统拥有和管理
2. **渲染职责**：对话 UI 由哪个系统负责渲染
3. **循环依赖**：如何避免两系统间的循环依赖

### Constraints

- **设计约束**：对话树数据结构（DialogueTreeConfig）由策划编辑，存储在 NPC AI 系统
- **UI约束**：Gritty Takedowns 系统负责对话 UI 渲染和玩家输入处理
- **事件约束**：通过事件总线解耦，不使用直接函数调用
- **输入约束**：支持键盘（数字键/方向键/Enter/点击）和手柄（方向键/A/X）

### Requirements

- **必须**：定义 DialogueTreeConfig JSON 格式（dialogue_id/npc_id/branches）
- **必须**：定义 ConfrontationStartRequest / DialogueChoice / DialogueResult 事件
- **必须**：定义 DialogueEmotion 和 DialogueResultType 枚举
- **必须**：解决 Gritty Takedowns ↔ NPC AI 的循环依赖
- **必须**：处理边缘情况（NPC 状态变化、配置缺失、vulnerability 过滤）

---

## Decision

### 架构决策

采用**数据所有权分离 + 事件驱动解耦**架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DialogTree 接口协议架构                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   NPC AI System ──────► DialogueTreeConfig ──────► Gritty Takedowns     │
│         ▲                (数据所有权)                  │                │
│         │                                               │                │
│         │◄──── DialogueResult ◄──── DialogueChoice ────┘                │
│         │                                               │                │
│         │◄──── InteractionEvent ◄──── (对话结束)       │                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### DialogueTreeConfig JSON Schema

```json
{
  "dialogue_id": "npc_001_confrontation",
  "npc_id": "npc_001",
  "root_branch": "branch_start",
  "max_depth": 10,
  "branches": {
    "branch_id": {
      "speaker": "string",
      "text": "string",
      "emotion": "DialogueEmotion",
      "choices": [
        {
          "choice_id": "string",
          "text": "string",
          "next_branch": "branch_id | null",
          "allegiance_change": "int",
          "requires_vulnerability": "bool",
          "result_type": "DialogueResultType"
        }
      ]
    }
  }
}
```

### DialogueEmotion 枚举

| 值 | 说明 | UI 表现 |
|----|------|---------|
| `NEUTRAL` | 普通 | 标准对话气泡 |
| `AGITATED` | 激动 | 气泡边缘抖动 |
| `SCARED` | 恐惧 | 气泡颤抖 + 颜色变淡 |
| `ANGRY` | 愤怒 | 气泡变红 + 边缘锯齿 |

### DialogueResultType 枚举

| 值 | 说明 | Gritty Takedowns 处理 |
|----|------|----------------------|
| `CONTINUE` | 对话继续 | 显示下一分支内容 |
| `INTIMIDATE` | 威胁成功 | 触发威胁成功音效 + allegiance 大幅下降 |
| `BRIBE` | 贿赂成功 | 触发金币音效 + allegiance 提升 |
| `DECOY` | 欺骗成功 | 触发欺骗特效 + allegiance 变化 |

### 接口所有权划分

| 接口 | 拥有者 | 说明 |
|------|--------|------|
| `DialogueTree` 数据结构 | NPC AI 系统 | 定义并存储所有对话树 JSON 配置 |
| 对话 UI 渲染 | Gritty Takedowns | 渲染对话选项 UI，处理玩家输入 |
| `ConfrontationStartRequest` | Gritty Takedowns → NPC AI | 发起对峙请求 |
| `DialogueTreeConfig` | NPC AI 系统 → Gritty Takedowns | 返回对话树数据 |
| `DialogueChoice` | Gritty Takedowns → NPC AI 系统 | 发送玩家选择 |
| `DialogueResult` | NPC AI 系统 → Gritty Takedowns | 返回处理结果 |

### 事件时序

```
Gritty Takedowns ──ConfrontationStartRequest(npc_id)──► NPC AI System
                  ◄──DialogueTreeConfig (JSON)──────────────────────
                  ──渲染对话 UI（显示 text + choices）──► 玩家
                  ──DialogueChoice(choice_id, npc_id)──► NPC AI System
                  ◄──DialogueResult(result)──────────────────────────
                  ──继续渲染 或 对话结束──►
```

**超时机制**：

| 阶段 | 超时时间 | 超时处理 |
|------|---------|---------|
| `ConfrontationStartRequest` 等待响应 | 2.0s | 返回空配置，fallback 到默认选项（见 EC-1） |
| `DialogueChoice` 等待 `DialogueResult` | 1.0s | 显示默认结果（CONTINUE），对话继续 |
| NPC 状态响应（EC-2 场景） | 0.5s | 立即中断对话，避免 UI 挂起 |

> **为何需要超时**：fire-and-forget 事件在 NPC 已死亡或不可用时会永久无响应，超时机制防止 UI 层挂起。

---

## Formulas

### allegiance 变化计算

```
AllegianceDelta = BaseChange * ContextMultiplier * RelationshipMultiplier
```

| 变量 | 定义 | 可调性 | 值 |
|------|------|--------|-----|
| `BaseChange` | 对话选项中定义的值 | ✅ 可调 | -20 ~ +20 |
| `ContextMultiplier` | NPC 当前 Alert State | ✅ 可调 | UNDETECTED=1.0, SUSPECT=1.2, SEARCH=1.5 |
| `RelationshipMultiplier` | 派系关系 | ✅ 可调 | 敌对=0.8, 中立=1.0, 友好=1.2 |

最终变化：`Clamp(Allegiance + AllegianceDelta, -100, +100)`

---

## Tuning Knobs

| 类别 | 参数 | 默认值 | 安全范围 |
|------|------|--------|---------|
| 对话结构 | `MaxDialogueDepth` | 10 | 5~20 |
| 对话结构 | `AllegianceChangeMin` | -20 | -30~-10 |
| 对话结构 | `AllegianceChangeMax` | +20 | +10~+30 |
| UI 动画 | `DialogueFadeInDuration` | 0.2s | 0.1~0.5s |
| UI 动画 | `DialogueFadeOutDuration` | 0.15s | 0.1~0.3s |
| UI 动画 | `DialogueChoiceMaxDisplay` | 4 | 2~6 |
| UI 动画 | `DialogueEmotionShakeIntensity` | 0.05 | 0.02~0.1 |
| 时序 | `DialogueEndAnimationDuration` | 0.3s | 0.2~0.5s |
| 时序 | `DialogueInterruptionFadeDuration` | 0.15s | 0.1~0.3s |

---

## Alternatives Considered

### Alternative 1: Gritty Takedowns 拥有对话树数据

- **描述**：对话树配置存储在 Gritty Takedowns 系统，NPC AI 通过查询接口获取
- **优点**：Gritty Takedowns 可独立运作
- **缺点**：对话树与 NPC 绑定关系难以维护，数据同步复杂
- **拒绝理由**：对话树本质上是 NPC 的"对话能力"，应由 NPC AI 系统统一管理

### Alternative 2: 直接函数调用代替事件总线

- **描述**：Gritty Takedowns 直接调用 NPC AI 的 QueryDialogueTree() 方法
- **优点**：调用清晰，调试简单
- **缺点**：形成循环依赖，违反依赖倒置原则
- **拒绝理由**：事件驱动解耦避免循环依赖，更符合架构演进原则

### Alternative 3: 对话树存储在独立数据库

- **描述**：对话树配置存储在独立数据库服务，各系统通过网络协议访问
- **优点**：对话树可独立编辑和版本控制，支持运行时热更新
- **缺点**：引入网络延迟（即使是本地也需要序列化），增加架构复杂度
- **拒绝理由**：单机游戏不需要分布式对话树管理，本地事件传递更高效

### Alternative 4: 对话树硬编码在 UI 系统

- **描述**：对话内容直接硬编码在 Gritty Takedowns 的 UI 逻辑中
- **优点**：完全解耦，无需接口协议
- **缺点**：对话内容与 UI 逻辑耦合，难以由策划独立编辑
- **拒绝理由**：游戏需要可由策划配置的大量对话内容，硬编码不可维护

---

## Edge Cases

| # | 场景 | 处理方式 |
|---|------|---------|
| EC-1 | 无对话树配置时的默认行为 | 返回空配置，Gritty Takedowns 显示默认选项：<br>• "威胁" → `DialogueResultType.INTIMIDATE`，`allegiance_change = -10`<br>• "离开" → 取消对峙，发送 `InteractionEvent(DIALOGUE_CANCELLED, npc_id)`<br><br>**vulnerability 获取机制说明**：<br>玩家通过以下途径获取 NPC vulnerability：<br>1. 审问（Interrogate）已捆绑的 NPC → 获得该 NPC 的弱点信息<br>2. 搜身（Search）死亡的 NPC → 获得线索（knowledge）<br>3. Clue System 关联判定<br><br>Gritty Takedowns 系统维护 `PlayerInteractionContext.hasVulnerability` 标志位，通过 `DialogueChoice` 事件传递到 NPC AI 系统。DialogTree 分支中的 `requires_vulnerability` 字段在渲染时由 Gritty Takedowns 检查此标志位，未满足条件的选项不显示。详见 ADR-0011 §PlayerInteractionContext。 |
| EC-2 | 对话进行中 NPC 状态突变（被攻击、死亡） | NPC AI 发送 NPCStateChangedEvent，Gritty Takedowns 立即中断对话 |
| EC-3 | 对话树深度超出 max_depth | 强制结束对话，发送 `InteractionEvent(DIALOGUE_MAX_DEPTH_EXCEEDED, npc_id)` |
| EC-4 | 对话选项引用不存在的 next_branch | 视为对话结束，发送 `InteractionEvent(DIALOGUE_COMPLETED, npc_id)` |
| EC-5 | 对话树循环引用（branch_a → branch_b → branch_a） | NPC AI 系统在加载对话树时执行拓扑排序检测，发现循环时拒绝加载该配置并输出错误日志。运行时遍历深度限制为 `max_depth * 2`，超出则强制结束 |
| EC-6 | 长对话过程中玩家获取新 vulnerability | Gritty Takedowns 在每次渲染新选项前重新检查 `hasVulnerability` 状态，动态刷新可选选项列表 |

---

## Consequences

### Positive

- 数据所有权清晰，NPC AI 系统作为对话数据的唯一来源
- 事件驱动解耦避免循环依赖
- vulnerability 过滤机制在 Gritty Takedowns 侧实现，不污染 NPC AI 数据模型
- 支持对话树动态扩展，不影响下游系统

### Negative

- 事件传输比直接调用略有性能开销
- 调试时需要监控事件总线，增加了排查复杂度

### Risks

- **风险**：对话进行中 NPC 状态突变（被攻击、死亡）
  - **缓解**：NPC AI 发送 NPCStateChangedEvent，Gritty Takedowns 立即中断对话（见 EC-2）

---

## Performance Implications

- **CPU**：对话 UI 渲染（2-4个选项），轻量级
- **Memory**：对话树配置按需加载，不驻留内存
- **Network**：仅本地事件传递，无网络开销

---

## Migration Plan

1. NPC AI 系统先实现 DialogueTreeConfig 的 JSON Schema 定义
2. Gritty Takedowns 实现对话 UI 渲染模块
3. 两系统通过事件总线集成联调
4. 边缘情况逐步补充

---

## Validation Criteria

| ID | 验收条件 |
|----|---------|
| AC-1 | 对峙触发后对话 UI 正确显示 |
| AC-2 | requires_vulnerability 选项在未获取时不显示 |
| AC-3 | 玩家选择后 allegiance 正确变化 |
| AC-4 | 对话期间 NPC 被击杀，UI 正确关闭 |
| AC-5 | NPC 对话期间被攻击，对话立即中断 |
| AC-6 | 对话树深度达到 max_depth 时强制结束 |
| AC-7 | 引用无效分支时对话正常结束，不崩溃 |

---

## Related Decisions

- ADR-0011: 沉重处决架构（对话 UI 渲染职责）
- ADR-0004: NPC AI 行为架构（对话树数据所有权）
