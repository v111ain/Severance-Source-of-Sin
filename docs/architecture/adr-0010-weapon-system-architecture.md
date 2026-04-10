# ADR-0010: 武器系统 (Weapon System) 架构决策

## Status
**Proposed**（依赖 shared-types.md 的 APPROVED 类型定义）

## Date
2026-04-10

## Last Updated
2026-04-10

## Context

### Problem Statement

武器系统是《断绝：罪恶之源》"环境即武器"核心设计理念的系统实现。游戏中的所有武器——无论是传统的枪械还是场景中的砖块、灭火器、电线——都需要通过统一的架构进行管理。系统需要处理：

1. **武器数据模型**：定义所有武器的属性（伤害类型、穿透值、有效距离、动画标签）
2. **环境物件武器化**：将环境交互系统中的可拾取物件赋予武器能力
3. **武器状态追踪**：管理玩家当前持有的武器、可用物件列表、冷却状态
4. **伤害参数输出**：向 Health & Lethality 系统提供标准化的 `DamageRequest`
5. **爆炸物事件广播**：向 Health 系统发送爆炸位置参数，由 Health 系统执行伤害计算

### Constraints

- **设计约束**：武器系统不维护独立的环境物件状态，环境交互系统作为唯一数据源
- **爆炸物约束**：C4 和手榴弹使用缩小的 blast_radius（3m/2m），lethal_ratio = 0.3
- **热武器约束**：玩家只能有一把热武器处于 `Equipped` 状态，环境物件可同时持有多个
- **职责约束**：爆炸伤害计算由 Health System 执行（`ExplosionHandler` 归属 Health System）
- **平台约束**：PS5 平台需要支持手柄震动反馈（Xbox/PlayStation/Switch Pro）

### Requirements

- **必须**：定义 WeaponData 组件化数据模型（DamageComponent、RangeComponent、AmmoComponent、ExplosiveComponent）
- **必须**：定义武器状态机（Stored/Equipped/Holstered/Used/Empty/Reloading）
- **必须**：定义与环境交互系统的事件订阅关系（ObjectStateChangedEvent）
- **必须**：定义爆炸物事件接口（发送给 Health 系统，由 Health 执行计算）
- **必须**：定义与 Health 系统、Gritty Takedowns 系统的接口契约
- **必须**：遵循 ADR-0003 系统分层定义（Weapon System 属于 Feature Layer）

---

## Decision

### 架构决策

采用**组件化数据模型 + 事件驱动状态同步 + 爆炸计算委托**架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Weapon System 架构                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    WeaponSystem (Feature Layer)                       │   │
│  │  - 管理所有武器数据模板（WeaponTemplateLibrary）                     │   │
│  │  - 管理玩家持有武器的状态追踪                                       │   │
│  │  - 处理武器切换逻辑（Tab/Q/E/X）                                    │   │
│  │  - 订阅环境交互系统事件同步物件状态                                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│          ┌─────────────────────────┴─────────────────────────┐          │
│          ▼                                                   ▼          │
│  ┌───────────────────┐                           ┌───────────────────┐   │
│  │  WeaponData       │                           │  WeaponStateMachine│   │
│  │  (组件化数据模型)  │                           │  (玩家持有武器状态) │   │
│  │                   │                           │                   │   │
│  │ - DamageComponent │                           │ Stored          │   │
│  │ - RangeComponent  │                           │ Equipped       │   │
│  │ - AmmoComponent   │                           │ Holstered      │   │
│  │ - ExplosiveComp.  │                           │ Used           │   │
│  │ - AnimationTags   │                           │ Empty          │   │
│  │                   │                           │ Reloading      │   │
│  └───────────────────┘                           └───────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Event Subscriptions                             │   │
│  │  - ObjectStateChangedEvent (环境交互系统) → 同步物件状态            │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Event Publications                              │   │
│  │  - WeaponStateChangedEvent → UI系统                                │   │
│  │  - DamageRequest → Health系统（伤害计算由Health执行）                │   │
│  │  - ExplosionEvent → Health系统（ExplosionHandler在Health System）   │   │
│  │  - WeaponAwareness → NPC AI系统                                    │   │
│  │  - WeaponUsedEvent → 音频系统                                      │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ⚠️ 爆炸伤害计算由 Health System 的 ExplosionHandler 执行                │
│     Weapon System 仅发送位置参数，不进行伤害计算                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

### 1. WeaponData 组件化数据模型

