# Immersive Audio & Haptics System (沉浸式音频与震动系统)

> **Status**: Approved
> **Author**: Sound Designer Agent
> **Last Updated**: 2026-04-14
> **Engine**: Unity 6.3 LTS
> **Implements Pillar**: 沉浸式体验 (Immersive Experience)、沉重不洁的暴力 (Gritty Violence)
> **Revision Notes (2026-04-15)**: P1修复 - NPCStateChangedEvent 来源修正：由 Health 系统更正为 NPC AI 系统（NPC 死亡、警觉等状态变更是 NPC AI 系统的职责，Health 系统只负责生命值变化计算）
>
> **Revision Notes (2026-04-14)**: P0修复 - 澄清震动技术方案与平台支持能力；P1修复 - PriorityDifference定义、震动降级参数完善

## 1. Overview

沉浸式音频与震动系统 (Immersive Audio & Haptics System) 负责为游戏提供全方位的声音反馈和触感震动体验，是连接玩家操作与感官反馈的核心系统。

系统基于 Unity 内置音频系统（AudioSource/AudioListener）实现，无需第三方中间件（如 FMOD/Wwise）。震动反馈支持手柄和移动端设备，PC 端不强制要求震动功能。

**核心职责**：
- 为所有游戏系统（处决、环境交互、UI、NPC 行为等）提供音效触发
- 为手柄和移动端提供震动反馈
- 管理音频混音、优先级和并发控制
- 提供音量、震动强度的运行时调节能力

**设计约束**：
- 使用 Unity 内置音频系统（AudioSource/AudioListener）
- 震动功能：手柄（Xbox/PlayStation/Switch Pro）+ 移动端（iOS/Android）
- PC 震动：不强制支持，可选实现
- 音频格式：OGG/VAG（音乐/环境音）、WAV（短音效）

---

## 2. Player Fantasy

**核心情感目标：触达灵魂的临场感**

音频与震动不是简单的"按键反馈"，而是玩家与游戏世界之间最直接的感官桥梁。每一次枪声、每一次爆炸、每一次潜行击杀，都通过声音和震动传递情感重量，让玩家真正"感受"到他们创造的暴力与紧张。

**体验层次**：

- **物理层**：子弹击中墙壁的清脆声、金属碰撞的尖锐声——这些声音建立物理世界的可信度
- **情感层**：处决时的闷响与随之而来的寂静、危险逼近时的心跳加速——这些声音传递游戏的情感脉搏
- **反馈层**：手柄的每一次震动、屏幕的每一次颤抖——这些反馈确认玩家的每一个操作

**参考对标**：
- 《The Last of Us Part II》的音效设计——每一次近战搏斗都能感受到冲击
- 《Hades》的震动反馈——每次攻击都有恰到好处的触感回应
- 《Dead Space》的环境音频——寂静中的突然声响制造持续紧张

**震动反馈的哲学**：

震动不是越多越好。过度的震动会令玩家疲劳、失去敏感度。正确的震动设计遵循"少而精"原则：
- 只有重要的、有冲击力的时刻才触发震动
- 震动强度与事件重要性成正比
- 手游震动需要考虑设备续航，避免连续高频震动

---

## 3. Detailed Design

### 3.1 Audio Architecture (音频架构)

#### 3.1.1 音频总线路由

```
[音频源] → [音效总线 (SFX Bus)] ──┬──→ [主总线 (Master Bus)]
                                  ├──→ [环境音总线 (Ambient Bus)] ──→ [主总线]
                                  ├──→ [对话总线 (Dialogue Bus)] ──→ [主总线]
                                  └──→ [UI 总线 (UI Bus)] ──→ [主总线]
```

**总线定义**：

| 总线名称 | 内容 | 目标电平 | 备注 |
|---------|------|---------|------|
| Master Bus | 所有音频混合输出 | -3 dBFS | 挂载限制器防止削波 |
| SFX Bus | 所有游戏音效 | -6 dBFS | 动态范围压缩 (Ratio 4:1) |
| Ambient Bus | 环境音、背景音 | -9 dBFS | 低通滤波保留氛围感 |
| Dialogue Bus | NPC 对话、旁白 | -6 dBFS | 通话时优先占用 |
| UI Bus | 界面交互音 | -12 dBFS | 可独立静音 |

#### 3.1.2 音效分类与优先级

| 优先级 | 类别 | 示例音效 | 混音规则 |
|--------|------|---------|---------|
| P0 (最高) | 玩家受伤/死亡 | 玩家受到伤害、玩家死亡 | 打断所有其他音效，短暂全频段通过 |
| P1 | 处决反馈 | 潜行击杀、环境处决 | 音效完成后恢复混音 |
| P2 | 警报/危险 | NPC 发现玩家、触发陷阱 | 音乐 ducking 6 dB |
| P3 | 玩家动作 | 移动、潜行、环境交互 | 音量较低，不影响环境感知 |
| P4 | 环境音效 | 物件破碎、爆炸、门关闭 | 3D 定位，空间混音 |
| P5 (最低) | 环境氛围 | 背景音、风声、远处 NPC 对话 | 持续循环，无需打断 |

#### 3.1.3 并发控制

**同类别音效并发限制**：

| 类别 | 最大并发数 | 超出处理 |
|------|-----------|---------|
| 脚步音 | 2 | 淡出旧音效，播放新音效 |
| 破碎/撞击 | 3 | 停止最旧的，播放新的 |
| 爆炸 | 2 | 队列播放，间隔 100ms |
| UI 点击 | 4 | 允许堆叠 |

**跨类别冲突**：高优先级音效立即抢占，低优先级音效淡出。

### 3.2 SFX Specifications (音效规格)

