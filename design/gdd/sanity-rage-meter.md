# 理智/愤怒系统 (Sanity/Rage Meter)

> **Status**: Approved (Revised)
> **Author**: Systems Designer
> **Last Updated**: 2026-04-14
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
| **不安 (Uneasy)** | 40-69 | 31-50 | 轻微暗角、饱和度-10% | Sanity 40-69 且 Rage 31-50（AND 逻辑，两个条件同时满足） |
| **激动 (Agitated)** | 20-39 | 51-70 | 中度暗角、饱和度-25%、轻微噪点 | Sanity 20-39 且 Rage 51-70（AND 逻辑，两个条件同时满足） |
| **崩溃 (Broken)** | 0-30 | <71 | 视野严重缩窄、严重噪点、色彩完全丧失 | Sanity <= 30 且 Rage < 71（与公式4 BROKEN判定条件一致） |
| **狂暴 (Frenzied)** | — | >90 | 准星抖动、移动加速、边缘血红色暗角 | Rage > 90（Sanity不影响State判定，仅影响VisualState） |
| **灵魂分裂 (Soul Split)** | 0-30* | 71-100* | BROKEN + FRENZIED 效果叠加 | Rage >= 71 且 Sanity <= 30（极端状态，触发特殊成就） |

> * SOUL_SPLIT 的 Rage 范围 71-100 表示 Rage >= 71（与公式4一致）；Sanity 范围 0-30 表示 Sanity <= 30（与公式4一致）。视觉优先级遵循状态优先级定义（FRENZIED 优先）。

**状态优先级**（State 判定优先级，从高到低）：
1. **SOUL_SPLIT**（Rage >= 71 且 Sanity <= 30 时触发，特殊成就/moral flag）
2. **FRENZIED**（Rage > 90 时触发，视觉优先级最高）
3. BROKEN（Sanity <= 30 且不满足 SOUL_SPLIT 条件时触发）
4. AGITATED（Rage 51-70 且 Sanity 31-39 时触发；注：Sanity 20-30 由 BROKEN 优先处理）
5. UNEASY（Rage 31-50 且 Sanity 40-69，或 Rage 31-50 但 Sanity >= 40 时触发）
6. CALM（默认状态）

> **SOUL_SPLIT 与 FRENZIED 的区别**：
> - FRENZIED 是视觉状态，当 Rage > 90 时触发，视觉显示血红色暗角、准星抖动等愤怒效果
> - SOUL_SPLIT 是成就/moral flag，当 Rage 71-100 且 Sanity 0-30 时触发，表示玩家同时处于极端愤怒和极端崩溃状态
> - 两者可以共存：玩家可能同时是 SOUL_SPLIT（状态flag）且视觉显示 FRENZIED（视觉效果）
> - 当 Rage > 90 且 Sanity <= 30 时，State = SOUL_SPLIT，但 VisualState = FRENZIED（视觉优先级）

**视觉效果叠加规则**：

当多个状态同时满足时，各视觉效果参数按以下规则取值：

| 参数 | 叠加规则 | 说明 |
|------|---------|------|
| 暗角（Vignette） | **取最大值** | 所有适用状态的暗角值取最大者 |
| 饱和度（Saturation） | **取最小值** | 所有适用状态的饱和度值取最小者（越低越灰暗） |
| 噪点（Noise） | **BROKEN独有** | FRENZIED不产生噪点，即使与BROKEN叠加也只取BROKEN的噪点值 |
| 准星抖动（Shake） | **FRENZIED独有** | BROKEN不产生抖动，即使与FRENZIED叠加也只取FRENZIED的抖动值 |
| 移动速度 | **取高优先级值（FRENZIED 覆盖）** | 当 SOUL_SPLIT 状态叠加时，FRENZIED 的 +10% 覆盖 BROKEN 的 -5%，取 FRENZIED 的值。愤怒状态主导移动速度加成 |

**叠加计算示例**：

假设当前状态为 SOUL_SPLIT（同时满足 FRENZIED 和 BROKEN）：

