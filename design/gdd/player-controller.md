# Player Controller (玩家控制器)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-03-29
> **Implements Pillar**: 致命的脆弱感 (Lethal Fragility)

## Overview

玩家控制器 (Player Controller) 是《断绝：罪恶之源》中最基础的输入响应与状态管理系统。它负责将玩家的操作（移动、下蹲、互动、攻击）转化为游戏世界中主角“父亲”的实际位移与状态切换。有别于快节奏动作游戏，本系统严格遵循“致命的脆弱感”这一游戏支柱——移动是沉重且谨慎的，不支持无敌帧翻滚等超人类动作，而是强调借助掩体和视野进行隐蔽。它是连接玩家意图与所有其他交互系统（如环境破坏、NPC 监听、处决）的核心枢纽。

## Player Fantasy

玩家在操作主角时的核心情感体验应当是“克制的恐惧”与“爆发的野兽”之间的极端切换。

你不是身手敏捷、能飞檐走壁的超级特工，而是一个被夺走一切、体力有限的凡人父亲。在操作上：
*   **如履薄冰的沉重感**：常规移动时，角色具有一定的起步惯性和重量感。玩家必须清晰地意识到，每一次暴露在开阔地带都伴随着极高的死亡风险。潜行时，操作手感需要传递出屏息凝神的压抑感。
*   **不加掩饰的暴力感**：一旦玩家抓住时机发起攻击，控制器的反馈（位移捕捉、顿帧）应当极其果决、粗暴。

**参考对标**：《最后生还者》中乔尔那富有重量感的移动与受击硬直，结合《迈阿密热线》中一击必杀带来的极度紧迫感。玩家不是在“游玩一个动作英雄”，而是在“维系一个濒临崩溃的复仇者的生存”。

## Detailed Design

### Core Rules

*   **输入与朝向**：采用绝对方向输入（WASD/左摇杆）。角色永远面向其移动的方向；当停止移动时，角色保持最后的面朝方向。
*   **三种移动模式与噪音反馈**：
    *   **行走 (Walk - 默认)**：中等速度。产生小范围的脚步噪音。
    *   **潜行/下蹲 (Crouch - 切换键)**：慢速。完全静音（不产生脚步噪音）。降低碰撞体积高度，允许利用低矮掩体躲避敌人视线。
    *   **冲刺 (Sprint - 按住键)**：快速。产生大范围的脚步噪音。消耗体力槽。冲刺过程中无法发起攻击。
*   **体力限制 (Stamina)**：仅在冲刺时消耗。停止冲刺（行走或潜行）时自动恢复。体力耗尽时无法冲刺，强制降为行走状态。
*   **动作锁定 (Action Lock)**：当玩家按下“攻击/处决”或“深度互动”时，角色移动输入会被短时间锁定，直到动作动画完成。

### States and Transitions

| 状态 | 触发条件 | 可转移至 | 状态约束 / 效果 |
| --- | --- | --- | --- |
| **Idle (待机)** | 无方向输入 | Walk, Crouch, Sprint, Attack | 保持最后的面朝方向 |
| **Walk (行走)** | 仅方向输入 | Idle, Crouch, Sprint, Attack | 产生低度噪音 |
| **Sprint (冲刺)** | 方向输入 + 按住冲刺键 | Idle, Walk, Crouch (急停蹲下) | 产生高度噪音，消耗体力，禁止攻击 |
| **StaminaExhausted (体力耗尽修饰符)** | Sprint 状态下 `CurrentStamina` 降至 0 | Walk（当 `CurrentStamina >= MaxStamina * StaminaRegenPenaltyThreshold` 时，`IsStaminaExhausted = false`，Sprint 重新可用） | **修饰符（非独立状态）**；附加在 Walk 状态上的修饰符；维持 Walk 速度与噪音；所有 Sprint 输入被屏蔽；`IsStaminaExhausted = true`；体力以 `StaminaRegenRate` 正常恢复 |
| **Crouch (潜行)** | 潜行键 (Toggle/切换) | 站立状态下的任何移动；再次按潜行键主动解除；头顶碰撞检测通过时被动解除 | 移动静音，缩小可视体积 |
| **Attack (攻击)** | 玩家按下攻击键且当前状态允许攻击（Idle/Walk/Crouch） | 动画播放完毕 → Idle；被打断（受击/死亡）→ 对应状态 | 播放攻击动画并造成伤害；动画播放期间 StateMultiplier=0（速度为零，不能移动）；攻击键长按可在动画结束后衔接下一次攻击 |
| **Action (动作中)** | 按下攻击或互动键 | 动画结束后返回 Idle/Walk | 锁定玩家所有位移输入（速度为0），动画完成后自动释放 |

