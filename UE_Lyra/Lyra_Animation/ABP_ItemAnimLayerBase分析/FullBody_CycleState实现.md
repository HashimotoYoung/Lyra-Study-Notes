## FullBody_CycleState (AnimLayer)

主要负责人物处于跑步或走路循环状态时的 Pose Output

:warning: 选用了 NoOffset 版本的 VelocityDirection 属性, 这点和Start,Stop不一样,原因是应该以人物上半身方向(ActorRotation), 决定动画方向

##### 相关属性:

- `StrideWarpingCycleAlpha`: StrideWarping的Cycle值

  `HipFireUpperBodyOverrideWeight`

  `Vec PlayRateClampCycle` **有默认值**

- `Enum MainBP.LocalVelocityDirectionNoOffset` *NoOffset版本*

  `MainBP.LocalVelocityDirectionAngle` *NoOffset版本* 

  `MainBP.DisplacementSpeed` 

  `MainBP.GameplayTag_IsADS` 
  
  `MainBP.IsCrouching` 

##### 相关Anim序列:
- `Jog_Cardinals` 四方向

  `Walk_Cardinals` 同上

  `Crouch_Walk_Cardinals` 同上

#### 主节点 `Seq Player` 更新时:
- 使用 **`MainBP.DisplacementSpeed`** 更新cycle动画速率

![UpdateCycleAnim1](../../_img/UpdateCycleAnim1.png)

---
### OrientationWarping ??
todo...

---
### StrideWarping ??
todo...






