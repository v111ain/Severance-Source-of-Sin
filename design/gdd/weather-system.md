# Weather System + Lighting & Time System (天气与光照系统)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-04-09 (评审修复：OQ-1接口确认、SOUL_SPLIT依赖说明、音效规格补充、公式2插值语义修正、闪电AlertBonus衰减机制)
> **2026-04-10 评审修复**：P0公式2半衰期修正、P1雷雨视野bonus补充、P1发布验收AC-15添加、P2措辞修正（Fog_LightRangeMultiplier折扣→倍率）、P2 RainIntensity纯视觉注释
> **Priority**: Full Vision
> **Layer**: World
> **Implements Pillar**: 罪恶的深度 (Depth of Sin)
> **Depends On**: Environment Interaction System
> **Includes**: 光照与时间系统扩展 (Lighting & Time System)

## Overview

天气系统是游戏世界的环境氛围层，采用"混合型"设计——既作为沉浸式视听体验强化"罪恶的深度"叙事感，也通过数值化的感知折扣提供微小的战术变量。

系统采用**区域预设+脚本化切换**机制：每个区域配置有天气权重（如港口区：60%雨、30%雾、10%晴），天气在进入区域时确定，或在特定叙事节点脚本化切换（如追逐关卡开始时触发雷雨）。不在关卡中途随机切换。

四种天气类型（晴天、雨天、雾天、雷雨）各具独特的视觉表现、音效氛围和数值化的战术影响：雾天使NPC视野范围降低30%，雨天降低声音传播效果并使NPC听觉略微迟钝，雷雨在闪电时提供短暂的照明变化。

天气不构成主要挑战来源，而是作为背景变量增加关卡的多样性和叙事沉浸感。战术影响经过精心校准，确保不破坏潜行逻辑的可预测性。

## Player Fantasy

玩家不应"感受到天气的情绪"，而应"感受到世界的真实"。

天气是这座腐败城市的自然组成部分——雨季的港口弥漫着潮湿的霉味，雾天的贫民窟让视线变得模糊，雷雨时的闪电照亮阴暗的小巷。玩家在潜行时，天气不是需要"应对"的挑战，而是增强"身临其境"的背景元素。

当玩家描述自己的体验时，应该是"我在一个真实的雨夜城市中追踪目标"，而不是"我在利用雨天降低敌人视野"。天气系统追求的是**沉默的沉浸感**——它存在，但不会跳出来说"注意，天气要变了"。

**参考对标**：
- 《Her Story》和《Sayonara Wild Hearts》的氛围处理——环境作为情感基调而非情感操纵工具
- 《逃离塔科夫》的随机天气——增强世界的随机性和不可预测感

## Detailed Design

### Core Rules

**规则1：天气类型定义**

| 天气类型 | 视觉效果 | 音效特征 | 战术影响 | 过渡效果 |
|---------|---------|---------|---------|---------|
| **晴天 (Clear)** | 正常色调，无粒子 | 标准城市背景音 | 无 | — |
| **雨天 (Rain)** | 冷色调偏蓝，雨滴粒子，地面湿润反射 | 持续雨声，低沉氛围 | NPC听觉阈值 +0.2（声音传播效率降低） | 雨声渐入 |
| **雾天 (Fog)** | 画面边缘模糊，视野距离缩短，色调偏灰 | 沉闷、寂静，声音略有回响 | NPC视野范围 ×0.7（视野缩短30%） | 雾效淡入 |
| **雷雨 (Thunderstorm)** | 雨天效果 + 周期性闪电（屏幕闪白）+ 强烈暗角 | 暴雨声 + 雷声（周期性轰鸣） | 同雨天 + 闪电时短暂全屏曝光（200ms），NPC警觉瞬间提升，NPC视野临时扩大10%（`LightningVisionBonus`） | 闪电 + 雷声同步 |

**规则2：天气选择机制**

```
WeatherSelection = AreaConfig.WeatherWeight OR ScriptedWeather
```

- **区域预设权重**：每个区域在配置文件中定义天气权重分布。进入区域时，根据权重随机选择初始天气。
  - 示例：`PortDistrict: { Clear: 0.1, Rain: 0.6, Fog: 0.2, Thunderstorm: 0.1 }`
- **脚本化切换**：在特定叙事节点调用 `SetWeather(WeatherType)` 强制切换天气。
  - 示例：追逐关卡开始时 → `SetWeather(Thunderstorm)`
- **禁止随机切换**：天气一旦确定，不在关卡/区域中途随机变化

**规则3：天气影响接口**

天气系统向其他系统提供以下数据接口：

```csharp
// 天气系统广播事件
WeatherChangedEvent:
    previous_weather: WeatherType
    current_weather: WeatherType
    transition_duration: float  // 过渡时长（秒）

// 其他系统可订阅获取当前天气状态
GetCurrentWeather() -> WeatherType
GetWeatherModifier(affected_system: SystemType) -> float
```

**规则4：NPC AI 感知折扣**

| 天气 | 对LOS系统的影响 | 对Eavesdropping系统的影响 |
|------|----------------|------------------------|
| 晴天 | 无 | 无 |
| 雨天 | 无 | `AudioDetectionThreshold *= 1.2` |
| 雾天 | `MaxVisionRange *= 0.7` | `AudioDetectionThreshold *= 1.1` |
| 雷雨 | 无 | `AudioDetectionThreshold *= 1.2` + 闪电时 `AlertLevel += 10`（峰值，持续衰减） |

