## CMC 物理模拟相关流程

### Start New Physics 流
:memo: 根据 **当前的`CMC.MovementMode`** 模拟 `deltaTime` 内的运动, eventually outputs a new world transform state for the character(`UpdatedComponent`)
- Pure Local 流程, no "Network" things
- Each `MovementMode` has its own `Phys*` method for calculating Transform, Velocity, Acceleration...
  - 分别处理 `MOVE_Walking` 和 `MOVE_NavWalking` 
  - **当处于 `MOVE_Custom` 时, 会调用 `virtual PhysCustom()`**
- :pushpin: 在一次 Tick 中, `MovementMode` 可进行多次改变, 每次变换后会重新执行此流程 (**Get Called Recursively**)

```cpp
// 相关方法
void UCharacterMovementComponent::StartNewPhysics(float deltaTime, int32 Iterations)
```

---

### PhysWalking

:memo: 行走流程

1: 预设剩余时间: 	`float remainingTime = deltaTime`

2: :repeat: **Start Loop while** `remainingTime` > `1e-6f` 

- 获取 **timeTick** = `GetSimulationTimeStep(remainingTime, Iterations)`, 调用 `remainingTime -= timeTick`
  > :memo: 在LOW FPS的情况下(例如 deltaTime > 50ms), 会进行 "Substepping", 即将 remainingTime 分多次处理, 以确保每次模拟的时间不会大于 `MaxSimulationTimeStep`

- 缓存 "Old" Location 和 Floor 信息
  ```cpp
  UPrimitiveComponent * const OldBase = GetMovementBase();
  const FVector PreviousBaseLocation = (OldBase != NULL) ? OldBase->GetComponentLocation() : FVector::ZeroVector;
  const FVector OldLocation = UpdatedComponent->GetComponentLocation();
  const FFindFloorResult OldFloor = CurrentFloor;
  ```
- 处理 RootMotionSource
- 归零 `Velocity.z` 和 `Acceleration.z`

2.1: 在准备移动前, **Comput Velocity**
- **if** 无 AnimRootMotion && `!CurrentRootMotion.HasOverrideVelocity()`
  - :pushpin: 执行 Velocity 内部计算流程
- Apply RootMotion to Velocity
- **if** is in *Falling* 
  - 重置流程并 **return** : [StartNewPhysics(`remainingTime + timeTick, Iterations-1`)](#start-new-physics-流)

2.2: 进行 "沿地面向前移动"
- **if** is in *Falling* 
  - 递归流程并 **return** : [StartNewPhysics(`remainingTime, Iterations`)](#start-new-physics-流)
- **if** is in *Swimming*, 处理游泳并 **return** 


2.3: 处理 Ground 和 Fall 相关逻辑 :pushpin:
- 进行 "抓地检测" 
  - 记录 Floor Hit Result => `FFindFloorResult CurrentFloor`
- **if** "允许边缘掉落" && `CurrentFloor` not Walkable
  **else**
    - **if** `CurrentFloor` is Walkable
      **else if** 存在 *Floor Penetration* 情况
    - **if** is in *Swimming*, 处理游泳并 **return** 
    - **if** `CurrentFloor` not Walkable && 不存在 *Floor Penetration* 
      - :pushpin: 进行 "掉落检测": `CheckFall()`
      - 判断可掉落时, 会直接进入 [PhysFalling 流程](#physfalling), 并 **return**


2.4: Post Process
- **if** 仍处于 *Walking* 模式
  - 计算出 Actual Velocity = `(UpdatedComponent->GetComponentLocation() - OldLocation) / timeTick`
- **if** :x: 此次迭代没有造成任何位移: `UpdatedComponent->GetComponentLocation() == OldLocation`
  - **break loop**
> // End Loop

---

### PhysFalling

:memo: 跳跃/下落流程

1: 获取 "空中水平加速度(Z = 0)"
- `FVector FallAcceleration = GetFallingLateralAcceleration(deltaTime)`

2: :repeat: **Start Loop while** `remainingTime` > `1e-6f`
- 获取 **timeTick** = `GetSimulationTimeStep(remainingTime, Iterations)`, 调用 `remainingTime -= timeTick`

- 缓存 "Old" 信息
  ```cpp
  const FVector OldLocation = UpdatedComponent->GetComponentLocation();
  const FQuat PawnRotation = UpdatedComponent->GetComponentQuat();
  ```
- 处理 CurrentRootMotion

2.1: **Compute Velocity**
- **if** 无 AnimRootMotion && `!CurrentRootMotion.HasOverrideVelocity()`
  - :pushpin: 执行 Vel 内部计算流程
- **if** 跳跃 Force 仍持续 (`CharacterOwner->JumpForceTimeRemaining > 0`)
  - todo...
- Apply Gravity to Vel
- Apply RootMotion to Vel

2.2: 处理 JumpApex
- 通知 "Reach Jump Apex"
  ```cpp
  if (bNotifyApex && (Velocity.Z < 0.f)){
      bNotifyApex = false;
      NotifyJumpApex();
  }
  ```

2.3: **Compute this Iteration's Displacement** 并 Apply

- 获取跳跃后的 `FHitResult Hit`

2.4: **if** `Hit.bBlockingHit`, 进行 "跳跃碰撞处理"


> // End Loop