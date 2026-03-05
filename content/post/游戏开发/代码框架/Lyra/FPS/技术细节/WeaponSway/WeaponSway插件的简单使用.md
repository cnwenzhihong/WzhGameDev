---
created: 2026-01-08T14:11:26+08:00
modified: 2026-01-08T23:32:10+08:00
feature: thumbnails/external/44f4ca85e952366e8ff66be45cebd8a2.png
thumbnail: thumbnails/resized/46a79b4b8757f644582633f3b6ce861c_b89e22fb.jpg
tags:
  - blog
title: WeaponSway插件的简单使用
date: 2026-01-08T06:11:25.703Z
lastmod: 2026-01-08T15:32:11.458Z
---
# 基本配置

我们以`ABP_WeaponSway`为例，也可以直接将逻辑写在自己的动画蓝图

## ABP\_WeaponSway

找到插件文件夹中的`ABP_WeaponSway`双击，提示没有骨骼，选择自己的FPS骨骼并确认\
打开蓝图，删除报错节点,可以仅保留下图节点：

* SwayOffset：坐标偏移量
* SwayRot：旋转偏移量\
  ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260108135959168.png)\
  添加新变量，变量类型`Weapon Rotation Sway Fragmen`，暂时命名为：`WeaponSwayFragment`,编译后选择一个默认风格，如`WRS_Style_TF2`\
  根据自己的项目连线：

- Target：连入自己写的Fragment参数
- CurrRot：控制器旋转量
- HasInput：当前是否有鼠标输入
- WeaponMesh：用于计算旋转锚点，目前的版本不连可能会崩溃，之后改进，如果用不到可以随便填一个站位(不能为空)
- Invert Roll Pitch：是否交换Roll和Pitch，一般武器坐标系与UE不同，默认开启，如果坐标轴不对可以取消勾选
- Debug：调试，目前仅有显示锚点功能\
  ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260108140158726.png)

## 输入获取

> 注意: Enhanced Input只在有输入的时候Triggered，停止输入时需要自己手动置为0\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260108231828343.png)

## 主动画蓝图

找到我们自己项目的角色蓝图\
ClassDefaults->实现动画接口`ALI_WeaponSway`\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260108134005629.png)\
找到合适的地方链接`ABP_WeaponSway`\
在TwoBoneIK等手部IK之前链接我们的动画接口\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260108134304997.png)\
不同的项目使用的骨骼不同，比如Lyra中是标准IK\_Hand\_Gun，一些项目是Hand\_r/l，根据情况修改`ABP_WeaponSway`中的Modify节点，\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260108141004325.png)\
如果最终效果不明显，可以让SwayRot临时乘以一个数

# 方案选择
