
## GA 相关流程

---

### Give Ability 流
- Only runs on Authority 
```cpp
// 相关方法
public FGameplayAbilitySpecHandle ASC::GiveAbility(const FGameplayAbilitySpec& Spec)
protected virtual void ASC::OnGiveAbility(FGameplayAbilitySpec& Spec)
```
:pencil2: **Start** 
1: 实例化 `GA Spec`

- 获取 `UGameplayAbility`'s CDO 并**构造** `GA Spec`

2: 授予 `Spec` : `ASC::GiveAbility()`
- **if** *Not Authority*, **return (中断流程)**
- 加锁 `ABILITYLIST_SCOPE_LOCK()`
- 添加 `Spec` 到 `ASC::ActivatableAbilities`
  - `FGameplayAbilitySpec& OwnedSpec = ActivatableAbilities.Items[ActivatableAbilities.Items.Add(Spec)]`
    :warning: Add 时会进行拷贝, 因此这里 OwnedSpec != Spec

3: **if** GA is *InstancedPerActor*, **创建 a new instance** of  `UGameplayAbility` 
- **if** GA is *ReplicateYes*
  - 加入到 `Spec.ReplicatedInstances`
  - **注册为 ReplicatedSubObject**: `ASC->AddReplicatedSubObject(instance)`
- **else** 加入到 `Spec.NonReplicatedInstances`

4: ASC 侧在 GA 授予后的处理
- 注册 GA Triggers: `Spec.Ability->AbilityTriggers`
- 提供运行时上下文给 GA Instance : `UGA::OnGiveAbility(const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilitySpec& Spec)`
  

5: Mark GA Spec Dirty, finally return `Spec.Handle`
- 触发同步(*FFastArraySerializer*)

---

### Try Activate Ability 流

```cpp
// 相关方法
bool UAbilitySystemComponent::TryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate, bool bAllowRemoteActivation)
bool UAbilitySystemComponent::InternalTryActivateAbility(FGameplayAbilitySpecHandle Handle, FPredictionKey InPredictionKey, UGameplayAbility** OutInstancedAbility, FOnGameplayAbilityEnded::FDelegate* OnGameplayAbilityEndedDelegate, const FGameplayEventData* TriggerEventData)
```
:pencil2: **Start**
1: 执行 Activate 的基础条件判断

- ASC's `OwnerActor 和 AvatarActor` are both Valid 
-  `AvatarActor->LocalRole` != SimulatedProxy

1.1: **if** *NetMode* 和 GA's *NetExecutionPolicy* 不匹配, 尝试触发 **Remote Activate** 流程
- 满足相关条件则会调用对应的 **Server/Client [RPC]**, 并 **return 结束流程**
  > LocalPredicted GA 的实际激活只能发生在 LocallyControlled 端，但激活请求可以被 Server 转发


###### 2: 进入 Internal Activate 流程: `InternalTryActivateAbility()`

- ASC侧 进行 Can Activate 检测
  - check `NetMode`
  - check `NetExecutionPolicy`
- GA侧 进行 Can Activate 检测 
  - `InstancedGA->CanActivateAbility(...)`
> if any above false, **return**

