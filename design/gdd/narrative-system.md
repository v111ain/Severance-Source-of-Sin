# 叙事系统设计框架 (Narrative System Framework)

> **Status**: Approved
> **Author**: Narrative Director
> **Last Updated**: 2026-04-10
> **Revision Notes**: 2026-04-10 v1.0 设计审查修复后再次审查修复：P1-统一Section 3.1与Section 4.1的moral_standing引用关系；P1-修复dominant_trait判定公式变量名不一致；P1-添加AC-8b验收mercy_count软上限；P2-补充旁观者路线边界条件；P2-Section 4.1添加"影响的系统"列；P2-补充content_path路径格式约定
> **Priority**: Vertical Slice
> **Layer**: Narrative
> **Implements Pillar**: 罪恶的深度 (Depth of Sin), 致命的脆弱感 (Lethal Fragility), 沉重、不洁的暴力 (Gritty Violence)

---

## 1. Overview

本设计框架定义了《断绝：罪恶之源》的叙事系统架构，涵盖道德系统、角色背景模块化系统、主线叙事节点规划及碎片叙事策略。系统设计遵循"三位一体道德框架"——认知层、后果层、恢复层的三环紧扣机制，确保玩家的每一个道德选择都产生深远且可追溯的心理影响。

**核心设计哲学**：
- 叙事不是被动呈现的故事，而是玩家行为铸就的因果网络
- 道德不是数值的涨跌，而是身份认同的转变
- 真相不是被告知的，而是被发掘和承受的

**已确认的设计决策**：

| 决策项 | 选择 | 说明 |
|--------|------|------|
| 误杀发现机制 | LOS系统UI确认提示 | 击杀未知身份NPC后，UI显示"身份未确认"警告 |
| 道德状态衰减 | 事件驱动 | 道德画像取决于长期累计行为，非近期行为 |
| 模块数量上限 | 有上限 | 每类背景模块最多3个，共12个 |
| 核心叙事线锁定 | 有前置条件 | 关键剧情需要满足道德/模块条件解锁 |

---

## 2. Player Fantasy

### 2.1 核心情感体验

**"在黑暗中行走的判决者"**

玩家扮演的复仇者不是超级英雄，而是一个被悲剧压垮、却仍试图在黑暗中维持人性底线的普通人。每一个杀戮决定都必须经过内心的挣扎——不是因为系统惩罚，而是因为玩家真实看到了NPC背后的故事。

**情感弧线设计**：

| 阶段 | 玩家情感 | 叙事体验 |
|------|---------|---------|
| 开局 | 愤怒、悲痛、坚定 | 女儿被贩卖的创伤记忆，玩家带着明确的复仇动机 |
| 中期 | 困惑、动摇、反思 | 发现NPC的无辜背景，开始质疑"判决者"身份的正当性 |
| 高潮 | 撕裂、绝望、救赎 | 面对核心反派的最终抉择——杀/不杀/救赎 |
| 结局 | 平静/崩溃/扭曲 | 取决于累计行为，呈现三种截然不同的收场 |

### 2.2 叙事支柱对应

| 支柱 | 叙事系统实现 |
|------|-------------|
| **罪恶的深度** | 误杀悲剧揭示系统、道德状态累积、叙事变体 |
| **致命的脆弱感** | 背景模块化系统、技能来源揭示、玩家的局限性 |
| **沉重、不洁的暴力** | 道德惩罚机制、LOS身份确认、后果层展示 |

---

## 3. Detailed Design

### 3.1 道德系统架构

道德系统采用"三位一体框架"，分为认知层、后果层、恢复层：

```
┌─────────────────────────────────────────────────────────┐
│                     认知层 (Cognition)                  │
│  LOS身份确认 + UI确认提示 → 玩家在击杀前知晓NPC身份      │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                     后果层 (Consequence)                 │
│  击杀行为 → 道德惩罚 + 悲剧揭示 → 道德画像累积           │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                     恢复层 (Redemption)                  │
│  埋葬/寻找家人 → 理智恢复 + 叙事解锁 → 人性重建          │
└─────────────────────────────────────────────────────────┘
```

**moral_standing 与 Sanity 的分离设计说明**：

`moral_standing`（道德立场值）与 `Sanity`（理智值）是**两个独立追踪的系统**，但共享相同的上游事件源：

| 维度 | moral_standing（叙事系统） | Sanity（理智/愤怒系统） |
|------|--------------------------|------------------------|
| **追踪目的** | 玩家对NPC的道德评判（善/恶） | 玩家心理承受压力（精神健康） |
| **变化来源** | 击杀/放过行为本身 | 目睹暴力/悲剧的心理冲击 |
| **恢复方式** | 救赎行为（埋葬、寻找家人） | 时间流逝、仁慈行为 |
| **影响对象** | 对话变体、叙事分支、结局 | 视觉效果、Game Over条件 |

**设计意图**：
- 击杀Enemy：moral_standing +5（奖励正义执行），Sanity -5（杀戮的心理代价）
- 击杀Victim：moral_standing -25（道德错误），Sanity -15（误杀的悲剧冲击）
- 两者数值方向可能相同也可能相反（如击杀Enemy：moral_standing上升但Sanity下降），这是**刻意设计**——玩家可能"正确地杀人"但仍承受心理压力

这种分离确保：
1. 玩家不会因为"道德正确"就完全不受心理惩罚（保持"致命脆弱感"）
2. 玩家不会因为"心理坚强"就忽视道德后果（保持"罪恶的深度"）

#### 3.1.1 道德状态追踪机制

**数据结构定义**：

```csharp
struct MoralProfile {
    // 核心计数
    int mercy_count;           // 仁慈次数（放过/捆绑NPC）
    int kill_count;             // 击杀总数
    int accidental_kill_count; // 误杀次数（击杀无辜者）

    // 分类计数
    int enemy_kill_count;       // 击杀恶徒
    int accomplice_kill_count;  // 击杀帮凶
    int victim_kill_count;      // 击杀无辜者

    // 道德值
    int moral_standing;         // 道德立场值 (-100 ~ +100)
                                 // 负值=残暴，正值=仁慈

    // 主导特质
    MoralTrait dominant_trait;  // DOMINANT_MERCY / DOMINANT_CRUELTY /
                                 // DOMINANT_CALCULATING / DOMINANT_CAUTIOUS

    // 悲剧解锁记录
    List<string> unlocked_tragedies;  // 已解锁的悲剧故事ID
    List<string> triggered_tragedies; // 已触发的悲剧故事ID
}

enum MoralTrait {
    DOMINANT_MERCY,      // 仁慈主导：mercy_count显著高于kill_count
    DOMINANT_CRUELTY,    // 残暴主导：kill_count >> mercy_count，且victim_kill > 0
    DOMINANT_CALCULATING,// 计算主导：kills高但accidental_kill极低
    DOMINANT_CAUTIOUS    // 谨慎主导：mercy_count接近kill_count，accidental_kill为0
}
```

**误杀发现机制（LOS系统扩展）**：

根据已确认决策，LOS系统增加"已确认身份"的UI提示：

| 场景 | UI反馈 | 道德惩罚说明 |
|------|--------|-------------|
| 击杀已Tag的Enemy（恶徒） | 红色击杀确认 | 正义执行，无惩罚 |
| 击杀已Tag的Accomplice（帮凶） | 黄色击杀警告 | 道德失误，中等惩罚 |
| 击杀已Tag的Victim（无辜者） | 绿色击杀警告 + "误杀！"弹出 | 严重错误 |
| 击杀Unknown（未知身份） | 灰色警告 + "身份未确认！"弹出 | 盲目暴力，最严重惩罚 + 悲剧揭示 |

> **注**：上表中 moral_standing 变化值与 Section 4.1 公式定义一致，为本系统内部的统一记录值。实际 moral_standing 计算以 Section 4.1 公式为准。

**UI提示设计**：

```
┌──────────────────────────────────────┐
│                                      │
│           [NPC头顶标签]               │
│              ● VICTIM               │
│              (绿色)                   │
│                                      │
│      ┌─────────────────────┐          │
│      │  ⚠ 身份未确认! ⚠   │          │
│      │  击杀将导致严重后果  │          │
│      └─────────────────────┘          │
│                                      │
│           [准星瞄准中]                 │
│                                      │
└──────────────────────────────────────┘
```

### 3.2 道德状态衰减规则（事件驱动）

**核心原则**：道德状态不随时间衰减，而是跟随事件累积。

这确保了玩家的道德画像反映**长期累计行为**而非近期行为，与已确认决策一致。

**主导特质计算触发机制**：

`dominant_trait` 在以下情况下重新计算（非定时）：
- 每当玩家执行影响道德的行为时（击杀/仁慈）
- 每当玩家解锁新模块时
- 在关键叙事节点判定前

> **注意**：`TraitRecalcInterval`（30秒）是性能优化参数，防止过于频繁的特质重算，而非道德衰减机制。

**事件类型与道德影响**：

