# Lyra AI分析

### ULyraBotCreationComponent : UGameStateComponent

负责生成游戏内 Bot

##### 主要属性:

`TSubclassOf<AAIController> BotControllerClass`  
实际值为 "B_AI_Controller_LyraShooter"
- 继承自 `ALyraPlayerBotController` << `AModularAIController` << `AAIController`
- :pushpin: `ALyraPlayerBotController` 在构造函数中设置 `bWantsPlayerState = true`, 因此会生成 PlayerState

##### 主要方法:

#### `OnExperienceLoaded(const ULyraExperienceDefinition* Experience)`
Exp 加载完成时回调
-  **if** *Authority*, 生成Bots: `SpawnOneBot()`

<br>

#### `SpawnOneBot()` :star:

典型的生成Bot方法, 使用到 `FActorSpawnParameters`, 值得参考

  - 养成使用 `SpawnInfo.ObjectFlags |= RF_Transient` 的习惯

```cpp
void ULyraBotCreationComponent::SpawnOneBot()
{
   FActorSpawnParameters SpawnInfo;

   //碰撞相关, 即使有碰撞也生成
   SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
   //确保生成在当前的 ULevel 中
   SpawnInfo.OverrideLevel = GetComponentLevel();
   //添加标记: 不要存储 
   SpawnInfo.ObjectFlags |= RF_Transient;

   //先生成 BotControllerClass
   AAIController* NewController = GetWorld()->SpawnActor<AAIController>(BotControllerClass, FVector::ZeroVector, FRotator::ZeroRotator, SpawnInfo);

   ALyraGameMode* GameMode = GetGameMode<ALyraGameMode>();
   check(GameMode);

   if (NewController->PlayerState != nullptr) {
      //设置名称
      NewController->PlayerState->SetPlayerName(CreateBotName(NewController->PlayerState->GetPlayerId()));
   }

   // Lyra-override方法, 这里会通知 ULyraTeamCreationComponent 回调
   GameMode->GenericPlayerInitialization(NewController);

   // RestartPlayer !!!
   GameMode->RestartPlayer(NewController);

   // 推进Lyra PawnExt初始化流程
   if (NewController->GetPawn() != nullptr) {
      if (ULyraPawnExtensionComponent* PawnExtComponent = NewController->GetPawn()->FindComponentByClass<ULyraPawnExtensionComponent>()){
         PawnExtComponent->CheckDefaultInitialization();
      }
   }

   // Cache
   SpawnedBotList.Add(NewController);

}
```