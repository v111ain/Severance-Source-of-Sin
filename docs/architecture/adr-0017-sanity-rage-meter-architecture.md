# ADR-0017: 理智/愤怒系统 (Sanity/Rage Meter) 架构决策

## Status
**Proposed** (v5: 更新视觉效果请求接口，改用 ADR-0023 的统一 ScreenEffectRequestEvent)

## Date
2026-04-10

## Last Updated
2026-04-12 (v5: 更新视觉效果请求接口，改用 ADR-0023 的统一 ScreenEffectRequestEvent；shared-types.md §12.3 中的独立事件已标记为废弃)

## Context

### Problem Statement

理智/愤怒系统是《断绝：罪恶之源》"罪恶的深度"支柱的 Meta 层实现。它通过视听反馈将玩家的道德选择转化为持续的情绪压力——杀戮累积愤怒，发现悲剧消耗理智。系统不改变玩家数值能力，而是放大"感受"层面的沉浸感。关键挑战在于：

1. **双轨并行**：理智和愤怒是两个独立计量条，但视觉反馈需要协调
2. **状态优先级**：FRENZIED 和 BROKEN 可能同时触发，需要明确优先级
3. **衰减与锁定**：愤怒需要自然消散，但狂暴状态需要"锁定"以维持体验完整性
4. **事件驱动的副作用**：系统是纯消费者，不主动轮询，响应外部事件

### Constraints

- **视觉约束**：FRENZIED 视觉效果（准星抖动、移动加速、血红暗角）优先级高于 BROKEN
- **性能约束**：视觉效果插值必须无跳变，帧率影响 < 1ms
- **兼容性约束**：理智降至 0 不触发死亡，只触发极端视觉效果（区别于 Health 系统）
  > **Sanity = 0 极端效果定义**：
  > - 画面完全倒置（180° 旋转）或严重扭曲
  > - 持续高强度噪点（NoiseIntensity = 1.0）
  > - 极端暗角（VignetteIntensity = 1.0，视野几乎全黑）
  > - 色彩完全褪去（SaturationMultiplier = 0.0）
  > - **无准星抖动**（因为 FRENZIED 不激活，ShakeIntensity = 0）
  > - 玩家仍可移动和行动，但视觉几乎完全依赖记忆
  >
  > **Sanity = 0 与 Health 系统并行兼容性**：
  > 当 Health=满 且 Sanity=0 时，两个系统同时生效但互不干扰：
  > - ScreenEffects 的极端效果（倒置/噪点/暗角/褪色）完全覆盖游戏画面
  > - HUD 元素（如体力槽、理智槽、愤怒槽）通过**半透明叠加层**显示，确保玩家仍能获取关键游戏信息
  > - 半透明叠加层的透明度通过 `HUDOverlayOpacityRequest` 事件由 UI 系统控制（默认 0.3 opacity），既保证信息可读性，又不与极端视觉效果产生视觉冲突
  > - Health 系统不受 Sanity = 0 影响；后续受到伤害时正常触发受伤动画和 Health 变化
- **衰减约束**：愤怒消散期间如果玩家再次击杀，必须立即停止消散并从当前值累加

### Requirements

- **必须**：定义双轨计量系统（Sanity 0-100，Rage 0-100）
- **必须**：定义 6 档心理状态（CALM / UNEASY / AGITATED / BROKEN / FRENZIED / SOUL_SPLIT）
- **必须**：定义状态阈值锁定机制（Frenzy Threshold Lock）
- **必须**：定义与 ScreenEffectsManager 的视觉效果请求接口（通过 ADR-0023 定义的 ScreenEffectRequestEvent）
- **必须**：定义与 UI 系统的 HUDOverlayOpacityRequest 接口
- **必须**：定义与 Clue&Journal 的事件订阅接口（ClueDiscoveredEvent）
- **必须**：定义愤怒消散互斥机制（消散期间击杀立即停止消散）

---

## Decision

### 架构决策