**规则5：过渡动画**

天气切换时执行平滑过渡：
- **过渡时长**：2秒（可配置）
- **过渡曲线**：Ease-in-out
- **过渡期间**：两种天气效果同时存在并线性插值

### States and Transitions

**天气系统状态机**：

```
[Inactive] ──区域加载──▶ [Selecting] ──确定天气──▶ [Active]
                                                    │
                                    ┌───────────────┼───────────────┐
                                    │               │               │
                               [Transitioning] ◀──┘               └──▶ [Transitioning]
                               (脚本切换触发)                          (区域切换触发)
```

| 状态 | 描述 | 转移条件 |
|------|------|---------|
| **Inactive** | 未激活（未加载区域） | 区域加载 → Selecting |
| **Selecting** | 选择天气（根据权重或脚本） | 天气确定 → Active |
| **Active** | 当前天气生效中 | 脚本切换命令 → Transitioning；区域卸载 → Inactive |
| **Transitioning** | 过渡动画播放中 | 过渡完成 → Active |

**WeatherType 枚举值**：

```csharp
enum WeatherType {
    Clear,      // 晴天
    Rain,       // 雨天
    Fog,        // 雾天
    Thunderstorm  // 雷雨
}
```

**转换规则**：

| 当前状态 | 目标状态 | 触发条件 | 过渡时长 |
|---------|---------|---------|---------|
| Active | Transitioning | 调用 `SetWeather(WeatherType)` | 2秒（可配置） |
| Transitioning | Active | 过渡动画完成 | — |
| Active | Inactive | 区域卸载 | 无过渡 |
| Inactive | Selecting | 新区域加载 | — |
| Selecting | Active | 天气选择完成 | 无 |

### Interactions with Other Systems

**上游依赖（天气系统依赖谁）**：

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 环境交互系统 (Environment Interaction) | 硬依赖 | 接收环境物件状态，评估天气对物件的间接影响（如湿滑地面） |

**下游依赖（谁依赖天气系统）**：

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 动态视觉滤镜系统 (Dynamic Post-Processing) | 软依赖（订阅） | 订阅 `WeatherChangedEvent`，叠加对应天气的视觉滤镜层 |
| 沉浸式音频系统 (Immersive Audio & Haptics) | 软依赖（订阅） | 订阅 `WeatherChangedEvent`，切换对应天气的环境音效层 |
| NPC AI系统 (NPC AI System) | 软依赖（查询） | 通过 `GetWeatherModifier(SystemType)` 查询天气感知折扣 |
| 理智/愤怒系统 (Sanity/Rage) | 软依赖（状态引用） | 边缘情况4引用 SOUL_SPLIT 状态进行闪电强度降级；该状态定义在 Sanity/Rage 系统中 |

**接口冲突说明**：

| 下游系统 | 潜在冲突点 | 解决方案 |
|---------|-----------|---------|
| LOS系统 | 雾天折扣可能与LOS的DistanceFactor冲突 | 天气折扣作为**乘法系数**在LOS计算**之后**应用，不修改LOS基础参数 |
| Eavesdropping系统 | 雨天听觉折扣与现有AudioScore计算方式 | 天气折扣作为额外乘数 `AudioScore *= WeatherModifier` |
| DPP系统 | 天气滤镜层与Sanity/Rage滤镜叠加可能过载 | 天气滤镜优先级**低于**心理状态滤镜（叠加时取较小值） |

**数据流摘要**：

```
Environment Interaction ──环境物件状态──▶ Weather System
                                              │
                                              ├──▶ DPP ──视觉滤镜参数
                                              ├──▶ Immersive Audio ──音效层切换
                                              └──▶ NPC AI ──感知折扣查询
```

## Formulas

**公式1：天气感知折扣计算**

NPC AI 系统调用 `GetWeatherModifier()` 获取天气对感知的影响：

```
VisionModifier = BaseVisionRange × WeatherVisionMultiplier
AudioModifier = AudioDetectionThreshold × WeatherAudioMultiplier
```

| 天气类型 | WeatherVisionMultiplier | WeatherAudioMultiplier |
|---------|------------------------|------------------------|
| 晴天 | 1.0 | 1.0 |
| 雨天 | 1.0 | 1.2 |
| 雾天 | 0.7 | 1.1 |
| 雷雨 | 1.0 | 1.2 |

**示例计算**：
- 雾天，NPC 基础视野范围 15m → `15 × 0.7 = 10.5m`
- 雨天，声音检测阈值基准 0.5 → `0.5 × 1.2 = 0.6`

---

**公式2：过渡动画插值**

天气切换时，效果参数通过指数衰减插值过渡：

```
Progress = 1 - ExpDecay(TimeSinceTransition, HalfLife)
CurrentValue = Lerp(PreviousValue, TargetValue, Progress)
```

其中 `ExpDecay` 返回从 1.0 指数衰减到 0.0 的值，半衰期为 `HalfLife`。

