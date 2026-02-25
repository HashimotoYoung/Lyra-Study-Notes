## Main Properties:

#### 速度组:

#### `LocalVelocityDirectionAngle : float`
- 通过 `UKismetAnimationLibrary::CalculateDirection()`计算出的**角色在自身坐标系下的移动方向角度**, 也可看作 "速度相对(朝向)角度"

#### `LocalVelocityDirectionNoOffset : Enum`
- 不带offset的版本,基于 `LocalVelocityDirectionAngle`

#### `LocalVelocityDirectionAngleWithOffset : float`
- 等于 LocalVelocityDirection - RootYawOffset, 代表下半身朝向

#### `LocalVelocityDirection : Enum`
带offset的版本, 基于 `LocalVelocityDirectionAngleWithOffset`

- :star: 一些动画序列需要选择**基于Offset方向或NoOffset方向**
例如, 当人物向右扭转一定角度后, 此时按右键移动时: `LocalVelocityDirectionNoOffset = Right`, `LocalVelocityDirection = Backward`, Start状态会选择播放 Start_Bwd 动画

<br>

#### 转身组:

#### `AdditiveLeanAngle : float` ??
#### `CarinalDirectionDeadZone : float`
- 死区

#### `CarinalDirectionFromAcceleration : Enum`

<br>

#### 朝向组:

#### `RootYawOffset : float` :star: 
**下半身 (RootBone) 相对上半身(ActorRotation) Yaw偏差值** 
- 假设人物当前Yaw=130, 右转 20 度后, `RootYawOffset` 变为 -20 (反向), 
- 默认情况下 AnimBP 每帧在输出Pose的最后阶段会对 RootBone 旋转 `RootYawOffset` 度, 从而呈现出 "ActorRotation 变但模型不变" 的表象
  - 上半身经过 Aim 流程后会和 ActorRotation 保持朝向一致
  - 在 BlendOut 阶段, RootYawOffset 逐渐变为 0, 对应表现为: 下半身 **"逐渐扭正"** 最终和上半身保持一致

#### `RootYawOffsetMode : Enum `   
决定了 `RootYawOffset` 的更新时算法, 共有三种状态

- Accumulate：在人物Idle/Stop状态**更新时**设置
- Hold: 在人物Start状态**更新时**设置
 // 此状态下 `RootYawOffset` 值会保持为切换时的值, 不会改变, 应该是为了用于 Transtion(Start => Cycle) 判断
- BlendOut：默认状态,每帧优先重置

#### `TurnYawCurveValue : float`
- 可理解为 "有效动画旋转剩余Yaw值", 具体可参考 `ProcessTurnYawCurve()`方法

<br>

#### Pivot 组:

#### `LastPivotTime : float`

Pivot播放时, 针对90度移动变换时的动画保持阈值, todo...

- 在 `SetUpPivotAnim()` 时设为常量 0.2s, 随更新时递减
- **主要作用一:** 用于 Transition to Cycle 的判断, 当触发90度移动时, 进行 Pivot 状态保护, 避免过快进入 Cycle 状态
- **主要作用二:** 在更新时如果 `LastPivotTime > 0`, 允许快速切换 Pivot 动画到 "Desired方向"

<br>

#### 射击组

#### `TimeSinceFiredWeapon : float`

---

## BP Functions:

### `SelectCardinalDirectionfromAngle()`

根据 Local Velocity Angle, 返回 **方向动画枚举**: `Enum_CardinalDirection`

##### Input 参数:
- `float Angle` Local Velocity Angle

  `float DeadZone` 初始化死区值 [Link](./Basics.md#转身组)

  `Enum_CardinalDirection CurrentDirection`

  `bool UseCurrentDirection`

##### 流:

1. **if** `UseCurrentDirection == true`, 根据 `CurrentDirection`值来**扩大前后死区值** `BwdDeadzone (Local)` or `FwdDeadzone (Local)`

2. **if**`AbsAngle (Local) <= FwdDeadzone + 45`, 返回 `Enum_CardinalDirection Foward`
**if**`AbsAngle >= 135 - BwdDeadzone`, 返回 `Enum_CardinalDirection Back`

3. 脱离前后死区的情况下, 返回 `Angle` > 0 ? `Enum_CardinalDirection Right` || `Enum_CardinalDirection Left`

---





