# Dynamic Post-Processing System (动态视觉滤镜系统)

> **Status**: Approved
> **Author**: Technical Director + Systems Designer
> **Last Updated**: 2026-04-09
> **Priority**: Alpha
> **Layer**: Presentation
> **Implements Pillar**: 罪恶的深度 (Depth of Sin)
> **Depends On**: Sanity/Rage System, Screen Effects
>
> **Revision Notes (2026-04-08)**:
> - ✅ OQ-1 已解决：渲染方案确定为 Volume Framework (URP)
> - ✅ OQ-2 已解决：补充完整性能预算（GPU成本<2ms、降级策略、各平台目标）
> - ✅ 补充完整的 Edge Cases（6个边缘情况）
> - ✅ 补充完整的 Acceptance Criteria（15个验收标准）
> - ✅ OQ-3 已解决：与场景美术协调方案确立
> - ✅ OQ-4 已解决：调色板参数完整定义
>
> **2026-04-10 设计审查修复**：
> - ✅ P0: DPP 状态阈值直接引用 sanity-rage-meter.md 公式4，不再重复定义，避免不一致
>
> **2026-04-09 设计审查修复**：
> - ✅ DPP 系统 Tuning Knobs 章节补充完成：涵盖状态阈值、过渡动画、滤镜强度、性能降级、可玩性保护、调试参数共6大类、40+参数

## Overview

动态视觉滤镜系统是游戏心理状态的可视化呈现层。它将玩家在"理智/愤怒系统"中累积的心理状态转化为实时变化的视觉滤镜效果，让玩家**感受到**而非仅仅是**看到**自己的精神状态变化。

**与 Screen Effects 的区别**：
- **Screen Effects**：基础屏幕特效（震动、暗角、噪点），由多个系统共享调用
- **Dynamic Post-Processing**：基于玩家心理状态的**整体视觉风格**变化，是 Screen Effects 的上游消费者

---

## Player Fantasy

**"你眼中的世界，取决于你的内心。"**

当玩家处于愤怒的狂战士状态时，世界被血红色浸染；当理智崩溃时，世界褪去色彩变得灰暗。这种视觉变化不是UI装饰，而是玩家心理状态的外化投射。

**参考对标**：
- 《Silent Hill 2》的 fog world——心理状态改变视觉呈现
- 《SOMA》的深海恐惧滤镜——环境反映心理
- 《Control》的 threshold transitions——状态转换时的视觉突变

---

## Detailed Design

### Core Rules

**规则1：滤镜层叠架构**

视觉滤镜由多个滤镜层叠加构成：

| 滤镜层 | 来源 | 说明 |
|--------|------|------|
| Base | 关卡美术 | 场景原始视觉效果 |
| LightingFilter | 光照系统 | 基于区域光照等级的色调和暗角调整 |
| SanityFilter | 理智系统 | 饱和度、锐度、边缘暗角 |
| RageFilter | 愤怒系统 | 红色色调、血丝纹理、颤抖 |
| StateFilter | 当前状态 | CALM/UNEASY/AGITATED/BROKEN/FRENZIED/SOUL_SPLIT |

**规则2：心理状态来源**

> **重要**：DPP 不维护独立的状态阈值。本系统的心理状态**直接引用** `sanity-rage-meter.md` Section 4 公式4 定义的状态判定逻辑。
>
> DPP 系统通过监听 Sanity/Rage 系统广播的 `PsychologicalState` 事件获取当前状态。根据该事件的状态值，DPP 系统应用对应的视觉滤镜。
>
> **状态优先级（来自 sanity-rage-meter.md 公式4）**：
> 1. `Rage > 90` → FRENZIED（愤怒溢出，忽略 Sanity 条件）
> 2. `Sanity < 20` → BROKEN（理智崩溃，Rage 不影响）
> 3. `Rage > 70 OR Sanity < 40` → AGITATED
> 4. `Rage > 30 OR Sanity < 70` → UNEASY
> 5. 默认 → CALM
>
> **SOUL_SPLIT 判定**：当 `Sanity < 20 AND Rage >= 71` 时，Sanity/Rage 系统会发送 `PsychologicalState.SOUL_SPLIT`。DPP 接收到此状态后，应用 BROKEN + FRENZIED 效果叠加（详见规则3）。

**规则3：心理状态到滤镜的映射**

| 状态 | 主导滤镜 | 视觉效果描述 |
|------|---------|-------------|
| CALM | 无 | 正常画面 |
| UNEASY | Sanity 轻 filter | 轻微暗角，饱和度-10% |
| AGITATED | Sanity 中 filter | 中度暗角，饱和度-25%，轻微噪点 |
| BROKEN | Sanity 重 filter | 视野严重缩窄，严重噪点，色彩几乎丧失 |
| FRENZIED | Rage filter | 血红色暗角，准星抖动，色调偏暖 |
| SOUL_SPLIT | Sanity 重 + Rage | BROKEN + FRENZIED 效果叠加 |

**规则4：滤镜插值过渡**

状态切换时的滤镜过渡不是瞬间的，而是通过 Lerp 插值平滑过渡：
- 过渡时间：0.5秒~2.0秒（可配置）
- 插值曲线：ease-in-out

**规则5：光照变化对滤镜的影响**

DPP 系统订阅 `LightingChangedEvent`，根据当前光照等级调整视觉滤镜：

> **优先级说明**：光照滤镜层叠加在心理状态滤镜**之上**，但优先级低于心理状态。当两者冲突时，取心理状态滤镜的值（确保极端心理状态下的视觉效果不被光照覆盖）。

**光照等级到滤镜参数的映射**：

| 光照等级 | 色调调整 | 暗角增量 | 说明 |
|---------|---------|---------|------|
| 明亮 (Bright) | 无 | 无 | 正常画面 |
| 正常 (Normal) | 无 | +0.05 | 轻微暗角 |
| 昏暗 (Dim) | 饱和度 -10% | +0.15 | 降低饱和度，中度暗角 |
| 黑暗 (Dark) | 饱和度 -25% | +0.30 | 严重降低饱和度，强烈暗角 |

> **叠加规则**：
> - 色调调整基于当前心理状态的 `SaturationMultiplier` 乘算
> - 暗角增量在心理状态暗角**基础上**叠加
> - 当心理状态为 BROKEN 或 SOUL_SPLIT 时，**忽略光照暗角增量**（极端心理状态优先）

**闪电曝光效果**：

当收到 `LightningFlashEvent`（来自天气系统）时：
- 全局曝光 +0.5（200ms）
- 色调略微增亮（所有颜色的 Exposure +0.3）
- 不影响暗角

> **与天气系统的协调**：闪电曝光效果与天气系统的 `LightningFlashEvent` 协调。详见 weather-system.md 的"闪电全局曝光"章节。

---

## Formulas

**公式1：滤镜强度插值**

```
FilterIntensity = Lerp(TargetIntensity, CurrentIntensity, ExpDecay(TimeSinceChange, HalfLife))
```

| 状态 | TargetIntensity | 说明 |
|------|----------------|------|
| CALM | 0.0 | 无特效 |
| UNEASY | 0.3 | 轻微 |
| AGITATED | 0.6 | 中等 |
| BROKEN | 1.0 | 极端 |
| FRENZIED | 0.8 | 强 |
| SOUL_SPLIT | 1.0 | 最大值 |

**公式2：色调偏移**

| 状态 | TargetHueShift |
|------|---------------|
| CALM | 0° |
| UNEASY | -5° (轻微冷色调) |
| AGITATED | -10° |
| BROKEN | -20° (灰冷) |
| FRENZIED | +15° (暖红) |

---

## Edge Cases

### 边缘情况1：多个状态同时满足条件

**问题**：玩家同时满足 AGITATED 和 FRENZIED 的触发条件。

**处理**：
- 状态优先级裁决：FRENZIED > BROKEN > AGITATED > UNEASY > CALM
- 使用最高优先级的状态作为主导状态
- 视觉效果叠加时，取所有适用效果的最大值

### 边缘情况2：快速状态切换（乒乓效应）

**问题**：玩家在临界阈值附近反复横跳（如 Rage 在 69-71 之间波动），导致状态频繁切换。

**处理**：
- 使用**迟滞阈值**（Hysteresis）：进入状态需要一个阈值，退出需要另一个更低的阈值
- 示例：进入 FRENZIED 需要 Rage > 90，退出需要 Rage < 75

### 边缘情况3：SOUL_SPLIT 状态的视觉效果过载

