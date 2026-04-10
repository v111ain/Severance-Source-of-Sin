# 理智/愤怒系统 (Sanity/Rage Meter)

> **Status**: Approved (Revised)
> **Author**: Systems Designer
> **Last Updated**: 2026-04-07
> **Priority**: Vertical Slice
> **Layer**: Meta
> **Implements Pillar**: 罪恶的深度 (Depth of Sin)

## 1. Overview

理智/愤怒系统是一个 **Meta 系统**，通过视听反馈将玩家的行为选择转化为持续的情绪压力。它不改变玩家的数值能力，而是放大玩家"感受"层面的沉浸感。

**核心定位**：
- 玩家的每一个选择（杀/不杀、发现悲剧真相）都会在心理层面留下痕迹
- 理智和愤怒是**双轨并行**的独立计量条：
  - Sanity（理智）：目睹悲剧/暴力下降，发现真相/救赎上升
  - Rage（愤怒）：杀戮/威胁上升，时间推移/仁慈行为下降
- 系统效果是**累积且可逆的**：通过正确行动可以恢复理智，通过时间推移愤怒会逐渐消散

**状态优先级说明**：
当Sanity和Rage同时达到极端值时，视觉反馈优先级为 **FRENZIED > BROKEN**。这是因为愤怒状态（如准星抖动、移动加速）更容易通过外部行为观察，而理智崩溃（如视野缩窄、色彩丧失）更多是内部感受。两者可以同时生效，但视觉表现优先显示愤怒状态。

---

## 2. Player Fantasy

**"血腥与眼泪的重量。"**

玩家在执行每一个行为时，都应该意识到**代价是真实的**：
- **愤怒的代价**：无节制的杀戮让玩家陷入"狂战士"状态——准星抖动、移动加速但判断力下降，屏幕边缘被血色侵蚀
- **理智的代价**：目睹过多的死亡、发现NPC的悲剧背景，会让玩家陷入"抑郁"状态——视野变窄、色彩褪去、世界仿佛被抽空了意义

**情绪弧线设计**：
- *开局*：理智高、愤怒低 → 视野清晰、色彩饱和、准星稳定
- *中期*：根据玩家选择出现分化
  - 杀戮路线：愤怒升高、理智缓慢下降 → 准星抖动加剧、边缘血红色暗角
  - 探索路线：理智稳定或略升、愤怒缓慢下降 → 画面舒适、情绪平静
- *高潮*：极端状态触发
  - 愤怒溢出（>90%）→ 屏幕完全被血红色覆盖，进入"狂暴"视觉效果
  - 理智崩溃（<10%）→ 视野严重缩窄、严重噪点、色彩完全丧失
- *结局影响*：高愤怒/低理智 vs 低愤怒/高理智 → 不同的结局画面风格

---

## 3. Detailed Design

### 3.1 Core Rules

**规则1：双轨并行计量**

| 计量条 | 范围 | 描述 |
|--------|------|------|
| **Sanity (理智)** | 0-100 | 玩家心理健康程度。目睹悲剧/暴力下降；发现真相/救赎行为上升 |
| **Rage (愤怒)** | 0-100 | 玩家累积的攻击性能量。杀戮/威胁上升；时间推移/仁慈行为下降 |

**规则2：状态阈值**

| 状态 | Sanity 范围 | Rage 范围 | 视觉效果 | 说明 |
|------|-------------|-----------|----------|------|
| **平静 (Calm)** | 70-100 | 0-30 | 正常画面 | 默认状态 |
| **不安 (Uneasy)** | 40-69 | 31-50 | 轻微暗角、饱和度-10% | 任一指标触发 |
| **激动 (Agitated)** | 20-39 | 51-70 | 中度暗角、饱和度-25%、轻微噪点 | 任一指标触发 |
| **崩溃 (Broken)** | 0-19 | <71 | 视野严重缩窄、严重噪点、色彩完全丧失 | Sanity极端时 |
| **狂暴 (Frenzied)** | <70 | 71-100 | 准星抖动、移动加速、边缘血红色暗角 | Rage极端时 |
| **灵魂分裂 (Soul Split)** | 0-19 | 71-100 | BROKEN + FRENZIED 效果叠加 | 极端状态，触发特殊成就 |

