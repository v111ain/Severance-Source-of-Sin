# 武器系统 (Weapon System)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-04-10
> **Implements Pillar**: 环境即武器 (Environment as a Weapon)、沉重、不洁的暴力 (Gritty, Dirty Violence)
> **Revision Notes**: 2026-04-08 重大修订：
> - 爆炸物重设计（混合型：近距离击杀/远距离高硬直）
> - 可破坏物件武器化（灯泡、电线等补充 WeaponData）
> - 热武器进度引入（来源限制，非时间限制）
> - Visual/Audio Requirements 补充
> - WeaponData 组件化重构
> - 武器切换机制明确
> **2026-04-08 协同修订（修复设计审查问题）**：
> - P0: 订阅环境交互系统的 `ObjectStateChangedEvent`，不再自行维护物件状态
> - P0: 状态机统一，环境交互系统为唯一数据源
> - P1: 动画标签统一引用 `DESTRUCT_*` 和 `WEAPON_MELEE_*`
> - P2: 空间分区优化归属 Health 系统
> - P2: Health 系统补充 `ExplosionEvent` 数据结构
> **2026-04-09 修复 P2**：
> - 明确 `WeaponQueryResponse` 包含 `weapon_id`，用于 Gritty Takedowns 构建 `DamageRequest` 的 `source` 参数
> **2026-04-09 设计审查修复**：
> - P0: 修正 C4 爆炸伤害示例计算错误（1.5m 距离应为 112.5，非 37.5）
> - P1: 澄清 `DESTRUCT_GLASS` 标签在环境交互与武器系统中的语义一致性
> - P1: 补充手榴弹倒计时音效详细规格（触发时机、频率变化、音量变化）
> - P2: 补充 Haptic Feedback 实现规格（平台适配、波形类型、衰减机制）
> **2026-04-10 战斗团队评审修复**：
> - P1: 公式2 投掷物命中判定中 `melee_range` 变量补全定义（1.5m）及安全范围（1.0m~2.0m）

## Overview

武器系统 (Weapon System) 是管理游戏中所有武器和可投掷物件的中央数据模块。本系统统一了"环境即武器"的设计理念——无论是场景中的灭火器、砖块，还是电线，都通过本系统进行武器属性的标准化管理。

核心职责：
1. **武器数据管理**：维护所有武器物件的属性（伤害类型、穿透值、有效距离、动画标签）
2. **环境物件武器化**：将环境交互系统中的可拾取物件赋予武器能力
3. **武器状态追踪**：管理玩家当前持有的武器、可用物件列表、冷却状态
4. **伤害参数输出**：向 Health & Lethality 系统提供标准化的 `DamageRequest`

武器系统不负责判定"何时使用武器"（这是 Gritty Takedowns 的职责），而是负责"武器是什么"和"武器能做什么"。

## Player Fantasy

**"万物皆可致命。"**

玩家在游戏中不应只依赖传统的"武器"（刀、枪），而应随时保持对环境的观察。灭火器可以砸晕敌人，电线可以勒颈，砖块可以投掷。每一个物件都有其独特的战术价值。

核心情感：
- **发现感**：看到灭火器时，脑中立刻浮现"这个可以用来制造混乱或击杀"
- **策划感**：选择一个环境物件作为武器本身就是一种策略决策
- **掌控感**：手中的武器（无论是砖块还是钢管）都有明确的威力和限制

参考《Hotline Miami》中拾取任何物件都能改变局势的设计思路。

## Detailed Design

### Core Rules

**武器分类体系（完整版）**：

| 大类 | 子类 | 伤害类型 | 示例 |
|------|------|----------|------|
| **环境物件 (Environmental)** | 硬质重物 | Lethal/Blunt | 砖块、金属物件、重型垃圾桶 |
| | 软质物件 | Blunt | 枕头、塑料袋、报纸堆 |
| | 锐器 | Lethal | 破碎玻璃、刀片、金属碎片 |
| | 长柄 | Lethal/Blunt | 钢管、电线、雨伞 |
| | 投掷物 | Lethal/Blunt | 砖块、瓶子、花盆 |
| | **可破坏物件** | **Lethal/Blunt** | **灯泡（破碎）、电线（勒颈）、玻璃窗（破碎）** |
| **热武器 (Hot Weapons)** | 手枪 | Lethal (高穿透) | 手枪、冲锋枪 |
| | 步枪/狙击 | Lethal (超高穿透) | 步枪、狙击枪 |
| | 霰弹枪 | Lethal (近距离) | 霰弹枪 |
| | **爆炸物（控制型）** | **Lethal/Blunt (混合型)** | **手榴弹、C4** |
| | 诱饵 | — | 闪光弹、烟雾弹 |

