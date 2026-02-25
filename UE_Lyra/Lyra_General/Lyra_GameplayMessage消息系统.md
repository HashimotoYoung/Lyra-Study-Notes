# Lyra Gameplay Message

> :books: 基于 Message Bus Pattern 的游戏内消息系统


- **基于 TAG:** 使用 `FGameplayTag` 作为 "Message Channel"
  - 支持 TAG 层级特性, 允许 Listener 监听例如 `Message.UI` 并接收所有 `Message.UI.XXX` 消息
- 通过使用 C++ Template, 支持发送 **Specific-Type** Payload without knowing what type it is
- **类型安全检查:** 当广播消息时, `GameplayMessageSubsystem` 会**比较 Senders Payload Type 与 Listeners Registered Payload Type**. 如果类型不匹配，会自动跳过
- **完全同步:** 当调用 `BroadcastMessage()` 时，System 会立即遍历 `ListenerMap` 并依次回调


---
### UGameplayMessageSubsystem : UGameInstanceSubsystem

Manager 类, 由插件提供

##### 主要属性:
`TMap<FGameplayTag, FChannelListenerList> ListenerMap`
- 基于 "Channel" 记录所有 Listeners

##### 主要方法:
#### `RegisterListener()` (General)
- :star: **Thunk with `void*`:** 将 `TFunction<void(FGameplayTag, const FMessageStructType&)>` 类型回调, **封装为通用类型回调:** `TFunction<void(FGameplayTag, const UScriptStruct*, const void*)`, 以便于缓存 
  - 回调时会执行 TypeCast
- dereference the `FMessageStructType*` and pass to `const FMessageStructType&`
```cpp
template <typename FMessageStructType>
FGameplayMessageListenerHandle RegisterListener(FGameplayTag Channel, TFunction<void(FGameplayTag, const FMessageStructType&)>&& Callback, EGameplayMessageMatch MatchType = EGameplayMessageMatch::ExactMatch)
{
    auto ThunkCallback = [InnerCallback = MoveTemp(Callback)](FGameplayTag ActualTag, const UScriptStruct* SenderStructType, const void* SenderPayload)
    {
        InnerCallback(ActualTag, *reinterpret_cast<const FMessageStructType*>(SenderPayload));
    };

    const UScriptStruct* StructType = TBaseStructure<FMessageStructType>::Get();
    return RegisterListenerInternal(Channel, ThunkCallback, StructType, MatchType);
}
```
<br>

#### `RegisterListener()` (Bind Object)

- `void(TOwner::* Function)(FGameplayTag, const FMessageStructType&)` 是 **member function pointer type**
  - non-member 型写法为  `bool(*FuncPtr)(bool, int)`
- 调用相关**语法:** `(ObjPtr->*Function)(x, y)` 或 `(Obj.*Function)(x, y)`
```cpp
template <typename FMessageStructType, typename TOwner = UObject>
FGameplayMessageListenerHandle RegisterListener(FGameplayTag Channel, TOwner* Object, void(TOwner::* Function)(FGameplayTag, const FMessageStructType&))
{
    TWeakObjectPtr<TOwner> WeakObject(Object);
    return RegisterListener<FMessageStructType>(Channel,
        [WeakObject, Function](FGameplayTag Channel, const FMessageStructType& Payload)
        {
            if (TOwner* StrongObject = WeakObject.Get())
            {
                (StrongObject->*Function)(Channel, Payload);
            }
        });
}
```
<br>

#### `RegisterListenerInternal()`

Listener 注册内部实现方法

- 将 Callback 封装成具体的 struct 实例, 
  ```cpp
  USTRUCT()
  struct FGameplayMessageListenerData
  {
    TFunction<void(FGameplayTag, const UScriptStruct*, const void*)> ReceivedCallback;
    int32 HandleID;
    EGameplayMessageMatch MatchType;
    TWeakObjectPtr<const UScriptStruct> ListenerStructType = nullptr;
    bool bHadValidType = false;
  };
  ```

- :pushpin: 使用并分配 `List.HandleID` 确保 Handle 唯一性
```cpp
FGameplayMessageListenerHandle UGameplayMessageSubsystem::RegisterListenerInternal(FGameplayTag Channel, TFunction<void(FGameplayTag, const UScriptStruct*, const void*)>&& Callback, const UScriptStruct* StructType, EGameplayMessageMatch MatchType)
{
	FChannelListenerList& List = ListenerMap.FindOrAdd(Channel);
	FGameplayMessageListenerData& Entry = List.Listeners.AddDefaulted_GetRef();
	Entry.ReceivedCallback = MoveTemp(Callback);
	Entry.ListenerStructType = StructType;
	Entry.bHadValidType = StructType != nullptr;
	Entry.HandleID = ++List.HandleID;
	Entry.MatchType = MatchType;
	return FGameplayMessageListenerHandle(this, Channel, Entry.HandleID);
}
```

--- 

### Gameplay Message Processor

Lyra 自定义的 **World层** Processor, 负责监听并处理具体Channel

- `UAssistProcessor/UElimProcessor/...` << `UGameplayMessageProcessor` << `UActorComponent`
  - 通过 GFA 添加到 **`LyraGameState`**

- *BeginPlay* 时监听指定 Channel(`FGameplayTag`) 并记录 Handle, *EndPlay* 时通过 Handle 取消监听
- Listener **指定具体的 Payload Type** (e.g. `FLyraVerbMessage`)

```cpp
// in AssistProcessor.cpp

void UAssistProcessor::StartListening() {
	UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);
	AddListenerHandle(MessageSubsystem.RegisterListener(TAG_Lyra_Elimination_Message, this, &ThisClass::OnEliminationMessage));
	AddListenerHandle(MessageSubsystem.RegisterListener(TAG_Lyra_Damage_Message, this, &ThisClass::OnDamageMessage));
}

// Handlers
void UAssistProcessor::OnDamageMessage(FGameplayTag Channel, const FLyraVerbMessage& Payload){
	...
}
```
---

#### FGameplayMessageListenerHandle :star:
Listener 句柄
- 在创建的同时赋值 `TWeakObjectPtr<UGameplayMessageSubsystem>`,
  Listener Callers 可因此直接缓存 Handle 并使用 `Handle.Unregister()`, 写法简洁

```cpp
USTRUCT(BlueprintType)
struct GAMEPLAYMESSAGERUNTIME_API FGameplayMessageListenerHandle
{
public:
	FGameplayMessageListenerHandle() {}
	void Unregister();
private:

	UPROPERTY(Transient)
	TWeakObjectPtr<UGameplayMessageSubsystem> Subsystem;
	UPROPERTY(Transient)
	FGameplayTag Channel;
	UPROPERTY(Transient)
	int32 ID = 0;

	FDelegateHandle StateClearedHandle;

	friend UGameplayMessageSubsystem;

	FGameplayMessageListenerHandle(UGameplayMessageSubsystem* InSubsystem, FGameplayTag InChannel, int32 InID) : Subsystem(InSubsystem), Channel(InChannel), ID(InID) {}
};
```