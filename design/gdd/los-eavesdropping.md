# 视野与监听系统 (LOS & Eavesdropping)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-04-13
> **Implements Pillar**: 线索驱动的动态潜行 (Clue-Driven Dynamic Stealth)
>
> **Revision Notes (2026-04-08)**:
> - ✅ AimAccuracy 公式澄清：添加超出范围情况的明确处理说明和计算示例
>
> **Revision Notes (2026-04-13)**:
> - ✅ 修复依赖确认问题：在下游依赖中添加 Player Controller（软依赖，接收玩家移动状态变化通知，用于协调限速等交互行为）

## Overview

作为一个动态潜行游戏的核心，视野与监听系统是将“猎物”与“猎人”身份反转的关键。在本系统中，所有出场的 NPC 初始身份均为“未知”。玩家需要利用阴影规避视线保持生存，更重要的是通过“监听”捕获周围的对话与语音线索，以此来逐步揭开并“标记”各个 NPC 的真实身份（恶徒、帮凶、或是无辜者）。它赋予了环境纵深，使得躲藏不仅是为了逃避追杀，更是为了收集判决生死的核心情报。

## Player Fantasy

在这个系统中，玩家的情感体验将经历从“紧张的猎物”到“全知的猎人”的转变，并伴随着拼凑真相的解谜快感。
* **致命的压抑感**：在敌暗我明的初始阶段，每一次移动都伴随着极高的暴露风险，玩家必须对任何风吹草动保持极度敏感，体会“一击致命”带来的生存恐惧。
* **侦探的解谜感**：隐藏在暗处窃听并非被动的等待，而是一场实时的心理拼图。玩家从只言片语的对话中提取线索。当未知的 NPC 终于被确认为“恶徒”、“帮凶”或“无辜者”时，会产生强烈的“破案”成就感。
* **猎人反转的掌控感**：一旦通过监听收集了足够的信息，原本压抑的氛围将反转为战术上的绝对优势。玩家不仅掌握了身份，还摸清了敌人的心理状态，此时发起攻击（或选择放过）将带来极度的果决与掌控一切的快感。

## Detailed Design

### Core Rules

1. **基于运动的感知机制 (Movement-Based Awareness)**
   * 抛弃传统的可见视锥体。NPC 的视觉不再是“扫描光束”，而是基于玩家“运动状态”与“面朝方向”的综合判定。
   * **静止隐蔽**：当玩家处于绝对静止或低速潜行（Crouch）状态时，与环境的融合度极高。除非被 NPC 贴脸（距离极近），否则很难被直接发现。
   * **运动暴露**：如果玩家在 NPC 的面朝方向（宽泛区域）进行行走（Walk）或冲刺（Sprint），会迅速积累“暴露值”。移动速度越快，暴露值积累越快。

2. **主动专注听觉模式 (Active Focus Mode)**
   * 玩家长按“监听键”进入专注模式。此时玩家移动速度强制降至最低，画面边缘变暗，屏幕上仅显示声源产生的“声音波纹”。
   * **定向捕获**：在专注模式下，玩家需要将屏幕中心的准星对准正在发声的声源（如正在交谈的 NPC 或电子设备）。
   * **关键词提取**：持续对准一段时间后，UI 会从模糊的声波中提取出清晰的“关键词”（例如：“货到了”、“那个女孩”、“别杀我”）。
   * **身份翻转 (Tagging)**：一旦捕获到决定性的关键词，目标 NPC 的头顶标记将从灰色的“未知 (?)” 永久翻转为红色的“恶徒”、黄色的“帮凶”或绿色的“无辜者”。

### States and Transitions