**爆炸物设计（混合型）**：

| 距离范围 | 伤害类型 | 效果 |
|---------|----------|------|
| 0 ~ blast_radius × 0.3 | Lethal | 近距离秒杀（爆炸中心） |
| blast_radius × 0.3 ~ blast_radius | Blunt (高硬直) | 远距离高硬直，但不致死 |

- **设计理念**：玩家必须靠近目标才能秒杀，远距离只是制造混乱/硬直工具
- **blast_radius 缩小**：C4: 8m → 3m，手榴弹: 5m → 2m
- **此设计保留潜行策略价值**：爆炸物不能替代潜行暗杀，只能辅助

**武器数据模型（组件化设计）**：

```csharp
// 组件化设计：武器能力由组件组合而成，支持灵活扩展
WeaponData {
    weapon_id: string              // 唯一标识
    display_name: string           // 显示名称
    category: WeaponCategory       // Environmental / HotWeapon

    // 组件插槽（按需组合）
    damage_component: DamageComponent     // 伤害能力
    range_component: RangeComponent      // 范围/距离能力
    ammo_component: AmmoComponent?      // 弹药能力（热武器专用，可选）
    explosive_component: ExplosiveComponent?  // 爆炸能力（爆炸物专用）
    animation_tags: string[]       // 动画标签列表（用于Gritty Takedowns调用）
    is_consumable: bool            // 是否消耗品
    weight: float                  // 重量单位（用于携带限制）
}

// 伤害组件
DamageComponent {
    damage_type: DamageType        // Lethal / Blunt
    penetration: int               // 1-10（来自Health系统穿透规则）
    base_damage: float             // 基础伤害倍率
}

// 范围组件
RangeComponent {
    range_type: RangeType          // melee / short / medium / long / throwable
    effective_range: float         // 有效距离（米）
    throw_range: float?            // 投掷距离（可选）
}

// 弹药组件（热武器）
AmmoComponent {
    ammo_type: AmmoType            // 手枪弹/步枪弹/霰弹/炸药
    ammo_capacity: int             // 弹夹容量
    current_ammo: int              // 当前弹药
    reload_time: float              // 换弹时间（秒）
}

// 爆炸组件（爆炸物）
ExplosiveComponent {
    blast_radius: float             // 爆炸半径（米）
    base_damage: float             // 基础伤害（爆炸中心）
    lethal_radius_ratio: float      // 致死半径比例（默认0.3，即30%半径内致死）
    detonation_type: DetonationType // 即时/延时/遥控
    is_controllable: bool           // 是否可遥控引爆
}
```

**WeaponTemplateLibrary**：所有 `WeaponData` 预定义为数据资产（ScriptableObject），策划可通过 Inspector 配置新增武器，无需修改代码。

### States and Transitions

**⚠️ 状态说明（重要）**：

武器系统**不维护独立的环境物件状态机**。环境物件的状态由环境交互系统作为**唯一数据源（Single Source of Truth）**管理。

武器系统通过订阅环境交互系统广播的 `ObjectStateChangedEvent` 来同步状态：
- 当收到 `ObjectPickedUp` → 武器系统将对应物件标记为 `Held`
- 当收到 `ObjectDropped` → 武器系统将对应物件标记为 `Available`
- 当收到 `ObjectUsed` → 武器系统根据物件类型更新为 `Used` 或 `Depleted`

**热武器状态机**：

| 状态 | 描述 | 可转移至 | 触发事件 |
|------|------|---------|---------|
| `Stored` | 在背包/枪套中 | `Equipped`（装备） | — |
| `Equipped` | 正在手持 | `Holstered`（收起）、`Used`（开火/投掷） | `WeaponStateChanged(Equipped)` |
| `Holstered` | 收起在枪套 | `Equipped`、`Stored` | `WeaponStateChanged(Holstered)` |
| `Used` | 开火/爆炸中 | `Equipped`（继续射击）、`Holstered`（换弹/收起）、`Empty`（弹药耗尽） | `WeaponStateChanged(Used)` |
| `Empty` | 弹药耗尽 | `Reloading`（换弹中） | `WeaponStateChanged(Empty)` |
| `Reloading` | 换弹中 | `Equipped`（换弹完成） | `WeaponStateChanged(Reloading)` |

**状态转换事件**：

所有状态转换都会触发 `WeaponStateChangedEvent`，供 UI 系统订阅以更新武器图标显示。

```
WeaponStateChangedEvent {
    weapon_id: string
    old_state: WeaponState
    new_state: WeaponState
}
```