> **⚠️ 初始化时序说明（已知限制）**：WeaponData 作为 ScriptableObject，其引用字段在第一次访问时可能尚未初始化。
> WeaponTemplateLibrary 通过 `Awake()` 构建映射表，但 ScriptableObject 的加载顺序在不同平台可能不一致。
>
> **Mitigation**：所有查询方法都包含防御性检查（null 返回而非异常），确保安全返回。
> 这是 Unity ScriptableObject 系统的已知限制，接受为合理的工程权衡。
>
> **验证方式**：通过 `WeaponTemplateLibrary.RebuildLookupMaps()` Editor menu item 手动重建映射表，确保编辑器下数据一致。



```csharp
// WeaponData.cs
[CreateAssetMenu(menuName = "Game/Weapons/WeaponData")]
public class WeaponData : ScriptableObject
{
    public string weapon_id;
    public string display_name;
    public WeaponCategory category;  // Environmental / HotWeapon（定义见 shared-types.md §3.1）

    // 组件插槽（按需组合）
    public DamageComponent damage_component;
    public RangeComponent range_component;
    public AmmoComponent ammo_component;           // 可选：热武器
    public ExplosiveComponent explosive_component; // 可选：爆炸物
    public AnimationTagsComponent animation_tags;

    public bool is_consumable;
    public float weight;
}

// 伤害组件
[System.Serializable]
public class DamageComponent
{
    public DamageType damage_type;  // LETHAL / BLUNT（定义见 shared-types.md §2.1）
    public int penetration;          // 1-10（来自Health系统穿透规则）
    public float base_damage;        // 基础伤害倍率
}

// 范围组件
[System.Serializable]
public class RangeComponent
{
    public RangeType range_type;     // 定义见 shared-types.md §3.1.1
    public float effective_range;    // 有效距离（米）
    public float throw_range;        // 投掷距离（可选）
}

// 弹药组件（热武器）
[System.Serializable]
public class AmmoComponent
{
    public AmmoType ammo_type;       // 定义见 shared-types.md §3.1.2
    public int ammo_capacity;

    [SerializeField]
    [Tooltip("当前弹药数（通过 CurrentAmmo 属性访问）")]
    private int _currentAmmo;         // 私有字段，公开属性访问

    /// <summary>
    /// 当前弹药数（属性访问器，防止外部意外修改）
    /// </summary>
    /// <remarks>
    /// <b>线程安全说明</b>：此属性在主线程独占环境下是安全的。
    /// 如果游戏存在 Job System 多线程访问需求，请使用以下方案之一：
    /// 1. 使用 <c>System.Threading.Interlocked</c> 包装整数操作
    /// 2. 使用 <c>volatile</c> 声明 <c>_currentAmmo</c> 字段
    /// Unity 主线程独占访问场景下无需修改。
    /// </remarks>
    public int CurrentAmmo
    {
        get => _currentAmmo;
        private set => _currentAmmo = Mathf.Clamp(value, 0, ammo_capacity);
    }

    /// <summary>
    /// 消耗弹药（返回实际消耗数量）
    /// </summary>
    public int Consume(int amount)
    {
        int consumed = Mathf.Min(amount, _currentAmmo);
        _currentAmmo -= consumed;
        return consumed;
    }

    /// <summary>
    /// 填充弹药
    /// </summary>
    public void Refill(int amount = -1)
    {
        _currentAmmo = amount < 0 ? ammo_capacity : Mathf.Min(amount, ammo_capacity);
    }

    public float reload_time;       // 秒
}

// 爆炸组件（爆炸物）
[System.Serializable]
public class ExplosiveComponent
{
    public float blast_radius;           // 爆炸半径（米）
    public float lethal_radius_ratio;    // 致死半径比例（默认0.3）
    public DetonationType detonation_type; // 定义见 shared-types.md §3.1.3
    public bool is_controllable;
}

// 动画标签组件
[System.Serializable]
public class AnimationTagsComponent
{
    public string[] tags;  // 引用环境交互系统的 DESTRUCT_* / WEAPON_MELEE_* 标签
}

// RangeType、AmmoType、DetonationType 定义于 shared-types.md §3.1.1-3.1.3
// WeaponCategory 定义于 shared-types.md §3.1

### 2. WeaponTemplateLibrary

```csharp
// WeaponTemplateLibrary.cs
public class WeaponTemplateLibrary : ScriptableObject
{
    [Header("Dynamic Weapon Lists (Recommended)")]
    [Tooltip("推荐使用列表方式，新增武器只需在 Inspector 中添加，无需修改代码")]
    public List<WeaponData> hotWeapons;
    public List<WeaponData> environmentalWeapons;