**问题**：BROKEN + FRENZIED 效果叠加可能造成画面过于混乱，影响可玩性。

**处理**：
- 在 SOUL_SPLIT 状态下，自动启用**可玩性保护机制**
- 降低噪点强度上限（0.3 而非 0.5）
- 保证最低饱和度（0.2 而非 0.1）
- 添加"模糊优先级"：准星抖动 > 色调 > 噪点 > 暗角

### 边缘情况4：Volume Framework 性能降级

**问题**：在低性能设备上，多个 Volume 层叠加导致帧率下降。

**处理**：
- 检测帧率，如果持续低于 30fps，自动降级到单一 Global Volume
- 合并多个滤镜层为预设组合（使用 Texture LUT）
- 提供"性能模式"选项：只保留暗角和饱和度，禁用噪点和色调偏移

### 边缘情况5：状态切换时的视觉跳变

**问题**：即使使用了插值过渡，某些状态之间的切换仍然显得突兀。

**处理**：
- 相邻状态之间使用 0.5 秒过渡
- 极端状态切换（CALM → FRENZIED）使用 1.5 秒过渡
- 使用**过渡曲线**（而非线性插值）：ease-in-out-cubic

### 边缘情况6：游戏暂停时的状态处理

**问题**：游戏暂停时，滤镜状态应该如何处理？

**处理**：
- 暂停时冻结当前滤镜状态
- 恢复时从冻结状态继续插值过渡（而非重置）
- 可选：暂停时显示特殊的"心理状态"UI（如呼吸/心跳可视化）

---

## Dependencies

### 上游依赖

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 理智/愤怒系统 | 硬依赖 | 接收 `PsychologicalState` 枚举（CALM/UNEASY/AGITATED/BROKEN/FRENZIED/SOUL_SPLIT），详见 `sanity-rage-meter.md` 状态机定义 |
| Screen Effects | 硬依赖 | 调用暗角/噪点/色调等基础特效 |
| **光照系统 (Lighting System)** | 软依赖（订阅） | 订阅 `LightingChangedEvent`，根据当前光照等级调整视觉滤镜参数（色调、暗角强度） |

### Screen Effects 接口映射

DPP 系统通过以下接口调用 Screen Effects 系统：

| DPP 需求 | Screen Effects 接口 | 参数范围 | 验证状态 |
|----------|-------------------|---------|---------|
| 饱和度调整 | `SaturationMultiplier` | 0.0 ~ 1.0 | ✅ 兼容 |
| 暗角效果 | `VignetteIntensity` | 0.0 ~ 1.0 | ✅ 兼容 |
| 噪点效果 | `NoiseIntensity` | 0.0 ~ 1.0 | ✅ 兼容 |
| 准星抖动 | `ShakeIntensity` | 0.0 ~ 10.0 | ✅ 兼容 |
| 色调偏移 | `HueShift` | -30° ~ +30° | ✅ 兼容 |
| 色调强度 | `HueIntensity` | 0.0 ~ 1.0 | ✅ 兼容 |

**接口兼容性结论**：Screen Effects 提供的所有接口参数完全满足 DPP 需求，无需扩展或修改。

### 下游依赖

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 无 | — | 本系统是最顶层呈现层 |

---

## Tuning Knobs

### 状态阈值参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `HysteresisEnterRage` | int | 90 | 85~95 | 进入 FRENZIED 状态的 Rage 阈值 |
| `HysteresisExitRage` | int | 75 | 70~80 | 退出 FRENZIED 状态的 Rage 阈值（迟滞） |
| `HysteresisEnterBroken` | int | 20 | 15~25 | 进入 BROKEN 状态的 Sanity 阈值 |
| `HysteresisExitBroken` | int | 35 | 30~40 | 退出 BROKEN 状态的 Sanity 阈值（迟滞） |
| `HysteresisEnterAgitated` | int | 71 | 66~76 | 进入 AGITATED 状态的 Rage 上限/ Sanity 下限阈值 |
| `HysteresisExitAgitated` | int | 55 | 50~65 | 退出 AGITATED 状态的阈值（迟滞） |

