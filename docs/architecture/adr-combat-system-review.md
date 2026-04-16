# ADR 战斗系统架构评审报告

**评审日期**：2026-04-14
**评审范围**：ADR-0008（生命值/击杀）、ADR-0010（武器系统）、ADR-0011（殊死击倒）
**辅助参考**：ADR-0018（事件总线ICD）、ADR-0023（屏幕特效）、ADR-0024（动画系统）

---

## ADR-0008 评审结果

### 发现问题

#### 问题 1：爆炸伤害衰减公式在 ADR-0008 和 ADR-0010 中定义不一致

| 文档 | 公式 | 说明 |
|------|------|------|
| ADR-0008 §4 | `finalDamage = base_damage * (1 - distance / radius) * 1.5` | 硬编码 stagger_multiplier = 1.5 |
| ADR-0010 §5.1 | `hitChance = Lerp(1.0, minChance, t)` | 投掷物命中概率计算 |

**冲突说明**：ADR-0008 中爆炸伤害的 Blunt 分支使用了硬编码的 `stagger_multiplier = 1.5f`，但这个值没有在 WeaponTuningSO 中配置化。ADR-0010 的投掷物系统有独立的调参机制，但两者的伤害数值范围不保证一致。

**影响**：爆炸物平衡性调参会与投掷物系统割裂

---

#### 问题 2：stagger_multiplier 未配置化

ADR-0008 ExplosionHandler 第 471 行：
```csharp
float stagger_multiplier = 1.5f;
```

此值硬编码，无法通过 TuningSO 调整。如果需要平衡团队反馈认为爆炸硬直效果太强或太弱，必须修改代码。

---

#### 问题 3：ARMOR_IGNORE_PENETRATION = int.MaxValue 可能导致问题

ADR-0008 §4：
```csharp
public const int ARMOR_IGNORE_PENETRATION = int.MaxValue;
```

穿透值使用 `int.MaxValue` 来绕过护甲判定。虽然功能正确，但：
1. `int.MaxValue` 是一个魔法值，应该定义为常量并说明其语义
2. 如果未来穿透值类型改变（如改为 byte 0-255），此常量会溢出

---

### 建议

1. **配置化 stagger_multiplier**：将 `1.5f` 提取到 `WeaponTuningSO.explosionStaggerMultiplier`
2. **语义化穿透常量**：将 `ARMOR_IGNORE_PENETRATION = -1` 或使用枚举 `PenetrationLevel.IGNORE_ARMOR`，并添加注释说明

---

## ADR-0010/0011 评审结果

### 发现问题

#### 问题 4：ADR-0010 中 WeaponAwareness 结构与 ADR-0018/ shared-types.md 命名不一致

**ADR-0010 §6** 定义：
```csharp
public struct WeaponAwareness
```

**ADR-0018 §3.2** 记录：
> `WeaponAwareness` → `WeaponAwarenessEvent`（已废弃别名）

**shared-types.md §3.6**：
> 已重命名为 `WeaponAwarenessEvent`，`WeaponAwareness` 作为别名保留用于过渡期兼容

**问题**：ADR-0010 代码中仍在使用旧名称 `WeaponAwareness`，虽然通过别名兼容可以正常工作，但文档应该更新以反映最新状态。

---

#### 问题 5：base_damage 语义在 ADR-0010 和 shared-types.md 中不一致

**ADR-0010 DamageComponent**：
```csharp
public float base_damage;  // 基础伤害倍率
```

**shared-types.md DamageRequest**：
```csharp
public float damage_amount;  // 实际伤害值（环境物件=1.0，热武器来自武器配置）
```

**问题**：
- ADR-0010 称 `base_damage` 为"倍率"，暗示需要与其他值相乘
- shared-types.md 称 `damage_amount` 为"实际伤害值"
- 但 ADR-0008 处理 DamageRequest 时直接使用 `damage_amount` 作为最终伤害值

**潜在冲突**：如果 `base_damage` 是倍率，那 HealthSystem 接收到的 `damage_amount` 应该已经被乘过了。但如果设计者误解了这个语义，可能导致伤害计算错误。