采用**事件驱动的双轨状态机架构**，理智/愤怒系统作为纯粹的消费者订阅来自 GrittyTakedowns、Clue&Journal、Health&Lethality 的事件，转化为内部状态和视觉效果请求后推送给下游的 ScreenEffects 和 DynamicPostProcessing。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  Sanity/Rage Meter System 架构                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【事件输入】                                                                │
│                                                                              │
│  ┌──────────────────┐    KillTagEvent{kill_tag}                             │
│  │ GrittyTakedowns  │ ──────────────────────────────────────────────────┐  │
│  └──────────────────┘                                                      │  │
│                                                                              │  │
│  ┌──────────────────┐    ClueDiscoveredEvent{                                │  │
│  │   Clue&Journal   │      clue_category, discovery_stage,                │  │
│  └──────────────────┘      narrative_significance}                         │  │
│                                                                            │  │
│  ┌──────────────────┐    PlayerDamaged{damage_type, is_lethal}            │  │
│  │ Health&Lethality │ ──────────────────────────────────────────────────┘  │
│  └──────────────────┘                                                        │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    SanityRageMeter (MonoBehaviour Singleton)          │   │
│  │                                                                       │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │   │
│  │  │  SanityTracker  │  │   RageTracker   │  │  StateMachine       │  │   │
│  │  │  (理智跟踪器)     │  │   (愤怒跟踪器)    │  │  (心理状态机)        │  │   │
│  │  │  ◆ 0-100 范围   │  │  ◆ 0-100 范围    │  │  ◆ 6 档状态         │  │   │
│  │  │  ◆ 恢复逻辑     │  │  ◆ 消散逻辑      │  │  ◆ 优先级判定       │  │   │
│  │  │  ◆ 衰减曲线     │  │  ◆ 锁定机制      │  │                     │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘  │   │
│  │                                                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │               VisualEffectCalculator                             │ │   │
│  │  │               (视觉效果计算器)                                    │ │   │
│  │  │  ◆ VignetteIntensity = Lerp(0.0, 0.8, 1.0 - Sanity/100)        │ │   │
│  │  │  ◆ NoiseIntensity = Lerp(0.0, 0.5, 1.0 - Sanity/50)           │ │   │
│  │  │  ◆ SaturationMultiplier = Lerp(0.3, 1.0, Sanity/100)          │ │   │
│  │  │  ◆ ShakeIntensity = Lerp(0.0, 8.0, Rage/100)                  │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│  【效果输出】                      ▼                                          │
│                           ┌────────────────────┐                             │
│                           │   效果请求事件      │                             │
│                           └────────────────────┘                             │
│                                    │                                          │
│          ┌─────────────────────────┼─────────────────────────┐              │
│          ▼                         ▼                         ▼              │
│  ┌───────────────┐         ┌───────────────┐         ┌───────────────┐      │
│  │ ScreenEffects │         │  Player       │         │ DynamicPost   │      │
│  │               │         │  Controller   │         │ Processing    │      │
│  │ VignetteReq   │         │               │         │               │      │
│  │ NoiseReq      │         │ MovementSpeed │         │ PsychState    │      │
│  │ SaturationReq │         │ Multiplier    │         │               │      │
│  │ ShakeReq      │         │               │         │               │      │
│  └───────────────┘         └───────────────┘         └───────────────┘      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 核心数据结构

#### 心理状态枚举

> **类型定义说明**：`PsychologicalState` 枚举已统一定义于 shared-types.md §12.1，实现时应从 shared-types 引用。

```csharp
// 引用位置：Assets/Game/Features/SanityRage/Types/PsychologicalState.cs
// 引用方式：using PsychState = SharedTypes.PsychologicalState;

public enum PsychologicalState
{
    CALM,       // Sanity >= 70 且 Rage <= 30
    UNEASY,     // (Sanity 40-69 OR Rage 31-50) AND NOT AGITATED/FRENZIED/BROKEN/SOUL_SPLIT
    AGITATED,   // (Sanity 20-39 OR Rage 51-70) AND NOT FRENZIED/BROKEN/SOUL_SPLIT
    BROKEN,     // Sanity < 20 AND Rage <= 70
    FRENZIED,   // Rage >= 71 AND Sanity < 70  ← 注：Rage = 71 时触发（闭区间）
    SOUL_SPLIT  // Sanity < 20 AND Rage > 70   ← 注：Rage > 70 时触发（开区间）
}
```

> **类型定义说明**：`PsychologicalStateEvent` 事件已统一定义于 shared-types.md §12.2，实现时应从 shared-types 引用。

```csharp
// 引用位置：Assets/Game/Features/SanityRage/Events/SanityRageEvents.cs
// 引用方式：从 shared-types 引用，而非在本系统重复定义

public struct PsychologicalStateEvent
{
    public PsychologicalState State;
    public string Reason;  // 状态变更原因（用于调试）
}
```

