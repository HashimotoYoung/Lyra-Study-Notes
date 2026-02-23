### FAQ:

*1. IMC中选用 "鼠标XY2D" 的情况下无法触发 `Input_LookMouse` 事件, why?*
- 进入UI界面时, **CommonUI**会调用InputMode相关API锁定鼠标输入模式:`UGameViewportClient::SetMouseLockMode()`
  进入Gameplay后, "W_ShooterHUDLayout_C_0" 内会调用 `UCommonUIActionRouterBase::SetActiveRoot(FActivatableTreeRootPtr NewActiveRoot)` 来重新设置InputMode

---

# Lyra Input

> Lyra 中的玩家输入模块基于 EnhancedInput 插件实现

### FMappableConfigPair
配置类, 负责给 EnhancedInputSystem 提供 Input Mapping Context

- 本质上是一个 **Proxy** for `UPlayerMappableInputConfig`

- **使用方式:** 
  - Used with ***GFA_AddInputConfig***
    会在`UGameFeatureAction_AddInputConfig::OnGameFeatureRegistering()`阶段进行注册


##### 主要成员:
```cpp
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UPlayerMappableInputConfig> Config;

UPROPERTY(EditAnywhere)
bool bShouldActivateAutomatically = true;

static bool RegisterPair(const FMappableConfigPair& Pair);
static void UnregisterPair(const FMappableConfigPair& Pair);
```
- 运行时初始化 EnhancedInputSystem 
```cpp
// in ULyraHeroComponent.cpp
// inside InitializePlayerInput(UInputComponent* PlayerInputComponent) 

if (Pair.bShouldActivateAutomatically && Pair.CanBeActivated()) {
	FModifyContextOptions Options = {};
	Options.bIgnoreAllPressedKeysUntilRelease = false;
	// Actually add the config to the local player							
	Subsystem->AddPlayerMappableConfig(Pair.Config.LoadSynchronous(), Options);	
}
```

---

### ULyraInputConfig : UDataAsset 
:memo: 输入配置类, 定义**映射关系** between `InputAction` and `FGameplayTag`

- `ULyraPawnData` 默认引用一份 `ULyraInputConfig`, 用于 Character 初始化
- Used by ***GFA_AddInputBinding*** 
```cpp
USTRUCT(BlueprintType)
struct FLyraInputAction {
	TObjectPtr<const UInputAction> InputAction = nullptr;
	FGameplayTag InputTag;
};

class ULyraInputConfig : public UDataAsset {
	const UInputAction* FindNativeInputActionForTag(const FGameplayTag& InputTag, bool bLogNotFound = true) const;
	const UInputAction* FindAbilityInputActionForTag(const FGameplayTag& InputTag, bool bLogNotFound = true) const;
	TArray<FLyraInputAction> NativeInputActions;
	TArray<FLyraInputAction> AbilityInputActions;
};
```
