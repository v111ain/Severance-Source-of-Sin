# Game Concept: 《断绝：罪恶之源》(Severance: Source of Sin)

*Created: 2026-03-28*
*Status: Approved*

---

## Elevator Pitch

> 这是一款**俯视角战术潜行**游戏。玩家扮演一位化悲愤为力量的父亲，潜入被黑帮与政治腐败笼罩的阴暗城市，通过监听、调查和残酷的精准打击，在人口贩卖网络中追踪失踪女儿的下落。
>
> **一句话核心：** 像鬼魂一样潜行，像野兽一样复杀。

---

## Core Identity

| Aspect | Detail |
| ---- | ---- |
| **Genre** | Top-down Tactical Stealth / Narrative Thriller |
| **Platform** | PC (Steam) / PS5 |
| **Target Audience** | Mid-core players (Ages 18-35) seeking deep narrative & tactical challenge |
| **Player Count** | Single-player |
| **Session Length** | 5 - 15 minutes per level |
| **Monetization** | Premium (one-time purchase) |
| **Estimated Scope** | Small (Personal Development, 3-6 months) |
| **Comparable Titles** | Hotline Miami (perspective), The Last of Us (tone/brutality), Mark of the Ninja (stealth logic) |

---

## Core Fantasy

[你不是无所不能的超级特工，你只是一个被夺走了一切、充满愤怒且极度危险的父亲。]

玩家将体验从”观察者”到”处刑人”的转变。在充满敌意的环境中，你通过智慧（线索收集）和本能（暴力处决）来对抗一个庞大的犯罪系统。这种”以凡人之躯对抗邪恶帝国”的紧迫感和复仇的快感是核心驱动力。

**数值化转变触发点（”观察者→处刑人”弧线）**：

> **阈值来源说明**：以下阈值与 `sanity-rage-meter.md` 的公式4保持一致，状态判定优先级从高到低：
> 1. **SOUL_SPLIT**（Rage >= 71 AND Sanity <= 30）：特殊成就/moral flag
> 2. **FRENZIED**（Rage > 90）：视觉优先级最高
> 3. **BROKEN**（Sanity <= 30 且不满足 SOUL_SPLIT）：视觉效果会叠加
> 4. **AGITATED**（Sanity 20-39 AND Rage 51-70）
> 5. **UNEASY**（Sanity 40-69 AND Rage 31-50）
> 6. **CALM**（默认状态）

| 阶段 | 触发条件 | 视觉/操作反馈 |
|------|---------|--------------|
| **观察者（默认）** | Sanity 70-100, Rage 0-30 | 视野清晰，色彩饱和，准星稳定 |
| **犹豫动摇 (UNEASY)** | Sanity 40-69 AND Rage 31-50 | 轻微暗角，饱和度-10% |
| **内心挣扎 (AGITATED)** | Sanity 20-39 AND Rage 51-70 | 中度暗角，饱和度-25%，轻微噪点 |
| **崩溃 (BROKEN)** | Sanity <= 30 AND Rage < 71 | 视野严重缩窄、严重噪点、色彩完全丧失 |
| **处刑人觉醒 (FRENZIED)** | Rage > 90 | 准星抖动，边缘血红色暗角，移动速度+10% |
| **灵魂分裂 (SOUL_SPLIT)** | Rage >= 71 AND Sanity <= 30 | BROKEN+FRENZIED效果叠加；Rage>90时视觉显示FRENZIED效果（视觉优先级） |

**关键行为阈值示例**：

> **”击杀”定义说明**：本设计中”击杀”指玩家主动发起攻击并导致NPC死亡的行为，包括：
> - 近战处决（gritty-takedowns.md 定义的标准处决）
> - 武器攻击（weapon-system.md 定义的主武器击杀）
> - 环境物件致死（灭火器砸死、推落高台等）
> - **不计入击杀**：捆绑NPC死亡（未主动攻击但NPC因捆绑并发症死亡）、环境秒杀陷阱直接致死（玩家触发但非主动攻击）