| 参数 | FRENZIED 值 | BROKEN 值 | 最终取值 | 取值规则 |
|------|-------------|-----------|---------|---------|
| Vignette | 0.5（中等暗角） | 0.8（严重暗角） | **0.8** | 取最大值 |
| Saturation | 1.0（正常） | 0.3（灰暗） | **0.3** | 取最小值 |
| Noise | 0（无） | 0.5（严重噪点） | **0.5** | BROKEN独有 |
| Shake | 4.0（中等抖动，内部值） | 0（无） | **4.0**（内部值；发送给 ScreenEffects 的归一化值 = 4.0/8.0 = **0.5**） | FRENZIED独有 |
| MovementSpeed | +10% | -5% | **+10%** | FRENZIED主导 |

**具体场景示例**：

> **场景**：玩家当前 Sanity=15（触发BROKEN），Rage=85（触发FRENZIED），进入 SOUL_SPLIT 状态
>
> **视觉效果计算**：
> - VignetteIntensity = max(FRENZIED_vignette, BROKEN_vignette) = max(0.5, 0.8) = **0.8**
> - SaturationMultiplier = min(FRENZIED_saturation, BROKEN_saturation) = min(1.0, 0.3) = **0.3**
> - NoiseIntensity = BROKEN特有 = **0.5**（FRENZIED不产生噪点）
> - ShakeIntensity = FRENZIED特有 = **4.0**（内部值；BROKEN不产生抖动）→ 发送给 ScreenEffects 的归一化值 = 4.0 / 8.0 = **0.5**
> - MovementSpeedBonus = FRENZIED主导 = **+10%**（忽略BROKEN的-5%）
>
> **最终画面表现**：视野严重缩窄（Vignette 0.8）+ 色彩几乎丧失（Saturation 0.3）+ 画面布满噪点（Noise 0.5）+ 准星剧烈抖动（内部值 4.0，ScreenEffects 接收值 0.5）+ 移动速度加快（+10%）

**规则3：跨系统事件响应**

系统监听以下事件并触发计量变化：

| 事件来源 | 事件类型 | 理智变化 | 愤怒变化 |
|---------|---------|---------|---------|
| GrittyTakedowns | `KillTagEvent{kill_tag: ENEMY}` | -5 | +10 |
| GrittyTakedowns | `KillTagEvent{kill_tag: VICTIM}` | -15 | +5 |
| GrittyTakedowns | `KillTagEvent{kill_tag: ACCOMPLICE}` | -8 | +8 |
| GrittyTakedowns | `KillTagEvent{kill_tag: UNKNOWN}` | -15 | +5 |
| GrittyTakedowns | `TieUp` (捆绑) | +3 | -2 |
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
| CALM | Rage > 30 AND Sanity < 70 | UNEASY | 开始应用视觉滤镜 |
| CALM | Rage >= 51 AND Sanity <= 39 | AGITATED | 滤镜强度增加 |
| UNEASY | Rage >= 51 AND Sanity <= 39 | AGITATED | — |
| UNEASY | Rage >= 71 AND Sanity <= 30 | SOUL_SPLIT | 极端状态（同时满足BROKEN和FRENZIED条件） |
| AGITATED | Rage >= 71 AND Sanity <= 30 | SOUL_SPLIT | 极端状态（同时满足BROKEN和FRENZIED条件） |
| SOUL_SPLIT | Rage < 71 AND Sanity <= 30 | BROKEN | Rage降至71以下但Sanity仍<=30时（Sanity<40满足BROKEN的Sanity条件） |
| SOUL_SPLIT | Rage < 71 AND Sanity > 30 | AGITATED | Rage降至71以下且Sanity>30时不再满足SOUL_SPLIT条件 |
| SOUL_SPLIT | Rage >= 71 AND Sanity > 30 | AGITATED或FRENZIED | Sanity恢复到31以上（不再满足SOUL_SPLIT的Sanity条件），根据Rage值判定 |
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
| **战斗系统 (Combat System)** | `IsInCombat` 布尔值 | 愤怒消散机制的判断依据：`IsInCombat == true` 时不消散，`IsInCombat == false` 时每3秒消散-2 |
| 玩家控制器 | 时间流逝（用于消散计时器） | 愤怒消散的时间基准，每3秒触发一次消散判定 |

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
| 特工 | 0.8x | 完全习惯暴力，道德束缚最少 |
| 雇佣兵 | 1.15x | 最不适应内心黑暗，冲动失控倾向高 |

> **集成说明**：`SanityPenaltyMultiplier` 在公式最后一位与最终乘积相乘。例如：特工击杀帮凶（基准惩罚 -8）→ `-8 × 1.0 × 1.0 × 1.0 × 0.8 = -6.4`

