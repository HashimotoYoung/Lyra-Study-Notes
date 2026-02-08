## Weapon 核心流程

### 1. 开火射击流:

**玩家输入 (Client):**
- Override 控制层输入后处理方法 (`AEchoPlayerController::PostProcessInput`)
- LyraASC负责处理输入信息: [ProcessAbilityInput()](../Lyra_GAS/Base_GAS.md#processabilityinputfloat-deltatime-bool-bgamepaused)
  - // 执行 TryActivateAbility()

**激活 GA_RangedWeapon (Client):** 

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
   - if GA受本地控制: `IsLocallyControlled()`
     - **开始Client预测射击流程:** [StartRangedWeaponTargeting()](../Lyra_GAS/GA_WeaponFire分析.md#startrangedweapontargeting)
    // 内部会触发 BP_Event `OnRangedWeaponTargetDataReady`
   - 播放 开火Montage: `AbilityTask_PlayMontageAndWait()`
     - 经过指定时间后(`FireDelayTimeSecs`), **EndAbility**: `SetTimerByEvent()`
	 - Montage 完成或打断后 **EndAbility**

**激活 GA_RangedWeapon (Server):** 
- 调用 `ActivateAbility()`

---

### `OnRangedWeaponTargetDataReady(const FGameplayAbilityTargetDataHandle& TargetData)`

BP_Event("GA_Weapon_Fire"), called on Server 和 开火者Client 
负责处理击中目标后的逻辑, 包括 Apply GE

- 获取 **GC_Param** <= `FHitResult` <= TargetData.Data[0]
  // 这里只采用首个目标的击中结果
- 在 Owner.ASC 上 **ExecuteGC("Tag_开火")**
- foreach data in TargetData.Data
  - 获取 GC_Param <= `FHitResult` <= data
  - **if** 判断击中: 
    - 在 Owner.ASC上 **ExecuteGC("Tag_击中Impact")**
    - **if** HasAuthority: 可选择 **SpawnActor** 在Impact位置
      // Lyra中无实际应用
- **if** HasAuthority: **ApplyGE** "GE_Damage" to `FGameplayAbilityTargetDataHandle`