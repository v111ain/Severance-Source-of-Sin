# ADR-0008: 脆弱度与伤害系统 (Health & Lethality) 架构决策

## Status
**Accepted**

## Date
2026-04-09

## Last Updated
2026-04-09

## Context

### Problem Statement

Health & Lethality 系统是《断绝：罪恶之源》"致命的脆弱感"游戏支柱的核心实现。它不采用传统的数值血量系统，而是使用**离散的死亡状态机**——任何 Lethal 伤害都能直接致死。本系统需要处理：

1. **伤害判定**：Lethal vs Blunt 伤害类型，命中部位，护甲穿透
2. **状态机**：Healthy → Staggered → Downed → Dead 的状态转换
3. **事件广播**：向 NPC AI、Gritty Takedowns、Immersive Audio 广播状态变化
4. **爆炸处理**：混合型爆炸伤害（近距离 Lethal，远距离 Blunt）

### Constraints

- **致命性约束**：无护甲状态下，Lethal 伤害命中 Head/Torso 必须立即死亡
- **护甲约束**：护甲仅覆盖 Torso，一次 Lethal 伤害后进入 Staggered 而非死亡
- **玩家约束**：玩家不受 Downed 状态影响（与 NPC 不同）
- **性能约束**：爆炸范围伤害需要空间分区优化，避免 O(n) 全实体遍历

### Requirements

- **必须**：定义 Health State 状态机（Healthy/Staggered/Downed/Dead）
- **必须**：定义伤害类型和部位系统（Lethal/Blunt, Head/Torso/Limbs）
- **必须**：定义护甲穿透机制（Penetration vs ArmorLevel）
- **必须**：定义爆炸伤害混合判定（lethal_ratio 半径）
- **必须**：定义与其他系统的事件接口

---

## Decision

### 架构决策

采用**离散状态机 + 伤害类型分类**架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Health & Lethality 架构                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                      HealthSystem (Core Component)                 │   │
│  │  - 单例模式，全局访问                                              │   │
│  │  - 管理所有实体的生命状态                                          │   │
│  │  - 处理 DamageRequest 和 ExplosionEvent                           │   │
│  │  - 协调各子系统工作                                                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                      │
│          ┌─────────────────────────┴─────────────────────────┐          │
│          ▼                                                   ▼          │
│  ┌───────────────────┐                           ┌───────────────────┐   │
│  │  HealthStateMachine│                           │  ArmorSystem      │   │
│  │  (实体状态机)       │                           │  (护甲系统)        │   │
│  │                   │                           │                   │   │
│  │ Healthy          │                           │ ArmorLevel        │   │
│  │     ↓            │                           │ ArmorDurability   │   │
│  │ Staggered        │                           │ ArmorState        │   │
│  │     ↓            │                           │                   │   │
│  │ Downed (仅 NPC)  │                           └───────────────────┘   │
│  │     ↓            │                                                        │
│  │ Dead             │                                                        │
│  └───────────────────┘                                                        │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     SpatialDamageCalculator                       │   │
│  │  - 爆炸/范围伤害计算                                              │   │
│  │  - 空间分区优化（Quadrant/Octree）                                │   │
│  │  - lethal_ratio 判定                                             │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### HealthSystem 核心类

```csharp
// HealthSystem.cs
/// <summary>
/// Health System 主控制器 - 管理所有实体的生命状态
/// </summary>
public class HealthSystem : MonoBehaviour
{
    public static HealthSystem Instance { get; private set; }

    // 子系统引用（DamageHandler 和 ExplosionHandler 是普通类）
    private DamageHandler _damageHandler;
    private ExplosionHandler _explosionHandler;

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        _damageHandler = new DamageHandler();
        _explosionHandler = new ExplosionHandler();
    }

    public void Initialize()
    {
        // 订阅事件
        EventBus.Instance.Subscribe<DamageRequest>(OnDamageRequest);
        EventBus.Instance.Subscribe<ExplosionEvent>(OnExplosionEvent);

        _explosionHandler.Initialize(_damageHandler);
    }

    private void OnDamageRequest(DamageRequest request)
    {
        _damageHandler.ProcessDamageRequest(request);
    }

    private void OnExplosionEvent(ExplosionEvent explosion)
    {
        _explosionHandler.ProcessExplosion(explosion);
    }
}
```

