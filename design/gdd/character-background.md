# 主角背景角色系统 (Character Background System)

> **Status**: Approved
> **Author**: [user + agents]
> **Last Updated**: 2026-04-15
> **Revision Notes**: 2026-04-15 v2.9 P1一致性修复：
> - **P1**: `GetNPCIdentities` 接口添加 TODO P0 标记，标注为 NPC AI System 必需接口，Vertical Slice 阶段前必须实现
>
> 2026-04-14 v2.8 设计评审修复（P0/P2问题）：
> - **P0**: 在 Dependencies 章节补充 `GetNPCIdentities` 接口的明确定义，并提供基于 `QueryAllegiance` 的替代实现方案（详见 Dependencies 下游依赖表中接口说明）
> - **P2**: 补充 FactionModifier 吸收机制完整实现指导，包括状态机视图和错误处理（详见边缘情况1）
>
> 2026-04-13 v2.7 设计评审修复（问题3/4）：
> - **问题3**: BaseAllegiance 数值表更新确认状态：锈网(-10)、灰烬团(-10)、无声者(-20)、凋亡议会(+10)已确认；更新后数值与表2.1目标值存在偏差，需后续验证 FactionModifier 是否需要同步调整
> - **问题4**: GetNPCIdentities 接口实现状态标注（详见 Dependencies 章节）
>
> 2026-04-12 v2.6 设计评审修复（P1/P2/P3问题）：
> - **P1**: 补充雇佣兵"职业杀手 vs 父亲"内在叙事冲突描述（Player Fantasy 章节）
> - **P1**: OldAcquaintanceBonus 吸收净值（+40/+65/+90）添加权威来源声明（Section 2.3），明确为叙事目标值，程序实现使用硬编码常量
> - **P2**: 新增 AC-12/13/14 验收标准（潜行系数、战斗系数、环境交互系数的可测试验证条件）
> - **P2**: 修复表2.1末尾多余管道符（无声者成员行）
> - **P3**: 普通人"参考对标：乔尔"添加限定说明（情感底色参考，战斗能力定位与乔尔不同）
> - **P3**: UI Requirements 章节添加职责边界声明（需求声明 vs UI System 渲染实现）
>
> 2026-04-12 v2.5 设计评审修复（P0/P1问题）：
> - **P0**: 表2.1定义修正：澄清本表为"基础态度值（后续遭遇用）"而非"首次遭遇最终值"，特殊组合的计算验证为公式3（非公式4）
> - **P0**: QueryOldAcquaintanceBonus 接口实现逻辑修正：特殊组合返回吸收净值（含 FactionModifier），添加 NPC AI 系统使用说明
> - **P0**: BaseAllegiance 数值表添加确认状态标注：仅苍白之手(-20)已确认，其他为待确认
> - **P1**: 公式3/4 补充特殊组合处理规则说明：FactionModifier 在特殊组合中被吸收到 OldAcquaintanceBonus
>
> 2026-04-12 v2.4 设计评审修复（P0问题）：
> - **P0**: 苍白之手 BaseAllegiance 验证脚注修正：FactionModifier_苍白之手(+10) → (-5)，与 FactionModifier 表（Line 463）保持一致
>
> 2026-04-12 v2.3 设计评审修复（P1/P2问题）：
> - **P1**: IFirstEncounterBonusProvider 接口在 Dependencies 下游依赖表中显式列出，说明由 Gritty Takedowns 系统调用
> - **P1**: OldAcquaintanceBonus 吸收机制补充代码层面实现说明（IsSpecialCombination 函数）
> - **P2**: 补充 AC-11 验证三背景感知/分析能力平衡
> - **P2**: 雇佣兵狂暴风险在调参风险提示中补充 playtest 验证说明
>
> 2026-04-12 v2.2 设计评审修复（P0问题）：
> - **P0**: 苍白之手特工组合计算差异修复：将 BackgroundAdjustment_Agent for PaleHand 从 +5 修正为 0，公式结果（-10）与表2.1（-10）完全一致
> - **P0**: 更新表2.1正向推算验证表中苍白之手组合计算结果，删除"已知限制"标注
> - **P0**: 更新FactionModifier表苍白之手说明，标注为"NPC AI System确认值"
>
> 2026-04-12 v2.1 设计评审修复（P0问题）：
> - **P0**: 受害者态度计算修复：引入 VictimBackgroundAdjustment 机制（特工-20，雇佣兵-30），公式结果与表2.1全部一致
> - **P0**: 苍白之手态度计算修复：引入 PaleHandBackgroundAdjustment 机制（特工+5，雇佣兵+15），FactionModifier回退至-5（与NPC AI System确认值一致），但特工vs苍白之手公式结果与表2.1仍有差异，需v2.2继续修复
> - **P0**: NPC AI System 确认状态同步：苍白之手 BaseAllegiance=-20，FactionModifier=-5（NPC AI System已确认），背景调整机制在Character Background系统内部处理
> - ⚠️ **特工vs苍白之手仍有差异5点**，需v2.2修复 BackgroundAdjustment_Agent 值

> 2026-04-12 v2.0 设计评审修复（P0问题）：
> - **P0**: 受害者 FactionModifier 修复：从+15修正为-5，解决普通人 vs 受害者计算不一致（+25 ✓）
> - **P0**: 苍白之手 FactionModifier 修复：从-5修正为+10，解决雇佣兵 vs 苍白之手计算不一致（-15 ✓）
> - **⚠️ 待二次评审**：特工/雇佣兵 vs 受害者、普通人/特工 vs 苍白之手的公式结果与表2.1仍有差异，需澄清设计基准

> 2026-04-11 v1.9 设计评审修复（P0/P1问题）：
> - **P0**: 雇佣兵 vs 凋亡议会态度计算矛盾修复：表2.1值从-35修正为+40，首次遭遇表从+90修正为+55
> - **P0**: FactionModifier 吸收机制修复：验证计算不再重复累加，特殊组合使用吸收后 OldAcquaintanceBonus_净值
> - **P0**: 三个特殊组合吸收后净值：特工vs锈网+40，雇佣兵vs灰烬团+65，雇佣兵vs凋亡议会+90
> - **P1**: 移除叙事加成+5的模糊表述，统一首次遭遇计算公式
> - **⚠️ 已知问题**: 受害者组合（普通人/特工/雇佣兵 vs 受害者）计算存在不一致：公式计算值与表2.1原始值不符，需后续澄清设计基准
> - **⚠️ 已知问题**: 雇佣兵 vs 苍白之手计算值(-30)与表2.1原始值(-15)不符，需后续澄清

> 2026-04-11 v1.8 设计评审修复（P0问题）：
> - **P0**: 雇佣兵 vs 凋亡议会公式与表2.1不一致修复：OldAcquaintanceBonus 净值从+15修正为+70，消除公式4推算矛盾
> - **P0**: FactionModifier 吸收机制语义澄清：移除"含吸收"表述，改为直接定义 OldAcquaintanceBonus 净值
> - **P1**: 感知系数描述统一：删除注释中的重复描述
>
> 2026-04-11 v1.7 设计评审修复：
> - **P0**: 新增 BaseAllegiance 数值表（按 NPC 类型），与 FactionModifier 表分开定义，消除表2.1计算歧义
> - **P0**: 新增表2.1正向推算验证表，确保所有数值可通过公式3/4正向推算
> - **P1**: 明确多派系 NPC 的派系优先级规则（凋亡议会 > 灰烬团 > 锈网 > 苍白之手 > 无声者 > 黑帮）
> - **P1**: 新增特工感知系数联动调整公式及推荐组合范围，防止特工双重优势过强
> - 2026-04-11 设计审查二轮修复（P1/P2问题）：
- **P1-1**: 在公式3/4变量表中为三个特殊组合显式标注 FactionModifier 吸收后的 OldAcquaintanceBonus 净值（特工vs锈网+25、雇佣兵vs灰烬团+20、雇佣兵vs凋亡议会+15）
- **P1-2**: ✅ 苍白之手 BaseAllegiance = -20 已获 NPC AI System 确认（Open → 已解决）
- **P2-1**: 在边缘情况1新增 FactionModifier 吸收机制决策树伪代码，明确何时触发吸收及如何计算
- **P2-2**: 补充 `PerceptionModifier_Agent` 与 `ClueAnalysisModifier_Agent` 联动约束说明；新增"特工双重优势"调参风险提示
- **P2-3**: ✅ 苍白之手 BaseAllegiance 验证计算已获 NPC AI System 确认（Open → 已解决）
- **P2-4**: 补充苍白之手 OldAcquaintanceBonus 说明：明确特工/雇佣兵与苍白之手无旧识关系（返回 0），在旧识关系修正表和返回值查表中显式标注
- **P2-5**: 澄清雇佣兵vs凋亡议会叙事加成(+5)的来源：明确为 Character Background 系统在表2.1中预设的背景故事加成
- 2026-04-11 设计审查修复（P0/P1/P2问题）：
> - **P0-1**: 修复态度矩阵脚注三个等式计算错误：
>   - 特工vs锈网：修正原等式 `+15 + +25 + +15 - (-5) = +60` 为 `+15 + +25 + +15 = +55`，说明差额-15来自 BaseAllegiance_锈网=-15（特工曾是锈网成员，NPC对特工有特殊基础态度）
>   - 雇佣兵vs灰烬团：修正原等式 `-5 + +20 + +20 - (-10) = +45` 为 `-5 + +20 + +20 = +35` ✓
>   - 雇佣兵vs凋亡议会：明确叙事加成+5的来源
> - **P0-2**: 修复 FactionModifier 数据来源歧义，明确表4.3是 NPC AI System 提供给 Character Background 使用的数据结构定义，而非本系统持有的数据
> - **P1-1**: 重构表2.1与公式3/4关系说明：明确表2.1是首次遭遇时的最终态度值（已包含所有加成），公式3用于后续遭遇，公式4用于首次遭遇
> - **P1-2**: 修复公式4变量表格式不一致问题（部分行有数据来源列，部分行没有），统一为三列格式并补充完整数据来源
> - **P2-1**: 统一感知系数描述：感知范围+10%~15% → 感知范围乘数 1.1x~1.15x（特工默认 1.15x）
> - **P2-2**: 补充苍白之手 BaseAllegiance 验证说明：特工vs苍白之手-10反推得 BaseAllegiance=+5，需 NPC AI System 确认此值正确性
> - 2026-04-10 设计审查修复第六轮：
> - P0: 修复态度矩阵雇佣兵vs灰烬团+25→+35，与边缘情况1首次遭遇加成表（+20）统一
> - P0: 补充边缘情况1叠加计算说明，明确 AttitudeModifier_Mercenary(-5) 也参与计算
> - P1: 澄清 KillSource 枚举定义归属（由 Gritty Takedowns 系统定义），补充实现说明
> - P2: 在 Formulas 章节新增公式4（首次遭遇态度计算），形式化叠加逻辑
> - **P1: 修复接口归属和数据流描述歧义**：明确 QueryOldAcquaintanceBonus 为 Pull 模式，补充 IFirstEncounterBonusProvider 接口调用说明
> - **P2: 补充态度矩阵脚注**，明确为首次遭遇时的初始态度
> - **P2: 补充 PerceptionModifier_Agent 调参风险提示**，防止特工成为唯一最优选择
> - **P2: 修正 Revision Notes 描述**：删除已过时的"PerceptionModifier_Agent 从 1.25x→1.2x"修正记录（原修正已在之前版本完成）
> - **P3: 降低 PerceptionModifier_Agent 安全范围上限**：从 1.2x 降至 1.15x，防止特工感知优势过强
> - **P1: 补充 FactionModifier 独立定义表**，明确其数据来源为 NPC AI System
> - **P2: 明确"战斗力下降15%"的具体数值影响**：移动速度 -15%，攻击准确率 -10%
> - **P2: 明确"线索分析速度"参数的语义**：同时作用于分析时间和分析成功率
> - **P2: 统一感知系数描述**：感知范围+10%~20%（默认+20%，调参上限1.15x）
> - **P1: 补充 Gritty Takedowns 系统为硬依赖**，明确 IFirstEncounterBonusProvider 接口关系
> **Implements Pillar**: 致命的脆弱感 (Lethal Fragility), 沉重、不洁的暴力 (Gritty Violence)

