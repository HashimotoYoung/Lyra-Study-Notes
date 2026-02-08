# Animation Tick 流程

?? `UAnimInstance::ParallelEvaluteAnimation()` when called
?? 属性或get方法能变成propertyaccess的条件
?? 了解 fastPath and warnaboutblueprint..设置(在animbp的classsetting中)

### 前提知识点:
-  一般情况下 `UAnimInstance (UObject)` 的 **Outer** 为 `USkeletalMeshComponent`
- UE使用双重缓冲技术 for parallel animation evaluation
- `GetProxyOnXXThread` 内部会调用 `HandleExistingParallelEvaluationTask`

---
## Let's Tick Anim

动画系统的 Tick 从`USkeletalMeshComponent::TickComponent()`开始, 该方法中:

首先进行 *Tick Pose* 流, 主要负责确定骨骼在当前帧的 “意图姿势”，即计算出动画蓝图最终想让骨骼达到的理想状态。
- 更新 Anim Instance 逻辑状态
- 更新 Montage

然后进行 *Evaluate* 流, 主要负责根据 TickPose 流计算出的 “意图姿势” 来**计算最终骨骼姿态**
- 处理动画融合 (Blending)
- 应用最终 IK (Inverse Kinematics) 
- 创建并行评估 (Parallel Evaluation) 任务
<br>

#### *Tick Pose* 流 :

- `UAnimInstance::UpdateAnimation(float delta)` 
    - if `ShouldOnlyTickMontages==true` 
    进入更新Montage流程并 **return**
      `PreUpdateAnimation()`
        -  `GetProxyOnGameThread<FAnimInstanceProxy>().PreUpdate(this,delta)` 
        // 动画 **Proxy** 开始预更新, **交换数据** AnimInstance => Proxy
    - `UpdateMontage()`
      相关AnimSubsystem::`OnPreUpdate_GameThread`
      // 缓存通过PropertyAcess访问的属性或方法返回值
    - `NativeUpdateAnimation()` 
      `BlueprintUpdateAnimation()` 
      // 以上两个方法分别对应c++和蓝图, 更新逻辑状态
      `if (bShouldImmediatelUpdate==true)`
        - todo........


#### *Evaluate* 流 : 
- `USkeletalMeshComponent::RefreshBoneTransforms()`
    - **填充 `AnimEvaluateContext` 属性**
    - `UAnimInstance::PreEvaluateAnimation()`
        - `GetProxyOnGameThread<FAnimInstanceProxy>().PreEvaluateAnimation(this);`
        // 给 Proxy中的MainInstanceProxy、Skeleton赋值。
    - **当启用了多线程 Evaluate时** :star:`DispatchParallelEvaluationTasks()` 
        - `SwapEvaluationContextBuffers()`
          // 准备context, **转移Skele数据** SkelComp => Context
          // 这里本质上是 offload the Evaluate phase to the Task Graph system.
        - 创建一个 `FParallelAnimationEvaluationTask` 
          创建一个 `FParallelAnimationCompletionTask`
          `TickFunction->GetCompletionHandle()->DontCompleteUntil(TickCompletionEvent)`
          //告知Tick层**需要等待 CompletionTask** 完成:
    

<br>

#### EvaluationTask 流 (在WorkerThread)
DoTask()中, 获取SkelMeshComp的引用, 执行并行Evalute逻辑:
- `USkeletalMeshComponent::ParallelAnimationEvaluation()`
    - `UAnimInstance::ParallelUpdateAnimation()`
        - :star:`GetProxyOnAnyThread<FAnimInstanceProxy>().UpdateAnimation();`
          - `NativeThreadSafeUpdateAnimation()`
          - `BlueprintThreadSafeUpdateAnimation()` 重要接口


#### CompletionTask 流 (在GameThread)
DoTask()中, 获取SkelMeshComp的引用, 执行Complete逻辑:
- `USkeletalMeshComponent::CompleteParallelAnimationEvaluation()`
    - **安全释放** Evaluation Task 
    - `SwapEvaluationContextBuffers()`
      // Evalute阶段结束 **返还Skele数据** Context => SkelComp
    - :star:**`PostAnimEvaluation()`** 开始PostAnimEvaluate流程
        - `UAnimInstance::PostUpdateAnimation()`
            - `GetProxyOnGameThread<FAnimInstanceProxy>().PostUpdate()`
        - `UAnimInstance::PostEvaluateAnimation()`
          执行PostEvaluateAnim通知
          FAnimInstanceProxy会clear数据
        - `FinalizeAnimationUpdate()`

---
### Anim 相关数据结构:

### `FAnimationEvaluationContext`

This struct is basically a 数据缓存池, holds all the data needed to run animation evaluation on a worker thread without touching the live `USkeletalMeshComponent` data.
<br>

### `FAnimInstanceProxy`
- **AnimGraph-accessed-data** moved from `UAnimInstance` to this struct

lockingwrappers?

- 从AnimGraph的视角, only the Proxy is **accessible**

---

### Anim 相关方法:

#### `HandleExistingParallelEvaluationTask()`

- 只在 GameThread 调用: Make the GameThread to pause and wait for the Animation Thread to finish its calculations

```ts
bool USkeletalMeshComponent::HandleExistingParallelEvaluationTask(bool bBlockOnTask, bool bPerformPostAnimEvaluation)
{
	//检测当前 USkeletalMeshComponent的ParallelAnimationEvaluationTask引用是否不为空
	if (IsRunningParallelEvaluation())
	{
        //决定是否要等待
		if (bBlockOnTask)
		{
            //检测当前调用线程
			check(IsInGameThread());

            //keypoint: 阻塞GameThread, 进入等待状态
			FTaskGraphInterface::Get().WaitUntilTaskCompletes(ParallelAnimationEvaluationTask, ENamedThreads::GameThread);

            //进入PostAnimEvaluation阶段
			CompleteParallelAnimationEvaluation(bPerformPostAnimEvaluation); //Perform completion now
		}
		return true;
	}
	return false;
}
```
