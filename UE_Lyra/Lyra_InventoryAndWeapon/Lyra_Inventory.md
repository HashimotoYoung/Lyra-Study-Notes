
# Lyra Inventory 
> :books: Lyra 采用**组件思想**设计游戏内物品, 定义一个 "物品"(`ULyraInventoryItemDefinition`) 是由 **1-n 个 "碎片"(`ULyraInventoryItemFragment`) 组合而成的**, 以实现灵活按需配置 
- 每一种 Fragment 赋予了物品不同的特性和功能
- Fragment works on both 配置层 and 运行时

---

#### InventoryItem / Equipment / Weapon 之间的关系

- Weapon 是一种 Equipment, 属于 **运行时** 概念
  - `ULyraWeaponInstance` 继承自 `ULyraEquipmentInstance`
  - :x: There is no `ULyraWeaponDefinition` 
- Equipment **基于 InventoryItem 而存在**
  - 存在 "装备型" Fragment, 包含了装备定义
	```cpp
	class UInventoryFragment_EquippableItem : public ULyraInventoryItemFragment 
	{
		UPROPERTY(EditAnywhere, Category=Lyra)
		TSubclassOf<ULyraEquipmentDefinition> EquipmentDefinition;
	};
	```
###### :pushpin: 当尝试 Equip 时:
1. 需要先获取 物品实例 Item(`ULyraInventoryItemInstance*`)
2. **if** 此 Item 持有 `UInventoryFragment_EquippableItem` 类型碎片 => 获取其 `EquipmentDefinition`
3. `ULyraEquipmentManagerComponent` 负责**创建 装备实例** EquippedItem(`ULyraEquipmentInstance*`) 并添加到 `EquipmentList (FastArray)` 中进行 Replicate
4. 物品实例 会作为 **Instigator** 和 装备实例 进行绑定
	  - `EquippedItem->SetInstigator(Item)`


---

### ULyraInventoryItemDefinition : UObject

Lyra 中物品的 **配置层** 定义
- Abstract, 需要通过 Blueprint 子类具体化

```cpp
UCLASS(Blueprintable, Const, Abstract)
class ULyraInventoryItemDefinition : public UObject
{
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Display)
	FText DisplayName; // 名称

	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Display, Instanced)
	TArray<TObjectPtr<ULyraInventoryItemFragment>> Fragments; // 碎片

	const ULyraInventoryItemFragment* FindFragmentByClass(TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const;
};
```
---

### ULyraInventoryItemInstance : UObject

Lyra 中物品的 **运行时** 定义
- Contained by `ULyraInventoryManagerComponent : UActorComponent`
- 支持堆叠, 并引用对应配置
  ```cpp
  // in ULyraInventoryItemInstance.h
  UPROPERTY(Replicated)
  FGameplayTagStackContainer StatTags

  UPROPERTY(Replicated)
  TSubclassOf<ULyraInventoryItemDefinition> ItemDef
  ```

---

### ULyraInventoryManagerComponent : UActorComponent

"背包" 组件, 定义了一系列方法来管理 InventoryItems (添加/删除...)

- Added dynamically to `Controller` by GameFeature "ShootCore"
- Contains an array of `ULyraInventoryItemInstance`
  ```cpp
  // in ULyraInventoryManagerComponent.h
  UPROPERTY(Replicated)
  FLyraInventoryList InventoryList
  ```