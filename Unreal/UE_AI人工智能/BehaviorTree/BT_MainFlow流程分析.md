

:star: `Evaluation and Execution` are **相互交错** and occur within the single "tick" of the Behavior Tree for each frame. 
- :x: it’s not like “evaluate everything first, then execute.”

Unreal Behavior Trees are Event-Driven , what this mean??
> https://github.com/meniku/NPBehave 帮助理解 Event-Driven bt tree

---

## How Behavior Tree System Ticks ?? 
需要把evalute阶段也考虑进来 , 分析UBehaviorTreeComponent::TickComponent

In Unreal Engine Behavior Trees, the tree **does not always execute from the root at each frame**. It employs a system of "active" or "in-progress" nodes to optimize performance and maintain ongoing behaviors.


- 首次 Execution (or After an Abort): When a Behavior Tree **first** starts executing, or after a branch has been **aborted** (e.g., due to a decorator's condition changing), the evaluation process begins from the root node.

- "**In Progress"** Tasks: Once a Task node starts executing and signals that it's "in progress" (by not immediately calling "Finish Execute"), the Behavior Tree remembers that active Task and its entire branch.
    - 当位于该阶段时, the BT system will **jump directly back to the active Task node** and continue to tick it 

### TreeTick vs GameTick
行为树本身可以**自定义 tick internal**, 每次tick时bt会从root点 (也可能是某个composite点)开始evaluate. 
- 当一个TaskNode处于running时, 该node会由UBehaviorComponent负责驱动来 **tick every frame**



