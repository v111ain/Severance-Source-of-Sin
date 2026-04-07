# Dynamic Post-Processing System (动态视觉滤镜系统)

> **Status**: Draft
> **Author**: [待指定]
> **Last Updated**: 2026-04-07
> **Priority**: Alpha
> **Layer**: Presentation
> **Implements Pillar**: 罪恶的深度 (Depth of Sin)
> **Depends On**: Sanity/Rage System, Screen Effects

## Overview

动态视觉滤镜系统是游戏心理状态的可视化呈现层。它将玩家在"理智/愤怒系统"中累积的心理状态转化为实时变化的视觉滤镜效果，让玩家**感受到**而非仅仅是**看到**自己的精神状态变化。

**与 Screen Effects 的区别**：
- **Screen Effects**：基础屏幕特效（震动、暗角、噪点），由多个系统共享调用
- **Dynamic Post-Processing**：基于玩家心理状态的**整体视觉风格**变化，是 Screen Effects 的上游消费者

## Status: Not Started

本文档为初稿，需要在 Alpha 阶段完善。

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
| SanityFilter | 理智系统 | 饱和度、锐度、边缘暗角 |
| RageFilter | 愤怒系统 | 红色色调、血丝纹理、颤抖 |
| StateFilter | 当前状态 | CALM/UNEASY/AGITATED/BROKEN/FRENZIED |

**规则2：心理状态到滤镜的映射**

| 状态 | 主导滤镜 | 视觉效果描述 |
|------|---------|-------------|
| CALM (70-100 Sanity, 0-30 Rage) | 无 | 正常画面 |
| UNEASY (40-69 Sanity, 31-50 Rage) | Sanity轻 filter | 轻微暗角，饱和度-10% |
| AGITATED (20-39 Sanity, 51-70 Rage) | Sanity中 filter | 中度暗角，饱和度-25%，轻微噪点 |
| BROKEN (0-19 Sanity, <71 Rage) | Sanity重 filter | 视野严重缩窄，严重噪点，色彩几乎丧失 |
| FRENZIED (<70 Sanity, 71-100 Rage) | Rage filter | 血红色暗角，准星抖动，色调偏暖 |
| SOUL_SPLIT (0-19 Sanity, 71-100 Rage) | Sanity重+Rage | BROKEN + FRENZIED 效果叠加 |

**规则3：滤镜插值过渡**

状态切换时的滤镜过渡不是瞬间的，而是通过 Lerp 插值平滑过渡：
- 过渡时间：0.5秒~2.0秒（可配置）
- 插值曲线：ease-in-out

---

## Formulas

**公式1：综合滤镜强度计算**

```
FinalFilterIntensity = SanityFilterIntensity * SanityWeight + RageFilterIntensity * RageWeight
```

其中权重基于当前状态动态调整。

**公式2：色调偏移**

```
HueShift = Lerp(0, TargetHueShift, FilterTransitionProgress)
```

| 状态 | TargetHueShift |
|------|---------------|
| CALM | 0° |
| UNEASY | -5° (轻微冷色调) |
| AGITATED | -10° |
| BROKEN | -20° (灰冷) |
| FRENZIED | +15° (暖红) |

---

## Dependencies

### 上游依赖

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 理智/愤怒系统 | 硬依赖 | 接收 `PsychologicalState` 枚举，获取当前心理状态 |
| Screen Effects | 硬依赖 | 调用暗角/噪点/色调等基础特效 |

### 下游依赖

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 无 | — | 本系统是最顶层呈现层 |

---

## Open Questions

| # | 问题 | 负责人 | 说明 |
|---|------|--------|------|
| OQ-1 | **渲染实现方案** | 技术美术 | 使用 Unity Post-processing Stack v2 还是 Volume Framework？ |
| OQ-2 | **性能预算** | 性能分析师 | 滤镜层叠的 GPU 成本？目标帧率？ |
| OQ-3 | **与场景美术的协调** | 关卡美术 | 如何确保滤镜变化不影响关卡可玩性？ |
| OQ-4 | **调色板定义** | 美术设计师 | 每个状态的具体色调/RGB 值？ |

---

## Next Steps

1. 确定渲染实现方案（OQ-1）
2. 定义每个心理状态的精确滤镜参数
3. 创建状态转换的过渡动画原型
4. 性能测试和优化
