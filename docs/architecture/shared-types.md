# 共享类型定义 (Shared Types)

> **版本**: 2.1.0
> **创建日期**: 2026-04-10
> **状态**: APPROVED
> **维护者**: 架构师
> **更新日期**: 2026-04-12 (v2.1.0 — 修复 InputDeviceType 枚举值；修复 ADR-0018/0019/0020 跨 ADR 一致性问题)

---

## 1. 概述

本文档统一定义跨 ADR 共享的事件类型和枚举，避免重复定义导致的类型不一致问题。

**使用规则**：
- 所有 ADR 必须引用本文档中的类型定义，不得在自己的文档中重复定义
- 如需新增共享类型，应先更新本文档，再由各 ADR 引用
- 共享类型应遵循单一职责原则，类型名称应准确反映其业务含义

---

## 2. 伤害相关类型

### 2.1 DamageType

伤害类型枚举，定义所有可能的伤害分类：

```csharp
// 伤害类型
public enum DamageType
{
    LETHAL,  // 致命伤害（可立即致死）
    BLUNT   // 钝击伤害（导致硬直/倒地）
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/DamageType.cs`

**使用系统**：Weapon System、Gritty Takedowns、Health System

---

### 2.2 HitLocation

命中部位枚举，定义伤害命中的身体区域：

```csharp
// 命中部位
public enum HitLocation
{
    HEAD,   // 头部（致命区域）
    TORSO,  // 躯干（可被护甲保护）
    LIMBS   // 四肢（减伤区域）
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/HitLocation.cs`

**使用系统**：Weapon System、Gritty Takedowns、Health System

---

### 2.3 DamageRequest

伤害请求结构，由伤害发起方（Weapon System、Gritty Takedowns）发送到 Health System 执行伤害计算：

```csharp
// 伤害请求
public struct DamageRequest
{
    public int target_id;           // 目标实体 ID
    public DamageType damage_type;  // 伤害类型（LETHAL/BLUNT）
    public float damage_amount;     // 实际伤害值（环境物件=1.0，热武器来自武器配置）
    public int penetration;        // 武器穿透等级（穿透判定：penetration >= armorLevel 时穿透）
    public HitLocation hit_location; // 命中部位（HEAD/TORSO/LIMBS）
    public int source_entity_id;    // 伤害来源实体 ID（用于死亡追踪）
    public string source;           // 伤害来源标识（weapon_id 或 "bare_hands"）
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/DamageRequest.cs`

**发布者**：Weapon System、Gritty Takedowns
**订阅者**：Health System

**穿透判定规则**：
- `penetration >= armorLevel`：穿透成功，无视护甲直接致死
- `penetration < armorLevel`：穿透失败，护甲吸收伤害

---

### 2.4 ExplosionEvent

爆炸事件，由 Weapon System 发送，Health System（ExplosionHandler）接收并执行伤害计算：

```csharp
// 爆炸事件
public struct ExplosionEvent
{
    public Vector3 position;         // 爆炸中心位置
    public float radius;           // 爆炸半径（米）
    public float lethal_ratio;      // 致死半径比例（默认 0.3，即 30% 半径内致死）
    public float base_damage;       // 爆炸物基础伤害（C4=150，手榴弹=80）
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/ExplosionEvent.cs`

**发布者**：Weapon System
**订阅者**：Health System（ExplosionHandler）

**伤害计算规则**（由 Health System 执行）：
- `distance <= radius * lethal_ratio`：Lethal 伤害（全额 base_damage）
- `distance > radius * lethal_ratio`：Blunt 伤害（按距离衰减 * stagger_multiplier(1.5)）

---

## 3. 武器相关类型

### 3.1 WeaponCategory

武器类别枚举：

```csharp
public enum WeaponCategory
{
    Environmental,  // 环境物件（砖块、消防栓、电线等）
    HotWeapon       // 热武器（枪械、爆炸物）
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/WeaponCategory.cs`

---

### 3.1.1 RangeType

武器射程类型枚举：

```csharp
public enum RangeType
{
    melee,    // 近战（必须贴身）
    short,    // 短距离（~5m）
    medium,   // 中距离（~15m）
    long,     // 远距离（~30m）
    throwable // 可投掷
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/RangeType.cs`

---

### 3.1.2 AmmoType

弹药类型枚举：

```csharp
public enum AmmoType
{
    Pistol,
    Rifle,
    Shotgun,
    Explosive
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/AmmoType.cs`

---

### 3.1.3 DetonationType

引爆类型枚举：

```csharp
public enum DetonationType
{
    Instant,  // 即时引爆
    Timed,    // 延时引爆
    Remote    // 遥控引爆
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/DetonationType.cs`

---

### 3.2 ObjectCategory

可武器化的环境物件类别枚举，与 Environment Interaction 系统共享：

```csharp
public enum ObjectCategory
{
    // 可破坏环境物件
    Brick,
    MetalPipe,
    FireExtinguisher,
    GlassBulb,
    Wire,
    BrokenGlass,
    // 爆炸类物件（可远程触发/连锁引爆）
    GasolineCan,    // 汽油桶
    PropaneTank,    // 丙烷罐
    Grenade,        // 手榴弹
    C4,             // C4 炸弹
    // 热武器
    Pistol,
    Rifle,
    Shotgun,
    // 通用
    Unknown
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/ObjectCategory.cs`

**说明**：
- 此枚举由 Environment Interaction 系统定义，Weapon System 通过 `ObjectStateChangedEvent` 接收状态变化
- `GetTemplateByCategory()` 方法用于根据物件类别获取对应的 WeaponData

---

### 3.3 WeaponState

武器状态枚举：

```csharp
public enum WeaponState
{
    Stored,      // 在背包/枪套中
    Equipped,    // 正在手持
    Holstered,   // 收起在枪套
    Used,        // 开火/爆炸中
    Empty,       // 弹药耗尽
    Reloading    // 换弹中
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/WeaponState.cs`

---

### 3.4 WeaponStateChangedEvent

武器状态变化事件：

```csharp
public struct WeaponStateChangedEvent
{
    public string weapon_id;
    public WeaponState old_state;
    public WeaponState new_state;
}
```

**定义位置**：`Assets/Game/Features/WeaponSystem/Events/WeaponEvents.cs`

**发布者**：Weapon System
**订阅者**：UI System

---

### 3.5 ObjectState / ObjectStateChangedEvent

环境物件状态枚举和状态变化事件：

```csharp
// 环境物件状态枚举
public enum ObjectState
{
    Spawned,    // 生成（初始状态）
    Available,  // 可拾取
    Held,       // 被玩家持有
    Used,       // 已使用（一次性物件）
    Depleted,   // 资源耗尽
    Dropped     // 被丢弃
}

// 环境物件状态变化事件
public struct ObjectStateChangedEvent
{
    public string object_id;
    public ObjectCategory object_category;
    public ObjectState new_state;
    public Vector3 position;
}
```

**定义位置**：
- `ObjectState`：`Assets/Game/Foundation/Shared/Types/ObjectState.cs`
- `ObjectStateChangedEvent`：`Assets/Game/Features/WeaponSystem/Events/WeaponEvents.cs`

**发布者**：Environment Interaction System
**订阅者**：Weapon System

---

### 3.6 WeaponAwareness

NPC 感知到武器的事件：

```csharp
public struct WeaponAwareness
{
    public string weapon_id;
    public Vector3 position;
    public WeaponCategory weapon_type;
}
```

**定义位置**：`Assets/Game/Features/WeaponSystem/Events/WeaponEvents.cs`

**发布者**：Weapon System
**订阅者**：NPC AI System

---

### 3.7 WeaponUsedEvent

武器使用事件：

```csharp
public struct WeaponUsedEvent
{
    public string weapon_id;
    public string usage_type;  // "fire", "throw", "melee", "explosion"
}
```

**定义位置**：`Assets/Game/Features/WeaponSystem/Events/WeaponEvents.cs`

**发布者**：Weapon System
**订阅者**：Audio System

---

### 3.8 WeaponQueryRequest / WeaponQueryResponse

武器数据查询接口，由 Gritty Takedowns 系统查询 Weapon System 获取武器信息：

```csharp
// 武器查询请求
public struct WeaponQueryRequest
{
    public string weapon_id;
}

// 武器查询响应
public struct WeaponQueryResponse
{
    public string weapon_id;
    public WeaponData data;  // 包含 damage_component.damage_type, damage_component.penetration, animation_tags.component.tags
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/WeaponQuery.cs`

**发布者**：Gritty Takedowns
**订阅者**：Weapon System