**建议**：统一命名和语义。建议将 `DamageComponent.base_damage` 改名为 `baseDamageValue`（基础伤害数值），与 `damage_amount` 的语义对齐。

---

#### 问题 6：投掷物命中概率的 minHitChance 在 ADR-0010 和 ADR-0011 中语义需要澄清

**ADR-0010 §5**：
```csharp
float minChance = _tuning?.minHitChance ?? 0.3f;
return Mathf.Clamp(hitChance, minChance, 1.0f);
```

**ADR-0011** 中 `hasEnvironmentWeapon` 检查只判断是否有武器，不检查命中概率。

**问题**：投掷物在玩家持有时一定能命中目标吗？还是说命中概率影响的是"是否命中"这个结果？如果是后者，ADR-0011 的 `hasEnvironmentWeapon` 检查可能不完整。

**建议**：明确投掷物命中概率与交互可用性的关系。如果 `hasEnvironmentWeapon` 意味着"持有并可以投掷"，则应该包含命中概率检查或说明命中失败的后果。

---

## 跨系统冲突清单

### 冲突 1：动画事件流程与 GrittyTakedowns 的职责边界不清

**问题描述**：

ADR-0024 AnimationEventBridge 直接发布 `InteractionEvent`：
```csharp
// ADR-0024 §2.5 AnimationEventBridge.cs
if (eventName == "OnStealthKillHit")
{
    EventBus.Instance.Publish(new InteractionEvent { ... });
}
```

这存在架构问题：
1. **职责越界**：AnimationEventBridge 属于 Presentation Layer，它不应该知道"StealthKill 成功"这个 Feature Layer 的语义
2. **事件时序错误**：`InteractionEvent` 是交互的**最终结果**事件，应该由 GrittyTakedowns 在动画完成并处理完所有逻辑后发布
3. **DamageRequest 未发送**：`OnStealthKillHit` 事件应该触发 GrittyTakedowns 的 StealthKillHandler，由该 Handler 发送 `DamageRequest`

**正确的流程应该是**：
```
动画播放到"命中帧"
    → AnimationEventBridge.OnAnimationEvent("OnStealthKillHit")
    → GrittyTakedowns.StealthKillHandler.OnStealthKillHitConfirmed()
        → 发送 DamageRequest 到 HealthSystem
        → 动画播放完成
        → 发布 InteractionEvent
```

**影响范围**：ADR-0011（StealthKillHandler）、ADR-0024（AnimationEventBridge）、ADR-0008（Health System 接收 DamageRequest）

---

### 冲突 2：WorldState 与 HealthState 的同步协议未完全落实

**问题描述**：

ADR-0011 定义了两套独立的状态机：

| 系统 | 状态 | 说明 |
|------|------|------|
| HealthSystem (ADR-0008) | HEALTHY / STAGGERED / DOWNED / DEAD | 生命状态 |
| NPCAI (ADR-0004) | FREE / UNCONSCIOUS / TIED / DEAD | 世界状态 |

ADR-0011 的 `CanTieUp` 检查：
```csharp
return state == WorldState.UNCONSCIOUS && healthState == HealthState.DOWNED;
```

ADR-0011 的 `CanEnvironmentKill` 检查：
```csharp
return (state == WorldState.FREE || state == WorldState.UNCONSCIOUS)
    && identity == NPCIdentityType.ENEMY
    && hasWeapon;
```

**问题**：
1. `WorldState.UNCONSCIOUS` 与 `HealthState.DOWNED` 必须同步，但同步协议未经确认
2. 如果 NPC 被 BLUNT 攻击进入 STAGGERED（HealthState），但 WorldState 仍是 FREE，CanEnvironmentKill 会返回 true（允许处决），但这不是设计意图
3. 反之，如果 NPC 被 LETHAL 攻击进入 DOWNED，但 WorldState 未同步为 UNCONSCIOUS，CanTieUp 会失败

**影响范围**：ADR-0008（Health System 状态广播）、ADR-0011（交互条件判定）、ADR-0004（NPC AI 世界状态管理）

---

### 冲突 3：WeaponQueryRequest/Response 与 QueryBus 模式混用

**问题描述**：

