# UI 系统实现审查报告

> **审查日期**: 2026-04-08
> **审查者**: UX Designer
> **审查范围**: Assets/UI/ 目录下的核心 UI 组件
> **基于文档**:
> - design/gdd/ui-system.md (Approved)
> - design/ux/ui-ux-spec.md (Draft)
> - design/ux/ui-visual-spec.md (Draft)

---

## 1. 发现的问题列表

### 1.1 严重问题 (必须修复)

| # | 问题 | 位置 | 设计规范要求 | 实际实现 | 影响 |
|---|------|------|------------|---------|------|
| P-01 | **动画时长不匹配** | UIPanel.cs | 显示 0.25s，隐藏 0.15s | 显示 0.3s，隐藏 0.2s | 动画感觉比设计更慢 |
| P-02 | **Alert 最大数量不匹配** | AlertManager.cs | 最多 3 个 | 最多 5 个 | 屏幕可能显示过多警告 |
| P-03 | **HUD 淡入时长不匹配** | HUDController.cs | 0.2s | 0.3s | HUD 出现/消失动画拖沓 |
| P-04 | **揭示动画中断逻辑缺失** | DiscoveryOverlay.cs | 第一次确认立即完成动画 | 无此逻辑 | 玩家无法跳过揭示动画 |
| P-05 | **DiscoveryAnimationComplete 事件未发送** | DiscoveryOverlay.cs | 动画完成/中断时发送事件给 World Map | 注释掉的代码 | 无法与 World Map 系统集成 |
| P-06 | **游戏输入屏蔽机制缺失** | InputBlocker.cs | 屏蔽游戏输入（移动、攻击等） | 只屏蔽 UI 射线检测 | 菜单打开时游戏逻辑仍在运行 |

### 1.2 中等问题 (应该修复)

| # | 问题 | 位置 | 设计规范要求 | 实际实现 | 影响 |
|---|------|------|------------|---------|------|
| M-01 | **双输入支持缺失** | 全局 | 检测最后活跃设备，切换 KB/M 和手柄映射 | 无输入设备检测 | 手柄用户无法正确导航菜单 |
| M-02 | **手柄模式 HUD 布局缺失 (OQ-3)** | HUDController.cs | 手柄模式下重新定位 HUD 元素 | 无此逻辑 | 手柄操作可能与 HUD 冲突 |
| M-03 | **菜单焦点导航不完整** | PauseMenu.cs | WASD/方向键/左摇杆导航 | 只有按钮点击 | 无法仅用手柄操作菜单 |
| M-04 | **减少动画选项缺失** | 全局 | 开启后所有动画时长 <= 0.1s | 无此选项 | 敏感用户可能感到不适 |
| M-05 | **文本缩放系统缺失** | 全局 | 支持 75%~200% 缩放 | 无缩放系统 | 无法满足无障碍需求 |

### 1.3 轻微问题 (建议修复)

| # | 问题 | 位置 | 备注 |
|---|------|------|------|
| L-01 | **缺少快速存档按钮** | PauseMenu.cs | 设计中有快速存档按钮，实际未实现 |
| L-02 | **无障碍设置菜单未实现** | PauseMenu.cs | 设计中应有独立的无障碍设置面板 |
| L-03 | **Alert 堆叠间距不精确** | AlertManager.cs | 设计 8px，实现 10px |

---

## 2. 符合规范的部分

### 2.1 架构设计

| 符合项 | 实现位置 | 说明 |
|--------|---------|------|
| UI 面板状态机 | UIPanel.cs | 正确实现 Hidden -> Appearing -> Visible -> Hiding -> Hidden |
| 单例模式 | UIManager, HUDController, AlertManager | 正确使用双检查锁定单例 |
| 层级架构 | UIManager.cs | Game HUD Layer / Alert Layer / Menu Layer / Overlay Layer / Input Blocking Layer |
| 面板排序管理 | UIManager.cs | 正确管理 sortingOrder，支持 BringToFront/SendToBack |

### 2.2 事件系统

| 符合项 | 实现位置 | 说明 |
|--------|---------|------|
| 面板显示/隐藏事件 | UIPanel.cs | PanelShowEvent, PanelShownEvent, PanelHideEvent, PanelHiddenEvent |
| UIEventBus 架构 | 全局 | 使用统一事件总线解耦 |

### 2.3 核心功能

| 符合项 | 实现位置 | 说明 |
|--------|---------|------|
| 面板显示/隐藏动画 | UIPanel.cs | 使用 AnimationHelper + AnimationCurve |
| Alert 队列管理 | AlertManager.cs | 支持队列、最大数量、合并模式 |
| Alert 堆叠计数 | AlertManager.cs | 正确实现同类型警报合并 |
| 输入屏蔽注册机制 | UIPanel.cs | Show 时注册 RegisterInputBlocker，Hide 时注销 |
| 暂停菜单结构 | PauseMenu.cs | Resume/Settings/MainMenu/Quit 四个选项 |
| 确认对话框 | PauseMenu.cs | 独立 ConfirmationDialog 实现 |

### 2.4 Canvas 配置

| 符合项 | 实现位置 | 设计规范 | 实际值 |
|--------|---------|---------|--------|
| RenderMode | UIManager.cs | ScreenSpaceOverlay | ScreenSpaceOverlay |
| CanvasScaler | UIManager.cs | 1920x1080 参考分辨率 | 1920x1080 |
| Match Width/Height | UIManager.cs | 0.5 | 0.5 |

---

## 3. 需要修复的内容

### 3.1 动画时长修复 (P-01, P-02, P-03)

**UIPanel.cs 第 75-80 行**:
```csharp
// 当前值（错误）
protected const float DEFAULT_SHOW_DURATION = 0.3f;  // 应为 0.25f
protected const float DEFAULT_HIDE_DURATION = 0.2f;  // 应为 0.15f
```

**HUDController.cs 第 92 行**:
```csharp
// 当前值（错误）
[SerializeField]
private float _fadeDuration = 0.3f;  // 应为 0.2f
```

**AlertManager.cs 第 39 行**:
```csharp
// 当前值（错误）
[SerializeField]
private int _maxVisibleAlerts = 5;  // 应为 3
```

### 3.2 揭示动画中断逻辑 (P-04, P-05)

DiscoveryOverlay.cs 需要添加:
1. 第一次确认键检测
2. 动画立即完成逻辑
3. DiscoveryAnimationComplete 事件发送

```csharp
// 需要添加的字段
private bool _hasFirstConfirmProcessed = false;

// 需要修改 DiscoverySequenceRoutine
// 添加玩家输入检测和跳过逻辑
```

### 3.3 游戏输入屏蔽 (P-06)

当前 InputBlocker 只影响 UI 射线检测，需要扩展为可以通知游戏系统屏蔽玩家输入。

建议方案:
1. InputBlocker 添加 `IsGameplayInputBlocked` 属性
2. 添加 `OnInputBlockedChanged` 事件
3. 游戏系统订阅此事件来暂停玩家控制器

### 3.4 双输入支持 (M-01, M-03)

需要添加:
1. `InputDeviceHelper.cs` - 检测最后活跃输入设备
2. 修改 BaseMenuController 支持手柄导航
3. 添加焦点循环导航（当前代码只有 `_firstSelected`，没有导航逻辑）

### 3.5 手柄模式 HUD 布局 (M-02)

HUDController.cs 需要添加:
```csharp
// 检测当前输入设备
private bool IsGamepadActive();

// 手柄模式下重新定位 HUD 元素
private void ApplyGamepadLayout();
```

### 3.6 减少动画选项 (M-04)

需要:
1. 在设置系统中添加 `ReduceMotion` 选项
2. 在所有动画代码中检查此选项
3. 启用时将动画时长限制在 0.1s 以内

### 3.7 文本缩放系统 (M-05)

需要:
1. 全局 TextScaleFactor (0.75 ~ 2.0)
2. 基础字号 16px 应用此因子
3. UI 元素使用相对字号

---

## 4. 改进建议

### 4.1 高优先级

1. **修复动画时长**: 确保所有动画时长符合设计规范
2. **实现揭示动画跳过**: 玩家第一次确认应该立即完成动画
3. **实现 DiscoveryAnimationComplete 事件**: 解除 World Map 的输入阻塞

### 4.2 中优先级

4. **添加输入设备检测**: 检测 KB/M 和手柄切换
5. **实现手柄菜单导航**: 支持左摇杆导航，循环焦点
6. **添加手柄 HUD 布局**: 根据设计文档 OQ-3 重新定位 HUD 元素
7. **实现减少动画选项**: 在无障碍设置中提供开关

### 4.3 低优先级

8. **添加快速存档按钮**: 暂停菜单应包含快速存档入口
9. **创建无障碍设置面板**: 独立的文本大小/对比度/色盲模式设置
10. **修正 Alert 间距**: 从 10px 改为 8px

---

## 5. 附录：验收清单

### 5.1 功能验收 (基于 ui-system.md)

| ID | 验收条件 | 状态 | 备注 |
|----|---------|------|------|
| AC-1 | HUD 在游戏进行时正确显示所有必要元素 | 通过 | |
| AC-2 | 体力槽在消耗/恢复时正确更新 | 通过 | |
| AC-3 | 菜单可以正常打开/关闭 | 通过 | |
| AC-4 | 揭示动画期间输入被正确屏蔽 | **未通过** | 需要修复 P-06 |
| AC-5 | 揭示动画完成后输入正确恢复 | **未通过** | 需要修复 P-05 |
| AC-6 | 多个警告可以正确堆叠显示 | 通过 | 但数量超限 |

### 5.2 集成验收 (基于 ui-system.md)

| ID | 验收条件 | 状态 | 备注 |
|----|---------|------|------|
| AC-7 | 与 World Map 系统正确同步 | **未通过** | 事件未发送 |
| AC-8 | 与 Sanity/Rage 系统正确同步 | 通过 | HUD 有对应组件 |
| AC-9 | 与 Audio 系统正确同步 | 通过 | 事件已发布 |
| AC-10 | 与 Clue 系统正确同步 | 通过 | ClueCounter 已实现 |

### 5.3 UX 验收 (基于 ui-ux-spec.md)

| 验收条件 | 状态 | 备注 |
|---------|------|------|
| 所有菜单可通过键盘完全操作 | 部分通过 | 只有按钮点击 |
| 所有菜单可通过手柄完全操作 | **未通过** | 无手柄导航 |
| 焦点导航符合规范 | **未通过** | 缺少导航逻辑 |
| 手柄模式下 HUD 正确重新定位 | **未通过** | 无此功能 |
| Alert 警告正确堆叠（最多 3 个） | **未通过** | 最多 5 个 |
| 揭示动画期间输入正确屏蔽 | **未通过** | 只屏蔽 UI |
| 揭示动画可被跳过 | **未通过** | 无此功能 |
| 辅助功能：文本大小可调 | **未通过** | 无此功能 |
| 辅助功能：减少动画模式 | **未通过** | 无此功能 |

---

## 6. 总结

UI 系统实现整体架构良好，遵循了设计文档的层级结构和事件总线模式。但存在以下关键问题需要修复:

1. **动画时长与设计不符** - 影响玩家体验的流畅性
2. **揭示动画跳过逻辑缺失** - 影响玩家交互自由
3. **游戏输入屏蔽不完整** - 菜单打开时游戏逻辑仍在运行
4. **双输入支持缺失** - 手柄用户无法正常使用菜单
5. **手柄 HUD 布局缺失** - 影响手柄操作舒适度

建议优先修复 P-04、P-05、P-06 三个严重问题，这些直接影响游戏的核心体验。
