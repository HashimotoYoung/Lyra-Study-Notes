# GAS Attribute

### 对比 `FGameplayAttributeData / FGameplayAttribute / FProperty`

:books: GAS中的 Attribute 概念在代码层主要由 `FGameplayAttributeData` 和 `FGameplayAttribute` 体现

**`FGameplayAttributeData`** is the actual Data Holder, 包含:
- **BaseValue:** 基准值
- **CurrentValue:** BaseValue + temporary modifications from GE 


**`FGameplayAttribute`** 可看作是一种轻量级的 属性标识符 (Identifier) 或 反射句柄 (Reflection Handle)。
- 主要用于配合反射获取具体的 `FGameplayAttributeData`

  ```cpp
  // 可通过 FGameplayAttribute 获取到 UAttributeSet
  const UAttributeSet* AttributeSet = ASC->GetAttributeSubobject(Attribute.GetAttributeSetClass());
  ```


**`FProperty`** is a core Unreal **reflection system class**, that represents a single C++ member variable (a "property") of a `UObject` or `UStruct`.
- 示例
    ```cpp
    // Example in a UAttributeSet.cpp
    // 这里 Health 是属性名称
    FGameplayAttribute UMyAttributeSet::GetHealthAttribute()
    {
        // Internally, it finds the FProperty for the Health member
        static FProperty* HealthProperty = FindFieldChecked<FProperty>(UMyAttributeSet::StaticClass(), GET_MEMBER_NAME_CHECKED(UMyAttributeSet, Health));
        return FGameplayAttribute(HealthProperty);
    }
    ```

---


## 通过 `ULyraHealthSet` 了解 AS

> :memo: The AttributeSet defines, holds, and manages changes to Attributes

#### :star: 类关系: 

- `UAttributeSet` is "Owned" by `ASC` **logically, not structurally**
  => :pushpin: AS.Outer 和 ASC.Owner 通常应当为同一个 Actor (例如`Character`,`PlayerState`)
    - 此 Actor 应继承 `IAbilitySystemInterface`, 并在构造函数中负责创建 AS
      ```cpp
      ALyraPlayerState::ALyraPlayerState(const FObjectInitializer& ObjectInitializer) 
      {
        ...
        CreateDefaultSubobject<ULyraHealthSet>(TEXT("HealthSet"));
        CreateDefaultSubobject<ULyraCombatSet>(TEXT("CombatSet"));
      }
      ```
      ```cpp
      // in UAttributeSet.h
      AActor* GetOwningActor() const;
      UAbilitySystemComponent* GetOwningAbilitySystemComponent() const;
       ```
    - `ASC::InitializeComponent()` 时会**查询 Siblings** 是否存在AS, 并注册到队列 `ASC.SpawnedAttributes` 中
  - One ASC can "own" Multiple AttributeSets, but you should **not** have more than one AttributeSet **of the same class**

#### 如何定义 Attribute:

- 通常在在自定义Set的.h中

```cpp
// in ULyraHealthSet.h
class ULyraHealthSet : public ULyraAttributeSet 
{
    ATTRIBUTE_ACCESSORS(ULyraHealthSet, Health);
    ATTRIBUTE_ACCESSORS(ULyraHealthSet, Damage);
protected:
    UFUNCTION()
	void OnRep_Health(const FGameplayAttributeData& OldValue);
private:
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Lyra|Health", Meta = (HideFromModifiers, AllowPrivateAccess = true))
    FGameplayAttributeData Health;

	UPROPERTY(BlueprintReadOnly, Category="Lyra|Health", Meta=(AllowPrivateAccess=true))
    FGameplayAttributeData Healing;
}
```
- 一些默认写法
```cpp
// in ULyraHealthSet.cpp
void ULyraHealthSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const {
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);
	DOREPLIFETIME_CONDITION_NOTIFY(ULyraHealthSet, Health, COND_None, REPNOTIFY_Always);
}

void ULyraHealthSet::OnRep_Health(const FGameplayAttributeData& OldValue) {
	GAMEPLAYATTRIBUTE_REPNOTIFY(ULyraHealthSet, Health, OldValue);
}
```