> **持续时间格式说明**：
> - **范围值**（如 `0.3-0.5s`）：表示该音效持续时间在给定范围内随机，取决于音效变体和播放条件
> - **固定值**（如 `0.15s`）：表示该音效持续时间固定
> - **`1 帧`**：表示该音效持续时间仅为单帧（约 16.67ms @ 60fps），用于瞬时反馈音效
> - **`持续`**：表示该音效为持续循环音效，无固定结束时间

#### 3.2.1 处决系统音效

| 音效名称 | 描述 | 频率特性 | 持续时间 | 音量范围 | 空间属性 | 变体数量 |
|---------|------|---------|---------|---------|---------|---------|
| stealth_kill | 潜行击杀——刀刃划过喉咽的闷响 | 低频为主 (80-200Hz)，无高频 | 0.3-0.5s | 60-70% | 2D，正中央 | 3 |
| stealth_kill_vocal | 目标最后一声闷哼 | 低频 (100-300Hz)，气息声 | 0.2-0.4s | 40-50% | 3D，攻击位置 | 3 |
| environment_execute | 环境处决——撞击/碎裂声 | 宽频段冲击，低频厚实 | 0.5-1.0s | 80-100% | 3D，攻击位置 | 4 |
| execute_hit_stop | 处决命中瞬间——顿帧音效 | 纯低频脉冲 (50Hz) | 1 帧 | 100% | 2D | 1 |
| finish_off | 补刀——终结一击 | 中频冲击 (200-500Hz) | 0.2-0.3s | 70-85% | 3D，目标位置 | 3 |
| tie_up | 捆绑——绳索收紧声 | 低频摩擦声 + 高频扣紧声 | 1.0-2.0s | 30-40% | 3D，NPC 位置 | 2 |

#### 3.2.2 环境交互系统音效

| 音效名称 | 描述 | 频率特性 | 持续时间 | 音量范围 | 空间属性 | 变体数量 |
|---------|------|---------|---------|---------|---------|---------|
| object_highlight | 物件进入交互范围 | 低频嗡鸣 (150Hz) | 0.15s | 25-30% | 2D，正中央 | 2 |
| pickup_metal | 拾取金属物件 | 清脆金属碰撞声 | 0.2-0.3s | 50-60% | 3D，物件位置 | 3 |
| pickup_wood | 拾取木质物件 | 沉闷木质声 | 0.2-0.3s | 45-55% | 3D，物件位置 | 3 |
| pickup_glass | 拾取玻璃物件 | 清脆叮声 | 0.15-0.25s | 55-65% | 3D，物件位置 | 2 |
| place_object | 放置物件 | 沉闷落地声 | 0.3-0.4s | 60-70% | 3D，物件位置 | 2 |
| throw_ready | 投掷物准备就绪 | 短促提示音 | 0.1s | 35% | 2D，正中央 | 1 |
| explosion_small | 小型爆炸（灭火器） | 中等冲击，衰减快 | 0.5-0.8s | 85-95% | 3D，爆炸位置 | 3 |
| explosion_large | 大型爆炸（汽油桶） | 强冲击，低频厚实 | 1.0-1.5s | 95-100% | 3D，爆炸位置 | 3 |
| destructible_glass | 玻璃破碎 | 尖锐高频碎片声 | 0.4-0.6s | 60-70% | 3D，破碎位置 | 4 |
| destructible_wood | 木头破碎 | 噼啪断裂声 | 0.5-0.8s | 55-65% | 3D，破碎位置 | 3 |
| destructible_light | 灯泡打碎 | 电流滋滋 + 玻璃破碎 | 0.3-0.5s | 50-60% | 3D，破碎位置 | 2 |
| door_open | 门打开 | 铰链吱呀声 | 0.5-1.0s | 40-50% | 3D，门位置 | 3 |
| door_close | 门关闭 | 沉闷关闭声 | 0.3-0.5s | 45-55% | 3D，门位置 | 3 |

#### 3.2.3 UI 交互音效

| 音效名称 | 描述 | 频率特性 | 持续时间 | 音量范围 | 变体数量 |
|---------|------|---------|---------|---------|---------|
| ui_click | 菜单点击确认 | 中频清脆 | 0.05-0.1s | 40% | 3 |
| ui_hover | 选项悬停 | 极轻提示音 | 0.05s | 20% | 2 |
| ui_back | 返回/取消 | 低沉短促 | 0.08s | 35% | 2 |
| ui_success | 操作成功 | 上升音调 | 0.2-0.3s | 50% | 2 |
| ui_fail | 操作失败 | 下降音调 | 0.2-0.3s | 45% | 2 |
| radial_menu_open | 径向菜单展开 | 机械展开音 | 0.15s | 45% | 1 |
| radial_menu_select | 径向菜单选择 | 轻微咔嗒 | 0.05s | 30% | 1 |

#### 3.2.4 NPC 行为音效

| 音效名称 | 描述 | 频率特性 | 持续时间 | 音量范围 | 空间属性 | 触发条件 |
|---------|------|---------|---------|---------|---------|---------|
| npc_footstep | NPC 脚步声 | 低频冲击 + 摩擦 | 0.2-0.3s | 30-50% | 3D，NPC 位置 | NPC 移动时 |
| npc_notice | NPC 感知异常 | 警觉提示音 | 0.3-0.5s | 50-60% | 3D，NPC 位置 | SUSPECT 状态 |
| npc_alert | NPC 发现玩家 | 警报声 | 0.5-1.0s | 70-80% | 3D，NPC 位置 | ALERT 状态 |
| npc_combat | 战斗状态 | 激烈音效 | 持续 | 75-85% | 3D，NPC 位置 | COMBAT 状态 |
| npc_death | NPC 死亡 | 沉闷倒地 + 呻吟 | 1.0-2.0s | 60-70% | 3D，NPC 位置 | NPC 死亡 |
| npc_groan | NPC 不适/疼痛 | 低沉呻吟 | 0.5-1.0s | 35-45% | 3D，NPC 位置 | NPC 受创 |