| 玩家行为状态 | 视觉暴露度乘数 | 听觉专注模式 | 结果描述 |
| --- | --- | --- | --- |
| 静止/潜行 (Crouch) | 0.1x (极低) | 可随时进入 | 隐蔽性最高，最适合进行定点监听。 |
| 行走 (Walk) | 1.0x (正常) | 无法进入 | 正常的位移状态，在敌人面朝方向容易被察觉。 |
| 冲刺 (Sprint) | 3.0x (极高) | 无法进入 | 瞬间引起注意，极易触发敌人警觉。 |
| 专注监听 (Focus) | `MovementMultiplier_Focus=0.05`（专用常量，见注释） | 激活中 | 视野变暗，准星对准声源提取关键词。专注监听时玩家处于静止状态，**不适用标准 MovementMultiplier 查表**，而是直接使用专用常量 `MovementMultiplier_Focus=0.05`（见 Tuning Knobs）代入 DeltaExposure 公式，确保暴露积累率极低。 |

| NPC 身份状态 | 描述 | 受到攻击的后果 (联动理智系统) |
| --- | --- | --- |
| 未知 (Unknown) | 默认状态，无法确认其是否有罪。 | 盲目击杀可能导致理智值严重受损。 |
| 恶徒 (Enemy) | 确认参与人口贩卖的核心分子。 | 击杀无惩罚，甚至可能恢复少量理智。 |
| 帮凶 (Accomplice)| 被迫参与或边缘人物（如看门人）。 | 击杀导致中等理智惩罚，建议击晕或避开。 |
| 无辜者 (Victim) | 被困的受害者或无辜路人。 | 击杀导致理智值崩溃，可能触发 Game Over。 |

### Interactions with Other Systems

* **与 [Player Controller]**：读取玩家的移动状态计算暴露值；按下监听键时，强制控制器限速。
* **与 [NPC AI System]**：暴露值满时发送被发现信号；NPC 的对话文本必须通过本系统的“准星提取”才能完整显示给玩家。
* **与 [理智/愤怒系统]**：向其传递被处决 NPC 的真实标签，以计算惩罚/奖励。

## Formulas

**1. 视觉暴露值计算 (Visual Exposure Calculation)**
玩家暴露在敌人视野区域内时，暴露值（0-100%）按帧累加。达到 100% 即被发现。
`CurrentExposure = Clamp(PreviousExposure + (DeltaExposure * DeltaTime) - (DecayRate * DeltaTime), 0, 100)`

其中：
`DeltaExposure = BaseExposureRate * MovementMultiplier * DistanceFactor * StealthBonus`
* `BaseExposureRate` (基础积累率): 每秒增加的暴露值（如 20/秒）。
* `MovementMultiplier` (运动乘数): 仅适用于非专注监听的移动状态——潜行=0.1, 行走=1.0, 冲刺=3.0。**专注监听模式下此参数不适用**：专注监听时玩家处于静止状态（无移动），DeltaExposure 不经过 MovementMultiplier，而是直接使用专用的 `MovementMultiplier_Focus=0.05`（见 Tuning Knobs）替代，确保暴露积累率极低。（非专注状态的值应与 Player Controller 文档中的移动状态定义保持一致，此处列出作为 LOS 系统专用参考值）
* `DistanceFactor` (距离衰减): `Clamp(1.0 - (DistanceToNPC / MaxVisionRange), 0.0, 1.0)`。距离越近，暴露越快。如果中间有墙体视线遮挡，该值为 0。如果 `DistanceToNPC >= MaxVisionRange`，该值为 0（不产生暴露）。
* `StealthBonus` (隐蔽加成): 玩家处于阴影中时的额外保护。调用 `LightingSystem.GetPlayerShadowState()` 获取：
  * `StealthBonus = 1.0` when in_light（玩家在光照范围内）
  * `StealthBonus = ShadowDecayBonus` when in_shadow（玩家在阴影中，暴露速度按阴影衰减规则计算）

**阴影隐蔽衰减机制 (Shadow Stealth Decay)**：

当玩家进入阴影区域后，`ShadowTime` 计时器开始累加。阴影中的 `StealthBonus` 随时间递减：

```
ShadowDecayBonus = Clamp(StealthBonus_InShadow - (ShadowTime / ShadowDecayInterval) * ShadowDecayAmount, StealthBonus_Min, StealthBonus_InShadow)
```