### Interactions with Other Systems

**数据流入 (Inputs)**：

| 来源系统 | 数据内容 | 用途 |
|---------|---------|------|
| **环境交互系统** | `ObjectStateChangedEvent(object_id, object_category, new_state, position)` | **订阅环境交互系统的物件状态变化事件**，当收到 `ObjectSpawned` 时注册 WeaponData；当收到 `ObjectPickedUp/Dropped/Used` 时更新持有状态 |
| 玩家控制器 | 玩家持有的物件ID列表 | 查询当前可用武器 |
| 背包/库存系统 | 热武器携带状态（如果有） | 管理热武器栏位 |

**订阅说明**：武器系统**不主动查询**环境物件状态，而是被动接收环境交互系统广播的 `ObjectStateChangedEvent`。这确保了环境交互系统作为物件状态的唯一数据源。

**物件与武器数据映射（完整版）**：

| 环境交互物件类别 | 映射到武器系统 | WeaponData 来源 |
|----------------|--------------|----------------|
| 武器类（砖块、钢管）| 环境物件 | WeaponTemplateLibrary - 砖块/钢管模板 |
| 爆炸类（灭火器、汽油桶）| 环境物件（爆炸物子类）| WeaponTemplateLibrary - 灭火器/汽油桶模板 |
| 投掷类（硬币、瓶盖）| 环境物件（投掷物子类）| WeaponTemplateLibrary - 硬币/瓶盖模板 |
| **可破坏类（灯泡、电线、玻璃窗）** | **环境物件（可破坏子类）** | **WeaponTemplateLibrary - 灯泡/电线/玻璃模板** |

**可破坏物件 WeaponData 定义**：

| 物件 | damage_type | 效果 | 动画标签 |
|------|------------|------|---------|
| 灯泡 | Lethal (近距离) | 破碎后砸向头部，一击死亡 | `DESTRUCT_GLASS` |
| 电线 | Lethal | 勒颈，秒杀 | `WEAPON_MELEE_LG`（长柄类） |
| 玻璃窗（破碎边缘） | Lethal | 刺穿，秒杀 | `DESTRUCT_GLASS` |

**动画标签来源**：动画标签统一引用环境交互系统定义的 `DESTRUCT_*` 和 `WEAPON_MELEE_*` 标签体系。具体映射关系见环境交互系统文档的"物件动画标签系统"章节。

**语义说明**：`DESTRUCT_GLASS` 标签在环境交互系统中表示"玻璃制品被破坏"的动画效果。武器系统在可破坏物件处决中使用同一标签，是因为复用了环境交互系统的动画资产（破碎动画本身），而非重新定义动画。物件破碎后作为武器的"砸向头部"效果是通过动画层叠加实现的，无需新增标签。

**热武器进度引入（来源限制，非时间限制）**：

| 热武器 | 获取来源 | 说明 |
|--------|---------|------|
| 手枪 | 敌人掉落、任务奖励 | 第一章可设计为敌人掉落 |
| 步枪/狙击 | 敌人掉落、任务奖励 | 中后期章节敌人掉落 |
| 手榴弹 | 敌人掉落、任务奖励 | 弹药有限，作为任务工具 |
| C4 | **任务目标相关** | 必须用来炸开门/容器，非自由获取 |
| 诱饵 | 敌人掉落、任务奖励 | 闪光弹/烟雾弹作为战术工具 |

**获取来源由关卡设计师在关卡设计中决定，不做全局时间线限制。**

**数据流说明**：
1. 环境交互系统在场景物件生成时广播 `ObjectStateChangedEvent(object_id=Spawned, object_category, new_state=Available)`
2. 武器系统订阅该事件，根据 `object_category` 从 WeaponTemplateLibrary 查询对应的 `WeaponData` 模板
3. 后续物件状态变化（拾取/丢弃/使用）由环境交互系统广播，武器系统仅做状态同步

**数据流出 (Outputs)**：

| 目标系统 | 数据内容 | 触发条件 |
|---------|---------|---------|
| 沉重处决系统 | `WeaponQueryResponse(weapon_id, WeaponData)` | 处决系统查询武器数据；WeaponData 中包含 `weapon_id`，Gritty Takedowns 将其作为 `source` 参数转发给 Health 系统 |
| Health & Lethality | `DamageRequest(target_id, damage_type, penetration, source)` | 任何武器造成伤害；`source` 来自 WeaponData 的 `weapon_id` |
| Health & Lethality | `ExplosionEvent(position, radius, lethal_ratio, damage)` | 爆炸物引爆（包含致死半径比例） |
| NPC AI系统 | `WeaponAwareness(weapon_id, position, weapon_type)` | 热武器被检测到（警报源） |
| 沉浸式音频系统 | `WeaponUsedEvent(weapon_id, usage_type)` | 武器使用，用于音效触发 |
| UI 系统 | `WeaponStateChangedEvent(weapon_id, old_state, new_state)` | 武器状态变化（更新图标） |

