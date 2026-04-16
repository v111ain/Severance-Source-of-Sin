# 共享类型定义 (Shared Types)

> **版本**: 2.6.0
> **创建日期**: 2026-04-10
> **状态**: APPROVED
> **维护者**: 架构师
> **更新日期**: 2026-04-15 (v2.6.0 — ADR评审修复：新增Exposure值到AlertState阈值映射表 §19.1.3；验证SaveCompletedEvent/SaveCorruptedEvent定义于§10.1；验证PlayerDamagedEvent字段；验证DialogueEmotion定义于§13.1)

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
    public float damage_amount;     // 已应用远程衰减的实际伤害值（环境物件=1.0，热武器=base_damage×RangeMultiplier）
    public int penetration;        // 武器穿透等级（穿透判定：penetration >= armorLevel 时穿透）
    public HitLocation hit_location; // 命中部位（HEAD/TORSO/LIMBS）
    public int source_entity_id;    // 伤害来源实体 ID（用于死亡追踪）
    public string source;           // 伤害来源标识（weapon_id 或 "bare_hands"）

    /// <summary>
    /// 射击距离（米），用于远程伤害衰减计算
    /// 由 Weapon System 在发送 DamageRequest 前计算并填充
    /// </summary>
    public float distance;
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/DamageRequest.cs`

**发布者**：Weapon System、Gritty Takedowns
**订阅者**：Health System

**远程伤害衰减处理**：
- `WeaponSystem` 在发送 `DamageRequest` 前，调用 `RangeMultiplier(distance)` 计算远程衰减系数
- `HealthSystem` 收到 `DamageRequest` 时，`damage_amount` **已包含远程衰减**
- 衰减公式定义见 ADR-0010 §8.1：`RangeMultiplier(distance)` 返回 0.7~1.0 之间的值

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

### 3.6 WeaponAwarenessEvent

NPC 感知到武器的事件：

```csharp
/// <summary>
/// NPC 感知到武器事件（已于 2026-04-12 从 WeaponAwareness 更名为 WeaponAwarenessEvent）
/// </summary>
[Obsolete("Use WeaponAwarenessEvent instead. Removed after v1.0.")]
public struct WeaponAwareness
{
    public string weapon_id;
    public Vector3 position;
    public WeaponCategory weapon_type;

    /// <summary>
    /// 隐式转换为 WeaponAwarenessEvent
    /// </summary>
    public static implicit operator WeaponAwarenessEvent(WeaponAwareness old)
    {
        return new WeaponAwarenessEvent
        {
            weapon_id = old.weapon_id,
            position = old.position,
            weapon_type = old.weapon_type
        };
    }
}

/// <summary>
/// NPC 感知到武器事件
/// 发布者：Weapon System
/// 订阅者：NPC AI System
/// </summary>
public struct WeaponAwarenessEvent
{
    public string weapon_id;
    public Vector3 position;
    public WeaponCategory weapon_type;
}
```

**定义位置**：`Assets/Game/Features/WeaponSystem/Events/WeaponEvents.cs`

**发布者**：Weapon System
**订阅者**：NPC AI System

> **命名说明**：已于 2026-04-12 从 `WeaponAwareness` 更名为 `WeaponAwarenessEvent`，遵循 ADR-0018 的 `Subject + Did + Context + Event` 命名规范。`WeaponAwareness` 作为别名保留用于过渡期兼容。

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
    public WorldState old_world_state;
    public WorldState new_world_state;
    public HealthState old_health_state;   // 用于同步验证
    public HealthState new_health_state;   // 用于 GrittyTakedowns 判定
    public DamageType damage_type;  // LETHAL / BLUNT / NONE

    /// <summary>
    /// 【新增 v2.4】是否有目击者
    /// 用于 Sanity 系统判断是否触发"目睹 NPC 死亡"惩罚
    /// </summary>
    public bool has_witness;

    /// <summary>
    /// 【新增 v2.4】目击者距离（如果 has_witness=true）
    /// 用于 Sanity 系统计算惩罚衰减
    /// </summary>
    public float witness_distance;
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/NPCStateChangedEvent.cs`

**发布者**：Health System（通过 NPCStateManager 协调）
**订阅者**：NPC AI System、Gritty Takedowns、**Sanity/Rage System**