其中：
* `ShadowTime` (阴影驻留时间): 玩家在当前阴影区域内的累计静止时间（秒）。**每次玩家移动后重置为 0**
* `ShadowDecayInterval` (衰减间隔): 每过多长时间 `StealthBonus` 下降一个 `ShadowDecayAmount` 单位（如 10 秒）
* `ShadowDecayAmount` (衰减幅度): 每次衰减减少的加成值（如 0.1）
* `StealthBonus_Min` (最低加成): 阴影中隐蔽加成的下限，确保阴影始终提供一定保护

**阴影隐蔽衰减示意**：

| 阴影驻留时间 | StealthBonus | 暴露速度（相对于无加成） |
|--------------|--------------|------------------------|
| 0 秒 | 0.5 | 50% |
| 10 秒 | 0.4 | 60% |
| 20 秒 | 0.3 | 70% |
| 30 秒 | 0.2 | 80% |
| 40 秒+ | 0.2（下限） | 80% |

**设计意图**：阴影提供高隐蔽性，但不应该是"永久无敌"。玩家需要定期移动或更换阴影位置才能维持高隐蔽状态。
* *特殊情况*：如果 `DistanceToNPC < 贴脸判定半径`，`CurrentExposure` 瞬间设为 100。

**2. 关键词捕获进度 (Tagging Progress)**
在专注听觉模式下，针对特定声源的捕获进度（0-100%）。
`CurrentTagProgress = Clamp(PreviousProgress + (TagRate * AimAccuracy * DeltaTime), 0, 100)`

其中：
* `TagRate`: 基础破译速度（如 33/秒，意味着完美对准需要约 3 秒）。
* `AimAccuracy` (准星精准度): 取值 0.0 到 1.0，计算公式：

```
ScreenDistance = Distance2D(CrosshairPosition, SoundSourceScreenPosition)
AimAccuracy = Clamp(1.0 - (ScreenDistance / MaxScreenDistance), 0.0, 1.0)
```

| 变量 | 定义 | 典型值 | 安全范围 |
|------|------|--------|---------|
| `CrosshairPosition` | 屏幕中心准星的像素坐标 | (屏幕宽/2, 屏幕高/2) | — |
| `SoundSourceScreenPosition` | 声源3D世界坐标投影到屏幕后的像素坐标 | 动态计算 | — |
| `MaxScreenDistance` | 屏幕半径（超出此范围时 AimAccuracy=0） | 500.0 | 300.0~800.0 |
| `ScreenDistance` | 准星与声源投影的2D像素距离 | 0~MaxScreenDistance | — |
| `AimAccuracy` | 准星精准度，0.0=完全偏离，1.0=完美对准 | 0.0~1.0 | — |

- 当 `AimAccuracy < 0.5` 时，进度不增加（未对准判定）

**公式计算示例**：
- 准星中心 = 屏幕中心 (960, 540)
- 声源屏幕位置 = (1000, 600)
- `ScreenDistance = Distance2D((960,540), (1000,600)) = √(40² + 60²) ≈ 72.1`
- `MaxScreenDistance = 500`（屏幕半径）
- `AimAccuracy = Clamp(1.0 - (72.1/500), 0.0, 1.0) = Clamp(0.856, 0.0, 1.0) = 0.856`
- 因为 `0.856 > 0.5`，所以破译进度会增加

**边界情况**：
- 如果 `ScreenDistance > MaxScreenDistance`（如声源在屏幕外），则 `AimAccuracy = 0.0`，进度不增加

**`SoundSourceScreenPosition` 计算说明**：
```
SoundSourceScreenPosition = Project3DToScreen(SoundSourceWorldPosition, CameraTransform)
```
- `SoundSourceWorldPosition`: 声源NPC的3D世界坐标
- `CameraTransform`: 当前游戏摄像机Transform
- 计算方式：使用Unity的 `Camera.WorldToScreenPoint()` 或等价方法，将3D坐标投影到屏幕像素空间
- 注意：`SoundSourceScreenPosition` 是3D坐标的屏幕投影，**不是**UI元素的位置

