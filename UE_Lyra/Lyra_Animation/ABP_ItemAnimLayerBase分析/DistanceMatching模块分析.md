### FAQ:

*1. 为什么向前跑动的动画序列 like "MF_Rifle_Jog_Fwd_Start" has `Enable Root Motion` checked ?*

- The reason the animation asset has "Enable Root Motion" checked is not to drive the character's capsule, but to **extract data** from the animation. 在Lyra中主要用于Distance Matching.
  - 该类动画同时 has `Force Root Lock` checked, 因此角色移动本质上还是由 `UCharacterMovementComponent` 驱动
- "Enable Root Motion" checked in Animation Asset 并不等价于 "Root Motion is applied to Character in-game".

---

## Distance Matching 模块

### `AdvanceTimeByDistanceMatching()` 
**根据移动距离适配步进动画**

- 根据DeltaTime内移动的距离和Curve推算出移动后的时间点, 进行Clamp后, 推进 SequenceEvaluator 
- :star: 从 `GetTimeAfterDistanceTraveled`方法可看出, 适配DistanceMatching特性的动画序列, 应含有 Distance相关的 ***Monotonic*** 动画曲线

```cpp
FSequenceEvaluatorReference UAnimDistanceMatchingLibrary::AdvanceTimeByDistanceMatching(const FAnimUpdateContext& UpdateContext, const FSequenceEvaluatorReference& SequenceEvaluator,
  float DistanceTraveled, FName DistanceCurveName, FVector2D PlayRateClamp)
{
  SequenceEvaluator.CallAnimNodeFunction<FAnimNode_SequenceEvaluator>(
    TEXT("AdvanceTimeByDistanceMatching"),
    [&UpdateContext, DistanceTraveled, DistanceCurveName, PlayRateClamp](FAnimNode_SequenceEvaluator& InSequenceEvaluator)
    {
      if (const FAnimationUpdateContext* AnimationUpdateContext = UpdateContext.GetContext())
      {
        const float DeltaTime = AnimationUpdateContext->GetDeltaTime(); 

        if (DeltaTime > 0 && DistanceTraveled > 0)
        {
          if (const UAnimSequenceBase* AnimSequence = Cast<UAnimSequence>(InSequenceEvaluator.GetSequence()))
          {
            const float CurrentTime = InSequenceEvaluator.GetExplicitTime();
            const float CurrentAssetLength = InSequenceEvaluator.GetCurrentAssetLength();
            const bool bAllowLooping = InSequenceEvaluator.GetShouldLoop();

            // 获取曲线ID
            const USkeleton::AnimCurveUID CurveUID = UE::Anim::DistanceMatchingUtility::GetCurveUID(AnimSequence, DistanceCurveName);

            // 关键方法, 推算出Traveled后的Explicit时间
            float TimeAfterDistanceTraveled = UE::Anim::DistanceMatchingUtility::GetTimeAfterDistanceTraveled(AnimSequence, CurrentTime, DistanceTraveled, CurveUID, bAllowLooping);

            // 考虑循环
            if (TimeAfterDistanceTraveled < CurrentTime)
            {
              TimeAfterDistanceTraveled += CurrentAssetLength;
            }
            
            // ExplicitTimeDelta / 实际DeltaTime
            float EffectivePlayRate = (TimeAfterDistanceTraveled - CurrentTime) / DeltaTime;

            // Clamp the effective play rate.
            if (PlayRateClamp.X >= 0.0f && PlayRateClamp.X < PlayRateClamp.Y)
            {
              EffectivePlayRate = FMath::Clamp(EffectivePlayRate, PlayRateClamp.X, PlayRateClamp.Y);
            }

            // Advance animation time by the effective play rate.
            float NewTime = CurrentTime;
            // clamp后计算实际前进的位置
            FAnimationRuntime::AdvanceTime(bAllowLooping, EffectivePlayRate * DeltaTime, NewTime, CurrentAssetLength);

            if (!InSequenceEvaluator.SetExplicitTime(NewTime))
            {
              UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Could not set explicit time on sequence evaluator, value is not dynamic. Set it as Always Dynamic."));
            }
          }
          else
          {
            UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Sequence evaluator does not have an anim sequence to play."));
          }
        }
      }
      else
      {
        UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("AdvanceTimeByDistanceMatching called with invalid context"));
      }
    });

  return SequenceEvaluator;
}
```
<br>

