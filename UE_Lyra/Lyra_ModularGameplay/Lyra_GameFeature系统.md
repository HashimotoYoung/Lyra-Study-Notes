
# Lyra Game Features

> :books: Lyra ModularGameplay 三大系统之一, 和 GFCM 强耦合

Game Features 插件本身类似于一个 **Plugin 生成器**, 专门用于生成与游戏玩法紧密相关的 `GF_Plugin`, which are:

- **单向依赖:** 一般来说插件是不依赖项目的, but `GF_Plugins` are desgined to be *project-based*, should rely on the core gameplay logic and **could rely on other plugins**
  - 另一方面, 游戏核心逻辑不应依赖具体的 `GF_Plugin`
- **指定目录**: `GF_Plugins` are required to be placed under `{ProjectName}/Plugins/GameFeatures/`
- **动态激活:** `GF_Plugins` can be enabled or disabled **at runtime**.

<br>

#### 一个 GF Plugin 可包含的内容有:
1. Content Assets: Includes `Blueprints`, `Materials`, `Skeletal Meshes`, `Animations`, `Textures`, etc., all stored **within the plugin’s `Content/` folder.**
2. C++ Code
3. Dependencies on other Plugins are recorded in `.uplugin` file
4. :pushpin: ***Game Feature Data Asset* :** Every GF plugin has one central `UGameFeatureData` asset. This asset, which lives in the root of the plugin's content folder, is the **核心配置文件 for the feature**. It's where you define ***the list of Game Feature Actions*** to run when the feature is activated.
 
---

### Game Feature Action (GFA)
> :books: GFA 可看作是 Game Feature 的 "逻辑执行单元"
- 一份 Game Feature Data Asset 可包含多个 GFAs
- 每个 GFA 定义了 What happens when **the feature** is ACTIVATED, 例如:

	- **Add components** to the player pawn.
	- **Add gameplay abilities** to the player's ASC
	- **Inject new HUD widgets** into the UI
	- **Add new input mappings**
- :pushpin: GFA 自身的运行流程是相对独立的, 例如 Lyra 中的一份 `ULyraExperienceDefinition` 可包含多个 GFAs, 并自主驱动其 Activation
  - :warning: 使用了 UPROPERTY(Instanced)
  ```cpp
  class ULyraExperienceDefinition : public UPrimaryDataAsset
  {
	...
	UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
	TArray<TObjectPtr<UGameFeatureAction>> Actions;
  }
  ```

---

### 通过 Experience 驱动 GFA

- Game Feature Subsystem 是一个Engine级系统, 意味着 Game Features 本身是 ***GameInstance-scoped***
- :star: Lyra 通过利用 Experience 机制, 将**部分 Game Features** 限定为 ***World-scoped***

  - 每份 `ULyraExperienceDefinition` 引用了 a list of GFAs
  - 在 "ExperienceFullLoadCompleted" 时 Activate 
  - 在 "World->EndPlay" 时 Deactivate
  - **核心函数:** `FGameFeatureStateChangeContext::SetRequiredWorldContextHandle(FName Handle)`
  
<br>

### 使用 `IGameFeatureStateChangeObserver` 添加自定义逻辑 
> 例如添加自定义 GameplayCue 路径 

- 使用自定义 Game Feature Policy: `ULyraGameFeaturePolicy`
  - 继承自 `UDefaultGameFeaturesProjectPolicies`
- 定义功能性子类
  - `ULyraGameFeature_AddGameplayCuePaths :  UObject,  IGameFeatureStateChangeObserver`
  - 实现关键方法 `OnGameFeatureRegistering(...)` 
- 初始化时进行注册
  ```cpp
  void ULyraGameFeaturePolicy::InitGameFeatureManager() {
	Observers.Add(NewObject<ULyraGameFeature_HotfixManager>());
	Observers.Add(NewObject<ULyraGameFeature_AddGameplayCuePaths>());

	UGameFeaturesSubsystem& Subsystem = UGameFeaturesSubsystem::Get();
	for (UObject* Observer : Observers) {
		Subsystem.AddObserver(Observer);
	}

	Super::InitGameFeatureManager();
  }
  ```
- CallStack
	```cpp
	UEngine::Init()
	=> UGameFeaturesSubsystem::LoadBuiltInGameFeaturePlugins()
	=> UGameFeaturesSubsystem::OnGameFeatureRegistering()
	=> ULyraGameFeature_AddGameplayCuePaths::OnGameFeatureRegistering()
	=> UGameplayCueManager::InitializeRuntimeObjectLibrary()
	```

---

### `UGameFeatureAction_AddComponents` 示例


- `GameFeatureAction_AddComponents` 为内置 GFA, 其它类型的 GFA 中绝大多数是由 Lyra 自定义的

```cpp
void UGameFeatureAction_AddComponents::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
	FContextHandles& Handles = ContextHandles.FindOrAdd(Context);

	Handles.GameInstanceStartHandle = FWorldDelegates::OnStartGameInstance.AddUObject(this, 
		&UGameFeatureAction_AddComponents::HandleGameInstanceStart, FGameFeatureStateChangeContext(Context));

	ensure(Handles.ComponentRequestHandles.Num() == 0);

	// 加入到现存的 Worlds 中
	for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
	{
		if (Context.ShouldApplyToWorldContext(WorldContext))
		{
			this.AddToWorld(WorldContext, Handles);
		}
	}
}

void UGameFeatureAction_AddComponents::AddToWorld(const FWorldContext& WorldContext, FContextHandles& Handles) {
	...
	//获取GFCM
	UGameFrameworkComponentManager* GFCM = UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GameInstance)) 
	
	//判断server或client添加
	const ENetMode NetMode = World->GetNetMode();
	const bool bIsServer = NetMode != NM_Client;
	const bool bIsClient = NetMode != NM_DedicatedServer;

	for (const FGameFeatureComponentEntry& Entry : ComponentList) {
		const bool bShouldAddRequest = (bIsServer && Entry.bServerComponent) || (bIsClient && Entry.bClientComponent);
		if (bShouldAddRequest) {
			if (!Entry.ActorClass.IsNull()) {
				//加载Soft引用
				TSubclassOf<UActorComponent> ComponentClass = Entry.ComponentClass.LoadSynchronous();
				if (ComponentClass) {
					//Cache 请求
					Handles.ComponentRequestHandles.Add(GFCM->AddComponentRequest(Entry.ActorClass, ComponentClass));
				}
				else if (!Entry.ComponentClass.IsNull()) {
					UE_LOG(LogGameFeatures, Error, TEXT("[GameFeatureData %s]: Failed to load component class %s. Not applying component."), *GetPathNameSafe(this), *Entry.ComponentClass.ToString());
				}
			}
		}
	}// end for
}
```