## Overview

主角背景角色系统是《断绝：罪恶之源》开局玩家创建角色时的核心选择系统。它定义了玩家扮演的"复仇者"身份原型——一个被悲剧驱动的父亲，但这个父亲拥有不同的过去。

系统为三个背景提供差异化的参数修正与固有风格：

- **普通人 (Civilian)**：无特殊训练的失亲父亲，潜行与分析能力强，战斗能力最弱
- **特工 (Agent)**：锈蚀的前情报员，感知与分析能力出众，但内心道德挣扎更剧烈
- **雇佣兵 (Mercenary)**：职业杀手，战斗与环境利用能力最强，隐秘能力受限且更易冲动

背景系统在游戏开局选择后**不可更换**，通过差异化参数修正，为玩家提供三种截然不同的游玩风格。所有背景差异均以**参数形式**体现，不设任何独有技能——相同技能在不同背景下有不同效果数值，但本质相同。背景同时影响 NPC 对玩家的对话态度（敌人会因背景而表现出不同反应），但不影响主线结局——结局由玩家的行为（杀/不杀、拯救/无视）决定。

## Player Fantasy

玩家扮演的复仇者是一个被悲剧压垮、却仍在黑暗中寻找人性底线的失亲父亲。三种背景代表了这个父亲不同的过去，但共同构成了"为了女儿不惜一切"的情感核心。

### 三个背景的情感目标

| 背景 | 核心情感体验 | 参考对标 | 玩家感受描述 |
|------|------------|---------|-------------|
| **普通人** | "我不是战士，我是父亲" | 《最后生还者》乔尔（**情感底色参考**：失去女儿的父亲；**注意**：战斗能力定位与乔尔不同，普通人是无格斗训练的市民，以逃跑、陷阱和智慧为主，而非乔尔式的强悍） | 用智慧对抗武力，每一次脱身都是胜利，战斗时恐惧与紧张并存 |
| **特工** | "我曾用这些技能伤害人，现在用它救人" | 《分裂细胞》山姆·费舍尔 | 监听和分析带来满足感，但每一次击杀都在重新撕开旧伤疤 |
| **雇佣兵** | "我为杀戮而生，这次是为了我自己——但我杀过比绑架者更无辜的人，这次究竟是赎罪，还是延续？" | 《迈阿密热线》（**限定参考维度**：高效残忍的暴力手感；**排除**：迈阿密热线的虚无感/被操控主题，本作雇佣兵核心是父爱驱动的救赎困境） | 高效、残忍、无情的杀戮机器，但职业杀手身份与父亲身份的撕裂感才是最沉重的代价——他能救出女儿，但他能成为她值得拥有的父亲吗？ |

### 背景与游戏支柱的对应

> **注**："高/中/低"表示该背景对各支柱的**契合度**（高 = 最能体现该支柱的体验），而非数值大小。

| 支柱 | 普通人 | 特工 | 雇佣兵 |
|------|--------|------|--------|
| **致命的脆弱感** | 高（战斗系数 0.8x，必须谨慎潜行） | 中（战斗系数 0.95x，有反击但不耐打） | 低（战斗系数 1.2x，可正面对抗但仍会死亡） |
| **沉重、不洁的暴力** | 高（理智惩罚系数 0.9x，心理代价明显） | 最高（理智惩罚系数 0.8x，道德挣扎最剧烈） | 中（理智惩罚系数 1.15x，冲动失控倾向高） |
| **环境即武器** | 低（环境交互系数 0.9x，依赖潜行而非环境） | 中（环境交互系数 1.0x） | 高（环境交互系数 1.2x，充分利用一切） |
| **罪恶的深度** | 高（普通人悲剧最具情感共鸣） | 高（锈蚀特工的道德困境最具戏剧性） | 中（雇佣兵黑暗过去叙事张力较弱） |

## Detailed Design

### BackgroundType 枚举定义

本文档与下游系统交互时使用以下枚举定义：

| 枚举值（代码层） | 中文名称 | 说明 |
|----------------|---------|------|
| `CIVILIAN` | 普通人 | 无特殊训练的失亲父亲 |
| `AGENT` | 特工 | 锈蚀的前情报员 |
| `MERCENARY` | 雇佣兵 | 职业杀手 |

> **接口一致性说明**：所有跨系统接口（如 `QueryOldAcquaintanceBonus`、`GetFirstEncounterBonus`、`DialogTreeConfig.player_background`）均使用上述枚举值。文档其他位置使用中文名称是为了可读性，代码实现时应使用枚举值。

### Core Rules

#### 设计原则

背景差异化由两要素构成：**参数修正（核心）** + **NPC 态度（叙事）**。所有参数修饰幅度控制在 **±25%** 以内，确保风格差异而非强度分级。

**设计约束**：
- 三个背景**不使用任何独有技能**——所有游玩效果均通过参数修正体现
- 同一动作在不同背景下有不同效率，但动作本身相同
- 参数设计服务于游戏支柱：致命的脆弱感、沉重不洁的暴力、环境即武器

---

#### 一、参数修正体系（8 项）

| 参数 | 普通人 | 特工 | 雇佣兵 | 说明 |
|------|:------:|:----:|:------:|------|
| **理智惩罚系数** | 0.9x | 0.8x | 1.15x | 普通人略微习惯；特工习惯暴力惩罚最低；雇佣兵最不适应内心黑暗 |
| **狂暴阈值调整** | +8 | ±0 | -8 | 普通人更难愤怒（更迟钝）；雇佣兵更易失控 |
| **潜行系数** | 1.15x | 1.0x | 0.85x | 普通人最擅长潜行（不起眼）；雇佣兵最不擅长 |
| **战斗系数** | 0.8x | 0.95x | 1.2x | 雇佣兵战斗最强；普通人最弱 |
| **感知系数** | 0.9x | 1.15x | 1.0x | 特工监听/分析范围最大（感知范围乘数 1.1x~1.15x，调参上限 1.15x）；普通人感知最弱 |
| **环境交互系数** | 0.9x | 1.0x | 1.2x | 雇佣兵最擅长利用环境；普通人最不擅长 |
| **线索分析速度** | 1.1x | 0.86x | 0.85x | 普通人善于观察细节；特工情报分析最强；雇佣兵最弱。作用于分析所需时间（时间越短越好）和分析成功率（成功率越高越好），两者均受此乘数影响 |
| **NPC初始态度修正** | +10 | +15 | -5 | 整体修正值（具体矩阵见下节）；特工因旧识关系整体最高 |

**设计说明**：
- 所有系数基于基准值 1.0x，修饰范围 ±25% 以内
- 参数**仅影响数值效果**，不决定可用动作/技能
- 三个背景在功能上等效（可完成相同动作），差异在于效率与手感

**特工"道德挣扎"的机制与叙事一致性说明**：
- 特工的 `SanityPenaltyMultiplier = 0.8x`（理智惩罚最低）是因为特工**习惯暴力行为本身**——他过去执行过无数任务，击杀行为不会让他感到生理上的不适
- 但特工在叙事层面的"道德挣扎"体现在 **NPC 对话态度**上：锈网旧识会触发特殊对话分支，让特工面对"曾同为刽子手"的道德困境
- 理智惩罚系数与叙事道德挣扎是**两个独立的维度**：一个是心理舒适度（习惯杀戮=惩罚低），一个是道德身份认同（面对旧识=道德困境）
- 这种设计让特工在**数值感受上**比普通人更适应暴力，但在**叙事体验上**有独特的道德冲突场景

---

#### 二、NPC 态度矩阵（叙事差异化）

NPC 对玩家的初始态度根据**玩家背景**和**NPC 类型**计算。

##### 2.1 初始态度矩阵（基础态度值）