### 1. Health State 状态机

```csharp
// HealthState.cs
public enum HealthState
{
    HEALTHY,    // 健康
    STAGGERED,  // 硬直（可被处决）
    DOWNED,     // 倒地/重伤（仅 NPC，玩家不受此状态影响）
    DEAD        // 死亡
}

// 命中部位枚举
public enum HitLocation
{
    HEAD,   // 头部（致命）
    TORSO,  // 躯干（可被护甲保护）
    LIMBS   // 四肢（减伤）
}

// 状态转换规则表
public static class HealthStateTransitions
{
    // 常量定义
    public const float BLUNT_WINDOW_SECONDS = 3.0f;  // 连续 Blunt 判定时间窗口
    public const float DOWNED_RECOVERY_TIME = 15.0f; // 倒地后恢复时间（秒）
    public const float STAGGER_DURATION = 5.0f;      // 硬直状态持续时间（秒）

    public static HealthState GetNextState(
        HealthState current,
        DamageType damageType,
        HitLocation hitLocation,
        bool hasArmor,
        bool armorSavedLife,  // true = 护甲成功保护了目标（穿透失败），false = 穿透成功/无护甲
        bool isContinuousBlunt = false)
    {
        if (current == HealthState.DEAD)
            return HealthState.DEAD;  // 死亡后不再转换

        // Lethal 伤害
        if (damageType == DamageType.LETHAL)
        {
            // DOWNED 状态下再受 Lethal → 立即死亡（处决终结）
            if (current == HealthState.DOWNED)
                return HealthState.DEAD;

            // LIMBS 部位：Lethal 伤害按 Blunt 处理（减伤效果）
            if (hitLocation == HitLocation.LIMBS)
            {
                if (isContinuousBlunt && current == HealthState.STAGGERED)
                    return HealthState.DOWNED;
                return HealthState.STAGGERED;
            }

            // 护甲成功保护（armorSavedLife = true）：Lethal 伤害被护甲吸收 → 硬直，护甲耐久 -1
            // 无护甲或穿透成功（armorSavedLife = false）：Lethal 伤害直接致死
            if (hasArmor && armorSavedLife)
                return HealthState.STAGGERED;

            // 无护甲 或 穿透成功 → 死亡
            return HealthState.DEAD;
        }

        // Blunt 伤害
        if (damageType == DamageType.BLUNT)
        {
            // 连续 Blunt 伤害（在 3 秒窗口内再次受到 Blunt）且当前已是 Staggered → 倒地
            if (isContinuousBlunt && current == HealthState.STAGGERED)
                return HealthState.DOWNED;

            return HealthState.STAGGERED;  // 首次硬直
        }

        return current;
    }
}
```

### 2. 护甲穿透机制