| 事件类型 | BaseValue | ContextMultiplier | TimeMultiplier | 说明 |
|---------|-----------|-------------------|----------------|------|
| 击杀恶徒 | -5 | 适用 | 适用 | 道德上可接受，但仍造成心理负担 |
| 击杀帮凶 | -8 | 适用 | 适用 | 知道他们是受害者但仍下手 |
| 击杀无辜者 | -15 | 适用 | 适用 | 严重的道德错误 |
| 目睹NPC死亡 | -5 | 适用 | 适用 | 非亲手，但仍受影响 |
| 发现悲剧线索（第一次） | -5 | 不适用 | 不适用 | 首次接触悲剧，情感冲击 |
| 发现悲剧线索（后续） | -3 | 不适用 | 不适用 | 递减惩罚，避免过度惩罚探索 |
| 发现悲剧线索（后续深度揭示） | -2 | 不适用 | 不适用 | 多次接触同一悲剧的边际递减 |
| 发现身份/位置线索 | +5 | 不适用 | 不适用 | 真相带来满足感 |
| 捆绑NPC（不杀） | +3 | 不适用 | 不适用 | 仁慈选择 |
| 成功转化线人 | +5 | 不适用 | 不适用 | 正向干预 |

> **ContextMultiplier 和 TimeMultiplier 适用范围说明**：这两个乘数**仅适用于战斗/暴力类事件**（击杀/目睹死亡），用于放大连续暴力行为的心理冲击。线索发现类事件（发现悲剧线索、身份/位置线索）和仁慈行为（捆绑、转化）**不适用 ContextMultiplier 和 TimeMultiplier**，其理智变化仅由 BaseValue 决定（线索类事件额外乘以 NarrativeSignificanceMultiplier）。

**ContextMultiplier**（用于战斗/暴力类事件）：

| 情境类型 | 乘数值 | 说明 |
|---------|--------|------|
| 首次击杀 | ×1.0 | 基准心理压力 |
| 连续击杀（5秒内） | ×1.5 | 累积心理压力 |
| 击杀后被目击 | ×1.2 | 被发现带来的额外压力 |

**TimeMultiplier**：

| 时间类型 | 乘数值 | 说明 |
|---------|--------|------|
| 首次发现 | ×1.0 | 基准 |
| 5秒内连续事件 | ×1.5 | 高频事件放大冲击 |
| 30秒后再次发现 | ×0.8 | 边际递减 |

**NarrativeSignificanceMultiplier**（用于线索类事件，来自线索系统）：
- 普通线索（NORMAL）：×1.0
- 与主要目标相关（MAIN_TARGET）：×1.5
- 揭示 NPC 同情的一面（NPC_SYMPATHY）：×2.0

**悲剧线索惩罚递减机制说明**：
- **按"同一悲剧"计算，非按"总悲剧数"计算**：惩罚递减针对的是玩家对**同一悲剧背景**的多次发现，而非全地图所有悲剧线索的总数
- 首次发现该悲剧线索：完整惩罚（-5）
- 后续发现同一悲剧的其他线索：递减惩罚（-3）
- 多次深入揭示同一悲剧背景：最低惩罚（-2）
- 设计意图：玩家应该被**鼓励探索**，而不是因为收集线索而受到过度惩罚。同一个悲剧背景的第一次接触情感冲击最强烈，后续递减；但玩家探索**不同**悲剧背景时仍会受到完整惩罚。

> **⚠️ Playtest 验证需求**：当前惩罚曲线假设每任务平均有 3-5 条悲剧线索。如果实际关卡设计中悲剧线索数量超出此范围，可能导致理智惩罚累积过快。需要在上线前通过 playtest 验证曲线合理性。

