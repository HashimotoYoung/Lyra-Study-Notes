https://dev.epicgames.com/documentation/en-us/unreal-engine/understanding-networked-movement-in-the-character-movement-component-for-unreal-engine

https://zhuanlan.zhihu.com/p/34257208

https://zhuanlan.zhihu.com/p/664568047

## CMC 分析

---


### 几个移动模式

---

FScopedMovementUpdate

---

移动同步解决方案

autonomousProy

simul

---

输入绑定

---

# 📘 Unreal Engine 5: Lyra Architecture Analysis

> **Technical analysis and source code study of Epic Games' "Lyra Starter Game".** > *Targeting: Gameplay Programming, Game Architecture, and System Design.*

---

## 📖 About This Project
This repository documents my deep-dive into the architecture of **Lyra**, Unreal Engine 5's premier example project. 
Rather than just using the features, I analyze the **underlying C++ frameworks** and **design patterns** that drive modern UE5 development.

**Key Focus Areas:**
* **Modular Gameplay:** Decoupling logic using Plugins and Game Features.
* **GAS (Gameplay Ability System):** High-level ability architecture and attribute replication.
* **Advanced Animation:** Thread-safe animation graphs and Linked Anim Layers.
* **Component-Based Design:** How Epic structures code for reusability.

---

## 🗂️ Technical Documentation Index

### 1. Modular Gameplay Architecture
*Analysis of how Lyra manages game modes and feature injection.*
* **[Experience System Deep Dive](Lyra/Lyra_ModularGameplay/Lyra_Experience系统.md)** * *How `LyraExperienceDefinition` drives the loading of Game Features and Action Sets.*
* **[Camera System Architecture](Lyra/Lyra_Camera.md)** * *Breakdown of the Camera Mode stack and how it blends different view states.*

### 2. Gameplay Ability System (GAS)
*Source code analysis of the ability framework.*
* **[Ability System Component (ASC)](Unreal/UE_GAS分析/GAS_AbilitySystemComp.md)** * *Initialization, Attribute Sets, and how Lyra extends the base ASC.*
* *(In Progress)* **Gameplay Tags & Native Gameplay Events**

### 3. Animation Systems
*Study of modern UE5 animation techniques.*
* **[Animation Layering (ItemAnimLayerBase)](Lyra/Lyra_Animation/ABP_ItemAnimLayerBase分析/Basics.md)** * *How Lyra uses Linked Anim Layers to swap weapon animations without changing the main state machine.*

### 4. Networking & Character Movement (CMC)
* *(Coming Soon)* **LyraCharacterMovementComponent Analysis** * *Prediction, specific movement modes, and networking optimization.*
* *(Coming Soon)* **Replication Graph & Bandwidth Management**

---

## 🛠️ Technical Skills Demonstrated
Through this study, I have strengthened my proficiency in:
* **C++ / Blueprint Interop:** Understanding how C++ manages the heavy lifting while exposing flexible APIs to designers.
* **Engine Source Reading:** Navigating and interpreting raw UE5 engine code.
* **System Architecture:** Understanding data-driven design (Data Assets vs. Hard Coding).

---

## 📬 Contact
I am currently seeking opportunities in **Gameplay Programming** (Beijing / Shanghai / Guangzhou / Shenzhen).

* **Email:** `hashimotoyoung[at]163.com`
* **Engine Version:** Unreal Engine 5.2+

---
*Note: This repository is for educational purposes and references code from Epic Games' Lyra Starter Game.*