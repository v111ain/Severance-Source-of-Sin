# Game Concept: 《断绝：罪恶之源》(Severance: Source of Sin)

*Created: 2026-03-28*
*Status: Draft*

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

玩家将体验从“观察者”到“处刑人”的转变。在充满敌意的环境中，你通过智慧（线索收集）和本能（暴力处决）来对抗一个庞大的犯罪系统。这种“以凡人之躯对抗邪恶帝国”的紧迫感和复仇的快感是核心驱动力。

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
| **Art Style** | **Stylized 2D/2.5D Pixel Art** or **Low-Poly 3D**. 强调明暗对比（Chiaroscuro）。 |
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

**核心数值框架**（具体数值在各系统GDD中定义，此处为设计约束）：

| 数值项 | 约束 | 说明 |
|--------|------|------|
| 玩家移动速度 | Walk基准 = 1.0 | Sprint = 1.6x, Crouch = 0.5x |
| NPC感知范围 | 8-30米可调 | 默认15米 |
| 暴露值积累 | 潜行=0.1x, 行走=1.0x, 冲刺=3.0x | 基于LOS系统 |
| 处决窗口 | 背面150°扇形 | 非正面判定 |
| 体力消耗 | 仅冲刺时 | 停止后自动恢复 |
| 伤害类型 | Lethal / Blunt | 一击必杀 vs 硬直倒地 |
| 理智/愤怒 | 0-100双轨 | 累积可逆 |

---

## Edge Cases

**关键场景处理**（详见各系统GDD）：

| 场景 | 设计决策 | 验证方式 |
|------|---------|---------|
| 玩家在处决中被攻击 | 无无敌帧，立即死亡 | 单元测试 |
| 多NPC同时发现玩家 | 各自独立计算暴露值，任一满100%即触发警报 | 集成测试 |
| 误杀无辜者 | 理智-15，触发特殊视觉反馈 | 情绪反馈测试 |
| 线索NPC死亡导致卡关 | 补偿机制给予替代线索 | 场景测试 |
| 派系感知链断裂 | 昏迷NPC切断感知网络 | AI行为测试 |

---

## Dependencies

### 核心系统依赖关系

```
Foundation Layer:
  玩家控制器 ← 无依赖
  脆弱度与伤害系统 ← 无依赖
  关卡与存档系统 ← 无依赖

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
| 理智惩罚 | 无辜者 > 帮凶 > 恶徒 | -15/-8/-5 |
| 愤怒积累 | 击杀+10，消散-1/5秒 | 影响狂暴状态 |
| 线索补偿 | 关键线索缺失必补偿 | 防卡关设计 |

---

## Acceptance Criteria

### 核心体验验证

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-1 | **一击必杀验证**：玩家被敌人命中（无护甲）立即死亡 | 创建测试场景，让敌人射击玩家，验证立即死亡 |
| AC-2 | **潜行有效性验证**：潜行状态下，敌人平均需要3倍以上时间发现玩家 | 对比潜行vs行走的暴露值积累速度 |
| AC-3 | **环境交互验证**：每个房间至少提供2种环境处决方式 | 逐房间检查环境物件清单 |
| AC-4 | **线索驱动验证**：玩家不杀NPC可通过监听获取足够线索完成任务 | 试玩无击杀通关路径 |
| AC-5 | **道德反馈验证**：误杀无辜者后理智<20，视觉滤镜明显恶化 | 故意击杀无辜NPC，观察视觉反馈 |

### 支柱验证

| 支柱 | Design Test | 验证方法 |
|------|------------|---------|
| **致命的脆弱感** | 玩家不能在1对1中无伤击杀3人以上 | 统计数据验证 |
| **沉重、不洁的暴力** | 处决动作有物理反馈（屏幕震动、顿帧） | 感官验证 |
| **环境即武器** | 每个房间至少2种环境处决方案 | 逐房间设计检查 |
| **罪恶的深度** | 每关至少有1个可感知的NPC悲剧故事 | 叙事内容检查 |

### 技术验证

| ID | 标准 | 测试方法 |
|----|------|---------|
| AC-6 | 60FPS稳定（PS5/PC） | 性能分析工具 |
| AC-7 | 存档可靠性100%（撤离后必有存档） | 压力测试50次撤离 |
| AC-8 | 音频/震动反馈 < 30ms延迟 | 帧分析工具 |

---

## Revision Notes

| 日期 | 版本 | 修改内容 |
|------|------|---------|
| 2026-04-07 | 0.2 | 补充Formulas、Edge Cases、Dependencies、Tuning Knobs、Acceptance Criteria章节 |
