# UI System (UI 系统)

> **Status**: Approved
> **Author**: UI Programmer + UX Designer
> **Created**: 2026-04-08
> **Last Updated**: 2026-04-08
> **Priority**: MVP
> **Layer**: Presentation
> **Implements Pillar**: 所有支柱（支撑系统）
> **Depends On**: World Map系统

## Overview

UI 系统是游戏的所有用户界面元素的中心管理系统，包括 HUD（抬头显示）、菜单系统、对话框、提示 UI 和覆盖层。本系统负责：
- 管理所有 UI 元素的显示/隐藏、布局和动画
- 处理 UI 输入（菜单导航、确认取消等）
- 与游戏系统协调，在正确时机显示正确信息
- 提供揭示动画的输入屏蔽功能（与其他系统的关键集成点）

**设计原则**：
- UI 是游戏体验的支撑层，不喧宾夺主
- 保持界面简洁，避免信息过载
- 所有 UI 动画必须流畅、可中断
- 输入屏蔽必须精确、可控

---

## Player Fantasy

**"信息恰到好处，沉浸感不受打扰。"**

玩家在游戏中不应该"看到 UI"，而应该"看到游戏"。UI 的最佳状态是：
- 关键时刻的信息恰到好处地出现
- 玩家需要时能快速找到关键数据（体力、弹药、任务目标）
- 菜单操作流畅，不打断游戏节奏
- 揭示动画和过场 UI 有沉浸感，不突兀

---

## Detailed Design

### Core Rules

**规则1：UI 层级架构**

```
[Canvas - 全屏]
    │
    ├── [Game HUD Layer]          # 游戏进行中的 HUD
    │       ├── Health/Stamina Display
    │       ├── Weapon/Item Slots
    │       ├── Objective Reminder
    │       └── Interaction Prompts
    │
    ├── [Alert Layer]            # 警告/提示信息
    │       ├── Damage Indicators
    │       ├── Status Effect Icons
    │       └── Mission Updates
    │
    ├── [Menu Layer]             # 菜单系统
    │       ├── Pause Menu
    │       ├── Inventory
    │       ├── Journal/Clue Log
    │       ├── Settings
    │       └── Game Over Screen
    │
    ├── [Overlay Layer]          # 覆盖层（揭示动画等）
    │       ├── Location Discovery
    │       ├── Story Narration
    │       └── Loading Overlay
    │
    └── [Input Blocking Layer]    # 输入屏蔽层
            └── Blocks all input during special sequences
```

**规则2：UI 状态机**

所有 UI 面板共享以下状态：

```
[Hidden] ──显示请求──▶ [Appearing] ──动画完成──▶ [Visible]
    ▲                              │
    │                         [被中断] │
    │                              │
    └────────[隐藏请求]────────────┘
```

| 状态 | 描述 | 可转移至 |
|------|------|---------|
| Hidden | 面板不可见，不接收输入 | Appearing（收到显示请求） |
| Appearing | 面板正在播放进入动画 | Visible（动画完成）/ Hidden（动画被中断） |
| Visible | 面板完全可见，接收输入 | Hidden（收到隐藏请求） |

**规则3：输入屏蔽机制**

当 UI 需要屏蔽玩家输入时，使用以下接口：

| 场景 | 屏蔽范围 | 屏蔽内容 |
|------|---------|---------|
| 揭示动画播放中 | 地图交互输入 | 选择地区、查看详情等 |
| 菜单打开 | 全部游戏输入 | 移动、攻击、交互 |
| 对话进行中 | 全部游戏输入 | 对话选择外的所有输入 |
| 过场动画 | 全部游戏输入 | 全部输入 |

**BlockingReason 枚举定义**

输入屏蔽的原因枚举，用于 `IsInputBlocked = (BlockingLayerActive == true) AND (BlockingReason != NONE)` 判定：

| 枚举值 | 说明 | 优先级 | 触发条件 | 持续时间 | 负责系统 |
|--------|------|--------|----------|----------|----------|
| `NONE` | 无屏蔽原因，输入正常 | 0（最低） | 不适用 | 不适用 | 不适用 |
| `DIALOG` | 对话进行中 | 1 | 玩家进入 NPC 对话范围并触发对话 | 对话持续时间 | Dialog Tree 系统 |
| `MENU` | 菜单打开（暂停/系统菜单） | 2 | 玩家按下暂停键/ESC | 菜单打开到关闭的持续时间 | UI 系统 |
| `CINEMATIC` | 过场动画播放中 | 3 | 过场动画开始播放 | 过场动画时长 | Narrative System |
| `STUNNED` | 角色被击晕/昏迷 | 4 | 受到击晕效果时（如敌人攻击命中） | 击晕持续时间（由战斗系统控制） | 战斗系统 |
| `REVELATION` | 揭示动画播放中（World Map 地点揭示） | 5 | World Map 系统触发地点揭示事件 | 揭示动画时长（可被玩家确认键中断） | World Map 系统 |
| `INTERACTION` | 特殊交互进行中（如拾取、开门） | 6 | 玩家与环境物件/NPC 交互触发 | 交互动画持续时间 | 环境交互系统 |
| `HURT` | 角色受伤（受伤动画播放时屏蔽输入） | 6 | 玩家受到伤害且触发受伤动画 | 受伤动画时长 | Health 系统 |
| `DEATH` | 角色死亡（死亡动画/黑屏期间屏蔽输入） | 7 | 玩家生命值归零 | 死亡动画/黑屏持续时间 | Health 系统 |
| `FORCED` | 强制输入屏蔽（最高优先级） | 8（最高） | 特定极端情况（如载入画面、特殊过场） | 由触发源决定 | 任意系统 |

> **优先级说明**：当多个屏蔽原因同时发生时，取优先级最高的 `BlockingReason`。例如，对话中触发揭示动画时，`BlockingReason = REVELATION`（优先级5）而非 `DIALOG`（优先级1）。

**同优先级屏蔽冲突处理规则**：

当多个屏蔽原因具有相同优先级时（如 `INTERACTION` 和 `HURT` 都是优先级 6），使用以下裁决规则：

| 规则 | 说明 |
|------|------|
| **新请求优先** | 同优先级时，新发生的屏蔽请求覆盖旧的屏蔽请求 |
| **时间戳判定** | 通过屏蔽请求的时间戳（`BlockingRequestTimestamp`）确定新旧 |
| **持续时间保留** | 新屏蔽请求覆盖后，原屏蔽请求的持续时间计时器不重置（如果原屏蔽仍在持续，可用于恢复） |
| **主动解除** | 高优先级屏蔽结束后，自动恢复低优先级的屏蔽请求（如果其持续时间尚未结束） |

**裁决公式**：

```
ActiveBlockingReason = Max(ActiveBlockingReasons)

// 同优先级裁决
if (存在多个相同优先级的屏蔽原因):
    ActiveBlockingReason = 最晚发生的屏蔽原因（最大时间戳）
```

**示例**：玩家在交互过程中受到伤害（`INTERACTION` 和 `HURT` 同时触发，优先级都是 6）：
- `INTERACTION` 在时间戳 T=10.0s 触发
- `HURT` 在时间戳 T=10.5s 触发
- 由于 `HURT` 的时间戳更晚（10.5 > 10.0），`ActiveBlockingReason = HURT`
- 当 `HURT` 的受伤动画结束后，`INTERACTION` 自动恢复（如果其交互动画仍在进行中）

**Alert Layer 队列合并规则**

Alert Layer 负责显示警告/提示信息，采用以下队列管理规则：