### Interactions with Other Systems

*   **提供给 [视野与监听系统]**：控制器每帧提供玩家当前的 `WorldPosition` (世界坐标) 和 `FacingDirection` (面朝向量)。
*   **提供给 [NPC AI系统]**：控制器基于当前移动状态，向世界广播 `NoiseEvent` (噪音事件)。

**NoiseEvent 数据结构**：
```
NoiseEvent:
    position: Vector3          # 噪音发生的世界坐标
    radius: Float             # 噪音广播半径（潜行=0m, 行走=3m, 冲刺=8m）
    noise_type: Enum          # WALK / CROUCH / SPRINT / INTERACTION / NONE
    duration: Float            # 噪音持续时间（秒），默认 0.5s
    can_interrupt: Bool        # 是否可被打断（默认 true）
    source_entity_id: Int     # 产生噪音的实体ID（玩家或其他）
```

**射线检测接口规范**：

| 职责 | 拥有者 | 说明 |
|------|--------|------|
| 射线检测执行 | 玩家控制器 | 每帧执行物理射线投射（长度 2.0m，锥形 60°），性能优化：仅在有输入时激活 |
| 检测结果存储 | 玩家控制器 | 将 RaycastResult 写入共享内存位置 |
| 可交互性判定 | 环境交互系统 | 读取 RaycastResult，判定物件是否在交互范围内（距离+朝向） |

**射线检测返回值格式**：
```
RaycastResult:
    hit: Boolean              # 是否命中物件
    object_id: Int            # 命中的物件ID（如果 hit=true）
    object_type: Enum         # 物件类型（WEAPON/EXPLOSIVE/DESTRUCTIBLE/INTEL/OBSTACLE/MECHANISM/THROWABLE）
    distance: Float           # 命中距离
    normal: Vector3           # 命中点法线
```

**设计意图**：将"检测"(Raycast)和"判定"(InteractionCheck)解耦。玩家控制器专注于输入响应和物理检测，环境交互系统专注于交互逻辑。环境交互系统拥有 `RaycastResult` 的解释权，玩家控制器仅负责提供原始数据。

*   **向 [环境交互系统] 发起请求**：当按下互动键时，控制器向正前方发出射线/盒体检测，触发对应物件的交互逻辑。
*   **移交控制权给 [沉重处决系统]**：当触发处决时，玩家控制器将自身的 `IsLocked` 状态设为 True，由处决系统接管角色位置和动画，完成后再归还控制权。

#### `IsLocked` 接口规范（所有权：玩家控制器）

**接口所有权澄清**：
- `IsLocked` 及其相关数据结构由**玩家控制器**拥有
- 外部系统（如沉重处决系统）通过标准接口请求锁定，不得直接修改 `IsLocked` 状态
- 锁定持有者负责在动作完成后主动释放锁定

| 属性 | 类型 | 说明 |
|------|------|------|
| `IsLocked` | bool | **动作锁定标志**。当为 `true` 时，玩家控制器的位移输入被忽略，角色位置和动画由外部系统（如沉重处决系统）接管。 |
| `LockOwner` | System | 当前锁定持有者的系统标识（如 `"GrittyTakedowns"`）。用于判断是否有权限解锁。 |
| `LockTimestamp` | float | 锁定开始的时间戳（游戏内时间），用于计算锁定持续时间。 |

**锁定/解锁接口**：

| 接口 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `AcquireLock(requester: System, duration: float)` | 请求系统ID，预期锁定时长 | `bool` | 尝试获取锁定。成功返回 `true`，若已被其他系统锁定则返回 `false`。 |
| `ReleaseLock(requester: System)` | 请求系统ID | `bool` | 尝试释放锁定。只有锁定持有者可以释放，成功返回 `true`。 |
| `ForceReleaseLock()` | 无 | `void` | 强制释放锁定（仅用于紧急情况，如玩家死亡）。 |

**锁定行为规则**：
- 锁定期间，Player Controller 的 `ProcessInput()` 函数跳过位移输入处理
- 锁定持有者负责在 `duration` 到期后主动调用 `ReleaseLock()`，或调用 `ForceReleaseLock()` 紧急解锁
- 若锁定持有者未在 `duration` 内释放，Player Controller 会在超时后自动解锁（防止死锁）
- **超时阈值**：`LockTimeout = 5.0 秒`。当锁定持续时间超过此值时，Player Controller 自动强制释放锁定，并记录错误日志。

