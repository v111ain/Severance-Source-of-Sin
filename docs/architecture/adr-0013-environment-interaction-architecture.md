# ADR-0013: 环境交互系统 (Environment Interaction) 架构决策

## Status
**Accepted**

## Date
2026-04-10

## Last Updated
2026-04-11 (v4: 确认 §13.5 为 InteractionState 统一定义位置；补充 ChainRadius 与 ScaleMultiplier 交互规则的物件类别枚举说明；明确 EventQueueCapacity 与性能目标的约束关系)

## Context

### Problem Statement

环境交互系统是"环境即武器"设计理念的核心承载系统。俯视角潜行游戏中，环境物件（灭火器、汽油桶、重物等）不仅是静态装饰，更是玩家的潜在武器和工具。系统需要解决：

1. **物件管理**：管理七类可交互物件（武器类/爆炸类/可破坏类/情报类/障碍类/机关类/投掷类）的状态和行为
2. **交互判定**：验证物件是否在交互范围内（距离+朝向），响应玩家交互输入
3. **状态驱动**：通过状态机驱动物件交互流程，为下游系统提供唯一数据源
4. **事件广播**：物件被触发时生成 EnvironmentalEvent，通知 NPC AI 系统

### Constraints

- **设计约束**："环境即武器"理念要求物件交互具有战术意义和戏剧性反馈
- **性能约束**：连锁爆炸限制最大3层，单帧处理10个环境事件不造成帧率下降
- **数据源约束**：环境交互系统是物件状态的唯一数据源（Single Source of Truth），下游系统不得自行维护状态副本
- **动画约束**：使用标签体系实现动画模块化复用，每个标签类别最多3种独特变体
- **平台约束**：支持 PC & PS5

### Requirements

- **必须**：定义七类物件的分类体系、交互半径、前向角度参数
- **必须**：定义物件状态机（Available/InUse/OnCooldown/Depleted）
- **必须**：定义与环境交互的射线/盒体检测接口
- **必须**：定义 EnvironmentalEvent 数据结构（事件类型/位置/半径/持续时间）
- **必须**：定义 ObjectStateChangedEvent 供武器系统订阅
- **必须**：支持远程触发机制（射击引爆爆炸物）
- **必须**：支持连锁爆炸（ChainRadius + ChainExplosionDelay）
- **必须**：遵循 ADR-0003 系统分层定义（Environment Interaction 属于 Core Layer）

---

## Decision

### 架构决策

