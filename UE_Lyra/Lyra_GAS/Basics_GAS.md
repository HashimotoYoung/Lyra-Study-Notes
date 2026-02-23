
## Main Classes 介绍

### ULyraAbilitySystemComponent

##### 主要属性:

`TObjectPtr<ULyraTagRelationshipMapping> TagRelationshipMapping` 
- 引用**数据资产** **`ULyraTagRelationshipMapping`**: 该**配置类**持有 Array< ULyraGameAbility之间的Tag关系* >, 并提供API辅助处理Tag相关的ASC方法, 例如 `virtural ApplyAbilityBlockAndCancelTags()` 


`TArray<FGameplayAbilitySpecHandle> InputPressedSpecHandles`  
`TArray<FGameplayAbilitySpecHandle> InputReleasedSpecHandles`
`TArray<FGameplayAbilitySpecHandle> InputHeldSpecHandles`
- 用于**临时Cache**和Input关联的GA: 玩家输入 => InputTag => Ability
- `InputPressed/ReleasedSpecHandles`会每帧清空

`int32 ActivationGroupCounts[(uint8)ULyraAbilityActivationGroup::MAX];`
- todo...

##### 主要方法:

### `ProcessAbilityInput(float DeltaTime, bool bGamePaused)`

:memo: Called **each frame** in `ALyraPlayerController::PostProcessInput()`;

1: 检测该帧内 "Pressed GA" ( `InputHeldSpecHandles/InputPressedSpecHandles`), 收集并找出满足激活条件的GA:
   - 首先进行 **Lyra Activation Policy 判断**: 例如 `LyraAbilityCDO->GetActivationPolicy() == ELyraAbilityActivationPolicy::WhileInputActive`
   - 赋值: `AbilitySpec->InputPressed` = True
   - 对于 **已经激活的** GA, 执行 `ASC::AbilitySpecInputPressed(FGameplayAbilitySpec& Spec)`

2: :star: 尝试激活所有收集到的 GA:  `ASC::TryActivateAbility()`
- // Lyra会在**一定程度上**控制GA的收集顺序
  // 唯二调用 *TryActivateAbility* 的地方, 另一处在 `ULyraGameplayAbility::TryActivateAbilityOnSpawn()`

3: 处理**该帧内产生**的"Release GA"(`InputReleasedSpecHandles`)
   - 赋值: `AbilitySpec->InputPressed` = False
   - 对于 **Active** GA, CALL `ASC::AbilitySpecInputReleased(FGameplayAbilitySpec& Spec)`

4: 清空数据 InputPressed/ReleasedSpecHandles 
   // InputHeldSpecHandles 在松开回调里 Remove