**超时机制设计说明**：
- 默认超时时间 5 秒足以覆盖绝大多数正常动作动画时长（处决动画通常 1-3 秒）
- 超时后自动解锁防止玩家因锁定系统故障而永久卡死
- 超时释放视为异常终止，锁定持有者应在下一次动作开始前检查并重置状态

## Formulas

**1. 移动速度计算**
`CurrentVelocity = InputDirection * BaseSpeed * StateMultiplier * (1.0 + RageSpeedBonus)`

*   **InputDirection**: 玩家输入的单位方向向量（长度为 0 到 1 之间）。
*   **BaseSpeed**: 基础移动速度（默认 5.0 m/s）。
*   **StateMultiplier (状态乘数)**:
    *   Walk (默认) = 1.0
    *   Sprint (冲刺) = 1.6
    *   Crouch (潜行) = 0.5
    *   Attack (攻击) = 0（攻击动画播放期间，位移速度为零）

**Action Lock 与 IsLocked 的独立机制澄清**：

| 机制 | 来源 | 作用 | 独立性说明 |
|------|------|------|-----------|
| **Action Lock** | Player Controller 内部状态机 | 攻击/互动动画播放期间，StateMultiplier设为0，速度为零，动画完成后自动释放 | 独立于IsLocked，是状态机内部的速度控制机制 |
| **IsLocked** | Player Controller 的系统级接口 | 外部系统（如GrittyTakedowns）通过 `AcquireLock()` 请求接管，位移输入被完全忽略 | 独立于Action Lock，是供外部系统使用的锁定接口 |

**叠加关系**：Action Lock 和 IsLocked 可以叠加。当玩家在攻击动画期间（Action Lock激活），外部系统同时通过 `AcquireLock()` 接管时，两种锁定同时生效，位移输入被完全忽略。这种叠加用于处决等场景——攻击动画的位移锁定（StateMultiplier=0）确保动画播放期间的视觉一致性，而IsLocked确保外部系统在必要时完全接管控制权。
*   **RageSpeedBonus (愤怒速度加成)**:
    *   来源：理智/愤怒系统（`sanity-rage-meter.md`）的 `MovementSpeedMultiplier`
    *   范围：0.0 ~ 0.1（+10% 上限）
    *   叠加规则：当玩家同时处于 Crouch 状态和 FRENZIED 状态时，`FinalSpeed = BaseSpeed × 0.5 × (1.0 + RageSpeedBonus)`
    *   数据流向：Sanity/Rage 系统 → 玩家控制器的速度计算模块（每帧更新）

**2. 体力消耗与恢复系统**
`CurrentStamina = Clamp(PreviousStamina + DeltaStamina, 0, MaxStamina)`
*   **消耗 (当处于 Sprint 状态)**:
    `DeltaStamina = -(StaminaDrainRate * DeltaTime)`
*   **恢复 (当处于 Walk / Crouch / Idle 状态)**:
    需满足条件 `TimeSinceLastSprint > StaminaRegenDelay`（即停止冲刺后有一定的喘息延迟才开始恢复）：
    `DeltaStamina = (StaminaRegenRate * DeltaTime)`

## Edge Cases

*   **对角线移动速度叠加**：
    *   *问题*：如果同时按下 W 和 D，未归一化的向量会导致速度变为 1.414 倍。
    *   *处理*：在计算 `CurrentVelocity` 之前，必须对输入向量 `InputDirection` 进行归一化（Normalize）或使用圆形死区限制，确保最大输入幅度为 1.0。
