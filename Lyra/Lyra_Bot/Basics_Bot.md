# Lyra AI分析


### `ULyraBotCreationComponent`

负责处理游戏中的Bot生成, 继承自 `UGameStateComponent`


##### 类关系:
##### 主要属性:
##### 主要方法:

#### `OnExperienceLoaded(const ULyraExperienceDefinition* Experience)`
依赖Experience; 如果 `HasAuthority()`, 生成Bots
<br>

#### :star: `SpawnOneBot()`: 

一个典型的生成Bot方法, 使用`FActorSpawnParameters`, 值得参考

- 先生成 AIController
  - :warning: 养成使用 `SpawnInfo.ObjectFlags |= RF_Transient` 的习惯
   ```cpp
   FActorSpawnParameters SpawnInfo;
   //碰撞相关配置, 即使有碰撞也生成
   SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
   //确保生成在当前的 ULevel 中
   SpawnInfo.OverrideLevel = GetComponentLevel();
   //添加重要标记: 不要存储 
   SpawnInfo.ObjectFlags |= RF_Transient;

   AAIController* NewController = GetWorld()->SpawnActor<AAIController>(BotControllerClass, FVector::ZeroVector, FRotator::ZeroRotator, SpawnInfo);
   ```
- :memo: **`APlayerState` 会随着 `AController` 一起生成**
   ```cpp
   ALyraGameMode* GameMode = GetGameMode<ALyraGameMode>();
   check(GameMode);

   if (NewController->PlayerState != nullptr)
   {
      //设置名称
      NewController->PlayerState->SetPlayerName(CreateBotName(NewController->PlayerState->GetPlayerId()));
   }

   // Lyra-override方法, 这里会通知 ULyraTeamCreationComponent 回调
   GameMode->GenericPlayerInitialization(NewController);

   // RestartPlayer !!!
   GameMode->RestartPlayer(NewController);

   // 推进Lyra PawnExt初始化流程
   if (NewController->GetPawn() != nullptr)
   {
      if (ULyraPawnExtensionComponent* PawnExtComponent = NewController->GetPawn()->FindComponentByClass<ULyraPawnExtensionComponent>())
      {
         PawnExtComponent->CheckDefaultInitialization();
      }
   }
   //本地记录
   SpawnedBotList.Add(NewController);
   ```