**武器切换机制**：

| 操作 | 按键 | 说明 |
|------|------|------|
| 切换热武器 | `Tab` | 在 Stored 热武器间循环切换 |
| 切换环境物件 | `Q` / `E` | 在 Held 环境物件列表中循环切换 |
| 快速丢弃当前武器 | `X` | 立即丢弃当前持有的环境物件/收起热武器 |
| 查看所有持有武器 | `I` | 打开背包界面（可拖拽排序） |

**切换逻辑**：
- 玩家只能有一把热武器处于 `Equipped` 状态
- 环境物件可以同时持有多个（无上限，建议通过 `X` 快速丢弃管理）
- 丢弃机制替代持有上限设计，提升掌控感

## Formulas

**公式1：爆炸物范围伤害（混合型）**

```
// 混合型爆炸伤害
if (distance <= blast_radius × lethal_radius_ratio):
    // 近距离：Lethal 伤害
    FinalDamageType = Lethal
    FinalDamage = base_damage
else:
    // 远距离：Blunt 伤害（高硬直，但不致死）
    FinalDamageType = Blunt
    FinalDamage = base_damage × (1 - distance / blast_radius) × stagger_multiplier

当 distance >= blast_radius 时，伤害 = 0
```

| 变量 | 定义 | 典型值 |
|------|------|--------|
| base_damage | 爆炸物基础伤害 | 手榴弹=80, C4=150 |
| distance | 受击点与爆炸中心的距离 | 0 ~ blast_radius |
| blast_radius | 爆炸半径 | 手榴弹=2m, C4=3m |
| lethal_radius_ratio | 致死半径比例 | 0.3（30%半径内致死） |
| stagger_multiplier | 硬直伤害倍率 | 1.5（Blunt 伤害放大） |

**示例**：C4爆炸（blast_radius=3m, lethal_radius_ratio=0.3, base_damage=150）
- 距离0m（中心 ≤ 0.9m）：Lethal → **150**（全额致死）
- 距离1.5m（> 0.9m）：Blunt → 150 × (1 - 1.5/3) × 1.5 = 150 × 0.5 × 1.5 = **112.5**（高硬直）
- 距离3m：0（无伤害）

**性能优化说明**：爆炸伤害计算由 Health 系统执行。Health 系统使用空间分区（Quadrant/Octree）预筛选范围内实体，避免 O(n) 全实体遍历。详见 Health & Lethality 系统文档。

**公式2：投掷物命中判定**

```
HitChance(range) = Clamp(1 - (range - melee_range) / (throw_range - melee_range), min_hit_chance, 1.0)
```

| 变量 | 定义 | 典型值 | 安全范围 |
|------|------|--------|---------|
| range | 投掷距离 | 0 ~ throw_range | — |
| melee_range | 近战范围阈值（在此范围内必中） | 1.5m | 1.0m ~ 2.0m |
| throw_range | 投掷物最大距离 | 8m | 5.0m ~ 15.0m |
| min_hit_chance | 最低命中概率 | 0.3 | 0.1 ~ 0.5 |

**公式3：热武器弹药消耗**

```
FirearmDamage(weapon_type, distance) = BaseDamage × RangeMultiplier(distance) × ArmorPenetrationMultiplier
```

| 变量 | 定义 | 典型值 |
|------|------|--------|
| RangeMultiplier | 距离修正 | 近距离=1.0, 远距离=0.7 |
| ArmorPenetrationMultiplier | 穿透修正 | 高穿透武器=1.2 |

## Edge Cases

**边缘情况1：爆炸物在密闭空间引爆**

*问题*：C4在狭小房间引爆，爆炸范围会延伸出房间边界吗？

*处理*：
- 爆炸伤害使用几何距离计算，不考虑物理遮挡
- 碰撞体遮挡只影响视线（LOS），不影响爆炸范围伤害
- 设计师可通过调整 `blast_radius` 适配不同空间大小

**边缘情况2：投掷物命中天花板/墙壁**

*问题*：玩家投掷砖块，砖块弹到墙上再击中NPC（MVP 简化为直线弹道）

*处理*：
- MVP 阶段：使用直线弹道，无弹跳
- Alpha 阶段（物理模拟）：第一次碰撞后伤害衰减 50%，第二次衰减 75%，第三次停止