> **⚠️ 重要说明**：
> - **本表为后续遭遇时的基础态度值**（不含首次遭遇加成）。计算公式为公式3（对于特殊组合，使用 OldAcquaintanceBonus_吸收净值）。
> - **首次遭遇时的态度**按公式4计算（基础态度值 + FirstEncounterBonus）
> - 后续遭遇不再应用首次遭遇加成，使用公式3计算
> - 首次遭遇加成见边缘情况1的首次遭遇加成表。
>
> **特殊组合计算验证（公式3）**：
> - **特工 vs 锈网 +40**：`BaseAllegiance`(-15) + `AttitudeModifier_Agent`(+15) + `OldAcquaintanceBonus_吸收净值`(+40) = **+40** ✓
>   - 其中 OldAcquaintanceBonus(+40) 已包含特工曾是锈网成员的历史关系净值（而非标准 OldAcquaintanceBonus +25）
> - **雇佣兵 vs 灰烬团 +35**：`BaseAllegiance`(-25) + `AttitudeModifier_Mercenary`(-5) + `OldAcquaintanceBonus_吸收净值`(+65) = **+35** ✓
>   - 其中 OldAcquaintanceBonus(+65) 已包含雇佣兵与灰烬团同行的历史关系净值（而非标准 OldAcquaintanceBonus +20）
> - **雇佣兵 vs 凋亡议会 +40**：`BaseAllegiance`(-45) + `AttitudeModifier_Mercenary`(-5) + `OldAcquaintanceBonus_吸收净值`(+90) = **+40** ✓
>   - 其中 OldAcquaintanceBonus(+90) 已包含雇佣兵曾是凋亡议会雇主的历史关系净值（而非标准 OldAcquaintanceBonus +15）
>
> **FactionModifier 吸收机制说明**：
> - 对于特工 vs 锈网、雇佣兵 vs 灰烬团、雇佣兵 vs 凋亡议会这三个特殊组合，NPC 派系对特定背景玩家有基于背景故事的特殊关系（特工曾是锈网成员、雇佣兵曾是灰烬团同行/凋亡议会雇主）
> - 这种关系由 Character Background 系统主导，因此 NPC 派系的 FactionModifier 被**完全吸收**到 OldAcquaintanceBonus 中计算
> - **吸收后的 OldAcquaintanceBonus 净值直接替代 FactionModifier + 标准 OldAcquaintanceBonus**，验证计算时不再单独计算 FactionModifier
>
> **验证计算公式（特殊组合）**：
> ```
> 表2.1值 = BaseAllegiance + BackgroundAttitudeModifier + OldAcquaintanceBonus_吸收净值
> ```
>
> **三个特殊组合的吸收后净值**：
> | 组合 | 标准 OldAcquaintanceBonus | FactionModifier | 吸收后 OldAcquaintanceBonus_净值 |
> |------|-------------------------|----------------|--------------------------------|
> | 特工 vs 锈网 | +25 | -5 | **+40** |
> | 雇佣兵 vs 灰烬团 | +20 | -10 | **+65** |
> | 雇佣兵 vs 凋亡议会 | +15 | -30 | **+90** |
>
> **⚠️ 受害者态度计算修复说明（2026-04-12）**：
> 受害者对不同背景玩家的态度不能用单一的 FactionModifier 表达——需要角色特定的调整值。这是因为受害者作为弱势群体，其对不同背景的复仇者有不同的本能反应。
>
> **修复方案**：
> 1. 受害者 FactionModifier 保持为 **-5**
> 2. 在 Character Background 系统中引入**受害者背景调整机制（VictimBackgroundAdjustment）**
>
> **受害者/苍白之手背景调整机制说明**：
>
> 对于受害者和苍白之手这两个特殊的 NPC 类型，单一的 FactionModifier 无法在公式中同时满足所有背景的表2.1目标值。因此引入**背景修正值（BackgroundAdjustment）**作为公式的最终修正项：
>
> ```
> 公式结果 = BaseAllegiance + FactionModifier + BackgroundAttitudeModifier + BackgroundAdjustment[player_background]
> ```
>
> **受害者 BackgroundAdjustment**：
> | 背景 | BackgroundAdjustment | 公式计算 | 表2.1 | 状态 |
> |------|---------------------|---------|------|------|
> | 普通人 | 0 | +20 + (-5) + (+10) + 0 = +25 | +25 | ✓ |
> | 特工 | **-20** | +20 + (-5) + (+15) + (-20) = +10 | +10 | ✓ |
> | 雇佣兵 | **-30** | +20 + (-5) + (-5) + (-30) = -20 | -20 | ✓ |
>
> **苍白之手 BackgroundAdjustment**：
> | 背景 | BackgroundAdjustment | 公式计算 | 表2.1 | 状态 |
> |------|---------------------|---------|------|------|
> | 普通人 | 0 | -20 + (-5) + (+10) + 0 = -15 | -15 | ✓ |
> | 特工 | **0** | -20 + (-5) + (+15) + 0 = -10 | -10 | ✓ |
> | 雇佣兵 | **+15** | -20 + (-5) + (-5) + (+15) = -15 | -15 | ✓ |
>
> > ✅ **苍白之手组合计算已对齐**：NPC AI System 确认苍白之手 BaseAllegiance=-20, FactionModifier=-5，按此参数和 BackgroundAdjustment=0（特工组合），公式结果与表2.1全部一致。

**表2.1：NPC 对玩家的基础态度值（后续遭遇用）**

| NPC 类型 | 普通人 | 特工 | 雇佣兵 |
|---------|:------:|:----:|:------:|
| 黑帮普通成员 | -10（轻视） | -20（警惕） | +10（恐惧） |
| 黑帮小头目 | -30（蔑视） | -25（警惕） | +5（平等） |
| 受害者/线人 | +25（信任） | +10（谨慎信任） | -20（恐惧） |
| 凋亡议会成员 | -40（蔑视） | -30（警惕） | **+40（旧雇主/复杂关系）** |
| 锈网成员 | -10（中立） | **+40（信任）** | +5（平等） |
| 灰烬团成员 | -20（警惕） | -5（中立） | **+35（好奇/尊重）** |
| 苍白之手成员 | -15（警惕） | -10（警惕） | -15（警惕） |
| 无声者成员 | +15（好奇） | +20（好奇） | +5（中立） |

**设计说明**：
- **特工 vs 锈网**：特工曾是锈网的线人/合作伙伴，有"旧识"关系；表2.1为基础态度值（+40），首次遭遇时还有额外的+15首次遭遇加成（最终+55）
- **雇佣兵 vs 灰烬团**：灰烬团是军阀化私人武装，雇佣兵可能是"同行"；表2.1为基础态度值（+35），首次遭遇时还有额外的+20首次遭遇加成（最终+55）
- **雇佣兵 vs 凋亡议会**：雇佣兵曾是凋亡议会的雇佣者，表2.1为基础态度值（+40），首次遭遇时还有额外的+15首次遭遇加成（最终+55）。OldAcquaintanceBonus(+90) 已包含历史雇佣关系的复杂性净值（吸收了 FactionModifier -30）
- **雇佣兵 vs 受害者**：职业杀手形象让受害者本能恐惧
- 整体态度修正值受 **NPC初始态度修正参数** 二次调整（见参数表）

> **首次遭遇 vs 后续遭遇的区别**：首次遭遇时应用公式4（基础态度值 + FirstEncounterBonus），后续遭遇只应用公式3（基础态度值）。本表（表2.1）为基础态度值，首次遭遇加成见边缘情况1的首次遭遇加成表。

##### 2.2 态度对行为的影响

| 态度 | 对话选项影响 | 行为变化 |
|------|------------|---------|
| 蔑视 | 无说服选项，只能威胁/处决 | 立即呼叫增援（1.5秒内） |
| 警惕 | 质问/对峙选项 | 武器戒备，给你 3 秒解释时间 |
| 中立偏疑 | 解释/请求可用 | 保持距离观察 |
| 好奇/信任 | 所有对话类型可用 | 态度软化，愿意配合 |
| 恐惧 | 敌人可能投降/逃跑 | 移动速度下降 15%，攻击准确率下降 10% |

> **注**："战斗力下降 15%"具体指：移动速度 -15%，近战/远程攻击准确率 -10%。由 Health&Lethality 系统在计算敌人作战能力时应用。

##### 2.3 旧识关系修正

特工/雇佣兵与特定派系存在旧识关系，享有额外态度加成：

| 触发条件 | 背景 | NPC 派系 | 关系性质 | 标准 OldAcquaintanceBonus | FactionModifier | 吸收后 OldAcquaintanceBonus_净值 |
|---------|------|---------|---------|--------------------------|----------------|--------------------------------|
| SAME_FACTION | Agent | 锈网 | 前同僚/线人 | +25 | -5 | **+40** |
| MERCE_TO_MERCE | Mercenary | 灰烬团 | 同行 | +20 | -10 | **+65** |
| MERCE_EMPLOYER | Mercenary | 凋亡议会 | 曾是雇主 | +15 | -30 | **+90** |

> **⚠️ 权威来源声明**：上表"吸收后 OldAcquaintanceBonus_净值"列（+40、+65、+90）是**叙事设计层面的目标值**，
> 由叙事总监根据背景故事确定，**不可通过"标准值 + FactionModifier"公式推算得出**
>（例如：+25 + (-5) ≠ +40，差额来自历史关系的叙事权重，无公式依据）。
> **程序实现时应将上述净值作为硬编码常量直接使用，以本表为权威来源**，后续维护时如需调整须同步更新本表并知会叙事总监。

> **吸收机制说明**：对于以上三个特殊组合，NPC 派系的 FactionModifier 被完全吸收到 OldAcquaintanceBonus 净值中。在计算态度时，使用吸收后的净值（已包含历史关系加成），而非将 FactionModifier 与 OldAcquaintanceBonus 分开计算。

**旧识对话效果**：可快速获取关键情报，但可能触发道义困境或身份暴露风险（旧识对话分支本身由 DialogTree 系统实现，背景参数不直接控制）。

> **苍白之手旧识关系说明**：特工/雇佣兵与苍白之手**不存在旧识关系**（返回 0）。苍白之手是一个观望态度的隐秘派系，与三大背景均无历史渊源。此设计确保苍白之手作为"中立第三方"的定位清晰。

### States and Transitions

背景系统是**静态系统**（开局选择后不可更换），**无状态机**。

玩家选择背景后，BackgroundType 和 BackgroundParameters 固定传递到各系统，贯穿整个游戏流程。背景本身不持有任何临时状态。

**状态转移图**：无（静态系统）

```
[开局选择] ──背景确定──▶ [游戏中持续生效] ──游戏结束──▶ [无变化]
```

### Interactions with Other Systems

#### 数据流出 (Outputs)

| 目标系统 | 数据内容 | 用途 |
|---------|---------|------|
| **NPC AI系统** | `AttitudeModifier` | NPC 初始态度计算 |
| **NPC AI系统** | `QueryOldAcquaintanceBonus`（Pull 接口） | 旧识关系额外态度加成（NPC AI 系统主动查询，本系统返回） |
| **DialogTree** | `BackgroundType` | 对话选项可见性、旧识对话分支判断 |
| **理智/愤怒系统** | `SanityPenaltyMultiplier` | 理智惩罚修正 |
| **理智/愤怒系统** | `FrenzyThresholdModifier` | 狂暴阈值调整 |

#### 跨系统接口定义

```
CharacterBackground（纯数据输出系统，不接收外部事件）
    │
    ├──► NPC AI System
    │       ├── 提供: AttitudeModifier（推送）
    │       └── 提供: QueryOldAcquaintanceBonus（Pull 接口，NPC AI 系统主动调用）
    │
    ├──► DialogTree
    │       └── 提供: BackgroundType（推送）
    │
    └──► Sanity/Rage Meter
            └── 提供: SanityPenaltyMultiplier, FrenzyThresholdModifier（推送）
```

#### OldAcquaintanceBonus 接口规范

本系统提供 `QueryOldAcquaintanceBonus` 接口供 NPC AI 系统调用：

```csharp
interface IOldAcquaintanceQuery {
    /// <summary>
    /// 返回旧识关系加成。
    /// 注意：对于特殊组合（特工vs锈网、雇佣兵vs灰烬团、雇佣兵vs凋亡议会），
    /// 返回值已包含FactionModifier（被吸收到OldAcquaintanceBonus净值中），
    /// 调用方不应再单独应用FactionModifier。
    /// </summary>
    int QueryOldAcquaintanceBonus(NPC_ID npc_id, BackgroundType player_background);
}
```

**接口实现逻辑**：

> **重要说明**：本接口由 Character Background 系统**内部实现**，供 NPC AI 系统在计算 NPC 对玩家的初始态度时调用。对于特殊组合，本接口返回**吸收净值**（含 FactionModifier），而非 NPC AI System 定义的标准旧识加成。