**状态优先级**：当同时满足多个状态条件时，按以下优先级显示视觉效果：
1. FRENZIED（愤怒优先级最高）
2. BROKEN（理智崩溃）
3. AGITATED
4. UNEASY
5. CALM（默认）

**视觉效果叠加规则**：
- 暗角（Vignette）：取所有适用状态的最大值
- 饱和度（Saturation）：取所有适用状态的最小值（越低越灰暗）
- 噪点（Noise）：BROKEN状态特有，FRENZIED不产生噪点
- 准星抖动（Shake）：FRENZIED状态特有，BROKEN不产生抖动
- 移动速度：FRENZIED状态下+10%，BROKEN状态下-5%

**规则3：跨系统事件响应**

系统监听以下事件并触发计量变化：

| 事件来源 | 事件类型 | 理智变化 | 愤怒变化 |
|---------|---------|---------|---------|
| GrittyTakedowns | `KillTagEvent{kill_tag: ENEMY}` | -5 | +10 |
| GrittyTakedowns | `KillTagEvent{kill_tag: VICTIM}` | -15 | +5 |
| GrittyTakedowns | `KillTagEvent{kill_tag: ACCOMPLICE}` | -8 | +8 |
| GrittyTakedowns | `TieUp` (捆绑) | 0 | -2 |
| Clue&Journal | `ClueDiscoveredEvent` (悲剧线索，第一次) | -5 | 0 |
| Clue&Journal | `ClueDiscoveredEvent` (悲剧线索，后续) | -3 | 0 |
| Clue&Journal | `ClueDiscoveredEvent` (悲剧线索，后续深度揭示) | -2 | 0 |
| Clue&Journal | `ClueDiscoveredEvent` (身份/位置) | +5 | 0 |
| 环境 | 目睹NPC死亡（未亲手击杀） | -5 | +3 |
| 玩家控制器 | 玩家受到轻伤 | 0 | +5 |
| 玩家控制器 | 玩家受到重伤 | -10 | +10 |

### 3.2 States and Transitions

**玩家心理状态机**：

```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    ▼                                         │
[CALM] ◀─────── (+Sanity恢复) ─────── [+UNEASY]             │
   │                                        │                 │
   │ (Rage上升/Sanity下降)                  │ (进一步恶化)    │
   │                                        ▼                 │
   │                              [AGITATED]                 │
   │                                        │                 │
   │  (极端行为)                             │ (极端行为)      │
   ▼                                        ▼                 │
[FRENZIED]                          [BROKEN]                 │
   │                                        │                 │
   │  (时间/仁慈行为)                        │ (发现真相/救赎) │
   └────────────────────────────────────────┘                 │
```

**状态转移条件**：

| 当前状态 | 触发条件 | 目标状态 | 效果 |
|---------|---------|---------|------|
| CALM | Rage > 30 OR Sanity < 70 | UNEASY | 开始应用视觉滤镜 |
| CALM | Rage > 50 OR Sanity < 50 | AGITATED | 滤镜强度增加 |
| UNEASY | Rage > 50 OR Sanity < 40 | AGITATED | — |
| UNEASY | Rage > 70 OR Sanity < 20 | BROKEN/FRENZIED | 极端视觉效果 |
| AGITATED | Rage > 70 OR Sanity < 20 | BROKEN/FRENZIED | — |
| BROKEN | Sanity > 40 | AGITATED | 恢复 |
| FRENZIED | Rage < 50 | AGITATED | 恢复 |
| ANY | Rage > 90 | FRENZIED (溢出) | 特殊"狂暴"视觉 |

### 3.3 Interactions with Other Systems

**数据流入 (Inputs)**：