| 规则 | 说明 |
|------|------|
| **同类型合并** | 同一类型的警告（如同一种伤害指示器）在 `AlertDuration`（默认3秒）内只显示一次，后来的相同警告更新现有警告的显示时间而非创建新警告 |
| **不同类型垂直堆叠** | 不同类型的警告可以同时显示，按发布时间垂直堆叠（ newest 在顶部） |
| **最大数量限制** | 同时显示的警告数量受 `MaxAlerts`（默认3个）限制。当达到上限时，最早的警告自动消失（"最早"按创建时间 EventCreationTime 判定） |
| **优先级覆盖** | 高优先级警告（如"NPC 被击杀！"）可以强制显示，即使达到 `MaxAlerts` 上限也会替换最早的低优先级警告 |
| **手动关闭** | 玩家可以手动关闭警告（点击或按键），立即从队列移除 |

**警告优先级定义**：

| 优先级 | 警告类型 | 示例 |
|--------|---------|------|
| 高 | 击杀/死亡警告 | "NPC被击杀！"、"玩家受伤！" |
| 中 | 状态变化警告 | "理智下降"、"进入警戒状态" |
| 低 | 任务更新提示 | "新任务！"、"任务目标更新" |

---

## Interactions with Other Systems

### 数据流入 (Inputs)

| 来源系统 | 数据内容 | 处理方式 |
|---------|---------|---------|
| 玩家控制器 | 玩家状态（移动、攻击、交互） | HUD 显示/隐藏相关元素 |
| Sanity/Rage 系统 | 心理状态 | 调整 HUD 样式（色调、透明度） |
| Clue & Journal | 线索数量、任务进度 | 更新 HUD 提示 |
| World Map | 揭示动画触发 | 播放揭示动画并屏蔽输入 |
| **NPC AI 系统** | `AlertStateChangedEvent`（Alert State 转换） | **订阅此事件以更新 Alert Layer**：当 NPC Alert State 转换为 ALERT/ESCAPE/COMBAT 时，Alert Layer 显示"进入警戒状态"警告（中优先级）；当 NPC 发现玩家（UNDETECTED→SUSPECT）时，显示"感觉有动静"提示（低优先级）；**映射关系**：UNDETECTED/SUSPECT/SEARCH → 低优先级提示，ALERT → 中优先级警告，ESCAPE/COMBAT → 高优先级警告 |

> **HUD 色调调整与 DPP 职责边界说明**：
> - **UI 系统负责**：HUD 元素本身的色调调整（如 HUD 背景、文字颜色的变化），以及 HUD 透明度响应心理状态
> - **Dynamic Post-Processing（DPP）负责**：场景级视觉滤镜（整体画面色调、饱和度、暗角、噪点等），DPP 的滤镜效果覆盖整个游戏画面，包括 HUD
> - 两者协作方式：UI 系统调整 HUD 局部色调以呼应心理状态，DPP 提供全局视觉风格变化。极端心理状态（如 BROKEN、SOUL_SPLIT）下的全局滤镜效果不受 HUD 色调调整影响

### 数据流出 (Outputs)

| 目标系统 | 数据内容 | 说明 |
|---------|---------|------|
| 事件总线 | `UIButtonClicked` | UI 交互事件，用于 Audio 系统播放音效 |
| World Map | `DiscoveryAnimationComplete` | 揭示动画播放完毕，解除输入屏蔽 |
| 任意系统 | `UIPanelOpened/PanelClosed` | UI 状态变化通知 |
| **Screen Effects** | `ShakeRequest`（通过事件总线） | HUD 专注模式时发送震动请求，增强反馈感 |

> **VignetteRequest 说明**：极端心理状态下的暗角效果由 Sanity/Rage 系统通过 `VignetteRequest` 直接发送给 Screen Effects（详见 Screen Effects 的 Effects Triggers Matrix），UI 系统不直接发送此事件。 |

### World Map 揭示动画的输入屏蔽职责

**职责归属**：UI 系统负责在揭示动画播放期间屏蔽玩家输入。

**工作流程**：
```
1. World Map 系统发送 LocationRevealed 事件
2. UI 系统接收事件，开始播放揭示动画
3. UI 系统设置 Input Blocking Layer 为 active
4. 玩家输入被屏蔽（地图选择、菜单访问等）
5. 动画播放完毕（正常完成或被玩家中断）
6. UI 系统发送 DiscoveryAnimationComplete 事件
7. World Map 系统收到事件后解除输入阻塞
```

**屏蔽的输入类型**：
- **被屏蔽的输入**：地图拖拽/缩放、地区选择、菜单打开（ESC）、快速存档、移动、攻击
- **可触发中断的输入**：确认键（Enter/左键/A/X）→ 直接完成动画
- **始终不屏蔽的输入**：无

**动画中断处理**：
- 玩家点击"确认"按钮 → 动画立即完成 → 发送 DiscoveryAnimationComplete
- 动画正常播放完毕 → 发送 DiscoveryAnimationComplete

**BlockingReason 裁决公式**：

当多个屏蔽原因同时发生时，使用以下公式确定 ActiveBlockingReason：

```
ActiveBlockingReason = Max(ActiveBlockingReasons)
```

裁决规则：
- 遍历所有当前处于激活状态的 BlockingReason
- 取优先级数值最大的（Priority 值最大）作为 ActiveBlockingReason
- 最终输入屏蔽判定：`IsInputBlocked = (BlockingLayerActive == true) AND (ActiveBlockingReason != NONE)`

**示例**：当对话（DIALOG，Priority=1）和揭示动画（REVELATION，Priority=5）同时发生时：
- ActiveBlockingReason = Max(1, 5) = REVELATION
- 揭示动画优先级更高，输入继续被屏蔽

---

## HUD Elements

### 主 HUD 布局

```
┌─────────────────────────────────────────────────────────┐
│ [体力槽]                                    [弹药/武器槽] │
│                                                         │
│                                                         │
│                                                         │
│                    [屏幕中央]                            │
│                  (准星/交互提示)                        │
│                                                         │
│                                                         │
│                                                         │
│ [任务目标]                              [线索数量/日志]  │
└─────────────────────────────────────────────────────────┘
```

### HUD 元素定义

| 元素 | 位置 | 显示内容 | 触发条件 |
|------|------|---------|---------|
| 体力槽 | 左上 | 体力值（仅在不满时显示） | 体力消耗/恢复中 |
| 武器槽 | 右上 | 当前武器图标 | 切换武器时 |
| 准星 | 中央 | 交互提示/瞄准 | 可交互时 |
| 任务目标 | 左下 | 当前任务描述 | 任务激活时 |
| 线索计数 | 右下 | 已收集/总线索 | 有线索时 |
| 心理状态 | 右下 | 理智/愤怒图标 | 状态异常时 |

### 专注模式 UI（来自 LOS 系统）

```
┌─────────────────────────────────────────┐
│                                         │
│                                         │
│            （画面边缘暗角）               │
│                                         │
│                  ◎ ← 声源波纹             │
│                 ╱╲                      │
│                ╱  ╲                     │
│               ●━━━━ ← 准星              │
│                                         │
│                                         │
│  [V] 专注监听                           │
└─────────────────────────────────────────┘
```

---

## Menu System

### 菜单层级

```
[主菜单]
    ├── [开始游戏]
    ├── [继续游戏]（有存档时）
    ├── [设置]
    │       ├── 画面设置
    │       ├── 音效设置
    │       ├── 控制设置
    │       └── 无障碍设置
    └── [退出游戏]

[游戏内暂停菜单]
    ├── [继续游戏]
    ├── [快速存档]
    ├── [设置]
    └── [返回主菜单]
```

### 菜单操作

| 操作 | PC | PS5 |
|------|-----|-----|
| 打开/关闭菜单 | ESC | Options |
| 确认选择 | Enter / 左键 | A/X |
| 取消/返回 | ESC / 右键 | B/Circle |
| 导航 | WASD / 方向键 | 左摇杆 |
| 滚动列表 | 滚轮 / W/S | L1/R1 |

---

## Visual/Audio Requirements

### 视觉风格

- **整体风格**：简洁、功能导向，避免过度装饰
- **色调**：跟随游戏整体色调（犯罪/黑暗主题）
- **字体**：清晰可读，最小 16px
- **图标**：简洁图形，高对比度

### 色调规范

| 元素 | 颜色 | Hex | 使用场景说明 |
|------|------|-----|-------------|
| 正常状态文字 | 白色 | #FFFFFF | 默认文字显示 |
| 警告文字 | 黄色 | #F4A261 | UI 层警告提示（如任务目标更新、数量不足） |
| 错误/危险文字 | 红色 | #E63946 | UI 层严重警告（如死亡、任务失败） |
| 提示文字 | 灰色 | #95A5A6 | 次要提示、禁用状态 |
| 背景遮罩 | 半透明黑 | #000000 (60% opacity) | 菜单/面板背景 |

> **UI 警告色与 World Map 危险色的区别**：
> - **UI 警告色**（黄色 #F4A261、红色 #E63946）用于 UI 元素本身的状态表达，如 HUD 警告文字、背包物品不足提示
> - **World Map 危险色**用于地图上的危险区域标记、敌人分布显示等地理信息
> 两者服务于不同的信息维度，UI 系统不直接控制 World Map 的颜色定义

### 音效反馈

| 事件 | 音效 |
|------|------|
| 菜单打开 | 低沉"嗡"声 |
| 菜单关闭 | 反向"嗡"声 |
| 按钮悬停 | 轻微"滴"声 |
| 按钮确认 | 清脆"咔"声 |
| 取消/返回 | 低沉"咚"声 |
| 数值变化 | 细微"嗒"声 |

---

## Formulas

### UI 淡入淡出公式

```
Alpha = 1.0 - ExpDecay(TimeSinceChange, HalfLife)
CurrentAlpha = Lerp(CurrentValue, TargetValue, Alpha)
```

**语义说明**：
- `Alpha`：插值系数，从0.0线性增长到1.0（经过1.0-ExpDecay转换）
- 当 `TimeSinceChange = 0` 时，`Alpha = 0.0`，`CurrentAlpha = Lerp(CurrentValue, TargetValue, 0.0) = CurrentValue`（无过渡，保持当前值）
- 当 `TimeSinceChange → ∞` 时，`Alpha → 1.0`，`CurrentAlpha = Lerp(CurrentValue, TargetValue, 1.0) = TargetValue`（完成过渡，到达目标值）
- **本实现与 Screen Effects 系统一致**：使用 `1.0 - ExpDecay` 形式，确保过渡从起点开始逐渐到达目标值
- 过渡曲线：初期变化快（视觉效果明显），后期变化慢（收敛稳定）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `TargetAlpha` (Visible) | 1.0 | 面板完全可见时的透明度 |
| `TargetAlpha` (Hidden) | 0.0 | 面板完全隐藏时的透明度 |
| HalfLife | 0.2s | 透明度过渡半衰期 |

### 输入屏蔽判定

```
IsInputBlocked = (BlockingLayerActive == true) AND (BlockingReason != NONE)
```

---

## Edge Cases

### 边缘情况1：菜单打开时游戏暂停

**问题**：暂停菜单打开时，游戏逻辑是否应该暂停？

**处理**：
- 暂停菜单打开时，游戏时间强制暂停
- 背景音乐淡出
- 所有游戏逻辑（AI、计时器等）停止
- 恢复时，所有逻辑从暂停点继续

### 边缘情况2：多个 UI 面板同时请求显示

**问题**：任务更新提示和伤害警告同时触发。

**处理**：
- Alert Layer 使用队列管理
- 同一类型警告合并（多次伤害只显示一次）
- 不同类型警告垂直堆叠
- 超过 3 个警告时，最早的自动消失

