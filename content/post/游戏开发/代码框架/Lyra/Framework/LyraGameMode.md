---
created: 2026-02-23T16:19:39+08:00
modified: 2026-03-04T20:55:42+08:00
feature: thumbnails/external/434688f2107c26ebc66bfd5e7ecd6af7.png
thumbnail: thumbnails/resized/0400c91dabb016cd0b2a25f70f65e876_86cf658e.webp
tags:
  - blog
cover: https://pic1.zhimg.com/v2-a75a8fdca2cb1c3ed11f104353ce0940_1440w.jpg
title: LyraGameMode
date: 2026-02-23T08:19:37.905Z
lastmod: 2026-03-06T15:13:38.070Z
---
# 概述

## 核心逻辑

1. 更改ModularGamePlay基类
2. 绝大部分逻辑为加载和配置`ExperienceDefinition`中的所有参数服务\
   ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260306223927683.png)

## 关键思路

### Experience加载阶段

| 阶段                           | 作用                                           | 函数                                               |
| ---------------------------- | -------------------------------------------- | ------------------------------------------------ |
| start                        | 等待一帧,准备加载                                    | GameMode::initgame                               |
| match                        | 按优先级找`ExperienceDefinition`                  | GameMode::HandleMatchAssignmentIfNotExpectingOne |
| set                          | 获得Experience                                 | ExperienceManager::SetCurrentExperience          |
| **Loading**                  | 按服务器/客户端加载Experience中的Actions ActionSet绑定的资产 | StartExperienceLoad                              |
| **LoadingGameFeatures**      | 加载GameFeature                                | OnExperienceLoadComplete                         |
| **LoadingChaosTestingDelay** | Timer延迟测试                                    | OnExperienceFullLoadCompleted                    |
| **ExecutingActions**         | 执行所有Action的回调函数                              | OnExperienceFullLoadCompleted                    |
| **Loaded**                   | 按高中低优先级广播完成Delegate                          | OnExperienceFullLoadCompleted                    |

### Gamemode初始化流程

InitGame → 选副本（Experience）→ 加资源（ActionSet + GameFeature）→ 执行动作（Action）→ 玩家出生（RestartPlayer）→ 同步世界状态（GameState）

* **InitGameState**\
  新增`ULyraExperienceManagerComponent`资源加载完成的回调，方便加载完成后开展Gameplay逻辑(RestartPlayer)
* **InitGame**\
  按优先级查找并初始化Experience，加载资源。
* **OnExperienceLoaded**\
  Experience资源加载完成之后回调，并执行RestartPlayer相关的逻辑。
* **RestartPlayer**\
  引入`ULyraPlayerSpawningManagerComponent`管理PlayerRestart，并接入PawnData

## 参考资料