| 来源系统 | 数据内容 | 处理方式 |
|---------|---------|---------|
| GrittyTakedowns | `KillTagEvent{kill_tag: NPCIdentityType}` | 根据击杀目标身份（ENEMY/ACCOMPLICE/VICTIM）计算理智惩罚和愤怒变化 |
| **叙事系统 (Narrative System)** | `SanityRecoveryEvent{type, base_recovery, module_multiplier, final_recovery}` | 接收救赎行为的理智恢复（埋葬受害者、找到家人等）。Narrative 系统已根据公式计算 final_recovery 值，本系统直接应用 |
| Clue&Journal | `ClueDiscoveredEvent{clue_id, clue_category}` | 根据线索类别计算理智变化（悲剧线索为负） |
| Health&Lethality | `PlayerDamaged{damage_type, is_lethal}` | 受伤时愤怒上升，重伤理智也下降 |
| 玩家控制器 | 时间流逝 | 愤怒每5秒自动-1（自然消散） |

**SanityRecoveryEvent 事件数据结构**：

```csharp
struct SanityRecoveryEvent {
    RecoveryType type;      // BURY_VICTIM / FIND_FAMILY / MERCY_ACTS / TRAGEDY_UNLOCK
    int base_recovery;      // 基础恢复值（由 Narrative 系统根据公式计算）
    float module_multiplier;// 模块加成（如有 REL_02 / TRUTH_02 则 > 1.0）
    int final_recovery;     // 最终恢复值 = base_recovery * module_multiplier
}

enum RecoveryType {
    BURY_VICTIM,     // 埋葬受害者
    FIND_FAMILY,     // 找到家人
    MERCY_ACTS,      // 仁慈行为累积
    TRAGEDY_UNLOCK   // 悲剧故事解锁（部分抵消误杀惩罚）
}
```

**处理逻辑**：
- 收到 `SanityRecoveryEvent` 后，直接将 `final_recovery` 值加到当前 Sanity
- Sanity 上限为 100（超过部分截断）
- 不需要二次计算（Narrative 系统已根据 Section 4.3 公式完成计算）

**数据流出 (Outputs)**：

| 目标系统 | 数据内容 | 用途 |
|---------|---------|------|
| ScreenEffects | `VignetteRequest{intensity}` | 理智低时增加边缘暗角 |
| ScreenEffects | `NoiseRequest{intensity}` | 理智极低时增加视觉噪点 |
| ScreenEffects | `SaturationRequest{multiplier}` | 理智低时降低色彩饱和度 |
| ScreenEffects | `ShakeRequest{intensity}` | 愤怒高时增加准星抖动 |
| 玩家控制器 | `MovementSpeedMultiplier` | 愤怒高时轻微增加移动速度（+10%上限） |
| DynamicPostProcessing | `PsychologicalState` | 通知后处理系统当前心理状态 |

---

## 4. Formulas

**公式1：理智值变化计算**

```
EffectiveSanityDelta = BaseValue[EventType] × ContextMultiplier × TimeMultiplier × NarrativeSignificanceMultiplier × SanityPenaltyMultiplier
```

| 变量 | 定义 | 来源 |
|------|------|------|
| `BaseValue[EventType]` | 事件基础惩罚值 | 本系统定义（见事件类型表） |
| `ContextMultiplier` | 情境乘数 | 本系统定义（连续击杀/首次击杀等） |
| `TimeMultiplier` | 时间乘数 | 本系统定义（首次/连续/间隔） |
| `NarrativeSignificanceMultiplier` | 叙事重要性乘数 | 线索系统（Clue & Journal） |
| **`SanityPenaltyMultiplier`** | **背景理智惩罚系数** | **角色背景系统（Character Background）** |

**SanityPenaltyMultiplier**（来自角色背景系统）：

| 背景 | 系数 | 说明 |
|------|------|------|
| 普通人 | 0.9x | 略微习惯暴力，心理承受力中等 |
| 特工 | 0.8x | 习惯暴力惩罚最低，道德挣扎最剧烈 |
| 雇佣兵 | 1.15x | 最不适应内心黑暗，冲动失控倾向高 |

> **集成说明**：`SanityPenaltyMultiplier` 在公式最后一位与最终乘积相乘。例如：特工击杀帮凶（基准惩罚 -8）→ `-8 × 1.0 × 1.0 × 1.0 × 0.8 = -6.4`