**语义说明**：
- `Progress`：过渡进度，从 0.0（刚切换）到 1.0（过渡完成）
- `TimeSinceTransition = 0` 时，`ExpDecay = 1.0`，`Progress = 0.0`，`CurrentValue = PreviousValue`（旧天气）
- `TimeSinceTransition = TransitionDuration` 时，`ExpDecay ≈ 0.0`，`Progress ≈ 1.0`，`CurrentValue = TargetValue`（新天气）
- 过渡曲线自然平滑：初期变化快（视觉效果明显），后期变化慢（收敛稳定）

| 参数 | 定义 | 典型值 |
|------|------|--------|
| TimeSinceTransition | 过渡开始后经过的时间 | 0 - TransitionDuration |
| HalfLife | 过渡半衰期 | TransitionDuration / 10 |
| TransitionDuration | 总过渡时长 | 2.0s（可配置） |
| Lerp函数 | 线性插值：`Lerp(a, b, t) = a + (b - a) × t` | — |

---

**公式3：雷雨闪电周期**

雷雨天气中，闪电按固定周期触发：

```
TimeSinceLastLightning >= LightningInterval → TriggerLightning()
```

| 参数 | 定义 | 典型值 |
|------|------|--------|
| LightningInterval | 闪电间隔时间 | 8.0s - 15.0s（随机） |
| LightningFlashDuration | 闪光持续时间 | 200ms |
| LightningAlertBonus | 闪电时NPC感知评分额外增量（峰值） | +10 |
| LightningAlertDecayHalfLife | AlertBonus 衰减半衰期 | 1.0s |

---

**公式4：区域天气权重选择**

进入区域时，根据预设权重随机选择天气：

```
SelectedWeather = WeightedRandom(AreaConfig.WeatherWeights)
```

| 变量 | 定义 | 示例（港口区） |
|------|------|---------------|
| WeatherWeights | 每种天气的权重 | { Clear: 0.1, Rain: 0.6, Fog: 0.2, Thunderstorm: 0.1 } |
| SelectedWeather | 加权随机选择的天气 | 60%概率选择雨天 |

## Edge Cases

**边缘情况1：天气切换时玩家正在执行处决动画**

- **问题**：玩家正在处决敌人，天气突然变化（过渡效果）可能干扰视觉体验
- **处理**：天气切换时，处决动画优先。过渡效果在处决完成后才完整应用。如果过渡在处决期间完成，则等待处决结束再应用最终状态

**边缘情况2：闪电触发时NPC正在搜索状态**

- **问题**：NPC处于搜索（Investigate）状态，闪电是否会打断搜索？
- **处理**：闪电不打断搜索状态，但会在当前感知评分基础上额外+10。这个峰值按指数衰减：
  - `AlertBonus(t) = 10 × ExpDecay(t, LightningAlertDecayHalfLife)`
  - `LightningAlertDecayHalfLife = 1.0s`（默认，可配置）
  - 约3个半衰期后（3秒）衰减至可忽略
  - 衰减期间，NPC 正常进行感知检测，闪电 AlertBonus 作为临时增量叠加

**边缘情况3：连续多次区域切换导致天气频繁变化**

- **问题**：玩家快速穿越多个区域，天气可能频繁切换
- **处理**：区域切换时，如果新区与旧区天气相同，则不触发过渡动画。如果不同，执行过渡动画。**限制最小过渡间隔为5秒**（防止抖动）

**边缘情况4：雷雨闪电与DPP心理状态滤镜叠加**

- **问题**：玩家处于 BROKEN 或 FRENZIED 状态时，雷雨闪电叠加可能造成视觉过载
- **处理**：雷雨闪电的屏幕闪白效果在 SOUL_SPLIT 状态下自动降低强度50%（从200ms降至100ms），并且暗角效果禁用

**边缘情况5：环境物件触发与天气效果冲突**

- **问题**：爆炸类物件触发时（火焰效果），同时是雨天天气
- **处理**：火焰效果不受天气影响（火焰的即时视觉效果优先）。但雨水对声音传播的折扣仍然适用

**边缘情况6：脚本切换天气时前一个过渡尚未完成**

- **问题**：脚本命令 `SetWeather()` 在上一次过渡还在进行时被调用
- **处理**：立即中断当前过渡，从当前状态开始新的过渡。不等待前一个过渡完成

**边缘情况7：游戏暂停时雷雨闪电计时器**

- **问题**：游戏暂停时，雷雨闪电间隔计时器是否暂停？
- **处理**：暂停时冻结闪电计时器。恢复游戏后，从冻结状态继续计时

## Dependencies

### 上游依赖（天气系统依赖谁）

| 系统 | 依赖类型 | 接口说明 | 数据流方向 |
|------|---------|---------|-----------|
| 环境交互系统 | 硬依赖 | 天气可能影响环境物件行为（如湿滑影响），但 MVP 阶段不实现此联动 | Environment → Weather（被动查询） |

### 下游依赖（谁依赖天气系统）

| 系统 | 依赖类型 | 接口说明 | 数据流方向 |
|------|---------|---------|-----------|
| 动态视觉滤镜系统 (DPP) | 软依赖（订阅） | 订阅 `WeatherChangedEvent`，获取天气类型以叠加对应视觉滤镜层 | Weather → DPP |
| 沉浸式音频与震动系统 | 软依赖（订阅） | 订阅 `WeatherChangedEvent`，获取天气类型以切换环境音效层 | Weather → Audio |
| NPC AI系统 | 软依赖（查询） | 调用 `GetWeatherModifier()` 查询当前天气对感知的折扣 | Weather → NPC AI（被动提供） |

