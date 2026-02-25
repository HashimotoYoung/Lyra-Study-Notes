# UE Transform 基础

## Rotation 相关:
### 1. 转换 Vector 到不同坐标系

一句话总结: Rotate **Local** to World, Unrotate **World** to Local

- **`FRotatorB->RotateVector( LocalVecA )`:**
  // 施加 B 旋转 to vector A (local space)
  // Returns the result (world space)

- **`FRotatorB->UnrotateVector( WorldVecA )`:** 
  // 抵消 B 旋转 for vector A (world space)
  // Returns the result in **local space** relative to B.

#### 和Unity的对比

Rotate **Local** to World
- UE: `WorldVelocity = LocalVelocity >> ActorWorldRotation`
- Unity: `Vector3 world = transform.rotation * local;`

Unrotate **World** to Local

- `UE版本: LocalVelocity = WorldVelocity << ActorWorldRotation`
  `Unity版本: Vector3 local = Quaternion.Inverse(transform.rotation) * world;`
  
##### SourceCode:
```cpp
FVector UKismetMathLibrary::GreaterGreater_VectorRotator(FVector A, FRotator B)
{
    return B.RotateVector(A);
}

FVector UKismetMathLibrary::LessLess_VectorRotator(FVector A, FRotator B)
{
    return B.UnrotateVector(A);
}
```
---
### 2. 计算速度方向相关

### `UKismetAnimationLibrary::CalculateDirection(const FVector& Velocity, const FRotator& BaseRotation)` 
计算 **速度相对于朝向** 的角度, it tells if the object moves to the **left or right of its forward direction**

- **返回值:** 范围(-180,180)的float值, 例如 90 对应 right 方向
- :warning: 参数 Velocity 和 Rotator, should be in the **same coordinate space**

```cpp
//假设 BaseRotation 为 actor's worldRotation
// Velocity 为 actor's worldVelocity
float UKismetAnimationLibrary::CalculateDirection(const FVector& Velocity, const FRotator& BaseRotation)
{
	if (!Velocity.IsNearlyZero())
	{
		//rotator 等于 rotation matrix
		const FMatrix RotMatrix = FRotationMatrix(BaseRotation);

		//提取 worldForward
		const FVector ForwardVector = RotMatrix.GetScaledAxis(EAxis::X);

		//提取 worldRight
		const FVector RightVector = RotMatrix.GetScaledAxis(EAxis::Y);
		const FVector NormalizedVel = Velocity.GetSafeNormal2D();

		// Is roughly forward 结果区间为(-1,1)
		const float ForwardCosAngle = static_cast<float>(FVector::DotProduct(ForwardVector, NormalizedVel));
		//将(-1,1)映射到Degree区间(0,180)
		float ForwardDeltaDegree = FMath::RadiansToDegrees(FMath::Acos(ForwardCosAngle));

		// depending on where right vector is, flip it
		const float RightCosAngle = static_cast<float>(FVector::DotProduct(RightVector, NormalizedVel));
		if (RightCosAngle < 0.f)
		{
			ForwardDeltaDegree *= -1.f;
		}

		//结果为 +/-(0,180) 
		return ForwardDeltaDegree;
	}

	return 0.f;
}
```
---
### 3. FRotator 和 FVector 转换
- 假设 `BaseRay` 方向 = North-East and slightly upward
- 则 `BaseRay.Rotation()` represents the specific orientation , 返回 FRotator(Pitch=20°, Yaw=45°, Roll=0°)

- 这里 `BaseRayLocalUp`, `BaseRayLocalFwd`, `BaseRayLocalRight`的**皆为单位向量**
  - `BaseRayLocalFwd` == `BaseRay.GetSafeNormal()` == `BaseRotator.Vector()`

```cpp
FVector BaseRay = CameraLoc - SafeLoc;
FRotator BaseRotator = BaseRay.Rotation();
FRotationMatrix BaseRayMatrix(BaseRotator);

FVector BaseRayLocalUp, BaseRayLocalFwd, BaseRayLocalRight;
BaseRayMatrix.GetScaledAxes(BaseRayLocalFwd, BaseRayLocalRight, BaseRayLocalUp);
```
