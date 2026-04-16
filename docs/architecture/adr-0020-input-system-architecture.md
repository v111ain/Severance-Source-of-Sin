# ADR-0020: Input System 输入系统架构 (Input System Architecture)

## Status
**Proposed**

## Date
2026-04-11

## Last Updated
2026-04-13

## Context

### Problem Statement

ADR-0009 定义了 PlayerController 的移动状态机和锁定机制，但**输入处理层**的架构决策尚未明确。当前存在的问题：

1. **输入分散**：各系统自行处理输入（PlayerController、LOSSystem、GrittyTakedowns）
2. **键位硬编码**：输入配置难以调整
3. **平台差异**：PC/PS5 手柄键位映射未统一处理
4. **重映射支持**：玩家自定义键位缺乏统一方案
5. **输入优先级**：多个系统同时需要输入时缺乏仲裁

### Constraints

- **平台**：PC (Steam) & PS5
- **引擎**：Unity 6.3 LTS
- **包**：Unity.InputSystem (1.9.0+)
- **兼容**：需支持键鼠和手柄

### Requirements

- **必须**：统一输入抽象层（Input Abstraction）
- **必须**：支持输入重映射（Remapping）
- **必须**：支持输入优先级仲裁
- **必须**：支持平台特定输入（PC 键鼠 / PS5 手柄）
- **必须**：提供输入状态查询接口

---

## Decision

### 架构决策

采用 **分层输入架构**，将输入分为三层：

```
+-----------------------------------------------------------------------------+
|                         分层输入架构                                         |
+-----------------------------------------------------------------------------+
|                                                                             |
|   +---------------------------------------------------------------------+   |
|   |                    Input Abstraction Layer (输入抽象层)               |   |
|   |  - InputAction 包装                                                  |   |
|   |  - 平台差异屏蔽                                                      |   |
|   |  - InputContext 管理                                                 |   |
|   +---------------------------------------------------------------------+   |
|                                      |                                      |
|                                      v                                      |
|   +---------------------------------------------------------------------+   |
|   |                    Input Arbitration Layer (输入仲裁层)               |   |
|   |  - 输入优先级管理                                                     |   |
|   |  - 输入拦截/转发                                                      |   |
|   |  - ActionLock 集成                                                   |   |
|   +---------------------------------------------------------------------+   |
|                                      |                                      |
|                                      v                                      |
|   +---------------------------------------------------------------------+   |
|   |                    Input Consumer Layer (输入消费层)                  |   |
|   |  - PlayerController (移动)                                           |   |
|   |  - LOSSystem (专注监听)                                               |   |
|   |  - GrittyTakedowns (交互)                                            |   |
|   |  - UISystem (菜单)                                                   |   |
|   +---------------------------------------------------------------------+   |
|                                                                             |
+-----------------------------------------------------------------------------+
```

### 1. InputAction 包装