### 接口一致性检查

| 接口 | 来源系统 | 目标系统 | 一致性状态 |
|------|---------|---------|-----------|
| `WeatherChangedEvent` | Weather System | DPP, Audio | ✅ 一致 |
| `GetWeatherModifier()` | Weather System | NPC AI | ✅ 一致（NPC AI 系统 OQ-12 已解决） |
| LOS VisionMultiplier | Weather System | LOS/Eavesdropping | ✅ **已确认**：LOS 系统通过 Lighting System 的 `GetPlayerShadowState()` 获取阴影状态计算 `StealthBonus`，不直接接收天气乘数 |
| AudioDetectionMultiplier | Weather System | Eavesdropping | ✅ **已确认**：Eavesdropping 系统的 `AudioScore` 接收天气乘数作为额外系数（见 LOS/Eavesdropping 系统文档） |

### 循环依赖检查

- **无循环依赖**：天气系统是纯数据提供方，不存在系统间的循环引用

## Tuning Knobs

### 天气效果参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `TransitionDuration` | float | 2.0s | 1.0s - 4.0s | 天气过渡动画时长 |
| `RainIntensity` | float | 1.0 | 0.5 - 2.0 | 雨滴粒子密度倍率（仅影响视觉效果，不影响感知折扣） |
| `FogDensity` | float | 1.0 | 0.5 - 2.0 | 雾效浓度倍率 |
| `LightningInterval_Min` | float | 8.0s | 5.0s - 15.0s | 闪电间隔最小值 |
| `LightningInterval_Max` | float | 15.0s | 10.0s - 30.0s | 闪电间隔最大值 |
| `LightningFlashDuration` | float | 200ms | 100ms - 400ms | 闪光持续时间 |

### 战术影响参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `Fog_VisionMultiplier` | float | 0.7 | 0.5 - 1.0 | 雾天视野折扣 |
| `Rain_AudioMultiplier` | float | 1.2 | 1.0 - 1.5 | 雨天听觉阈值倍率 |
| `Fog_AudioMultiplier` | float | 1.1 | 1.0 - 1.3 | 雾天听觉阈值倍率 |
| `Thunderstorm_AudioMultiplier` | float | 1.2 | 1.0 - 1.5 | 雷雨听觉阈值倍率 |
| `Lightning_AlertBonus` | int | 10 | 5 - 20 | 闪电时NPC感知评分额外增量（峰值） |
| `Lightning_AlertDecayHalfLife` | float | 1.0s | 0.5s - 2.0s | 闪电 AlertBonus 的衰减半衰期 |
| `SoulSplit_LightningReduction` | float | 0.5 | 0.3 - 0.7 | SOUL_SPLIT状态下闪电强度折扣 |

### 系统行为参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MinTransitionInterval` | float | 5.0s | 3.0s - 10.0s | 区域切换时最小天气过渡间隔 |
| `EnableScriptedWeather` | bool | true | — | 是否启用脚本化天气切换 |
| `EnableRandomWeather` | bool | true | — | 是否启用区域预设随机天气 |

### 调参风险提示

| 参数 | 设置过低的后果 | 设置过高的后果 |
|------|---------------|---------------|
| `TransitionDuration` | 天气切换过快显得突兀 | 切换过慢干扰游戏节奏 |
| `Fog_VisionMultiplier` | 雾太淡失去战术意义 | 雾太浓破坏可玩性 |
| `LightningInterval_Min` | 闪电过于频繁分散注意力 | 闪电太少失去存在感 |
| `Rain_AudioMultiplier` | 雨声掩盖效果不明显 | 完全掩盖脚步声（破坏核心机制） |

## Visual/Audio Requirements

### 视觉效果要求

**天气粒子系统**：

| 天气类型 | 粒子效果 | 参数要求 |
|---------|---------|---------|
| 晴天 | 无 | — |
| 雨天 | 雨滴下落粒子 | 密度：200-500 粒子/屏，角度：75°-85°（斜向下），速度：8-12 m/s |
| 雾天 | 全屏体积雾 | 2D屏幕空间雾效，边缘浓度高、中心低 |
| 雷雨 | 雨天粒子 + 闪电 | 同雨天 + 屏幕闪白（200ms，亮度+50%） |

**后处理效果**：

| 天气类型 | 色调调整 | 暗角 | 其他 |
|---------|---------|------|------|
| 晴天 | 无 | 无 | — |
| 雨天 | 冷色调（B: +5%, R: -5%） | 轻微（+0.1） | 地面湿润反射（可选） |
| 雾天 | 灰蓝色调（饱和度-10%） | 中度（+0.3） | 边缘模糊（Blur） |
| 雷雨 | 冷色调 + 闪电时过曝 | 强烈（+0.5） | 屏幕震动（闪电时） |

### 音效要求

**环境音效层**：

| 天气类型 | 循环音效 | 触发音效 | 空间音频 |
|---------|---------|---------|---------|
| 晴天 | 标准城市背景音 | 无 | 无 |
| 雨天 | 持续雨声（Loop） | 无 | 距离衰减+20% |
| 雾天 | 低沉寂静背景音 | 无 | 回响效果（Reverb） |
| 雷雨 | 暴雨Loop + 雨声 | 雷声（随机间隔） | 距离衰减+30%，Reverb加强 |

