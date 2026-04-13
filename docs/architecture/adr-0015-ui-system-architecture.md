# ADR-0015: UI 系统 (UI System) 架构决策

## Status
**Accepted**

## Date
2026-04-10

## Last Updated
2026-04-11 (v6: 确认 shared-types §12.2/12.3 引用路径正确性；补充 UI 系统分辨率自适应实现方式说明)

## Context

### Problem Statement

UI 系统是游戏所有用户界面元素的中心管理系统，包括 HUD、菜单系统、对话框、提示 UI 和覆盖层。作为 Presentation 层，UI 系统需要：

1. **层级管理**：管理 Game HUD Layer / Alert Layer / Menu Layer / Overlay Layer / Input Blocking Layer 的显示/隐藏和层级关系
2. **状态驱动**：所有 UI 面板共享 Hidden/Appearing/Visible 状态机
3. **输入屏蔽**：在揭示动画、菜单打开、对话进行中等场景屏蔽玩家输入
4. **数据消费**：作为所有游戏系统的数据消费者，订阅并显示来自下游系统的状态变化

### Constraints

- **设计约束**：UI 是游戏体验的支撑层，不喧宾夺主，保持界面简洁
- **输入约束**：输入屏蔽必须精确、可控，支持动画中断处理
- **平台约束**：支持 PC (ESC/WASD/Enter) 和 PS5 (Options/左摇杆/A/X)
- **性能约束**：UI 动画必须流畅（60fps），支持淡入淡出插值

### Requirements

- **必须**：定义 UI 层级架构（五层结构）
- **必须**：定义 UI 面板状态机（Hidden/Appearing/Visible）
- **必须**：定义输入屏蔽机制（World Map 揭示动画的 Input Blocking 职责归属）
- **必须**：定义与所有游戏系统的接口（事件订阅关系）
- **必须**：定义 HUD 布局（体力槽/武器槽/准星/任务目标/线索计数）
- **必须**：遵循 ADR-0003 系统分层定义（UI System 属于 Presentation Layer）

---

## Decision

### 架构决策

采用**分层面板架构 + 状态机驱动 + 事件订阅**架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         UI System 架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  [Canvas - 全屏]                                                            │
│      │                                                                     │
│      ├── [Game HUD Layer]          # 游戏进行中的 HUD                       │
│      │       ├── Health/Stamina Display                                │
│      │       ├── Weapon/Item Slots                                      │
│      │       ├── Objective Reminder                                    │
│      │       └── Interaction Prompts                                    │
│      │                                                                     │
│      ├── [Alert Layer]             # 警告/提示信息                          │
│      │       ├── Damage Indicators                                     │
│      │       ├── Status Effect Icons                                   │
│      │       └── Mission Updates                                        │
│      │                                                                     │
│      ├── [Menu Layer]              # 菜单系统                              │
│      │       ├── Pause Menu                                               │
│      │       ├── Inventory                                                │
│      │       ├── Journal/Clue Log                                        │
│      │       └── Settings                                               │
│      │                                                                     │
│      ├── [Overlay Layer]           # 覆盖层（揭示动画等）                     │
│      │       ├── Location Discovery                                     │
│      │       ├── Story Narration                                        │
│      │       └── Loading Overlay                                         │
│      │                                                                     │
│      └── [Input Blocking Layer]    # 输入屏蔽层                            │
│              └── Blocks all input during special sequences                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### UI 面板状态机

```
[Hidden] ──显示请求──▶ [Appearing] ──动画完成──▶ [Visible]
    ▲                              │
    │                         [被中断]
    │
    └────────[隐藏请求]────────────────┘
```

### UI 层级优先级

各 Layer 按以下优先级排序（数值越大显示越靠前）：

| Layer | 优先级 | 说明 |
|-------|--------|------|
| Game HUD Layer | 0 | 游戏进行时恒显示，最低优先级 |
| Alert Layer | 1 | 警告/提示信息，覆盖 HUD |
| Overlay Layer | 2 | 揭示动画/过场，覆盖 Alert |
| Menu Layer | 3 | 菜单系统，覆盖所有游戏 UI |
| Input Blocking Layer | 4 | **始终置顶**，屏蔽所有输入 |

> **注**：Input Blocking Layer 优先级最高，确保特殊序列（揭示动画、过场）期间所有输入被正确屏蔽。

### Sanity/Rage 系统集成

UI 系统订阅以下事件以响应玩家心理状态变化（事件定义见 ADR-0017）：