```cpp
static float GetTimeAfterDistanceTraveled(const UAnimSequenceBase* AnimSequence, float CurrentTime, float DistanceTraveled, USkeleton::AnimCurveUID CurveUID, const bool bAllowLooping)
{
  float NewTime = CurrentTime;
  if (AnimSequence != nullptr)
  {
    // 距离区间不等于0时
    if (!FMath::IsNearlyZero(UE::Anim::DistanceMatchingUtility::GetDistanceRange(AnimSequence, CurveUID)))
    {
      float AccumulatedDistance = 0.f;

      const float SequenceLength = AnimSequence->GetPlayLength();
      //采样时间
      const float StepTime = 1.f / 30.f;

      // Distance Matching expects the distance curve on the animation to increase monotonically. If the curve fails to increase in value
      // after a certain number of iterations, we abandon the algorithm to avoid an infinite loop.
      const int32 StuckLoopThreshold = 5;
      int32 StuckLoopCounter = 0;

      // Traverse the distance curve, accumulating animated distance until the desired distance is reached.
      while ((AccumulatedDistance < DistanceTraveled) && (bAllowLooping || (NewTime + StepTime < SequenceLength)))
      {
        // 曲线距离值
        const float CurrentDistance = AnimSequence->EvaluateCurveData(CurveUID, NewTime);
        // step后的曲线距离值
        const float DistanceAfterStep = AnimSequence->EvaluateCurveData(CurveUID, NewTime + StepTime);
        const float AnimationDistanceThisStep = DistanceAfterStep - CurrentDistance;

        if (!FMath::IsNearlyZero(AnimationDistanceThisStep))
        {
          // Keep advancing if the desired distance hasn't been reached.
          if (AccumulatedDistance + AnimationDistanceThisStep < DistanceTraveled)
          {
            // 推进时间 by StepTime
            FAnimationRuntime::AdvanceTime(bAllowLooping, StepTime, NewTime, SequenceLength);
            AccumulatedDistance += AnimationDistanceThisStep;
          }
          // Once the desired distance is passed, find the approximate time between samples where the distance will be reached.
          else
          {
            // actualTime / (DistanceTraveled - AccumulatedDistance) = StepTime / AnimationDistanceThisStep
            const float DistanceAlpha = (DistanceTraveled - AccumulatedDistance) / AnimationDistanceThisStep;
            FAnimationRuntime::AdvanceTime(bAllowLooping, DistanceAlpha * StepTime, NewTime, SequenceLength);
            AccumulatedDistance = DistanceTraveled;
            break;
          }

          StuckLoopCounter = 0;
        }
        else
        {
          ++StuckLoopCounter;
          if (StuckLoopCounter >= StuckLoopThreshold)
          {
            UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Failed to advance any distance after %d loops on anim sequence (%s). Aborting."), StuckLoopThreshold, *GetNameSafe(AnimSequence));
            break;
          }
        }
      }
    }
  }

  return NewTime;
}
```

---

### `DistanceMatchToTarget()` 

- :star: `GetAnimPositionFromDistance`
  - 根据具体的 **距离点(InDistance)** 找到对应时间
  - 二分查找写法 Good

```cpp
FSequenceEvaluatorReference UAnimDistanceMatchingLibrary::DistanceMatchToTarget(const FSequenceEvaluatorReference& SequenceEvaluator,
  float DistanceToTarget, FName DistanceCurveName)
{
  SequenceEvaluator.CallAnimNodeFunction<FAnimNode_SequenceEvaluator>(
    TEXT("DistanceMatchToTarget"),
    [DistanceToTarget, DistanceCurveName](FAnimNode_SequenceEvaluator& InSequenceEvaluator)
    {
      if (const UAnimSequenceBase* AnimSequence = Cast<UAnimSequence>(InSequenceEvaluator.GetSequence()))
      {
        const USkeleton::AnimCurveUID CurveUID = UE::Anim::DistanceMatchingUtility::GetCurveUID(AnimSequence, DistanceCurveName);
        if (AnimSequence->HasCurveData(CurveUID))
        {
          // By convention, distance curves store the distance to a target as a negative value.
          // 需要曲线存负值
          const float NewTime = UE::Anim::DistanceMatchingUtility::GetAnimPositionFromDistance(AnimSequence, -DistanceToTarget, CurveUID);
          if (!InSequenceEvaluator.SetExplicitTime(NewTime))
          {
            UE_LOG(LogAnimDistanceMatchingLibrary, Warning, TEXT("Could not set explicit time on sequence evaluator, value is not dynamic. Set it as Always Dynamic."));
          }
        }
      }
    });
  return SequenceEvaluator;
}
```
<br>

```cpp
static float GetAnimPositionFromDistance(const UAnimSequenceBase* InAnimSequence, const float& InDistance, USkeleton::AnimCurveUID CurveUID)
{  
  FAnimCurveBufferAccess BufferCurveAccess(InAnimSequence, CurveUID);
  if (BufferCurveAccess.IsValid())
  {
    const int32 NumKeys = BufferCurveAccess.GetNumSamples();
    if (NumKeys < 2)
    {
      return 0.f;
    }

    // Some assumptions: 
    // - keys have unique values, so for a given value, it maps to a single position on the timeline of the animation.
    // - key values are sorted in increasing order.
    // 二分查找
    int32 First = 1;
    int32 Last = NumKeys - 1;
    int32 Count = Last - First;

    while (Count > 0)
    {
      int32 Step = Count / 2;
      int32 Middle = First + Step;

      if (InDistance > BufferCurveAccess.GetValue(Middle))
      {
        First = Middle + 1;
        Count -= Step + 1;
      }
      else
      {
        Count = Step;
      }
    }

    const float KeyAValue = BufferCurveAccess.GetValue(First - 1);
    const float KeyBValue = BufferCurveAccess.GetValue(First);
    const float Diff = KeyBValue - KeyAValue;
    const float Alpha = !FMath::IsNearlyZero(Diff) ? ((InDistance - KeyAValue) / Diff) : 0.f;

    const float KeyATime = BufferCurveAccess.GetTime(First - 1);
    const float KeyBTime = BufferCurveAccess.GetTime(First);
    return FMath::Lerp(KeyATime, KeyBTime, Alpha);
  }

  return 0.f;
}
```
---
### `SetPlayrateToMatchSpeed()`
动态修改Sequence Player的 playRate, Lyra中只用于Cycle状态
1. 计算 animSpeed = Anim序列起始到终点的Root位移差 / Anim序列时长
2. 计算 desiredPlayRate = SpeedToMatch(`DisplacementSpeed`) / animSpeed
3. Clamp后 Set

```cpp
FSequencePlayerReference UAnimDistanceMatchingLibrary::SetPlayrateToMatchSpeed(const FSequencePlayerReference& SequencePlayer, float SpeedToMatch, FVector2D PlayRateClamp)
```