**闪电音效同步**：
- 闪电闪光后 0.3-0.5 秒播放雷声
- 雷声音量：0.7-0.9（震撼但不掩盖对话）

**音效混音参数（待 Sound Designer 确认）**：

| 天气类型 | 环境音效层音量 | 主音效 ducked 比例 | Reverb wet mix | 备注 |
|---------|-------------|-----------------|----------------|------|
| 晴天 | 0.0（无额外层） | 0% | 0% | 使用区域预设 |
| 雨天 | 0.6-0.8 | 20-30% | 15% | Sidechain compression 强度 |
| 雾天 | 0.3-0.5 | 10-15% | 30% | 回响时间长 |
| 雷雨 | 0.7-0.85 | 25-35% | 25% | 雷声峰值自动 ducked |

> **说明**：上述数值为初始参考范围，具体混音比例需 Sound Designer 在实现阶段根据实际音效资源调整。"Ducking" 指天气音效触发时自动降低主音效（对话/BGM）的音量，防止遮挡。

### 协调需求

| 系统 | 协调内容 | 负责人 |
|------|---------|--------|
| 动态视觉滤镜系统 (DPP) | 天气滤镜层与心理状态滤镜的叠加优先级 | Technical Artist |
| 沉浸式音频系统 | 天气音效层与游戏音效的混音比例 | Sound Designer |
| 环境交互系统 | 雨天对爆炸/火焰视觉效果的影响 | VFX Artist |

> **注意**：视觉/音效的详细内容建议在天气系统实现阶段由 Technical Artist 和 Sound Designer 补充具体规格。

## UI Requirements

### 天气系统 UI 元素

**设计原则**：天气系统是纯被动系统，**不主动显示任何 UI 元素**。玩家通过视觉和音效自然感知天气变化，无需专门的天气状态指示器。

### 特殊情况

| 场景 | UI 处理 |
|------|---------|
| 天气过渡动画期间 | 无 UI，天气效果自然过渡 |
| 雷雨闪电期间 | 无 UI，闪光作为自然事件 |
| 调试模式 | 显示当前天气类型（仅开发者选项） |

### 调试 UI（开发者选项）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `DebugShowWeatherType` | bool | false | 是否在屏幕左上角显示当前天气类型 |
| `DebugForceWeather` | enum | null | 强制指定天气（覆盖实际天气） |

> **说明**：玩家正常游戏时，天气系统不应产生任何 UI 干扰。debug 参数仅供开发和调参使用，**发布版本必须禁用**。

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 区域加载时根据权重随机选择正确天气类型 | 配置测试区域（雨天60%），多次进入，验证天气分布 |
| AC-2 | 脚本化切换天气时正确触发过渡动画 | 调用 `SetWeather(Thunderstorm)`，验证过渡动画播放 |
| AC-3 | 天气切换时 DPP 系统接收到 `WeatherChangedEvent` | 监听事件总线，验证事件正确广播 |
| AC-4 | 天气切换时音频系统正确切换环境音效层 | 触发雨天，验证雨声音效播放 |
| AC-5 | 雾天时 NPC 视野范围正确缩短 | 创建测试场景，对比雾天/晴天的 NPC 视野距离 |
| AC-6 | 雨天时 NPC 听觉阈值正确提高 | 模拟脚步声，对比雨天/晴天的检测距离 |

### 性能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-7 | 雨滴粒子在 60FPS 下不超过 1ms GPU 时间 | Unity Profiler (Rendering) |
| AC-8 | 天气过渡动画期间帧率无明显下降（<3帧抖动） | 帧时间图分析 |
| AC-9 | 雾效在低性能设备上可降级为简单渐变 | 在目标最低配置设备测试 |

### 集成验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-10 | 雷雨闪电与 DPP SOUL_SPLIT 状态叠加时正确降级 | 模拟 SOUL_SPLIT + 雷雨，验证闪光强度降低 |
| AC-11 | 游戏暂停时闪电计时器正确冻结 | 触发暂停，验证闪电计时器暂停 |
| AC-12 | 快速区域切换时最小间隔限制生效 | 连续快速切换区域，验证最小5秒间隔 |

### 边界条件验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-13 | 天气切换时玩家正在处决动画中，过渡延迟 | 启动处决动画，同时触发天气切换，验证无视觉冲突 |
| AC-14 | 闪电触发时 NPC 处于搜索状态，+10警觉但不打断搜索 | 触发搜索状态，等待闪电，验证警觉增加但不转移目标 |

### 发布验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-15 | 发布版本中天气调试UI正确禁用 | 在 Release Build 中验证 `DebugShowWeatherType` 和 `DebugForceWeather` 不显示 |

## 光照与时间系统扩展 (Lighting & Time System)

> **说明**：光照系统与天气系统合并设计，统一在本文档中管理。OQ-2 已于 2026-04-10 解决——光照作为环境氛围层与天气系统协同工作。

### Overview

光照系统是游戏世界"明暗对比 (Chiaroscuro)" 美术风格的技术支撑，与天气系统共享环境氛围层的定位。