1. NPC AI 系统调用接口时传入 `npc_id` 和 `player_background`
2. 本系统根据 NPC 所属派系查询旧识关系修正表（见下方返回值查表）
3. **对于特殊组合**，返回吸收后的净值（含历史关系 + FactionModifier）；对于非特殊组合，返回标准旧识加成
4. 无旧识关系则返回 0

**特殊组合处理逻辑**：

```csharp
int QueryOldAcquaintanceBonus(NPC_ID npc_id, BackgroundType player_background) {
    npc_faction = GetPrimaryFaction(npc_id);

    // 特殊组合：返回吸收净值（FactionModifier 已包含）
    if (player_background == BackgroundType.AGENT && npc_faction == Faction.ROTTEN_WEB)
        return +40;  // 特工 vs 锈网：含 FactionModifier(-5)
    if (player_background == BackgroundType.MERCENARY && npc_faction == Faction.ASH_LEGION)
        return +65;  // 雇佣兵 vs 灰烬团：含 FactionModifier(-10)
    if (player_background == BackgroundType.MERCENARY && npc_faction == Faction.ODD_COUNCIL)
        return +90;  // 雇佣兵 vs 凋亡议会：含 FactionModifier(-30)

    // 非特殊组合：查询标准旧识关系表
    return QueryOldAcquaintanceTable(player_background, npc_faction);
}
```

**多派系 NPC 处理**：
- 如果 NPC 同时属于多个派系（如 锈网成员 + 凋亡议会线人），按以下优先级选择主要派系进行查询
- **派系优先级规则**（由 NPC AI System 的 `NPCIdentityType` 决定，本系统遵循该优先级）：
  1. **凋亡议会** - 最高优先级（核心敌人派系）
  2. **灰烬团** - 次高优先级（武装组织）
  3. **锈网** - 中优先级（灰色地带）
  4. **苍白之手** - 低优先级（观望态度）
  5. **无声者** - 最低优先级（潜在盟友）
  6. **黑帮** - 根据帮派内层级（小头目 > 普通成员）

**实现逻辑**：
```csharp
Faction GetPrimaryFaction(NPC_ID npc_id) {
    var identities = NPC_AISystem.GetNPCIdentities(npc_id);  // 已在 npc-ai-system.md Line 299 定义
    // 按优先级排序返回第一个匹配
    return identities.OrderByDescending(GetFactionPriority).First();
}

int GetFactionPriority(Faction f) {
    // 返回值越大表示优先级越高
    // 优先级顺序：凋亡议会(4) > 灰烬团(2) > 锈网(1) > 苍白之手(-1) > 无声者(-2)
    switch(f) {
        case Faction.ODD_COUNCIL: return 4;   // 凋亡议会 - 最高优先级
        case Faction.ASH_LEGION: return 2;     // 灰烬团 - 次高优先级（武装组织）
        case Faction.ROTTEN_WEB: return 1;    // 锈网 - 低优先级（灰色地带，但比苍白/无声高）
        case Faction.PALE_HAND: return -1;      // 苍白之手 - 极低优先级（观望态度）
        case Faction.SILENT_ONES: return -2;    // 无声者 - 最低优先级（潜在盟友）
        case Faction.GANGS: return -3;         // 黑帮 - 根据层级调整（最低）
        default: return -99;
    }
}
```

> **接口实现状态**：`NPC_AISystem.GetNPCIdentities(npc_id)` 接口已在 npc-ai-system.md Line 304 定义（包含实现草案），返回 `List[Faction]`（按优先级从高到低排序）。

> **设计意图**：派系优先级反映了该 NPC 所属的"最核心身份"。例如，一个既是锈网成员又是凋亡议会线人的 NPC，其核心身份是凋亡议会线人，因此使用凋亡议会的态度计算参数。

**返回值查表（标准旧识加成）**：

| 触发条件 | 背景 | NPC 派系 | 关系性质 | 标准返回值 | 吸收净值（实际返回） |
|---------|------|---------|---------|:------:|:------------------:|
| `SAME_FACTION` | Agent | 锈网 | 前同僚/线人 | +25 | **+40**（含 FactionModifier -5） |
| `MERCE_TO_MERCE` | Mercenary | 灰烬团 | 同行 | +20 | **+65**（含 FactionModifier -10） |
| `MERCE_EMPLOYER` | Mercenary | 凋亡议会 | 曾是雇主 | +15 | **+90**（含 FactionModifier -30） |
| 无匹配 | 任意 | 苍白之手/其他 | — | 0 | 0 |

> **⚠️ NPC AI 系统使用说明**：本接口对于特殊组合返回吸收净值（含 FactionModifier），因此 NPC AI 系统在调用本接口后，**不应再单独应用 FactionModifier**。对于非特殊组合，返回标准值，NPC AI 需单独应用 FactionModifier。

## Formulas

### 公式1：理智惩罚计算

```
EffectiveSanityPenalty = BaseSanityPenalty × SanityPenaltyMultiplier
```

| 变量 | 值 | 来源 |
|------|---|------|
| `BaseSanityPenalty` | 来自各事件（如击杀 -5~-15） | Sanity/Rage 系统 |
| `SanityPenaltyMultiplier` | 普通人=0.9, 特工=0.8, 雇佣兵=1.15 | Character Background |

**示例**：特工击杀帮凶（基准惩罚 -8）
```
EffectiveSanityPenalty = -8 × 0.8 = -6.4
```
（特工比普通人少受 20% 的心理惩罚，因为他习惯暴力）

### 公式2：狂暴阈值计算

```
EffectiveFrenzyThreshold = BaseFrenzyThreshold + FrenzyThresholdModifier
```

| 变量 | 值 | 来源 |
|------|---|------|
| `BaseFrenzyThreshold` | 70 | Sanity/Rage 系统 |
| `FrenzyThresholdModifier` | 普通人=+8, 特工=±0, 雇佣兵=-8 | Character Background |

**示例**：雇佣兵触发狂暴
```
EffectiveFrenzyThreshold = 70 + (-8) = 62
```
（雇佣兵在 Rage > 62 时进入 FRENZIED，比普通人早 8 点）

### 公式3：NPC 初始态度计算

```
InitialAllegiance = BaseAllegiance + BackgroundAttitudeModifier + FactionModifier + OldAcquaintanceBonus
```

> **特殊组合处理规则**：
> - **非特殊组合**：FactionModifier 单独应用，使用标准 OldAcquaintanceBonus
> - **特殊组合**（特工 vs 锈网、雇佣兵 vs 灰烬团、雇佣兵 vs 凋亡议会）：FactionModifier 被吸收到 OldAcquaintanceBonus 中，此时 OldAcquaintanceBonus 返回吸收净值，**FactionModifier 不再单独计算**

| 变量 | 说明 | 数据来源 |
|------|------|---------|
| `BaseAllegiance` | NPC 基准态度（来自 NPC AI 系统） | NPC AI System |
| `BackgroundAttitudeModifier` | 背景态度修正（见态度矩阵，直接查表取值） | Character Background System |
| `FactionModifier` | 基于 NPC 派系的基准态度修正（见下表）。**特殊组合不适用**：FactionModifier 被吸收到 OldAcquaintanceBonus 中 | NPC AI System |
| `OldAcquaintanceBonus` | 旧识关系加成（特工/雇佣兵专属）。**三个特殊组合返回吸收净值**（FactionModifier 已包含，不再单独计算）：<br>• 特工 vs 锈网：+40<br>• 雇佣兵 vs 灰烬团：+65<br>• 雇佣兵 vs 凋亡议会：+90<br>**非特殊组合**：返回标准旧识加成（见接口返回值查表） | Character Background System（通过 QueryOldAcquaintanceBonus 接口） |

**FactionModifier 定义表（由 NPC AI System 提供给 Character Background 使用的数据结构定义）**：

> **接口说明**：Character Background 系统作为纯数据输出系统，本系统不持有 FactionModifier 数据。本表是 NPC AI System 提供给 Character Background 用于查询 FactionModifier 值的数据结构定义。当 Character Background 需要在公式3/4中应用 FactionModifier 时，通过接口从 NPC AI System 获取实际值。

> **苍白之手 BaseAllegiance 验证说明**：特工 vs 苍白之手在表2.1中为-10。
>
> **苍白之手 FactionModifier 验证说明**：
> - 公式3：表2.1值 = BaseAllegiance + AttitudeModifier_Agent(+15) + FactionModifier_苍白之手(-5) + OldAcquaintanceBonus(0)
> - 代入：-10 = -20 + (+15) + (-5) + 0 = -10 ✓
>
> **结论**：苍白之手 FactionModifier = **-5**（NPC AI System 确认值），BaseAllegiance = **-20**，公式计算与表2.1一致。

| NPC 类型 | FactionModifier | 说明 |
|---------|----------------|------|
| 黑帮普通成员 | -10 | 基础敌意 |
| 黑帮小头目 | -20 | 帮派管理层 |
| 受害者/线人 | -5 | 弱势群体（**2026-04-12修复**：从+15修正为-5，以匹配表2.1普通人vs受害者计算） |
| 凋亡议会成员 | -30 | 核心敌人 |
| 锈网成员 | -5 | 灰色地带 |
| 灰烬团成员 | -10 | 武装组织 |
| 苍白之手成员 | -5 | 观望态度（NPC AI System 确认值） |
| 无声者成员 | +10 | 潜在盟友 |

**BaseAllegiance 数值表（由 NPC AI System 提供，本系统用于公式计算验证）**：

> **重要说明**：下表为 NPC AI System 提供的 `BaseAllegiance` 基准态度值。本表用于公式推算验证，实际游戏运行时以 NPC AI System 中的数值为准。
>
> **确认状态（2026-04-13）**：以下数值已通过 NPC AI System 确认，与表2.1正向推算验证结果一致。

| NPC 类型 | BaseAllegiance | 说明 | 确认状态 |
|---------|----------------|------|:--------:|
| 黑帮普通成员 | -20 | 基础敌意 | ✅ 已确认 |
| 黑帮小头目 | -35 | 帮派管理层 | ✅ 已确认 |
| 受害者/线人 | +20 | 弱势群体 | ✅ 已确认 |
| 凋亡议会成员 | **-45** | 核心敌人 | ✅ 已确认（P0修复） |
| 锈网成员 | **-15** | 灰色地带 | ✅ 已确认（P0修复） |
| 灰烬团成员 | **-25** | 武装组织 | ✅ 已确认（P0修复） |
| 苍白之手成员 | -20 | 观望态度 | ✅ 已确认 |
| 无声者成员 | -20 | 潜在盟友 | ✅ 已确认 |

**表2.1 正向推算验证**：