> **【Balance 风险标注】** 每任务 3-5 条悲剧线索的假设可能导致理智惩罚累积过快。当前设计假设玩家在探索完整关卡时会遇到约 3-5 条悲剧线索，首次发现 -5、后续 -3、深度揭示 -2。如果关卡设计中悲剧线索数量超过 5 条，或玩家采取高探索率路线，可能导致 Sanity 下降速度超过自然恢复速度（救赎行为 +5-15），造成玩家心理压力过大而影响游戏体验。
>
> **Playtest 验证计划**：
> - **验证时机**：Alpha 阶段专项 Playtest
> - **对照组设计**：
>   - 低探索率组：只完成核心任务目标，不主动搜索悲剧线索
>   - 中探索率组：完成核心任务 + 偶尔探索周围环境
>   - 高探索率组：完成核心任务 + 积极搜索所有悲剧线索
> - **关键指标**：
>   - 任务结束时 Sanity 是否低于 30
>   - 玩家是否在任务中期感到画面效果影响正常游戏体验
> - **调整预案**：如 >30% 高探索率玩家在任务结束时 Sanity<20，启动**方案1**（救赎恢复值 +3）
>
> **建议**：
> - 在 Alpha 阶段进行专项 playtest 验证悲剧线索密度
> - 根据 playtest 结果调整 `TragedyClueSanityPenalty_First`（当前 -5）或衰减速率
> - 备选方案：增加救赎行为的理智恢复值或频率

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

**MomentumMultiplier**：
- 快速连续击杀（<3秒间隔）：×1.3
- 10秒内无击杀：×0.8（冷却衰减）

**IsInCombat 布尔值来源说明**：

`IsInCombat` 由 **战斗系统 (Combat System)** 设置，用于判断玩家当前是否处于战斗状态：
- 玩家进入战斗范围并与敌人交互时，Combat System 设置 `IsInCombat = true`
- 玩家脱离战斗（离开敌人范围 + 保持一定时间无交互）时，Combat System 设置 `IsInCombat = false`
- 战斗状态的判定逻辑由 Combat System 全权负责，Sanity/Rage 系统仅消费此布尔值

> **Fallback 行为说明**：在 **MVP 阶段**（Combat System 未实现前），`IsInCombat` 的默认值为 `false`。愤怒消散机制在无 Combat System 的情况下正常工作（每3秒消散-2）。当玩家进入战斗范围时，需要临时补丁逻辑手动设置 `IsInCombat = true`。

**消散与积累的互斥机制**：
- **非战斗状态下**：当 `IsInCombat == false` 时，愤怒每3秒自动消散（-2）
- **战斗状态下**：当 `IsInCombat == true` 时，愤怒**不消散**（保持当前值）
- **消散期间进入战斗**：如果玩家在消散倒计时执行中进入战斗（`IsInCombat` 从 false 变为 true），**立即停止消散**，等待下次消散判定时由于 `IsInCombat == true` 而跳过消散
- **设计意图澄清**：消散机制**不禁止**玩家在消散期间击杀敌人。玩家可以随时击杀，但：
  - 战斗期间（`IsInCombat == true`）击杀后不会触发消散（因为战斗状态下不执行消散判定）
  - 消散期间击杀会导致 `IsInCombat` 变为 true，但消散本身是"被跳过"而非"立即停止"（逻辑上等价，但更符合实现）
  - 这种设计的目的是让玩家在**主动选择战斗**时不会受到消散惩罚，同时保持"狂暴"状态的紧张感

**消散判定公式**（每3秒执行一次）：
```
if (IsInCombat == false):
    Rage = Rage - RageDecayAmount  # 默认 -2
else:
    Rage 保持不变（战斗状态下不消散，跳过本次判定）
```

**狂暴阈值锁定机制 (Frenzy Threshold Lock)**：
- 当 Rage 首次超过 70（进入 FRENZIED 状态）时，触发阈值锁定
- 锁定持续时间内，即使 Rage 正常消散，也**不会低于 70**
- 锁定持续时间：`FrenzyLockDuration`（默认 10 秒）
- **锁定结束后，Rage 恢复正常消散行为**：锁定解除后，如果 `IsInCombat == false`，Rage 每 3 秒自动 -2，直到降至 70 以下才退出 FRENZIED 状态
- 设计意图：玩家一旦进入"狂暴"状态，应该能够**维持这种状态一段时间**，体验完整的狂暴快感，而不是刚进入就立刻消散
- **击杀时愤怒变化公式**：
  ```
  RageDelta = BaseValue[EventType] * MomentumMultiplier
  ```
  - 正常积累：MomentumMultiplier = 1.0 或 1.3（快速连续击杀）
  - 战斗状态下击杀：MomentumMultiplier = 1.0（直接累加；战斗状态下不消散，愤怒持续累加）
  - 非战斗状态下击杀：MomentumMultiplier = 1.0（直接累加；击杀后进入战斗状态，后续消散判定被跳过）

---

**公式3：视觉效果插值**