#### 状态优先级规则

```
优先级 1 (最高): SOUL_SPLIT — Sanity < 20 AND Rage > 70，FRENZIED + BROKEN 效果叠加
优先级 2: FRENZIED — Rage >= 71 AND Sanity < 70，屏幕被血红色覆盖
优先级 3: BROKEN — Sanity < 20 AND Rage <= 70，视野严重缩窄、严重噪点
优先级 4: AGITATED — (Rage > 70 OR Sanity < 40) AND NOT FRENZIED/BROKEN/SOUL_SPLIT
优先级 5: UNEASY — (Rage > 30 OR Sanity < 70) AND NOT AGITATED/FRENZIED/BROKEN/SOUL_SPLIT
优先级 6 (默认): CALM
```

> **阈值说明**：使用闭区间 `>=` 和开区间 `<` 避免边界歧义。
> - `Rage = 71` 时 → FRENZIED（因为 71 >= 71）
> - `Sanity = 19, Rage = 71` 时 → SOUL_SPLIT（因为 19 < 20 AND 71 > 70）
> - `Sanity = 19, Rage = 70` 时 → BROKEN（因为 19 < 20 AND 70 <= 70）

**视觉效果叠加规则**：
- 暗角（Vignette）：取所有适用状态的最大值
- 饱和度（Saturation）：取所有适用状态的最小值（越低越灰暗）
- 噪点（Noise）：BROKEN/SOUL_SPLIT 状态特有，FRENZIED 不产生噪点
- 准星抖动（Shake）：FRENZIED/SOUL_SPLIT 状态特有，BROKEN 不产生抖动
- 移动速度：FRENZIED/SOUL_SPLIT 状态下 +10%，BROKEN 状态下 -5%

**SOUL_SPLIT 特殊叠加说明**：
SOUL_SPLIT 是极端双轨状态，**同时叠加 FRENZIED 和 BROKEN 的所有效果**：
- VignetteIntensity = FRENZIED 规则计算（Sanity < 70 时增加）
- NoiseIntensity = BROKEN 规则计算（Sanity < 50 时开始）
- SaturationMultiplier = BROKEN 规则计算（越低越灰暗）
- ShakeIntensity = FRENZIED 规则计算（Rage 高时增加）
- MovementSpeedBonus = +10%（取 FRENZIED 的正向加成，不执行 BROKEN 的 -5% 惩罚）

> **SOUL_SPLIT 速度加成设计意图澄清**：
> 这个 +10% 速度提升是**有意为之的矛盾体验**，而非设计失误：
> 1. **肾上腺素机制**：极端情绪下人体会释放大量肾上腺素，暂时忽略生理极限。"跑得更快但完全看不见"正是这种机制的体现——玩家的肉体被强迫超越极限，但精神已经完全崩溃。
> 2. **受苦美学**：《泰坦 Souls》等游戏"受苦美学"的体现——玩家在最低谷时仍有一线生机（跑得快），但代价是视野完全崩溃（Noise + Vignette 双重 debuff）。
> 3. **战术权衡**：速度提升 + 视野崩溃 = 玩家可以快速逃离危险区域，但无法精确导航。这创造了"本能逃跑 vs 精准定位"的张力。
> 4. **玩家体验循环**：在 SOUL_SPLIT 状态下，玩家可能：a) 乱冲乱撞地逃跑（速度优势），或 b) 停下来试图恢复理智（主动降 Rage/Sanity）。这种两难选择增加了情感深度。
>
> **设计意图确认**：如果 playtest 反馈表明这个机制体验不佳，可通过 Tuning Knob `SOUL_SPLIT_SpeedBonus` 调整（安全范围 0%~+15%），或完全禁用（设为 0%）。

**MovementSpeedBonus 计算公式**：
```
if (State == PsychologicalState.SOUL_SPLIT || State == PsychologicalState.FRENZIED)
    MovementSpeedBonus = +10%
else if (State == PsychologicalState.BROKEN)
    MovementSpeedBonus = -5%
else
    MovementSpeedBonus = 0%
```
> **优先级说明**：SOUL_SPLIT 和 FRENZIED 均使用 +10%，不执行 BROKEN 的 -5% 惩罚。三个状态互斥（按优先级判定只会是其中之一）。

### 愤怒消散与锁定机制

