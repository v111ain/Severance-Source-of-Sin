# 共享常量定义 (Shared Constants)

> **版本**: 1.2.0
> **创建日期**: 2026-04-10
> **状态**: APPROVED
> **维护者**: 架构师
> **更新日期**: 2026-04-10 (v1.2 — 新增 LOS/Player/Health/GrittyTakedowns/WorldMap 常量)

---

## 1. 空间分区常量

### GRID_SIZE

**值**: `10` (米)

**定义位置**: `Assets/Game/Foundation/Shared/Constants/GameConstants.cs`

**用途**: 所有使用空间分区优化的系统（LOS System、Health System 等）统一使用此常量。

**选择依据**:
- NPC 最大感知范围（MaxVisionRange）通常为 15-20m
- 每个格子覆盖 10m × 10m，任意感知范围内的查询最多遍历 3×3 = 9 个格子
- 100 个 NPC 均匀分布时，每格子约 1 个 NPC，遍历开销极低
- 10m 格子既避免了过小格子导致的内存开销，也避免了过大格子导致的精确度问题

**使用示例**:

```csharp
// SpatialPartition.cs
using Game.Foundation.Shared;

public class SpatialPartition
{
    private Dictionary<Vector2Int, List<int>> _grid = new();

    public List<int> GetNPCsInRange(Vector3 playerPos, float range)
    {
        int cellRange = Mathf.CeilToInt(range / GameConstants.SPATIAL_GRID_SIZE);
        // ...
    }
}
```

---

## 2. 护甲穿透常量

### ARMOR_IGNORE_PENETRATION

**值**: `int.MaxValue` (= 2,147,483,647)

**定义位置**: `Assets/Game/Foundation/Shared/Constants/GameConstants.cs`

**用途**: 爆炸伤害等无视护甲的伤害使用此常量作为穿透值。穿透判定逻辑为 `weaponPenetration >= armorLevel`，因此 int.MaxValue 确保必定穿透。

**使用示例**:

```csharp
// ExplosionHandler.cs
using Game.Foundation.Shared;

public class ExplosionHandler
{
    public const int ARMOR_IGNORE_PENETRATION = GameConstants.ARMOR_IGNORE_PENETRATION;

    public void ProcessExplosion(ExplosionEvent explosion)
    {
        var request = new DamageRequest
        {
            penetration = ARMOR_IGNORE_PENETRATION  // 无视护甲
        };
    }
}
```

---

## 3. 单例模式规范

所有需要全局访问的子系统统一使用 **MonoBehaviour + Instance** 单例模式：

```csharp
public class MySystem : MonoBehaviour
{
    public static MySystem Instance { get; private set; }

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }
}
```

**优势**:
- 避免"穷人的单例"（无参构造函数 + static Instance）导致的初始化顺序问题
- 与 Unity 生命周期兼容（可使用协程、Invoke 等）
- 可挂载到 GameObject，便于在 Inspector 中调试

---

## 4. LOS System 常量

### 暴露值计算常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `BASE_EXPOSURE_RATE` | `20f` | 基础暴露速率（%/秒），行走时 |
| `DECAY_RATE` | `30f` | 暴露值衰减速率（%/秒） |
| `MAX_VISION_RANGE` | `15f` | NPC 最大感知范围（米） |
| `VISION_CONE_ANGLE` | `120f` | NPC 视野锥角度（度） |
| `PROXIMITY_THRESHOLD` | `1.5f` | 贴脸判定距离（米） |

**定义位置**: `Assets/Game/Core/LOS/LOSConfigSO.cs`

**使用系统**: LOS System (Core Layer)

### 专注监听模式常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `MAX_SCREEN_DISTANCE` | `200f` | 屏幕距离阈值（像素） |
| `MIN_AIM_ACCURACY` | `0.5f` | 最低瞄准精度（0-1） |
| `TAG_RATE` | `33.3f` | 标签捕获速率（%/秒），3秒填满 |
| `FOCUS_ALIGNMENT_TIME` | `1f` | 准星对准后开始累积的延迟（秒） |

### 移动状态暴露乘数

