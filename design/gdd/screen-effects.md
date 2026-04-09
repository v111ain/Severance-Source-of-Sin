# 屏幕特效系统 (Screen Effects)

> **Status**: Infrastructure
> **Author**: Technical Director
> **Created**: 2026-04-08
> **Last Updated**: 2026-04-08
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
| 屏幕震动 | ShakeIntensity | 0.0 ~ 10.0 | 震动强度 |
| 色调偏移 | HueShift | -30° ~ +30° | 整体色调偏移 |
| 色调强度 | HueIntensity | 0.0 ~ 1.0 | 色调偏移的应用强度 |

**规则2：特效请求格式**

所有特效通过事件总线接收请求：

```
ScreenEffectRequest:
    effect_type: Enum          # VIGNETTE / SATURATION / NOISE / SHAKE / HUE_SHIFT
    intensity: Float           # 目标强度值
    transition_duration: Float  # 过渡时间（秒）
    priority: Int              # 优先级，高优先级可打断低优先级
```

**规则3：特效叠加规则**

当多个系统同时请求同一特效时：
1. 取所有请求中**最高优先级**的值
2. 同优先级时，取**最新**请求的值
3. 所有过渡使用 **ease-in-out** 插值

**规则4：特效组合预设**

为简化调用，部分常用组合提供预设接口：

| 预设名 | Vignette | Saturation | Noise | Shake |
|--------|----------|------------|-------|-------|
| SANITY_CALM | 0.0 | 1.0 | 0.0 | 0.0 |
| SANITY_BROKEN | 0.8 | 0.3 | 0.5 | 0.0 |
| RAGE_FRENZIED | 0.6 | 0.8 | 0.0 | 8.0 |
| PLAYER_HURT | 0.3 | 0.9 | 0.1 | 3.0 |
| PLAYER_DEAD | 1.0 | 0.0 | 0.8 | 5.0 |

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
CurrentValue = Lerp(TargetValue, CurrentValue, ExpDecay(TimeSinceChange, HalfLife))
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| HalfLife | 0.2s | 过渡半衰期 |

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
| `ShakeRequest` | Sanity/Rage | 愤怒 > 70% | ShakeIntensity | 愤怒值映射（0.0~8.0） | 0.05s | 高 |
| `HueShiftRequest` | DPP | 心理状态变化 | HueShift + HueIntensity | 按状态变化值 | 0.2s | 低 |
| `ScreenEffectPreset` | Health | 玩家受伤 | PLAYER_HURT 预设 | Vignette=0.3, Saturation=0.9, Noise=0.1, Shake=3.0 | 0.15s | 高 |
| `ScreenEffectPreset` | Health | 玩家死亡 | PLAYER_DEAD 预设 | Vignette=1.0, Saturation=0.0, Noise=0.8, Shake=5.0 | 0.5s | 最高 |

**预设特效组合（完整定义）**：

| 预设名 | Vignette | Saturation | Noise | Shake | HueShift | HueIntensity |
|--------|----------|------------|-------|-------|----------|--------------|
| SANITY_CALM | 0.0 | 1.0 | 0.0 | 0.0 | 0° | 0.0 |
| SANITY_BROKEN | 0.8 | 0.3 | 0.5 | 0.0 | 0° | 0.0 |
| RAGE_FRENZIED | 0.6 | 0.8 | 0.0 | 8.0 | +15° | 0.3 |
| PLAYER_HURT | 0.3 | 0.9 | 0.1 | 3.0 | 0° | 0.0 |
| PLAYER_DEAD | 1.0 | 0.0 | 0.8 | 5.0 | -10° | 0.5 |

#### 本系统发出的事件

| 事件名 | 方向 | 负载 | 说明 |
|--------|------|------|------|
| `ScreenEffectChanged` | → 事件总线 | `{effect_type, current_value}` | 特效值已更新 |

## Formulas

### 插值过渡公式

```
CurrentValue = Lerp(TargetValue, CurrentValue, ExpDecay(TimeSinceChange, HalfLife))

Lerp(a, b, t) = a * (1 - t) + b * t
ExpDecay(t, half_life) = 0.5 ^ (t / half_life)
```

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
| `MaxVignetteIntensity` | float | 0.9 | 0.5 ~ 1.0 | 暗角上限（保留视野） |
| `MaxShakeIntensity` | float | 10.0 | 5.0 ~ 15.0 | 震动上限 |
| `MinSaturationForPlayability` | float | 0.3 | 0.2 ~ 0.5 | 保证可玩性的最低饱和度 |

### 调参风险提示

- `MaxVignetteIntensity` 设置过高 → 画面太暗，影响游戏可玩性
- `MinSaturationForPlayability` 设置过低 → 玩家难以区分敌人和环境
- `ShakeTransitionHalfLife` 设置过长 → 震动响应迟钝，失去反馈感

## Visual/Audio Requirements

### 技术实现

本系统的视觉特效基于 Unity Post-Processing Stack v2 或 Volume Framework 实现：

| 特效 | Unity 组件 | 配置方式 |
|------|----------|---------|
| Vignette | Bloom / Post-processing | 通过脚本动态调整 intensity 参数 |
| Saturation | Color Adjustments | 通过脚本动态调整 saturation 参数 |
| Noise | Film Grain / Dithering | 通过脚本动态调整 intensity 参数 |
| Shake | 屏幕空间位移 shader | 通过脚本动态调整抖动幅度 |

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
| OQ-1 | 具体使用 Post-processing Stack v2 还是 Volume Framework？ | 技术美术 | 影响实现方式 |
| OQ-2 | 震动效果是否需要方向性（如左下角震动 vs 全屏震动）？ | 游戏设计师 | 影响 shader 设计 |

---

## Change Log

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-08 | 0.1 | 初稿创建 | Technical Director Agent |