    [Header("Legacy Field Assignments (Deprecated)")]
    [Tooltip("保留以兼容旧版数据迁移。新项目请使用上面的列表方式。")]
    public WeaponData brick;
    public WeaponData metal_pipe;
    public WeaponData fire_extinguisher;
    public WeaponData glass_bulb;
    public WeaponData wire;
    public WeaponData broken_glass;
    public WeaponData pistol;
    public WeaponData rifle;
    public WeaponData shotgun;
    public WeaponData grenade;
    public WeaponData c4;

    // 按 ID 查询模板（返回 null 如果不存在，不会抛异常）
    public WeaponData GetTemplate(string weapon_id)
    {
        if (string.IsNullOrEmpty(weapon_id)) return null;

        // 防御性检查：确保映射表已初始化
        if (!_initialized)
            BuildLookupMaps();

        // 启动时构建映射表，运行时 O(1) 查询
        // 热武器
        if (_hotWeaponMap.TryGetValue(weapon_id, out var hotWeapon))
            return hotWeapon;

        // 环境物件
        if (_environmentalMap.TryGetValue(weapon_id, out var envWeapon))
            return envWeapon;

        return null;
    }

    // 按类别映射到 WeaponData（供环境交互系统调用）
    public WeaponData GetTemplateByCategory(ObjectCategory category)
    {
        // 使用映射表查找（支持新旧两种数据源）
        var weaponId = CategoryToWeaponId(category);
        return weaponId != null ? GetTemplate(weaponId) : null;
    }

    /// <summary>
    /// 将 ObjectCategory 转换为对应默认武器 ID
    /// </summary>
    private string CategoryToWeaponId(ObjectCategory category)
    {
        return category switch
        {
            ObjectCategory.Brick => "brick",
            ObjectCategory.MetalPipe => "metal_pipe",
            ObjectCategory.FireExtinguisher => "fire_extinguisher",
            ObjectCategory.GlassBulb => "glass_bulb",
            ObjectCategory.Wire => "wire",
            ObjectCategory.BrokenGlass => "broken_glass",
            ObjectCategory.Pistol => "pistol",
            ObjectCategory.Rifle => "rifle",
            ObjectCategory.Shotgun => "shotgun",
            ObjectCategory.Grenade => "grenade",
            ObjectCategory.C4 => "c4",
            _ => null
        };
    }

    // 启动时构建 ID → WeaponData 映射表
    private Dictionary<string, WeaponData> _hotWeaponMap = new();
    private Dictionary<string, WeaponData> _environmentalMap = new();
    private bool _initialized = false;

    private void Awake()
    {
        BuildLookupMaps();
    }

#if UNITY_EDITOR
    [ContextMenu("Rebuild Lookup Maps")]
    private void RebuildLookupMaps()
    {
        BuildLookupMaps();
    }

    private void OnValidate()
    {
        // Editor 模式下保持映射表最新
        // 使用 ContextMenu 作为主要重建方式（更可靠）
        // OnValidate 作为备份，在 Inspector 修改时触发
        UnityEditor.EditorApplication.delayCall += () =>
        {
            if (this != null)
                BuildLookupMaps();
        };
    }
#endif

    private void BuildLookupMaps()
    {
        _hotWeaponMap.Clear();
        _environmentalMap.Clear();

        void Register(WeaponData data)
        {
            if (data == null) return;
            if (data.category == WeaponCategory.HotWeapon)
                _hotWeaponMap[data.weapon_id] = data;
            else
                _environmentalMap[data.weapon_id] = data;
        }

        // 优先使用列表方式注册（推荐）
        if (hotWeapons != null)
            foreach (var w in hotWeapons)
                Register(w);

        if (environmentalWeapons != null)
            foreach (var w in environmentalWeapons)
                Register(w);

        // 兼容旧版字段注册（如果列表为空则回退到字段）
        if ((hotWeapons == null || hotWeapons.Count == 0) ||
            (environmentalWeapons == null || environmentalWeapons.Count == 0))
        {
            Register(pistol);
            Register(rifle);
            Register(shotgun);
            Register(grenade);
            Register(c4);
            Register(brick);
            Register(metal_pipe);
            Register(fire_extinguisher);
            Register(glass_bulb);
            Register(wire);
            Register(broken_glass);
        }

        _initialized = true;
    }

