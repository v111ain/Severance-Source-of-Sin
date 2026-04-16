# ADR-0028: UI 设计规范 (UI Design Specification)

## Status
**Accepted**

## Date
2026-04-14

## Last Updated
2026-04-14 (v1: 新建 UI 设计规范文档，统一颜色/字体/间距/动画规范)

## Context

### Problem Statement

ADR-0014/0015/0016/0017 等 UI 相关 ADR 中涉及大量视觉参数（颜色、动画时长、缓动曲线），但缺乏统一的 UI 设计语言定义。本文档建立统一的 UI 设计规范，确保各系统的 UI 实现保持视觉一致性。

> **UI 事件订阅规范**：所有 UI 面板的事件订阅必须遵循 shared-types.md §13.x UI系统事件订阅规范。本文档定义视觉规范，shared-types.md 定义交互规范，两者共同构成完整的 UI 系统规范。

### Constraints

- **风格约束**：深色血腥风格，与游戏"致命的脆弱感"支柱一致
- **性能约束**：所有动画必须 < 1ms CPU 开销（60fps）
- **平台约束**：PC (1080p/1440p/4K) 和 PS5 (1080p/4K)

---

## 颜色规范

| 用途 | 名称 | Hex | RGBA |
|------|------|-----|------|
| 主色调 | Blood Red | `#8B0000` | RGBA(139, 0, 0, 255) |
| 辅助色 | Dark Gray | `#2D2D2D` | RGBA(45, 45, 45, 255) |
| 强调色 | Burnt Orange | `#CC4400` | RGBA(204, 68, 0, 255) |
| 背景色 | Near Black | `#0D0D0D` | RGBA(13, 13, 13, 255) |
| 文字色 | Light Gray | `#E0E0E0` | RGBA(224, 224, 224, 255) |
| 危险色 | Alert Red | `#FF2200` | RGBA(255, 34, 0, 255) |
| 理智色 | Sanity Blue | `#1A4D8F` | RGBA(26, 77, 143, 255) |
| 狂暴色 | Rage Orange | `#FF6600` | RGBA(255, 102, 0, 255) |
| 成功色 | Toxic Green | `#2D8B2D` | RGBA(45, 139, 45, 255) |

---

## 字体规范

| 用途 | 字体 | 大小 | 字重 | 颜色 |
|------|------|------|------|------|
| 标题 (H1) | Oswald | 28px | Bold | `#E0E0E0` |
| 标题 (H2) | Oswald | 22px | Bold | `#E0E0E0` |
| 正文 | Inter | 15px | Regular | `#E0E0E0` |
| 数值显示 | JetBrains Mono | 18px | Medium | `#E0E0E0` |
| 按钮文字 | Inter | 14px | SemiBold | `#E0E0E0` |
| 注释 | Inter | 12px | Regular | `#888888` |

> **字体来源**：Oswald（标题，Google Fonts 开源）、Inter（正文，Google Fonts 开源）、JetBrains Mono（数值，JetBrains 开源）。如需授权版本可替换为adobe-clean。

---

## 间距规范

| 类别 | 间距 | 用途 |
|------|------|------|
| 基础网格 | 8px | 所有间距以此为基准 |
| 组件内边距 | 16px | 按钮、卡片内部留白 |
| 组件间距 | 24px | 两个相邻组件之间的距离 |
| 区域间距 | 48px | 区域之间的间隔 |
| 屏幕边缘 | 24px | UI 元素与屏幕边缘的最小距离 |

---

## 动画规范

### 通用动画时长

> **⚠️ UI 动画时长引用 ADR-0015 作为权威来源**：以下时长与 [ADR-0015 UI 系统架构](./adr-0015-ui-system-architecture.md) 保持一致。
>
> | 参数 | ADR-0015 定义 | 说明 |
> |------|--------------|------|
> | `UIFadeInDuration` | 0.2s | 默认淡入时长 |
> | `UIFadeOutDuration` | 0.15s | 默认淡出时长 |
> | `MenuAppearDuration` | 0.25s | 菜单出现动画 |

