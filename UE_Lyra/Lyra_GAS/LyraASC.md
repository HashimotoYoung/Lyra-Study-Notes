
## Main Classes 介绍

### ULyraAbilitySystemComponent : `UAbilitySystemComponent`

##### 主要属性:

`TObjectPtr<ULyraAbilityTagRelationshipMapping> TagRelationshipMapping` 
- 引用**数据资产** **`ULyraAbilityTagRelationshipMapping`**
  - 该类主要负责集中初始化 `ULyraGameAbility` 间的 Tag 关系(Cancel/Block)


`TArray<FGameplayAbilitySpecHandle> InputPressedSpecHandles`  
`TArray<FGameplayAbilitySpecHandle> InputReleasedSpecHandles`  
`TArray<FGameplayAbilitySpecHandle> InputHeldSpecHandles`
- 用于**临时缓存** "和Input关联的GASpec"


`int32 ActivationGroupCounts[(uint8)ULyraAbilityActivationGroup::MAX]`
- 见 [ActivationGroup](./LyraGA.md#activation-group-激活组)

##### 主要方法:

#### `AbilityInputTagPressed(const FGameplayTag& InputTag)`
#### `AbilityInputTagReleased(const FGameplayTag& InputTag)`
- 遍历 GA Specs (`ActivatableAbilities.Items`), 根据 `DynamicAbilityTags` 找到对应的 GASpec 并缓存 (`this.InputPressedSpecHandles/InputReleasedSpecHandles`)

#### `ProcessAbilityInput(float DeltaTime, bool bGamePaused)`

:memo: 负责检测并激活 "和Input关联的GASpec"
- :pushpin: 在 Tick 的 特定阶段 统一处理 GA 相关输入
  - Called **every frame** in `ALyraPlayerController::PostProcessInput()`, 


1 遍历该帧内 "Pressed GAs" (`this.InputHeldSpecHandles/InputPressedSpecHandles`), 过滤出 Activatable GA:
   - 进行 **Lyra Activation Policy 判断**: 
     - 例如 `LyraAbilityCDO->GetActivationPolicy() == ELyraAbilityActivationPolicy::WhileInputActive`
   - 赋值: `AbilitySpec->InputPressed = true` 
   - 对于 **已经激活的** GA, 执行 `ASC::AbilitySpecInputPressed(FGameplayAbilitySpec& Spec)`

2: :pushpin: 尝试激活所有收集到的 GA: `ASC::TryActivateAbility()`
- Lyra会在**一定程度上**控制GA的收集顺序  
- 唯二调用 *TryActivateAbility* 的地方, 另一处为 `ULyraGameplayAbility::TryActivateAbilityOnSpawn()`

3: 处理**该帧内产生**的 "Release GAs" (`this.InputReleasedSpecHandles`)
   - 赋值: `AbilitySpec->InputPressed = false` 
   - 对于 **Active** GA, CALL `ASC::AbilitySpecInputReleased(FGameplayAbilitySpec& Spec)`

4: Clear `InputPressed/ReleasedSpecHandles`   
   - `InputHeldSpecHandles` 在松开回调里 Remove


---

### :star: Lyra 如何联系 Input 和 GA

GA 层配置和初始化:
- `ULyraAbilitySet` 为每一种 GA (`TSubclassOf<ULyraGameplayAbility>`) **配对一个 InputTag** (`FGameplayTag`)
  - `ULyraAbilitySet` 为 Experience 系统相关配置
- 在授予 GA 阶段, 将配置的 InputTag 添加到 `GASpec.DynamicAbilityTags`
   ```cpp
   // in ULyraAbilitySet::GiveToAbilitySystem()
   ULyraGameplayAbility* AbilityCDO = AbilityToGrant.Ability->GetDefaultObject<ULyraGameplayAbility>();

   // 创建 GA Spec
   FGameplayAbilitySpec AbilitySpec(AbilityCDO, AbilityToGrant.AbilityLevel);
   AbilitySpec.SourceObject = SourceObject;
   AbilitySpec.DynamicAbilityTags.AddTag(AbilityToGrant.InputTag);

   // Give GA 
   const FGameplayAbilitySpecHandle AbilitySpecHandle = LyraASC->GiveAbility(AbilitySpec);
   ```
  
Input 层配置和初始化:
  - `FLyraInputAction` 为每一种 InputAction (`TObjectPtr<const UInputAction>`) **配对一个 InputTag** (`FGameplayTag`)
  - `ULyraHeroComponent` 组件负责绑定 Enhanced Input 相关回调, 并转发到 Lyra ASC 的相关方法:
    - `AbilityInputTagPressed(const FGameplayTag& InputTag)`

> 最终建立流程: InputAction >> InputTag >> GA

