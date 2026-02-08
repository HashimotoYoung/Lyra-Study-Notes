todo... 预测, 同步

# Gameplay Effect 相关流程

:books: GE 的核心流程为 Apply GameplayEffect, 在该流程中会**根据GE's DurationType** 走向两个不同的子流程: [Add GE Spec 流](#add-ge-spec-流) 和 [Execute GE Spec 流](#execute-ge-spec-流)

---

## 1. Apply GameplayEffect 流

:memo: 此示例以 `GA->ApplyGameplayEffectToTarget()` 为入口, 此方法最终会调转至 **核心API:** **`ASC::ApplyGameplayEffectSpecToSelf()`**


### Apply GE to Target 流
- GE Spec 实例会在过程中被**创建**
- :warning: 返回类型为 `FActiveGameplayEffectHandle`, 而非 `FGameplayEffectSpecHandle`
- :warning: 一个 `FGameplayAbilityTargetDataHandle` 可对应多个 `FGameplayAbilityTargetData`, 一个 `FGameplayAbilityTargetData` 可对应多个 `Actors`
```cpp
TArray<FActiveGameplayEffectHandle> UGameplayAbility::ApplyGameplayEffectToTarget(
  const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, 
  const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayAbilityTargetDataHandle& Target, 
  TSubclassOf<UGameplayEffect> GameplayEffectClass, float GameplayEffectLevel, int32 Stacks) const
```
:pencil2: **Start**
1: Apply 前的相关检测:
- **if** ! ( *HasAuthority* || `ShouldPredictTargetGameplayEffects()` ), **return**
- [Predict] **if** `HasAuthorityOrPredictionKey` ?? 
  - :pushpin:**生成 GE Spec**: `ASC::MakeOutgoingGameplayEffectSpec()`

  - **遍历** `Target.Data`, 对于每个 `FGameplayAbilityTargetData`: 
    - 获取 PredictionKey: `ActorInfo->ASC->GetPredictionKeyForNewAction()`
    - => **step 2**

2: 执行 `FGameplayAbilityTargetData::ApplyGameplayEffectSpec(FGameplayEffectSpec& InSpec, 
FPredictionKey PredictionKey)` 

- 获取 *目标 Actors* = `FGameplayAbilityTargetData::GetActors()`
  >开发时需 override (默认返回空数组)
