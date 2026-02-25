### FAQ:
*1. Why `UGameplayCueManager` 继承自 `UDataAsset` ?*

- GCMgr 不只是运行时逻辑管理器，而是一个 Data-driven 的全局配置与调度中心 
- 因此 GCMgr 需要 **“资产化”**, 以存储大量**可编辑的数据**, 例如:
  - Gameplay Cue 扫描路径 (GameplayCue paths) 
    [GameplayCueSet]
    Editor 与 Runtime 的 ObjectLibrary
    Preallocation 配置
    异步加载的 AssetHandle 缓存

- 需要有 *GameInstance级* 的生命周期
<br>

*2. What is `IGameplayCueInterface` ?*
- Its Implementer 可看作是一个 **Custom-Actor-Local** GC Handler

---
# GAS GameplayCue 分析 
GameplayCue is a **network-efficient** system to manage **cosmetic effects** (特效, 声音等)

GC system 的设计模式类似于 **Publish-Subscribe Pattern**
  - Publisher : `UAbilitySystemComponent`
  - Router/Manager : `GameplayCueManager`
    - Message : `FGameplayTag GameplayCueTag`
  - Subscriber : 各种 `GameplayCueNotify` (GCN)

#### 类关系:
- GC System is a part of GAS, `UGameplayCueManager` is coupled with and primarily invoked by `ASC`

- GC 可以用于 **non-ASC** `Actor` 
<br>


#### ASC 可通过三种行为与 GC 系统交互:
|     |:star:核心APIs              | 对应 Event Type                        | GCN 生命周期           | 典型用途               |
| --------- | -------------------- | ----------------------------------- | ---------------- | ------------------ |
| **Execute GC** | `ExecuteGameplayCue` | `EGameplayCueEvent::Execute`        | 瞬时（One-shot） | 命中闪光、一次性音效、爆炸特效    |
| **Add GC** | `AddGameplayCue`     | `EGameplayCueEvent::OnActive`<br>`EGameplayCueEvent::WhileActive`| Ongoing, 直到被移除   | Buff 光环、燃烧特效、持续 UI |
| **Remove GC** | `RemoveGameplayCue`  | `EGameplayCueEvent::Removed`        | 结束事件         | 关闭特效、清理状态          |


```cpp
// in UAbilitySystemComponent.h
void ExecuteGameplayCue(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);
void AddGameplayCue(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);
void RemoveGameplayCue(const FGameplayTag GameplayCueTag);
// 专门用于 GE
void HandleDeferredGameplayCues(const FActiveGameplayEffectsContainer* GameplayEffectsContainer)
```

---
## 1. GameplayCue 相关类
### UGameplayCueManager 

##### 主要属性:
`FGameplayCueObjectLibrary RuntimeGameplayCueObjectLibrary` UPROPERTY(transient)
- 管理运行时GC相关资源 

