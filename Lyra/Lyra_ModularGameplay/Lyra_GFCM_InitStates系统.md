> [GFCM前提介绍](./Lyra_GFCM_Extensions系统.md#gameframeworkcomponentmanager)

## Init States System
:books: 在Client端, Actor的初始化通常涉及服务器数据同步, 异步资源加载的多方面因素, 而GameFeature的加入使该流程进一步复杂化. Init States 系统提供了一套代码框架来 **协调/规范化每个Actor 的初始化流程**, 处理初始化时的依赖关系和竞态条件, 确保Actor能顺利进入到 "GameReady" 状态
<br>

#### 相关 Concepts:

###### Actor Feature (Unique `FName`)
- 纯 GFCM 的"功能"概念, 和 GameFeature机制 无任何关联
- **一个 Actor 可 Track  多个 Features**
> **补充思考:** 尽管“Actor Feature”这个概念本身不绑定于 GameFeatureAction，但在 Lyra/Modular Gameplay 中，这个 Feature Name 往往是为了对应一个或一组由 Game Feature Plugin 注入的功能。机制上解耦，但在设计上对齐

###### InitState (`FGameplayTag`)
- 用于描述 "某Actor的某Feature所处于的状态", InitStates 之间会**存在明确的前后递进关系**

###### Feature Implementer:
- 指实现了特定 Feature 功能并负责推进其 Feature 状态的 Class（通常是 Component 或 Actor）, 需要继承 `IGameFrameworkInitStateInterface`
- 原则上, 一个 Feature 应由一个 Implementer 来主导状态推进
<br>

### 设计思路: "集中初始化" 
IniStates System 的设计模式有些类似于 *Mediator Pattern*, 在初始化流程中 Classes 之间不再相互依赖, 而是统一交互/监听 Feature's InitState. 每当 Feature 成功推进到一个新的 State 后, 意味着一些数据已经完成初始化, 进而会广播通知 *状态变换回调*, 让 Listeners 可以安全获取这些数据

- **设计目的:** 解决 *dependency-ordered* initialization 问题, 弥补 BeginPlay 本身不可靠的初始化顺序. 
- **底层实现:** 本质上是**以 Actor 为颗粒度**, 给 Registered Actors 的 **每个Feature** 提供一套 "单向"状态机
- :memo: **"Callback"推进:** 该 System 没有实现 Tick 方法去主动检测, ***任何 "状态推进尝试" 必须需要由 Feature Implementer 主动调用***
  - 以 `LyraPawnExtensionComponent` 为例, 会由外部主动触发 `ContinueInitStateChain()` 方法尝试推进Feature状态
    ```cpp
    // ULyraPawnExtensionComponent.cpp
    void ULyraPawnExtensionComponent::SetupPlayerInputComponent()
    {
      CheckDefaultInitialization();
    }

    void ULyraPawnExtensionComponent::CheckDefaultInitialization()
    {
      CheckDefaultInitializationForImplementers();
      static const TArray<FGameplayTag> StateChain = { LyraGameplayTags::InitState_Spawned, LyraGameplayTags::InitState_DataAvailable, LyraGameplayTags::InitState_DataInitialized, LyraGameplayTags::InitState_GameplayReady };
      ContinueInitStateChain(StateChain);
    }
    ```
  - :warning: 如果外部数据(初始化依赖的)更新后, 始终没有通知 => Feature Implementer => InitStates System 去推进状态 (调用 `ContinueInitStateChain()`或`TryToChangeInitState()`), 则会阻断整个流程

---

### `UGameFrameworkComponentManager( InitStates Part )`

##### 主要属性:

`TArray<FGameplayTag> InitStateOrder`
- 按顺序存放的 InitState 链路, 通常在GameInstance::Init时设置

##### `TMap<FObjectKey, FActorFeatureData> ActorFeatureMap` :star: 
- Key 为 Registered Actor, 该数据记录 Actor 依赖哪些 Features, 以及这些 Features 处于何种状态, 哪个 UObj 负责提供 Feature
- 
  ```cpp
  struct FActorFeatureData
  {
    TWeakObjectPtr<UClass> ActorClass; //类引用
    TArray<FActorFeatureState> RegisteredStates; //所有 Features
    //Feautre状态推进时的callback
    TArray<FActorFeatureRegisteredDelegate> RegisteredDelegates; 
  }

  struct FActorFeatureState
  {
    FName FeatureName; // 名称
    FGameplayTag CurrentState; // 当前InitState
    TWeakObjectPtr<UObject> Implementer; //可为 null
  }
  ```


##### 主要方法:
#### `ProcessFeatureStateChange(AActor* Actor, const FActorFeatureState* StateChange)`
- **执行时机:** 当某Actor 某 Feature 的 State 成功推进后执行, 通知回调
  - `IGameFrameworkInitStateInterface` 定义了Bind接口和Callback接口
 
- :bulb: 在回调过程中有可能会再次推进 State, 因此使用 **"Breadth-First"** 方式回调 , 写法值得参考
```cpp
void UGameFrameworkComponentManager::ProcessFeatureStateChange(AActor* Actor, const FActorFeatureState* StateChange)
{
  StateChangeQueue.Emplace(Actor, *StateChange);
  if (CurrentStateChange == INDEX_NONE)
  {
    CurrentStateChange = 0;
    //会持续处理新加的回调
    while (CurrentStateChange < StateChangeQueue.Num())
    {
      CallFeatureStateDelegates(StateChangeQueue[CurrentStateChange].Key, StateChangeQueue[CurrentStateChange].Value);
      CurrentStateChange++;
    }
    StateChangeQueue.Empty();
    CurrentStateChange = INDEX_NONE;
  }
}
```
---

### `IGameFrameworkInitStateInterface`
提供了一系列具有默认实现的接口方法来简化注册, 推进Feature状态等逻辑

- 应当由 Feature Implementer 继承
  - 在 Lyra 中为 `LyraHeroComponent`, `LyraPawnExtensionComponent`
- [相关用例](#init-lyra-character-流)

##### 主要接口方法:
#### `TryToChangeInitState(FGameplayTag DesiredState)`
  - Implementer 判断是否可以推进状态: `bool CanChangeInitState() override`
  - **if** True:
    - **在Actor层** 执行回调: `HandleChangeInitState()`
    - **在System层** Update AFM 中记录的 `CurrentState`, 执行[通知回调](#processfeaturestatechangeaactor-actor-const-factorfeaturestate-statechange)
#### `ContinueInitStateChain(const TArray<FGameplayTag>& InitStateChain)`
- **尝试多次推进 Feature 到指定 State:** 依据指定的 状态链 参数, **循环执行** `TryToChangeInitState` 中的逻辑
- 核心方法, 可以无责任调用

---

## Init Lyra Character 流


> LyraHeroComponent / LyraPawnExternsionComponent 继承了接口:**`IGameFrameworkInitStateInterface`**
> 

1\. LyraGameInstance 初始化时使用GFCM系统注册了一条**自定义的 InitState 链**:
- *"Spawned"*, *"DataAvailable"*, *"DataInitialized"*, *"GameplayReady"*

2\. 当Charater在Server/Client生成时, it Attaches and Registers all Components
- Both Comps 在 *OnRegister* 时调用 `RegisterInitStateFeature()`
  // **将 Owner Actor 和 Feautre 注册到** [ActorFeatureMap (AFM)](#tmapfobjectkey-factorfeaturedata-actorfeaturemap-star)
  

3\. 当 HeroComp *BeginPlay* 时: (PawnExternsionComp类似)
- 注册监听 PawnExtension Feature 的 IniState 切换
  - `BindOnActorInitStateChanged()`

  - Callback 记录在 AFM 中
- 尝试推进 Owner 到 *"Spawned"* 状态: `TryToChangeInitState()`


- 自动推进检测 : `CheckDefaultInitialization()`
  - 调用 `ContinueInitStateChain()`

4\. 当外部初始化相关数据更新后会主动调用 `CheckDefaultInitialization()` 方法, 持续推进Feature状态直到 "GameplayReady"