| 事件 | moral_standing变化 | 说明 |
|------|-------------------|------|
| 击杀Enemy | +5 | 正义执行 |
| 击杀Accomplice | -10 | 知道对方是被迫 |
| 击杀Victim | -25 | 误杀无辜 |
| 击杀Unknown | -50 | 在未确认身份情况下击杀（严重惩罚，因为违反LOS系统的设计意图） |
| 捆绑NPC（不杀） | +3 | 仁慈选择 |
| 转化线人成功 | +8 | 正向干预 |
| 发现悲剧线索 | +2 | 理解对方背景 |
|埋葬受害者 | +10 | 给予尊严 |
| 找到家人 | +15 | 救赎行为 |

**主导特质计算**：

```
// dominant_trait 在行为触发时重新计算

if (kill_count == 0 AND mercy_count == 0):
    dominant_trait = DOMINANT_CAUTIOUS  # 零行为玩家：旁观者
elif (mercy_count > kill_count * 1.5):
    dominant_trait = DOMINANT_MERCY
elif (victim_kill_count > 0 AND kill_count > mercy_count * 2):
    dominant_trait = DOMINANT_CRUELTY
elif (kill_count > 10 AND victim_kill_count <= 2):
    dominant_trait = DOMINANT_CALCULATING
elif (mercy_count >= kill_count * 0.8 AND mercy_count <= kill_count * 1.2 AND victim_kill_count == 0):
    dominant_trait = DOMINANT_CAUTIOUS
else:
    dominant_trait = DOMINANT_MERCY  # 默认仁慈
```

**设计说明**：零行为玩家（从未击杀、从未仁慈）的 `dominant_trait` 设为 `DOMINANT_CAUTIOUS`，与边缘情况2的"旁观者"特殊结局设计一致。

### 3.3 状态上限设定

| 状态 | 硬上限 | 软上限 | 说明 |
|------|--------|--------|------|
| moral_standing | -100 ~ +100 | — | 超出部分截断 |
| mercy_count | 无硬上限 | kill_count * 3 | 统计用；软上限用于对话变体判定 |
| kill_count | 无上限 | — | 统计用 |
| accidental_kill_count | 无上限 | — | 触发特殊标记 |
| unlocked_tragedies | 最多20个 | — | 防止内存溢出 |

**mercy_count 软上限说明**：当 mercy_count > kill_count * 3 时，额外的仁慈行为不再提供道德奖励，但仍计入统计。这防止玩家通过大量捆绑行为刷道德值，同时保持系统的真实性。

> ⚠️ **Playtest 验证需求（Vertical Slice 阶段）**：`kill_count * 3` 的软上限比例需要通过 playtest 验证实际合理性。如果实际游玩中玩家普遍在达到软上限前就进入对话变体阶段，或软上限导致玩家无法触发某些叙事内容，需要调整比例。建议在 Vertical Slice 阶段收集以下数据：
> - 典型玩家的 kill_count 与 mercy_count 比例分布
> - mercy_count 软上限触发的频率
> - 软上限是否影响对话变体触发

**道德阈值里程碑**：

| 阈值 | 触发效果 |
|------|---------|
| moral_standing < -50 | NPC对话变体解锁：恐惧路线 |
| moral_standing > +50 | NPC对话变体解锁：希望路线 |
| victim_kill_count >= 3 | 特殊标记"杀手"，部分NPC会求饶或反抗 |
| accidental_kill_count >= 5 | 触发"连环误杀"特殊事件链 |

### 3.4 角色背景模块系统

**模块分类与数量上限**：

| 模块类型 | 代码标识 | 数量上限 | 前置条件 |
|---------|---------|---------|---------|
| 创伤记忆 | TRAUMA | 3个 | 无 |
| 技能来源 | SKILL_ORIGIN | 3个 | 背景选择后解锁 |
| 人际关系 | RELATIONSHIP | 3个 | 主线进度要求 |
| 隐藏真相 | HIDDEN_TRUTH | 3个 | 需满足特定道德条件 |

**总计**：每角色最多12个背景模块（4类 x 3个）

#### 3.4.1 模块解锁条件设计

**TRAUMA（创伤记忆）**：

| 模块ID | 模块名称 | 解锁条件 | 内容摘要 |
|--------|---------|---------|---------|
| TRAUMA_01 | 绑架当日 | 默认解锁 | 回忆女儿被绑的场景 |
| TRAUMA_02 | 警局绝望 | 主线到达Chapter 2 | 报警被敷衍的创伤 |
| TRAUMA_03 | 背叛的友人 | 发现线人背叛事件 | 曾信任的朋友出卖了你 |

**SKILL_ORIGIN（技能来源）**：

| 模块ID | 模块名称 | 解锁条件 | 内容摘要 |
|--------|---------|---------|---------|
| SKILL_01 | 前特工训练 | 选择Agent背景 | CIA审讯技巧的代价 |
| SKILL_02 | 街头的智慧 | 选择Civilian背景 | 自学成才的生存技能 |
| SKILL_03 | 雇佣兵往事 | 选择Mercenary背景 | 杀人赚钱的道德堕落 |

> ⚠️ **互斥说明**：SKILL_01/02/03 互斥解锁——三者是同一角色的三种背景出身路线。玩家只能选择一种背景，因此最多解锁其中1个模块（而非3个）。这是设计意图，确保技能来源与角色背景一致。

> ⚠️ **依赖说明**：SKILL 系列模块依赖 Character Background System 的 `BackgroundType` 接口。当前 `BackgroundType` 已在 `character-background.md` 中定义并传递给各系统，接口已就绪。模块解锁由背景系统通过 `INarrativeModuleQuery` 接口查询并触发解锁事件。

**RELATIONSHIP（人际关系）**：

| 模块ID | 模块名称 | 解锁条件 | 内容摘要 |
|--------|---------|---------|---------|
| REL_01 | 失踪的女儿 | 默认解锁 | 女儿小米的生活片段回忆 |
| REL_02 | 破碎的婚姻 | mercy_count >= 5 | 妻子离开的原因 |
| REL_03 | 线人的网络 | 主线到达Chapter 3 | 各路线人的背景故事 |

**HIDDEN_TRUTH（隐藏真相）**：

| 模块ID | 模块名称 | 解锁条件 | 内容摘要 |
|--------|---------|---------|---------|
| TRUTH_01 | 贩毒集团的保护伞 | moral_standing < -30 | 政府内部的腐败 |
| TRUTH_02 | 女儿的真正位置 | 收集到5个关键线索 | 小米被关押的真实地点 |
| TRUTH_03 | 主谋的身份 | 主线最终章 | 幕后黑手的完整背景 |

#### 3.4.2 模块解锁优先级

当多个模块同时满足解锁条件时，按以下规则决定解锁顺序：

1. **优先级顺序**：REL > TRAUMA > SKILL > TRUTH（按类别优先级）
2. **同类模块内**：按模块ID字母顺序排列
3. **解锁间隔**：每个模块解锁间隔0.5秒（给玩家反应时间）
4. **超过3个同时满足**：显示"多个故事待解锁"提示，玩家可手动查看

> **设计决策**：采用类别优先级而非随机或全部同时解锁，确保叙事节奏可控。同类内按字母顺序避免主观排序争议。

#### 3.4.3 非同一帧模块检查规则

当模块在不同时间点分别满足解锁条件时，解锁顺序遵循以下规则：

1. **每次模块检查都是独立的**：Narrative 系统在每次触发模块检查时（如玩家执行道德行为、解锁新模块、到达新 Chapter），会扫描所有未解锁模块
2. **非"同时满足"的处理**：如果两个模块在不同时间点满足条件（如 REL_02 在 mercy_count 达到5时解锁，TRAUMA_02 在 Chapter 2 到达时解锁），则按各自满足条件的时间顺序解锁，不强制类别优先级
3. **类别优先级仅适用于同一帧**：当多个模块在同一帧内同时满足条件时，才按 REL > TRAUMA > SKILL > TRUTH 的优先级排序

> **设计意图**：避免"同一帧同时满足"时 UI 混乱，同时保持非同时满足时的自然解锁节奏。

#### 3.4.4 模块前置条件矩阵

```
模块ID ─────┬──────────────┬─────────────┬──────────────┬─────────────
TRAUMA_01   │ 无            │ 无           │ 无            │ 无
TRAUMA_02   │ -             │ -            │ Chapter 2     │ -
TRAUMA_03   │ -             │ -            │ 发现背叛事件  │ -
SKILL_01    │ Agent背景     │ -            │ -             │ -
SKILL_02    │ Civilian背景  │ -            │ -             │ -
SKILL_03    │ Mercenary背景 │ -            │ -             │ -
REL_01      │ 无            │ 无           │ 无            │ 无
REL_02      │ mercy>=5      │ -            │ -             │ -
REL_03      │ -             │ -            │ Chapter 3     │ -
TRUTH_01    │ -             │ moral<-30    │ -             │ -
TRUTH_02    │ -             │ 5个关键线索   │ -             │ -
TRUTH_03    │ -             │ -            │ 最终章        │ -

注：SKILL 系列模块的前置条件"背景选择"通过 Character Background System 的 BackgroundType 接口实现，
    当玩家选择对应背景时自动触发模块解锁事件，无需额外条件判断。
```

#### 3.4.5 模块内容结构

每个背景模块包含以下结构：