**消散判定（基于真实时间累积）**：
```csharp
// 使用 Time.unscaledDeltaTime 确保暂停时不受影响
private float _rageDecayAccumulator = 0f;
private const float RAGE_DECAY_INTERVAL = 3f;      // 消散检查间隔（秒）
private const float RAGE_DECAY_AMOUNT = 2f;         // 每次消散减少量
// 实际衰减率 = RAGE_DECAY_AMOUNT / RAGE_DECAY_INTERVAL ≈ 0.67 点/秒

void Update()
{
    if (IsInCombat == false)
    {
        _rageDecayAccumulator += Time.unscaledDeltaTime;

        // 每 RAGE_DECAY_INTERVAL 秒执行一次消散
        while (_rageDecayAccumulator >= RAGE_DECAY_INTERVAL)
        {
            Rage = Mathf.Max(0, Rage - RAGE_DECAY_AMOUNT);
            _rageDecayAccumulator -= RAGE_DECAY_INTERVAL;
        }
    }
    else
    {
        // 战斗状态下，清除累积器（退出战斗后消散立即开始，而非延续战斗前的累积时间）
        // 这是**预期行为**：战斗期间玩家无暇顾及愤怒消散，战斗结束后情绪应当"重新"开始积累消散间隔
        _rageDecayAccumulator = 0f;
    }
}
```

> **为什么不直接用 Timer**：直接用固定 timer 会产生时间累积误差。使用 `_rageDecayAccumulator` 累积真实时间差，确保无论帧率高低，**平均**消散速率始终为 `RAGE_DECAY_AMOUNT / RAGE_DECAY_INTERVAL` (约 0.67/秒)。

### IsInCombat 判定逻辑

> **权威来源说明**：IsInCombat 的判定逻辑在 NPCManager 中实现（见 ADR-0004 §7），以下仅作为参考说明。

**IsInCombat 判定条件**：
- **进入战斗**：任意 NPC 的 AlertState 进入 `ALERT` 或以上状态（发现玩家），或玩家受到任意伤害
- **退出战斗**：所有 NPC 的 AlertState 回到 `UNDETECTED` 且 10 秒内无伤害事件
- **设计意图**：战斗状态表示玩家处于潜在威胁中，愤怒应该维持高位而非消散

> **接口调用说明**：实际实现时应调用 `NPCManager.Instance.IsAnyNPCInCombat()` 接口，而非重复实现判定规则。本文档仅作为参考说明，方便阅读而非作为实现依据。

**Frenzy Threshold Lock（狂暴阈值锁定）**：

| 常量 | 值 | 说明 |
|------|-----|------|
| `FRENZY_THRESHOLD` | 70 | 进入 FRENZIED 状态的愤怒阈值 |
| `FRENZY_LOCK_DURATION` | 10 秒 | 狂暴阈值锁定持续时间 |
| `RAGE_DECAY_INTERVAL` | 3 秒 | 愤怒消散检查间隔 |
| `RAGE_DECAY_AMOUNT` | 2 点 | 每次消散减少量 |
| `COMBAT_EXIT_DELAY` | 10 秒 | 战斗状态退出延迟 |
| `SOUL_SPLIT_SPEED_BONUS` | 10% | SOUL_SPLIT 状态移动速度加成（安全范围 0%~+15%） |

- **锁定触发**：当 Rage **首次**超过 70 进入 FRENZIED 状态时，触发阈值锁定，同时记录锁定开始时间
- **重入规则**：锁定期间内，如果玩家再次触发 FRENZIED（通过消散后再积累或直接击杀），**不重置**锁定计时器，锁定时间保持首次触发时的计时
  > 这样设计是为了避免玩家通过"刷狂暴"来无限延长锁定，破坏体验节奏
- **锁定效果**：锁定持续期间，Rage 消散时**不会低于 70**（即 `Rage = Max(Rage, 70)` 在消散计算后生效）
- **锁定清除**：锁定到期后立即清除，Rage 恢复正常消散（可低于 70）
- **设计意图**：玩家一旦进入"狂暴"状态，应该能够**维持这种状态一段时间**，但不应通过反复触发来无限续杯

**消散期间击杀行为**：
- 立即停止消散
- 从**当前值**开始累加新击杀的愤怒值（不抵消之前消散的愤怒）