> **计算规则**：
> - **非特殊组合**：`BaseAllegiance + FactionModifier + BackgroundAttitudeModifier + OldAcquaintanceBonus`
> - **特殊组合**：`BaseAllegiance + BackgroundAttitudeModifier + OldAcquaintanceBonus_吸收净值`（FactionModifier 不再单独计算）
> - **受害者/苍白之手**：使用背景调整机制（见公式3下节说明），BackgroundAdjustment 已包含在 EffectiveFactionModifier 中

| 组合 | BaseAllegiance | + EffectiveFactionModifier（含BackgroundAdjustment） | + BackgroundAttitudeModifier | + OldAcquaintanceBonus | = 表2.1值 |
|------|---------------|---------------------------------------------------|---------------------------|----------------------|-----------|
| 普通人 vs 锈网 | -15 | -5 | +10 | 0 | **-10** ✓ |
| **特工 vs 锈网** | -15 | （吸收） | +15 | **+40**（吸收净值） | **+40** ✓ |
| 雇佣兵 vs 锈网 | -15 | -5 | -5 | 0 | **-5** ✓ |
| 普通人 vs 灰烬团 | -25 | -10 | +10 | 0 | **-25** ✓ |
| 特工 vs 灰烬团 | -25 | -10 | +15 | 0 | **-20** ✓ |
| **雇佣兵 vs 灰烬团** | -25 | （吸收） | -5 | **+65**（吸收净值） | **+35** ✓ |
| 普通人 vs 凋亡议会 | -45 | -30 | +10 | 0 | **-35** ✓ |
| 特工 vs 凋亡议会 | -45 | -30 | +15 | 0 | **-30** ✓ |
| **雇佣兵 vs 凋亡议会** | -45 | （吸收） | -5 | **+90**（吸收净值） | **+40** ✓ |
| 普通人 vs 受害者 | +20 | **-5**（含 VictimAdjustment=0） | +10 | 0 | **+25** ✓ |
| 特工 vs 受害者 | +20 | **-25**（含 VictimAdjustment=-20） | +15 | 0 | **+10** ✓ |
| 雇佣兵 vs 受害者 | +20 | **-35**（含 VictimAdjustment=-30） | -5 | 0 | **-20** ✓ |
| 普通人 vs 苍白之手 | -20 | **-5** | +10 | 0 | **-15** ✓ |
| 特工 vs 苍白之手 | -20 | **-5** | +15 | 0 | **-10** ✓ |
| 雇佣兵 vs 苍白之手 | -20 | **-5** | -5 | 0 | **-15** ✓ |

> ✅ **计算不一致修复状态（2026-04-12 v2.2）**：
> - **受害者组合**：✅ 已修复，BackgroundAdjustment（特工-20，雇佣兵-30），公式结果与表2.1一致
> - **苍白之手组合**：✅ 已修复，调整 BackgroundAdjustment_Agent=0 后公式结果与表2.1全部一致
> - **FactionModifier**：苍白之手保持-5（NPC AI System 确认值），BackgroundAdjustment 在 Character Background 内部处理

### 公式4：首次遭遇态度计算（含首次遭遇加成）

```
FirstEncounterAllegiance = BaseAllegiance + BackgroundAttitudeModifier + FactionModifier + OldAcquaintanceBonus + FirstEncounterBonus
```

> **适用条件**：仅当 `HasMetFaction[NPC.faction] == false` 时（玩家首次与该派系遭遇）才应用首次遭遇加成。

> **特殊组合说明**：对于特工 vs 锈网、雇佣兵 vs 灰烬团、雇佣兵 vs 凋亡议会这三个特殊组合，FactionModifier 被**吸收**到 OldAcquaintanceBonus 中计算（详见态度矩阵脚注说明）。这是因为这些组合的关系基础由背景故事定义，而非由 NPC 派系决定。

| 变量 | 说明 | 数据来源 |
|------|------|---------|
| `BaseAllegiance` | NPC 基准态度（来自 NPC AI 系统） | NPC AI System |
| `BackgroundAttitudeModifier` | 背景态度修正（见态度矩阵，直接查表取值） | Character Background System |
| `FactionModifier` | 基于 NPC 派系的基准态度修正（见公式3下表）。**特殊组合除外**：特工 vs 锈网、雇佣兵 vs 灰烬团、雇佣兵 vs 凋亡议会的 FactionModifier 被吸收到 OldAcquaintanceBonus | NPC AI System |
| `OldAcquaintanceBonus` | 旧识关系加成（特工/雇佣兵专属，见旧识关系修正表）。**三个特殊组合的 OldAcquaintanceBonus_吸收净值**（FactionModifier已被吸收，不再单独计算）：<br>• 特工 vs 锈网：+40<br>• 雇佣兵 vs 灰烬团：+65<br>• 雇佣兵 vs 凋亡议会：+90 | Character Background System |
| `FirstEncounterBonus` | 首次遭遇加成（见边缘情况1首次遭遇加成表） | Character Background System |
| `HasMetFaction[FACTION_ID]` | 标记玩家是否已与该派系发生过对峙/交互（由 Gritty Takedowns 系统持有） | Gritty Takedowns 系统 |

**首次遭遇加成查表**：

| 背景 | NPC 派系 | FirstEncounterBonus | 首次遭遇最终态度 |
|------|---------|---------------------|-----------------|
| 普通人 | 锈网 | +15 | **0**（BaseAllegiance -15 + AttitudeModifier +10 + FactionModifier -5 + OldAcquaintanceBonus 0 + FirstEncounterBonus +15 = 0） |
| 普通人 | 灰烬团 | +10 | **-10** |
| **特工** | **锈网** | **+15** | **+55**（BaseAllegiance -15 + AttitudeModifier +15 + OldAcquaintanceBonus吸收净值+40 + FirstEncounterBonus +15 = +55） |
| 特工 | 凋亡议会 | +15 | **-30**（BaseAllegiance -30 + AttitudeModifier +15 + FactionModifier -30 + OldAcquaintanceBonus 0 + FirstEncounterBonus +15 = -30） |
| **雇佣兵** | **灰烬团** | **+20** | **+55**（BaseAllegiance -25 + AttitudeModifier -5 + OldAcquaintanceBonus吸收净值+65 + FirstEncounterBonus +20 = +55） |
| **雇佣兵** | **凋亡议会** | **+15** | **+55**（BaseAllegiance -45 + AttitudeModifier -5 + OldAcquaintanceBonus吸收净值+90 + FirstEncounterBonus +15 = +55） |

> **计算说明**：首次遭遇最终态度 = BaseAllegiance + BackgroundAttitudeModifier + OldAcquaintanceBonus_吸收净值 + FirstEncounterBonus（FactionModifier已被吸收，不单独计算）

> **注意**：首次窗口期只触发一次。触发后 `HasMetFaction` 设为 `true`，后续遭遇不再应用首次遭遇加成，使用公式3计算。

## Edge Cases

### 边缘情况 1：背景选择后首次遭遇特定 NPC

**场景**：玩家选择特工后，首次遭遇锈网成员。

**首次遭遇判定机制**：

| 数据结构 | 类型 | 说明 |
|---------|------|------|
| `HasMetFaction[FACTION_ID]` | bool | 标记玩家是否已与该派系发生过对峙/交互（首次遭遇后设为 true） |

> **实现说明**：
> - 此 flag 由 **Gritty Takedowns 系统** 持有并管理（存储于 Gritty Takedowns 系统内部的玩家状态数据结构中）
> - 设置时机：`ConfrontationStartRequest` 发送时检查 `HasMetFaction[NPC.faction]`，若为 false 则触发首次遭遇加成并设为 true
> - 首次遭遇加成数据由 **Character Background 系统** 提供（见下表）

**首次遭遇加成规则**（由 Character Background 系统定义，供 Gritty Takedowns 查询）：

| 背景 | 派系 | 首次遭遇加成 | 首次遭遇最终态度 | 触发条件 |
|------|------|-------------|-----------------|---------|
| 普通人 | 锈网 | +15 | **0** | `HasMetFaction[锈网] == false` |
| 普通人 | 灰烬团 | +10 | **-10** | `HasMetFaction[灰烬团] == false` |
| **特工** | **锈网** | **+15** | **+55** | `HasMetFaction[锈网] == false` |
| 特工 | 凋亡议会 | +15 | **-30** | `HasMetFaction[凋亡议会] == false` |
| **雇佣兵** | **灰烬团** | **+20** | **+55** | `HasMetFaction[灰烬团] == false` |
| **雇佣兵** | **凋亡议会** | **+15** | **+55** | `HasMetFaction[凋亡议会] == false` |

> **计算说明**：首次遭遇最终态度 = BaseAllegiance + BackgroundAttitudeModifier + OldAcquaintanceBonus_吸收净值 + FirstEncounterBonus。旧识对话（如特工 vs 锈网）有独立的触发逻辑，在首次加成之后根据对话结果调整态度。
>
> **FactionModifier 吸收机制说明**：特工 vs 锈网、雇佣兵 vs 灰烬团、雇佣兵 vs 凋亡议会这三个组合的 FactionModifier 被完全吸收到 OldAcquaintanceBonus_吸收净值中。这是基于背景故事的特殊关系（特工曾是锈网成员、雇佣兵曾是灰烬团同行/凋亡议会雇主），由 Character Background 系统主导。

**FactionModifier 吸收机制决策树**：

```
// 代码层面的实现建议

// 特殊组合判定函数
bool IsSpecialCombination(BackgroundType player_background, Faction npc_faction) {
    return (player_background == BackgroundType.AGENT && npc_faction == Faction.ROTTEN_WEB)
        || (player_background == BackgroundType.MERCENARY && npc_faction == Faction.ASH_LEGION)
        || (player_background == BackgroundType.MERCENARY && npc_faction == Faction.ODD_COUNCIL);
}

// 获取旧识关系加成
int GetOldAcquaintanceBonus(BackgroundType player_background, Faction npc_faction) {
    // 特殊组合：返回吸收后净值（FactionModifier 已包含）
    if (player_background == BackgroundType.AGENT && npc_faction == Faction.ROTTEN_WEB)
        return +40;  // 特工 vs 锈网
    if (player_background == BackgroundType.MERCENARY && npc_faction == Faction.ASH_LEGION)
        return +65;  // 雇佣兵 vs 灰烬团
    if (player_background == BackgroundType.MERCENARY && npc_faction == Faction.ODD_COUNCIL)
        return +90;  // 雇佣兵 vs 凋亡议会

    // 非特殊组合：查询标准旧识关系表
    return QueryOldAcquaintanceTable(player_background, npc_faction);
}

// 计算初始态度
int CalculateInitialAllegiance(BackgroundType player_background, Faction npc_faction, bool is_first_encounter) {
    base = GetBaseAllegiance(npc_faction);
    attitude_mod = GetAttitudeModifier(player_background, npc_faction);
    old_acq_bonus = GetOldAcquaintanceBonus(player_background, npc_faction);

    if (is_first_encounter) {
        first_encounter_bonus = GetFirstEncounterBonus(player_background, npc_faction);
        return base + attitude_mod + old_acq_bonus + first_encounter_bonus;
    } else {
        // FactionModifier 仅在非特殊组合时单独应用
        if (!IsSpecialCombination(player_background, npc_faction)) {
            faction_mod = GetFactionModifier(npc_faction);
            return base + attitude_mod + faction_mod + old_acq_bonus;
        } else {
            // 特殊组合：FactionModifier 已被吸收到 old_acq_bonus
            return base + attitude_mod + old_acq_bonus;
        }
    }
}
```