**使用场景**：环境处决时，Gritty Takedowns 通过此接口查询 WeaponData 获取动画标签

---

## 4. 动作锁定相关类型

### 4.1 ActionLockType

动作锁定类型枚举，支持多锁并发：

```csharp
[Flags]
public enum ActionLockType
{
    None = 0,
    Interaction = 1 << 0,    // 交互锁定（Gritty Takedowns）
    Dialogue = 1 << 1,       // 对话锁定（Dialogue System）
    Cutscene = 1 << 2,       // 过场动画锁定（Cinematic System）
    Stagger = 1 << 3,        // 硬直锁定（Health System）
    All = Interaction | Dialogue | Cutscene | Stagger
}
```

**定义位置**：`Assets/Game/Foundation/Shared/ActionLock/ActionLockType.cs`

---

### 4.2 ActionLockSystem

动作锁定系统，管理玩家控制权接管，支持多锁并发和超时自动释放：

```csharp
public class ActionLockSystem
{
    public static ActionLockSystem Instance { get; private set; }

    // 当前活跃锁列表
    private List<ActiveLock> _activeLocks = new();

    /// <summary>
    /// 锁状态变化事件
    /// 发布时机：有任何锁被获取或释放时
    /// </summary>
    public event Action<bool> OnLockChanged;

    /// <summary>
    /// 请求获取动作锁定
    ///
    /// 锁仲裁规则（按优先级从高到低）：
    /// 1. Cutscene > Dialogue > Interaction > Stagger（高优先级锁会阻塞低优先级请求）
    /// 2. 同一请求者再次 Acquire 同类型锁时，更新 Timeout（不重复添加）
    /// 3. 不同请求者的锁类型无重叠时，可并发持有
    /// 4. 同类型锁被不同请求者持有时，后者请求失败（返回 false）
    /// </summary>
    public bool AcquireLock(string requester, ActionLockType lockType, float expectedDuration)
    {
        if (string.IsNullOrEmpty(requester))
            return false;

        // 同一请求者再次请求同类型锁：更新 Timeout
        var existingLock = _activeLocks.FirstOrDefault(l =>
            l.Requester == requester && l.LockType == lockType);
        if (existingLock != null)
        {
            existingLock.Timeout = expectedDuration + 1f;
            return true;
        }

        // 检查类型冲突（不同请求者持有相同类型的锁）
        var conflictingLock = _activeLocks.FirstOrDefault(l =>
            l.LockType.HasFlag(lockType) && l.Requester != requester);
        if (conflictingLock != null)
            return false;

        var wasEmpty = _activeLocks.Count == 0;

        // 添加新锁
        _activeLocks.Add(new ActiveLock
        {
            Requester = requester,
            LockType = lockType,
            AcquireTime = Time.time,
            Timeout = expectedDuration + 1f
        });

        PlayerController.Instance.SetInputEnabled(false);

        // 通知锁状态变化
        if (wasEmpty != (_activeLocks.Count == 0))
            OnLockChanged?.Invoke(_activeLocks.Count > 0);

        return true;
    }

    /// <summary>
    /// 释放动作锁定
    /// </summary>
    public bool ReleaseLock(string requester)
    {
        var lockToRelease = _activeLocks.FirstOrDefault(l => l.Requester == requester);
        if (lockToRelease == null)
            return false;

        var wasEmpty = _activeLocks.Count == 0;

        _activeLocks.Remove(lockToRelease);

        if (_activeLocks.Count == 0)
            PlayerController.Instance.SetInputEnabled(true);

        // 通知锁状态变化
        if (wasEmpty != (_activeLocks.Count == 0))
            OnLockChanged?.Invoke(_activeLocks.Count > 0);

        return true;
    }

    /// <summary>
    /// 紧急释放（玩家死亡等情况）
    /// </summary>
    public void ForceReleaseAll()
    {
        _activeLocks.Clear();
        PlayerController.Instance.SetInputEnabled(true);
        OnLockChanged?.Invoke(false);
    }

    /// <summary>
    /// 检查指定请求者是否持有锁
    /// </summary>
    public bool HasLock(string requester) => _activeLocks.Any(l => l.Requester == requester);

    /// <summary>
    /// 获取当前活跃锁的数量（调试用）
    /// </summary>
    public int GetActiveLockCount() => _activeLocks.Count;

    /// <summary>
    /// 每帧更新（检查超时）
    /// </summary>
    public void Update()
    {
        var now = Time.time;
        var expiredLocks = _activeLocks.Where(l => now - l.AcquireTime > l.Timeout).ToList();

        foreach (var expired in expiredLocks)
        {
            Debug.LogWarning($"[ActionLock] Lock owned by {expired.Requester} timed out. Force releasing.");
            _activeLocks.Remove(expired);
        }

        if (_activeLocks.Count == 0)
            PlayerController.Instance.SetInputEnabled(true);

        // 检查是否有锁因超时被移除，需要通知
        if (expiredLocks.Count > 0)
            OnLockChanged?.Invoke(_activeLocks.Count > 0);
    }

    private class ActiveLock
    {
        public string Requester;
        public ActionLockType LockType;
        public float AcquireTime;
        public float Timeout;
    }
}
```

**定义位置**：`Assets/Game/Foundation/Shared/ActionLock/ActionLockSystem.cs`

**锁仲裁优先级**：
| 优先级 | 锁类型 | 说明 |
|--------|--------|------|
| 1（最高） | Cutscene | 过场动画锁定，其他所有请求都会被阻塞 |
| 2 | Dialogue | 对话锁定，阻塞 Interaction 和 Stagger |
| 3 | Interaction | 交互锁定（如 Gritty Takedowns），阻塞 Stagger |
| 4（最低） | Stagger | 硬直锁定 |

**并发规则**：
- 不同请求者可以同时持有不同类型的锁（如 Dialogue + Interaction 可共存，因为它们是不同的 ActionLockType）
- 同一请求者可以持有多个不同类型的锁
- 同类型锁不能被不同请求者同时持有

**超时自动释放**：每个锁有 Timeout 保护（expectedDuration + 1f），防止死锁。

---

## 5. NPC 状态相关类型

### 5.1 WorldState

NPC 世界状态枚举：

```csharp
public enum WorldState
{
    FREE,        // 自由状态（正常行为）
    UNCONSCIOUS, // 无意识（昏迷/被击倒）
    TIED,        // 捆绑状态
    DEAD         // 死亡
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/WorldState.cs`

**使用系统**：NPC AI System、Gritty Takedowns

---

### 5.2 NPCStateChangedEvent

NPC 状态变化事件：

```csharp
public struct NPCStateChangedEvent
{
    public int npc_id;
    public EntityType entity_type;  // NPC（固定值）
    public WorldState old_state;
    public WorldState new_state;
    public DamageType damage_type;  // LETHAL / BLUNT / NONE
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/NPCStateChangedEvent.cs`

**发布者**：Health System
**订阅者**：NPC AI System、Gritty Takedowns

---

### 5.3 AlertState

NPC 警戒状态枚举：

```csharp
public enum AlertState
{
    UNDETECTED,  // 未被察觉
    SUSPECT,     // 可疑（玩家暴露但未确认）
    SEARCH,      // 搜索中
    ALERT,       // 已警戒
    ESCAPE,      // 逃离（受伤后撤退）
    COMBAT       // 战斗状态
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/AlertState.cs`

---

### 5.4 AlertStateChangedEvent

NPC 警戒状态变化事件：

```csharp
public struct AlertStateChangedEvent
{
    public int npc_id;
    public AlertState old_state;
    public AlertState new_state;
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/AlertStateChangedEvent.cs`

**发布者**：NPC AI System
**订阅者**：Gritty Takedowns

---

### 5.5 NPCIdentityType

NPC 身份标签类型：

```csharp
public enum NPCIdentityType
{
    UNKNOWN,     // 未确认
    ENEMY,       // 恶徒（可处决）
    ACCOMPLICE,  // 帮凶
    VICTIM       // 受害者（不可攻击）
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/NPCIdentityType.cs`

**使用系统**：LOS System（身份标签管理）、Gritty Takedowns（交互选项判断）

---

### 5.6 NPCSizeCategory

NPC体型分类，用于 Gritty Takedowns 系统的捆绑时间计算：

```csharp
public enum NPCSizeCategory
{
    Small,   // 体型小，乘数 0.0f，range=[3,4]
    Medium,  // 体型中等，乘数 0.25f，range=[3,5]
    Large    // 体型大，乘数 1.0f，range=[3,6]
}
```

