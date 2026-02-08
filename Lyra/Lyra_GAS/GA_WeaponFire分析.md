

### ULyraGameplayAbility_RangedWeapon

远程射击 GA 

- 继承自 `ULyraGameplayAbility_FromEquipment` <= `ULyraGameplayAbility` 

##### 主要属性:

`FDelegateHandle OnTargetDataReadyCallbackDelegateHandle;`
- 回调句柄, 用于EndAbility时解除Bind
```cpp
// in ULyraGameplayAbility; 使用了 **Instanced** 标记
UPROPERTY(EditDefaultsOnly, Instanced, Category = Costs)
TArray<TObjectPtr<ULyraAbilityCost>> AdditionalCosts;

// in ULyraGameplayAbility; 每个Ability绑定一种相机类型?
TSubclassOf<ULyraCameraMode> ActiveCameraMode;
```
<br>

##### 主要方法:

### `StartRangedWeaponTargeting()`

启动本地瞄准射击流程; ClientOnly

一个 `FGameplayAbilityTargetDataHandle` 对应一个持有相同ID的 `FLyraServerSideHitMarkerBatch`
在一次射击流程中, FoundHits 的**数量**和 `FLyraGameplayAbilityTargetData_SingleTargetHit`的**数量**和 HitMarkers的**数量**一致

- 获取武器状态组件: `let wsComp = ULyraWeaponStateComponent()`
- :warning: 创建**Server预测窗**: 	`FScopedPredictionWindow ScopedPrediction(MyAbilityComponent, CurrentActivationInfo.GetActivationPredictionKey());`
  // 本方法没有在Server调用, 这里没有实际意义?
- 执行[本地瞄准判断](#performlocaltargetingout-tarrayfhitresult-outhits), 计算出**本地击中结果 FoundHits** (`TArray<FHitResult>`)
- :star: 根据 FoundHits, 填充 **`FGameplayAbilityTargetDataHandle`** 数据 
  - **巧妙设置UniqueID:** 以待确认批次Count作为 TargetData.UniqueId
  - **避免内存泄漏:** 在Heap上New出TargetHit对象后, 立刻用 Handle来Hold住

   ```cpp
  //Handle 持有 TArray<TSharedPtr<FGameplayAbilityTargetData>> Data
   FGameplayAbilityTargetDataHandle TargetData;
   
   // 当前未确认批次号作为id
   TargetData.UniqueId = WeaponStateComponent ? WeaponStateComponent->GetUnconfirmedServerSideHitMarkerCount() : 0;

   if (FoundHits.Num() > 0)
   {
      const int32 CartridgeID = FMath::Rand();
      for (const FHitResult& FoundHit : FoundHits)
      {
         FLyraGameplayAbilityTargetData_SingleTargetHit* NewTargetData = new FLyraGameplayAbilityTargetData_SingleTargetHit();
         NewTargetData->HitResult = FoundHit;
         NewTargetData->CartridgeID = CartridgeID;
         TargetData.Add(NewTargetData);
      }
   }
   ```
- **if** 当前武器是非Projectile类型: CALL `WeaponStateComponent->AddUnconfirmedServerSideHitMarkers(InTargetData, FoundHits)` 
  :memo: **方法简介:**  
  - 基于 TargetData.UniqueId, 新建一个 **"待确认HitMarker批次"**
    //`FLyraServerSideHitMarkerBatch& NewUnconfirmedHitMarker = UnconfirmedServerSideHitMarkers.Emplace_GetRef(InTargetData.UniqueId);`
  - :star: 根据 FoundHits, 填充 HitMarkers 数据
  - 添加到`WeaponStateComponent::UnconfirmedServerSideHitMarkers`队列中, 等待服务端回调进行确认

- **开始进行本地预测:** [OnTargetDataReadyCallback()](#ontargetdatareadycallbackconst-fgameplayabilitytargetdatahandle-indata-fgameplaytag-applicationtag)

---

### `PerformLocalTargeting(OUT TArray<FHitResult>& OutHits)`
本地瞄准判断, 输出FHitResult

---

### `OnTargetDataReadyCallback(const FGameplayAbilityTargetDataHandle& InData, FGameplayTag ApplicationTag)`
接收射击目标回调; BothSide
- **if** 成功获取 AbilitySpec <= `CurrentSpecHandle`
  - 新建 Client 预测窗, 生成 预测Key  **P1**
  - :star: **MoveTemp** `InData` 到 Local 的原因:
    1. 接管Ownership, 下方代码执行时, 数据有效
    2. 防止SharedPtr引用计数增加
    ```cpp
    FGameplayAbilityTargetDataHandle LocalTargetDataHandle(MoveTemp(const_cast<FGameplayAbilityTargetDataHandle&>(InData)));
    ```
  - **if** Is Local Client 
    - :star: CALL **ServerRPC**  `ServerSetReplicatedTargetData(...)`
      // 向Server发送 LocalTargetDataHandle 和 预测Key **P1** 等数据
      // Server端接收后会调用`OnTargetDataReadyCallback()`
      <br>
  
  *#if WITH_SERVER_CODE 服务端专属* 
  - **if** 获取到武器状态组件 `ULyraWeaponStateComponent*` <= Controller <= `CurrentActorInfo`
    - Server 遍历 `LocalTargetDataHandle`, 尝试 **Replace Some HitResult**
// Lyra中没有实际执行 Replace 的逻辑
    - CALL **ClientRPC** `ULyraWeaponStateComponent->ClientConfirmTargetData(uint16 UniqueId, bool bSuccess, const TArray<uint8>& HitReplaces)`
    :memo: **方法简介:**
      - 客户端找到 未确认HitMarker批次 `FLyraServerSideHitMarkerBatch& Batch` with **Same UniqueID**
      - 根据 HitReplaces 排除 HitMarker数据, 只处理成功的HitMarker数据
        // HitReplaces中int值 i 代表: 该 `FGameplayAbilityTargetDataHandle` 中的 Data[i] 需要在客户端进行纠正/排除
      - 从 `UnconfirmedServerSideHitMarkers` 中移除

    ```cpp
    TArray<uint8> HitReplaces;
    for (uint8 i = 0; (i < LocalTargetDataHandle.Num()) && (i < 255); ++i)
    {
        if (FGameplayAbilityTargetData_SingleTargetHit* SingleTargetHit = static_cast<FGameplayAbilityTargetData_SingleTargetHit*>(LocalTargetDataHandle.Get(i)))
        {
          if (SingleTargetHit->bHitReplaced) HitReplaces.Add(i);
        }
    }
    WeaponStateComponent->ClientConfirmTargetData(LocalTargetDataHandle.UniqueId, bIsTargetDataValid, HitReplaces);
    ```

  *#endif*
  <br>
  
   - **if** 成功释放GA: `CommitAbility(...) == true` // Client和Server 都会生效
     - 添加武器散射: `GetWeaponInstance()->AddSpread()`
     - 执行[目标击中流](../Lyra_Weapon/Weapon核心流程.md#onrangedweapontargetdatareadyconst-fgameplayabilitytargetdatahandle-targetdata)
   - **else** `K2_EndAbility()`
- Server 调用 `ConsumeClientReplicatedTargetData` 来通知 ASC 清理掉当前 PredictionKey 下缓存的 TargetData，完成本次网络同步和预测的生命周期