### 3.3 Haptics Specifications (震动规格)

#### 3.3.1 震动设备支持矩阵

| 设备类型 | 支持状态 | 震动引擎 | 备注 |
|---------|---------|---------|------|
| PlayStation 5 DualSense | **MVP阶段简化支持** | Unity InputSystem + Haptics API | **自适应扳机仅在Full Vision阶段支持**。MVP阶段使用 `InputSystem.Haptic` 的基础震动脉冲，不支持自适应扳机的精细阻力控制 |
| Xbox Series Controller | 完整支持 | Unity Input.GetJoystickVibration() | 需测试不同型号 |
| Nintendo Switch Pro | 完整支持 | Unity Input.GetJoystickVibration() | Joy-Con 独立支持 |
| iOS (iPhone 8+) | **MVP阶段简化支持** | Unity Handheld.Vibrate() | **精细震动曲线(Pulse_Rumble/Continuous_Low)仅在Full Vision阶段支持**。MVP阶段仅支持单次脉冲震动 |
| Android (中高端) | **MVP阶段简化支持** | Unity Handheld.Vibrate() | 低端机可能降级。精细震动曲线同上 |
| PC (键盘鼠标) | 不支持 | N/A | 可选实现鼠标震动 |

> **技术方案澄清 (2026-04-14)**：
> - **MVP阶段**：所有平台统一使用基础震动API（`Handheld.Vibrate()` / `Input.GetJoystickVibration()`），仅支持单次脉冲震动
> - **Full Vision阶段**：根据平台特性升级到原生API（PS5 DualSense Core Haptics、iOS Core Haptics），实现 `Pulse_Rumble`、`Continuous_Low` 等精细震动曲线
> - **震动模式降级规则**：MVP阶段 `Pulse_Rumble` → `Pulse_Single`（单次脉冲），`Continuous_Low` → `Haptic_Light`（短时震动）

#### 3.3.2 震动强度分级

| 等级 | 强度范围 | 适用场景 | 持续时间 |
|------|---------|---------|---------|
| Haptic_Light | 0-25% | UI 反馈、轻微接触 | 30-50ms |
| Haptic_Medium | 30-50% | 玩家移动、潜行 | 50-100ms |
| Haptic_Heavy | 60-80% | 处决命中、环境爆炸 | 100-200ms |
| Haptic_Critical | 90-100% | 玩家受伤、死亡 | 200-400ms |

#### 3.3.3 震动模式定义

| 模式名称 | 强度曲线 | 描述 | 适用场景 |
|---------|---------|------|---------|
| Pulse_Single | 固定强度，单次脉冲 | 最基础的震动模式 | UI 点击 |
| Pulse_Double | 两次短促脉冲，间隔 100ms | 确认感更强 | 物品拾取 |
| Pulse_Rumble | 逐渐增强后衰减 | 渐入渐出 | 爆炸接近 |
| Continuous_Low | 持续低频震动 | 长时反馈 | 玩家移动 |
| Impact_Burst | 瞬间最大强度后快速衰减 | 冲击感 | 撞击、处决 |

#### 3.3.4 震动与音效对应表

> **MVP阶段降级说明**：
> - `Pulse_Rumble`（爆炸接近）在MVP阶段降级为 `Pulse_Single`（单次脉冲），强度 60-80%（小型）/ 90-100%（大型）
> - `Continuous_Low`（持续低频）在MVP阶段降级为 `Haptic_Light`（短时震动），强度 0-25%，持续时间 30-50ms
> - `Impact_Burst + Continuous_Low` 组合（玩家受伤）在MVP阶段降级为 `Impact_Burst + Haptic_Light`
> - Full Vision阶段将根据平台原生API实现完整的精细震动曲线

| 事件 | 震动等级 | 震动模式（MVP降级后） | 持续时间 | 与音效同步 |
|------|---------|---------|---------|-----------|
| 潜行击杀 | Haptic_Heavy | Pulse_Single | 150ms | 是，命中瞬间 |
| 环境处决 | Haptic_Critical | Impact_Burst | 250ms | 是，命中瞬间 |
| 补刀 | Haptic_Heavy | Pulse_Single | 100ms | 是 |
| 玩家受伤 | Haptic_Critical | Impact_Burst + Haptic_Light | 300ms + 200ms | 是 |
| 玩家死亡 | Haptic_Critical | 连续 3 次 Impact_Burst | 各 200ms，间隔 100ms | 是 |
| 爆炸（小型） | Haptic_Heavy | Pulse_Single（降级后） | 200ms，强度 60-80% | 是 |
| 爆炸（大型） | Haptic_Critical | Pulse_Single（降级后） | 400ms，强度 90-100% | 是 |
| 物件拾取 | Haptic_Light | Pulse_Double | 80ms | 是 |
| 物件放置 | Haptic_Medium | Pulse_Single | 100ms | 是 |
| 门打开 | Haptic_Light | Pulse_Single | 50ms | 是 |
| 脚步（玩家） | Haptic_Light | Haptic_Light（降级后） | 每步 30-50ms，强度 0-25% | 与脚步动画同步 |
| UI 点击 | Haptic_Light | Pulse_Single | 30ms | 与音效同步 |

#### 3.3.5 Haptics 与其他系统同步机制

**同步原则**：
震动反馈与音效/视觉反馈的同步误差必须控制在 30ms 以内（AC-H5 验收标准）。

**Audio系统震动与屏幕特效同步**：
Audio系统触发震动时，通过事件总线发送 `ShakeRequest` 事件（与DPP系统一致），由Screen Effects系统订阅并处理屏幕震动效果。**不存在 `ScreenEffectsSystem.TriggerShake()` 方法**，所有调用方必须通过事件总线发送 `ShakeRequest` 事件。

**同步实现**：

