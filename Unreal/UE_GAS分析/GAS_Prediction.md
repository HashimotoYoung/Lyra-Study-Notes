todo: FReplicatedPredictionKeyItem::OnRep(), FPredictionKey::NetSerialize

# GAS 预测机制

### FPredictionKey

预测键, 其核心作用为: 关联 **"逻辑流程"**

主要由两个 `int16` 构成:
- `Current`: 唯一ID
- `Base`: "root" FPredictionKey's ID (`Current`)


---
### FScopedPredictionWindow

- :pushpin: 该 Struct 的核心在于 "两构一析"
- 专门服务于 `ASC.ScopedPredictionKey`

##### 主要属性:

```cpp
FPredictionKey	ScopedPredictionKey // 没实际用到 ?
TWeakObjectPtr<UAbilitySystemComponent> Owner
bool ClearScopedPredictionKey 
bool SetReplicatedPredictionKey
FPredictionKey RestoreKey // 缓存值
```

##### 主要方法:

#### `FScopedPredictionWindow(UAbilitySystemComponent* AbilitySystemComponent, bool CanGenerateNewKey=true)`

:memo: Used when clients try activate *LocalPredicted* GA 

构造函数1, 只在 NetSimulating 端生效:

  1. 设置 `ClearScopedPredictionKey = true`
  2. 缓存 ASC 当前 Pkey:
     - `RestoreKey = OwnerASC->ScopedPredictionKey` 
  3. 驱动 ASC 生成新 Pkey:
     - `OwnerASC->ScopedPredictionKey.GenerateDependentPredictionKey()` 
<br>

#### `FScopedPredictionWindow(UAbilitySystemComponent* AbilitySystemComponent, FPredictionKey InPredictionKey, bool InSetReplicatedPredictionKey = true)`

构造函数2, 只在 Authority 端生效:

  1. 设置 `ClearScopedPredictionKey = true`
  2. 缓存 ASC 当前 Pkey:
     - `RestoreKey = OwnerASC->ScopedPredictionKey` 
  3. 赋值由客户端上传的 Pkey 给 ASC:
     - `OwnerASC->ScopedPredictionKey = InPredictionKey` 
  4. 标记析构时 replicate
     - `SetReplicatedPredictionKey = InSetReplicatedPredictionKey`
<br>

#### `~FScopedPredictionWindow()`
- **if** `SetReplicatedPredictionKey == true` 
  -  添加 `OwnerASC->ScopedPredictionKey` 到 `ReplicatedPredictionKeyMap` 中, 等待同步给上传者

- 还原 `RestoreKey` 给 ASC
  - ASC 没有激活任何 GA 的状况下, 其 `ScopedPredictionKey` 应为 ``{Base:0,Current:0}``
---


### FPredictionKeyDelegates
- 单例类, 主要负责处理 "预测执行" 失败后的 revert
- 底层由 DelegateMap <`Key.Current`, callback> 实现
```cpp
// 注册接口
FPredictionKeyEvent& FPredictionKeyDelegates::NewRejectedDelegate(FPredictionKey::KeyType Key)
{
	TArray<FPredictionKeyEvent>& DelegateList = Get().DelegateMap.FindOrAdd(Key).RejectedDelegates;
	DelegateList.Add(FPredictionKeyEvent());
	return DelegateList.Top();
}

// 广播接口
void FPredictionKeyDelegates::BroadcastRejectedDelegate(FPredictionKey::KeyType Key) {
	// Intentionally making a copy of the delegate list since it may change when firing one of the delegates
	static TArray<FPredictionKeyEvent> DelegateList;
	DelegateList.Reset();
	DelegateList = Get().DelegateMap.FindOrAdd(Key).RejectedDelegates;
	for (auto& Delegate : DelegateList){
		Delegate.ExecuteIfBound();
	}
}
```

---

### Local Predict 要点梳理

本地预测总体可分为三步 Client ASK >> Server Check & Do >> Client Reconciliation


#### 1. Client ASK:

##### 区别处理 *first-class Predictive Action* 和 *Side Effects* 
- 前者基本指代 Activate GA 
  - 特点在于 Client 会主动询问 Server **进行确认并答复**
- 后者主要指代本地预测后所产生的 "假设性结果"
  - 例如 Appl GE, 播放 Anim Montage, 播放 GCue 
  - 不会主动询问

##### 将 `FPredictionKey` 与 *逻辑流程* 进行关联 


- Client 在开始 "预测执行" 时, 会利用 `FScopedPredictionWindow` 刷新 `ASC.ScopedPredictionKey` (获取新的 Current 值), 并 "分配" 一个 Predict Scope
  ```cpp
  {
    FScopedPredictionWindow ScopedPredictionWindow(this, true);
	// # Predict Scope
  }
  ```
- :pushpin: 在 # Predict Scope 所涉及的整个 CallStack 里, 所有 Side Effects 可获取到 `ASC.ScopedPredictionkey` 从而**建立起逻辑关联** 
  - 可通过注册 `ASC.ScopedPredictionkey` +  Callback 的方式实现 **REVERT** 
    ```cpp
    // 示例
    FPredictionKey PredictionKey = ASC->GetPredictionKeyForNewAction();
    if (PredictionKey.IsValidKey()) {
      PredictionKey.NewRejectedDelegate().BindUObject(this, &UAbilitySystemComponent::OnPredictiveMontageRejected, NewAnimMontage);
    }
    ```

#### 2. Server Check & Do:

无论 Server 确认激活成功或失败, 都会告知 Client (associated with the `FPredictionKey` )
  - [client-confirm-ability-succeed-流](./GAS_GA/GA_MainFlow.md#client-confirm-ability-succeed-流)
  - [client-confirm-ability-fail-流](./GAS_GA/GA_MainFlow.md#client-confirm-ability-fail-流)

#### 3. Client Reconciliation
- 在以本地预测模式执行完相关逻辑后, Client 会等待 Server 同步 `ASC::ReplicatedPredictionKey` 
  - 同步此属性时, Client 会确认并接受同步效果, 并且清理(retire)本地预测实例