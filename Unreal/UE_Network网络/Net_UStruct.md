## UStruct 分析

> :books: `UStruct` 是和 `UObject` 处于同一层级, 但相对轻量化的数据结构
- `UStruct` 支持 UPROPERTY(), 意味着其成员变量可参与:
  - Reflection, Serialization, Replication, Blueprint exposure
- **`UStruct` 不支持 UFUNCTION()**

---

### UStruct vs UObject 内存管理:

#### UObject:
- 当创建 `UObject` 时, it will be registerd into GC系统
  - 需使用下面两种方式进行创建
    - `NewObject<T>()`
    - `CreateDefaultSubobject<T>()` (Constructor Only)
  - `UObject` is reference type, so it is strictly **Heap-allocated**
- Unlike C#, `UObject` uses **Fixed Address Stability**, means:
  - 无法直接作为成员变量: :x: `UMyObject someObj` 
  - 无法以Value形式持有: :x: `TArray<UMyObject>`

#### UStruct:
GC系统 **dont care** about `UStruct`
- :warning: 因此当 `UObject*` 作为成员变量时, 通常需要标记 `UPROPERTY()`, 否则 UE 会 "看不到" 该指针, 容易导致 "Dangling pointer"  


---

### UStruct 网络同步:

**必要条件** for Replicating a UStruct

1. UPROPERTY(Replicated)
   ```cpp
   // AMyActor.h
   UPROPERTY(Replicated)
   FMyStruct Data;
   ```
2. 在 `GetLifetimeReplicatedProps()` 中进行注册
   ```cpp
   void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
   {
		Super::GetLifetimeReplicatedProps(OutLifetimeProps);
		DOREPLIFETIME(AMyActor, Data);
   }
   ```

3. **选择一种同步方式:**

   1. **Standard:** 单纯把 UStruct 中需要同步的 member properties 标记为 `UPROPERTY()`
      - :warning: `UPROPERTY(Replicated)` 不支持用在  USTRUCT 中
      > :memo: 此方式下, Structs **replicate atomically**. This means if you change `Data.Count` on the server, the engine sends the entire struct to the client.
   
   2. **With Net Serializer:** 
      - 可通过添加 *Trait* 后, 实现 `NetSerialize` 方法, **自定义网络序列化** 
        - `bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)`
      - 用于实现各种优化, 例如 delta replication, data compression
      - `UPROPERTY()` is no longer required for members
      ```cpp
      
      // .h
      template<>
      struct TStructOpsTypeTraits<FContextualAnimSceneBindings> : public TStructOpsTypeTraitsBase2<FContextualAnimSceneBindings>
      {
        enum
        {
          WithNetSerializer = true
        };
      };
      ```