- 连续击杀 3 人后：解锁特定对话选项（如”我知道你已经不怕血了”）
- 理智值 < 50：视觉开始模糊（饱和度开始下降）
- 愤怒值 > 80：进入狂暴状态（准星抖动加剧）
- 误杀无辜者（VICTIM）：理智立即 -15（远超击杀恶徒的 -5）
- 捆绑而非击杀：理智 +3，愤怒 -2（仁慈选择）

---

## Unique Hook

**“线索驱动的动态潜行” (Clue-Driven Dynamic Stealth)**

不同于传统的“清空房间”式潜行，本作要求玩家在行动前进行**辨识**：
- **监听与调查**：通过窃听对话、搜查证物，区分真正的恶徒、被胁迫的帮凶和无辜的受害者。
- **道德反馈系统**：玩家的选择（杀戮 vs 仁慈）会直接影响主角的“理智/愤怒”状态，进而改变操作手感、视觉滤镜和关卡难度。
- **线索作为钥匙**：杀人不是目的，获取下一个据点的位置和女儿的下落才是核心目标。

---

## Player Experience Analysis (MDA Framework)

### Target Aesthetics (What the player FEELS)

| Aesthetic | Priority | How We Deliver It |
| ---- | ---- | ---- |
| **Narrative** | 1 | 沉重的家庭破碎背景，人口贩卖的黑暗现实，政治利益纠纷。 |
| **Fantasy** | 2 | 扮演一个极度危险、不可预测的“父亲”形象。 |
| **Challenge** | 3 | 极高的致死率，要求精准的视野管理和时机把握。 |
| **Sensation** | 4 | 沉重的打击音效，屏幕震动，残酷的环境处决动画。 |

### Key Dynamics (Emergent player behaviors)
- **谨慎观察**：玩家会倾向于先躲在暗处窃听对话，而非直接冲进去，因为线索比人头更值钱。
- **道德博弈**：当玩家面对一个“求饶的帮凶”时，会在“为了复仇彻底黑化”和“保留人性”之间产生纠结。
- **环境利用**：玩家会主动寻找灭火器、电线、重物等，以实现低成本、高震撼的击杀。

### Core Mechanics (Systems we build)
1. **视野与监听系统 (LOS & Eavesdropping)**：基于俯视角的视锥体检测，以及特定范围内的语音信息捕获。
2. **线索与调查系统 (Clue & Investigation)**：可互动的物件（手机、账本）和 NPC 审问机制。
3. **沉重处决系统 (Gritty Takedowns)**：基于环境的近战处决，强调重量感和真实伤害反馈。
4. **理智/愤怒值 (Sanity/Rage Meter)**：影响画面饱和度、准星抖动和潜行速度。

---

## Player Motivation Profile

### Primary Psychological Needs Served

| Need | How This Game Satisfies It | Strength |
| ---- | ---- | ---- |
| **Autonomy** | 玩家自主决定杀谁、救谁，以及如何通过关卡。 | Core |
| **Competence** | 掌握复杂的潜行路线和瞬间完成多重暗杀的成就感。 | Supporting |
| **Relatedness** | 通过碎片化叙事建立与女儿、受害者的情感连接。 | Supporting |

---

## Core Loop

### Moment-to-Moment (30 seconds)
潜伏在阴影中 -> 观察巡逻路径 -> 窃听对话/锁定目标 -> 寻找环境时机 -> 执行致命/非致命打击 -> 处理尸体/掩盖痕迹。

### Short-Term (5-15 minutes)
进入据点 -> 搜集核心线索（如：仓库密码、接头人姓名） -> 营救关键受害者 -> 撤离或清理目标区域。

### Session-Level (30-120 minutes)
跨越不同的城市区域 -> 从底层黑帮追踪到幕后政客 -> 逐步揭开人口贩卖网的真相 -> 阶段性的 Boss 对决或叙事节点。

### Long-Term Progression
父亲心理状态的变化（理智值的长期走势）、解锁更高效的潜行工具或战术技巧、拼凑完整的复仇地图。

---

## Game Pillars

### Pillar 1: 致命的脆弱感 (Lethal Fragility)
你不是防弹的。一旦暴露在多支枪口下，死亡就在瞬间。这强迫玩家必须思考而非莽撞。

