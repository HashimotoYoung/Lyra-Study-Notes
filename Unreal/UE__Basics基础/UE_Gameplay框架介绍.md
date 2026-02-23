# Unreal Gameplay 框架介绍
### 类关系图
Game Instance:

PlayerController 关系:
- Spawn并且Owns `PlayerState`
  // **optional** 对于 AIController 
- Spawn并且Owns `PlayerCameraManager`
- Spawn并且Owns `AHUD`
- Possess `Pawn`
- Refers 出生点Actor


# 1. Game
## GameInstance 

- **Lifecycle:** 随Engine
- **独立性:** Server 和 Client 有着各自**独立的`GameInstance`**, 扮演着不同的角色
- **Not Replicated:** There is no built-in synchronization between server `GameInstance` and client `GameInstance`
-  Acts as a manager for any number of game instance subsystems
<br>

## GameMode
- **Lifecycle:** 随`UWorld`
- The `GameMode` instance exists **only on the server** in a  mutiplayer game
- Only one `GameMode` may be in use **at any given time**
- GameMode is the **first actor** to instantiate upon level load. Upon creation, it instantiates the remaining framework actors, **the first two** are `GameState` and `PlayerState`.
<br>

## GameState

The `GameState` should manage **information that is meant to be known to all connected clients and is specific to the Game Mode** but is not specific to any individual player. 

Think of the `GameState` as the "**Replicated Snapshot of the Current Game's State."**

- **Lifecycle:** 随`Uworld`
- **Replicated to All Clients:** A single `GameState` instance exists on the server, and copies of it are replicated to **every connected client**.
- 使用时需要考虑哪些Property应该放在GameState,哪些放在PlayerState

:star:**对比`GameMode`和`GameState`:** 

`GameMode` **decides**, `GameState` **tells** --- The `GameMode` enacts the rules on the server, and the GameState broadcasts the results or current status of those rules to all players.

---

# 2. Player 
Unreal中设计了多个class to represent the *concept of Player*
One **“player”** involves **input**, **network connection**, **control logic**, and a **physical representation**

### ULocalPlayer : UPlayer

Represents a player using this local machine’s input device 

- `ULocalPlayer` 只存在于Client端, 其生命周期随 GameInstance

<br>

### UNetConnection : UPlayer

---

### PlayerController

The player controller class is a **manager class** that 
- 处理输入信息; 处理HUD; **Possess and Control** player's physical representations in the game

---

### Pawn 

主要特征:

- :star: `APawn` is the ***most fundamental actor*** that can be possessed by a controller
- 默认无移动组件, 碰撞信息
<br>

### Character

---

