# GE Attribute Modify 属性修改 

---
## 1. GE Modifiers (属性修改器)

:books: GE 通过使用 Modifiers(Mods) 来实现 **Modify Attribute** 的功能, mods define the rules **how** an attribute should be changed.

- 一个 GE 可配有 0-N 个 Mods; One Mod 允许修改 **Exactly One** Attribute 
- 支持 Prediction 

Modifier 自身只关注三件事:
  1. **Which Attribute** to Modify
  2. **Outputs** a `float` (Evaluated Magnitude)
  3. **How** the Magnitude should be applied (加减乘除)

  > :pushpin: 至于输出结果最终被应用于 BaseValue 还是 CurrentValue, 由其 OwnerGE's DurationType 决定

---

### FGameplayModifierInfo

Modifier 的配置类, 无任何成员方法

- UE中还定义了一个 `FGameplayEffectExecutionScopedModifierInfo` 类,其数据结构和此类基本一致, 专门用于 GE Executions 

##### 类关系:

- Contained by `UGameplayEffect`  as `TArray<FGameplayModifierInfo> Modifiers` 
  - 因此通常以 `GESpec.Def->Modifiers` 的形式被使用

##### 主要属性:

`FGameplayAttribute Attribute` 需要修改的属性

`TEnumAsByte<EGameplayModOp::Type> ModifierOp = EGameplayModOp::Additive`

- 定义使用哪种运算类型 apply the Evaluated Magnitude, 默认为 "加"

`FGameplayEffectModifierMagnitude ModifierMagnitude`
- `FGameplayEffectModifierMagnitude` 类主要负责执行内部计算流程并输出 *Evaluated Magnitude*
- 典型使用示例:
	```cpp
	void FGameplayEffectSpec::CalculateModifierMagnitudes() {
		for(int32 ModIdx = 0; ModIdx < Modifiers.Num(); ++ModIdx) {
			const FGameplayModifierInfo& ModDef = Def->Modifiers[ModIdx];
			FModifierSpec& ModSpec = Modifiers[ModIdx];
			
			// ModifierMagnitude 输出计算结果赋给 ModSpec.EvaluatedMagnitude
			if (ModDef.ModifierMagnitude.AttemptCalculateMagnitude(*this, ModSpec.EvaluatedMagnitude) == false) {
				ModSpec.EvaluatedMagnitude = 0.f;
			}
		}
	}
	```
  
---

## 2. GE Aggregator (属性聚合器)

> :memo: "The **Current Value** of an attribute is the **Base Value** evaluated through the attribute’s FAggregator, which **aggregates all modifiers from Active GEs** on the ASC."

#### 生命周期:
- :pushpin: When Created: 当 Apply **Duration-based && No Peroid** 型GE
- Who Contains: 由 `FActiveGameplayEffectsContainer` 负责创建, 持有和管理
  - 也可以认为 Aggregtors are Owned by ASC
  - Aggregator 是基于 Attribute 存在的计算节点 ( Exists **per-attribute-per-ASC** ) 
  - :warning: 在引用 Aggregator 时需要留意具体引用了哪个 ASC 的 Aggregator

#### 三个主要功能和特性:

1. 添加/删除 GE Modifier: 
   - Aggregator 的设计是建立在 Mod 的基础之上的, 一个 Aggregator 必须要在添加 Mod 后才真正具有评估能力 
2. 执行内部评估流程, 并最终输出一个 *Aggregated Evaluated Magnitude* (`float`)
3. [在 Dirty 时通知](#broadcast-on-dirty)
   - 通知的对象一般包括: OwnerASC 和 Dependents (Active GEs)

#### Broadcast On Dirty

> Aggregator 通过此特性建立起 Attributes 之间的关联性

What does “Dirty” mean: 
- 当一个 Aggregator "Get Dirty" 时, 意味着 the aggregator’s last evaluated result **is no longer valid**, 因此需要进行 **re-evaluation**

When gets Dirty: 
- 有 Mod 添加/删除
- BaseValue changed
- 等任何可能影响评估结果的 "输入" 发生变化

What happen on Dirty: 
- [see function](#broadcastondirty-star)

---

### FAggregator

GE Aggregator 的实现类

##### 类关系:

- Contained by `FActiveGameplayEffectsContainer` as `TMap<FGameplayAttribute, FAggregatorRef> AttributeAggregatorMap`
- Refered by `FGameplayEffectAttributeCaptureSpec`

##### 主要属性:
```cpp
float BaseValue; // Cache
FAggregatorModChannelContainer ModChannels; // 内部计算实现
TArray<FActiveGameplayEffectHandle>	Dependents; // 监听此 Aggregator 的 GE
FOnAggregatorDirty OnDirty // Delegate
FOnAggregatorDirty OnDirtyRecursive // Delegate
```

##### 主要方法:

#### `BroadcastOnDirty()` :star:
- 回调通知 Owner: [OnAttributeAggregatorDirty](./GE_Classes.md#onattributeaggregatordirtyfaggregator-aggregator-fgameplayattribute-attribute-bool-fromrecursivecallfalse)

- 遍历通知每个 Dependent (依赖于该 Aggregator 的 **Active GE**): `FActiveGEContainer::OnMagnitudeDependencyChange(FActiveGameplayEffectHandle Handle, const FAggregator* ChangedAgg)`
  1. 根据 Handle 找到 `FActiveGameplayEffect` 
  2. 找到 **依赖于 ChangedAgg 的 Modifier, 并执行 re-evaluation**
  3. :pushpin: 重新评估会进而造成 ModDef.Attribute 的 Aggregator "Get Dirty", 会再次触发 `BroadcastOnDirty()`, 形成**链式传播**

#### `Others:`

```cpp
// 添加/删除/更新 Mod, 执行后会立即调用 BroadcastOnDirty()
// UpdateAggregatorMod() 执行时也会添加 Mod
AddAggregatorMod()
RemoveAggregatorMod(FActiveGameplayEffectHandle ActiveHandle)
UpdateAggregatorMod()

// 添加/删除依赖者
AddDependent(FActiveGameplayEffectHandle Handle)
RemoveDependent(FActiveGameplayEffectHandle Handle)

// 输出评估值
float Evaluate(const FAggregatorEvaluateParameters& Parameters) const
```