采用**状态机驱动 + 事件广播 + 组件化物件模板**架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Environment Interaction System 架构                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                 EnvironmentInteractionSystem (Core Layer)             │   │
│  │  - 持有所有 InteractiveObject 的状态引用                                │   │
│  │  - 订阅玩家控制器的射线/盒体检测结果                                     │   │
│  │  - 订阅物件的交互输入事件 (InteractionKeyPressed)                       │   │
│  │  - 发布 EnvironmentalEvent 到事件总线                                  │   │
│  │  - 发布 ObjectStateChangedEvent 到事件总线                            │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                   InteractiveObject (Component)                       │   │
│  │  - ObjectCategory: 物件类别                                          │   │
│  │  - InteractionState: Available / InUse / OnCooldown / Depleted     │   │
│  │    （注意：此状态与 shared-types.md §3.5 的 ObjectState 不同，       │   │
│  │     现已统一定义为 InteractionState，见 shared-types.md §13.5）       │   │
│  │  - InteractionRadius: 交互半径 (0.5m - 2.5m)                          │   │
│  │  - FieldOfView: 前向角度 (45° - 180°)                                 │   │
│  │  - AnimationTags: 动画标签列表                                        │   │
│  │  - RemoteTriggerable: bool                                           │   │
│  │  - ChainTriggerable: bool                                            │   │
│  │  - ChainRadius: float                                                │   │
│  │  - CooldownDuration: float                                           │   │
│  │  - ScaleMultiplier: float (0.5 - 2.0)                                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ### 动画标签体系 (Animation Tags)                                       │
│                                                                          │
│  物件通过 AnimationTag 关联可复用的动画片段，减少必须制作的独立动画数量：     │
│                                                                          │
│  | 标签 | 描述 | 可复用物件示例 | 动画变体数 |
│  |------|------|---------------|------------|
│  | `WEAPON_MELEE_SM` | 小型近战武器拾取/挥舞 | 砖块、玻璃瓶、小刀 | 3 |
│  | `WEAPON_MELEE_LG` | 大型近战武器拾取/挥舞 | 钢管、铁棍、铁管 | 3 |
│  | `EXPLOSIVE_CAN` | 罐装爆炸物交互 | 灭火器、汽油桶、丙烷罐 | 3 |
│  | `EXPLOSIVE_TOSS` | 投掷类爆炸物 | 手榴弹、燃烧瓶 | 3 |
│  | `DESTRUCT_GLASS` | 玻璃制品破坏 | 灯泡、玻璃窗、酒瓶 | 2 |
│  | `DESTRUCT_WOOD` | 木制品破坏 | 木箱、木门、架子 | 2 |
│  | `THROW_SM` | 小型投掷物 | 硬币、瓶盖、小石子 | 2 |
│  | `OBSTACLE_PUSH` | 可推动障碍物 | 箱子、桶、椅子 | 2 |
│  | `MECH_SWITCH` | 开关类机关 | 电灯开关、电路闸门 | 2 |
│  | `INTEL_PICKUP` | 情报类物件拾取 | 手机、账本、文件 | 1 |（MVP 阶段情报类物件交互变化较少，单变体足够。如后续需要更多变体，可复用 DESTRUCT_GLASS 的玻璃破碎动画标签作为"情报物件破坏获取"的备选）
│                                                                          │
│  > **MVP 限制**：每个标签类别最多 3 种独特变体，超出限制必须复用现有标签。
>
> **INTEL_PICKUP 动画复用说明**：`INTEL_PICKUP`（情报类物件拾取）的动画变化较少，MVP 阶段单变体足够。如后续需要更多变体，可按以下优先级扩展：
> 1. 复用 `THROW_SM`（小型投掷物）的抬手/释放动画序列，调整物件轨迹
> 2. 复用 `DESTRUCT_GLASS`（玻璃制品破坏）的破碎粒子效果
> 3. 新增独立 `INTEL_PICKUP` 动画变体
>
> 当前 MVP 采用方案 1（复用 THROW_SM），避免跨类别复用带来的语义混淆。  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 物件状态机

> **状态命名说明**：本 ADR 中的 `Available / InUse / OnCooldown / Depleted` 状态已统一定义为 `InteractionState`（shared-types.md §13.5），以区别于 shared-types §3.5 中的 `ObjectState`（物件生命周期状态）。

```
[Available] ──玩家交互──▶ [InUse] ──动画完成──▶ [OnCooldown]
     ▲                              │                    │
     │                         [远程触发]                  │
     │                              │                    ▼
     │                              └─────────────────▶ [Depleted]
     │                                                        │
     └───────────────────────（重置/刷新）─────────────────────┘
```

### 事件流向

```
玩家控制器 ──射线检测结果──▶ EnvironmentInteractionSystem
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
            NPC AISystem         WeaponSystem          ClueSystem
        (EnvironmentalEvent) (ObjectStateChanged)   (任务更新)
```

---

## Formulas

### 核心公式摘要

**交互范围判定**：
```
IsInRange = (DistanceToObject ≤ InteractionRadius) AND IsFacingObject
IsFacingObject = AngleToObject ≤ FieldOfView / 2
```

**物件冷却时间**：
```
RemainingCooldown = Max(0, CooldownDuration - TimeSinceUsed)
IsAvailable = (CurrentState == Available) OR (RemainingCooldown == 0)
```

**环境事件生成**：
```
EventRadius = BaseRadius * Object.ScaleMultiplier * DistanceFalloff
DistanceFalloff = Clamp(1.0 - (DistanceToObject / EventRadius), 0.0, 1.0)
```