```csharp
struct BackgroundModule {
    string module_id;           // 唯一标识
    string title;               // 显示标题
    string description;         // 简短描述（UI用）

    // 内容载体
    MediaType media_type;       // TEXT / AUDIO / IMAGE / VIDEO
    string content_path;        // 内容文件路径，格式约定：Assets/Narrative/Modules/{Category}/{ModuleID}.{ext}
                                 // 例：Assets/Narrative/Modules/TRAUMA/TRAUMA_01.asset
                                 // 具体路径格式由内容管线决定，确保与资源加载系统兼容

    // 解锁条件
    List<UnlockCondition> conditions;

    // 叙事效果
    List<NarrativeEffect> effects;  // 对话变体、剧情分支等
}

struct UnlockCondition {
    ConditionType type;         // FLAG / STAT / PROGRESS / BACKGROUND
    string key;                 // 条件键
    object value;               // 条件值
    ComparisonOperator op;     // >= / == / <= / >
}

struct NarrativeEffect {
    EffectType type;            // DIALOGUE_VARIANT / BRANCH_UNLOCK /
                                 // TRUTH_REVEAL / ACHIEVEMENT
    string target_id;           // 目标ID
    string modifier;            // 变体修饰符
}
```

### 3.5 主线叙事节点规划

#### 3.5.1 叙事结构：三幕式

```
┌────────────────────────────────────────────────────────────────┐
│                         第一幕：觉醒                            │
│                    Chapter 1-2 (约2-3小时)                      │
│                                                                │
│  核心冲突：玩家从普通人转变为"判决者"                           │
│  叙事目标：建立复仇动机，掌握核心玩法（LOS监听、道德判断）        │
│  关键节点：                                                     │
│    - 开局：创伤记忆激活，见到女儿最后影像                        │
│    - 第一个目标：追踪第一个恶徒，学习监听系统                    │
│    - 第一次误杀机会：发现目标可能是无辜者                        │
│    - Chapter 1结束：发现犯罪网络的存在                          │
│    - Chapter 2：进入更黑暗的领域，发现系统性的腐败               │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                         第二幕：挣扎                            │
│                    Chapter 3-4 (约3-4小时)                      │
│                                                                │
│  核心冲突：判决者的正当性与自我怀疑                              │
│  叙事目标：玩家开始看到每个NPC背后的故事，道德困惑加深            │
│  关键节点：                                                     │
│    - 发现悲剧揭示系统：每个恶徒都有自己的悲剧背景                  │
│    - 核心NPC"锈蚀的线人"：他是否值得信任？                       │
│    - 第一次重大抉择：杀/放走主要帮凶                             │
│    - Chapter 3结束：发现女儿的真正位置线索                       │
│    - Chapter 4：面对自己过去的创伤（背景模块展开）               │
│    - 第二次重大抉择：揭露腐败 vs 私下复仇                        │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                        第三幕：审判                            │
│                    Chapter 5-6 (约2-3小时)                      │
│                                                                │
│  核心冲突：最终救赎还是彻底堕落                                  │
│  叙事目标：玩家的累计行为决定结局走向                           │
│  关键节点：                                                     │
│    - 突入最终地点前的准备：检查道德状态                          │
│    - 面对主要反派：揭示其与玩家的隐藏联系                        │
│    - 第三次重大抉择：杀死/放过/救赎反派                         │
│    - 结局序列：根据dominant_trait和moral_standing呈现不同结局   │
└────────────────────────────────────────────────────────────────┘
```

#### 3.5.2 关键叙事节点列表

| 节点ID | 节点名称 | Chapter | 前置条件 | 道德变体 |
|--------|---------|---------|---------|---------|
| NODE_01 | 觉醒 | Ch1 | 无 | 无 |
| NODE_02 | 第一次追踪 | Ch1 | NODE_01 | 2种对话 |
| NODE_03 | 误杀机会 | Ch1 | NODE_02 | 3种结果 |
| NODE_04 | 网络浮现 | Ch2 | NODE_03 | 无 |
| NODE_05 | 锈蚀线人 | Ch2 | NODE_04 | 4种对话 |
| NODE_06 | 第一次放过 | Ch2 | mercy触发 | 2种后续 |
| NODE_07 | 悲剧揭示I | Ch3 | kill触发 | 3种揭示 |
| NODE_08 | 第二次追踪 | Ch3 | NODE_07 | 2种对话 |
| NODE_09 | 重大抉择I | Ch3 | NODE_08 | 3种分支 |
| NODE_10 | 悲剧揭示II | Ch4 | kill>=5 | 4种揭示 |
| NODE_11 | 过去与现在 | Ch4 | 特定模块 | 3种对话 |
| NODE_12 | 重大抉择II | Ch4 | NODE_11 | 3种分支 |
| NODE_13 | 最终地点 | Ch5 | NODE_12 | 无 |
| NODE_14 | 真相大白 | Ch5 | NODE_13 | 3种揭示 |
| NODE_15 | 最终审判 | Ch6 | NODE_14 | 6种结局 |

**六种结局类型**：

1. **救赎结局** — moral_standing > +50，dominant_trait = DOMINANT_MERCY

```
救赎结局：光明中的和解

小米被找到了。
你还活着，她也还活着。
在警察的保护下，你看着女儿被送上救护车的那一刻，
所有的愤怒、暴力、和黑暗，都在这一瞬间得到了救赎。

你放下了枪。
不是因为我无法再杀人——而是我不再需要了。

NPC们的命运：
- 恶徒被全部逮捕，主谋在逃亡中被击毙
- 帮凶们因配合作证而获得从轻处理
- 小米的母亲（如果还活着）收到了消息

玩家获得成就："救赎者"——你用仁慈战胜了黑暗。

道德评价：moral_standing > +50，dominant_trait = DOMINANT_MERCY
这是一个关于人性战胜绝望的故事。
```

2. **正义结局** — moral_standing > +30，dominant_trait = DOMINANT_CALCULATING

```
正义结局：精确的审判

小米被找到了。
你没有冲动，没有失控——每一个决定都经过冷静的计算。
你知道谁是真正的敌人，谁是被迫的帮凶，谁是无辜的受害者。
你的每一次击杀，都是精确的、必要的。

当最后一发子弹穿透主谋的心脏时，
你知道这不是复仇，这是正义。

NPC们的命运：
- 恶徒被精确处决，无误杀
- 帮凶们被捆绑并移交警方
- 受害者们得到解救

玩家获得成就："判官"——正义的天平从未倾斜。

道德评价：moral_standing > +30，dominant_trait = DOMINANT_CALCULATING
这是一个关于理性与正义的故事。
```

3. **毁灭结局** — moral_standing < -50，dominant_trait = DOMINANT_CRUELTY

```
毁灭结局：黑暗的吞噬

小米没有被找到。
或者说——当你终于找到她的时候，一切都已经太晚了。
你已经变成了你发誓要消灭的那类人。

你的双手沾满了鲜血——有罪的，也有无辜的。
愤怒吞噬了你，理智崩溃，暴力是唯一的语言。
当最后一个敌人倒下时，你发现自己的女儿正躲在角落里，
用恐惧的目光看着你——她认不出你了。

NPC们的命运：
- 恶徒全部死亡，但部分是无辜者
- 帮凶们在恐惧中死去
- 小米的母亲不知去向

玩家获得成就："毁灭者"——你成为了新的恶魔。

道德评价：moral_standing < -50，dominant_trait = DOMINANT_CRUELTY
这是一个关于暴力如何吞噬人性的故事。
```

4. **扭曲结局** — moral_standing < -30，dominant_trait = DOMINANT_CALCULATING

```
扭曲结局：新的判官

小米被找到了，但你不满足于此。
你发现这个犯罪网络背后有着更深的腐败——政府的保护伞，警方的内鬼。
你杀死了主谋，但你没有停下来。
你开始清除"所有"你认为有罪的人——不再需要审判，不需要证据。
你成了新的判官，一个没有任何限制的判官。

当你站在犯罪网络的废墟上，
你意识到：你已经成为你最初发誓要消灭的那种人——
只不过这一次，你认为自己是正义的。

NPC们的命运：
- 恶徒被清除（但标准已扭曲）
- 帮凶们全部消失
- 犯罪网络瓦解，但新的暴力秩序建立

玩家获得成就："扭曲者"——正义的假面下隐藏着新的黑暗。

道德评价：moral_standing < -30，dominant_trait = DOMINANT_CALCULATING
这是一个关于复仇如何腐蚀正义的故事。
```

5. **平静结局** — moral_standing 在 -30 ~ +30 之间，未触发任何极端特质

```
平静结局：灰色的余烬

小米被找到了。
你没有成为圣人，也没有成为恶魔。
你杀人了——但你知道为什么。
你放过了某些人——但不是因为软弱，而是因为选择。

当你最终放下武器，走出黑暗的时候，
你知道这只是暂时的平静。罪恶不会消失，
但你也没有被它吞噬。

在警局的审讯室里，你平静地交代了一切。
没有眼泪，没有笑容——只有沉默。

NPC们的命运：
- 恶徒被逮捕（部分因证据不足释放）
- 帮凶们获得保释
- 小米的下落公开，但未来不确定

玩家获得成就："幸存者"——你在黑暗中保持了平衡。

道德评价：moral_standing 在 -30 ~ +30 之间，dominant_trait = DOMINANT_CAUTIOUS
这是一个关于在极端环境中保持人性的故事。
```

6. **旁观者结局** — 零行为玩家专属

| 触发条件 | 详细说明 |
|---------|---------|
| kill_count = 0 | 玩家全程未击杀任何NPC |
| mercy_count = 0 | 玩家未执行任何仁慈行为（捆绑、转化等） |
| dominant_trait = DOMINANT_CAUTIOUS | 由零行为公式计算得出 |

**旁观者路线的叙事特征**：

| 阶段 | 玩家行为限制 | 叙事体验 |
|------|------------|---------|
| Chapter 1-2 | 可潜行完成所有任务，但不可击杀/捆绑任何NPC | 只能收集线索、监听对话，无法直接干预 |
| Chapter 3-4 | 关键NPC只能选择"放过"或"离开" | 玩家成为信息的旁观者而非裁决者 |
| Chapter 5-6 | 最终对峙只有两个选项："离开"或"报警" | 无法参与最终战斗 |

**结局呈现**：

```
旁观者结局：灰色的正义

玩家选择报警后，坐在警车里看着犯罪网络被瓦解。
没有血腥的复仇，没有亲手伸张的正义——
只有一通电话，和一个仍然失踪的女儿。

NPC们的命运：
- 恶徒被警方逮捕（部分因证据不足释放）
- 帮凶们作证换取减刑
- 受害者们得到安置
- 但小米的下落仍然成谜...

玩家获得成就："旁观者"——你选择了不成为审判者。

道德评价：moral_standing = 0（中立），但这不是"平静结局"——
而是一个关于无力感和选择代价的故事。
```

#### 3.5.3 节点前置条件设计

```csharp
struct NarrativeNode {
    string node_id;
    string title;

    // 前置条件（所有条件必须满足）
    List<NodeCondition> prerequisites;

    // 道德变体条件
    List<MoralVariant> variants;

    // 后续节点
    List<string> next_nodes;  // 根据条件选择
}

struct NodeCondition {
    ConditionType type;
    string key;
    object value;
    ComparisonOperator op;
}

// 条件类型
enum ConditionType {
    STORY_PROGRESS,     // 故事进度节点完成
    MORAL_STANDING,     // 道德值范围
    DOMINANT_TRAIT,     // 主导特质
    MODULE_UNLOCKED,    // 模块已解锁
    KILL_COUNT,         // 击杀数
    MERCY_COUNT,        // 仁慈数
    FLAG_SET,           // 特定flag
    NPC_RELATIONSHIP    // NPC关系值
}

// 道德变体示例：NODE_05 "锈蚀线人"
MoralVariant:
    - condition: dominant_trait == DOMINANT_MERCY
      dialog_tree: DIALOG_LINER_MERCY
      narrative_effect: +REL_02 unlocked
    - condition: dominant_trait == DOMINANT_CRUELTY
      dialog_tree: DIALOG_LINER_CRUEL
      narrative_effect: -5 sanity（玩家会听到线人的求饶）
    - condition: moral_standing < -30
      dialog_tree: DIALOG_LINER_CORRUPT
      narrative_effect: +TRUTH_01 unlocked
    - condition: default
      dialog_tree: DIALOG_LINER_NEUTRAL
```

### 3.6 叙事变体系统

根据已确认决策，NPC对话根据玩家dominant_trait切换版本。

#### 3.6.1 对话变体触发机制

```
┌─────────────────────────────────────────────────────┐
│              对话变体选择流程                        │
└─────────────────────────────────────────────────────┘

玩家触发与NPC对话
        │
        ▼
GrittyTakedowns 发送 ConfrontationStartRequest(npc_id)
        │
        ▼
NPC AI 系统加载 DialogueTreeConfig
        │
        ▼
NPC AI 系统调用 QueryDominantTrait() 获取 dominant_trait
        │              ┌─────────────────────────────────┐
        │              │  叙事系统提供 INarrativeMoralQuery  │
        │              │  接口实现：                          │
        │              │    - QueryDominantTrait() → MoralTrait │
        │              │    - QueryMoralStanding() → int      │
        │              │    - QueryHasTrait(trait) → bool      │
        │              └─────────────────────────────────┘
        ▼
NPC AI 系统根据 dominant_trait 预筛选分支
        │
        ▼
返回包含变体信息的 DialogueTreeConfig
        │
        ▼
GrittyTakedowns 渲染对话 UI（按 `variant_id` 过滤并显示可用分支）
        │
        │  注意：GrittyTakedowns 仅负责 UI 渲染和输入处理，
        │  不持有 DialogueTree 数据——数据由 NPC AI 系统拥有和管理
        ▼
对话结束 → GrittyTakedowns 发送 `DialogueChoice` 到 NPC AI 系统处理
```

**与 DialogTree 接口的集成说明**：

| 步骤 | 执行者 | 动作 |
|------|-------|------|
| 1 | GrittyTakedowns | 发送 `ConfrontationStartRequest(npc_id)` 到 NPC AI 系统 |
| 2 | NPC AI 系统 | 调用 `NarrativeSystem.QueryDominantTrait()` 获取玩家主导特质 |
| 3 | NPC AI 系统 | 根据 dominant_trait 选择对应分支，填充 `DialogueTreeConfig.variant_id` |
| 4 | NPC AI 系统 | 返回 `DialogueTreeConfig` 到 GrittyTakedowns |
| 5 | GrittyTakedowns | 渲染对话 UI：**仅显示**当前 `variant_id` 对应的对话分支；玩家选择后发送 `DialogueChoice` 到 NPC AI 系统处理 |

**实现约束**：
- NPC AI 系统不直接访问叙事系统的内部状态，仅通过 `INarrativeMoralQuery` 接口查询
- 对话变体选择在 NPC AI 系统侧完成（而非 GrittyTakedowns 侧），确保 DialogTree 数据封装
- GrittyTakedowns 仅持有 `DialogueTreeConfig` 的临时副本用于渲染，**不持久化存储**对话树数据

#### 3.6.2 四种对话变体类型

| 变体类型 | 触发条件 | 叙事效果 | 示例 |
|---------|---------|---------|------|
| MERCY | dominant_trait == DOMINANT_MERCY | NPC表现出更多信任，愿意分享更多信息 | "你看起来不像他们...我相信你" |
| CRUEL | dominant_trait == DOMINANT_CRUELTY | NPC表现出恐惧，可能会试图逃跑或反击 | "求求你别杀我！我还有家人！" |
| CALCULATING | dominant_trait == DOMINANT_CALCULATING | NPC会更加防备，但会尝试做交易 | "你想要什么？钱？我可以给你情报" |
| CAUTIOUS | dominant_trait == DOMINANT_CAUTIOUS | NPC会试探性地合作，但仍保持警惕 | "你为什么不动手？我们做个交易吧" |

**UI/音效差异化说明**：
- **MERCY 变体**：对话框边框使用冷色调（蓝/灰），NPC立绘表情平和；音效为轻柔的环境音
- **CRUEL 变体**：对话框边框使用红色调，NPC立绘表情恐惧/愤怒；音效包含低频威胁音
- **CALCULATING 变体**：对话框边框使用中性灰色，NPC立绘表情冷静/算计；音效包含金属质感UI音
- **CAUTIOUS 变体**：对话框边框使用黄色警告色调，NPC立绘表情紧张/试探；音效包含悬疑环境音
- 变体切换有过渡动画（0.2秒边框颜色渐变），切换时播放对应变体的差异化音效提示（由 Audio Director 配置）

#### 3.6.3 追踪数据与UI展示

```csharp
// HUD显示的道德追踪数据
struct MoralTrackingUI {
    int total_kills;           // 击杀总数
    int mercy_acts;            // 仁慈行为数
    int accidental_kills;      // 误杀数
    MoralTrait current_trait;  // 当前主导特质
    int moral_standing;        // 道德立场值

    // UI显示优先级：
    // 1. 如果accidental_kills > 0，显示误杀警告
    // 2. 如果有新的tragedy unlocked，显示"故事解锁"提示
    // 3. 如果dominant_trait变化，显示特质变化通知
}
```

---

## 4. Formulas

### 4.1 道德立场值计算

```
moral_standing_new = Clamp(moral_standing_old + EventDelta, -100, 100)
```

| 事件 | EventDelta | 影响的系统 | 公式说明 |
|------|------------|-----------|---------|
| 击杀Enemy | +5 | moral_standing | 正义执行 |
| 击杀Accomplice | -10 | moral_standing | 道德失误 |
| 击杀Victim | -25 | moral_standing + victim_kill_count | 严重错误 |
| 击杀Unknown | -50 | moral_standing + 悲剧解锁 | 盲目暴力（在未确认身份情况下击杀，严重违反LOS系统设计意图） |
| 捆绑NPC | +3 | moral_standing + mercy_count | 仁慈选择 |
| 转化线人 | +8 | moral_standing | 正向干预 |
| 发现悲剧线索 | +2 | moral_standing | 共情理解（仅首次发现有效） |
| 埋葬受害者 | +10 | moral_standing + sanity恢复 | 给予尊严 |
| 找到家人 | +15 | moral_standing + sanity恢复 | 救赎行为 |

### 4.2 悲剧揭示计算（后果层核心公式）