| 系统对 | 同步方式 | 关键参数 |
|--------|---------|---------|
| 震动 ↔ 音效 | 同一事件触发，震动命令在音效播放调用时同步发送 | 音效播放 API 调用点即为震动触发点 |
| 震动 ↔ 屏幕特效 | 震动触发时通过事件总线发送 `ShakeRequest` | 使用 Screen Effects 系统的 `ShakeRequest` 事件（无 TriggerShake 方法） |
| 震动 ↔ 慢动作 | 处决命中时：震动 250ms → 屏幕特效延迟 50ms → 恢复游戏 | 时序链：Haptic → ShakeRequest → TimeScale |

> **接口修正 (2026-04-14)**：`ScreenEffectsSystem.TriggerShake()` 方法不存在。正确调用方式是通过事件总线发送 `ShakeRequest` 事件，由 Screen Effects 系统订阅并处理。

**边缘情况处理**：

| 场景 | 处理方式 |
|------|---------|
| 震动触发但音效未播放 | 震动正常触发，不等待音效 |
| 音效播放但震动失败 | 记录错误日志，音效继续播放 |
| 设备不支持震动 | 自动降级，音效/视觉反馈不受影响 |

**验证方法**：
- AC-H5：慢动作录像逐帧分析，震动与音效误差 < 30ms
- AC-H6：低电量设备自动降级验证

### 3.4 Ambience System (氛围音系统)

#### 3.4.1 环境音层结构

每个关卡/场景的环境音由多个音层叠加构成：

```
[环境音] = [Base Layer] + [Detail Layer] + [One-Shot Layer]
```

| 层级 | 描述 | 示例 | 音量占比 |
|------|------|------|---------|
| Base Layer | 持续背景音 | 风声、空调声、远处城市 | 40-50% |
| Detail Layer | 周期性细节音 | 水滴、吱呀声、间歇性对话 | 20-30% |
| One-Shot Layer | 随机触发音 | 突然的门声、远处的喊叫 | 10-20% |

#### 3.4.2 关卡环境音配置

| 关卡类型 | Base Layer | Detail Layer | One-Shot 频率 |
|---------|-----------|--------------|---------------|
| 废弃仓库 | 风声、滴水、金属嘎吱声 | 每 5-15s 掉落金属 | 每 10-30s 远处门声 |
| 地下通道 | 水流声、低频嗡鸣 | 每 3-8s 滴水 | 每 15-45s 远处脚步声 |
| 办公区域 | 空调声、日光灯嗡嗡 | 每 10-20s 纸张翻动 | 每 20-60s 打印机声 |
| 室外夜间 | 虫鸣、风声、远处犬吠 | 每 5-10s 猫叫 | 每 30-90s 车辆经过 |

---

## 4. Formulas

### 4.1 音量衰减公式

**3D 空间音效音量计算**：

```
DistanceAttenuation = 1.0 / (1.0 + DistanceToListener * DistanceCoefficient)
EffectiveVolume = BaseVolume * DistanceAttenuation * EnvironmentMultiplier
```

| 变量 | 定义 | 默认值 | 备注 |
|------|------|--------|------|
| DistanceToListener | 声源到玩家的直线距离 | 0 - ∞ | 单位：米 |
| DistanceCoefficient | 距离衰减系数 | 0.2 | 越大衰减越快 |
| BaseVolume | 音效原始音量 | 0.0 - 1.0 | 资源预设值 |
| EnvironmentMultiplier | 环境遮蔽系数 | 0.5 - 1.0 | 墙壁/障碍物阻隔 |

**示例计算**：
- 爆炸发生在玩家 5 米处，DistanceCoefficient = 0.2
- DistanceAttenuation = 1.0 / (1.0 + 5 * 0.2) = 1.0 / 2.0 = 0.5
- 如果 BaseVolume = 1.0，EffectiveVolume = 0.5

### 4.2 震动强度缩放公式

**根据事件重要性调整震动强度**：

```
ScaledIntensity = Clamp(BaseIntensity * EventMagnitudeMultiplier * DeviceCapabilityMultiplier, 0.0, 1.0)
```

| 变量 | 定义 | 范围 | 备注 |
|------|------|------|------|
| BaseIntensity | 预设震动强度 | 0.0 - 1.0 | 根据事件类型预设 |
| EventMagnitudeMultiplier | 事件规模乘数 | 0.5 - 2.0 | 大型爆炸 > 小型爆炸 |
| DeviceCapabilityMultiplier | 设备能力系数 | 0.4 - 1.0 | 高端设备全速，低端设备降低 |

**设备能力系数参考值**：

| 设备类型 | 系数 | 原因 |
|---------|------|------|
| PS5 DualSense | 1.0 | 最新硬件，完全支持 |
| Xbox Elite | 0.9 | 强劲震动性能 |
| Xbox Standard | 0.8 | 标准震动 |
| Switch Pro | 0.7 | 震动强度较低 |
| iPhone 14+ | 0.9 | Core Haptics 完全支持 |
| Android 高端 | 0.7-0.9 | 差异较大 |
| Android 中端 | 0.4-0.6 | 降级使用 |

### 4.3 音效淡入淡出公式

**平滑过渡防止音频突变**：

```
CurrentVolume = Lerp(CurrentVolume, TargetVolume, 1.0 - ExpDecay(TimeSinceChange, HalfLife))
```

| 变量 | 定义 | 典型值 |
|------|------|--------|
| TargetVolume | 目标音量 | 0.0 - 1.0 |
| TimeSinceChange | 自变化以来的时间 | 0 - ∞ (秒) |
| HalfLife | 半衰期（音量减半所需时间） | 0.1 - 0.5s |

**淡出触发条件**：
- 高优先级音效抢占总线：低优先级淡出 100ms
- 玩家进入菜单：所有音效淡出 200ms
- 场景切换：所有音效淡出 500ms

### 4.4 Ducking 公式

