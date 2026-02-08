
## FullBody_StopState (AnimLayer)

和 FullBody_StartState 结构类似
停止状态不处理任何 **Warping** 相关
初始Pose为 `DistanceMatchToTarget( 0 或者 predictDistance )`

##### 相关属性:


- `HipFireUpperBodyOverrideWeight`
  // 以下属性用于预测停止距离
  `Enum MainBP.LocalVelocityDirection` R
  `MainBP.LocalVelocityDirectionAngleWithOffset` R;速度角度
  `MainBP.DisplacementSpeed` R
  `MainBP.GameplayTag_IsADS` R
  `MainBP.IsCrouching` R

##### 相关Anim序列:
- `Jog_Stop_Cardinals` 四个方向
  `ADS_Stop_Cardinals`
  `Crouch_Stop_Cardinals`
<br>

#### 主节点 `Sequence Evaluator` 变为相关时:
[ShouldDistanceMatchStop()](Base.md#bool-shoulddistancematchstop) // 无加速度但仍有移动速度时

- 在设置完 Sequence 之后进行检测
  **if** `ShouldDistanceMatchStop() == true` 
    - // 什么都不做, 交由Update处理

  **else** 
    - `const distanceToTarget = 0`
      CALL `DistanceMatchToTarget(distanceToTarget)`
      // :star: **直接推进到Stop动画序列中角色 停稳不动 的时段**
<br>

#### 主节点 `Sequence Evaluator` 更新时:
[GetPredictStopDistance()](Base.md#float-getpredictedstopdistance)
- 角色仍有移动速度时, 使用 "预测停止距离" 来推进 Evaluator
  :star: 意味着Stop动画序列通常不会从 0 时刻开始播放, 而是从"首次预测距离"所在的时间点开始播放

```ts
let predictDistance = GetPredictStopDistance()
if( ShouldDistanceMatchStop() && predictDistance > 0) {
  DistanceMatchToTarget(predictDistance)
}else{
  //播完动画
  USequenceEvaluatorLibrary::AdvanceTime()
}
```