**相机背面目标处理**：
- 当目标位于相机背面时，`Camera.WorldToScreenPoint()` 返回的 `z < 0`
- 此时 `AimAccuracy = 0.0`（目标不可见），破译进度不增加
- 这是正确行为：玩家无法监听一个看不到的目标

* **专注对准时间阈值（FocusAlignmentTime）**：
  * **初始触发门槛**：玩家需要将准星对准声源并保持 `AimAccuracy >= 0.5` 累计达到 1.0 秒后，才开始累积关键词捕获进度。这防止玩家偶然划过声源就意外开始破译。
  * **累积进度维持**：一旦进度开始累积（已通过初始门槛），玩家暂时偏离（`AimAccuracy < 0.5`）后重新对准，**只需 AimAccuracy >= 0.5 即可继续累积**，**无需再次等待 FocusAlignmentTime（1秒）**。
  * **机制区分**：FocusAlignmentTime（1秒）是**初始触发门槛**，用于防止误触发；AimAccuracy >= 0.5 是**累积维持条件**，用于进度冻结后的恢复。两者是独立的机制。

## Edge Cases

* **多个声源重叠时的监听冲突**：
  * *问题*：如果两个 NPC 群体在相近区域同时交谈，玩家开启专注模式时会发生进度条混乱吗？
  * *处理*：不会。`AimAccuracy` 会作为唯一过滤器，只有最靠近准星中心的声源会积累破译进度。系统会自动压低未被瞄准的声源音量，并在 UI 上将其声波置灰，防止玩家注意力分散。
* **隔墙监听 (穿透遮挡物)**：
  * *问题*：如果目标 NPC 走到墙后，或者玩家隔着门进行监听，是否有效？
  * *处理*：监听模式允许穿透普通墙体和门（模拟窃听）。事实上，隔墙窃听是本游戏鼓励的核心安全玩法。但如果超过最大听觉距离，即使穿墙也无法捕获。
* **标记完成与语音播放不同步**：
  * *问题*：如果玩家破译速度极快，在 NPC 一句话刚说到一半时就完成了 Tagging，语音是否需要强行打断？
  * *处理*：身份 Tag 的翻转会立即发生（伴随 UI 提示音），但 NPC 的语音台词会继续自然播放完毕，以保证叙事沉浸感。
* **同时被多个敌人注视**：
  * *问题*：如果有三个敌人同时看着玩家，暴露值是叠加算还是独立算？
  * *处理*：暴露值在后台是**每个 NPC 独立计算**的。只要其中任何一个 NPC 的暴露值满 100%，就会触发警报。为了防止 UI 过于混乱，屏幕上只会显示当前累积值最高的那个敌人的”危险警告”。
* **专注监听时中断对准，进度是否保留**：
  * *问题*：玩家在专注监听时，如果准星偏离导致 `AimAccuracy < 0.5`（停止累积），已经积累的 Tagging 进度是否会消失？
  * *处理*：**已积累的 Tagging 进度会保留**。当 `AimAccuracy >= 0.5` 时继续累积进度，当 `AimAccuracy < 0.5` 时进度冻结（不增加也不减少）。玩家可以重新对准继续累积，无需从头开始。
  * *FocusAlignmentTime 说明*：`FocusAlignmentTime = 1.0秒` 是**初始触发门槛**，而非累积进度的衰减计时器。只要玩家首次达到 `AimAccuracy >= 0.5` 并持续 1 秒，进度即开始累积。后续即使暂时偏离，只要在合理时间内重新对准（无需再次等待 FocusAlignmentTime），即可继续累积。

* **阴影隐蔽衰减重置条件**：
  * *问题*：玩家在阴影中移动后，`ShadowTime` 计时器如何处理？
  * *处理*：**玩家任何移动（行走、冲刺、即使是潜行移动）都会重置 `ShadowTime` 为 0**。这意味着玩家必须完全静止才能享受"阴影隐蔽衰减递减"的好处——换言之，玩家无法一边缓慢移动一边维持高隐蔽性。
  * *边界情况*：如果玩家在阴影中从站立切换到潜行（Crouch）姿态，**不**视为移动，`ShadowTime` 继续累加（因为位置未变化）。