### 过渡动画参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `NormalTransitionDuration` | float | 0.5s | 0.3~1.0s | 相邻状态间的过渡时间 |
| `ExtremeTransitionDuration` | float | 1.5s | 1.0~2.0s | 极端状态切换（CALM↔FRENZIED）的过渡时间 |
| `TransitionCurve` | enum | EaseInOutCubic | Linear/EaseIn/EaseOut/EaseInOut/EaseInOutCubic | 过渡曲线类型 |

### 滤镜强度参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `CalmVignette` | float | 0.0 | 0.0~0.2 | CALM 状态暗角强度 |
| `UneasyVignette` | float | 0.2 | 0.1~0.3 | UNEASY 状态暗角强度 |
| `AgitatedVignette` | float | 0.4 | 0.3~0.5 | AGITATED 状态暗角强度 |
| `BrokenVignette` | float | 0.8 | 0.6~0.8 | BROKEN 状态暗角强度 |
| `FrenziedVignette` | float | 0.6 | 0.5~0.7 | FRENZIED 状态暗角强度 |
| `SoulSplitVignette` | float | 0.8 | 0.6~0.8 | SOUL_SPLIT 状态暗角强度 |
| `CalmSaturation` | float | 1.0 | 0.8~1.0 | CALM 状态饱和度倍率 |
| `BrokenSaturation` | float | 0.3 | 0.2~0.4 | BROKEN 状态饱和度倍率（下限保护） |
| `FrenziedHueShift` | float | 15° | 10°~20° | FRENZIED 状态色调偏移（暖红） |
| `BrokenHueShift` | float | -20° | -25°~-15° | BROKEN 状态色调偏移（冷灰） |
| `FrenziedShake` | float | 8.0 | 6.0~10.0 | FRENZIED 状态准星抖动强度 |
| `SoulSplitShake` | float | 4.0 | 3.0~5.0 | SOUL_SPLIT 状态准星抖动强度（减半保护） |
| `SoulSplitNoise` | float | 0.3 | 0.2~0.4 | SOUL_SPLIT 状态噪点强度（减半保护） |

### 性能降级参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `DegradationTriggerThreshold` | float | 0.9 | — | 降级触发阈值（目标帧率的 90%） |
| `DegradationLevel1Fps` | float | 54fps (60x0.9) | — | 触发 Level 1 降级的帧率 |
| `DegradationLevel2Fps` | float | 48fps (60x0.8) | — | 触发 Level 2 降级的帧率 |
| `DegradationLevel3Fps` | float | 36fps (60x0.6) | — | 触发 Level 3 降级的帧率 |

### 可玩性保护参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MinSaturationFloor` | float | 0.3 | 0.2~0.4 | 所有状态最低饱和度保护 |
| `MaxVignetteCap` | float | 0.8 | — | 所有状态最大暗角强度（Screen Effects 上限） |
| `VisionCenterPreservation` | float | 0.4 | 0.3~0.5 | 屏幕中心保留视野比例（40%） |
| `HueShiftMin` | float | -20° | — | 色调偏移下限（冷色保护） |
| `HueShiftMax` | float | 15° | — | 色调偏移上限（暖色保护） |

### 调试参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `DebugShowState` | bool | false | — | 是否在屏幕显示当前心理状态 |
| `DebugShowFilterValues` | bool | false | — | 是否显示当前滤镜参数值 |
| `DebugForceState` | enum | null | — | 强制指定状态（覆盖实际状态，用于测试） |
| `DebugDisableDegradation` | bool | false | — | 禁用性能降级（用于测试） |

---

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 系统能接收 Sanity/Rage 系统的 PsychologicalState 并正确映射到对应滤镜 | 模拟各状态输入，验证滤镜效果 |
| AC-2 | 状态切换时滤镜平滑过渡，无跳变 | 在临界值附近触发状态切换，观察过渡 |
| AC-3 | FRENZIED 状态正确显示血红色色调偏移（+15°） | 模拟 Rage > 90，测量色调偏移值 |
| AC-4 | BROKEN 状态正确显示灰冷色调偏移（-20°） | 模拟 Sanity < 20，测量色调偏移值 |
| AC-5 | SOUL_SPLIT 状态正确叠加 BROKEN + FRENZIED 效果 | 模拟 Sanity < 20 AND Rage >= 71，验证叠加效果 |
| AC-6 | 迟滞阈值正确防止乒乓效应 | 在临界值附近反复横跳，验证状态不会频繁切换 |

### 性能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-7 | 所有滤镜叠加时 GPU 占用 < 2ms | 帧时间分析工具 |
| AC-8 | 状态切换时帧率无明显下降（< 3 帧抖动） | 慢动作录制，逐帧分析 |
| AC-9 | 低性能设备上自动降级到性能模式 | 在目标最低配置设备上测试 |

### 集成验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-10 | 与 Sanity/Rage 系统正确同步状态 | 监听事件总线，验证 PsychologicalState 事件 |
| AC-11 | 与 Screen Effects 系统正确协调 | 同时调用时验证特效层叠正确 |
| AC-12 | 游戏暂停/恢复时滤镜状态正确处理 | 触发暂停恢复，验证滤镜行为 |

### 视觉验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-13 | 极端状态下（SOUL_SPLIT）仍保持基本可玩性 | 主观评估画面是否过于混乱 |
| AC-14 | 色调偏移在色域范围内，无色彩溢出 | 使用色彩校准工具测量 |
| AC-15 | 滤镜效果在不同显示器/电视上表现一致 | 在 PS5 和 PC 不同显示设备上测试 |

---

## Open Questions

| # | 问题 | 负责人 | 说明 |
|---|------|--------|------|
| OQ-1（已解决） | **渲染实现方案** | 技术美术 | ✅ **已解决**：使用 **Volume Framework (URP)** 作为主要方案，Post-processing Stack v2 作为兼容回退。详见下方「渲染实现方案」章节。 |
| OQ-2（已解决） | **性能预算** | 性能分析师 | ✅ **已解决**：详见下方「性能预算」章节。 |
| OQ-3（已解决） | **与场景美术的协调** | 关卡美术 | ✅ **已解决**：详见下方「场景美术协调方案」章节。 |
| OQ-4（已解决） | **调色板定义** | 美术设计师 | ✅ **已解决**：详见下方「调色板参数定义」章节。 |
| OQ-5（已解决） | ~~**光照变化事件订阅**~~ | ~~系统设计师~~ | ~~2026-04-30~~ | ✅ **已于 2026-04-10 解决**：在 Dependencies 中添加了光照系统作为软依赖（订阅）。在 Core Rules 规则1中添加了 LightingFilter 滤镜层，在规则5中定义了光照等级到滤镜参数的映射。详见「规则1：滤镜层叠架构」和「规则5：光照变化对滤镜的影响」章节。 |

### 性能预算（OQ-2 已解决）

**决策**：基于 URP Volume Framework 架构，设定以下性能目标：

#### 滤镜层叠 GPU 成本预算

| 滤镜层 | GPU 成本 | 说明 |
|--------|---------|------|
| Base (Color Grading) | < 0.3ms | 基础色调 |
| Sanity Filter (Saturation + Vignette + Film Grain) | < 0.8ms | 理智滤镜三层叠加 |
| Rage Filter (Hue Shift + Vignette) | < 0.5ms | 愤怒滤镜双层叠加 |
| State Override | < 0.4ms | 极端状态完全覆盖 |
| **总计（峰值）** | **< 2.0ms** | 所有层叠加上限 |

#### 性能分级目标

| 平台 | 目标帧率 | 滤镜模式 | 说明 |
|------|---------|---------|------|
| PC (高配) | 60 FPS | 全部滤镜层叠加 | < 2ms GPU 成本 |
| PC (低配) | 45 FPS | 简化模式（仅暗角+饱和度） | < 1ms GPU 成本 |
| PS5 Performance Mode | 60 FPS | 全部滤镜层叠加 | < 2ms GPU 成本 |
| PS5 Quality Mode | 30 FPS | 全部滤镜层叠加 | < 4ms GPU 成本（宽松） |
| PS5 Balanced | 40 FPS | 全部滤镜层叠加 | < 2.5ms GPU 成本 |

#### 降级策略

触发条件：帧率持续低于目标 10% 以上时，自动降级：

| 降级级别 | 触发条件 | 禁用效果 | 预算节省 |
|---------|---------|---------|---------|
| Level 0 (Full) | 帧率达标 | — | — |
| Level 1 (Reduced) | 帧率 < 90% 目标 | 禁用 Film Grain (噪点) | -0.3ms |
| Level 2 (Minimal) | 帧率 < 80% 目标 | 禁用 Hue Shift (色调偏移) | -0.3ms |
| Level 3 (Emergency) | 帧率 < 60% 目标 | 仅保留 Vignette | < 0.5ms |