```csharp
// ArmorSystem.cs
public class ArmorSystem
{
    public int ArmorLevel { get; private set; }      // 1-10
    public int Durability { get; private set; }     // 当前耐久
    public ArmorState State { get; private set; }     // ACTIVE / DESTROYED

    private Vector3 _armorPosition;  // 护甲位置（由 Entity 初始化时设置）

    public ArmorSystem(int level, int durability, Vector3 armorPosition)
    {
        ArmorLevel = level;
        Durability = durability;
        State = ArmorState.ACTIVE;
        _armorPosition = armorPosition;
    }

    /// <summary>
    /// 穿透判定（统一入口）
    /// </summary>
    /// <returns>穿透结果，包含是否穿透、是否吸收、新护甲状态</returns>
    public PenetrationResult ProcessPenetration(int weaponPenetration, int ownerEntityId)
    {
        if (State == ArmorState.DESTROYED)
        {
            return new PenetrationResult
            {
                IsPenetrated = true,
                IsAbsorbed = false,
                NewArmorState = ArmorState.DESTROYED
            };
        }

        if (weaponPenetration >= ArmorLevel)
        {
            // 穿透成功，无视护甲直接致死
            return new PenetrationResult
            {
                IsPenetrated = true,
                IsAbsorbed = false,
                NewArmorState = State
            };
        }
        else
        {
            // 穿透失败，护甲吸收伤害，消耗耐久
            Durability--;
            if (Durability <= 0)
            {
                State = ArmorState.DESTROYED;

                // 广播护甲破坏事件（供视觉效果系统放置破损护甲模型）
                EventBus.Instance.Publish(new ArmorDestroyedEvent
                {
                    npc_id = ownerEntityId,
                    position = _armorPosition
                });
            }

            return new PenetrationResult
            {
                IsPenetrated = false,
                IsAbsorbed = true,
                NewArmorState = State
            };
        }
    }

    /// <summary>
    /// 获取护甲位置（供 ArmorDestroyedEvent 使用）
    /// </summary>
    public Vector3 GetArmorPosition() => _armorPosition;
}

public enum ArmorState
{
    ACTIVE,
    DESTROYED
}

public struct PenetrationResult
{
    public bool IsPenetrated;     // 是否穿透护甲
    public bool IsAbsorbed;       // 护甲是否吸收了伤害
    public ArmorState NewArmorState;
}
```

**注意**：`ArmorSystem.ProcessPenetration(int weaponPenetration, int ownerEntityId)` 是穿透判定的唯一入口，`DamageHandler` 必须使用此方法，不得在其他地方重复穿透逻辑。护甲破坏事件（`ArmorDestroyedEvent`）也在此方法内广播，`ownerEntityId` 用于标识护甲所属实体，`_armorPosition` 在构造时由 Entity 注入。

### 3. 伤害请求处理

```csharp
// DamageHandler.cs
public class DamageHandler
{
    // 连续 Blunt 伤害追踪（时间窗口内多次 Blunt 才算"连续"）
    private Dictionary<int, float> _lastBluntTime = new();

    // 清理过期记录的阈值（超过此时间未受击则移除记录）
    private const float BLUNT_RECORD_EXPIRY = 30f;

    public void ProcessDamageRequest(DamageRequest request)
    {
        var target = EntityManager.Instance.GetEntity(request.target_id);
        if (target == null)
            return;

        // 死亡时清理该实体的 Blunt 记录
        if (target.HealthState == HealthState.DEAD)
        {
            _lastBluntTime.Remove(target.EntityId);
            return;
        }

        // 获取护甲系统（仅 NPC 有护甲，玩家无护甲）
        var armor = target.GetComponent<ArmorSystem>();
        bool armorSavedLife = false;

        // 穿透判定（仅躯干受击且有护甲时）
        if (armor != null && request.hit_location == HitLocation.TORSO)
        {
            var result = armor.ProcessPenetration(request.penetration, target.EntityId);
            // armorSavedLife = true 表示护甲成功吸收了伤害（穿透失败）
            armorSavedLife = !result.IsPenetrated;
        }

        // 检查连续 Blunt 伤害
        bool isContinuousBlunt = IsContinuousBlunt(target.EntityId, request.damage_type);

        // 状态转换
        var newState = HealthStateTransitions.GetNextState(
            target.HealthState,
            request.damage_type,
            request.hit_location,
            armor != null,       // hasArmor
            armorSavedLife,      // armorSavedLife = true 表示护甲吸收了伤害
            isContinuousBlunt);

        if (newState != target.HealthState)
        {
            var oldState = target.HealthState;
            target.SetHealthState(newState);

            // 广播事件
            BroadcastHealthEvent(target, oldState, newState, request);

            // 倒地状态需要向 DownedRecoverySystem 注册
            if (newState == HealthState.DOWNED && target is NPCEntity)
            {
                DownedRecoverySystem.Instance.RegisterDowned(target.EntityId);

                // 清理任何待处理的 stagger 恢复计时器（防止 Downed → Stagger → Downed 时序问题）
                // 注意：ClearStaggerRecovery 是幂等操作，即使该 NPC 不在 STAGGERED 状态也不会出错
                StaggerRecoverySystem.Instance.ClearStaggerRecovery(target.EntityId);
            }
        }

        // 更新最后受击时间
        if (request.damage_type == DamageType.BLUNT)
        {
            _lastBluntTime[target.EntityId] = Time.time;
        }
    }

    private bool IsContinuousBlunt(int entityId, DamageType damageType)
    {
        if (damageType != DamageType.BLUNT)
            return false;

        if (_lastBluntTime.TryGetValue(entityId, out var lastTime))
        {
            // 清理过期记录
            if (Time.time - lastTime > BLUNT_RECORD_EXPIRY)
            {
                _lastBluntTime.Remove(entityId);
                return false;
            }
            return (Time.time - lastTime) <= HealthStateTransitions.BLUNT_WINDOW_SECONDS;
        }
        return false;
    }

    private void BroadcastHealthEvent(
        Entity target,
        HealthState oldState,
        HealthState newState,
        DamageRequest request)
    {
        if (target is NPCEntity)
        {
            // NPC 状态变化由 NPC AI 系统广播
            EventBus.Instance.Publish(new NPCStateChangedEvent
            {
                npc_id = target.EntityId,
                entity_type = EntityType.NPC,
                old_state = oldState,
                new_state = newState,
                damage_type = request.damage_type
            });
        }
        else if (target is PlayerEntity)
        {
            // 玩家状态变化由 Health System 广播
            EventBus.Instance.Publish(new PlayerDamagedEvent
            {
                player_id = target.EntityId,
                damage_type = request.damage_type,
                hit_location = request.hit_location,
                source_entity_id = request.source_entity_id
            });
        }
    }
}
```