**定义位置**：`Assets/Game/Core/NPCAI/NPCSizeCategory.cs`

**计算公式**（见 ADR-0011 §TieUpCalculator）：
```
duration = baseTime * (1.0 + npcSizeMultiplier - playerSkillBonus)
duration = Clamp(duration, minDuration=3s, maxDuration=6s)
```

**说明**：
- 此类型由 NPC AI System 定义（Core Layer），Gritty Takedowns 通过 NPCController.QuerySizeCategory() 获取
- **⚠️ 注意**：之前的版本将此值描述为"直接加成时间"，这是错误的。实际是乘数，用于参与公式计算
- 体型乘数与 ADR-0011 §TieUpCalculator 和 §TieUpDurationProvider 保持一致

**使用系统**：Gritty Takedowns（捆绑时间计算）、NPC AI（动画选择）

---

### 5.7 Faction / FactionAllegiance

派系枚举和派系忠诚度：

```csharp
public enum Faction
{
    凋亡议会,
    锈网,
    灰烬团,
    无声者,
    苍白之手,
    中立
}

/// <summary>
/// NPC 对特定派系的忠诚度（影响派系感知共享和群体 AI 行为）
/// 注意：这是派系关系属性，不同于 NPC 个人属性 Bravery
/// </summary>
public struct FactionAllegiance
{
    public Faction faction;
    public int loyalty;  // 范围 [0, 100]，影响 NPC 对派系指令的服从度
}
```

**定义位置**：`Assets/Game/Core/NPCAI/Faction/FactionRelations.cs`

**相关属性**：
- **Bravery（勇气）**：NPC 个人属性，定义于 NPCData.cs，影响抵抗威胁/贿赂的能力，范围 **[1, 10]**，值越小越容易被吓唬/贿赂
- **DeceptionResistance（欺骗抗性）**：NPC 个人属性，定义于 NPCData.cs，影响欺骗成功的概率，范围 [0, 1]，值越高越难被欺骗（>= 1.0 时完全免疫欺骗）
- **Allegiance（忠诚度）**：派系关系属性，定义于 FactionRelations.cs，影响派系感知共享，范围 [0, 100]

> **注意**：
> - Bravery 和 DeceptionResistance 是 NPC **个人属性**（存储在 NPCData.cs），与派系忠诚度 FactionAllegiance.faction/loyalty 是不同概念，请勿混淆。
> - **贿赂判定**：NPC Bravery <= bribeBraveryThreshold 时可被贿赂（阈值由 GrittyTakedownsTuningSO 配置，默认 3）
> - **欺骗免疫**：当 DeceptionResistance >= 1.0 时，NPC 完全免疫欺骗（见 ADR-0011 §DeceptionChecker）

---

### 5.8 MissingReasonType

线索缺失原因类型（定义于 ADR-0016）：

```csharp
/// <summary>
/// 线索缺失原因类型
/// </summary>
public enum MissingReasonType
{
    NONE,           // 正常状态，无缺失
    NPC_DEAD,        // NPC 已死亡，线索随其知识一同消失
    PLAYER_CHOICE,  // 玩家主动放弃（如拒绝任务）
    ZONE_LOCKED,    // 区域未解锁
    QUEST_FAILED     // 关联任务失败
}
```

**定义位置**：`Assets/Game/Features/ClueJournal/Types/MissingReasonType.cs`

**使用系统**：Clue & Journal System（ADR-0016）

---

### 5.9 AlertTrigger

警戒触发原因枚举：

```csharp
public enum AlertTrigger
{
    PERCEPTION,   // 感知触发（视觉/听觉/记忆）
    FACTION,      // 玩家主动行为
    SHARED,       // 派系感知共享
    EXECUTION     // 处决被目击
}
```

**定义位置**：`Assets/Game/Core/NPCAI/AlertFSM/AlertTrigger.cs`

---

## 6. Health System 类型

### 6.1 HealthState

生命状态枚举：

```csharp
public enum HealthState
{
    HEALTHY,    // 健康
    STAGGERED,  // 硬直（受到 BLUNT 伤害第一次倒地，可被处决）
    DOWNED,     // 倒地/重伤（再次受到 BLUNT 伤害，或审问/捆绑后）
    DEAD        // 死亡
}
```

**定义位置**：`Assets/Game/Foundation/Health/HealthState.cs`

**使用系统**：Health System、NPC AI System（检测 NPC 死亡）

---

### 6.1.1 WorldState ↔ HealthState 同步协议

> **⚠️ 重要**：NPC 状态涉及两套独立的状态机：
> - **WorldState**（NPC AI System 管理）：FREE / UNCONSCIOUS / TIED / DEAD
> - **HealthState**（Health System 管理）：HEALTHY / STAGGERED / DOWNED / DEAD
>
> NPCController 负责在两系统间协调同步，详见 ADR-0004 和 ADR-0011 §4。

**同步规则**：

| Health System 事件 | WorldState 变化 | HealthState 变化 | 说明 |
|-------------------|-----------------|------------------|------|
| 受到 BLUNT 伤害且 STAGGERED | 保持 FREE | STAGGERED | 第一次倒地硬直 |
| 再次受到 BLUNT 伤害（STAGGERED 状态下）| → UNCONSCIOUS | DOWNED | 倒地可捆绑 |
| 受到 LETHAL 伤害 | → DEAD | DEAD | 立即死亡 |
| 审问/捆绑完成 | → TIED | DOWNED（保持）| 捆绑状态 |
| TieUpRelease | → FREE | DOWNED → HEALTHY（苏醒动画后）| 解绑恢复 |
| WakeUp（自然唤醒）| → FREE | DOWNED → HEALTHY | 醒来 |

**关键约束**：
- WorldState.UNCONSCIOUS 仅在 HealthState.DOWNED 时可由 Gritty Takedowns 捆绑
- NPCController 负责在 HealthSystem 发送 HealthStateChangedEvent（ADR-0008 §6.5）后更新 WorldState

---

### 6.2 ArmorState

护甲状态枚举：

```csharp
public enum ArmorState
{
    ACTIVE,
    DESTROYED
}
```

**定义位置**：`Assets/Game/Foundation/Health/ArmorSystem.cs`

---

### 6.3 ArmorDestroyedEvent

护甲破坏事件：

```csharp
public struct ArmorDestroyedEvent
{
    public int npc_id;
    public Vector3 position;  // 护甲破坏位置（用于视觉效果放置）
}
```

**定义位置**：`Assets/Game/Foundation/Health/Events/ArmorEvents.cs`

**发布者**：Health System（ArmorSystem）
**订阅者**：视觉效果系统

---

### 6.4 ExplosionAlertEvent

爆炸警报事件（爆炸造成 NPC 死亡时广播）：

```csharp
public struct ExplosionAlertEvent
{
    public Vector3 position;
    public float radius;
    public int victim_id;
    public bool killer_is_player;
}
```

**定义位置**：`Assets/Game/Foundation/Health/Events/HealthEvents.cs`

---

## 7. 玩家相关类型

### 7.1 IPlayer 接口

```csharp
/// <summary>
/// 玩家标识接口
/// 用于需要检测玩家对象的组件（如 AreaLightingDetector）
/// 实现者：PlayerController 或其子组件
/// </summary>
public interface IPlayer
{
    string PlayerId { get; }
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/IPlayer.cs`

**使用者**：AreaLightingDetector（ADR-0022 区域光照系统）

---

### 7.2 PlayerMovementState

玩家移动状态枚举：

```csharp
public enum PlayerMovementState
{
    IDLE,          // 待机
    WALK,          // 行走
    SPRINT,        // 冲刺
    CROUCH,        // 潜行（站立）
    CROUCH_WALK,   // 潜行移动
    ACTION         // 动作中（锁定）
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/PlayerMovementState.cs`

---

### 7.3 PlayerMovementStateChangedEvent

玩家移动状态变化事件：

```csharp
public struct PlayerMovementStateChangedEvent
{
    public PlayerMovementState old_state;
    public PlayerMovementState new_state;
}
```

**定义位置**：`Assets/Game/Foundation/PlayerController/Events/PlayerEvents.cs`

**发布者**：Player Controller
**订阅者**：LOS System（暴露值计算）

---

### 7.4 NoiseType

噪声类型枚举：

```csharp
public enum NoiseType
{
    NONE,
    WALK,
    SPRINT,
    CROUCH,
    INTERACTION
}
```

**定义位置**：`Assets/Game/Foundation/PlayerController/Noise/NoiseBroadcaster.cs`

---