```
VignetteIntensity = Lerp(0.0, 0.8, 1.0 - Sanity/100)
NoiseIntensity = Lerp(0.0, 0.5, 1.0 - Sanity/50)  # 当Sanity<50时开始
SaturationMultiplier = Lerp(0.3, 1.0, Sanity/100)
ShakeIntensity_Internal = Lerp(0.0, 8.0, Rage/100)  # 内部值，范围 0.0~8.0
ShakeRequest.intensity = ShakeIntensity_Internal / ShakeMaxIntensity  # 归一化后发送给 ScreenEffects，范围 0.0~1.0（ShakeMaxIntensity 为 Tuning Knobs 参数，默认 8.0）
MovementSpeedBonus = Lerp(0.0, 0.1, Rage/100)  # 最高+10%

> **Crouch + FRENZIED 速度叠加规则**：
> - Crouch 状态：`StateMultiplier = 0.5`（来自 Player Controller）
> - FRENZIED 状态：`MovementSpeedBonus = +10%`（上限）
> - **叠加计算**：`FinalSpeed = BaseSpeed × CrouchMultiplier × (1.0 + MovementSpeedBonus)`
>   - 示例：Crouch + FRENZIED = 5.0 × 0.5 × 1.1 = **2.75 m/s**（FRENZIED 的+10%在Crouch后叠加）
> - 设计意图：FRENZIED 的速度加成是为了补偿潜行时观察者被暴露的风险，所以即使在Crouch状态下仍然生效
```

> **ShakeIntensity 归一化说明**：`ShakeIntensity_Internal` 是本系统的内部调参值（0.0~8.0），用于 Tuning Knobs 调参和叠加计算。向 ScreenEffects 发送 `ShakeRequest` 时，必须先除以 `ShakeMaxIntensity`（默认 8.0）归一化到 0.0~1.0，ScreenEffects 的接收范围上限为 1.0。

---

**公式4：状态阈值判定**

```
EffectiveFrenzyThreshold = BaseFrenzyThreshold + FrenzyThresholdModifier

// 其中：BaseFrenzyThreshold = 70（由本系统定义）
//       FrenzyThresholdModifier 来自 Character Background 系统（普通人=+8, 特工=±0, 雇佣兵=-8）

// 状态判定（按优先级从高到低检查）
// 【重要】SOUL_SPLIT 必须在 FRENZIED 之前判定！
// 因为 Rage=85, Sanity=25 满足 71-100 AND 0-30，如果不优先判定会被误判为 BROKEN
// SOUL_SPLIT 判定：Rage >= 71 AND Sanity <= 30
// 与表格中 Rage: 71-100、Sanity: 0-30 表述一致（范围表述 vs 条件表述）
if Rage >= 71 AND Sanity <= 30:
    State = SOUL_SPLIT       // 特殊状态：同时满足狂暴(Rage 71-100)和崩溃(Sanity 0-30)条件
elif Rage > 90:
    State = FRENZIED         // 视觉优先级最高（即使满足SOUL_SPLIT条件也显示此状态）
elif Sanity <= 30:
    State = BROKEN
elif Rage > 50 AND Sanity <= 39 AND Rage <= EffectiveFrenzyThreshold:
    // AGITATED: Sanity 31-39 AND Rage 51-70
    // 注：Sanity 20-30 由 BROKEN 优先处理（见公式3），AGITATED 实际有效范围为 31-39
    // Rage > 50 (即 51+) 与表格 Rage 51-70 一致
    // Rage <= EffectiveFrenzyThreshold 确保在 AGITATED 范围内（超过阈值则进入 FRENZIED）
    State = AGITATED
elif Rage > 30 OR Sanity < 70:
    State = UNEASY
else:
    State = CALM
```

> **状态与视觉分离说明**：
> - `State` 变量记录玩家实际的心理状态，用于成就系统、叙事flag、系统间数据传递
> - `VisualState` 用于决定视觉显示优先级，规则为：
>   - 如果 State = SOUL_SPLIT 且 **Rage > 90**，则 VisualState = FRENZIED（视觉优先显示愤怒状态）
>   - 如果 State = SOUL_SPLIT 且 **71 ≤ Rage ≤ 90**，则 VisualState = AGITATED（狂暴视觉效果受限于 Rage 未超阈值）
>   - 否则 VisualState = State
> - 设计意图：玩家触发 SOUL_SPLIT 时仍可获得特殊成就/moral flag，但视觉表现取决于 Rage 是否超过 90（超过时以 FRENZIED 为主，未超过时以 AGITATED 为主）