*   **体力耗尽时的强制惩罚 (StaminaExhausted 状态正式定义)**：
    *   *问题*：玩家如果反复点按冲刺键，可能会在体力为 0 附近反复横跳，导致动画和状态抽搐。
    *   *处理*：如果体力降至 0，强制退出 Sprint，进入 Walk 状态，同时触发 `StaminaExhausted` 惩罚修饰符。详细行为定义如下：
        *   **本质**：`StaminaExhausted` 是附加在 Walk 状态上的修饰符，对应控制器内部标志位 `IsStaminaExhausted: bool`，不是独立的平行状态。Walk 状态的所有规则（速度、噪音等）继续生效。
        *   **进入条件**：`CurrentStamina == 0 AND 当前状态 == Sprint`。控制器检测到此条件时，立即将 `IsStaminaExhausted = true` 并强制切换到 Walk 状态。
        *   **持续效果**：控制器在处理每帧输入时，若 `IsStaminaExhausted == true`，则所有 Sprint 输入请求（按住冲刺键）一律被屏蔽返回，不触发状态切换。角色以正常 Walk 速度和噪音级别移动。体力仍以 `StaminaRegenRate` 正常恢复（`StaminaRegenDelay` 延迟同样生效）。
        *   **退出条件**：每帧检测 `CurrentStamina >= MaxStamina * StaminaRegenPenaltyThreshold`（即 ≥ 30%，对应 `StaminaRegenPenaltyThreshold = 0.30`）。条件首次满足时，`IsStaminaExhausted = false`，Sprint 重新可用，不需要玩家任何额外操作。
        *   **与 Crouch 的交互**：`StaminaExhausted` 修饰符不阻止玩家进入 Crouch 状态。玩家在 StaminaExhausted 期间可以正常下蹲；从 Crouch 回到 Walk/Sprint 时，Sprint 的屏蔽规则仍然生效直到体力恢复阈值。
        *   **设计意图**：防止"体力抖动"的同时传递"上气不接下气"的沉重感，强化"致命的脆弱感"支柱——玩家过度冲刺会进入一段脆弱的受限窗口期。
*   **处决动画期间受击**：
    *   *问题*：由于不是无敌特工，如果在处决敌人（处于 Action Lock）时被其他敌人开枪击中怎么办？
    *   *处理*：玩家在处决期间**没有无敌帧 (No i-frames)**。如果受到致命伤害，处决动作会立刻中断并转入死亡状态，被处决的敌人存活（或视动画进度判定为死亡）。这进一步强化了“致命的脆弱感”支柱，要求玩家必须确保环境安全才能执行处决。
*   **狭窄空间下蹲卡位**：
    *   *问题*：在低矮掩体下从潜行（Crouch）切换回站立（Walk/Sprint）时，头顶有碰撞物导致穿模或卡死。
    *   *处理*：在执行 `Crouch -> Walk` 状态转换前，向上方发射射线检测。如果有碰撞体阻挡，则拒绝解除下蹲状态，直到玩家移动到开阔区域。

**Crouch → Idle/Walk 状态转换的详细触发条件**：

| 转换类型 | 触发条件 | 说明 |
|---------|---------|------|
| **Crouch → Idle** | 潜行状态下松开方向键 | 停止移动后保持下蹲姿态，进入待机 |
| **Crouch → Walk** | 潜行状态下按下方向键 | 开始移动时自动切换为下蹲行走 |
| **Crouch → Sprint** | 潜行状态下按方向键+冲刺键 | 解除下蹲后立即进入冲刺（需要足够体力） |
| **主动解除** | 再次按潜行键 | 玩家主动切换回站立姿态 |
| **被动解除** | 头顶碰撞检测通过 | 当从低矮掩体移动到开阔区域时自动解除 |

> **注意**：从 Crouch 转换到站立（Idle 或 Walk）时，如果头顶空间不足（上方 `RaycastLength` 范围内有碰撞物），转换会被拒绝，角色保持 Crouch 状态。玩家需要侧向移动到无遮挡区域后才能站立。

## Dependencies

*   **上游依赖 (本系统依赖谁)**：
    *   **Sanity/Rage 系统 (sanity-rage-meter.md)**：硬依赖。读取 `RageSpeedBonus`（愤怒速度加成）用于公式1的速度计算。当玩家处于 FRENZIED 状态时，Sanity/Rage 系统向本系统发送 `MovementSpeedBonus`（范围 0.0~0.1），本系统将其应用于 `CurrentVelocity = InputDirection * BaseSpeed * StateMultiplier * (1.0 + RageSpeedBonus)` 公式中。
*   **下游依赖 (谁依赖本系统)**：
    *   **视野与监听系统 (LOS & Eavesdropping)**: 软依赖（需要本系统提供玩家坐标与面朝方向）。**视野状态查询**：LOS 系统提供 `GetPlayerVisibilityState()` 查询接口，返回玩家当前的视野暴露状态。**移动乘数由本系统自主决定**：Formulas 中的 `StateMultiplier` 由本系统根据玩家当前移动状态（Walk/Sprint/Crouch）独立计算，无需引用 LOS 系统。
    *   **NPC AI系统 (NPC AI System)**: 软依赖（需要本系统广播的 `NoiseEvent`）。
    *   **环境交互系统 (Environment Interaction)**: 硬依赖（需要本系统触发射线检测）。
    *   **沉重处决系统 (Gritty Takedowns)**: 硬依赖（需要接管本系统的 `IsLocked` 状态）。