### 4. 爆炸伤害处理

```csharp
// ExplosionHandler.cs
public class ExplosionHandler
{
    // 爆炸伤害穿透值：int.MaxValue 确保爆炸必定穿透护甲
    // 注意：穿透值 >= ArmorLevel 时穿透成功，int.MaxValue 绕过所有护甲
    public const int ARMOR_IGNORE_PENETRATION = int.MaxValue;

    private SpatialDamageCalculator _spatialCalculator;
    private DamageHandler _damageHandler;

    public void Initialize(DamageHandler damageHandler)
    {
        _damageHandler = damageHandler;
        _spatialCalculator = new SpatialDamageCalculator();
    }

    public void ProcessExplosion(ExplosionEvent explosion)
    {
        // 使用空间分区获取范围内的所有实体
        var entitiesInRange = _spatialCalculator.GetEntitiesInRadius(
            explosion.position,
            explosion.radius);

        foreach (var entity in entitiesInRange)
        {
            float distance = Vector3.Distance(explosion.position, entity.Position);

            // lethal_ratio 半径内：Lethal 伤害（全额 base_damage）
            // 超出 lethal_ratio 但在爆炸半径内：Blunt 伤害（按距离衰减 + stagger_multiplier）
            DamageType damageType;
            float finalDamage;
            if (distance <= explosion.radius * explosion.lethal_ratio)
            {
                damageType = DamageType.LETHAL;
                finalDamage = explosion.base_damage;
            }
            else
            {
                damageType = DamageType.BLUNT;
                float stagger_multiplier = 1.5f;
                finalDamage = explosion.base_damage * (1 - distance / explosion.radius) * stagger_multiplier;
            }

            // 爆炸伤害绕过护甲（穿透值设为 int.MaxValue）
            var request = new DamageRequest
            {
                target_id = entity.EntityId,
                damage_type = damageType,
                damage_amount = finalDamage,
                penetration = ARMOR_IGNORE_PENETRATION,  // int.MaxValue = 无视护甲
                source = "Explosion"
            };

            _damageHandler.ProcessDamageRequest(request);
        }

        // 检查是否有 NPC 死亡，广播爆炸警报
        CheckAndBroadcastExplosionAlert(explosion, entitiesInRange);
    }

    private void CheckAndBroadcastExplosionAlert(
        ExplosionEvent explosion,
        List<Entity> affectedEntities)
    {
        // 当前实现：爆炸杀死多个 NPC 时只广播一次警报
        // 这是简化设计，避免同一爆炸触发多次警报事件
        // 如需对每个死亡单独广播，可移除 break 并改为聚合广播
        foreach (var entity in affectedEntities)
        {
            if (entity is NPCEntity npc && npc.HealthState == HealthState.DEAD)
            {
                EventBus.Instance.Publish(new ExplosionAlertEvent
                {
                    position = explosion.position,
                    radius = explosion.radius,
                    victim_id = npc.EntityId,
                    killer_is_player = true  // 简化为玩家引爆
                });
                break;  // 同一爆炸只广播一次警报
            }
        }
    }
}
```