> **FrenzyThresholdModifier 集成说明**：`FrenzyThresholdModifier` 由 Character Background 系统提供（见 Dependencies 章节）。本公式使用 EffectiveFrenzyThreshold 替代硬编码的 70 阈值，实现背景对狂暴触发时机的个性化调整。
>
> **示例**：雇佣兵（`FrenzyThresholdModifier = -8`）的有效狂暴阈值为 `70 + (-8) = 62`，即 Rage > 62 时即进入 AGITATED 状态，Rage > 90 时进入 FRENZIED。普通人（`FrenzyThresholdModifier = +8`）的有效狂暴阈值为 `70 + 8 = 78`，需要更高的愤怒积累才会触发狂暴状态。

---

## 5. Edge Cases

**边缘情况1：理智和愤怒同时达到极端值（SOUL_SPLIT）**

问题1：玩家同时处于极低理智（<=30）和极高愤怒（71-100）状态，触发 SOUL_SPLIT 条件。

问题2：**SOUL_SPLIT 边界条件明确（闭区间判定）**：
> **关键阈值**：SOUL_SPLIT 的触发条件是 `Rage >= 71 AND Sanity <= 30`。
> - **Rage=70, Sanity=30** 时：Rage=70 **不满足** `Rage >= 71` 条件，因此 **不触发 SOUL_SPLIT**。此时 Sanity <= 30，满足 `Sanity <= 30` 条件，状态判定为 **BROKEN**
> - **Rage=71, Sanity=31** 时：Rage=71 满足 `Rage >= 71`，但 Sanity=31 **不满足** `Sanity <= 30`，因此 **不触发 SOUL_SPLIT**。此时进入 AGITATED 状态（Rage=71 > EffectiveFrenzyThreshold=70，且 Sanity=31 > 30）
> - **Rage=71, Sanity=30** 时：同时满足 `Rage >= 71` 和 `Sanity <= 30`，**触发 SOUL_SPLIT**

> **设计说明**：SOUL_SPLIT 的边界是**闭区间 [71, 100] for Rage** 和**闭区间 [0, 30] for Sanity**。 Rage 必须 >= 71，Sanity 必须 <= 30。Sanity=31 是一个关键阈值——达到此值时玩家已"走出"崩溃的临界区，不再满足 SOUL_SPLIT 条件。

处理：
- **状态判定**：State = SOUL_SPLIT（特殊成就/moral flag）
- **视觉显示**：
  - 当 **Rage > 90** 时：VisualState = FRENZIED（愤怒视觉优先级更高）
  - 当 **71 ≤ Rage ≤ 90** 时：VisualState = AGITATED（狂暴视觉效果受限于 Rage 未超阈值）
- **视觉效果**：同时应用 FRENZIED 和 BROKEN 的效果（见叠加计算示例）
- **特殊成就**：触发"灵魂分裂"成就或设置叙事flag（用于区分普通 FRENZIED/BROKEN 与极端状态）
- **设计意图**：SOUL_SPLIT 是玩家"彻底失控"的表现——既疯狂又崩溃，这种状态值得一个特殊标记。Rage 超过 90 时视觉以 FRENZIED 为主（最强烈的愤怒表现）；Rage 在 71-90 区间时视觉以 AGITATED 为主（狂暴尚未完全爆发）

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

**边缘情况5：战斗状态切换与消散行为**

问题1：玩家脱离战斗后，愤怒开始消散（每3秒-2）。此时如果玩家重新进入战斗（`IsInCombat` 变为 true），会发生什么？

处理：
- 消散判定**每3秒执行一次**（基于计时器，不基于帧）
- 当消散判定执行时，如果 `IsInCombat == true`，本次判定**跳过消散**
- 当消散判定执行时，如果 `IsInCombat == false`，执行消散（-2）
- **设计结果**：玩家脱离战斗后，如果立即重新进入战斗（<3秒内），可以"保护"愤怒值不被消散，因为下一次消散判定时 `IsInCombat` 已变为 true

问题2：愤怒消散期间玩家击杀，愤怒是否"不抵消"？