**连续战斗场景补充说明**：
> 如果玩家在消散期间（`_rageDecayAccumulator > 0`）再次进入战斗，行为如下：
> 1. `_rageDecayAccumulator` 被清除为 0（按照 §消散判定逻辑）
> 2. 战斗状态持续期间不执行消散
> 3. 战斗结束后，消散从 0 开始重新累积（而非从战斗前的累积时间继续）
>
> 这是**预期行为**：战斗状态表示玩家无暇顾及愤怒消散，每次战斗都是独立的情绪高峰。

### 公式定义

#### 理智值变化计算

```
SanityDelta = BaseValue[discovery_stage] * NarrativeSignificanceMultiplier
```

| 事件类型 | BaseValue | 说明 |
|---------|-----------|------|
| 击杀恶徒 (ENEMY) | -5 | 道德上可接受，但仍造成心理负担 |
| 击杀帮凶 (ACCOMPLICE) | -8 | 知道他们是受害者但仍下手 |
| 击杀无辜者 (VICTIM) | -15 | 严重的道德错误 |
| 目睹 NPC 死亡 | -5 | 非亲手，但仍受影响 |
| 发现悲剧线索（第一次） | -5 | 首次接触悲剧，情感冲击 |
| 发现悲剧线索（后续） | -3 | 递减惩罚，避免过度惩罚探索 |
| 发现悲剧线索（后续深度揭示） | -2 | 多次接触同一悲剧的边际递减 |
| 发现身份/位置线索 | +5 | 真相带来满足感 |
| 捆绑 NPC（不杀）(TieUp) | +3 | 仁慈选择 |
| 成功转化线人 | +5 | 正向干预 |

| NarrativeSignificanceMultiplier | 值 |
|-------------------------------|-----|
| NORMAL | ×1.0 |
| MAIN_TARGET | ×1.5 |
| NPC_SYMPATHY | ×2.0 |

#### 愤怒值变化计算

```
RageDelta = BaseValue[EventType] * MomentumMultiplier
```

| 事件类型 | BaseValue |
|---------|-----------|
| 击杀任意 NPC | +10 |
| 威胁 NPC | +3 |
| 玩家受伤 | +5 |
| 玩家重伤 | +10 |
| 捆绑 NPC (TieUp) | -2 |
| 时间流逝（每 5 秒） | -1 |

| MomentumMultiplier | 值 | 冲突处理 |
|-------------------|-----|----------|
| 正常情况 | ×1.0 | 无冲突时的默认值 |
| 快速连续击杀（<3 秒间隔） | ×1.3 | 优先级高于 10 秒内无击杀 |
| 10 秒内无击杀 | ×0.8 | 仅在非快速连续击杀时生效 |

> **冲突处理原则**：快速连续击杀触发时，忽略"10 秒内无击杀"的减成。因为连续击杀表示玩家处于狂暴状态中，不应受到惩罚性衰减。

**Rage 值范围约束**：
- Rage 值被约束在 [0, 100] 范围内
- `Rage = Mathf.Clamp(Rage + RageDelta, 0, 100)`，先计算增量后再 Clamp 总值
- 这确保 Fury Lock 锁定的是"值"而非"增量"后的中间状态

**单次 Rage 变化上限**：
- 为避免 Rage 瞬时大幅跳跃破坏体验，单次 Rage 变化量（正或负）不超过 **15 点**
- 公式：`RageDelta = Clamp(BaseValue * MomentumMultiplier, -15, 15)`
- 触发上限时，视觉效果可额外叠加"愤怒冲击"短暂闪烁提示玩家

> **潜在影响说明**：实施 Clamp(-15, 15) 可能导致在极端情况下（快速连续击杀 VICTIM，BaseValue=10 × MomentumMultiplier=1.3 = 13，实际为 13 < 15 不触发 Clamp；但若 BaseValue=-20 则 Clamp 后只能减少 15） Rage 增长速率低于预期，**可能导致狂暴（FRENZIED）触发延迟**。这是可接受的体验权衡——防止瞬时大幅跳跃比快速触发狂暴更重要。

#### 视觉效果插值公式

```
VignetteIntensity = Lerp(0.0, 0.8, 1.0 - Sanity/100)
NoiseIntensity = Sanity < 50 ? Lerp(0.0, 0.5, 1.0 - Sanity/50) : 0.0  // 仅当 Sanity < 50 时生效
SaturationMultiplier = Lerp(0.3, 1.0, Sanity/100)
ShakeIntensity = Lerp(0.0, 8.0, Rage/100)
MovementSpeedBonus = Lerp(0.0, 0.1, Rage/100)  // 最高 +10%
```