### 5. 倒地恢复机制（仅 NPC）

```csharp
// DownedRecoverySystem.cs
/// <summary>
/// 倒地恢复系统 - 负责管理 NPC 从 Downed 状态到 Staggered 状态的恢复计时
/// 使用 MonoBehaviour 单例模式（统一规范）
/// </summary>
public class DownedRecoverySystem : MonoBehaviour
{
    public static DownedRecoverySystem Instance { get; private set; }

    private Dictionary<int, float> _downedTimers = new();

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }

    public void RegisterDowned(int npcId)
    {
        _downedTimers[npcId] = 0f;
    }

    private void Update()
    {
        UpdateDownedRecovery(Time.deltaTime);
    }

    private void UpdateDownedRecovery(float deltaTime)
    {
        foreach (var kvp in _downedTimers.ToList())
        {
            var npcId = kvp.Key;
            var elapsed = kvp.Value + deltaTime;

            var npc = NPCManager.Instance.GetNPC(npcId);
            if (npc == null || npc.HealthState != HealthState.DOWNED)
            {
                _downedTimers.Remove(npcId);
                continue;
            }

            if (elapsed >= HealthStateTransitions.DOWNED_RECOVERY_TIME)
            {
                // 恢复至 Staggered
                npc.SetHealthState(HealthState.STAGGERED);
                _downedTimers.Remove(npcId);

                // 启动 StaggerDuration 计时器
                npc.StartStaggerRecoveryTimer();
            }
            else
            {
                _downedTimers[npcId] = elapsed;
            }
        }
    }
}
```

### 6. Entity 基类扩展

```csharp
// Entity.cs（扩展）
// 包含 Health System 相关接口
// 注意：HealthState 直接存储在 Entity 基类中，而非独立的 HealthStateMachine 组件
// HealthStateMachine.cs 组件用于需要独立状态的非 Entity 对象（如环境物件）
public partial class Entity
{
    public HealthState HealthState { get; private set; }

    public void SetHealthState(HealthState newState)
    {
        HealthState = newState;
    }

    public void StartStaggerRecoveryTimer()
    {
        StaggerRecoverySystem.Instance.RegisterStaggerRecovery(EntityId, HealthStateTransitions.STAGGER_DURATION);
    }
}

// HealthStateMachine.cs（独立组件，用于非 Entity 对象）
/// <summary>
/// 独立状态机组件 - 用于需要追踪生命状态但不是 Entity 的对象
/// 对于 PlayerEntity 和 NPCEntity，直接使用 Entity.HealthState 属性
/// </summary>
public class HealthStateMachine : MonoBehaviour
{
    public HealthState CurrentState { get; private set; }
    // ... 类似 Entity 的实现
}

// 硬直恢复系统 - 与 DownedRecoverySystem 类似
public class StaggerRecoverySystem : MonoBehaviour
{
    public static StaggerRecoverySystem Instance { get; private set; }

    private Dictionary<int, float> _staggerTimers = new();

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }

    public void RegisterStaggerRecovery(int entityId, float duration)
    {
        _staggerTimers[entityId] = 0f;
    }

    // 供外部调用：当 NPC 离开 STAGGERED 状态时清理计时器
    public void ClearStaggerRecovery(int entityId)
    {
        _staggerTimers.Remove(entityId);
    }

    private void Update()
    {
        UpdateStaggerRecovery(Time.deltaTime);
    }

    private void UpdateStaggerRecovery(float deltaTime)
    {
        foreach (var kvp in _staggerTimers.ToList())
        {
            var entityId = kvp.Key;
            var elapsed = kvp.Value + deltaTime;

            var entity = EntityManager.Instance.GetEntity(entityId);
            if (entity == null || entity.HealthState != HealthState.STAGGERED)
            {
                _staggerTimers.Remove(entityId);
                continue;
            }

            if (elapsed >= HealthStateTransitions.STAGGER_DURATION)
            {
                entity.SetHealthState(HealthState.HEALTHY);
                _staggerTimers.Remove(entityId);
            }
            else
            {
                _staggerTimers[entityId] = elapsed;
            }
        }
    }
}
```

