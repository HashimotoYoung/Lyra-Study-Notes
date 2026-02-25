# Lyra Game Phase

>:books: Lyra 使用 GAS + SubSystem 来实现 "游戏阶段" 的设计

- `ALyraGameState` 本身持有 ASC 组件
- **每一个 "Phase" 被封装成一个 GA (`ULyraGamePhaseAbility`)**
  - Phase GAs only run on Server
- 多个 Phase 可同时存在: 见 `OnBeginPhase()` 实现

---

### ULyraGamePhaseSubsystem : UWorldSubsystem

负责游戏内阶段转换管理

- **Created per-world:** 当执行 `ULyraSystemStatics::PlayNextGame()` 时会调用 `ServerTravel()`, 从而会创建新的 `ULyraGamePhaseSubsystem` 实例
- 使用了 [Struct-Wrapped 观察者模式](../Lyra_Tips.md#1-struct-wrapped-观察者模式)
```cpp
class ULyraGamePhaseSubsystem : public UWorldSubsystem
{
...
private:
	struct FLyraGamePhaseEntry
	{
	public:
		FGameplayTag PhaseTag;
		FLyraGamePhaseDelegate PhaseEndedCallback;
	};

    //使用Handle为key
	TMap<FGameplayAbilitySpecHandle, FLyraGamePhaseEntry> ActivePhaseMap;

	struct FPhaseObserver
	{
	public:
		bool IsMatch(const FGameplayTag& ComparePhaseTag) const;
	
		FGameplayTag PhaseTag;
		EPhaseTagMatchType MatchType = EPhaseTagMatchType::ExactMatch;
		FLyraGamePhaseTagDelegate PhaseCallback;
	};

	TArray<FPhaseObserver> PhaseStartObservers;
	TArray<FPhaseObserver> PhaseEndObservers;

	friend class ULyraGamePhaseAbility;
};
```

---

### :pushpin: `OnBeginPhase()` 实现部分
- Called in  `PhaseGA::ActivateAbility()`
```cpp
void ULyraGamePhaseSubsystem::OnBeginPhase(const ULyraGamePhaseAbility* PhaseAbility, const FGameplayAbilitySpecHandle PhaseAbilityHandle)
{
	const FGameplayTag IncomingPhaseTag = PhaseAbility->GetGamePhaseTag();
  
	// 获取ASC
	ULyraAbilitySystemComponent* GameState_ASC = World->GetGameState()->FindComponentByClass<ULyraAbilitySystemComponent>();

	// ActivePhaseMap 记录 GA Handle, 方便获取
	TArray<FGameplayAbilitySpec*> ActivePhases;
	for (const auto& KVP : ActivePhaseMap)
	{
		const FGameplayAbilitySpecHandle ActiveAbilityHandle = KVP.Key;
		if (FGameplayAbilitySpec* Spec = GameState_ASC->FindAbilitySpecFromHandle(ActiveAbilityHandle))
		{
			ActivePhases.Add(Spec);
		}
	}

	// keypoint
	// 此处会触发 Old Phase -> CancelAbility(), 因此会先执行 Old Phase 的 End 回调
	// PhaseGA::EndAbility 时会 Remove handle from ActivePhaseMap
	// !!! 使用 MatchesTag 方法, 因此可兼容 Phase 和 Sub Phase 同时保持Active
	for (const FGameplayAbilitySpec* ActivePhase : ActivePhases)
	{
		const ULyraGamePhaseAbility* ActivePhaseAbility = CastChecked<ULyraGamePhaseAbility>(ActivePhase->Ability);
		const FGameplayTag ActivePhaseTag = ActivePhaseAbility->GetGamePhaseTag();

		if (!IncomingPhaseTag.MatchesTag(ActivePhaseTag))
		{
			FGameplayAbilitySpecHandle HandleToEnd = ActivePhase->Handle;
			GameState_ASC->CancelAbilitiesByFunc([HandleToEnd](const ULyraGameplayAbility* LyraAbility, FGameplayAbilitySpecHandle Handle) {
				return Handle == HandleToEnd;
				}, true);
		}
	}

	//Notify 阶段开始
	FLyraGamePhaseEntry& Entry = ActivePhaseMap.FindOrAdd(PhaseAbilityHandle);
	Entry.PhaseTag = IncomingPhaseTag;

	for (const FPhaseObserver& Observer : PhaseStartObservers)
	{
		if (Observer.IsMatch(IncomingPhaseTag))
			Observer.PhaseCallback.ExecuteIfBound(IncomingPhaseTag);
	}
}
```