| 事件 | 来源系统 | UI 响应 |
|------|---------|---------|
| `PsychologicalStateEvent{state}` | Sanity/Rage 系统 | 根据心理状态调整 HUD 色调（FRENZIED 时脉冲红色，SOUL_SPLIT 时双重效果）。**事件定义见 shared-types.md §12.2** |
| `VignetteRequest{intensity}` | Sanity/Rage 系统 | 低理智时 HUD 边缘暗角增强。**事件定义见 shared-types.md §12.3** |
| `BlurRequest{intensity}` | Sanity/Rage 系统 | 低理智时 HUD 模糊效果增强。**事件定义见 shared-types.md §12.3** |
| `HUDOverlayOpacityRequest{opacity}` | Sanity/Rage 系统 | Sanity=0 时 HUD 半透明叠加层显示（确保信息可读）。**事件定义见 shared-types.md §12.3** |
| `ShakeRequest{intensity}` | Sanity/Rage 系统 | 高愤怒时 HUD 准星抖动。**事件定义见 shared-types.md §12.3** |

> **事件来源说明**：`PsychologicalState` / `VignetteRequest` / `BlurRequest` / `HUDOverlayOpacityRequest` / `ShakeRequest` 事件由 ADR-0017 (Sanity/Rage 系统) 发布，UI 系统作为消费者订阅这些事件。

**色调调整公式**：
> **参数说明**：Sanity 值范围为 0-100（详见 ADR-0017 §双轨计量系统）

```
HUDTintColor = Lerp(NormalColor, AlertColor, 1.0 - Sanity/100)
// 说明：Sanity=100 时返回 NormalColor（正常），Sanity=0 时返回 AlertColor（警报）
// 此公式与 ADR-0017 的 VignetteRequest 风格保持一致（都是 Lerp(min, max, ratio)）
HUDBlurIntensity = Clamp((50 - Sanity) / 50, 0, 0.5)
// 对应 BlurRequest{Intensity}，由 Sanity/Rage 系统计算并发布
```

**颜色参数定义**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `NormalColor` | Color | `#FFFFFF` (纯白) | HUD 正常状态色调 |
| `AlertColor` | Color | `#FF3333` (警戒红) | HUD 警报状态色调（低理智时渗入） |

> **策划配置说明**：`NormalColor` 和 `AlertColor` 作为可调参数存储在 `UISanityTuningSO` 中，策划可根据美术风格调整具体数值。

### World Map 揭示动画的输入屏蔽职责

```
1. World Map 系统发送 LocationRevealed 事件
2. UI 系统接收事件，开始播放揭示动画
3. UI 系统设置 Input Blocking Layer 为 active
4. 玩家输入被屏蔽（地图拖拽/缩放/选择/菜单/快速存档）
5. 动画完成（正常完成或被玩家中断）
6. UI 系统发送 DiscoveryAnimationComplete 事件
7. World Map 系统收到事件后解除输入阻塞
```

**与 ActionLockSystem 的关系说明**：

> World Map 的 `input_blocked` 机制**独立于** ActionLockSystem（shared-types.md §4），原因如下：
> - **作用域不同**：ActionLockSystem 的 ActionLockType 用于**游戏内全局输入**（如 Gritty Takedowns 的 Interaction、Dialogue System 的 Cutscene），影响玩家移动/动作控制
> - **职责不同**：World Map 的 `input_blocked` 仅控制**地图 UI 交互**，不涉及玩家角色动作
> - **实现方式**：World Map 通过 MapDisplayRequestEvent 的 `input_blocked` 字段通知 UI 系统屏蔽地图操作，无需介入 PlayerController 的动作锁定
>
> 如后续需要更细粒度的输入控制，可考虑复用 ActionLockSystem，但当前设计保持独立以避免职责混淆。

---

## Formulas

### 核心公式

**UI 淡入淡出**：
```
CurrentAlpha = Lerp(CurrentAlpha, TargetAlpha, ExpDecay(TimeSinceChange, HalfLife))
```
| 参数 | 默认值 | 说明 |
|------|--------|------|
| HalfLife | 0.2s | 透明度过渡半衰期 |

**输入屏蔽判定**：
```
IsInputBlocked = (BlockingLayerActive == true) AND (BlockingReason != NONE)
BlockingPriority = HighestPriority(ActiveBlockers)
```

**Alert 合并规则**：
```
当同类型 Alert 同时触发时：
  - 取较大 intensity 的值
  - 叠加 duration（不超过 MaxAlertDuration）
当不同类型 Alert 堆叠时：
  - 按时间顺序垂直排列
  - 超出 MaxAlerts 数量时，最早的 Alert 被移除
```
| 参数 | 默认值 | 说明 |
|------|--------|------|
| MaxAlertDuration | 10.0s | 单个 Alert 最大持续时间 |
| MaxAlerts | 3 | 同类型 Alert 最大队列数量 |

### Input Blocking Layer 优先级机制

