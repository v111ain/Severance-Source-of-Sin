# Health & Lethality (脆弱度与伤害系统)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-03-29
> **Implements Pillar**: 致命的脆弱感 (Lethal Fragility)

## Overview

脆弱度与伤害系统 (Health & Lethality) 是定义游戏容错率的底层模块。与传统的数值血量不同，本游戏采用极简的“致死性”判定：玩家和人类敌人一样，都是血肉之躯。本系统不处理复杂的伤害公式或抗性减免，而是处理“是否命中”、“是否致命”以及“受击后的硬直/倒地状态”。它是实现游戏“致命脆弱感”支柱的数值化体现。

## Player Fantasy

**“走错一步，就是死亡。”**

这个系统存在的目的不是为了让玩家感到强大，而是为了让玩家感到**畏惧和公平**。
*   **对等法则**：玩家必须明白，自己用来轻易杀死敌人的手段（一枪爆头、背后暗杀），敌人同样可以轻易用在自己身上。
*   **刀尖舔血**：因为一击必杀的风险永远悬在头顶，所以当玩家成功策划并执行了一套无伤连杀时，肾上腺素的爆发和成就感会极其强烈。
*   参考《迈阿密热线》中一颗流弹带来的瞬间重开，以及《只狼》中绝地难度下的被一击致命。

## Detailed Design

### Core Rules

*   **非传统血量**：本系统不使用 100/100 的连续血量槽，而是使用**离散的“状态标签”**（健康、硬直、重伤倒地、死亡）。
*   **伤害类型划分 (Tiered Damage)**：
    *   `Lethal` (致命伤害)：如枪击、利器刺穿、高处坠落重物砸中、背刺处决。
    *   `Blunt` (钝击伤害)：如空手挥拳、普通木棍挥击、被门撞到。
*   **部位与护甲 (Hitboxes & Armor)**：
    *   受击部位分为：`Head` (头部), `Torso` (躯干), `Limbs` (四肢 - 视实现成本可选合并入躯干)。
    *   防弹衣 (Armor) 仅覆盖 `Torso`。它可以抵挡一次 `Lethal` (子弹) 伤害，使目标进入 `Staggered` (硬直) 状态而非立即死亡（护甲随之损坏）。护甲无法抵御穿透伤害（高穿透武器可无视护甲直接致死），也无法抵御近战处决。
    *   **注**：护甲抵挡 Lethal 伤害的结果是 `Staggered` 状态，与穿透规则中的"穿透失败 → Staggered"机制一致。穿透规则描述的是穿透力评估后的结果，此处描述的是护甲的标准防护机制。

### States and Transitions

所有具有生命体的实体（玩家和NPC）共享以下状态机：

| 当前状态 | 受到的伤害类型 & 部位 | 目标状态 | 效果描述 |
| --- | --- | --- | --- |
| **Healthy (健康)** | `Lethal` (命中无护甲部位) | **Dead (死亡)** | 瞬间死亡，触发Ragdoll或死亡动画。 |
| **Healthy** | `Lethal` (命中护甲部位) | **Staggered (硬直)** | 护甲爆裂，角色陷入大硬直（约1-2秒），无敌帧仅限硬直动画前摇。 |
| **Healthy** | `Blunt` (任意部位) | **Staggered (硬直)** | 角色被打断当前动作，陷入硬直。 |
| **Staggered (硬直)** | `Lethal` (任意部位) | **Dead (死亡)** | 毫无还手之力地被击杀。 |
| **Staggered** | `Blunt` (任意部位) | **Downed (倒地/重伤)** | 连续遭到钝击，角色失去平衡倒地。此时可被执行“地面处决”。 |
| **Downed (倒地)** | 任何伤害 | **Dead (死亡)** | 补刀。如果长时间未受击，可能缓慢恢复至 Staggered 并爬起。 |

### Interactions with Other Systems

