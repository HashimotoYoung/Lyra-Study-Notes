## 1. Main Hierarchy

### Outer

描述 `UObject` 与 `UObject` 间的层级关系

- Every `UObject` **must** have an Outer
  - "root" Outers 为各种 `UPackage` objs ,例如: `Level Package`, `Asset Package`, `Transient Package`
  - 如果创建 UObject 时没有显示指定 Outer, 则默认为 `Transient Package`
- **唯一名称:** `UObject`'s name must be **Unique within its Outer**

- :pushpin: 主要作用:
  - Garbage Collection
  - Serialization
  - Reflection System

#### 典型 Outer 关系:
- `AActor` => `ULevel` (99%的情况下)
- `UActorComponent` => `AActor`  
  - :warning: `USceneComponent`'s Outer 通常为 `AActor`, `USceneComponent` 之间则通过 Attachment 建立父子关系
- `UStaticMesh`, `UTexture`, `UMaterial`, `USoundWave`, `UAnimSequence`, etc. => `UPackage`
  - `UMaterialInstanceConstant` / `UMaterialInstanceDynamic` => `UPackage` (for saved instances) or `UMaterial` (for dynamic/runtime instances)
- `UClass, UFunction, UProperty, UEnum, UStruct` => `UPackage`

<br>

#### Subobjects  :mag:

- 定义: A Subobject is a `UObject` whose `Outer` is not a `UPackage`, but another `UObject`.
  - :warning: `Actor` 是个例外, it is never counted as Subobject

---
### Owner (Actor)

Actor层新增概念, 描述 `AActor` 与 `AActor` 间的归属关系
```cpp
// in Actor.h
UPROPERTY(ReplicatedUsing = OnRep_Owner)
TObjectPtr<AActor> Owner;
```
- An actor's Owner can be null
- :pushpin: 主要作用: 
  - **RPC, Relevancy** (`bNetUseOwnerRelevancy`, `bOnlyRelevantToOwner`)
  - **Visibility** (`bOwnerNoSee`,`bOnlyOwnerSee`)
  - 详见 [Ownership](../UE_Network网络/_Network总览.md/#5-ownership)

#### 典型 Owner 关系:

The Owner **varies greatly** depending on the type of Actor and its role:

- 自己的 `APlayerState` => `AIController/PlayerController` 

- Weapons or Inventory Items => `APawn/ACharacter` that is holding or carrying them.

- Projectiles => `APawn/ACharacter` that fired them.

- Actors spawned from `UChildActorComponent` => Owned by the AActor that contains the `UChildActorComponent`.

- Actors spawned dynamically => When you call `UWorld::SpawnActor()`, you can explicitly pass an `AActor*` as the Owner parameter. 

<br>

#### Locally Controlled 

> :books: 指代 "是否受 **本地Machine** 控制":
> 在 DS 端, actors controlled by `AIController` are considered as locally controlled, and any actor controlled by `PlayerController` are not

- **从 `APawn` 开始**出现了 Is Locally Controlled 的概念
  ```cpp
  bool APawn::IsLocallyControlled() const {
    return ( Controller && Controller->IsLocalController() );
  }
  ```
- :pushpin: Control链 一般基于 Owner 实现
  ```cpp
  bool AMyWeapon::IsLocallyControlled() const {
      APawn* OwnerPawn = Cast<APawn>(GetOwner());
      return OwnerPawn && OwnerPawn->IsLocallyControlled();
  }
  ```

#### Possessed By

Each Controller is designed to possess one Pawn at a time.

- 当玩家进入汽车驾驶后，通常会调用 `PlayerController::Possess()` 来控制汽车：
  - 汽车的 LocalRole 会升级为 `AutonomousProxy`
  - 原先 Pawn 会被降级为 `SimulatedProxy`

`virtual void APawn::PossessedBy(AController* NewController)`
// 核心"绑定"函数, 只在 Server 调用:
- SET `APawn.Owner = NewController`
- SET `APawn.Controller = NewController`
- SET `APawn.PlayerState = NewController->PlayerState`

---

## 2. Misc Hierarchy

UActorComponent's `AActor* OwnerPrivate` => AActor

Actor-to-Actor Attachment

Child Actor Component (`UChildActorComponent`)

Manager/Subsystem Patterns