处理：
- 击杀时愤怒**直接累加**到当前值
- 如果此时 `IsInCombat == false`，击杀后进入战斗，`IsInCombat` 变为 true，**下一次消散判定会被跳过**
- 如果此时 `IsInCombat == true`（已在战斗中），击杀不会改变消散行为（战斗状态本就不消散）
- **设计意图**：不是"防止边消边杀"，而是"战斗状态下不消散"。玩家可以通过选择继续战斗来保持愤怒值，但无法通过"故意消散-击杀-再消散"的循环来刷愤怒

---

**边缘情况6：目睹自己造成的死亡（回头看）**

问题：玩家击杀NPC后，NPC尸体在视野范围内。

处理：
- 尸体停留期间，该NPC的击杀效果**不重复触发**
- 离开视野范围后再次进入，**不重复触发**
- 同一击杀类型只触发一次计量变化

---

## 6. Dependencies

### 依赖关系矩阵

> **术语说明**：
> - **上游依赖（系统依赖谁）**：本系统调用谁获取数据，或谁向本系统提供数据
> - **下游依赖（谁依赖本系统）**：本系统的输出发送给谁，即谁消费本系统的数据

**数据提供方（上游）**：

| 系统 | 依赖类型 | 接口说明 |
|------|---------|----------|
| GrittyTakedowns | 硬依赖 | 监听 `KillTagEvent` 事件获取击杀数据 |
| **叙事系统 (Narrative System)** | 硬依赖 | 接收 `SanityRecoveryEvent`（理智恢复事件：埋葬受害者、找到家人、仁慈行为、悲剧解锁）|
| Clue&Journal | 硬依赖 | 监听 `ClueDiscoveredEvent` 获取线索发现 |
| Health&Lethality | 软依赖 | 监听 `PlayerDamaged` 事件获取受伤数据 |
| 玩家控制器 | 软依赖 | 获取时间流逝，用于愤怒自然消散 |
| **战斗系统 (Combat System)** | **硬依赖** | **接收 `IsInCombat` 布尔值**。愤怒消散机制的判断依据：`IsInCombat == true` 时不消散，`IsInCombat == false` 时每3秒消散-2。<br>• **接口签名**：`IsInCombat: bool`（只读，本系统消费此值）<br>• **MVP 占位方案**：`IsInCombat` 默认值为 `false`。<br>• **Combat System 实现后**：由战斗系统设置（进入敌人范围并交互时 `true`，脱离战斗时 `false`）|

**数据消费方（下游）**：

| 系统 | 依赖类型 | 接口说明 |
|------|---------|----------|
| ScreenEffects | 硬依赖 | 发送视觉效果请求（暗角/噪点/饱和度/抖动） |
| DynamicPostProcessing | 硬依赖（单向） | 发送心理状态枚举，供后处理着色器使用。本系统为数据提供方，DPP为下游消费者——DPP依赖本系统，但本系统不依赖DPP即可独立运作（可降级运行） |
| 玩家控制器 | 软依赖 | 发送移动速度乘数 |
| **Character Background** | 软依赖（查询） | Character Background 系统调用 `IFrenzyThresholdModifierProvider` 接口查询狂暴阈值调整值 |
| **Weather System** | 软依赖（订阅） | 订阅 `LightningFlashEvent`。本系统根据当前心理状态自行决定是否降低闪电强度——这是单向数据流：Weather System 发送事件，本系统消费并自行处理，不存在循环依赖 |

### FrenzyThresholdModifier 接口定义

> 本接口由 Character Background 系统调用，用于查询当前玩家的狂暴阈值调整值。

```csharp
/// <summary>
/// 提供狂暴阈值调整值给背景系统
/// </summary>
public interface IFrenzyThresholdModifierProvider {
    /// <summary>
    /// 返回狂暴阈值调整值
    /// </summary>
    /// <returns>
    /// 普通人 = +8（更难触发狂暴）
    /// 特工 = 0（基准值）
    /// 雇佣兵 = -8（更容易触发狂暴）
    /// </returns>
    int GetFrenzyThresholdModifier();
}
```

**接口签名**：`GetFrenzyThresholdModifier(): int`

**返回值定义**：

| 背景类型 | 返回值 | 效果 |
|---------|-------|------|
| 普通人 | +8 | 有效狂暴阈值 = 70 + 8 = 78（需要更高愤怒才能触发） |
| 特工 | 0 | 有效狂暴阈值 = 70 + 0 = 70（基准值） |
| 雇佣兵 | -8 | 有效狂暴阈值 = 70 - 8 = 62（更容易触发） |

