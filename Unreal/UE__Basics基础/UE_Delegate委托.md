
https://rawsourcecode.io/posts/unreal-delegates-under-the-hood

# UE Delegate


### Standard Delegates (标准委托) 
- 不支持蓝图
- 可绑定 Lambda
- 对参数类型有限制要求:
  - :heavy_check_mark: `int`, `bool`, `UObject*`, `UStruct(只包含简单类型)` 等
    - 按值传递时, 类型需要为 trivially-copyable
  - :x: `TArray`, `FString`, `UObject`, `UStruct(包含复杂类型)` 等 
    - 对于这些类型, **可使用 const reference**  

##### 单播示例:
```cpp
DECLARE_DELEGATE_OneParam(FOnScoreChangedSignature, int32 /* NewScore */);
// BindUObject requires that the target be a UObject
OnScoreChangedDelegate.BindUObject(this, &ThisClass::OnScoreChanged);
// BindRaw is for if the target is not a UObject
OnScoreChangedDelegate.BindRaw(SomeSlateThing, &SSlomeSlateThing::OnScoreChangedRaw);
// BindLambda is useful for simpler anonymous functions
OnScoreChangedDelegate.BindLambda([](int32 NewScore)
{
    // Do something with score
});
```
##### 多播示例:
```cpp
DECLARE_MULTICAST_DELEGATE_OneParam(FOnScoreChangedSignature, int32 /* NewScore */);
// AddUObject requires that the target be a UObject
OnScoreChangedDelegate.AddUObject(this, &ThisClass::OnScoreChanged);
// AddRaw is for if the target is not a UObject
OnScoreChangedDelegate.AddRaw(SomeSlateThing, &SSlomeSlateThing::OnScoreChangedRaw);
// AddLambda is useful for simpler anonymous functions
OnScoreChangedDelegate.AddLambda([](int32 NewScore)
{
	// Do something with score
});
```
---
### Dynamic Delegates (动态委托) 

`DECLARE_DYNAMIC_DELEGATE_OneParam`:

- 基于反射系统, 性能略差
- 支持蓝图
- **只可绑定 `UFUNCTION` 方法**
  - :x: 绑定参数不支持 非const reference

##### 单播示例:
```cpp
DECLARE_DYNAMIC_DELEGATE_OneParam(FOnScoreChangedSignature, int32, NewScore);

// If we have a UFUNCTION()-marked function `OnScoreChanged(int32 NewScore)
// we can subscribe using BindDynamic and the ThisClass macros
OnScoreChangedDelegate.BindDynamic(this, &ThisClass::OnScoreChanged);
```
##### 多播示例:
```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnScoreChangedSignature, int32, NewScore);

// If we have a UFUNCTION()-marked function `OnScoreChanged(int32 NewScore)
// we can subscribe using AddUniqueDynamic and the ThisClass macros
OnScoreChangedDelegate.AddUniqueDynamic(this, &ThisClass::OnScoreChanged);
```

---

## UE Delegate Features

### 1. 可使用 `struct` 绑定 `TMap` 等类型

```cpp
USTRUCT(BlueprintType)
struct FScoreData
{
    UPROPERTY(BlueprintReadWrite)
    TMap<FName, int32> ScoreMap;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnScoreChangedSignature, FScoreData, ScoreData);

UCLASS()
class ABUIPlayerState : public APlayerState
{
public:
    UPROPERTY(BlueprintAssignable)
    FOnScoreChangedSignature OnScoreChangedDelegate;
};
```
<br>

### 2. Bind Callback with Payload 
可以绑定**多参方法**到**单参Delegate**
```cpp
struct FExample
{
    void TestCallbackWithExtraParam ( float F1, float F2 ){
        //do something
    }
};

DECLARE_DELEGATE_OneParam( DelegateType, float );
DelegateType TheDelegate;

TSharedRef Example = MakeShared<FExample>();
TheDelegate.BindSP( Example, &FExample::TestCallbackWithExtraParam, 2.0f );
TheDelegate.ExecuteIfBound(1.0f);
```
##### 另一个示例:
```cpp
DECLARE_DELEGATE_TwoParams(FMyDelegate, int A, int B);
FMyDelegate D = FMyDelegate::CreateUObject(this, &Class::Func, 999);

void Func(int A, int B, int ExtraParam);
```
<br>

### 3. `::FDelegate`自动生成
当声明 **multi-cast type 委托** 时, Unreal 会自动提供一个 **nested single-cast type 版本**(`::FDelegate`)
```cpp
DECLARE_MULTICAST_DELEGATE_OneParam(FOnLyraExperienceLoaded, const ULyraExperienceDefinition* /*Experience*/);

//约等价于:
struct FOnLyraExperienceLoaded
{
    struct FDelegate
    {
        void BindUObject(...);
        void Execute(...);
    };

    void Add(FDelegate&&);
    void Broadcast(...);
};
```
```cpp
// in ULyraBotCreationComponent.cpp
void ULyraBotCreationComponent::BeginPlay()
{
  ...
  ExperienceComponent->CallOrRegister_OnExperienceLoaded_LowPriority(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}

// in ULyraExperienceManagerComponent.cpp
void ULyraExperienceManagerComponent::CallOrRegister_OnExperienceLoaded_LowPriority(FOnLyraExperienceLoaded::FDelegate&& Delegate)
{
	if (IsExperienceLoaded())
	{
		Delegate.Execute(CurrentExperience);
	}
	else
	{
        //FOnLyraExperienceLoaded OnExperienceLoaded_LowPriority
		OnExperienceLoaded_LowPriority.Add(MoveTemp(Delegate));
	}
}
```
<br>

### 4. What is `FDelegateHandle`
- A **uint64 Global ID** returned when bind a function
- Identifies a *specific delegate binding* , 而非 delegate instance

- 主要用于让 **standard-multicast-delegate** 解除指定 binding

```cpp

DECLARE_MULTICAST_DELEGATE(FMyMulticastDelegate); 

FMyMulticastDelegate MyMulticastEvent;
FDelegateHandle MyHandle = MyMulticastEvent.AddUObject(this, &UMyObject::MyFunction); 

MyMulticastEvent.Remove(MyHandle);
```
---

### TFunction
更轻量化的 delegate, 常作为参数类型使用
##### 使用示例:
```cpp
// 声明 variable
TFunction<bool(int32, int32)> FunctionTakingTwoIntsAndReturningABool = nullptr;

// 绑定static方法: Assigning a pointer to a static function:
const TFunction<void()> StaticFunctionReference = &AMyClass::MyStaticFunction;

// 绑定实例回调: Assigning a "pointer" to a non-static member function (done via a lambda):
const TFunction<void()> NonStaticFunctionReference = [WeakThis = TWeakObjectPtr<AMyClass>(this)]()
{
	if(WeakThis.IsValid())
	{
		WeakThis->MyNonStaticFunction();
	}
};

// Assigning a lambda function:
const TFunction<void()> LambdaFunction= [](){};

// Examples of invoking TFunction<>:
StaticFunctionReference();

const TFunction<void()>& SelectedFunction = FMath::RandBool() ? NonStaticFunctionReference : LambdaFunction;
SelectedFunction();

(FMath::RandBool() ? NonStaticFunctionReference : LambdaFunction).operator()();

// Using TFunction<> type as an argument in a function prototype:
void MyFunction(TFunction<bool(int32, int32)> Argument);
```