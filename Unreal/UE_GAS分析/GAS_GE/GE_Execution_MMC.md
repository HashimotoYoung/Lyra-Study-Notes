## FAQ:

---

## 1. GE Executions (属性计算器)

**Executions** decide how to change it *(code-driven)*.
> Calculating damage received based on a complex formula reading from many attributes on the Source and the Target

- Only be used **with `Instant/Periodic` GE**
- Only **Calls on Server**, 因为需要负责处理复杂属性计算
- **关于 MMC/Capture/ExecCalc:** [Topic](./GAS_GEExecutionCalculation.md)

---

## UGameplayEffectCalculation 

Generally, GE会**先Process** Modifiers list(MMC) , 然后**再Execute**(ExecCalc). `ExecutionCalculations` can **only** be used with `Instant GE`，and are **ignored on** `Duration/Infinite GE`, unless they have a `Periodic effect` configured.
- **单例工具类(CDO)**
- This recalculation will not run `PreAttributeChange()` in the `AttributeSet` so any **clamping** must be done here again.
- 功能强大,可以直接 **Modify multiple attributes at once**


`UGameplayEffectCalculation.h` 本体很轻量化:
```cpp
UCLASS(BlueprintType, Blueprintable, Abstract)
class GAMEPLAYABILITIES_API UGameplayEffectCalculation : public UObject
{
	GENERATED_UCLASS_BODY()
public:
	/** Simple accessor to capture definitions for attributes */
	virtual const TArray<FGameplayEffectAttributeCaptureDefinition>& GetAttributeCaptureDefinitions() const;
protected:
	/** Attributes to capture that are relevant to the calculation */
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Attributes)
	TArray<FGameplayEffectAttributeCaptureDefinition> RelevantAttributesToCapture;
};
```
#### 一个 Implementation 示例

Unreal 建议 ExecCalc 类采用 **"static member function + static local struct"** pattern 来进行属性绑定, 这样做的好处有:

- Initialization on First Use (Lazy Initialization), **节省内存**
- **避免污染 Global**, 外界看不到`struct FDamageStatics` 和 `static const FDamageStatics& DamageStatics()` (C++的Internal Linkage)
```cpp

// DamageExecutionCalculation.h
#pragma once

#include "CoreMinimal.h"
#include "GameplayEffectExecutionCalculation.h"
#include "DamageExecutionCalculation.generated.h"

UCLASS()
class YOURGAME_API UDamageExecutionCalculation : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()
public:
    UDamageExecutionCalculation();
    virtual void Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;
};

// DamageExecutionCalculation.cpp
#include "DamageExecutionCalculation.h"
#include "AbilitySystemComponent.h"
#include "YourAttributeSet.h" // Replace with your actual attribute set

// Define capture definitions
struct FDamageStatics
{
    DECLARE_ATTRIBUTE_CAPTUREDEF(AttackPower);
    DECLARE_ATTRIBUTE_CAPTUREDEF(Defense);

    FDamageStatics()
    {
        // Capture AttackPower from Source (attacker)
        DEFINE_ATTRIBUTE_CAPTUREDEF(UYourAttributeSet, AttackPower, Source, false);
        // Capture Defense from Target (defender)
        DEFINE_ATTRIBUTE_CAPTUREDEF(UYourAttributeSet, Defense, Target, false);
    }
};

static const FDamageStatics& DamageStatics()
{
    static FDamageStatics DStatics;
    return DStatics;
}

UDamageExecutionCalculation::UDamageExecutionCalculation()
{
    // Register the attributes we want to capture
    RelevantAttributesToCapture.Add(DamageStatics().AttackPowerDef);
    RelevantAttributesToCapture.Add(DamageStatics().DefenseDef);
}

void UDamageExecutionCalculation::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, OUT FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    UAbilitySystemComponent* TargetAbilitySystemComponent = ExecutionParams.GetTargetAbilitySystemComponent();
    UAbilitySystemComponent* SourceAbilitySystemComponent = ExecutionParams.GetSourceAbilitySystemComponent();

    AActor* SourceActor = SourceAbilitySystemComponent ? SourceAbilitySystemComponent->GetAvatarActor() : nullptr;
    AActor* TargetActor = TargetAbilitySystemComponent ? TargetAbilitySystemComponent->GetAvatarActor() : nullptr;

    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

    // Get captured attribute values
    float AttackPower = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().AttackPowerDef, FAggregatorEvaluateParameters(), AttackPower);

    float Defense = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().DefenseDef, FAggregatorEvaluateParameters(), Defense);

    // Simple damage calculation: AttackPower - Defense
    float DamageDealt = FMath::Max(0.0f, AttackPower - Defense);

    // You can also get set by caller magnitudes
    float BonusDamage = 0.0f;
    if (Spec.GetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag(FName("Data.Damage")), BonusDamage))
    {
        DamageDealt += BonusDamage;
    }

    // Apply the damage to health
    if (DamageDealt > 0.0f)
    {
        OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(DamageStatics().HealthProperty, EGameplayModOp::Additive, -DamageDealt));
    }
}
```

