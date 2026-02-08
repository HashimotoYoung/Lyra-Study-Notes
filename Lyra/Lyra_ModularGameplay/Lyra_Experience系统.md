
FrontEnd 如何告知gamemode to start experience?



### FAQ:
*1. 为什么 `ULyraExperienceManagerComponent` act as a singleton-like manager 而要继承  `UGameStateComponent`?*

- 共享GameState的 lifetime, make it acts like a **singleton per-world**
- 利用 Replication:    
  `UPROPERTY(ReplicatedUsing = OnRep_CurrentExperience)`
  `TObjectPtr<const ULyraExperienceDefinition> CurrentExperience`
<br>

*2. Experience 和 GameFeature 的关系是什么?*

- **"Use a"** 关系, 每一个 Experience 需要一或多个GameFeature来支持玩法, 
  - Experience 仅记录GF的名称(FString), 例如 "ShooterCore"
- :books: GameFeatures 是 Project 的 "玩法基石" **(Reusable building blocks of gameplay systems)**
<br>

*3. 为何在 "L_DefaultEditorOverview" 关卡中 PIE 时, Client模式不生效 (always Standalone模式) ?*
- Lyra自定义世界设置 (`ALyraWorldSettings:AWorldSettings`) 里的 	`bool ForceStandaloneNetMode` 设置项会在 PreCreatePIEInstances 阶段 override PlayMode


---

