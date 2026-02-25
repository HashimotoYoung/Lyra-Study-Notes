# GAS ASC 

---

### UAbilitySystemComponent 

ASC 基本可看作是一个 **Manager** of GAS Elements
- 继承自 UGameplayTasksComponent
  - ASC 负责Tick `UAbilityTask`
- 封装GAS核心APIs给其Owner使用
- 负责 Replicate GA,GE, AttributeSet
  - `Attributes` are replicated internally by their owning `AttributeSet`
<br>

##### 主要属性:
`TArray<TObjectPtr<UAttributeSet>> SpawnedAttributes`
- UPROPERTY(Replicated, ReplicatedUsing=OnRep_SpawnedAttributes)
- **AttributeSet 相关**, 需注意 `Attributes` are **replicated by their AS**

`FActiveGameplayEffectsContainer ActiveGameplayEffects`
- UPROPERTY(Replicated)
- **GameplayEffect 相关**

`FActiveGameplayCueContainer ActiveGameplayCues`  
`FActiveGameplayCueContainer MinimalReplicationGameplayCues`
- UPROPERTY(Replicated),   
  - `MinimalReplicationGameplayCues` 的同步条件为 `COND_SkipOwner`
- **GameplayCue 相关**

`FGameplayTagCountContainer GameplayTagCountContainer`
- 记录 **Owned** GameplayTags

`TObjectPtr<AActor> OwnerActor`  
`TObjectPtr<AActor> AvatarActor`

- 皆为 UPROPERTY(ReplicatedUsing = OnRep_OwningActor)
- Owner 和 Avatar 一般**需要 implement  `IAbilitySystemInterface`**
- 通常在 Owner's **Constructor** 中创建 ASC
  ```cpp
  // in ALyraPlayerState.cpp 
  AbilitySystemComponent = ObjectInitializer.CreateDefaultSubobject<ULyraAbilitySystemComponent>(this, TEXT("AbilitySystemComponent"));
  AbilitySystemComponent->SetIsReplicated(true);
  AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
  ```

`EGameplayEffectReplicationMode ReplicationMode`
- 影响 **Both GE and GC** 的同步方式

`FGameplayAbilityReplicatedDataContainer AbilityTargetDataMap`
- todo

`TSharedPtr<FGameplayAbilityActorInfo>	AbilityActorInfo` [Link](./GAS_GA/GA_Classes.md#fgameplayabilityactorinfo)

`FPredictionKey	ScopedPredictionKey` [Link](./GAS_ReplicationAndPrediction.md#prediction-原理)

<br>

##### 主要属性(GA):

`FGameplayAbilitySpecContainer ActivatableAbilities`
- UPROPERTY(ReplicatedUsing=OnRep_ActivateAbilities)
  - Simulated Proxy 不同步
  ```cpp
  Params.Condition = COND_ReplayOrOwner;
  DOREPLIFETIME_WITH_PARAMS_FAST(UAbilitySystemComponent, ActivatableAbilities, Params);
  ```
- :pushpin: 该 ASC 持有的 **GA Specs**, 反映了哪些 GA 可用

`int32 AbilityScopeLockCount`  
`TArray<FGameplayAbilitySpecHandle, TInlineAllocator<2>> AbilityPendingRemoves`  
`TArray<FGameplayAbilitySpec, TInlineAllocator<2> > AbilityPendingAdds`

- 这三个属性用于 [Ability Scoped Lock 机制](#ability-scoped-lock-机制)

##### 主要属性(预测):

`FReplicatedPredictionKeyMap ReplicatedPredictionKeyMap`
  - UPROPERTY(Replicated)

---

### :warning: ASC 注意事项:
- `InitAbilityActorInfo()` relies on **Fully-Replicated** `PlayerController`
  因此通常可以在 PC 中调用以下方法, 以确保初始化完善, 例如:
    ```cpp
    void ALyraPlayerController::OnRep_PlayerState() {
            Super::OnRep_PlayerState();
            BroadcastOnPlayerStateChanged();

            if (GetWorld()->IsNetMode(NM_Client))  {
                if (ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>()){
                    if (ULyraAbilitySystemComponent* LyraASC = LyraPS->GetLyraAbilitySystemComponent()) {
                        // Calls InitAbilityActorInfo
                        LyraASC->RefreshAbilityActorInfo();
                        LyraASC->TryActivateAbilitiesOnSpawn();
                    }
                }
            }
        }
    ```



