# GAS Target 目标选取

---
### FGameplayAbilityTargetData

承载目标相关的数据信息, designed to be passed across the network 

- 纯方法Struct, **必须继承使用**
  - :pushpin: Substructs 需要实现 `NetSerialize()`, 并override `UScriptStruct* GetScriptStruct()`, 以保证正确解析
- About **`FGameplayAbilityTargetDataHandle`**: 
  - Contains `TArray<TSharedPtr<FGameplayAbilityTargetData>>`
  - 持有 UniqueID

##### 使用示例 : 
- Allocate => 初始化 => Add to Handle => 返回 handle
```cpp
FGameplayAbilityTargetDataHandle UAbilitySystemBlueprintLibrary::AbilityTargetDataFromLocations(const FGameplayAbilityTargetingLocationInfo& SourceLocation, const FGameplayAbilityTargetingLocationInfo& TargetLocation) {
	// Construct TargetData
	FGameplayAbilityTargetData_LocationInfo* NewData = new FGameplayAbilityTargetData_LocationInfo();
	NewData->SourceLocation = SourceLocation;
	NewData->TargetLocation = TargetLocation;

	// Give it a handle and return
	FGameplayAbilityTargetDataHandle	Handle;
	Handle.Data.Add(TSharedPtr<FGameplayAbilityTargetData_LocationInfo>(NewData));
	return Handle;
}
```
接管 Ownership 的写法
```cpp
// inside some function
FGameplayAbilityTargetDataHandle LocalTargetDataHandle(MoveTemp(const_cast<FGameplayAbilityTargetDataHandle&>(InData)));
```

---

### FGameplayAbilityReplicatedDataContainer
同步数据缓存容器, 缓存内容为: 一次GA激活期间涉及的Targets和Events等信息
- Contained in `ASC` as `AbilityTargetDataMap` 
  - 主要用于 Server 缓存GA激活过程中 Client 上发的目标相关数据
  - :warning: 服务于本地计算, No *Replicated* Mark
- 类 Array\<KeyValuePair> 结构:
  - Key:  `FGameplayAbilitySpecHandle + PredictionKey`
  - Value: `FAbilityReplicatedDataCache`


####  相关方法

> :memo: 以 Lyra 射击流程为例, Server 选择相信 Client 收集的Target
##### 1. 注册 Set Delegate 回调
- Server在激活射击GA时注册回调并缓存 handle, 等待 Client 发送目标
```cpp
// in ULyraGameplayAbility_RangedWeapon::ActivateAbility()
...
OnTargetDataReadyCallbackDelegateHandle = MyAbilityComponent->AbilityTargetDataSetDelegate(CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey()).AddUObject(this, &ThisClass::OnTargetDataReadyCallback);
```
##### 2. Client 收集 TargetData 后, 发送给 Server
- Client 本地计算击中目标, 封装成 `FGameplayAbilityTargetDataHandle` 后上传

```cpp
// Server RPC
void ASC::ServerSetReplicatedTargetData(FGameplayAbilitySpecHandle AbilityHandle, FPredictionKey AbilityOriginalPredictionKey, const FGameplayAbilityTargetDataHandle& ReplicatedTargetDataHandle, FGameplayTag ApplicationTag, FPredictionKey CurrentPredictionKey);
```
##### 3. Server 处理 TargetData
`ServerSetReplicatedTargetData_Implementation()`

1. 根据 Key 找到对应的 `FAbilityReplicatedDataCache`
2. 赋值各项属性:
    ```cpp
    ReplicatedData->TargetData = ReplicatedTargetDataHandle;
    ReplicatedData->ApplicationTag = ApplicationTag;
    ReplicatedData->bTargetConfirmed = true;
    ReplicatedData->bTargetCancelled = false;
    ReplicatedData->PredictionKey = CurrentPredictionKey;
    ```
3. :pushpin: **Broadcast** `ReplicatedData->TargetSetDelegate`
   - 触发 `GA::OnTargetDataReadyCallback()`

##### 4. 射击GA结束时, 清理 Set Delegate 回调
- Consume 方法负责重置 `FAbilityReplicatedDataCache` 中 TargetData 相关的数据
```cpp
// in ULyraGameplayAbility_RangedWeapon::EndAbility()
...
MyAbilityComponent->AbilityTargetDataSetDelegate(CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey()).Remove(OnTargetDataReadyCallbackDelegateHandle);
MyAbilityComponent->ConsumeClientReplicatedTargetData(CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey());
```