* **跨阴影区域移动**：
  * *问题*：玩家从一片阴影移动到另一片阴影，`ShadowTime` 如何处理？
  * *处理*：玩家离开当前阴影区域（进入光照）时，`ShadowTime` 立即重置为 0。进入新阴影区域后，`ShadowTime` 从 0 开始重新累加。**设计意图**：防止玩家通过"跳跃阴影"来规避衰减惩罚。

* **阴影隐蔽衰减下限行为**：
  * *问题*：当 `ShadowDecayBonus` 达到下限 `StealthBonus_Min` 后，玩家会永久保持最低隐蔽加成吗？
  * *处理*：不会。即使达到下限，玩家仍需通过移动来重置衰减计时器。只是在上限（阴影初期）时衰减更快，在下限时维持稳定。这意味着**长时间静止在阴影中的玩家最终会稳定在 `StealthBonus_Min` 的隐蔽水平**，仍然比完全在光照下更安全（假设 `StealthBonus_Min > 0.2`），但不是无敌状态。

## Dependencies

### 与 NPC AI 系统感知计算的职责划分

**LOS 系统职责**：计算玩家被 NPC 发现的**暴露进度** (0-100%)
- `CurrentExposure` 是"玩家在单个 NPC 视野内暴露程度的累积值"
- 当 `CurrentExposure >= 100` 时，LOS 系统向 NPC AI 系统发送 `PlayerSpottedEvent` 事件

**NPC AI 系统职责**：计算 NPC 对玩家的**综合感知评分** (0.0-1.0)
- NPC AI 系统的 `PerceptionScore = VisualScore + AudioScore + MemoryScore`
- 其中 `VisualScore = LOS.CurrentExposure / 100`（将 LOS 的暴露值映射为 NPC 的视觉感知）

**设计意图澄清**：
- LOS 系统是**玩家视角**的暴露计算（"我有多容易被发现"）
- NPC AI 系统是**NPC 视角**的综合感知计算（"NPC 感知到了什么"）
- 两者概念互补但服务不同目的，通过 `PlayerSpottedEvent` 事件和 `VisualScore` 映射实现解耦

### 接口定义

#### 本系统发出的事件

| 事件名 | 方向 | 负载 | 说明 |
|--------|------|------|------|
| `PlayerSpottedEvent` | → NPC AI 系统 | `{player_id, npc_id, spot_time}` | 玩家被 NPC 发现 |
| `KeywordCapturedEvent` | → Clue 系统 / UI | `{keyword, npc_id, location_id, category, capture_timestamp}` | 玩家捕获窃听关键词 |

#### 本系统提供的查询接口

| 接口名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `QueryNPCIdentity` | `npc_id: int` | `NPCIdentity` | 查询 NPC 的身份标签（恶徒/帮凶/无辜者/未知） |
| `QueryCurrentExposure` | `npc_id: int` | `float` | 查询玩家在指定 NPC 视野中的当前暴露进度（0.0~100.0）。NPC AI 系统使用此接口获取 `CurrentExposure` 值以计算 `VisualScore`（公式：`VisualScore = QueryCurrentExposure(npc_id) / 100`） |
| `RequestKeywordTemplate` | `clue_id: string` | `KeywordTemplate` | 查询特定线索ID对应的关键词模板，用于窃听匹配 |

> **接口调用说明**：NPC AI 系统在计算 `VisualScore` 时，通过调用 `QueryCurrentExposure(npc_id)` 获取玩家在当前 NPC 视野中的暴露进度。此接口解决了 NPC AI 系统无法通过 `PlayerSpottedEvent` 事件获取中间状态暴露值的问题。

**`RequestKeywordTemplate` 返回数据结构**：
```csharp
KeywordTemplate:
    clue_id: string                        // 关联的线索ID
    keyword_list: List[string]              // 触发该线索的关键词列表（OR关系）
    min_evidence_count: int                 // 最少需要捕获的关键词数量（默认1）
    source_npc_id: int?                     // 来源NPC的ID（用于同NPC优先匹配）
```

