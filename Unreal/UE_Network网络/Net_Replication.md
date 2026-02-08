
Dirty? 
forceupdate?
// todo: conditional replication

---
### FAQ:

###### How ensure multiple related properties arrive / apply together on a client ?
1. Combine properties into a struct and replicate the struct
2. 使用 `FastArraySerializer` for $\text{TArrays}$ of $\text{structs}$.
3. 使用 Custom RPC for atomic updates
4. 使用 OnRep_**StateVersion** 结构

---

# UE Replication 

#### 基础知识

- Replicated Properties are **passive**, **automatic**, and only replicated to **relevant** clients

---

## Variable Replication 

> :books: Replication is based on a TREE, where Actor is the ROOT

- Replicates *State*
- Property Replication 基于*Reliable* UDP, 当发生丢包时, Server会重新发送该属性
  - Server 永远发送 *latest value* => 补发时属性值可能会变 
- Tied to [Relevancy](./_Network总览.md#4-relevancy)

<br>

#### 必要条件 for Making a Property Replicated:

1\. Property's Type is *Replicable*
   - `int`, `bool`, `UStruct` 支持
   - 自定义 `UObject / UObject*` 需要 override `IsSupportedForNetworking()`
     - `UObject*` 是唯一支持同步的 pointer type

2\. Property's Owning Chain roots to an **Replicated Actor** 
   - :pushpin: `UObject` 需要 be registered as ***Replicated Subobject*** of the Actor

3\. Property marked with `Replicated` or `ReplicatedUsing`

4\. Property 在 `GetLifetimeReplicatedProps()` 中被注册 
   - 可定制化注册 *DOREPLIFETIME_CONDITION(...)*

<br>

#### `OnRep_XXX()` Notify

- `RepNotify Event` in **Blueprint** is generally **also called on the Server** for convince, while `OnRep` in C++ is only called on Clients.

- **`OnRep` 的调用时机:** Until client A becomes **relevant** to the changed property, **at that moment** the client receives the update value and `OnRep()` gets called

  - :pushpin: `OnRep` is **triggered AFTER** property update, it sees the new value

:heavy_check_mark: What `OnRep` should do:
- 显示层 refresh, UI 更新
- "Idempotent(可安全重复执行的)" lightweight logic

--- 

## Creation and Destruction 

- Replicates *Existence*

---

## Movement Replication

- Replicates *Transform*

---

