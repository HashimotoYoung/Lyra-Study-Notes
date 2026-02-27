### FAQ:
*1. 为什么 `ULyraExperienceManagerComponent` act as a singleton-like manager 而要继承  `UGameStateComponent`?*

- 共享GameState的 lifetime, make it acts like a **singleton per-world**
- For Replication:  
  ```cpp  
  UPROPERTY(ReplicatedUsing = OnRep_CurrentExperience)
  TObjectPtr<const ULyraExperienceDefinition> CurrentExperience
  ```
<br>

*2. Experience 和 Game Feature 的关系是什么?*

- "Use a" 关系, 每一个 Experience 需要一个或多个 GameFeatures 来支持玩法 
  - Experience 仅记录GF的名称(FString), 例如 "ShooterCore"
- 每个 Experience 自身包含一系列 GFAs


*3. 为何在 "L_DefaultEditorOverview" 关卡中 PIE 时, Client模式不生效 (always Standalone模式) ?*
- Lyra自定义世界设置 (`ALyraWorldSettings:AWorldSettings`) 里的 	`bool ForceStandaloneNetMode` 设置项会在 PreCreatePIEInstances 阶段 override PlayMode


---

## Lyra Experience

> :books: Lyra ModularGameplay 三大系统之一, 在设计上是相对独立的系统

Experience 可视为一个 “Advanced Game Mode”, 主要由两部分构成 `ULyraExperienceManagerComponent`
and `ULyraExperienceDefinition`

#### Features:
- **延迟启动:** The actual Start of Gameplay is **delayed until** `OnExperienceLoaded`, usually long after `BeginPlay()`.
- :memo: **游戏模式"组件化":** Lyra 弱化了传统 GameMode 的逻辑重心, 
`ALyraGameMode` 更像是逻辑“宿主/壳”, 玩法规则主要通过 Experience 注入（GameStateComponent、ControllerComponent 等）

- Experience 在设计上与 `ULevel` 弱耦（同一关卡可搭配不同 Experience）
  - 可能需要特定 Actor, 例如 `PlayerStart`

---
### ULyraExperienceManagerComponent : `UGameStateComponent`

Lyra Experience 的**代码逻辑层**实现
- 提供相关 public API 用于Experiecen加载完成后回调
  -  `CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate&& Delegate)`

- 负责处理加载/卸载 Experience

---
### ULyraExperienceDefinition : `UPrimaryDataAsset`

Lyra Experience 的**配置资产层**实现, 包含:
1.  支持其玩法所需要的 [Game Features 名称](./Lyra_GameFeature系统.md) : `TArray<FString> GameFeaturesToEnable`
2. 玩家 Pawn 的相关配置: `TObjectPtr<const ULyraPawnData> DefaultPawnData`
3. Experience 专属 GFAs: `TArray<TObjectPtr<UGameFeatureAction>> Actions`
4. Experience 专属 ActionSets: `TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets`

<br>

#### 配置示例 "B_LyraShooterGame_ControlPoints"

1: GameFeautresToEnable
  - GF名称: "ShooterCore"

2: DefaultPawnData (**`LyraPawnData`** , "HeroData_ShooterGame" ) // :pushpin: **纯配置, 无GFA参与**


- Character Class: `B_Hero_ShooterMannequin`
- `ULyraAbilitySet` 资产: "AbilitySet_ShooterHero"
  - 包含 **11 个 Granted Gameplay Abilities:** "GA_Hero_Jump", "GA_Reset", "GA_SpawnEffect" 等...
    ```cpp
    // Granted Gameplay Abilities
    USTRUCT(BlueprintType)
    struct FLyraAbilitySet_GameplayAbility {
      TSubclassOf<ULyraGameplayAbility> Ability = nullptr;  // 类引用
      int32 AbilityLevel = 1;  // 等级
      FGameplayTag InputTag;  // 
    }
    ```
  - **1个 Granted Gameplay Effect**: "GE_IsPlayer"
