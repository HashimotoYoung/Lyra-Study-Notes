## What `ULyraGameplayAbility` Adds to `UGameplayAbility`?

### Activation Group 激活组

:memo: 赋予了 GA 之间 **新的概念关系**, 类似于 "激活Track" 的概念, 每种 GA 必须要选择一种 Group
- 共三种, Lyra GA 默认初始化为 Independent
- Group 关系完全基于属性判断(`ActivationGroup`), 和 GA 自身类型无关
- 不同 Group 会有对应的逻辑流程处理
  ```cpp
   UENUM(BlueprintType)
   enum class ELyraAbilityActivationGroup : uint8
   {
      Independent,
      Exclusive_Replaceable,
      Exclusive_Blocking,
      MAX	UMETA(Hidden)
   };
  ```


##### 相关属性:
- Lyra ASC 会额外记录 GA 激活数量
```cpp
// In ULyraGameplayAbility.h
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
ELyraAbilityActivationGroup ActivationGroup;
```
```cpp
// In ULyraAbilitySystemComponent.h
int32 ActivationGroupCounts[(uint8)ELyraAbilityActivationGroup::MAX]
```

##### 相关示例:

1. 当 GA 通知激活时 (override `UGA::NotifyAbilityActivated(...)`), ASC 负责为 GA 所在 Group 添加记数
   - 	`LyraASC.ActivationGroupCounts[(uint8)Group]++;`
2. **if** Group ==  `Exclusive_Replaceable` || `Exclusive_Blocking`:
   - :pushpin: 遍历 `ASC.ActivatableAbilities` 尝试找到符合条件(`Exclusive_Replaceable`)的 GA Instance 并**执行 Cancel GA 流**

---

### Activation Policy 激活策略

新增的 GA 属性(可配置), 赋予GA额外的激活逻辑

##### 相关属性:
```cpp
// In ULyraGameplayAbility.h
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Ability Activation")
ELyraAbilityActivationPolicy ActivationPolicy;
```

##### 相关示例:
对于 `ELyraAbilityActivationPolicy::OnSpawn` 型 GA
- Lyra 在授予 GA 时 (override `OnGiveAbility()`), 会遍历 `ASC.ActivatableAbilities`, 找到此类 GA Spec, 并尝试激活



---

### 支持绑定 Camera Modes

##### 相关属性:

##### 相关方法:

---

### 额外 Cost

:memo: 在 GAS 基础上新增的一套自定义 Cost 体系

相关属性:

相关方法:

[Override] 


