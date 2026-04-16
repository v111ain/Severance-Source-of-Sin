# ADR-0001: 事件驱动架构（Event-Driven Architecture）

## Status
**Accepted**

## Date
2026-04-09

## Last Updated
2026-04-14

### Revision History

| 日期 | 修订内容 | 负责人 |
|------|----------|--------|
| 2026-04-14 | `NoiseEvent` → `NoiseMadeEvent`：统一事件命名规范，与 shared-types.md §22.4 保持一致 | Systems Designer |

## Context

### Problem Statement

跨系统通信存在两种模式：轮询（Polling）和事件订阅（Event-Driven）。在《断绝：罪恶之源》的开发过程中，发现多个系统之间存在轮询使用，导致：

1. **紧耦合**：轮询系统需要知道被轮询者的内部结构
2. **性能浪费**：无意义的重复查询，消耗 CPU
3. **状态同步困难**：轮询得到的是"快照"，可能错过中间状态变化
4. **接口冲突**：如 Gritty Takedowns 系统声称"AlertStateChanged 事件尚未定义"，实际 NPC AI 已定义

### Constraints

- 必须兼容 Unity 6.3 LTS 的事件系统
- 事件总线必须支持跨系统通信（NPC AI ↔ Gritty Takedowns, LOS ↔ Clue System 等）
- 事件订阅必须可 Debug，方便排查问题
- 不能引入过重的第三方消息队列框架

### Requirements

- **必须**：消除所有跨系统轮询
- **必须**：统一事件命名规范（`Subject` + `Did` + `Context` + `Event`）
- **必须**：每个事件有明确的"发送方（Owner）"和"订阅方（Subscribers）"
- **必须**：支持事件参数传递
- **应该**：支持事件 Debug/Log 可视化

---

## Decision

### 架构决策

采用**事件驱动架构（Event-Driven Architecture）**，通过统一事件总线（Event Bus）实现跨系统通信：

```
┌─────────────────────────────────────────────────────────────┐
│                     事件总线 (Event Bus)                      │
├─────────────────────────────────────────────────────────────┤
│  Publish/Subscribe Pattern                                    │
│                                                              │
│  [Sender System] ──Publish──▶ [Event Bus] ──Dispatch──▶ [Subscriber] │
│                                                              │
│  事件流向（从左到右为 Publish，从右到左为 Dispatch）：         │
│  Health System     ──NPCStateChangedEvent──▶ Gritty T.       │
│  NPC AI System     ──AlertStateChangedEvent──▶ Gritty T.     │
│  LOS System        ──PlayerSpottedEvent──▶ NPC AI           │
│  Player Controller ──NoiseMadeEvent──▶ NPC AI                │
└─────────────────────────────────────────────────────────────┘
```

### 事件命名规范

所有事件必须遵循：`[Subject] + [Did] + [Context] + [Event]`

> **QueryBus 同步查询模式已取消 (2026-04-15)**：根据 ADR-0018 §4，所有系统间状态查询已改为**事件订阅模式**。
> 原 `QueryNPCIdentity`、`QueryAlertState` 等 Query 类型已废弃，不应再使用。

| 类型 | 规则 | 示例 |
|------|------|------|
| **事件 (Event)** | `Subject` + `Did` + `Context` + `Event` | `PlayerDamagedEvent`, `NPCStateChangedEvent` |
| **请求 (Request)** | `Subject` + `Request` | `DamageRequest`, `DialogueStartRequest` |

> **已废弃的类型**：
> - ~~查询 (Query)~~ → 改为订阅 `XXXChangedEvent`，在回调中缓存状态
> - ~~响应 (Response)~~ → Query 取消后不再需要独立 Response 类型

### 强制要求

1. **所有外部事件必须包含 `Event` 后缀**
2. **使用完整单词**，不使用缩写
3. **同一事件只有一个标准名称**，其他均为别名（如 `NPCDeath` → `NPCStateChangedEvent`）
4. **事件名使用 PascalCase**

### 事件所有权

每个事件有唯一的**拥有者系统**，拥有者负责：
- 定义事件的完整数据结构
- 决定事件何时触发
- 维护事件的文档说明

### Event Bus 实现方案

推荐使用 **ScriptableObject-based Event Bus**，作为全局单例：

### 订阅Token机制

为解决事件订阅后无法取消导致的内存泄漏问题，EventBus 采用 **SubscriptionToken** 机制：

```csharp
/// <summary>
/// 订阅Token，用于唯一标识一次订阅并支持取消订阅
/// </summary>
public readonly struct SubscriptionToken : IEquatable<SubscriptionToken>
{
    public Guid Id { get; }
    public Type EventType { get; }

    public SubscriptionToken(Type eventType)
    {
        Id = Guid.NewGuid();
        EventType = eventType;
    }

    public bool Equals(SubscriptionToken other) => Id == other.Id;
    public override bool Equals(object obj) => obj is SubscriptionToken other && Equals(other);
    public override int GetHashCode() => Id.GetHashCode();
    public static bool operator ==(SubscriptionToken left, SubscriptionToken right) => left.Equals(right);
    public static bool operator !=(SubscriptionToken left, SubscriptionToken right) => !left.Equals(right);
}
```

