## Lyra GameFrameworkComponentManager
>:books: Lyra ModularGameplay 三大系统之一, 继承自 `UGameInstanceSubsystem` 

内部实际由**两个子系统组成**: [*Extension Handlers System* ](#extension-handlers-system-ehs) 和 [*Init States System*](./Lyra_GFCM_InitStates系统.md)
- 两者代码层面基本自洽, 无相互依赖
- GFCM 被 Game Feature 高度依赖, 和 Experience 无耦合

---

## Extension Handlers System (EHS)

> :books: 可看作是一个中介管理系统, 此系统是将 Game Feature 从 “静态资产” 转化为 “运行时 Actor 行为” 的关键桥梁

主要提供以下功能:
1. **Dynamic Component Injection:** 
   - 当一个 Actor 注册为 Receiver 时, EHS 会查询此 Actor 的 Class/ParentClass 信息, 然后添加相应的 Components, **该功能以 Class 为颗粒度**
2. **Generic Code Injection:** 
   - 当一个 Actor 注册为 Receiver 时, EHS 会查询此 Actor 的 Class/ParentClass 信息, 然后回调执行通用逻辑代码, **该功能以 Class 为颗粒度** 
   - [用法示例](#add-extension-handler-用法示例)

3. **Clean up when GF deactivates:**
   - [Auto Cleanup](#auto-cleanup-star)


#### :mag: 相关 Concepts:
- *ReceiverClass :* 指注册使用 EHS 的 `TSoftClassPtr<AActor>`, 以路径方式记录
- *Receiver :* 指注册使用 EHS 的实际 `Actor` 


---
### UGameFrameworkComponentManager(ExtensionHandlers Part)

##### 主要属性:

##### `TMap<FComponentRequestReceiverClassPath, TSet<UClass*>> ReceiverClassToComponentClassMap` :star:
- Key: *ReceiverClass* 的路径信息, 底层实现为 `TArray<FName>`
- Value: 该 *ReceiverClass* 对应的 **所有需要动态注入 ActorComponent's UClass Objs**

`TMap<FComponentRequestReceiverClassPath, FExtensionHandlerEvent> ReceiverClassToEventMap`

- Key: 同上
- Value: 实际类型为 `TMap<FDelegateHandle, FExtensionHandlerDelegateInternal>`

##### `TMap<UClass*, TSet<FObjectKey>> ComponentClassToComponentInstanceMap` :star:
- Key: ActorComponent's UClass Obj
- :pushpin: Value: 追踪 **所有通过 GFCM 创建的** Component objs 
##### `TMap<FComponentRequest, int32> RequestTrackingMap;`
- 记录请求记数, 避免重复注册
<br>

##### 主要方法:
### `AddReceiverInternal(AActor* Receiver)` :star:

负责将 Actor 注册为 *Receiver*, Lyra 中所有 Modular Actor 会在 `PreInitializeComponents()` 中默认调用此方法

- 基于继承关系**遍历:** `for (UClass* Class = Receiver->GetClass(); Class && Class != AActor::StaticClass(); Class = Class->GetSuperClass())`
   - 尝试获取 ComponentClass from `ReceiverClassToComponentClassMap`
     - 在 *Receiver* 上创建 Component 实例: `CreateComponentOnInstance(Receiver,ComponentClass)`
   - 尝试获取 HandlerEvent from `ReceiverClassToEventMap`
     - 执行回调

<br>

### `AddComponentRequest()`
```cpp
TSharedPtr<FComponentRequestHandle> UGameFrameworkComponentManager::AddComponentRequest(
const TSoftClassPtr<AActor>& ReceiverClass, TSubclassOf<UActorComponent> ComponentClass)
```
核心方法, 主要被用于 GFA_AddComponents
1. 获取 *ReceiverClass* 的路径信息: `FComponentRequestReceiverClassPath ReceiverClassPath(ReceiverClass)`

2. 封装请求: `FComponentRequest NewRequest{ ReceiverClassPath, ComponentClassPtr }`

3. 更新请求记数 for [RequestTrackingMap](#tmapfcomponentrequest-int32-requesttrackingmap) 
**if** "是此类请求的第一次调用" (`RequestCount == 1`):
   - :pushpin: 注册 **ReceiverClassPath** 和 **ComponentClassPtr** 到 [ReceiverClassToComponentClassMap](#tmapfcomponentrequestreceiverclasspath-tsetuclass-receiverclasstocomponentclassmap-star)

     > 通过这一步 Actor类 和 Added Components类形成映射关系 
   - GFCM 会立即查询并获取 **当前 World** 中所有匹配 *ReceiverClass* 类型的 Actors, 并创建Component实例
     - :warning: 如果这些 Actors 在生成时没有注册为 *Receiver*, 会报警告提示


4. 返回 Handle  
   - `return MakeShared<FComponentRequestHandle>(this, ReceiverClass, ComponentClass)`

<br>

#### "双向握手" 机制 
> 无论谁先到，GFCM 都能保证逻辑最终 “合流”

- Case A (Actor 已存在): 当 GFA 调用 `AddComponentRequest()` 时，GFCM 会立即遍历当前 World 的 Actors，如发现匹配的类则直接注入组件

- Case B (GFA 先激活): 新的 Actor 在被 Spawn 时, 会在 `PreInitializeComponents()` 中调用 `AddReceiverInternal()`, GFCM 此时会查询已有的 "Requests"，发现匹配则注入组件。

---

### Auto Cleanup :star:
EHS使用 **Request 记数机制** 来实现自动清理功能

- 当 GFA 调用 `AddComponentRequest()` 和 `AddExtensionHandler()` 时:
  - GFCM 将调用封装成一次请求 (`FComponentRequest newReq`) 并记录在 `this.RequestTrackingMap` 
  - 返回句柄 `MakeShared<FComponentRequestHandle>(this, data...)` 给 GFA 持有
    > `FComponentRequestHandle` 的结构和 `FComponentRequest` 基本一致, 方便析构时查询
- 当 GF Deactivate 时, 相应 GFA 会 Clear Containers of `TSharedPtr<FComponentRequestHandle>`
- :pushpin: **In `FComponentRequestHandle`'s Destructor**
  - 通知 GFCM 减一请求记数, 当记数 == 0 时,清理对应数据 in `ComponentClassToComponentInstanceMap/ReceiverClassToComponentClassMap`
   ```cpp
   FComponentRequestHandle::~FComponentRequestHandle() {
      UGameFrameworkComponentManager* LocalManager = OwningManager.Get();
      if (LocalManager) {
         if (ComponentClass.Get()) LocalManager->RemoveComponentRequest(ReceiverClass, ComponentClass);
         if (ExtensionHandle.IsValid()) LocalManager->RemoveExtensionHandler(ReceiverClass, ExtensionHandle); 
      } 
   }
   ```
---

### Add Extension Handler 用法示例


> :memo: **基于 Event Name** 进行处理, 以 `class UGameFeatureAction_AddInputBinding` 为例
 
当 GF 被激活时, GFA 向 GFCM 注册一个 Extension Handler: 
- 指定 `APawn` 类型为 *ReceiverClass*
- **注册 Handler 回调:** `HandlePawnExtension`
   ```cpp
   // in UGameFeatureAction_AddInputBinding.cpp
   GFCM::FExtensionHandlerDelegate AddAbilitiesDelegate = GFCM::FExtensionHandlerDelegate::CreateUObject(this, &ThisClass::HandlePawnExtension, ChangeContext);
   TSharedPtr<FComponentRequestHandle> ExtensionRequestHandle = GFCM->AddExtensionHandler(APawn::StaticClass(), AddAbilitiesDelegate);
   ```
:pushpin: 当 `LyraHeroComponent` 初始化完成后, 主动通知 EHS 执行 Extention Handle
- 发送具体的 OwnerPawn 和 EventName (`FName`)
  ```cpp
  // in LyraHeroComponent.cpp
  GFCM::SendGameFrameworkComponentExtensionEvent(const_cast<APawn*>(OwnerPawn), NAME_BindInputsNow);
  ```

- `GFA_AddInputBinding` 在 Handler 中, 根据 EventName 进行对应处理 
  ```cpp
   // in UGameFeatureAction_AddInputBinding.cpp
   void UGameFeatureAction_AddInputBinding::HandlePawnExtension(AActor* Actor, FName EventName, FGameFeatureStateChangeContext ChangeContext) {

      APawn* AsPawn = CastChecked<APawn>(Actor);
      FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

      if (EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved) {
         RemoveInputMapping(AsPawn, ActiveData);
      }

      if (EventName == ULyraHeroComponent::NAME_BindInputsNow) {
         AddInputMappingForPlayer(AsPawn, ActiveData);
      }
   }
  ```