### 关键接口定义

#### 事件订阅（Inputs）

| 事件 | 来源 | 处理逻辑 |
|------|------|----------|
| `KillTagEvent{kill_tag: NPCIdentityType}` | GrittyTakedowns | 根据击杀目标身份计算理智/愤怒变化 |
| `ClueDiscoveredEvent{clue_id, clue_category, discovery_stage, narrative_significance}` | Clue&Journal | 根据线索类别和发现阶段计算理智变化。**事件定义见 shared-types.md §11.4** |
| `PlayerDamagedEvent{damage_type, is_lethal}` | Health&Lethality | 根据受伤程度计算理智/愤怒变化 |
| `CombatStateChangedEvent{is_in_combat}` | NPC AI 系统 | 订阅此事件判断是否退出战斗状态 |

> **类型定义说明**：`ClueDiscoveredEvent` 及其相关的 `ClueCategory`（§11.1）、`DiscoveryStage`（§11.2）、`NarrativeSignificance`（§11.3）已统一定义于 shared-types.md，实现时应从 shared-types 引用。

#### CombatStateChangedEvent 接口定义

> **权威来源说明**：`CombatStateChangedEvent` 的权威定义位于 ADR-0004（NPC AI 行为架构）§7 事件订阅与发布关系部分。ADR-0017 仅作为消费者引用此事件，不重复定义。

```csharp
// 引用位置：ADR-0004 §7（权威定义）
// 消费者：SanityRageMeter (ADR-0017)

public struct CombatStateChangedEvent
{
    /// <summary>
    /// 当前是否处于战斗状态
    /// true = 至少有一个 NPC 处于 ALERT/ESCAPE/COMBAT 状态
    /// false = 所有 NPC 都处于 UNDETECTED/SUSPECT/SEARCH 状态
    /// </summary>
    public bool IsInCombat;

    /// <summary>
    /// 触发战斗状态变更的原因（可选，用于调试）
    /// </summary>
    public string Reason;
}
```

**IsInCombat 判定逻辑说明**：
> IsInCombat 的实际判定逻辑在 NPCManager 中实现（ADR-0004），Sanity/Rage 系统仅通过事件订阅接收结果，不重复实现判定逻辑。实际实现时应调用：
> ```csharp
> bool isInCombat = NPCManager.Instance.IsAnyNPCInCombat();
> ```
> 而非在 Sanity/Rage 系统中内联判定规则。

#### 事件发布（Outputs）

| 事件 | 目标 | 数据内容 | 接口类型 |
|------|------|----------|----------|
| `ScreenEffectRequestEvent` | ScreenEffectsManager | 屏幕后处理效果请求（暗角/噪点/饱和度/抖动/模糊）。**事件定义见 ADR-0023** | 事件发布 |
| `HUDOverlayOpacityRequest{opacity}` | UI System | Sanity=0 时 HUD 半透明叠加层透明度。**事件定义见 shared-types.md §12.3.1** | 事件发布 |
| `MovementSpeedMultiplier{multiplier}` | PlayerController | 愤怒高时轻微增加移动速度 | 事件发布 |
| `PsychologicalStateEvent{state}` | DynamicPostProcessing | 通知后处理系统当前心理状态。**事件定义见 shared-types.md §12.2** | 事件发布 |

> **接口类型说明**：
> - 屏幕后处理效果（暗角/噪点/饱和度/抖动/模糊）通过统一的 `ScreenEffectRequestEvent` 发送，由 ScreenEffectsManager（ADR-0023）集中处理
> - HUD 透明度请求保持独立（`HUDOverlayOpacityRequest`），因为它是 UI 层特效，不通过 ScreenEffectsManager
> - 其他事件直接发布到 EventBus

---

## Alternatives Considered

### Alternative 1: 单一计量条（理智 - 愤怒合并）

- **描述**：用一个 -100 到 +100 的计量条表示心理状态（负值 = 理智，正值 = 愤怒）
- **Pros**：UI 显示更简洁；状态判定更简单
- **Cons**：无法表达"双轨并行"的复杂性；CALM 状态需要同时满足两个条件；视觉效果无法独立控制
- **Rejection Reason**：设计文档明确要求"双轨并行"的独立计量条，且两者可以同时生效

### Alternative 2: 状态驱动（非插值）