> **⚠️ 参数语义澄清**：本节的 `+2`、`-5/-3/-2`、`+8` 均为 Sanity/Rage 系统接口参数，**不影响 moral_standing 计算**。其中：
> - `+2` — moral_standing 正向奖励（主动发现悲剧线索）
> - `-5/-3/-2` — Sanity 心理惩罚（目睹悲剧的代价）
> - `+8` — Sanity 惩罚抵消值（误杀解锁悲剧时分摊惩罚）

**数值用途对照表（消除混淆）**：

| 数值 | 使用位置 | 作用于 | 含义 | 触发场景 |
|------|---------|-------|------|---------|
| `+2` | Section 4.1 表格 | moral_standing | 主动发现悲剧线索的**道德奖励** | 玩家通过监听/审问/搜身等主动行为发现NPC的悲剧背景 |
| `-5/-3/-2` | Sanity系统公式 | Sanity | 目睹悲剧的**理智惩罚** | 发现悲剧线索时，Sanity/Rage系统计算的惩罚值（Narrative系统传递给Sanity系统） |
| `+8` | 本节公式 | Sanity（惩罚抵消） | 误杀解锁悲剧时的**惩罚抵消值** | 误杀后解锁悲剧故事，将部分惩罚转化为故事意义（但仍保留部分惩罚效果） |

> **设计意图区分**：
> - `+2` 是**正向激励**：鼓励玩家主动了解NPC的背景，促进共情
> - `-5/-3/-2` 是**心理代价**：每次发现悲剧都会对玩家心理造成冲击
> - `+8` 是**救赎转化**：将负面行为（误杀）部分转化为故事解锁，保留惩罚同时给予"意义"

**三者不矛盾**：+2 发生于主动探索时，-5/-3/-2 是发现悲剧的心理代价，+8 是误杀后的惩罚分摊机制。

**误杀惩罚转化公式**：

```
SanityPenalty = BaseSanityPenalty - TragedyUnlockValue
MoralPenalty = BaseMoralPenalty - StoryRevelationBonus
```

示例（击杀无辜者）：
```
BaseSanityPenalty = -15
TragedyUnlockValue = +8 (解锁了一个悲剧故事)
StoryRevelationBonus = +7 (故事揭示的正面价值)

FinalSanityPenalty = -15 + 8 = -7
FinalMoralPenalty = -25 + 7 = -18
```

**设计意图**：将部分惩罚转化为故事解锁，让负面行为也有"意义"，但仍保持惩罚效果。

### 4.3 理智恢复计算

```
SanityRecovery = BaseRecovery * RedemptionMultiplier * ModuleBonus
```

| 行为 | BaseRecovery | RedemptionMultiplier | ModuleBonus（乘数） |
|------|-------------|----------------------|-------------------|
| 埋葬受害者 | +10 | 1.0 | 1.2（如解锁REL_02） |
| 寻找家人 | +15 | 1.5 | 1.33（如解锁TRUTH_02） |
| 仁慈捆绑 | +3 | 1.0 | 1.0（无加成） |
| 发现悲剧真相 | +5 | 1.2 | 1.0（无加成，仅首次） |

**示例**：
```
埋葬受害者（无REL_02）：10 * 1.0 * 1.0 = +10
埋葬受害者（有REL_02）：10 * 1.0 * 1.2 = +12
寻找家人（有TRUTH_02）：15 * 1.5 * 1.33 = +30
```

### 4.4 主导特质判定公式

```
dominant_trait = CalculateDominantTrait(mercy_count, kill_count, victim_kill_count)

function CalculateDominantTrait(m, k, v):  # m=mercy_count, k=kill_count, v=victim_kill_count
    if k == 0 AND m == 0:
        return DOMINANT_CAUTIOUS  # 零行为玩家
    else if m > k * 1.5:
        return DOMINANT_MERCY
    else if v > 0 AND k > m * 2:
        return DOMINANT_CRUELTY
    else if k > 10 AND v <= 2:
        return DOMINANT_CALCULATING
    else if m >= k * 0.8 AND m <= k * 1.2 AND v == 0:
        return DOMINANT_CAUTIOUS
    else:
        return DOMINANT_MERCY  # Default
```

**变量名统一说明**：
- `victim_kill_count`（即 `v`）：击杀无辜者的次数，与数据结构中的 `MoralProfile.victim_kill_count` 一致
- `accidental_kill_count`：与 `victim_kill_count` 为同一概念，用于标记"误杀"场景（如 LOS 身份确认后的惩罚计算）
- 判定公式统一使用 `victim_kill_count`，与 Section 3.1 数据结构中的字段名保持一致

### 4.5 mercy_count 软上限公式

```
effective_mercy_count = min(mercy_count, kill_count * 3)
```

**公式说明**：
- 当 `mercy_count <= kill_count * 3` 时：全部仁慈行为计入 moral_standing 计算
- 当 `mercy_count > kill_count * 3` 时：超出部分的仁慈行为**不提供 moral_standing 奖励**，但仍计入 mercy_count 统计（用于 dominant_trait 判定）
- 设计意图：防止玩家通过大量捆绑行为刷道德值，同时保持系统真实性

### 4.6 模块解锁判定

```
is_unlocked = ALL(condition.check() for condition in module.conditions)
```

---

## 5. Edge Cases

### 5.1 道德系统边缘情况

**边缘情况1：连续误杀导致理智崩溃**

问题：玩家在短时间内连续误杀多名无辜NPC。

处理：
- 首次误杀：正常惩罚 + 警告UI
- 第二次误杀：惩罚加倍 + 屏幕剧烈闪红 + "你失去了什么？"内心独白
- 第三次误杀：理智强制降至20以下 + 触发"杀手"标记 + 部分NPC会主动攻击玩家（不再求饶）
- 第四次误杀：理智系统进入"灵魂分裂"特殊状态

**边缘情况2：完全不做道德选择的玩家**

问题：玩家全程只潜行不杀，也不触发任何mercy事件。

处理：
- moral_standing维持初始值0
- dominant_trait为DOMINANT_CAUTIOUS（零行为玩家，公式第一分支）
- 某些需要"判决"触发的节点无法解锁
- 结局导向"旁观者"特殊结局（不同于其他所有结局）

> **旁观者路线与模块解锁互斥说明**：选择旁观者路线（零行为）的玩家将无法满足依赖行为计数的模块解锁条件，包括 REL_02（需 mercy_count >= 5）、边缘情况3的 DOMINANT_CALCULATING 路线（需 kill_count > 10）。这是设计意图——不同的游玩风格对应不同的叙事体验，旁观者路线本身就是一种完整且有意义的选择路径。

**旁观者路线边界条件补充**：
- 玩家有 mercy_count 但 kill_count = 0：按 dominant_trait 判定走对应路线（如 mercy_count >= 5 则 REL_02 解锁，同时 dominant_trait 可能变为 DOMINANT_MERCY），**不触发旁观者结局**
- 玩家发现悲剧线索、收集线索但从未击杀或仁慈：仍视为"零行为"，触发旁观者路线
- 旁观者路线的判断优先级：**最高**——只要 kill_count = 0 AND mercy_count = 0，即使用 moral_standing 很高或很低，也走旁观者路线

**边缘情况3：杀戮数远超仁慈数但无误杀**

问题：玩家杀了很多恶徒和帮凶，但没有误杀无辜者。

处理：
- moral_standing可能为负（因为击杀Accomplice也有惩罚）
- dominant_trait判定为DOMINANT_CALCULATING
- NPC对话变体切换为"交易"路线
- 解锁隐藏真相TRUTH_01（系统腐败主题）

**边缘情况4：道德值达到上限**

问题：moral_standing = +100 或 -100。

处理：
- 硬上限截断
- UI显示"道德标杆"特殊标记
- 触发隐藏成就：+100解锁"圣人"，-100解锁"恶魔"
- 某些NPC会有特殊反应（如看到"+100"道德值的玩家会完全信任或完全恐惧）

### 5.2 背景模块系统边缘情况

**边缘情况5：满足多个模块解锁条件**

问题：同一时刻玩家满足了3个以上模块的解锁条件。

处理：
- 按模块ID字母顺序依次解锁（避免UI混乱）
- 每个模块解锁间隔0.5秒（给玩家反应时间）
- 如果超过3个同时满足，显示"多个故事待解锁"提示

**边缘情况6：模块内容缺失**

问题：玩家解锁了模块，但对应内容文件不存在。

处理：
- 显示通用回退内容："这段记忆模糊不清..."
- 记录错误日志
- 不阻塞游戏进程

### 5.3 叙事节点边缘情况

**边缘情况7：无法满足任何后续节点条件**

问题：玩家行为导致无法满足任何后续节点的前置条件。

处理：
- 设计"死路"节点，显示"你的选择让你无处可去..."
- 提供有限的重玩选项（回到最近的检查点）
- 不设计完全死锁的情况

**边缘情况8：道德变体条件重叠**

问题：玩家同时满足多个变体条件。

处理：
- 按优先级选择：DOMINANT_CRUELTY > DOMINANT_MERCY > DOMINANT_CALCULATING > DOMINANT_CAUTIOUS
- CRUEL优先级最高，确保最极端的行为有最明显的后果

---

## 6. Dependencies

### 6.1 系统依赖关系矩阵

