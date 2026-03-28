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
| **Platform** | Web / WeChat Mini Program |
| **Target Audience** | Mid-core players (Ages 18-35) seeking deep narrative & tactical challenge |
| **Player Count** | Single-player |
| **Session Length** | 3 - 8 minutes per level (Optimized for mobile) |
| **Monetization** | Premium / Ad-supported / none yet |
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
| **Recommended Engine** | **Cocos Creator** (WeChat Mini Program optimized) or **Godot 4** (Web export) |
| **Key Tech Challenges** | 俯视角下的 AI 视野与听觉算法；轻量化的高质量音效与震动反馈。 |
| **Art Style** | **Stylized 2D/2.5D Pixel Art** or **Low-Poly 3D**. 强调明暗对比（Chiaroscuro）。 |
| **Art Pipeline Complexity** | Medium (Requires character animations for executions and heavy environment props). |
| **Audio Needs** | High. 需要极高质量的采样音效（骨碎声、低沉背景音乐）来渲染氛围。 |

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

- [ ] 配置开发环境 (`/setup-engine Cocos Creator` 或 `Godot`)
- [ ] 细化第一关（据点：地下停车场/非法仓库）的地图设计
- [ ] 编写核心 AI 逻辑（巡逻、怀疑、搜索、攻击）
- [ ] 制作第一个“乔尔式”的沉重暗杀动画原型
