# 屏幕特效系统 (Screen Effects)

> **Status**: Infrastructure
> **Author**: Technical Director
> **Created**: 2026-04-08
> **Last Updated**: 2026-04-13
> **Priority**: Infrastructure
> **Layer**: Infrastructure
> **Depends On**: 无 (基础层)
> **Related Systems**: Dynamic Post-Processing, Immersive Audio & Haptics, Sanity/Rage Meter

## Overview

屏幕特效系统 (Screen Effects) 是游戏的基础视觉特效基础设施，负责管理所有屏幕级别的视觉效果（暗角、噪点、饱和度、屏幕震动）。本系统被多个上层系统共享调用，是连接游戏逻辑与视觉反馈的核心桥梁。

**核心职责**：
- 提供基础屏幕特效接口供其他系统调用
- 管理特效参数的插值过渡
- 与 Unity 后处理管线集成

**设计原则**：
- 本系统仅负责基础特效的**施加**，不包含业务逻辑
- 特效的**触发条件**由调用方系统（Sanity/Rage、DPP、Audio）决定
- 所有特效支持平滑过渡，避免突兀的视觉跳变

## Player Fantasy

屏幕特效是玩家"感受"游戏心理状态的直接通道。当玩家愤怒时，准星抖动；当理智崩溃时，视野缩窄——这些效果不是UI装饰，而是玩家内心状态的外化投射。

本系统确保这些视觉反馈**平滑、可控、可调**，让玩家在极端心理状态下仍能清晰感知周遭世界。

## Detailed Design

### Core Rules

**规则1：特效类型**

本系统管理以下基础屏幕特效：

| 特效类型 | 参数名 | 范围 | 说明 |
|---------|--------|------|------|
| 暗角 | VignetteIntensity | 0.0 ~ 1.0 | 边缘暗化程度 |
| 饱和度 | SaturationMultiplier | 0.0 ~ 1.0 | 色彩饱和度（0=灰度） |
| 噪点 | NoiseIntensity | 0.0 ~ 1.0 | 画面噪点强度 |
| 屏幕震动 | ShakeIntensity | 0.0 ~ 1.0 | 震动强度（0.0=无震动，0.5=中等，1.0=最高震动） |

**ShakeIntensity 物理意义说明**：本系统仅负责 0.0~1.0 范围内的归一化插值和上限钳制（MaxShakeIntensity = 1.0）。ShakeIntensity 的实际物理意义（如震动幅度、频率）由请求方系统（Sanity/Rage）解释和映射，本系统不进行额外解释。

> **调用方归一化义务**：所有向本系统发送 `ShakeRequest` 的系统，必须在发送前将自身内部值归一化到 0.0~1.0 范围。具体规则：
> - **Sanity/Rage Meter**：内部 `ShakeIntensity = Lerp(0.0, 8.0, Rage/100)`，发送时需除以 8.0：`ShakeRequest.intensity = ShakeIntensity / 8.0`
> - **DPP 系统**：内部调参值 `FrenziedShake = 8.0`、`SoulSplitShake = 4.0`，发送时同样除以 8.0
> - 本系统对超出 1.0 的值会自动 clamp 并记录警告日志，但**不负责归一化转换**，该职责由调用方承担
| 色调偏移 | HueShift | -30° ~ +30° | 整体色调偏移 |
| 色调强度 | HueIntensity | 0.0 ~ 1.0 | 色调偏移的应用强度 |

**规则2：特效请求格式**

所有特效通过事件总线接收请求：

```
ScreenEffectRequest:
    effect_type: Enum          # VIGNETTE / SATURATION / NOISE / SHAKE / HUE_SHIFT / HUE_INTENSITY
    intensity: Float           # 目标强度值
    transition_duration: Float  # 过渡时间（秒）
    priority: Int              # 优先级，高优先级可打断低优先级
```

> **effect_type 枚举说明**：
> - `VIGNETTE`：暗角效果
> - `SATURATION`：饱和度
> - `NOISE`：噪点
> - `SHAKE`：屏幕震动
> - `HUE_SHIFT`：色调偏移
> - `HUE_INTENSITY`：色调强度（色调偏移的应用程度）