```
                    ┌──────────────────────────────────────────────┐
                    │                                              │
                    ▼                                              │
┌─────────────────┐  │  ┌────────────────────────────────────────┐  │
│ LOS System      │  │  │      Narrative System                  │  │
│ - NPC Identity  │──┼─▶│  - Moral Profile                       │  │
│ - Tagging UI    │  │  │  - Background Modules                   │  │
└─────────────────┘  │  │  - Narrative Nodes                     │  │
                    │  └────────────────────────────────────────┘  │
                    │                    │                         │
                    │                    ▼                         │
                    │  ┌────────────────────────────────────────┐  │
                    │  │                                        │  │
                    │  ▼                                        │  │
┌─────────────────┐  │  ┌────────────────────────────────────────┐  │
│ GrittyTakedowns │─▶│  │   Sanity/Rage System (理智/愤怒系统)    │  │
│ - KillTagEvent  │  │  │  - KillTagEvent → 理智惩罚（惩罚路径）   │  │
└─────────────────┘  │  │  - SanityRecoveryEvent → 理智恢复（救赎） │  │
                    │  └────────────────────────────────────────┘  │
                    │                                              │
                    └──────────────────────────────────────────────┘
```

### 6.2 上游依赖（系统依赖谁）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **LOS System** | 硬依赖 | 接收 `KeywordCapturedEvent` 确认NPC身份；发送 `NPCIdentityConfirmedEvent` |
| **GrittyTakedowns** | 硬依赖 | 接收 `KillTagEvent{kill_tag: NPCIdentityType}` 获取击杀类型；触发道德计算 |
| **Clue&Journal** | 硬依赖 | 接收 `ClueDiscoveredEvent` 获取线索发现；触发悲剧揭示 |
| **Character Background** | 硬依赖 | 查询背景选择；提供模块内容；影响对话变体 |

### 6.3 下游依赖（谁依赖本系统）

| 系统 | 依赖类型 | 接口说明 |
|------|---------|---------|
| **Sanity/Rage System** | 硬依赖 | 发送 `SanityRecoveryEvent` 理智恢复事件（埋葬受害者、找到家人等救赎行为）；**不直接发送惩罚事件**（惩罚通过 GrittyTakedowns → KillTagEvent 路径） |
| **Dialog Tree System** | 硬依赖 | 通过 `QueryDominantTrait` 接口提供 dominant_trait 查询；发送对话触发事件 |
| **Screen Effects** | 软依赖 | 接收心理状态变化；提供极端状态视觉效果 |
| **Achievement System** | 软依赖 | 接收道德里程碑达成通知 |

### 6.4 事件流定义

**本系统发送的事件数据结构**：

```csharp
// 理智恢复事件（救赎行为触发）
struct SanityRecoveryEvent {
    RecoveryType type;      // BURY_VICTIM / FIND_FAMILY / MERCY_ACTS / TRAGEDY_UNLOCK
    int base_recovery;      // 基础恢复值（由本系统根据公式计算）
    float module_multiplier;// 模块加成（如有 REL_02 / TRUTH_02）
    int final_recovery;     // 最终恢复值 = base_recovery * module_multiplier
}

// 道德后果事件（击杀行为后由本系统记录，不直接发送给 Sanity）
struct MoralConsequenceEvent {
    KillTag kill_tag;       // ENEMY / ACCOMPLICE / VICTIM / UNKNOWN
    int moral_standing_delta;  // 道德立场变化值
    bool triggered_tragedy_unlock; // 是否触发悲剧解锁
}

// 叙事节点解锁事件
struct NarrativeNodeUnlockedEvent {
    string node_id;
    MoralTrait dominant_trait_at_unlock; // 解锁时的主导特质
}

// 对话变体查询接口（DialogTree 系统调用）
interface INarrativeMoralQuery {
    MoralTrait QueryDominantTrait();
    int QueryMoralStanding();
    bool QueryHasTrait(MoralTrait trait);
}
```

**dominant_trait 重新计算触发机制**：

dominant_trait 的重新计算是由 **Narrative 系统内部订阅自身事件后自动触发**，不需要外部系统显式调用。具体触发点如下：

| 触发时机 | 调用方法 | 说明 |
|---------|---------|------|
| 击杀事件处理完成 | `RecalculateDominantTrait()` | 在击杀处理流程（流程2）中，moral_standing 更新后自动调用 |
| 仁慈行为注册完成 | `RecalculateDominantTrait()` | 在仁慈行为流程（流程3）中，mercy_count 更新后自动调用 |
| 救赎行为完成 | `RecalculateDominantTrait()` | 在救赎行为流程（流程4）中，moral_standing 更新后自动调用 |
| 叙事节点判定前 | `RecalculateDominantTrait()` | 在叙事节点触发流程（流程6）检查前置条件前调用 |

> **实现说明**：Narrative 系统内部维护一个事件订阅列表，当订阅的事件（如 `KillTagEvent`、`TieUpEvent`）被触发时，Narrative 系统自动执行对应的处理函数并重新计算 dominant_trait。外部系统（GrittyTakedowns、LOS System 等）只需发送事件，不需要显式调用 `RecalculateDominantTrait()`。

> **性能优化**：`TraitRecalcInterval`（默认30秒）作为节流阈值，防止在短时间内多次触发重算时产生性能开销。当距离上次重算时间小于 `TraitRecalcInterval` 时，跳过本次重算。

**关键事件流**：

```
1. LOS身份确认流程：
   LOS.TagComplete → NPCIdentityConfirmedEvent → Narrative.MarkNPCIdentified()
   → [更新 kill_count / mercy_count 统计]

2. 击杀处理流程：
   GrittyTakedowns.OnKill → KillTagEvent{npc_id, kill_tag} → Narrative.CalculateMoralPenalty()
   → [更新 moral_standing]
   → [自动触发 RecalculateDominantTrait()]
   → [触发 TragedyUnlockCheck()]
   → [如触发悲剧解锁，发送 SanityRecoveryEvent{type: TRAGEDY_UNLOCK, final_recovery: +8}]
   注：Sanity/Rage 系统接收 GrittyTakedowns 的 KillTagEvent 计算理智惩罚，
      本系统仅在悲剧解锁时发送额外的理智恢复事件

3. 仁慈行为流程：
   GrittyTakedowns.OnTieUp → TieUpEvent → Narrative.RegisterMercy()
   → [更新 mercy_count / moral_standing]
   → [自动触发 RecalculateDominantTrait()]
   → [计算 final_recovery = BaseRecovery * RedemptionMultiplier * ModuleBonus]
   → [发送 SanityRecoveryEvent{type: MERCY_ACTS, final_recovery} 到 Sanity/Rage System]

4. 救赎行为流程（埋葬/寻找家人）：
   玩家执行埋葬 → Narrative.RegisterBury()
   → [mortal_standing +10，解锁 REL_02 条件检查]
   → [自动触发 RecalculateDominantTrait()]
   → [发送 SanityRecoveryEvent{type: BURY_VICTIM, base_recovery: 10, module_multiplier: 1.2, final_recovery: 12}]

   玩家找到家人 → Narrative.RegisterFindFamily()
   → [mortal_standing +15，解锁 TRUTH_02 条件检查]
   → [自动触发 RecalculateDominantTrait()]
   → [发送 SanityRecoveryEvent{type: FIND_FAMILY, base_recovery: 15, module_multiplier: 1.33, final_recovery: 20}]

5. 模块解锁流程：
   Narrative.CheckModuleUnlock() → [条件满足]
   → [发送 ModuleUnlockedEvent 到 UI 系统显示]
   → [更新 next_nodes 可达性]

6. 叙事节点触发流程：
   Narrative.AdvanceStory() → [自动触发 RecalculateDominantTrait()]
   → [检查前置条件]
   → [NodeConditionMet]
   → [GrittyTakedowns 向 DialogTree 系统发送 ConfrontationStartRequest]
   → [DialogTree 系统调用 QueryDominantTrait() 获取当前 dominant_trait]
   → [根据 dominant_trait 选择对应 dialog_tree_id]
   → [DialogTreeSystem.LoadDialog(variant_id)]
```

**dominant_trait 传递给 DialogTree 的机制**：

DialogTree 系统在发起对话时，需要知道玩家当前的 dominant_trait 以选择正确的变体。由于 DialogTree 归 NPC AI 系统所有（dialog-tree-interface.md），本系统提供查询接口：

```
DialogTree 系统 ← QueryDominantTrait() ← 叙事系统
                            ↓
                     返回 MoralTrait 枚举值
                            ↓
                     DialogTree 系统根据 trait 选择变体分支
```

具体流程：
1. 玩家触发对峙 → GrittyTakedowns 发送 `ConfrontationStartRequest(npc_id)` 到 NPC AI 系统
2. NPC AI 系统加载对应 `DialogueTreeConfig`
3. NPC AI 系统调用 `QueryDominantTrait()` 获取玩家主导特质
4. NPC AI 系统根据 dominant_trait 预筛选对话分支（标记可选分支）
5. 将包含变体信息的 `DialogueTreeConfig` 返回给 GrittyTakedowns
6. Gritty Takedowns 渲染对话 UI 时，仅显示当前特质可用的分支

---

## 7. Tuning Knobs