| 事件类型 | BaseValue | 说明 |
|---------|-----------|------|
| 击杀恶徒 | -5 | 道德上可接受，但仍造成心理负担 |
| 击杀帮凶 | -8 | 知道他们是受害者但仍下手 |
| 击杀无辜者 | -15 | 严重的道德错误 |
| 目睹NPC死亡 | -5 | 非亲手，但仍受影响 |
| 发现悲剧线索（第一次） | -5 | 首次接触悲剧，情感冲击 |
| 发现悲剧线索（后续） | -3 | 递减惩罚，避免过度惩罚探索 |
| 发现悲剧线索（后续深度揭示） | -2 | 多次接触同一悲剧的边际递减 |
| 发现身份/位置线索 | +5 | 真相带来满足感 |
| 捆绑NPC（不杀） | +3 | 仁慈选择 |
| 成功转化线人 | +5 | 正向干预 |

**ContextMultiplier**（用于战斗/暴力类事件）：
- 连续击杀（5秒内）：×1.5（累积心理压力）
- 首次击杀：×1.0
- 击杀后被目击：×1.2

**TimeMultiplier**：
- 首次发现：×1.0
- 5秒内连续事件：×1.5
- 30秒后再次发现：×0.8

**NarrativeSignificanceMultiplier**（用于线索类事件，来自线索系统）：
- 普通线索（NORMAL）：×1.0
- 与主要目标相关（MAIN_TARGET）：×1.5
- 揭示 NPC 同情的一面（NPC_SYMPATHY）：×2.0

**悲剧线索惩罚递减机制说明**：
- 首次发现悲剧线索：完整惩罚（-5）
- 后续发现同类型悲剧线索：递减惩罚（-3）
- 多次深入揭示同一悲剧背景：最低惩罚（-2）
- 设计意图：玩家应该被**鼓励探索**，而不是因为收集线索而受到过度惩罚。第一条悲剧线索的情感冲击最强烈，后续递减避免玩家因"知道太多"而崩溃。

> **⚠️ Playtest 验证需求**：当前惩罚曲线假设每任务平均有 3-5 条悲剧线索。如果实际关卡设计中悲剧线索数量超出此范围，可能导致理智惩罚累积过快。需要在上线前通过 playtest 验证曲线合理性。

**线索系统事件接收**：
ClueDiscoveredEvent 携带以下字段用于计算：
- `clue_category`：线索类别（TRAGEDY/IDENTITY/LOCATION 等）
- `discovery_stage`：发现阶段（FIRST/SUBSEQUENT/DEEP_REVEAL）
- `narrative_significance`：叙事重要性（NORMAL/MAIN_TARGET/NPC_SYMPATHY）

最终惩罚 = BaseValue[discovery_stage] × NarrativeSignificanceMultiplier

---

**公式2：愤怒值变化计算**

```
RageDelta = BaseValue[EventType] * MomentumMultiplier
```

| 事件类型 | BaseValue |
|---------|-----------|
| 击杀任意NPC | +10 |
| 威胁NPC | +3 |
| 玩家受伤 | +5 |
| 玩家重伤 | +10 |
| 捆绑NPC | -2 |
| 时间流逝（每5秒） | -1 |

**MomentumMultiplier**：
- 快速连续击杀（<3秒间隔）：×1.3
- 10秒内无击杀：×0.8（冷却衰减）

**消散与积累的互斥机制**：
- 愤怒消散期间，如果玩家再次击杀，消散**立即停止**，从当前值累加
- MomentumMultiplier 的"冷却衰减"只在**非战斗状态**下生效
- **消散判定公式**（每3秒执行一次）：
  ```
  if (IsInCombat == false):
      Rage = Rage - RageDecayAmount  # 默认 -2
  else:
      Rage 保持不变（战斗状态下不消散）
  ```

**狂暴阈值锁定机制 (Frenzy Threshold Lock)**：
- 当 Rage 首次超过 70（进入 FRENZIED 状态）时，触发阈值锁定
- 锁定持续时间内，即使 Rage 正常消散，也**不会低于 70**
- 锁定持续时间：`FrenzyLockDuration`（默认 10 秒）
- 锁定结束后，Rage 正常消散
- 设计意图：玩家一旦进入"狂暴"状态，应该能够**维持这种状态一段时间**，体验完整的狂暴快感，而不是刚进入就立刻消散
- **击杀时愤怒变化公式**（无论是否在消散状态）：
  ```
  RageDelta = BaseValue[EventType] * MomentumMultiplier
  ```
  - 正常积累：MomentumMultiplier = 1.0 或 1.3（快速连续击杀）
  - 消散期间击杀：MomentumMultiplier = 1.0（直接累加，**不抵消**之前消散的愤怒）