> **同步说明**：根据 §6.1.1 协议，NPCStateManager 在 HealthState 变化时同步更新 WorldState，并通过此事件广播完整的双状态快照。
>
> **目击判定补充（v2.4）**：Sanity 系统订阅此事件，当 `new_world_state=DEAD` 时通过 `has_witness` 和 `witness_distance` 字段判断玩家是否目击了死亡事件，从而决定是否触发"目睹 NPC 死亡"的理智惩罚。

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
    Small,   // 体型小，offset=0.0f → 乘数 1.0f
    Medium,  // 体型中等，offset=0.25f → 乘数 1.25f
    Large    // 体型大，offset=1.0f → 乘数 2.0f
}
```

**定义位置**：`Assets/Game/Core/NPCAI/NPCSizeCategory.cs`

**计算公式**（见 ADR-0011 §TieUpCalculator）：
```
duration = baseTime * (1.0f + npcSizeOffset - playerSkillBonus)
duration = Clamp(duration, minDuration=3s, maxDuration=6s)
```

**说明**：
- 此类型由 NPC AI System 定义（Core Layer），Gritty Takedowns 通过 NPCController.QuerySizeCategory() 获取
- ADR-0011 §TieUpCalculator 中 npcSizeMultiplier 实为 offset，最终乘数 = (1.0f + offset)
- **统一声明**：本定义（Small×1.0, Medium×1.25, Large×2.0）为权威版本，与 ADR-0011 §TieUpCalculator 实际产生效果一致

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

### 5.10 CombatStateChangedEvent

NPC 战斗状态变化事件（由 ADR-0004 NPC AI 系统发布）：

```csharp
/// <summary>
/// NPC 战斗状态变化事件
/// 发布者：NPC AI System
/// 订阅者：Sanity/Rage System（战斗状态影响心理）
/// </summary>
public struct CombatStateChangedEvent
{
    /// <summary>
    /// 是否有任意 NPC 处于战斗状态
    /// </summary>
    public bool IsInCombat;
}
```

**定义位置**：`Assets/Game/Core/NPCAI/Events/CombatStateChangedEvent.cs`

**发布者**：NPC AI System（当任意 NPC 进入/离开 COMBAT AlertState 时发布）
**订阅者**：Sanity/Rage System

---

### 5.11 KeywordCapturedEvent

LOS窃听关键词捕获事件（由 LOS System 发布，Clue Journal 订阅）：

```csharp
/// <summary>
/// LOS窃听系统捕获到 NPC 对话关键词时发布
/// 订阅者：Clue & Journal System（触发线索发现）
/// </summary>
public struct KeywordCapturedEvent
{
    /// <summary>
    /// 捕获到的关键词
    /// </summary>
    public string keyword;

    /// <summary>
    /// 发出对话的 NPC ID
    /// </summary>
    public int npc_id;

    /// <summary>
    /// 位置区域 ID
    /// </summary>
    public string location_id;

    /// <summary>
    /// 关键词类别（用于 ClueJournal 匹配）
    /// </summary>
    public ClueCategory category;

    /// <summary>
    /// 捕获时间戳
    /// </summary>
    public float capture_timestamp;
}
```

**定义位置**：`Assets/Game/Core/LOS/Events/KeywordCapturedEvent.cs`

**发布者**：LOS System
**订阅者**：Clue & Journal System

---

### 5.11.1 NPCIdentityConfirmedEvent

NPC 身份确认事件（由 LOS System 发布，Clue Journal 订阅）：

```csharp
/// <summary>
/// NPC 身份确认事件
/// 当 LOS System 的 NPCIdentityManager 置信度达到 1.0 时发布
/// 订阅者：Clue & Journal System（触发线索发现）
/// </summary>
public struct NPCIdentityConfirmedEvent
{
    /// <summary>
    /// NPC ID
    /// </summary>
    public int npc_id;

    /// <summary>
    /// 确认的身份类型
    /// </summary>
    public NPCIdentityType confirmed_identity;

    /// <summary>
    /// 确认时间戳
    /// </summary>
    public float timestamp;
}
```

**定义位置**：`Assets/Game/Core/LOS/Events/NPCIdentityConfirmedEvent.cs`

**发布者**：LOS System（NPCIdentityManager）
**订阅者**：Clue & Journal System

---

### 5.12 NPCSpeakingChangedEvent

NPC 发声状态变化事件（由 NPC AI System 发布，LOS System 订阅以维护声音源追踪）：

```csharp
/// <summary>
/// NPC 发声状态变化事件
/// 由 NPC AI System 发布，LOS System 订阅以更新 _activeSoundSources 列表
/// FocusListener 依赖此事件判断 NPC 当前是否在说话
/// </summary>
public struct NPCSpeakingChangedEvent
{
    /// <summary>
    /// NPC ID
    /// </summary>
    public int npc_id;

    /// <summary>
    /// 是否正在说话
    /// </summary>
    public bool is_speaking;

    /// <summary>
    /// 声音源位置（如果正在说话）
    /// </summary>
    public Vector3 position;
}
```

**定义位置**：`Assets/Game/Core/NPCAI/Events/NPCSpeakingChangedEvent.cs`

**发布者**：NPC AI System
**订阅者**：LOS System（FocusListener）

**使用场景**：LOS System 维护 _activeSoundSources 列表用于：
1. 窃听优先级：NPC 说话时更容易被窃听
2. 暴露判定：NPC 发声时玩家的暴露值增长速度下降（NPC 注意力被分散）

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

**同步机制**：
- Health System 处理完伤害后，发布 `NPCStateChangedEvent`（携带完整双状态快照）
- NPCController 订阅该事件，根据 HealthState 变化更新 WorldState
- 详细同步逻辑见 [ADR-0008 §9](../architecture/adr-0008-health-lethality-architecture.md#9-npc状态变化广播) 和 [ADR-0004 NPCController](../architecture/adr-0004-npc-ai-behavior-architecture.md)。

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
- NPCController 订阅 HealthSystem 发布的 `NPCStateChangedEvent`，根据事件中的 HealthState 变化驱动 WorldState 更新

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

### 7.4 PlayerPositionUpdatedEvent

玩家位置更新事件：

```csharp
/// <summary>
/// 玩家位置更新事件
/// 由 PlayerController 在玩家位置变化时发布，Camera System 等系统订阅
/// 用于锁定相机时获取玩家当前位置
///
/// **发布频率注意**：
/// PlayerController 应控制事件发布频率（建议每 0.05s 或每帧一次），
/// 避免过于频繁的事件发布影响性能。LockOnCameraBehavior 依赖此事件
/// 计算相机中点，位置精度要求不高（使用 Vector3.Lerp 已足够平滑）。
/// </summary>
public struct PlayerPositionUpdatedEvent
{
    public Vector3 position;