**ScaleMultiplier 与 ChainRadius 的交互规则**：
> 当物件具有 `ChainTriggerable = true` 时，ChainRadius 按照以下规则与 ScaleMultiplier 交互：
> - **爆炸类物件**（ObjectCategory = FireExtinguisher / GasolineCan / PropaneTank / Grenade / C4）：`EffectiveChainRadius = ChainRadius * ScaleMultiplier`
>   - 原因：大型爆炸物（ScaleMultiplier > 1.0）应触发更远的连锁反应，符合物理直觉
>   - 小型爆炸物（ScaleMultiplier < 1.0）连锁半径按比例缩小
> - **非爆炸类物件**（ObjectCategory = Brick / MetalPipe / GlassBulb 等）：`ChainRadius` 不受 ScaleMultiplier 影响，保持固定值
>   - 原因：推动障碍物等物件的连锁效果与尺寸无关
>
> **爆炸类物件类别枚举**（定义于 shared-types.md §3.2 ObjectCategory）：
> ```csharp
> public enum ObjectCategory
> {
>     // 爆炸类物件
>     FireExtinguisher,  // 灭火器
>     GasolineCan,       // 汽油桶
>     PropaneTank,       // 丙烷罐
>     Grenade,           // 手榴弹
>     C4,                // C4 炸弹
>     // ... 其他类别
> }
> ```
>
> **示例**：丙烷罐（BaseChainRadius = 3.0m, ScaleMultiplier = 1.5）爆炸时，EffectiveChainRadius = 4.5m，可引爆 4.5m 内的其他可连锁物件。

---

## Edge Cases

| # | 场景 | 处理方式 |
|---|------|---------|
| EC-1 | 范围内多个物件优先级 | 优先选择距离最近的，玩家可通过移动摇杆切换 |
| EC-2 | 物件交互时 NPC 正看着该物件 | 物件仍可交互，NPC 立即发现玩家动作（Alert 状态） |
| EC-3 | 动作锁定状态中尝试交互 | 输入被忽略，必须等待动画完成 |
| EC-4 | 冷却中物件的远程触发 | OnCooldown → 触发无效但消耗弹药；Depleted → 无响应 |
| EC-5 | 连锁爆炸 | ChainTriggerable + ChainRadius，每级 0.5s 延迟，限 MaxChainDepth 层。<br><br>**EventQueue 行为与合并规则**：<br>- **队列容量**：EventQueue 最大容量为 10 个事件（EventQueueCapacity = 10）<br>- **入队时机**：新事件到达时直接入队，队列未满则正常添加<br>- **合并触发时机**：当新事件到达时队列已满（已有 10 个事件），该新事件不直接入队，而是触发合并流程<br>- **合并规则**：<br>  1. 同类型事件（均为 EXPLOSION）→ 合并为单个事件，intensity 叠加，duration 取最大值<br>  2. 不同类型事件 → 合并为 `EnvironmentalEvent{type=CHAOS, intensity=sum_of_intensities, duration=max_duration}`，NPC AI 系统收到 CHAOS 事件时额外增加 3 秒混乱持续时间<br><br>合并后的事件视为已处理完毕，下一帧可继续正常入队。 |
| EC-6 | 投掷物飞行中玩家被攻击 | 投掷物保持轨迹落地，玩家按 Health 规则处理 |
| EC-7 | 关卡重置时物件状态 | 全部恢复 Available。<br><br>**三状态超时刷新规则**：<br>- **Available**：无超时机制，持续保持可用<br>- **OnCooldown**：冷却计时器独立运行，当 `TimeSinceUsed >= CooldownDuration` 时自动刷新为 Available（由 EC-6 的冷却公式 `RemainingCooldown = Max(0, CooldownDuration - TimeSinceUsed)` 驱动）<br>- **Depleted**：独立于 OnCooldown 机制，采用 30 秒连续计时刷新为 Available<br><br>**计时器行为**：Depleted 计时为**连续计时**（玩家离开区域后计时不暂停），防止玩家反复进出刷新的 exploits。OnCooldown 计时同样为连续计时，不受玩家位置影响。<br><br>**计时器重置**：若需要暂停计时（如玩家死亡后重新读取检查点），CheckpointSystem 发送 `CheckpointRestoreRequestEvent` 时附带 `ResetCooldownTimers = true`，EnvironmentInteractionSystem 收到后重置所有 OnCooldown 和 Depleted 计时器。<br><br>**与 World Map 系统集成**：当 `AreaClearedEvent` 触发时（NPC AI 系统检测到地区内敌人全灭），EnvironmentInteractionSystem 重置所有物件为 Available 状态。若需要立即重置（如玩家撤离后重新进入），由 CheckpointSystem 发送 `CheckpointRestoreRequestEvent` 触发地区初始化流程，EnvironmentInteractionSystem 订阅此事件完成物件重置。详见 ADR-0012。 |

---

## Tuning Knobs