# Lyra Experience
> [How Experiences are Loaded](#load-experience-流) 

Lyra ModularGameplay的三大系统之一, 但实际上是相对独立的系统, **不依赖**其它两个系统

Experience 可视为一个 “Advanced Game Mode”, Lyra 为每种 Experience 配套了一个 `ULyraExperienceDefinition:UPrimaryDataAsset` , contains:
1.  支持其玩法所需要的 [GameFeatures](./Lyra_GameFeature.md): `TArray<FString> GameFeaturesToEnable`
2. Player 配置: `TObjectPtr<const ULyraPawnData> DefaultPawnData`
3. 专属于 Experience 的 GFA: `TArray<TObjectPtr<UGameFeatureAction>> Actions`
4. 相关 "ActionSet" 引用: `TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets`

#### 相关特性:

- A level in Lyra **doesn't have a traditional Game Mode**. Instead, it has a "Default Experience"(在World Setting中设置)
- Experience 在设计层面上是 **Decoupled** with ULevel 
  - 但也需要一些特定Actor 例如 PlayerStart
- In Lyra, the actual start of Gameplay is **delayed until** `OnExperienceLoaded`, usually long after `BeginPlay`.

---

### Experience 配置示例(`B_LyraShooterGame_ControlPoints`)

1: GameFeautresToEnable
  - GF名称: "ShooterCore"

2: DefaultPawnData (**`LyraPawnData`**) // :star: **纯配置, 无GFA参与**
  - "HeroData_ShooterGame" 
    - 指定的 Pawn Class: `B_Hero_ShooterMannequin`
    - `LyraAbilitySet` 资产 : "AbilitySet_ShooterHero"
      - **11 个 Granted Gameplay Abilities** ([Lyra GA Config](#lyra-ga-配置项))
"GA_Hero_Jump", "GA_Reset", "GA_SpawnEffect" 等...
      - **1个 Granted Gameplay Effect**: "GE_IsPlayer"
    - `LyraAbilityTagRelationshipMapping` 资产
    -  `LyraInputConfig` 资产: "InputData_Hero"
       - **5个 Native `InputAction`, 6个 Ability `InputAction`**  
    - `LyraCameraMode`: "CM_ThirdPerson" [Intro](../Lyra_Camera.md#ulyracameramode--uobject)

3: ActionSets ( **三份`LyraExperienceActionsSet` 资产** ): // **基于GFA**
  - "LAS_ShooterGame_SharedInput" 
    - *GFA_AddInputBinds*:
      - `LyraInputConfig` 资产: "InputData_ShooterGame_AddOns"
        - **6个 Ability `InputAction` 资产** 
    - *GFA_AddInputConfig*:
      - **4个 `PlayerMappableInputConfig` 资产**
      <br>
  - "LAS_ShooterGame_StandardComponents" 
    - *GFA_AddComponents* **共4个**:
      - Two for `LyraPlayerController`
      - `B_QuickBarComponent` for `Controller`
      - `NameplateSource` for `B_Hero_ShooterMannequin`
      <br>
  - "LAS_ShooterGame_SharedHUD"
    - *GFA_AddWidgets*
      - **1个 `ULyraHUDLayout`**: "W_ShooterHUDLayout"
      - **11个 `Widgets`**: "W_EliminationFeed" 等...

4: Experience专属 Actions: 
  - *GFA_AddWidgets*:
    - **2个 `Widgets`**: "W_ControlPointStatusWidget", "W_CPScoreWidget"
  - *GFA_AddAbilities*:
    - `LyraAbilitySet` 资产: "AbilitySet_ControlPoint"
      - **2个 Granted Gameplay Abilities**: "GA_ShowLeaderboard_CP", "GA_AutoRespawn"
  - *GFA_AddComponents*:
    - **5个 `UGameStateComponent`** for `LyraGamState`: "B_ControlPointScoring", "B_TeamSetup_TwoTeams" 等...

    - **1个 `UControllerComponent`**: "B_PickRandomCharacter"
<br>

#### Lyra GA Config
```cpp
USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayAbility
{
  TSubclassOf<ULyraGameplayAbility> Ability = nullptr;  // 类引用
  int32 AbilityLevel = 1;  // 等级
  FGameplayTag InputTag;  // 和GA绑定的Input数据
};
```

---

## Load Experience 流

1: Start on Server, One frame after  `ALyraGameMode::InitGame()` 
- 获取 ExperienceId (`FPrimaryAssetId`) 
- 设置 Current Experience: `ULyraExperienceManagerComponent::SetCurrentExperience(FPrimaryAssetId ExperienceId)`
  - 获取对应的 `ULyraExperienceDefinition` 配置数据, Set and **Replicate** 
  
    ```cpp
    UPROPERTY(ReplicatedUsing=OnRep_CurrentExperience)
    TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;
    ```
  - Server 开始 Load Experience: `StartExperienceLoad()`

2: Client 在 ***OnRep_CurrentExperience*** 后, 开始 Load Experience
- 切换状态为: *ELState::Loading* 
- **开始异步加载 Assets** associated with Experience Definition, 并注册回调 `OnExperienceLoadComplete()`

3: After Callback, 开始 Load Game Features (*异步计数法*)
- 收集所有需要加载的GF名称, 获取其 URL, 记录总数: `NumGameFeaturePluginsLoading = GameFeaturePluginURLs.Num()` 
- **if** `NumGameFeaturePluginsLoading` > 0  
  - 切换状态为: *ELState::LoadingGameFeatures*
  - **开始异步加载并Activate 所有 GameFeautres:** 
  // `UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin()`
  - 一个 GF 在 Activated 后, 会执行回调并 `--NumGFPLoading`
    when `NumGFPLoading == 0`, 代表 Exp 已被完全加载, 执行总回调

4: 当 ALL Experience 加载完成后 (`OnExperienceFullLoadCompleted()`)
- 切换状态为: *ELState::ExecutingActions*
- **Activate *GameFeautreActions* in the Experience**
  Activate *GameFeautreActions* in the Experience's ActionsSets

- 切换状态为: *ELState::Loaded*
- 按优先级顺序, **Broadcast 三种 `OnExperienceLoaded` delegates**

<br>

## Unload Experience 流
:star: 在 `ULyraExperienceManagerComponent::EndPlay()` 时执行, 因此将 GameFeature 限定为了一个 **World层** 的功能

- 卸载 **所有由当前Exp加载的 GFs**
  `UGameFeaturesSubsystem::Get().DeactivateGameFeaturePlugin(GFPluginURL)`
- 卸载 由当前Exp加载的 GFA 和 ActionSets