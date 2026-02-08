# UCLASS 

## USTRUCT 宏
> :warning: 指标记了 **USTRUCT宏的c++ struct**

UStruct in UE are designed to be **data containers**, not objects with behavior.
- `UStruct` 内部可声明`UPROPERTY`, 但**不允许声明** `UFUNCTION`
- **UStructs are considered value types and are not garbage collected**, their lifetime is tied to the `UObject` that contains them
- Cannot be spawned/instantiated directly in the world: You can't just `new USTRUCT()`
- Do not have a `UWorld` pointer


- UStructs ARE NOT considered for replication. However, UProperty variables ARE considered for replication. 怎么理解?


---
## class UClass
- 继承自UStruct
- 类似于C#中的System.Type