```csharp
// GameInputAction.cs
/// <summary>
/// 封装 Unity InputSystem 的 InputAction，提供统一的查询接口
/// 支持平台差异屏蔽和输入状态缓存
/// </summary>
public class GameInputAction
{
    /// <summary>
    /// 输入动作类型
    /// </summary>
    public enum InputActionType
    {
        /// <summary>连续值输入（如摇杆、WASD）</summary>
        Value,
        /// <summary>按钮式输入（按下/释放）</summary>
        Button
    }

    private InputAction _action;
    private InputAction _keyboardAction;    // 键盘专用
    private InputAction _gamepadAction;     // 手柄专用

    public string Name { get; }
    public InputActionType ActionType { get; private set; }

    // 当前帧状态（缓存，避免每帧多次查询）
    private bool _isPressed;
    private bool _wasPressed;
    private Vector2 _value;

    public bool IsPressed => _isPressed;
    public bool WasPressedThisFrame => _isPressed && !_wasPressed;
    public bool WasReleasedThisFrame => !_isPressed && _wasPressed;
    public Vector2 Value => _value;

    public GameInputAction(string name, InputActionType type)
    {
        Name = name;
        ActionType = type;
    }

    public void Initialize(InputAction keyboardAction, InputAction gamepadAction)
    {
        _keyboardAction = keyboardAction;
        _gamepadAction = gamepadAction;

        // 默认启用键盘
        _action = _keyboardAction;
        _action.Enable();
    }

    /// <summary>
    /// 切换输入设备
    /// </summary>
    public void SetInputDevice(InputDeviceType device)
    {
        if (_action != null)
            _action.Disable();

        InputAction newAction = device switch
        {
            InputDeviceType.Keyboard     => _keyboardAction,
            InputDeviceType.PS5Gamepad   => _gamepadAction,
            InputDeviceType.PS4Gamepad   => _gamepadAction,
            InputDeviceType.XboxGamepad  => _gamepadAction,
            InputDeviceType.GenericGamepad => _gamepadAction,
            _ => null  // 未知类型设为 null，后续统一处理
        };

        // 未知设备类型警告（仅在非键盘设备时警告）
        if (newAction == null && device != InputDeviceType.Keyboard)
        {
            Debug.LogWarning($"[GameInputAction] Unknown device type: {device}. Falling back to gamepad if available, else keyboard.");
        }

        // P0 修复：未知设备回退到手柄（而非键盘）
        // 因为如果玩家正在使用手柄，切换到键盘会导致控制突然跳变
        // 只有在手柄不可用时才回退到键盘
        _action = newAction ?? (_gamepadAction ?? _keyboardAction);

        if (_action != null)
        {
            _action.Enable();
            if (_action.bindings.Count == 0)
                Debug.LogWarning($"[GameInputAction] No bindings for {Name} on {device}");
        }
        else
        {
            Debug.LogWarning($"[GameInputAction] No valid action for {Name} on device {device}");
        }

        // 切换设备时重置状态，避免旧设备状态残留导致第一帧误读
        _isPressed = false;
        _wasPressed = false;
        _value = Vector2.zero;
    }

    /// <summary>
    /// 每帧更新（由 InputManager 调用）
    /// 逻辑顺序：先保存上一帧状态，再读当前帧，确保 WasPressedThisFrame 计算正确
    /// </summary>
    public void Update()
    {
        bool previousPressed = _isPressed;

        if (_action != null && _action.enabled)
        {
            _isPressed = _action.IsPressed();
            _value = _action.ReadValue<Vector2>();
        }
        else
        {
            _isPressed = false;
            _value = Vector2.zero;
        }

        _wasPressed = previousPressed;
    }
}

// InputDeviceType.cs
// 定义位置：Assets/Game/Foundation/Shared/Types/InputDeviceType.cs
public enum InputDeviceType
{
    Keyboard,
    PS5Gamepad,
    PS4Gamepad,
    XboxGamepad,
    GenericGamepad
}

// InputDeviceHelper.cs
/// <summary>
/// InputDeviceType 辅助判断方法
/// 用于从 Unity InputSystem 的 Gamepad 实例推断具体设备类型
/// </summary>
public static class InputDeviceHelper
{
    /// <summary>
    /// 已知设备 VID/PID 列表（可通过配置更新）
    /// 当 PS5 系统更新后新的手柄型号后，可通过 InputConfigSO 更新此列表
    /// 格式：(VID, PID, DeviceType)
    /// </summary>
    private static List<(string vid, string pid, InputDeviceType type)> _knownDevices = new()
    {
        // Sony
        ("054c", "0ce6", InputDeviceType.PS5Gamepad),  // DualSense
        ("054c", "0df2", InputDeviceType.PS5Gamepad),  // DualSense Edge
        ("054c", "05c4", InputDeviceType.PS4Gamepad), // DualShock 4
        // Microsoft
        ("045e", null, InputDeviceType.XboxGamepad),   // Xbox Core
        ("045e", "02fd", InputDeviceType.XboxGamepad), // Xbox Series
    };

    /// <summary>
    /// 更新已知设备列表（当 PS5 系统更新后调用此方法添加新设备）
    /// </summary>
    /// <param name="vid">厂商 ID</param>
    /// <param name="pid">产品 ID（可为空字符串表示匹配所有该厂商设备）</param>
    /// <param name="type">设备类型</param>
    public static void RegisterKnownDevice(string vid, string pid, InputDeviceType type)
    {
        _knownDevices.Add((vid, pid, type));
    }

    /// <summary>
    /// 从配置加载已知设备列表（可由 InputConfigSO 覆盖）
    /// </summary>
    public static void LoadKnownDevicesFromConfig(InputConfigSO config)
    {
        if (config == null) return;
        // 配置中可以定义额外的已知设备，格式：vid,pid,type
        // 具体实现取决于 InputConfigSO 的字段设计
    }

    /// <summary>
    /// 判断是否为 PS5 手柄（DualSense / DualSense Edge）
    /// 采用多重检测策略：VID/PID（最高优先级）→ 设备名 → DeviceClass → Xbox 排除
    /// </summary>
    public static bool IsPS5Controller(Gamepad gamepad)
    {
        if (gamepad == null || gamepad.device == null || gamepad.device.description == null)
            return false;

        var id = gamepad.device.description.hardware;
        var name = gamepad.device.description.name;
        var deviceClass = gamepad.device.description.deviceClass;

        // 方法1：精确匹配 VID/PID（最可靠，优先检查）
        if (MatchKnownDevice(id, InputDeviceType.PS5Gamepad))
            return true;

        // 方法2：通过设备名称匹配
        if (name.Contains("DualSense") || name.Contains("DualSense Edge"))
            return true;

        // 方法3：通过设备类型判断（InputSystem 1.5+）
        if (deviceClass == GamepadDeviceClass.PS5)
            return true;

        // 方法4：Xbox 手柄排除（必须优先于方法5，避免误判）
        if (name.Contains("Xbox", StringComparison.OrdinalIgnoreCase) || deviceClass == GamepadDeviceClass.Xbox)
            return false;

        // 方法5：通过设备功能特征判断（兜底逻辑）
        // 【修复P1-5 PS5判断逻辑】原实现：任何支持haptic的非Xbox手柄都返回true，会误判Switch Pro Controller等
        // 修复后：方法5作为"未知设备"处理，返回false并记录警告，而非直接判定为PS5
        if (gamepad.CanProduceHaptics())
        {
            var capabilities = gamepad.GetHapticCapabilities();
            if (capabilities.supportsDualRumble && !string.IsNullOrEmpty(id))
            {
                // 已排除 Xbox，继续检查其他已知厂商
                // Microsoft VID: 045e, 2dc8 (Xbox), 0738 (Saitek), 1038 (SteelSeries)
                if (id.Contains("045e") || id.Contains("2dc8") || id.Contains("0738") || id.Contains("1038"))
                    return false;
                // 【修复】不再直接返回true，而是视为未知设备
                // 由后续的"未知设备"处理逻辑统一管理
            }
        }

        // 未知设备：限频警告（每 30 秒最多一次），避免刷屏
        if (!_loggedUnknownDevices.Contains(name))
        {
            Debug.LogWarning($"[InputDeviceHelper] Unknown controller: {name} (ID: {id}). Consider updating known devices list via RegisterKnownDevice().");
            _loggedUnknownDevices.Add(name);
            _lastLogTime = Time.time;
        }
        else if (Time.time - _lastLogTime >= UNKNOWN_DEVICE_LOG_INTERVAL)
        {
            // 同一设备重复出现时，按间隔重新记录（避免刷屏）
            Debug.LogWarning($"[InputDeviceHelper] Unknown controller (repeating): {name} (ID: {id})");
            _lastLogTime = Time.time;
        }
        return false;
    }

    /// <summary>
    /// 匹配已知设备（VID/PID）
    /// </summary>
    private static bool MatchKnownDevice(string hardwareId, InputDeviceType type)
    {
        foreach (var (vid, pid, deviceType) in _knownDevices)
        {
            if (deviceType != type) continue;
            if (!hardwareId.Contains(vid)) continue;
            if (string.IsNullOrEmpty(pid) || hardwareId.Contains(pid))
                return true;
        }
        return false;
    }

    private static readonly HashSet<string> _loggedUnknownDevices = new();
    private static float _lastLogTime = 0f;
    private const float UNKNOWN_DEVICE_LOG_INTERVAL = 30f;

    /// <summary>
    /// 判断是否为 PS4 手柄（DualShock 4）
    /// </summary>
    public static bool IsPS4Controller(Gamepad gamepad)
    {
        if (gamepad == null || gamepad.device == null || gamepad.device.description == null)
            return false;

        var id = gamepad.device.description.hardware;
        var name = gamepad.device.description.name;
        var deviceClass = gamepad.device.description.deviceClass;

        // 方法1：VID/PID 匹配
        if (MatchKnownDevice(id, InputDeviceType.PS4Gamepad))
            return true;

        // 方法2：通过设备名称匹配
        if (name.Contains("DualShock") || name.Contains("DS4"))
            return true;

        // 方法3：通过设备类型判断
        if (deviceClass == GamepadDeviceClass.PS4)
            return true;

        return false;
    }

    /// <summary>
    /// 判断是否为 Xbox 手柄
    /// </summary>
    public static bool IsXboxController(Gamepad gamepad)
    {
        if (gamepad == null || gamepad.device == null || gamepad.device.description == null)
            return false;

        var id = gamepad.device.description.hardware;
        var name = gamepad.device.description.name;
        var deviceClass = gamepad.device.description.deviceClass;

        // 方法1：VID/PID 匹配
        if (MatchKnownDevice(id, InputDeviceType.XboxGamepad))
            return true;

        // 方法2：通过设备名称匹配
        if (name.Contains("Xbox", StringComparison.OrdinalIgnoreCase))
            return true;

        // 方法3：通过设备类型判断
        if (deviceClass == GamepadDeviceClass.Xbox)
            return true;

        return false;
    }

    /// <summary>
    /// 从 Gamepad 自动推断 InputDeviceType
    /// </summary>
    public static InputDeviceType InferDeviceType(Gamepad gamepad)
    {
        if (IsPS5Controller(gamepad))
            return InputDeviceType.PS5Gamepad;
        if (IsPS4Controller(gamepad))
            return InputDeviceType.PS4Gamepad;
        if (IsXboxController(gamepad))
            return InputDeviceType.XboxGamepad;

        return InputDeviceType.GenericGamepad;
    }
}
```

### 2. InputManager 核心

