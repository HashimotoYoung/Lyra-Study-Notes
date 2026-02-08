## GameFrameworkComponentManager
Lyra ModularGameplay 三大系统之一, 继承 `UGameInstanceSubsystem`, 其内部实际包含了**两个子系统**: *Extension Handlers System* 和 [*Init States System*](./Lyra_GFCM_InitStates系统.md)
- 两者代码实现部分自洽, 几乎无交集 !?
- GFCM 被 GameFeature系统 高度依赖, 但和 Experience系统 没有耦合
---

## Extension Handlers Subsystem

可看作是一个中介管理系统, 主要服务于 GameFeature , 提供以下功能:

1. **Dynamic Component Injection:** 
当注册了一个 Actor 时, EHS 会查询此 Actor 的 Class/ParentClass 信息, 然后添加相应的 Components, **该功能以 Class 为颗粒度**
2. **Generic Code Injection:** 
当注册了一个 Actor 时, EHS 会查询此 Actor 的 Class/ParentClass 信息, 然后通过回调执行通用逻辑代码, **以 Class 为颗粒度**,  [用法示例](#addextensionhandler-用法)

3. **Clean up when Game Feature deactivates:**
见 [Auto Cleanup](#star-auto-cleanup)


#### 相关 Concepts:
###### Receiver Class (`TSoftClassPtr<AActor>`)
- 指注册使用 EHS 的 Actor Class, 会以路径方式进行记录
###### Receiver (`Actor`)
- 指注册使用 EHS 的实际 Actors, 会在注册时被 GFCM 自动添加 Components

---
## `UGameFrameworkComponentManager(ExtensionHandlers Part)`

##### 主要属性:
##### `TMap<FComponentRequestReceiverClassPath, TSet<UClass*>> ReceiverClassToComponentClassMap` :star:
- Key: Receiver类的路径信息, 底层实现为 TArray\<FName>
- Value: 该 Receiver 类的**每个 ActorComponent's CMO**

`TMap<FComponentRequestReceiverClassPath, FExtensionHandlerEvent> ReceiverClassToEventMap`

记录回调, 以便Receiver加入时执行特定逻辑

- ValueType = `TMap<FDelegateHandle, FExtensionHandlerDelegateInternal>`

##### `TMap<UClass*, TSet<FObjectKey>> ComponentClassToComponentInstanceMap` :star:
- Key: ActorComponent's CMO
- :star: Value: 追踪**所有通过GFCM系统创建的**的 component objs
##### `TMap<FComponentRequest, int32> RequestTrackingMap;`
起保护作用, 避免重复注册
<br>

##### 主要方法:
### `AddReceiverInternal(AActor* Receiver)`

:star: **注册为 Receiver :** m在 Lyra 中,所有 ModularActor 会在 `PreInitializeComponents()` 中调用此方法

1. 按继承关系遍历: `for (UClass* Class = Receiver->GetClass(); Class && Class != AActor::StaticClass(); Class = Class->GetSuperClass())`
   - 尝试读取 ComponentClass 信息 from [ReceiverClassToComponentClassMap](#tmapfcomponentrequestreceiverclasspath-tsetuclass-receiverclasstocomponentclassmap),
  然后创建Component实例: `CreateComponentOnInstance(Receiver,ComponentClass)`
<br>

### `AddComponentRequest()`
核心方法, 用于 UGameFeatureAction_AddComponents
```cpp
TSharedPtr<FComponentRequestHandle> UGameFrameworkComponentManager::AddComponentRequest(
const TSoftClassPtr<AActor>& ReceiverClass, TSubclassOf<UActorComponent> ComponentClass)
```

1. 获取 Receiver类 的路径信息: `FComponentRequestReceiverClassPath ReceiverClassPath(ReceiverClass)`
   - 这里 ReceiverClass **只能为Actor子类**
2. 封装请求: `FComponentRequest NewRequest{ ReceiverClassPath, ComponentClassPtr }`
   - :memo: ComponentClassPtr 指向 **CMO**
3. 更新数据: [RequestTrackingMap](#tmapfcomponentrequest-int32-requesttrackingmap) 记数+1, 
**if** 是第一次调用 (`count == 1`):
   - :star: 添加 **ReceiverClassPath** 和 **ComponentClassPtr** 到 [ReceiverClassToComponentClassMap](#tmapfcomponentrequestreceiverclasspath-tsetuclass-receiverclasstocomponentclassmap)
        ```cpp
        TSet<UClass*>& ComponentClasses = ReceiverClassToComponentClassMap.FindOrAdd(ReceiverClassPath);
        ComponentClasses.Add(ComponentClassPtr);
        ```
     // 至此注册部分已经完成, Actor类和Added Components类已形成映射关系 
   - **即时添加:** GFCM 会立刻查询并获取当前World中所有 ReceiverClass(Actor) 实例,
   在每个Actor上创建Component实例: `CreateComponentOnInstance()`

4. **返回 [Handle](#star-fcomponentrequesthandle-设计)**  
   `return MakeShared<FComponentRequestHandle>(this, ReceiverClass, ComponentClass)`

---

### :star: Auto Cleanup
EHS使用 **Request 机制** 来实现自动清理功能

- 当某个 GFA 执行 `AddComponentRequest()` 和 `AddExtensionHandler()` 时:
  - GFCM 创建 **`FComponentRequest newReq`** 记录在其 RequestTrackingMap 
  - 返回 `MakeShared<FComponentRequestHandle>(this, data...)` 给各个 GFA 的 container 成员变量持有
  - **FComponentRequestHandle 的数据结构和 FComponentRequest 基本一致, 方便析构时查询**
- 当GF Deactivate时, 相应 GFA 会清空 containers of `TSharedPtr<FComponentRequestHandle>`
- **在 FComponentRequestHandle 的析构函数中:**
  - 通知 GFCM 减少请求记数, 当记数 = 0 时,清理对应数据 in `ComponentClassToComponentInstanceMap` and `ReceiverClassToComponentClassMap`
   ```cpp
   FComponentRequestHandle::~FComponentRequestHandle() {
      UGameFrameworkComponentManager* LocalManager = OwningManager.Get();
      if (LocalManager)
      {
         if (ComponentClass.Get())
         {
            LocalManager->RemoveComponentRequest(ReceiverClass, ComponentClass);
         }
         if (ExtensionHandle.IsValid())
         {
            LocalManager->RemoveExtensionHandler(ReceiverClass, ExtensionHandle);
         }
      } 
   }
   ```
---

### AddExtensionHandler 用法


> **基于 Event Name** 进行处理, 以 class UGameFeatureAction_AddInputBinding 为例
 
当被激活时 (`GFA_AddInputBinding::AddToWorld()`), 向 GFCM 注册一个 Extension Handler
- 指定 APawn 作为接收者类 **(Receiver Class)**
   ```cpp
   // in UGameFeatureAction_AddInputBinding.cpp
   GFCM::FExtensionHandlerDelegate AddAbilitiesDelegate = GFCM::FExtensionHandlerDelegate::CreateUObject(this, &ThisClass::HandlePawnExtension, ChangeContext);
   TSharedPtr<FComponentRequestHandle> ExtensionRequestHandle = GFCM->AddExtensionHandler(APawn::StaticClass(), AddAbilitiesDelegate);
   ```
当 LyraHeroComponent 处理完自身的 Input 初始化后, 发送通知给 Extension Handler
- 只需发送 OwnerPawn 和 **Event Name (`FName`)**
   ```cpp
   // in LyraHeroComponent.cpp
   GFCM::SendGameFrameworkComponentExtensionEvent(const_cast<APawn*>(OwnerPawn), NAME_BindInputsNow);
   ```

- GFA_AddInputBinding 在 回调Handler(`HandlePawnExtension`) 中收到 Event `NAME_BindInputsNow` 后, 进行 `AddInputMappingForPlayer()` 的处理