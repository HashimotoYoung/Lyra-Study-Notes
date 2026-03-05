
# :books: Lyra 学习笔记 

## 📖 简介

本 Repo 用于记录对 UE 示例项目 Lyra Starter Game 的学习研究

- 主要侧重于 Gameplay 方向
- 基于引擎版本 **Unreal Engine 5.2+**
- 🙏 如发现错误或理解偏差, 欢迎指正和交流

> :pushpin: 寻求 UE Gameplay 方向的工作机会, 有意者可留个联系方式

## 🗂️ 文档索引

### Modular Gameplay (模块化玩法)

- [Lyra Experience 系统](./UE_Lyra/Lyra_ModularGameplay/Lyra_Experience系统.md#lyra-experience)  
  - 游戏运行时主入口, 负责配置并驱动实际 Gameplay

- [Game Feature 简介](./UE_Lyra/Lyra_ModularGameplay/Lyra_GameFeature系统.md#lyra-game-features)  
  - 提供功能级热插拔与逻辑注入机制, 保持核心代码库整洁

- [GameFrameworkComponentManager 分析](./UE_Lyra/Lyra_ModularGameplay/Lyra_GFCM_Extensions系统.md#lyra-gameframeworkcomponentmanager)
  - 协助 Game Feature 生效, 并协调组件初始化



### GAS
- [AbilitySystemComponent](./Unreal/UE_GAS分析/GAS_AbilitySystemComp.md#gas-asc): GAS 核心运行组件
- [Gameplay Abilities](./Unreal/UE_GAS分析/GAS_GA/GA_Basics.md#gameplay-ability-总览): "技能" 行为的封装
- [Gameplay Effect](./Unreal/UE_GAS分析/GAS_GE/GE_Basics.md#gas-ge-总览): 效果与Buff 
- [AttributeSet](./Unreal/UE_GAS分析/GAS_Attribute.md#gas-attribute): 属性定义, 如血量、体力
- [Gameplay Cue](./Unreal/UE_GAS分析/GAS_GCue.md#gas-gameplaycue-分析): 表现层视觉特效/音效
- [GAS Prediction](./Unreal/UE_GAS分析/GAS_Prediction.md#gas-预测机制): 客户端预测与服务器校正

### Animation 
- [Locomotion 状态机](./UE_Lyra/Lyra_Animation/ABP_Mannequin_Base分析/FSM_Locomotion分析.md#fsm-locomotion-分析)
  - 基于 Anim Layer 构建的角色移动状态机

### Gameplay 

- [Camera 相机系统](./UE_Lyra/Lyra_General/Lyra_Camera.md#lyra-camera-相机)
  - 基于 CameraMode 栈的相机管理
- [游戏内 Phases 机制](./UE_Lyra/Lyra_General/Game_Phase设计.md#lyra-game-phase)
  - 局内流程管理 (如开始, 战斗中, 结算)
- [Input 输入相关](./UE_Lyra/Lyra_General/Lyra_Input输入.md#lyra-input)
  - 基于 Enhanced Input 的数据驱动输入映射体系
- [GameplayMessage 系统分析](./UE_Lyra/Lyra_General/Lyra_GameplayMessage消息系统.md#lyra-gameplay-message)
  - 轻量级消息总线, 用于模块间解耦通信

### Inventory and Weapon 
- [物品/背包系统](./UE_Lyra/Lyra_InventoryAndWeapon/Lyra_Inventory.md#lyra-inventory)
  - 基于 "碎片" 的组件化物品定义体系
- [武器系统](./UE_Lyra/Lyra_InventoryAndWeapon/Lyra_Weapon.md#lyra-weapon-and-equipment)
  - 武器装备, 切换与射击逻辑




