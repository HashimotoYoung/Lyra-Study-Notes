
## Gameplay Effect Context 

---

### FGameplayEffectContext

GE上下文类, **鼓励继承**

##### 类关系:

- 配合句柄: `FGameplayEffectContextHandle` (TSharedPtr型)
- 创建时机: 通常随 GESpec 一起生成
  > :star: It is common to share **One** `FGameplayEffectContext` with **Multiple** `FGameplayEffectSpec`, when multiple effects **originate from the same action**
  ```cpp
  // 示例
  FGameplayEffectSpecHandle UAbilitySystemComponent::MakeOutgoingSpec(TSubclassOf<UGameplayEffect> GameplayEffectClass, float Level, FGameplayEffectContextHandle Context) const
  {
		SCOPE_CYCLE_COUNTER(STAT_GetOutgoingSpec);
		if (Context.IsValid() == false)
		{
			// 此处创建一个Context 并返回其 Handle
			Context = MakeEffectContext();
		}

		if (GameplayEffectClass)
		{
			UGameplayEffect* GameplayEffect = GameplayEffectClass->GetDefaultObject<UGameplayEffect>();

			FGameplayEffectSpec* NewSpec = new FGameplayEffectSpec(GameplayEffect, Context, Level);
			return FGameplayEffectSpecHandle(NewSpec);
		}

		return FGameplayEffectSpecHandle(nullptr);
  }
  ```

##### 主要属性:

`TWeakObjectPtr< AActor > Instigator`: 发起者
  - 指代 The **OwnerActor** of ASC (例如 Lyra 中的 `LyraPlayerState`)

`TWeakObjectPtr< AActor > EffectCauser` : 施法者
  -  指代 The **AvatarActor** of ASC (例如武器,飞弹, Lyra 中的 `ALyraCharacter`)

`TWeakObjectPtr< UObject > SourceObject` : GE源头 
  - 如果Context由GA生成, 则默认实现为 `AbilitySpec->SourceObject` (例如 Lyra 中的 `UWeaponInstance`)
    - :pushpin: The SourceObject of GA 通常指代 **the UObject who grants the GA to ASC**

`TArray<TWeakObjectPtr<AActor>> Actors` // 所有涉及到的Actors ?

`TSharedPtr<FHitResult> HitResult` //只有一个 HitResult ?
<br>

#### About `FGameplayEffectContext` Subclasses

- 需要 Override `GetScriptStruct()`
- 需要 Override `Duplicate()`: 以便能够对特定属性 **DeepCopy**
- 需要 Override `NetSerialize()`: 如果新增属性需要同步
- Implement `TStructOpsTypeTraits`
- Override `AllocGameplayEffectContext()`
- 这里 *NewContext = *this 是一种常用的**ShallowCopy**写法,指针本身会被复制,从而指向同一个instance