---

**公式3：视觉效果插值**

```
VignetteIntensity = Lerp(0.0, 0.8, 1.0 - Sanity/100)
NoiseIntensity = Lerp(0.0, 0.5, 1.0 - Sanity/50)  # 当Sanity<50时开始
SaturationMultiplier = Lerp(0.3, 1.0, Sanity/100)
ShakeIntensity = Lerp(0.0, 8.0, Rage/100)
MovementSpeedBonus = Lerp(0.0, 0.1, Rage/100)  # 最高+10%
```

---

**公式4：状态阈值判定**

```
EffectiveFrenzyThreshold = BaseFrenzyThreshold + FrenzyThresholdModifier

// 其中：BaseFrenzyThreshold = 70（由本系统定义）
//       FrenzyThresholdModifier 来自 Character Background 系统（普通人=+8, 特工=±0, 雇佣兵=-8）

if Rage > 90:
    State = FRENZIED
elif Sanity < 20:
    State = BROKEN
elif Rage > EffectiveFrenzyThreshold OR Sanity < 40:
    State = AGITATED
elif Rage > 30 OR Sanity < 70:
    State = UNEASY
else:
    State = CALM
```

> **FrenzyThresholdModifier 集成说明**：`FrenzyThresholdModifier` 由 Character Background 系统提供（见 Dependencies 章节）。本公式使用 EffectiveFrenzyThreshold 替代硬编码的 70 阈值，实现背景对狂暴触发时机的个性化调整。
>
> **示例**：雇佣兵（`FrenzyThresholdModifier = -8`）的有效狂暴阈值为 `70 + (-8) = 62`，即 Rage > 62 时即进入 AGITATED 状态，Rage > 90 时进入 FRENZIED。普通人（`FrenzyThresholdModifier = +8`）的有效狂暴阈值为 `70 + 8 = 78`，需要更高的愤怒积累才会触发狂暴状态。

---

## 5. Edge Cases

**边缘情况1：理智和愤怒同时达到极端值**

问题：玩家同时处于极低理智（<20）和极高愤怒（>90）状态。

处理：
- 优先显示 FRENZIED 状态（愤怒优先级更高）
- 视觉效果叠加：血红色暗角 + 严重噪点
- 触发特殊成就/标记"灵魂分裂"（可选叙事flag）

---

**边缘情况2：快速反复击杀后立即捆绑**

问题：玩家在短时间内击杀3个NPC（愤怒+30），然后捆绑第4个（愤怒-2）。

处理：
- 捆绑带来的愤怒下降**不立即抵消**击杀积累
- 需要更长的恢复时间（捆绑只是减速，不是逆转）
- 视觉反馈显示"愤怒仍在积累，但速度放缓"

---

**边缘情况3：理智恢复超过100**

问题：通过连续发现正面线索，理智值超过100。

处理：
- 硬上限100，超出部分不记录
- 设计意图：保持"完美心理状态难以维持"的紧迫感

---

**边缘情况4：零理智状态**

问题：理智降至0。

处理：
- 屏幕变为灰度/黑白，边缘极度暗角
- **不触发死亡**（区别于Health系统）
- 视觉效果使用 SaturationMultiplier 的最低值（0.3），配合 NoiseIntensity 达到"几乎失真"的效果
- 玩家可以继续行动，但所有视觉反馈严重恶化
- 效果持续直到理智通过自然恢复或正面事件回到>20

---

**边缘情况5：愤怒消散期间再次积累**

问题：愤怒正在消退（每5秒-1），玩家在此期间再次击杀。

处理：
- 立即停止消散
- 从**当前值**开始累加新击杀的愤怒值
- 防止"边消边杀"的无效循环

---

**边缘情况6：目睹自己造成的死亡（回头看）**

问题：玩家击杀NPC后，NPC尸体在视野范围内。