- **描述**：视觉效果直接切换为预设档位（0%/25%/50%/75%/100%），而非平滑插值
- **Pros**：实现简单；性能更好；状态明确
- **Cons**：视觉跳变明显，破坏沉浸感；不符合"渐进式心理压力"的体验设计
- **Rejection Reason**：Lerp 插值实现成本低（CPU < 0.1ms），视觉效果显著更平滑

### Alternative 3: 无阈值锁定（进入即消散）

- **描述**：狂暴状态进入后立即开始消散，玩家难以维持狂暴体验
- **Pros**：简化实现；更"真实"的愤怒衰减
- **Cons**：玩家刚进入狂暴就消散，体验碎片化；削弱"狂战士"状态的情感冲击力
- **Rejection Reason**：设计文档明确要求"玩家一旦进入狂暴状态，应该能够维持这种状态一段时间"

---

## Consequences

### Positive

- **清晰的职责分离**：系统只负责"响应事件 → 更新状态 → 计算效果"，不涉及具体游戏逻辑
- **可组合的视觉效果**：各效果（暗角/噪点/饱和度/抖动）独立计算，下游 ScreenEffects 负责合成
- **可测试的公式**：所有计算公式都是纯函数，输入输出明确
- **狂暴锁定增强沉浸感**：玩家能真正体验"狂战士"状态的持续紧张感

### Negative

- **调参复杂**：6 档状态 × 4 种视觉效果 = 24 个阈值参数，需要大量 playtest 迭代
- **阈值重叠时的优先级规则**：需要明确文档记录，否则容易产生歧义
- **理智为 0 不死**：可能让部分期待"严重后果"的玩家失望（但这是设计意图）

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **R-1** | 惩罚曲线过重导致玩家"情感崩溃" | Playtest 验证，根据反馈调整 BaseValue |
| **R-2** | 狂暴阈值锁定持续时间不合理 | 提供 FrenzyLockDuration Tuning Knob 供策划调整 |
| **R-3** | 愤怒消散与战斗状态判断冲突 | 明确 IsInCombat 的定义（被 NPC 发现即进入战斗） |
| **R-4** | 状态边界条件歧义导致预期外状态 | 使用闭区间/开区间明确定义，消除 Rage = 71 等边界情况歧义 |
| **R-5** | Rage 单次跳跃过大破坏体验 | 实施 15 点上限 Clamp，并添加"愤怒冲击"视觉效果补偿 |

---

## Performance Implications

| 维度 | 影响 | 说明 |
|------|------|------|
| **CPU** | 极低 | 每帧仅计算 Lerp 插值，无复杂逻辑；消散检查每 3 秒一次 |
| **Memory** | 极低 | 仅存储两个 int 值（Sanity/Rage）和枚举状态 |
| **Load Time** | 无 | 无资源加载 |
| **Network** | 无 | 纯本地系统 |

---

## Implementation Dependencies

本系统的实现依赖于以下 ADR 定义的系统：

| 依赖系统 | 依赖关系 | 说明 | 最小接口集 |
|----------|----------|------|------------|
| [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) | 必须 | 所有事件订阅/发布基于 EventBus | - |
| [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) | 必须 | CombatStateChangedEvent 来源；AlertState 枚举定义 | `NPCManager.IsAnyNPCInCombat(): bool`<br>`NPCManager.GetAlertState(npcId): AlertState`<br>注：以 ADR-0004 实际签名为准 |
| [ADR-0008: 脆弱度与伤害系统](./adr-0008-health-lethality-architecture.md) | 必须 | PlayerDamagedEvent 定义 | - |
| [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) | 必须 | KillTagEvent 定义 | - |
| [ADR-0016: 线索与日志系统](./adr-0016-clue-journal-architecture.md) | 必须 | ClueDiscoveredEvent 来源（上游依赖）。注：此为接口声明式依赖，非实现依赖；两系统通过 EventBus 解耦，无循环引用问题 | - |
| [ADR-0015: UI 系统](./adr-0015-ui-system-architecture.md) | 必须 | ScreenEffects 消费者；PsychologicalStateEvent 订阅；Sanity=0 时 HUD 半透明叠加显示 | - |
| Shared Types: 伤害与命中类型 | 必须 | NPCIdentityType 枚举定义 | - |

---

## Migration Plan

本决策不涉及对现有代码的迁移（无现有实现）。实施步骤：