**设计要点**：
1. 每个 `SubscriptionToken` 包含唯一 `Guid` 和 `EventType`
2. `Subscribe<T>` 返回 `SubscriptionToken`，调用方需保存
3. `Unsubscribe(token)` 根据 Token 取消订阅
4. EventBus 内部维护 `Dictionary<SubscriptionToken, Delegate>` 用于 O(1) 查找

```csharp
// Assets/Game/Infrastructure/EventBus/EventBus.cs
[CreateAssetMenu(menuName = "Game/EventBus")]
public class EventBus : ScriptableObject
{
    private Dictionary<Type, List<Delegate>> _subscribers = new();
    private Dictionary<SubscriptionToken, Delegate> _tokenToHandler = new();
    private static volatile EventBus _instance;
    private static readonly object _lock = new();

    public static EventBus Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                    {
                        _instance = Resources.Load<EventBus>("EventBus");
                        if (_instance == null)
                            throw new InvalidOperationException(
                                "EventBus asset not found in Resources. " +
                                "Ensure 'Assets/Game/Infrastructure/EventBus/EventBus.asset' exists."
                            );
                    }
                }
            }
            return _instance;
        }
    }

    /// <summary>
    /// 订阅事件，返回SubscriptionToken用于取消订阅
    /// </summary>
    public SubscriptionToken Subscribe<T>(Action<T> callback)
    {
        var type = typeof(T);
        if (!_subscribers.ContainsKey(type))
            _subscribers[type] = new List<Delegate>();
        _subscribers[type].Add(callback);

        var token = new SubscriptionToken(type);
        _tokenToHandler[token] = callback;
        return token;
    }

    /// <summary>
    /// 通过SubscriptionToken取消订阅（推荐方式）
    /// </summary>
    public void Unsubscribe(SubscriptionToken token)
    {
        if (_tokenToHandler.TryGetValue(token, out var handler))
        {
            var type = token.EventType;
            if (_subscribers.TryGetValue(type, out var list))
            {
                list.Remove(handler);
                if (list.Count == 0)
                    _subscribers.Remove(type);
            }
            _tokenToHandler.Remove(token);
        }
    }

    /// <summary>
    /// 传统取消订阅方式（通过callback引用）
    /// </summary>
    public void Unsubscribe<T>(Action<T> callback)
    {
        var type = typeof(T);
        if (_subscribers.TryGetValue(type, out var list))
        {
            list.Remove(callback);
            if (list.Count == 0)
                _subscribers.Remove(type);
        }
        // 同时从token映射中移除
        var toRemove = _tokenToHandler
            .Where(kvp => kvp.Value == callback)
            .Select(kvp => kvp.Key)
            .ToList();
        foreach (var token in toRemove)
            _tokenToHandler.Remove(token);
    }

    public void Publish<T>(T eventData)
    {
        var type = typeof(T);
        if (_subscribers.TryGetValue(type, out var callbacks))
            foreach (var callback in callbacks.ToArray())
                ((Action<T>)callback)(eventData);
    }
}
```

**使用示例**：
```csharp
public class MySystem : MonoBehaviour
{
    private SubscriptionToken _token;

    private void OnEnable()
    {
        // 订阅并保存Token
        _token = EventBus.Instance.Subscribe<PlayerDamagedEvent>(OnPlayerDamaged);
    }

    private void OnDisable()
    {
        // 通过Token取消订阅，防止内存泄漏
        EventBus.Instance.Unsubscribe(_token);
    }

    private void OnPlayerDamaged(PlayerDamagedEvent evt) { ... }
}
```

> **重要**：所有订阅必须在 `OnEnable` 中订阅，`OnDisable` 中取消订阅。MonoBehaviour 销毁时未取消的订阅会导致内存泄漏。

### 对象池设计要点

为减少 GC 压力，频繁触发的事件（如 `NoiseMadeEvent`）应使用对象池：

1. **池化策略**：在 `EventBus` 外部包装 `PooledEventBus` 层
2. **回收时机**：事件被所有订阅方处理完毕后自动回收
3. **特殊处理**：死亡/伤害等低频事件可不做池化，保持代码简洁

示例所有权表：

| 事件名称 | 拥有者 | 订阅方 |
|---------|-------|--------|
| `PlayerDamagedEvent` | Health System | Gritty Takedowns, Sanity/Rage, Immersive Audio |
| `NPCStateChangedEvent` | Health System | Gritty Takedowns, Clue System, Sanity/Rage, NPC AI |

---

## Alternatives Considered

### Alternative 1: 保留轮询机制

- **描述**：各系统保留轮询接口，直接查询目标系统状态
- **优点**：
  - 实现简单，不需要额外框架
  - 调试直观，可以随时打断点查看状态
- **缺点**：
  - 紧耦合，轮询方需要了解被轮询方内部结构
  - 性能浪费，重复无用查询
  - 错过中间状态变化
  - 跨系统边界轮询难以维护
