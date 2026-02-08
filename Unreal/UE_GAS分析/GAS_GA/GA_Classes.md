# GA 类分析

---

### UGameplayAbility : UObject

- Instanced GAs **ARE replicated sub-objects**.

##### 类关系:
- A `UGA` instance is owned and managed by the `ASC`, but not as a replicated child object 
- GA与GA之间可以通过 Tags, 建立互斥, 阻碍等逻辑关系
- Is the Owner of `AbilityTasks` (继承了 `IGameplayTaskOwnerInterface`)

##### 主要属性:
`FGameplayTagContainer AbilityTags`
- 代表GA自身的Tag, 此外GA也持有很多与游戏逻辑相关的TagContainer,例如 `FGameplayTagContainer BlockedTags`...

`FGameplayAbilityActivationInfo	CurrentActivationInfo`
- 激活时信息, [详见](#fgameplayabilityactivationinfo)
##### `mutable const FGameplayAbilityActorInfo* CurrentActorInfo`  
- 可用于获取 ASC, [详见](#fgameplayabilityactorinfo)

`mutable FGameplayAbilitySpecHandle CurrentSpecHandle`
- Instanced GA 专用属性

`TSubclassOf<UGameplayEffect> CostGameplayEffectClass`
`TSubclassOf<UGameplayEffect> CooldownGameplayEffectClass`
- 默认可选择分别绑定一个**Cost/Cooldown类**, 并在Commit GA时自动Apply

`mutable int8 ScopeLockCount`
`mutable TArray<FPostLockDelegate> WaitingToExecute`
- todo...

`TArray<FAbilityTriggerData> AbilityTriggers`
- 决定此GA是否可被特定的Tags触发
- 使用示例:
  ```cpp
  ULyraGameplayAbility_Death::ULyraGameplayAbility_Death(const FObjectInitializer& ObjectInitializer) : Super(ObjectInitializer) {
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;
    bAutoStartDeath = true;
    //判断 CDO 
    if (HasAnyFlags(RF_ClassDefaultObject)) {
      FAbilityTriggerData TriggerData;
      TriggerData.TriggerTag = LyraGameplayTags::GameplayEvent_Death;
      TriggerData.TriggerSource = EGameplayAbilityTriggerSource::GameplayEvent;
      AbilityTriggers.Add(TriggerData);
    }
  }
  ```
其它属性:
```cpp
// 各种 Policy
TEnumAsByte<EGameplayAbilityInstancingPolicy::Type>	InstancingPolicy
TEnumAsByte<EGameplayAbilityNetExecutionPolicy::Type> NetExecutionPolicy
TEnumAsByte<EGameplayAbilityReplicationPolicy::Type> ReplicationPolicy 

TArray<TObjectPtr<UGameplayTask>> ActiveTasks // AbilityTasks
TObjectPtr<UAnimMontage> CurrentMontage // Montage引用
FGameplayEventData CurrentEventData
```

##### 主要方法:
```cpp
// 返回 true if this ability is currently executing on a client using prediction
bool UGameplayAbility::IsPredictingClient()
```

---

### FGameplayAbilitySpec : FFastArraySerializerItem

> :books: `FGameplayAbilitySpec` 可看作是一个 **"允许使用某 `UGameplayAbility` 的许可证"**

##### 类关系:
- Refers to the `UGA`'s CDO by default
- :pushpin: Can hold refers to **1-N `UGA` instances** (if use instanced GA)
- 配合句柄: `FGameplayAbilitySpecHandle` 
##### 主要属性:

`FGameplayAbilitySpecHandle Handle` // 作为唯一ID

`TObjectPtr<UGameplayAbility> Ability` // 指向 **`UGA`'s CDO**

`uint8 ActiveCount` // UPROPERTY(NotReplicated)
- 当前激活的 `UGA` instance 数量

`TArray<TObjectPtr<UGameplayAbility>> NonReplicatedInstances` // UPROPERTY(NotReplicated)
`TArray<TObjectPtr<UGameplayAbility>> ReplicatedInstances` 
- **Instanced GA 专属,** 对于 *InstancedPerActor* GA, 队列数量为1
- Any single `UGA instance` will reside in **either of them**, 取决于 `UGA::ReplicationPolicy `


`TMap<FGameplayTag, float> SetByCallerTagMagnitudes`
- todo...

```cpp
int Level; // 可指定等级
int InputID; // 可绑定输入

UPROPERTY(NotReplicated)
FGameplayAbilityActivationInfo	ActivationInfo; //详见 FGameplayAbilityActivationInfo 部分
```

<br>

#### :mag: Created Only on Authority 
> :books: 对于 ds 模式, 只有 Server 能够**创建** `FGameplayAbilitySpec` 和 `FGameplayAbilitySpecHandle`, 并将其**授予** some ASC, Clients only get replicated, 即便是 *LocalOnly* 型GA

---

### FGameplayAbilityActorInfo

> :books: 一个轻量化的 **ASC Context Cache** 结构体, 可看作是一个 ASC 的 "化身"
- Represents a shared, authoritative context about **ASC / OwnerActor / AvatarActor**  

  - :warning: 并不涵盖 Target 相关的信息
  - **鼓励继承**, 以加入Custom Component

- ASC 在 `OnRegister()` 时默认被分配一份, 会在 `InitAbilityActorInfo()` 时对其初始化
  ```cpp
  void UAbilitySystemComponent::InitAbilityActorInfo(AActor* InOwnerActor, AActor* InAvatarActor) {
    check(AbilityActorInfo.IsValid());
    bool WasAbilityActorNull = (AbilityActorInfo->AvatarActor == nullptr);
    bool AvatarChanged = (InAvatarActor != AbilityActorInfo->AvatarActor);
    this.AbilityActorInfo->InitFromActor(InOwnerActor, InAvatarActor, this);
    ...
  }
  ```
- :pushpin: 此类是联结 GA 和 ASC 关系的中介, 当 Give GA 时, [ASC 会分享 Ability Actor Info 给 GA](#ugameplayability--uobject) 
  ```cpp
  UAbilitySystemComponent* UGameplayAbility::GetAbilitySystemComponentFromActorInfo() const {
      if (!ensure(CurrentActorInfo)) return nullptr;
      return CurrentActorInfo->AbilitySystemComponent.Get();
  }
  ```
成员属性为 a set of **TWeakObjectPtr**
  - :warning: `PlayerController` 的赋值与否, 会影响 Activate GA 等相关逻辑的判断
```cpp
// shouldn't be null 
TWeakObjectPtr<AActor> OwnerActor;
TWeakObjectPtr<AActor> AvatarActor;
TWeakObjectPtr<UAbilitySystemComponent>	AbilitySystemComponent; 

// often be null
TWeakObjectPtr<APlayerController> PlayerController; 
TWeakObjectPtr<USkeletalMeshComponent>	SkeletalMeshComponent;
TWeakObjectPtr<UAnimInstance>	AnimInstance;
TWeakObjectPtr<UMovementComponent>	MovementComponent;
```
---

### FGameplayAbilityActivationInfo

承载一次 GA 激活的相关信息, 轻量化

- 每次 Activate GA 时会被生成一份, 并赋值给 `FGameplayAbilitySpec`和`UGA`

##### 主要属性 : 

`mutable TEnumAsByte<EGameplayAbilityActivationMode::Type>	ActivationMode`
- 代表本次激活的"模式", 例如在本地预测执行GA时, 会设置为 `Predicting` 模式

##### `FPredictionKey PredictionKeyWhenActivated` :pushpin:
- 当预测 GA 执行时, 会**单独 Cache**激活时的 Pkey 值, 因此后续 AbilityTasks 可通过
`Ability->GetCurrentActivationInfo().GetActivationPredictionKey()` 确认自身是在哪次激活下进行的

`uint8 bCanBeEndedByOtherInstance:1`
- 用于 runtime 标记是否能被远程取消或结束


##### 方法示例 : 
```cpp
// 切换为预测模式
void FGameplayAbilityActivationInfo::SetPredicting(FPredictionKey PredictionKey) {
  ActivationMode = EGameplayAbilityActivationMode::Predicting;
  PredictionKeyWhenActivated = PredictionKey;
  bCanBeEndedByOtherInstance = true;
}
```