```
InputBlockingRequest {
    Source: string           // 请求来源（WorldMap / Cutscene / Dialogue / etc.）
    Priority: int            // 优先级（数值越大越高）
    BlockedInputs: InputMask // 被屏蔽的输入类型
}

InputMask 定义（按位掩码）：

| 位 | 输入 | PC 键位 | PS5 按键 |
|----|------|--------|---------|
| 0 | 移动 | WASD / 方向键 | 左摇杆 |
| 1 | 攻击 | 鼠标左键 / F | R2 / 右扳机 |
| 2 | 交互 | E / F | 圆圈 / O |
| 3 | 跳跃 | 空格 | X / 方块 |
| 4 | 蹲伏 | C / Ctrl | 按下左摇杆 |
| 5 | 菜单 | ESC | Options |
| 6 | 快速存档 | Q | 触摸板 |
| 7 | 地图拖拽/缩放 | 鼠标拖拽 | 右摇杆 |

优先级仲裁规则：
1. 高优先级请求可覆盖低优先级请求
2. 同优先级请求不能相互覆盖，需 Source 自己处理
3. UI 确认键（确认/取消）在任何情况下都应被识别，除非被同优先级或更高优先级明确屏蔽
```

> **注**：Input Blocking Layer 的"屏蔽所有输入"指的是屏蔽游戏操作（移动/攻击/交互），UI 导航和确认键由各层自行处理，确保特殊序列中玩家仍能完成必要的 UI 操作。

---

## Tuning Knobs

| 类别 | 参数 | 默认值 | 安全范围 |
|------|------|--------|---------|
| 显示 | `HUDOpacity` | 0.9 | 0.5~1.0 |
| 显示 | `AlertDuration` | 3.0s | 1.0~5.0s |
| 显示 | `MaxAlerts` | 3 | 1~5 |
| 动画 | `UIFadeInDuration` | 0.2s | 0.1~0.5s |
| 动画 | `UIFadeOutDuration` | 0.15s | 0.05~0.3s |
| 动画 | `MenuAppearDuration` | 0.25s | 0.1~0.5s |
| 位置 | `HUDMargins` | (20,20,20,20) | — |
| 位置 | `AlertVerticalOffset` | 100px | — |

> **像素值说明**：`AlertVerticalOffset` 等像素值基于**设计分辨率**（1080p）定义。在运行时，系统根据实际屏幕分辨率与设计分辨率的比例自动缩放。实现方式：
> ```csharp
> float scaleFactor = Screen.height / 1080f;
> float scaledOffset = baseOffset * scaleFactor;
> ```
> 例如，在 4K 分辨率下，100px 会自动缩放为约 200px，确保 UI 元素在不同分辨率下保持相对一致的视觉位置。

---

## Edge Cases

| # | 场景 | 处理方式 |
|---|------|---------|
| EC-1 | 菜单打开时游戏暂停 | **GameTimeSystem 负责暂停**（订阅 Menu Layer 的 `PauseMenuOpened` 事件并设置 `time_scale = 0`）；**AudioSystem 负责音乐淡出**（订阅同一事件执行 `MusicFadeOut(0.3s)`）。**事件定义见 shared-types.md §13.4**。<br><br>**时序说明（ASCII 图）**：事件触发时，AudioSystem 与 GameTimeSystem **并行处理**（非串行）：<br><br>**打开菜单时序**：<br>```<br>时间轴  ─────────────────────────────────────────────────────▶<br><br>事件    PauseMenuOpened 发布<br>    │<br>    ├──────────────────────────────────────────────────┐<br>    │                    ▼                                │<br>    │  ┌────────────────────┐   ┌────────────────────┐   │<br>    │  │   AudioSystem      │   │  GameTimeSystem    │   │<br>    │  │   (并行处理)       │   │   (并行处理)       │   │<br>    │  └─────────┬──────────┘   └─────────┬──────────┘   │<br>    │            │                         │               │<br>    │            ▼                         ▼               │<br>    │     MusicFadeOut(0.3s)      time_scale = 0         │<br>    │            │                         │               │<br>    │            │                         │               │<br>    │            ◀──────── 0.3s ────────▶               │<br>    │            │                                          │<br>    └──────────────────────────────────────────────────┘<br>    │<br>    │ (0.3s 后音乐淡出完成，游戏逻辑已暂停)<br>```<br><br>**关闭菜单时序**：<br>```<br>时间轴  ─────────────────────────────────────────────────────▶<br><br>事件    PauseMenuClosed 发布<br>    │<br>    ├──────────────────────────────────────────────────┐<br>    │                    ▼                                │<br>    │  ┌────────────────────┐   ┌────────────────────┐   │<br>    │  │   AudioSystem      │   │  GameTimeSystem    │   │<br>    │  │   (并行处理)       │   │   (并行处理)       │   │<br>    │  └─────────┬──────────┘   └─────────┬──────────┘   │<br>    │            │                         │               │<br>    │            ▼                         ▼               │<br>    │     MusicFadeIn(0.3s)       time_scale = 1         │<br>    │            │                         │               │<br>    │            ◀──────── 0.3s ────────▶               │<br>    │            │                                          │<br>    └──────────────────────────────────────────────────┘<br>    │<br>    │ (0.3s 后音乐淡入完成，游戏逻辑已恢复)<br>```<br><br>**幂等性说明**：`PauseMenuOpened` 事件具有幂等性。如果游戏已经处于暂停状态，再次打开菜单不会重复发布 `PauseMenuOpened` 事件（UI 系统内部维护 `is_menu_open` 标志位，重复请求时直接忽略）。 |
| EC-2 | 多个面板同时请求显示 | Alert Layer 队列管理，同类型合并，不同类型堆叠，超过3个时最早的消失 |
| EC-3 | 揭示动画期间快速操作 | 第一次确认键完成动画，后续忽略，动画完成后才响应新输入 |
| EC-4 | HUD 与游戏元素重叠 | HUD 使用透明度背景，关键区域保留为空，提供位置调整选项 |
| EC-5 | 同优先级 Input Blocking 请求冲突 | 同优先级请求支持队列化（最大队列长度 2），超出时最早的请求被替换。不同优先级间高优先级覆盖低优先级 |
| EC-6 | 揭示动画卡死（超时） | 动画超时时间 5.0s，超时后强制完成动画并发送 `DiscoveryAnimationComplete`，防止 UI 永久挂起 |