- **拒绝理由**：
  - 与"消除跨系统内部耦合"的核心架构原则冲突
  - 已被 `interface-alignment-meeting.md` 会议记录证明存在问题（Gritty Takedowns 误以为事件未定义）

### Alternative 2: 使用 Unity.SendMessage 或直接方法调用

- **描述**：通过 `UnityEngine.SendMessage` 或直接引用调用其他系统方法
- **优点**：
  - Unity 内置，无需额外代码
  - 同步调用，逻辑简单
- **缺点**：
  - 强耦合，发送方需要持有接收方引用
  - 无法解耦，系统边界模糊
  - 难以扩展，添加新订阅方需要修改发送方代码
- **拒绝理由**：
  - 不支持一对多通信（Gritty Takedowns 需要同时通知 NPC AI 和 Immersive Audio）

### Alternative 3: 使用完整中介者模式（Mediator Pattern）

- **描述**：引入完整的中介者框架，所有通信必须通过中介者
- **优点**：
  - 完全解耦
  - 集中管理通信逻辑
- **缺点**：
  - 过度设计，中介者本身成为超级上帝对象
  - 引入不必要的复杂性
  - 增加学习成本
- **拒绝理由**：
  - 我们的系统规模不需要完整中介者
  - 事件总线已经足够满足需求，且更轻量

---

## Consequences

### Positive

- **解耦合**：发送方和订阅方无需相互引用，通过事件总线解耦
- **性能优化**：消除无意义的重复轮询
- **状态完整性**：事件驱动确保不错过任何状态变化
- **可扩展性**：新增订阅方无需修改发送方代码
- **可 Debug**：事件总线可集中记录所有事件，便于排查问题

### Negative

- **异步性**：事件是异步的，某些情况下需要额外处理时序问题
- **调试复杂性**：跨系统事件流需要在 Debug 工具中可视化
- **内存开销**：事件对象创建有一定 GC 压力（可通过对象池优化）

### Risks

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **事件风暴** | 系统间事件过多导致难以追踪 | 维护 `event-bus-icd.md`，明确每个事件的所有权和用途 |
| **事件遗漏订阅** | 新系统忘记订阅必要事件 | 在系统 GDD 的 Dependencies 章节明确列出事件依赖 |
| **事件循环触发** | A→B→C→A 形成死循环 | 在事件命名规范中约定禁止循环触发，Code Review 检查 |

---

## Performance Implications

| 指标 | 影响 | 说明 |
|------|------|------|
| **CPU** | ✅ 降低 | 消除轮询，减少无用 CPU 消耗 |
| **Memory** | ⚠️ 轻微增加 | 事件对象创建有 GC 压力，可通过对象池缓解 |
| **Load Time** | 无影响 | 事件总线在运行时初始化 |
| **Network** | 无影响 | 仅限单机游戏 |

---

## Migration Plan

### Phase 1: 事件总线基础设施
- [ ] 创建 `Assets/Game/Infrastructure/EventBus/EventBus.cs`
- [ ] 创建 `EventBus` ScriptableObject 资源文件
- [ ] 定义事件命名规范（`event-bus-icd.md`）
- [ ] 消除 `NPCDeath`/`NPCKilled` 等命名冲突
- [ ] 补充缺失接口定义（`QueryNPCIdentity`、`KeywordCapturedEvent`）

### Phase 2: 系统改造
- [ ] Gritty Takedowns：消除 `AlertStateChanged` 轮询，改为订阅 `AlertStateChangedEvent`
- [ ] Health System：确保 `NPCStateChangedEvent` 由 NPC AI 广播
- [ ] 其他系统：检查并消除所有跨系统轮询

### Phase 3: Debug 工具
- [ ] 实现事件总线可视化 Debug 面板
- [ ] 实现事件历史记录（可选，生产环境关闭）

---

## Validation Criteria

1. **无轮询验证**：`grep -r "Query.*State\|Get.*State\|Check.*State" Assets/Game/` 不应在跨系统代码中出现轮询调用
2. **事件所有权验证**：每个事件在 `event-bus-icd.md` 中有唯一 Owner
3. **编译验证**：所有系统 GDD 中引用的事件已在 `event-bus-icd.md` 中定义
4. **集成测试**：Gritty Takedowns 能正确接收 `AlertStateChangedEvent`
5. **Event Bus 单例验证**：Resources 目录下存在且仅存在一个 EventBus 实例

---

## Related Decisions

- [ADR-0003: 系统分层架构定义](./adr-0003-system-layers.md) — 定义了 Foundation/Core/Feature 层，是事件总线存在的基础
- [ADR-0004: NPC AI 行为架构](./adr-0004-npc-ai-behavior-architecture.md) — NPC AI 通过事件总线订阅 AlertStateChangedEvent
- [ADR-0005: 存档/持久化架构](./adr-0005-save-persistence-architecture.md) — 存档系统通过事件总线发布 SaveCompletedEvent 等
- [ADR-0006: 网络同步架构](./adr-0006-network-synchronization-architecture.md) — 网络系统通过事件总线发布 PlayerJoinedEvent 等
- [事件总线接口控制文档](./adr-0018-event-bus-icd.md) — 事件定义的权威文档