### 7. SpatialDamageCalculator 实现

```csharp
// SpatialDamageCalculator.cs
/// <summary>
/// 空间分区伤害计算器 - 用于爆炸等范围伤害的空间查询优化
/// 使用与 LOS System 相同的格子分区策略（10m 格子）
/// 统一使用 GameConstants.SPATIAL_GRID_SIZE（见 shared-constants.md）
/// </summary>
public class SpatialDamageCalculator
{
    private const int GRID_SIZE = GameConstants.SPATIAL_GRID_SIZE;
    private Dictionary<Vector2Int, List<int>> _grid = new();

    public void RegisterEntity(int entityId, Vector3 position)
    {
        var cell = WorldToCell(position);
        if (!_grid.ContainsKey(cell))
            _grid[cell] = new List<int>();
        _grid[cell].Add(entityId);
    }

    public void UnregisterEntity(int entityId, Vector3 position)
    {
        var cell = WorldToCell(position);
        if (_grid.TryGetValue(cell, out var list))
        {
            list.Remove(entityId);
        }
    }

    public List<Entity> GetEntitiesInRadius(Vector3 center, float radius)
    {
        var result = new List<Entity>();
        int cellRange = Mathf.CeilToInt(radius / GRID_SIZE);
        var centerCell = WorldToCell(center);

        for (int x = -cellRange; x <= cellRange; x++)
        {
            for (int z = -cellRange; z <= cellRange; z++)
            {
                var cell = new Vector2Int(centerCell.x + x, centerCell.y + z);
                if (_grid.TryGetValue(cell, out var entities))
                {
                    foreach (var entityId in entities)
                    {
                        var entity = EntityManager.Instance.GetEntity(entityId);
                        if (entity != null)
                        {
                            float distance = Vector3.Distance(center, entity.Position);
                            if (distance <= radius)
                            {
                                result.Add(entity);
                            }
                        }
                    }
                }
            }
        }
        return result;
    }

    private Vector2Int WorldToCell(Vector3 pos)
    {
        return new Vector2Int(
            Mathf.FloorToInt(pos.x / GRID_SIZE),
            Mathf.FloorToInt(pos.z / GRID_SIZE));
    }
}
```

### 8. Unity 项目结构

```
Assets/Game/
├── Foundation/
│   └── Health/
│       ├── HealthSystem.cs              # 主系统管理器
│       ├── HealthState.cs               # 状态枚举和转换规则
│       ├── HealthStateMachine.cs        # 单个实体的状态机
│       ├── DamageHandler.cs             # 伤害请求处理
│       ├── ExplosionHandler.cs          # 爆炸伤害处理
│       ├── ArmorSystem.cs               # 护甲系统（含穿透判定）
│       ├── DownedRecoverySystem.cs      # 倒地恢复系统
│       ├── StaggerRecoverySystem.cs     # 硬直恢复系统
│       ├── SpatialDamageCalculator.cs    # 空间分区伤害计算
│       └── HealthConfigSO.cs            # 配置 ScriptableObject
```

### 9. 事件接口定义

```csharp
// Health System 发出的事件

// 实体类型枚举（【重要】此枚举定义在共享的 EventTypes.cs 中，各 ADR 不得重复定义）
// 本文档中的引用仅作说明用途，实际类型应从 EventTypes.cs 导入
// public enum EntityType
// {
//     NPC,
//     Player
// }

// NPCStateChangedEvent（由 Health System 直接广播）
public struct NPCStateChangedEvent
{
    public int npc_id;
    public EntityType entity_type;  // NPC（固定值）
    public HealthState old_state;
    public HealthState new_state;
    public DamageType damage_type;  // LETHAL / BLUNT / NONE
}

// PlayerDamagedEvent（由 Health System 广播）
public struct PlayerDamagedEvent
{
    public int player_id;
    public DamageType damage_type;
    public HitLocation hit_location;
    public int source_entity_id;
    // 注意：is_lethal 字段已移除
    // 接收方可通 damage_type == DamageType.LETHAL 判断是否为致命伤害
    // 或通过 HealthState == HealthState.DEAD 判断是否已死亡
}

// ExplosionAlertEvent（爆炸造成 NPC 死亡时广播）
public struct ExplosionAlertEvent
{
    public Vector3 position;
    public float radius;
    public int victim_id;
    public bool killer_is_player;
}

// ArmorDestroyedEvent（NPC 护甲被破坏时广播）
// 由 ArmorSystem.ProcessPenetration() 内部广播，属于 Health System 事件体系的一部分
// 用于通知视觉效果系统放置破损护甲模型
public struct ArmorDestroyedEvent
{
    public int npc_id;
    public Vector3 position;  // 护甲破坏位置（用于视觉效果放置）
}

// FriendlyFireExplosionEvent（友军误伤爆炸时广播）
public struct FriendlyFireExplosionEvent
{
    public int victim_id;
    public int killer_id;
    public FactionRelation faction_relation;
    public Vector3 explosion_position;
}

// Health System 接收的事件

// ⚠️ DamageRequest 定义见 shared-types.md
// ⚠️ ExplosionEvent 定义见 shared-types.md
```

---

## Alternatives Considered

### Alternative 1: 传统数值 HP 系统

- **描述**：使用 100/100 连续血量，每次伤害减少固定数值
- **Pros**：实现简单，玩家有"容错空间"
- **Cons**：
  - 与"一击必杀"的核心体验冲突
  - 需要额外机制防止玩家被秒杀
- **拒绝理由**：
  - 违反"致命的脆弱感"游戏支柱
  - 《只狼》《迈阿密热线》证明离散状态机可行

### Alternative 2: 仅区分 Lethal/Non-Lethal

- **描述**：不区分伤害类型，只区分是否致命
- **Pros**：实现更简单
- **Cons**：无法表现"硬直"和"倒地"的处决窗口
- **拒绝理由**：
  - 硬直/倒地状态是 Gritty Takedowns 系统的基础
  - 连续 Blunt 伤害可导致倒地的设计增加了战斗深度

---

## Consequences

### Positive

- **一击必杀的紧张感**：任何 Lethal 伤害都可能导致死亡
- **护甲战术选择**：玩家可以选择穿透护甲或消耗护甲耐久
- **处决窗口**：Staggered/Downed 状态为 Gritty Takedowns 提供触发条件
- **爆炸战术**：范围伤害混合 Lethal/Blunt，增加战术多样性

### Negative

- **玩家可能感到"不公平"**：需要良好的视觉/音频反馈解释死亡原因
- **倒地恢复机制复杂**：NPC 从 Downed 恢复到 Healthy 需要两个计时器
- **护甲仅护 torso**：玩家可能抱怨"为什么打腿没有意义"

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **玩家频繁死亡** | 难度过高导致挫败感 | 设计阶段充分 playtest；提供"重开"快速机制 |
| **护甲视觉不清** | 玩家不知道敌人有护甲 | NPC 模型必须有明显护甲特征（如防弹背心） |
| **爆炸伤害过强** | 一次爆炸死一片，破坏体验 | 合理调优 lethal_ratio；爆炸后 NPC 进入 ALERT 而非立即死亡 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 0.5ms/帧 | 爆炸伤害使用空间分区，避免 O(n) |
| **Memory** | < 10MB | HealthStateMachine 按实体分配 |
| **Network** | 无影响 | 单机游戏 |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 HealthState 枚举和转换规则
- [ ] 创建 HealthStateMachine 组件
- [ ] 创建 HealthConfigSO

### Phase 2: 伤害处理
- [ ] 实现 DamageHandler（含连续 Blunt 伤害时间窗口）
- [ ] 实现 ArmorSystem（含统一穿透判定入口）
- [ ] 集成 Event Bus 事件发送

### Phase 3: 爆炸系统
- [ ] 实现 SpatialDamageCalculator
- [ ] 实现 ExplosionHandler
- [ ] 实现 ExplosionAlertEvent 广播

### Phase 4: 恢复机制
- [ ] 实现 DownedRecoverySystem
- [ ] 实现 StaggerDuration 计时器
- [ ] 验证恢复路径（Down → Stagger → Healthy）

### Phase 5: Gritty Takedowns 集成
- [ ] 与 Gritty Takedowns 系统确认 DamageRequest 接口契约
- [ ] 验证 Gritty Takedowns 能正确订阅 NPCStateChangedEvent（检测 Staggered/Downed 状态）
- [ ] 验证 Gritty Takedowns 发送的 DamageRequest 能被 Health System 正确处理
- [ ] 验证处决后 Health System 能正确广播状态变化

### Phase 6: 集成测试
- [ ] 验证 Lethal 伤害立即致死
- [ ] 验证护甲吸收机制
- [ ] 验证穿透判定（Penetration >= ArmorLevel = 穿透成功）
- [ ] 验证连续 Blunt 伤害导致倒地（3 秒窗口）
- [ ] 验证爆炸混合伤害（lethal_ratio 半径内 Lethal）
- [ ] 验证事件正确广播到 NPC AI、Immersive Audio

---

## Validation Criteria

1. **Lethal 立即致死**：无护甲实体被 Lethal 命中 Head/Torso，立即转为 Dead
2. **护甲吸收**：有护甲实体被 Lethal 命中 Torso，进入 Staggered，护甲耐久 -1
3. **穿透成功**：Penetration >= ArmorLevel 时无视护甲直接致死
4. **Blunt 累积**：连续 Blunt 伤害正确转为 Downed 状态
5. **倒地恢复**：NPC 在 Downed 状态 15 秒后自动恢复至 Staggered
6. **爆炸混合伤害**：lethal_ratio 半径内 Lethal，外围 Blunt
7. **事件广播**：状态变化正确广播 NPCStateChangedEvent/PlayerDamagedEvent
8. **玩家无 Downed**：玩家不受 Downed 状态影响

---

## Related Decisions

- [ADR-0001: 事件驱动架构](./adr-0001-event-driven-architecture.md) — Health System 通过 Event Bus 与 NPC AI、Gritty Takedowns、Immersive Audio 通信
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — Health System 属于 Foundation Layer
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC AI 订阅 NPCStateChangedEvent
- [ADR-0010: 武器系统](./adr-0010-weapon-system-architecture.md) — Weapon System 发送 ExplosionEvent 到 Health System
- [ADR-0011: 沉重处决系统](./adr-0011-gritty-takedowns-architecture.md) — Gritty Takedowns 发送 DamageRequest 到 Health System
- [共享类型定义](./shared-types.md) — **DamageRequest、ExplosionEvent、HealthState、HitLocation 等跨 ADR 类型统一定义在此**
- [共享常量定义](./shared-constants.md) — GRID_SIZE 和 ARMOR_IGNORE_PENETRATION 等跨 ADR 常量
- [Health & Lethality GDD](../../design/gdd/health-lethality.md) — 本 ADR 的设计依据
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 事件定义的权威文档
