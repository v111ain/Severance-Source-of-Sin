# ADR-0010: 武器系统 (Weapon System) 架构决策

## Status
**Proposed**（依赖 shared-types.md 的 APPROVED 类型定义）

> **⚠️ HapticFeedbackManager 废弃通知**：本 ADR 中的 `HapticFeedbackManager` 实现已废弃，统一使用 [ADR-0020](./adr-0020-input-system-architecture.md) 的 `HapticFeedbackManager`。
> 实际使用时，请引用 ADR-0020 的实现。

> **v1.4.1 更新**：修复以下评审问题
> - CategoryToWeaponId 数据驱动改造
> - 投掷物命中概率边界计算修正
> - weapon_id 唯一性验证
> - TryGetTemplate 方法新增
> - HapticFeedbackManager 依赖注入说明修正
> - GasolineCan/PropaneTank 映射补充
> - WeaponQueryRequest 空值验证

> **shared-types.md 类型完整性验证**：以下类型已在 shared-types.md（v1.3.0, APPROVED）中正确定义：
> - `DamageType`（§2.1）✅
> - `DamageRequest`（§2.3）✅
> - `ExplosionEvent`（§2.4）✅
> - `WeaponCategory`（§3.1）✅
> - `RangeType`（§3.1.1）✅
> - `AmmoType`（§3.1.2）✅
> - `DetonationType`（§3.1.3）✅
> - `ObjectCategory`（§3.2）✅
> - `WeaponState`（§3.3）✅
> - `WeaponStateChangedEvent`（§3.4）✅
> - `ObjectStateChangedEvent`（§3.5）✅
> - `WeaponQueryRequest/WeaponQueryResponse`（§3.8）✅

## Date
2026-04-10

## Last Updated
2026-04-14 (v1.4.3 — 距离衰减阈值定义)

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

### 1. WeaponData 组件化数据模型

> **⚠️ 初始化时序说明（已知限制）**：WeaponData 作为 ScriptableObject，其引用字段在第一次访问时可能尚未初始化。
> WeaponTemplateLibrary 通过 `Awake()` 构建映射表，但 ScriptableObject 的加载顺序在不同平台可能不一致。
>
> **Enhanced Mitigation（v1.2）**：
> 1. 所有查询方法都包含防御性检查（null 返回而非异常），确保安全返回
> 2. 添加 `IPreprocessBuild` 预处理，在打包前强制调用 `RebuildLookupMaps()` 验证映射表完整性
> 3. 提供 Editor 工具 `[ValidateWeaponLibrary]` MenuItem，一键验证所有 WeaponData 引用完整性
> 4. 构建时自动输出验证报告，列出所有孤立或缺失的 WeaponData 引用
>
> **验证方式**：
> - Editor 下：右键 `WeaponTemplateLibrary` → **Validate Weapon Library**
> - 打包前：`Unity Editor` → **Build → Validate Weapon System** 自动验证
>
> 这是 Unity ScriptableObject 系统的已知限制，通过多层防护缓解。

```csharp
// 验证工具（Editor only）
#if UNITY_EDITOR
[UnityEditor.MenuItem("Game/Weapons/Validate Weapon Library")]
public static void ValidateWeaponLibrary()
{
    var library = UnityEditor.Selection.activeObject as WeaponTemplateLibrary;
    if (library == null) return;

    var issues = new List<string>();
    var allIds = library.GetAllTemplateIds().ToList();

    foreach (var id in allIds)
    {
        var template = library.GetTemplate(id);
        if (template == null)
            issues.Add($"[MISSING] weapon_id '{id}' has no corresponding WeaponData");
    }

    if (issues.Count == 0)
        UnityEditor.EditorUtility.DisplayDialog("Validation Passed",
            $"All {allIds.Count} weapons validated successfully.", "OK");
    else
    {
        string message = string.Join("\n", issues);
        UnityEditor.EditorUtility.DisplayDialog("Validation Failed",
            $"Found {issues.Count} issues:\n\n{message}", "OK");
    }
}

// 打包前验证（IPreprocessBuild）
public class WeaponSystemPreprocessBuild : IPreprocessBuild
{
    public int callbackOrder => 0;

    public void OnPreprocessBuild(BuildTarget target, string path)
    {
        var libraries = UnityEditor.AssetDatabase.FindAssets("t:WeaponTemplateLibrary")
            .Select(guid => UnityEditor.AssetDatabase.LoadAssetAtPath<WeaponTemplateLibrary>(
                UnityEditor.AssetDatabase.GUIDToAssetPath(guid)))
            .Where(l => l != null);

        foreach (var library in libraries)
        {
            library.RebuildLookupMaps();
            var allIds = library.GetAllTemplateIds().ToList();

            // 检查 weapon_id 唯一性（重复时抛出 Error 而非仅 Warning）
            if (_duplicateWeaponIds.Count > 0)
            {
                throw new BuildFailedException(
                    $"[WeaponSystem] Duplicate weapon_id(s) found: {string.Join(", ", _duplicateWeaponIds)}. " +
                    "Each weapon_id must be unique across hotWeapons and environmentalWeapons.");
            }

            foreach (var id in allIds)
            {
                if (library.GetTemplate(id) == null)
                    throw new BuildFailedException(
                        $"[WeaponSystem] Missing WeaponData for weapon_id: {id}");
            }

            // 检查 Legacy 模式是否遗漏爆炸物
            CheckLegacyModeCompleteness(library);
        }
    }

    private void CheckLegacyModeCompleteness(WeaponTemplateLibrary library)
    {
        // Legacy 字段中缺少 GasolineCan 和 PropaneTank
        // 如果使用 Legacy 字段（列表为空），发出警告
        var usingLegacyFields = (library.hotWeapons == null || library.hotWeapons.Count == 0) &&
                                (library.environmentalWeapons == null || library.environmentalWeapons.Count == 0);

        if (usingLegacyFields)
        {
            // 检查 Legacy 字段中是否包含 GasolineCan 和 PropaneTank
            // 由于 Legacy 字段使用 PascalCase 命名（如 GasolineCan），我们需要检查 WeaponData 引用
            // 如果需要支持这两个爆炸物，必须迁移到列表方式
            UnityEngine.Debug.LogWarning(
                "[WeaponSystem] Using Legacy field assignment mode. " +
                "Note: GasolineCan and PropaneTank are NOT available in Legacy fields. " +
                "If you need these explosives, migrate to the list-based approach (hotWeapons/environmentalWeapons).");
        }
    }
}
#endif
```

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
    public float baseDamageValue;    // 基础伤害数值（最终伤害值，已含所有修正）
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
    public int ammo_capacity;        // 弹匣容量
    public int current_ammo;         // 当前弹药数
    public float reload_time;         // 换弹时间（秒）
}

// 爆炸组件（爆炸物）
[System.Serializable]
public class ExplosiveComponent
{
    public float blast_radius;           // 爆炸半径（米）
    public float lethal_radius_ratio;    // 致死半径比例（默认0.3）
    public DetonationType detonation_type; // 定义见 shared-types.md §3.1.3
    public bool is_controllable;

    /// <summary>
    /// 可选的 baseDamageValue 覆盖值。
    /// 如果不设置（<= 0），则使用 DamageComponent.baseDamageValue。
    /// 用于爆炸物有独立伤害配置的场景。
    /// </summary>
    [Tooltip("可选：爆炸物独立的基础伤害数值。如果 <= 0，则使用 DamageComponent.baseDamageValue")]
    public float override_base_damage;
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
    public WeaponData Brick;              // PascalCase 命名（兼容 C# 规范）
    public WeaponData MetalPipe;
    public WeaponData FireExtinguisher;
    public WeaponData GlassBulb;
    public WeaponData Wire;
    public WeaponData BrokenGlass;
    public WeaponData Pistol;
    public WeaponData Rifle;
    public WeaponData Shotgun;
    public WeaponData Grenade;
    public WeaponData C4;

    // 按 ID 查询模板（返回 null 如果不存在，不会抛异常）
    public WeaponData GetTemplate(string weapon_id)
    {
        if (TryGetTemplate(weapon_id, out var template))
            return template;
        return null;
    }

    /// <summary>
    /// 尝试获取武器模板（推荐方法）
    /// </summary>
    /// <param name="weapon_id">武器 ID</param>
    /// <param name="template">输出模板（如果存在）</param>
    /// <returns>true = 模板存在，false = 模板不存在或 weapon_id 为空</returns>
    /// <remarks>
    /// <b>设计理由</b>：此方法明确区分"不存在"和"初始化失败"两种情况。
    /// 当返回 false 时，调用方可通过 <c>template == null</c> 判断为"不存在"；
    /// 若初始化失败（_initialized == false 但仍然无法获取），则应抛出异常而非静默返回。
    /// </remarks>
    public bool TryGetTemplate(string weapon_id, out WeaponData template)
    {
        template = null;

        if (string.IsNullOrEmpty(weapon_id))
        {
            Debug.LogWarning($"[WeaponTemplateLibrary] TryGetTemplate called with null/empty weapon_id");
            return false;
        }

        // 防御性检查：确保映射表已初始化
        if (!_initialized)
            BuildLookupMaps();

        // 启动时构建映射表，运行时 O(1) 查询
        // 热武器
        if (_hotWeaponMap.TryGetValue(weapon_id, out var hotWeapon))
        {
            template = hotWeapon;
            return true;
        }

        // 环境物件
        if (_environmentalMap.TryGetValue(weapon_id, out var envWeapon))
        {
            template = envWeapon;
            return true;
        }

        return false;
    }

    // 按类别映射到 WeaponData（供环境交互系统调用）
    public WeaponData GetTemplateByCategory(ObjectCategory category)
    {
        // 使用映射表查找（支持新旧两种数据源）
        var weaponId = CategoryToWeaponId(category);
        return weaponId != null ? GetTemplate(weaponId) : null;
    }

    /// <summary>
    /// 将 ObjectCategory 转换为对应默认武器 ID（数据驱动方式）
    /// </summary>
    /// <remarks>
    /// <b>设计理由</b>：相较于硬编码 switch 语句，数据驱动方式：
    /// 1. 新增 ObjectCategory 枚举值时无需修改此方法（遵循开闭原则）
    /// 2. 映射关系集中管理，便于维护
    /// 3. 支持运行时动态配置（如通过 ScriptableObject 定义特殊映射）
    ///
    /// <b>默认映射表</b>：
    /// | ObjectCategory | weapon_id |
    /// |----------------|-----------|
    /// | Brick | brick |
    /// | MetalPipe | metal_pipe |
    /// | FireExtinguisher | fire_extinguisher |
    /// | GlassBulb | glass_bulb |
    /// | Wire | wire |
    /// | BrokenGlass | broken_glass |
    /// | Pistol | pistol |
    /// | Rifle | rifle |
    /// | Shotgun | shotgun |
    /// | Grenade | grenade |
    /// | C4 | c4 |
    /// | GasolineCan | gasoline_can |
    /// | PropaneTank | propane_tank |
    ///
    /// <b>扩展方式</b>：如需新增映射，可通过 Inspector 配置 categoryToWeaponIdMapping 列表，
    /// 或重写 <c>CategoryToWeaponId</c> 方法自定义映射逻辑。
    /// </remarks>
    private string CategoryToWeaponId(ObjectCategory category)
    {
        if (_categoryToWeaponIdMap == null)
            BuildCategoryToWeaponIdMap();

        return _categoryToWeaponIdMap.TryGetValue(category, out var weaponId) ? weaponId : null;
    }

    /// <summary>
    /// 类别到武器 ID 的映射表（支持运行时配置）
    /// </summary>
    [Tooltip("ObjectCategory 到 weapon_id 的映射表，可通过 Inspector 配置")]
    [SerializeField]
    private List<CategoryWeaponMapping> categoryToWeaponIdMapping = new();

    /// <summary>
    /// 运行时映射缓存（避免每次查询都遍历列表）
    /// </summary>
    private Dictionary<ObjectCategory, string> _categoryToWeaponIdMap;

    /// <summary>
    /// 类别-武器映射结构
    /// </summary>
    [System.Serializable]
    public class CategoryWeaponMapping
    {
        public ObjectCategory category;
        public string weaponId;
    }

