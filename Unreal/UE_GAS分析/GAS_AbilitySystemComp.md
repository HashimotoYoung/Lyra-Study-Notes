# GAS ASC 

---

### UAbilitySystemComponent

ASC 基本可看作是一个 **Manager** of GAS Elements
- 继承自 `UGameplayTasksComponent`
  - ASC 负责 Tick `UAbilityTask`
- 负责 Replicate GA, GE, AttributeSet
  - `Attributes` are replicated internally by their owning `AttributeSet`
<br>

##### 主要属性:

`TSharedPtr<FGameplayAbilityActorInfo>	AbilityActorInfo`: [link](./GAS_GA/GA_Classes.md#fgameplayabilityactorinfo)

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

`TArray<TObjectPtr<UAttributeSet>> SpawnedAttributes`
- UPROPERTY(Replicated, ReplicatedUsing=OnRep_SpawnedAttributes)
- **AttributeSet 相关**

`FActiveGameplayEffectsContainer ActiveGameplayEffects`
- UPROPERTY(Replicated)
- **GameplayEffect 相关**

`FActiveGameplayCueContainer ActiveGameplayCues`  
`FActiveGameplayCueContainer MinimalReplicationGameplayCues`
- UPROPERTY(Replicated)
  - `MinimalReplicationGameplayCues` 的 Rep 条件为 `COND_SkipOwner`
- **GameplayCue 相关**

`FGameplayTagCountContainer GameplayTagCountContainer`
- 记录 **Owned** GameplayTags

`EGameplayEffectReplicationMode ReplicationMode`
- 影响 **Both GE and GC** 的同步方式

<br>

##### 主要属性(GA相关):

`FGameplayAbilitySpecContainer(:FFastArraySerializer) ActivatableAbilities` :star:

**GA 容器**

- **UPROPERTY**(ReplicatedUsing=OnRep_ActivateAbilities)
  - 使用 Rep 条件 `COND_ReplayOrOwner`, 因此不同步 Simulated Proxy 

- `ActivatableAbilities.Items` 实际为 **`TArray<FGameplayAbilitySpec>`**,   
  可进而获取到 GA **CDO**(`AbilitySpec.Ability`) 和 GA 实例 (`AbilitySpec.GetAbilityInstances()`)

`int32 AbilityScopeLockCount`  
`TArray<FGameplayAbilitySpecHandle, TInlineAllocator<2>> AbilityPendingRemoves`  
`TArray<FGameplayAbilitySpec, TInlineAllocator<2> > AbilityPendingAdds`

- 这三个属性用于 [Ability Scoped Lock 机制](./GAS_GA/GA_Basics.md#ability-scoped-lock)

`FGameplayAbilityReplicatedDataContainer AbilityTargetDataMap`

- 缓存一次GA激活期间涉及的 Targets 和 Events 等信息, [link](./GAS_TargetData.md)


##### 主要属性(Prediction相关):

`FReplicatedPredictionKeyMap ReplicatedPredictionKeyMap`  
`FPredictionKey	ScopedPredictionKey` 
- [GAS预测相关](./GAS_Prediction.md)

---

#### :warning: 注意事项:
- `InitAbilityActorInfo()` relies on **Fully-Replicated** `PlayerController`
  因此通常可以在 PC 中调用以下方法, 以确保初始化完整, 例如:
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