**规则3：特效叠加规则**

当多个系统同时请求同一特效时：
1. 取所有请求中**最高优先级**的值
2. 同优先级时，取**最新**请求的值
3. 所有过渡使用 **ease-in-out** 插值

**时间戳判定机制说明**：
当多个请求具有相同优先级时，系统通过请求事件携带的 `timestamp`（事件发送时间）判定"最新"：
- 每个 `ScreenEffectRequest` 事件包含事件发送时的时间戳字段 `timestamp`
- 系统维护一个 `current_timestamp` 变量，记录当前生效请求的时间戳
- 当裁决同优先级请求时，比较各请求的 `timestamp`，取最大值（最新）
- 新请求的 `timestamp` 必须大于当前 `current_timestamp` 才能打断当前特效
- 如果新请求的 `timestamp` 小于等于 `current_timestamp`，则忽略该请求（防止乱序事件导致的问题）
- `timestamp` 使用游戏内统一时钟（Game Time），确保跨系统事件顺序一致

**规则4：特效组合预设**

为简化调用，部分常用组合提供预设接口：

| 预设名 | Vignette | Saturation | Noise | Shake | HueShift | HueIntensity |
|--------|----------|------------|-------|-------|----------|--------------|
| SANITY_CALM | 0.0 | 1.0 | 0.0 | 0.0 | 0° | 0.0 |
| SANITY_BROKEN | 0.8 | 0.3 | 0.5 | 0.0 | -20° | 0.0 |
| RAGE_FRENZIED | 0.6 | 0.8 | 0.0 | 1.0 | +15° | 0.3 |
| PLAYER_HURT | 0.3 | 0.9 | 0.1 | 0.6 | 0° | 0.0 |
| PLAYER_DEAD | 0.8 | 0.0 | 0.8 | 1.0 | -10° | 0.5 |
| DPP_UNEASY | 0.2 | 0.9 | 0.0 | 0.0 | -5° | 0.3 |
| DPP_AGITATED | 0.4 | 0.75 | 0.1 | 0.0 | -10° | 0.5 |
| DPP_SOUL_SPLIT | 0.8 | 0.3 | 0.3 | 0.5 | +15° | 0.6 |

> **与 DPP 系统预设值对齐说明**：
> - DPP_UNEASY/DPP_AGITATED/DPP_SOUL_SPLIT 预设值与 dynamic-post-processing.md 附录B调色板参数保持一致
> - DPP_SOUL_SPLIT 的 Shake=0.5 为归一化值（内部值4.0除以8.0），与 DPP 系统 SoulSplitShake=4.0 归一化后为 0.5 的逻辑一致
> - 愤怒状态（RAGE_FRENZIED）对应 DPP 的 FRENZIED 状态：DPP 内部 FrenziedShake=8.0 归一化为 1.0（对应 Rage>90% 的 RageIntensity），因此本预设 Shake 值应设为 1.0 与 DPP 保持一致

> **RAGE_FRENZIED 预设说明**：
> - 本预设是"纯愤怒状态（FRENZIED）"的视觉风格定义，代表愤怒主导（高 Rage）时的视觉特征
> - 实际运行时，FRENZIED 状态的视觉参数由 Sanity/Rage 系统根据公式3动态计算并通过 `ScreenEffectRequest` 发送
> - 本预设定义的是 FRENZIED 状态的**视觉风格上限**（血红色暗角、强烈抖动、色调偏移），但实际值随 Sanity 和 Rage 动态变化
> - **与 Sanity/Rage 公式的关系**：Vignette 和 Saturation 主要由 Sanity 驱动（低 Sanity → 高 Vignette、低 Saturation），但 FRENZIED 状态下 Rage 主导时，这些值可能与公式计算结果不同；Shake 由 Rage 直接驱动，与公式一致
> - 设计意图：确保"纯愤怒"与"低理智崩溃"有明确的视觉区分 — 前者以红色调和高抖动为主，后者以灰暗色调和噪点为主
> - **Shake=1.0 说明**：RAGE_FRENZIED 的 Shake 值为 1.0，这是归一化值（对应 DPP 系统内部值 8.0 除以 8.0 归一化）。此预设代表 FRENZIED 状态下的最大抖动强度，与 DPP 系统 `FrenziedShake = 8.0` 归一化为 1.0 的逻辑一致

## States and Transitions

**特效状态机**

```
[Idle] ──收到请求──▶ [Transitioning] ──过渡完成──▶ [Active]
      ▲                              │
      └─────────收到新请求────────────┘
```

| 状态 | 描述 | 可转移至 |
|------|------|---------|
| Idle | 无特效应用，使用默认值 | Transitioning（收到请求） |
| Transitioning | 正在过渡到目标值 | Active（过渡完成）/ Transitioning（收到新请求） |
| Active | 特效已稳定应用 | Transitioning（收到新请求） |

**过渡插值公式**

```
Alpha = 1.0 - ExpDecay(TimeSinceChange, HalfLife)
CurrentValue = Lerp(CurrentValue, TargetValue, Alpha)
```

**语义说明**：
- `Alpha`：过渡完成度系数，从0.0渐变到1.0
- 当 `TimeSinceChange = 0` 时，`Alpha = 0.0`，`CurrentValue = Lerp(CurrentValue, TargetValue, 0.0) = CurrentValue`（无过渡，保持当前值）
- 当 `TimeSinceChange → ∞` 时，`Alpha → 1.0`，`CurrentValue = Lerp(CurrentValue, TargetValue, 1.0) = TargetValue`（完全过渡到目标值）
- 过渡曲线：初期变化快（视觉效果明显），后期变化慢（收敛稳定）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| HalfLife | 0.2s | 过渡半衰期（此处为 Vignette 参考值；各特效有独立 HalfLife，详见 Tuning Knobs 的 `*TransitionHalfLife` 参数组；与 `DefaultTransitionDuration` 的数值关系见 Formulas 章节） |

## Interactions with Other Systems

### 上游依赖（本系统依赖谁）

无。本系统是基础设施层，不依赖任何其他系统。

### 下游依赖（谁依赖本系统）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **Sanity/Rage 系统** | 硬依赖 | 接收 VignetteRequest、NoiseRequest、SaturationRequest、ShakeRequest |
| **Dynamic Post-Processing** | 硬依赖 | 接收色调偏移和强度请求 |
| **Immersive Audio & Haptics** | 软依赖 | 震动触发时同步调用屏幕震动（通过事件总线） |
| **Health & Lethality** | 软依赖 | 玩家受伤/死亡时触发预设 |
| **UI 系统** | 软依赖 | 专注模式/HUD 反馈时接收 ShakeRequest 和 VignetteRequest |

### 事件接口定义

#### 本系统订阅的事件

| 事件名 | 来源 | 用途 |
|--------|------|------|
| `VignetteRequest` | Sanity/Rage 系统 | 理智低时增加边缘暗角 |
| `SaturationRequest` | Sanity/Rage 系统 | 理智低时降低色彩饱和度 |
| `NoiseRequest` | Sanity/Rage 系统 | 理智极低时增加视觉噪点 |
| `ShakeRequest` | Sanity/Rage 系统 | 愤怒高时增加准星抖动 |
| `HueShiftRequest` | DPP 系统 | 心理状态变化时的色调偏移 |
| `ScreenEffectPreset` | Health 系统 | 玩家受伤/死亡时触发预设 |

### Effects Triggers Matrix（事件→视觉效果映射表）

