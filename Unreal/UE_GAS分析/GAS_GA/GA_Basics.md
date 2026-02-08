# Gameplay Ability 总览

> A GA is an in-game action that an Actor can own and trigger repeatedly

- GA runs  on **the owning client and/or the server** ( 取决于`NetExecutionPolicy` ) 
- :x: GA don't run on Simulated Proxies, which only **receive results** via  replication/RPC ( through `AbilityTasks`/`GameplayCues` )



:star: GA 本质上是 **decision logic flow**, 而非 authoritative state, 也没有内置 Tick 方法

- GA decides what should happen, other than store gameplay state, 
  - Authoritative state lives in GE;ASC;AS... 
- **BUT**, The use of `AbilityTasks` can make **Instanced GA** "Stateful", so they can:
  - store variables
  - wait, tick, react to events
  - has the concept of Active, existing from `ActivateAbility()` to `EndAbility()`

---

### Instancing Policy
GA的核心配置属性, 决定了 How many `UGameplayAbility` will be **instantiated**

1. Instanced Per Execution: 每次创建
   - 一般用于角色大招
2. Instanced Per Actor: One for each `ASC`
   - reused and so need reset data manually
   - 该类 GA 在同一时刻只能有一个 instance in Active (当需要重新Activate该GA时, 要先End)

3. Non Instanced: Operates on **CDO** , 效率高
   - **Cannot store state**, meaning no dynamic variables and no binding to `AbilityTask` delegates.
   - 该类 GA 需要完全由C++实现, NO BP 
   - 一般用于基础 Ability, 例如普攻, 跳跃

<br>

### Net Execution Policy
GA的配置属性, 决定了 **Who has permission to run** this GA
1. Local Only: 仅在Client执行

2. Server Only: 仅在Server执行
   - 比较适用于 **Passive GA** , 一般会在 `OnAvatarSet()` 中进行处理:
     ```cpp
     void UGDGameplayAbility::OnAvatarSet(const FGameplayAbilityActorInfo * ActorInfo, const FGameplayAbilitySpec & Spec) {
        Super::OnAvatarSet(ActorInfo, Spec);
        if (bActivateAbilityOnGranted){
          ActorInfo->AbilitySystemComponent->TryActivateAbility(Spec.Handle, false);
        }
     }
     ```
3. Server Initiated: todo...
4. **Local Predicted:**
   - GA => 先在Client执行,并发送请求 => 在Server执行, 并进行**预测纠正**

---

### Ability Scoped Lock 

:bulb: 一种 "重入保护" 写法技巧

- 底层利用 C++ RAII 机制实现
  ```cpp
  // 示例
  struct FAbilityListLock
  {
      FAbilitySystemComponent* ASC;
      FAbilityListLock(UAbilitySystemComponent* InASC) : ASC(InASC) { ++ASC->AbilityScopeLockCount; }
      ~FAbilityListLock() { --ASC->AbilityScopeLockCount; }
  };
  // 快捷宏定义
  #define ABILITYLIST_SCOPE_LOCK() FAbilityListLock AbilityListLock(this);
  ```
通常由两个部分构成: Guard 和 Lock
- 在执行 "dangerous code" 前加入 **Guard**
  - 一般指代触发**结构修改**的代码, 例如添加或删除 ArrayItem
- 当触发 Guard 时, 通常意味着 CallStack 中存在 "fragile code"
  - 通常会补充 "延迟处理" 相关逻辑  
  ```cpp
  FGameplayAbilitySpecHandle ASC::GiveAbility(const FGameplayAbilitySpec& Spec) {
      // Guard 示例
      if (this.AbilityScopeLockCount > 0) {
          AbilityPendingAdds.Add(Spec); // defer addition
          return Spec.Handle; 
      }
      // Safe to add directly
      FGameplayAbilitySpec& OwnedSpec = ActivatableAbilities.Items[ActivatableAbilities.Items.Add(Spec)];
      ...
  }
  ```

- 在执行 "fragile code" ( 例如遍历Array ) 前加入 **Lock**
  ```cpp
  void ASC::CancelAbility(UGameplayAbility* Ability) {
    // Lock 示例
    ABILITYLIST_SCOPE_LOCK();

    for (FGameplayAbilitySpec& Spec : ActivatableAbilities.Items) {
      if (Spec.Ability == Ability) CancelAbilitySpec(Spec, nullptr);
    }
  }
  ```
---