**边缘情况3：热武器弹药耗尽时切换**

*问题*：玩家正在射击，弹药耗尽会发生什么？

*处理*：
- 最后一发子弹射出后，自动转入 `Empty` 状态
- 如果玩家背包中有同类型弹药，显示换弹提示
- 如果无弹药，提示"弹药耗尽"，玩家必须切换其他武器或寻找弹药

**边缘情况4：玩家同时持有环境物件和热武器**

*问题*：玩家右手拿钢管，左手拿手枪

*处理*：
- 玩家只能有一把热武器处于 `Equipped` 状态
- 环境物件可以同时持有多个（无持有上限，通过 `X` 快速丢弃管理）
- `Tab` 切换热武器，`Q`/`E` 切换环境物件

**边缘情况5：爆炸物被NPC捡起**

*问题*：玩家扔了C4，NPC捡起来扔回来

*处理*：
- NPC 可捡起处于 `Available` 状态的爆炸物（AI 感知）
- NPC 持有爆炸物时，AI 评估威胁级别并可能触发逃跑/呼叫/投掷
- NPC 投掷爆炸物：玩家受到同等爆炸伤害（按混合型公式计算）
- NPC 被自己人的爆炸物炸死：触发派系感知变化

**边缘情况6：爆炸物击杀 vs 潜行击杀的警报扩散差异**

*问题*：爆炸物秒杀 NPC 是否会触发警报扩散？

*处理*：
- 爆炸物属于"范围控制"工具，引爆即暴露
- 爆炸物击杀 NPC 后，范围内所有 AI 进入 ALERT 状态
- 这与"潜行击杀不触发警报"形成鲜明对比，强化风险/回报感

## Dependencies

**上游依赖（本系统依赖谁）**：

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **玩家控制器 (Player Controller)** | 硬依赖 | 读取玩家位置、持有物件列表、状态 |
| **环境交互系统 (Environment Interaction)** | **硬依赖（订阅）** | **订阅 ObjectStateChangedEvent 获取物件状态变化；不自行维护物件状态副本** |

**下游依赖（谁依赖本系统）**：

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **Health & Lethality 系统** | 硬依赖 | 接收 `DamageRequest` 和 `ExplosionEvent` 执行伤害计算 |
| **沉重处决系统 (Gritty Takedowns)** | 硬依赖 | 查询 `WeaponData`；接收 `DamageRequest` 执行处决伤害 |
| **NPC AI系统 (NPC AI System)** | 软依赖 | 接收 `WeaponAwareness` 事件检测热武器威胁 |
| **沉浸式音频系统** | 软依赖 | 接收 `WeaponUsedEvent` 用于音效触发 |
| **UI 系统** | 软依赖 | 接收 `WeaponStateChangedEvent` 更新武器图标 |
| **理智/愤怒系统 (Sanity/Rage)** | 软依赖 | 间接通过 `KillTagEvent` 传递击杀信息 |

**注意**：Health 系统是**接收方**，不依赖武器系统。武器系统向 Health 发送事件，Health 执行判定后广播 `PlayerDamagedEvent` 等。这是单向依赖关系。

**接口边界说明**：

| 接口 | 方向 | 负载 | 说明 |
|------|------|------|------|
| `ObjectStateChangedEvent` | ← 环境交互系统 | `{object_id, object_category, new_state, position}` | **订阅**：物件状态变化事件（包含 Spawned/PickedUp/Dropped/Used） |
| `WeaponQueryRequest` | ← 武器系统 | `{weapon_id}` | Gritty Takedowns 查询武器数据 |
| `WeaponQueryResponse` | → Gritty Takedowns | `{weapon_id, WeaponData}` | 返回武器数据；Gritty Takedowns 使用返回的 `weapon_id` 作为 `DamageRequest` 的 `source` 参数 |
| `WeaponStateChangedEvent` | → UI 系统 | `{weapon_id, old_state, new_state}` | 武器状态变化，更新 HUD 图标 |
| `DamageRequest` | → Health 系统 | `{target_id, damage_type, penetration, source}` | 伤害请求；`source` 为执行伤害的武器 ID |
| `ExplosionEvent` | → Health 系统 | `{position, radius, lethal_ratio, base_damage}` | 爆炸物引爆（混合型：近距离Lethal，远距离Blunt） |
| `WeaponAwareness` | → NPC AI 系统 | `{weapon_id, position, weapon_type}` | 热武器被检测到 |
| `WeaponUsedEvent` | → 音频系统 | `{weapon_id, usage_type}` | 武器使用事件 |