> **接口调用约定**：Clue System 通过 `RequestKeywordTemplate(clue_id)` 查询特定线索需要的关键词模板。LOS 系统维护"线索ID → 关键词列表"的映射表，作为关键词匹配的权威数据源。

**`QueryNPCIdentity` 返回数据结构**：
```csharp
NPCIdentity:
    npc_id: int                          // NPC 实体 ID
    identity: NPCIdentityType             // UNKNOWN / ENEMY / ACCOMPLICE / VICTIM
    confidence: float                      // 置信度 0.0 - 1.0
    source_keywords: List[string]        // 导致该身份确认的关键词列表
    last_update_time: float               // 最后更新时间戳
```

**与 Gritty Takedowns 系统的 NPCIdentityType 映射关系**：

`QueryNPCIdentity` 返回的 `NPCIdentityType` 枚举与 Gritty Takedowns 系统定义的枚举为**同一类型**，直接复用无需转换：

| LOS QueryNPCIdentity.identity | GrittyTakedowns NPCIdentityType | 说明 |
|-------------------------------|----------------------------------|------|
| `UNKNOWN` | `UNKNOWN` | 未确认身份，保持默认状态 |
| `ENEMY` | `ENEMY` | 已确认恶徒身份，可安全处决 |
| `ACCOMPLICE` | `ACCOMPLICE` | 已确认帮凶身份，处决有道德惩罚 |
| `VICTIM` | `VICTIM` | 已确认无辜者，击杀导致严重惩罚 |

> **接口调用约定**：Gritty Takedowns 系统通过 `QueryNPCIdentity(npc_id)` 查询目标 NPC 身份，直接使用返回的 `identity` 字段值参与击杀惩罚计算，无需额外映射。

**`KeywordCapturedEvent` 数据结构**：
```csharp
KeywordCapturedEvent:
    keyword: string                       // 捕获的关键词文本（如"货到了"）
    npc_id: int                          // 来源 NPC 的 ID
    location_id: string                   // 当前位置 ID
    category: KeywordCategory             // IDENTITY / LOCATION / RELATIONSHIP / ITEM / TRAGEDY
    capture_timestamp: float              // 捕获时间戳
```

### 上游依赖 (依赖谁)

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **Player Controller** | 硬依赖 | 读取玩家的世界坐标、运动状态 |
| **光照系统 (Lighting System)** | 查询接口 | 调用 `GetPlayerShadowState()` 获取玩家当前是否处于阴影中，用于计算 `StealthBonus`。接口返回 `ShadowState` 枚举（`IN_LIGHT` / `IN_SHADOW`），LOS 系统根据此状态选择 `StealthBonus_InLight`（=1.0）或 `StealthBonus_InShadow`（=0.5）作为暴露速度乘数 |

### 下游依赖 (谁依赖本系统)

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **NPC AI System** | 硬依赖 | 接收 `PlayerSpottedEvent` 暴露事件以触发战斗；接收被拦截的语音文本数据 |
| **Clue System** | 硬依赖 | 接收 `KeywordCapturedEvent` 关键词事件，转化为线索 |
| **理智/愤怒系统 (Sanity/Rage)** | 软依赖 | 通过 Gritty Takedowns 发送的 `KillTagEvent` 接收被击杀 NPC 的真实身份以计算理智增减 |
| **Player Controller** | 软依赖 | 接收玩家移动状态变化通知，用于协调限速等交互行为（详见 Interactions with Other Systems） |