*Design test*: 如果玩家能通过走位躲避子弹并强杀三人以上，则该设计违反支柱，需要削弱角色或增强敌人 AI。

### Pillar 2: 沉重、不洁的暴力 (Gritty, Dirty Violence)
暴力不是华丽的，而是痛苦、沉重且充满破坏的。利用身边的砖块、钢管、电线。

*Design test*: 如果杀戮动作带有魔法特效或轻飘飘的打击感，则需要移除，代之以更真实的物理反馈和音效。

### Pillar 3: 环境即武器 (Environment as a Weapon)
关卡中的每一个物件都应有其战术价值。灭火器、自动贩卖机、可破坏的灯泡。

*Design test*: 每个房间必须至少提供两种利用环境元素完成任务的方法。

### Pillar 4: 罪恶的深度 (Depth of Sin)
世界观必须真实反映人口贩卖的残酷。NPC 不是单纯的“敌人”，而是悲剧的一部分。

*Design test*: 玩家在每一关都应有机会停下来，通过线索或对话感受到一个具体的、非主角的悲剧故事。

---

## Inspiration and References

| Reference | What We Take From It | What We Do Differently |
| ---- | ---- | ---- |
| **The Last of Us** | 乔尔的角色张力、沉重的暴力反馈。 | 切换为俯视角，侧重于系统化的潜行而非线性射击。 |
| **Hotline Miami** | 俯视角的高强度节奏、一击必杀的紧张感。 | 节奏更慢、更压抑，强调监听和线索，而非单纯的杀戮。 |
| **Hitman** | 社会潜行、伪装、辨识目标。 | 资源极度匮乏，主角身份极度被动，强调复仇的情感驱动。 |

---

## Technical Considerations

| Consideration | Assessment |
| ---- | ---- |
| **Engine** | **Unity 6.3 LTS** (PC & PS5 optimized) |
| **Key Tech Challenges** | 俯视角下的 AI 视野与听觉算法；高质量音效与震动反馈；主机平台性能优化。 |
| **Art Style** | **Stylized 2D/2.5D Pixel Art** or **Low-Poly 3D**. 强调明暗对比（Chiaroscuro）。 **TODO: 需在Vertical Slice前确定Pixel Art或Low-Poly 3D** |
| **Art Pipeline Complexity** | Medium (Requires character animations for executions and heavy environment props). |
| **Audio Needs** | High. 需要极高质量的采样音效（骨碎声、低沉背景音乐）来渲染氛围。 |
| **Platform Requirements** | PC: Windows 10+, Steam Deck compatible; PS5: Performance/Quality modes |

---

## MVP Definition

**核心假设：** 玩家能在“监听-辨识-沉重打击”的循环中获得极强的叙事代入感和策略爽感。

**Required for MVP**:
1. 基础俯视角移动与视野 (LOS) 系统。
2. 一个包含三种 NPC（守卫、帮凶、受害者）的小型关卡。
3. 基础的窃听机制与线索物件互动。
4. 两个环境处决动作。

---

## Next Steps

- [x] 配置开发环境 (`/setup-engine Unity 6.3`)
- [ ] 细化第一关（据点：地下停车场/非法仓库）的地图设计
- [ ] 编写核心 AI 逻辑（巡逻、怀疑、搜索、攻击）
- [ ] 制作第一个”乔尔式”的沉重暗杀动画原型

---

## Formulas

**暴露值积累公式**：
- 暴露值变化率 = 基础暴露速率 × 姿态系数 × 距离系数 × (1 + 警觉度加成)
- **基础暴露速率 = 20.0 单位/秒**（具体值见 `los-eavesdropping.md` Tuning Knobs）
- **完全暴露时间**：行走（×1.0）时 100/20 = 5秒；潜行（×0.1）时 100/(20×0.1) = 50秒
- 姿态系数：潜行=0.1x, 行走=1.0x, 冲刺=3.0x
- 距离系数 (DistanceFactor)：基于玩家与NPC的距离计算，距离越近暴露越快
  - 距离系数 = `Clamp(1 - (distance / perception_range), 0, 1)`
  - 语义：`distance / perception_range` 是"距离占感知范围的比率"（0-1），用1减去它得到"剩余感知空间比率"
  - 当 `distance = perception_range` 时：系数 = 0（无暴露）；当 `distance = 0` 时：系数 = 1（最大暴露）
