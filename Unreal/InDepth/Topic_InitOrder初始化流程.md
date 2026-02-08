# 总初始化流程

## `GEngineLoop.PreInit()` 流程

1. Loads up **core engine modules** like `"CoreUObject"`,`"Engine"`,`"Render"`
<br>

2. Loads up **project/plugin modules**, when project module is loaded:
    - Engine frist registers any `UObject` classes defined, make reflection system aware of these classes
    - Builds up **CDO** for each Class
<br>

3. Calls modules' `StartupModule()` functions
<br>

4. Some **configuration file** loading happens: `Engine.ini`, `DefaultGame.ini` etc.
    - :warning: full project config (`DefaultGame.ini`) is typically read later in `GEngine->Init()` when `LoadConfig()` is called.

---

## `GEngineLoop.Init()` 流程

首先创建 **UGameEngine**: `NewObject<UEngine>(GetTransientPackage(), EngineClass)`

### `UGameEngine::Init()` 
调用父类 `UEngine::Init()`
  - Core rendering pipeline Setup
  - Asset Management: `UAssetManager::StartInitialLoading()`
      - 标记为 *startup loading* 的 primary assets 开始加载
  - 初始化子系统: `UEngineSubsystem`

**SPAWN** 并 Initilaize 三个关键实例: **UGameInstance** / **UGameViewportClient** / **ULocalPlayer**
- `GameInstance = NewObject<UGameInstance>(this, GameInstanceClass)`
	`GameInstance->InitializeStandalone()`
  - `UGameInstance::Init()`
- `CreateGameViewport( ViewportClient )`
  `ViewportClient->SetupInitialLocalPlayer()`
  - `UGameInstance::CreateLocalPlayer(...)`
<br>

### `UGameEngine::Start()`

- 调用 `GameInstance::StartGameInstance()`
  - 该阶段主要步骤之一为加载Level,  该方法可分为以下四步:
  ```cpp
  UEngine::LoadMap(FWorldContext& WorldContext, FURL URL, 
  class UPendingNetGame* Pending, 
  FString& Error)
  ```

#### Step 1. 构建 UWorld 并初始化

如果 `WorldContext.World()` 不为空 => Try **Unload** 当前 UWorld 

重建 UWorld:

- Firstly calls `LoadPackage()` to read the `.umap` file from disk into memory as a `UPackage`
- It then uses the package data to reconstruct the UWorld Object in memory
- 重建阶段会 **SPAWN PersistentLevel** 和该level内的 `Actors` and `ActorComponents`
  - call `UWorld::PostLoad()` 

设置 World 并初始化:
- call `WorldContext.SetCurrentWorld(NewWorld)` and  `UWorld::InitWorld()`
  - 初始化子系统: `UWorldSubsystem`
- **if** NetMode 不是客户端模式, **SPAWN GameModeBase**   
  // `UGameInstance::CreateGameModeForURL(FURL InURL, UWorld* InWorld)`


Fully Loads the Map...
<br>

#### Step 2. 初始化 Actors in World: `UWorld::InitializeActorsForPlay()`

初始化 GameMode: `GameMode::InitGame()`
- **SPAWN GameSession**

foreach level in World.Levels, call **`ULevel::RouteActorInitialize(0)`**
> :star:该方法的内部实现值得借鉴
  - 遍历 Actors in the Level, call `PreInitializeComponents()`
    - `AGameModeBase` 负责 **SPAWN GameStateBase**
  - 二次遍历 Actors in the Level,  call `InitializeComponents()`, then `PostInitializeComponents()`
<br>

#### Step 3. Local Player 登录

- 执行 [Spawn Play Actor 流](#spawn-play-actor-流) for all active local players
<br>

#### Step 4. `UWorld::BeginPlay()`
- 这里双端的 Being Play 流程 [Diverse](#begin-play-双端流程)
---
### Begin Play 双端流程

> :memo: In Unreal, Actors and their Components replicate separately

#### Server 端:
`UWorld::BeginPlay()`  世界启动 

=> `AGameModeBase::StartPlay()`
- Client 端没有GameMode, 因此会**截止在这一步**

=> `AGameState::HandleBeginPlay()` 
- :star: **Replicate `bReplicatedHasBegunPlay = true`**

=> `AWorldSettings::NotifyBeginPlay()`
- 遍历World中的 All Actors, 调用 **`AActor::DispatchBeginPlay(true)`**
<br>

#### Client 端:

`UWorld::BeginPlay()` 

**[OnRep] `bReplicatedHasBegunPlay`:** `AGameStateBase::OnRep_ReplicatedHasBegunPlay()`

=> `AWorldSettings::NotifyBeginPlay()`
- :star: 遍历World中的 ***Level / Replicated*** Actors (**会 miss 尚未同步到客户端的 actors**), 调用: 

=> **`AActor::DispatchBeginPlay(true)`:**
  - 标记 `ActorHasBegunPlay` 属性为 `EActorBeginPlayState::BeginningPlay`
  - Collect ***Replicated*** Components, then call `UActorComponent::BeginPlay()` ???
  - 通知蓝图 *Event_BeginPlay* 
  - 标记 `ActorHasBegunPlay` 属性为 `EActorBeginPlayState::HasBegunPlay`


> :books: **对于后续到达的 Actors/Components:** 任何在`NotifyBeginPlay()`之后才被同步和实例化的 Actors/Components，会在实例化完成时检查`if World->bBegunPlay == true`，并调用`DispatchBeginPlay()`来 “追上” 世界进度

---

### Spawn Play Actor 流

在 Server 端处理登录流程, 并生成 PlayerController, PlayerState, PCM

```cpp
ULocalPlayer::SpawnPlayActor(const FString& URL,FString& OutError, UWorld* InWorld)
```

- **if** *InWorld* 非 `NM_Client` 模式:
`UWorld::SpawnPlayActor(UPlayer* NewPlayer...)`
  - 开始 Login 流程: `APlayerController* newPC1 = AGameModeBase::Login()`

    - `AGameModeBase::SpawnPlayerControllerCommon()`
      - **SPAWN PlayerController** 
        // `APlayerController* NewPC = UWorld::SpawnActor<APlayerController>()`
        // `UGameplayStatics::FinishSpawningActor(NewPC)`
        - `AActor::FinishSpawning()`
          - `AActor::PostActorConstruction()`
            - `APlayerController::PostInitialzieComponents()`
              - **SPAWN PlayerState**
              - **SPAWN PlayerCameraManager**

  - Login 完成后会返回新建的 PlayerController, 进行初始化: 
    - `pc->SetPlayer(NewPlayer)` 
  - 开始 Post Login 流程: `AGameModeBase::PostLogin(pc)`
    - Try spawn a default Pawn for playerController to Possess
    - 进入[ Restart Player ](#restart-player-流)流
- **else**
  todo...

---

### Restart Player 流

GameModeBase的重要流程之一
生成 DefaultPawn for PC; Controller Possesion; 注册 Component;

- `AGameModeBase::RestartPlayer(AController* NewPlayer)`
  `RestartPlayerAtPlayerStart(NewPlayer, AActor* StartSpot)`
  - `APawn* NewPawn = SpawnDefaultPawnFor(NewPlayer, StartSpot)`
    // 默认会用到 StartSpot
    - `SpawnDefaultPawnAtTransform_Implementation()` // 常用于Override
      - `UClass* PawnClass = GetDefaultPawnClassForController(NewPlayer)`
      // 默认情况下, 该方法不会用到 Controller 参数, 直接返回成员变量`DefaultPawnClass`
        **`UWorld::SpawnActor(PawnClass)`**
        - `AActor::PostSpawnInitialize()`
          - `RegisterAllComponents()`
            - `UActorComponent::RegisterComponentWithWorld()` 
            // 使用频率高, 在很多地方被调用
              - **`OnRegister()`** // 常用于Override
  - if `IsValid(NewPawn)`
    `NewPlayer->AController::SetPawn(APawn* InPawn)` 
    //绑定 controller 和 pawn
  - if `IsValid(NewPlayer->GetPawn()) == FALSE`
    then `FailedToRestartPlayer()`
    // 失败
    else `FinishRestartPlayer()`
      - `AController::Possess(APawn* InPawn)`
        - **`APawn::PossessedBy(AController* NewController)`**
    
---