| 触发事件 | 来源系统 | 触发条件 | 应用特效 | 目标值 | 过渡时长 | 优先级 |
|---------|---------|---------|---------|-------|---------|-------|
| `VignetteRequest` | Sanity/Rage | 理智 < 50% | Vignette | 随理智下降递增（0.0~0.8） | 0.2s | 中 |
| `VignetteRequest` | Sanity/Rage | 理智 < 20% | Vignette | 0.8 | 0.2s | 高 |
| `SaturationRequest` | Sanity/Rage | 理智 < 50% | SaturationMultiplier | 随理智下降递减（1.0~0.3） | 0.3s | 中 |
| `SaturationRequest` | Sanity/Rage | 理智 < 20% | SaturationMultiplier | 0.3 | 0.3s | 高 |
| `NoiseRequest` | Sanity/Rage | 理智 < 30% | NoiseIntensity | 随理智下降递增（0.0~0.5） | 0.1s | 中 |
| `ShakeRequest` | Sanity/Rage | 愤怒 > 70%（AGITATED 阈值，FRENZIED 阈值为 90%） | ShakeIntensity | 愤怒值映射（0.0~1.0，0.5=中等强度）。阈值映射见下方脚注¹ | 0.05s | 高 |
| `HueShiftRequest` | DPP | 心理状态变化 | HueShift + HueIntensity | 按状态变化值 | 0.2s | 低 |
| `ScreenEffectPreset` | DPP | DPP_UNEASY | Vignette=0.2, Saturation=0.9, Noise=0.0, HueShift=-5°, HueIntensity=0.3 | 0.3s | 低 |
| `ScreenEffectPreset` | DPP | DPP_AGITATED | Vignette=0.4, Saturation=0.75, Noise=0.1, HueShift=-10°, HueIntensity=0.5 | 0.3s | 低 |
| `ScreenEffectPreset` | DPP | DPP_SOUL_SPLIT | Vignette=0.8, Saturation=0.3, Noise=0.3, Shake=0.5, HueShift=+15°, HueIntensity=0.6 | 0.5s | 高 |
| `ScreenEffectPreset` | Health | 玩家受伤 | PLAYER_HURT 预设 | Vignette=0.3, Saturation=0.9, Noise=0.1, Shake=0.6 | 0.15s | 高 |
| `ScreenEffectPreset` | Health | 玩家死亡 | PLAYER_DEAD 预设 | Vignette=0.8, Saturation=0.0, Noise=0.8, Shake=1.0（注意：Vignette受MaxVignetteIntensity=0.8上限约束） | 0.5s | 最高 |

**脚注¹：ShakeRequest 阈值映射说明**
- Sanity/Rage 系统发送的 `ShakeRequest.intensity` 已是归一化值（0.0~1.0），由 `ShakeIntensity_Internal / 8.0` 计算得出
- 愤怒 70% 时 → intensity = 0.7（Lerp(0.0, 8.0, 0.7)/8.0 = 5.6/8.0），对应 **AGITATED** 状态
- 愤怒 90% 时 → intensity ≈ 0.9（Lerp(0.0, 8.0, 0.9)/8.0 = 7.2/8.0），对应 **FRENZIED** 状态
- 愤怒 100% 时 → intensity = 1.0（Lerp(0.0, 8.0, 1.0)/8.0 = 8.0/8.0）

**预设特效组合（完整定义）**：

| 预设名 | Vignette | Saturation | Noise | Shake | HueShift | HueIntensity |
|--------|----------|------------|-------|-------|----------|--------------|
| SANITY_CALM | 0.0 | 1.0 | 0.0 | 0.0 | 0° | 0.0 |
| SANITY_BROKEN | 0.8 | 0.3 | 0.5 | 0.0 | -20° | 0.0 |
| RAGE_FRENZIED | 0.6 | 0.8 | 0.0 | 1.0 | +15° | 0.3 |
| PLAYER_HURT | 0.3 | 0.9 | 0.1 | 0.6 | 0° | 0.0 |
| PLAYER_DEAD | 0.8 | 0.0 | 0.8 | 1.0 | -10° | 0.5 |
| DPP_UNEASY | 0.2 | 0.9 | 0.0 | 0.0 | -5° | 0.3 |
| DPP_AGITATED | 0.4 | 0.75 | 0.1 | 0.0 | -10° | 0.5 |
| DPP_SOUL_SPLIT | 0.8 | 0.3 | 0.3 | 0.5 | +15° | 0.6 |

> **DPP_SOUL_SPLIT Shake=0.5 说明**：SOUL_SPLIT 状态叠加了 BROKEN（高暗角+噪点）和 FRENZIED（抖动+色彩偏移）效果。为防止视觉过载，DPP_SOUL_SPLIT 的 Shake 设为 0.5 而非 FRENZIED 的 1.0。这是经过设计的折衷方案，详见 `sanity-rage-meter.md` 的"视觉效果叠加规则"。