**高优先级音频打断低优先级时的音量压制**：

Ducking 参数统一使用 **dB 单位**，与 Tuning Knobs 中的 `DuckingReductionPerLevel = 6 dB` 保持一致。

```
TotalDuckingReduction_dB = Sum of (DuckingReductionPerLevel × PriorityDifference)
TotalDuckingLinear = 10 ^ (-TotalDuckingReduction_dB / 20)
EffectiveVolume = SourceVolume * TotalDuckingLinear
Clamp(TotalDuckingReduction_dB, 0.0, 24.0)
```

| 变量 | 定义 | 默认值 | 备注 |
|------|------|--------|------|
| `DuckingReductionPerLevel` | 每级优先级差的衰减量 | 6 dB | Tuning Knobs 中可调 |
| `PriorityDifference` | 高优先级与低优先级的级数差，取绝对值 | 1-4 | `PriorityDifference = |HighPriorityLevel - LowPriorityLevel|`。优先级定义：P0=0（最高）到P5=5（最低） |
| `TotalDuckingReduction_dB` | 总衰减量（dB） | 0 - 24 dB | Clamp 到 0-24 dB |
| `TotalDuckingLinear` | 总衰减量（线性） | 1.0 - 0.0 | 用于音量计算 |

**优先级差值映射表**：

| 优先级差 (PriorityDifference) | 衰减量 (dB) | 衰减后剩余音量 | 说明 |
|------------------------------|------------|--------------|------|
| 1 级（P0打断P1，P1打断P2...） | 6 dB | ~50% | 轻微压制 |
| 2 级（P0打断P2，P1打断P3...） | 12 dB | ~25% | 中度压制 |
| 3 级（P0打断P3，P1打断P4...） | 18 dB | ~12.5% | 明显压制 |
| 4 级（P0打断P4，P1打断P5...） | 24 dB | ~6% | 接近静音 |

> **PriorityDifference 计算说明**：
> - 由于优先级数值越小越高（P0=最高，P5=最低），差值计算使用绝对值确保结果为正
> - P0打断P5时差值为5，但实际应用中P0只会打断比它低的优先级（P1-P5），差值范围为1-4
> - 表中未列出5级差值，因为P0打断P5（差值=5）时衰减量为30dB，超过24dB上限被clamp

**示例计算**：
- P3 音效（玩家动作）被 P1 音效（处决）打断
- 优先级差 = 2 级
- `TotalDuckingReduction_dB = 2 × 6 dB = 12 dB`
- `TotalDuckingLinear = 10^(-12/20) ≈ 0.251`（约 75% 衰减）
- `EffectiveVolume = SourceVolume × 0.251`
- **注意**：12 dB 衰减后剩余约 25% 音量，而非完全静音。完全静音需要极大的 dB 值（如 60 dB）。

---

## 5. Edge Cases

### 5.1 音频边缘情况

#### 边缘情况 1：多个同优先级音效同时触发

**问题**：玩家快速连续触发多个相同类型音效（如连续两次环境交互）

**处理**：
- 检查上次同类音效播放时间
- 如果间隔 < 200ms，启动随机音调偏移（-5% 到 +5%）
- 确保不会完全相同的音效连续播放

#### 边缘情况 2：音效触发时音频设备断开

**问题**：播放音效时，音频设备（耳机）突然断开

**处理**：
- Unity 自动处理设备重连
- 如果 3 秒内未恢复，暂停所有音频输出
- 恢复后检查是否需要重播被跳过的音效（仅关键音效重播）

#### 边缘情况 3：大量环境音同时触发（爆炸连锁）

**问题**：汽油桶连锁爆炸，短时间内多个爆炸音效同时触发

**处理**：
- 设置最大并发爆炸音效 = 2
- 超出排入队列，间隔 100ms 播放
- 队列超过 5 个时，最早的音效被跳过

#### 边缘情况 4：静音模式下的音频反馈

**问题**：玩家将游戏设为静音，重要音效可能错过

**处理**：
- 关键事件（NPC 发现玩家、处决成功）必须同时有视觉反馈
- 屏幕边缘闪光（红色 = 危险，白色 = 交互）
- UI 提示文字（如"目标已标记"）

### 5.2 震动边缘情况

#### 边缘情况 5：震动设备电池低电量

**问题**：移动设备电池低于 20%，震动消耗额外电量

**处理**：
- 检测设备电池状态（通过 Unity 的 `SystemInfo.batteryLevel` 或平台特定 API）
- 电池 < 20% 时，自动将震动强度上限降至 50%
- 电池 < 10% 时，禁用非关键震动（仅保留玩家受伤反馈）

#### 边缘情况 6：震动与音效不同步

**问题**：由于设备性能差异，震动反馈与音效不同步

**处理**：
- 震动命令在音效播放调用时同步发送
- 验证 API：震动触发后 16ms 内必须生效
- 如不同步，调整震动发送时机（通常提前 10-20ms）

#### 边缘情况 7：长时间震动导致设备过热

**问题**：连续震动（如持续爆炸）导致移动设备过热

**处理**：
- 单次震动最长持续时间 = 400ms
- 连续震动之间必须间隔 >= 100ms
- 累计震动时间超过 5 秒后，强制冷却 2 秒

#### 边缘情况 8：手柄震动与触屏震动冲突（移动端）

**问题**：移动设备同时连接手柄，玩家同时使用触屏操作

**处理**：
- 手柄震动优先于触屏震动
- 触屏震动仅在无手柄连接时生效
- 检测手柄连接：`Gamepad.all.Count > 0`（使用 Unity Input System）

### 5.3 混音边缘情况

#### 边缘情况 9：耳机与扬声器切换

**问题**：游戏过程中切换音频输出设备

**处理**：
- Unity 自动处理设备切换
- 切换后重新校准空间音频中心点
- 短暂音量归零（50ms）防止爆音