#### What is Meta Attribute ?

Meta Attribute 是一种**设计层**概念, 此类 Attribute 通常被设置为只在 Server 端起作用 (*Not Replicated*), 用于协助其他 Attribute 的计算, (例如 `FGameplayAttributeData Healing`)
  - :pushpin: Usually skip  `OnRep` and `GetLifetimeReplicatedProps` steps

---

#### **What `ATTRIBUTE_ACCESSORS(ULyraHealthSet, Health)` makes** ?
1. **static** `FGameplayAttribute` GetXXXAttribute() 方法
2. Getter for **CurrentValue** (`FGameplayAttributeData`)
3. Setter for **BaseValue**
   - 赋值时触发 `FActiveGameplayEffectsContainer::SetAttributeBaseValue(...)`
4. Initializer for Both Values
```cpp
// Macro 
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

// --- 生成部分 ----
static FGameplayAttribute GetHealthAttribute() {
    static FProperty* Prop = FindFieldChecked<FProperty>(ULyraHealthSet::StaticClass(), FName(TEXT("MemberName")));
    // 通过构造函数自动转换
    return Prop;
}

float GetHealth() const {
    return Health.GetCurrentValue();
}

void SetHealth(float NewVal) {
    UAbilitySystemComponent* AbilityComp = GetOwningAbilitySystemComponent();
    if (ensure(AbilityComp)) {
        AbilityComp->SetNumericAttributeBase(GetHealthAttribute(), NewVal);
    };
}

void InitHealth(float NewValue)
{
    Health.SetBaseValue(NewValue);
    Health.SetCurrentValue(NewValue);
}
```
<br>

#### What `GAMEPLAYATTRIBUTE_REPNOTIFY` makes ?
- 主要负责通知 Aggregator 更新 (调用此方法时 BaseValue 已经 Replicated)
```cpp
// 展开
{
static FProperty* ThisProperty = FindFieldChecked<FProperty>(ClassName::StaticClass(), GET_MEMBER_NAME_CHECKED(ClassName, PropertyName));
GetOwningAbilitySystemComponentChecked()->SetBaseAttributeValueFromReplication(FGameplayAttribute(ThisProperty), PropertyName, OldValue);
}
```

---

### 关键 virtual 方法梳理:

#### `PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)`
- **Before** change value, 常用于 Clamp NewValue
    ```cpp
    void UAuraAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) {
        Super::PreAttributeChange(Attribute, NewValue);
        if (Attribute == GetHealthAttribute()) {
            NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
        }
    }
    ```

#### `PostAttributeChange(const FGameplayAttribute& Attribute, float OldValue, float NewValue)`
- **After** change value, 常用于修正其它相关属性
    ```cpp
    void ULyraHealthSet::PostAttributeChange(const FGameplayAttribute& Attribute, float OldValue, float NewValue) {

        Super::PostAttributeChange(Attribute, OldValue, NewValue);

        if (Attribute == GetMaxHealthAttribute()) {
            // Make sure current health is not greater than the new max health.
            if (GetHealth() > NewValue) {
                ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent();
                LyraASC->ApplyModToAttribute(GetHealthAttribute(), EGameplayModOp::Override, NewValue);
            }
        }

        if (bOutOfHealth && (GetHealth() > 0.0f)) {
            bOutOfHealth = false;
        }
    }
    ```
<br>

#### `bool PreGameplayEffectExecute(FGameplayEffectModCallbackData &Data)`
- **调用时机:** Before `PreAttributeChange()`
  - :pushpin: Only triggers in [Execute GE 流](./GA_GE/GE_MainFlow流程.md#execute-ge-spec-流)  from an **Instant GE**

- :warning: **调用频率:** Apply 一个 GE Spec 时可能触发多次, 取决于 `GE.Modifiers` 和 `GE.Executions` 的数量 


#### `PostGameplayEffectExecute(FGameplayEffectModCallbackData &Data)`
- **调用时机:** After `PostAttributeChange()`
- 调用频率和 Pre方法 保持一致



