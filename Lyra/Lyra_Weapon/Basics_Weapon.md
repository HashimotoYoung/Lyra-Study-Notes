
# Lyra Weapon

## 1. 武器逻辑层
### `ULyraEquipmentInstance: UObject`

装备实例基类

##### 主要属性:
```cpp
UPROPERTY(ReplicatedUsing=OnRep_Instigator)
TObjectPtr<UObject> Instigator;

UPROPERTY(Replicated)
TArray<TObjectPtr<AActor>> SpawnedActors;
```

##### 主要方法:
---

### `ULyraWeaponInstance: ULyraEquipmentInstance`
*逻辑层* 武器实例
- 继承示例: `B_WeaponInstance_Pistol `<= `B_WeaponInstance_Base` <= `ULyraRangedWeaponInstance` <= `ULyraWeaponInstance`

##### 类关系:
- Contained by `ULyraEquipmentManagerComponent`

##### 主要属性:

```cpp
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Animation)
FLyraAnimLayerSelectionSet EquippedAnimSet;

UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Animation)
FLyraAnimLayerSelectionSet UneuippedAnimSet;
```
##### 主要方法:
#### `TSubclassOf<UAnimInstance> PickBestAnimLayer(bool bEquipped, const FGameplayTagContainer& CosmeticTags) const`


---
### `ULyraEquipmentManagerComponent : UPawnComponent`

LyraCharacter的八大组件之一: 武器管理组件

##### 主要属性:
`FLyraEquipmentList EquipmentList` *Replicated*
//:star: **Links 玩家 and 武器(逻辑层)**


##### 主要方法:
#### `ULyraEquipmentInstance* EquipItem(TSubclassOf<ULyraEquipmentDefinition> EquipmentDefinition);`


<br>

### :star: `struct FLyraEquipmentList : public FFastArraySerializer`
网络端重点

---

### `ULyraWeaponStateComponent :  UControllerComponent`
负责武器状态跟踪; 客户端预测记录;
选择继承 UControllerComponent 的原因:
- Replicate to Owner only

##### 类关系:


##### 主要属性:


##### 主要方法:


---
## 2. 武器配置层

### `ULyraEquipmentDefinition`

负责定义一件装备, 包含:

- 装备实例 SubclassType: `TSubclassOf<ULyraEquipmentInstance>`
- 装备相关的Ability配置
- Array<`FLyraEquipmentActorToSpawn`>: 武器的实际Actor, 通常为一件
  - 武器Actor的 ClassType
  - 绑定的骨骼**Socket**: `FName`
  - 位置Offset

```cpp
USTRUCT()
struct FLyraEquipmentActorToSpawn
{
	UPROPERTY(EditAnywhere, Category = Equipment)
	TSubclassOf<AActor> ActorToSpawn;

	UPROPERTY(EditAnywhere, Category = Equipment)
	FName AttachSocket;

	UPROPERTY(EditAnywhere, Category = Equipment)
	FTransform AttachTransform;
};


UCLASS(Blueprintable, Const, Abstract, BlueprintType)
class ULyraEquipmentDefinition : public UObject
{
	// Class to spawn
	UPROPERTY(EditDefaultsOnly, Category = Equipment)
	TSubclassOf<ULyraEquipmentInstance> InstanceType;

	// Gameplay ability sets to grant when this is equipped
	UPROPERTY(EditDefaultsOnly, Category = Equipment)
	TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;

	// Actors to spawn on the pawn when this is equipped
	UPROPERTY(EditDefaultsOnly, Category = Equipment)
	TArray<FLyraEquipmentActorToSpawn> ActorsToSpawn;
};
```
---
### `ULyraInventoryItemDefinition`
Lyra 将一个 "Inventory Item" 的定义为**由若干个"碎片"`(ULyraInventoryItemFragment)`组成**, 以实现灵活按需配置, 每个 Fragment 依据类型处理不同功能

- :star: **InventoryItem 的概念层级高于 Equipment**
  - `UInventoryFragment_EquippableItem` 引用 `ULyraEquipmentDefinition`


```cpp
class ULyraInventoryItemDefinition : public UObject
{
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Display)
	FText DisplayName;

	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Display, Instanced)
	TArray<TObjectPtr<ULyraInventoryItemFragment>> Fragments;

public:
	const ULyraInventoryItemFragment* FindFragmentByClass(TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const;
};


class UInventoryFragment_EquippableItem : public ULyraInventoryItemFragment
{
	GENERATED_BODY()

public:
	UPROPERTY(EditAnywhere, Category=Lyra)
	TSubclassOf<ULyraEquipmentDefinition> EquipmentDefinition;
};
```
---
### Lyra武器配置示例

`B_Hero_ShooterMannequin` **has a** `LyraInventoryItemDefinition`("ID_Pistol")
"ID_Pistol" has 五个 Fragments:
  1. `InventoryFragment_EquippableItem`, 其**引用**了 BP `LyraEquipmentDefinition`("WID_Pistol")
   该装备定义包含:
	  - 手枪Type Reference: `B_WeaponInstance_Pistol`
	  - 手枪相关能力DA: "AbilitySet_ShooterPistol"
	  - 手枪Actor: `B_Pistol`: 手枪Actor
		等数据...  
  2. todo...

---

## 3. 武器表现层

### 蓝图 B_Weapon : `AActor`
  
B_Pistol 等武器Actor的父类
***表现层*** 武器实例, 持有 `SkeletalMeshComponent`, 展示武器外观

##### 类关系:
- :x: 和逻辑层武器实例 `ULyraWeaponInstance` 没有交互
##### 主要属性:


`BWeaponFire(Actor) weaponFire`: 开火效果
`BWeaponImpact(Actor) weaponImpact`: 击中效果
`BWeaponDecal(Actor) weaponDecal`: 弹坑效果
- 以上三个属性皆为蓝图引用, 主要用于协助处理Event Fire的表现层效果
- **用时创建**, 并被 AttachedTo B_Weapon's `SkeletalMeshComponent`

##### 主要方法:

### BP_Event Fire
