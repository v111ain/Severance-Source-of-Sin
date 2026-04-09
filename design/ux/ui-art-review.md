# UI 视觉规范审查报告

> **审查日期**: 2026-04-08
> **审查人**: Art Director
> **审查范围**: Assets/UI/ 目录下的所有代码文件
> **依据文档**:
> - design/ux/ui-visual-spec.md (视觉设计规范)
> - design/gdd/ui-system.md (UI 系统设计)

---

## 1. 视觉规范符合度分析

### 1.1 色调规范

| 规范定义 | 设计文档值 | 发现问题 |
|---------|-----------|---------|
| 正常文字 #FFFFFF | #FFFFFF, #ECF0F1 | 部分符合 |
| 警告文字 #F4A261 | #F4A261 近似值 | 部分符合 |
| 错误/危险 #E63946 | #E63946 近似值 | 部分符合 |
| 提示文字 #95A5A6 | 灰色调 | 未明确定义 |
| 背景遮罩 #000000 (60%) | Color.black @ 60% | 符合 |

**详细分析:**

#### HealthDisplay.cs (第 70-76 行)
```csharp
_normalColor = new Color(0.2f, 0.7f, 0.3f);      // RGB: 51,179,76 → #33B34C，非规范 #2ECC71
_lowHealthColor = new Color(0.9f, 0.7f, 0.2f);   // RGB: 230,179,51 → #E6B333，非规范 #F4A261
_criticalHealthColor = new Color(0.8f, 0.2f, 0.2f); // RGB: 204,51,51 → #CC3333，非规范 #E63946
```
**问题**: 颜色值使用浮点数硬编码，与规范 Hex 值存在偏差，导致色调不准确。

#### ClueCounter.cs (第 349-359 行)
```csharp
private Color GetNormalColor() => new Color(0.3f, 0.5f, 0.8f);   // RGB: 77,128,204 → #4D80CC
private Color GetCompleteColor() => new Color(0.2f, 0.7f, 0.3f);  // RGB: 51,179,76 → #33B34C
```
**问题**: 线索计数器的正常颜色和完成颜色完全自定义，未使用规范定义的颜色系统。

#### Crosshair.cs (第 82-92 行)
```csharp
_normalColor = Color.white;                          // 符合 #FFFFFF
_targetingColor = new Color(0.9f, 0.7f, 0.2f);       // 接近 #F4A261 ✓
_lockedColor = new Color(0.8f, 0.2f, 0.2f);          // 接近 #E63946 ✓
_reloadingColor = new Color(0.3f, 0.5f, 0.8f);       // 自定义蓝色，无规范定义
```
**问题**: 瞄准和锁定状态颜色基本符合，但重新装填状态颜色无规范定义。