> **RAGE_FRENZIED 预设说明**：
> - 本预设是"纯愤怒状态（FRENZIED）"的视觉风格定义，代表愤怒主导（高 Rage）时的视觉特征
> - 实际运行时，FRENZIED 状态的视觉参数由 Sanity/Rage 系统根据公式3动态计算并通过 `ScreenEffectRequest` 发送
> - 本预设定义的是 FRENZIED 状态的**视觉风格上限**（血红色暗角、强烈抖动、色调偏移），但实际值随 Sanity 和 Rage 动态变化
> - **与 Sanity/Rage 公式的关系**：Vignette 和 Saturation 主要由 Sanity 驱动（低 Sanity → 高 Vignette、低 Saturation），但 FRENZIED 状态下 Rage 主导时，这些值可能与公式计算结果不同；Shake 由 Rage 直接驱动，与公式一致
> - 设计意图：确保"纯愤怒"与"低理智崩溃"有明确的视觉区分 — 前者以红色调和高抖动为主，后者以灰暗色调和噪点为主
> - **Shake=1.0 说明**：RAGE_FRENZIED 的 Shake 值为 1.0，这是归一化值（对应 DPP 系统内部值 8.0 除以 8.0 归一化）。此预设代表 FRENZIED 状态下的最大抖动强度，与 DPP 系统 `FrenziedShake = 8.0` 归一化为 1.0 的逻辑一致

#### 本系统发出的事件

| 事件名 | 方向 | 负载 | 说明 |
|--------|------|------|------|
| `ScreenEffectChanged` | → 事件总线 | `{effect_type, current_value}` | 特效值已更新 |

## Formulas

### 插值过渡公式

```
CurrentValue = Lerp(CurrentValue, TargetValue, 1.0 - ExpDecay(TimeSinceChange, HalfLife))

Lerp(a, b, t) = a * (1 - t) + b * t
ExpDecay(t, half_life) = 0.5 ^ (t / half_life)
```

### 参数语义说明：DefaultTransitionDuration 与 HalfLife

两个参数在层次和用途上有明确区分：

**`HalfLife`（半衰期）**
- 用途：ExpDecay 公式的内部技术参数，控制每帧插值的衰减速率
- 作用域：各特效拥有独立的 HalfLife，在 Tuning Knobs 中以 `*TransitionHalfLife` 形式暴露
- 示例值：VignetteTransitionHalfLife = 0.2s，SaturationTransitionHalfLife = 0.3s

**`DefaultTransitionDuration`（默认过渡时长）**
- 用途：面向策划的请求参数，作为 `ScreenEffectRequest.transition_duration` 字段未指定时的兜底默认值
- 作用域：系统级全局参数，不直接参与 ExpDecay 计算

**数值关系（以 Vignette 为基准）**

```
DefaultTransitionDuration = 1.5 × VignetteTransitionHalfLife
                          = 1.5 × 0.2s = 0.3s
```

| 时间点 | 相当于几个半衰期 | 过渡进度（残余差距） | 效果 |
|--------|----------------|---------------------|------|
| 0.2s（1× HalfLife） | 1.0 | 50% 差距剩余 | 视觉上变化明显 |
| 0.3s（1.5× HalfLife） | 1.5 | ~35% 差距剩余 | DefaultTransitionDuration 基准点 |
| 0.6s（3× HalfLife） | 3.0 | ~12.5% 差距剩余 | 视觉上接近收敛 |
| 0.86s（4.32× HalfLife） | 4.32 | ~5% 差距剩余 | 数学上的 95% 完成点 |

> **调参配比原则**：调整 HalfLife 时建议同步维持 `DefaultTransitionDuration ≈ 1.5 × VignetteTransitionHalfLife` 的比例关系，确保设计意图一致。若各特效 HalfLife 差异很大，`DefaultTransitionDuration` 应对应主视觉特效（Vignette）的节奏，其余特效以各自 HalfLife 为主。

### 默认值定义