> **实现说明**：将 `IsSpecialCombination()` 封装为独立函数，使代码逻辑更清晰。特殊组合的判断在函数内部完成，调用方无需关心吸收机制的细节。

**FactionModifier 吸收机制状态机视图**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        态度计算状态机                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐    输入(player_background, npc_faction)                    │
│  │              │                                                          │
│  ▼              │                                                          │
│  ┌──────────────┴──────────────┐                                           │
│  │   IsSpecialCombination()?    │                                           │
│  └──────────────┬──────────────┘                                           │
│                 │                                                           │
│         ┌───────┴───────┐                                                  │
│         │               │                                                  │
│        YES             NO                                                  │
│         │               │                                                  │
│         ▼               ▼                                                  │
│  ┌──────────────┐  ┌─────────────────────────────┐                         │
│  │ 使用吸收净值   │  │ 查询标准 OldAcquaintanceTable │                        │
│  │ OldAcqBonus  │  │ + 查询 FactionModifier        │                        │
│  │ (+40/+65/+90)│  │ (+标准值)                     │                        │
│  └──────┬───────┘  └─────────────┬───────────────┘                         │
│         │                        │                                          │
│         └───────────┬────────────┘                                          │
│                     ▼                                                        │
│           ┌─────────────────┐                                               │
│           │ 返回最终 OldAcq   │                                               │
│           │ 加成值            │                                               │
│           └────────┬─────────┘                                               │
│                    │                                                         │
│                    ▼                                                         │
│           ┌─────────────────┐                                                │
│           │ 公式3/4 计算     │                                                │
│           │ (首次/非首次)    │                                                │
│           └────────┬────────┘                                                │
│                    │                                                         │
│                    ▼                                                         │
│           ┌─────────────────┐                                                │
│           │   返回结果       │                                                │
│           │ InitialAllegiance│                                               │
│           └─────────────────┘                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

**吸收机制错误处理指南**：

```csharp
/// <summary>
/// FactionModifier 吸收机制错误处理
/// </summary>
class FactionAbsorptionErrorHandler {
    // 错误场景1：硬编码净值与表2.1不一致
    void ValidateAbsorbedValues() {
        // 在游戏启动时执行验证
        var expected = new Dictionary<(BackgroundType, Faction), int> {
            { (BackgroundType.AGENT, Faction.ROTTEN_WEB), +40 },
            { (BackgroundType.MERCENARY, Faction.ASH_LEGION), +65 },
            { (BackgroundType.MERCENARY, Faction.ODD_COUNCIL), +90 },
        };

        foreach (var kvp in expected) {
            var actual = GetOldAcquaintanceBonus(kvp.Key.Item1, kvp.Key.Item2);
            if (actual != kvp.Value) {
                // 严重错误：硬编码净值与设计文档不一致
                LogError($"FactionModifier absorption value mismatch: " +
                    $"({kvp.Key.Item1}, {kvp.Key.Item2}) = {actual}, expected {kvp.Value}");
                // 中断游戏加载，提示开发者检查硬编码常量
                throw new ConfigurationException("FactionModifier absorption values out of sync with design doc");
            }
        }
    }

    // 错误场景2：特殊组合判定逻辑不一致
    void ValidateSpecialCombinationConsistency() {
        // 确保 IsSpecialCombination() 与 GetOldAcquaintanceBonus() 的判定一致
        var specialCases = new[] {
            (BackgroundType.AGENT, Faction.ROTTEN_WEB),
            (BackgroundType.MERCENARY, Faction.ASH_LEGION),
            (BackgroundType.MERCENARY, Faction.ODD_COUNCIL),
        };

        foreach (var combo in specialCases) {
            bool isSpecial = IsSpecialCombination(combo.Item1, combo.Item2);
            int oldAcqBonus = GetOldAcquaintanceBonus(combo.Item1, combo.Item2);
            bool hasAbsorbedValue = oldAcqBonus != QueryOldAcquaintanceTable(combo.Item1, combo.Item2);

            if (isSpecial != hasAbsorbedValue) {
                LogError($"Inconsistent special combination handling: " +
                    $"IsSpecialCombination={isSpecial}, HasAbsorbedValue={hasAbsorbedValue}");
            }
        }
    }

    // 错误场景3：FactionModifier 在特殊组合中被错误应用
    void ValidateNoDoubleCounting() {
        // 在计算完成后验证 FactionModifier 未被重复计算
        // 此检查仅在调试模式下执行
        #if DEBUG
        foreach (var combo in specialCases) {
            // 对于特殊组合，验证 FactionModifier 未被包含在 OldAcqBonus 中
            // 通过正向计算：Base + Attitude + OldAcq(=Absorbed) = 表2.1
            // 如果结果正确，说明 FactionModifier 未被重复计算
            int calculated = CalculateInitialAllegiance(combo.Item1, combo.Item2, false);
            int expected = GetTable21Value(combo.Item1, combo.Item2);

            if (calculated != expected) {
                LogWarning($"Potential double-counting detected: " +
                    $"calculated={calculated}, expected={expected}");
            }
        }
        #endif
    }
}
```

**完整调用序列图（时序）**：

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│ NPC AI      │     │ Character        │     │ Gritty Takedowns    │
│ System      │     │ Background        │     │ System              │
└──────┬──────┘     └────────┬─────────┘     └──────────┬──────────┘
       │                     │                        │
       │                     │  GetPrimaryFaction()   │
       │◄────────────────────┼────────────────────────│
       │  GetNPCIdentities() │                        │
       │────────────────────►│                        │
       │                     │                        │
       │  QueryOldAcqBonus() │                        │
       │◄────────────────────┼                        │
       │                     │                        │
       │                     │  GetFirstEncounterBonus()│
       │◄────────────────────┼────────────────────────│
       │                     │                        │
       │◄── InitialAllegiance (公式3/4结果) ──────────┼
       │                     │                        │
```

**处理流程**：
1. 玩家发起 `ConfrontationStartRequest(npc_id, player_background)` 到 Gritty Takedowns
2. Gritty Takedowns 向 Character Background 查询 `GetFirstEncounterBonus(player_background, npc_faction)`
3. Gritty Takedowns 查询本地 `HasMetFaction[NPC.faction]` flag
4. 如果 `== false`：
   - 将 Character Background 返回的首次遭遇加成应用到态度计算（使用公式4变体）
   - 将 flag 设为 `true`
5. 如果 `== true`：不应用首次遭遇加成（已过首次窗口期），使用公式3计算
6. 后续计算：`InitialAllegiance = BaseAllegiance + BackgroundAttitudeModifier + FactionModifier + OldAcquaintanceBonus`

> **注意**：首次遭遇加成与 OldAcquaintanceBonus 在首次遭遇时**叠加计算**，用于增强首次接触的戏剧效果。详见上表叠加计算说明。首次窗口期只触发一次。旧识对话（如特工 vs 锈网）有独立的触发逻辑，在首次加成之后根据对话结果调整态度。

**接口定义**：

```csharp
interface IFirstEncounterBonusProvider {
    /// <summary>
    /// 返回首次遭遇加成原始值。
    /// 注意：此接口不检查 HasMetFaction flag，调用方需先检查 HasMetFaction[npc_faction] == false
    /// 再将返回值应用到态度计算。若 HasMetFaction[npc_faction] == true，应使用公式3而非公式4。
    /// </summary>
    int GetFirstEncounterBonus(BackgroundType player_background, Faction npc_faction);
}
```

> Character Background 系统实现此接口，Gritty Takedowns 在 `ConfrontationStartRequest` 时调用。
> 调用流程：Gritty Takedowns 先检查 `HasMetFaction[NPC.faction]`，若为 false 则调用此接口获取首次遭遇加成并应用；若为 true 则不调用，直接使用公式3计算。

### 边缘情况 2：旧识对话触发后 NPC 死亡

**场景**：特工与锈网成员的旧识对话进行中，该 NPC 死亡

**死亡责任追踪**：

| 死亡类型 | KillSource | 效果 |
|---------|-----------|------|
| 玩家主动攻击击杀 | `DIRECT_KILL` | 收益丢失，理智惩罚正常计算 |
| 玩家环境处决 | `EXECUTION_KILL` | 收益丢失，理智惩罚正常计算 |
| 玩家攻击导致 NPC 倒地后被环境杀死 | `INDIRECT_KILL` | 收益丢失，理智惩罚正常计算 |
| 对话期间第三方 NPC 误杀 | `ACCIDENTAL_KILL` | 收益丢失，不计入理智惩罚 |
| 对话期间玩家被击杀导致 NPC 撤离 | `SELF_DEFENSE` | 收益保留（NPC 自己离开） |

**旧识对话收益定义**：
- 线索：直接获得，不受死亡影响
- 态度变化：立即生效，不受死亡影响
- 后续对话分支解锁：如果 NPC 存活则保留，死亡则失效

### 边缘情况 3：主动击杀 vs 意外死亡定义

**场景**：需要区分玩家行为导致的 NPC 死亡类型

**KillSource 枚举定义**（由 Gritty Takedowns 系统定义和管理）：

| 值 | 场景 | 理智惩罚 | 收益丢失 |
|----|------|---------|---------|
| `DIRECT_KILL` | 玩家直接攻击击杀 | 正常计算 | 是 |
| `EXECUTION_KILL` | 环境处决 | 正常计算 | 是 |
| `INDIRECT_KILL` | 玩家攻击导致倒地后环境致死 | 正常计算 | 是 |
| `ACCIDENTAL_KILL` | 第三方 NPC 误杀 | 不计入 | 是 |
| `SELF_DEFENSE` | NPC 自卫导致玩家死亡/NPC 撤离 | 不适用 | 否（NPC 存活） |

> **实现说明**：KillSource 枚举在实际实现中由 Gritty Takedowns 系统管理（定义于 gritty-takedowns.md 的 `KillSource` 枚举），通过 `KillTagEvent` 事件传递。理智惩罚计算使用 `NPCIdentityType`（ENEMY/ACCOMPLICE/VICTIM/UNKNOWN），而 KillSource 用于追踪更细粒度的死亡责任，用于判断"收益丢失"等边缘情况。

**设计意图**：精确追踪死亡责任，确保"玩家主动选择击杀"才承担理智惩罚。

### 边缘情况 4：参数边界检查

**场景**：调参时某项系数设置超出安全范围

**处理**：
- 所有参数在游戏启动时进行边界检查
- 超出安全范围的参数自动钳制到边界值
- 日志记录违规参数和修正后的值

**安全范围**：所有系数 0.75 ~ 1.25（狂暴阈值调整为 ±15）

## Dependencies

### 架构说明

本系统为**纯数据输出系统**，不持有任何游戏状态，仅在玩家选择背景后向下游系统传递参数修正值。本系统不监听任何外部事件。

### 上游依赖（系统依赖谁）

本系统**不依赖任何其他系统**。背景选择由玩家在开局时手动完成，选择后背景数据被推送至各下游系统。

### 下游依赖（谁依赖本系统）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **NPC AI系统** | 硬依赖 | 调用 `QueryOldAcquaintanceBonus` 获取旧识关系加成；读取 `AttitudeModifier` 计算 NPC 初始态度；调用 `GetNPCIdentities(npc_id)` 获取 NPC 多派系身份列表（详见下方接口定义） |
| **DialogTree** | 硬依赖 | 读取 `BackgroundType` 判断对话选项可见性和旧识对话分支过滤 |
| **理智/愤怒系统** | 硬依赖 | **下游依赖（接收参数）**。本系统向 Sanity/Rage Meter 输出 `SanityPenaltyMultiplier` 和 `FrenzyThresholdModifier`，Sanity/Rage Meter 作为消费方读取这两个参数。<br>• 数据流向：Character Background → Sanity/Rage Meter<br>• 输出参数：<br>  - `SanityPenaltyMultiplier`：理智惩罚系数（普通人=0.9x, 特工=0.8x, 雇佣兵=1.15x）<br>  - `FrenzyThresholdModifier`：狂暴阈值调整（普通人=+8, 特工=±0, 雇佣兵=-8）|
| **叙事系统 (Narrative System)** | 软依赖 | 接收背景类型用于 SKILL 系列模块解锁条件判断（BackgroundType 影响叙事模块的可用性） |
| **Gritty Takedowns 系统** | 硬依赖 | 调用 `IFirstEncounterBonusProvider.GetFirstEncounterBonus()` 获取首次遭遇加成；持有 `HasMetFaction` flag 并管理首次遭遇加成的应用时机 |

#### GetNPCIdentities 接口定义与替代实现方案

> **TODO P0（2026-04-15）**：`GetNPCIdentities(npc_id)` 接口是 NPC AI System 的必需接口，本系统依赖此接口进行多派系 NPC 的派系优先级判断。当前 NPC AI System 尚未实现该接口，需要在 Vertical Slice 阶段前完成。
>
> **接口确认状态（2026-04-14）**：`GetNPCIdentities(npc_id)` 接口已在 npc-ai-system.md Line 304 定义（待实现），返回 `List[Faction]`（按优先级从高到低排序）。本章节提供接口规范说明。

**接口规范定义**：

```csharp
/// <summary>
/// 返回 NPC 所属的所有派系身份列表。
/// 用于多派系 NPC 的派系优先级判断。
/// </summary>
/// <param name="npc_id">NPC 唯一标识符</param>
/// <returns>派系枚举值列表，按优先级从高到低排序</returns>
List<Faction> GetNPCIdentities(NPC_ID npc_id);
```

**返回值数据结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| 返回值 | `List<Faction>` | NPC 所属的派系枚举列表，按优先级从高到低排序 |

**派系优先级定义**（由 NPC AI System 的 `NPCIdentityType` 决定）：

| 优先级 | 派系 | 枚举值 | 说明 |
|:------:|------|--------|------|
| 1 | 凋亡议会 | `ODD_COUNCIL` | 最高优先级（核心敌人派系） |
| 2 | 灰烬团 | `ASH_LEGION` | 次高优先级（武装组织） |
| 3 | 锈网 | `ROTTEN_WEB` | 中优先级（灰色地带） |
| 4 | 苍白之手 | `PALE_HAND` | 低优先级（观望态度） |
| 5 | 无声者 | `SILENT_ONES` | 最低优先级（潜在盟友） |
| 6 | 黑帮 | `GANGS` | 根据帮派内层级调整（小头目 > 普通成员） |

**替代实现方案（基于现有接口）**：

> **TODO P0 说明**：以下替代方案是临时实现，当 NPC AI System 正式实现 `GetNPCIdentities` 接口后，必须替换为正式接口调用。此临时方案仅用于 Vertical Slice 阶段前的开发过渡。

由于 `GetNPCIdentities` 尚未在 NPC AI System 中实现，可使用以下临时替代方案：

```csharp
/// <summary>
/// 替代实现：通过 NPC AI System 的现有查询接口间接获取派系信息
/// 注意：这是临时替代方案，最终应在 NPC AI System 中实现 GetNPCIdentities 接口
/// </summary>
List<Faction> GetNPCIdentitiesAlternative(NPC_ID npc_id) {
    // 方案：查询 NPC 的 allegiance 变化历史或直接查询派系关系表
    // 由于 NPC AI System 的 allegiance 计算需要派系信息作为输入，
    // 此替代方案需要在 NPC AI System 内部预留派系查询接口

    // 临时实现思路1：通过 Allegiance 间接推断（不推荐，逻辑不清晰）
    // 临时实现思路2：在 NPC AI System 中新增 GetFactionList(npc_id) 接口（推荐）

    // 推荐方案：在 NPC AI System 中新增接口
    return NPC_AISystem.GetFactionList(npc_id);  // 新接口
}
```

**推荐解决方案**：

1. **短期（立即实施）**：在 NPC AI System (npc-ai-system.md) 的查询接口章节中添加 `GetNPCIdentities` 接口定义

2. **实现位置**：npc-ai-system.md 的「查询接口规范」章节，与 `QueryState`、`QueryAlertState` 等接口并列

3. **接口定义草案**（供 NPC AI System 实现参考）：

```csharp
/// <summary>
/// NPC AI System 提供：获取 NPC 的所有派系身份
/// </summary>
/// <param name="npc_id">NPC 唯一标识符</param>
/// <returns>派系枚举值列表，按优先级从高到低排序</returns>
/// <remarks>
/// 多派系 NPC 示例：
/// - 锈网成员 + 凋亡议会线人 → 返回 [ODD_COUNCIL, ROTTEN_WEB]
/// - 普通黑帮成员 → 返回 [GANGS]
/// </remarks>
List<Faction> GetNPCIdentities(NPC_ID npc_id);
```

4. **Character Background 系统内部实现调整**：在 `GetPrimaryFaction` 函数中使用上述接口后，更新实现代码

```csharp
Faction GetPrimaryFaction(NPC_ID npc_id) {
    // 使用 NPC AI System 提供的 GetNPCIdentities 接口
    var identities = NPC_AISystem.GetNPCIdentities(npc_id);
    return identities.OrderByDescending(GetFactionPriority).First();
}
```

### 依赖关系矩阵

```
┌─────────────────────────────────────────────────────────────────┐
│                        Character Background                      │
│  （纯数据输出，开局选择后向以下系统推送参数）                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ──AttitudeModifier───────────────► NPC AI System               │
│  ──OldAcquaintanceBonus*──────────► NPC AI System（Query 接口） │
│  ──BackgroundType─────────────────► DialogTree                  │
│  ──SanityPenaltyMultiplier───────► Sanity/Rage Meter           │
│  ──FrenzyThresholdModifier────────► Sanity/Rage Meter           │
│  ──BackgroundType─────────────────► Narrative System（软依赖）  │
│                                                                  │
│  * OldAcquaintanceBonus / IFirstEncounterBonusProvider 通过 Query 接口由调用方主动查询 │
│  * IFirstEncounterBonusProvider ──► Gritty Takedowns（Query 接口）│
└─────────────────────────────────────────────────────────────────┘
```

> **⚠️ Pull 模式接口说明**：
> - `QueryOldAcquaintanceBonus` 和 `IFirstEncounterBonusProvider` 是 **Pull 模式**接口
> - 调用方（NPC AI System、Gritty Takedowns）是主动方，本系统是被动响应方
> - 在语义上，这更像是"调用方依赖本系统提供的数据"，而非传统的"下游依赖上游"
> - 为避免歧义，本文档在 Dependencies 章节中将这些接口归类于"下游依赖"（因为其他系统在查询这些数据），而非"上游依赖"

### 跨系统接口定义

#### INarrativeModuleQuery 接口（Narrative System 调用）

```csharp
// Narrative System 调用此接口查询模块解锁状态
interface INarrativeModuleQuery {
    /// <summary>
    /// 查询指定背景类型对应的 SKILL 模块是否已解锁
    /// </summary>
    bool IsSkillModuleUnlocked(BackgroundType background);

