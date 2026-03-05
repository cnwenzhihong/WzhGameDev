---
created: 2026-02-23T16:31:26+08:00
modified: 2026-03-04T01:32:22+08:00
feature: thumbnails/external/1a2535a737b31fb8199089da2b0a50fc.jpg
thumbnail: thumbnails/resized/0e6a9bd03df2e3f53ef531047206aba3_86cf658e.webp
tags:
  - blog
cover: https://pica.zhimg.com/v2-14465b19d25ee29bfe69d834fd9e0110_1440w.jpg
title: Lyra Experience
date: 2026-02-23T08:31:25.450Z
lastmod: 2026-03-03T17:32:23.223Z
---
# 简单描述

## 核心需求

> 将项目中所有用到的类进行分类，并抽象出几种数据格式来描述各种类别。\
> 这些类别可以自由组合来变成一个集合，这个集合可以称为Experience。

很像是Actor和Component的关系——不同的Actor可以拥有不同的Component,不同的数据集可以拥有相同的数据。

* Actions：添加规则——Add能力、组件、UI
* PawnData：配置角色3C——PawnClass、CameraMode、Ability、InputConfig\
  ![](https://pica.zhimg.com/v2-ffe2b16cc70e2d561d4ef92947168e0c_r.jpg)

> \[!tip]- ExprienceData子类\
> ![](https://pic3.zhimg.com/v2-23ce1fa8dee2dffe14d522848a310466_1440w.jpg)

> \[!Tip]- Action类型和PawnData\
> ![](https://picx.zhimg.com/v2-cc9b18f6539c88637e99460b35dc6cb9_1440w.jpg)\
> ![](https://picx.zhimg.com/v2-b523e8cbaa4a1b0a1c3ff66e68f5f111_1440w.jpg)

> \[!Note]- 技能与组件在Lyra中的赋予方式\
> i. **PawnData** AddAbility\
> ii. Actions in **Experiences**\
> iii. Actions in **Game Features**\
> iv. **Equipment**(装备:不同的武器可能执行不同的主动或被动技能)

## 技术点

把数据拿出来必然会需要数据的处理包括，加载卸载等，减轻内存的负担。

## 参考资料

[UE5 Lyra核心概念（一）- Experience](https://zhuanlan.zhihu.com/p/648951586)

# Experience初始化流程

#### 初始化总览

![](https://picx.zhimg.com/v2-17b4ad9e7c989302159b319e2028111f_1440w.jpg)

#### **StartExperienceLoad**

设置Experience时调用\
加载**Experience**资源

#### **OnExperienceLoadComplete**

资源加载完成后调用\
加载GameFeature![](https://pic2.zhimg.com/v2-b45c5f8f4807f078cecdde3aba2a22d7_1440w.jpg)加载完成后通过回调执行GameFeature的初始化逻辑![](https://pic1.zhimg.com/v2-26e15b59d0dfd1ea2f417756ae9785c4_1440w.jpg)

#### OnGameFeaturePluginLoadComplete

每个GameFeature激活完成后自行调用一次\
计数，所有GameFeature完成后调用**OnExperienceLoaded**

#### **OnExperienceLoaded**

所有GameFeature初始化逻辑完成后调用

执行不同优先级的回调函数：

* CallOrRegister\_OnExperienceLoaded\_HighPriority
* CallOrRegister\_OnExperienceLoaded
* CallOrRegister\_OnExperienceLoaded\_LowPriority\
  **OnExperienceFullLoadCompleted**：\
  ![v2-1ac0b25865e1c40a1bae744a38c968af\_1440w.jpg](https://pic4.zhimg.com/v2-1ac0b25865e1c40a1bae744a38c968af_1440w.jpg)开发者可以将自己的回调函数根据实际情况注册为三种之中的某种，来决定它在加载完成之后的回调时机。