    // 调试接口：列出所有已注册模板
    public IEnumerable<string> GetAllTemplateIds()
    {
        return _hotWeaponMap.Keys.Concat(_environmentalMap.Keys);
    }
}
```

### 3. 武器状态机

> **注意**：`WeaponState` 枚举统一定义在 `shared-types.md §3.3`，禁止在此文档中重复定义。

```csharp
// WeaponStateMachine.cs
public class WeaponStateMachine
{
    private Dictionary<string, WeaponState> _weaponStates = new();
    private string _equippedHotWeaponId = string.Empty;  // 确保只有一把热武器处于 Equipped

    /// <summary>
    /// 判断指定武器是否为热武器
    /// </summary>
    /// <param name="weapon_id">武器 ID</param>
    /// <param name="library">
    /// 武器模板库引用。调用方需确保传入非 null 的 library，
    /// 否则热武器独占性检查将无法执行（hasWeapon 视为 false）。
    /// </param>
    private bool IsHotWeapon(string weapon_id, WeaponTemplateLibrary library)
    {
        var template = library?.GetTemplate(weapon_id);
        return template?.category == WeaponCategory.HotWeapon;
    }

    /// <summary>
    /// 设置武器状态（无热武器独占性检查的重载）
    /// </summary>
    /// <remarks>
    /// 仅在不涉及热武器切换时使用（如环境物件状态更新）。
    /// 热武器切换请使用 <see cref="SetState(string, WeaponState, WeaponTemplateLibrary)"/>。
    /// </remarks>
    public void SetState(string weapon_id, WeaponState newState)
    {
        SetState(weapon_id, newState, null);
    }

    /// <summary>
    /// 设置武器状态（带热武器独占性检查）
    /// </summary>
    /// <param name="weapon_id">武器 ID</param>
    /// <param name="newState">新状态</param>
    /// <param name="library">
    /// 武器模板库引用（用于热武器独占性检查）。
    /// 传入 null 时热武器检查将被跳过。
    /// </param>
    public void SetState(string weapon_id, WeaponState newState, WeaponTemplateLibrary library)
    {
        var oldState = _weaponStates.GetValueOrDefault(weapon_id, WeaponState.Stored);

        // 热武器独占性检查
        if (newState == WeaponState.Equipped && IsHotWeapon(weapon_id, library))
        {
            // 将之前的热武器设为 Holstered
            if (!string.IsNullOrEmpty(_equippedHotWeaponId) && _equippedHotWeaponId != weapon_id)
            {
                _weaponStates[_equippedHotWeaponId] = WeaponState.Holstered;
                EventBus.Instance.Publish(new WeaponStateChangedEvent(_equippedHotWeaponId, WeaponState.Holstered, newState));
            }
            _equippedHotWeaponId = weapon_id;
        }

        _weaponStates[weapon_id] = newState;

        if (oldState != newState)
        {
            EventBus.Instance.Publish(new WeaponStateChangedEvent(weapon_id, oldState, newState));
        }
    }

    public WeaponState GetState(string weapon_id)
        => _weaponStates.GetValueOrDefault(weapon_id, WeaponState.Stored);
}
```

### 4. 爆炸事件发送（伤害计算由 Health System 执行）

```csharp
// WeaponSystem.cs - 发送爆炸事件
public void TriggerExplosion(string weapon_id, Vector3 position)
{
    var weaponData = GetWeaponData(weapon_id);
    if (weaponData?.explosive_component == null) return;

    var explosive = weaponData.explosive_component;
    var damage = weaponData.damage_component;

    // Weapon System 发送完整爆炸参数，伤害计算由 Health System 的 ExplosionHandler 执行
    // base_damage 来源：统一使用 DamageComponent 的 base_damage（由设计文档定义，如 C4=150，手榴弹=80）
    EventBus.Instance.Publish(new ExplosionEvent
    {
        position = position,
        radius = explosive.blast_radius,
        lethal_ratio = explosive.lethal_radius_ratio,
        base_damage = damage.base_damage  // 统一使用 DamageComponent 的 base_damage
    });

    // 触发音效和视觉反馈
    EventBus.Instance.Publish(new WeaponUsedEvent
    {
        weapon_id = weapon_id,
        usage_type = "explosion"
    });
}
```

**⚠️ base_damage 来源说明**：
- `ExplosionEvent.base_damage` 值来源于 `DamageComponent.base_damage`
- `ExplosiveComponent` 不单独存储 `base_damage`，只存储爆炸参数（blast_radius、lethal_ratio）
- 设计确保 `DamageComponent.base_damage` 与爆炸物配置同步

### 5. 投掷物命中判定

```csharp
// ThrowableHandler.cs
public class ThrowableHandler
{
    // 硬编码默认值（fallback）
    private const float DEFAULT_MIN_HIT_CHANCE = 0.3f;

