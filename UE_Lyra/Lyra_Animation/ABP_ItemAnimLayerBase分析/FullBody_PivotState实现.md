## FullBody_PivotState (AnimLayer)



![](../../_img/FullBody_PivotState.png)

### `UpdateHipFireRaiseWeaponPose()`
todo...

---

## Pivot 状态机

使用PivotA和PivotB两个**配置一模一样的状态**来回相互切换

### Pivot A 状态

![](../../_img/PivotSM_PivotA.png)

##### 相关属性:
- `StrideWarpingBlendInDurationScaled` // Warping相关
- `float StrideWarpingPivotAlpha` // 状态专属

  `Vec PivotStartingAcceleration` // 状态专属

  `float TimeAtPivotStop` // 状态专属

  `MainBP.CardinalDirectionFromAcceleration`

  // 状态专属, 核心属性, 在 `MainBP.UpdateAcceleration()` 中更新

- `MainBP.LocalVelocityDirectionAngleWithOffset`

  `MainBP.DisplacementSpeed`

  `MainBP.LocalVelocity2D`

  `MainBP.LocalAcceleration2D` // 核心属性

  `MainBP.LastPivotTime` **W**

##### 相关Anim序列 (共12个):
- `Jog_Pivot_Cardinals` 四方向

  `ADS_Pivot_Cardinals`

  `Crouch_Pivot_Cardinals`

##### Transitions:
- => Pivot B
  - `Dot(MainBP.LocalVelocity2D, MainBP.LocalAcceleration2D) < 0` 
    && `MainBP.IsMovingPerpendicularToInitialPivot == false`
    && `Dot(PivotStartingAcceleration, MainBP.LocalAcceleration2D) < 0`
    // **永远以当前加速度为基准**
    // 和 PivotB => PivotA 共享

#### 主节点 `Seq Evaluator` 变为相关时:
1. 初始化加速度 Vec
   - `PivotStartingAcceleration = MainBP.LocalAcceleration2D`
2. 根据**加速度方向枚举** 获取 Desired Pivot序列
   - `let animSeq = GetDesiredPivotSeq( MainBP.CardinalDirectionFromAcceleration )`

3. 初始化
   - `SetSequence(animSeq)`
   - `SetExplicitTime(0)`
   - `StrideWarpingPivotAlpha = 0`
   - `TimeAtPivotStop = 0`
4. :pushpin: **写入 MainBP 属性:** `MainBP.LastPivotTime = 0.2`(固定值)

#### 主节点 `Seq Evaluator` 更新时:
1. 获取Evaluator累积时间
   - `let ExplicitTime = GetAccumulatedTime()`
2. **在 Last Pivot Time 阶段**, 尝试获取 Desired Pivot序列, 如果和当前序列不一致则**重设动画**
   - **if** `MainBP.LastPivotTime > 0`
     - `let newSeq = GetDesiredPivotSeq( MainBP.CardinalDirectionFromAcceleration )`

     - **if** `newSeq != 当前播放Seq` 
       - `SetSequenceWithInertialBlending( newSeq )`
       - `PivotStartingAcceleration = MainBP.LocalAcceleration2D` // 重设初始加速度 
3. **if** `Dot(MainBP.LocalVelocity2D, MainBP.LocalAcceleration2D) < 0` // 加速度和速度方向不一致时
    - todo... distancematching

    **else** // 保持一致时
    - todo ... `AdvanceTimeByDistanceMatching`



