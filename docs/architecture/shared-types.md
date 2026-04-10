# 共享类型定义 (Shared Types)

> **版本**: 1.2.0
> **创建日期**: 2026-04-10
> **状态**: APPROVED
> **维护者**: 架构师
> **更新日期**: 2026-04-10 (v1.2.0 — 修复：NPCSizeCategory multiplier 与 ADR-0011 对齐、Bravery 范围修正为 [1,10]、新增 WorldState↔HealthState 同步协议)

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
    // 热武器
    Pistol,
    Rifle,
    Shotgun,
    Grenade,
    C4,
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

        // 添加新锁
        _activeLocks.Add(new ActiveLock
        {
            Requester = requester,
            LockType = lockType,
            AcquireTime = Time.time,
            Timeout = expectedDuration + 1f
        });

        PlayerController.Instance.SetInputEnabled(false);
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

        _activeLocks.Remove(lockToRelease);

        if (_activeLocks.Count == 0)
            PlayerController.Instance.SetInputEnabled(true);

        return true;
    }

    /// <summary>
    /// 紧急释放（玩家死亡等情况）
    /// </summary>
    public void ForceReleaseAll()
    {
        _activeLocks.Clear();
        PlayerController.Instance.SetInputEnabled(true);
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

NPC 体型分类，用于 Gritty Takedowns 系统的捆绑时间计算：

```csharp
public enum NPCSizeCategory
{
    Small,   // 体型小，捆绑时间加成为 0.0s
    Medium,  // 体型中等，捆绑时间加成为 0.5s
    Large    // 体型大，捆绑时间加成为 1.0s
}
```

**定义位置**：`Assets/Game/Core/NPCAI/NPCSizeCategory.cs`

**说明**：
- 此类型由 NPC AI System 定义（Core Layer），Gritty Takedowns 通过 NPCController.QuerySizeCategory() 获取
- 体型加成用于计算捆绑时间，详见 ADR-0011 §TieUpCalculator
- **与 TuningSO 的关系**：TuningSO 中的 baseTieUpTime (4.0s) 是基准时间，体型加成为调整系数

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

### 5.8 AlertTrigger

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

### 7.1 PlayerMovementState

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

### 7.2 PlayerMovementStateChangedEvent

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

### 7.3 NoiseType

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

### 7.4 NoiseEvent

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

### 7.5 PlayerDamagedEvent

玩家受伤事件：

```csharp
public struct PlayerDamagedEvent
{
    public int player_id;
    public DamageType damage_type;
    public HitLocation hit_location;
    public int source_entity_id;
}
```

**定义位置**：`Assets/Game/Foundation/Shared/Events/PlayerDamagedEvent.cs`

**发布者**：Health System
**订阅者**：Gritty Takedowns（打断交互）

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
    public int choice_index;
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

## 11. 修改日志

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-10 | 1.0.0 | 初稿创建，统一 DamageRequest、ExplosionEvent、WeaponQuery、ActionLockSystem 等跨 ADR 类型 | 架构师 Agent |
| 2026-04-10 | 1.2.0 | 修复：CheckpointRestoreRequestEvent 补充 arrest_location 字段、LoadingScreenRequestEvent 补充 source_location 字段（与 ADR-0012 保持一致） | 架构师 Agent |
| 2026-04-10 | 1.1.0 | 新增 NPC AI/Health/Player/Weapon/GrittyTakedowns/WorldMap 完整类型定义 | 架构师 Agent |
| 2026-04-10 | 1.1.1 | 修复：AreaClearedEvent 补充完整字段（area_id, enemy_count, is_full_clear）、新增 LoadCompletedEvent（Event Bus ICD v1.2.3） | 架构师 Agent |
| 2026-04-10 | 1.2.0 | 修复：NPCSizeCategory multiplier 与 ADR-0011 §TieUpCalculator 对齐；Bravery 范围修正为 [1,10] 并明确与贿赂/欺骗的关系；新增 §6.1.1 WorldState↔HealthState 同步协议 | 架构师 Agent |
