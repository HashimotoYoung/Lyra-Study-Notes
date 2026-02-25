## 1. GE 基础类

### UGameplayEffect : UObject

- `UGameplayEffect` subclasses 本质上为数据模板 


##### 主要属性:

```cpp
// 用于 Modify Attribute  
TArray<FGameplayModifierInfo> Modifiers
TArray<FGameplayEffectExecutionDefinition> Executions

// 用于 Invoke GCue
TArray<FGameplayEffectCue>	GameplayCues
```

---

### FGameplayEffectSpec 

GE 的具体应用实例 

##### 类关系:
- When an GA / ASC wants to apply a GE, it creates a `FGameplayEffectSpec` from `UGameplayEffect`'s **CDO** via `ASC::MakeOutgoingSpec()`
- When a **Duration-based** `FGameplayEffectSpec` is successfully applied to the ASC, it generates a  `FActiveGameplayEffect`, which are then added to the ASC's `FActiveGameplayEffectsContainer`
##### 主要属性:

`TArray<FModifierSpec> Modifiers`

- Cache the **last evaluated value**  of the modifier magnitude, 通常会在处理 Modify 相关的逻辑前, 调用相关方法(例如 `CalculateModifierMagnitudes()`) 进行一次更新

`TArray<FGameplayEffectModifiedAttribute> ModifiedAttributes`
- 记录各属性变化总值 (此GE造成的)
  ```cpp
  struct FGameplayEffectModifiedAttribute {
    UPROPERTY()
    FGameplayAttribute Attribute;
    UPROPERTY()
    float TotalMagnitude;
  }
  ```
<br>