1. **Phase 1**：实现 `SanityRageMeter` 单例和状态枚举
2. **Phase 2**：实现事件订阅（KillTagEvent / ClueDiscoveredEvent / PlayerDamaged）
3. **Phase 3**：实现愤怒消散和阈值锁定机制
4. **Phase 4**：实现 `VisualEffectCalculator` 插值计算
5. **Phase 5**：实现效果请求事件发布接口
6. **Phase 6**：与 ScreenEffects / DynamicPostProcessing / PlayerController 联调

---

## Validation Criteria

| ID | 标准 | 测试方法 |
|----|------|----------|
| VC-1 | 击杀恶徒后，Rage +10，Sanity -5 | 执行击杀，监听 rage_changed 和 sanity_changed 事件 |
| VC-2 | 愤怒每 3 秒自动 -2（非战斗状态） | 积累愤怒后等待，观察 rage_changed 事件 |
| VC-3 | 愤怒消散期间再次击杀，立即停止消散 | 积累愤怒，在消散过程中执行击杀，验证数值不降反升 |
| VC-4 | Rage > 90 时状态为 FRENZIED | 模拟 Rage = 95，验证 State 输出 |
| VC-5 | Sanity < 20 且 Rage > 70 时状态为 SOUL_SPLIT | 同时设置两值，验证优先级 |
| VC-6 | FRENZIED 优先于 BROKEN | 设置 Sanity = 10, Rage = 95，验证状态为 FRENZIED |
| VC-7 | 视觉效果 Lerp 插值无跳变 | 观察低理智状态下的视觉效果过渡 |
| VC-8 | Frenzy Lock 持续时间内 Rage 不会低于 70 | 触发狂暴后等待 10 秒，验证 Rage >= 70 |
| VC-9 | Rage = 71, Sanity = 50 → 状态为 FRENZIED | 边界条件：Rage >= 71 进入 FRENZIED |
| VC-10 | Sanity = 19, Rage = 70 → 状态为 BROKEN | 边界条件：Rage <= 70 时为 BROKEN |
| VC-11 | Sanity = 19, Rage = 71 → 状态为 SOUL_SPLIT | 边界条件：同时满足极端值 |
| VC-12 | SOUL_SPLIT 状态下 Noise + Shake 同时生效 | 验证两个效果叠加 |
| VC-13 | Rage 单次变化不超过 15 | **单元测试（正向）**：创建测试用例，模拟 `RageDelta = BaseValue * MomentumMultiplier = 20 * 1.5 = 30`，验证实际 RageDelta 被 Clamp 为 15。<br>**单元测试（负向）**：模拟 `RageDelta = BaseValue * MomentumMultiplier = -20 * 1.5 = -30`，验证实际 RageDelta 被 Clamp 为 -15。<br>**集成测试**：快速连续击杀两个 VICTIM（BaseValue=-15，×1.3 Momentum），验证第二次击杀的 RageDelta 不超过 15。 |
| VC-14 | CombatStateChangedEvent 触发战斗状态判断 | 模拟 NPC 进入 ALERT，验证 IsInCombat = true |
| VC-15 | SOUL_SPLIT 状态下移动速度 +10% 符合设计意图 | **[Future Validation]** Playtest 验证玩家在 SOUL_SPLIT 状态反馈是否为"肾上腺素"体验（速度提升有感但视觉 debuff 明显），如不符合则调整 MovementSpeedBonus 数值。此验证项需要在完整可玩的 milestone 版本中进行，当前阶段仅做代码逻辑验证。 |

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — 本系统遵循事件驱动原则
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — CombatStateChangedEvent 定义；IsInCombat 判定依赖的 AlertState 枚举
- [ADR-0008: 脆弱度与伤害系统](./adr-0008-health-lethality-architecture.md) — PlayerDamagedEvent 定义
- [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) — KillTagEvent 定义；TieUp 转化线人机制
- [ADR-0014: DialogTree 接口协议](./adr-0014-dialog-tree-interface-architecture.md) — 转化线人流程通过 DialogTree 实现
- [ADR-0016: 线索与日志系统](./adr-0016-clue-journal-architecture.md) — ClueDiscoveredEvent 定义
- [ADR-0015: UI 系统](./adr-0015-ui-system-architecture.md) — HUDOverlayOpacityRequest 消费者
- [ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — ScreenEffectRequestEvent 统一接口；ScreenEffectsManager 集中处理屏幕后处理效果
- [Shared Types: 伤害与命中类型](./shared-types.md) — NPCIdentityType 枚举定义