```csharp
// InputManager.cs
public class InputManager : MonoBehaviour
{
    public static InputManager Instance { get; private set; }

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        InitializeActions();
    }

    private void OnDestroy()
    {
        // 清理仲裁器状态，避免 Editor Play Mode 结束后优先级残留
        InputArbitrator.Instance.ForceReleaseAll();
        if (Instance == this)
            Instance = null;
    }

    // 玩家输入
    private GameInputAction _moveAction;
    private GameInputAction _sprintAction;
    private GameInputAction _crouchAction;
    private GameInputAction _actionAction;       // 交互/攻击
    private GameInputAction _focusAction;         // 专注监听
    private GameInputAction _weaponPrevAction;   // 上一武器
    private GameInputAction _weaponNextAction;   // 下一武器
    private GameInputAction _mapAction;          // 地图
    private GameInputAction _pauseAction;        // 暂停

    // UI 输入（用于菜单模式）
    private GameInputAction _uiNavigateAction;
    private GameInputAction _uiConfirmAction;
    private GameInputAction _uiCancelAction;

    // 当前输入设备
    public InputDeviceType CurrentDevice { get; private set; } = InputDeviceType.Keyboard;

    // 设备检测节流（避免每帧检测）
    private float _lastDeviceCheckTime = 0f;
    private const float DEVICE_CHECK_INTERVAL = 0.25f; // 每 250ms 检测一次

    private void InitializeActions()
    {
        // Movement — 必须使用 2D Vector Composite，直接 AddBinding 无法产生 Vector2
        _moveAction = new GameInputAction("Move", GameInputAction.InputActionType.Value);
        _moveAction.Initialize(
            CreateMoveAction("Move", isKeyboard: true),
            CreateMoveAction("Move", isKeyboard: false)
        );

        // Sprint
        _sprintAction = new GameInputAction("Sprint", GameInputAction.InputActionType.Button);
        _sprintAction.Initialize(
            CreateButtonAction("Sprint", "<Keyboard>/leftShift"),
            CreateButtonAction("Sprint", "<Gamepad>/leftTrigger")
        );

        // Crouch
        _crouchAction = new GameInputAction("Crouch", GameInputAction.InputActionType.Button);
        _crouchAction.Initialize(
            CreateButtonAction("Crouch", "<Keyboard>/leftCtrl"),
            CreateButtonAction("Crouch", "<Gamepad>/buttonB")
        );

        // Action (Interact/Attack)
        _actionAction = new GameInputAction("Action", GameInputAction.InputActionType.Button);
        _actionAction.Initialize(
            CreateButtonAction("Action", "<Keyboard>/e"),
            CreateButtonAction("Action", "<Gamepad>/buttonA")
        );

        // Focus
        _focusAction = new GameInputAction("Focus", GameInputAction.InputActionType.Button);
        _focusAction.Initialize(
            CreateButtonAction("Focus", "<Keyboard>/v"),
            CreateButtonAction("Focus", "<Gamepad>/buttonY")
        );

        // Map
        _mapAction = new GameInputAction("Map", GameInputAction.InputActionType.Button);
        _mapAction.Initialize(
            CreateButtonAction("Map", "<Keyboard>/tab"),
            CreateButtonAction("Map", "<Gamepad>/touchpadButton") // PS5 touchpad = 地图
        );

        // ⚠️ PS5 平台说明：
        // Sony PS5 提交要求：Touchpad 按钮在系统层面标记为 "Options" 按钮的替代，
        // 但在游戏内可用于自定义绑定。认证时需确保 touchpad 按钮可响应。
        //
        // **认证风险与缓解方案**：
        // - **Primary 方案**：使用 touchpad 按钮（"<Gamepad>/touchpadButton"）
        //   原因：更符合玩家直觉（Tap to view map）
        // - **Fallback 方案**：若 Sony 认证对 touchpad 映射有特殊要求导致认证失败，
        //   替换为 PS5 View 按钮（"<Gamepad>/view"）
        // 实际提交前需在 Player Settings > PS5 > Default Button Mapping 中验证按钮响应。

        // Pause
        _pauseAction = new GameInputAction("Pause", GameInputAction.InputActionType.Button);
        _pauseAction.Initialize(
            CreateButtonAction("Pause", "<Keyboard>/escape"),
            CreateButtonAction("Pause", "<Gamepad>/start")
        );

        // Weapon Switch
        _weaponPrevAction = new GameInputAction("WeaponPrev", GameInputAction.InputActionType.Button);
        _weaponPrevAction.Initialize(
            CreateButtonAction("WeaponPrev", "<Keyboard>/q"),
            CreateButtonAction("WeaponPrev", "<Gamepad>/leftShoulder")
        );

        _weaponNextAction = new GameInputAction("WeaponNext", GameInputAction.InputActionType.Button);
        _weaponNextAction.Initialize(
            CreateButtonAction("WeaponNext", "<Keyboard>/r"),
            CreateButtonAction("WeaponNext", "<Gamepad>/rightShoulder")
        );

        // UI Navigation — 同样需要 2D Composite
        _uiNavigateAction = new GameInputAction("UINavigate", GameInputAction.InputActionType.Value);
        _uiNavigateAction.Initialize(
            CreateMoveAction("UINavigate", isKeyboard: true),
            CreateMoveAction("UINavigate", isKeyboard: false)
        );

        _uiConfirmAction = new GameInputAction("UIConfirm", GameInputAction.InputActionType.Button);
        _uiConfirmAction.Initialize(
            CreateButtonAction("UIConfirm", "<Keyboard>/enter"),
            CreateButtonAction("UIConfirm", "<Gamepad>/buttonA")
        );

        _uiCancelAction = new GameInputAction("UICancel", GameInputAction.InputActionType.Button);
        _uiCancelAction.Initialize(
            CreateButtonAction("UICancel", "<Keyboard>/escape"),
            CreateButtonAction("UICancel", "<Gamepad>/buttonB")
        );

        DetectCurrentDevice();
    }

    /// <summary>
    /// 创建方向移动 InputAction（使用 2DVector Composite）
    /// 必须使用 Composite，否则 4 个单独 binding 无法合成为 Vector2
    /// </summary>
    private InputAction CreateMoveAction(string name, bool isKeyboard)
    {
        var action = new InputAction(name, UnityEngine.InputSystem.InputActionType.Value);
        if (isKeyboard)
        {
            action.AddCompositeBinding("2DVector")
                .With("Up",    "<Keyboard>/w")
                .With("Down",  "<Keyboard>/s")
                .With("Left",  "<Keyboard>/a")
                .With("Right", "<Keyboard>/d");
        }
        else
        {
            action.AddBinding("<Gamepad>/leftStick");
        }
        return action;
    }

    /// <summary>
    /// 创建按钮 InputAction（单个 binding）
    /// 注意：使用 UnityEngine.InputSystem.InputActionType 而非自定义枚举，避免类型冲突
    /// </summary>
    private InputAction CreateButtonAction(string name, string binding)
    {
        var action = new InputAction(name, UnityEngine.InputSystem.InputActionType.Button);
        action.AddBinding(binding);
        return action;
    }

    private void DetectCurrentDevice()
    {
        if (Keyboard.current != null && Keyboard.current.anyKey.wasPressedThisFrame)
        {
            SwitchDevice(InputDeviceType.Keyboard);
        }
        else if (Gamepad.current != null && Gamepad.current.allControls.Any(c => c.IsPressed()))
        {
            SwitchDevice(InputDeviceHelper.InferDeviceType(Gamepad.current));
        }
    }

    public void SwitchDevice(InputDeviceType device)
    {
        if (CurrentDevice == device) return;

        CurrentDevice = device;

        _moveAction.SetInputDevice(device);
        _sprintAction.SetInputDevice(device);
        _crouchAction.SetInputDevice(device);
        _actionAction.SetInputDevice(device);
        _focusAction.SetInputDevice(device);
        _mapAction.SetInputDevice(device);
        _pauseAction.SetInputDevice(device);
        _weaponPrevAction.SetInputDevice(device);
        _weaponNextAction.SetInputDevice(device);

        EventBus.Instance.Publish(new InputDeviceChangedEvent { Device = device });
    }

    private void Update()
    {
        _moveAction.Update();
        _sprintAction.Update();
        _crouchAction.Update();
        _actionAction.Update();
        _focusAction.Update();
        _mapAction.Update();
        _pauseAction.Update();
        _weaponPrevAction.Update();
        _weaponNextAction.Update();
        _uiNavigateAction.Update();
        _uiConfirmAction.Update();
        _uiCancelAction.Update();

        // 节流检测设备切换（避免每帧检测）
        // 使用 Time.time >= lastCheck + interval 而非 time - lastCheck >= interval
        // 避免浮点精度累积误差导致提前触发
        if (Time.time >= _lastDeviceCheckTime + DEVICE_CHECK_INTERVAL)
        {
            _lastDeviceCheckTime = Time.time;
            DetectCurrentDevice();
        }
    }

    // ==================== 对外接口 ====================

    // 玩家移动
    public Vector2 GetMoveInput() => _moveAction.Value;
    public bool IsSprintHeld() => _sprintAction.IsPressed;
    public bool WasCrouchToggled() => _crouchAction.WasPressedThisFrame;
    public bool WasActionPressed() => _actionAction.WasPressedThisFrame;
    public bool IsFocusHeld() => _focusAction.IsPressed;

    // 武器切换
    public bool WasWeaponPrevPressed() => _weaponPrevAction.WasPressedThisFrame;
    public bool WasWeaponNextPressed() => _weaponNextAction.WasPressedThisFrame;

    // 系统
    public bool WasMapPressed() => _mapAction.WasPressedThisFrame;
    public bool WasPausePressed() => _pauseAction.WasPressedThisFrame;

    // UI
    public Vector2 GetUINavigateInput() => _uiNavigateAction.Value;
    public bool WasUIConfirmPressed() => _uiConfirmAction.WasPressedThisFrame;
    public bool WasUICancelPressed() => _uiCancelAction.WasPressedThisFrame;
}

public struct InputDeviceChangedEvent
{
    public InputDeviceType Device;
}

// HapticFeedbackManager.cs
/// <summary>
/// 手柄 haptic 反馈管理器 [已修复]
///
/// **单例模式说明**：
/// 与 EventBus（ScriptableObject 单例）、ResourceManager（MonoBehaviour 单例）不同，
/// HapticFeedbackManager 使用 MonoBehaviour 单例模式，因为：
/// 1. 需要通过协程实现定时停止 haptic（StopPCHapticDelayed 等）
/// 2. 协程需要 MonoBehaviour 的 StartCoroutine 支持
/// 3. 持久化使用 DontDestroyOnLoad，与 InputManager 生命周期绑定
///
/// **self-host 模式**：
/// HapticFeedbackManager 在自身 Awake 中设置协程宿主为自身实例，无需外部注入。
///
/// **IsPS5Controller 委托**：
/// 复用 InputDeviceHelper.IsPS5Controller，消除与 InputDeviceHelper 的逻辑重复。
/// </summary>
public class HapticFeedbackManager : MonoBehaviour
{
    public static HapticFeedbackManager Instance { get; private set; }

    // Haptic 反馈预设
    public enum HapticPreset
    {
        None,
        Light,       // 轻度反馈（UI 交互）
        Medium,      // 中度反馈（脚步、碰撞）
        Heavy,       // 重度反馈（攻击、爆炸）
        Pulse,       // 脉冲反馈（警告、警戒）
        Continuous   // 连续反馈（移动、冲刺）
    }

    /// <summary>
    /// Haptic 反馈上下文枚举（替代字符串硬编码，类型安全）
    /// </summary>
    public enum HapticContext
    {
        UI_Select,
        UI_Back,
        Footstep,
        Sprint,
        Crouch,
        Land,
        AttackMelee,
        AttackRange,
        HitReceived,
        Explosion,
        StealthKill,
        AlertNPC,
        NPCSuspicious,
        NPCSearch,
        Death
    }

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    private void OnDestroy()
    {
        if (Instance == this)
            Instance = null;
    }

    // 各平台 Haptic 参数配置
    private struct HapticParams
    {
        public float LowFrequencySpeed;  // 低频振动速度 (0.0-1.0)
        public float HighFrequencySpeed; // 高频振动速度 (0.0-1.0)
        public float Intensity;           // 整体强度 (0.0-1.0)
        public float Duration;           // 持续时间（秒）
    }

    private static readonly Dictionary<HapticPreset, HapticParams> _pcHapticParams = new()
    {
        { HapticPreset.None,      new HapticParams { LowFrequencySpeed = 0, HighFrequencySpeed = 0, Intensity = 0, Duration = 0 } },
        { HapticPreset.Light,     new HapticParams { LowFrequencySpeed = 0.3f, HighFrequencySpeed = 0.3f, Intensity = 0.4f, Duration = 0.1f } },
        { HapticPreset.Medium,    new HapticParams { LowFrequencySpeed = 0.5f, HighFrequencySpeed = 0.5f, Intensity = 0.6f, Duration = 0.15f } },
        { HapticPreset.Heavy,     new HapticParams { LowFrequencySpeed = 0.8f, HighFrequencySpeed = 0.8f, Intensity = 1.0f, Duration = 0.2f } },
        { HapticPreset.Pulse,     new HapticParams { LowFrequencySpeed = 1.0f, HighFrequencySpeed = 0.5f, Intensity = 0.8f, Duration = 0.1f } },
        { HapticPreset.Continuous, new HapticParams { LowFrequencySpeed = 0.4f, HighFrequencySpeed = 0.2f, Intensity = 0.5f, Duration = 0.05f } }
    };

    // PS5 DualSense 特定参数
    private static readonly Dictionary<HapticPreset, (float gain, float intensity, float brightness)> _ps5HapticParams = new()
    {
        { HapticPreset.None,      (0f, 0f, 0f) },
        { HapticPreset.Light,     (0.4f, 0.4f, 0.5f) },
        { HapticPreset.Medium,    (0.6f, 0.6f, 0.7f) },
        { HapticPreset.Heavy,     (1.0f, 1.0f, 1.0f) },
        { HapticPreset.Pulse,     (0.8f, 0.5f, 1.0f) },
        { HapticPreset.Continuous, (0.5f, 0.3f, 0.6f) }
    };

    /// <summary>
    /// 触发 haptic 反馈
    /// </summary>
    /// <param name="preset">反馈预设</param>
    /// <param name="controllerIndex">手柄索引（0 = 第一个手柄）</param>
    public void TriggerHaptic(HapticPreset preset, int controllerIndex = 0)
    {
        if (preset == HapticPreset.None) return;

        var gamepad = Gamepad.all.Count > controllerIndex ? Gamepad.all[controllerIndex] : null;
        if (gamepad == null) return;

        // 委托给 InputDeviceHelper.IsPS5Controller，消除重复逻辑
        if (InputDeviceHelper.IsPS5Controller(gamepad))
            TriggerPS5Haptic(gamepad, preset);
        else
            TriggerPCHaptic(gamepad, preset);
    }

    /// <summary>
    /// 触发连续 haptic 反馈（用于持续状态）
    /// </summary>
    public void TriggerContinuousHaptic(HapticPreset preset, int controllerIndex = 0)
    {
        if (preset == HapticPreset.None) return;

        var gamepad = Gamepad.all.Count > controllerIndex ? Gamepad.all[controllerIndex] : null;
        if (gamepad == null) return;

        if (InputDeviceHelper.IsPS5Controller(gamepad))
        {
            var (gain, intensity, brightness) = _ps5HapticParams[preset];

            // PS5 DualSense: 先设置强度，再激活
            var capabilities = gamepad.GetPS5HapticCapabilities();
            if (capabilities.supportsDoubleGripIntensity)
                gamepad.SetPS5HapticIntensity(intensity, brightness);
            else if (capabilities.supportsHapticIntensity)
                gamepad.SetPS5HapticIntensity(intensity);

            gamepad.SetHapticActiveState(true);
        }
        else
        {
            var pcParams = _pcHapticParams[preset];
            gamepad.SetMotorSpeeds(pcParams.LowFrequencySpeed, pcParams.HighFrequencySpeed);
        }
    }

    /// <summary>
    /// 停止连续 haptic 反馈
    /// </summary>
    public void StopHaptic(int controllerIndex = 0)
    {
        var gamepad = Gamepad.all.Count > controllerIndex ? Gamepad.all[controllerIndex] : null;
        if (gamepad == null) return;

        if (InputDeviceHelper.IsPS5Controller(gamepad))
            gamepad.SetHapticActiveState(false);
        else
            gamepad.SetMotorSpeeds(0f, 0f);
    }

    private void TriggerPCHaptic(Gamepad gamepad, HapticPreset preset)
    {
        var hapticParams = _pcHapticParams[preset];
        gamepad.SetMotorSpeeds(hapticParams.LowFrequencySpeed, hapticParams.HighFrequencySpeed);

        if (hapticParams.Duration > 0)
            StartCoroutine(StopPCHapticDelayed(gamepad, hapticParams.Duration));
    }

    private System.Collections.IEnumerator StopPCHapticDelayed(Gamepad gamepad, float delay)
    {
        yield return new UnityEngine.WaitForSeconds(delay);
        gamepad.SetMotorSpeeds(0f, 0f);
    }

    private void TriggerPS5Haptic(Gamepad gamepad, HapticPreset preset)
    {
        var (gain, intensity, brightness) = _ps5HapticParams[preset];

        var capabilities = gamepad.GetPS5HapticCapabilities();
        if (capabilities.supportsDoubleGripIntensity)
            gamepad.SetPS5HapticIntensity(intensity, brightness);
        else if (capabilities.supportsHapticIntensity)
            gamepad.SetPS5HapticIntensity(intensity);

        var hapticParams = _pcHapticParams[preset];
        if (hapticParams.Duration > 0)
            StartCoroutine(StopPS5HapticDelayed(gamepad, hapticParams.Duration));
    }

    private System.Collections.IEnumerator StopPS5HapticDelayed(Gamepad gamepad, float delay)
    {
        yield return new UnityEngine.WaitForSeconds(delay);
        gamepad.SetHapticActiveState(false);
    }

    /// <summary>
    /// 上下文相关的 haptic 反馈（根据当前游戏状态自动选择预设）
    /// 使用 HapticContext 枚举替代字符串硬编码，确保类型安全
    ///
    /// **TODO：调用者集成**
    /// 此方法需要由以下系统在相应事件触发时调用：
    /// - PlayerController：在脚步声、落地、冲刺时调用 HapticContext.Footstep/Sprint/Crouch/Land
    /// - CombatSystem：在攻击、受击、死亡时调用 HapticContext.AttackMelee/AttackRange/HitReceived/Death
    /// - NPCAI：在警戒、搜索时调用 HapticContext.AlertNPC/NPCSuspicious/NPCSearch
    /// - UISystem：在 UI 交互时调用 HapticContext.UI_Select/UI_Back
    ///
    /// 示例调用：
    ///   HapticFeedbackManager.Instance.TriggerContextualHaptic(HapticFeedbackManager.HapticContext.Footstep);
    /// </summary>
    public void TriggerContextualHaptic(HapticContext context, int controllerIndex = 0)
    {
        var preset = context switch
        {
            // UI 交互
            HapticContext.UI_Select     => HapticPreset.Light,
            HapticContext.UI_Back       => HapticPreset.Light,

            // 移动相关
            HapticContext.Footstep      => HapticPreset.Light,
            HapticContext.Sprint        => HapticPreset.Continuous,
            HapticContext.Crouch        => HapticPreset.Medium,
            HapticContext.Land          => HapticPreset.Medium,

            // 战斗相关
            HapticContext.AttackMelee   => HapticPreset.Heavy,
            HapticContext.AttackRange   => HapticPreset.Heavy,
            HapticContext.HitReceived   => HapticPreset.Heavy,
            HapticContext.Explosion     => HapticPreset.Heavy,
            HapticContext.StealthKill   => HapticPreset.Heavy,

            // NPC 相关
            HapticContext.AlertNPC      => HapticPreset.Pulse,
            HapticContext.NPCSuspicious => HapticPreset.Pulse,
            HapticContext.NPCSearch     => HapticPreset.Medium,

            // 其他
            HapticContext.Death         => HapticPreset.Heavy,
            _                           => HapticPreset.None
        };

        TriggerHaptic(preset, controllerIndex);
    }
}
```

### 3. 输入仲裁层

```csharp
// InputArbitrator.cs
/// <summary>
/// 输入仲裁器：管理输入优先级，决定哪个消费者接收输入
/// 集成 ActionLock 机制
///
/// **生命周期注意**：
/// InputArbitrator 使用静态字段单例，在 Editor 中通过 PlayModeStateChange 监听自动清理状态，
/// 解决了 Editor Play Mode 结束后静态状态残留的问题。
/// ForceReleaseAll() 可通过 InputManager.OnDestroy 调用，作为双保险。
///
/// **单例模式说明**：
/// InputArbitrator 使用静态单例而非 MonoBehaviour 单例，原因如下：
/// 1. InputArbitrator 无需挂载到 GameObject，不依赖 MonoBehaviour 生命周期
/// 2. 需要在 PlayModeStateChanged 回调中访问 static 实例进行清理
/// 3. 与 EventBus 的单例模式保持一致（均为静态单例）
///
/// 注意：与 InputManager/HapticFeedbackManager 的 MonoBehaviour 单例模式不同，
/// 这是因为 InputArbitrator 不需要协程支持且需响应 Editor 事件。
/// 若未来需要协程支持，可考虑重构为 MonoBehaviour 单例。
/// </summary>
public class InputArbitrator
{
    public static InputArbitrator Instance { get; private set; }

    /// <summary>
    /// 输入消费者标识枚举
    /// 使用枚举替代字符串常量，避免拼写错误并提供编译时检查
    /// </summary>
    public enum InputConsumer
    {
        PlayerController,
        LOSSystem,
        GrittyTakedowns,
        UISystem,
        DialogueSystem,
        PauseMenu
    }

    // [已修复] 消费者优先级常量（数字越大优先级越高）
    // 优先级定义：PlayerController(10) < LOSSystem(20) < GrittyTakedowns(30) < DialogueSystem(40) < UISystem(50) < PauseMenu(100)
    private static readonly Dictionary<InputConsumer, int> _consumerPriorities = new()
    {
        { InputConsumer.PlayerController, 10 },   // 基础移动/交互，最低优先级
        { InputConsumer.LOSSystem, 20 },           // 专注监听需要临时接管输入
        { InputConsumer.GrittyTakedowns, 30 },     // 处决/对话时需要限制部分输入
        { InputConsumer.DialogueSystem, 40 },     // 对话需要屏蔽战斗等干扰输入
        { InputConsumer.UISystem, 50 },            // UI 操作需要屏蔽游戏输入
        { InputConsumer.PauseMenu, 100 }          // 暂停菜单最高优先级
    };

    // 当前激活的消费者
    private InputConsumer _activeConsumer;

    // 锁定的输入（ActionLock 影响的输入类型）
    private HashSet<string> _lockedInputs = new();

    private InputArbitrator() { }

    static InputArbitrator()
    {
#if UNITY_EDITOR
        // 静态构造函数中注册 Play Mode 监听，确保在任何 MonoBehaviour 之前就能响应状态变化
        EditorApplication.playModeStateChanged += OnPlayModeStateChanged;
#endif
    }

    ~InputArbitrator()
    {
#if UNITY_EDITOR
        // 析构函数中取消订阅，防止内存泄漏
        EditorApplication.playModeStateChanged -= OnPlayModeStateChanged;
#endif
    }

#if UNITY_EDITOR
    private static void OnPlayModeStateChanged(PlayModeStateChange state)
    {
        if (state == PlayModeStateChange.ExitingPlayMode || state == PlayModeStateChange.EnteredEditMode)
        {
            // 切换到编辑模式或退出播放模式时，清除所有锁定状态
            if (Instance != null)
                Instance.ForceReleaseAll();
        }
    }
#endif

    /// <summary>
    /// 获取消费者优先级
    /// </summary>
    public int GetPriority(InputConsumer consumer) =>
        _consumerPriorities.TryGetValue(consumer, out var p) ? p : 0;

    /// <summary>
    /// 注册输入消费者。如果消费者已存在则**更新其优先级**（以最后注册为准）。
    /// </summary>
    /// <param name="consumer">消费者标识</param>
    /// <param name="priority">优先级（数字越大优先级越高）</param>
    public void RegisterConsumer(InputConsumer consumer, int priority)
    {
        _consumerPriorities[consumer] = priority;
    }

    /// <summary>
    /// [已修复] 设置当前激活的消费者。仅当新消费者优先级 >= 当前消费者时才能切换。
    /// 如果消费者尚未注册，自动以默认优先级（0）注册后激活。
    ///
    /// **调用时机和调用者**：
    /// | 调用者 | 调用时机 | 设置的 Consumer |
    /// |--------|----------|-----------------|
    /// | PlayerController | 正常游戏时 | PlayerController (10) |
    /// | LOSSystem | 进入专注监听模式时 | LOSSystem (20) |
    /// | GrittyTakedowns | 开始处决/交互时 | GrittyTakedowns (30) |
    /// | DialogueSystem | 开始对话时 | DialogueSystem (40) |
    /// | UISystem | 打开 UI 菜单时 | UISystem (50) |
    /// | PauseMenu | 打开暂停菜单时 | PauseMenu (100) |
    ///
    /// **优先级规则**：
    /// - 数字越大优先级越高
    /// - 仅当新消费者优先级 >= 当前消费者优先级时才能切换
    /// - 消费者开始工作时调用 SetActiveConsumer，结束后应调用 ReleaseConsumer 或切回原消费者
    /// </summary>
    public void SetActiveConsumer(InputConsumer consumer)
    {
        var newPriority = GetPriority(consumer);
        var currentPriority = _activeConsumer != default
            ? GetPriority(_activeConsumer)
            : 0;

        if (newPriority >= currentPriority || _activeConsumer == default)
        {
            _activeConsumer = consumer;
            Debug.Log($"[InputArbitrator] Active consumer: {consumer} (priority: {newPriority})");
        }
        else
        {
            Debug.LogWarning($"[InputArbitrator] Cannot switch to lower-priority consumer {consumer} (priority: {newPriority}) while {_activeConsumer} (priority: {currentPriority}) is active.");
        }
    }

    /// <summary>
    /// [已修复] 释放活跃消费者，将控制权交回给 PlayerController
    /// 在当前消费者完成工作后调用（如对话结束、UI 关闭）
    /// </summary>
    public void ReleaseConsumer()
    {
        _activeConsumer = default;
        Debug.Log("[InputArbitrator] Active consumer released, returning control to PlayerController.");
    }

    /// <summary>
    /// 检查输入是否应该被路由到指定消费者
    /// </summary>
    public bool ShouldRouteTo(InputConsumer consumer)
    {
        if (_activeConsumer == default)
            return true;

        return _activeConsumer == consumer;
    }

    /// <summary>
    /// 锁定指定输入类型（由 ActionLock 调用）
    /// </summary>
    public void LockInput(string inputType) => _lockedInputs.Add(inputType);

    /// <summary>
    /// 解锁指定输入类型
    /// </summary>
    public void UnlockInput(string inputType) => _lockedInputs.Remove(inputType);

    /// <summary>
    /// 检查输入是否被锁定
    /// </summary>
    public bool IsInputLocked(string inputType) => _lockedInputs.Contains(inputType);

    /// <summary>
    /// 释放所有锁并重置活跃消费者（由 PlayModeStateChanged 自动调用 + InputManager.OnDestroy 双保险）
    /// </summary>
    public void ForceReleaseAll()
    {
        _lockedInputs.Clear();
        _activeConsumer = default;
        Debug.Log("[InputArbitrator] All locks released and active consumer reset.");
    }
}
```