- 特殊处理 when *InstancedPerActor* `GASpec->IsActive()` 的情况
  - **if** GA 允许重复激活 => 为当前 GA Instance 执行 [End Ability 流](#end-ability-流) 
  - **else** **return** // :x: 不支持二次激活

3: 执行 Activate Ability
- 首先为本次激活**生成**一个Activation Info: `FGameplayAbilityActivationInfo`


**if** *Authority*

- 初始化 Info's `PredictionKeyWhenActivated`
  - **if** 参数 `InPredictionKey` 有效, 则直接使用该Key 
    > LocalPredicted GA 通常会传入此值
- 创建**预测窗**:
  - `FScopedPredictionWindow ScopedPredictionWindow(this, ActivationInfo.GetActivationPredictionKey())`
- **if** GA is *ServerInitiated*, [RPC] 通知 Client 激活成功:		
  - [Client Confirm GA Succeed 流](#client-confirm-ability-succeed-流)
- :pushpin: GA侧 进行激活: `UGA::CallActivateAbility(...)`

**else if** GA is *LocalPredicted*  [Client]
- 创建**预测窗**: 
  - `FScopedPredictionWindow ScopedPredictionWindow(this, true)`
- 初始化 Info 为 *预测模式*
- [RPC] 请求 Server 尝试激活:
  - [Server Try Activate GA 流](#server-try-activate-ability-流)
- :pushpin: GA侧 进行激活: `UGA::CallActivateAbility(...)`
- **预测窗**销毁

3.1:  **if** GA is Instanced, 设置 Activion Info
- `InstancedAbility->SetCurrentActivationInfo(ActivationInfo)`

4: Mark GA Spec Dirty 

---

#### [Server] Try Activate Ability 流
- 本流程针对 Client tells Server to Activate GA 这种情况
- Only runs on Authority 
```cpp
void UAbilitySystemComponent::InternalServerTryActivateAbility(FGameplayAbilitySpecHandle Handle, bool InputPressed, const FPredictionKey& PredictionKey, const FGameplayEventData* TriggerEventData)
```
:pencil2: **Start**
1: 基础检测 
 - check `Spec != null`, `GetNetSecurityPolicy`...
 - **if** any False, [RPC] 告知 Client 激活失败  
   - `ClientActivateAbilityFailed(Handle, PredictionKey.Current)`

2: 准备 Activate :pushpin:
 - 重置 TargetData 相关数据: `AbilityTargetDataMap` 
   - `AbilityTargetDataMap`, `FGameplayAbilitySpecHandleAndPredictionKey`
 - 创建**预测窗**:
   - `FScopedPredictionWindow ScopedPredictionWindow(this, PredictionKey)`

3: [Internal Activate GA 流](#2-进入-internal-activate-流程-internaltryactivateability)
- **if** 内部激活 return False :
  - [RPC] 告知 Client 激活失败 
  - Mark GA Spec Dirty 



---
#### [GA] Activate Ability 流
- GA层 的激活流程
```cpp
protected void UGameplayAbility::CallActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, FOnGameplayAbilityEnded::FDelegate* OnGameplayAbilityEndedDelegate, const FGameplayEventData* TriggerEventData)
```
:pencil2: **Start**
1: Pre Activate 处理
- FlushServerMoves ? (todo...)
- 初始化 Current Info 
  - 只对 InstancedGA 生效, 包括 ActorInfo 和 AcitivationInfo
- Apply Loose Tags to self ASC (`this.ActivationOwnedTags`)
- Apply Block and Cancel Tags
  - 可触发 [Cancel Ability 流](#cancel-ability-流)
- `Spec->ActiveCount` 加1

2: 执行 Activate 
- **if** `GA.bHasBlueprintActivate == true`, 通知 BP 层 Activate 并 **return**
  - BP 自己负责调用 Commit GA

3: 执行 Commit GA: `UGA::CommitAbility()`
- 进行 Commit Check 
  - **最后一次机会检测**, 也意味着 CommitAbility 自带成功或失败的概念

- 实际 Commit , 可Override
  - Apply CoolDown GE and Cost GE 
  - 通知BP层 Commit Execute

---

### End Ability 流

- 无论 GA 释放成功/失败/取消, 最终都应该执行此流程来收尾
- 参数 `bWasCancelled` 表明是否因 *Cancel* 而结束
```cpp
// 相关方法
protected virtual void UGameplayAbility::EndAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateEndAbility, bool bWasCancelled)
void UAbilitySystemComponent::NotifyAbilityEnded(FGameplayAbilitySpecHandle Handle, UGameplayAbility* Ability, bool bWasCancelled)

// 相关属性
ASC::ActivatableAbilities
```
:pencil2: **Start**

1: [GA层] 先检测 End 流程能否正常执行, 用于防止被多次调用的情况 
- `Owner ASC != null && Spec->IsActive()...`

2: 开始 End Ability
- **if** `GA.ScopeLockCount > 0`, **defer** End 流程, and **return**
  - 通过注册 Callback 实现 defer: `this.WaitingToExecute.Add(FPostLockDelegate::CreateUObject(this, &UGameplayAbility::EndAbility, ...));`
- 通知 BP 层 End GA

3: 执行各种 Clean 工作

- Unregister from the World's managers
- **Broadcast:** `FOnGameplayAbilityEnded OnGameplayAbilityEnded`
- 切换状态: `this.bIsActive = false`
- 通知 `this.ActiveTasks` 结束并清空此队列

4: 针对 ASC 侧进行处理
- **if** `bReplicateEndAbility == true` 且 GA is *LocalPredicted* or *ServerInitiated*  
  - [RPC] 通知 Server or Client 执行 Cancel/End GA 流   
    // `ReplicateEndOrCancelAbility()`
- Remove `Tags/GameplayCues` this GA added **from the ASC**
- 清理 TargetData 相关数据
  - `ClearAbilityReplicatedDataCache()`

5: GA 层 End 完毕, 通知 ASC 层 PostProcess: `ASC::NotifyAbilityEnded(...)`
- `GASpec->ActiveCount` 减1
- **Broadcast** GA结束: `AbilityEndedCallbacks`, `OnAbilityEnded`
- **if** *Authority*
  - **if** `Spec->RemoveAfterActivation && Spec->ActiveCounnt == 0`
    - 尝试 Remove GA Spec from `ASC.ActivatableAbilities`
  - **else** Mark GA Spec Dirty

<br>

#### Cancel Ability 流

- Cancel 这个概念在GAS中是以 **GA Instance 为颗粒度**
  - `UGA::SetCanBeCanceled(bool bCanBeCanceled)`
  - Non-Instanced GA 默认 "Always Cancelable"
- 最终会实际执行 End Ability 流
  - Cancel 流会尝试触发 Remote Cancel [RPC], 因此传参时会指定 `bReplicateEndAbility = false`, 避免触发 Remote End

---
#### [Client] Confirm Ability Succeed 流

```cpp
void UAbilitySystemComponent::ClientActivateAbilitySucceedWithEventData_Implementation(FGameplayAbilitySpecHandle Handle, FPredictionKey PredictionKey, FGameplayEventData TriggerEventData)
```
:pencil2: **Start**
1: 尝试获取 GA Spec from Handle 
- **if** Spec not found , 意味着 `ActivatableAbilities` 尚未同步完成, defer handle and **return**
  - **生成并添加**一份 `FPendingAbilityInfo` 到 `PendingServerActivatedAbilities` 队列

- 设置 `Spec->ActivationInfo` 为 *确认模式*

2: 根据 `NetExecutionPolicy` 执行不同 Logic

- **if** GA is *LocalPredicted*
  - **if** GA is Instanced-type, **通过 `PredictionKey`** 找到对应的GA instance, 成功确认


- **else** (例如 GA is *ServerInitiated*)
  - 进入 [GA Activate Ability 流](#ga-activate-ability-流)

<br>

#### [Client] Confirm Ability Fail 流
- 注意这里参数类型 `int16 PredictionKey`
- 默认没有 revert 逻辑?
```cpp
void ASC::ClientActivateAbilityFailed_Implementation(FGameplayAbilitySpecHandle Handle, int16 PredictionKey)
```

1. :pushpin: **Broadcast "客户端预测激活被拒绝, 需要进行处理"**
   - `FPredictionKeyDelegates::BroadcastRejectedDelegat(PredictionKey)`
2. 设置 `Spec->ActivationInfo` 为 *拒绝模式*
3. 设置对应的 `Ability->CurrentActivationInfo` 为 *拒绝模式*
---
#### Mark GA Spec Dirty

---
### Ability Batching
todo...