`TArray<FGameplayCuePendingExecute> PendingExecuteCues` UPROPERTY(transient)  
`int32 GameplayCueSendContextCount` UPROPERTY(transient)
- [Execute GC 流](#execute-gc-流) 相关属性

##### 主要方法:
#### `HandleGameplayCue()`
[Handle GC 流](#handle-gc-流)中的核心API

#### `AddPendingCueExecuteInternal(FGameplayCuePendingExecute& PendingCue)`

[Execute GC 流](#execute-gc-流)中的核心API

---
### UGameplayCueSet : UDataAsset
GCN 的 Container 类, 定义了 Handle GC 流程中的核心APIs
##### 类关系:
- Contained as `UGamplayCueManager.RuntimeGameplayCueObjectLibrary.CueSet`

##### 主要属性:

`TArray<FGameplayCueNotifyData> GameplayCueData`   
`TMap<FGameplayTag, int32> GameplayCueDataMap` 

- GameplayCueData 包含了所有GCN的相关数据信息; 
  同时定义了配套的查询 Map, in32为ArrayIndex; 
  ```cpp
  //GCN 的数据封装类
  struct FGameplayCueNotifyData
  {
    UPROPERTY(EditAnywhere, Category=GameplayCue)
    FGameplayTag GameplayCueTag;

    UPROPERTY(EditAnywhere, Category=GameplayCue, meta=(AllowedClasses="/Script/GameplayAbilities.GameplayCueNotify_Static, /Script/GameplayAbilities.GameplayCueNotify_Actor"))
    FSoftObjectPath GameplayCueNotifyObj; // GCN 资产路径
    
    UPROPERTY(transient)
    TObjectPtr<UClass> LoadedGameplayCueClass; // runtime类指针

    int32 ParentDataIdx; //父级 GameplayCueTag 对应ID
  }
  ```

##### 主要方法:

##### `bool HandleGameplayCueNotify_Internal(AActor* TargetActor, int32 DataIdx, EGameplayCueEvent::Type EventType, FGameplayCueParameters& Parameters)`
[Handle GC 流](#handle-gc-流) 的必经API

---

### GCN Classes 


### `UGameplayCueNotify_Static : UObject`
- In practice, 主要用于处理 Execute GC Event 

### `AGameplayCueNotify_Actor : Actor`
- In practice, 主要用于处理 Add/Remove GC Event

---

## 2. GameplayCue 相关流程 

:books: GameplayCue的运转流程大体可分为 Invoke GC Event **(包括 Execute / Add / Remove GC)** 和 Handle GC Event 两个阶段 

该流程中存在**四个基本要素:**
1. **GC Actor:** GCs are **always handled relative to a Target `Actor`** — even “world” effects are anchored to an actor context.
   - 对于 Actor **associated with ASC**: 会通过该 ASC 执行 Handle 流程
   - 对于 普通 Actor: 可由 GameplayCueManager 直接 **本地Handle**
2. **GC Tag:** 本质为以 "GameplayCue." 开头的 `FGameplayTag`
3. **GC Param:** 携带的Payload
4.  [四种 EGameplayCueEvent](#21-gcue-reliability-梳理)
<br>

GAS 主要提供了以下**三种途径** Invoke GC:

1. **When Apply GE** 
   - `UGameplayEffect` 本身持有 `TArray<FGameplayEffectCue>`
   - Apply 时, 会基于 GE 的 **持续类型(DurationType)** 走不同逻辑: 
     - `Instant` GE : Execute 
     - `Duration/Infinite` GE : OnActive → WhileActive → OnRemove 
   - [Eventually routed to the ASC's APIs](#asc-可通过三种行为与-gc-系统交互)

2. **Call from GA** 
   - `UGameplayAbility` 提供了一系列 GC 相关APIs 
   - [Eventually routed to the ASC's APIs](#asc-可通过三种行为与-gc-系统交互)

3. **Call from `UGameplayCueFunctionLibrary`**
   - 可选择性绕过ASC, 直接本地 Handle GC
---

### Execute GC 流
内部可分为 Add Pending Cue , Flush Pending Cue 两个子流程
- :star: 使用了 Context + Pending Request 的设计来**批量处理**GC请求
- 对应事件类型为 `EGameplayCueEvent::Executed`
```cpp
//相关API:
void UGameplayCueManager::AddPendingCueExecuteInternal(FGameplayCuePendingExecute& PendingCue)

//相关Struct
FGameplayCuePendingExecute, FScopedGameplayCueSendContext

//FScopedGameplayCueSendContext实现
FScopedGameplayCueSendContext::FScopedGameplayCueSendContext() {
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->StartGameplayCueSendContext();
}
FScopedGameplayCueSendContext::~FScopedGameplayCueSendContext() {
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->EndGameplayCueSendContext();
}
```
#### Add Pending Cue 流:
- 当 ASC 尝试 Execute GC 时, GameplayCueManager 会将调用封装为一个 Request (`FGameplayCuePendingExecute`), 记录到 `GCM.PendingExecuteCues` 
  - Request 包含 `FGameplayCueParameters`, `FGameplayEffectSpecForRPC` 等数据
- **if** `GCM.GameplayCueSendContextCount == 0`, 立即进入Flush 流
  - :pushpin: 当从GA施加GE时, GA会在 Apply 前生成一个 `FScopedGameplayCueSendContext`, 从而触发 `GameplayCueSendContextCount` 加1, 因此不会立即Flush
  - 直到 Apply完成 => `FScopedGameplayCueSendContext`析构 => Flush 
  
#### Flush Pending Cue 流:
- 遍历 `PendingExecuteCues`
  - **if** *Authroity*, 调用 Multicast (例如 `NetMulticast_InvokeGameplayCueExecuted_FromSpec`)
  - **else if** *Local Prediction*, 执行本地 Handle GC 流
- 清空 `PendingExecuteCues`

---

### Add GC 流

### Remove GC 流

---

### Handle GC 流

GameplayCueManager 根据接收到的 GameplayCueTag, 找到对应的 GCN 并发送对应 Event 

:pencil2: Start from GCM's **核心API:** 

```cpp
void UGameplayCueManager::HandleGameplayCue(
AActor* TargetActor, FGameplayTag GameplayCueTag, 
EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters, 
EGameplayCueExecutionOptions Options)
```
1: **if** Run-On-Server, then **return**
- **[Server]** 默认会忽略 GC 处理, 可改动

2: 尝试 **CAST** TargetActor => gci (`IGameplayCueInterface`) 
- 检查 TargetActor 是否 **Accept GC**

3: **if** Accept, 交由 Global `UGameplayCueSet` 进行处理
// `RuntimeGameplayCueObjectLibrary.CueSet->HandleGameplayCue(TargetActor, GCTag, EventType, Parameters)`
  
  - 尝试找到匹配 GameplayCueTag 的 **GCN's *LoadedGameplayCueClass* (`UClass*`)**
  - 获取/生成 **GCN instance** (`UGameplayCueNotify_Static*` / `AGameplayCueNotify_Actor*`), 进行内部处理:
    - 根据 EventType 调用对应方法, 例如 `WhileActive(AActor* MyTarget, const FGameplayCueParameters& Parameters)`
  - :pushpin: **递归执行该流程** with GameplayCueTag's Parent Tag


4: **if** Accept && gci != NULL, 进行二次处理
// `gci->HandleGameplayCue(TargetActor, GameplayCueTag, EventType, Parameters)`

5: 强制网络更新: `TargetActor->ForceNetUpdate()`

---
### 2.1 GC Reliability 梳理


- Event `Execute`: :x: Always sent by Unreliable Multicast

| Role           | Event         | GE-Applied GC                                                                       | Other (Non-GE) GC                                                    |
| -------------- | ------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| **主控端** | `OnActive`    | :heavy_check_mark: Repl  (`FActiveGameplayEffectsContainer::NetDeltaSerialize`) | :x: Unreliable Multicast                                               |
|                | `WhileActive` | :heavy_check_mark: Repl (`FActiveGameplayEffectsContainer::NetDeltaSerialize`) | :x: Unreliable Multicast                                               |
|                | `Remove`      | :heavy_check_mark: Repl (`RemoveActiveGameplayEffectGrantedTagsAndModifiers`)  | :heavy_check_mark: Repl (guaranteed cleanup)                    |
| **模拟端**  | `OnActive`    | :x: Unreliable Multicast                                                              | :x: Unreliable Multicast                                               |
|                | `WhileActive` | :heavy_check_mark: Repl (`ASC::MinimalReplicationGameplayCues`)                | :heavy_check_mark: Repl (`ASC::MinimalReplicationGameplayCues`) |
|                | `Remove`      | :heavy_check_mark: Repl (`ASC::MinimalReplicationGameplayCues`)                | :heavy_check_mark: Repl (`ASC::MinimalReplicationGameplayCues`) |


---

## 3. GameplayCue 资源管理
todo: InitObjectLibrary