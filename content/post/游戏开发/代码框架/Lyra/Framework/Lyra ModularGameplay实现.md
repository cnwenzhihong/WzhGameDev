---
created: 2026-02-23T22:40:02+08:00
modified: 2026-03-04T00:51:00+08:00
feature: thumbnails/external/740262ed62a64fd45f29c75599b6ccff.png
thumbnail: thumbnails/resized/dd69cab1a4712744fa7df147b1223cf7_86cf658e.webp
tags:
  - blog
cover: https://pic1.zhimg.com/v2-f86d8393daee802886de7daf40aef8c8_1440w.jpg
title: Lyra ModularGameplay实现
date: 2026-02-23T14:40:01.148Z
lastmod: 2026-03-03T16:51:01.612Z
---
# UGameFrameworkComponent

比较简单的Manager+子组件结构\
[《InsideUE5》GameFeatures架构（五）AddComponents](https://zhuanlan.zhihu.com/p/492893002)\
![](https://pic2.zhimg.com/v2-8006b78840008e492ce39f5312b572e3_1440w.jpg)\
ULyraExperienceManagerComponent的基类就是UGameFrameworkComponent

> \[!important] 可以看出，对于常规的管理类，可以用SubSystem，如果需要同步，则可以采用GameState

## Reciver Modular Actor

网络/动态加入管理\
ModularPlayerController里注册GameFrameworkComponent的接收器。\
![](https://pic3.zhimg.com/v2-fa6f4a56a46b31688cd8bdcd7baeb6f4_1440w.jpg)

由于ControllerComponent的逻辑是需要自定义的，所以它必须实现两个函数ReceivedPlayer 和 PlayerTick。前者做Component的初始化，后者做Component的逻辑执行。那么它就要在Controller里覆写这两个函数，调用所有的Component的执行。

![](https://picx.zhimg.com/v2-ab5a2bbcd732d3fb3476e00d35c84029_1440w.jpg)

## AddComponent