| 动画类型 | 时长 | 缓动曲线 | 与 ADR-0015 对应 |
|----------|------|----------|-----------------|
| 过渡动画（面板出现/消失） | 0.25s | ease-in-out | 对应 MenuAppearDuration |
| 悬停效果（按钮高亮） | 0.15s | ease-out | — |
| 弹窗出现 | 0.25s | ease-out | 对应 MenuAppearDuration |
| 数值变化（血量条减少） | 0.2s | linear | — |
| 淡入淡出 | 0.2s / 0.15s | ease-in-out | 对应 UIFadeIn/UIFadeOutDuration |
| 缩放出现 | 0.25s | ease-out (scale 0.95 → 1.0) | 对应 MenuAppearDuration |

### 缓动曲线定义

```
ease-in-out:     CubicBezier(0.42, 0, 0.58, 1.0)
ease-out:        CubicBezier(0, 0, 0.58, 1.0)
ease-in:         CubicBezier(0.42, 0, 1.0, 1.0)
linear:          CubicBezier(0, 0, 1.0, 1.0)
```

### 心理状态切换动画

所有心理状态（CALM / AGITATED / BROKEN / FRENZIED / SOUL_SPLIT）之间的切换：

> **⚠️ 时长与 ADR-0023/GDD DPP 统一**：心理状态切换时，UI 动画与屏幕后处理效果（DPP）同步过渡。时长引用 [GDD DPP §过渡动画参数](./design/gdd/dynamic-post-processing.md#过渡动画参数)：
> - `NormalTransitionDuration = 0.5s`：相邻状态间的过渡（AGITATED↔BROKEN 等）
> - `ExtremeTransitionDuration = 1.5s`：极端状态切换（CALM↔FRENZIED、正常↔SOUL_SPLIT）

| 切换类型 | 时长 | 缓动曲线 |
|----------|------|----------|
| 相邻状态切换 | 0.5s | ease-in-out |
| 极端状态切换 | 1.5s | ease-in-out |
| SOUL_SPLIT 进入 | 1.5s | ease-in-out |

> **屏幕视觉效果（Lerp 插值）**：暗角、噪点、饱和度等通过 Lerp 公式自动平滑过渡，无需额外动画。详见 [GDD DPP 规则4：滤镜插值过渡](./design/gdd/dynamic-post-processing.md#规则4滤镜插值过渡)。

---

## HUD 布局锚点

使用 12 宫格锚点系统定义 HUD 元素位置：

```
┌──────────────────────────────────────────────┐
│  [1,1]          [5,1]          [9,1]        │
│                                              │
│  [1,5]  HUD LEFT    [5,5]  CENTER   [9,5]  │
│                                              │
│  [1,9]          [5,9]          [9,9]        │
└──────────────────────────────────────────────┘
```

| 元素 | 锚点 | 位置说明 |
|------|------|----------|
| 体力槽 | [1,9] | 左下角，垂直排列 |
| 理智计量条 | [1,7] | 体力槽上方，与狂暴计量条成对显示 |
| 狂暴计量条 | [1,5] | 理智计量条上方，垂直排列 |
| 武器槽 | [9,9] | 右下角 |
| 准星 | [5,5] | 屏幕中央 |
| 任务目标 | [9,1] | 右上角 |
| 线索计数 | [9,3] | 右上角任务目标下方 |
| 互动提示 | [5,9] | 底部中央 |

> **Sanity/Rage 计量条设计说明**：
> - 理智计量条和狂暴计量条在屏幕左侧垂直排列，遵循 ADR-0017 的双轨计量系统设计
> - 两者的视觉参数（颜色、动画）定义于 ADR-0017 §色调调整公式
> - UI 布局实现应引用本规范的锚点系统

---

## 图标风格指南

| 规格 | 值 |
|------|---|
| 基础尺寸 | 24x24px / 32x32px / 48x48px |
| 线宽 | 2px（24px）、3px（32px+） |
| 风格 | 扁平化、线性图标，避免渐变 |
| 颜色 | 单色（`#E0E0E0`），激活态使用 `#FF2200` |
| 格式 | SVG（编辑器）/ Texture2D（运行时） |

---

## 对话气泡样式

| 规格 | 值 |
|------|---|
| 背景色 | `#2D2D2D`（90% 透明度） |
| 边框 | 2px `#8B0000` |
| 圆角 | 8px |
| 内边距 | 16px |
| 最大宽度 | 屏幕宽度的 60% |
| 出现动画 | 0.25s ease-out，scale 0.95 → 1.0 |

### 对话情绪样式（DialogueEmotion）

> **类型定义**：DialogueEmotion 枚举定义于 shared-types.md §13.1
> **规范来源**：ADR-0014 §DialogueEmotion 枚举

| 情绪状态 | 背景色 | 边框 | 动画效果 | 参数 |
|----------|--------|------|----------|------|
| `NEUTRAL` | `#2D2D2D` | 2px `#8B0000` | 无 | 标准样式 |
| `AGITATED` | `#2D2D2D` | 2px `#8B0000` | 边缘抖动 | shake intensity: 0.05,频率: 20Hz |
| `SCARED` | `#1A1A2D`（变淡） | 2px `#4444AA` | 气泡颤抖 + 颜色变淡 | tremble: 2px, 频率: 15Hz |
| `ANGRY` | `#4D0000`（变红） | 2px `#FF2200` | 边缘锯齿 | jaggedness: 3px |

> **实现说明**：
> - 情绪样式通过 Unity UI 的 Custom Emoji/Shape 组件实现
> - 动画参数与 ADR-0014 §Tuning Knobs 保持一致
> - 情绪切换使用 0.2s ease-in-out 过渡动画

---

## 面板状态样式

| 状态 | 透明度 | 缩放 | 偏移 |
|------|--------|------|------|
| Hidden | 0 | 0.95 | Y +10px |
| Appearing | 0 → 1 | 0.95 → 1.0 | Y +10px → Y 0 |
| Visible | 1 | 1.0 | Y 0 |
| Disappearing | 1 → 0 | 1.0 → 0.95 | Y 0 → Y -10px |

---

## 交互提示样式

| 场景 | 样式 |
|------|------|
| 可交互物件 | 黄色轮廓（`#CC4400`）+ 0.5s 脉冲动画 |
| 敌人感知状态 | 红色轮廓（`#FF2200`）+ 0.3s 脉冲动画 |
| 安全状态 | 绿色轮廓（`#2D8B2D`）+ 无动画 |
| 锁定/不可用 | 灰色轮廓（`#555555`）+ 无动画 |

---

## 组件清单

| 组件 | 说明 | 文件 |
|------|------|------|
| `BloodRedColor` | 主色调常量 | `UI/Colors/BloodRedColor.cs` |
| `DarkGrayColor` | 辅助色常量 | `UI/Colors/DarkGrayColor.cs` |
| `UIFonts` | 字体配置 | `UI/Styles/UIFonts.cs` |
| `UIFontsSO` | 字体 ScriptableObject | `UI/Styles/UIFontsSO.asset` |
| `UISPacing` | 间距常量 | `UI/Styles/UISPacing.cs` |
| `UIAnimationPresets` | 动画预设 | `UI/Styles/UIAnimationPresets.cs` |
| `HUDLayoutAnchors` | HUD 锚点常量 | `UI/Layout/HUDLayoutAnchors.cs` |
| `DialogueBubbleStyle` | 对话气泡样式 | `UI/Components/DialogueBubbleStyle.cs` |
| `InteractionPromptStyle` | 交互提示样式 | `UI/Components/InteractionPromptStyle.cs` |

---

## Validation Criteria

1. 所有 UI 组件使用统一的颜色常量，无硬编码颜色值
2. 所有动画时长与本文档一致（误差 ±10ms）
3. HUD 布局符合锚点规范
4. 面板状态动画使用统一定义的状态机
5. 图标风格一致（线性、扁平、单色）

---

## Related Decisions

- [ADR-0014: DialogTree 接口协议](./adr-0014-dialog-tree-interface-architecture.md) — 对话气泡样式参考本规范
- [ADR-0015: UI 系统](./adr-0015-ui-system-architecture.md) — UI 系统实现参考本规范
- [ADR-0016: Clue Journal](./adr-0016-clue-journal-architecture.md) — Journal UI 布局参考本规范
- [ADR-0017: Sanity/Rage Meter](./adr-0017-sanity-rage-meter-architecture.md) — 心理状态动画参考本规范