    /// <summary>
    /// 构建类别到武器 ID 的映射缓存
    /// </summary>
    /// <remarks>
    /// 优先使用 Inspector 配置的映射表（categoryToWeaponIdMapping），
    /// 如果为空则回退到硬编码的默认映射（向后兼容）。
    /// </remarks>
    private void BuildCategoryToWeaponIdMap()
    {
        _categoryToWeaponIdMap = new Dictionary<ObjectCategory, string>();

        // 如果 Inspector 配置了映射，使用配置的映射
        if (categoryToWeaponIdMapping != null && categoryToWeaponIdMapping.Count > 0)
        {
            foreach (var mapping in categoryToWeaponIdMapping)
            {
                if (mapping.category != ObjectCategory.Unknown)
                    _categoryToWeaponIdMap[mapping.category] = mapping.weaponId;
            }
        }
        else
        {
            // 回退到硬编码默认映射（确保基本功能可用）
            _categoryToWeaponIdMap[ObjectCategory.Brick] = "brick";
            _categoryToWeaponIdMap[ObjectCategory.MetalPipe] = "metal_pipe";
            _categoryToWeaponIdMap[ObjectCategory.FireExtinguisher] = "fire_extinguisher";
            _categoryToWeaponIdMap[ObjectCategory.GlassBulb] = "glass_bulb";
            _categoryToWeaponIdMap[ObjectCategory.Wire] = "wire";
            _categoryToWeaponIdMap[ObjectCategory.BrokenGlass] = "broken_glass";
            _categoryToWeaponIdMap[ObjectCategory.Pistol] = "pistol";
            _categoryToWeaponIdMap[ObjectCategory.Rifle] = "rifle";
            _categoryToWeaponIdMap[ObjectCategory.Shotgun] = "shotgun";
            _categoryToWeaponIdMap[ObjectCategory.Grenade] = "grenade";
            _categoryToWeaponIdMap[ObjectCategory.C4] = "c4";
            _categoryToWeaponIdMap[ObjectCategory.GasolineCan] = "gasoline_can";
            _categoryToWeaponIdMap[ObjectCategory.PropaneTank] = "propane_tank";
        }
    }

#if UNITY_EDITOR
    [ContextMenu("Rebuild Category-Weapon Mapping")]
    private void RebuildCategoryWeaponMapping()
    {
        BuildCategoryToWeaponIdMap();
    }
#endif

    // 启动时构建 ID → WeaponData 映射表
    private Dictionary<string, WeaponData> _hotWeaponMap = new();
    private Dictionary<string, WeaponData> _environmentalMap = new();
    private bool _initialized = false;

    /// <summary>
    /// 存储重复的 weapon_id（用于警告/Error）
    /// </summary>
    private List<string> _duplicateWeaponIds = new();

    /// <summary>
    /// 调用层级说明：
    /// - Play Mode: Awake() → BuildLookupMaps() → _initialized = true
    /// - Editor Inspector 修改: OnValidate() → delayCall → BuildLookupMaps()
    /// - Editor ContextMenu: RebuildLookupMaps() → BuildLookupMaps()
    ///
    /// 注意：OnValidate 使用 delayCall 异步调用，避免在 Inspector 修改过程中过早触发。
    /// 在 Play Mode 下，Awake() 会首先执行并设置 _initialized = true，
    /// 后续 OnValidate 的 delayCall 会检查 _initialized 状态，如果已初始化则直接返回。
    /// </summary>
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
        _duplicateWeaponIds.Clear();