### 7.5 NoiseEvent

噪声广播事件：

```csharp
public struct NoiseEvent
{
    public Vector3 position;
    public float radius;
    public NoiseType noise_type;
    public float duration;
    public bool can_interrupt;
    public int source_entity_id;
}
```

**定义位置**：`Assets/Game/Foundation/PlayerController/Noise/NoiseBroadcaster.cs`

**发布者**：Player Controller
**订阅者**：NPC AI System（听觉感知）

---

### 7.6 PlayerDamagedEvent

玩家受伤事件：

```csharp
public struct PlayerDamagedEvent
{
    public int player_id;
    public DamageType damage_type;
    public float damage_amount;      // 实际伤害值（用于 SanityRage 计算精神影响）
    public HitLocation hit_location;
    public int source_entity_id;
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/PlayerDamagedEvent.cs`

**发布者**：Health System
**订阅者**：Gritty Takedowns（打断交互）、SanityRage（精神影响计算）

---

## 8. 实体相关类型

### 8.1 EntityType

实体类型枚举：

```csharp
public enum EntityType
{
    NPC,
    Player
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/EntityType.cs`

---

## 9. 交互相关类型

### 9.1 InteractionType

玩家-NPC 交互类型枚举：

```csharp
public enum InteractionType
{
    // 处决类
    StealthKill,       // 潜行击杀
    EnvironmentKill,   // 环境处决
    FinishOff,         // 补刀

    // 强制类
    Intimidate,        // 威胁
    Bribe,             // 贿赂
    Deceive,           // 欺骗

    // 对峙类
    MaintainDistance,  // 保持距离威胁
    Dialogue,          // 对话选项

    // 情报类
    Search,            // 搜身
    Interrogate,       // 审问

    // 控制类
    TieUp,             // 捆绑
    Release,           // 解开

    // 转化类
    Convert            // 转化线人
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Types/InteractionType.cs`

---

### 9.2 InteractionResult

交互结果枚举：

```csharp
public enum InteractionResult
{
    Success,
    Failed,
    Interrupted,
    Cancelled
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Types/InteractionResult.cs`

---

### 9.3 InteractionEvent

通用交互事件：

```csharp
public struct InteractionEvent
{
    public InteractionType type;
    public int target_id;
    public string source;      // weapon_id 或 "bare_hands"
    public InteractionResult result;
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Events/InteractionEvent.cs`

---

### 9.4 InteractionStateChangedEvent

玩家交互状态变化事件：

```csharp
public struct InteractionStateChangedEvent
{
    public PlayerInteractionState old_state;
    public PlayerInteractionState new_state;
}

public enum PlayerInteractionState
{
    Idle,          // 未交互
    CanInteract,   // 可交互（检测到目标）
    Interacting    // 交互中（动画播放）
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Events/InteractionEvents.cs`

---

### 9.5 KillTagEvent

击杀标签事件（发送到 Sanity 系统）：

```csharp
public struct KillTagEvent
{
    public int npc_id;
    public NPCIdentityType kill_tag;  // 恶徒/帮凶/受害者
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Events/KillTagEvent.cs`

---

### 9.6 KnowledgeGainedEvent

线索获取事件（发送到 Clue 系统）：

```csharp
public struct KnowledgeGainedEvent
{
    public int npc_id;
    public List<string> knowledge_list;
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Events/KnowledgeGainedEvent.cs`

---

### 9.7 ExecutionWitnessedEvent

处决被目击事件（发送到 NPC AI 系统）：

```csharp
public struct ExecutionWitnessedEvent
{
    public int victim_id;
    public int witness_id;
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Events/ExecutionWitnessedEvent.cs`

---

### 9.8 DialogueChoice / DialogueResult

对话相关事件：

```csharp
// 玩家对话选项选择（发送到 NPC AI 系统处理）
public struct DialogueChoice
{
    public string dialogue_id;
    public string choice_id;  // 修正自 choice_index（v1.3.0）
}

// NPC AI 系统返回的对话结果
public struct DialogueResult
{
    public string dialogue_id;
    public bool success;
    /// <summary>
    /// 派系态度变化值，范围 [-30, +30]
    /// - 负值表示态度下降（威胁成功等）
    /// - 正值表示态度上升（贿赂成功等）
    /// 具体范围由 Tuning Knob 控制（见 ADR-0014 §Tuning Knobs：AllegianceChangeMin/AllegianceChangeMax）
    /// </summary>
    public int allegiance_change;
    public List<string> knowledge_gained;
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Events/DialogueEvents.cs`

---

## 10. World Map 类型

### 10.1 AreaState

地区状态机状态：

```csharp
public enum AreaState
{
    LOCKED,          // 锁定
    AVAILABLE,       // 可进入
    EXPLORING,       // 探索中
    EXTRACTED,       // 撤离成功
    DIED,            // 死亡
    ARRESTED,        // 被捕
    RETURNING_TO_MAP, // 返回中
    MAP_MODE         // 地图模式
}
```

**定义位置**：`Assets/Game/Foundation/WorldMap/AreaStateMachine.cs`

---

### 10.2 ExplorationState

地区探索完成度：

```csharp
public enum ExplorationState
{
    UNEXPLORED,  // 未探索
    EXPLORED,     // 已探索但未完成核心目标
    COMPLETED,    // 完成核心目标
    CLEARED       // 清除所有敌人
}
```

**定义位置**：`Assets/Game/Foundation/WorldMap/WorldMapTypes.cs`

---

### 10.3 World Map 事件

| 事件名 | 发布者 | 订阅者 | 携带数据 |
|--------|--------|--------|----------|
| `LocationRevealedEvent` | Clue System | World Map | location_id |
| `DiscoveryAnimationCompleteEvent` | UI System | World Map | location_id |
| `AreaUnlockEvent` | 剧情系统 | World Map | area_id |
| `AreaEnteredEvent` | World Map | NPC AI | area_id |
| `AreaClearedEvent` | NPC AI | World Map | area_id, enemy_count, is_full_clear |
| `ThreatDissipatedEvent` | NPC AI | World Map/UI | area_id, reason, wait_time_elapsed |
| `MapWaitingStartedEvent` | World Map | UI | area_id, max_wait_time |
| `MapWaitingCancelledEvent` | World Map | UI | area_id, reason |
| `LoadCompletedEvent` | Loading Screen System | World Map | destination, destination_type, was_successful |
| `CheckpointRestoreRequestEvent` | CheckpointSystem | World Map | area_id, checkpoint_position, arrest_location |
| `LoadingScreenRequestEvent` | World Map | Loading Screen | destination, destination_type, source_location |
| `MapDisplayRequestEvent` | World Map | UI | world_state |
| `AreaInfoRequestEvent` | World Map | UI | area_id |

**ThreatDissipatedEvent 原因枚举**（与 Event Bus ICD v1.2.2 保持一致）：
```csharp
public enum DissipateReason
{
    PROBABILITY_TRIGGER,  // 概率触发消散（每次检测有 30% 概率消散）
    MAX_WAIT_TIMEOUT       // 最大等待时间（30秒）超时后强制消散
}
```

**MapWaitingCancelledEvent 原因枚举**（与 Event Bus ICD 保持一致）：
```csharp
public enum CancelReason
{
    PLAYER_MOVED,      // 玩家主动移动
    PLAYER_ATTACKED,   // 玩家被攻击
    THREAT_ESCALATED   // 威胁升级
}
```

---

## 11. Clue & Journal 类型

### 11.1 ClueCategory

线索类别枚举（定义于 ADR-0016）：

```csharp
/// <summary>
/// 线索类别
/// </summary>
public enum ClueCategory
{
    IDENTITY,     // 身份线索：NPC 的真实身份、背景
    LOCATION,     // 位置线索：地点、入口、隐藏区域
    RELATIONSHIP, // 关系线索：NPC 之间的关系网络
    ITEM,         // 物品线索：关键道具、证据
    TRAGEDY       // 悲剧线索：悲剧事件、受害者信息
}
```

**定义位置**：`Assets/Game/Features/ClueJournal/Types/ClueCategory.cs`

**使用系统**：Clue & Journal System（ADR-0016）、Sanity/Rage System（ADR-0017）

---

### 11.2 DiscoveryStage

线索发现阶段枚举（定义于 ADR-0016）：

```csharp
/// <summary>
/// 线索发现阶段
/// </summary>
public enum DiscoveryStage
{
    FIRST_REVEAL,   // 首次揭示
    FOLLOWUP,       // 后续跟进
    DEEP_REVEAL     // 深度揭示
}
```