    /// <summary>
    /// 计算投掷物命中概率
    /// </summary>
    /// <param name="range">实际投掷距离</param>
    /// <param name="melee_range">近战判定距离</param>
    /// <param name="throw_range">最大投掷距离</param>
    /// <param name="min_hit_chance">最小命中概率（可由 TuningSO 配置，传 null 使用默认值）</param>
    /// <returns>命中概率 [0, 1]</returns>
    public float CalculateHitChance(float range, float melee_range, float throw_range, float? min_hit_chance = null)
    {
        if (range <= melee_range)
            return 1.0f;  // 近战范围内必中

        float minChance = min_hit_chance ?? DEFAULT_MIN_HIT_CHANCE;
        return Mathf.Clamp(
            1 - (range - melee_range) / (throw_range - melee_range),
            minChance,
            1.0f);
    }
}
```

> **注意**：`min_hit_chance` 优先使用 TuningSO 配置值（见 §8），未配置时使用默认值 0.3f。

### 5.1 投掷物轨迹计算

```csharp
// ThrowableTrajectory.cs
public class ThrowableTrajectory
{
    private const float GRAVITY = -9.81f;  // 重力加速度（米/秒²）

    /// <summary>
    /// 计算投掷物轨迹上的位置
    /// </summary>
    /// <param name="origin">投掷起始位置</param>
    /// <param name="direction">投掷方向（单位向量）</param>
    /// <param name="speed">投掷速度（米/秒）</param>
    /// <param name="time">飞行时间（秒）</param>
    /// <returns>时间为 t 时投掷物所在位置</returns>
    public static Vector3 CalculatePosition(Vector3 origin, Vector3 direction, float speed, float time)
    {
        // 水平位移：direction * speed * time
        // 垂直位移：0.5f * GRAVITY * time²
        return origin + direction * speed * time + Vector3.up * 0.5f * GRAVITY * time * time;
    }

    /// <summary>
    /// 估算投掷飞行时间
    /// </summary>
    /// <param name="distance">水平距离（米）</param>
    /// <param name="speed">投掷速度（米/秒）</param>
    /// <returns>飞行时间（秒）</returns>
    public static float EstimateFlightTime(float distance, float speed)
    {
        // 简化模型：假设抛射角为 45° 时水平距离最大
        // v * cos(45°) * t = distance → t = distance / (v * 0.707)
        return distance / (speed * 0.707f);
    }

    /// <summary>
    /// 计算投掷命中点（射线检测）
    /// </summary>
    /// <param name="origin">投掷起始位置</param>
    /// <param name="direction">投掷方向（单位向量）</param>
    /// <param name="speed">投掷速度（米/秒）</param>
    /// <param name="maxTime">最大飞行时间（秒）</param>
    /// <param name="layerMask">碰撞检测层掩码</param>
    /// <returns>命中点位置（未命中返回 null）</returns>
    public static Vector3? RaycastThrow(Vector3 origin, Vector3 direction, float speed, float maxTime, LayerMask layerMask)
    {
        // 自适应步长：速度越快，步长越小，确保高速投掷物不漏检
        // 最小步长 10ms，最大步长 50ms
        float step = Mathf.Clamp(0.05f / (speed / 5f), 0.01f, 0.05f);
        Vector3 prevPos = origin;

        for (float t = step; t <= maxTime; t += step)
        {
            Vector3 currentPos = CalculatePosition(origin, direction, speed, t);

            // 射线检测：从前一个位置到当前位置
            if (Physics.Raycast(prevPos, currentPos - prevPos, out RaycastHit hit, Vector3.Distance(prevPos, currentPos), layerMask))
            {
                return hit.point;
            }

            // 如果 Y 坐标变为负值，说明已经落地
            if (currentPos.y < 0)
                break;

            prevPos = currentPos;
        }

        return null;
    }
}
```

> **注意**：投掷物轨迹计算由 Weapon System 维护，供 ThrowableHandler 在计算命中概率时使用。
> 轨迹计算不涉及伤害判定，伤害由 Health System 的 DamageRequest 处理。

### 6. 事件接口定义

```csharp
// ===== 共享类型（定义见 shared-types.md）=====
// - DamageRequest          → shared-types.md §2.3
// - ExplosionEvent         → shared-types.md §2.4
// - WeaponCategory         → shared-types.md §3.1
// - ObjectCategory         → shared-types.md §3.2
// - WeaponState            → shared-types.md §3.3
// - WeaponQueryRequest / WeaponQueryResponse → shared-types.md §3.8