## Tuning Knobs

*   *这些参数将暴露给策划在 Unity Inspector 中直接调整，无需修改代码。*
*   `BaseSpeed` (基础移速): **5.0 m/s**（安全范围: 3.0 - 8.0 m/s）
*   `SprintMultiplier` (冲刺速度乘数): **1.6**（安全范围: 1.4 - 2.0）
*   `CrouchMultiplier` (下蹲速度乘数): **0.5**（安全范围: 0.3 - 0.7）
*   `MaxStamina` (最大体力值): **100**（安全范围: 50 - 200）
*   `StaminaDrainRate` (冲刺时的体力消耗速率): **20/s**（安全范围: 10 - 40/s）
*   `StaminaRegenRate` (体力恢复速率): **15/s**（安全范围: 10 - 30/s）
*   `StaminaRegenDelay` (停止冲刺到开始恢复体力的延迟时间): **1.0 s**（安全范围: 0.5 - 2.0 s）
*   `StaminaRegenPenaltyThreshold` (体力耗尽后恢复阈值): **0.30 (30%)**（安全范围: 0.20 - 0.40）
*   `NoiseRadius_Walk` (行走时的噪音广播半径): **3.0 m**（安全范围: 1.0 - 5.0 m）
*   `NoiseRadius_Sprint` (冲刺时的噪音广播半径): **8.0 m**（安全范围: 5.0 - 15.0 m）
*   `NoiseRadius_Crouch` (潜行时的噪音广播半径): **0.0 m**（静音）
*   `RaycastLength` (射线检测长度): **2.0 m**（安全范围: 1.0 - 3.0 m）
*   `RaycastConeAngle` (射线检测锥形角度): **60°**（安全范围: 30° - 90°）
*   `LockTimeout` (锁定超时时间): **5.0 秒**（安全范围: 3.0 - 10.0 秒）

## Visual/Audio Requirements

*   **动画**：需要 Idle, Walk, Sprint, Crouch Walk 至少四套基础移动动画。
*   **音效**：脚步声需要根据材质（水泥、水洼、地毯）进行区分。冲刺时的脚步声必须明显比行走时沉重且响亮。当体力耗尽时，需要加入主角剧烈喘息的音频反馈。
*   **特效**：冲刺时脚底可增加轻微的扬尘粒子效果（视美术风格而定）。

## UI Requirements

*   **体力槽 (Stamina Bar)**：仅在不满（消耗或恢复中）时显示，完全恢复后自动淡出隐藏，以保持画面的沉浸感。
*   **状态提示**：潜行状态下，屏幕边缘或 UI 给予轻微的暗角/图标提示，告知玩家当前处于隐蔽模式。

## Acceptance Criteria

*   玩家可以通过 WASD 进行平滑的 8 向移动，且对角线移动速度不异常增加。
*   按下切换键可以正确在 Walk 和 Crouch 状态间切换，碰撞体高度发生可见改变。
*   按住冲刺键可以进入 Sprint 状态，体力槽开始下降；体力为 0 时强制退出冲刺并锁定，直到体力恢复至阈值。
*   移动停止时，角色保持最后的面朝方向不变。
*   按下攻击键（模拟处决）期间，所有移动输入被无视，直到模拟动画结束。
*   （跨系统验证）当玩家处于不同移动状态时，能向外广播带有正确半径的噪音数据结构。

## Open Questions

*   *问题1*：体力耗尽后的”喘息状态”是否需要限制角色的转身速度？（目前仅限制为 Walk 速度，由后续手感测试决定）
*   *问题2*：在斜坡或楼梯上移动时，速度是否需要做坡度衰减？（如果在 3D 物理下开发需要考虑，纯 2D 则可忽略）

## Change Log

| 日期 | 修改人 | 修改内容 |
|------|--------|----------|
| 2026-04-13 | Claude | 修复 P1 问题：1) 公式1中 Attack 状态说明更清晰，明确速度为零；2) 状态转换表中 Action 行补充”速度为0”和”动画完成后自动释放”描述；3) 在 Tuning Knobs 中添加 LockTimeout 参数 |
| 2026-04-15 | Claude | 修复 P2 问题：1) 状态转换表添加 Attack 状态条目（含进入/退出条件、行为、与移动的关系）；2) Crouch→Stand 头顶检测引用统一参数 RaycastLength 替代硬编码值 1.8m；修复 P1 问题：在状态转换表 StaminaExhausted 行添加"修饰符（非独立状态）"标注，消除状态机歧义 |