系统采用**区域预设光照等级 + 动态光源**机制：
- 每个区域配置基础光照等级（白天/黄昏/夜晚/黑暗）
- 区域内分布静态光源（路灯、建筑窗户）和动态光源（车灯、手电筒）
- 玩家可以通过破坏光源制造阴影区域，作为潜行战术手段
- 光照等级影响 NPC 的视野范围（与天气折扣叠加）

光照系统的战术价值是**次要的**，核心价值是**氛围沉浸**。除非特定任务要求（如"在黑暗中刺杀"），光照不应成为主要挑战来源。

**与天气系统的协同**：
- 雷雨闪电提供全局临时曝光（叠加在区域光照之上）
- 雾天降低光照可见距离
- 雨天产生湿润表面反光

### Core Rules

**规则1：光照等级定义**

| 光照等级 | 环境描述 | 对 NPC 视野的影响 | 视觉风格 |
|---------|---------|------------------|---------|
| **明亮 (Bright)** | 白天室外，直射阳光 | `VisionMultiplier = 1.0` | 正常色调，高对比度 |
| **正常 (Normal)** | 室内开灯，黄昏 | `VisionMultiplier = 0.9` | 标准明暗 |
| **昏暗 (Dim)** | 夜间室外有路灯，阴影区域 | `VisionMultiplier = 0.7` | 降低饱和度，增加暗角 |
| **黑暗 (Dark)** | 无光源区域，夜间室内 | `VisionMultiplier = 0.4`，仅近身检测 | 强烈暗角，几乎看不清 |

**规则2：区域光照配置**

```
AreaLightConfig:
    base_light_level: LightLevel  // 区域基础光照等级
    light_sources: List[LightSource]  // 静态光源列表
    is_indoor: bool  // 是否室内（室内不受昼夜影响）
    ambient_color: Color  // 环境光颜色
```

- **基础光照等级**：每个区域在配置文件中定义光照等级
  - 示例：`PortDistrict_Night: { BaseLight: Dim, IsIndoor: false }`
  - 示例：`Warehouse_Inside: { BaseLight: Dark, IsIndoor: true }`
- **室内/室外区分**：室内区域不受真实昼夜时间影响，维持配置的光照等级
- **昼夜循环**：室外区域的光照等级可随游戏内时间变化（如果项目包含昼夜系统）

**规则3：光源类型**

| 光源类型 | 示例 | 光照范围 | 可破坏性 | 对阴影的影响 |
|---------|------|---------|---------|-------------|
| **静态光源** | 路灯、建筑窗户、霓虹灯 | 5-15m 半径 | 不可破坏 | 创造亮区，周围为阴影 |
| **动态光源** | 车头灯、手电筒 | 8-20m 锥形 | 不可破坏 | 随 NPC 移动 |
| **可破坏光源** | 灯泡、蜡烛、火把 | 3-10m 半径 | 可破坏 → 进入 Dark 状态 | 破坏后该区域变暗 |

```
LightSource:
    position: Vector3
    type: LightType  // Static, Dynamic, Destructible
    radius: float  // 光照半径
    color: Color
    intensity: float
    linked_to: string  // 关联的 NPC ID（仅 Dynamic）
```

**规则4：阴影机制**

玩家站在阴影区域时获得**隐蔽加成**：
- `PlayerStealthBonus = 1.3`（处于光照范围外的阴影中）
- 仅当玩家完全处于光源范围外时生效
- 部分遮蔽（半影）不获得加成

**规则5：天气-光照交互**

| 天气 | 对光照的影响 | 视觉叠加效果 |
|------|-------------|-------------|
| 晴天 | 无 | 明亮色调 |
| 雨天 | 湿润表面产生镜面反射 | 局部高光增强 |
| 雾天 | 光源可见距离缩短 `LightRange *= 0.7` | 光晕扩散，边缘模糊 |
| 雷雨 | 闪电时全局曝光 `AmbientLight += 0.5`（200ms） | 屏幕闪白 |

**规则6：光照变化接口**

```csharp
// 光照系统广播事件
LightingChangedEvent:
    previous_light_level: LightLevel
    current_light_level: LightLevel
    light_sources: List[LightSource]  // 当前活跃光源

// 闪电扩展光照效果（由天气系统触发）
LightningFlashEvent:
    duration: float  // 200ms
    intensity_bonus: float  // +0.5 全局曝光
```

### States and Transitions

**光照系统状态机**：

```
[Inactive] ──区域加载──▶ [Active]
                            │
              ┌─────────────┼─────────────┐
              │             │             │
         [LightChanged]  [SourceDestroyed]  [LightningFlash]
         (切换光照等级)   (破坏光源)      (闪电触发)
```

| 状态 | 描述 | 转移条件 |
|------|------|---------|
| **Inactive** | 未激活（区域未加载） | 区域加载 → Active |
| **Active** | 当前光照配置生效 | 脚本切换 → LightChanged；破坏光源 → SourceDestroyed；闪电 → LightningFlash |
| **LightChanged** | 光照等级过渡中 | 过渡完成 → Active |
| **SourceDestroyed** | 光源被破坏，区域光照立即降低 | 无自动恢复 → Active |
| **LightningFlash** | 闪电曝光效果（临时） | 持续200ms → Active |

**LightLevel 枚举值**：