    /// <summary>
    /// 实体 ID（用于精确验证，防止误接收其他实体位置）
    /// </summary>
    public int entityId;

    /// <summary>
    /// 事件发布时间戳（用于防止使用过期位置数据）
    /// </summary>
    public float timestamp;
}
```

**定义位置**：`Assets/Game/Foundation/PlayerController/Events/PlayerEvents.cs`

**发布者**：Player Controller
**订阅者**：Camera System（LockOnCameraBehavior）

---

### 7.5 NoiseType

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

### 7.5 NoiseMadeEvent

噪声广播事件：

```csharp
/// <summary>
/// 噪声广播事件（已于 2026-04-12 从 NoiseEvent 更名为 NoiseMadeEvent）
/// </summary>
[Obsolete("Use NoiseMadeEvent instead. Removed after v1.0.")]
public struct NoiseEvent
{
    public Vector3 position;
    public float radius;
    public NoiseType noise_type;
    public float duration;
    public bool can_interrupt;
    public int source_entity_id;

    /// <summary>
    /// 隐式转换为 NoiseMadeEvent
    /// </summary>
    public static implicit operator NoiseMadeEvent(NoiseEvent old)
    {
        return new NoiseMadeEvent
        {
            position = old.position,
            radius = old.radius,
            noise_type = old.noise_type,
            duration = old.duration,
            can_interrupt = old.can_interrupt,
            source_entity_id = old.source_entity_id
        };
    }
}

/// <summary>
/// 噪声广播事件
/// 发布者：Player Controller
/// 订阅者：NPC AI System（听觉感知）
/// </summary>
public struct NoiseMadeEvent
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

> **命名说明**：已于 2026-04-12 从 `NoiseEvent` 更名为 `NoiseMadeEvent`，遵循 ADR-0018 的 `Subject + Did + Context + Event` 命名规范。`NoiseEvent` 作为别名保留用于过渡期兼容。

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

    /// <summary>
    /// 受伤后的健康状态（用于订阅者判断是否需要打断交互等）
    /// </summary>
    public HealthState new_state;
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

### 9.6 TakedownAnimationCompleteEvent

处决动画完成事件（由 AnimationEventBridge 发布，GrittyTakedowns 订阅）：

```csharp
public struct TakedownAnimationCompleteEvent
{
    public int EntityId;              // 目标实体 ID
    public int AnimationHash;         // 动画哈希值（用于验证）
    public InteractionType TakedownType;  // 处决类型
}
```

**使用场景**：AnimationEventBridge 在收到 `OnStealthKillHit` 等动画事件后发布此事件，通知 GrittyTakedowns 动画已完成，由 GrittyTakedowns 发布最终的 InteractionEvent。

**定义位置**：`Assets/Game/Features/Animation/Events/TakedownAnimationCompleteEvent.cs`

---

### 9.7 KillTagEvent

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

## 10.1 Save/Load System 类型

> **补充日期**：2026-04-14
> **来源**：ADR-0005 评审修复

### SaveCompletedEvent

存档完成事件（定义于 ADR-0005）：

```csharp
/// <summary>
/// 存档完成事件
/// 由 SaveManager 发布，UI System 订阅以更新存档槽位显示
/// </summary>
public struct SaveCompletedEvent
{
    /// <summary>
    /// 存档槽位 ID
    /// </summary>
    public string slot_id;

    /// <summary>
    /// 是否为自动存档（true=自动存档，false=手动存档）
    /// </summary>
    public bool is_auto_save;
}
```

**定义位置**：`Assets/Game/Foundation/SaveSystem/Events/SaveCompletedEvent.cs`

**发布者**：SaveManager
**订阅者**：UI System

---

### SaveCorruptedEvent

存档损坏事件（定义于 ADR-0005）：

```csharp
/// <summary>
/// 存档损坏事件
/// 由 SaveManager 发布，UI System 订阅以显示错误提示
/// </summary>
public struct SaveCorruptedEvent
{
    /// <summary>
    /// 损坏的存档槽位 ID
    /// </summary>
    public string slot_id;
}
```

**定义位置**：`Assets/Game/Foundation/SaveSystem/Events/SaveCorruptedEvent.cs`

**发布者**：SaveManager
**订阅者**：UI System

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

### 11.5 VulnerabilityUncoveredEvent

线索发现触发 vulnerability 解锁事件（定义于 ADR-0016）：

