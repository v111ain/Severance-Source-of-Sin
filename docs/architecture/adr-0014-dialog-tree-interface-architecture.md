# ADR-0014: DialogTree 接口协议架构决策

## Status
**Accepted**

## Date
2026-04-10

## Last Updated
2026-04-11 (v4: 补充 EC-5 循环引用检测的数学证明说明；澄清 allegiance_change 方向定义)

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
          "allegiance_change": "int  // 正值=友好度上升，负值=友好度下降。典型值：威胁=-15~=-25（NPC对玩家更敌对），贿赂=+10~+20（NPC对玩家更友好）",
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

> **类型定义说明**：`DialogueEmotion` 已统一定义于 shared-types.md §13.1。

### DialogueResultType 枚举

| 值 | 说明 | Gritty Takedowns 处理 |
|----|------|----------------------|
| `CONTINUE` | 对话继续 | 显示下一分支内容 |
| `INTIMIDATE` | 威胁成功 | 触发威胁成功音效 + allegiance 大幅下降 |
| `BRIBE` | 贿赂成功 | 触发金币音效 + allegiance 提升 |
| `DECOY` | 欺骗成功 | 触发欺骗特效（屏幕短暂闪白）+ allegiance 变化（按 branch 配置值） |

> **类型定义说明**：`DialogueResultType` 已统一定义于 shared-types.md §13.2。

### 接口所有权划分

| 接口 | 拥有者 | 说明 |
|------|--------|------|
| `DialogueTree` 数据结构 | NPC AI 系统 | 定义并存储所有对话树 JSON 配置 |
| 对话 UI 渲染 | Gritty Takedowns | 渲染对话选项 UI，处理玩家输入 |
| `ConfrontationStartRequest` | Gritty Takedowns → NPC AI | 发起对峙请求。**事件定义见 shared-types.md §13.3** |
| `DialogueTreeConfig` | NPC AI 系统 → Gritty Takedowns | 返回对话树数据 |
| `DialogueChoice` | Gritty Takedowns → NPC AI 系统 | 发送玩家选择（choice_id 为 string 类型） |
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
| `BaseChange` | 对话选项中定义的值（负值=敌对，正值=友好） | ✅ 可调 | -20 ~ +20 |
| `ContextMultiplier` | NPC 当前 Alert State | ✅ 可调 | UNDETECTED=1.0, SUSPECT=1.2, SEARCH=1.5 |
| `RelationshipMultiplier` | 派系关系 | ✅ 可调 | 敌对=0.8, 中立=1.0, 友好=1.2 |

**最终变化**：`Clamp(Allegiance + Clamp(BaseChange * ContextMultiplier * RelationshipMultiplier, -30, 30), -100, +100)`