// ===== 订阅的事件（来自环境交互系统）=====
// ObjectStateChangedEvent 定义见 shared-types.md §3.5
// ObjectState 枚举值：Spawned / Available / Held / Used / Depleted / Dropped
// 注意：ADR-0010 早期版本曾使用 "PickedUp"，已废弃，统一使用 shared-types.md 定义

// ===== 发布的事件 =====
// WeaponStateChangedEvent 定义见 shared-types.md §3.4

public struct WeaponAwareness
{
    public string weapon_id;
    public Vector3 position;
    public WeaponCategory weapon_type;  // 定义见 shared-types.md §3.1
}

/// <summary>
/// WeaponAwareness 事件订阅方：NPC AI System
/// NPC AI 订阅此事件用于：
/// - 感知附近环境武器的存在（影响 NPC 警戒决策）
/// - 环境处决威胁评估
/// 详见 ADR-0004 的 PerceptionComponent 和 FactionNetworkComponent
/// </summary>

public struct WeaponUsedEvent
{
    public string weapon_id;
    public string usage_type;  // "fire", "throw", "melee", "explosion"
}
```

### 7. Unity 项目结构（Feature Layer）

```
Assets/Game/
├── Features/WeaponSystem/              # Feature Layer（遵循 ADR-0003）
│   ├── WeaponSystem.cs                 # 主系统管理器
│   ├── WeaponData.cs                   # 武器数据 ScriptableObject
│   ├── WeaponTemplateLibrary.cs         # 武器模板库
│   ├── WeaponStateMachine.cs            # 武器状态机
│   │   └── WeaponState 枚举移至 shared-types.md §3.3
│   ├── Components/
│   │   ├── DamageComponent.cs
│   │   ├── RangeComponent.cs
│   │   ├── AmmoComponent.cs
│   │   ├── ExplosiveComponent.cs
│   │   └── AnimationTagsComponent.cs
│   ├── Handlers/
│   │   └── ThrowableHandler.cs           # 投掷物命中判定
│   ├── Events/
│   │   ├── WeaponEvents.cs              # 所有武器相关事件定义
│   │   └── WeaponEventIds.cs            # 事件 ID 常量
│   └── Config/
│       └── WeaponTuningSO.cs            # 调参配置
```

### 8. 手柄震动反馈抽象层

```csharp
// HapticFeedbackConfig.cs
// 平台适配层抽象，支持 PS5/Xbox/Switch Pro/PC
using UnityEngine;

public interface IHapticFeedback
{
    void PlayFeedback(HapticPattern pattern, float intensity);
}

/// <summary>
/// 手柄震动反馈单例实现（可在运行时替换为平台特定实现）
/// </summary>
public class HapticFeedbackManager : MonoBehaviour, IHapticFeedback
{
    public static IHapticFeedback Instance { get; private set; }

    private void Awake()
    {
        // 确保只有一个实例存在
        if (Instance != null)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    private void OnEnable()
    {
        // Scene 切换时自动重新注册
        UnityEngine.SceneManagement.SceneManager.sceneLoaded += OnSceneLoaded;
    }

    private void OnDisable()
    {
        UnityEngine.SceneManagement.SceneManager.sceneLoaded -= OnSceneLoaded;
    }

    private void OnSceneLoaded(UnityEngine.SceneManagement.Scene scene, UnityEngine.SceneManagement.LoadSceneMode mode)
    {
        // Scene 切换后确保 Instance 仍然有效
        // 如果当前实例被销毁但 Instance 未清空（理论上不会发生），进行修复
        if (this != null && Instance == null)
        {
            Instance = this;
        }
    }

    public void PlayFeedback(HapticPattern pattern, float intensity)
    {
        // 平台特定实现（示例：PS5 DualSense）
        if (Application.platform == RuntimePlatform.PS5)
        {
            // PS5-specific haptic call
        }
        // 其他平台...
    }
}

public enum HapticPattern
{
    None,
    WeaponFire,        // 开火震动
    WeaponReload,      // 换弹震动
    WeaponEmpty,       // 空仓震动
    WeaponImpact,      // 命中反馈
    Explosion,         // 爆炸反馈
    InteractionPrompt  // 交互提示
}

public class HapticFeedbackConfig : ScriptableObject
{
    // 平台分级震动强度
    [Header("PS5 DualSense")]
    public float ps5FireIntensity = 0.8f;
    public float ps5ExplosionIntensity = 1.0f;