```csharp
/// <summary>
/// Vulnerability 解锁事件
/// 由 Clue & Journal System 发布，Gritty Takedowns 系统订阅
/// 用于当线索发现后解锁 NPC 的 vulnerability，使玩家可进行审问
/// </summary>
public struct VulnerabilityUncoveredEvent
{
    /// <summary>
    /// NPC ID
    /// </summary>
    public int npc_id;

    /// <summary>
    /// 解锁来源：INTERROGATION / SEARCH / CLUE_DISCOVERY
    /// </summary>
    public VulnerabilitySource source;

    /// <summary>
    /// Vulnerability 数据（具体内容由 NPC AI 系统通过 QueryNPCVulnerabilityResponse 返回）
    /// </summary>
    public VulnerabilityData vulnerability;

    /// <summary>
    /// 触发该事件的线索 ID（如果 source == CLUE_DISCOVERY）
    /// </summary>
    public string clue_id;

    /// <summary>
    /// 事件时间戳
    /// </summary>
    public float timestamp;
}

/// <summary>
/// Vulnerability 解锁来源
/// </summary>
public enum VulnerabilitySource
{
    INTERROGATION,    // 审问已捆绑 NPC
    SEARCH,           // 搜身死亡 NPC
    CLUE_DISCOVERY    // 线索发现触发
}

/// <summary>
/// Vulnerability 数据结构（由 NPC AI 系统提供）
/// </summary>
public struct VulnerabilityData
{
    /// <summary>
    /// Vulnerability 类型
    /// </summary>
    public VulnerabilityType type;

    /// <summary>
    /// 描述文本（用于 UI 显示）
    /// </summary>
    public string description;
}

/// <summary>
/// Vulnerability 类型
/// </summary>
public enum VulnerabilityType
{
    WEAKNESS,     // 弱点
    SECRET,       // 秘密
    FEAR,         // 恐惧
    LIE           // 谎言
}
```

**定义位置**：`Assets/Game/Features/ClueJournal/Events/VulnerabilityEvents.cs`

**发布者**：Clue & Journal System
**订阅者**：Gritty Takedowns 系统（ADR-0011/ADR-0014）、UI 系统（ADR-0015，用于显示 vulnerability 发现提示）

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

### 12.2.1 RageChangedEvent

愤怒值变化事件（用于验证标准）：

```csharp
/// <summary>
/// 愤怒值变化事件
/// 由 Sanity/Rage System 在愤怒值发生变化时发布
/// </summary>
public struct RageChangedEvent
{
    /// <summary>
    /// 变化前的愤怒值
    /// </summary>
    public float OldValue;

    /// <summary>
    /// 变化后的愤怒值
    /// </summary>
    public float NewValue;

    /// <summary>
    /// 变化原因（用于调试）
    /// </summary>
    public string Reason;
}
```

**定义位置**：`Assets/Game/Features/SanityRage/Events/SanityRageEvents.cs`

**发布者**：Sanity/Rage System
**订阅者**：UI System（显示愤怒条变化）

---

### 12.2.2 SanityChangedEvent

理智值变化事件（用于验证标准）：

```csharp
/// <summary>
/// 理智值变化事件
/// 由 Sanity/Rage System 在理智值发生变化时发布
/// </summary>
public struct SanityChangedEvent
{
    /// <summary>
    /// 变化前的理智值
    /// </summary>
    public float OldValue;

    /// <summary>
    /// 变化后的理智值
    /// </summary>
    public float NewValue;

    /// <summary>
    /// 变化原因（用于调试）
    /// </summary>
    public string Reason;
}
```

**定义位置**：`Assets/Game/Features/SanityRage/Events/SanityRageEvents.cs`

**发布者**：Sanity/Rage System
**订阅者**：UI System（显示理智条变化）

---

### 12.3 视觉效果请求事件（已废弃）