ADR-0010 §6.1.1 和 ADR-0011 §7 使用事件驱动的方式查询 WeaponData：
```csharp
// ADR-0010
EventBus.Instance.Publish(new WeaponQueryRequest { weapon_id = ... });
// 异步等待 WeaponQueryResponse
```

但 ADR-0018 §2.2 定义的 Query 模式是同步的：
```csharp
// ADR-0018
TResult result = QueryBus.Instance.Query<TRequest, TResult>(request);
```

**问题**：
1. WeaponQueryRequest 是通过 EventBus 发布的（异步），而不是通过 QueryBus（同步）
2. 这与 ADR-0018 定义的 Query 模式不一致

**建议**：
1. 要么将 WeaponQueryRequest 改为通过 QueryBus 处理（同步模式）
2. 要么将 WeaponQueryRequest 明确标记为"事件风格的请求-响应"，与 QueryBus 的同步 Query 区分开

---

### 冲突 4：ScreenEffectSource.CombatSystem 与 HealthSystem 的屏幕特效职责重叠

**问题描述**：

ADR-0023 定义了两个可能请求受伤闪红的来源：
- `ScreenEffectSource.HealthSystem` — 受伤闪红
- `ScreenEffectSource.CombatSystem` — 战斗震动

但 ADR-0008 HealthSystem 处理伤害后应该请求屏幕特效，ADR-0011 GrittyTakedowns 在处决时也可能请求屏幕特效。

**问题**：受伤特效应该由 HealthSystem 还是 CombatSystem 请求？处决时是否需要屏幕特效？

**建议**：在 ADR-0008 或 ADR-0011 中明确屏幕特效的请求方和效果类型。

---

### 冲突 5：ADR-0010 的 WeaponUsedEvent 与 ADR-0011 的动画时序

**问题描述**：

ADR-0010 定义：
```csharp
public struct WeaponUsedEvent
{
    public string weapon_id;
    public string usage_type;  // "fire", "throw", "melee", "explosion"
}
```

ADR-0024 AnimationEventBridge 在 `OnWeaponSwing` 时发布此事件：
```csharp
EventBus.Instance.Publish(new WeaponUsedEvent { usage_type = "melee" });
```

**问题**：
1. `OnWeaponSwing` 动画事件在动画播放时触发，不代表命中
2. 武器"使用"（Swing）和"命中"是两个不同的事件
3. ADR-0011 的 EnvironmentKill 需要的是"命中"信息，而不是"使用"信息

---

## 总结

| 优先级 | 问题 | 影响系统 |
|--------|------|----------|
| **高** | 动画事件流程与 GrittyTakedowns 职责边界不清 | ADR-0011, ADR-0024, ADR-0008 |
| **高** | WorldState ↔ HealthState 同步协议未落实 | ADR-0008, ADR-0011, ADR-0004 |
| **中** | base_damage / damage_amount 语义不一致 | ADR-0010, ADR-0008 |
| **中** | WeaponQueryRequest 模式与 QueryBus 不一致 | ADR-0010, ADR-0011 |
| **低** | WeaponAwareness 命名过时 | ADR-0010 |
| **低** | stagger_multiplier 硬编码 | ADR-0008 |

---

## 附录：验证建议

### 必须验证的场景

1. **爆炸物测试**：
   - C4 在 3m 半径内是否造成 Lethal 伤害
   - 手榴弹在 2m 半径内是否造成 Lethal 伤害
   - 超出 lethal_ratio 但在 blast_radius 内的伤害是否为 Blunt 且有 1.5 倍 stagger

2. **潜行击杀流程**：
   - 动画播放 → AnimationEvent → DamageRequest → Health 处理 → NPCStateChangedEvent → InteractionEvent
   - 验证事件顺序和内容

3. **WorldState ↔ HealthState 同步**：
   - BLUNT 攻击后 WorldState 仍为 FREE，HealthState 为 STAGGERED
   - DOWNED 后 WorldState 应变为 UNCONSCIOUS

4. **投掷物交互**：
   - 持有环境物件时 CanEnvironmentKill 返回 true
   - 投掷后物件状态变更正确