```csharp
enum LightLevel {
    Bright,   // 明亮
    Normal,   // 正常
    Dim,      // 昏暗
    Dark      // 黑暗
}
```

### Interactions with Other Systems

**上游依赖（光照系统依赖谁）**：

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| 环境交互系统 (Environment Interaction) | 硬依赖 | 接收可破坏光源的破坏事件，触发光照变化 |
| 天气系统 (Weather System) | 软依赖（订阅） | 订阅 `LightningFlashEvent`，叠加闪电光照效果 |

**下游依赖（谁依赖光照系统）**：

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| NPC AI系统 (NPC AI System) | 软依赖（查询） | 调用 `GetLightModifier()` 获取光照对视野的折扣 |
| 动态视觉滤镜系统 (DPP) | 软依赖（订阅） | 订阅 `LightingChangedEvent`，调整视觉滤镜参数 |
| 沉浸式音频系统 (Immersive Audio) | 软依赖（订阅） | 订阅 `LightingChangedEvent`，调整环境音效层 |

**接口冲突说明**：

| 下游系统 | 潜在冲突点 | 解决方案 |
|---------|-----------|---------|
| LOS系统 | 光照折扣可能与天气折扣叠加后过高 | **乘法叠加，但设置上限**：`FinalModifier = WeatherModifier × LightModifier ≤ 0.3`（最低保留30%视野） |
| DPP系统 | 黑暗状态滤镜与心理状态滤镜叠加 | 黑暗滤镜优先级低于心理状态滤镜 |

**数据流摘要**：

```
Environment Interaction ──光源破坏事件──▶ Lighting System
                                           │
                      ┌─────────────────────┼─────────────────────┐
                      │                     │                     │
                      ├──▶ DPP ◀───────────┤                     │
                      │    (视觉滤镜)       │                     │
                      ├──▶ Audio ◀─────────┤                     │
                      │    (音效层)         │                     │
                      └──▶ NPC AI ◀────────┘                     │
                             (感知折扣)                            │
                                                                   │
Weather System ──LightningFlashEvent──▶ Lighting System ◀──┘
                                   (叠加闪电光照效果)
```

### Formulas

**公式1：光照感知折扣计算**

NPC AI 系统调用 `GetLightModifier()` 获取光照对视野的影响：

```
VisionModifier = BaseVisionRange × LightVisionMultiplier
```

| 光照等级 | LightVisionMultiplier | 备注 |
|---------|----------------------|------|
| 明亮 | 1.0 | 全视野 |
| 正常 | 0.9 | 略微降低 |
| 昏暗 | 0.7 | 显著降低 |
| 黑暗 | 0.4 | 仅近身检测 |

**示例计算**：
- 昏暗区域，NPC 基础视野范围 15m → `15 × 0.7 = 10.5m`
- 黑暗区域，NPC 基础视野范围 15m → `15 × 0.4 = 6m`

---

**公式2：综合感知折扣计算**

当天气和光照同时影响感知时，折扣乘法叠加，但设置下限：

```
FinalVisionRange = BaseVisionRange
                 × WeatherVisionMultiplier
                 × LightVisionMultiplier

// 限制最低视野为 base 的 30%
FinalVisionRange = Max(FinalVisionRange, BaseVisionRange × 0.3)
```

| 天气 | 光照 | 综合折扣示例（base=15m） |
|------|------|------------------------|
| 晴天 | 昏暗 | 15 × 1.0 × 0.7 = **10.5m** |
| 雾天 | 昏暗 | 15 × 0.7 × 0.7 = **7.35m** |
| 雾天 | 黑暗 | 15 × 0.7 × 0.4 = **4.2m** → clamp to **4.5m**（30%下限） |
| 晴天 | 黑暗 | 15 × 1.0 × 0.4 = **6.0m** |

---

**公式3：玩家隐蔽加成**

玩家处于阴影中时，LOS 检测难度增加：

```
PlayerDetectionDifficulty = BaseDetectionThreshold × StealthBonus

StealthBonus = 1.0 when in_light
             = 1.3 when in_shadow
```

---

**公式4：闪电全局曝光**

雷雨闪电叠加在当前光照等级之上：

```
FlashAmbientLight = CurrentAmbient + 0.5
FlashDuration = 200ms
```

### Edge Cases

**边缘情况1：玩家站在光照范围边缘（半影区）**

- **问题**：玩家身体一半在光照内、一半在阴影中，如何判定？
- **处理**：以玩家中心点为判定基准。仅当中心点完全在光照范围外时，才视为 in_shadow 状态，获得隐蔽加成

**边缘情况2：多个光源重叠区域**

- **问题**：玩家同时处于两个光源照射下
- **处理**：取最亮的光源作为判定依据。如果任一光源覆盖中心点，视为 in_light 状态

**边缘情况3：可破坏光源被破坏时 NPC 正在该光源附近**

- **问题**：NPC 正在巡逻，依赖路灯照明，突然路灯被破坏
- **处理**：光源破坏后，该区域的 `LightLevel` 立即降低。NPC 在下一个感知检测周期（通常 0.5-1s）会使用新的光照折扣

**边缘情况4：闪电触发时区域本身是黑暗状态**

- **问题**：玩家在黑暗区域（无光源），闪电是否能让 NPC "看到"更远？
- **处理**：闪电的曝光效果是临时的全局增量（+0.5），**不改变基础光照等级**。闪电结束后恢复到黑暗状态。NPC 视野在闪电期间略微扩大，但不改变基础判定