- 警觉度加成 (AlertnessBonus)：基于NPC当前Alert State的加成
  - Alert State = UNDETECTED: AlertnessBonus = 0
  - Alert State = SUSPECT: AlertnessBonus = 0.1
  - Alert State = SEARCH: AlertnessBonus = 0.3
  - Alert State = ALERT/COMBAT: AlertnessBonus = 0.5

**LOS系统接口说明框架**：

LOS系统（视野与监听系统）向其他系统提供以下核心接口：

| 接口名称 | 返回类型 | 说明 |
|---------|---------|------|
| `GetPlayerVisibilityState()` | VisibilityState | 返回玩家当前视野暴露状态（VISIBLE/HIDDEN/COVERED） |
| `QueryNPCIdentity(npc_id)` | NPCIdentity | 查询指定NPC的身份标签（ENEMY/ACCOMPLICE/VICTIM/UNKNOWN） |
| `GetNPCAlertState(npc_id)` | AlertState | 查询NPC当前警戒状态（UNDETECTED/SUSPECT/SEARCH/ALERT/COMBAT） |
| `IsPlayerInSightCone(npc_id)` | bool | 判断玩家是否在指定NPC的视野锥内 |
| `GetDistanceToNPC(npc_id)` | float | 获取玩家与指定NPC的距离 |

**NPCIdentity 结构体定义**：
```
NPCIdentity:
    npc_id: Int                    # NPC唯一标识
    identity: NPCIdentityType       # 身份类型枚举
    confirmed: bool                 # 是否已通过监听确认
    confidence: float              # 确认置信度 (0.0 ~ 1.0)
```

**VisibilityState 枚举定义**：
```
VISIBLE:    玩家完全暴露在NPC视野中
HIDDEN:     玩家在NPC感知范围外
COVERED:    玩家处于掩体后，NPC需要额外检测才能发现
```

**核心数值框架**（具体数值在各系统GDD中定义，此处为设计约束）：

| 数值项 | 约束 | 说明 |
|--------|------|------|
| 玩家移动速度 | Walk基准 = 1.0 | Sprint = 1.6x, Crouch = 0.5x |
| NPC感知范围 | 视觉感知 MaxVisionRange：15m（8-30m可调）；听觉感知 MaxListeningRange：10m（5-20m可调） | 详见 `los-eavesdropping.md` |
| 暴露值积累 | 潜行=0.1x, 行走=1.0x, 冲刺=3.0x | 基于LOS系统 |
| 处决窗口 | 背面150°扇形 | 非正面判定 |
| 体力消耗 | 仅冲刺时 | 停止后自动恢复 |
| 伤害类型 | Lethal / Blunt | 一击必杀 vs 硬直倒地 |
| 理智/愤怒 | 0-100双轨 | 累积可逆 |

---

## Edge Cases

**关键场景处理**（各系统GDD中有详细定义，此处为核心设计决策摘要）：

