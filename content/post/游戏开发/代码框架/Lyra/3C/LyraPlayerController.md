---
created: 2026-02-23T23:05:35+08:00
modified: 2026-03-04T01:31:16+08:00
feature: thumbnails/external/5b572622d790e7c3ea6cc1f03d77b41c.png
thumbnail: thumbnails/resized/5e59abda4d897665eddf19cd7e2be5c7_86cf658e.webp
tags:
  - UE/Lyra
  - 3C/Controller
  - blog
cover: https://pic1.zhimg.com/v2-dd9962ae48d2adc4780e0e49d137c626_1440w.jpg
title: LyraPlayerController
date: 2026-02-23T15:05:34.478Z
lastmod: 2026-03-03T17:31:17.552Z
---
# PlayerController

## PlayerController概览

除命令行作弊功能，都是外部系统支持，理应放在独立Component中

* ModularPlayerController——ModularGameplay支持
* CommonPlayerController——CommonUI支持
* ILyraCameraAssistInterface——处理TPS相机穿模
* ILyraTeamAgentInterface——Team系统支持\
  ![](https://pica.zhimg.com/v2-f8a52fe28625888b52c7a6ff774e93b0_1440w.jpg)

## 实现

### 命令行作弊

在服务器用命令行作弊\
![](https://pic1.zhimg.com/v2-9999b8c9cd51ef374182694533bfa8a2_1440w.jpg)

### ModularPlayerController

注册自身\
![](https://pic3.zhimg.com/v2-fa6f4a56a46b31688cd8bdcd7baeb6f4_1440w.jpg)\
为子Component提供必要函数：如RecivePlayer、Tick\
![](https://picx.zhimg.com/v2-ab5a2bbcd732d3fb3476e00d35c84029_1440w.jpg)

### CommonPlayerController

扩展了本地玩家（LocalPlayer）在接收到相关的数据之后的后处理，也就是初始化LocalPlayer。\
![](https://pic1.zhimg.com/v2-5452cc3e12ece27b58f6a7e0096152d0_1440w.jpg)\
接收到相关数据之后，对LocalPlayer的回调函数进行广播。\
![](https://pica.zhimg.com/v2-4ee9ba6d9ea6672cab81607aeed2152c_1440w.jpg)

### ILyraCameraAssistInterface

处理场景相机和物体穿插的情况

> \[!note]- 三个接口：
>
> * GetIgnoredActorsForCameraPentration 收集可以让摄像机穿透的Actors
> * GetCameraPreventPenetrationTarget 收集不能让摄像机穿透的Actors
> * OnCameraPenetratingTarget 当穿透的行为发生了之后，事件响应。

### ILyraTeamAgentInterface

由于Lyra是一个红蓝阵营对行的TPS游戏，所以直接在PlayerController里实现了一些跟Team相关的逻辑实现，不细说。\
![](https://pic4.zhimg.com/v2-d3e938873f480fc11a7b6a7991a3a569_1440w.jpg)

# LyraPlayerState

## LyraPlayerState总览

和Controller一样，做各个系统的杂事，往往和网络同步相关

* Framework：ModularGameplay、Exprence的PawnData设置
* GAS：AbilityComp、TagStack用于动态存放int值(子弹、个人分数)
* 团队Id

## 详细实现

### Framework

Modular Gameplay\
让所有的子Modular Component支持执行数据拷贝，override`CopyProperties`\
![](https://pica.zhimg.com/v2-b41550eda38b3df9b0c63940ef74570c_1440w.jpg)

Exprence：设置PawnData\
![](https://pic1.zhimg.com/v2-94324b966897680258a5115d4c272396_1440w.jpg)

### GAS

LyraAbilitySystemComponent: GAS支持\
![](https://picx.zhimg.com/v2-f480bcef580df001850ef92292305e6d_1440w.jpg)

StatTags:存放动态的Int型变量，如子弹数量、分数\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260224000259858.png)

![](https://pic1.zhimg.com/v2-5c105f0b2627b3d95539abfc94ca7760_1440w.jpg)

### Other

TeamId：支持Lyra团队框架\
![](https://pica.zhimg.com/v2-6ef2cfd4636cea5d74dd6ec1d125373a_1440w.jpg)

# Lyra Input

## 配置Input

### Lyra中配置区域：

* Exprience：直接添加配置好的`ActionSet`，同时包含`LyraInputConfig`和`InputMapping`，如：`LAS_ShooterGame_SharedInput`
* GameFeature：仅手动`AddInputMapping`，如\
  `IMC_Default`：移动跳跃蹲伏等不需要做成能力的locomotion\
  `IMC_ShooterGame`：近战瞄准切换武器等能力相关\
  都包含于`ShooterCore`
* PawnData：只有`LyraActionConfig`的配置

### 配置逻辑

Input的复杂性来源于多了两个中间层：

* InputAction：来源于EnhanceInput
* Tag：来源于Lyra

> 映射关系：Input --`InputMapping`-> InputAction --`LyraInputConfig`-> Tag --`BindAbility/NativeAction`->Func/Ability

基于上述两个中间层，需要配置两个映射文件

* `InputMapping`配置`按键`到`InputAction`的映射
* `LyraInputConfig`配置`InputAction`到Tag的映射

#### `InputMapping`

配置`按键`到`InputAction`的映射\
文件以 IMC\_ 开头，如`IMC_ShooterGame`

> EnhanceInput只需要IMC便可绑定Action，也就是：按键->`InputAction`->具体逻辑

![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226170815666.png)

#### `LyraInputConfig`

配置`InputAction` 到`GameplayTag`的绑定\
在`InputAction`和函数/Ability中间加了中间层Tag，InputAction->Tag->Func/Ability

两个数组对应不同的绑定方式：\
`NativeInputAction`：手动绑定函数，如Move、Look\
`AbilityInputAction`：直接激活Ability\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226215230939.png)

### 配置实现

PawnData中配置输入映射表：

* InputConfig:配置`LyraInputConfig`
* 没有配置`InputMapping`的变量\
  ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226192759384.png)

GameFeature/Experience中配置映射表：

* `AddInputMapping`——添加`InputMapping`
* `AddInputBind`——添加`LyraInputConfig`\
  ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226191525990.png)

`AbilitySet`中配置GA和Tag的映射：\
注意：这里和Ability内部都配置了相同的InputTag\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226015327274.png)

Ability内部中配置GA和Tag的映射：

* Ability Triggers：`GameplayTag`->触发Ability
* Activation Policy: 触发方式\
  ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226011225904.png)

## 绑定Input到具体实现

### 直接触发Ability

* 需要在`LyraInputConfig`中添加对应的`AbilityInputAction`

> \[!note]- Ability的输入绑定流程：
>
> 1. `ULyraHeroComponent::InitializePlayerInput`\
>    绑定所有Ability按键到`AbilityInputTagPressed`和`Input_AbilityInputTagReleased`函数![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260227012222004.png)
> 2. 按下时找到所有Tag映射的Ability的handle\
>    ![](https://pic1.zhimg.com/v2-b6d32a3518c2cb2f8eb46b551340121e_r.jpg)
> 3. Tick里检查记录的`AbilityHandle`，获取到`AbilitySpec`
> 4. 调用 `TryActivateAbility`函数激活Ability

### 代码绑定

`ULyraHeroComponent::InitializePlayerInput`：\
需要获取InputComponent、InputConfig、InputTag才能绑定

> \[!tip]- 使用的复杂度\
> 优点当然是解耦，但缺点是换一个项目InputAction就得换，其他获取Input值的代码也需要以上三样才能正常获取输入。无法通过约定俗称的名字默认获取，如"Move\_Forward"、"MoveRight"

![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226012037892.png)\
Input\_Move：\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226012315162.png)\
Input\_Move\_Cancel为空

### 手动触发GA

`B_Hero_ShooterMannequin`:\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226014720421.png)

## 一些代码

### ULyraHeroComponent::InitializePlayerInput

手动绑定Input的代码都在`ULyraHeroComponent::InitializePlayerInput`中，但该函数的第一段代码容易被迷惑\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226170450477.png)\
`DefaultInputMapping`仅用于`B_SimpleHeroPawn`中，并未用于Hero相关的逻辑。\
由此猜测，该变量只是用于不需要复杂配置的简单角色配置。

LyraIC->AddInputMappings(InputConfig, Subsystem)：\
空函数，仅仅做Check

## Lyra中的配置逻辑和情况

### Experience

B\_LyraShooterGame\_ControlPoints包含：`LAS_ShooterGame_SharedInput`![|425](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226013956344.png)\
`LAS_ShooterGame_SharedInput`：\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260226014140487.png)\
InputData\_ShooterGame\_AddOns：

* InputTag.Ability.ShowLeaderboard
* InputTag.Weapon.ADS
* InputTag.Weapon.Grenade
* InputTag.Ability.Emote
* InputTag.Ability.Quickslot.Drop
* InputTag.Ability.Melee

IMC\_ShooterGame：\
IA\_ShowScoreboard\
IA\_ADS\
IA\_Grenade\
IA\_Emote\
IA\_Melee\
IA\_QuickSlot1、2、3、CycleBackward、CycleForward\
获得：\
梳理InputAction错综复杂的文件配置