| 状态 | 乘数 |
|------|------|
| `Idle` | 0.1 |
| `Walk` | 1.0 |
| `Sprint` | 3.0 |
| `Crouch` / `CrouchWalk` | 0.1 |
| `Action` | 0.0 |

---

## 5. Player Controller 常量

### 体力系统常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `MAX_STAMINA` | `100f` | 最大体力值 |
| `STAMINA_DRAIN_RATE` | `20f` | 冲刺时每秒消耗体力 |
| `STAMINA_REGEN_RATE` | `15f` | 每秒恢复体力 |
| `STAMINA_REGEN_DELAY` | `2f` | 停止冲刺后延迟恢复（秒） |
| `EXHAUSTION_THRESHOLD` | `30f` | 耗尽后需恢复到此阈值才能冲刺 |

### 移动系统常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `BASE_SPEED` | `5f` | 基础移动速度（米/秒） |
| `SPRINT_MULTIPLIER` | `1.6f` | 冲刺速度倍率 |
| `CROUCH_MULTIPLIER` | `0.5f` | 潜行速度倍率 |

### 噪声广播常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `WALK_NOISE_RADIUS` | `3f` | 行走时噪声半径（米） |
| `SPRINT_NOISE_RADIUS` | `8f` | 冲刺时噪声半径（米） |
| `CROUCH_NOISE_RADIUS` | `0f` | 潜行时静音 |

**定义位置**: `Assets/Game/Foundation/PlayerController/Config/PlayerControllerConfigSO.cs`

---

## 6. Health System 常量

### 状态转换常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `BLUNT_WINDOW_SECONDS` | `3.0f` | 连续 Blunt 判定时间窗口（秒） |
| `DOWNED_RECOVERY_TIME` | `15.0f` | 倒地后恢复时间（秒） |
| `STAGGER_DURATION` | `5.0f` | 硬直状态持续时间（秒） |
| `BLUNT_RECORD_EXPIRY` | `30f` | 清理过期 Blunt 记录的阈值（秒） |

**定义位置**: `Assets/Game/Foundation/Health/HealthState.cs`

---

## 7. Gritty Takedowns 常量

### 捆绑时间常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `BASE_TIE_UP_TIME` | `4.0f` | 基础捆绑时间（秒） |
| `MIN_TIE_UP_DURATION` | `3.0f` | 最短捆绑时间（秒） |
| `MAX_TIE_UP_DURATION` | `6.0f` | 最长捆绑时间（秒） |

### 捆绑取消阈值

| 常量名 | 值 | 说明 |
|--------|---|------|
| `MAX_CANCEL_DISTANCE` | `3.0f` | 超过此距离取消捆绑（米） |
| `MIN_CANCEL_ANGLE` | `120f` | 视角偏离角度阈值（度） |

**定义位置**: `Assets/Game/Foundation/Shared/TieUpCalculator.cs`

---

## 8. World Map 常量

### 危险区域等待协议常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `WAIT_CHECK_INTERVAL` | `0.5f` | 检测威胁消散的间隔（秒） |
| `THREAT_DISSIPATE_PROBABILITY` | `0.3f` | 每次检测时威胁消散的概率 |
| `MAX_WAIT_TIME` | `30.0f` | 最大等待时间（秒），超时后强制消散 |

**设计依据**:
- `THREAT_DISSIPATE_PROBABILITY = 0.3`：期望消散时间为 `0.5秒 / 0.3 ≈ 1.67` 秒（约 2 秒），符合"短时威胁"的设计预期
- `MAX_WAIT_TIME = 30 秒`：作为硬性上限，防止玩家卡在危险区域

**定义位置**: `Assets/Game/Foundation/WorldMap/WorldMapConfigSO.cs`

---

## 9. 修改日志

| 日期 | 版本 | 修改内容 | 作者 |
|------|------|---------|------|
| 2026-04-10 | 1.0.0 | 初稿创建，定义 GRID_SIZE 和 ARMOR_IGNORE_PENETRATION | 架构师 Agent |
| 2026-04-10 | 1.1.0 | 新增跨 ADR 类型引用说明，指向 shared-types.md | 架构师 Agent |
| 2026-04-10 | 1.2.0 | 新增 LOS/Player/Health/GrittyTakedowns/WorldMap 常量 | 架构师 Agent |