## Tuning Knobs

| 参数 | 类型 | 默认值 | 安全范围 | 说明 |
|------|------|--------|---------|------|
| **穿透等级参数** | | | | | **注：穿透值定义在此处，health-lethality.md 穿透公式引用这些值** |
| `Penetration_Pistol` | int | 3 | 1~6 | 手枪穿透等级（与护甲等级 1-10 比较，≥ 即穿透成功） |
| `Penetration_Rifle` | int | 6 | 3~8 | 步枪穿透等级 |
| `Penetration_Sniper` | int | 9 | 6~10 | 狙击枪穿透等级 |
| `Penetration_Shotgun` | int | 4 | 2~7 | 霰弹枪穿透等级（近距离多弹丸，综合穿透评估） |
| `Penetration_EnviroSmall` | int | 2 | 1~4 | 环境小物件（砖块、玻璃瓶）穿透等级 |
| `Penetration_EnviroLarge` | int | 5 | 3~7 | 环境大物件（钢管、金属重物）穿透等级 |
| **爆炸物参数** | | | | |
| `BlastRadius_C4` | float | 3.0m | 2.0m ~ 5.0m | C4爆炸半径（已缩小） |
| `BlastRadius_Grenade` | float | 2.0m | 1.5m ~ 4.0m | 手榴弹爆炸半径（已缩小） |
| `LethalRadiusRatio` | float | 0.3 | 0.2 ~ 0.5 | 致死半径比例（30%半径内秒杀） |
| `StaggerMultiplier` | float | 1.5 | 1.0 ~ 2.0 | 远距离硬直伤害倍率 |
| `ExplosionBaseDamage_C4` | float | 150 | 100 ~ 200 | C4基础伤害 |
| `ExplosionBaseDamage_Grenade` | float | 80 | 50 ~ 120 | 手榴弹基础伤害 |
| **投掷物参数** | | | | |
| `MinHitChance` | float | 0.3 | 0.1 ~ 0.5 | 投掷物最低命中概率 |
| `ThrowRange_Throwable` | float | 8.0m | 5.0m ~ 15.0m | 投掷物最大距离 |
| `RicochetDamage衰减` | float | 0.5, 0.75 | — | Alpha阶段弹射衰减（第1/2次） |
| **热武器参数** | | | | |
| `ReloadTime_Pistol` | float | 2.0s | 1.5s ~ 3.0s | 手枪换弹时间 |
| `ReloadTime_Rifle` | float | 2.5s | 2.0s ~ 4.0s | 步枪换弹时间 |
| `ReloadTime_Shotgun` | float | 3.0s | 2.5s ~ 5.0s | 霰弹枪换弹时间 |

**调参风险提示**：

| 参数 | 风险 |
|------|------|
| `LethalRadiusRatio` 设置过大 | 近距离秒杀范围过大，削弱控制工具定位 |
| `StaggerMultiplier` 设置过高 | 远距离硬直过强，爆炸物变成万能硬直工具 |
| `BlastRadius` 设置过大 | 爆炸物过于强力，削弱潜行策略 |
| `MinHitChance` 设置过高 | 远距离投掷过于准确，失去风险/回报感 |

## Visual/Audio Requirements

### 视觉风格

**核心关键词**：厚重感、破坏感、金属质感

| 物件类型 | 视觉要求 | 参考 |
|---------|---------|------|
| 硬质重物 | 金属/混凝土材质，厚重阴影，投掷时有轨迹残影 | 《Hotline Miami》 |
| 软质物件 | 布料/塑料变形动画，近战时挤压效果 | 《Payday 2》 |
| 锐器 | 刀刃高光，血迹飞溅方向与刀刃一致 | 《Dishonored》 |
| 长柄 | 钢管挥舞时有重量感，电线勒颈有拉伸变形 | 《The Last of Us》 |
| 爆炸物 | 引爆前有明显警告光/倒计时，爆炸有冲击波扩散 | 《Bioshock》 |

### 爆炸物特效

| 阶段 | 效果 |
|------|------|
| 引爆前 | 闪烁红光（0.5s间隔），音效：滴答声渐强 |
| 引爆瞬间 | 白色闪光 → 橙红色火球膨胀 → 黑色烟尘扩散 |
| 爆炸后 | 持续2s的烟尘粒子，地面烧焦痕迹（永久留存） |

### 音效设计

**近战武器**：