> **InputArbitrator 与 ActionLockSystem 协同说明**：
> - **ActionLockSystem**（shared-types.md §4.2）：管理玩家控制权的**锁定/解锁**
> - **InputArbitrator**：管理输入**路由**，决定哪个消费者接收输入
> 两者协同工作：GrittyTakedowns 等系统获取 ActionLock 后，再调用 InputArbitrator.SetActiveConsumer() 设置输入路由。

### 4. 输入重映射

```csharp
// InputRemapManager.cs
/// <summary>
/// 输入重映射管理器
/// 支持玩家自定义键位
/// 使用扁平化结构确保 JsonUtility 兼容性（JsonUtility 不支持嵌套 Dictionary）
/// </summary>
public class InputRemapManager : MonoBehaviour
{
    public static InputRemapManager Instance { get; private set; }

    // 键位映射配置（运行时使用）
    private Dictionary<string, RemapBinding> _pcRemaps = new();
    private Dictionary<string, RemapBinding> _gamepadRemaps = new();

    private const string REMAP_CONFIG_PATH = "remap_config.json";

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        LoadRemapConfig();
    }

    /// <summary>
    /// 配置文件（可序列化）
    /// 使用扁平化结构以确保 JsonUtility 兼容性：
    /// - 使用 FlatRemapEntry 替代 List&lt;List&lt;string&gt;&gt;
    /// - FlatRemapEntry 将嵌套列表展平为 paths 和 displayNames 的独立条目
    /// </summary>
    [Serializable]
    private class RemapConfigData
    {
        public List<string> pcActionNames = new();
        public List<FlatRemapEntry> pcEntries = new();  // 扁平化结构替代嵌套 List
        public List<string> gamepadActionNames = new();
        public List<FlatRemapEntry> gamepadEntries = new();
    }

    /// <summary>
    /// 扁平化的重映射条目（JsonUtility 兼容）
    /// </summary>
    [Serializable]
    private class FlatRemapEntry
    {
        public List<string> paths = new();
        public List<string> displayNames = new();
    }

    private void LoadRemapConfig()
    {
        var path = Path.Combine(Application.persistentDataPath, REMAP_CONFIG_PATH);
        if (File.Exists(path))
        {
            try
            {
                var json = File.ReadAllText(path);
                var config = JsonUtility.FromJson<RemapConfigData>(json);
                if (config != null)
                {
                    _pcRemaps = RemapConfigDataToDictionary(config.pcActionNames, config.pcEntries);
                    _gamepadRemaps = RemapConfigDataToDictionary(config.gamepadActionNames, config.gamepadEntries);
                }
            }
            catch (Exception ex)
            {
                Debug.LogWarning($"[InputRemapManager] Failed to load remap config: {ex.Message}");
            }
        }
    }

    public void SaveRemapConfig()
    {
        var (pcActionNames, pcEntries) = DictionaryToRemapConfigData(_pcRemaps);
        var (gamepadActionNames, gamepadEntries) = DictionaryToRemapConfigData(_gamepadRemaps);

        var config = new RemapConfigData
        {
            pcActionNames = pcActionNames,
            pcEntries = pcEntries,
            gamepadActionNames = gamepadActionNames,
            gamepadEntries = gamepadEntries
        };

        var path = Path.Combine(Application.persistentDataPath, REMAP_CONFIG_PATH);

        try
        {
            // 先写入临时文件，再 Rename（原子操作），避免写入失败导致配置损坏
            var tempPath = path + ".tmp";
            File.WriteAllText(tempPath, JsonUtility.ToJson(config));

            // 验证写入内容
            var written = File.ReadAllText(tempPath);
            var expected = JsonUtility.ToJson(config);
            if (written != expected)
            {
                Debug.LogError($"[InputRemapManager] Config write verification failed. Expected {expected.Length} bytes, got {written.Length} bytes.");
                File.Delete(tempPath);
                return;
            }

            // 原子替换
            File.Delete(path);  // 如果已存在，删除旧文件（跨平台兼容）
            File.Move(tempPath, path);

#if DEVELOPMENT_BUILD
            Debug.Log($"[InputRemapManager] Remap config saved to {path}");
#endif
        }
        catch (Exception ex)
        {
            Debug.LogWarning($"[InputRemapManager] Failed to save remap config: {ex.Message}");
        }
    }

    private (List<string>, List<FlatRemapEntry>) DictionaryToRemapConfigData(Dictionary<string, RemapBinding> dict)
    {
        var actionNames = new List<string>();
        var entries = new List<FlatRemapEntry>();

        foreach (var kvp in dict)
        {
            actionNames.Add(kvp.Key);
            var entry = new FlatRemapEntry
            {
                paths = new List<string>(),
                displayNames = new List<string>()
            };
            foreach (var binding in kvp.Value.bindings)
            {
                entry.paths.Add(binding.path);
                entry.displayNames.Add(binding.displayName);
            }
            entries.Add(entry);
        }

        return (actionNames, entries);
    }

    private Dictionary<string, RemapBinding> RemapConfigDataToDictionary(List<string> actionNames, List<FlatRemapEntry> entries)
    {
        var dict = new Dictionary<string, RemapBinding>();
        if (actionNames == null || entries == null) return dict;

        for (int i = 0; i < actionNames.Count && i < entries.Count; i++)
        {
            var bindings = new List<RemapBindingItem>();
            var entry = entries[i];
            for (int j = 0; j < entry.paths.Count && j < entry.displayNames.Count; j++)
            {
                bindings.Add(new RemapBindingItem
                {
                    path = entry.paths[j],
                    displayName = entry.displayNames[j]
                });
            }
            dict[actionNames[i]] = new RemapBinding { actionName = actionNames[i], bindings = bindings };
        }

        return dict;
    }

    public List<RemapBindingItem> GetBindings(string actionName, InputDeviceType device)
    {
        var remaps = device == InputDeviceType.Keyboard ? _pcRemaps : _gamepadRemaps;
        if (remaps.TryGetValue(actionName, out var binding))
            return binding.bindings;
        return new List<RemapBindingItem>();
    }

    /// <summary>
    /// 重映射指定 Action
    /// 参数改为 List&lt;RemapBindingItem&gt;（单个 action 的绑定列表），
    /// 而非原来的 List&lt;RemapBinding&gt;（多个 action 的绑定）以匹配方法语义
    /// </summary>
    public void RemapAction(string actionName, InputDeviceType device, List<RemapBindingItem> newBindings)
    {
        // 验证所有路径格式合法
        foreach (var item in newBindings)
        {
            if (!IsValidInputPath(item.path))
            {
                Debug.LogWarning($"[InputRemapManager] Invalid input path: {item.path} for action {actionName}");
                return;
            }
        }

        var remaps = device == InputDeviceType.Keyboard ? _pcRemaps : _gamepadRemaps;
        remaps[actionName] = new RemapBinding { actionName = actionName, bindings = newBindings };

        EventBus.Instance.Publish(new InputRemappedEvent
        {
            ActionName = actionName,
            Device = device
        });

        SaveRemapConfig();
    }

    /// <summary>
    /// 验证 InputSystem 路径格式（必须以 &lt;Device&gt;/ 开头）
    /// </summary>
    private bool IsValidInputPath(string path)
    {
        if (string.IsNullOrEmpty(path))
            return false;

        if (!path.StartsWith("<") || !path.Contains(">/"))
            return false;

        var validDevices = new HashSet<string>
        {
            "Keyboard", "Mouse", "Gamepad", "Touch",
            "PS5Gamepad", "PS4Gamepad", "XboxGamepad",
            "Joystick", "Sensor"
        };

        return validDevices.Any(d => path.StartsWith($"<{d}>"));
    }

    /// <summary>
    /// 重置该设备所有 Action 绑定为默认
    /// </summary>
    public void ResetToDefaults(InputDeviceType device)
    {
        ResetToDefaults(device, actionName: null);
    }

    /// <summary>
    /// 重置指定 Action 或所有 Action 的绑定
    /// </summary>
    /// <param name="device">设备类型</param>
    /// <param name="actionName">要重置的 Action 名称，传 null 表示重置该设备所有绑定</param>
    public void ResetToDefaults(InputDeviceType device, string actionName)
    {
        if (device == InputDeviceType.Keyboard)
        {
            if (actionName == null) _pcRemaps.Clear();
            else _pcRemaps.Remove(actionName);
        }
        else
        {
            if (actionName == null) _gamepadRemaps.Clear();
            else _gamepadRemaps.Remove(actionName);
        }

        EventBus.Instance.Publish(new InputRemappedEvent
        {
            ActionName = actionName,  // null 表示全部重置
            Device = device
        });

        SaveRemapConfig();
    }
}

/// <summary>
/// 运行时使用的绑定结构（非序列化）
/// </summary>
public struct RemapBinding
{
    public string actionName;
    public List<RemapBindingItem> bindings;
}

/// <summary>
/// 绑定项（运行时使用）
/// </summary>
public struct RemapBindingItem
{
    public string path;
    public string displayName;
}
```

