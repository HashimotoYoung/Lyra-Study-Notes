## CMC 主要流程

### Tick Component 流 

```cpp
ACharacter* CharacterOwner: 玩家角色
UCMC::TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction *ThisTickFunction)
```
:pencil2: **Start:**
1: 消耗玩家输入
- `FVector InputVector = ConsumeInputVector()`

1.1: AvoidanceLockTimer -= DeltaTime ??

2: **if** *CharacterOwner (Local Role)* is *Authority* or *Autonoumous*
- **if** on Client:
  - 处理 `FNetworkPredictionData_Client_Character` ??
  `ClientUpdatePositionAfterServerUpdate`
- **if** *CharacterOwner* 由本地控制 (*IsLocallyControlled*)
  - **if** *Authority*
    - 执行 [Perform Movement 流](#perform-movement-流)
  - **else if** *Autonoumous*
    - 执行 [Local Move 流](#local-move-流)
- **else if** *CharacterOwner's* 远端角色为 *Autonoumous*: 
  - MaybeMove??
  - 服务端处理主控型角色的 Tick 流程

3: **if**  is *Simulated*
- 模拟端 Tick: `SimulatedTick(DeltaTime)`

---
### Local Move 流

:memo: 由 **Autonoumous** 端执行的本地预测移动流程
```cpp
void UCharacterMovementComponent::ReplicateMoveToServer(float DeltaTime, const FVector& NewAcceleration)
```
:pencil2: **Start:**
1: 基础检测
- **if** Possess 流程尚未完成, **return**
  - `PC->AcknowledgedPawn != CharacterOwner`
- **if** PC.Player 为 NULL, **return**
- :pushpin: 获取本地预测数据 **ClientData** , **return** if NULL
  - `FNetworkPredictionData_Client_Character* ClientData = GetPredictionData_Client_Character()`

2: 更新 Moves
- 更新 DeltaTime, `UpdateTimeStampAndDeltaTime`?
- 找到 **OldMove** 
  - 指代 the *Oldest*, *Unacknowledged*, *Important* `FSavedMove_Character`
- **创建** **NewMove** from `CreateSavedMove`?
- 拷贝 character 的当前移动状态到 NewMove
  - `FSavedMove_Character::SetMoveFor()`
  


2.1: 尝试 Combine Moves,  **if** `ClientData->PendingMove` != NULL
- ClientData.PendingMove

3: 开始执行本地移动
- 计算 **`this.Acceleration`**
- 执行 [Perform Movement 流](#perform-movement-流) (基于 `NewMove->DeltaTime`)
- PostUpdate NewMove

4: 上报 Moves 数据到 Server :zap:
- Pack Moves 并调用 RPC (触发 [Server Handle Move Data 流](#server-handle-move-data-computer))

5: 清空 `ClientData->PendingMove`


---

### Perform Movement 流
- 负责 Applies external physics
- 处理 AnimRootMotion 和 RootMotionSources
- Only one Parameter: `float DeltaSeconds` 

```cpp
// 相关属性
bTeleportedSinceLastUpdate
bForceNextFloorCheck

// 相关方法
void UCharacterMovementComponent::PerformMovement(float DeltaSeconds)
```
:pencil2: **Start:**
1: 检查当前帧是否存在 "Teleport" 现象
  - 通过比较  UpdatedComponent.worldPos 和 this.LastUpdateLocation
    - `this.bTeleportedSinceLastUpdate = UpdatedComponent->GetComponentLocation() != this.LastUpdateLocation`

- `bTeleportedSinceLastUpdate` 为 True 时, 意味着 "Something snapped the character's pos **outside of the normal CMC physics loop**"

1.1: 尝试更新 `CurrentRootMotion.LastPreAdditiveVelocity`

2: Move 执行前的准备阶段
- :arrow_right: 进入 `FScopedMovementUpdate`
- 执行 [Maybe Update Based Movement 流](#try-update-based-movement-流)
- 清理 无效的 RootMotion Sources
- 施加 *Accumulated Forces*
- 在**移动前**, 更新一次 Character State
  - 切换站立/下蹲
- 处理 *Pending Launch*
- 清理 *Accumulated Forces*
- 二次尝试更新 `CurrentRootMotion.LastPreAdditiveVelocity`
- Prepare Root Motion
- Apply Root Motion to Velocity
- 清理 Jump 输入

3: **Change Position** 
- 本地模拟: [Start New Physics 流](./CMC_LocalPhysicFlows.md#start-new-physics-流)

4: Move 执行后的处理一
- 在**移动后**, 更新一次 Character State
- **if** 无 AnimRootMotion || 允许在有RootMotion的情况下旋转
  - **Change Rotation:** `PhysicsRotation(DeltaSeconds)`
- **if** Has AnimRootMotion 
  - todo...
- **else if** Has RootMotionSource
  - todo...
- 调用 virtual 空方法: 	`OnMovementUpdated(DeltaSeconds, OldLocation, OldVelocity)`
- :arrow_left: 离开 `FScopedMovementUpdate`

4.1: Move 执行后的处理二
- SET 组件速度: `UpdatedComponent->ComponentVelocity = this.Velocity`
- **Broadcast** "角色移动已更新": `OnCharacterMovementUpdated`
- Save Base Location

5: :computer: Server 端专属处理 

- **if** `bHasAuthority && UpdatedComponent && !IsNetMode(NM_Client)` 并且 **位置或旋转有改变**, 
更新 Server 端时间戳: `this.ServerLastTransformUpdateTimeStamp`
  - **if** *CharacterOwner* is remotely Autonomous 
    - 取值为  `GetPredictionData_Server_Character()->ServerAccumulatedClientTimeStamp`
  - **else**
    - 取值为世界时间 `MyWorld->GetTimeSeconds()`

6: 缓存本次计算结果 (世界坐标系)
- `LastUpdateLocation = NewLocation`
  `LastUpdateRotation = NewRotation`
	`LastUpdateVelocity = Velocity`

---

#### Simulate Movement 流

---

### Move Autonomous 流


---

#### Maybe Update Based Movement 流

---
### Server Handle Move Data :computer:

```cpp
void UCharacterMovementComponent::ServerMove_HandleMoveData(const FCharacterNetworkMoveDataContainer& MoveDataContainer)
```