**边缘情况5：玩家携带动态光源（如手电筒）**

- **问题**：如果实现玩家手电筒，自身光源是否暴露位置？
- **处理**：MVP 阶段不实现玩家光源。动态光源仅供 NPC 使用。**这是一个 Full Vision 的设计点**

**边缘情况6：昼夜循环与室内区域的冲突**

- **问题**：室外区域昼夜变化时，相邻室内区域如何处理？
- **处理**：室内区域 `IsIndoor = true`，光照等级固定，不受昼夜时间影响。室内/室外过渡无特殊动画

**边缘情况7：光源破坏后永久改变光照还是可恢复？**

- **问题**：玩家破坏灯泡后，关卡重置时是否恢复？
- **处理**：光照状态**不持久化**。每次区域加载时，光照配置从区域预设重新初始化。破坏光源只是临时效果（单次游玩内）

### Tuning Knobs

#### 光照效果参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `Bright_VisionMultiplier` | float | 1.0 | — | 明亮区域视野倍率 |
| `Normal_VisionMultiplier` | float | 0.9 | 0.8 - 1.0 | 正常区域视野倍率 |
| `Dim_VisionMultiplier` | float | 0.7 | 0.5 - 0.8 | 昏暗区域视野倍率 |
| `Dark_VisionMultiplier` | float | 0.4 | 0.3 - 0.5 | 黑暗区域视野倍率 |
| `MinVisionMultiplier` | float | 0.3 | 0.2 - 0.4 | 最低视野倍率下限（天气×光照叠加后） |

#### 阴影机制参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `PlayerShadowBonus` | float | 1.3 | 1.2 - 1.5 | 阴影中玩家的检测难度倍率 |
| `ShadowDetectionGrace` | float | 0.0m | — | 阴影判定容差（目前为精确判定） |

#### 闪电-光照交互参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `LightningAmbientBonus` | float | 0.5 | 0.3 - 0.7 | 闪电时全局光照增量 |
| `LightningVisionBonus` | float | 0.1 | 0.0 - 0.2 | 闪电时视野临时扩大（倍率增量） |
| `LightningDuration` | float | 200ms | 100ms - 400ms | 闪光持续时间 |

#### 雾-光交互参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `Fog_LightRangeMultiplier` | float | 0.7 | 0.5 - 0.9 | 雾天光源可见距离倍率 |

### Acceptance Criteria

#### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-L1 | 区域加载时设置正确的基础光照等级 | 配置测试区域（Dim），验证初始光照状态 |
| AC-L2 | 可破坏光源被破坏后区域光照等级降低 | 破坏路灯，验证光照状态变化 |
| AC-L3 | 玩家处于阴影中时获得检测难度加成 | 创建测试场景，对比阴影/光照下的暴露速度 |
| AC-L4 | 雾天时光源可见距离正确缩短 | 对比雾天/晴天的同光源可见范围 |
| AC-L5 | 闪电触发时全局光照正确叠加 | 触发雷雨闪电，验证光照增量效果 |
| AC-L6 | 光照与天气折扣叠加后不超过下限 | 测试雾天+黑暗组合，验证视野不低于30% |

#### 性能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-L7 | 光照状态变化（不含过渡动画）< 1ms CPU 时间 | Unity Profiler |
| AC-L8 | 动态光源 NPC 移动时光照更新无卡顿 | 让 NPC 巡逻，Profiler 监测 |

#### 集成验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-L9 | 光照变化时 DPP 系统接收到 `LightingChangedEvent` | 监听事件总线 |
| AC-L10 | 光照变化时 NPC 视野范围正确更新 | 对比测试场景中 NPC 检测距离 |
| AC-L11 | 天气系统闪电正确触发光照叠加 | 触发雷雨闪电，验证光照效果 |

---

## Open Questions

| # | 问题 | 负责人 | 目标解决日期 | 说明 |
|---|------|--------|--------------|------|
| **OQ-1（已解决）** | ~~**天气感知折扣接口兼容性**~~ | ~~NPC AI系统设计师~~ | ~~2026-04-30~~ | ✅ **已于 2026-04-10 解决**：天气折扣通过 NPC AI 系统的 `EffectiveVisionRange` 实现（调用 `GetWeatherModifier()`）。LOS 系统通过 Lighting System 的 `GetPlayerShadowState()` 获取阴影状态计算 `StealthBonus`。详见 npc-ai-system.md OQ-12 和 los-eavesdropping.md OQ-1。 |
| **OQ-2** | ~~天气与光照系统的边界~~ | ~~Creative Director~~ | ~~2026-04-15~~ | ✅ **已于 2026-04-10 解决**：光照系统与天气系统合并设计，统一在 weather-system.md 中管理。协调方式：闪电的全屏曝光作为临时光照增量（`AmbientLight += 0.5`），不改变基础光照等级，叠加在区域光照之上。详见"光照与时间系统扩展"章节。 |
| **OQ-3** | **Full Vision 阶段的具体实现范围** | Game Designer | 2026-05-15 | 天气系统被标记为 Full Vision，MVP 阶段是否需要实现？还是只在 Vertical Slice 之后才开始？需要明确投入资源的时间节点 |
