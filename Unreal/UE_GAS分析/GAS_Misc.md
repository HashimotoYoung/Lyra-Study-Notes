## Gameplay Event

The Gameplay Event System **is part of the GAS** — it is **not a general-purpose event system** for all actors in Unreal.

- Used primarily to **trigger activate abilities or gameplay logic**, so they are typically **reliable** 
- 通过 `someASC->HandleGameplayEvent()` 来发送事件 **to `someASC`**, 这也意味着该system并不是传统的Observer模式, 更像是一个GAS内部的默认消息接口
	- 以 EventTag `(就是GameplayTag)` 作为channel 
	- Sent with payload: `FGameplayEventData` 

示例:
```cpp
FGameplayEventData EventData;
EventData.EventTag = FGameplayTag::RequestGameplayTag("Event.Weapon.Fire");
EventData.Instigator = Instigator;
EventData.Target = TargetActor;
EventData.OptionalObject = SomeData;

someASC->HandleGameplayEvent(EventData.EventTag, &EventData);
```

---
### GAS Handle(句柄) 分类

#### 1. "共享指针"型 Handle

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

<br>

#### 2. "UniqueID"型 Handle
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