#### 性能测试方法

| 指标 | 目标 | 测试工具 |
|------|------|---------|
| 单帧 DPP 系统总耗时 | < 2ms | Unity Profiler (Rendering) |
| 状态切换帧率抖动 | < 3 帧 | 帧时间图 |
| 峰值 GPU 占用 | < 15% 总 GPU 时间 | GPU Profiler |
| 内存占用（Volume Profiles） | < 2MB | Unity Profiler |

### 渲染实现方案（OQ-1 已解决）

**决策**：采用 **Volume Framework (URP)** 作为主要渲染方案。

**理由**：
1. Unity 6.3 LTS + URP 是官方推荐的标准配置
2. Volume Framework 支持运行时动态混合多个滤镜层，性能更优
3. 与 Screen Effects 系统的插值机制天然契合
4. 支持 PS5/PC 双平台优化

**回退方案**：如果 URP Volume Framework 在特定平台出现兼容性问题，可回退到 Post-processing Stack v2。

**Volume Framework 配置**：

| Volume Profile | 优先级 | 滤镜组件 |
|---------------|--------|----------|
| GlobalVolume | 0（最低） | Base Color Grading |
| SanityVolume | 1 | Saturation, Vignette, Film Grain |
| RageVolume | 2 | Hue Shift (Red), Vignette (Red) |
| StateOverrideVolume | 3（最高） | 完全覆盖（极端状态） |

**滤镜组件映射**：

| 效果 | URP 组件 | DPP 参数 |
|------|---------|----------|
| 饱和度 | Color Adjustments - Saturation | -100 ~ 0 |
| 暗角 | Vignette - Intensity | 0.0 ~ 0.8 |
| 噪点 | Film Grain - Intensity | 0.0 ~ 0.5 |
| 色调偏移 | Color Adjustments - Hue Shift | -30° ~ +30° |
| 色调强度 | Color Adjustments - Post-exposure | -2 ~ +2 |

---

## Next Steps

1. ✅ 确定渲染实现方案（OQ-1 已解决：Volume Framework URP）
2. ✅ 确定性能预算（OQ-2 已解决）
3. ✅ 与场景美术协调，确保滤镜变化不影响可玩性（OQ-3 已解决）
4. ✅ 定义每个状态的精确调色板参数（OQ-4 已解决）
5. [ ] 创建 Volume Profile 配置原型
6. [ ] 实现状态过渡动画原型
7. [ ] 性能测试和优化

---

## 附录A：场景美术协调方案（OQ-3 已解决）

### 设计约束

为确保滤镜变化不影响关卡可玩性，制定以下设计约束：

**1. 视野保留原则**
- 任意状态下，屏幕中心 40% 区域必须保持足够亮度，确保玩家能清晰看到主要游戏内容
- Vignette 最大强度限制为 0.8（Screen Effects 系统限制），保留 20% 边缘视野

**2. 色彩可辨性原则**
- 最低饱和度（SaturationMultiplier）限制为 0.3，确保玩家能区分敌人（红色）和环境
- 色调偏移（HueShift）限制在 -20°~+15° 范围内，避免色彩溢出

**3. 对比度保障原则**
- 愤怒状态（FRENZIED）色调偏暖时，确保红色敌人与背景仍有足够对比度
- 理智崩溃状态（BROKEN）色调偏冷时，确保阴影中的敌人轮廓可见

### 关卡设计指南

| 状态 | 建议的关卡光照设计 |
|------|-------------------|
| CALM | 标准光照，使用完整色域 |
| UNEASY | 略微降低环境光照（-10%），允许轻微氛围暗角 |
| AGITATED | 降低环境光照（-25%），增加局部高光 |
| BROKEN | 最低环境光照（-40%），使用高对比度聚光灯引导玩家 |
| FRENZIED | 暖色调环境光（+10%），血红暗角由 DPP 添加 |
| SOUL_SPLIT | 结合 BROKEN 和 FRENZIED 的光照设计 |

### 美术资源要求

