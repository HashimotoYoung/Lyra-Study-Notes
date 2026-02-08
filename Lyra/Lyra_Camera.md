todo: 相机Blend算法

[Nancy Camera Ease](https://github.com/NanceDevDiaries/Tutorials/tree/main/CameraRotationLag)

# Lyra Camera 
---

### ULyraCameraMode : UObject

相机模式类, 负责每帧 Update 其View属性和Blend相关属性

##### 类关系:
- Contained by `ULyraCameraModeStack` 
- Outer is `ULyraCameraComponent`
##### 主要属性:
`float BlendTime` 
`float BlendWeight` 
- Blend 相关属性

**`FLyraCameraModeView View`** :star:
- 简化版POV数据, 也定义了 CameraMode 系统会修改哪些值; 每帧更新

  ```cpp
  struct FLyraCameraModeView
  {
  public:
    FLyraCameraModeView();
    void Blend(const FLyraCameraModeView& Other, float OtherWeight);
  public:
    FVector Location;
    FRotator Rotation;
    FRotator ControlRotation;
    float FieldOfView;
  };
  ```
##### 主要方法:
#### `void UpdateCameraMode(float DeltaTime)`
Update默认接口; **非Virtual**; 见 [UpdateCameraMode流](#updatecameramode-流)
```cpp
//获取相机起始支点, 用于计算相机位置
virtual FVector GetPivotLocation() const;
virtual FRotator GetPivotRotation() const;
virtual void UpdateView(float DeltaTime);
virtual void UpdateBlending(float DeltaTime);
```
<br>

### ULyraCameraMode_ThirdPerson 

继承自 `ULyraCameraMode`, 第三人称模式

##### 主要属性:

`FVector CurrentCrouchOffset` //每帧插值更新
`float CrouchOffsetBlendPct` //蹲姿Offset融合百分比,每帧增加
`TObjectPtr<const UCurveVector> TargetOffsetCurve`
- 曲线资产引用, 用于计算相机位置
- Lyra中对应资产为 "ThirdPersonOffsetCurve"
<br>

##### 主要方法:
#### `virtual UpdateView(float DeltaTime) override`



---

### ULyraCameraModeStack : UObject

CameraMode Container; 负责生成并持有CameraMode, 实现 CameraMode 混合算法

##### 主要属性:

`TArray< TObjectPtr<ULyraCameraMode> > CameraModeInstances`
- 缓存所有用到的CameraMode

`TArray< TObjectPtr<ULyraCameraMode> > CameraModeStack`
- 动态管理 "Active" 的 CameraMode, 会主动清理; index_0 代表Stack的Top
<br>

##### 主要方法:
#### `EvaluateStack(float DeltaTime, FLyraCameraModeView& OutCameraModeView)` 
Update时调用, 计算 CameraModeView

#### `PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass)` 
ExistingStackContribution的算法值得学习

#### `BlendStack(FLyraCameraModeView& OutCameraModeView)`

---

### ULyraCameraComponent : UCameraComponent

通过 Override `GetCameraView()` 实现了一套 Stack-CameraMode 的相机控制模式，实现了更为精细的碰撞检测以及数据驱动

##### 类关系:
- Contains a `ULyraCameraModeStack`
- :memo: **Is the Outer of `ULyraCameraModeStack` and `ULyraCameraMode`**

##### 主要属性:

`CameraModeStack : TObjectPtr<ULyraCameraModeStack>`
- `LyraCameraComponent` 不直接持有 CameraMode, 而是使用 Stack 进行一层封装

`DetermineCameraModeDelegate : FLyraCameraModeDelegate`
- BindTo => `TSubclassOf<ULyraCameraMode> ULyraHeroComponent::DetermineCameraMode()`
- 通过该委托将 CameraMode 的切换权给 `LyraHeroComponent`
<br>

##### 主要方法:

#### `GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView) override` 
见 [GetCameraView 流](#getcameraview-流)

---

## 主要流程

### GetCameraView 流
> 核心Override方法, 涵盖了每帧相机的更新逻辑 

1: 检测当前相机模式: `UpdateCameraModes()`
- 通过委托向 `LyraHeroComponent` 请求一个 `TSubclassOf<ULyraCameraMode>` 
- 从本地缓存中获取或创建对应的 `CameraMode` instance 
- Push到 *`CameraModeStack`*, 入栈时计算初始 Blend 权重


2: 定义默认视图数据(`FLyraCameraModeView CameraModeView`), 开始执行 Lyra Camera Mode 计算流程: `CameraModeStack->EvaluateStack(DeltaTime, CameraModeView)`

- **Update Stack:** 从栈顶开始(*index=0*), **遍历** member *`TArray<TObjectPtr<ULyraCameraMode>>`*
  - 执行 [CurCamraMode->UpdateCameraMode流](#updatecameramode-流) 
  - **if** `CurCamraMode.BlendWeight >= 1`, **break** 并删除后续CM 
- **Blend Stack:**
  - todo...

3: 至此 CameraModeView 已更新完毕, 进行三项后处理:
- 同步 Rot 至 `PlayerController.ControlRotation` 
  - **当前相机朝向决定控制器的朝向**
- 同步 Pos/Rot/FOV 至 `LyraCameraComponent`
  - :pushpin: 必要步骤, 很多模块只查询 `CameraCompoent` 的 Transform
- 填充 *DesiredView* 的各项属性
  > *DesiredView* 最终被 `PlayerCameraManager` 用于渲染 POV

---

### UpdateCameraMode 流
> 以 ULyraCameraMode_ThirdPerson 为例

1: **更新 *View* 中的各项属性:** `virtual UpdateView(deltaTime)`

- 通过 Outer 获取 **相机Target**(`ACharacter*`), 检查是否下蹲并更新 ***CurrentCrouchOffset***
  - `let CrouchedHeightAdjustment = CharaCDO->CrouchedEyeHeight - CharaCDO->BaseEyeHeight`

(1): 获取相机基准位置和旋转

- 获取 **PivotLocation**
  - Lyra 在这里会添加Offset, 以确保 PivotLocation **不受Capsule形变的影响** (例如下蹲)
  ```cpp
  const float DefaultHalfHeight = CapsuleCompCDO->GetUnscaledCapsuleHalfHeight();
  const float ActualHalfHeight = CapsuleComp->GetUnscaledCapsuleHalfHeight();
  const float HeightAdjustment = (DefaultHalfHeight - ActualHalfHeight) + TargetCharacterCDO->BaseEyeHeight;
  return TargetCharacter->GetActorLocation() + (FVector::UpVector * HeightAdjustment);
  ```
- 获取 **PivotRotation** = `Target->GetViewRotation()`
  - 会对 `PivotRotation.Pitch` 进行 Clamp
- 赋值 *View* => `View.Location = PivotLocation + CurrentCrouchOffset`

(2): 调整相机位置 (曲线)
  - :star: 读取曲线配置; **Add "Pitch-Based" Curve Offset** to *View* 

    ```cpp
    if (TargetOffsetCurve) {
      const FVector TargetOffset = TargetOffsetCurve->GetVectorValue(PivotRotation.Pitch);
      //转换为世界坐标
      View.Location = PivotLocation + PivotRotation.RotateVector(TargetOffset);
    }
    ```

(3): 调整相机位置 (防穿透): `UpdatePreventPenetration(DeltaTime)`
- 定义 SafeLocation = 玩家位置(`GetActorLocation()`)
- 找出 View's AimLine 上距离 SafeLocation 最近的投影点: `ClosestPointOnLineToCapsuleCenter`
- 再次赋值 SafeLocation = "Capsule上距离 `ClosestPointOnLineToCapsuleCenter` 最近的点"
  - 通过 `UPrimitiveComponent
::GetSquaredDistanceToCollision()` 方法
- :star: 最后以该点为起始, **围绕方向 (`View.Location - SafeLocation`) 进行多次 *SphereCast*, 并根据 Min Hit Distance 调整 View.Location**

**2: 更新计算 *BlendWeight* 属性:** `virtual UpdateBlending(deltaTime)`