| 类别 | 参数 | 默认值 | 安全范围 |
|------|------|--------|---------|
| 交互半径 | `InteractionRadius_Weapon` | 1.0m | 0.5m - 2.0m |
| 交互半径 | `InteractionRadius_Explosive` | 1.75m | 1.0m - 3.0m |
| 交互半径 | `InteractionRadius_Throwable` | 2.0m | 1.0m - 3.5m |
| 冷却时间 | `Cooldown_Throwable` | 5.0s | 2.0s - 15.0s |
| 冷却时间 | `Cooldown_Obstacle` | 10.0s | 5.0s - 30.0s |
| 连锁参数 | `ChainExplosionDelay` | 0.5s | 0.2s - 1.0s |
| 连锁参数 | `MaxChainDepth` | 3 | 1 - 5 |
| 事件队列 | `EventQueueCapacity` | 10 | 3 - 10 |
| 效果半径 | `SoundRadius_Explosion` | 8.0m | 5.0m - 25.0m |

> **注**：`EventQueueCapacity = 10` 是事件队列的最大容量约束，与性能目标"单帧处理 10 个环境事件"是同一个约束的两方面描述：
> - **容量视角**：EventQueue 最多同时容纳 10 个待处理事件
> - **性能视角**：系统在单帧内可处理的上限为 10 个事件
>
> 当队列满（已有 10 个事件）时，新到达的事件不直接入队，而是触发合并流程，合并为一个"混乱"事件（CHAOS 类型），确保事件处理不会无限积压。

---

## Alternatives Considered

### Alternative 1: 集中式物件管理器

- **描述**：所有物件状态存储在全局 ObjectManager，单一系统处理所有交互逻辑
- **优点**：数据访问简单，调试方便
- **缺点**：违反单一职责原则，系统臃肿；难以扩展新物件类型
- **拒绝理由**：不符合 ADR-0003 的系统分层原则，Core Layer 系统应保持精简

### Alternative 2: 物件自治（每个物件独立系统）

- **描述**：每个物件拥有独立的状态机和行为逻辑
- **优点**：高度解耦，新增物件不影响现有系统
- **缺点**：协调整体行为困难（连锁爆炸、同时触发），事件管理复杂
- **拒绝理由**：对于俯视角潜行游戏，物件间的协同效果（连锁爆炸、诱饵组合）是核心体验，需要统一管理

---

## Consequences

### Positive

- 状态机驱动使物件行为可预测、可调试
- 事件广播机制解耦了 NPC AI、武器系统等下游消费者
- 唯一数据源设计避免状态不一致问题
- 标签体系动画复用显著减少动画制作工作量

### Negative

- 所有物件状态集中在系统内，可能造成内存压力（需要优化数据结构）
- 事件队列机制增加了系统复杂度

### Risks

- **风险**：连锁爆炸递归调用可能导致性能问题
  - **缓解**：限制最大连锁层数（MaxChainDepth = 3），每级增加0.5s延迟
- **风险**：物件冷却刷新逻辑复杂
  - **缓解**：统一由 EnvironmentInteractionSystem 管理，避免分散逻辑

---

## Performance Implications

- **CPU**：单帧处理10个环境事件不造成帧率下降（EventQueueCapacity = 10）
- **Memory**：物件状态使用结构体存储，按需实例化
- **Load Time**：物件模板库（ObjectTemplateLibrary）预加载，运行时零开销

---

## Migration Plan

本决策不涉及对现有代码的迁移。实现从 ADR-0013 状态为 Proposed 时开始，先实现物件状态机和基础交互，再扩展远程触发和连锁爆炸功能。

---

## Validation Criteria

| ID | 验收条件 |
|----|---------|
| AC-1 | 玩家站在物件交互范围内时，物件显示高亮提示 |
| AC-2 | 动画期间玩家无法移动 |
| AC-3 | 冷却期间物件不可再交互 |
| AC-4 | 爆炸类物件放置后引爆成功 |
| AC-5 | 射击可远程触发的物件时触发效果 |
| AC-6 | 连锁爆炸正确触发，3层后自动停止 |
| AC-7 | 环境物件触发时 NPC AI 接收到 EnvironmentalEvent |

---

## Related Decisions

- ADR-0003: 系统分层架构（Core Layer 定义）
- ADR-0004: NPC AI 行为架构（EnvironmentalEvent 消费）
- ADR-0010: 武器系统架构（ObjectStateChangedEvent 订阅）