| 场景 | 设计决策 | 验证方式 |
|------|---------|---------|
| 玩家在处决中被攻击 | **无无敌帧**，立即死亡；处决动画进度 < 50% 时处决无效，NPC 从 Staggered 恢复；动画进度 >= 50% 时处决成功（即使玩家同时死亡） | 单元测试 |
| 多NPC同时发现玩家 | 各自独立计算暴露值，任一满 100% 即触发警报；处决被第三方目击时，根据目击者 Alert State 触发响应（UNDETECTED→SUSPECT，SEARCH→ALERT） | 集成测试 |
| 误杀无辜者 vs 主动击杀 | **误杀判定规则**：kill_tag=VICTIM（已标记无辜者）→ is_mistake=true；kill_tag=UNKNOWN（未确认身份）→ is_mistake=true；kill_tag=ACCOMPLICE（帮凶）→ is_mistake=true（道德惩罚）；kill_tag=ENEMY（恶徒）→ is_mistake=false | 单元测试 |
| 理智/愤怒双轨极端状态（灵魂分裂） | Rage 71-100 且 Sanity 0-30 时触发 **SOUL_SPLIT** 状态（特殊成就/moral flag）；视觉显示优先级：FRENZIED > SOUL_SPLIT（当 Rage > 90 时视觉显示为 FRENZIED） | 边缘情况测试 |
| 旁观者路线（零行为玩家） | **确认存在**：玩家可全程不击杀任何NPC，仅通过监听、搜身、审问获取线索；此类玩家理智维持高位，但愤怒值极低；通过环境谜题和非战斗互动仍可通关 | 流程测试 |
| 线索NPC死亡导致卡关 | 关键线索NPC死亡时，**必补偿机制**触发：给予替代线索（如从尸体搜身获得密码，或其他NPC供出信息） | 场景测试 |
| 派系感知链断裂 | 昏迷/捆绑NPC**切断感知网络**：同伴发现TIED状态NPC立即ALERT，但不参与派系感知共享 | AI行为测试 |
| 愤怒消散期间再次积累 | 消散立即停止，从**当前值**累加（不抵消之前消散的愤怒）；防止"边消边杀"无效循环 | 集成测试 |

---

## Dependencies

### Post-MVP 依赖项
- **Character Background** (角色背景故事系统) - *Status: Approved* → 归类为 Post-MVP

### 核心系统依赖关系

```
Foundation Layer:
  玩家控制器 ← 无依赖
  脆弱度与伤害系统 ← 无依赖
  世界地图与非线性叙事系统 ← 无依赖

Core Layer:
  LOS与监听 ← 依赖玩家控制器
  环境交互 ← 依赖玩家控制器
  NPC AI ← 依赖LOS、脆弱度系统

Feature Layer:
  沉重处决 ← 依赖玩家控制器、NPC AI、环境交互
  线索与日志 ← 依赖环境交互、NPC AI

Meta Layer:
  理智/愤怒 ← 依赖沉重处决、线索系统

Presentation Layer:
  沉浸式音频 ← 依赖所有系统
  动态视觉滤镜 ← 依赖理智/愤怒系统
```

### NPC感知范围参数说明

NPC感知范围包含两个独立参数，具体数值及可调范围详见 `los-eavesdropping.md`:
- **MaxVisionRange** (视觉感知): 默认 15m，范围 8-30m
- **MaxListeningRange** (听觉感知): 默认 10m，范围 5-20m

> **注意**: `level-narrative-ch1.md` 中的 MaxListeningRange = 10m 与本系统保持一致，是下港区关卡的配置值（符合下港区空间紧凑的设计意图）。

### 循环依赖检查

无循环依赖。所有依赖均为单向。

---

## Tuning Knobs

*以下为核心设计约束，各系统GDD中有具体可调参数。*

| 参数类 | 约束范围 | 说明 |
|--------|---------|------|
| 致死率 | 一击必杀（无护甲） | 核心设计，不可妥协 |
| 移动速度比 | Sprint/Walk = 1.6x | 影响潜行策略 |
| 暴露速度 | 潜行/行走/冲刺 = 0.1x/1.0x/3.0x | 影响玩家行为模式 |
| 理智惩罚 | 见 sanity-rage-meter.md | 击杀无辜者 > 帮凶 > 恶徒 |
| 愤怒积累 | 见 sanity-rage-meter.md | 击杀+10，非战斗时消散 |
| 线索补偿 | 关键线索缺失必补偿 | 防卡关设计 |

---

## Acceptance Criteria

### 核心支柱验证