| 特效 | 默认值 | 说明 |
|------|-------|------|
| VignetteIntensity | 0.0 | 无暗角 |
| SaturationMultiplier | 1.0 | 全饱和度 |
| NoiseIntensity | 0.0 | 无噪点 |
| ShakeIntensity | 0.0 | 无震动 |
| HueShift | 0° | 无色调偏移 |
| HueIntensity | 0.0 | 不应用色调偏移 |

## Edge Cases

### 边缘情况1：收到冲突请求

**问题**：多个系统同时请求不同值的同一特效

**处理**：
- 优先级裁决：高优先级打断低优先级
- 同优先级：最新请求打断旧请求
- 过渡过程中收到新请求，从当前过渡位置开始新过渡

### 边缘情况2：特效值超出范围

**问题**：请求的特效值超出 [0.0, 1.0] 范围

**处理**：
- 自动 clamp 到安全范围内
- 记录警告日志

### 边缘情况3：过渡被打断

**问题**：过渡过程中收到新请求

**处理**：
- 中断当前过渡
- 从当前值开始新过渡
- 无需等待当前过渡完成

## Tuning Knobs

*以下参数暴露给策划在引擎 Inspector 中直接调整，无需修改代码。*

### 过渡参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `DefaultTransitionDuration` | float | 0.3s | 0.1s ~ 1.0s | 默认过渡时间 |
| `VignetteTransitionHalfLife` | float | 0.2s | 0.1s ~ 0.5s | 暗角过渡半衰期 |
| `SaturationTransitionHalfLife` | float | 0.3s | 0.1s ~ 0.5s | 饱和度过渡半衰期 |
| `NoiseTransitionHalfLife` | float | 0.1s | 0.05s ~ 0.3s | 噪点过渡半衰期 |
| `ShakeTransitionHalfLife` | float | 0.05s | 0.02s ~ 0.2s | 震动过渡半衰期（响应更快） |

### 安全限制

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MaxVignetteIntensity` | float | 0.8 | 0.5 ~ 1.0 | 暗角上限（保留视野，与DPP系统MaxVignetteCap保持一致） |
| `MaxShakeIntensity` | float | 1.0 | 0.5 ~ 1.0 | 震动上限（归一化范围，与请求方解释无关） |
| `MinSaturationForPlayability` | float | 0.3 | 0.2 ~ 0.5 | 保证可玩性的最低饱和度 |

### 调参风险提示

- `MaxVignetteIntensity` 设置过高 → 画面太暗，影响游戏可玩性
- `MinSaturationForPlayability` 设置过低 → 玩家难以区分敌人和环境
- `ShakeTransitionHalfLife` 设置过长 → 震动响应迟钝，失去反馈感

## Visual/Audio Requirements

### 技术实现

本系统的视觉特效基于 **Unity Volume Framework (URP)** 实现：

| 特效 | Unity 组件 | 配置方式 |
|------|----------|---------|
| Vignette | Bloom / Post-processing | 通过脚本动态调整 intensity 参数 |
| Saturation | Color Adjustments | 通过脚本动态调整 saturation 参数 |
| Noise | Film Grain / Dithering | 通过脚本动态调整 intensity 参数 |
| Shake | 屏幕空间位移 shader | 通过脚本动态调整抖动幅度和方向 |

**震动 shader 方向性参数接口**：
```
ShakeDirection:
    direction: Vector2       // 震动方向向量（归一化），(1, 0) = 右侧，(0, 1) = 上方
    direction_intensity: float // 方向性强度（0.0 ~ 1.0），0.0 = 全方向随机震动，1.0 = 严格按direction震动
    angular_variance: float    // 角度随机范围（度），例如 45 表示 direction ± 22.5° 内随机