> **⚠️ 已废弃 (DEPRECATED)**
>
> 以下事件已废弃，**请改用 ADR-0023 定义的统一 `ScreenEffectRequestEvent` 结构**。
>
> 废弃原因：各系统独立定义效果请求导致重复和不一致。
> 统一后，所有屏幕后处理效果请求均通过 `ScreenEffectRequestEvent` 发送，由 ScreenEffectsManager 集中处理。
>
> **ScreenEffectType 枚举定义位置**：`ScreenEffectType` 枚举定义于 [ADR-0023 §ScreenEffectType](./adr-0023-screen-effects-system-architecture.md#screeneffecttype-枚举)，包含 Vignette、Noise、Saturation、Hue、Blur、**Jitter**、ChromaticAberration、FilmGrain 等值。本文档仅引用该枚举，不重复定义。
>
> **迁移指南**：将原有的独立事件替换为 ScreenEffectRequestEvent，例如：
> - `VignetteRequest{Intensity=X}` → `ScreenEffectRequestEvent{effectType=Vignette, intensity=X, sourceSystem=..., requesterId=...}`
> - `NoiseRequest{Intensity=X}` → `ScreenEffectRequestEvent{effectType=Noise, intensity=X, sourceSystem=..., requesterId=...}`
>
> **例外**：`HUDOverlayOpacityRequest` 属于 UI 层特效，不通过 ScreenEffectsManager 处理，保持不变。
>
> **相关决策**：[ADR-0023: 屏幕特效系统](./adr-0023-screen-effects-system-architecture.md) — 统一效果请求架构，包括 ScreenEffectType 枚举定义

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

### 13.x UI 系统事件订阅规范

所有 UI 系统必须遵循以下订阅原则：

| 原则 | 说明 |
|------|------|
| **订阅时机** | 在 UI Panel 激活时（OnEnable）订阅，禁用时（OnDisable）取消订阅 |
| **订阅管理** | 通过 UIManager 统一管理订阅，避免重复订阅 |
| **事件类型限制** | 只能订阅 ViewModel 变更事件和全局状态事件（暂停、对话开始等） |

**订阅示例**：
```csharp
public class ClueJournalPanel : MonoBehaviour
{
    private void OnEnable()
    {
        EventBus.Subscribe<ClueDiscoveredEvent>(OnClueUnlocked);
        EventBus.Subscribe<GamePausedEvent>(OnGamePaused);
    }

    private void OnDisable()
    {
        EventBus.Unsubscribe<ClueDiscoveredEvent>(OnClueUnlocked);
        EventBus.Unsubscribe<GamePausedEvent>(OnGamePaused);
    }
}
```

**必须订阅的事件**（UI 系统）：
- `ClueDiscoveredEvent` — 线索发现时更新 HUD 计数
- `NPCStateChangedEvent` — NPC 死亡时更新任务标记
- `PsychologicalStateEvent` — 心理状态变化时更新 UI 特效

> **约束**：UI 系统禁止订阅业务逻辑直接事件（如 DamageRequest），只应订阅业务结果的广播事件。

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

### 13.4 PauseMenuOpenedEvent / PauseMenuClosedEvent

暂停菜单事件（定义于 ADR-0015）：

> **命名规范 (2026-04-15 修复)**：事件名称必须包含 `Event` 后缀（见 ADR-0001 §3.14）。
> 原 `PauseMenuOpened`/`PauseMenuClosed` 已废弃，应使用 `PauseMenuOpenedEvent`/`PauseMenuClosedEvent`。

```csharp
/// <summary>
/// 暂停菜单打开事件
/// </summary>
public struct PauseMenuOpenedEvent
{
    /// <summary>
    /// 暂停原因
    /// </summary>
    public PauseReason reason;
}

/// <summary>
/// 暂停菜单关闭事件
/// </summary>
public struct PauseMenuClosedEvent
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
| `NPC_DEATH_SOURCE` | NPC 死亡后才可获取（从尸体搜身获取）。NPC 存活时该线索不存在于 Journal 中，死亡事件触发 Clue 实例创建 |
| `NPC_KNOWLEDGE_BONUS` | NPC 存活期间可获取（对话/审问发现）。线索在 NPC 存活时就存在于 Journal（状态 NOT_DISCOVERED），需玩家主动发现；NPC 死亡后该线索标记为 MISSING |

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

## 19. Event Bus 查询类型（ADR-0018）[已废弃]

> **⚠️ DEPRECATED**: ADR-0018 宣布取消 QueryBus 同步查询模式，所有 Query 类型已标记为废弃。

> **⚠️ 已废弃 (DEPRECATED)**
>
> ADR-0018 宣布取消 QueryBus 同步查询模式，改为纯事件驱动架构。
> 以下 Query 类型已废弃，请使用事件订阅模式替代。
>
> **迁移指南**：
> - `QuerySoundSourceScreenPosition` → 订阅 `NPCSpeakingChangedEvent` + `PlayerPositionUpdatedEvent`，在回调中计算屏幕位置
> - `PerceptionQueryRequest` → 订阅 `WeatherStateChangedEvent` + `LightingStealthBonusChangedEvent`，在回调中缓存感知系数
> - `PerceptionPermissionsQuery` → 使用 LOS System 提供的查询接口或事件

### 19.1 QuerySoundSourceScreenPosition [已废弃]

LOS 声音源屏幕位置查询（定义于 ADR-0018）：

```csharp
/// <summary>
/// 查询声音源的屏幕空间位置
/// 类型：Query（同步查询，通过 QueryBus 返回）
/// [已废弃] 请使用事件订阅模式替代
/// </summary>
[Obsolete("Use event subscription (NPCSpeakingChangedEvent + PlayerPositionUpdatedEvent) instead. Deprecated after ADR-0018.")]
public struct QuerySoundSourceScreenPosition
{
    /// <summary>
    /// 目标 NPC ID
    /// </summary>
    public int npc_id;
}
```

**Handler**：LOSSystem
**Response**：`QuerySoundSourceScreenPositionResponse`（屏幕空间位置和有效性）

```csharp
public struct QuerySoundSourceScreenPositionResponse
{
    public Vector3 ScreenPosition;  // 屏幕空间位置
    public bool IsValid;             // NPC 是否可见
}
```

**定义位置**：`Assets/Game/Core/LOS/Events/QuerySoundSourceScreenPosition.cs`

**使用说明**：
```csharp
// 调用方式（通过 QueryBus）
var response = QueryBus.Instance.Query<QuerySoundSourceScreenPosition, QuerySoundSourceScreenPositionResponse>(
    new QuerySoundSourceScreenPosition { npc_id = npcId }
);
if (response.IsValid)
{
    Vector3 screenPos = response.ScreenPosition;
}
```

---

### 19.1.1 PerceptionQueryRequest / PerceptionQueryResponse [已废弃]

感知查询请求和响应（由 ADR-0003 World Layer 原则定义，用于替代直接方法调用）：

```csharp
/// <summary>
/// 感知查询请求（Query 模式）
/// 外部系统通过 QueryBus 查询 NPC 的感知范围
/// [已废弃] 请使用事件订阅模式（WeatherStateChangedEvent + LightingStealthBonusChangedEvent）替代
/// </summary>
[Obsolete("Use event subscription (WeatherStateChangedEvent + LightingStealthBonusChangedEvent) instead. Deprecated after ADR-0018.")]
public struct PerceptionQueryRequest
{
    /// <summary>
    /// 查询发送者（用于验证和追踪）
    /// </summary>
    public EntityQuery Sender;

    /// <summary>
    /// 基础感知范围
    /// </summary>
    public float BaseRange;
}
```

**Handler**：NPCAIComponent
**Response**：`PerceptionQueryResponse`

```csharp
/// <summary>
/// 感知查询响应
/// </summary>
public struct PerceptionQueryResponse
{
    /// <summary>
    /// 查询发送者（应与请求中的 Sender 一致）
    /// </summary>
    public EntityQuery Sender;

    /// <summary>
    /// 有效感知范围（已应用 Weather/Lighting 系数）
    /// </summary>
    public float EffectiveRange;

    /// <summary>
    /// 查询是否成功
    /// </summary>
    public bool Success;
}
```

**定义位置**：`Assets/Game/Core/NPCAI/Events/PerceptionQuery.cs`

**使用说明**：
```csharp
// 调用方式（通过 QueryBus）
var response = QueryBus.Instance.Query<PerceptionQueryRequest, PerceptionQueryResponse>(
    new PerceptionQueryRequest
    {
        Sender = EntityQuery.FromComponent<PlayerController>(),
        BaseRange = 10f
    }
);
if (response.Success)
{
    float range = response.EffectiveRange;
}
```

**设计背景**：ADR-0003 World Layer 原则规定"不主动调用其他系统，仅通过 Event Bus 广播"。原有的 `NPCAIComponent.QueryEffectivePerceptionRange()` 直接方法调用违反此原则，已改为通过 QueryBus 的事件化查询模式。

---

### 19.1.2 PerceptionPermissionsQuery / PerceptionPermissionsResponse [已废弃]

感知权限查询（用于 GrittyTakedowns 验证玩家是否在 NPC 感知范围内）：

```csharp
/// <summary>
/// 感知权限查询
/// 由 GrittyTakedowns 系统发起，验证玩家是否在 NPC 感知范围内
/// [已废弃] 请使用 LOS System 提供的查询接口或事件模式替代
/// </summary>
[Obsolete("Use LOS System query interface or event-based approach instead. Deprecated after ADR-0018.")]
public struct PerceptionPermissionsQuery
{
    /// <summary>
    /// 目标 NPC ID
    /// </summary>
    public int NPCId;

    /// <summary>
    /// 玩家位置
    /// </summary>
    public Vector3 PlayerPosition;
}
```

**Handler**：LOSSystem
**Response**：`PerceptionPermissionsResponse`

```csharp
/// <summary>
/// 感知权限查询响应
/// </summary>
public struct PerceptionPermissionsResponse
{
    /// <summary>
    /// 是否有感知权限
    /// </summary>
    public bool HasPermission;

    /// <summary>
    /// 权限检查失败原因（用于调试）
    /// </summary>
    public string FailureReason;
}
```

**定义位置**：`Assets/Game/Core/LOS/Events/PerceptionPermissionsQuery.cs`

---

### 19.1.3 ExposureValueChangedEvent

NPC 暴露值变化事件（由 LOS System 发布，NPC AI System 订阅以实现渐进式感知状态转换）：

```csharp
/// <summary>
/// NPC 暴露值变化事件
/// 由 LOS System 发布，NPC AI System 订阅以实现渐进式感知状态转换
///
/// 感知阈值（由 LOSConfigSO 配置）：
/// - SUSPECT 阈值：通常 30-50（玩家暴露但未确认）
/// - SEARCH 阈值：通常 60-80（NPC 开始搜索）
/// - ALERT 阈值：通常 100（玩家被确认发现）
/// </summary>
public struct ExposureValueChangedEvent
{
    /// <summary>
    /// NPC ID
    /// </summary>
    public int npc_id;

    /// <summary>
    /// 当前暴露值 [0-100]
    /// </summary>
    public float exposure_value;

    /// <summary>
    /// 暴露值变化量（正值=增加，负值=衰减）
    /// </summary>
    public float delta;

    /// <summary>
    /// 玩家位置
    /// </summary>
    public Vector3 player_position;
}
```

**定义位置**：`Assets/Game/Core/LOS/Events/ExposureValueChangedEvent.cs`

**发布者**：LOS System
**订阅者**：NPC AI System（用于渐进式感知状态转换：UNDETECTED → SUSPECT → SEARCH）

> **Exposure 值与 AlertState 阈值映射表** [已修复]：
>
> | AlertState | Exposure 值范围（0-100） | 阈值（归一化 0-1） | 说明 |
> |------------|--------------------------|-------------------|------|
> | `UNDETECTED` | 0 | < 0.3 | 玩家未暴露 |
> | `SUSPECT` | 30-50 | 0.3-0.6 | 玩家暴露但未确认 |
> | `SEARCH` | 60-80 | 0.6-0.8 | NPC 开始搜索 |
> | `ALERT` | > 80 | > 0.8 | 玩家被确认发现 |
> | `ESCAPE` / `COMBAT` | 100 | 1.0 | NPC 进入逃离/战斗状态 |
>
> **实现注意**：实际阈值由 `LOSConfigSO` 配置，上述数值为典型默认值。NPC AI 根据 `exposure_value` 与阈值的比较结果进行状态转换。

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

### 22.4 感知系数叠加规则（Weather × Lighting × LOS）

> **来源**：从 ADR-0021、ADR-0022 和 ADR-0007 提取，解决跨 ADR 系数组合的二义性。
> 本节定义 Weather System、Lighting System 和 LOS System 的感知系数如何组合。

**问题背景**：
- Weather System 的 `PerceptionModifier` 包含：`visionDistanceMultiplier`、`visionAngleMultiplier`、`hearingSensitivityMultiplier`、`soundPropagationMultiplier`
- Lighting System 的 `AreaLightingCoefficients` 包含：`shadowStealthMultiplier`、`lightSensitivityMultiplier`、`finalIllumination`
- LOS System 订阅 `LightingStealthBonusChangedEvent` 获取阴影加成（`exposure_multiplier`）

**组合规则**：

> **⚠️ 重要澄清**：表格中的"独立使用"指该系数单独影响结果（不受另一系统影响），而非"不使用"。所有感知系数都会参与最终计算。

| 下游系统 | Weather PerceptionModifier | Lighting AreaLightingCoefficients | 组合方式 |
|---------|---------------------------|----------------------------------|---------|
| NPC AI 感知 | `visionDistanceMultiplier` | — | **独立**：直接作为感知范围乘数，不与 Lighting 组合 |
| NPC AI 感知 | — | `shadowStealthMultiplier` | **独立**：直接作为潜行加成，不与 Weather 组合 |
| NPC AI 感知 | `visionAngleMultiplier` | `lightSensitivityMultiplier` | **乘法组合**：`Weather.visionAngleMultiplier × Lighting.lightSensitivityMultiplier` |
| LOS System | `visionDistanceMultiplier` | — | **独立**：直接作为视野距离乘数 |
| LOS System | `hearingSensitivityMultiplier` | — | **独立**：直接作为听觉灵敏度乘数 |
| LOS System | `soundPropagationMultiplier` | — | **独立**：直接作为声音传播乘数 |
| LOS System | — | `LightingStealthBonusChangedEvent.exposure_multiplier` | **乘法组合**：`Weather.visionDistanceMultiplier × exposure_multiplier` |

> **实现对应**：ADR-0004 NPCController.QueryEffectivePerceptionRange() 的实际计算为：
> `effectiveRange = baseRange * weatherMod.visionDistanceMultiplier * lightMod.shadowStealthMultiplier`
> 表格中的"乘法组合"对应代码中的 `*` 运算，"独立"对应直接赋值不参与乘法。

**LOS System 暴露速度计算公式**：
```
deltaExposure = BaseExposureRate × movementMultiplier × distanceFactor × exposureMultiplier
```

其中：
- `BaseExposureRate`：基础暴露速度（来自 LOS System 配置）
- `movementMultiplier`：玩家移动状态加成（站立/蹲伏/奔跑）
- `distanceFactor`：距离因子（距离越近越高）
- `exposureMultiplier`：`Weather × Lighting.exposure_multiplier`（乘法组合）

**组合公式**：
```
最终感知系数 = Weather_PerceptionModifier × Lighting_AreaLightingCoefficients
最终暴露乘数 = Weather_visionDistanceMultiplier × Lighting_exposure_multiplier
```

**示例**：
```csharp
// Night + PitchBlack + Storm 场景
float finalVisionAngle = weather.visionAngleMultiplier * lighting.lightSensitivityMultiplier;
// = 0.5 (Storm弱) × 0.5 (PitchBlack) = 0.25

float finalStealthBonus = lighting.shadowStealthMultiplier;
// = 2.0 (PitchBlack)，与 Weather 无关

float finalExposureMultiplier = weather.visionDistanceMultiplier * lighting.exposureMultiplier;
// = 0.4 (Storm强) × 0.77 (阴影) = 0.308
```

**注意**：
- `shadowStealthMultiplier` 和 `lightSensitivityMultiplier` 是 Lighting System 独立维护的参数，Weather System 不提供对应的折扣系数
- `exposure_multiplier`（阴影暴露乘数）由 Lighting System 发布，Weather System 通过 `visionDistanceMultiplier` 影响最终值
- LOS System 同时订阅 `WeatherStateChangedEvent`（获取 Weather 感知系数）和 `LightingStealthBonusChangedEvent`（获取阴影加成）

---

## 20. Audio/Haptic 系统类型（ADR-0025）

> **补充日期**：2026-04-14
> **来源**：ADR-0025 评审修复

### HapticType 枚举

触觉反馈类型枚举（用于 PS5 DualSense 等设备的触觉反馈）：

```csharp
/// <summary>
/// 触觉反馈类型枚举
/// 用于 PS5 DualSense 手柄的触控板振动和自适应扳机震动
/// 定义位置：Assets/Game/Features/Audio/Haptics/HapticType.cs
/// </summary>
public enum HapticType
{
    /// <summary>爆炸震动：强烈且短促</summary>
    Explosion,

    /// <summary>处决震动：最大阻力反馈</summary>
    Execution,

    /// <summary>战斗震动：中等强度</summary>
    Combat
}
```

**使用系统**：Audio System（ADR-0025）、HapticFeedbackManager

### HapticRequest 事件

触觉反馈请求事件：

```csharp
/// <summary>
/// 触觉反馈请求事件
/// 由 AudioManager 在适当时机发布，HapticFeedbackManager 处理平台差异
/// </summary>
public struct HapticRequest
{
    /// <summary>
    /// 触觉反馈类型
    /// </summary>
    public HapticType type;

    /// <summary>
    /// 震动强度 [0.0 - 1.0]
    /// </summary>
    public float intensity;
}
```

**发布者**：AudioManager
**订阅者**：HapticFeedbackManager（平台特定实现）

**事件发布时机**（见 ADR-0025 §TriggerScreenEffectSync）：

| SFXCategory | HapticType | Intensity |
|-------------|------------|-----------|
| Explosion | Explosion | 1.0f |
| ExplosionSmall | Explosion | 0.6f |
| StealthKill | Execution | 1.0f |
| EnvironmentKill | Execution | 1.0f |
| Weapon | Combat | 0.7f |

---

## 21. 网络同步相关类型（ADR-0006）

### 21.1 网络连接状态事件

多人游戏中的连接状态变化事件：

```csharp
/// <summary>
/// 玩家断开连接事件
/// </summary>
public struct PlayerDisconnectedEvent
{
    public int PlayerId;
}

/// <summary>
/// 玩家重新连接事件
/// </summary>
public struct PlayerReconnectedEvent
{
    public int PlayerId;
}

/// <summary>
/// 重新连接失败事件
/// </summary>
public struct ReconnectFailedEvent
{
    public int PlayerId;
}

/// <summary>
/// 重新连接超时事件
/// </summary>
public struct ReconnectTimeoutEvent
{
    public int PlayerId;
}

/// <summary>
/// 主机迁移开始事件
/// </summary>
public struct HostMigrationStartedEvent
{
    public int OldHostId;
    public int NewHostId;
}

/// <summary>
/// 【新增 v2.4】玩家作弊检测事件
/// 当反作弊系统检测到玩家作弊行为时发布
/// </summary>
public struct PlayerCheatDetectedEvent
{
    /// <summary>
    /// 玩家 ID
    /// </summary>
    public int PlayerId;

    /// <summary>
    /// 作弊类型
    /// </summary>
    public CheatType CheatType;

    /// <summary>
    /// 检测时间戳
    /// </summary>
    public float Timestamp;
}

/// <summary>
/// 作弊类型枚举
/// </summary>
public enum CheatType
{
    HEALTH_TAMPERING,    // 生命值篡改
    TELEPORT,            // 传送作弊
    SPEED_HACK,          // 加速作弊
    AIMBOT_SUSPECTED,    // 可疑自瞄
    DAMAGE_AMPLIFICATION // 伤害放大
}
```

**定义位置**：`Assets/Game/Infrastructure/Network/Events/NetworkEvents.cs`

**发布者**：Network System（反作弊模块）
**订阅者**：UI System（显示警告）、Game System（处理作弊响应）

---

## 22. 修改日志

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
| 2026-04-14 | 2.3.0 | 新增 CombatStateChangedEvent（§5.10）、KeywordCapturedEvent（§5.11）、RageChangedEvent（§12.2.1）、SanityChangedEvent（§12.2.2）；统一 NoiseEvent→NoiseMadeEvent、WeaponAwareness→WeaponAwarenessEvent 命名（带别名兼容）；新增网络事件（§21：PlayerDisconnectedEvent 等）；修复 ADR-0017 PlayerDamagedEvent 结构描述错误；ADR-0018 状态更新为 Accepted | 架构师 Agent |
| 2026-04-15 | 2.4.0 | ADR评审修复：新增 NPCIdentityConfirmedEvent（§5.11.1）、ExposureValueChangedEvent（§19.1.3）、PlayerCheatDetectedEvent（§21）；NPCStateChangedEvent 新增 has_witness/witness_distance 字段（§5.2）；更新 ADR-0005 BuildSaveData 为 QueryBus 模式；修复 ADR-0017 SOUL_SPLIT 抖动描述矛盾；更新 ADR-0015 引用章节（ScreenEffectRequestEvent 改引用 ADR-0023） | 架构师 Agent |