- `LyraAbilityTagRelationshipMapping` 资产
-  `LyraInputConfig` 资产: "InputData_Hero"
    - **5个 `Native InputAction`, 6个 `Ability InputAction`**  
- `LyraCameraMode`: "CM_ThirdPerson" 

3: ActionSets ( **三个共享的 `LyraExperienceActionsSet` 资产** ): // 基于GFA
  - "LAS_ShooterGame_SharedInput" 
    - *GFA_AddInputBinds*:
      - `LyraInputConfig` 资产: "InputData_ShooterGame_AddOns"
        - **6个 `Ability InputAction` 资产** 
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

4: Experience 专属 Actions:  // 基于GFA
  - *GFA_AddWidgets*:
    - **2个 `Widgets`**: "W_ControlPointStatusWidget", "W_CPScoreWidget"
  - *GFA_AddAbilities*:
    - `LyraAbilitySet` 资产: "AbilitySet_ControlPoint"
      - **2个 Granted Gameplay Abilities**: "GA_ShowLeaderboard_CP", "GA_AutoRespawn"
  - *GFA_AddComponents*:
    - **5个 `UGameStateComponent`** for `LyraGamState`: "B_ControlPointScoring", "B_TeamSetup_TwoTeams" 等...

    - **1个 `UControllerComponent`**: "B_PickRandomCharacter"

---

### Load Experience 流

:pencil2: **Start**

1: Server 在 `ALyraGameMode::InitGame()` 执行后的下一帧启动流程
- 获取 Experience ID (`FPrimaryAssetId`) 
- 设置当前 Experience: `ULyraExperienceManagerComponent::SetCurrentExperience(ExperienceId)`
  - 获取对应的 `ULyraExperienceDefinition` 配置数, Set Value and **Replicate** 
    ```cpp
    UPROPERTY(ReplicatedUsing=OnRep_CurrentExperience)
    TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;
    ```
  - Server 开始加载 Exp
    - Client 在 `OnRep` 后开始加载
 

2: 开始加载 Experience: `ULyraExperienceManagerComponent::StartExperienceLoad()`
- 切换状态 => *ELState::Loading* 
- **开始异步加载 Experience-associated Assets** 并注册回调 `OnExperienceLoadComplete()`

3: After Callback, 开始加载 Game Features (*异步计数*)
- 收集所有需要加载的 GF 名称, 获取其 URL 并记录总数: `NumGFPLoading = GameFeaturePluginURLs.Num()` 
- **if** `NumGFPLoading` > 0  
  - 切换状态 => *ELState::LoadingGameFeatures*
  - **通知 `UGameFeaturesSubsystem` 开始异步加载并激活所有 Game Feautres**  
    - `UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin()`
  - 一个 GF 在被 Activate 后, 会回调并减一 `NumGFPLoading`
  - When `NumGFPLoading == 0`, 代表 Exp 已被完全加载 => 执行总回调

4: 总回调 `OnExperienceFullLoadCompleted()`
- 切换状态 => *ELState::ExecutingActions*
- **Activate All GFAs** of the Experience and its ActionSets
  - 例如添加组件 `"B_ControlPointScoring"(GameStateComponent)`
- 切换状态 =>  *ELState::Loaded*

4.1:  按优先级顺序, **Broadcast 三种 `OnExperienceLoaded` delegates**, 以处理顺序依赖问题

  - High Priority: 框架底层

  - Normal Priority: 游戏流程
    - :pushpin: 进而触发 `UAsyncAction_ExperienceReady::OnReady` 的 Broadcast   
    - `"B_ControlPointScoring"` 在回调后, 会通知 `ULyraGamePhaseSubsystem` **开始游戏内流程(Start Phase)**

  - Low Priority: Bot 创建

<br>

### Unload Experience 流

在 `ULyraExperienceManagerComponent::EndPlay()` 时开始执行

- 卸载 **所有由当前Exp加载的 GFs**
  - `UGameFeaturesSubsystem::Get().DeactivateGameFeaturePlugin(GFPluginURL)`
- 卸载 由当前Exp加载的 GFAs 和 ActionSets