### 5. InputConfigSO 配置定义

```csharp
// InputConfigSO.cs
[CreateAssetMenu(fileName = "InputConfig", menuName = "Game/Input/InputConfig")]
public class InputConfigSO : ScriptableObject
{
    [Header("Device Detection")]
    [Tooltip("设备检测间隔（秒）")]
    public float deviceCheckInterval = 0.25f;

    [Header("Default Bindings - Keyboard")]
    public string moveUpKey = "<Keyboard>/w";
    public string moveDownKey = "<Keyboard>/s";
    public string moveLeftKey = "<Keyboard>/a";
    public string moveRightKey = "<Keyboard>/d";
    public string sprintKey = "<Keyboard>/leftShift";
    public string crouchKey = "<Keyboard>/leftCtrl";
    public string actionKey = "<Keyboard>/e";
    public string focusKey = "<Keyboard>/v";
    public string mapKey = "<Keyboard>/tab";
    public string pauseKey = "<Keyboard>/escape";

    [Header("Default Bindings - Gamepad")]
    public string gamepadMove = "<Gamepad>/leftStick";
    public string gamepadSprint = "<Gamepad>/leftTrigger";
    public string gamepadCrouch = "<Gamepad>/buttonB";
    public string gamepadAction = "<Gamepad>/buttonA";
    public string gamepadFocus = "<Gamepad>/buttonY";
    public string gamepadMap = "<Gamepad>/touchpadButton";
    public string gamepadPause = "<Gamepad>/start";

    [Header("Haptic Feedback")]
    [Tooltip("是否启用 Haptic 反馈")]
    public bool hapticEnabled = true;

    [Tooltip("Haptic 反馈强度缩放（0.0-1.0）")]
    [Range(0f, 1f)]
    public float hapticIntensityScale = 1.0f;

    [Header("Input Arbitration")]
    [Tooltip("输入仲裁是否启用")]
    public bool arbitrationEnabled = true;

    [Header("Action Lock Settings")]
    [Tooltip("动作锁定超时时间（秒）")]
    public float actionLockTimeout = 5.0f;
}
```