    [Header("Xbox Controller")]
    public float xboxFireIntensity = 0.6f;
    public float xboxExplosionIntensity = 0.9f;

    [Header("Switch Pro")]
    public float switchFireIntensity = 0.5f;
    public float switchExplosionIntensity = 0.8f;

    public float GetIntensity(HapticPattern pattern, RuntimePlatform platform)
    {
        return (pattern, platform) switch
        {
            (HapticPattern.Explosion, RuntimePlatform.PS5) => ps5ExplosionIntensity,
            (HapticPattern.Explosion, RuntimePlatform.XboxOne) => xboxExplosionIntensity,
            (HapticPattern.Explosion, RuntimePlatform.Switch) => switchExplosionIntensity,
            (HapticPattern.WeaponFire, RuntimePlatform.PS5) => ps5FireIntensity,
            (HapticPattern.WeaponFire, RuntimePlatform.XboxOne) => xboxFireIntensity,
            (HapticPattern.WeaponFire, RuntimePlatform.Switch) => switchFireIntensity,
            _ => 0.5f
        };
    }
}
```

// WeaponSystem.cs - 调用示例
public void TriggerHaptic(HapticPattern pattern)
{
    var platform = Application.platform;
    var intensity = _hapticConfig.GetIntensity(pattern, platform);
    IHapticFeedback.Instance?.PlayFeedback(pattern, intensity);
}
```

**相关文件**：`Assets/Game/Features/WeaponSystem/Config/HapticFeedbackConfig.cs`

---

## Alternatives Considered

### Alternative 1: 武器系统自行计算爆炸伤害

- **描述**：Weapon System 包含 `ExplosionHandler`，自行计算范围内实体受到的伤害
- **Pros**：Weapon System 自包含，调试直观
- **Cons**：
  - 违反单一职责原则：伤害计算属于 Health System
  - 与 ADR-0008 冲突（ADR-0008 已定义 `ExplosionHandler` 在 Health System）
  - 重复实现空间分区逻辑
- **拒绝理由**：
  - ADR-0008 明确将 `ExplosionHandler` 放在 Health System
  - 伤害计算统一由 Health System 执行是架构共识

### Alternative 2: 统一武器数据模型（非组件化）

- **描述**：使用单一的 WeaponData 类包含所有属性（damage、range、ammo、explosive），无组件化设计
- **Pros**：数据结构简单，查找属性快
- **Cons**：
  - 热武器和爆炸物有大量空字段，浪费内存
  - 新增武器类型需要修改核心类
- **拒绝理由**：
  - 组件化设计更符合 SOLID 原则，扩展性好
  - 环境物件（无弹药/无爆炸）和热武器属性差异大，组件化更灵活

---

## Consequences

### Positive

- **组件化扩展性**：新增武器类型只需组合现有组件，无需修改核心类
- **事件驱动解耦**：环境交互系统作为唯一数据源，避免状态不一致
- **爆炸物战术深度**：混合型伤害设计（近距离秒杀/远距离硬直）增加了战术选择
- **热武器独占性**：确保玩家不能同时持有多把热武器，保持游戏节奏
- **职责清晰**：爆炸伤害计算由 Health System 统一执行，避免重复实现

### Negative

- **组件序列化的复杂性**：Unity Inspector 中组件化数据显示不如单一类直观
- **爆炸物调参敏感性**：lethal_ratio 和 blast_radius 需要精细平衡
- **手柄震动适配工作量**：多平台震动 API 差异需要适配层
- **WeaponStateMachine 依赖 WeaponTemplateLibrary**：热武器独占性检查需要传入 library 引用，增加了调用复杂度

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **爆炸物过于强力** | blast_radius 或 lethal_ratio 过大导致爆炸物替代潜行 | 缩小 blast_radius（C4 3m，手榴弹 2m）；Playtest 验证 |
| **投掷物命中率失衡** | 远距离投掷过于简单或困难 | min_hit_chance 参数调优（默认0.3） |
| **热武器弹药系统** | 弹药消耗节奏失控 | 与关卡设计师协调热武器获取来源 |
| **震动反馈平台差异** | 不同平台震动效果差异大 | 提供平台适配层，分级震动强度 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 0.5ms/帧 | 武器状态查询 < 1帧 |
| **Memory** | < 2MB | WeaponTemplateLibrary 数据资产 |
| **Network** | 无影响 | 单机游戏 |

---

## Migration Plan

> **⚠️ WeaponState 枚举迁移说明**
>
> `WeaponState` 枚举已统一定义在 `shared-types.md §3.3`。
> - **迁移时机**：当 shared-types.md 合并后，Weapon System 代码中的本地定义应删除
> - **向后兼容**：通过类型别名（alias）保持兼容：`using WeaponState = SharedTypes.WeaponState;`
> - **废弃警告**：在 shared-types.md 完成合并前，禁止在 WeaponSystem 代码中定义新的 WeaponState 枚举值

### Phase 1: 基础框架
- [ ] 创建 WeaponData ScriptableObject 模板定义（Feature Layer）
- [ ] 创建 WeaponTemplateLibrary
- [ ] **WeaponState 枚举已在 shared-types.md §3.3 中定义，直接引用**
- [ ] 定义所有武器事件类型

### Phase 2: 组件系统
- [ ] 实现 DamageComponent、RangeComponent
- [ ] 实现 AmmoComponent（热武器）
- [ ] 实现 ExplosiveComponent（爆炸物）
- [ ] 实现 AnimationTagsComponent

### Phase 3: 事件集成
- [ ] 订阅 ObjectStateChangedEvent
- [ ] 实现武器状态同步逻辑
- [ ] 实现武器切换机制（Tab/Q/E/X）

### Phase 4: 伤害输出
- [ ] 实现 ThrowableHandler（投掷物命中判定）
- [ ] 与 Health 系统联调 DamageRequest 接口
- [ ] **不**实现 ExplosionHandler（由 Health System 实现，见 ADR-0008）

### Phase 5: Gritty Takedowns 集成
- [ ] 实现 WeaponQueryRequest/WeaponQueryResponse 接口
- [ ] 验证 Gritty Takedowns 能正确获取 WeaponData

### Phase 6: 沉浸式反馈
- [ ] 实现 WeaponUsedEvent 到音频系统
- [ ] 实现手柄震动适配层

---

## Validation Criteria

1. **目录结构验证**：`Assets/Game/Features/WeaponSystem/` 目录存在（遵循 ADR-0003）
2. **环境物件状态同步**：订阅 ObjectStateChangedEvent 后，拾取/丢弃物件状态正确更新
3. **武器切换机制**：Tab/Q/E/X 按键正确切换热武器/环境物件
4. **爆炸事件发送**：ExplosionEvent 正确发送到 Health 系统，由 Health 执行伤害计算
5. **投掷物命中判定**：远距离命中率不低于 DEFAULT_MIN_HIT_CHANCE (0.3f)
6. **热武器弹药**：弹药耗尽正确转入 Empty 状态，换弹时间符合 ReloadTime
7. **WeaponQuery 接口**：Gritty Takedowns 查询返回正确的 WeaponData（含 weapon_id）
8. **事件广播**：WeaponStateChangedEvent/DamageRequest/ExplosionEvent 正确发送到目标系统

### shared-types.md 类型存在性验收
9. **shared-types.md 完整性**：验证以下类型已在 shared-types.md 中正确定义：
    - `DamageType`（§2.1）
    - `DamageRequest`（§2.3）
    - `ExplosionEvent`（§2.4）
    - `WeaponCategory`（§3.1）
    - `RangeType`（§3.1.1）
    - `AmmoType`（§3.1.2）
    - `DetonationType`（§3.1.3）
    - `ObjectCategory`（§3.2）
    - `WeaponState`（§3.3）
    - `WeaponStateChangedEvent`（§3.4）
    - `ObjectStateChangedEvent`（§3.5）
    - `WeaponQueryRequest` / `WeaponQueryResponse`（§3.8）

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — Weapon System 通过 Event Bus 与其他系统通信
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — **Weapon System 属于 Feature Layer**，目录为 `Features/WeaponSystem/`
- [ADR-0008: 脆弱度与伤害系统](./adr-0008-health-lethality-architecture.md) — **ExplosionHandler 属于 Health System**，Weapon System 仅发送位置参数
- [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) — Gritty Takedowns 查询 Weapon System 获取 WeaponData
- [共享类型定义](./shared-types.md) — **DamageRequest、ExplosionEvent、ObjectCategory 等跨 ADR 类型统一定义在此**
- [Weapon System GDD](../../design/gdd/weapon-system.md) — 本 ADR 的设计依据
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 事件定义的权威文档
