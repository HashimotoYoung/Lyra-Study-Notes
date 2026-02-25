
## FullBody_StopState (AnimLayer)

和 FullBody_StartState 结构类似

- 此状态不处理任何 Warping 相关
- 初始 Pose 为 `DistanceMatchToTarget( 0 或者 predictDistance )`

##### 相关属性:


- `HipFireUpperBodyOverrideWeight`

  以下属性用于预测停止距离

  `Enum MainBP.LocalVelocityDirection` 

  `MainBP.LocalVelocityDirectionAngleWithOffset` // 速度角度

  `MainBP.DisplacementSpeed` 

  `MainBP.GameplayTag_IsADS` 

  `MainBP.IsCrouching` 

##### 相关Anim序列:
- `Jog_Stop_Cardinals` 四个方向

  `ADS_Stop_Cardinals`

  `Crouch_Stop_Cardinals`


#### 主节点 `Seq Evaluator` 变为相关时:

- 设置 Anim Sequence 

- **if** [ShouldDistanceMatchStop()](./Basics.md#bool-shoulddistancematchstop) 
    - // Do nothing, 交由 Update 处理

  **else** 
    - CALL `DistanceMatchToTarget(distanceToTarget:0)`

      // :star: **直接推进到 Stop序列 中角色停稳不动的时段**

#### 主节点 `Seq Evaluator` 更新时:

- 角色仍有移动速度时, 使用 "预测停止距离" 来推进 Evaluator
  ```ts
  let predictDistance = GetPredictStopDistance()
  if (ShouldDistanceMatchStop() && predictDistance > 0) {
    DistanceMatchToTarget(predictDistance)
  }else{
    //播完动画
    USequenceEvaluatorLibrary::AdvanceTime()
  }
  ```
  - // [GetPredictStopDistance()](./Basics.md#float-getpredictedstopdistance)