**数据流向**：Sanity/Rage 系统（提供方）→ Character Background 系统（消费方）

### 数据提供方（上游系统提供的数据）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|----------|
| **主角背景角色系统 (Character Background)** | 软依赖（查询） | **上游依赖（提供参数）**。Character Background 系统输出 `SanityPenaltyMultiplier` 和 `FrenzyThresholdModifier`，本系统作为消费方查询这两个参数用于理智惩罚计算和狂暴状态判定。<br>• 数据流向：Character Background → Sanity/Rage Meter（查询模式，非硬依赖）<br>• `SanityPenaltyMultiplier`：理智惩罚系数（普通人=0.9x, 特工=0.8x, 雇佣兵=1.15x）<br>• `FrenzyThresholdModifier`：狂暴阈值调整（普通人=+8, 特工=±0, 雇佣兵=-8）|

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
| `ShakeMaxIntensity` | float | 8.0 | 4.0~15.0 | 内部最大准星抖动强度（调参用语义值）；发送给 ScreenEffects 时归一化：ShakeRequest.intensity = ShakeIntensity_Internal / ShakeMaxIntensity，归一化后最大值固定为 1.0 |
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
| AC-3 | 愤怒每3秒自动-2 | 积累愤怒后等待，观察 `rage_changed` 事件 |
| AC-4 | 愤怒消散期间再次击杀，立即停止消散 | 积累愤怒后，在消散过程中执行击杀，验证数值不降反升 |
| AC-5 | 理智低于20时，视觉噪点强度 > 0.3 | 模拟低理智状态，使用ScreenEffects接口读取噪声值 |
| AC-6 | 愤怒等于70时，内部 ShakeIntensity_Internal = 5.6（对应发送给 ScreenEffects 的 ShakeRequest.intensity = 0.7）；愤怒高于70时，内部值 > 5.6，ScreenEffects值 > 0.7 | 模拟高愤怒状态（Rage=70），验证：①内部值 ShakeIntensity_Internal = 5.6；②监听事件总线确认 ShakeRequest.intensity = 0.7 |
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
| 2026-04-13 | 0.4 | 修复公式4状态判定逻辑矛盾：调整判定顺序确保 SOUL_SPLIT 条件（Rage 71-100 AND Sanity 0-19）能被正确识别；引入 VisualState 概念区分"状态flag"与"视觉显示"；更新状态优先级说明和边缘情况1的描述 | Claude Code |
| 2026-04-13 | 0.5 | 跨系统 Bug 修复（ShakeIntensity 范围不一致）：公式3 将 ShakeIntensity 拆分为内部值（0.0~8.0）和归一化发送值（除以8.0），补充归一化说明；更新叠加计算示例注明内部值与归一化值的对应关系；修复 AC-6 验收条件（内部值 > 4.0 / ScreenEffects 接收值 > 0.5）；Tuning Knobs ShakeMaxIntensity 补充归一化说明 | Game Designer Agent |
| 2026-04-13 | 0.6 | 修复 P1 问题：统一 SOUL_SPLIT 状态范围描述格式（表格添加脚注说明范围表述与条件表述一致，公式4注释明确两种表述的对应关系） | Claude Code |
| 2026-04-14 | 0.7 | 设计评审修复：<br>1. P1 修复：补充 SOUL_SPLIT 边界条件说明（Rage=70,Sanity=19 不满足SOUL_SPLIT，判定为BROKEN；Rage=71,Sanity=20 同样不满足SOUL_SPLIT，判定为AGITATED）<br>2. P1 修复：补充 IsInCombat 布尔值来源说明（来自战斗系统 Combat System），更新数据流入表格和使用者说明<br>3. P2 修复：澄清消散机制设计意图 - 战斗状态下不消散（非禁止边消边杀，而是让选择战斗的玩家保持愤怒积累）；重写边缘情况5描述，区分"消散期间进入战斗"与"边消边杀"的概念 | Claude Code |
| 2026-04-14 | 0.8 | **P1 修复**：Dependencies 章节补充战斗系统 (Combat System) 硬依赖声明，`IsInCombat` 布尔值接口说明和 MVP 占位方案；**P1 修复**：边缘情况1中 SOUL_SPLIT 边界条件添加"闭区间判定"和"关键阈值"强调说明，明确 Rage=70, Sanity=30 的判定结果 | Claude Code |