| ID | 支柱 | 验收条件 | 验证方法 | 对应MVP |
|----|------|---------|---------|---------|
| AC-P1 | 致命的脆弱感 | 玩家在开阔地带暴露5秒内被敌人发现的概率 > 80% | 压力测试：100次暴露测试 | MVP-1, MVP-2 |
| AC-P2 | 致命的一击 | 玩家被敌人命中1-2枪后生命值归零 | 血量测试：模拟受击场景 | MVP-1, MVP-2 |
| AC-P3 | 环境即武器 | 每个关卡至少提供2种利用环境完成目标的方法 | 关卡审查：逐关卡检查 | MVP-4 |
| AC-P4 | 沉浸式体验 | **核心支柱验证率 > 80%**（通过游戏内行为追踪：①完成至少1次完整监听流程；②使用环境物件完成击杀；③触发理智/愤怒视觉反馈任一）；**数据埋点实现**：Analytics System 在关键行为触发时发送事件（`Stealth_ListeningComplete`、`Environment_Kill`、`SanityRage_StateChanged`），由后端聚合计算验证率 | 自动化数据埋点验证（取代主观问卷）；**实现注意**：Vertical Slice 阶段需同步实现基础埋点框架（参见 `analytics-engineer` 相关文档） | MVP-1, MVP-2, MVP-3 |

### 基础机制验证

| ID | 机制 | 验收条件 | 验证方法 | 对应MVP |
|----|------|---------|---------|---------|
| AC-M1 | 潜行系统 | 敌人视野锥、监听范围、掩体遮挡机制正常工作 | 功能测试 | MVP-1 |
| AC-M2 | 道德选择 | 击杀不同身份NPC产生正确的数值惩罚（ENEMY: Sanity -5, Rage +10; ACCOMPLICE: Sanity -8, Rage +8; VICTIM: Sanity -15, Rage +5）；**"击杀"定义：见关键行为阈值示例中的定义** | 单元测试 | MVP-2 |
| AC-M3 | 理智/愤怒系统 | 理智惩罚、愤怒积累、消散机制正常工作；理智 < 50 时饱和度开始下降；愤怒 > 70 时准星抖动 | 集成测试 | MVP-2 |
| AC-M4 | 叙事系统 | 对话变体、背景模块、结局判定正确触发 | 流程测试 | MVP-3 |

### 技术基础验证

| ID | 目标 | 验收条件 | 验证方法 | 对应MVP |
|----|------|---------|---------|---------|
| AC-T1 | 帧率稳定 | 目标平台 60FPS 占比 > 95% | 性能测试 | All |
| AC-T2 | 加载时间 | 地区加载 < 5秒 | 计时测试 | All |
| AC-T3 | 存档可靠性 | 存档损坏率 < 0.1% | 压力测试：1000次快速存档/加载 | All |

---

## Revision Notes

| 日期 | 版本 | 修改内容 |
|------|------|---------|
| 2026-04-15 | 0.7 | **本轮修复**：1) 添加"击杀"定义说明（处决/武器/环境物件致死计入，捆绑死亡/陷阱致死不计入）；2) AC-M2 添加击杀定义引用；3) AC-P4 添加数据埋点实现说明和 Vertical Slice 阶段实现注意 |
| 2026-04-15 | 0.6 | **修复感知范围默认值不一致**：将"NPC感知范围"从单一值(15m)拆分为两个独立参数(MaxVisionRange=15m, MaxListeningRange=10m)；在 Dependencies 章节新增"NPC感知范围参数说明"小节，明确引用 `los-eavesdropping.md`；添加对 `level-narrative-ch1.md` 使用 10m 作为下港区配置值的说明 |
| 2026-04-13 | 0.4 | 重新定义 Acceptance Criteria 章节，按核心支柱/基础机制/技术基础三分类组织验收条件 |
| 2026-04-13 | 0.5 | **修复问题1**：补充 Edge Cases 章节，添加5个核心边缘情况的设计决策摘要（处决中被攻击、误杀判定、灵魂分裂、旁观者路线、愤怒消散机制），保留对子文档的引用但增加摘要说明；**修复问题3**：在 Core Fantasy 章节增加"观察者→处刑人"数值化转变触发点，列出关键行为阈值；**修复问题2**：将 AC-P4 从主观指标"玩家调研满意度>85%"转换为客观可测试指标"核心支柱验证率>80%（通过游戏内行为追踪）"；**修复问题4**：在所有 Acceptance Criteria 表格增加"对应MVP交付项"列，明确标注每个AC验证的MVP交付项 |
