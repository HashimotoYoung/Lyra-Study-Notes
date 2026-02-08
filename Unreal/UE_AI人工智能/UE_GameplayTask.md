# GameplayTask 系统


### UGameplayTasksComponent
> `class  UGameplayTasksComponent : UActorComponent, IGameplayTaskOwnerInterface`

负责管理和Tick `UGameplayTask`

##### 类关系:
- 默认被添加在 **AIController** 上 
  - BT中的 MoveTo节点 内部通过创建GameplayTask来实现
- 是 GAS 中 `UAbilitySystemComponent` 的父类
##### 主要属性:

```cpp
UPROPERTY(ReplicatedUsing = OnRep_SimulatedTasks)
TArray<TObjectPtr<UGameplayTask>> SimulatedTasks

FGameplayResourceSet CurrentlyClaimedResources;
```

##### 主要方法: