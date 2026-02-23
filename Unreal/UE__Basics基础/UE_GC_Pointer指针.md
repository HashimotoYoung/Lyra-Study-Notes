# UE Garbage Collector
Unreal's GC **Only Cares About UObjects**

- Only member-variables marked with **`UPROPERTY()`** are under GC Tracking

- **For Actors, the GC does not destroy them immediately**. Instead, actors are marked for deletion when they are no longer referenced or explicitly destroyed via methods like `Destroy()`
  - 这是一个 Design choice for fully clean up


## GC 执行流程
基础原则: the GC primarily cares about the **Reachability** of UObject instances

1. **从 Roots 出发:** The GC begins by identifying a set of *root* UObjects. (例如: `UGameInstance`, 已加载的 `ULevels`...)

2. **遍历 References 并 "Mark" (标记阶段):** From these roots, GC follows `UObject*` references and `UPROPERTY()` references to find all reachable objects. includes:
    - Direct `UPROPERTY()` `UObject*` pointer
    - Objects in `TArray<UObject*>`, `TMap`, etc 
    - Hard references through asset references
   > As it encounters each UObject, it marks it as *reachable*.

3. **确认 "Unmarked" Objects (扫荡阶段):** GC scans all `UObjects` in memory. Any unmarked `UObject` (**no strong `UObject` reference path led to it from any root**) is considered non-reachable and thus eligible for garbage collection.

4. **释放:** These non-reachable `UObjects` are then systematically destroyed, and their memory is reclaimed.

---

# UE Pointers 指针

### TUniquePtr / TSharedPtr / TWeakPtr
Used for **non-UObject classes and structs allocated with `new` or `MakeShared/MakeUnique`**
- Cannot be marked with `UPROPERTY()` (has compile Error)

- `TUniquePtr` **不可复制**, 可以Move
    ```cpp
    //比较常用的c++写法,返回时触发Move机制
    std::unique_ptr<Foo> createFoo() {
        return std::make_unique<Foo>();
    }
    ```
- `TWeakPtr` 依赖于 `TSharedPtr` 生成
    ```cpp
    TSharedPtr<FMyClass> MySharedPtr = MakeShared<FMyClass>(); // Control block created
    TWeakPtr<FMyClass> MyWeakPtr(MySharedPtr);     
    ```
- 三者关系:
  - :heavy_check_mark: *TSharedPtr + TWeakPtr* 
  - :x: *TUniquePtr + TSharedPtr*, *TUniquePtr + TWeakPtr*


#### `TSharedRef`:
- TSharedRef works like a **Raw Reference** (`MyObject&`)
- 和TSharedPtr基本一致,but ensures **validity** (cannot be NULL)

<br>

###### :mag:  Shared Pointer 底层逻辑: 

When the first `TSharedPtr` is created for an object (often via `MakeShared`), Unreal allocates a small, separate data structure like a **`Control Block`**：
- when `SharedRefCount==0`, **`TheActualObjPtr`** 对象被销毁
- when `SharedRefCount==0 && WeakRefCount==0`, **`ControlBloack`** 自身被销毁 
```cpp
[TSharedPtr<T>] --指向--> [Control Block] //分配在heap上
[TWeakPtr<T>] --指向-->   {
                            T* TheActualObjPtr;
                            int SharedRefCount;
                            int WeakRefCount;
                          }                    
```
---

### TObjectPtr / TWeakObjectPtr

:books: 类似于C#, UObject 的 **Ownership** 完全由UE GC系统管理, `TObjectPtr<>` is for **STRONG** referencing (防止 UObj 被回收), 常搭配 `TWeakObjectPtr` 一起使用
- `TObjectPtr` 在打包时会被自动替换为 C++ rawPointer
- Dont manually destroy a UObject **referenced by `TObjectPtr`**, 只需将 ptr 设为 null
- Owner's destruction clears `UPROPERTY` pointers, 避免产生 *dangling pointers*:
    ```cpp
    UCLASS()
    class MYGAME_API UObjectA : public UObject
    {
        GENERATED_BODY()
    public:
        // NO UPROPERTY() - not automatically cleared
        TObjectPtr<UObjectB> MyPtr;  
        // HAS UPROPERTY() - This ptr will be cleared when ObjectA is destroyed
        UPROPERTY()
        TObjectPtr<UObjectB> MyPtr2;  
    };
    ```
- :memo: if a UObject is only referenced by a `TWeakObjectPtr` 或 a raw Pointer (no `UPROPERTY()`), it will be GCed as soon as the next GC pass runs.
<br>

#### 比较 TWeakObjectPtr / FWeakObjectPtr / FObjectKey
`FObjectKey` is a **Persistent Identifier** for a UObject
- 常用于作为TMap中的 key, 
- 非指针, 不影响 GC
- 通过 `::ResolveObjectPtr()` 方法获取 Obj
    ```cpp
    FObjectKey(int32 Index, int32 Serial): ObjectIndex(Index), ObjectSerialNumber(Serial)
    { }
    int32   ObjectIndex;
    int32	ObjectSerialNumber;

    //底层使用 FWeakObjectPtr 获取UObject
    UObject* ResolveObjectPtr() const {
        FWeakObjectPtr WeakPtr;
        WeakPtr.ObjectIndex = ObjectIndex;
        WeakPtr.ObjectSerialNumber = ObjectSerialNumber;
        return WeakPtr.Get();
    }
    ```

`TWeakObjectPtr` is a *Non-Owning, templated, weak* pointer
- 使用目的: **safe observation** and **breaking cyclic references**
- 本质是一个 **Mordern Wrapper** of `FWeakObjectPtr`, to provide:
  - Type safety (`T*` instead of raw `UObject*`)
  - Helper funcs like `IsValid(),Get(),Pin()`
---

### `struct A : public TSharedFromThis<A>` 用法

Inheriting `TSharedFromThis<FStreamableHandle>` allows `FStreamableHandle` to **create a safe `TSharedPtr` to self** (by using `SharedThis(this)`), so it can be passed around without risking double delete or invalid memory access. 

#### Why 不能直接返回 `TSharedPtr<FStreamableHandle>(this)`?

- Suppose a object is already owned by some `TSharedPtr`,
  If inside the object you do: `return TSharedPtr<FStreamableHandle> SelfPtr(this)`,
  :x: This creates a new, separate reference count. When both shared pointers go out of scope, **the object would be deleted twice (*Double Delete*)**.


#### How `TSharedFromThis` solves it

If the class inherits from `TSharedFromThis<FStreamableHandle>`, then inside the class you can call:
- `TSharedPtr<FStreamableHandle> SelfPtr = AsShared()`

- :warning: for class that inherits from `TSharedFromThis`, you **must create an instance of that class using `MakeShared`**, otherwise the `AsShared()` func will cause error.

---

## Pointers 相关示例:

```cpp
class MYGAME_API AMyActor : public AActor
{
private:
    //good PREVENTS GC - strong reference with UPROPERTY
    //特性: Serialized, replicated, Blueprint-exposed
    UPROPERTY()
    TObjectPtr<UStaticMesh> StrongRef;

    //这个用raw pointer是允许的, 因为是UObject*, 用TObjectPtr更好
    UPROPERTY()
    UStaticMesh* Mesh;

    //good ALLOWS GC - weak reference
    UPROPERTY()
    TWeakObjectPtr<UStaticMesh> WeakRef;

    //good, selfControl
    TSharedPtr<MyPlainClass> ptr;

    //------------------good/bad 分界线----------------------

    // BAD: ALLOWS GC - 容易造成Dangling Pointer
    // 这个可能是最容易犯的错 ！！！！！ No UPROPERTY
    TObjectPtr<UStaticMesh> InvisibleRef;
    UStaticMesh* RawRef;

    // BAD: UPROPERTY 不支持 非UObject-derived class 
    UPROPERTY()
    MyPlainClass* ref;
    UPROPERTY()
    MyPlainClass inst;

    // BAD: This will cause compilation errors
    TObjectPtr<MyPlainClass> ptr;
    TSharedPtr<AActor> ActorPtr;    
    TSharedPtr<UMyCustomClass> CustomPtr;  
};

// 这种写法会被 GC, 如果需要保持, 要将MyObj->AddRootSet()
static UObject* MyObj = NewObject<UMyObject>();
```
<br>

### :warning: 常见错误示例:

```cpp
void ManualDestructionDanger()
{
    UPROPERTY()
    TObjectPtr<UMyObject> StrongRef;

    UMyObject* RawRef;  // No UPROPERTY
    
    UMyObject* MyObject = NewObject<UMyObject>();

    StrongRef = MyObject;
    RawRef = MyObject;
    
    // Manually force destruction
    MyObject->MarkAsGarbage();
    GEngine->ForceGarbageCollection(true);
    
    // StrongRef becomes safely null
    // RawRef becomes dangling pointer!
    
    if (StrongRef.IsValid()) { /* Safe - will be false */ }
    if (RawRef != nullptr) { /* DANGER - might still be true! */ }
}
```
BadExample: Component references after owner destruction
```cpp
class MYGAME_API AMyActor : public AActor
{
private:
    UStaticMeshComponent* ComponentPtr;  // Raw pointer to component
    
public:
    void StoreComponentReference(AActor* OtherActor)
    {
        // Get component from another actor
        ComponentPtr = OtherActor->FindComponentByClass<UStaticMeshComponent>();
    }
    
    void UseComponent()
    {
        // If OtherActor was destroyed, ComponentPtr is dangling
        ComponentPtr->SetVisibility(false);  // CRASH!
    }
};
```
BadExample: Callback on destoryed
```cpp
class MYGAME_API ATargetActor : public AActor
{
public:
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnTargetDestroyed, AActor*, Target);
    
    UPROPERTY(BlueprintAssignable)
    FOnTargetDestroyed OnDestroyed;
    
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override
    {
        OnDestroyed.Broadcast(this);  // Broadcasts raw pointer
        Super::EndPlay(EndPlayReason);
    }
};

class ASomeOtherActor: public AActor {

    // Somewhere else - storing the raw pointer from delegate
    // 假设这是被Broadcast的对象
    void OnTargetDestroyed(AActor* DestroyedActor)
    {
        LastDestroyedActor = DestroyedActor;  // Raw pointer storage
    }

    void LaterCall(){
      // Later usage - DANGER!
        LastDestroyedActor->GetName();  // Dangling pointer!
    }
}
```