#### When `Execute_Implementation` called?
1. When a `GameplayEffect` is **executed(执行时)**
2. When a `Duration/Infinite GameplayEffect` **"ticks (apply periodic effect)"**, if period configured

---


## UGameplayModMagnitudeCalculation (MMC)

继承自 **`UGameplayEffectCalculation`**, 该类的唯一职责是: return a float from `CalculateBaseMagnitude_Implementation()`

- **工具类**, 但功能相比父类有限制 
- MMC (当然父类也可以) has the capability to Capture the value of Attributes. Attributes can either be snapshotted or not.
- :star: 作为Modifier, 能够 Capture **multiple** Attributes, 但只能 Affect **One** Attribute (指returned float)

#### MMC 应用示例:
```cpp
//.h
//在头文件中定义要Capture的属性
//一般用static,表明 this is a shared singleton
static FGameplayEffectAttributeCaptureDefinition ManaDef;
static FGameplayEffectAttributeCaptureDefinition MaxManaDef;

//.cpp
//Constructor中进行Capture
UPAMMC_PoisonMana::UPAMMC_PoisonMana()
{
	//绑定属性
	ManaDef.AttributeToCapture = UPAAttributeSetBase::GetManaAttribute();
	//标明要用 Who's ASC
	ManaDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
    //
	ManaDef.bSnapshot = false;

	MaxManaDef.AttributeToCapture = UPAAttributeSetBase::GetMaxManaAttribute();
	MaxManaDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
	MaxManaDef.bSnapshot = false;

	RelevantAttributesToCapture.Add(ManaDef);
	RelevantAttributesToCapture.Add(MaxManaDef);
}

float UPAMMC_PoisonMana::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec & Spec) const
{
	// Gather the tags from the source and target as that can affect which buffs should be used
	const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
	const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

	FAggregatorEvaluateParameters EvaluationParameters;
	EvaluationParameters.SourceTags = SourceTags;
	EvaluationParameters.TargetTags = TargetTags;

	float Mana = 0.f;
	GetCapturedAttributeMagnitude(ManaDef, Spec, EvaluationParameters, Mana);
	Mana = FMath::Max<float>(Mana, 0.0f);

	float MaxMana = 0.f;
	GetCapturedAttributeMagnitude(MaxManaDef, Spec, EvaluationParameters, MaxMana);
	MaxMana = FMath::Max<float>(MaxMana, 1.0f); // Avoid divide by zero

	float Reduction = -20.0f;
	if (Mana / MaxMana > 0.5f)
	{
		// Double the effect if the target has more than half their mana
		Reduction *= 2;
	}
	
	if (TargetTags->HasTagExact(FGameplayTag::RequestGameplayTag(FName("Status.WeakToPoisonMana"))))
	{
		// Double the effect if the target is weak to PoisonMana
		Reduction *= 2;
	}

	return Reduction;
}
```
#### :star: When `CalculateBaseMagnitude_Implementation` is called?
可理解为等价于Add **FixedFloat** 的时机, 具体为:
- When a `GameplayEffect` is **applied(附加时)**
