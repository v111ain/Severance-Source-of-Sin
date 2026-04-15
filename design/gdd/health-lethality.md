# Health & Lethality (脆弱度与伤害系统)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-04-10
> **Implements Pillar**: 致命的脆弱感 (Lethal Fragility)
> **Revision Notes**: 2026-04-08 协同修订（配合武器系统修复设计审查问题）：
> - 补充 `ExplosionEvent` 数据结构定义
> - 补充 Dependencies 接口说明表（含空间分区优化归属说明）
> **2026-04-13 语义修复**：
> - P1: Line 125 穿透公式注释歧义修复："穿透成功"与"无视护甲"的逻辑关系用 `→` 箭头明确连接，避免与赋值语句的 `=` 混淆
> - P1: 战斗团队评审修复：
> - P1: 爆炸伤害边界统一使用 `<=`（边界=致死），与 Weapon System 保持一致
> - P1: 穿透公式使用 `>=`（穿透值 ≥ 护甲等级 = 穿透成功），Open Questions 描述已修正
> - P2: 补充 Tuning Knobs 安全范围（StaggerDuration、DownedRecoveryTime、ArmorDurability）
> - P2: 补充 Downed 恢复路径说明（地面处决窗口期 → Staggered → Healthy）
> - P2: 新增 `ExplosionAlertEvent` 和 `FriendlyFireExplosionEvent` 接口定义

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
    - 当 NPC 状态发生转移时，由 Health System 广播 `NPCStateChangedEvent`，NPC AI 系统订阅此事件
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

**`NPCStateChangedEvent` 数据结构**（由 Health 系统广播，NPC AI 系统订阅）：

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
    当 `RecoveryProgress >= 1.0` 时，状态从 `Downed` 转移回 `Staggered`。Staggered 状态在持续 `StaggerDuration` 时间后，自动转移回 `Healthy`。

**NPC 状态完整生命周期**：
```
Healthy ←─────── StaggerDuration ─────── Staggered
  ▲                                        │
  │                                        │ (再次受击 Blunt)
  │                                        ▼
  │                                   Downed
  │                                        │
  └─────── RecoveryProgress >= 1.0 ←───────┘
```

## Edge Cases

*   **多重伤害同时帧判定**：
    *   *问题*：如果同一帧受到一个 Blunt 和一个 Lethal 伤害如何处理？
    *   *处理*：Lethal 伤害优先级永远高于 Blunt。如果同时发生，只结算 Lethal 伤害。
*   **穿透伤害**：
    *   *问题*：高威力子弹（如狙击枪）打穿带护甲的躯干。
    *   *处理*：武器系统定义 `Penetration` 属性（1-10级）。穿透判定公式如下：

```
# 穿透判定（每次命中时重新评估）
# ============================================================
# 【设计决策】当穿透成功时，护甲完全不生效，目标直接死亡
# ============================================================
if (ArmorState == DESTROYED):
    # 护甲已损毁，无视护甲，直接判定
    if (DamageType == Lethal):
        TargetState = DEAD
    else:
        TargetState = STAGGERED
elif (weapon_penetration >= target_armor_level):
    # 穿透值 >= 护甲等级 → 穿透成功（无视护甲，直接致死）
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
- 如果穿透值 >= 护甲等级，穿透成功，角色立即死亡
- 如果穿透值 < 护甲等级，穿透失败，消耗1点护甲耐久，角色进入 Staggered
- 护甲耐久耗尽后变为 DESTROYED 状态，不再提供保护
- **护甲恢复机制**：

护甲是"有限的战术资源"，其恢复机制设计必须服务于"致命的脆弱感"核心设计。

**恢复方式**：

| 恢复类型 | 恢复内容 | 触发条件 | 典型场景 |
|---------|---------|---------|---------|
| **主动修复** | 护甲耐久完全恢复 | 消耗医疗包资源或访问特定 NPC | 战斗间隙的资源管理 |
| **自动解除** | 护甲状态从 DESTROYED 恢复为 ACTIVE，耐久不自动回复 | 脱离战斗后等待冷却时间 | 战斗失败/撤退后的惩罚性恢复 |

**主动修复详细规则**：
- 玩家可使用 `ArmorRepairKit`（护甲修复包）道具主动修复护甲
- 修复使用条件：护甲状态为 ACTIVE（未损毁）但耐久 < 最大值，或刚经历战斗（冷却时间 3 秒内未受到伤害）
- 修复效果：护甲耐久恢复至最大值（Light=1, Heavy=2）
- 修复过程中角色处于轻微硬直（约 0.5 秒），无法移动或攻击

**自动解除详细规则**：
- 护甲进入 DESTROYED 状态后，经过 `ArmorReactivationDelay` 时间自动解除损毁
- 解除后护甲状态变为 ACTIVE，但耐久归零（称为 FRAGILE 脆弱状态）
- 脆弱状态下的护甲再承受一次穿透失败将直接永久损毁（PERMANENTLY_DESTROYED，无法自动解除，只能通过道具修复）
- 自动解除计时器在玩家受到任何伤害时重置（防止玩家蹲守等待）

**护甲状态机扩展**：
```
ACTIVE (有护甲)
    │
    │ (穿透失败 -1 耐久)
    ▼
WORN (耐久 > 0) ────► 主动修复 ───► ACTIVE
    │
    │ (耐久归零)
    ▼
DESTROYED (损毁)
    │
    │ (ArmorReactivationDelay 后自动解除)
    ▼
FRAGILE (脆弱) ────► 主动修复 ───► ACTIVE
    │
    │ (再承受穿透失败)
    ▼
PERMANENTLY_DESTROYED (永久损毁，无法自动恢复)
```

**护甲恢复公式**：

1. **自动解除延迟计算**：
   `ReactivationTimer = ArmorReactivationDelay × (1 - ArmorQualityBonus)`

   | 变量 | 定义 | 典型值 |
   |------|------|--------|
   | `ArmorReactivationDelay` | 基础自动解除延迟 | 60s |
   | `ArmorQualityBonus` | 护甲品质修正（影响自动解除速度） | Light=0.0, Heavy=0.3 |
   | `ReactivationTimer` | 实际需要的等待时间 | Light=60s, Heavy=42s |

2. **脆弱状态惩罚**：
   - 脆弱状态下的护甲被穿透失败时，直接永久损毁
   - 脆弱状态下的护甲被修复后，品质降级（Heavy → Light）

**战术影响分析**：
- 护甲恢复机制迫使玩家在"使用资源修复"和"冒险继续战斗"之间做出选择
- 自动解除机制确保护甲不会永久损失，但等待过程使玩家在脆弱期暴露于风险中
- FRAGILE 状态的引入增加了"护甲即将永久损毁"的高风险状态，提升紧张感
- 修复过程的硬直使修复决策具有风险（被敌人趁虚而入）

*   **爆炸伤害（混合型）**：
    *   *问题*：手榴弹、C4 等爆炸物造成范围伤害，与穿透伤害是独立机制。
    *   *处理*：爆炸伤害由武器系统计算，Health 系统执行最终判定：

**`ExplosionEvent` 数据结构**（武器系统 → Health 系统）：
```csharp
ExplosionEvent:
    position: Vector3        // 爆炸中心位置
    radius: float           // 爆炸半径（米）
    lethal_radius_ratio: float      // 致死半径比例（如 0.3 表示 30% 半径内致死）
    base_damage: float      // 基础伤害（爆炸中心）
    stagger_multiplier: float // 硬直伤害倍率（远距离Blunt伤害的硬直效果放大系数）
