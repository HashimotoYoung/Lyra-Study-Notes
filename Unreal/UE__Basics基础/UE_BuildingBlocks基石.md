
## FAQ:

###### *1. Construction Script的作用 ?*

-  Used to define 构建相关 logic that runs at both edit time (主要用于编辑) and  **at runtime** (When you use a node like **Spawn Actor from Class** during gamepla, called before **begin play**)
<br>

###### *2. `UPORPERTY()` 的作用 ?*

让 UnrealEngine 能够**知道**该variable, 从而加入Reflection系统；
Make it can be **serialized, replicated, edited in the editor**, etc.
<br>

###### *3. GetTransientPackage()的作用 ?*

该方法会返回 **Transient** Package, which is a special **global UPackage object** in Unreal Engine, it acts as a **"dummy" ROOT outer** for temporary objects.

主要用于生成 temporary UObject , like dynamic materials, runtime textures, temporary components

- Transient objects are not saved to disk
- Automatically garbage collected when no longer referenced
- Perfect for runtime-generated content that doesn't need persistence


```cpp
// Transient - garbage collected when no references exist
UGameplayEffect* TempEffect = NewObject<UGameplayEffect>(GetTransientPackage());
// Lives until GC runs and no variables reference it

// Persistent - tied to owner's lifetime
UGameplayEffect* PersistentEffect = NewObject<UGameplayEffect>(this);
// Lives as long as 'this' object exists

// 空的Outer? 这种写法 is not recommanded in practice
// 没用被引用的话会被立即回收
UGameplayEffect* Effect = NewObject<UGameplayEffect>();
```
<br>

###### *4. 哪些Unreal类通常should be inherited, 哪些类通常not ?*
Should:
- GameInstance; GameState; GameMode; PlayerController; PlayerState; Pawn/Character;

Not:
- UWorld, ULevel
<br>

###### *5. How usually C++ calls BP function ?*
- 使用 `UFUNCTION(BlueprintNativeEvent)` / `UFUNCTION(BlueprintImplementableEvent)`
- 使用 `DynamicDelegate.BroadCast()`
- Call Blueprint Functions via Reflection (不建议)

|                                         | BlueprintNativeEvent              | BlueprintImplementableEvent    |
| ----------------------------------------------- | ------------------------------------- | ---------------------------------- |
| C++ implementation               | :white_check_mark: Can have  `_Implementation`  | :x:   |
| Blueprint override                | :white_check_mark: Can                                 | :white_check_mark: Can                            |


<br>

###### *6. 如何正确地 **Attach** 一个 `USceneComponent` ？*

主要有两种方式:

`Comp->SetupAttachment(ParentComp);`
- 一般用在 Construtor, 负责 set up the **CDO**
- 常用于通过 CreateDefaultSubobject 生成的组件

`Comp->AttachToComponent(ParentComp);`
- 一般用在 BeginPlay, 事件回调等方法中
- 常用于通过 NewObject 生成的组件

:warning: **After Attach**, 记得注册组件: `Comp->RegisterComponent()`


---
# UE Basics

## UObject 

**大部分情况下**, 新建的 Class 应该去继承 `UObject`或 `USTRUCT`, 原因如下:

1. Supports [Garbage Collection](./UE_GC_Pointer指针.md#ue-garbage-collector)
2. Serialization/Deserialization supported
3. :star: Networking/Replication: Unreal 的 **built-in replication** is **based on metadata**, which comes from unreal's **reflection system**, 而只有`UObject`和`USTRUCT`支持反射
4. Blueprint Integration
5. Editor Integration: 例如编辑默认值
6. **Class/Object Instantiation and Management:** The engine has robust systems for creating, finding, and managing UObjects. 内部支持比较完善

#### 核心方法:
#### `NewObject<T>() / NewObject<T>(UObject* Outer)`

- 每个UObject都会有一个**Outer**,当传空时会调用`Outer=GetTransientPackage()`

#### `CreateDefaultSubobject<T>()`
- **Purpose:** **can used only within a UObject's constructor** to create default subobjects (e.g., `UActorComponents` attached to an `AActor`).
- 该方法主要用于`UActorComponent`和其他 `UObject`, `CreateDefaultSubobject<AActor>` is a wrong usage :x:

- When `CreateDefaultSubobject()` is called on the CDO, it **creates a new subobject from scratch**.

- When `CreateDefaultSubobject()` is called on a runtime instance (or something created from an archetype):
	- It does not create a brand new subobject blindly.
	- It tries to find the subobject on the archetype (usually the CDO) and use it as a template to construct the new one, This is **how subobjects are cloned and initialized** correctly with their default values and properties.

---


# Class Default Object

一句话总结: CDO is effectively a **singleton** for each `UClass` that has all the **default values**  in it. It serves as an template for the class instances

- **基于UObject:** The CDO mechanic only exists for **`UObject Subclasses`**
- **创建时机:** 在所属Module被loaded时创建 (一般是 **PreInit** 阶段)
- `if(HasAnyFlags(RF_ClassDefaultObject))` 可用此判断区分CDO和普通实例

```cpp
class MyClass : UObject {
	UPROPERTY()
	int a = 5;
}

void MyClass:ReadCDO(){

	//get actual class, 也叫 most derived class
	//如果该方法 called from a instance of (SomeSubClass : MyClass), myUClass would be the SomeSubClass
	UClass* myUClass = GetClass();

	//most derived class, 并不一定是MyClass的CDO
	const MyClass* CDO = myUClass->GetDefaultObject<MyClass>();
	//可能 Print 5
	log(CDO->a)

	//是 MyClass 的CDO
	const MyClass* NoDerivedCDO = GetDefault<MyClass>();
	//Print 5
	log(NoDerivedCDO->a)
}
```

---

# Actor

### Actor's Life Cycle
![img](../_img/actorLifeCycle.png)

---

# UWorld

`UWorld` 包含所有Actors, it includes not just the visual geometry but also the game logic, physics simulation, audio, lighting...

- 当调用 `OpenLevel()` or `ServerTravel()`时:
  - :star: The old `UWorld` is destroyed and a new `UWorld` is created 
  - 在transition过程中会出现多个 `UWorld` 同时存在的情况. but most of the time only one UWorld stays active
<br>

- 值得细品: "While you don't switch UWorlds **directly**, you **indirectly** switch UWorlds by **loading a new level**". 例如调用`UGameplayStatics::OpenLevel(GetWorld(), "NewMapName");`
<br>

### Classes who "Live" on the UWorld 

Aside from `ULevel`, many **classes instance's lifetime** are closely tied to `UWorld` instance, include:

1. No `Actors` would survive without `UWorld`
2. **`PlayerController`**
3. **`GameMode`:** When a `UWorld` is created, the engine's game loop checks which `GameMode` class is assigned to the level **(from the World Settings or Project Settings)** and then automatically spawns a `GameMode` instance 
4. **`GameState`**
5. Active **network connections** for gameplay (e.g., player replication, RPCs, PlayerControllers) are indeed tied to the `UWorld` instance.
<br>

#### Examples of 切换 UWorlds

**1. Loading from a Main Menu to a Game Level:**
```
When you start the game, an initial UWorld for your main menu map is created.

When the player clicks "Start Game," you call OpenLevel().

//The engine then tears down the existing UWorld (and all its Actors, GameMode, GameState, etc.) for the main menu.

A brand new UWorld is created for your gameplay level.
```

**2. Transitioning Between Levels in a Game:**
```
When a player completes Level 1 and moves to Level 2, you again call OpenLevel().

The `UWorld` for Level 1 is destroyed.

A new `UWorld` for Level 2 is created.
```
**3. Returning to Main Menu:**
```
After a game match, when players return to the main menu, `OpenLevel()` is called for the main menu map.

The `UWorld` for the game match is destroyed.

A new `UWorld` for the main menu is created.
```
<br>

#### UWorld 的设计理念

- :star: **Clean State:** Each `UWorld` represents a ***Distinct Simulation Environment***. Destroying and recreating it ensures a clean slate, removing all previous Actors, components, and state from the old level. This prevents memory leaks, dangling pointers, and unexpected interactions from previous levels.

- **GameMode / GameState Changes:** As discussed, the `GameMode` and `GameState` are tied to the `UWorld` and are created/destroyed with it. This allows each level to enforce its own specific rules and state.

- **Memory Management:** It allows the engine to aggressively deallocate all memory associated with the previous level and load only what's needed for the new one.

---

# ULevel

Each `UWorld` has exactly **one Persistent Level** (`ULevel*`) which is always loaded, and an array of **Streaming Levels** (`TArray<ULevelStreaming*>`)


### Persistent Level
- :warning: There is no way to **explicitly remove or unload** the Persistent Level at runtime. Instead, you can call `UGameplayStatics::OpenLevel()` to load a new one
	- 这点和 Unity 的 Scene 有些类似
### Streaming Level
- can load/unload dynamically
- be referenced by Persistent Level
- :warning: StreamingLevel 是一个**相对概念**, 每个`.umap`文件对应了一份`ULevel`数据, 该`ULevel`可以被其他`ULevel`引用, 成为其StreamingLevel, 也可以自己作为 Persistent Level 去独立加载

#### :bulb: Tip
A recommended approach is to have a **dedicated, empty persistent level** that acts as a **level manager** to handle the loading and unloading of all the other content levels.

---

# FString/FText/FName

| Type      | 底层 | Use For                                                       | Localized? | Editable in Editor? | Fast?     | Comparable?  |
| --------- |---| ------------------------------------------------------------- | ---------- | ------------------- | --------- | ------------ |
| `FString` |`TArray<TCHAR>`| General, **唯一 Mutable** | ❌ No       | ✅ Yes               | ⚠️ Slower | ✅ Yes (slow) |
| `FText`   |TSharedRef + FString| User-facing, localized text (UI, dialog)  | ✅ Yes      | ✅ Yes               | ❌ Slowest | ⚠️ Complex   |
| `FName`   | index to a Global name table| Identifiers, asset names, enums, tags, **case-insensitive** | ❌ No       | ⚠️ Sometimes        | ✅ Fastest | ✅ Yes (fast) |