[# Lyra的数据驱动之数据加载与引用](https://zhuanlan.zhihu.com/p/505766201)\
[Lyra-Experience](https://zhuanlan.zhihu.com/p/1931763088924862388)

# ModularGameplay

更换Gameplay基类为`ModularGamePlay`子类\
![](https://pic4.zhimg.com/v2-8a265fc91262b9620501eeeeb59b1a9b_1440w.jpg)

# ExperienceManager

## 初始化Experience

### Step0 开店准备：绑定ExperienceLoaded回调+调用查找函数

> \[!example]- Experience 就像**一份完整套餐菜单**：\
> 里面包含：
>
> * 做哪些菜
> * 需要哪些设备
> * 厨房要执行哪些步骤

#### 1️⃣ **准备好呼叫系统**

> 如果菜做好了，可以通知服务员。\
> `InitGameState`\
> 绑定 `ExperienceLoaded` 回调 -to> `OnExperienceLoaded`

#### 2️⃣ **服务员问顾客吃什么**

> 餐厅确定：客人点哪套套餐\
> `InitGame`\
> 下一帧找 `Experience` -by> `HandleMatchAssignmentIfNotExpectingOne`

> \[!note]- InitGame与 InitGameState\
> `InitGameState`：在`PreComponentInit`中调用\
> `InitGame`：先于`PreComponentInit`执行。新地图游戏逻辑的开始：场景已经加载完，对象也创建好了。\
> 所以需要等一帧，让`InitGameState`的回调得以绑定\
> ![](https://pica.zhimg.com/v2-41557ed02113a4a4b70d2fba1443c1e4_1440w.jpg)\
> ![](https://pica.zhimg.com/v2-b6459bfaf30b78e6426353a67586fb74_1440w.jpg)

### Step1 选套餐：按优先级寻找Experience

#### 1️⃣**客户选择套餐**

用户比较直，喜欢按顺序点餐，没有则选择更低优先级\
`HandleMatchAssignmentIfNotExpectingOne`\
找到后调用`OnMatchAssignmentGiven`

> \[!note]- Experience顺序：匹配 > URL > PIE > 命令行 > World > Editor默认
>
> * Matchmaking assignment (if present)
> * URL Options override
> * Developer Settings (PIE only)
> * Command Line override
> * World Settings
> * Default experience
>
> > 顾客意愿(比喻不够准确)：隐藏套餐 > 特色套餐 > 时令套餐 > 活动套餐 > 基础套餐

#### 2️⃣**门店后厨获得套餐名**

`ULyraExperienceManagerComponent::SetCurrentExperience`\
现在开始由后厨`ULyraExperienceManagerComponent`管理\
设置套餐为用户所选，后厨开始做菜`StartExperienceLoad`\
![](https://pic1.zhimg.com/v2-8e2937f5f173bcd2cf26114620370dc4_1440w.jpg)

> `StartExperienceLoad`在客户端则由`OnRep_CurrentExperience`调用

### Step2 准备食材：加载和初始化Experience

#### 1️⃣**后厨配置食材**：Actions加载

`StartExperienceLoad`\
按规则加载`Experience`自己和`ActionSet`中所有`GameFeatureAction`配置的资产\
->加载完成后调用`OnExperienceLoadComplete`

> 状态标识：`LoadState`=ELyraExperienceLoadState::Loading

> \[!example]- `ActionSet` 就像固定**备菜清单**\
> 没有`ActionSet`当然也可以，但把重复流程提出来单独放置，减少了查找和新模式一点点加的麻烦
>
> * 切萝卜丝、莴笋丝、土豆丝作为XX炒三丝的 **切丝备菜方案**
> * 切葱、拍蒜、切香菜是很多湘菜的 **固定配料方案**
> * 用料酒、盐、花椒粉腌制里脊肉同样是锅包肉/小炒肉等方案的 **腌制方案**

##### 异步加载Experience引用资产详解

`BundleAssetList`=自己+所有ActionSet\
`BundlesToLoad`：根据Action对资产的标识决定：Server、Client、Equipped(全部)，只需要根据自己所处端让`BundlesToLoad`添加对应标识即可

> \[!example]- 关键截图\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260306014942957.png)\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260306015806382.png)

> \[!info]- Lyra与AssetBounds\
> 资料：[AssetManager系列之Asset Bundles](https://zhuanlan.zhihu.com/p/81155196)\
> Bounds在Lyra中用作区分服务器和客户端资产，可以在加载时便将软引用资产加载。Equipped并未被使用。
>
> > 决定资产所属Bounds(Server/Client)的方式
> >
> > 1. 引用的资产自己决定：AddComp![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260305233050211.png)![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260306011715921.png)
> > 2. `GameFeatureAction`统一决定：AddWidgets添加到Client、Abilities双端、Input双端
>
> 加载代码：\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260305220128805.png)

> \[!important]- **ChangeBundleStateForPrimaryAssets**\
> 用于**动态加载或卸载**指定 Primary Asset 的某些资源 Bundle。![](https://pic1.zhimg.com/v2-ca16cfed06cc53bf441b64442d28ef74_r.jpg)\
> 将指定 PrimaryAsset 的部分资源（通过 Bundle 分组）异步加载进内存\
> 从内存中卸载部分 Bundle（如果 `RemoveBundles` 指定）\
> 支持加载完成回调 `DelegateToCall`\
> 可指定加载优先级，如 `AsyncLoadHighPriority`

> \[!note]- 源代码有但没有用到的内容\
> AssetManager.LoadAssetList与RawAssetList\
> 直接按规则加载Experience自己引用的资产\
> `TSet<FPrimaryAssetId> PreloadAssetList`： 可以存放除了ActionSet以外的主资产

#### 2️⃣后厨准备厨具餐具：`GameFeature`加载

`OnExperienceLoadComplete`\
加载`GameFeature` -> 加载完成后调用`OnExperienceFullLoadCompleted`

> 状态标识：`LoadState`=ELyraExperienceLoadState::LoadingGameFeatures\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260225164146023.png)

> \[!note]- 检查GameFeature全部加载完成方案：回调+计数器\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260225170348507.png)

到这里,Experience加载已经完成,后续便是脏代码以及广播完成消息.

***

#### 3️⃣加载延迟测试

`OnExperienceFullLoadCompleted`\
通过控制台命令参数增加一个Timer延迟

> 状态标识：`LoadState`=ELyraExperienceLoadState::LoadingChaosTestingDelay

```cmd
  lyra.chaos.ExperienceDelayLoad.MinSecs
  lyra.chaos.ExperienceDelayLoad.RandomSecs
```

![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260306192622662.png)

#### 4️⃣检查与善后：执行`GameFeatureAction`

`OnExperienceFullLoadCompleted`

> 状态标识：`LoadState`= ELyraExperienceLoadState::ExecutingActions

一些零零散散的收尾工作\
对Experience和ActionSets中的所有Action手动执行`GameFeatureAction`相关回调\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260225165953409.png)

#### 5️⃣告知完成

OnExperienceFullLoadCompleted\
备菜环节全部完成,按优先级,广播厨师取食材/工具\
所有资源相关的操作都完成,按优先级广播完成的消息

> 状态标识：`LoadState`= ELyraExperienceLoadState::::Loaded;

触发委托Delegate，分高中低。\
![](https://pic4.zhimg.com/v2-31b1140b348afaea6f07429da64951c3_1440w.jpg)

> \[!example]- 不同优先级的回调\
> High Priority
>
> * `ULyraTeamCreationComponent`**队伍**\
>   组队信息创建，PlayerState会注册到队伍中。
> * `ULyraFrontendStateComponent`**UI**\
>   处理带有主菜单界面的Experience Loaded之后的一些处理，主要是打开主菜单界面。
>
> **Normal Priority**
>
> * ★`ALyraGameMode` **Gameplay**\
>   调用了GameModeBase::RestartPlayer，开始创建玩家角色。
> * ★`ALyraPlayerState` **PawnData**\
>   给PlayerState设置PawnData，为角色初始化GAS。
> * `UAsyncAction_ExperienceReady`::Step3\_HandleExperienceLoaded\
>   自定义的异步蓝图方法，用于等待ExperienceLoad后进行一些逻辑。
>
> **Low Priority**
>
> * `ULyraBotCreationComponent`**AI**。
>
> **最低,硬编码**\
> ULyraSettingsLocal 配置和设置\
> 直接写死在Delegate后面,仅客户端

## GameMode的`OnExperienceLoaded`

在`ExperienceManager`中,我们完成了ExperienceDefinine中`Actions` `ActionSet` `GameFeature`三个变量的加载,只差`PawnData`\
回到GameMode处理

### 更改`RestartPlayer`时机到`ExperienceLoaded`

**GameMode**通过在`InitGameState`注册的回调函数，在**Experience**资源加载完后，接入引擎Gameplay逻辑,开始RestartPlayer\
![](https://pica.zhimg.com/v2-288a776f53e6d3b40f7a3c1ecac41dbe_1440w.jpg)\
GameMode在Experience完全加载(Loaded)之前无法生成角色\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260306202620598.png)

> \[!note]- 角色重生逻辑移到了`ULyraPlayerSpawningManagerComponent`管理:\
> ![](https://pica.zhimg.com/v2-e162ce95674f9f38d4ba758f583098ec_1440w.jpg)

## PlayerState的`ExperienceLoaded`

处理`PawnData`

`GetPawnDataForController`获取并设置PawnData

> \[!tip] 的PawnData获取顺序：\
> PlayerState->Experience\
> 一般取的是`Experience`的`PawnData`

添加`PawnData`中的`AbilitySet`：(ALyraPlayerState::SetPawnData)\
![](https://pic2.zhimg.com/v2-602452841a95d5e6990572b6e38b7cd9_1440w.jpg)

> \[!tip]- Gamemode override了必要函数，以传递Lyra的**Pawn**和**PawnData**\
> ![](https://pica.zhimg.com/v2-89beabbb8abccebc82c84b287bdcf526_1440w.jpg)

# GameState

## GameState核心

帮GameMode和其他组件、系统同步属性/消息

* 将服务器帧率`GAverageFPS`同步到客户端
* 放置`ExperienceManagerComponent` 和`AbilitySystemComponent`
* 使用`UGameplayMessage`实现了全新的`MessageBroadcast`系统

## 客户端同步服务器帧率

每帧在服务器获取服务器帧率`GAverageFPS`后属性同步到客户端。\
![](https://pic4.zhimg.com/v2-20d9b1f1700098f6d6bd9f5ae8951615_1440w.jpg)

> \[!note]- GAverageFPS\
> GAverageFPS是引擎本体的全局变量，它在每一帧的FEngineLoop::Tick()中执行相关的计算逻辑。关于引擎的启动流程可以参考[Unreal Engine 的启动流程](https://zhuanlan.zhihu.com/p/610523485) 。\
> 在tick中调用 CalculateFPSTimings 完成FPS的计算。\
> ![](https://pica.zhimg.com/v2-e08c17b6a2699a030282963a493854d4_1440w.jpg)\
> 这里的平均帧率有个小的算法，用之前平均值的0.75 + 当前帧时间的0.25。\
> 那么当前帧的执行时间算法又有所不同，常规的是按照本帧的时间-上帧的时间，但如果开启了垂直同步，并且同步的类型是2（Game线程与GPU的垂直同步时间同步：[UE4 关于主循环的资料](https://zhuanlan.zhihu.com/p/225465983)）的时候去获取RHI的每帧时间。\
> ![](https://pic2.zhimg.com/v2-7994d15b80eb9ce2c16c88d08174aeab_1440w.jpg)

## 提供组件

`ULyraExperienceManagerComponent`在GameMode中讲述

### ULyraAbilitySystemComponent

在GameState上放了一个AbilityComp\
调用位置：`ULyraGamePhaseSubsystem::StartPhase`，用来向客户端同步当前的游戏阶段（**GamePhase**）\
![](https://pic4.zhimg.com/v2-8cb48d5aae3592803ad7d7c8304366f3_1440w.jpg)

## 协助`UGameplayMessage`发送消息

[# GameplayMessageRouter 插件源码分析| Site Unreachable](https://zhuanlan.zhihu.com/p/494820956)\
可以让蓝图与c++针对不指定具体类型的结构体进行交互\
![](https://picx.zhimg.com/v2-42612580808525a939804ce33007c655_1440w.jpg)
