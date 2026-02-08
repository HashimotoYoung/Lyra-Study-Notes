todo: GE Stack, tag

# GAS GE 总览

:books: GameplayEffect 代表GAS中的逻辑层效果, 有四个主要作用:

- Modify **Attributes** through *Modifiers and Executions*
- Add/Remove **`FGameplayTag`** to the target `ASC`
- Grants **GA** to the target `ASC`
- Invok **Gameplay Cue**


---

### GE 主要按 *持续类型* 分为三种

#### Instant GE (瞬时 / "永恒") 

1. Apply Permanent changes to **Attribute's `BaseValue`**.
2. Can invoke `Execute` event on the GC
3. **Cannot** grant `GameplayTags` and `GameplayAbilities` , not even for one frame. 
4. Dont persist in memory

#### Duration GE (时限) 和 Infinite GE (永久) 

Duration GE 可看作是有时间限制的 Infinite GE (which never expire on their own and must be manually removed by an Ability or the ASC)

1. Apply Temporary changes to **Attribute's `CurrentValue`**. 
2. Can invoke `Add/Remove` event on the GC
3. Can be temporarily **turned off / on** if their Ongoing Tag Requirements are not met
4. :pushpin: 可通过配置 Period 字段 to apply **Periodic GE**:
   - Periodic GE is essentially treated like **Instant GE**
   - Can not be Predicted

---


### The Source and Target of GE

> [GE Context 介绍](./GE_Context.md)

The Source and Target of a Gameplay Effect refer to the **Actors involved in its application**. 


- For a **successful applied** GE, target 是**必然存在**的
- Source is optional, but NULL source is **highly discouraged and limits functionality**
- `GameplayEffectSpec` 一般会通过 `GameplayEffectContext` 来获取Source/Target

:warning: **注意:** 对于GE来说, Source/Target更多是指**概念上**的, 在**代码层面**并没有一个绝对意义上的Source

以下三种常被当作GE's Source
```cpp
USTRUCT()
struct GAMEPLAYABILITIES_API FGameplayEffectContext
{
	/** Returns the physical actor tied to the application of this effect */
	virtual AActor* GetEffectCauser() const
	{
		return EffectCauser.Get();
	}
	/** Returns the ability system component of the instigator of this effect */
	virtual UAbilitySystemComponent* GetInstigatorAbilitySystemComponent() const
	{
		return InstigatorAbilitySystemComponent.Get();
	}
	/** Returns the object this effect was created from. */
	virtual UObject* GetSourceObject() const
	{
		return SourceObject.Get();
	}
}
```
---