```

**统一命名说明**：为与武器系统保持一致，本系统所有 `lethal_ratio` 参数更名为 `lethal_radius_ratio`。

**空间分区参数**：
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `SpatialPartitionType` | enum | Quadrant（2D）/ Octree（3D） | 空间分区算法选择 |
| `SpatialPartitionCellSize` | float | 2.0m | 分区单元格大小 |
| `SpatialPartitionMaxDepth` | int | 8 | 最大递归深度 |
| `SpatialPartitionRebuildInterval` | float | 0.0s | 重建间隔（0=每帧重建） |

        1. Health 系统接收 `ExplosionEvent(position, radius, lethal_radius_ratio, base_damage, stagger_multiplier)`
        2. 使用空间分区（Quadrant/Octree）预筛选范围内实体，避免 O(n) 全实体遍历
        3. 对范围内每个实体，基于距离计算伤害类型：
           - 若 `distance <= radius × lethal_radius_ratio`：Lethal 伤害 → DEAD
           - 若 `distance > radius × lethal_radius_ratio` 且在爆炸半径内：`Blunt` 伤害 → Staggered/Downed
        4. 爆炸伤害绕过护甲直接作用于角色（护甲不被摧毁）
        5. **爆炸警报广播**：当爆炸造成任意 NPC 死亡时，Health 系统广播 `ExplosionAlertEvent`：
           - `position`：爆炸中心
           - `radius`：爆炸半径（用于计算影响范围）
           - `victim_id`：被击杀的 NPC ID
           - `killer_is_player`：是否由玩家引爆
           - NPC AI 系统订阅此事件后，强制范围内所有 NPC（除 victim）进入 ALERT 状态
        6. **友军误伤处理**：当 NPC 被同派系成员的爆炸物炸死时，Health 系统广播 `FriendlyFireExplosionEvent`：
           - `victim_id`：被炸死的 NPC
           - `killer_id`：引爆爆炸物的 NPC（可能是友军）
           - `faction_relation`：凶手与受害者的派系关系
           - NPC AI 系统根据派系关系计算惩罚：友军误伤 → faction_hostility += 30；凶手进入 GUILT 特殊状态

*   **倒地无敌帧**：
    *   *问题*：敌人倒地过程中模型碰撞体发生剧烈变化，导致后续子弹打空。
    *   *处理*：角色在播放 `Staggered -> Downed` 动画的下落期间，依然接收 Lethal 伤害并可随时转化为死亡。

## Dependencies

*   **上游依赖**：无 (基础层)。
*   **下游依赖**：
    *   **武器系统 (Weapon System)**: 软依赖 (武器系统发送 `DamageRequest` 和 `ExplosionEvent` 到本系统，但不依赖本系统的输出)。
    *   **NPC AI系统**: 硬依赖 (NPC AI 系统订阅本系统广播的 `NPCStateChangedEvent`，需要依据自身和友军的健康状态切换行为树)。
    *   **沉重处决系统**: 硬依赖 (需要查询目标是否处于 Staggered/Downed 状态以触发环境处决；向 Health 系统发送 DamageRequest 执行伤害)。
    *   **道具/库存系统 (Item System)**: 硬依赖 (护甲修复需要消耗 `ArmorRepairKit` 道具；道具系统需提供查询/扣除/冷却机制)。

**接口扩展**：

| 接口 | 方向 | 负载 | 说明 |
|------|------|------|------|
| `ArmorRepairRequest` | → 道具系统 | `{player_id, armor_type}` | 玩家请求修复护甲 |
| `ArmorRepairResult` | ← 道具系统 | `{success, remaining_kits}` | 修复请求的结果（成功/失败/资源不足） |
| `ArmorRepairCooldownQuery` | → 道具系统 | `{player_id}` | 查询玩家是否处于修复冷却期 |
| `ArmorRepairCooldownEvent` | ← 道具系统 | `{player_id, cooldown_remaining}` | 冷却时间更新事件（用于 UI 显示） |

**接口说明**：

| 接口 | 方向 | 负载 | 说明 |
|------|------|------|------|
| `DamageRequest` | ← 武器系统/沉重处决 | `{target_id, damage_type, penetration, source}` | 伤害请求 |
| `ExplosionEvent` | ← 武器系统 | `{position, radius, lethal_radius_ratio, base_damage, stagger_multiplier}` | 爆炸物引爆（混合型：近距离Lethal，远距离Blunt）。`stagger_multiplier`参数值来源于武器系统的 Tuning Knobs 参数 `StaggerMultiplier`（默认值 1.5，安全范围 1.0~2.0），由武器系统在构造 ExplosionEvent 时直接传入。Health 系统不关心其计算过程，只负责将其应用于 Blunt 伤害的硬直效果计算 |
| `NPCStateChangedEvent` | → NPC AI 系统 | `{npc_id, old_state, new_state, damage_type}` | NPC 状态变化（由 Health 系统广播，NPC AI 系统订阅） |
| `PlayerDamagedEvent` | → 音频系统 | `{player_id, damage_type, hit_location, is_lethal}` | 玩家受击（由 Health 系统广播） |
| `ExplosionAlertEvent` | → NPC AI 系统 | `{position, radius, victim_id, killer_is_player}` | 爆炸物引爆时广播，强制范围内 NPC 进入 ALERT 状态（详见 Edge Cases） |
| `FriendlyFireExplosionEvent` | → NPC AI 系统 | `{victim_id, killer_id, faction_relation, explosion_position}` | 友军误伤爆炸时广播，触发派系感知惩罚（详见 Edge Cases） |

## Tuning Knobs

| 参数 | 类型 | 默认值 | 安全范围 | 说明 |
|------|------|--------|---------|------|
| `StaggerDuration` | float | 1.5s | 1.0s ~ 2.5s | 硬直持续时间。太短失去"硬直惩罚"意义，太长让敌人无敌 |
| `DownedRecoveryTime` | float | 15s | 10s ~ 30s | 倒地后自动苏醒时间。太短让"地面处决"窗口价值降低，太长玩家可从容补刀。苏醒后先进入 Staggered，再经 StaggerDuration 转回 Healthy |
| `ArmorDurability_Light` | int | 1 | 1 | 轻装护甲：可承受穿透失败的次数 |
| `ArmorDurability_Heavy` | int | 2 | 2 | 重装护甲：可承受穿透失败的次数 |
| `ArmorReactivationDelay` | float | 60s | 30s ~ 120s | 护甲从 DESTROYED 状态自动解除损毁的延迟时间。过长使玩家脆弱期太长，过短让护甲失去"稀缺感" |
| `ArmorQualityBonus_Light` | float | 0.0 | 0.0 | 轻装护甲的品质修正（影响自动解除速度）。轻装护甲品质较低，无修正 |
| `ArmorQualityBonus_Heavy` | float | 0.3 | 0.2 ~ 0.4 | 重装护甲的品质修正。重装护甲材质更好，自动解除更快 |
| `ArmorRepairStaggerDuration` | float | 0.5s | 0.3s ~ 1.0s | 使用修复包时的硬直持续时间。太短使修复无风险，太长让敌人有可乘之机 |
| `ArmorRepairCooldown` | float | 3s | 2s ~ 5s | 修复冷却时间（最近受击后的等待期）。防止玩家边打边修无脑续航 |

**安全范围设计依据**：

| 参数 | 低于下限的影响 | 高于上限的影响 |
|------|---------------|---------------|
| `StaggerDuration` < 1.0s | 硬直几乎无法被利用，失去处决窗口 | 敌人硬直过长，玩家可无风险连击 |
| `DownedRecoveryTime` < 10s | "地面处决"决策时间过短 | 玩家可从容处理多个倒地敌人 |
| `ArmorDurability` > 2 | — | 护甲过强，削弱 Lethal 武器价值 |
| `ArmorReactivationDelay` < 30s | 脆弱期过短，护甲稀缺感不足 | 脆弱期过长，玩家被迫放弃护甲战术 |
| `ArmorQualityBonus` < 0.2 或 > 0.4 | 品质差异不明显，护甲分级失去意义 | 重装护甲过于强力 |
| `ArmorRepairStaggerDuration` < 0.3s | 修复无风险，破坏资源管理决策 | 修复过于危险，玩家放弃修复 |
| `ArmorRepairCooldown` < 2s | 玩家可边打边修，破坏战斗节奏 | 冷却过长，战斗后修复等待感明显 |

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
*   **护甲恢复验收**：
    *   护甲进入 DESTROYED 状态后，经过 `ArmorReactivationDelay` 时间自动解除，状态变为 FRAGILE，耐久归零。
    *   玩家使用修复包后，护甲耐久恢复至最大值，状态从 FRAGILE/WORN 转回 ACTIVE。
    *   FRAGILE 状态下的护甲再承受穿透失败，直接永久损毁为 PERMANENTLY_DESTROYED，无法自动恢复。
    *   修复过程中玩家被强制进入轻微硬直（`ArmorRepairStaggerDuration`），无法移动或攻击。
    *   修复冷却时间（`ArmorRepairCooldown`）内未受到伤害才允许修复，防止边打边修。

## Open Questions

*   *问题1*：是否允许”部位破坏”？（例如打碎膝盖导致敌人只能在地上爬行）。目前设计中暂时没有，以控制动画成本，但可作为后续 Alpha 阶段的评估项。
*   *问题2（已解决）*：**玩家死亡条件澄清** — 玩家的唯一死亡条件是 Lethal 伤害直接致死。Downed 状态仅适用于 NPC，玩家不受 Downed 恢复机制影响。这与”致命的脆弱感”支柱保持一致。
*   *问题3（已解决）*：**事件命名统一** — Health 系统发送 `PlayerDamagedEvent`（玩家受伤）和触发 NPC AI 系统的 `NPCStateChangedEvent`（NPC 状态变化）。详见 Interactions with Other Systems。
*   *问题3（已解决）*：**穿透与护甲判定顺序** — 判定顺序为：1) 检查 Penetration >= ArmorLevel → 直接致死（穿透成功，无视护甲）；2) Penetration < ArmorLevel → 穿透失败，消耗护甲耐久，转为 Staggered。护甲耗尽后（ArmorState = DESTROYED），后续 Lethal 打击直接致死。注意：穿透值与护甲等级相等时视为穿透成功（>=），这是经过修订后的一致版本。
*   *问题4（已解决）*：**Downed 状态恢复路径** — NPC 从 Downed 恢复时，路径为：Healthy → (受击) → Staggered → (再受击) → Downed → (恢复计时) → Staggered → (StaggerDuration 结束) → Healthy。NPC 不会直接从 Downed 跳回 Healthy，必须先经过 Staggered 状态作为过渡。玩家不受 Downed 机制影响（详见问题2）。
*   *问题5（已解决）*：**护甲恢复机制** — 护甲恢复采用”主动修复 + 自动解除”混合机制。主动修复（消耗道具）完全恢复耐久；自动解除在等待 `ArmorReactivationDelay` 后将 DESTROYED 转为 FRAGILE（耐久归零）。详见 Edge Cases。