- 所有关键视觉元素（敌人轮廓、交互物件）必须有明确的边缘高光
- 避免使用与状态色调冲突的环境色（如 BROKEN 状态避免使用蓝绿色环境）
- 紧急出口/安全区域使用与状态色调对比强烈的颜色标记

---

## 附录B：调色板参数定义（OQ-4 已解决）

### 完整色调映射表

| 状态 | HueShift | HueIntensity | SaturationMultiplier | VignetteIntensity | NoiseIntensity | ShakeIntensity |
|------|----------|--------------|---------------------|-------------------|----------------|-----------------|
| **CALM** | 0° | 0.0 | 1.0 | 0.0 | 0.0 | 0.0 |
| **UNEASY** | -5° | 0.3 | 0.9 | 0.2 | 0.0 | 0.0 |
| **AGITATED** | -10° | 0.5 | 0.75 | 0.4 | 0.1 | 0.0 |
| **BROKEN** | -20° | 0.7 | 0.3 | 0.8 | 0.5 | 0.0 |
| **FRENZIED** | +15° | 0.6 | 0.8 | 0.6 | 0.0 | 8.0 |
| **SOUL_SPLIT** | +15° | 0.6 | 0.3 | 0.8 | 0.3 | 4.0 |

**说明**：
- SOUL_SPLIT 状态下噪声强度上限为 0.3（而非 0.5），确保可玩性
- SOUL_SPLIT 状态下准星抖动减半（4.0 而非 8.0），避免过度混乱

### 色调偏移实现说明

**HueShift 参数映射到 Screen Effects**：
```
ScreenEffects.ReceiveRequest(
    effect_type: HUE_SHIFT,
    intensity: TargetHueShift / 30.0,  // 归一化到 0.0~1.0
    transition_duration: 0.5~1.5s,
    priority: 10  // 高优先级
)
```

**HueIntensity 参数映射到 Screen Effects**：
```
ScreenEffects.ReceiveRequest(
    effect_type: HUE_INTENSITY,
    intensity: HueIntensity,  // 0.0~1.0 直接使用
    transition_duration: 0.5~1.5s,
    priority: 10
)
```

### 预设组合（用于 URP Volume Profile）

**CALM 预设**：
- Saturation: 0 (无变化)
- Vignette: 0.0
- Film Grain: 0.0
- Hue Shift: 0°
- Post-exposure: 0

**UNEASY 预设**：
- Saturation: -4.5 (轻微降低)
- Vignette: 0.15
- Film Grain: 0.0
- Hue Shift: -5°
- Post-exposure: -0.2

**AGITATED 预设**：
- Saturation: -11.25 (中等降低)
- Vignette: 0.3
- Film Grain: 0.08
- Hue Shift: -10°
- Post-exposure: -0.4

**BROKEN 预设**：
- Saturation: -31.5 (严重降低)
- Vignette: 0.6
- Film Grain: 0.4
- Hue Shift: -20°
- Post-exposure: -0.8

**FRENZIED 预设**：
- Saturation: -9 (轻微降低，保留一定色彩)
- Vignette: 0.5 (血红暗角)
- Film Grain: 0.0
- Hue Shift: +15°
- Post-exposure: +0.3 (增亮以补偿暗角)

### 调色板 RGB 参考值（美术参考）

| 状态 | 主色调 | Hex | 用途 |
|------|--------|-----|------|
| CALM | 中性 | #FFFFFF | 正常画面基准 |
| UNEASY | 冷灰蓝 | #B8C4D0 | 轻微不安感 |
| AGITATED | 灰蓝 | #8A9AAB | 中度压抑 |
| BROKEN | 冷灰 | #6B7A8A | 视野缩窄，色彩丧失 |
| FRENZIED | 暖血红 | #E63946 | 愤怒浸染 |
| SOUL_SPLIT | 暗红灰 | #8B4A4A | 极端状态叠加 |

**Unity URP Color Adjustments 配置示例**：
```csharp
// BROKEN 状态配置
colorAdjustments.saturation = -31.5f;  // 严重去饱和
colorAdjustments.hueShift = -20f;       // 色调偏冷

// FRENZIED 状态配置
colorAdjustments.saturation = -9f;      // 轻微去饱和
colorAdjustments.hueShift = 15f;        // 暖红偏移
colorAdjustments.postExposure = 0.3f;    // 增亮补偿暗角
```