| 武器类型 | 命中音效 | 挥动音效 |
|---------|---------|---------|
| 硬质重物 |沉闷的钝击声 + 骨头碎裂 | 呼啸的风声 + 金属碰撞 |
| 软质物件 | 沉闷的噗声 | 布料撕裂声 |
| 锐器 | 湿滑的刺穿声 | 刀刃破风声 |
| 长柄 | 电线勒颈的窒息声 | 金属呼啸 |

**爆炸物**：

| 阶段 | 音效 | 详细规格 |
|------|------|---------|
| 引爆前 | 手榴弹：3秒倒计时哔哔声（0.5秒间隔，共6次）；C4：无声（静默待命） | 手榴弹倒计时音效触发时机：引爆键按下后立即开始，每0.5秒一次哔哔声，音量逐渐增大（60%→100%），音调逐渐升高（440Hz→880Hz）。C4放置后显示"RDX ARMED"提示，无提示音。 |
| 引爆瞬间 | 沉闷的BOOM声，带低频震动（20Hz sub-bass） | 低频震动持续100ms，与视觉冲击波同步 |
| 爆炸后 | 持续的火焰噼啪声，NPC惨叫 | 火焰噼啪声持续2秒，随烟尘粒子消散渐弱 |

**热武器**：

| 类型 | 开火音效 | 换弹音效 |
|------|---------|---------|
| 手枪 | 短促的枪声（.45 ACP） | 弹匣落地 + 金属插入声 |
| 步枪 | 较长的枪声（带回响） | 金属咔哒声序列 |
| 霰弹枪 | 震撼的散弹爆发声 | 泵动操作的金属声 |

### 震动反馈

| 武器类型 | 震动强度 | 持续时间 | 波形类型 |
|---------|---------|---------|---------|
| 硬质重物命中 | 强（10mm振幅） | 150ms | 脉冲波形，单次爆发 |
| 爆炸物引爆 | 极强（20mm振幅） | 300ms | 低频正弦波 + 高频噪声混合 |
| 手枪开火 | 中（5mm振幅） | 50ms | 短促矩形波 |
| 步枪开火 | 中强（7mm振幅） | 80ms | 短促矩形波，连发时有衰减 |
| 近战空挥 | 弱（2mm振幅） | 30ms | 轻柔正弦波 |

**Haptic Feedback 实现规格**：
- **API 层**：通过 Unity 的 `InputSystem.Haptic` 接口实现跨平台震动
- **平台适配**：
  - PlayStation 5：使用 DualSense 自适应触发器 API，支持不同武器的阻力等级
  - Xbox/PC：使用 Xbox Gamepad rumble，振幅范围 0.0~1.0
  - Switch Pro Controller：使用 HD rumble
- **波形叠加**：同一武器的开火后坐力与命中震动可叠加（如开枪命中敌人）
- **衰减机制**：霰弹枪连发时振幅从 100% 衰减至 60%（模拟后坐力累积）
- **静音场景**：玩家使用消音器时震动强度降低 70%，保持手感反馈但不喧宾夺主

## UI Requirements

### 武器 HUD

**武器图标栏**（屏幕底部中央）：

| 槽位 | 显示内容 | 状态 |
|------|---------|------|
| 热武器槽（1个） | 当前装备的热武器图标 | 弹药数量、是否空仓 |
| 环境物件槽（最多3个） | 当前持有的环境物件图标 | 是否可用、消耗数量 |

**图标状态**：

| 状态 | 视觉表现 |
|------|---------|
| 可用 | 正常显示，白色边框 |
| 冷却中 | 灰色遮罩 + 冷却进度动画 |
| 空弹药/消耗完毕 | 红色边框 + 闪烁 |
| 切换选中 | 黄色边框 + 脉动呼吸动画 |

### 武器切换提示

| 操作 | 屏幕提示 |
|------|---------|
| Tab（切换热武器） | 小型图标弹出显示切换的武器名，0.5s淡出 |
| Q/E（切换环境物件） | 同上 |
| X（丢弃当前武器） | "已丢弃 [武器名]"，0.5s淡出 |

### 爆炸物警告

当爆炸物在范围内时：
- 屏幕边缘红色脉冲
- 显示倒计时（手榴弹）
- C4/遥控爆炸物：显示 "RDX ARMED" 警告

### 弹药显示

| 弹药状态 | 显示方式 |
|---------|---------|
| 满弹药 | 白字显示数量 |
| 低弹药（≤30%） | 黄字 |
| 空弹药 | 红字 + "EMPTY" 闪烁 |

### 武器信息面板（I键打开）

- 列出所有持有的武器（热武器 + 环境物件）
- 显示每个武器的 `WeaponData`（伤害类型、有效距离等）
- 可拖拽排序
- 快捷键提示（Tab/Q/E/X）