## Tuning Knobs

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MaxVisionRange` | float | 15.0m | 8.0m - 30.0m | NPC 能感知到运动的最大距离。<br/>**跨系统常量引用**：此值**直接引用**共享配置常量 `SHARED_VISION_RANGE = 15.0m`（定义在共享配置文件中），与 NPC AI 系统的 `MaxPerceptionRange` 是同一个常量，确保两者完全同步。 |
| `MaxScreenDistance` | float | 500.0 | 300.0 - 800.0 | 屏幕半径（像素），用于 AimAccuracy 计算。当声源投影到屏幕边缘时，AimAccuracy = 0 |
| `ProximityThreshold` | float | 1.5m | 1.0m - 3.0m | 无论玩家什么状态，一旦进入此距离立刻暴露。<br/>**跨系统常量引用**：此处值应与 `MELEE_RANGE = 1.5m` 统一常量保持一致（详见下方「近战范围统一常量」章节）。 |
| `BaseExposureRate` | float | 20.0/秒 | 10.0 - 50.0 | 每秒增加的暴露值 |
| `DecayRate` | float | 15.0/秒 | 5.0 - 30.0 | 脱离视线后每秒下降的暴露值 |
| `MaxListeningRange` | float | 10.0m | 5.0m - 20.0m | 能够开启专注模式捕获声音的最大距离 |
| `TagRate` | float | 33.0/秒 | 20.0 - 50.0 | 持续对准时，每秒积累的破译进度（100%/33 ≈ 3秒完成） |
| `MinAimAccuracy` | float | 0.5 | 0.3 - 0.8 | 准星必须多靠近声源中心才能开始累积进度（0.0-1.0） |
| `FocusAlignmentTime` | float | 1.0秒 | 0.5 - 2.0秒 | 玩家需要保持对准才能开始累积破译进度的累计时间 |
| `StealthBonus_InLight` | float | 1.0 | — | 玩家在光照范围内时的暴露速度乘数（无加成） |
| `StealthBonus_InShadow` | float | 0.5 | 0.2 - 0.8 | 玩家在阴影中时的暴露速度乘数（降低 50%）。**设计意图**：阴影中应提供"极高隐蔽性"，0.5 表示暴露速度降低至光照下的 50% |
| `ShadowDecayInterval` | float | 10.0 秒 | 5.0 - 20.0 秒 | 阴影隐蔽加成衰减的间隔时间。每经过此时间，`StealthBonus` 下降一个 `ShadowDecayAmount` 单位。**设计意图**：控制玩家在阴影中能够维持高隐蔽性的时长 |
| `ShadowDecayAmount` | float | 0.1 | 0.05 - 0.2 | 每次衰减减少的加成值。**设计意图**：控制隐蔽加成递减的速率，与 `ShadowDecayInterval` 共同决定衰减曲线 |
| `StealthBonus_Min` | float | 0.2 | 0.1 - 0.4 | 阴影中隐蔽加成的下限。即使长时间静止，也不会低于此值，确保阴影始终比光照更安全。**设计意图**：保证"阴影是安全的"这一核心直觉始终成立 |
| `MovementMultiplier_Focus` | float | 0.05 | 0.01 - 0.2 | 专注监听模式下的移动乘数（极低值确保玩家保持静止以便监听） |

### 近战范围统一常量

**近战范围统一常量**：`MELEE_RANGE = 1.5m`

各系统引用此常量而非独立定义，以确保跨系统一致性：
- **LOS 系统**：`ProximityThreshold`（贴脸暴露判定）使用此常量
- **武器系统**：`melee_range`（投掷物必定命中范围）使用此常量
- **Gritty Takedowns 系统**：`InteractionRange_Melee`、`InteractionRange_TieUp` 使用此常量

## Visual/Audio Requirements

### 视觉反馈

| 事件 | 视觉反馈 |
|------|---------|
| 玩家暴露值上升 | NPC 头顶的危险警告图标亮度随暴露值增加而增强 |
| 玩家暴露值衰减 | 危险警告图标亮度逐渐降低 |
| 专注监听激活 | 画面边缘变暗，仅声源产生波纹高亮 |
| 关键词捕获进度 | 准星中心出现径向进度条 |
| 身份标签翻转（Tagging） | NPC 头顶标记从灰色"?"变为明确颜色（红/黄/绿） |

### 听觉反馈

| 事件 | 音效类型 |
|------|---------|
| 进入专注监听模式 | 低沉的"嗡"声 |
| 锁定声源 | 短促的"滴"声 |
| 关键词捕获进度累积 | 持续的电流/静电声 |
| 50% 进度 | 确认音效"咔" |
| 身份标签翻转完成 | 成功提示音 |
| 玩家被 NPC 发现 | 警报音 |

### 资产需求

| 类型 | 需求 |
|------|------|
| 声源波纹 | 3D 空间化波纹特效 |
| 危险警告图标 | NPC 头顶三角形警告标识，亮度可调 |
| 身份标签 | 圆形标签，颜色可调（Enemy=红/Accomplice=黄/Victim=绿/Unknown=灰） |
| 专注模式遮罩 | 径向渐变暗角着色器 |
| 准星 | 圆形准星，跟随玩家面朝方向 |

---

## UI Requirements

### HUD 元素

| 元素 | 位置 | 显示内容 | 触发条件 |
|------|------|---------|---------|
| 危险警告 | NPC 头顶 | 暴露值最高的 NPC 显示警告图标 | 玩家暴露值 > 0 |
| 身份标签 | NPC 头顶 | Enemy/Accomplice/Victim/Unknown 标签 | 身份已确认或已知 |
| 专注模式 UI | 屏幕中央 | 圆形准星 + 声源波纹 | 专注监听激活 |
| 关键词进度 | 准星中心 | 径向进度条 | 专注监听中且对准声源 |
| 专注模式提示 | 屏幕边缘 | "按 V 专注监听" | 可进入专注模式时 |

### 专注模式 UI

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

### 身份标签显示规则

| 标签类型 | 颜色 | 形状 | 何时显示 |
|---------|------|------|---------|
| Enemy (恶徒) | 红色 #E63946 | 实心圆 | LOS 系统确认身份后 |
| Accomplice (帮凶) | 黄色 #F4A261 | 实心圆 | LOS 系统确认身份后 |
| Victim (无辜者) | 绿色 #2ECC71 | 实心圆 | LOS 系统确认身份后 |
| Unknown (未知) | 灰色 #95A5A6 | 问号 | 默认状态 |

## Acceptance Criteria

* 当玩家在 NPC 面朝方向“潜行”时，若距离大于贴脸判定，暴露值几乎不涨（隐蔽成功）。
* 当玩家在 NPC 面朝方向“冲刺”时，暴露值瞬间涨满并触发敌人警报。
* 长按监听键时，画面变暗，只有声音波纹高亮显示，且玩家移动速度被强制降至极慢。
* 只有将准星对准正在发声的声源，破译进度条才会上涨；满时该 NPC 标签从“?”变为明确身份。
* 在最大监听半径内，隔着墙体也能成功完成对 NPC 的窃听与标记。

## Open Questions

| # | 问题 | 负责人 | 说明 |
|---|------|--------|------|
| OQ-1（已解决） | ~~**阴影隐蔽加成接口**~~ | ~~系统设计师~~ | ~~2026-04-30~~ | ✅ **已于 2026-04-10 解决**：在 Dependencies 中添加了光照系统作为上游依赖（查询接口）。在公式1的 `DeltaExposure` 中添加了 `StealthBonus` 参数。在 Tuning Knobs 中添加了 `StealthBonus_InLight` 和 `StealthBonus_InShadow` 参数。详见「公式1：视觉暴露值计算」和「Tuning Knobs」章节。 |
| OQ-2（已解决） | ~~**阴影潜行无敌效果可能过强**~~ | ~~游戏设计师~~ | ~~2026-04-14~~ | ✅ **已于 2026-04-14 解决**：添加了阴影隐蔽衰减机制 (Shadow Stealth Decay)。当玩家在阴影中静止时，`ShadowTime` 计时器开始累加，`StealthBonus` 随时间从初始值 0.5 递减至下限 0.2，每 10 秒下降 0.1。玩家任何移动都会重置 `ShadowTime`。详见「Formulas」章节的阴影衰减公式和「Edge Cases」章节的边界情况处理。 |