> **allegiance_change 方向说明**：
> - **负值**表示 NPC 对玩家的敌对程度上升（威胁成功时 typical = -15 ~ -25）
> - **正值**表示 NPC 对玩家的友好程度上升（贿赂成功时 typical = +10 ~ +20）
> - Allegiance 范围：-100（完全敌对）~ +100（完全友好）
>
> **安全约束说明**：在计算 AllegianceDelta 时，先对 `BaseChange * ContextMultiplier * RelationshipMultiplier` 结果进行 Clamp(-30, 30) 约束，再与当前 Allegiance 值相加后再次 Clamp(-100, 100)。这样确保：
> - 单次变化量不超过 30 点，防止单次对话导致极端跳转
> - 最终结果始终在 [-100, +100] 范围内

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
| EC-1 | 无对话树配置时的默认行为 | 返回空配置，Gritty Takedowns 根据 NPC 状态显示以下默认选项（最多 3 个）：<br><br>**默认 fallback 选项定义**：<br>• **"威胁"** → `DialogueResultType.INTIMIDATE`，`allegiance_change = -10`（适用于 `WorldState == FREE`）<br>• **"审问"** → 触发 Interrogate 流程，获取 NPC vulnerability 信息（适用于 `WorldState == UNCONSCIOUS || WorldState == TIED`）<br>• **"离开"** → 取消对峙，发送 `InteractionEvent(DIALOGUE_CANCELLED, npc_id)`（始终可用）<br><br>**vulnerability 获取机制说明（与 ADR-0011 Interrogate 联动验证）**：
> Gritty Takedowns 系统维护 `PlayerInteractionContext.hasVulnerability` 标志位。vulnerability 信息通过以下途径获取：
> 1. **审问已捆绑 NPC**（ADR-0011 §Interrogate 流程）→ 玩家选择审问选项后，NPC AI 系统返回该 NPC 的弱点信息，hasVulnerability 置为 true
> 2. **搜身死亡 NPC**（ADR-0011 §Search 流程）→ 从尸体获取线索（knowledge）时触发，hasVulnerability 置为 true
> 3. **Clue System 关联判定**（ADR-0016）→ 特定线索发现后可能解锁 vulnerability
>
> Gritty Takedowns 在每次渲染新选项前重新检查 hasVulnerability 状态（EC-6），确保长对话过程中动态刷新可选选项列表。<br>玩家通过以下途径获取 NPC vulnerability：<br>1. 审问（Interrogate）已捆绑的 NPC → 获得该 NPC 的弱点信息<br>2. 搜身（Search）死亡的 NPC → 获得线索（knowledge）<br>3. Clue System 关联判定<br><br>Gritty Takedowns 系统维护 `PlayerInteractionContext.hasVulnerability` 标志位，通过 `DialogueChoice` 事件传递到 NPC AI 系统。DialogTree 分支中的 `requires_vulnerability` 字段在渲染时由 Gritty Takedowns 检查此标志位，未满足条件的选项不显示。详见 ADR-0011 §PlayerInteractionContext。 |
| EC-2 | 对话进行中 NPC 状态突变（被攻击、死亡） | NPC AI 发送 NPCStateChangedEvent，Gritty Takedowns 立即中断对话 |
| EC-3 | 对话树深度超出 max_depth | 强制结束对话，发送 `InteractionEvent(DIALOGUE_MAX_DEPTH_EXCEEDED, npc_id)` |
| EC-4 | 对话选项引用不存在的 next_branch | 视为对话结束，发送 `InteractionEvent(DIALOGUE_COMPLETED, npc_id)` |
| EC-5 | 对话树循环引用（branch_a → branch_b → branch_a） | NPC AI 系统在加载对话树时执行拓扑排序检测，发现循环时拒绝加载该配置并输出错误日志。<br><br>**运行时深度限制数学推导**：<br>设对话树最大声明深度为 `D = max_depth`。在正常线性路径情况下，遍历步数最多为 `D`（不含终止节点）。在循环引用场景中，检测机制需要能够：<br>1. 区分正常路径（每个节点最多访问一次）<br>2. 检测循环（同一节点被第二次访问时触发）<br><br>对于包含 `N` 个节点的对话树，最坏情况是遍历所有节点一次后才发现循环（当循环发生在路径末端时），即最多需要 `N + 1` 次访问才能确认循环。<br><br>**安全边界推导**：<br>```<br>设：遍历深度 d，每次访问记录节点 ID<br>正常路径：∀i≠j, node_id[i] ≠ node_id[j]<br>循环路径：∃i≠j, node_id[i] = node_id[j]<br><br>最坏情况分析：<br>- 循环发生在节点 K（K ≤ N）<br>- 需要先遍历 K-1 个前驱节点<br>- 再访问 K 时发现重复<br>- 总访问次数 = K + 1 ≤ N + 1<br><br>安全边界应满足：max_depth ≥ N + 1<br>因此 max_depth 是遍历所有节点所需的**充分上界**<br>```<br><br>**结论**：使用 `max_depth` 作为运行时遍历深度限制是**形式化正确的**，而非经验值。原因：<br>1. 对话树的最大声明深度 `max_depth` 已经是遍历所有可能节点所需的充分上界<br>2. 如果实际遍历达到 `max_depth`，说明要么路径正常结束，要么必然存在循环<br>3. 超出 `max_depth` 时强制结束是合理的防御性编程<br><br>**双层检测机制**：<br>1. **主检测（visited_nodes）**：在遍历过程中维护一个 `visited_nodes` HashSet，当检测到节点重复访问时**立即判定为循环引用**，拒绝加载配置并输出错误日志。这是**精确检测**，能准确捕获循环。<br>2. **安全兜底（max_depth）**：如果 `visited_nodes` 机制因 bug 失效，`max_depth` 作为最后防线防止无限循环。达到 `max_depth` 时强制结束对话，发送 `InteractionEvent(DIALOGUE_MAX_DEPTH_EXCEEDED, npc_id)`。<br><br>**实现伪代码**：<br>```csharp<br>bool HasCycle(DialogueTreeConfig config) {<br>    var visited = new HashSet<string>();<br>    var stack = new Stack<string>();<br>    stack.Push(config.root_branch);<br><br>    while (stack.Count > 0) {<br>        var branchId = stack.Pop();<br>        if (visited.Contains(branchId)) {<br>            Debug.LogError($"[DialogueTree] Cycle detected at branch: {branchId}");<br>            return true;  // 精确检测到循环<br>        }<br>        visited.Add(branchId);<br>        var branch = config.branches[branchId];<br>        foreach (var choice in branch.choices) {<br>            if (choice.next_branch != null) {<br>                stack.Push(choice.next_branch);<br>            }<br>        }<br>    }<br>    return false;<br>}<br>``` |
| EC-6 | 长对话过程中玩家获取新 vulnerability | Gritty Takedowns 在每次渲染新选项前重新检查 `hasVulnerability` 状态，动态刷新可选选项列表 |