### 爆炸物爆炸半径预览

玩家放置 C4 后：
- 地面上显示爆炸范围圆圈（红色半透明）
- 显示致死范围（内圈，深红色）
- 显示硬直范围（外圈，浅红色）

## Acceptance Criteria

**功能验收**：

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 环境物件被拾取后状态正确转为 `Held` | 拾取砖块，验证状态转换和 `WeaponStateChangedEvent` |
| AC-2 | 环境物件执行后正确发送 `DamageRequest` 到 Health 系统 | 使用砖块处决NPC，监控 Health 系统接收 |
| AC-3 | **爆炸物混合型伤害**：近距离（≤30%半径）造成 Lethal，远距离造成 Blunt 高硬直 | 在不同距离引爆C4，验证0.9m处=秒杀，1.5m处=硬直 |
| AC-4 | 投掷物在超出范围后命中概率不低于 min_hit_chance | 远距离投掷测试，统计命中率 |
| AC-5 | 热武器弹药耗尽后正确转入 `Empty` 状态 | 射击直到弹药耗尽，验证状态 |
| AC-6 | 热武器换弹时间符合 `ReloadTime` 参数 | 计时换弹过程 |
| AC-7 | NPC 能正确检测到热武器并触发 `WeaponAwareness` 事件 | 玩家拔出手枪，观察NPC反应 |
| AC-8 | Gritty Takedowns 能通过 `WeaponQueryRequest` 获取正确的 `WeaponData` | 执行环境处决，验证调用的武器数据 |
| AC-9 | 武器切换机制正常工作（Tab/Q/E/X） | 验证各按键切换逻辑 |
| AC-10 | 可破坏物件（灯泡、电线）有正确的 WeaponData | 拾取灯泡，验证有 Lethal 伤害和对应动画标签 |

**跨系统验收**：

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-11 | 武器系统发送的 `DamageRequest` 被 Health 系统正确处理 | 监控 Health 系统日志 |
| AC-12 | 爆炸物触发 `ExplosionEvent` 造成混合型范围伤害 | 引爆C4，验证范围内实体按距离受到不同伤害 |
| AC-13 | 武器使用事件正确触发音频系统播放音效 | 使用不同武器，验证音效播放 |
| AC-14 | `WeaponStateChangedEvent` 正确通知 UI 系统更新图标 | 拾取/丢弃武器，验证 HUD 图标更新 |
| AC-15 | 爆炸物击杀 NPC 后触发警报扩散 | 爆炸后观察范围内 NPC 进入 ALERT 状态 |

**性能预算**：

| 指标 | 预算 | 说明 |
|------|------|------|
| 武器状态查询响应时间 | < 1帧 | 切换武器应即时响应 |
| 爆炸伤害计算 | < 2ms | 空间分区预筛选优化后 |
| 武器系统内存占用 | < 2MB | WeaponTemplateLibrary 数据资产 |
| UI 图标更新延迟 | < 1帧 | WeaponStateChangedEvent 处理 |

## Open Questions

| # | 问题 | 状态 | 负责人 | 目标日期 |
|---|------|------|--------|---------|
| OQ-1 | **库存/重量系统的具体实现**：✅ **已解决**：MVP阶段简化为"热武器固定栏位"（Tab切换），暂不实现重量系统。理由：垂直切片阶段重点验证核心玩法，重量系统作为 Alpha 阶段可选扩展。 | 已解决 | 游戏设计师 | MVP 评审时 |
| OQ-2 | **C4 的引爆方式**：玩家放置C4后，是手动遥控引爆（按键盘），还是需要额外操作？是否需要"延时引爆"作为逃生手段？ | 待定 | 游戏设计师 | Vertical Slice 设计时 |
| OQ-3 | **投掷物物理模拟**：✅ **已解决**：MVP阶段使用直线弹道，Alpha阶段增加物理弹射模拟（弹跳衰减50%/75%）。 | 已解决 | — | — |
| OQ-4 | **NPC 使用热武器**：NPC 是否能捡起并使用热武器（射击玩家）？这会增加AI复杂性，但也是重要的威胁来源。 | 待定 | AI 程序员 | Alpha 阶段 |
| OQ-5 | **武器升级/解锁系统**：✅ **已解决**：不做全局时间线限制，由关卡设计师在关卡设计中决定热武器获取来源。 | 已解决 | — | — |
| OQ-6 | **C4 作为任务工具的关卡设计**：C4 必须用来炸开门/容器时，关卡设计师如何防止玩家滥用（例如炸死多个NPC触发警报）？ | 待定 | 关卡设计师 | Vertical Slice 设计时 |