> **配置文件位置**：`Resources/Config/InputConfig.asset`
>
> **说明**：`InputConfigSO` 通过 `Resources.Load` 同步加载，作为 InputManager 的默认配置。
> 若需运行时修改，可通过 `InputManager.Instance.GetComponent<InputConfigSO>()` 引用。

### 6. Unity 项目结构

以下类型在本 ADR 中定义，**必须**同步添加到 `shared-types.md`：

```csharp
// InputDeviceType.cs
// 定义位置：Assets/Game/Foundation/Shared/Types/InputDeviceType.cs
public enum InputDeviceType
{
    Keyboard,
    PS5Gamepad,
    PS4Gamepad,
    XboxGamepad,
    GenericGamepad
}
```

```csharp
// InputDeviceChangedEvent.cs
// 定义位置：Assets/Game/Foundation/Shared/Events/InputDeviceChangedEvent.cs
public struct InputDeviceChangedEvent
{
    public InputDeviceType Device;
}

// InputRemappedEvent.cs
// 定义位置：Assets/Game/Foundation/Shared/Events/InputRemappedEvent.cs
public struct InputRemappedEvent
{
    /// <summary>
    /// 被重映射的 Action 名称
    /// null = 全部重置（调用 ResetToDefaults(device) 时）
    /// </summary>
    public string ActionName;
    public InputDeviceType Device;
}
```

