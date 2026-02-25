## FullBody_StartState (AnimLayer)

##### 相关属性:

- `StrideWarpingStartAlpha` todo...

  `HipFireUpperBodyOverrideWeight`

  `Vec PlayRateClampStartsPivots` **config值 (0.6 , 5.0)**

- `Enum MainBP.LocalVelocityDirection`

  `MainBP.LocalVelocityDirectionAngleWithOffset` 速度角度

  `MainBP.DisplacementSpeed`

  `MainBP.GameplayTag_IsADS`

  `MainBP.IsCrouching`

  `MainBP.DisplacementSinceLastUpdate`

##### 相关Anim序列:
- `Jog_Start_Cardinals` 四个方向

  `ADS_Start_Cardinals`

  `Crouch_Start_Cardinals`


#### 主节点 `Seq Evaluator` 变为相关时:
- 根据相关属性找到合适的 AnimSequence for Start
- Set `StrideWarpingStartAlpha` = 0

#### 主节点 `Seq Evaluator` 更新时: 

todo...

1. 计算 `StrideWarpingStartAlpha`

2. 计算 **动态播放速率Clamp**:
    - `let actualPlayRateClamp = PlayRateClampStartsPivots`
3. CALL [AdvanceTimeByDistanceMatching()](./Basics.md#advancetimebydistancematching) with `MainBP.DisplacementSinceLastUpdate`, `actualPlayRateClamp`

---

### HipFire Node `(SequenceEvaluator)` ??