`FGameplayEffectAttributeCaptureSpecContainer CapturedRelevantAttributes`: UPROPERTY(NotReplicated)  
`FTagContainerAggregator	CapturedSourceTags`: UPROPERTY(NotReplicated)  
`FTagContainerAggregator	CapturedTargetTags`: UPROPERTY(NotReplicated)
- [Capture Data 相关](./GE_DataCapture捕获.md#ge-中的-捕获-机制)

##### 其它属性:
```cpp
TObjectPtr<const UGameplayEffect> Def // 指向 GE 模板
float Duration; Period; ChanceToApplyToTarget; Level; // GE相关属性
FGameplayEffectContextHandle EffectContext // 上下文信息句柄
```
<br>

##### 主要方法:

#### `Initialize(const UGameplayEffect* InDef, const FGameplayEffectContextHandle& InEffectContext, float Level = FGameplayEffectConstants::INVALID_LEVEL)`

- 设置 Level 等基本属性
- 绑定 GE Context: `FGameplayEffectContextHandle& InEffectContext`
- Capture **Source** Tags 
- Capture **Source** Attributes 

---

### FGameplayEffectSpecForRPC
todo...

---

### FActiveGameplayEffectsContainer :  `FFastArraySerializer`

:star: 虽然命名为"Container", 但实际为 Manager 类,是 GE 中**最核心**的数据结构
##### 类关系:
- Owned by ASC with UPROPERTY(**Replicated**) 

##### 主要属性:
`UAbilitySystemComponent* Owner` // 被 ASC 持有  
`TArray<FActiveGameplayEffect>	GameplayEffects_Internal` // 持有 active GEs

`TMap<FGameplayTag, TSet<FActiveGameplayEffectHandle> >	ActiveEffectTagDependencies`
- 快捷对照表, 记录 Tag 和 `FActiveGameplayEffects` owned  

**`TMap<FGameplayAttribute, FAggregatorRef>		AttributeAggregatorMap`** :star:
- Container类负责为对应属性生成/管理对应的Aggregator

- :pushpin: 如果一个属性没有绑定任何 Aggregator, 则其 **CurrentValue === BaseValue**

`mutable int32 ScopedLockCount`   
`FActiveGameplayEffect*	PendingGameplayEffectHead`  
`FActiveGameplayEffect** PendingGameplayEffectNext`
- Apply GE 流 相关


##### 主要方法:

#### `FActiveGameplayEffect* ApplyGameplayEffectSpec()`
```cpp
FActiveGameplayEffect* ApplyGameplayEffectSpec(const FGameplayEffectSpec& Spec, FPredictionKey& InPredictionKey, bool& bFoundExistingStackableGE)
```
- [Apply GE Spec 流](./GAS_GE流程.md#apply-ge-spec-流)  主入口
<br>

#### `ExecuteActiveEffectsFrom()`
```cpp
void ExecuteActiveEffectsFrom(FGameplayEffectSpec &Spec, FPredictionKey PredictionKey)
```

- [Execute GE Spec 流](./GAS_GE流程.md#execute-ge-spec-流) 主入口
<br>

#### `ApplyModToAttribute()`
```cpp
ApplyModToAttribute(const FGameplayAttribute &Attribute, TEnumAsByte<EGameplayModOp::Type> ModifierOp, float ModifierMagnitude, const FGameplayEffectModCallbackData* ModData=nullptr)
```
负责修改 Attribute's **BaseValue** 
- 内部会调用 `SetAttributeBaseValue()`方法

- :warning: 注意该方法和 Modifier 类无实际耦合, 可用于**直接修改Attribute** 


<br>

#### `OnAttributeAggregatorDirty(FAggregator* Aggregator, FGameplayAttribute Attribute, bool FromRecursiveCall=false)`
属性修改的核心API之一, 有两个主要来源: 
1. Aggregator 在dirty时广播回调
2. *GAMEPLAYATTRIBUTE_REPNOTIFY* 时

- **if** Not Authority && **执行网络同步时** && 第一次处理时
  > 此时 Attri's BaseValue 已同步
  - 更新 `Aggregator's BaseValue`
    - from `Owner->GetNumericAttributeBase(Attribute)`
- 触发 Aggregator 重新评估
  - `float NewValue = Aggregator->Evaluate(EvaluationParameters)`
- 更新 Attri's Current Value
  - `InternalUpdateNumericalAttribute(Attribute, NewValue, nullptr, bFromRecursiveCall)`



<br>

#### `AddActiveGameplayEffectGrantedTagsAndModifiers(FActiveGameplayEffect& Effect, bool bInvokeGameplayCueEvents)`

:memo: 在确认"启用"一个GE后, 添加其所携带的 Tag 和 Mods

1: 加锁 *FScopedActiveGameplayEffectLock*
2: 处理 Modifiers
  - **if** `Effect.Spec` is No Period, **遍历 `Effect.Spec.Modifiers`, 对于每个Mod:**
    - **if** Owner ASC 没有任何 AS 包含该 Modifier 所指定的 attribute  
      - **continue** 
    - :pushpin: **FindOrCreate 一个 `FAggregator` 并和该 attribute 进行绑定** , 
    - 添加 Mod 到此 `FAggregator` 中
      - `FAggregator::AddAggregatorMod(...)`
  
  - **else** todo... (period处理)

3: 授予 *附带的 Tags* 到 Owner ASC

4: 授予 `GA Specs` 到 Owner ASC

5: 触发 Add GCue 流
- `Owner->InvokeGameplayCueEvent(Effect.Spec, EGameplayCueEvent::OnActive)`
`Owner->InvokeGameplayCueEvent(Effect.Spec, EGameplayCueEvent::WhileActive)`

6: **Broadcast** "已添加一个 Active GE" : `OnActiveGameplayEffectAddedDelegateToSelf`

---

### FActiveGameplayEffect : `FastArraySerializerItem`

##### 主要属性:
`FTimerHandle PeriodHandle`, `FTimerHandle DurationHandle`

`FPredictionKey	PredictionKey`
- 附带的预测Key

`bool bIsInhibited`
- UPROPERTY(NotReplicated)
- 表示此 GE 处于启用(`false`)或禁止(`true`)状态

---
### FGameplayEffectContext

GE的上下文类, **鼓励继承**
- `WithNetSerializer`, `WithCopy`
- 需要用好 `Duplicate()` 已捕捉Apply GE时的实际情景
##### 类关系:

- Shared via `FGameplayEffectContextHandle` (TSharedPtr型)
- 创建时机: 通常在**生成 GESpec 之前或一起创建**，并由 GESpec 持有其 Handle
  > :star: It is common for **multiple** `FGameplayEffectSpecs` to share a **single** `FGameplayEffectContext` when they originate from the same gameplay action
  ```cpp
  // 示例
  FGameplayEffectSpecHandle UAbilitySystemComponent::MakeOutgoingSpec(TSubclassOf<UGameplayEffect> GameplayEffectClass, float Level, FGameplayEffectContextHandle Context) const
  {
		SCOPE_CYCLE_COUNTER(STAT_GetOutgoingSpec);
		if (Context.IsValid() == false) {
			// 此处创建一个Context 并返回其 Handle
			Context = MakeEffectContext();
		}

		if (GameplayEffectClass) {
			UGameplayEffect* GameplayEffect = GameplayEffectClass->GetDefaultObject<UGameplayEffect>();
			FGameplayEffectSpec* NewSpec = new FGameplayEffectSpec(GameplayEffect, Context, Level);
			return FGameplayEffectSpecHandle(NewSpec);
		}
		return FGameplayEffectSpecHandle(nullptr);
  }
  ```

##### 主要属性:

`TWeakObjectPtr<AActor> Instigator`
  - GE 发起者, 通常指代 ASC's **OwnerActor** (例如`LyraPlayerState`)

`TWeakObjectPtr<AActor> EffectCauser` 
  - GE 施法者, 通常指代 ASC's **AvatarActor**  (例如`ALyraCharacter`), 也可为武器,飞弹等投射物

`TWeakObjectPtr<UObject> SourceObject` 
  - GE 源头, 如果此 Context 由 GA 生成, 则默认实现为 `= AbilitySpec->SourceObject` (通常指代 the UObject **who grants the GA to ASC**, 例如 Lyra 中的 `UWeaponInstance`)


`TArray<TWeakObjectPtr<AActor>> Actors` 所有涉及到的Actors 

<br>

#### :mag: About `FGameplayEffectContext` Subclasses

- 需要 Override `GetScriptStruct()`
- 需要 Override `Duplicate()`: 以便能够对特定属性 **DeepCopy**
- 需要 Override `NetSerialize()`: 如果新增属性需要同步
- Implement `TStructOpsTypeTraits`
- Override `AllocGameplayEffectContext()`


---

## 2. GE AttriCapture 相关类

### `FGameplayEffectAttributeCaptureSpec`
- 主要属性:
  ```cpp
  FGameplayEffectAttributeCaptureDefinition BackingDefinition 
  FAggregatorRef AttributeAggregator
  ```

### `FGameplayEffectAttributeCaptureDefinition`

- 主要属性:
    ```cpp
    FGameplayAttribute AttributeToCapture // 绑定的属性
    EGameplayEffectAttributeCaptureSource AttributeSource // Source 或 Target 二选一
    bool bSnapshot // 是否snap??
    ```
