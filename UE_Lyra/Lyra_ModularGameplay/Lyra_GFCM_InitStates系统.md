## Init States System
> :books: 在 Client 端, Actor 的初始化通常涉及服务器数据同步, 异步资源加载等多方面因素, 而 GameFeature 的加入使该流程进一步复杂化. Init States System 提供了一套代码框架来 **协调/规范化 Actors 的初始化流程**, 处理初始化时的依赖关系和竞态条件, 确保 Actors 能顺利进入到 "GameReady" 状态

<br>

#### :mag: 相关 Concepts:

##### *Actor Feature* (Unique `FName`) :
- 纯 GFCM 层的 "功能" 概念, 和 GameFeature 无关联
- **一个 Actor 可以 TRACK 多个 Features**
> :thinking: 尽管 “Actor Feature” 本身不绑定于 Game Feature Action, 但在 Lyra 中, Feature Name 通常会对应一个或一组由 GF Plugin 注入的功能. 机制上解耦, 但在设计上对齐

##### *InitState* (`FGameplayTag`) :
- 表示 "某个 Actor 的某个 Feature 所处于的状态", InitStates 之间**存在明确的前后递进关系**

##### *Feature Implementer* :
- 指负责实现 Feature 功能, 并推进 Feature 状态的具体 **Class** (Usually `Component/Actor`) 
  - 需要继承 `IGameFrameworkInitStateInterface`
- 原则上, 一个 Feature 应由一个 Implementer 来主导状态推进

<br>

### 设计思路: "集中初始化" 

IniStates System 的设计模式有些类似于 *Mediator Pattern*, 在初始化流程中 Classes 之间不再相互依赖, 而是统一交互/监听 Feature's InitState. **当 Feature 推进到新的 State 时, 意味着部分数据已经初始化完成**, 进而会 Broadcast "状态变换", 让 Listeners 可以安全获取这些数据

- **设计目的:** 解决 *dependency-ordered* initialization 问题, 弥补 `BeginPlay` 本身不可靠的初始化顺序. 
- **底层实现:** 本质上是**以 Actor 为颗粒度**, 给 Registered Actors 的 **Each Feature** 提供一套单向状态机
- :pushpin: **"主动" 推进:** ISS 没有实现 Tick 方法, 任何 " 状态推进尝试 " 需要由 *Feature Implementer* 主动触发 
  - 以 `LyraPawnExtensionComponent:IGameFrameworkInitStateInterface`为例, 外部会主动调用 `ContinueInitStateChain()` 来尝试推进 Feature 状态

    ```cpp
    // in ULyraPawnExtensionComponent.cpp
    void ULyraPawnExtensionComponent::CheckDefaultInitialization() {
      CheckDefaultInitializationForImplementers();
      static const TArray<FGameplayTag> StateChain = { LyraGameplayTags::InitState_Spawned, LyraGameplayTags::InitState_DataAvailable, LyraGameplayTags::InitState_DataInitialized, LyraGameplayTags::InitState_GameplayReady };
      ContinueInitStateChain(StateChain);
    }

    // Get Called in ALyraCharacter::SetupPlayerInputComponent()
    void ULyraPawnExtensionComponent::SetupPlayerInputComponent() {
      CheckDefaultInitialization();
    }
    ```
:warning: 如果外部数据 (初始化依赖的) 更新后, 始终没有通知 Feature Implementer 去推进状态 (指没有调用 `ContinueInitStateChain()/TryToChangeInitState()`), **则会阻断整个初始化流程**

---

### UGameFrameworkComponentManager(InitStates Part) 

##### 主要属性:

`TArray<FGameplayTag> InitStateOrder`
- 按顺序存放的 InitState 链路, 通常在 `GameInstance::Init` 中设置

##### `TMap<FObjectKey, FActorFeatureData> ActorFeatureMap` :star: 
用于记录每个 Actor 依赖哪些 Features, 以及这些 Features 处于何种状态, 哪个 `UObj` 负责提供 Feature
- Key: Registered Actor
- Value:
  ```cpp
  struct FActorFeatureData {
    TWeakObjectPtr<UClass> ActorClass; //Actor类引用
    TArray<FActorFeatureState> RegisteredStates; //Features
    TArray<FActorFeatureRegisteredDelegate> RegisteredDelegates; //状态推进 callback
  }

  struct FActorFeatureState {
    FName FeatureName; //Feature 名称
    FGameplayTag CurrentState; //Feature 当前状态
    TWeakObjectPtr<UObject> Implementer; //可为 null
  }
  ```


##### 主要方法:

#### `ProcessFeatureStateChange(AActor* Actor, const FActorFeatureState* StateChange)`
在某 Actor 的某 Feature 成功推进 State 后执行, 通知回调
- `IGameFrameworkInitStateInterface` 定义了 Bind 和 Callback 相关 API
- :bulb: 因为在回调执行过程中有可能会再次推进 State, 这里使用 **"Breadth-First"** 方式回调 , 值得参考
```cpp
void UGameFrameworkComponentManager::ProcessFeatureStateChange(AActor* Actor, const FActorFeatureState* StateChange) {
  StateChangeQueue.Emplace(Actor, *StateChange);
  if (CurrentStateChange == INDEX_NONE) {
    CurrentStateChange = 0;
    //会持续处理新加的回调
    while (CurrentStateChange < StateChangeQueue.Num()) {
      CallFeatureStateDelegates(StateChangeQueue[CurrentStateChange].Key, StateChangeQueue[CurrentStateChange].Value);
      CurrentStateChange++;
    }
    StateChangeQueue.Empty();
    CurrentStateChange = INDEX_NONE;
  }
}
```
---

### IGameFrameworkInitStateInterface

核心 Interface, 需要由 *Feature Implementer* 继承

- Lyra 中的两个 Implementer 为 `LyraHeroComponent`, `LyraPawnExtensionComponent`
- :pushpin: Implementer 的主要职责:
  1. **Override** `bool CanChangeInitState()` 和 `void HandleChangeInitState()`
  2. **声明 Feature Name**, 例如: `const FName ULyraPawnExtensionComponent::NAME_ActorFeatureName("PawnExtension")`

- 相关用例: [Init Lyra Character 流](#init-lyra-character-流)

##### 主要方法:
#### `TryToChangeInitState(FGameplayTag DesiredState)`
  - Implementer 首先判断是否可以推进状态: 
    - `virtual bool CanChangeInitState()`
  - **if** 判断为 True:
    - **在 Actor 层** 执行回调: `virtual void HandleChangeInitState()`
    - **在 ISS 层** Update `ActorFeatureMap`  中的 `CurrentState`, 并执行[通知回调](#processfeaturestatechangeaactor-actor-const-factorfeaturestate-statechange)
#### `ContinueInitStateChain(const TArray<FGameplayTag>& InitStateChain)`
- 依据指定的 "状态链" 参数, **尝试循环推进 Feature  State** 
  - Would call `TryToChangeInitState()` Looply


---

### Init Lyra Character 流

:pencil2: **Start**
1: `LyraGameInstance` 在初始化时向 GFCM 注册一条 **自定义 InitState 链**
- *"Spawned"* => *"DataAvailable"* => *"DataInitialized"* => *"GameplayReady"*
- :warning: 不同 Feature 可共用同一种 InitState 链

2: 当 Charater 在 Server or Client 生成时, 会注册 Components, including `LyraHeroComponent` and `LyraPawnExtensionComponent`
- Both Comps 在 *`OnRegister()`* 时调用接口方法 `RegisterInitStateFeature()`
  => **注册 OwnerActor 和 Feautre 信息到** [ActorFeatureMap ](#tmapfobjectkey-factorfeaturedata-actorfeaturemap-star)
  

3: 在 HeroComp *`BeginPlay()`* 时: (PawnExtensionComp 类似)
- 会注册监听 PawnExtension Feature 的 InitState 变化
  - `BindOnActorInitStateChanged(...)`
- 尝试推进 "OwnerActor's HeroComp Feature" 到 *"Spawned"* 状态: `TryToChangeInitState()`
- 执行一次自动推进 : `CheckDefaultInitialization()` => `ContinueInitStateChain()`

4: 当外部初始化相关数据更新后, 会主动调用 `CheckDefaultInitialization()` 方法, 推进 Feature's InitState 直到变成 "GameplayReady"

<br>

#### Actors Features 之间的依赖关系示例 :star: 
- 这里 `ULyraHeroComponent` 会在确认 "OwnerPawn's PawnExtension Feature" 达到 *InitState_DataInitialized* 后才会允许推进自身的 Feature's InitState

```cpp
// in ULyraHeroComponent.cpp
bool ULyraHeroComponent::CanChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) const {
  //...
  else if (CurrentState == LyraGameplayTags::InitState_DataAvailable && DesiredState == LyraGameplayTags::InitState_DataInitialized)
  {
    ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();
    return LyraPS && Manager->HasFeatureReachedInitState(Pawn, ULyraPawnExtensionComponent::NAME_ActorFeatureName, LyraGameplayTags::InitState_DataInitialized);
  }
  //...
  return false
}
```
- 这里 `ULyraHeroComponent` 在 "OwnerPawn's PawnExtension Feature" 达到 *InitState_DataInitialized* 时, 尝试主动推进自身的 Feature's InitState
```cpp
// in ULyraHeroComponent.cpp
void ULyraHeroComponent::OnActorInitStateChanged(const FActorInitStateChangedParams& Params) {
	if (Params.FeatureName == ULyraPawnExtensionComponent::NAME_ActorFeatureName) {
		if (Params.FeatureState == LyraGameplayTags::InitState_DataInitialized) {
			CheckDefaultInitialization();
		}
	}
}
```