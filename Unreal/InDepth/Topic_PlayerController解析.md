## PlayerController 解析

##### 主要属性:

#### `FRotator ControlRotation`
- 世界坐标系, 代表控制器旋转

#### `FRotator RotationInput`
- 缓存每帧旋转相关的输入, tick结束前清空
  主要用于`UpdateRotation()`

##### 主要方法:

### `TickActor()`

- 检查 PlayerInput 并调用 `PlayerTick(float DeltaTime)`
    - 会触发Input相关的Tick和callback, 例如:
    `ULyraHeroComponent::Input_LookMouse()`
      => `Pawn->AddControllerYawInput(Value.X)`;
         
- 处理 PCM 的 TargetView


```ts
//APlayerController::TickActor( float DeltaSeconds, ELevelTick TickType, FActorTickFunction& ThisTickFunction )

const bool bIsClient = IsNetMode(NM_Client);
const bool bIsLocallyControlled = IsLocalPlayerController();

//server端 check
if ((GetRemoteRole() == ROLE_AutonomousProxy) && !bIsClient && !bIsLocallyControlled)
{
    //移动相关 先忽略
    if (IsValid(GetPawn()) && GetPawn()->GetRemoteRole() == ROLE_AutonomousProxy && GetPawn()->IsReplicatingMovement())
    {
        UMovementComponent* PawnMovement = GetPawn()->GetMovementComponent();
        INetworkPredictionInterface* NetworkPredictionInterface = Cast<INetworkPredictionInterface>(PawnMovement);
        if (NetworkPredictionInterface && IsValid(PawnMovement->UpdatedComponent))
        {
            FNetworkPredictionData_Server* ServerData = NetworkPredictionInterface->HasPredictionData_Server() ? NetworkPredictionInterface->GetPredictionData_Server() : nullptr;
            if (ServerData)
            {
                UWorld* World = GetWorld();
                if (ServerData->ServerTimeStamp != 0.f)
                {
                    const float WorldTimeStamp = World->GetTimeSeconds();
                    const float TimeSinceUpdate = WorldTimeStamp - ServerData->ServerTimeStamp;
                    const float PawnTimeSinceUpdate = TimeSinceUpdate * GetPawn()->CustomTimeDilation;
                    // See how long we wait to force an update. Setting MAXCLIENTUPDATEINTERVAL to zero allows the server to disable this feature.
                    const AGameNetworkManager* GameNetworkManager = (const AGameNetworkManager*)(AGameNetworkManager::StaticClass()->GetDefaultObject());
                    const float ForcedUpdateInterval = GameNetworkManager->MAXCLIENTUPDATEINTERVAL;
                    const float ForcedUpdateMaxDuration = FMath::Min(GameNetworkManager->MaxClientForcedUpdateDuration, 5.0f);

                    // If currently resolving forced updates, and exceeded max duration, then wait for a valid update before enabling them again.
                    ServerData->bForcedUpdateDurationExceeded = false;
                    if (ServerData->bTriggeringForcedUpdates)
                    {
                        const float PawnTimeSinceForcingUpdates = (WorldTimeStamp - ServerData->ServerTimeBeginningForcedUpdates) * GetPawn()->CustomTimeDilation;
                        if (PawnTimeSinceForcingUpdates > ForcedUpdateMaxDuration * GetPawn()->GetActorTimeDilation())
                        {
                            if (ServerData->ServerTimeStamp > ServerData->ServerTimeLastForcedUpdate)
                            {
                                // An update came in that was not a forced update (ie a real move), since ServerTimeStamp advanced outside this code.
                                ServerData->ResetForcedUpdateState();
                            }
                            else
                            {
                                if (ServerData->bLastRequestNeedsForcedUpdates)
                                {
                                    // No valid updates, don't reset anything but don't mark as exceeded either
                                    // Keep forced updates going until new and valid move request is received
                                }
                                else
                                {
                                    // Waiting for ServerTimeStamp to advance from a client move.
                                    ServerData->bForcedUpdateDurationExceeded = true;
                                }
                            }
                        }
                    }
                    
                    const float CurrentRealTime = World->GetRealTimeSeconds();
                    const bool bHitch = (CurrentRealTime - LastMovementUpdateTime) > GameNetworkManager->ServerForcedUpdateHitchThreshold && (LastMovementUpdateTime != 0);
                    LastMovementHitch = bHitch ? CurrentRealTime : LastMovementHitch;
                    const bool bRecentHitch = bHitch || (CurrentRealTime - LastMovementHitch < GameNetworkManager->ServerForcedUpdateHitchCooldown);
                    LastMovementUpdateTime = CurrentRealTime;

                    // Trigger forced update if allowed
                    if (!bRecentHitch && ForcedUpdateInterval > 0.f && PawnTimeSinceUpdate > FMath::Max<float>(DeltaSeconds+0.06f, ForcedUpdateInterval * GetPawn()->GetActorTimeDilation()))
                    {
                        //UE_LOG(LogPlayerController, Warning, TEXT("ForcedMovementTick. PawnTimeSinceUpdate: %f, DeltaSeconds: %f, DeltaSeconds+: %f"), PawnTimeSinceUpdate, DeltaSeconds, DeltaSeconds+0.06f);
                        const USkeletalMeshComponent* PawnMesh = GetPawn()->FindComponentByClass<USkeletalMeshComponent>();
                        if (!ServerData->bForcedUpdateDurationExceeded && (!PawnMesh || !PawnMesh->IsSimulatingPhysics()))
                        {
                            const bool bDidUpdate = NetworkPredictionInterface->ForcePositionUpdate(PawnTimeSinceUpdate);

                            // Refresh this pointer in case it has changed (which can happen if character is destroyed or repossessed).
                            ServerData = NetworkPredictionInterface->HasPredictionData_Server() ? NetworkPredictionInterface->GetPredictionData_Server() : nullptr;

                            if (bDidUpdate && ServerData)
                            {
                                ServerData->ServerTimeLastForcedUpdate = WorldTimeStamp;

                                // Detect initial conditions triggering forced updates.
                                if (!ServerData->bTriggeringForcedUpdates)
                                {
                                    ServerData->ServerTimeBeginningForcedUpdates = ServerData->ServerTimeStamp;
                                    ServerData->bTriggeringForcedUpdates = true;
                                }

                                // Set server timestamp, if there was movement.
                                ServerData->ServerTimeStamp = WorldTimeStamp;
                            }
                        }
                    }
                }
                else
                {
                    // If timestamp is zero, set to current time so we don't have a huge initial delta time for correction.
                    ServerData->ServerTimeStamp = World->GetTimeSeconds();
                    ServerData->ResetForcedUpdateState();
                }
            }
        }
    }

    // 更新
    if (PlayerCameraManager != nullptr)
    {
        APawn* TargetPawn = PlayerCameraManager->GetViewTargetPawn();
        
        if ((TargetPawn != GetPawn()) && (TargetPawn != nullptr))
        {
            ??啥意思?
            TargetViewRotation = TargetPawn->GetViewRotation();
        }
    }
}
//这里是Client端
else if (GetLocalRole() > ROLE_SimulatedProxy)
{
    // Process PlayerTick with input.
    if (!PlayerInput && (Player == nullptr || Cast<ULocalPlayer>( Player ) != nullptr))
    {
        InitInputSystem();
    }

    if (PlayerInput)
    {
        //存在PlayerInput时 TickPlayerInput
        PlayerTick(DeltaSeconds);
    }

    ?? why
    if (!IsValid(this))
    {
        return;
    }

    // update viewtarget replicated info
    if (PlayerCameraManager != nullptr)
    {
        //如果PCM没有在看 playerPawn,转向PCM的ViewTarget?
        APawn* TargetPawn = PlayerCameraManager->GetViewTargetPawn();
        if ((TargetPawn != GetPawn()) && (TargetPawn != nullptr))
        {
            SmoothTargetViewRotation(TargetPawn, DeltaSeconds);
        }

        //?? 何时用到
        //playerPawn移动式,发送Camera
        if (bIsClient && bIsLocallyControlled && GetPawn() && PlayerCameraManager->bUseClientSideCameraUpdates)
        {
            UPawnMovementComponent* PawnMovement = GetPawn()->GetMovementComponent();
            if (PawnMovement != nullptr &&
                !PawnMovement->IsMoveInputIgnored() &&
                (PawnMovement->GetLastInputVector() != FVector::ZeroVector || PawnMovement->Velocity != FVector::ZeroVector))
            {
                PlayerCameraManager->bShouldSendClientSideCameraUpdate = true;
            }
        }
    }
}

if (IsValid(this))
{
    QUICK_SCOPE_CYCLE_COUNTER(Tick);
    Tick(DeltaSeconds);	// perform any tick functions unique to an actor subclass
}

// Clear old axis inputs since we are done with them.
RotationInput = FRotator::ZeroRotator;

if(!!NetworkPhysicsCvars::EnableNetworkPhysicsPrediction && GetLocalRole() == ROLE_AutonomousProxy && bIsClient)
{
    UpdateServerAsyncPhysicsTickOffset();
}

DONE
```
---

### `PlayerTick()`

--- 

### `UpdateRotation(float DeltaTime)`

**负责计算当前的 ControlRotation** 并 Apply to Pawn's Rotation
- 默认在 `PlayerTick()` 的最后执行

- :warning: Lyra 的 TPS 模式会走这里, 因此 Player Character 会一直跟着镜头旋转

:pencil2: **Start**  
1: 获取**当前帧的 `RotationInput`** 和 `ViewRotation = GetControlRotation()`, 交由 PCM 处理: `APlayerCameraManager::ProcessViewRotation()`
- 会先交给 member `UCameraModifier List`进行 process
- 然后做角度限制 `LimitViewPitch/Yaw/Roll()`

2: `SetControlRotation(ViewRotation)` 
- 如果 `Pawn != null` 则 `APawn::FaceRotation(FRotator ViewRotation, float DeltaTime)`
- **如果 Pawn 有设置 "基于Controller旋转"** like `bUseControllerRotationYaw = true`
  - 则直接在对应轴上 `SetActorRotation(ViewRotation)` 
  