- **遍历** target in *目标 Actors*, **if** target has ASC:
  - 拷贝 `FGameplayEffectSpec`  和 `FGameplayEffectContext` from InSpec
    > :pushpin: Provide each Target with **a fresh copy of the spec and context**
  - 填充 GATargetData 的相关数据到 GE Context 
  - 由 Target ASC 调用 [Apply GE to Self 流](#apply-ge-to-self-流-star)

---

### Apply GE to Self 流 :star:
- 注意在进入该流程前, GE Spec 已经在外部被生成
```cpp
FActiveGameplayEffectHandle ASC::ApplyGameplayEffectSpecToSelf(const FGameplayEffectSpec& GameplayEffect, FPredictionKey PredictionKey = FPredictionKey())
```
:pencil2: **Start**
1: [施加GE区域锁](#fscopedactivegameplayeffectlock-区域锁分析) 

2: 在 Do Apply 前按顺序进行5项检测, 未通过则 **return**
  - Check 权限和本地预测
  - Check Immunity: ASC 是否对该GE免疫
  - Check 本次施加成功率: `FMath::FRand()` < `Spec.GetChanceToApplyToTarget()`
  - Check 默认*Tag Requirements*
  - Check **自定义检测逻辑**: `UGameplayEffectCustomApplicationRequirement::CanApplyGameplayEffect()`


3.1: 开始 Do Apply GE, 先判断/处理 *Duration/Infinite* GE

**[Predict]** 定义 **`bool bTreatAsInfiniteDuration`** = Is *Instant* GESpec && *LocalPredict* 
定义 `FGameplayEffectSpec* OurCopyOfSpec` = NULL
- **if** Is *Duration/Infinite* GESpec || `bTreatAsInfiniteDuration`
  - => [Add GE Spec 流](#add-ge-spec-流), 如果未成功 Apply 则 **return**
  - `OurCopyOfSpec` = `&(AppliedEffect->Spec)`
- **if** `!OurCopyOfSpec` // Instant类型GE默认
  - `OurCopyOfSpec` = **拷贝构造一份 Spec**
  - **Capture** Target's Attributes
- **if** `bTreatAsInfiniteDuration`
  - 手动将该 GE Spec 改为 *Infinite* 类型

- 尝试 Invoke GC Events: `OnActive/WhilActive`相关

3.2: 后判断/处理 *Instant* GE 
- **if** `bTreatAsInfiniteDuration`
  - 尝试 Invoke GC Events: `Execute` 
    //:pushpin: 预测端不执行逻辑，只触发表现
- **else if** Is *Instant* GESpec
  - => [Execute GE Spec 流](#execute-ge-spec-流)

4: **[Authority]** 尝试在Apply时 Remove 已激活的GEs 
- `ActiveGameplayEffects.AttemptRemoveActiveEffectsOnEffectApplication()`

5: 尝试 Apply **Linked** GEs
- 会调用 `ApplyGameplayEffectSpecToSelf()`

6: 通知 Apply 完成
- 通知 *Self ASC* (this) to **Broadcast** `OnGameplayEffectAppliedDelegateToSelf`
- 通知 *Instigator ASC* to **Broadcast** `OnGameplayEffectAppliedToTarget`
  - *Instigator ASC* 可从Context中获取

---

### Add GE Spec 流
- 只有 Duration-based GE 会执行此流程
```cpp
FActiveGameplayEffect* FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(const FGameplayEffectSpec& Spec, FPredictionKey& InPredictionKey, bool& bFoundExistingStackableGE)
```
:pencil2: **Start** from the func

1: [施加GE区域锁](#fscopedactivegameplayeffectlock-区域锁分析) 

2: FindOrCreate a `FActiveGameplayEffect* AppliedActiveGE`

**if** 发现已存在 an **Active-Stackable** GE Spec
  - **if** *Not Authority*, **return** // 禁止预测该类GE
  - todo... handle stacakble
  - `AppliedActiveGE` = `ExistingStackableGE`

**else**

- **生成**句柄 NewHandle: `FActiveGameplayEffectHandle`
- **if** 存在区域锁 && `this.GameplayEffects_Internal` 没有空余位置, defer
  - todo...
- **else** **创建** `FActiveGameplayEffect` 并加到队列 :pushpin:
  - 传递 `InPredictionKey` 用于预测
  ```cpp
  AppliedActiveGE = new(GameplayEffects_Internal) FActiveGameplayEffect(NewHandle, Spec, GetWorldTime(), GetServerWorldTime(), InPredictionKey);
  ```

<br>

> 从这里开始的 Spec 为 `AppliedActiveGE->Spec` 

<br>

3: **Capture** Target's ActorTags and Attributes 



4: Handle Modifiers 
- 让 Spec 执行一次 *Modifier Magnitudes 计算*
- 满足以下条件时会 Build `Spec.ModifiedAttribute` list

5: 重新计算 Duration 

6: **if** GE has Peroid , 注册 Peroid Callbacks

7: [Server] 处理网络同步部分

8: Post Process
- **if** `ExistingStackableGE`
  - todo...
  
  **else** [Internal Add Active GE 流](#internal-add-active-ge-流)

<br>

#### Internal Add Active GE 流

主要处理当添加一个 `FActiveGameplayEffect`后, ASC 等模块所受到的影响
- 属于内部 private 流程
- **只有 Duration-based GE** 会进入此流程
```cpp
FActiveGameplayEffectsContainer::InternalOnActiveGameplayEffectAdded(FActiveGameplayEffect& Effect)
```
:pencil2: **Start**
1: [施加GE区域锁](#fscopedactivegameplayeffectlock-区域锁分析)

2: 记录 *Effect* 和 *Effect附带的Tags* 的关系到 `this.ActiveEffectTagDependencies`
- 目的:??

3: 建立绑定关系和 MMC 类or实例?? used by Effect.Modifiers

4: :star: **更新 Effect 的禁用状态:**  `FActiveGameplayEffect::CheckOngoingTagRequirements()`
   > :memo: 每个 FActiveGameplayEffect 都存在 **启用/禁用** 的概念 (`bIsInhibited`), 这里在Add时进行一次更新

检查是否满足切换条件
  - 判断完全基于 `GE->OngoingTagRequirements` 

**if** 需要切换, 启用(Add) 或 禁用(Remove) : **Effect 附带的 GrantedTags 和 Modifiers** 
> 这里以 Add 为例
- 施加区域锁 *FScopedAggregatorOnDirtyBatch*
- 处理 Modifiers, 会根据 if Effect has Peroid 做不同处理
  - **if**  No Period, **遍历** `Effect.Spec.Modifiers`, 对于每个Mod:
    - 检查 OwnerASC 是否配有 **AS** for the attribute (Modifier指定), continue if not.
    - **FindOrCreate 一个 `FAggregator`** for the attribute , then Add Mod to it
  
  - **else if** *$IsAuthority*  
    - todo... peroid处理
- Grant *附带的各种Tags* to Owner(ASC)
- **if** *$IsOwnerAuthority*, Grant GA Specs to Owner(ASC)
- 触发 Add GCue 流
- **Broadcast** "已添加一个 Active GE" : `Owner->OnActiveGameplayEffectAddedDelegateToSelf`



---

### Execute GE Spec 流 
只有 Instant GE 会执行此流程
```cpp
void FActiveGameplayEffectsContainer::ExecuteActiveEffectsFrom(FGameplayEffectSpec &Spec, FPredictionKey PredictionKey)
```

:pencil2: **Start** from the func

1: **Capture** Target(ASC)'s ActorTags 

2: Apply Modifiers 

让 Spec 执行一次 *Modifier Magnitudes 计算 (`CalculateModifierMagnitudes`)* 

遍历 `Spec.Modifiers`, 对**每个 Mod** 进行内部处理: `InternalExecuteMod()`
  - 获取 Mod 所绑定的 Attribute 和其 AttributeSet
    - 找不到直接 **return** ( 目标可能本身没有对应属性 )
  - :pushpin: 通知 AS : **`bool AS::PreGameplayEffectExecute()`**
  - **Apply Mod**  : `ApplyModToAttribute()`
    - 求出 NewBaseValue = Apply `ModifierOp`和 `ModifierMagnitude` 到当前BaseValue
    - 执行 [Set Base Value 流](#set-base-value-流)
  
  - 更新记录 `Spec.ModifiedAttributes` 中的变化总量 (`TotalMagnitude`)
  - :pushpin: 通知 AS : **`AS::PostGameplayEffectExecute()`**


3: 执行 Executions 流程

- **遍历** `SpecToUse.Def->Executions`,
- todo...
- Collect **Execution-conditional** GEs

4: 触发 GCue Events

5: Try Apply **Execution-conditional** GEs
- 会调用 `ASC::ApplyGameplayEffectSpecToSelf()`
---

## 2. Modify Attribute 相关流程

### Set Base Value 流
GAS中修改 Attribute.BaseValue 的逻辑流程
- ASC 提供了 API 直接修改 BaseValue (:warning:NO for CurrentValue)
- CurrentValue 基于 BaseValue 存在, 因此修改 BaseValue 后会立即更新 CurrentValue
```cpp

FActiveGameplayEffectsContainer::SetAttributeBaseValue(FGameplayAttribute Attribute, float NewBaseValue)
```
:pencil2: **Start**
1: 通知 AS  `NewBaseValue` (**传引用**): **`AS::PreAttributeBaseChange()`**

2: 获取 `FGameplayAttributeData` 并赋值 BaseValue

3: Update CurrentValue, 分两种情况:
- **if** 存在和 Attribute 相关联的 *Aggregators*  (`AttributeAggregatorMap 属性`)
  - todo
- **else** 执行内部更新流程 : `InternalUpdateNumericalAttribute()` 
   > 这里用 NewBaseValue 作为参数, 更新后默认 CurVal == BaseVal

  - 通知 AS `NewValue` (**传引用**): **`AS::PreAttributeChange()`**
    
  - 获取 `FGameplayAttributeData` 并赋值 CurrentValue
  - 通知 AS: **`AS::PostAttributeChange()`**
  - **BroadCast** 属性变换:  `AttributeValueChangeDelegates`

4: 通知 AS:  **`AS::PostAttributeBaseChange()`**

---

## 3. Key Takeaways

### `FScopedActiveGameplayEffectLock` 区域锁分析
```cpp
#define GAMEPLAYEFFECT_SCOPE_LOCK()	FScopedActiveGameplayEffectLock ActiveScopeLock(*this);
```
FActiveGameplayEffectsContainer::DecrementLock
- 特性:
- 目的: