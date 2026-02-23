# 行为树结点介绍
#### UBTNode 类架构:
- `UBTNode : UObject`
  - **`UBTCompositeNode`** :pushpin:
  - **`UBTTaskNode`** :pushpin:
    - `UBTTask_Wait` ...
  - `UBTAuxiliaryNode`
    - **`UBTDecorator`** :pushpin:
    - **`UBTService`** :pushpin:


## Composite `UBTCompositeNode`
例如 selector, sequencer

##### 主要属性:
**`TArray<FBTCompositeChild> Children`**
`TArray<TObjectPtr<UBTService>> Services`
通过该以上两个属性可知:
- :star: Decorator 节点 are always owned by the **parent composite of the node they decorate**

- :star: Service 节点 are always owned by the node they service, 可 Attach to Composite / Task

```cpp
struct FBTCompositeChild
{
	TObjectPtr<UBTCompositeNode> ChildComposite = nullptr;
	TObjectPtr<UBTTaskNode> ChildTask = nullptr;
	TArray<TObjectPtr<UBTDecorator>> Decorators;
	TArray<FBTDecoratorLogic> DecoratorOps;
};
```

---

## Task `UBTTaskNode`
Nodes "do" things
- **叶子节点**, no children allowed
- 本身持有 `TArray<TObjectPtr<UBTService>> Services`
##### 主要方法:
```cpp
virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
virtual void TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)
virtual EBTNodeResult::Type AbortTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
```
- 注意区分 Execute 和 Tick 的区别

`void SetNextTickTime(uint8* NodeMemory, float RemainingTime) const`
- 每个TaskNode可以定义Tick时间?

`virtual void TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds);`


---

## Decorator `UBTDecorator`
Gatekeeper for node, define **whether or not** a branch in the tree, or a single node can be **executed**
- 无法独立存在, **只能attach**到compsite node 或 task node 
- Also known as `Conditionals`, its conditions are **evaluated before the execution** of Node attached


#### :star: What "Notify Observer and Observer Aborts" means??
- **Decorator as the Observer:** The decorator itself acts as the "observer" to listen for changes to its specific condition.
- Notified 的 **Target** 是 BT System, 而不是所绑Node
- **Behavior Tree Re-evaluates/Aborts:** Upon receiving this notification, the Behavior Tree's core system then decides the appropriate action based on the `Observer Aborts` setting

#### 关于 Abort
- Abort is essentially a **command** to stop what you are currently doing, 是一个针对于在 **executing 中的 nodes** 的概念

---

## Service `UBTService`
Execute at their **defined frequency** as long as their branch is being executed

- 无法独立存在, **只能attach**到`composite node `
- The Service is activated and begins ticking before the `Composite node` (该service所附属的) starts executing its children.

> **For example**, a service can be used to determine which enemy is the best choice for the AI Pawn to pursue while the Pawn continues to act normally in its Behavior Tree toward its current enemy.

主要 Purposes:

- Read `Blackboard` data
- 执行环境检测逻辑
- Modify `Blackboard` data: **By this way** service node can affect the tree flow

---

### What is Reactive Behavior Tree ?

- 每个Tree保持尽可能轻量化并遵守Single Responsibility
- Tree的上层使用LongTerm Stack + Reactive Stack