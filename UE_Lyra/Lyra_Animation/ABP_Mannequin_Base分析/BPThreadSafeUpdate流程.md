## Blueprint ThreadSafe Update 流程

负责更新 Anim BP Variables, **可根据依赖关系, 区分为 第一层变量 和 第二层变量**

---

## 1. Update 第一层变量

### `UpdateLocationData()`

更新位移数据

##### 相关属性: 

- `WorldLocation` ***W***

  `float DisplacementSinceLastUpdate` ***W***; 注意该值非vector;

  `DisplacementSpeed` ***W***

  `isFirstUpdate` 

##### 流:
1. 计算当前帧的**位移差量 (float)**:
   - `DisplacementSinceLastUpdate` = Length(XY) between  `WorldLocation(上一帧位置)` and `PropertyAccess[GetActorLocation]`

2. **Update `WorldLocation`**

3. 计算当前帧的**位移速率 (float)**:
   - `DisplacementSpeed = DisplacementSinceLastUpdate / DeltaTime(Input参数)`

4. **if** 是第一次 Update (`isFirstUpdate==true`), 本次 tick 当作无效
   - Zero out `DisplacementSinceLastUpdate`, `DisplacementSpeed`

---

### `UpdateRotationData()`

更新旋转数据

##### 相关属性: 
- `WorldRotation` ***W***

  `float YawDeltaSinceLastUpdate` ***W***

  `float YawDeltaSpeed` ***W***

  `IsCrouching`

  `GameplayTag_IsADS`

  `float AdditiveLeanAngle` ***W***


##### 流:
1. **更新旋转数据:**
   - GET `PropertyAccess[GetActorRotation]` 
   - 获取当前帧 Yaw 差值
     - `YawDeltaSinceLastUpdate = GetActorRotation.z - WorldRotation.z`
   - `YawDeltaSpeed = YawDeltaSinceLastUpdate / DeltaTime`
   - **Update `WorldRotation`**

2. 使用 `YawDeltaSpeed` 计算 `AdditiveLeanAngle`  // 人物转的越快, 倾斜角度越大
	```cpp
	MagicFloat = isCrouching || GameplayTag_IsADS ? 0.025f: 0.0375f
	AdditiveLeanAngle = YawDeltaSpeed * MagicFloat 
	```

3. **if** 是第一次 Update (`isFirstUpdate==true`), 本次 tick 当作无效 
   - Zero out YawDelta

---

### `UpdateVelocityData()`

更新速度数据

##### 相关属性: 
- `LocalVelocity2D` ***W***

  `WorldVelocity` ***W***

  `float LocalVelocityDirectionAngle` ***W***

  `float LocalVelocityDirectionAngleWithOffset` ***W***

  `Enum LocalVelocityDirection` ***W***

  `Enum LocalVelocityDirectionNoOffset` ***W***

  `WorldRotation` 所以此方法必须在UpdateRotation之后调用

  `float CardinalDirectionDeadZone` [基本方位](./Basics.md#carinaldirectiondeadzone--float)

  `RootYawOffset` [模型旋转偏移](./Basics.md#rootyawoffset--float-star)

  `HasVelocity`  ***W***

##### 流:

1. 判断**上一帧**是否在移动: `let WasMovingLastUpdate =     LocalVelocity2D`
2. 更新 Local 速度:
   - `let WorldVelocity2D = PropertyAccess[TryGetPawnOwner.GetVelocity]`
   - :pushpin: **关键属性 `LocalVelocity2D = WorldVelocity2D << WorldRotation`**
3. 更新 Local 速度方向角度 (`float`):
   - `LocalVelocityDirectionAngle = CalculateDirection(WorldVelocity2D, WorldRotation)`
   - `LocalVelocityDirectionAngleWithOffset = LocalVelocityDirectionAngle - RootYawOffset`
4. 更新 Local 速度的**相对移动方位 Enum:**
   - `LocalVelocityDirection = SelectCardinalDirectionFromAngle(LocalVelocityDirectionAngleWithOffset,...)` 
     - [方法说明](./Basics.md#selectcardinaldirectionfromangle)
   - `LocalVelocityDirectionNoOffset = SelectCardinalDirectionFromAngle(LocalVelocityDirectionAngle,...)`

5. 判断当前帧是否在移动: 
   - SET `HasVelocity` from `LocalVelocity2D`

---

### `UpdateAccelerationData()`

更新加速度数据

PivotDirection2D 可理解为一个不断靠拢于 WorldAcceleration2D 的**动态加速度**

CardinalDirectionFromAcceleration 取其反向,说明其代表了Pivot转身后方位


##### 相关属性: 
- `WorldRotation` 此方法必须在UpdateRotation之后调用

  `LocalAcceleration2D` ***W***

  `HasAcceleration` ***W***

  `Enum CardinalDirectionFromAcceleration` ***W***, [基本加速度方位](./Basics.md#carinaldirectionfromacceleration--enum)

  `Vec PivotDirection2D` 此变量只在该方法用到

##### 流:

1. 计算 Local 加速度: 
   - `let WorldAcceleration2D = PropertyAccess[GetMoveComponent.GetCurrentAcceleration]` 
   - **关键属性: `LocalAcceleration2D = WorldAcceleration2D << WorldRotation`**

2. 计算当前帧是否有加速度: 
   - `HasAcceleration = LocalAcceleration2D.Length > 0`

3. 计算 `CardinalDirectionFromAcceleration`, 用于Pivot:
   - 使用向量Lerp更新PivotDirection2D: 
     - `PivotDirection2D = VLerp(PivotDirection2D, WorldAcceleration2D.normalized, 0.5f).normalized`
   - 计算出PivotDirection2D的 **相对方位**
     - `let angle = CalculateDirection(PivotDirection2D, WorldRotation)`
     - `let pivotCarinalDirection = SelectCardinalDirectionFromAngle(angle...)`
   - 取其反向作为结果 
     - `CardinalDirectionFromAcceleration = 取反(pivotCarinalDirection)`
---

## 2. Update 第二层变量


### `UpdateCharacterStateData()`

根据 CMC, 更新角色**状态相关**属性

##### 相关属性: 
- `IsOnGround` ***W***

- `IsCrouching` ***W***

  `CrouchStateChange` ***W***

- `GameplayTag_IsADS`

  `ADSStateChanged` ***W***

  `WasADSLastUpdate` ***W***

- `GameplayTag_IsFiring`

  `TimeSinceFiredWeapon` ***W***

- `IsJumping` ***W***

  `IsFalling` ***W***

##### 流: 
1. 更新在地状态 
   - `IsOnGround = GetMoveComponent.IsMovingOnGround`
2. 更新下蹲状态
   - SET `IsCrouching`, `CrouchStateChange` from `GetMoveComponent.IsCrouching`
3. 更新瞄准状态 
   - SET `WasADSLastUpdate`, `ADSStateChanged` from `GameplayTag_IsADS`
4. 更新武器开火状态 `TimeSinceFiredWeapon`:
   - **if** `GameplayTag_IsADS == true`, `TimeSinceFiredWeapon = 0`

     **else** `TimeSinceFiredWeapon += deltaTime`

5. 更新跳跃状态:
   - `IsJumping = IsFalling = false`
   - **if** Is *Falling* (`GetMoveComponent.MovementMode == falling`)
     - `IsJumping = WorldVelocity.z > 0`
     - `IsFalling = !IsJummping`


---

### `UpdateRootYawOffset()`

更新扭转相关数据, 主要负责计算 `Accumulate` 和 `BlendOut` 状态下的 `RootYawOffset`, 以及重置 `RootYawOffsetMode`

##### 相关属性: 

- `YawDeltaSinceLastUpdate`

  `Enum RootYawOffsetMode` ***W***, 有三种状态

  `float RootYawOffset` ***W***, [核心属性](./Basics.md#rootyawoffset--float-star)

  `AimYaw` ***W***

  `Vec RootYawOffsetAngleClamp` config值

  `Vec RootYawOffsetAngleClampCrouched` config值

   // 其 x,y 会作为Clamp角度的 min,max

  `GameplayTag_IsDashing`

##### 流:
1. 在`RootYawOffsetMode`为`Accumulate`模式时, 根据旋转积累属性:
   - `SetRootYawOffset( RootYawOffset - YawDeltaSinceLastUpdate )`
     
     // 只要人物在旋转, RootYawOffset 就会持续累积
 
2. 在`RootYawOffsetMode`为`BlendOut`模式时 **或** 人物在冲刺时(`GameplayTag_IsDashing = true`):
   - `let lerpValue = 浮点数弹簧插值算法(RootYawOffset, 0)`
   - `SetRootYawOffset( lerpValue )`
   
     // BlendOut模式下, 逐渐递减 RootYawOffset

3. :pushpin: Reset `RootYawOffsetMode = BlendOut`
   
   // 每帧重置, 因此外部需要在 Update 方法中设置为其它状态
<br>

#### `SetRootYawOffset(float newRootYawValue) 实现`
1. Zero out `RootYawOffset` and `AimYaw`
2. Clamp `newRootYawValue` 以避免上下半身扭转差异过大
3. `RootYawOffset = newRootYawValue` 
4. `AimYaw = -RootYawOffset` //始终保持相反值

---

### `UpdateJumpData()`

更新跳跃相关数据 TimeToJumpApex, 依赖于 `UpdateCharacterData()` 计算的 `IsJumping` 值

##### 相关属性: 
- `bool IsJumping`

  `WorldVelocity.Z` // 竖直方向速度

  `TimeToJumpApex` // 起跳到顶峰所需的时间

##### 流:
- **if** `!this.IsJumping`, `TimeToJumpApex = 0 `
  
  **else** `TimeToJumpApex = -1 * WorldVelocity.Z / GetMoveComponent.GetGravityZ()`


