
# Lyra Game Features

Lyra ModularGameplay的三大系统之一

Game Features 可看作是一个**Plugin 生成器**, 主要用于生成与游戏玩法紧密相关的 `GF_Plugin`, which are:

1. **依赖于项目:** 一般来说插件是不依赖项目的, 但是 `GF_Plugins` are desgined to be *project-based*, should rely on the core gameplay logic and **could rely on other plugins**
2. **隐身于项目:** core game 不应该知道 Game Features 的存在
3. **指定目录**: `GF_Plugins` are required to be placed under `{ProjectName}/Plugins/GameFeatures/`
4. **动态激活:** `GF_Plugins` can be enabled or disabled at **runtime**.
<br>

#### 一个 Game Feature Plugin 可包含的内容有:
1. Content Assets: Includes `Blueprints`, `Materials`, `Skeletal Meshes`, `Animations`, `Textures`, etc., all stored **within the plugin’s `Content/` folder.**
2. C++ Code
3. Dependencies on other Plugins are recorded in `.uplugin` file
4. :star: **Game Feature Data Asset:** Every Game Feature plugin has one central `GameFeatureData` asset. This asset, which lives in the root of the plugin's content folder, is the **main configuration file for the feature**. It's where you define the list of ***Game Feature Actions*** to run when the feature is activated.
 
---

### Game Feature Actions (GFA)

GFAs **define what happens when the feature is ACTIVATED**, 包括但不限于:

- *Add components* to the player pawn.
- *Add gameplay abilities* to the player's ability system component.
- *Inject new HUD widgets* into the UI.
- *Add new input mappings*.

---

## GameFeature 相关流程: 

### 使用 `IGameFeatureStateChangeObserver` 添加自定义Logic  
Lyra 通过使用 `UDefaultGameFeaturesProjectPolicies + IGameFeatureStateChangeObserver` 来实现GF相关逻辑, 例如添加自定义GameplayCue路径:

```cpp
// callstack:
UEngine::Init()
UGameFeaturesSubsystem::LoadBuiltInGameFeaturePlugins()
UGameFeaturesSubsystem::OnGameFeatureRegistering()
ULyraGameFeature_AddGameplayCuePaths::OnGameFeatureRegistering()
UGameplayCueManager::InitializeRuntimeObjectLibrary()
```
---

### 通过 Experience 系统 Activate/Deactivate GFs
> 以 UGameFeatureAction_AddComponents 为例

:books: Game Feature 本身是一个Engine级的系统, 意味着 Game Feature Actions 本质上也是在 **Engine层** 进行 Activate/Deactivate. **但在Lyra中**, Lyra Experience系统会在World EndPlay 阶段进行 Unload (Deactivate 相关GF和GFA), 因此将 GameFeature 限定为了一个 **World层** 的功能, 确保各个 World 的 AddComponents 能相互独立
- GameFeatureAction_AddComponents 为GF组件自带的Action, 其它大部分 GFA 由 Lyra 自定义




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
			AddToWorld(WorldContext, Handles);
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

