
## Tasks

2025/10/1:
- [x] 为何蹲下时摄像机会跟着动
- [x] 随镜头转身
- [x] HUD吞输入
- [x] 跳跃状态机
- [x] ULyraPawnExtensionComponent
- [x] 出生特效(GAS)

2025/11/1:
- [x] DistanceMatching, 同步组
- [x] 武器开火伤害
- [x] AI接入

2025/12/20:
- [x] 补充UIPolicy, 跑一次完整的GameMode
- [x] experience流程, GFCM初步分析
- [x] GF加载分析
- [x] 人物瞄准动画
- [x] 处理 _readme
- [x] 武器GC部分分析
- [x] 武器属性伤害分析
- [ ] 补完GAS中的GA,ASC部分后提交
- [ ] 完善GAS网络部分
- [ ] dash, 手雷
- [ ] LyraHealthComponent
- [ ] 完善input modifires/**triggers**

---

### 未完成 

IMC_ADS_SPEED 两个修改器
IA_Look 的 modifiers
LAS_ShooterGame_StandardComponents 需要补全comps
FullBodyAdditive
修复前进时下蹲的动画

---

## Questions :question:

#### 其它:
{
what is struct FEchoVerbMessage ?
UGameplayMessageProcessor

gameplaymessage system
verbmessage/verbmessagereplication/verbmessagereplicationentry
}


#### Anim: 
what is AN_PlayWeaponMontage(animnoitfy) ? 
答:配合人物动画播放
#### Input:

#### Camera:

#### GAS:

FActiveGameplayEffect 构造函数学习



#### Network:

ForceNetUpdate 详细解析 什么情况下需要调用 vs	bIsNetDirty = true;

Authoritative gameplay state (MUST replicate) VS Derived / transient data (MUST NOT replicate)

ustruct can be replicated auto?

OwnerActor->FlushNetDormancy();


LyraGameMode::TryDedicatedServerLogin ??

Lyra Replication Graph ??

lyra game session??

FGameplayTagStackContainer

ActorComponent::AddReplicatedSubObject(), learn registraion list

diff between Network Prediction / Interpolation?

```cpp
//分析
if (InPredictionKey.IsLocalClientKey() == false || IsNetAuthority())	// Clients predicting a GameplayEffect must not call MarkItemDirty
{
	MarkItemDirty(*AppliedActiveGE);

	ABILITY_LOG(Verbose, TEXT("Added GE: %s. ReplicationID: %d. Key: %d. PredictionLey: %d"), *AppliedActiveGE->Spec.Def->GetName(), AppliedActiveGE->ReplicationID, AppliedActiveGE->ReplicationKey, InPredictionKey.Current);
}
else
{
	// Clients predicting should call MarkArrayDirty to force the internal replication map to be rebuilt.
	MarkArrayDirty();

	// Once replicated state has caught up to this prediction key, we must remove this gameplay effect.
	InPredictionKey.NewRejectOrCaughtUpDelegate(FPredictionKeyEvent::CreateUObject(Owner, &UAbilitySystemComponent::RemoveActiveGameplayEffect_NoReturn, AppliedActiveGE->Handle, -1));
	
}
```

	//~FFastArraySerializer contract
	void PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize);
	void PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize);
	void PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize);
	//~End of FFastArraySerializer contract

	bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
	{
		return FFastArraySerializer::FastArrayDeltaSerialize<FLyraAppliedEquipmentEntry, FLyraEquipmentList>(Entries, DeltaParms, *this);
	}


#### Other:
蓝图宏和函数区别
async action outpin in listen gameplay for message

FTimerManager / FTimerHandle / FDelegateHandle / FObjectKey / FArchive

```

了解 TSoftObjectPtr和  TSoftClassPtr 作用

当加载 GCN Class时, 是否包含硬引用

加载蓝图是什么概念?

if (CueData.LoadedGameplayCueClass == nullptr)
{
    // See if the object is loaded but just not hooked up here
    CueData.LoadedGameplayCueClass = Cast<UClass>(CueData.GameplayCueNotifyObj.ResolveObject());
    if (CueData.LoadedGameplayCueClass == nullptr)
    {
        if (!CueManager->HandleMissingGameplayCue(this, CueData, TargetActor, EventType, Parameters))
        {
            return false;
        }
    }
}
```

---
a. Modular Gameplay (模块化游戏设计)
b. GAS
C. Inventory and equipment(背包和装备)
D. Gamplay messaging(游戏消息)
E. Ranged weapons(远程武器)
F. Aim assist(瞄准辅助)
G. Hit impacts and number pops(命中冲击和数字弹出)
H. Team management(团队管理)
I. Game phases(游戏阶段)
. Camera
. Bot / AI Integration
. Enhanced Input:

---

UGameFeatureAction_AddAbilities 回顾 深度解析