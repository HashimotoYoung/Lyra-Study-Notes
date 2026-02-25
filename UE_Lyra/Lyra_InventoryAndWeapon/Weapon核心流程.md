## Weapon 核心流程

### 开火射击流:

**玩家输入 (Client):**
- Override 控制层输入后处理方法 (`ALyraPlayerController::PostProcessInput`)
- LyraASC 负责处理输入信息: [ProcessAbilityInput()](../Lyra_GAS/Basics_GAS.md#processabilityinputfloat-deltatime-bool-bgamepaused)  
  // 会调用 `TryActivateAbility()`

**Activate GA_RangedWeapon (Client):** 

1. 调用 `ULyraGameplayAbility_RangedWeapon::ActivateAbility()`
   - :star: **重点分析1:** 主要在**Server端**起作用, ASC **找到或新增** 一个以 *当前AbilitySpec* 和 *当前ActivationPredictionKey* 为 KEY 的Delegate 进行绑定, 以便Server在收到 **RPC** `SetReplicatedTargetData()` 后, 能够执行正确的回调
      ```cpp
      //重点分析1
      this.OnTargetDataReadyCallbackDelegateHandle = MyASC->AbilityTargetDataSetDelegate(CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey()).AddUObject(this, &ThisClass::OnTargetDataReadyCallback);

      // 获取武器实例, 记录开火时间
      ULyraRangedWeaponInstance* WeaponData = GetWeaponInstance();
      WeaponData->UpdateFiringTime();

      // 触发Event
      Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
     ```

2. 触发 BP_Event ActivateAbility:
   - if GA is Locally Controlled
     - **开始 Client 预测射击流:** [StartRangedWeaponTargeting()](../Lyra_GAS/GA_WeaponFire分析.md#startrangedweapontargeting)  
       - 流程内会触发 BP_Event [击中目标](#onrangedweapontargetdatareadyconst-fgameplayabilitytargetdatahandle-targetdata)
   - 播放 开火Montage: `AbilityTask_PlayMontageAndWait()`
     - 经过指定时间后(`FireDelayTimeSecs`), **EndAbility**: `SetTimerByEvent()`
	 - Montage 完成或打断后 **EndAbility**

**Activate GA_RangedWeapon (Server):** 
- 调用 `ActivateAbility()`

---

### `OnRangedWeaponTargetDataReady(const FGameplayAbilityTargetDataHandle& TargetData)`

**BP_Event** in `GA_Weapon_Fire(BP)`, called on Server 和 开火者Client   
负责处理击中目标后的逻辑, 包括 Apply GE

- 获取 **GC_Param** << `FHitResult` << TargetData.Data[0]  
  // 这里只采用首个目标的击中结果
- 在 Owner.ASC 上 **ExecuteGC("Tag_开火")**
- foreach data in TargetData.Data
  - 获取 GC_Param << `FHitResult` << data
  - **if** 判断击中: 
    - 在 Owner.ASC上 **ExecuteGC("Tag_击中Impact")**
    - **if** HasAuthority: 可选择 **SpawnActor** 在Impact位置  
      // Lyra中无实际应用
- **if** HasAuthority, Apply `GE_Damage` to `FGameplayAbilityTargetDataHandle`