        void Register(WeaponData data)
        {
            if (data == null) return;
            if (string.IsNullOrEmpty(data.weapon_id))
            {
                Debug.LogWarning($"[WeaponTemplateLibrary] WeaponData with null/empty weapon_id found, skipping registration");
                return;
            }

            // 检查 weapon_id 唯一性
            if (data.category == WeaponCategory.HotWeapon)
            {
                if (_hotWeaponMap.ContainsKey(data.weapon_id))
                {
                    Debug.LogWarning($"[WeaponTemplateLibrary] Duplicate weapon_id '{data.weapon_id}' found in hotWeapons. Last occurrence will be used.");
                    _duplicateWeaponIds.Add(data.weapon_id);
                }
                _hotWeaponMap[data.weapon_id] = data;
            }
            else
            {
                if (_environmentalMap.ContainsKey(data.weapon_id))
                {
                    Debug.LogWarning($"[WeaponTemplateLibrary] Duplicate weapon_id '{data.weapon_id}' found in environmentalWeapons. Last occurrence will be used.");
                    _duplicateWeaponIds.Add(data.weapon_id);
                }
                _environmentalMap[data.weapon_id] = data;
            }
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
            Register(Pistol);
            Register(Rifle);
            Register(Shotgun);
            Register(Grenade);
            Register(C4);
            Register(Brick);
            Register(MetalPipe);
            Register(FireExtinguisher);
            Register(GlassBulb);
            Register(Wire);
            Register(BrokenGlass);
            // ⚠️ 注意：GasolineCan 和 PropaneTank 在 Legacy 字段中不存在
            // 如果需要支持这些爆炸物，请使用列表方式（hotWeapons/environmentalWeapons）添加
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
    private string _equippedHotWeaponId = string.Empty;

    // WeaponTemplateLibrary 作为状态机的内部依赖，确保热武器独占性检查始终生效
    private WeaponTemplateLibrary _library;

    public WeaponStateMachine(WeaponTemplateLibrary library)
    {
        _library = library ?? throw new System.ArgumentNullException(nameof(library));
    }

    /// <summary>
    /// 判断指定武器是否为热武器
    /// </summary>
    private bool IsHotWeapon(string weapon_id)
    {
        var template = _library?.GetTemplate(weapon_id);
        return template?.category == WeaponCategory.HotWeapon;
    }

    /// <summary>
    /// 设置武器状态（热武器独占性检查内置于状态机，无需外部调用方传参）
    /// </summary>
    /// <remarks>
    /// <b>热武器独占性保证</b>：状态机内部维护 <c>_equippedHotWeaponId</c>，
    /// 当任意热武器被设为 Equipped 时，之前处于 Equipped 状态的热武器自动转为 Holstered。
    /// 此逻辑在状态机内部执行，调用方无需感知。
    /// </remarks>
    public void SetState(string weapon_id, WeaponState newState)
    {
        var oldState = _weaponStates.GetValueOrDefault(weapon_id, WeaponState.Stored);

        // 热武器独占性检查（内置于状态机）
        if (newState == WeaponState.Equipped && IsHotWeapon(weapon_id))
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
    // base_damage 来源：统一使用 DamageComponent 的 baseDamageValue（由设计文档定义，如 C4=150，手榴弹=80）
    EventBus.Instance.Publish(new ExplosionEvent
    {
        position = position,
        radius = explosive.blast_radius,
        lethal_ratio = explosive.lethal_radius_ratio,
        base_damage = damage.baseDamageValue  // 统一使用 DamageComponent 的 baseDamageValue
    });

    // 触发音效和视觉反馈
    EventBus.Instance.Publish(new WeaponUsedEvent
    {
        weapon_id = weapon_id,
        usage_type = "explosion"
    });
}
```

**⚠️ baseDamageValue 来源说明**：
- `ExplosionEvent.base_damage` 值来源于 `DamageComponent.baseDamageValue`
- `ExplosiveComponent` 不单独存储 `baseDamageValue`，只存储爆炸参数（blast_radius、lethal_ratio）
- 设计确保 `DamageComponent.baseDamageValue` 与爆炸物配置同步

### 5. 投掷物命中判定

```csharp
// ThrowableHandler.cs
public class ThrowableHandler
{
    private readonly WeaponTuningSO _tuning;

    public ThrowableHandler(WeaponTuningSO tuning)
    {
        _tuning = tuning ?? throw new System.ArgumentNullException(nameof(tuning));
    }

    /// <summary>
    /// 计算投掷物命中概率
    /// </summary>
    /// <param name="range">实际投掷距离</param>
    /// <param name="melee_range">近战判定距离</param>
    /// <param name="throw_range">最大投掷距离</param>
    /// <returns>命中概率 [0, 1]</returns>
    public float CalculateHitChance(float range, float melee_range, float throw_range)
    {
        // 近战范围内必中
        if (range <= melee_range)
            return 1.0f;

        // 超过最大投掷距离，返回最小概率
        if (range >= throw_range)
            return _tuning?.minHitChance ?? 0.3f;

        // 从 TuningSO 获取最小命中概率配置
        float minChance = _tuning?.minHitChance ?? 0.3f;

        // 线性插值：近距离（melee_range）为 100%，远距离（throw_range）为 minChance
        float t = (range - melee_range) / (throw_range - melee_range);
        float hitChance = Mathf.Lerp(1.0f, minChance, t);
        return Mathf.Clamp(hitChance, minChance, 1.0f);
    }
}
```

> **配置来源**：`min_hit_chance` 由 `WeaponTuningSO.minHitChance`（见 §8）配置。
> 未配置时使用默认值 0.3f。

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
/// <param name="layerMask">碰撞检测层掩码。配置于 WeaponTuningSO.throwableLayerMask，包含 Environment（环境物件）、Destructible（可破坏物）等投掷物应检测的层级。</param>
    /// <returns>命中点位置（未命中返回 null）</returns>
    /// <remarks>
    /// <b>步长计算说明</b>：
    /// 步长通过 <c>Mathf.Clamp(0.05f / (speed / 5f), 0.01f, 0.05f)</c> 计算。
    /// - 速度 5m/s 时步长为 50ms（Nyquist 采样）
    /// - 速度 25m/s 时步长为 10ms（最小步长，防止漏检）
    /// - 速度更快时步长仍为 10ms（钳制在最大步长 50ms 与最小步长 10ms 之间）
    ///
    /// <b>调用限制</b>：建议投掷速度控制在 2.5m/s ~ 25m/s 范围内。
    /// 低于 2.5m/s 时步长固定为 50ms（可能漏检超近目标）；
    /// 高于 25m/s 时步长仍为 10ms（计算量增加但不漏检）。
    /// </remarks>
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

### 6.1 ThrowableHandler / ThrowableTrajectory 归属说明

> **⚠️ 接口声明**：以下接口供其他系统调用
>
> **跨系统调用方式**：Gritty Takedowns 等其他系统需要使用投掷物判定逻辑时，应通过以下方式：
> 1. **WeaponQueryRequest/WeaponQueryResponse（事件驱动）**：通过事件总线查询 WeaponData
> 2. **WeaponUsedEvent（事件驱动）**：订阅 WeaponSystem 发布的相关事件
>
> **设计理由**：投掷物判定涉及武器配置（throw_range、min_hit_chance 等），这些参数属于 WeaponSystem 的职责范围。事件驱动方式保持系统间松耦合，与 shared-types.md §3.8 定义的契约一致。

#### 6.1.1 WeaponQuery 接口（事件驱动方式）

> **⚠️ 接口实现方式统一**：Weapon System 采用**事件驱动**方式提供 WeaponQuery 接口。
> 与 Gritty Takedowns（ADR-0011）的 `WeaponQueryRequest/WeaponQueryResponse`（shared-types.md §3.8）保持一致。
>
> **设计理由**：
> - 事件驱动保持系统间松耦合
> - 与 shared-types.md 中定义的 `WeaponQueryRequest/WeaponQueryResponse` 契约一致
> - 便于跨系统追踪和调试

```csharp
// WeaponSystem.cs - 订阅 WeaponQueryRequest
public void OnWeaponQueryRequest(WeaponQueryRequest request)
{
    // ⚠️ 验证请求的 weapon_id 不为空
    if (string.IsNullOrEmpty(request.weapon_id))
    {
        Debug.LogWarning($"[WeaponSystem] WeaponQueryRequest with null/empty weapon_id ignored");
        return;
    }

    var weaponData = GetWeaponData(request.weapon_id);

    // 如果模板存在，验证返回的 weapon_id 与请求的匹配
    // 防止数据配置错误导致返回错误数据
    if (weaponData != null && weaponData.weapon_id != request.weapon_id)
    {
        Debug.LogError($"[WeaponSystem] WeaponData weapon_id mismatch: " +
            $"requested '{request.weapon_id}', got '{weaponData.weapon_id}'. " +
            $"This indicates a data configuration error in WeaponTemplateLibrary.");
    }

    EventBus.Instance.Publish(new WeaponQueryResponse
    {
        weapon_id = request.weapon_id,
        data = weaponData
    });
}

// WeaponSystem.cs - 查询接口（内部实现）
public WeaponData GetWeaponData(string weapon_id)
{
    return _library?.GetTemplate(weapon_id);
}
```

> **调用方式**：Gritty Takedowns（或其他系统）通过事件总线发送 `WeaponQueryRequest`，
> Weapon System 订阅并返回 `WeaponQueryResponse`。
> 详见 shared-types.md §3.8 和 ADR-0011 §7。

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

### 8. 调参配置（WeaponTuningSO）

```csharp
// Config/WeaponTuningSO.cs
[CreateAssetMenu(menuName = "Game/Weapons/Tuning")]
public class WeaponTuningSO : ScriptableObject
{
    [Header("投掷物参数")]
    [Tooltip("投掷物最小命中概率（默认 0.3）")]
    public float minHitChance = 0.3f;

    [Header("远程伤害衰减参数")]
    [Tooltip("中距离阈值（米），默认值 15.0f")]
    public float MediumRangeThreshold = 15.0f;

    [Tooltip("近距离阈值上限（米），默认值 8.0f")]
    public float CloseRangeThreshold = 8.0f;

    [Tooltip("远距离伤害倍率衰减下限，默认 0.7f（70%% 伤害）")]
    public float FarRangeMultiplierFloor = 0.7f;
}
```

#### 8.1 远程伤害衰减公式（RangeMultiplier）

> **适用场景**：热武器（FirearmDamage）远程伤害计算，由 Health System 在处理 DamageRequest 时调用。
> **调用位置**：GDD weapon-system.md 公式3 `FinalDamage = FirearmDamage × RangeMultiplier(distance)`

```csharp
/// <summary>
/// 计算远程伤害衰减倍率
/// </summary>
/// <param name="distance">目标距离（米）</param>
/// <param name="closeRange">近距离上限（米），默认 8.0f</param>
/// <param name="mediumRange">中距离阈值（米），默认 15.0f</param>
/// <param name="farRangeMultiplierFloor">远距离衰减下限，默认 0.7f</param>
/// <returns>伤害倍率 [0.7f, 1.0f]</returns>
/// <remarks>
/// | 距离区间 | RangeMultiplier | 说明 |
/// |----------|-----------------|------|
/// | distance < closeRange (8m) | 1.0f | 近距离全额伤害 |
/// | closeRange <= distance < mediumRange (15m) | 线性插值 [1.0, 0.7] | 中距离线性衰减 |
/// | distance >= mediumRange (15m) | 0.7f | 远距离最低伤害 |
/// </remarks>
public static float RangeMultiplier(
    float distance,
    float closeRange = 8.0f,
    float mediumRange = 15.0f,
    float farRangeMultiplierFloor = 0.7f)
{
    if (distance < closeRange)
        return 1.0f;

    if (distance >= mediumRange)
        return farRangeMultiplierFloor;

    // 中距离线性插值：closeRange → mediumRange 对应 1.0 → farRangeMultiplierFloor
    float t = (distance - closeRange) / (mediumRange - closeRange);
    return Mathf.Lerp(1.0f, farRangeMultiplierFloor, t);
}
```

**距离区间速查表**：

| 区间 | 条件 | RangeMultiplier | 示例 |
|------|------|-----------------|------|
| 近距离 | distance < 8m | 1.0 | 8m 内全额伤害 |
| 中距离 | 8m <= distance < 15m | 线性插值 (1.0 → 0.7) | 10m 时约为 0.86 |
| 远距离 | distance >= 15m | 0.7 | 15m 及以上最低伤害 |

> **配置来源**：`MediumRangeThreshold`、`CloseRangeThreshold`、`FarRangeMultiplierFloor` 由 `WeaponTuningSO`（见 §8）配置。

### 9. 手柄震动反馈抽象层

```csharp
// HapticFeedbackConfig.cs
// 平台适配层抽象，支持 PS5/Xbox/Switch Pro/PC
using UnityEngine;

public interface IHapticFeedback
{
    void PlayFeedback(HapticPattern pattern, float intensity);
}

/// <summary>
/// 手柄震动反馈管理器
/// 支持通过构造函数注入平台特定实现，默认使用通用平台实现
/// </summary>
/// <remarks>
/// <b>依赖注入方式</b>：通过构造函数注入 <c>IHapticFeedback</c> 实现。
/// 如果不注入，则使用 <c>DefaultHapticFeedback</c>（PC/通用平台）。
///
/// <b>使用示例</b>：
/// <code>
/// // 注入 PS5 特定实现
/// var ps5Haptic = new PS5HapticFeedback();
/// var hapticManager = new HapticFeedbackManager(ps5Haptic);
///
/// // 使用默认实现（PC）
/// var defaultManager = new HapticFeedbackManager();
/// </code>
///
/// <b>Feature Layer 规范</b>：通过构造函数注入依赖，而非使用单例模式，
/// 便于单元测试时替换为 Mock 实现。
/// </remarks>
[Obsolete("Use HapticFeedbackManager from ADR-0020 instead. This implementation is deprecated.")]
public class HapticFeedbackManager : IHapticFeedback
{
    // 平台特定实现
    private readonly IHapticFeedback _implementation;

    public HapticFeedbackManager(IHapticFeedback implementation = null)
    {
        // 默认实现（PC/通用平台），可注入平台特定实现
        _implementation = implementation ?? new DefaultHapticFeedback();
    }

    public void PlayFeedback(HapticPattern pattern, float intensity)
    {
        _implementation?.PlayFeedback(pattern, intensity);
    }
}

/// <summary>
/// 默认震动反馈实现（PC/通用平台）
/// </summary>
public class DefaultHapticFeedback : IHapticFeedback
{
    public void PlayFeedback(HapticPattern pattern, float intensity)
    {
        // 平台特定实现（示例：PS5 DualSense）
        if (Application.platform == RuntimePlatform.PS5)
        {
            // PS5-specific haptic call
        }
        // 其他平台使用默认实现
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

// WeaponSystem.cs - 依赖注入示例
public class WeaponSystem
{
    private readonly HapticFeedbackManager _hapticManager;
    private readonly HapticFeedbackConfig _hapticConfig;

    /// <summary>
    /// 构造函数（依赖注入）
    /// </summary>
    /// <param name="library">武器模板库（必须）</param>
    /// <param name="hapticConfig">震动反馈配置（可选，为 null 时使用默认实现）</param>
    public WeaponSystem(WeaponTemplateLibrary library, HapticFeedbackConfig hapticConfig = null)
    {
        _library = library ?? throw new System.ArgumentNullException(nameof(library));
        _hapticConfig = hapticConfig;

        // 通过构造函数注入 HapticFeedbackManager，而非硬编码 new
        // 这样可以支持平台特定实现和单元测试 Mock
        _hapticManager = new HapticFeedbackManager(_hapticConfig);
    }

    public void TriggerHaptic(HapticPattern pattern)
    {
        var platform = Application.platform;
        var intensity = _hapticConfig != null
            ? _hapticConfig.GetIntensity(pattern, platform)
            : 0.5f;  // 默认强度
        _hapticManager.PlayFeedback(pattern, intensity);
    }
}
```

**相关文件**：
- `Assets/Game/Features/WeaponSystem/Config/WeaponTuningSO.cs`
- `Assets/Game/Features/WeaponSystem/Config/HapticFeedbackConfig.cs`

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
5. **投掷物命中判定**：远距离命中率不低于 `WeaponTuningSO.minHitChance`（默认 0.3f）
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
- [ADR-0020: Input System 输入系统架构](./adr-0020-input-system-architecture.md) — **HapticFeedbackManager 统一实现位置**
- [共享类型定义](./shared-types.md) — **DamageRequest、ExplosionEvent、ObjectCategory 等跨 ADR 类型统一定义在此**
- [Weapon System GDD](../../design/gdd/weapon-system.md) — 本 ADR 的设计依据
- [事件总线 ICD](../../engine-reference/event-bus-icd.md) — 事件定义的权威文档

## 附录：v1.4.x 修改日志

| 日期 | 版本 | 修改内容 | 评审修复 |
|------|------|---------|----------|
| 2026-04-14 | v1.4.3 | 新增 MediumRangeThreshold = 15.0f、CloseRangeThreshold = 8.0f、FarRangeMultiplierFloor = 0.7f 定义；新增 RangeMultiplier(distance) 函数及距离区间速查表（T-07 修复） | - |
| 2026-04-11 | v1.4.2 | IPreprocessBuild 中 weapon_id 重复升级为 Error | - |
| 2026-04-11 | v1.4.2 | Validation 工具添加 Legacy 模式 GasolineCan/PropaneTank 警告 | - |
| 2026-04-11 | v1.4.2 | BuildLookupMaps 调用层级说明补充 | - |
| 2026-04-11 | v1.4.1 | CategoryToWeaponId 改为数据驱动，支持配置化映射 | #1 修复 |
| 2026-04-11 | v1.4.1 | CalculateHitChance 边界计算修正 | #2 修复 |
| 2026-04-11 | v1.4.1 | BuildLookupMaps 添加 weapon_id 唯一性验证 | #3 修复 |
| 2026-04-11 | v1.4.1 | 新增 TryGetTemplate 方法，明确返回语义 | #5 修复 |
| 2026-04-11 | v1.4.1 | HapticFeedbackManager 依赖注入说明修正 | #8 修复 |
| 2026-04-11 | v1.4.1 | GasolineCan/PropaneTank 映射添加到默认映射表 | #9 修复 |