> **重要**：在实现前，必须先将上述类型添加到 `shared-types.md`，以确保所有系统引用一致。

```
Assets/Game/Infrastructure/Input/
├── InputManager.cs                   # 单例，输入核心
├── GameInputAction.cs               # InputAction 包装
├── InputArbitrator.cs               # 输入仲裁
├── InputRemapManager.cs             # 重映射管理
├── InputDeviceHelper.cs             # 设备类型识别辅助（被 InputManager 和 HapticFeedbackManager 共用）
├── HapticFeedbackManager.cs         # Haptic 反馈管理
├── Events/
│   ├── InputDeviceChangedEvent.cs   # 设备切换事件
│   └── InputRemappedEvent.cs        # 重映射事件
├── Config/
│   └── InputConfigSO.cs            # 输入配置 ScriptableObject
└── Prefabs/
    └── InputManager.prefab         # Manager 预制件
```

---

## Alternatives Considered

### Alternative 1: 直接使用 Unity Input System（无包装）

- **描述**：各系统直接使用 `InputAction` 和 `InputActionReference`
- **Pros**：Unity 原生，无额外抽象
- **Cons**：
  - 平台差异处理重复
  - 输入重映射难以统一
  - 优先级仲裁缺失
- **拒绝理由**：需要大量重复代码处理平台差异

### Alternative 2: 统一 InputManager（无分层）

- **描述**：只有一个 InputManager，所有输入查询通过它
- **Pros**：简单
- **Cons**：无优先级仲裁，ActionLock 集成复杂
- **拒绝理由**：GrittyTakedowns 等系统需要临时接管输入，优先级仲裁必不可少

---

## Consequences

### Positive

- **统一入口**：所有输入通过 InputManager 查询
- **平台屏蔽**：PC/PS5 差异被抽象层屏蔽
- **重映射支持**：玩家可自由调整键位
- **优先级明确**：多系统输入冲突通过仲裁解决
- **ActionLock 集成**：动作锁定自动影响输入

### Negative

- **学习曲线**：新增的 InputArbitrator 概念需要理解
- **调试复杂性**：输入流经多层，增加调试难度

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **输入延迟** | 缓存导致输入响应迟 | 确保 InputManager.Update 在所有消费者之前 |
| **优先级混乱** | 多个系统同时持有相同优先级 | 定义清晰的优先级表 |
| **重映射丢失** | 配置写入失败 | 本地备份 + 云同步 |
| **PS5 认证风险** | Sony 可能对 touchpad 按钮映射有特殊要求导致认证失败 | 已实现备选方案：`<Gamepad>/view` 按钮作为地图快捷键替代品；提交前在 Player Settings 中验证按钮响应 |

---

## Performance Implications

| 指标 | 预期 | 说明 |
|------|------|------|
| **CPU** | < 0.1ms/帧 | 仅状态缓存更新 |
| **Memory** | < 1MB | 少量配置数据 |
| **Input Latency** | < 1帧 | 直接映射 InputSystem |

---

## Migration Plan

### Phase 1: 基础框架
- [ ] 创建 InputManager 单例
- [ ] 实现 GameInputAction 包装（含 2D Composite 支持）
- [ ] 配置所有 InputAction

### Phase 2: 仲裁层
- [ ] 实现 InputArbitrator
- [ ] 集成 ActionLock
- [ ] 与 PlayerController 集成

### Phase 3: 重映射
- [ ] 实现 InputRemapManager
- [ ] 实现配置持久化
- [ ] UI 集成

### Phase 4: 消费者迁移
- [ ] 迁移 LOSSystem 使用 InputManager
- [ ] 迁移 GrittyTakedowns 使用 InputManager
- [ ] 迁移 UISystem 使用 InputManager

---

## Validation Criteria

1. **输入响应**：WASD/左摇杆移动方向正确（Vector2 四方向）
2. **设备切换**：检测到不同设备自动切换
3. **重映射生效**：修改键位后正确响应
4. **ActionLock**：动作锁定时相关输入被屏蔽
5. **优先级仲裁**：高优先级消费者接管时低优先级无法接收输入
6. **Editor 重置**：Play Mode 结束后 InputArbitrator 状态清零

---

## Related Decisions

- [ADR-0009: Player Controller 架构](./adr-0009-player-controller-architecture.md) — PlayerController 的输入消费者
- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — Infrastructure Layer 的核心系统
- [shared-types.md](./shared-types.md) — ActionLockSystem 定义位置
- [ADR-0011: Gritty Takedowns 架构](./adr-0011-gritty-takedowns-architecture.md) — 需要接管输入的消费者