### 7.1 道德系统参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MoralStandingMin` | int | -100 | — | 道德值下限 |
| `MoralStandingMax` | int | +100 | — | 道德值上限 |
| `KillEnemyBonus` | int | +5 | +3 ~ +10 | 击杀恶徒奖励 |
| `KillAccomplicePenalty` | int | -10 | -15 ~ -5 | 击杀帮凶惩罚 |
| `KillVictimPenalty` | int | -25 | -35 ~ -15 | 击杀无辜者惩罚 |
| `KillUnknownPenalty` | int | -50 | -60 ~ -35 | 击杀未知身份惩罚（⚠️ 高风险参数，详见调参风险提示） |
| `MercyBonus` | int | +3 | +2 ~ +5 | 仁慈行为奖励 |
| `LineConversionBonus` | int | +8 | +5 ~ +12 | 转化线人奖励 |
| `TragedyClueBonus` | int | +2 | +1 ~ +5 | 发现悲剧奖励 |
| `BuryVictimBonus` | int | +10 | +5 ~ +15 | 埋葬受害者奖励 |
| `FindFamilyBonus` | int | +15 | +10 ~ +25 | 找到家人奖励 |
| `TraitRecalcInterval` | float | 30.0 | 15.0 ~ 60.0 | 主导特质重算节流间隔（秒）。**注意**：特质重算是事件驱动（非定时），此参数作为性能优化节流阈值，防止过于频繁的重算计算 |

### 7.2 悲剧揭示系统参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `TragedyUnlockMax` | int | 20 | 10 ~ 30 | 最多解锁悲剧数 |
| `TragedyUnlockValue` | int | +8 | +5 ~ +12 | 悲剧解锁抵消惩罚值 |
| `StoryRevelationBonus` | int | +7 | +3 ~ +10 | 故事揭示正面价值 |
| `FirstTragedySanity` | int | -15 | -20 ~ -10 | 首次悲剧揭示理智惩罚 |
| `SubsequentTragedySanity` | int | -5 | -8 ~ -3 | 后续悲剧理智惩罚（递减） |

### 7.3 背景模块系统参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MaxModulesPerType` | int | 3 | 2 ~ 5 | 每类模块最大数量 |
| `MaxTotalModules` | int | 12 | 8 ~ 20 | 模块总数上限 |
| `ModuleUnlockDelay` | float | 0.5 | 0.2 ~ 1.0 | 多模块同时解锁间隔（秒） |
| `MaxUnlockedTragedies` | int | 20 | 15 ~ 30 | 悲剧解锁上限 |

### 7.4 叙事变体系统参数

| 参数名 | 类型 | 默认值 | 安全范围 | 说明 |
|--------|------|--------|---------|------|
| `MercyThresholdRatio` | float | 1.5 | 1.2 ~ 2.0 | mercy > kill * ratio → MERCY特质 |
| `CrueltyAccidentalMin` | int | 1 | 1 ~ 3 | 误杀次数阈值（触发CRUEL） |
| `CalculatingKillMin` | int | 10 | 5 ~ 20 | 击杀次数阈值（触发CALCULATING） |
| `CalculatingAccidentalMax` | int | 2 | 1 ~ 5 | 误杀上限（CALCULATING条件） |
| `CautiousBalanceRatio` | float | 0.2 | 0.1 ~ 0.3 | mercy/kill平衡范围 |
| `CautiousAccidentalMax` | int | 0 | 0 ~ 2 | 误杀上限（CAUTIOUS条件） |

### 7.5 调参风险提示

| 参数 | 风险等级 | 风险 | 影响 |
|------|---------|------|------|
| `KillUnknownPenalty` 设置过低 | ⚠️ **高风险** | 玩家倾向于不确认身份就击杀，破坏LOS系统意义 | 削弱"认知层"设计 |
| `KillUnknownPenalty` 设置过高 | ⚠️ **高风险** | 玩家因害怕惩罚而不敢行动，节奏拖沓 | 影响游戏流畅度 |
| `KillVictimPenalty` 设置过高 | 中等 | 玩家因害怕误杀而不敢行动，节奏拖沓 | 影响游戏流畅度 |
| `MercyThresholdRatio` 设置过高 | 低 | 很难触发MERCY特质，对话变体单一 | 减少叙事多样性 |
| `MaxModulesPerType` 设置过低 | 低 | 可用背景故事太少，角色深度不足 | 影响角色塑造 |
| `TraitRecalcInterval` 过短 | 低 | 特质频繁变化，叙事体验不稳定 | 影响对话连贯性 |

> **⚠️ KillUnknownPenalty 特别说明**：此参数为**高风险调参项**，建议在 Vertical Slice 阶段专项测试。-60 的极端惩罚可能导致玩家在紧张战斗中不敢行动；-35 的低惩罚则可能让玩家忽视 LOS 系统设计意图。建议在 playtest 时收集以下数据：
> - 玩家在战斗中的身份确认行为频率
> - 因担心惩罚而放弃击杀的比例
> - 惩罚对玩家决策的实际影响

---

## 8. Acceptance Criteria

### 8.1 道德系统功能验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-1 | 击杀已确认身份的Enemy，moral_standing +5，无误杀计数 | 执行击杀，验证数值变化 |
| AC-2 | 击杀已确认身份的Accomplice，moral_standing -10 | 执行击杀，验证数值变化 |
| AC-3 | 击杀已确认身份的Victim，moral_standing -25，accidental_kill_count +1 | 执行击杀，验证数值变化 |
| AC-4 | 击杀Unknown身份，moral_standing -50，解锁一个悲剧故事 | 执行击杀，验证惩罚和故事解锁 |
| AC-4b | 在-50惩罚下，玩家后续战斗中的身份确认行为频率显著提升（惩罚合理性验证） | playtest数据收集：对比-50 vs -35惩罚下的身份确认率差异 |
| AC-5 | 捆绑NPC不杀，moral_standing +3，mercy_count +1 | 执行捆绑，验证数值变化 |
| AC-6 | moral_standing达到+100时，显示"道德标杆"特殊标记 | 积累道德值至上限，验证UI |
| AC-7 | moral_standing达到-100时，显示"道德深渊"特殊标记 | 降低道德值至下限，验证UI |
| AC-8 | mercy_count > kill_count * 1.5时，dominant_trait变为MERCY | 执行多次仁慈行为，验证特质变化 |
| AC-8b | mercy_count软上限在 kill_count < 10 时不被触发，在 kill_count > 10 时正确生效 | 记录 kill_count 和 mercy_count，验证 effective_mercy_count 计算结果 |

### 8.2 LOS身份确认UI验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-9 | 击杀Unknown身份NPC，屏幕显示"身份未确认！"警告 | 执行击杀，验证UI弹出 |
| AC-10 | 击杀Victim身份NPC，屏幕显示绿色"误杀！"警告 | 执行击杀，验证UI颜色和文字 |
| AC-11 | 在LOS监听过程中，NPC头顶标签正确显示身份颜色 | 执行监听，验证标签颜色变化 |
| AC-12 | 完成身份Tagging时，播放确认音效 | 完成监听，验证音效播放 |

### 8.3 悲剧揭示系统验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-13 | 首次发现悲剧线索，理智-15（完整惩罚） | 发现第一条悲剧线索，验证sanity变化 |
| AC-14 | 后续悲剧线索发现，理智-5（递减惩罚） | 发现第二条悲剧线索，验证sanity变化 |
| AC-15 | 误杀解锁悲剧故事时，惩罚分摊（-15变成-7+故事） | 误杀后验证sanity和unlocked_tragedies |
| AC-16 | 已解锁悲剧数量达到20时，不再解锁新悲剧 | 达到上限后误杀，验证无新解锁 |

### 8.4 理智恢复系统验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-17 | 埋葬受害者，sanity +10（+12如果有REL_02） | 执行埋葬，验证sanity变化 |
| AC-18 | 找到家人，sanity +15（+20如果有TRUTH_02） | 完成家人寻找，验证sanity变化 |
| AC-19 | 理智恢复不超过100（硬上限） | 多次恢复，验证上限截断 |
| AC-20 | moral_standing因道德行为变为正值，触发"希望"对话变体 | 积累道德值，触发对话验证变体 |

### 8.5 背景模块系统验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-21 | TRAUMA_01在游戏开始时自动解锁 | 新游戏开始，验证模块解锁状态 |
| AC-22 | 选择Agent背景，SKILL_01自动解锁 | 选择背景，验证模块解锁状态 |
| AC-23 | mercy_count >= 5时，REL_02解锁 | 执行5次仁慈行为，验证模块解锁 |
| AC-24 | moral_standing < -30时，TRUTH_01解锁 | 降低道德值，验证模块解锁 |
| AC-25 | 每类模块最多解锁3个（共12个上限） | 尝试解锁超过限制，验证无解锁 |

### 8.6 叙事节点验收

| ID | 验收条件 | 测试方法 |
|----|---------|---------|
| AC-26 | 完成前置节点后，后续节点自动解锁 | 验证节点状态变化 |
| AC-27 | DOMINANT_MERCY特质触发MERCY对话变体 | 积累仁慈行为，触发对话验证变体 |
| AC-28 | DOMINANT_CRUELTY特质触发CRUEL对话变体 | 积累杀戮行为，触发对话验证变体 |
| AC-29 | moral_standing < -50触发恐惧路线 | 降低道德值，验证路线变化 |
| AC-30 | moral_standing > +50触发希望路线 | 积累道德值，验证路线变化 |