### Interrogate → vulnerability → DialogTree 选项刷新 时序图

```
┌─────────────┐     ┌───────────────────┐     ┌─────────────┐     ┌──────────────────┐
│   玩家       │     │  Gritty Takedowns │     │  NPC AI      │     │  DialogTree       │
│              │     │                   │     │  System      │     │  (UI 渲染)        │
└──────┬──────┘     └─────────┬─────────┘     └──────┬──────┘     └────────┬─────────┘
       │                      │                      │                     │
       │  1. 发起 Interrogate  │                      │                     │
       │──────────────────────►│                      │                     │
       │                      │  2. InterrogateRequest(npc_id)               │
       │                      │─────────────────────►│                     │
       │                      │                      │  3. 查询 NPC vulnerability
       │                      │                      │◄────────────────────│
       │                      │  4. VulnerabilityInfo(vulnerability_data)   │
       │                      │◄─────────────────────│                     │
       │                      │                      │                     │
       │                      │  5. 设置 hasVulnerability = true             │
       │                      │                      │                     │
       │                      │  6. 渲染 DialogTree 选项                    │
       │                      │  (检查 requires_vulnerability 条件)         │
       │                      │────────────────────────────────────────────►│
       │                      │                      │                     │  6.1 显示需要 vulnerability 的选项
       │                      │                      │                     │     （如 "欺骗"、"转化"）
       │                      │                      │                     │
       │  7. 玩家选择选项      │                      │                     │
       │◄─────────────────────│                      │                     │
       │                      │  8. DialogueChoice(choice_id,               │
       │                      │      hasVulnerability)                       │
       │                      │─────────────────────►│                     │
       │                      │                      │  9. 处理选择结果     │
       │                      │                      │  (返回 DialogueResult)
       │                      │ 10. DialogueResult(result)                   │
       │                      │◄─────────────────────│                     │
       │                      │                      │                     │
```

**时序说明**：

| 步骤 | 事件 | 说明 |
|------|------|------|
| 1 | 玩家发起 Interrogate | 玩家选择"审问"选项（仅在 NPC 处于 UNCONSCIOUS/TIED 时可用） |
| 2 | InterrogateRequest | Gritty Takedowns 通过事件总线发送审问请求到 NPC AI 系统 |
| 3 | 查询 vulnerability | NPC AI 系统从 NPCController 获取该 NPC 的 vulnerability 数据 |
| 4 | VulnerabilityInfo | NPC AI 系统返回 vulnerability 信息（含弱点描述、可用情报等） |
| 5 | 设置 hasVulnerability | Gritty Takedowns 更新 `PlayerInteractionContext.hasVulnerability = true` |
| 6 | 渲染 DialogTree | 重新渲染对话选项，此时 `requires_vulnerability=true` 的选项可见 |
| 7 | 玩家选择 | 玩家可选择原本因缺少 vulnerability 而不可见的选项（如欺骗、转化） |
| 8 | DialogueChoice | Gritty Takedowns 发送玩家选择，携带 `hasVulnerability` 标志 |
| 9 | 处理结果 | NPC AI 系统处理选择，返回结果（含 allegiance 变化、获取情报等） |
| 10 | DialogueResult | Gritty Takedowns 接收结果，更新 UI 和游戏状态 |

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