### 边缘情况3：揭示动画期间玩家快速操作

**问题**：玩家在揭示动画期间反复按确认键。

**处理**：
- 动画开始时屏蔽所有游戏输入（移动、攻击、地图交互等），但**确认键可以触发中断**
- 第一次确认键：动画立即完成，发送 DiscoveryAnimationComplete
- 后续确认键：**忽略**（动画已完成，无需再次中断）
- 动画完成后：所有输入恢复正常响应

### 边缘情况4：HUD 元素与游戏元素重叠

**问题**：HUD 元素遮挡重要游戏信息（如视线交汇点）。

**处理**：
- HUD 元素使用透明度背景，确保不遮挡游戏内容
- 关键区域（屏幕中心）保留为空
- 提供 HUD 位置调整选项

---

## Tuning Knobs

### 显示参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| HUDOpacity | float | 0.9 | 0.5~1.0 | HUD 背景透明度 |
| AlertDuration | float | 3.0s | 1.0~5.0s | 提示显示时长 |
| MaxAlerts | int | 3 | 1~5 | 同时显示的最大警告数 |

### 动画参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| UIFadeInDuration | float | 0.2s | 0.1~0.5s | UI 淡入时长 |
| UIFadeOutDuration | float | 0.15s | 0.05~0.3s | UI 淡出时长 |
| MenuAppearDuration | float | 0.25s | 0.1~0.5s | 菜单出现动画时长 |

### 位置参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| HUDMargins | Vector4 | (20, 20, 20, 20) | HUD 与屏幕边缘距离 |
| AlertVerticalOffset | float | 100px | 警告距顶部距离 |

---

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | HUD 在游戏进行时正确显示所有必要元素 | 启动游戏，观察 HUD |
| AC-2 | 体力槽在消耗/恢复时正确更新 | 消耗体力，观察体力槽变化 |
| AC-3 | 菜单可以正常打开/关闭 | 按 ESC/Options，观察菜单 |
| AC-4 | 揭示动画期间输入被正确屏蔽 | 触发地点揭示，尝试移动/选择 |
| AC-5 | 揭示动画完成后输入正确恢复 | 等待动画完成，验证输入恢复 |
| AC-6 | 多个警告可以正确堆叠显示 | 触发多个警告，观察堆叠 |

### 集成验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-7 | 与 World Map 系统正确同步 | 揭示地点，观察揭示动画 |
| AC-8 | 与 Sanity/Rage 系统正确同步 | 状态变化，观察 HUD 色调 |
| AC-9 | 与 Audio 系统正确同步 | UI 操作，观察音效 |
| AC-10 | 与 Clue 系统正确同步 | 收集线索，观察线索计数 |

---

## Open Questions

| # | 问题 | 负责人 | 说明 |
|---|------|--------|------|
| OQ-1 | ✅ **已解决**：MVP阶段不需要小地图。俯视角本身提供了良好的空间感知，且开发小地图会分散UI资源。 | 游戏设计师 | 俯视角游戏中是否需要小地图 |
| OQ-2 | ✅ **已解决**：MVP采用最小化响应方案（透明度0.9→0.7 + 微弱色调偏移），极端状态由DPP主导 | 游戏设计师 | HUD样式是否需要根据心理状态变化 |
| OQ-3 | ✅ **已解决**：MVP采用响应式布局检测，手柄模式自动切换 | UX 设计师 | 游戏手柄是否需要特殊HUD布局 |

---

## Change Log

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-08 | 0.1 | 初稿创建 | UI Programmer Agent |
| 2026-04-15 | 0.2 | P2修复：BlockingReason枚举添加详细说明表格（触发条件、持续时间、负责系统）；添加同优先级屏蔽冲突处理规则（新请求优先+时间戳判定）；更新优先级说明和裁决公式 | Claude Code |