```

**震动方向计算规则**：
- 根据伤害来源方向确定震动方向：
  - 伤害来自左方 → 屏幕向右上方抖动（direction = (0.7, 0.7)）
  - 伤害来自右方 → 屏幕向左上方抖动（direction = (-0.7, 0.7)）
  - 伤害来自正前方 → 屏幕向上抖动（direction = (0, 1)）
  - 伤害来自正后方 → 屏幕向下抖动（direction = (0, -1)）
  - 伤害来自斜方向 → 按比例混合方向

### 性能预算

| 指标 | 预算 | 说明 |
|------|------|------|
| 单帧特效更新耗时 | < 0.5ms | 不影响游戏帧率 |
| GPU 特效成本 | < 2ms | 所有特效叠加 |
| 内存占用 | < 1MB | 参数数据 |

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 系统能接收 VignetteRequest 并平滑过渡到目标值 | 发送请求，观察视觉变化 |
| AC-2 | 系统能接收多个竞争请求并正确裁决 | 同时发送高低优先级请求，验证行为 |
| AC-3 | 过渡过程中收到新请求能立即切换 | 发送请求A，在过渡中发送请求B，验证连续性 |
| AC-4 | 所有特效值自动 clamp 到安全范围 | 发送超出范围的值，验证系统不崩溃 |
| AC-5 | 特效预设能正确触发组合效果 | 调用 PLAYER_HURT 预设，验证所有参数 |

### 性能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-6 | 单帧更新不影响帧率（< 0.5ms） | 性能监测工具 |
| AC-7 | 所有特效同时激活时 GPU 占用 < 2ms | 帧时间分析 |

### 集成验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-8 | Sanity/Rage 系统能正确触发视觉特效 | 模拟低理智状态，验证视觉反馈 |
| AC-9 | Audio 系统震动同步时屏幕震动正确 | 触发爆炸音效，验证同步震动 |

## Open Questions

| # | 问题 | 负责人 | 说明 |
|---|------|--------|------|
| **OQ-1（已解决）** | ~~具体使用 Post-processing Stack v2 还是 Volume Framework？~~ | ~~技术美术~~ | ✅ **已于 2026-04-14 解决**：统一采用 **Volume Framework (URP)** 作为主要实现方案，与 Dynamic Post-Processing 系统保持一致。 |
| **OQ-2（已解决）** | ~~震动效果是否需要方向性（如左下角震动 vs 全屏震动）？~~ | ~~游戏设计师~~ | ✅ **已于 2026-04-13 解决**：震动效果采用 **基于伤害来源方向计算** 的方向性设计。当玩家受伤时，震动方向根据伤害来源方向确定：<br>- 伤害来自左方 → 屏幕向右上方抖动<br>- 伤害来自右方 → 屏幕向左上方抖动<br>- 伤害来自正前方 → 屏幕向上抖动<br>- 伤害来自正后方 → 屏幕向下抖动<br>- 伤害来自斜方向 → 按比例混合方向<br>震动 shader 需支持屏幕空间位移向量输入，实现方向性抖动效果。 |

---

## Change Log

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-08 | 0.1 | 初稿创建 | Technical Director Agent |
| 2026-04-13 | 0.2 | 修复 ShakeIntensity 范围冲突：统一为 0.0~1.0 归一化范围，删除 Effects Triggers Matrix 中的 0.0~8.0 描述，调整 Tuning Knobs 安全范围为 0.5~1.0，更新预设组合中 PLAYER_HURT/PLAYER_DEAD/RAGE_FRENZIED 的 Shake 值，添加物理意义说明 | Technical Director Agent |
| 2026-04-13 | 0.3 | 跨系统 Bug 修复：在 ShakeIntensity 物理意义说明中补充「调用方归一化义务」——明确 Sanity/Rage Meter 和 DPP 系统在发送 ShakeRequest 前须将内部值除以 8.0 归一化到 0.0~1.0，本系统不承担转换职责 | Game Designer Agent |
| 2026-04-13 | 0.4 | P0 修复：明确 DefaultTransitionDuration（0.3s）与 HalfLife（0.2s）的语义关系。在 Formulas 章节新增「参数语义说明」小节，说明两者层次区分（技术参数 vs. 设计兜底值）、默认数值关系（DefaultTransitionDuration = 1.5 × VignetteTransitionHalfLife）及各时间点的过渡进度表；在 States and Transitions 的 HalfLife 参数表中补充跨引用，消除孤立 0.2s 导致的误读风险 | Game Designer Agent |