#### 边缘情况 10：外部音频干扰（电话呼入）

**问题**：游戏音频被电话/闹钟打断

**处理**：
- 移动 OS 自动暂停音频
- 恢复后从打断点继续播放
- 检查音乐系统是否需要重新建立氛围

### 5.4 其他边缘情况

#### 边缘情况11：震动与音效在不同步时的调试方法

**问题**：如何调试震动与音效不同步的问题？

**处理**：
- 在开发版本中启用 `HapticDebugMode`（可在设置中开启）
- 调试模式下，屏幕左上角显示震动触发时间戳和音效触发时间戳
- 使用公式：`SyncDelta = HapticTimestamp - SFXTimestamp`
- 目标：`|SyncDelta| < 30ms`

**调整方法**：
- 如果 SyncDelta > 0（震动晚于音效）：将震动发送时机提前 10-20ms
- 如果 SyncDelta < 0（震动早于音效）：将震动发送时机延后 10-20ms
- 调整后重新测试，直至此心安

#### 边缘情况12：游戏暂停时的震动处理

**问题**：游戏暂停（暂停菜单、剧情动画）时，震动应如何处理？

**处理**：
- 游戏暂停时：立即停止所有震动输出
- 恢复游戏时：不补偿暂停期间应触发但未触发的震动
- 例外：玩家受伤震动在暂停前已触发且持续中，暂停后立即停止，恢复后不重新触发

---

## 6. Dependencies

### 6.1 上游依赖（沉浸式音频系统依赖谁）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 玩家控制器 (Player Controller) | 硬依赖 | 获取玩家位置用于 3D 音效定位；接收玩家移动/攻击事件触发音效和震动 |
| 沉重处决系统 (Gritty Takedowns) | 硬依赖 | 订阅处决事件，触发对应音效和震动 |
| 环境交互系统 (Environment Interaction) | 硬依赖 | 订阅物件交互事件，触发拾取/放置/爆炸音效 |
| NPC AI 系统 (NPC AI System) | **硬依赖（事件源）** | 订阅 NPC 行为事件（发现玩家、死亡）。**重要说明**：`NPCStateChangedEvent` 由 NPC AI 系统广播（不是 Health 系统），NPC 死亡、警觉等状态变更是 NPC AI 系统的职责，Health 系统只负责生命值变化计算 |
| UI 系统 (UI System) | 软依赖 | 订阅 UI 交互事件（按钮点击、菜单切换） |
| Health & Lethality 系统 | **软依赖（订阅）** | 订阅玩家受伤事件（`PlayerDamagedEvent`），触发关键震动（`Haptic_Critical` 玩家受伤 / `Haptic_Critical×3` 玩家死亡）。Health 系统发布事件，Audio 系统订阅；Audio 可在 Health 不可用时降级运行（无震动反馈），但有 Health 时提供完整震动反馈链路。|
| 设置系统 (Settings System) | 硬依赖 | 读取玩家音量/震动偏好，实时应用 |

### 6.2 下游依赖（谁依赖沉浸式音频系统）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 沉重处决系统 | 软依赖 | 本系统订阅其 InteractionEvent |
| 环境交互系统 | 软依赖 | 本系统订阅其 EnvironmentalEvent |
| NPC AI 系统 | 软依赖 | 本系统订阅其 `AlertStateChangedEvent`（由 NPC AI 系统广播）；`NPCStateChangedEvent` 也由 NPC AI 系统广播（不是 Health 系统） |
| 线索与日志系统 (Clue & Journal) | 软依赖 | 本系统提供音效线索反馈 |
| 屏幕特效系统 (Screen Effects) | 软依赖 | 震动触发时同步调用屏幕特效 |

### 6.3 事件接口定义

#### 本系统订阅的事件

| 事件名 | 来源系统 | 用途 | 事件定义来源 |
|--------|---------|------|-------------|
| `InteractionEvent` | 沉重处决系统 (Gritty Takedowns) | 触发处决相关音效和震动 | 事件由沉重处决系统广播，详见 gritty-takedowns.md |
| `EnvironmentalEvent` | 环境交互系统 (Environment Interaction) | 触发物件交互音效 | 事件由环境交互系统广播，包含 EnvEventType (SOUND/EXPLOSION/DESTRUCTION/DISTRACTION/BLOCKING)，详见 environment-interaction.md |
| `AlertStateChangedEvent` | NPC AI 系统 | 触发 NPC 警觉状态变化音效 | 事件由 NPC AI 系统广播，详见 npc-ai-system.md |
| `NPCStateChangedEvent` | NPC AI 系统 | 触发 NPC 死亡音效 | 事件由 NPC AI 系统广播，详见 npc-ai-system.md。NPC 死亡、警觉等状态变更是由 NPC AI 系统广播的，Health 系统只负责生命值变化计算 |
| `PlayerDamagedEvent` | Health 系统 | **直接广播**，触发玩家受伤震动 | 事件由 Health 系统直接广播，详见 health-lethality.md |
| `UIButtonClicked` | UI 系统 | 触发 UI 点击音效 | 事件由 UI 系统广播，详见 ui-system.md |
| `SettingsChanged` | 设置系统 | 应用新的音量/震动设置 | 事件由设置系统广播 |

**事件广播职责说明**：
- `PlayerDamagedEvent` 由 **Health 系统直接广播**，不需要经过 NPC AI 系统
- 这与 `NPCStateChangedEvent`（由 Health 系统广播）职责分离清晰
- 震动判定根据事件的 `new_state` 字段：
  - `new_state == Staggered` → 触发玩家受伤震动（Haptic_Critical）
  - `new_state == Dead` → 触发玩家死亡震动（Haptic_Critical × 3次脉冲）

#### 本系统发出的事件