*   **监听 [所有攻击行为]**：系统接收一个 `DamageEvent` 数据包（包含：攻击者、受击者、伤害类型、命中部位）。
*   **事件广播**：
    - 当 NPC 状态发生转移时，由 NPC AI 系统广播 `NPCStateChangedEvent`（Health System 触发后 NPC AI 系统负责广播）
    - 当玩家状态发生转移时，Health System 广播 `PlayerDamagedEvent`
*   **提供给 [沉重处决系统]**：判定目标是否处于 `Staggered` 或 `Downed` 状态，这是许多正面环境处决的触发前置条件（即：先用砖头砸晕，再进行处决）。

**`PlayerDamagedEvent` 数据结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `player_id` | int | 玩家实体 ID |
| `damage_type` | DamageType | LETHAL / BLUNT |
| `hit_location` | HitLocation | HEAD / TORSO / LIMBS |
| `source_entity_id` | int | 伤害来源实体 ID |
| `is_lethal` | bool | 是否为致命伤害 |

**`NPCStateChangedEvent` 数据结构**（由 NPC AI 系统广播）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `npc_id` | int | NPC 实体 ID |
| `entity_type` | EntityType | NPC（固定值） |
| `old_state` | HealthState | 变化前的状态：Healthy / Staggered / Downed / Dead |
| `new_state` | HealthState | 变化后的状态 |
| `damage_type` | DamageType | LETHAL / BLUNT / NONE（用于区分死亡原因） |

**订阅方行为指南**：
- Immersive Audio 订阅 `PlayerDamagedEvent`，根据 `new_state` 判定震动：
  - `new_state == Staggered` → 触发 `player_hurt` 震动
  - `new_state == Dead` → 触发 `player_dead` 震动

## Formulas

*   本系统没有常规的 `HP = HP - Damage` 公式。
*   **恢复判定公式** (仅用于敌人从倒地状态苏醒)：
    `RecoveryProgress = TimeSinceLastHit / DownedRecoveryTime`
    当 `RecoveryProgress >= 1.0` 时，状态从 `Downed` 转移回 `Staggered`。

## Edge Cases

*   **多重伤害同时帧判定**：
    *   *问题*：如果同一帧受到一个 Blunt 和一个 Lethal 伤害如何处理？
    *   *处理*：Lethal 伤害优先级永远高于 Blunt。如果同时发生，只结算 Lethal 伤害。
*   **穿透伤害**：
    *   *问题*：高威力子弹（如狙击枪）打穿带护甲的躯干。
    *   *处理*：武器系统定义 `Penetration` 属性（1-10级）。穿透判定公式如下：

```
# 穿透判定（每次命中时重新评估）
if (ArmorState == DESTROYED):
    # 护甲已损毁，无视护甲，直接判定
    if (DamageType == Lethal):
        TargetState = DEAD
    else:
        TargetState = STAGGERED
elif (weapon_penetration > target_armor_level):
    # 穿透成功，无视护甲，直接触发 Lethal 死亡
    TargetState = DEAD
else:
    # 穿透失败，消耗护甲耐久，触发 Staggered
    ArmorDurability -= 1
    if (ArmorDurability <= 0):
        ArmorState = DESTROYED
    TargetState = STAGGERED
```

| 变量 | 定义 | 典型值 |
|------|------|--------|
| weapon_penetration | 武器的穿透等级（1-10） | 手枪=3, 步枪=6, 狙击枪=9 |
| target_armor_level | 目标的护甲等级（1-10） | 轻装=2, 重装=5, 超重装=8 |
| ArmorDurability | 护甲可承受穿透失败的次数 | 1-2次 |
| ArmorState | 护甲状态 | ACTIVE / DESTROYED |

**穿透机制说明**：
- 每次命中时，根据当前护甲状态独立判定
- 如果护甲已损毁，后续命中直接作用于角色（无论穿透值多高）
- 如果穿透值 > 护甲等级，穿透成功，角色立即死亡
- 如果穿透值 ≤ 护甲等级，穿透失败，消耗1点护甲耐久，角色进入 Staggered
- 护甲耐久耗尽后变为 DESTROYED 状态，不再提供保护
*   **倒地无敌帧**：
    *   *问题*：敌人倒地过程中模型碰撞体发生剧烈变化，导致后续子弹打空。
    *   *处理*：角色在播放 `Staggered -> Downed` 动画的下落期间，依然接收 Lethal 伤害并可随时转化为死亡。

## Dependencies

*   **上游依赖**：无 (基础层)。
*   **下游依赖**：
    *   **NPC AI系统**: 软依赖 (需要依据自身和友军的健康状态切换行为树)。
    *   **沉重处决系统**: 硬依赖 (需要查询目标是否处于 Staggered/Downed 状态以触发环境处决；向 Health 系统发送 DamageRequest 执行伤害)。

## Tuning Knobs

*   `StaggerDuration` (硬直持续时间，默认 1.5秒)
*   `DownedRecoveryTime` (倒地后自动苏醒的时间，默认 15秒)
*   `ArmorDurability` (护甲可抵挡致命伤的次数，默认 1，重装兵可为 2)

## Visual/Audio Requirements

*   **视觉**：
    *   需要被子弹命中的飙血特效（方向需与子弹弹道一致）。
    *   护甲碎裂时，需要有明显的视觉特效（如防弹衣碎片飞出）和模型材质变化（防弹衣破损）。
    *   状态转换需要有明确的动画：Staggered（踉跄后退）、Downed（跪地或趴下）。
*   **听觉**：
    *   击中肉体（沉闷的噗声）、击中护甲（清脆的防弹衣闷响）、击中头部（特殊的致命反馈音效）。

## UI Requirements

*   **极简UI**：没有传统的血条显示。玩家状态通过屏幕边缘特效（如濒死倒地时的屏幕变灰/血丝）来传达。敌人的状态（健康/硬直/倒地）完全通过动画姿态来展现。
*   **护甲提示**：如果在UI中不显示护甲值，敌人的护甲状态必须在角色模型上有非常高对比度的视觉特征。

## Acceptance Criteria

*   任何实体（玩家或敌人）在无护甲状态下被 Lethal 伤害击中躯干或头部，必须立刻转为死亡状态。
*   穿戴护甲的实体被 Lethal 伤害击中躯干时，护甲扣除 1 点耐久，实体进入 Staggered 状态且不会死亡。
*   如果使用带有高 Penetration 属性的武器击中护甲，可以无视护甲直接致死。
*   实体受到 Blunt 伤害时，播放硬直动画；连续受到 Blunt 伤害时正确转入 Downed 倒地状态。
*   状态切换时（如受击硬直），必须能够正确向世界广播事件，以便 AI 和处决系统能监听到。

## Open Questions

*   *问题1*：是否允许”部位破坏”？（例如打碎膝盖导致敌人只能在地上爬行）。目前设计中暂时没有，以控制动画成本，但可作为后续 Alpha 阶段的评估项。
*   *问题2（已解决）*：**玩家死亡条件澄清** — 玩家的唯一死亡条件是 Lethal 伤害直接致死。Downed 状态仅适用于 NPC，玩家不受 Downed 恢复机制影响。这与”致命的脆弱感”支柱保持一致。
*   *问题3（已解决）*：**事件命名统一** — Health 系统发送 `PlayerDamagedEvent`（玩家受伤）和触发 NPC AI 系统的 `NPCStateChangedEvent`（NPC 状态变化）。详见 Interactions with Other Systems。
*   *问题3（已解决）*：**穿透与护甲判定顺序** — 判定顺序为：1) 检查 Penetration > ArmorLevel → 直接致死；2) Penetration <= ArmorLevel → 消耗护甲，转为 Staggered，ArmorLevel--。护甲耗尽后，Lethal 打击直接致死。
