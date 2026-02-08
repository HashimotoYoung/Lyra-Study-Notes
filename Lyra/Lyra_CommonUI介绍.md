## Lyra CommonUI

#### 启动FrontEnd堆栈

```cpp
UnrealEditor-CommonUI.dll!UCommonUIActionRouterBase::ApplyUIInputConfig(const FUIInputConfig & NewConfig, bool bForceRefresh) 行 1325	C++
[内联框架] UnrealEditor-CommonUI.dll!UCommonUIActionRouterBase::SetActiveUIInputConfig(const FUIInputConfig &) 行 1289	C++
UnrealEditor-CommonUI.dll!FActivatableTreeRoot::ApplyLeafmostNodeConfig() 行 1321	C++
UnrealEditor-CommonUI.dll!FActivatableTreeRoot::UpdateLeafmostActiveNode(TSharedPtr<FActivatableTreeNode,1> BaseCandidateNode, bool bInApplyConfig) 行 1253	C++
UnrealEditor-CommonUI.dll!FActivatableTreeRoot::UpdateLeafNode() 行 1197	C++
UnrealEditor-CommonUI.dll!UCommonUIActionRouterBase::SetActiveRoot(TSharedPtr<FActivatableTreeRoot,1> NewActiveRoot) 行 1131	C++
UnrealEditor-CommonUI.dll!UCommonUIActionRouterBase::HandleRootNodeActivated(TWeakPtr<FActivatableTreeRoot,1> WeakActivatedRoot) 行 624	C++
UnrealEditor-CommonUI.dll!UCommonUIActionRouterBase::ProcessRebuiltWidgets() 行 872	C++
UnrealEditor-CommonUI.dll!UCommonUIActionRouterBase::Tick(float DeltaTime) 行 955	C++
[内联框架] UnrealEditor-CommonUI.dll!Invoke(bool(UCommonUIActionRouterBase::*)(float)) 行 66	C++
[内联框架] UnrealEditor-CommonUI.dll!UE::Core::Private::Tuple::TTupleBase<TIntegerSequence<unsigned int>>::ApplyAfter(bool(UCommonUIActionRouterBase::*)(float) &) 行 321	C++
UnrealEditor-CommonUI.dll!TBaseUObjectMethodDelegateInstance<0,UCommonUIActionRouterBase,bool __cdecl(float),FDefaultDelegateUserPolicy>::Execute(float <Params_0>) 行 535	C++
[内联框架] UnrealEditor-Core.dll!TDelegate<bool __cdecl(float),FDefaultDelegateUserPolicy>::Execute(float) 行 632	C++
[内联框架] UnrealEditor-Core.dll!FTSTicker::FElement::Fire(float) 行 165	C++
UnrealEditor-Core.dll!FTSTicker::Tick(float DeltaTime) 行 110	C++
UnrealEditor.exe!FEngineLoop::Tick() 行 6050	C++
```

---

### `UAsyncAction_PushContentToLayerForPlayer`
- Defined in **CommonGame** plugin, 用于 Push `UCommonActivatableWidget` 到指定的CommonUI Layer
- Derived from `UBlueprintAsyncActionBase` which is commonly used for implementing **Async Node** in blueprint
- `UAsyncAction_PushContentToLayerForPlayer::Activate()`是**开始**执行异步操作的callback方法, 一般情况下会在 调用 `PushContentToLayerForPlayer` 后的**同一帧内立即调用**