处理：
- 尸体停留期间，该NPC的击杀效果**不重复触发**
- 离开视野范围后再次进入，**不重复触发**
- 同一击杀类型只触发一次计量变化

---

## 6. Dependencies

### 上游依赖（系统依赖谁）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|----------|
| GrittyTakedowns | 硬依赖 | 监听 `KillTagEvent` 事件获取击杀数据 |
| **叙事系统 (Narrative System)** | 硬依赖 | 接收 `SanityRecoveryEvent`（理智恢复事件：埋葬受害者、找到家人、仁慈行为、悲剧解锁）|
| Clue&Journal | 硬依赖 | 监听 `ClueDiscoveredEvent` 获取线索发现 |
| Health&Lethality | 软依赖 | 监听 `PlayerDamaged` 事件获取受伤数据 |
| 玩家控制器 | 软依赖 | 获取时间流逝，用于愤怒自然消散 |

### 下游依赖（谁依赖本系统）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|----------|
| ScreenEffects | 硬依赖 | 发送视觉效果请求（暗角/噪点/饱和度/抖动） |
| DynamicPostProcessing | 软依赖 | 发送心理状态枚举，供后处理着色器使用 |
| 玩家控制器 | 软依赖 | 发送移动速度乘数 |
| **主角背景角色系统 (Character Background)** | 硬依赖 | 接收 `SanityPenaltyMultiplier`（理智惩罚修正）和 `FrenzyThresholdModifier`（狂暴阈值调整），在计算理智惩罚和判定狂暴状态时应用 |

### 依赖关系矩阵

```
GrittyTakedowns ──KillTagEvent──┐
                                 │
Clue&Journal ──ClueEvent──────▶ │  ┌─────────────────────┐
                                 │  │                     │
Narrative ──SanityRecoveryEvent─▶ │  │  Sanity/Rage Meter  │──VignetteRequest──▶ ScreenEffects
         System ──────────────▶ │  │                     │──NoiseRequest──▶
                                 │  │                     │──SaturationRequest──▶
                                 │  └─────────────────────┘
                                 │            │
                                 │            │──ShakeRequest──▶ ScreenEffects
                                 │            │──MovementSpeedMultiplier──▶ PlayerController
                                 │            │──PsychologicalState──▶ DynamicPostProcessing
```

---

## 7. Tuning Knobs

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `SanityMax` | int | 100 | — | 理智值上限 |
| `RageMax` | int | 100 | — | 愤怒值上限 |
| `RageDecayInterval` | float | 3.0 | 2.0~5.0 | 愤怒自然消散间隔（秒） |
| `RageDecayAmount` | int | 2 | 1~5 | 每次消散的愤怒值 |
| `FrenzyLockDuration` | float | 10.0 | 5.0~20.0 | 狂暴阈值锁定持续时间（秒） |
| `KillRageBase` | int | 10 | 5~20 | 击杀愤怒增量 |
| `KillSanityPenalty_Innocent` | int | -15 | -25~-5 | 误杀无辜者理智惩罚 |
| `TragedyClueSanityPenalty_First` | int | -5 | -3~-8 | 首次悲剧线索理智惩罚 |
| `TragedyClueSanityPenalty_Subsequent` | int | -3 | -2~-5 | 后续悲剧线索理智惩罚（递减） |
| `TragedyClueSanityPenalty_Deep` | int | -2 | -1~-4 | 深度揭示悲剧线索理智惩罚（边际递减） |
| `NarrativeSignificanceMultiplier_Normal` | float | 1.0 | 0.5~1.5 | 普通线索叙事重要性乘数 |
| `NarrativeSignificanceMultiplier_MainTarget` | float | 1.5 | 1.0~2.0 | 与主要目标相关乘数 |
| `NarrativeSignificanceMultiplier_NPCSympathy` | float | 2.0 | 1.5~3.0 | 揭示 NPC 同情乘数 |
| `VignetteMaxIntensity` | float | 0.8 | 0.5~1.0 | 暗角最大强度 |
| `NoiseMaxIntensity` | float | 0.5 | 0.3~0.8 | 噪点最大强度 |
| `SaturationMinMultiplier` | float | 0.3 | 0.1~0.5 | 最低饱和度乘数 |
| `ShakeMaxIntensity` | float | 8.0 | 4.0~15.0 | 最大准星抖动强度 |
| `MovementSpeedMaxBonus` | float | 0.1 | 0.05~0.2 | 最大移动速度加成 |
| `StateTransitionThreshold_High` | int | 70 | 60~80 | 状态进入/退出阈值（高） |
| `StateTransitionThreshold_Low` | int | 40 | 30~50 | 状态进入/退出阈值（低） |
| `StateTransitionThreshold_Critical` | int | 20 | 10~30 | 临界状态阈值 |

