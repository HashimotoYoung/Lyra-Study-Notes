## GE 中的 "捕获" 机制

### What Capture Data means ?

> :books: **定义:** At a specific moment , Take a **Snapshot** (or a Reference) of Attributes / Tags from source or target ASC, 并缓存在 GESpec 中.

- Captured Data 通常无需进行网络同步, because they are **reconstructible** from replicated data
- **Capture 时机:** 

#### **Capture 的目的** 
- 切断属性或状态之间的**动态依赖**
- To **preserve** the correct gameplay inputs at the correct time so a GE can be evaluated **deterministically, predictably, and safely across time**

---

### bSnapShot 的意义