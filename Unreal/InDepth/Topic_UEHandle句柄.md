# UE Handle

> :books: 总结 UE 中各种类型的句柄设计


### "共享指针"型 Handle

- Based on `TSharedPtr<>`
- 用于确保 "Content" 在被使用期间 Stay Alive
- 轻量化数据传输
- 用于实现网络传输中的 UStruct 多态化, 例如 `FGameplayAbilityTargetDataHandle`
```cpp
struct FGameplayEffectSpecHandle {
	FGameplayEffectSpecHandle();
	FGameplayEffectSpecHandle(FGameplayEffectSpec* DataPtr);

	TSharedPtr<FGameplayEffectSpec>	Data;
}
```
##### 相关示例:
- `FGameplayEffectSpecHandle`:  只能 Used locally (禁止实现 `NetSerialize()`)
- `FGameplayEffectContextHandle`
- `FGameplayAbilityTargetDataHandle`

---

### "UniqueID"型 Handle
- Based on `int`
- Usually contained by **所指对象类型** , 作为 ID 使用
```cpp
struct FGameplayAbilitySpec : public FFastArraySerializerItem {
	FGameplayAbilitySpec() : Ability(nullptr), Level(1), InputID(INDEX_NONE), SourceObject(nullptr), ActiveCount(0), InputPressed(false), RemoveAfterActivation(false), PendingRemove(false), bActivateOnce(false) { }

	/** Handle for outside sources to refer to this spec by */
	UPROPERTY()
	FGameplayAbilitySpecHandle Handle;
	...
}
```
##### 相关示例:
- `FGameplayAbilitySpecHandle`
- `FActiveGameplayEffectHandle`

---

### "功能ID" 型 Handle

> :brain: Obj 相关的数据不由 Obj 本身持有, 而是由其它 Mananger 持有并管理, Obj 只持有 Key 用于获取

以 `FTimerHandle` 为例
- Based on `uint64 Handle`
  - 默认为 0 
- 当 Actor *BeginPlay* 时, 会调用 `this.SetLifeSpan(InitialLifeSpan)`
  - 交由 TimerManager 负责初始化 `this.TimerHandle_LifeSpanExpired` 并**注册回调**
  - :pushpin: TimerMananger 会为每个 `FTimerHandle` 创建对应的 `FTimerData` 并维护管理
  - :pushpin: Handle 不仅作为唯一ID, 还持有一定的数据信息
- 当 Actor *EndPlay* 时, **if** `Actor.TimerHandle_LifeSpanExpired` is Valid (Handle != 0), 通知 TimerManager 进行清除

```cpp
void AActor::SetLifeSpan( float InLifespan ) {
	InitialLifeSpan = InLifespan;
	// Initialize a timer for the actors lifespan if there is one. Otherwise clear any existing timer
	if ((GetLocalRole() == ROLE_Authority || GetTearOff()) && IsValidChecked(this) && GetWorld()) {
		if( InLifespan > 0.0f) {
			GetWorldTimerManager().SetTimer( TimerHandle_LifeSpanExpired, this, &AActor::LifeSpanExpired, InLifespan );
		}
		else {
			GetWorldTimerManager().ClearTimer( TimerHandle_LifeSpanExpired );		
		}
	}
}
```