**定义位置**：`Assets/Game/Features/ClueJournal/Types/DiscoveryStage.cs`

---

### 11.3 NarrativeSignificance

叙事重要性枚举（定义于 ADR-0016）：

```csharp
/// <summary>
/// 叙事重要性
/// </summary>
public enum NarrativeSignificance
{
    NORMAL,         // 普通线索
    MAIN_TARGET,    // 主要目标相关
    NPC_SYMPATHY   // NPC 同情相关
}
```

**定义位置**：`Assets/Game/Features/ClueJournal/Types/NarrativeSignificance.cs`

---

### 11.4 ClueDiscoveredEvent

线索发现事件（定义于 ADR-0016）：

```csharp
/// <summary>
/// 线索发现事件
/// 由 Clue & Journal System 发布，Sanity/Rage System 订阅
/// 用于根据线索发现情境计算理智惩罚
/// </summary>
public struct ClueDiscoveredEvent
{
    /// <summary>
    /// 线索 ID
    /// </summary>
    public string clue_id;

    /// <summary>
    /// 线索类别
    /// </summary>
    public ClueCategory category;

    /// <summary>
    /// 发现阶段
    /// </summary>
    public DiscoveryStage discovery_stage;

    /// <summary>
    /// 叙事重要性
    /// </summary>
    public NarrativeSignificance narrative_significance;

    /// <summary>
    /// 发现方式来源 ID（NPC_ID 或物件 ID）
    /// </summary>
    public string source_id;
}
```

**定义位置**：`Assets/Game/Features/ClueJournal/Events/ClueEvents.cs`

**发布者**：Clue & Journal System
**订阅者**：Sanity/Rage System（ADR-0017）

---

## 12. Sanity/Rage 类型

### 12.1 PsychologicalState

心理状态枚举（定义于 ADR-0017）：

```csharp
/// <summary>
/// 心理状态枚举
/// </summary>
public enum PsychologicalState
{
    CALM,       // Sanity >= 70 且 Rage <= 30
    UNEASY,     // (Sanity 40-69 OR Rage 31-50) AND NOT AGITATED/FRENZIED/BROKEN/SOUL_SPLIT
    AGITATED,   // (Sanity 20-39 OR Rage 51-70) AND NOT FRENZIED/BROKEN/SOUL_SPLIT
    BROKEN,     // Sanity < 20 AND Rage <= 70
    FRENZIED,   // Rage >= 71 AND Sanity < 70
    SOUL_SPLIT  // Sanity < 20 AND Rage > 70
}
```

**定义位置**：`Assets/Game/Features/SanityRage/Types/PsychologicalState.cs`

**状态优先级**（数值越小优先级越高）：
| 优先级 | 状态 | 触发条件 |
|--------|------|----------|
| 1（最高） | SOUL_SPLIT | Sanity < 20 AND Rage > 70 |
| 2 | FRENZIED | Rage >= 71 AND Sanity < 70 |
| 3 | BROKEN | Sanity < 20 AND Rage <= 70 |
| 4 | AGITATED | (Rage > 70 OR Sanity < 40) AND NOT FRENZIED/BROKEN/SOUL_SPLIT |
| 5 | UNEASY | (Rage > 30 OR Sanity < 70) AND NOT AGITATED/FRENZIED/BROKEN/SOUL_SPLIT |
| 6（默认） | CALM | — |

---

### 12.2 PsychologicalStateEvent

心理状态变更事件（定义于 ADR-0017）：

```csharp
/// <summary>
/// 心理状态变更事件
/// 由 Sanity/Rage System 发布，UI System 和 DynamicPostProcessing 订阅
/// </summary>
public struct PsychologicalStateEvent
{
    /// <summary>
    /// 当前心理状态
    /// </summary>
    public PsychologicalState State;

    /// <summary>
    /// 状态变更原因（用于调试）
    /// </summary>
    public string Reason;
}
```

**定义位置**：`Assets/Game/Features/SanityRage/Events/SanityRageEvents.cs`

**发布者**：Sanity/Rage System
**订阅者**：UI System（ADR-0015）、DynamicPostProcessing

---

### 12.3 视觉效果请求事件（已废弃）

