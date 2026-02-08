
stride??

## FullBody_StartState (AnimLayer)


##### 相关属性:

- `StrideWarpingStartAlpha` **todo??**
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
<br>

#### 主节点 `Sequence Evaluator` 变为相关时:
- 根据相关属性找到合适的 AnimSequence for Start
- Set `StrideWarpingStartAlpha` = 0
<br>

#### 主节点 `Sequence Evaluator` 更新时: 

[AdvanceTimeByDistanceMatching](./Base.md#advancetimebydistancematching)

1. 计算 `StrideWarpingStartAlpha`:

2. 计算**动态播放速率Clamp**:
    - actualPlayRateClamp = `PlayRateClampStartsPivots`
3. **CALL `AdvanceTimeByDistanceMatching` 推进动画** with:
    - DistanceTraveled: `MainBP.DisplacementSinceLastUpdate` 
    - PlayRateClamp: actualPlayRateClamp
---

### HipFire Node `(SequenceEvaluator)` ??

---

### Warping