#### FocusModeUI.cs (第 96-99 行)
```csharp
_vignetteIntensity = 0.6f;   // 60% 符合规范
_vignetteColor = Color.black; // 应用时为 #000000，但规范定义 #0D0D0D
```
**问题**: 暗角颜色使用 Color.black (#000000) 而非规范定义的深夜黑 #0D0D0D。

#### AlertManager.cs (第 634-635 行)
```csharp
bg.color = new Color(0.1f, 0.1f, 0.1f, 0.9f); // RGB: 26,26,26 → #1A1A1A
```
**问题**: 动态创建的警报背景色为 #1A1A1A，而规范定义 HUD 背景色为 #1A1F2E。

---

### 1.2 字体规范

| 检查项 | 规范定义 | 实现情况 | 状态 |
|-------|---------|---------|------|
| 主字体 | Noto Sans CJK / Inter | 代码中未硬编码字体 | **待确认** |
| H1 标题 | 32px Bold | 未在代码中强制指定 | **待确认** |
| H2 面板标题 | 24px Bold | 未在代码中强制指定 | **待确认** |
| Body 正文 | 16px Regular | 未在代码中强制指定 | **待确认** |
| Caption | 12px | 未在代码中强制指定 | **待确认** |
| 数字等宽 | Inter Medium + tabular-nums | 未在代码中强制指定 | **待确认** |

**问题**: 代码层面未定义字体规范，字体回退链未实现。需要在实际 Text/GUI.Text 组件上配置，或通过全局样式表定义。

---

### 1.3 间距系统

| 检查项 | 规范定义 | 实现情况 | 状态 |
|-------|---------|---------|------|
| 基准间距 | 8px | 常量定义完整 | **符合** |
| HUD 边缘安全区 | 20px | UIManager.cs 第 189 行 referenceResolution | **符合** |
| Alert 间距 | 8px | AlertManager.cs 第 72 行 spacing = 10f | **轻微偏差** |
| Alert 最大宽度 | 400px | AlertManager.cs 第 75 行 _alertWidth = 400 | **符合** |

**AlertManager.cs (第 72 行)**: `spacing = 10f` 而非规范的 8px。

---

### 1.4 动效规范

| 动画类型 | 规范时长 | 代码实现 | 状态 |
|---------|---------|---------|------|
| HUD 淡入 | 0.2s | HUDController.cs 第 92 行 _fadeDuration = 0.3f | **偏差 +0.1s** |
| HUD 淡出 | 0.15s | 未单独定义，使用 UIPanel 默认 | **偏差** |
| 菜单出现 | 0.25s | BaseMenuController.cs 第 139 行 DEFAULT_OPEN_DURATION = 0.3f | **偏差 +0.05s** |
| 专注模式暗角 | 0.5s | FocusModeUI.cs 第 102 行 _focusTransitionDuration = 0.5f | **符合** |
| Alert 出现 | 0.2s | AlertManager.cs 第 83 行 _showAnimationDuration = 0.3f | **偏差 +0.1s** |
| 按钮悬停 | 0.1s | 代码中未定义按钮悬停动画 | **待实现** |

**UIPanel.cs (第 75-80 行)**:
```csharp
protected const float DEFAULT_SHOW_DURATION = 0.3f; // 规范 0.2s
protected const float DEFAULT_HIDE_DURATION = 0.2f; // 规范 0.15s
```

**UIAnimationHelper.cs (第 427-452 行)**: 扩展方法默认时长均为 0.3f，与规范不符。

---

## 2. 发现的问题列表

### 2.1 严重问题 (影响视觉一致性)

| # | 问题描述 | 位置 | 修复建议 |
|---|---------|------|---------|
| P-01 | 颜色值使用浮点数硬编码，与设计规范 Hex 值存在系统偏差 | HealthDisplay.cs, ClueCounter.cs, Crosshair.cs | 创建全局颜色常量类，统一管理所有 UI 颜色 |
| P-02 | 专注模式暗角使用 Color.black 而非 #0D0D0D | FocusModeUI.cs | 修正为 #0D0D0D (深夜黑) |
| P-03 | HUD 整体淡入时长 0.3s 而非规范的 0.2s | HUDController.cs, UIPanel.cs | 调整所有 HUD 面板淡入时长至 0.2s |

### 2.2 中等问题 (影响部分体验)

| # | 问题描述 | 位置 | 修复建议 |
|---|---------|------|---------|
| P-04 | 菜单出现动画 0.3s 而非规范的 0.25s | BaseMenuController.cs | 调整 DEFAULT_OPEN_DURATION 至 0.25s |
| P-05 | Alert 间距 10px 而非规范的 8px | AlertManager.cs | 调整 spacing 至 8px |
| P-06 | Alert 出现动画 0.3s 而非规范的 0.2s | AlertManager.cs | 调整 _showAnimationDuration 至 0.2s |
| P-07 | 动态创建的 Alert 背景色 #1A1A1A 而非 #1A1F2E | AlertManager.cs | 修正 CreateAlertDynamically 方法 |

### 2.3 轻微问题 (待确认)

| # | 问题描述 | 位置 | 修复建议 |
|---|---------|------|---------|
| P-08 | 代码层面未定义字体回退链 | 所有 Text 组件 | 确认项目字体配置或添加运行时字体回退 |
| P-09 | Crosshair._reloadingColor 使用自定义蓝色，无规范对应 | Crosshair.cs | 确认是否需要添加重装填状态规范 |
| P-10 | 缺少专注模式呼吸动画的循环实现 | FocusModeUI.cs | 确认专注模式波纹效果是否需要呼吸动画 |

---

## 3. 资源检查

### 3.1 命名规范检查

**符合项**:
- 文件命名使用 PascalCase (例如: `HealthDisplay.cs`, `AlertManager.cs`)
- 目录结构清晰分层 (Core/, HUD/, Menus/, Alerts/, FocusMode/, Overlays/)

**待确认项**:
- 需确认 Prefab 和资产文件是否符合 `[category]_[name]_[variant]_[size].[ext]` 命名规范

### 3.2 材质/纹理规范

- 代码中未硬编码纹理引用，依赖 Inspector 配置
- AlertManager 动态创建 UI 时使用纯色背景，未使用纹理
- FocusModeUI 的暗角效果依赖 Image 组件，未使用着色器

### 3.3 字体回退链

代码中 **未实现** 字体回退链。建议:
```
中文字体: "Noto Sans CJK SC" → "Source Han Sans SC" → "Microsoft YaHei" → "PingFang SC" → "SimHei"
英文字体: "Inter" → "Helvetica Neue" → "Helvetica" → "Arial"
数字字体: "Inter" → "Roboto Mono" → "Consolas"
```

---

## 4. 一致性检查

### 4.1 HUD 与菜单视觉风格一致性

**符合项**:
- 都使用 CanvasGroup 进行透明度管理
- 都支持淡入淡出动画
- 都遵循 UIPanel 状态机

**偏差项**:
- 颜色定义分散在各个组件中，未统一管理
- 动画时长参数不统一

### 4.2 UI 与游戏整体艺术风格协调

**符合项**:
- 整体风格为简洁、功能导向，符合犯罪/黑暗主题定位
- 透明度背景使用得当，不遮挡游戏内容

**需注意**:
- 部分颜色偏向明亮(如 HealthDisplay 的绿色)，需确认是否符合整体暗色调

---

## 5. 分辨率适配

### 5.1 当前实现

**UIManager.cs (第 187-191 行)**:
```csharp
scaler.uiScaleMode = CanvasScaler.ScaleMode.ScaleWithScreenSize;
scaler.referenceResolution = new Vector2(1920, 1080);
scaler.matchWidthOrHeight = 0.5f; // 宽高均等适配
```

### 5.2 适配评估

| 检查项 | 状态 | 说明 |
|-------|------|------|
| 参考分辨率 | **符合** | 1920x1080 符合规范 |
| 适配模式 | **符合** | ScaleWithScreenSize + 0.5 匹配 |
| 最小分辨率支持 | **待测试** | 未设置 minLogicalHeight/Width |
| 最大分辨率支持 | **待测试** | 未明确设置上限 |
| 相对布局 | **符合** | 使用 Anchor/Pivot 而非绝对坐标 |

---

## 6. 修复建议优先级

### 第一优先级 (P-01, P-02, P-03)
创建全局 `UIColors.cs` 颜色常量类，统一所有 UI 颜色值:
```csharp
public static class UIColors
{
    // 主色调
    public static readonly Color DeepNightBlack = new Color(0.05f, 0.05f, 0.05f); // #0D0D0D
    public static readonly Color SmokeBlue = new Color(0.1f, 0.12f, 0.18f);      // #1A1F2E
    public static readonly Color SteelGray = new Color(0.24f, 0.27f, 0.33f);    // #3D4654

    // 文字色
    public static readonly Color InterrogationWhite = new Color(0.93f, 0.94f, 0.95f); // #ECF0F1
    public static readonly Color NormalWhite = Color.white;                          // #FFFFFF
    public static readonly Color HintGray = new Color(0.58f, 0.65f, 0.65f);          // #95A5A6

    // 状态色
    public static readonly Color NeonRed = new Color(0.9f, 0.22f, 0.27f);    // #E63946
    public static readonly Color NeonYellow = new Color(0.96f, 0.64f, 0.38f); // #F4A261
    public static readonly Color NeonGreen = new Color(0.18f, 0.8f, 0.44f);  // #2ECC71
    public static readonly Color NeonBlue = new Color(0.2f, 0.6f, 0.86f);   // #3498DB
}
```

### 第二优先级 (P-04, P-05, P-06, P-07)
调整动效参数和间距至规范值。

### 第三优先级 (P-08, P-09, P-10)
字体回退链实现和状态颜色规范确认。

---

## 7. 资源需求更新

### 7.1 需要创建的资源

| 资源名 | 类型 | 用途 | 优先级 |
|-------|------|------|--------|
| UIColors.cs | C# 常量类 | 统一管理所有 UI 颜色 | **高** |
| UIConstants.cs | C# 常量类 | 统一管理动画时长、间距等常量 | **高** |
| UIFontStyles.cs | C# 样式类 | 定义字体层级和回退链 | 中 |

### 7.2 需要更新的着色器

| 着色器名 | 现状 | 更新需求 | 优先级 |
|---------|------|---------|--------|
| 暗角着色器 | FocusModeUI 使用 Image 组件 | 确认是否需要专用径向渐变着色器 | 低 |

---

## 8. 总结

### 符合度评分

| 类别 | 符合度 | 说明 |
|-----|-------|------|
| 色调规范 | 65% | 颜色值偏差较大，需系统化整改 |
| 字体规范 | N/A | 代码未定义，需配合项目字体配置 |
| 间距系统 | 85% | 基本符合，仅 Alert 间距轻微偏差 |
| 动效规范 | 60% | 时长偏差明显，需全面调整 |
| 分辨率适配 | 90% | 基础适配完善，需测试边界情况 |

### 建议行动

1. **立即创建** `UIColors.cs` 和 `UIConstants.cs` 统一管理常量
2. **审查** 所有使用颜色的组件，替换为新的常量引用
3. **调整** 动效时长参数至规范值
4. **确认** 项目字体配置和回退链实现
5. **测试** 不同分辨率下的 UI 表现

---

**报告状态**: 待审核
**下一步**: 与 UI Programmer 确认修复计划