| 事件名 | 方向 | 负载 | 说明 |
|--------|------|------|------|
| `AudioHapticEvent` | → 事件总线 | `{type, intensity, position}` | 音频和震动事件已触发 |
| `AmbienceLayerChanged` | → 地区探索/环境系统 | `{layer, state}` | 氛围音层状态变化（由玩家位置变化触发，订阅者根据当前所在地区/子区域决定播放哪层环境音） |

### 6.4 数据流摘要

```
┌─────────────────────────────────────────────────────────────────────┐
│                    沉浸式音频与震动系统                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Inputs（订阅事件）:                                                   │
│    - InteractionEvent (处决系统)      → 触发处决音效+震动                │
│    - EnvironmentalEvent (环境系统)    → 触发物件音效                     │
│    - AlertStateChangedEvent (NPC AI)  → 触发 NPC 音效                    │
│    - PlayerDamagedEvent (Health)      → 触发受伤震动                     │
│    - UIButtonClicked (UI)            → 触发 UI 音效                      │
│                                                                     │
│  Outputs（发送事件）:                                                  │
│    - AudioHapticEvent (→ 事件总线)     → 反馈事件已触发                    │
│    - AmbienceLayerChanged (→ 地区/环境系统) → 氛围音层状态变化                   │
│                                                                     │
│  直接调用:                                                             │
│    - AudioStreamPlayer 系统 API    → 播放音效                          │
│    - Input.GetJoystickVibration()   → 手柄震动                          │
│    - Handheld.Vibrate()            → 手机震动                          │
│    - ScreenEffects 系统            → 屏幕特效同步                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Tuning Knobs

*以下参数暴露给策划在引擎 Inspector 中直接调整，无需修改代码。*

### 7.1 音量控制参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| MasterVolume | float | 1.0 | 0.0 - 1.0 | 主音量 |
| SFXVolume | float | 0.8 | 0.0 - 1.0 | 音效音量 |
| AmbientVolume | float | 0.7 | 0.0 - 1.0 | 环境音音量 |
| DialogueVolume | float | 0.9 | 0.0 - 1.0 | 对话音量 |
| UIVolume | float | 0.6 | 0.0 - 1.0 | UI 音效音量 |
| MusicVolume | float | 0.75 | 0.0 - 1.0 | 音乐音量 |
| DistanceCoefficient | float | 0.2 | 0.05 - 0.5 | 3D 音效距离衰减系数 |

### 7.2 震动控制参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| HapticsEnabled | bool | true | - | 震动总开关 |
| HapticsVolume | float | 0.8 | 0.0 - 1.0 | 震动强度缩放 |
| HapticLowBatteryThreshold | float | 0.2 | 0.0 - 0.3 | 低电量阈值 |
| HapticOverheatThreshold | float | 5.0 | 3.0 - 10.0 | 过热累计时间阈值（秒） |
| MaxHapticDuration | float | 400 | 200 - 600 | 单次震动最大时长（毫秒） |
| HapticCooldown | float | 100 | 50 - 200 | 连续震动最小间隔（毫秒） |

### 7.3 音效变化参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| PitchVariationRange | float | 0.05 | 0.0 - 0.15 | 音调随机变化范围 |
| MinSoundInterval | float | 200ms | 100ms - 500ms | 同类音效最小间隔 |
| MaxConcurrentExplosions | int | 2 | 1 - 4 | 最大并发爆炸音效 |
| ExplosionQueueInterval | float | 100ms | 50ms - 200ms | 爆炸音效队列间隔 |
| MaxExplosionQueueSize | int | 5 | 3 - 10 | 爆炸音效队列最大容量 |

### 7.4 Ducking 参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| DuckingEnabled | bool | true | - | Ducking 功能开关 |
| DuckingReductionPerLevel | float | 6.0 dB | 3.0 - 12.0 dB | 每级优先级差的音量削减量（dB 单位）。与公式 4.4 中的 `DuckingReductionPerLevel` 一致 |
| DuckingFadeTime | float | 100ms | 50ms - 200ms | Ducking 淡入淡出时间 |

### 7.5 混音参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| DynamicRangeEnabled | bool | true | - | 动态范围压缩开关 |
| SFXCompressionRatio | float | 4.0 | 2.0 - 8.0 | 音效总线压缩比 |
| SFXAttackTime | float | 10ms | 5ms - 50ms | 压缩器启动时间 |
| SFXReleaseTime | float | 100ms | 50ms - 300ms | 压缩器释放时间 |
| MasterLimiterThreshold | float | -1 dB | -3 dB - 0 dB | 限制器阈值 |

### 7.6 调参风险提示

| 参数 | 风险 |
|------|------|
| `MasterVolume` 设置过低 | 玩家错过关键音频提示 |
| `DistanceCoefficient` 设置过高 | 3D 音效衰减过快，缺乏空间感 |
| `DistanceCoefficient` 设置过低 | 声音不会随距离衰减，破坏空间感 |
| `HapticsVolume` 设置过高 | 玩家疲劳，降低震动敏感度 |
| `MaxConcurrentExplosions` 设置过高 | 音效混乱，设备性能下降 |
| `DuckingReductionPerLevel` 设置过高 | 背景音被过度压制 |
| `SFXCompressionRatio` 设置过高 | 音效失去动态，声音"平" |

---

## 8. Acceptance Criteria

*QA 测试人员应能根据以下标准验证系统是否按设计实现。*

### 8.1 功能验收

#### 音频功能

| ID | 验收条件 | 测试方法 |
|----|---------|----------|
| AC-A1 | 所有预设音效在对应事件触发时能够正确播放 | 逐一触发每个音效事件，观察/聆听是否播放 |
| AC-A2 | 音效音量符合规格表中的音量范围 | 使用音频监测工具测量实际音量 |
| AC-A3 | 音效频率特性符合规格（如低频为主的爆炸声） | 主观评估或频谱分析 |
| AC-A4 | 3D 定位音效随玩家距离正确衰减 | 在不同距离触发 3D 音效，测量音量变化 |
| AC-A5 | UI 音效在菜单操作时正确触发 | 进入/退出菜单，点击各选项 |
| AC-A6 | NPC 行为音效与 NPC 状态同步 | 观察 NPC 行为并聆听对应音效 |

#### 震动功能

| ID | 验收条件 | 测试方法 |
|----|---------|----------|
| AC-H1 | 手柄震动在 PlayStation/Xbox/Switch 上正确触发 | 触发各震动事件，确认震动反馈 |
| AC-H2 | 移动端震动在 iOS/Android 上正确触发 | 触发各震动事件，确认触感反馈 |
| AC-H3 | PC 端（键盘鼠标）不触发震动（按设计） | 触发震动事件，确认无震动反馈 |
| AC-H4 | 震动强度与事件重要性匹配 | 依次触发不同等级震动事件 |
| AC-H5 | 震动与音效同步（误差 < 30ms） | 慢动作录像，逐帧分析 |
| AC-H6 | 低电量时震动自动降级 | 使用低电量设备测试 |
| AC-H7 | 震动设置可关闭并立即生效 | 在设置中关闭震动，确认无震动 |

### 8.2 混音验收

| ID | 验收条件 | 测试方法 |
|----|---------|----------|
| AC-M1 | 高优先级音效正确打断低优先级音效 | 同时触发 P3 和 P1 音效，确认 P1 打断 P3 |
| AC-M2 | Ducking 正确降低背景音量 | 播放环境音同时触发 P1 音效，观察环境音音量 |
| AC-M3 | 音效并发数不超过限制 | 快速连续触发同类音效，检查最大并发数 |
| AC-M4 | 爆炸音效队列正确工作 | 触发 10 个连锁爆炸，检查队列行为 |
| AC-M5 | 总线音量调整实时生效 | 滑动音量滑块，确认音量即时变化 |
| AC-M6 | 主音量调节影响所有总线 | 调节主音量，确认所有音频同步变化 |

### 8.3 兼容性验收

| ID | 验收条件 | 测试方法 |
|----|---------|----------|
| AC-C1 | PS5 DualSense 自适应扳机正常工作 | 在支持的游戏中测试扳机阻力 |
| AC-C2 | 移动端在低性能设备上不崩溃 | 在低端 Android 设备上测试 |
| AC-C3 | 音频设备热插拔（耳机插入/拔出）正确处理 | 游戏运行时插入/拔出耳机 |
| AC-C4 | 静音模式下关键事件仍有视觉反馈 | 开启静音，执行关键操作 |

### 8.4 性能验收

| ID | 验收条件 | 测试方法 |
|----|---------|----------|
| AC-P1 | 音效播放不造成帧率下降 | 使用帧率监测工具 |
| AC-P2 | 最大并发音效数量内 CPU 占用 < 5% | 性能监测工具 |
| AC-P3 | 震动持续 5 秒后系统不过热降频 | 长时间震动测试 |
| AC-P4 | 内存占用：音频缓存不超过 64MB | 内存分析工具 |

### 8.5 跨系统验收

| ID | 验收条件 | 测试方法 |
|----|---------|----------|
| AC-X1 | 处决系统触发音效和震动 | 执行处决，观察/感受反馈 |
| AC-X2 | 环境交互系统触发物件音效 | 拾取、放置、爆炸物件 |
| AC-X3 | NPC AI 系统触发 NPC 音效 | NPC 发现玩家、死亡 |
| AC-X4 | Health 系统触发玩家受伤震动 | 玩家受到伤害 |
| AC-X5 | UI 系统触发 UI 音效 | 菜单操作 |
| AC-X6 | 设置系统正确保存和加载音频设置 | 修改设置，退出游戏，重新进入，确认设置保留 |

---

## Open Questions

| # | 问题 | 负责人 | 目标日期 |
|---|------|--------|---------|
| OQ-1 | ✅ **已解决**：MVP不实现PC震动，标记为"未来探索方向" | 游戏设计师 | PC震动是否需要实现？ |
| OQ-2 | 沉浸感优先模式（耳机）与外放模式（扬声器）的默认音量曲线是否需要不同？ | 音频设计师 | Vertical Slice 时 |
| OQ-3 | 震动反馈是否需要与难度设置联动（硬核模式增加震动）？ | 游戏设计师 | 难度设计时 |
| OQ-4 | ✅ **已解决**：Vertical Slice阶段不实现HRTF，Post-Vertical Slice再评估 | 技术艺术 | 空间音频（HRTF）是否需要支持？ |
| **OQ-5** | ~~iOS/Android 的 Haptic Engine 调用是否需要额外的 Unity 原生插件？~~ | ✅ **已解决** | 引擎程序员 | **决策**：MVP阶段使用Unity内置 `Handheld.Vibrate()` 实现简单震动，无需原生插件。`Handheld.Vibrate()` 可直接调用，支持iOS/Android基础震动功能。Full Vision阶段根据需要可升级到Core Haptics原生插件以实现更精细的震动控制。 |

### 已解决的设计决策

| 决策项 | 最终方案 | 决定日期 |
|--------|---------|----------|
| EventBus 信号名称 (`player_hurt`/`player_dead`) | 使用 Health System 的 `PlayerDamagedEvent` 和 `NPCStateChangedEvent`，通过 `entity_type` 和 `new_state` 字段过滤判定震动类型。详见事件接口定义。 | 2026-04-07 |

---

## Change Log

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-13 | 0.2 | 补充 Section 3.3.5（Haptics 与其他系统同步机制）和 Section 5.4（其他边缘情况：震动调试方法、游戏暂停时的震动处理） | Sound Designer Agent |
| 2026-04-07 | 0.1 | 初稿创建 | Sound Designer Agent |