### 8.7 跨系统集成验收

| ID | 验收条件 | 测试方法 | 测试脚本 |
|----|---------|---------|---------|
| AC-31 | GrittyTakedowns的KillTagEvent正确触发道德计算 | 监听事件总线，验证数值计算 | `tests/integration/narrative/test_kill_tag_event_integration.cs` (planned) |
| AC-32 | Sanity/Rage系统接收正确的惩罚/奖励值 | 验证sanity变化与道德系统输出一致 | `tests/integration/narrative/test_sanity_recovery_event_integration.cs` (planned) |
| AC-32b | SanityRecoveryEvent.final_recovery与Section 4.3公式计算结果一致 | 输入不同救赎行为，验证final_recovery = BaseRecovery × RedemptionMultiplier × ModuleBonus | `tests/integration/narrative/test_sanity_recovery_calculation.cs` (planned) |
| AC-33 | Dialog Tree系统正确加载对应的变体 | 触发对话，验证加载的variant_id与QueryDominantTrait()返回值一致 | `tests/integration/narrative/test_dialog_variant_selection.cs` (planned) |
| AC-34 | Achievement系统接收道德里程碑通知 | 达成里程碑，验证成就解锁 | `tests/integration/narrative/test_moral_milestone_achievement.cs` (planned) |

---

## 9. Open Questions

| # | 问题 | 状态 | 负责人 | 说明 |
|---|------|------|--------|------|
| OQ-1 | **悲剧故事内容的写作进度** | 进行中 | Writer | 需要 Writer 确认每条悲剧故事的详细脚本。**预计交付：Vertical Slice 前（见下方时间线）** |
| OQ-2 | **对话变体的数量预算** | **已确认** | Creative Director | 保持4种变体（MERCY/CRUEL/CALCULATING/CAUTIOUS），覆盖不同玩家行为模式，Vertical Slice阶段验证 |
| OQ-3 | **模块内容的媒介比例** | **待定（紧急）** | Art Director | TEXT/AUDIO/IMAGE各占多少？是否需要过场动画？**必须在 Vertical Slice 开始前确认，否则影响制作管线** |
| OQ-4 | **核心叙事线锁定条件的最终确认** | 待定 | Creative Director | moral_standing阈值是否合适？是否需要加入时间限制？ |
| OQ-5 | **五种结局的具体描述** | 待定 | Narrative Director | 需要在后续迭代中细化每种结局的触发条件和呈现方式 |

### OQ-1 补充：12个背景模块内容框架（占位符）

> **内容交付时间线**：
> - **MVP 阶段**：TRAUMA_01、REL_01（默认解锁）需要完整内容
> - **Vertical Slice 阶段**：TRAUMA_02、REL_02、SKILL 系列模块需要完整内容
> - **Alpha 阶段**：所有 12 个模块内容必须就绪
>
> ⚠️ **重要**：模块内容验收需在对应阶段开始前完成，否则无法进行对应的系统集成测试。

> 以下为模块内容占位符，实际内容由 Writer 团队填充。

| 模块ID | 模块名称 | 内容媒介 | 内容摘要（占位） |
|--------|---------|---------|----------------|
| TRAUMA_01 | 绑架当日 | VIDEO + AUDIO | 女儿被绑的创伤画面（待填充） |
| TRAUMA_02 | 警局绝望 | AUDIO + IMAGE | 报警被敷衍的创伤（待填充） |
| TRAUMA_03 | 背叛的友人 | TEXT + AUDIO | 曾信任的朋友出卖了你（待填充） |
| SKILL_01 | 前特工训练 | VIDEO | CIA审讯技巧的代价（待填充） |
| SKILL_02 | 街头的智慧 | TEXT | 自学成才的生存技能（待填充） |
| SKILL_03 | 雇佣兵往事 | VIDEO | 杀人赚钱的道德堕落（待填充） |
| REL_01 | 失踪的女儿 | VIDEO + AUDIO | 女儿小米的生活片段回忆（待填充） |
| REL_02 | 破碎的婚姻 | AUDIO + TEXT | 妻子离开的原因（待填充） |
| REL_03 | 线人的网络 | TEXT + IMAGE | 各路线人的背景故事（待填充） |
| TRUTH_01 | 贩毒集团的保护伞 | TEXT + IMAGE | 政府内部的腐败（待填充） |
| TRUTH_02 | 女儿的真正位置 | VIDEO | 小米被关押的真实地点（待填充） |
| TRUTH_03 | 主谋的身份 | VIDEO | 幕后黑手的完整背景（待填充） |

**媒介比例建议（待 Art Director 确认）**：
- TEXT: 4个模块（REL_03, TRUTH_01, SKILL_02, TRAUMA_03）
- AUDIO: 2个模块（REL_02, TRAUMA_02）
- VIDEO: 4个模块（TRAUMA_01, SKILL_01, REL_01, TRUTH_02, TRUTH_03）
- IMAGE: 2个模块（REL_03, TRUTH_01）
- 实际比例可能在 Vertical Slice 阶段调整

---

## Change Log

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-10 | 0.1 | 初稿创建，集成三位一体道德框架、模块系统、叙事变体 | Narrative Director |
| 2026-04-10 | 0.2 | 设计审查修复：修复dominant_trait零行为玩家边缘情况、添加mortal_standing与Sanity分离设计说明、明确KillTagEvent类型定义、补充OQ-1模块内容框架占位符、更新Open Questions状态 | Claude Code |
| 2026-04-10 | 0.3 | 修复审查问题：澄清dominant_trait触发机制（事件驱动非定时）、clarify悲剧揭示数值区分（+2主动发现 vs +8误杀解锁）、解决OQ-2（确认4种对话变体）、添加mercy_count软上限、补充旁观者特殊结局设计、明确模块解锁优先级规则、修正3.4节段落编号 | Claude Code |
| 2026-04-10 | 0.4 | **P0修复**：定义 SanityRecoveryEvent 事件数据结构，澄清理智恢复事件流程（Narrative系统计算final_recovery值，Sanity系统直接应用）；移除未定义的SanityPenaltyEvent（惩罚通过KillTagEvent路径） |
| 2026-04-10 | 0.4 | **P1修复**：添加SKILL系列模块依赖说明（技能系统当前为Planned占位符）；补充dominant_trait传递给DialogTree的完整机制（QueryDominantTrait接口+DialogTree集成流程） |
| 2026-04-10 | 0.4 | **P2修复**：OQ-1添加内容交付时间线；OQ-3标注为紧急（必须在Vertical Slice前确认）；补充完整事件数据结构定义（INarrativeMoralQuery接口） |
| 2026-04-10 | 0.5 | **P1修复**：SKILL系列模块依赖从Skill System改为Character Background System（Skill System已删除）；添加variant_id字段到DialogTreeConfig（解决DialogTree接口不一致）；补充旁观者特殊结局详细设计 |
| 2026-04-10 | 0.5 | **P2修复**：Section 4.2添加数值用途对照表消除+2/-5/+8混淆；KillUnknownPenalty从-35提升至-50并调整安全范围；添加mercy_count软上限playtest验证说明；AC-31~AC-34添加测试脚本引用 |
| 2026-04-10 | 0.6 | **设计审查修复**：P0-旁观者路线与模块解锁互斥说明；P1-Section 3.1表格添加moral_standing变化值列统一术语；P1-添加AC-32b验证SanityRecoveryEvent.final_recovery计算正确性；P2-测试脚本标注(planned)；P2-KillUnknownPenalty标注为高风险参数并补充playtest验证说明 |
| 2026-04-10 | 0.7 | **设计审查后修复**：P1-SKILL模块互斥性注释（玩家三选一背景路线）；P1-补充五种标准结局详细叙事描述（~60行）；P1-明确DialogTree渲染职责（GrittyTakedowns仅渲染过滤，不持有数据） |
| 2026-04-10 | 0.8 | **设计审查修复**：P1-补充3.4.3节（非同一帧模块检查规则）；P1-旁观者结局重整为第6种独立结局与五种标准结局并列；P1-补充dominant_trait重新计算触发机制和调用路径说明；P2-简化Section 3.1表格（移除moral_standing变化值重复定义，引用Section 4.1）；P2-模块解锁优先级规则补充"非同一帧处理"说明 |
| 2026-04-10 | 0.9 | **设计审查后修复**：P0-统一dominant_trait判定变量（victim_kill_count vs accidental_kill_count）；P1-Section 4.3 ModuleBonus改为乘数语义并补充示例；P1-Section 7.5补充KillUnknownPenalty过高的风险描述；P2-AC-4b添加惩罚合理性验证；P2-补充对话变体UI/音效差异化说明 |
| 2026-04-10 | 1.0 | **设计审查修复后再次审查修复**：P1-统一Section 3.1与Section 4.1的moral_standing变化值引用关系；P1-修复dominant_trait判定公式函数签名与变量名不一致（accidental_kill_count→victim_kill_count）；P1-添加AC-8b验收mercy_count软上限；P2-补充旁观者路线边界条件说明；P2-Section 4.1表格添加"影响的系统"列；P2-补充BackgroundModule.content_path路径格式约定 |