> **⚠️ 已废弃 (DEPRECATED)**
>
> 以下事件已废弃，**请改用 ADR-0023 定义的统一 `ScreenEffectRequestEvent` 结构**。
>
> 废弃原因：各系统独立定义效果请求导致重复和不一致。
> 统一后，所有屏幕后处理效果请求均通过 `ScreenEffectRequestEvent` 发送，由 ScreenEffectsManager 集中处理。
>
> **迁移指南**：将原有的独立事件替换为 ScreenEffectRequestEvent，例如：
> - `VignetteRequest{Intensity=X}` → `ScreenEffectRequestEvent{effectType=Vignette, intensity=X, sourceSystem=..., requesterId=...}`
> - `NoiseRequest{Intensity=X}` → `ScreenEffectRequestEvent{effectType=Noise, intensity=X, sourceSystem=..., requesterId=...}`
>
> **例外**：`HUDOverlayOpacityRequest` 属于 UI 层特效，不通过 ScreenEffectsManager 处理，保持不变。
>
> **相关决策**：[ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — 统一效果请求架构

~~```csharp
~~/// <summary>
/// 暗角效果请求
/// </summary>
public struct VignetteRequest
{
    public float Intensity;  // 0.0 = 无暗角，1.0 = 完全暗角
}
~~```

~~```csharp
~~/// <summary>
/// 噪点效果请求
/// </summary>
public struct NoiseRequest
{
    public float Intensity;  // 0.0 = 无噪点，1.0 = 完全噪点
}
~~```

~~```csharp
~~/// <summary>
/// 饱和度请求
/// </summary>
public struct SaturationRequest
{
    public float Multiplier;  // 0.0 = 灰度，1.0 = 正常色彩
}
~~```

~~```csharp
~~/// <summary>
/// 抖动效果请求（准星抖动）
/// </summary>
public struct ShakeRequest
{
    public float Intensity;  // 0.0 = 无抖动，8.0 = 强烈抖动
}
~~```

~~```csharp
~~/// <summary>
/// HUD 模糊效果请求
/// </summary>
public struct BlurRequest
{
    public float Intensity;  // 0.0 = 无模糊，1.0 = 完全模糊
}
~~```

### 12.3.1 HUDOverlayOpacityRequest（保留）

> **注意**：此事件不属于废弃范围，它用于 UI 层透明度控制，不通过 ScreenEffectsManager 处理。

```csharp
/// <summary>
/// HUD 半透明叠加层透明度请求
/// 用于 Sanity=0 时 HUD 仍可透过极端视觉效果显示
/// </summary>
public struct HUDOverlayOpacityRequest
{
    public float Opacity;  // 0.0 = 完全透明，1.0 = 完全不透明
}
```

**定义位置**：`Assets/Game/Features/SanityRage/Events/SanityRageEvents.cs`

**发布者**：Sanity/Rage System
**订阅者**：UI System（ADR-0015）

**Tuning Knob 参数**：

| 参数 | 默认值 | 安全范围 | 说明 |
|------|--------|---------|------|
| `HUDOverlayOpacity` | 0.3 | 0.1~0.5 | Sanity=0 时 HUD 半透明叠加层默认透明度 |

---

## 13. Dialog & UI 类型

### 13.1 DialogueEmotion

对话情绪枚举（定义于 ADR-0014）：

```csharp
/// <summary>
/// 对话情绪枚举
/// </summary>
public enum DialogueEmotion
{
    NEUTRAL,   // 普通
    AGITATED,  // 激动
    SCARED,    // 恐惧
    ANGRY      // 愤怒
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Types/DialogueEmotion.cs`

**UI 表现**：
| 值 | UI 表现 |
|----|---------|
| `NEUTRAL` | 标准对话气泡 |
| `AGITATED` | 气泡边缘抖动 |
| `SCARED` | 气泡颤抖 + 颜色变淡 |
| `ANGRY` | 气泡变红 + 边缘锯齿 |

---

### 13.2 DialogueResultType

对话结果类型枚举（定义于 ADR-0014）：

```csharp
/// <summary>
/// 对话结果类型
/// </summary>
public enum DialogueResultType
{
    CONTINUE,   // 对话继续
    INTIMIDATE, // 威胁成功
    BRIBE,      // 贿赂成功
    DECOY       // 欺骗成功
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Types/DialogueResultType.cs`

**Gritty Takedowns 处理**：
| 值 | Gritty Takedowns 处理 |
|----|----------------------|
| `CONTINUE` | 显示下一分支内容 |
| `INTIMIDATE` | 触发威胁成功音效 + allegiance 大幅下降 |
| `BRIBE` | 触发金币音效 + allegiance 提升 |
| `DECOY` | 触发欺骗特效（屏幕短暂闪白）+ allegiance 变化 |

---

### 13.3 ConfrontationStartRequest

对峙开始请求事件（定义于 ADR-0014）：

```csharp
/// <summary>
/// 对峙开始请求事件
/// 由 Gritty Takedowns 系统发起，请求 NPC AI 系统提供对话树数据
/// </summary>
public struct ConfrontationStartRequest
{
    /// <summary>
    /// 目标 NPC ID
    /// </summary>
    public int npc_id;

    /// <summary>
    /// 请求来源（用于调试）
    /// </summary>
    public string source;
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Events/DialogueEvents.cs`

**发布者**：Gritty Takedowns System
**订阅者**：NPC AI System

**超时处理**：2.0s 内无响应返回空配置，fallback 到默认选项（见 ADR-0014 EC-1）

---

### 13.4 PauseMenuOpened / PauseMenuClosed

暂停菜单事件（定义于 ADR-0015）：

```csharp
/// <summary>
/// 暂停菜单打开事件
/// </summary>
public struct PauseMenuOpened
{
    /// <summary>
    /// 暂停原因
    /// </summary>
    public PauseReason reason;
}

/// <summary>
/// 暂停菜单关闭事件
/// </summary>
public struct PauseMenuClosed
{
    /// <summary>
    /// 关闭原因
    /// </summary>
    public PauseReason reason;
}

/// <summary>
/// 暂停原因
/// </summary>
public enum PauseReason
{
    PLAYER_REQUEST,  // 玩家主动请求（ESC/Options）
    SYSTEM_PAUSE,   // 系统暂停（剧情过场等）
    TUTORIAL        // 教程暂停
}
```

**定义位置**：`Assets/Game/Features/UISystem/Events/UIEventBus.cs`

**发布者**：UI System
**订阅者**：GameTimeSystem（设置 time_scale = 0）、AudioSystem（音乐淡出）

**时序说明**：两个订阅者并行处理，AudioSystem 音乐淡出（0.3s）与 GameTimeSystem 立即暂停（time_scale = 0）互不阻塞。

---

### 13.5 InteractionState

环境物件交互状态枚举（定义于 ADR-0013）

> **⚠️ 注意**：此枚举与 §3.5 的 `ObjectState`（物件生命周期状态）是**完全不同的概念**，请勿混淆：
> - `ObjectState`（Spawned/Available/Held/Used/Depleted/Dropped）：描述物件的整体生命周期
> - `InteractionState`（Available/InUse/OnCooldown/Depleted）：描述物件的玩家交互状态

```csharp
/// <summary>
/// 环境物件交互状态
/// 描述物件在玩家交互过程中的状态变化
/// </summary>
public enum InteractionState
{
    Available,    // 可交互（物件在交互范围内）
    InUse,       // 使用中（动画播放中）
    OnCooldown,  // 冷却中（动画完成，等待冷却计时器）
    Depleted     // 耗尽（资源耗尽，需要刷新）
}
```

**定义位置**：`Assets/Game/Features/EnvironmentInteraction/Types/InteractionState.cs`

**使用系统**：Environment Interaction System（ADR-0013）、UI System（ADR-0015 高亮提示）

**状态转移规则**：
```
[Available] ──玩家交互──▶ [InUse] ──动画完成──▶ [OnCooldown]
     ▲                              │                    │
     │                         [远程触发]                  │
     │                              │                    ▼
     │                              └─────────────────▶ [Depleted]
     │                                                        │
     └───────────────────────（重置/刷新）─────────────────────┘
```

---

### 13.6 DialogueChoice 修正说明

> **⚠️ 重要修正（v1.3.0）**：`DialogueChoice.choice_index` 已修正为 `DialogueChoice.choice_id`
>
> ADR-0014 和 shared-types 中原定义为 `choice_index: int`（索引），但这会导致配置变化时索引偏移问题。统一改为 `choice_id: string`（唯一标识符），更稳定且易于调试。

```csharp
/// <summary>
/// 玩家对话选项选择（发送到 NPC AI 系统处理）
/// </summary>
public struct DialogueChoice
{
    public string dialogue_id;   // 对话树 ID
    public string choice_id;     // 选项 ID（修正自 choice_index）
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Events/DialogueEvents.cs`

**发布者**：Gritty Takedowns System
**订阅者**：NPC AI System

---

## 14. Clue & Journal 类型（补充）

### 14.1 ClueSourceType

线索来源类型枚举（定义于 ADR-0016）：

```csharp
/// <summary>
/// 线索来源类型
/// </summary>
public enum ClueSourceType
{
    LOS_EAVESDROP,      // LOS 窃听关键词触发
    ENVIRONMENT,        // 环境物件交互
    NPC_DEATH_SOURCE,   // NPC 死亡本身触发（如从尸体获取线索）
    NPC_KNOWLEDGE_BONUS // NPC 存活时可获取，死亡后标记 MISSING
}
```

**定义位置**：`Assets/Game/Features/ClueJournal/Types/ClueSourceType.cs`

**使用系统**：Clue & Journal System（ADR-0016）

**语义说明**：
| 值 | 触发条件 |
|----|---------|
| `LOS_EAVESDROP` | 通过 LOS 窃听系统获取（关键词触发） |
| `ENVIRONMENT` | 通过环境物件交互获取 |
| `NPC_DEATH_SOURCE` | NPC 死亡本身触发（如从尸体获取线索），NPC 存活则无法获取 |
| `NPC_KNOWLEDGE_BONUS` | NPC 存活时：线索已存在于 Journal 中，状态为 NOT_DISCOVERED（可正常发现）；NPC 死亡时：NOT_DISCOVERED 状态的线索直接标记为 MISSING（无法再获取） |

---

## 15. Gritty Takedowns 配置类型

### 15.1 StealthKillAlertMode

潜行击杀警戒状态检查模式（定义于 ADR-0011）：

```csharp
/// <summary>
/// 潜行击杀警戒状态检查模式
/// </summary>
public enum StealthKillAlertMode
{
    /// <summary>
    /// 宽松模式：UNDETECTED / SUSPECT / SEARCH 状态均可执行潜行击杀
    /// </summary>
    Loose,

    /// <summary>
    /// 严格模式：仅 UNDETECTED 状态可执行潜行击杀（默认）
    /// </summary>
    Tight
}
```

**定义位置**：`Assets/Game/Features/GrittyTakedowns/Types/StealthKillAlertMode.cs`

**使用系统**：Gritty Takedowns（ADR-0011）

**行为说明**：

| 模式 | 可执行状态 | 适用场景 |
|------|-----------|----------|
| `Tight`（默认） | 仅 UNDETECTED | 保证游戏挑战性，潜行击杀是精英操作 |
| `Loose` | UNDETECTED / SUSPECT / SEARCH | 允许玩家失误后补救，降低难度 |

**配置位置**：`GrittyTakedownsTuningSO.stealthKillAlertMode`

---

## 16. 环境交互类型

### 16.1 EnvironmentalEventType

环境事件类型枚举（定义于 ADR-0013）：

```csharp
/// <summary>
/// 环境事件类型
/// </summary>
public enum EnvironmentalEventType
{
    EXPLOSION,   // 爆炸事件
    FIRE,        // 火灾事件
    DESTRUCTION, // 破坏事件
    CHAOS,       // 混乱事件（当多个不同类型事件合并时生成）
    ALERT,       // 警戒事件
    DISTRACTION  // 干扰事件
}
```

**定义位置**：`Assets/Game/Features/EnvironmentInteraction/Types/EnvironmentalEventType.cs`

**使用系统**：Environment Interaction System（ADR-0013）、NPC AI System

**CHAOS 事件生成规则**（见 ADR-0013 EC-5）：
当 EventQueue 满（已有 10 个事件）时，新到达的不同类型事件不直接入队，而是合并为单个 `EnvironmentalEvent{type=CHAOS, intensity=sum_of_intensities, duration=max_duration}`。

---

### 16.2 EnvironmentalEvent

环境事件结构（定义于 ADR-0013）：

```csharp
/// <summary>
/// 环境事件
/// 由 Environment Interaction System 发布，NPC AI System 订阅
/// </summary>
public struct EnvironmentalEvent
{
    /// <summary>
    /// 事件类型
    /// </summary>
    public EnvironmentalEventType Type;

    /// <summary>
    /// 事件中心位置
    /// </summary>
    public Vector3 Position;

    /// <summary>
    /// 事件影响半径
    /// </summary>
    public float Radius;

    /// <summary>
    /// 事件持续时间
    /// </summary>
    public float Duration;

    /// <summary>
    /// 事件强度（用于叠加计算）
    /// </summary>
    public float Intensity;

    /// <summary>
    /// 触发此事件的物件 ID
    /// </summary>
    public string SourceObjectId;
}
```

**定义位置**：`Assets/Game/Features/EnvironmentInteraction/Events/EnvironmentalEvents.cs`

**发布者**：Environment Interaction System
**订阅者**：NPC AI System

**EventQueue 合并规则**（见 ADR-0013 EC-5）：
- 队列容量：EventQueueCapacity = 10
- 入队时机：新事件到达时直接入队，队列未满则正常添加
- 合并触发：当新事件到达时队列已满（已有 10 个事件），触发合并
- 合并规则：
  1. 同类型事件（均为 EXPLOSION）→ 合并为单个事件，intensity 叠加，duration 取最大值
  2. 不同类型事件 → 合并为 `EnvironmentalEvent{type=CHAOS, intensity=sum_of_intensities, duration=max_duration}`

---

## 17. Input System 类型（ADR-0020）

### 17.1 InputDeviceType

输入设备类型枚举（定义于 ADR-0020）：

```csharp
/// <summary>
/// 输入设备类型
/// </summary>
public enum InputDeviceType
{
    Keyboard,
    PS5Gamepad,
    PS4Gamepad,
    XboxGamepad,
    GenericGamepad
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Types/InputDeviceType.cs`

**使用系统**：Input System（ADR-0020）

**设计说明**：使用详细的手柄类型枚举而非简单的 `Gamepad`，是为了支持不同平台的手柄差异化功能（如 PS5 DualSense 的 haptic feedback 特性）。

---

### 17.2 InputDeviceChangedEvent

输入设备切换事件（定义于 ADR-0020）：

```csharp
/// <summary>
/// 输入设备切换事件
/// 由 InputManager 发布，订阅者据此调整 UI 提示文本
/// </summary>
public struct InputDeviceChangedEvent
{
    /// <summary>
    /// 切换后的设备类型
    /// </summary>
    public InputDeviceType Device;
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/InputDeviceChangedEvent.cs`

**发布者**：InputManager
**订阅者**：UI System

---

### 17.3 InputRemappedEvent

输入重映射事件（定义于 ADR-0020）：

```csharp
/// <summary>
/// 输入重映射事件
/// 由 InputRemapManager 发布，InputManager 订阅后重新初始化对应 Action 的绑定
/// </summary>
public struct InputRemappedEvent
{
    /// <summary>
    /// 被重映射的 Action 名称，null 表示全部重置
    /// </summary>
    public string ActionName;

    /// <summary>
    /// 设备类型
    /// </summary>
    public InputDeviceType Device;
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/InputRemappedEvent.cs`

**发布者**：InputRemapManager
**订阅者**：InputManager

---

## 18. Resource Management 类型（ADR-0019）

### 18.1 ResourceCategory

资源分类枚举（定义于 ADR-0019）：

```csharp
/// <summary>
/// 资源分类
/// 用于 ResourceManager 的内存预算控制和生命周期管理
/// </summary>
public enum ResourceCategory
{
    /// <summary>游戏启动时预加载，仅在游戏退出时卸载</summary>
    Resident,

    /// <summary>进入场景时加载，离开场景时卸载</summary>
    SceneBound,

    /// <summary>首次访问时加载，引用计数归零时卸载</summary>
    OnDemand,

    /// <summary>分块渐进加载，可随时卸载</summary>
    Streaming
}
```

**定义位置**：`Assets/Game/Infrastructure/ResourceManager/Types/ResourceCategory.cs`

**使用系统**：ResourceManager（ADR-0019）

---

## 19. Event Bus 查询类型（ADR-0018）

### 19.1 QuerySoundSourceScreenPosition

LOS 声音源屏幕位置查询（定义于 ADR-0018）：

```csharp
/// <summary>
/// 查询声音源的屏幕空间位置
/// 类型：Query（同步查询，通过 QueryBus 返回）
/// </summary>
public struct QuerySoundSourceScreenPosition
{
    /// <summary>
    /// 目标 NPC ID
    /// </summary>
    public int npc_id;
}
```

**Handler**：LOSSystem
**Response**：`Vector3`（屏幕空间位置）

**定义位置**：`Assets/Game/Core/LOS/Events/QuerySoundSourceScreenPosition.cs`

**使用说明**：
```csharp
// 调用方式（通过 QueryBus）
var position = QueryBus.Instance.Query<QuerySoundSourceScreenPosition, Vector3>(
    new QuerySoundSourceScreenPosition { npc_id = npcId }
);
```

---

### 19.2 LoadingScreenRequestEvent

加载画面请求事件（定义于 ADR-0018，补充）：

```csharp
/// <summary>
/// 加载画面请求事件
/// 由 WorldMap System 发布，UI System 订阅后显示加载画面
/// </summary>
public struct LoadingScreenRequestEvent
{
    /// <summary>
    /// 目标场景/位置
    /// </summary>
    public string destination;

    /// <summary>
    /// 目标类型：city / area / loading_tip
    /// </summary>
    public string destination_type;

    /// <summary>
    /// 来源位置
    /// </summary>
    public string source_location;
}
```

**定义位置**：`Assets/Game/Foundation/WorldMap/Events/LoadingScreenRequestEvent.cs`

**发布者**：WorldMap System、SceneManagerWrapper
**订阅者**：UI System

---

### 19.3 AssetLoadedEvent

资源加载完成事件（定义于 ADR-0019）：

```csharp
/// <summary>
/// 资源加载完成事件
/// 由 ResourceManager 发布，订阅者可以响应资源加载完成
/// </summary>
public struct AssetLoadedEvent
{
    /// <summary>
    /// 资源的 Addressables 地址
    /// </summary>
    public string Address;

    /// <summary>
    /// 资源类型名称
    /// </summary>
    public string AssetType;

    /// <summary>
    /// 估算的内存占用（字节）
    /// </summary>
    public long EstimatedSizeBytes;
}
```

**定义位置**：`Assets/Game/Infrastructure/ResourceManager/Events/AssetEvents.cs`

**发布者**：ResourceManager
**订阅者**：Any System（按需订阅）

---

### 19.4 AssetUnloadedEvent

资源卸载完成事件（定义于 ADR-0019）：

```csharp
/// <summary>
/// 资源卸载完成事件
/// 由 ResourceManager 发布，订阅者可以响应资源卸载完成
/// </summary>
public struct AssetUnloadedEvent
{
    /// <summary>
    /// 被卸载资源的地址
    /// </summary>
    public string Address;
}
```

**定义位置**：`Assets/Game/Infrastructure/ResourceManager/Events/AssetEvents.cs`

**发布者**：ResourceManager
**订阅者**：Any System（按需订阅）

---

### 19.5 AssetReleaseEvent

资源引用计数归零事件（定义于 ADR-0019，补充）：

```csharp
/// <summary>
/// 资源引用计数归零事件
/// 由 ResourceManager 发布，当资源的引用计数从 1 变为 0 时触发（表示资源可以被卸载）
/// 与 AssetUnloadedEvent 的区别：此事件在引用计数归零时发布，卸载可能稍后进行
/// </summary>
public struct AssetReleaseEvent
{
    /// <summary>
    /// 被释放资源的地址
    /// </summary>
    public string Address;

    /// <summary>
    /// 资源的原始类型名称
    /// </summary>
    public string AssetType;
}
```

**定义位置**：`Assets/Game/Infrastructure/ResourceManager/Events/AssetEvents.cs`

**发布者**：ResourceManager
**订阅者**：Any System（按需订阅，用于预判资源即将卸载）

---

---

## 22. 游戏时间接口与 World Layer 时间事件

> **来源**：从 ADR-0021（天气系统）和 ADR-0022（光照系统）提取，消除重复定义。
> Weather/Lighting 两个系统均依赖此接口，集中定义此处以便统一维护。

### 22.1 IGameTimeProvider 接口

```csharp
/// <summary>
/// 游戏时间提供者接口
/// 由 WorldMap System 实现，Weather/Lighting 等 World Layer 系统通过此接口获取时间
/// 注意：World Layer 系统不直接依赖 WorldMap System，而是依赖此接口（依赖倒置）
/// </summary>
public interface IGameTimeProvider
{
    /// <summary>获取当前游戏内小时（0.0 ~ 24.0）</summary>
    float GetCurrentGameHour();

    /// <summary>获取游戏时间流逝速度（1.0 = 正常速度）</summary>
    float GetTimeScale();
}
```

**定义位置**：`Assets/Game/Core/Environment/Shared/IGameTimeProvider.cs`

**实现者**：WorldMap System（`WorldMapTimeProvider : IGameTimeProvider`）
**使用者**：WeatherSystemManager、LightingSystemManager

### 22.2 GameHourChangedEvent

```csharp
/// <summary>
/// 游戏时间变化事件
/// 由 WorldMap System 发布（每整点或每隔固定游戏分钟触发一次）
/// WeatherSystem 和 LightingSystem 订阅此事件驱动时段切换和天气生成，无需主动轮询
/// </summary>
public struct GameHourChangedEvent
{
    /// <summary>当前游戏内小时（0.0 ~ 24.0）</summary>
    public float currentHour;

    /// <summary>游戏时间流逝速度（1.0 = 正常速度）</summary>
    public float timeScale;
}
```

**定义位置**：`Assets/Game/Core/Environment/Shared/GameHourChangedEvent.cs`

**发布者**：WorldMap System
**订阅者**：WeatherSystemManager、LightingSystemManager

### 22.3 ScreenEffectRevokeEvent

```csharp
/// <summary>
/// 屏幕特效撤销事件（与 ScreenEffectRequestEvent 对称）
/// 请求方通过 EventBus 发布此事件撤销之前的效果请求，无需持有 ScreenEffectsManager 引用
/// </summary>
public struct ScreenEffectRevokeEvent
{
    /// <summary>撤销来源系统</summary>
    public ScreenEffectSource sourceSystem;

    /// <summary>要撤销的请求者 ID（与原 ScreenEffectRequestEvent.requesterId 一致）</summary>
    public string requesterId;

    /// <summary>事件时间戳</summary>
    public float timestamp;
}
```

**定义位置**：`Assets/Game/Infrastructure/ScreenEffects/Events/ScreenEffectRequestEvent.cs`（与 Request 共文件）

**发布者**：任何发布过 ScreenEffectRequestEvent 的系统（DialogueSystem、WeatherSystem 等）
**订阅者**：ScreenEffectsManager（唯一订阅者）

### 22.4 感知系数叠加规则（Weather × Lighting）

> **来源**：从 ADR-0021 和 ADR-0022 提取，解决跨 ADR 系数组合的二义性。
> 本节定义 Weather System 和 Lighting System 的感知系数如何组合作用于下游系统。

**问题背景**：
- Weather System 的 `PerceptionModifier` 包含：`visionDistanceMultiplier`、`visionAngleMultiplier`、`hearingSensitivityMultiplier`、`soundPropagationMultiplier`
- Lighting System 的 `AreaLightingCoefficients` 包含：`shadowStealthMultiplier`、`lightSensitivityMultiplier`、`finalIllumination`
- ADR-0021/0022 描述两者"相乘"，但维度不同，无法直接相乘

**组合规则**：

| 下游系统 | Weather PerceptionModifier | Lighting AreaLightingCoefficients | 组合方式 |
|---------|---------------------------|----------------------------------|---------|
| NPC AI 感知 | `visionDistanceMultiplier` | — | 直接使用 Weather 值 |
| NPC AI 感知 | — | `shadowStealthMultiplier` | 直接使用 Lighting 值 |
| NPC AI 感知 | `visionAngleMultiplier` | `lightSensitivityMultiplier` | **乘法组合**：`Weather × Lighting` |
| LOS System | `visionDistanceMultiplier` | — | 直接使用 Weather 值 |
| LOS System | `hearingSensitivityMultiplier` | — | 直接使用 Weather 值 |
| LOS System | `soundPropagationMultiplier` | — | 直接使用 Weather 值 |

**组合公式**：
```
最终感知系数 = Weather_PerceptionModifier × Lighting_AreaLightingCoefficients
```

**示例**：
```csharp
// Night + PitchBlack + Storm 场景
float finalVisionAngle = weather.visionAngleMultiplier * lighting.lightSensitivityMultiplier;
// = 0.5 (Storm弱) × 0.5 (PitchBlack) = 0.25

float finalStealthBonus = lighting.shadowStealthMultiplier;
// = 2.0 (PitchBlack)，与 Weather 无关
```

**注意**：`shadowStealthMultiplier` 和 `lightSensitivityMultiplier` 是 Lighting System 独立维护的参数，Weather System 不提供对应的折扣系数。

---

## 21. 修改日志

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-10 | 1.0.0 | 初稿创建，统一 DamageRequest、ExplosionEvent、WeaponQuery、ActionLockSystem 等跨 ADR 类型 | 架构师 Agent |
| 2026-04-10 | 1.2.0 | 修复：CheckpointRestoreRequestEvent 补充 arrest_location 字段、LoadingScreenRequestEvent 补充 source_location 字段（与 ADR-0012 保持一致） | 架构师 Agent |
| 2026-04-10 | 1.1.0 | 新增 NPC AI/Health/Player/Weapon/GrittyTakedowns/WorldMap 完整类型定义 | 架构师 Agent |
| 2026-04-10 | 1.1.1 | 修复：AreaClearedEvent 补充完整字段（area_id, enemy_count, is_full_clear）、新增 LoadCompletedEvent（Event Bus ICD v1.2.3） | 架构师 Agent |
| 2026-04-10 | 1.2.0 | 修复：NPCSizeCategory multiplier 与 ADR-0011 §TieUpCalculator 对齐；Bravery 范围修正为 [1,10] 并明确与贿赂/欺骗的关系；新增 §6.1.1 WorldState↔HealthState 同步协议 | 架构师 Agent |
| 2026-04-11 | 1.3.0 | 新增 ClueDiscoveredEvent、DialogueEmotion/DialogueResultType 枚举、ConfrontationStartRequest/PauseMenuOpened 事件、MissingReasonType 枚举、InteractionState 枚举；修正 DialogueChoice.choice_index → choice_id；澄清 ObjectState 与 InteractionState 的区分 | 架构师 Agent |
| 2026-04-11 | 1.4.0 | 修复章节编号跳跃：§14.x → §13.x；澄清引用路径 | 架构师 Agent |
| 2026-04-11 | 1.5.0 | 新增 BlurRequest/HUDOverlayOpacityRequest 事件至 §12.3；新增 ClueSourceType 枚举至 §14.1；修正章节编号 (§15→§16) | 架构师 Agent |
| 2026-04-11 | 1.6.0 | 新增 §16 环境交互类型：EnvironmentalEventType（含 CHAOS）、EnvironmentalEvent；ObjectCategory 枚举补充 GasolineCan/PropaneTank | 架构师 Agent |
| 2026-04-11 | 1.7.0 | 新增 ADR-0018/0019/0020 补录的类型：ResourceCategory、LoadingScreenRequestEvent、InputDeviceType、InputDeviceChangedEvent、InputRemappedEvent；QuerySoundSourceScreenPosition 查询结构 | 架构师 Agent |
| 2026-04-11 | 1.8.0 | 新增 AssetLoadedEvent、AssetUnloadedEvent；修复 ADR-0018 PlayerDamagedEvent 字段名 (target_id→player_id)；章节重编号 (§19→§21) | 架构师 Agent |
| 2026-04-12 | 1.9.0 | 新增 AssetReleaseEvent；PlayerDamagedEvent 新增 damage_amount 字段（修复 ADR-0018 评审问题） | 架构师 Agent |
| 2026-04-12 | 2.0.0 | 新增 §22 游戏时间接口：IGameTimeProvider、GameHourChangedEvent（从 ADR-0021/0022 提取，消除重复定义）；新增 ScreenEffectRevokeEvent（ADR-0023 评审修复） | 架构师 Agent |
| 2026-04-12 | 2.2.0 | 新增 §22.4 感知系数叠加规则（ADR-21/22/23 跨 ADR 评审修复）；修复 ADR-0021 gameHour 预留参数注释、WeatherForceChangeEvent 打断行为说明、WeatherTransitionTable 冗余字段；修复 ADR-0022 LookupCoefficients 硬编码系数（新增 AreaLightingCoefficientsTable）、AreaLightingDetector 缓存失效问题；修复 ADR-0023 ScreenEffectType Flags 语义歧义、Shake 混合注释、AdditiveLayerDecay UX 测试默认值标注 | 架构师 Agent |
