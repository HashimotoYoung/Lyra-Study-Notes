### FAQ:
*1. UDataAsset 和 UPrimaryDataAsset 的不同之处?*

- UPrimaryDataAsset 是 UDataAsset的[子集](#primary-asset-对比-secondary-asset)

---

## Asset Names and Packages
 
Every Asset in Unreal **lives in a Package**. Its identity is: `[PathToPackage]/[PackageName].[ObjectName]`
- **PathToPackage:** The folder structure in your Content Browser (e.g., `/Game/Characters/Player/`).

- **PackageName:** The name you give it in the Content Browser (e.g., `Hero_Mesh`). 换句话说, this is the name of the `.uasset` file 

- **Object Name(Asset Name):** The internal name of the UObject within the package (e.g., `Hero_Mesh`). 
    - :warning: Mostly, **Package name = main asset name**, but they are **independent identifiers** in Unreal.

Asset 特性:
- AssetName must be unique **within its Package**
- **FullPath 唯一性:** The complete path would be something like `/Game/Characters/Player/Hero_Mesh.Hero_Mesh` This full path is always **Globally Unique**.

在Unreal中, **"Package"** = the `.uasset` file itself. A single `.uasset` file in Unreal can be **either a single-object package or a multi-object package**. 

```cpp
//multi-object package的样子
Package: /Game/Weapons/Sword.uasset
├─ Main asset: Sword
├─ Sub-object: SwordMesh
└─ Sub-object: SwordMaterial
```


---

# AssetManager

The Asset management system in Unreal breaks all Assets into two types, **Primary Assets** and **Secondary Assets**. Primary Assets can be manipulated directly by the Asset Manager from their **Primary Asset ID**, which is obtained by calling the function `GetPrimaryAssetId`. 

The Asset Manager handles two different types of Assets: **Blueprint Classes**, and **non-Blueprint Assets** like Levels and Data Assets (Asset instances of the UDataAsset class)

:warning: Primary Assets 和 Secondary Assets 是**逻辑层**上的概念, 一个 Material 可以作为 Secondary Asset 也可以设为 Primary Asset

- AssetManager's 的三大主要职责:
    1. **Dynamic Loading/Unloading**: It provides the high-level functions for loading and unloading assets (主要负责 **Primary Assets**)  by their `PrimaryAssetId`.
    2. **Asset Discovery:** It scans for all assets in your project, especially Primary Assets, and keeps a **record** of them.
    3. **Asset Bundles:** It manages `Asset Bundles`, which are logical groupings of secondary assets that are loaded along with a primary asset (for example, a character's skeletal mesh, textures, and sounds might be bundled with the primary character asset).

<br>

## Primary Asset 对比 Secondary Asset

#### What is Primary Asset ?

A Primary Data Asset is a **Data Asset** that implements a `GetPrimaryAssetId` function and has asset bundle support, which allows it to be manually loaded/unloaded from the Asset Manager.
- :star: **Unreal Engine does not have Primary Assets by default**, Primary Asset Type 是一种**自定义的逻辑层Type**, 比如"Weapon","NPC" 
- 一般可在C++中继承 `UPrimaryDataAsset` 来创建一个 Primary Asset Type, 比如 `UWeaponData: UPrimaryDataAsset`
- **PrimaryAssetId = PrimaryAssetType + PrimaryAssetName:** 当创建了一个基于 `class UWeaponData` 的asset后, 比如 `DA_AssaultRifle.uasset`, 其 PrimaryAssetId 为 "WeaponData:DA_AssaultRifle"
- **为保证 PrimiaryAssetId 的全局唯一性**, 同一个Type下的 Primary Asset  Name 不能相同
- Primary Asset are the root objects that are directly known to the `Asset Manager` by its `PrimaryAssetId`.

<br>

#### Scan Rules很重要
only assets that **match the criteria in the Asset Manager's scan rules** are treated as primary assets.
- 意味着 primary assets 需要放在指定文件夹下

<br>

#### Secondary Asset
- A "Secondary Asset" is simply anything that gets loaded as a reference of a primary asset (**not tracked directly by the asset manager**)
- For example, textures, static meshes, and materials are almost always secondary assets.

---
## Asset Bundle
An Asset Bundle is basically a **named** group of **secondary assets** , usually used so the engine can load them **selectively** instead of always loading everything.

- identified by `FName`
- Asset Bundles live inside **the scope of a single Primary Asset**.
- Almost always work with **soft references**.

---

## FStreamableManager

As a **part** of `UAssetManager`, the `FStreamableManager` is a **native C++ struct** that performs the actual, low-level work of **asynchronously loading** assets and keeping them in memory. 
- can be accessed through the static function `UAssetManager::GetStreamableManager()`.
- 主要负责 load **Secondary Assets** asynchronously


### FStreamableHandle

FStreamableHandle is a **reference-counted handle** to an **async asset loading request**, allowing you to track, cancel, and respond to asset loads dynamically.

---


## "Asset Pointers" 介绍

FSoftClassPath, FSoftObjectPath

FSoftObjectPath is designed to reference a **UObject asset**, not a runtime- **UObject instance**

UObject Asset 对比 UObject Instance