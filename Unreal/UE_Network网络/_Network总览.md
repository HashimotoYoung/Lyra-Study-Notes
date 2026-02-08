
[UE5 -- Replication（网络复制）](https://zhuanlan.zhihu.com/p/578480318) read_0

[Compendium](https://cedric-neukirchen.net/docs/category/multiplayer-network-compendium) 一点一点品 read_0

[UE5 网络深入探索](https://zhuanlan.zhihu.com/p/27444865716) 底层介绍1 read_0

[UE4 Notes 网络同步相关](https://zhuanlan.zhihu.com/p/647463689) 底层介绍2 read_0

[Wizardcell Tips and Tricks](https://wizardcell.com/unreal/multiplayer-tips-and-tricks/) read_0

[UE4-多人游戏框架理解](https://stonelzp.github.io/ue4-multiplay-framework/)

todo:Replication Graph, OnRepNotify

---

# UE Network 相关概念简介

## 1. Replication
- 见 Net_Replication.md
---

## 2. Remote Procedure Call

> Use RPC for "One-off" events, Use Replication for persistent state

- **Can Only Defined** in `AActor / UActorComponent` subclass 
  - :x: 其它 `UObject` 不行
- **单向消息传递, 无返回值:** If you need to send information back, you must use another RPC in the opposite direction or use replicated properties.
- RPC parameters are serialized immediately, not lazily.


### Server RPC
Client => Server, **`ServerRPC` 是唯一能够让服务器从相应客户端获得数据的方式!?**

###### 调用条件:

1. **Replicated:** The Actor or Actor Component on which the RPC is called must be replicated 
2. **Ownership:** the `AActor/UActorComponent` calling from must be  owned by the client (`PlayerController`)

<br>

### Client RPC 

Server => the single **Owning** Client, 使用 Client RPC 的典型示例有:

- 显示UI信息到特定玩家: "你是狼人"

- 通知特定玩家播放音效/特效: the "hit" sound for the player who successfully landed a shot.

- :star: **纠正特定玩家的客户端预测:** The server might send a correction to a client's movement if their prediction diverged too much from the authoritative server state.

###### 调用条件:

1. **Replicated:** 同 Server RPC
2. **Ownership:** the `AActor/UActorComponent` calling from must be **owned** by the target client
<br>

### Multicast RPC

Server => Multiple Clients , 有以下特点:
- :x: Dont care Ownership
- Only **relevant clients** receive it,  not all clients 
- :pushpin: Executed **On Server itself** either


###### 调用条件:

1. **Replicated:** 同 ServerRPC
2. **Relevancy:** Only clients who have a replicated version of the Actor *currently* "alive" on their machine receive the call.
   - :warning: 不会重发给 later-become-relevant clients 
<br>

#### Per-Channel 顺序可保证
:memo: *Reliable* RPCs are guaranteed to be executed in the order they were sent — **per channel** 
- Different Actors == Different Network Channels

- :heavy_check_mark: Gurranteed Order: 对于**同一个Actor**的 *Reliable* RPC 
  - :warning:  RPCs called **in same frame** will not necessarily execute in same frame on the remote end.




---
## 3. Authority and Net Role

Authority 底层基于 `enum ENetRole` 实现 
Net Role 描述的是一个 Actor 相对于 Local Machine 的概念
- `ENetRole` is an **Actor's property**
- LocalRole reflects the **local machines's 控制权限** over the Actor object
  ```cpp
  bool AActor::HasAuthority() const {
    return (GetLocalRole() == ENetRole::ROLE_Authority);
  }
  ```
##### :warning: 需要注意的 Local Roles
- `PlayerController` is Authority on Both Server and the Owning Client
- `PlayerState` is always SimulatedProxy on ANY Client
- Some “Owned” actors are SimulatedProxy even on owning client
  - 例如武器
  
<br>

#### Net Mode

> :books: Net Mode describes how the **Local Machine (an Game Instance)** participates in a multiplayer session (whether it acts as a server, a client or both) 

- **基于 World 和 NetDriver:** `AActor::GetNetMode()` 的内部实现中会优先返回 `NetDriver->GetNetMode()`, 然后是 `World->GetNetMode()`
- **动态属性:** NetMode 可能随着 UWorld 实例而改变, 因为不同世界可能有不同的 NetMode, 例如当处于主菜单时, netmode == `NM_Standalone`, 当进入游戏后, netmode == `NM_Client`

---

## 4. Relevancy

**Relevancy** is fundamentally  a **server-side** concept, clients dont care much 

Relevancy 是一个 Actor 层的概念?

When an Actor's property changed, the Server only sends replication updates to clients whom **the actor is relevant to**.

- **Per Client:** The set of relevant actors is determined on a per-client, 也可以说是 **per-connection basis**
- Affects not only property replication but also [RPC](#warning-成功调用-client-rpc-的必要条件)

When an Actor becomes Irrelevant to a Client ?? 重点
Client-Side Destruction:??
Loss of Reference??

What if the Actor Becomes Relevant?

Server Re-replicates the Actor: 
Client-Side Re-spawn/Re-instantiation: 
RPCs can then succeed: 

### What "the Actor is relevent to the Client" means ?
By using the call `AActor::IsNetRelevantFor`, **The server determined** that the client needs to receive network updates for that specific Actor.

### Relevent to the Client 的判断逻辑
前提知识点:
- :star:Each client connection can have **zero 或 one** `Pawn` at any time
    - A **client connection** is tied to one `PlayerController`.
    - A `PlayerController` can possess **at most one** `Pawn` at a time.
    
```cpp
    //自身相关判断
+-> Actor::IsAlwaysRelevant? ------------------> YES → Replicate
|
+-> Actor::IsOwnedByClient? ------------------> YES → Replicate
|
+-> Actor::IsClient'sPawn (controlled by playerController)? -------------------> YES → Replicate
|
+-> Actor::IsClientPawnInstigator? ----------> YES → Replicate
|   //依赖于 owner
+-> UseOwnerRelevancy? (and owner is relevant?) --> YES → Replicate
|   //依赖于 AnotherActor
+-> AttachedToAnotherActor? ----------> Check base actor's relevancy
|
+-> Distance-basedRelevancyEnabled? --+--> IsWithinDistance? → YES → Replicate
|   //NotRelevant的判断
+-> IsHidden?  ------------------> YES → NOT Relevant
|
+-> Actor has no rootComponent or the root component does not have collision enabled?  ------------------> YES → NOT Relevant
|
+-> bOnlyRelevantToOwner but no valid owner? -----> YES → NOT Relevant
|
-> If all else fails -> NOT Relevant
```


---
## 5. Ownership
##### 概念关系:
- 影响 Relevancy
- :warning: Ownership 和 Authority 两者不相干, 应避免混淆
  - 在Client端, PlayerState's Role 通常为 `ROLE_SimulatedProxy`, 但是其Owner PlayerController's Role 通常为 `ROLE_AutonomousProxy`
  - 玩家持有的 Weapon Actor, 其 Role 通常为 `ROLE_SimulatedProxy`

#### *What "an Actor is owned by the Client" means ?*

- The actor's *Ownership Chain* ultimately resolves to the `APlayerController` associated with that client’s network connection.




:warning: 注意事项:

- 当一个 Actor 设置了 `bOnlyRelevantToOwner = true` , 但其 **OwnerConnection 链的 root** is not a `PlayerController` 时, 会造成该 Actor **is not relevant to any Clients** 
-  Only the Server can spawn **replicated Actors**, clients are only allowed to spawn `Actors` **locally**

### Relevancy 和 Owernship 是相对独立的Systems
- Relevancy is **essential**,while Ownership is **optional**


| Concept           | **Relevancy**                                        | **Ownership**                                         |
| ----------------- | ---------------------------------------------------- | ----------------------------------------------------- |
| Controls what?    | Which **clients receive replication** for this Actor (Who needs to know this actor) | Which **clients 允许 call Server RPCs** for this Actor  |
| Default behavior  | All replicated Actors are evaluated for relevancy (默认使用 **Distance Policy**)   | Actors have no owner unless you **explicitly set it** |
| Server-side?      | ✅ Yes — server controls it                           | ✅ Yes — server validates and replicates ownership     |
| Client-side use?  | ❌ Clients don’t control it                           | ❌ Clients don’t usually know the ownership chain      |
| Customizable?     | ✅ Yes — override `IsNetRelevantFor()`                | ✅ Yes — use `SetOwner()`                              |
| Affects RPC?      | ❌ No                                                 | ✅✅ Yes — Only **owning clients** can call Server RPCs  |
| Affects lifetime? | ❌ No                                                 | ✅ Yes — Owned actors are destroyed with their owner   |
| Affects Replication? | ✅ Yes                                                | ✅ partially Yes with `bOnlyRelevantToOwner=true` or other custom logic  |

---

## 6. UNetDriver

IN ACTOR
protected:
	UPROPERTY()
	FName NetDriverName;