**调参风险提示**：

| 参数 | 风险 |
|------|------|
| `RageDecayAmount` 设置过高 | 愤怒消散太快，削弱"狂暴"状态的紧张感 |
| `KillRageBase` 设置过低 | 击杀奖励太小，玩家失去"狂战士"体验 |
| `VignetteMaxIntensity` 设置过高 | 画面太暗，影响游戏可玩性 |
| `StateTransitionThreshold_Critical` 设置过高 | 临界状态太容易触发，削弱情感冲击力 |

---

## 8. Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 击杀恶徒后，Rage +10，Sanity -5 | 执行击杀，监听 `rage_changed` 和 `sanity_changed` 事件 |
| AC-2 | 击杀无辜者后，Sanity -15（高于恶徒惩罚） | 故意击杀标记为Innocent的NPC，验证惩罚差额 |
| AC-3 | 愤怒每5秒自动-1 | 积累愤怒后等待，观察 `rage_changed` 事件 |
| AC-4 | 愤怒消散期间再次击杀，立即停止消散 | 积累愤怒后，在消散过程中执行击杀，验证数值不降反升 |
| AC-5 | 理智低于20时，视觉噪点强度 > 0.3 | 模拟低理智状态，使用ScreenEffects接口读取噪声值 |
| AC-6 | 愤怒高于70时，准星抖动强度 > 4.0 | 模拟高愤怒状态，读取ShakeIntensity值 |
| AC-7 | 理智为0时，画面饱和度降至最低但不黑屏 | 强制设置Sanity=0，验证SaturationMultiplier ≈ 0.3 |
| AC-8 | Rage > 90时，状态判定为FRENZIED | 模拟Rage=95，验证State输出 |

### 跨系统验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-9 | ScreenEffects接收到正确的VignetteRequest | 监听事件总线，验证参数与当前Sanity匹配 |
| AC-10 | ScreenEffects接收到正确的ShakeRequest | 监听事件总线，验证参数与当前Rage匹配 |
| AC-11 | 玩家控制器接收到正确的移动速度乘数 | 在高愤怒状态下测试移动速度（预期+10%） |
| AC-12 | Clue&Journal的悲剧线索发现触发Sanity下降 | 发现悲剧线索，验证理智变化事件 |

### 边缘情况验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-13 | 理智和愤怒同时极端时，FRENZIED优先于BROKEN | 同时设置Sanity=10, Rage=95，验证状态为FRENZIED |
| AC-14 | 快速连续击杀后捆绑，愤怒下降幅度小于积累 | 3次快速击杀(+30)，然后捆绑(-2)，验证净效果 |
| AC-15 | 同一击杀只触发一次计量变化 | 击杀后反复进出视野，验证无重复事件 |

### 性能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-16 | 每帧处理事件数量不影响帧率（<16ms） | 在高频击杀场景下测量帧时间 |
| AC-17 | 视觉效果的Lerp插值无跳变 | 观察低理智状态下的视觉效果过渡 |

---

## Change Log

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-07 | 0.1 | 初稿创建 | Systems Designer Agent |
| 2026-04-10 | 0.2 | 设计审查修复：更新依赖矩阵，添加Narrative System的SanityRecoveryEvent监听说明；定义SanityRecoveryEvent完整数据结构；更新Section 3.3 Inputs表格，澄清Narrative System的理智恢复事件处理逻辑 | Claude Code |
| 2026-04-10 | 0.3 | 设计审查修复：在公式4中集成 FrenzyThresholdModifier，计算 EffectiveFrenzyThreshold 替代硬编码阈值，使不同背景的狂暴触发时机符合设计预期 | Claude Code |
