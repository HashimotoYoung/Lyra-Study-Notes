## FullBody JumpState  相关

:memo: 以下四个Layer皆通过**单个Sequence Player** 实现
- Jump Start
  // 起跳动画
- Jump Start Loop
  // 起跳到达到顶端的过程动画
- Jump Apex
  // 达到顶端动画, 开始下落动画
- Fall Loop
  // 开始下落到落地的过程动画

---

## FullBody_FallLandState (AnimLayer)

##### 相关属性:

- `GetMainBP.GroundDistance`

##### 相关Anim序列:

- `JumpFallLand`

#### 主节点 `Sequence Evaluator` 变为相关时:

CALL `SetExplicitTime(0)`

#### 主节点 `Sequence Evaluator` 更新时:

执行 [DistanceMatchToTarget](./Base.md#distancematchtotarget):
`let groundDistance = GetMainBP.GroundDistance`
`let jumpCurveName = "GroundDistance"`
CALL `DistanceMatchToTarget(seqEvaluator, groundDistance, jumpCurveName)`