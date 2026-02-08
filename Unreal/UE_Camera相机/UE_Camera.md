# UE Camera

## Camera 核心类介绍

### PlayerCameraManager : AActor

Camera系统中的**导演Actor**, created and **owned** by `APlayerController`

- :star: **Decides where the player camera should be every frame** (location, rotation, FOV).

- 处理 interpolation between camera positions (例如第三人称到第一人称)

- Handles post-processing effects (fade in/out, screen shakes, camera effects).

- Interfaces with gameplay logic (zoom, aiming, cinematic views).


- 主要用法: Override `APlayerCameraManager` in Blueprint or C++, then implement `BlueprintUpdateCamera()` or `UpdateViewTargetInternal()` to define custom behavior

##### 主要属性:
`ViewTarget : FTViewTarget`
- 代表 PCM 的观察目标, 一般情况下指玩家角色
    ```cpp
    USTRUCT(BlueprintType)
    struct FTViewTarget
    {
        //实际观察对象
        TObjectPtr<class AActor> Target;
        //包含 Pos,Rot,FOV 等一系列代表相机视角的数据
        struct FMinimalViewInfo POV;
    }
    ```
##### 主要方法:
#### `virtual UpdateViewTarget(FTViewTarget& OutVT, float DeltaTime)`
更新计算 POV for ViewTarget, 详见 [Update PCM 流](#update-pcm-流)

--- 

### ACameraActor : AActor

代表一个 Level 中的实体相机, Has a `CameraComponent`
- 本质上是 a pure **standalone viewpoint** — not automatically tied to players.
- Often used with `APlayerController::SetViewTargetWithBlend()` to smoothly transition the player's view to its location.

---

### UCameraComponent : USceneComponent

本质上, CameraComponent is **a data container** for camera settings and **a spatial marker**

- One actor can have mutiple `UCameraComponents`
- Contains 摄像机相关属性 (Location/Rotation; FOV; ProjectionMode等)

详见 `ULyraCameraComponent`
 
---

## Camera 主要流程

### Update PCM 流
`PlayerCameraManager` 的核心 Tick 流程, 负责计算更新 **`ViewTarget.POV`** for 玩家相机

**Start with** `APlayerController::UpdateCameraManager(float delta)`
=> `APlayerCameraManager::UpdateCamera(float delta)`

1: 如果 `IsLocalPlayer || !bUseClientSideCameraUpdates` 则进行 Camera 更新
- 生成 NewPOV: `FMinimalViewInfo NewPOV = ViewTarget.POV;`
- 检测当前 *ViewTarget* 是否 Valid
  - if not, 会更换 *ViewTarget*
- 计算 *ViewTarget* 的 POV 值, **分三种情况处理**: `UpdateViewTarget(ViewTarget, DeltaTime)`

  1. **if** ViewTarget.Target 的类型为 `ACameraActor`,则立即调用: `CamActor->GetCameraComponent()->GetCameraView(DeltaTime, OutVT.POV);`
      
  2. **else if** `pcm.CameraStyle` 不为 "Default", 使用对应 built-in 处理
  3. **else** **交由 ViewTarget 内部处理:** `UpdateViewTargetInternal(FTViewTarget& OutVT, float DeltaTime) ` 
  // *Most Time*
     -  **if** 存在蓝图 Implemented 方法, 则优先使用BP
     -  :star: **else** 执行 Actor 的默认相机计算接口: `OutVT.Target->AActor::CalcCamera(float DeltaTime, FMinimalViewInfo& OutResult)`
         > Actor 获取其 **First Active CameraComponent** 并执行:
         `UCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)`
- 再次赋值: `NewPOV = ViewTarget.POV;`

2: 缓存当前帧的计算结果 `this->FillCameraCache(NewPOV)` 
3: 通知 Server 更新结果
// todo...

---