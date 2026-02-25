# Lyra Tips 总结 

---

## :bulb: 通用
#### 1. 注册 `OnExperienceLoaded`回调 的写法
- 使用 Move 特性

```cpp
// LyraGameMode.cpp
void ALyraGameMode::InitGameState()
{
	Super::InitGameState();

	ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
	check(ExperienceComponent);
	//创建临时UObject用于Move
	ExperienceComponent->CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}
```
```cpp
// LyraExperienceManagerComponent.cpp
void ULyraExperienceManagerComponent::CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate&& Delegate)
{
	if (IsExperienceLoaded())
	{
		Delegate.Execute(CurrentExperience);
	}
	else
	{
		OnExperienceLoaded.Add(MoveTemp(Delegate));
	}
}
```

<br>

#### 2. Wrap 蓝图回调 to 原生回调

- 提供BP专用的`StartPhase()`接口, 补充 `FLyraGamePhaseDynamicDelegate`
- 使用 WeakLambda: 即使BP obj被销毁了, 回调也不会报错

```cpp
// ULyraGamePhaseSubsystem.cpp
void ULyraGamePhaseSubsystem::K2_StartPhase(TSubclassOf<ULyraGamePhaseAbility> PhaseAbility, const FLyraGamePhaseDynamicDelegate& PhaseEndedDelegate)
{
    //值得参考
	const FLyraGamePhaseDelegate EndedDelegate = FLyraGamePhaseDelegate::CreateWeakLambda(const_cast<UObject*>(PhaseEndedDelegate.GetUObject()), [PhaseEndedDelegate](const ULyraGamePhaseAbility* PhaseAbility) {
		PhaseEndedDelegate.ExecuteIfBound(PhaseAbility);
	});
	StartPhase(PhaseAbility, EndedDelegate);
}
```

<br>

#### 3. 使用 `TFunctionRef<...>` 作为参数Type
- 相比于delegate更轻量化
```cpp
// ULyraAbilitySystemComponent.h
...
typedef TFunctionRef<bool(const ULyraGameplayAbility* LyraAbility, FGameplayAbilitySpecHandle Handle)> TShouldCancelAbilityFunc;
void CancelAbilitiesByFunc(TShouldCancelAbilityFunc ShouldCancelFunc, bool bReplicateCancelAbility);
```

<br>

#### 4. 使用 `[this, WeakThis]` for Lambda 回调

- *[this]* 在这里是必须的,为了使用 member 变量和方法

```cpp
void UAsyncAction_PushContentToLayerForPlayer::Activate()
{
	if (UPrimaryGameLayout* RootLayout = UPrimaryGameLayout::GetPrimaryGameLayout(OwningPlayerPtr.Get()))
	{
		TWeakObjectPtr<UAsyncAction_PushContentToLayerForPlayer> WeakThis = this;
		
		StreamingHandle = RootLayout->PushWidgetToLayerStackAsync<UCommonActivatableWidget>(LayerName, 
		bSuspendInputUntilComplete, 
		WidgetClass, 
		[this, WeakThis](EAsyncWidgetLayerState State, UCommonActivatableWidget* Widget) 
		{
			if (WeakThis.IsValid())
			{
				switch (State)
				{
					case EAsyncWidgetLayerState::Initialize:
						BeforePush.Broadcast(Widget);
						break;
					case EAsyncWidgetLayerState::AfterPush:
						AfterPush.Broadcast(Widget);
						SetReadyToDestroy();
						break;
					case EAsyncWidgetLayerState::Canceled:
						SetReadyToDestroy();
						break;
				}
			}
			SetReadyToDestroy();
		});
	}
}
```
<br>

#### 5. In-Place 构造
- 例如当实现了 `FName` 构造函数时: 
  - :heavy_check_mark: `Some_Struct_Array.Emplace_GetRef( A_FName_Value )`

<br>

#### 6. UBlueprintAsyncActionBase 注意事项
- `UBlueprintAsyncActionBase`类本质上不属于任何 UWorld, 继承时可考虑 add 成员变量: `TWeakObjectPtr<UWorld> WorldPtr`

---
## :bulb: 设计模式 
#### 1. Struct-Wrapped 观察者模式

将 delegate 封装进 struct, 在 UE 中很常见

相对于传统 Multicast Delegate 的优点:

  - 广播时可精确 **Filter** Observers
  - 可携带 "Context" 数据
  - 方便动态管理, 例如删除特定 Observers

```cpp
// ULyraGamePhaseSubsystem.h
...
private:
struct FPhaseObserver
{
	bool IsMatch(const FGameplayTag& ComparePhaseTag) const;
	FGameplayTag PhaseTag;
	EPhaseTagMatchType MatchType = EPhaseTagMatchType::ExactMatch;
	FLyraGamePhaseTagDelegate PhaseCallback; // actual delegate
};

TArray<FPhaseObserver> PhaseStartObservers;
TArray<FPhaseObserver> PhaseEndObservers;
```

---
## :bulb: 资源加载

#### 避免链式硬加载

- `ULyraUserFacingExperienceDefinition` 使用了 **ID** 而非 References 来引用`ULyraExperienceDefinition`, 减少硬加载