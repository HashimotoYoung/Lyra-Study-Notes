## FullBody_IdleState (AnimLayer)

- 依赖状态机 "IdleSM" 输出 Pose 

---

## 状态机 "IdleSM" 

:memo: First-Level 状态机

![](../../_img/ABP_ItemAnimLayerBase_IdleSM.png)


### Idle 状态 

内部使用状态机 ["IdleStance"](#状态机-idlestance-内部-fsm) 输出Pose

##### 相关属性:
- `TimeUntilNextIdleBreak`: 状态专用, 倒计时Counter
  `IdleBreakDelayTime`

##### Transitions:
- => IdleBreak: `TimeUntilNextIdleBreak <= 0` 
- => TurnInPlaceRotation: **`ABS(MainBP.RootYawOffset) > 50`**(Shared)

#### Output 变为相关时:
- 计算出一个随机值, 设为`IdleBreakDelayTime`, 然后**重置** `TimeUntilNextIdleBreak = IdleBreakDelayTime`

#### Output 更新时:
- 当State不处于 **BlendingOut** 阶段时, Try **update / reset** `TimeUntilNextIdleBreak`
  ```ts
  if(!IsStateBlendingOut()) {
    let deltaTime = GetDeltaTime()
    if `CanPlayIdleBreak()`== true do TimeUntilNextIdleBreak -= deltaTime
    else CALL `ResetIdleBreakTransitionLogic()` //重置 TimeUntilNextIdleBreak
  }
  ```
---

### IdleBreak 状态

主要负责播放 `Idle_Breaks` 动画

##### 相关属性:
- `CurrentIdleBreakIndex`

##### 相关Anim序列:
- `Idle_Breaks` Array

##### Transitions:
- => Idle: 
  - P1: `MainBP.GameplayTag_IsFiring == true` 
    //人物射击时也是idle状态?
  - P2: 自动播放完毕
  - P2: [CanPlayIdleBreak()](./Basics.md#canplayidlebreak) == False
- => TurnInPlaceRotation
  - **`ABS(MainBP.RootYawOffset) > 50`**(Shared)
  
#### Output 变为相关时:
- 按 **顺序** 播放IdleBreak动画:
  ```ts
  CALL SetSequence(Idle_Breaks[CurrentIdleBreakIndex])
  CurrentIdleBreakIndex++
  ```

---

### TurnInPlaceRotation 状态 

负责播放idle状态下站立和下蹲的转身动画

##### 相关属性:
- `MainBP.RootYawOffset`

  `TurnInPlaceAnimTime`:

  // Evaluator 累积播放时间, 用于 `SetEplicitTime()`, 也会给 TurnInPlaceRecovery 状态使用

  `float TurnInPlaceRotationDirection`:

  // 状态专属, 取值为1或-1, 用于判断转向

##### 相关Anim序列:
- `TurnInPlaceLeft`(转身动画), `TurnInPlaceRight`, `CrouchTurnInPlaceLeft`,   `CrouchTurnInPlaceRight`

##### Transitions:
- => TurnInPlaceRecovery 
  - `GetCurve("TurnYawWeight") == 0` // 完全依赖于动画曲线设置

#### Output 变为相关时:
- SET `TurnInPlaceRotationDirection = MainBP.RootYawOffset > 0 ? -1:1`

  // :pushpin: Output 结点的该方法 **优先于 Source 节点 (如 SeqEvaluator) 执行**

  // RootYawOffset 值是和 "转向" 相反的, 所以这里取反, 此属性决定了播哪个转向动画

#### 主节点 `Seq Evaluator` 变为相关时:

- SET `TurnInPlaceAnimTime = 0`, 并调用 `SetEplicitTime(0)`

#### 主节点 `Seq Evaluator` 更新时:
- 根据当前转向值 `TurnInPlaceRotationDirection` 选择合适的Anim序列(`TurnInPlaceLeft 或 Right`)
  - CALL `SetSequenceWithInertialBlending()`

    // :warning:在 Update 中不断获取动画并设置, 以便角色在切换站立/下蹲时, 动画可以从正确的位置开始播放

- `TurnInPlaceAnimTime += GetDeltaTime()`

  `SetEplicitTime(TurnInPlaceAnimTime)`
---

### TurnInPlaceRecovery 状态 

负责将转向动画的剩余部分播放完毕

- 主节点 `Seq Player`(和 TurnInPlaceRotation 中的 `Seq Evaluator` 形成对比) 会直接从 `TurnInPlaceAnimTime` 时间点开始播放

##### Transitions:
- => TurnInPlaceRotation: **`ABS(MainBP.RootYawOffset) > 50`**(Shared)
- => Idle: 动画播放完毕时

#### 主节点 `SeqEvaluator` 更新时:
- CALL `SetSequenceWithInertialBlending()`

---

## 状态机 "IdleStance" (内部 FSM)

:memo: 主要通过读取 Main BP 相关属性, 输出 Idle 动画

![](../../_img/FullBody_IdleState_idleSM_idle_idleStance.png)

### Idle 状态 

##### 相关属性:
- `GetMainBP.GameplayTag_IsADS`

  `GetMainBP.IsCrouching`

  `GetMainBP.CrouchStateChange` // Transition判断

##### 相关Anim序列:
- `idleADS`, `idleHipFire`, `CrouchIdle`

##### Transitions:
- => StanceTransition: `GetMainBP.CrouchStateChange == true` 

#### Output 更新时:
- 根据 `GetMainBP.GameplayTag_IsADS` 和 `GetMainBP.IsCrouching`, 找出合适的 Idle Anim 序列

  => CALL `SetSequenceWithInertialBlending()` 

---

### StanceTransition 状态

持续时间短暂的过度用状态, 负责播放**切换下蹲时的Enter/Exit**动画

##### 相关属性:
- `GetMainBP.IsCrouching`

  `GetMainBP.CrouchStateChange` // Transition判断

##### 相关Anim序列:
- `CrouchIdleEntry`, `CrouchIdleExit`

##### Transitions:
- => Idle: 
  - P1: `GetMainBP.CrouchStateChange == true` 
  - P2: 动画播放完毕

#### Output 变为相关时:
- 根据 `GetMainBP.IsCrouching` 播放 `CrouchIdleExit` or `CrouchIdleEntry`