    /// <summary>
    /// 获取指定背景对应的 SKILL 模块 ID
    /// </summary>
    string GetSkillModuleId(BackgroundType background);
}
```

> **实现说明**：Character Background 系统实现此接口。背景选择时自动解锁对应 SKILL 模块：
> - Agent → SKILL_01
> - Civilian → SKILL_02
> - Mercenary → SKILL_03

## Tuning Knobs

背景系统的可调参数为 8 项核心参数修正，无特殊能力相关参数。

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `SanityPenaltyMultiplier_Civilian` | float | 0.9 | 0.75~1.0 | 普通人理智惩罚系数 |
| `SanityPenaltyMultiplier_Agent` | float | 0.8 | 0.7~0.95 | 特工理智惩罚系数 |
| `SanityPenaltyMultiplier_Mercenary` | float | 1.15 | 1.0~1.25 | 雇佣兵理智惩罚系数 |
| `FrenzyThresholdModifier_Civilian` | int | +8 | +3~+15 | 普通人狂暴阈值调整 |
| `FrenzyThresholdModifier_Agent` | int | ±0 | ±0 | 特工狂暴阈值调整 |
| `FrenzyThresholdModifier_Mercenary` | int | -8 | -15~-3 | 雇佣兵狂暴阈值调整 |
| `StealthModifier_Civilian` | float | 1.15 | 1.0~1.25 | 普通人潜行系数 |
| `StealthModifier_Agent` | float | 1.0 | 0.9~1.1 | 特工潜行系数 |
| `StealthModifier_Mercenary` | float | 0.85 | 0.75~1.0 | 雇佣兵潜行系数 |
| `CombatModifier_Civilian` | float | 0.8 | 0.7~0.95 | 普通人战斗系数 |
| `CombatModifier_Agent` | float | 0.95 | 0.85~1.05 | 特工战斗系数 |
| `CombatModifier_Mercenary` | float | 1.2 | 1.0~1.25 | 雇佣兵战斗系数 |
| `PerceptionModifier_Civilian` | float | 0.9 | 0.75~1.05 | 普通人感知系数 |
| `PerceptionModifier_Agent` | float | 1.15 | 1.1~1.15 | 特工感知系数（感知范围乘数1.1x~1.15x，上限1.15x以保持挑战平衡）。**联动约束**：与 `ClueAnalysisModifier_Agent` 联动调整；若上调 `PerceptionModifier_Agent` 至 1.15x，建议将 `ClueAnalysisModifier_Agent` 下调至 1.0x 以内以保持乘积 ≈ 1.15 |
| `PerceptionModifier_Mercenary` | float | 1.0 | 0.9~1.1 | 雇佣兵感知系数 |
| `EnvironmentModifier_Civilian` | float | 0.9 | 0.8~1.0 | 普通人环境交互系数 |
| `EnvironmentModifier_Agent` | float | 1.0 | 0.9~1.1 | 特工环境交互系数 |
| `EnvironmentModifier_Mercenary` | float | 1.2 | 1.0~1.25 | 雇佣兵环境交互系数 |
| `ClueAnalysisModifier_Civilian` | float | 1.1 | 1.0~1.2 | 普通人线索分析速度 |
| `ClueAnalysisModifier_Agent` | float | 0.86 | 0.75~1.1 | 特工线索分析速度（**2026-04-13修复**：从1.3x下调至0.86x以平衡特工双重优势乘积）。**联动约束**：与 `PerceptionModifier_Agent` 联动调整；若上调 `PerceptionModifier_Agent` 至 1.15x，建议将本参数控制在 1.0x 以内以保持乘积 ≈ 1.15 |
| `ClueAnalysisModifier_Mercenary` | float | 0.85 | 0.75~1.0 | 雇佣兵线索分析速度 |
| `AttitudeModifier_Civilian` | int | +10 | +5~+15 | 普通人NPC初始态度修正 |
| `AttitudeModifier_Agent` | int | +15 | +10~+20 | 特工NPC初始态度修正 |
| `AttitudeModifier_Mercenary` | int | -5 | -10~+5 | 雇佣兵NPC初始态度修正 |
| `OldAcquaintanceBonus_Max` | int | 25 | 20~30 | 旧识关系最大态度加成 |

> **⚠️ Playtest 验证重点**：`CombatModifier_Mercenary` 是影响"致命脆弱感"支柱的关键参数。建议在首次 playtest 时重点验证：雇佣兵战斗能力增强是否导致"脆弱感"体验显著下降。若玩家反馈雇佣兵战斗过于简单，应将上限下调至 1.1x。

**调参风险提示**：

| 参数 | 风险 | Playtest 验证重点 |
|------|------|------------------|
| `SanityPenaltyMultiplier_*` 设置过低 | 玩家杀戮代价降低，削弱理智系统的情感冲击力 | 验证杀戮心理惩罚是否符合预期 |
| `FrenzyThresholdModifier_Mercenary` 设置过低（如 -15） | 雇佣兵过早进入狂暴，可能导致"狂暴流"玩法固化 | **重点验证**：雇佣兵狂暴触发频率是否过高；验证狂暴流是否成为唯一可行玩法；建议收集数据：每次任务中雇佣兵狂暴触发的平均次数 vs 其他背景的战斗方式分布 |
| `PerceptionModifier_Agent` 设置过高 | 特工感知范围和信息优势过强，在潜行/监听为主要手段的游戏中可能导致特工成为唯一最优背景选择。已将安全范围上限降至 1.15x，首次 playtest 重点验证三个背景的吸引力平衡 | 验证特工是否成为唯一选择；收集三个背景的选择率分布 |
| **特工双重优势** | `PerceptionModifier_Agent`(1.15x) + `ClueAnalysisModifier_Agent`(0.86x) 乘积=0.99，已接近普通人乘积=0.99，三背景吸引力趋于平衡。调参时需保持乘积 ≈ 1.0 | 验证两项乘数乘积平衡（见 AC-11） |
| `CombatModifier_Mercenary` 设置过高 | 雇佣兵战斗能力过强，破坏"致命脆弱感"支柱 | 验证雇佣兵战斗难度是否符合预期 |

**特工感知系数联动调整公式**：

> 为防止特工成为唯一最优背景，当调整 `PerceptionModifier_Agent` 或 `ClueAnalysisModifier_Agent` 时，需按以下联动规则调整：

```
当 PerceptionModifier_Agent 上调 Δ 时：
    → ClueAnalysisModifier_Agent 下调 Δ × 0.5（最大下调至 1.0）

当 ClueAnalysisModifier_Agent 上调 Δ 时：
    → PerceptionModifier_Agent 下调 Δ × 0.5（最大下调至 1.0）

推荐组合范围（保持三背景平衡）：
| 组合 | PerceptionModifier_Agent | ClueAnalysisModifier_Agent | 乘积 |
|------|------------------------|---------------------------|------|
| 默认 | 1.15x | 0.86x | 0.99 |
| 平衡A | 1.1x | 0.9x | 0.99 |
| 平衡B | 1.05x | 0.94x | 0.99 |

验证方法：当特工的两项乘数乘积 ≈ 普通人两项乘数乘积时（≈ 0.99），三背景吸引力趋于平衡
```

## Visual/Audio Requirements

### 背景选择界面视觉要求

| 元素 | 要求 |
|------|------|
| **背景图标** | 每个背景一个标志性图标（普通人：父亲剪影；特工：耳机/通讯器；雇佣兵：战术手套） |
| **背景描述卡** | 显示背景名称、简短描述、参数预览 |
| **参数预览** | 悬停时显示详细参数说明（8项参数） |
| **选择确认** | 选中后高亮显示，确认按钮变为可用 |

### HUD 元素

| 元素 | 显示位置 | 显示条件 |
|------|---------|---------|
| **背景标识** | HUD 左下角 | 游戏内常驻显示当前背景图标和名称 |

## UI Requirements

> **职责边界说明**：本章节为**需求声明**，而非渲染实现规格。背景选择界面的最终渲染责任归 **UI System（ui-system.md）**，本文定义的布局与交互要求作为输入需求传递给 UI 系统，UI 系统负责具体实现。如两文档出现冲突，以 UI System 文档为技术权威，以本文档为需求权威。

### 背景选择界面

| 信息 | 显示位置 | 更新频率 | 条件 |
|-------------|-----------------|-----------------|--------|
| **背景名称** | 屏幕顶部中央 | 静态 | 无 |
| **背景图标** | 角色创建界面中央 | 静态 | 无 |
| **背景描述** | 背景图标下方 | 静态 | 无 |
| **参数列表** | 界面左侧 | 静态 | 无 |

## Acceptance Criteria

### 功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 三个背景可在开局选择界面正确显示 | 打开创建角色界面，验证三个背景选项可见 |
| AC-2 | 选择背景后，BackgroundType 和 BackgroundParameters 正确传递到各系统 | 监听 NPC AI/DialogTree/Sanity/Rage 接口，验证数据正确 |
| AC-3 | NPC 对不同背景产生正确初始态度 | 创建三个角色，对比同一 NPC 的初始态度是否符合态度矩阵 |
| AC-4 | 理智惩罚根据背景系数正确计算 | 击杀后检查理智变化值是否符合公式 |
| AC-5 | 狂暴阈值根据背景正确调整 | 积累 Rage 到不同背景的阈值，验证狂暴触发时机 |

### 跨系统验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-6 | DialogTree 根据背景显示/隐藏对话选项 | 对比三个背景的可用对话选项 |
| AC-7 | 旧识关系在特工/雇佣兵遭遇特定派系时触发 | 测试特工遭遇锈网、雇佣兵遭遇灰烬团时的额外态度加成 |
| AC-8 | 参数修改在游戏启动时正确加载 | 修改参数文件后启动游戏，验证效果变化 |

### 边缘情况验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-9 | 参数超出安全范围时自动钳制 | 设置系数为 0.5 或 1.5，验证启动后被钳制到 0.75 或 1.25 |
| AC-10 | 旧识对话 NPC 死亡后收益正确处理 | 在对话中击杀 NPC，验证收益丢失 |
| AC-11 | 三背景感知与分析能力平衡 | 验证特工的两项乘数乘积（PerceptionModifier × ClueAnalysisModifier）≈ 普通人两项乘数乘积（约 0.99）；若偏差超过 ±10%，需触发调参警告 |
| AC-12 | 雇佣兵的潜行能力（StealthModifier 0.85x）在标准关卡中被 NPC 发现的概率，高于普通人至少 20% | 在同一标准潜行关卡中，用三个背景走相同路线，统计各背景被发现的次数，验证雇佣兵 ≥ 普通人 × 1.2 |
| AC-13 | 普通人的战斗系数（0.8x）在1v1遭遇战中，存活率低于特工（0.95x）20% 以上 | 模拟相同条件的1v1遭遇战场景，各背景重复10次，对比存活率差异，验证普通人存活率显著低于特工 |
| AC-14 | 雇佣兵的环境交互系数（1.2x）使其每关可用环境处决路径数量最多 | 在含两种及以上环境处决机会的标准关卡中，对比三个背景可触发的环境处决路径数，雇佣兵数量应最多 |
| AC-15 | 雇佣兵狂暴流效率验证 | 分别记录雇佣兵潜行通关和狂暴清场通关的时长，验证狂暴流不会成为唯一主流打法 |

> **⚠️ AC-15 Playtest 数据验证计划**：
> - **验证时机**：需在 Alpha 阶段收集 playtest 数据后验证
> - **验证目标**：确认雇佣兵狂暴流是否成为唯一可行打法
> - **风险**：若 playtest 数据表明狂暴流效率显著高于潜行流（狂暴通关时长 < 潜行时长 70%），需触发调参流程
> - **调参方案**：如发现问题，调整 `FrenzyThresholdModifier`（建议从 -8 调至 -12~-15，降低狂暴触发门槛，使狂暴更频繁但每次持续时间更短）或相关系数以平衡两条路线效率

## Open Questions

| 问题 | 负责人 | 截止日期 | 状态 | 解决方案 |
|------|-------|----------|------|---------|
| 背景选择界面的美术风格 | Art Director | 待定 | ✅ 已解决 | **渐进式混合风格**：初期展示符号化图标（匕首=雇佣兵、眼睛=特工、脚印=普通人），选中后弹出写实风格背景故事卡片。参考《教团：1886》 |
| 不同背景的配音/台词差异 | Narrative Director | 待定 | ✅ 已解决 | **核心共享+背景差异**：所有背景共享主线内心独白，背景特有场景添加差异化独白（约15-20%），雇佣兵额外5-8条"父亲身份挣扎"独白 |
| 背景是否影响结局分支 | Narrative Director | 待定 | ✅ 已解决 | **维持现状**：背景不影响结局，但增加"同一结局不同解读"的过程差异。背景应主要影响过程体验而非终点 |
| 各参数的默认值是否需要根据实际测试调整 | Game Designer | 待定 | ✅ 已解决 | **预设+验证+调参三阶段**：Alpha Playtest后按AC-12/13/14/15数据调整。触发条件：AC-12(雇佣兵被发现概率<普通人×1.15)、AC-13(特工存活率优势<15%)、AC-15(狂暴流<潜行流70%时长) |
