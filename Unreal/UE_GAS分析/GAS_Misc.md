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