---

## Alternatives Considered

### Alternative 1: 各系统自行管理 UI

- **描述**：每个游戏系统（Health/NPC AI/Clue）自行管理其相关 UI 元素的显示/隐藏
- **优点**：开发简单直接
- **缺点**：UI 逻辑分散，难以维护一致的视觉风格和交互体验；输入屏蔽逻辑重复
- **拒绝理由**：违背单一职责原则，UI 外观和交互一致性无法保证

### Alternative 2: 集中式 UI 管理器

- **描述**：所有 UI 元素集中在 UIManager，统一管理显示/隐藏/动画
- **优点**：完全控制 UI 状态，一致性好
- **缺点**：UIManager 过于臃肿，违反 ADR-0003 分层原则
- **拒绝理由**：系统过于庞大，难以维护；分层架构要求 Presentation 层系统精简

---

## Consequences

### Positive

- 分层架构清晰，职责划分明确
- 状态机驱动使面板行为可预测、可调试
- 输入屏蔽职责归属清晰（UI 系统负责）
- Alert Layer 使用队列管理，避免信息过载

### Negative

- 五层架构增加了 UI 复杂度
- 跨层级的状态同步需要事件机制，增加了系统间耦合

### Risks

- **风险**：多个 UI 面板同时请求显示（任务更新 + 伤害警告同时触发）
  - **缓解**：Alert Layer 使用队列，同类型合并，不同类型垂直堆叠，超过3个时最早消失
- **风险**：揭示动画期间玩家快速操作导致输入处理异常
  - **缓解**：第一次确认键直接完成动画，后续忽略

---

## Performance Implications

- **CPU**：UI 淡入淡出使用 ExpDecay 插值，轻量级
- **Memory**：HUD 元素按需实例化，不驻留内存
- **Load Time**：UI 资源预加载，揭示动画期间无额外加载

---

## Migration Plan

1. 先实现基础 UI 层级架构和面板状态机
2. 实现 Game HUD Layer 的基础元素（体力槽、武器槽）
3. 实现 Menu Layer 和 Pause 功能
4. 实现 Input Blocking Layer 和 World Map 揭示动画集成
5. 实现 Alert Layer 队列管理

---

## Validation Criteria

| ID | 验收条件 |
|----|---------|
| AC-1 | HUD 在游戏进行时正确显示所有必要元素 |
| AC-2 | 体力槽在消耗/恢复时正确更新 |
| AC-3 | 菜单可以正常打开/关闭 |
| AC-4 | 揭示动画期间输入被正确屏蔽 |
| AC-5 | 揭示动画完成后输入正确恢复 |
| AC-6 | 多个警告可以正确堆叠显示 |
| AC-7 | 与 World Map 系统正确同步 |
| AC-8 | 与 Sanity/Rage 系统正确同步（色调调整） |
| AC-9 | 与 Clue & Journal 系统正确同步（线索计数更新） |
| AC-10 | UI 操作触发正确音效 |

---

## Related Decisions

- ADR-0003: 系统分层架构（Presentation Layer 定义）
- ADR-0012: 世界地图非线性叙事架构（揭示动画集成）
- ADR-0009: 玩家控制器架构（HUD 数据来源）
- ADR-0017: 理智/愤怒系统（Sanity/Rage 事件定义）
